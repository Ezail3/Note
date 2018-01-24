---
# mydumper浅析
---

## Ⅰ、背景

- mysqldump单线程备份，很慢
- 恢复慢，一张表一张表恢复，
- 如果备份了100G的数据，想恢复其中一个表，做不到(所有的表都在一个文件里)

所以推荐使用mydumper备份

- 备份并行，基于行，即使一张表也能并行，好强呐
- 恢复也是并行
- 恢复的时候可以只恢复指定表

完美(*^__^*) 

## Ⅱ、安装
```
yum install -y  glib2-devel mysql-devel zlib-devel pcre-devel openssl-devel cmake gcc gcc-c++
cd /usr/local/src
git clone https://github.com/maxbube/mydumper
cd mydumper
cmake .
make -j 4
make install
export LD_LIBRARY_PATH="/usr/local/mysql/lib:$LD_LIBRARY_PATH"
```
## Ⅲ、参数介绍
参数和mysqldump很多一样
```
-G --triggers
-E --events
-R --routines
--trx-consistency-only    等于--single-transaction
-t 开几个线程，默认4个
-o 备份到指定目录
-x 正则匹配
-c 压缩
-B 指定数据库
-T 指定表
-F --chunk-filesize 指定文件大小
--rows 100000   每10w行导出到一个文件
```

## Ⅳ、玩两手

### 备份
```
[root@VM_0_5_centos backup]# mydumper -G -E -R --trx-consistency-only -t 4 -c -B dbt3 -o /mdata/backup
另开一个会话看下show processlist;可以看到四个线程
(root@172.16.0.10) [(none)]> show processlist;
+--------+------+------------------+------+---------+------+-------------------+----------------------------------------------------------+
| Id     | User | Host             | db   | Command | Time | State             | Info                                                     |
+--------+------+------------------+------+---------+------+-------------------+----------------------------------------------------------+
| 137488 | root | 172.16.0.5:53046 | NULL | Query   |    0 | starting          | show processlist                                         |
| 137523 | root | 172.16.0.5:53546 | NULL | Query   |    3 | Sending to client | SELECT /*!40001 SQL_NO_CACHE */ * FROM `dbt3`.`customer` |
| 137524 | root | 172.16.0.5:53548 | NULL | Query   |    3 | Sending to client | SELECT /*!40001 SQL_NO_CACHE */ * FROM `dbt3`.`lineitem` |
| 137525 | root | 172.16.0.5:53550 | NULL | Query   |    1 | Sending to client | SELECT /*!40001 SQL_NO_CACHE */ * FROM `dbt3`.`partsupp` |
| 137526 | root | 172.16.0.5:53552 | NULL | Query   |    3 | Sending to client | SELECT /*!40001 SQL_NO_CACHE */ * FROM `dbt3`.`orders`   |
+--------+------+------------------+------+---------+------+-------------------+----------------------------------------------------------+
5 rows in set (0.00 sec)
```

**tips：**

mydumper 参数和其所跟的参数值不能连在一起，不然会报错
```
option parsing failed: Error parsing option -r, try --help
```

### 分析备份内容
进入备份目录
```
[root@VM_0_5_centos backup]# ll
total 1200340
ll
total 305044
-rw-r--r-- 1 root root       281 Jan 24 10:41 dbt3.customer-schema.sql.gz
-rw-r--r-- 1 root root   9173713 Jan 24 10:41 dbt3.customer.sql.gz
-rw-r--r-- 1 root root       401 Jan 24 10:41 dbt3.lineitem-schema.sql.gz
-rw-r--r-- 1 root root 221097124 Jan 24 10:42 dbt3.lineitem.sql.gz
-rw-r--r-- 1 root root       228 Jan 24 10:41 dbt3.nation-schema.sql.gz
-rw-r--r-- 1 root root      1055 Jan 24 10:41 dbt3.nation.sql.gz
-rw-r--r-- 1 root root       294 Jan 24 10:41 dbt3.orders-schema.sql.gz
-rw-r--r-- 1 root root  47020810 Jan 24 10:41 dbt3.orders.sql.gz
-rw-r--r-- 1 root root       264 Jan 24 10:41 metadata

篇幅有限未将所有表列出来
```
发现基于每张表备份并产生压缩文件，所以恢复的时候可以指定某张表恢复

喽一眼
```
[root@VM_0_5_centos backup]# cat metadata
Started dump at: 2018-01-24 10:35:50
SHOW MASTER STATUS:
	Log: bin.000001
	Pos: 154
	GTID:

Finished dump at: 2018-01-24 10:35:50
```

metadata文件    master-data=1    记录二进制日志位置

打开压缩文件
```
[root@VM_0_5_centos backup]# gunzip dbt3.customer-schema.sql.gz dbt3.customer.sql.gz dbt3-schema-create.sql.gz

[root@VM_0_5_centos backup]# cat dbt3-schema-create.sql 
CREATE DATABASE `dbt3` /*!40100 DEFAULT CHARACTER SET utf8mb4 */;

[root@VM_0_5_centos backup]# cat dbt3-schema-create.sql 
CREATE DATABASE `dbt3` /*!40100 DEFAULT CHARACTER SET utf8mb4 */;
[root@VM_0_5_centos backup]# cat dbt3.customer-schema.sql
/*!40101 SET NAMES binary*/;
/*!40014 SET FOREIGN_KEY_CHECKS=0*/;

CREATE TABLE `customer` (
  `c_custkey` int(11) NOT NULL,
  `c_name` varchar(25) DEFAULT NULL,
  `c_address` varchar(40) DEFAULT NULL,
  `c_nationkey` int(11) DEFAULT NULL,
  `c_phone` char(15) DEFAULT NULL,
  `c_acctbal` double DEFAULT NULL,
  `c_mktsegment` char(10) DEFAULT NULL,
  `c_comment` varchar(117) DEFAULT NULL,
  PRIMARY KEY (`c_custkey`),
  KEY `i_c_nationkey` (`c_nationkey`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1;

[root@VM_0_5_centos backup]# head -5 dbt3.customer.sql
/*!40101 SET NAMES binary*/;
/*!40014 SET FOREIGN_KEY_CHECKS=0*/;
/*!40103 SET TIME_ZONE='+00:00' */;
INSERT INTO `customer` VALUES
(1,"Customer#000000001","j5JsirBM9PsCy0O1m",15,"25-989-741-2988",711.56,"BUILDING","regular, regular platelets are fluffily according to the even attainments. blithely iron"),

```

综上：

|文件|作用|
|:-:|:-:|
|-schema.sql|每张表的表结构|
|.sql|数据文件|
|-schema-create.sql.gz|创建库| 

### 恢复
恢复使用myloader命令
```
-d 恢复文件目录
-t 指定线程数
-B 指定库
[root@VM_0_5_centos mdata]# myloader -d /mdata/backup -t 4 -B test
```

**tips:**

SSD上开4线程比source单线程快将近两倍(hdd盘可能性能提升会受一定影响)

## Ⅴ、mydumper原理：
这里有了mysqldump的基础就不开glog详细分析了

核心问题：并行怎么做到的？一张表都能并行导出，还要保持一致性

step1：

主线程session1:

flush tables with read lock;

整个数据库锁成只读，其他线程只能读，不能写，针对myisam做的

start transaction with consistent snapshot

开启一致性快照事务，针对innodb做的

show master status

获取二进制文件位置点

step2：

主线程创建执行备份任务的子线程并切换到事务隔离级别为rr

session2：

start transaction with consistent snapshot;

session3：

start transaction with consistent snapshot;

session4：

start transaction with consistent snapshot;

这样多个线程读到的内容是一致的

step3：

备份no-innodb

step4:

session1：

unlock tables;

step5:

备份innodb至备份结束

从整个流程来看，多个线程看到的数据是一致的，所以select各个表，搞出来的数据是一致的，其实就是利用了mvcc的特性

**问题:**

一张表怎么并行？

- 单表并行的前提是表中必须有唯一索引,且唯一索引必须是整型，不能是复合索引
- 先检测唯一索引，根据唯一索引对表进行分片再进行备份，提前切好，区间先算好(不是每个区间相等)，show processlist;中可以看出来
