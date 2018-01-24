---
# mysqlpump工具简单介绍
---

## Ⅰ、功能分析

### 多线程介绍
```
		    +-----------+
		    | mysqlpump |
		    +-----+-----+
			  |
        +-----------------------------------+
	|	          |                 |
    +---v---+         +---V---+         +---v---+
    |default|         |  DB1  |         |  DB2  |
    | queue |         | queue |         | queue |
    +---+---+         +---+---+         +---+---+
	|                 |                 |
        V	          V                 V
+-------+-------+ +-------+-------+ +-------+-------+
|+----+   +----+| |+----+   +----+| |+----+   +----+|
||thd1|...|thd3|| ||thd1|...|thd3|| ||thd1|...|thd3||
|+----+   +----+| |+----+   +----+| |+----+   +----+|
+---------------+ +---------------+ +---------------+
```

- mysqlpump是MySQL5.7的官方工具，用于取代mysqldump，其参数与mysqldump基本一样

- mysqlpump是多线程备份，但只能到表级别，单表备份还是单线程

- mysqldump备份时，有个默认队列（default），队列下开N个线程去备份数据库/数据库中的表

- 支持开多个队列(对应不同库/表)，然后每个队列设置不同线程，进行备份

### 优缺点

**优点：**

- 官方工具，听着牛逼

**缺点：**

- 只能并行到表级别，如果表特别大，开多线程和单线程是一样的，并行度不如mydumper

- 无法获取当前备份对应的binlog位置

### 重要参数
```
--default-parallelism	指定线程数，默认开2个线程进行并发备份

--parallel-schemas	指定哪些数据库进行并发备份

--set-gtid-purged=OFF   5.7.18后加入的参数，
```

## Ⅱ演示一手
```
[root@VM_0_5_centos ~]# mysqlpump --single-transaction --set-gtid-purged=OFF --parallel-schemas=2:employees --parallel-schemas=4:dbt3 -B employees dbt3 > /tmp/backup.sql
mysqlpump: [Warning] Using a password on the command line interface can be insecure.
Dump progress: 1/5 tables, 0/7559817 rows
Dump progress: 3/15 tables, 286750/12022332 rows
Dump progress: 3/15 tables, 686750/12022332 rows
Dump progress: 3/15 tables, 1042250/12022332 rows
...
Dump completed in 43732 milliseconds

新开一个会话看下情况
(root@172.16.0.10) [(none)]> show processlist;
+--------+------+------------------+------+---------+------+-------------------+------------------------------------------------------------------------------------------------------+
| Id     | User | Host             | db   | Command | Time | State             | Info                                                                                                 |
+--------+------+------------------+------+---------+------+-------------------+------------------------------------------------------------------------------------------------------+
| 138199 | root | 172.16.0.5:39238 | NULL | Query   |    0 | starting          | show processlist                                                                                     |
| 138267 | root | 172.16.0.5:39776 | NULL | Sleep   |    2 |                   | NULL                                                                                                 |
| 138268 | root | 172.16.0.5:39778 | NULL | Query   |    2 | Sending to client | SELECT SQL_NO_CACHE `emp_no`,`dept_no`,`from_date`,`to_date`  FROM `employees`.`dept_emp`            |
| 138269 | root | 172.16.0.5:39780 | NULL | Query   |    2 | Sending to client | SELECT SQL_NO_CACHE `emp_no`,`birth_date`,`first_name`,`last_name`,`gender`,`hire_date`  FROM `emplo |
| 138270 | root | 172.16.0.5:39782 | NULL | Query   |    2 | Sending to client | SELECT SQL_NO_CACHE `o_orderkey`,`o_custkey`,`o_orderstatus`,`o_totalprice`,`o_orderDATE`,`o_orderpr |
| 138271 | root | 172.16.0.5:39784 | NULL | Query   |    2 | Sending to client | SELECT SQL_NO_CACHE `p_partkey`,`p_name`,`p_mfgr`,`p_brand`,`p_type`,`p_size`,`p_container`,`p_retai |
| 138272 | root | 172.16.0.5:39786 | NULL | Query   |    2 | Sending data      | SELECT SQL_NO_CACHE `l_orderkey`,`l_partkey`,`l_suppkey`,`l_linenumber`,`l_quantity`,`l_extendedpric |
| 138273 | root | 172.16.0.5:39788 | NULL | Query   |    2 | Sending to client | SELECT SQL_NO_CACHE `c_custkey`,`c_name`,`c_address`,`c_nationkey`,`c_phone`,`c_acctbal`,`c_mktsegme |
| 138274 | root | 172.16.0.5:39790 | NULL | Sleep   |    2 |                   | NULL                                                                                                 |
| 138275 | root | 172.16.0.5:39792 | NULL | Sleep   |    1 |                   | NULL                                                                                                 |
+--------+------+------------------+------+---------+------+-------------------+------------------------------------------------------------------------------------------------------+
10 rows in set (0.00 sec)

可以看到138268和138269在备份employees库，138270，138271，138272，138273在备份dbt3，这里没打印全，不过这是真的，不吹牛逼
```

## Ⅲ、看下备份过程吧
```
session1:
(root@localhost) [(none)]> truncate mysql.general_log;
Query OK, 0 rows affected (0.10 sec)

(root@localhost) [(none)]> set global log_output = 'table';
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> set global general_log = 1;
Query OK, 0 rows affected (0.03 sec)

session2:
[root@VM_0_5_centos ~]# mysqlpump --single-transaction abc > /tmp/backup.sql
Dump completed in 592 milliseconds

(root@localhost) [(none)]> select thread_id,left(argument, 64) from mysql.general_log order by event_time;
省略部分输出：
+-----------+------------------------------------------------------------------+
|         7 | root@localhost on  using Socket                                  |
|         7 | FLUSH TABLES WITH READ LOCK                                      |
|         7 | SHOW WARNINGS                                                    |
|         7 | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ          |
|         7 | SHOW WARNINGS                                                    |
|         7 | START TRANSACTION WITH CONSISTENT SNAPSHOT                       |
|         7 | SHOW WARNINGS                                                    |
|         8 | root@localhost on  using Socket                                  |
|         8 | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ          |
|         8 | SHOW WARNINGS                                                    |
|         8 | START TRANSACTION WITH CONSISTENT SNAPSHOT                       |
|         8 | SHOW WARNINGS                                                    |
|         9 | root@localhost on  using Socket                                  |
|         9 | SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ          |
|         9 | SHOW WARNINGS                                                    |
|         9 | START TRANSACTION WITH CONSISTENT SNAPSHOT                       |
|         9 | SHOW WARNINGS                                                    |
|         7 | UNLOCK TABLES                                                    |
|         7 | SHOW WARNINGS                                                    |
|         9 | SET SQL_QUOTE_SHOW_CREATE= 1                                     |
|         9 | SHOW WARNINGS                                                    |
|         9 | SET TIME_ZONE='+00:00'                                           |
|         8 | SET SQL_QUOTE_SHOW_CREATE= 1                                     |
|         8 | SHOW WARNINGS                                                    |
|         8 | SET TIME_ZONE='+00:00'                                           |
|         3 | set global general_log = 0                                       |
+-----------+------------------------------------------------------------------+
1.线程7 进行 FLUSH TABLES WITH READ LOCK 。对表加一个读锁
2.线程7、8、9分别开启一个事物（RR隔离级别）去备份数据，由于之前锁表了，所以这三个线程备份出的数据是具有一致性的
3.线程7 解锁 UNLOCK TABLE
整个过程没有获取二进制位置点
```

## Ⅳ、compress-output
mysqlpump支持压缩输出，支持LZ4和ZLIB（ZLIB压缩比相对较高，但是速度较慢）
```
[root@VM_0_5_centos tmp]# mysqlpump --single-transaction --compress-output=lz4 abc > /tmp/backup_abc.sql
Dump completed in 511 milliseconds
```
## Ⅴ、备份恢复
未压缩的备份
```
mysql < backup.sql
```
压缩过的备份
```
先解压
zlib_decompress
lz4_decompress
lz4_decompress backup_abc.sql backup.sql
再导入
mysql < backup.sql
```
可以看出来，这个导入是单线程

后续关注，现在用不上
