---
# mysqldump详解
---

## Ⅰ、mysqldump的简单使用与注意点

### 重点
```
--single-transaction 
必须加（一个事务中导出数据，确保产生一致性的备份数据）
```

my.cnf中配上下面配置
```
[mysqldump]
single-transaction
master-data=2
```

### 基本选项
只备份innodb，用不了几个参数，记住下面几个即可，其他的没什么卵用
```
-A 备份所有的database
-B 备份哪几个数据库
-R 备份存储过程(-- routines)
-E 备份定时任务(-- events)
-d 只备份表结构
-w 备份过滤数据
-t 只备份数据
--triggers 备份触发器
--master-data=2 在备份文件中以注释的形式记录备份开始时binlog的position，默认值是1，不注释
```

常用的几种：
```
mysqldump --single-transaction -B test a > backup.sql    备份test库和a库
mysqldump --single-transaction test a > backup.sql       备份test库下的a表
mysqldump --single-transaction test a -w "c=12"> backup.sql
```

其他参数：
```
--lock-tables(-l)
在备份中依次锁住所有表，一般用于myisam备份，备份时数据库只能提供读操作，以此来保证数据一致性，该参数和--single-transaction是互斥的，所以实例中既存在myisam又存在innodb则，只能使用该参数

--lock-all-tables(-x)
比上面的参数力度更大，备份时将整个实例锁住
```

## Ⅱ、mysqldump实现原理剖析

### 开glog嗖哈一把看看嘛(*^__^*) 
```
session 1:
(root@localhost) [(none)]> truncate mysql.general_log;
Query OK, 0 rows affected (0.02 sec)

(root@localhost) [(none)]> set global general_log = 1;
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> set global log_output = 'table';
Query OK, 0 rows affected (0.00 sec)

session 2:
mysqldump --single-transaction --master-data=2 -B dump_test > /tmp/back.sql

session 1:
(root@localhost) [(none)]> set global general_log = 0;
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> set global log_output = 'file';
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> select thread_id,left(argument,64) from mysql.general_log;

(root@localhost) [(none)]> select argument from mysql.general_log where thread_id=239 order by event_time;

FLUSH /*!40101 LOCAL */ TABLES
# 先把表刷一把，减少ftwrl锁的等待时间
FLUSH TABLES WITH READ LOCK
# 把当前整个实例锁成只读
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ
# 设置备份线程事务隔离级别为rr
START TRANSACTION /*!40100 WITH CONSISTENT SNAPSHOT */
# 开启事务
SHOW VARIABLES LIKE 'gtid\_mode'                                                                     
SHOW MASTER STATUS
# 获得当前二进制日志位置
UNLOCK TABLES
# 释放实例级别的只读锁
SELECT LOGFILE_GROUP_NAME, FILE_NAME, TOTAL_EXTENTS, INITIAL_SIZE, ENGINE, EXTRA FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'UNDO LOG' AND FILE_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IS NOT NULL AND LOGFILE_GROUP_NAME IN (SELECT DISTINCT LOGFILE_GROUP_NAME FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('dump_test'))) GROUP BY LOGFILE_GROUP_NAME, FILE_NAME, ENGINE, TOTAL_EXTENTS, INITIAL_SIZE ORDER BY LOGFILE_GROUP_NAME 
SELECT DISTINCT TABLESPACE_NAME, FILE_NAME, LOGFILE_GROUP_NAME, EXTENT_SIZE, INITIAL_SIZE, ENGINE FROM INFORMATION_SCHEMA.FILES WHERE FILE_TYPE = 'DATAFILE' AND TABLESPACE_NAME IN (SELECT DISTINCT TABLESPACE_NAME FROM INFORMATION_SCHEMA.PARTITIONS WHERE TABLE_SCHEMA IN ('dump_test')) ORDER BY TABLESPACE_NAME, LOGFILE_GROUP_NAME                                                                 
SHOW VARIABLES LIKE 'ndbinfo\_version'                                                              
dump_test                                                                                           
SHOW CREATE DATABASE IF NOT EXISTS `dump_test`                                                      
SAVEPOINT sp
# 使用savepoint sp，便于回滚，用于快速释放metadata数据共享锁
show tables                                                                                          
show table status like 'dump\_inno'                                                                 
SET SQL_QUOTE_SHOW_CREATE=1                                                                         
SET SESSION character_set_results = 'binary'                                                        
show create table `dump_inno`                                                                       
SET SESSION character_set_results = 'utf8'                                                          
show fields from `dump_inno`                                                                        
show fields from `dump_inno`                                                                        
SELECT /*!40001 SQL_NO_CACHE */ * FROM `dump_inno`                                                  
SET SESSION character_set_results = 'binary'                                                        
use `dump_test`                                                                                     
select @@collation_database                                                                         
SHOW TRIGGERS LIKE 'dump\_inno'                                                                     
SET SESSION character_set_results = 'utf8'
# 以上部分备份数据
ROLLBACK TO SAVEPOINT sp                                                                             
RELEASE SAVEPOINT sp
# 回到savepoint sp，释放metadata的锁
# 每取一张表的数据，就rollback to savepoint sp（一个savepoint就够了）

root@localhost on  using Socket                                                                     
/*!40100 SET @@SQL_MODE='' */                                                                       
/*!40103 SET TIME_ZONE='+00:00' */                                                                  
```

### mysqldump流程小结
|-|操作|解析|
|:-:|:-:|:-:|
|step1|flush tables/flush tables with read lock|将实例锁成只读|
|step2|SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ/START TRANSACTION|开启rr隔离级别的事务|
|step3|SHOW MASTER STATUS/UNLOCK TABLES|获取二进制日志位置并释放全局读锁|
|step4|SAVEPOINT sp|设置回滚点|
|step5|select xxx|备份数据|
|step6|ROLLBACK TO SAVEPOINT sp/RELEASE SAVEPOINT sp|结束一张表备份，回到sp|

### 关键点细说

- start transaction with consistent snapshot

事务隔离级别是rr，开始事务，并且马上创建一个read_view,所以mysqldump备份的数据是备份开始时候的数据而不是备份结束时的数据(备份了30min，整个过程实例一直可读可写，备份的是30min之前的数据而不是30min之后的数据)，位置点问题

执行start transaction同时（而不是等到执行第一条sql）建立与本事务一致性读的snapshot

- --single-transaction

所有的数据库都是在一个事务里面读出来，而且事务隔离级别是如rr的，所以读到的数据是一致的
一致性备份：整个备份从start transaction开始，备份所有的表，所有的表的数据都是在一个事务里面，通过select导出来

- savepoint保存点（4.1还是5.0加进来的）

savepoint很少用，真正用的最多就是备份的时候，一张表备份完，会回滚到对应保存点，此时对应备份的表上面的元数据锁都释放，这时候可以这个表可以做ddl操作。

否则在一个事务里，持有元数据锁，要做ddl(比如其他线程想对这里一个表创建索引)，即使备份完了也做不了，要等所有备份结束才能动

没有savepoint时，只读锁要持有整个事务时间，而不是表备份的时间

## 常规操作

①备份并且压缩

mysqldump --single-transaction --master-data=1 --triggers -R -E -B sbtest | pv | gzip -c > sbtest.backup.tgz
压缩过的备份恢复
gunzip < sbtest.backup.tgz | mysql

②备份并且压缩到远程服务器

mysqldump --single-transaction --master-data=1 --triggers -R -E -B sbtest | gzip -c | ssh root@test-3 'cat > /tmp/sbtest.sql.gz'
备份校验，另行考虑

③备份文件使用
mysql < xxx.sql;

**tips:**

备份占用带宽很大，需要调度算法确保同一个集群中同时只有一个机器做备份，或者不每天做备份从而错开备份时间
