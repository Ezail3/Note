---
# mysqldump详解
---

## Ⅰ、mysqldump的简单使用与注意点

### 重点
```
--single-transaction 
必须加（一个事物中导出数据，确保产生一致性的备份数据）
```

my.cnf中配上下面配置
```
[mysqldump]
single-transcation
master-data=2
```

### 日常选项
只备份innodb，用不了几个参数，记住下面几个即可，其他的没什么卵用
```
-A 备份所有的database
-B 备份哪几个数据库
-R 备份存储过程(-- routines)
-E 备份定时任务(-- events)
-d 备份表结构
-w 备份过滤数据
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
开glog嗖哈一把看看嘛(*^__^*) 
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
