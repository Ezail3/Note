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

可以看到
```


后续关注，现在用不上

