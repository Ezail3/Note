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
-B 指定数据库
-T 指定表
-C 压缩
-F --chunk-filesize 指定文件大小
--rows 100000   每10w行导出到一个文件
```

## Ⅳ、玩两手
```
mydumper -G -E -R --trx-consistency-only -t 4 -C -B dbt3 -o /mdata/backup
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

进入备份目录
发现基于每张表备份并产生压缩文件，所以恢复的时候可以指定某张表恢复
metadata文件    master-data=1    记录二进制日志位置
打开压缩文件
-schema.sql文件        每张表的表结构
.sql                            数据文件
-schema-create.sql.gz文件    创建库

恢复：
myloader恢复
-d 恢复文件目录
-t 指定线程数
-B 指定库
myloader -d backup_20170511 -t 4 -B sb
SSD上开4线程比source单线程快将近两倍(hdd盘可能性能提升会受一定影响)

mydumper原理：
核心问题：并行怎么做到的？一张表都能并行导出，还要保持一致性
演示一个例子解释并行：
主线程session1:
flush tables with read lock;
整个数据库锁成只读，其他线程只能读，不能写

其他线程切换到事务隔离级别为rr
session2：
start transaction with consistent snapshot;

session3：
start transaction with consistent snapshot;

session4：
start transaction with consistent snapshot;

session1：
unlock tables;

其他线程开始做备份

多个线程看到的数据是一致的，select各个表，搞出来的数据是一致的

一张表怎么并行（前提是表中必须有唯一索引）：
先检测唯一索引，根据唯一索引对表进行分片再进行备份，提前切好，区间先算好(不是每个区间相等)，show processlist;中可以看出来
