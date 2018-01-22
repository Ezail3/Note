---
# binlog——逻辑复制的基础
---

## binlog定义和作用
### 定义
记录每次数据库的逻辑操作(包括表结构变更和表数据修改)

包含：binlog文件和index文件

### 作用
- 复制：从库读取主库binlog，本地回放实现复制
- 备份恢复：最近逻辑备份数据+binlog实现最大可能恢复
- innodb恢复：开启binlog的情况下，innodb事务提交是二阶段提交，发生crash的时候，innodb中事务有两种状态，一种是commit，一种是prepared，对于prepared状态的事务需要根据binlog来判断是提交还是回滚，以此来保证主从数据一致性

## 不同类型binlog对比
|-|statement|row|mixed|
|:-:|:-:|:-:|:-:|
|说明|记录操作的SQL语句|记录每一行数据的变更|混合模式|
|优点|易于理解|数据一致性高、可flashback|综合上述两种模式|
|缺点|不支持不确定SQL语句|每张表一定要有主键|之前版本bug比较多|
|线上使用|不推荐|推荐|不推荐|

### 再说一遍row
- 优点：记录每一行记录变化，确保证主从数据严格一致性
- 缺点：全表update，delete全表时binlog文件大，所以不建议用MySQL做类似操作；
调成statement看，会发现记录的是sql语句，不说太多，线上基本上不会用

写入数据量很大时，ROW格式下，commit会比较耗时间，因为他还要写binlog（ binlog在提交时才写入 ） 假设更新一张几百万的表，产生的binlog可能会有几百兆，当commit时，写入的数据量就是几百兆，所以会有“阻塞”等待的效果。但其实是在写binlog到磁盘

## 相关参数及使用命令
```
log_bin=bin              默认不打开，和oracle一样，不管事务大小，提交速度都一样)
log_bin_basename         设置binlog名，不设置默认为机器名，直接用上面的log_bin=bin也表示二进制文件以bin开头
binlog_format            之前为statement，5.6有几个小版本用的mixed，5.7开始默认row了
max_binlog_size          限定单个binlog文件大小，默认1G
binlog_do_db
binlog_ignore_db         binlog过滤
sync_binlog              默认是0，binlog文件每次写入内容不会立刻持久化到磁盘，具体持久化是交给操作系统做，固系统崩溃会导致binlog的丢失和不一致，建议设置为1，事务写入到binlog后立即fsync到磁盘
flush binary logs;       新生成一个binlog
show master status;      查看当前的binlog
```

**tips:**

①bin.999999满了之后怎么办？ 前面加1位
②binlog文件可能大于max_binlog_size,原因是一个事务产生的所有事件必须记录在同一个binlog中

## binlog内容
### index文件
有序地记录了当前MySQL服务所使用的所用binlog文件
MySQL运行过程中千万不要骚操作修改index文件，避免出问题

### binlog文件
执行show binlog events in 'xxx'；查看binlog文件内容，不指定文件默认看第一个binlog文件
```
(root@localhost) [test]> show binlog events;
+------------+------+----------------+-----------+-------------+--------------------------------------------------+
| Log_name   | Pos  | Event_type     | Server_id | End_log_pos | Info                                             |
+------------+------+----------------+-----------+-------------+--------------------------------------------------+
| bin.000001 |    4 | Format_desc    |         3 |         123 | Server ver: 5.7.18-log, Binlog ver: 4            |
| bin.000001 |  123 | Previous_gtids |         3 |         154 |                                                  |
| bin.000001 |  154 | Anonymous_Gtid |         3 |         219 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 |  219 | Query          |         3 |         313 | create database test                             |
| bin.000001 |  313 | Anonymous_Gtid |         3 |         378 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 |  378 | Query          |         3 |         474 | use `test`; create table a (a int)               |
| bin.000001 |  474 | Anonymous_Gtid |         3 |         539 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 |  539 | Query          |         3 |         649 | use `test`; create table b (b int) engine=myisam |
| bin.000001 |  649 | Anonymous_Gtid |         3 |         714 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 |  714 | Query          |         3 |         786 | BEGIN                                            |
| bin.000001 |  786 | Table_map      |         3 |         830 | table_id: 219 (test.a)                           |
| bin.000001 |  830 | Write_rows     |         3 |         870 | table_id: 219 flags: STMT_END_F                  |
| bin.000001 |  870 | Xid            |         3 |         901 | COMMIT /* xid=18 */                              |
| bin.000001 |  901 | Anonymous_Gtid |         3 |         966 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 |  966 | Query          |         3 |        1038 | BEGIN                                            |
| bin.000001 | 1038 | Table_map      |         3 |        1082 | table_id: 219 (test.a)                           |
| bin.000001 | 1082 | Update_rows    |         3 |        1128 | table_id: 219 flags: STMT_END_F                  |
| bin.000001 | 1128 | Xid            |         3 |        1159 | COMMIT /* xid=21 */                              |
| bin.000001 | 1159 | Anonymous_Gtid |         3 |        1224 | SET @@SESSION.GTID_NEXT= 'ANONYMOUS'             |
| bin.000001 | 1224 | Query          |         3 |        1296 | BEGIN                                            |
| bin.000001 | 1296 | Table_map      |         3 |        1340 | table_id: 219 (test.a)                           |
| bin.000001 | 1340 | Delete_rows    |         3 |        1380 | table_id: 219 flags: STMT_END_F                  |
| bin.000001 | 1380 | Xid            |         3 |        1411 | COMMIT /* xid=22 */                              |
| bin.000001 | 1411 | Rotate         |         3 |        1452 | bin.000002;pos=4                                 |
+------------+------+----------------+-----------+-------------+--------------------------------------------------+
24 rows in set (0.00 sec)
```
由此可见，binlog是由各类event组成，下面看下分析下event相关内容

|field|含义|
|:-:|:-:|
|(Log_name,Pos)|一个event开始的位置信息|
|End_log_pos|一个event结束的位置信息|
|Event_type|event类型|

①End_log_pos - Pos = 每个event占用的字节数
②show master status; 看到Position就是写到这么多偏移量的地方，也就是这么多个字节，也就是这个binlog文件的大小
③每个binlog前四个字节保留，不写数据

### Event类型分析
|Event_type|含义|
|:-:|:-:|
|Format_desc|一个binlog文件开始，记录server的版本号和二进制日志的版本号，5.7版本固定占119个字节|
|Previous_gtids/Anonymous_Gtid|5.7版本加进来的gtid|
|Query|开始一个sql语句|
|Table_map|操作的哪个库表|
|Write_rows|插入某条记录，具体看不到|
|Delete_rows|删除某条记录|
|Update_rows|更新某条记录|
|Xid|事务提交，可以看到事务号|
|Rotate|一个binlog文件结束,指向下一个event的起始位置（bin.xxx;pos=4）|

再强调，row记录的是每条记录的情况(每次操作的每个记录记下来)，而不是sql语句

## mysqlbinlog工具的使用
### 解析binlog
```
1、mysqlbinlog bin.000001
截取一段：
# at 1224
#171107 10:17:31 server id 3  end_log_pos 1296 CRC32 0xd4d80fa6 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1510021051/*!*/;
BEGIN
/*!*/;
# at 1296
#171107 10:17:31 server id 3  end_log_pos 1340 CRC32 0x73b187fa 	Table_map: `test`.`a` mapped to number 219
# at 1340
#171107 10:17:31 server id 3  end_log_pos 1380 CRC32 0x2e637fcd 	Delete_rows: table id 219 flags: STMT_END_F

BINLOG '
uxcBWhMDAAAALAAAADwFAAAAANsAAAAAAAEABHRlc3QAAWEAAQMAAfqHsXM=
uxcBWiADAAAAKAAAAGQFAAAAANsAAAAAAAEAAgAB//4CAAAAzX9jLg==
'/*!*/;
# at 1380
#171107 10:17:31 server id 3  end_log_pos 1411 CRC32 0x2a6353fd 	Xid = 22
COMMIT/*!*/;

这个解析出来at xxx什么的可以跟前面直接show binlog events对应起来，但是dml的内容有点小难懂，原因是为了方便传输解析出来的每行记录的内容被base64转换了

tips:
mysqlbinlog --base64-output=never xxx    非row格式下只看ddl，加密的dml不显示

2、mysqlbinlog --base64-output=decode-rows -v bin.000001
row格式下可以将密文转为伪sql
同样截取一段
# at 966
#171107 10:17:23 server id 3  end_log_pos 1038 CRC32 0x00be64e0 	Query	thread_id=3	exec_time=0	error_code=0
SET TIMESTAMP=1510021043/*!*/;
BEGIN
/*!*/;
# at 1038
#171107 10:17:23 server id 3  end_log_pos 1082 CRC32 0x5286fd55 	Table_map: `test`.`a` mapped to number 219
# at 1082
#171107 10:17:23 server id 3  end_log_pos 1128 CRC32 0x1ed2714c 	Update_rows: table id 219 flags: STMT_END_F
### UPDATE `test`.`a`
### WHERE
###   @1=1
### SET
###   @1=2
# at 1128
#171107 10:17:23 server id 3  end_log_pos 1159 CRC32 0xa254d40a 	Xid = 21
COMMIT/*!*/;
# at 1159
#171107 10:17:31 server id 3  end_log_pos 1224 CRC32 0x76a7413c 	Anonymous_GTID	last_committed=5	sequence_number=6
SET @@SESSION.GTID_NEXT= 'ANONYMOUS'/*!*/;

看到的是每行记录的内容，@n表示第几列
切记这搞出来的绝对不是sql语句哈，他只管你一行记录的内容，不管你的sql

tips:
-vv 可以看到更详细内容，比如每个列的类型和属性，通常一个v够看
insert和delete记录一整行记录
update记录前项和后项。全表更新会导致二进制日志特别大
```

**问题**：
binlog_format设为row，只知道变化，不知道sql语句，这咋办？

**解决**：
设置参数binlog_rows_query_log_events=1  建议打开
再去看binlog的events，会多一个叫Rows_query的events，它会记录下改变行内容的sql

### 常用参数
- 根据时间点解析
```
--start-datetime='xxx-xx-xx xx:xx:xx'
--stop-datetime='xxx-xx-xx xx:xx:xx'
```
- 根据二进制偏移量解析
```
--start-position=xxx
tips:
这是从xxx来解析，那从xxx+1开始呢？会报error，从这边开始读出来不是一个完整的event，xxx-1开始也是报错，读的时候，每个evnet都有个header，如果不是标准位置就会报错

ERROR: Error in Log_event::read_log_event(): 'read error', data_len: 16640, event_type: 90
ERROR: Could not read entry at offset 1158: Error in log format or read error.

--stop--position=xxx    
    到xxx结束，并不包含xxx这个点
    特殊情况：如果指向了一个Table_map的events,会抛出了一个warning
    
    WARNING: The range of printed events ends with a row event or a table map event that does not have the STMT_END_F flag set. This might be because the last statement was not fully written to the log, or because you are using a --stop-position or --stop-datetime that refers to an event in the middle of a statement. The event(s) from the partial statement have not been written to output.
```

通常通过datetime找position，再来进行恢复

## 通过mysqlbinlog恢复数据
```
mysqlbinlog binlog.00003 |mysql -S /tmp/mysql.sock -f
-f强制跳过错误
只恢复某一段，就加上--start-position或者--start-datetime等
```
官方文档：
如果存在多个二进制日志，并不建议一个一个恢复，而是用下面这个方法
```
mysqlbinlog binlog.[0-9]* |mysql -u root -p
```
一个一个恢复会报danger
说明：如果分两次操作，会被认为在两个session中操作，如果刚好用到一个临时表，一个session退出了，另一个session上去就出错了

另一种方法：
```
mysqlbinlog binlog.000001 > /tmp/statements.sql
mysqlbinlog binlog.000002 >> /tmp/statements.sql
mysql -u root -p -e "source /tmp/statements.sql"
```

## 清理binlog
这里介绍三种清理binlog的方法：
```
法1：purge
purge binary logs to 'xxx';
清理xxxbinlog文件之前的内容
purge binary logs before 'xxx'
清理xxx日期之前的内容

法2：rm
step1：MySQL停止服务
step2：按顺序rm掉binlog文件
step3：编辑index文件，将rm掉的binlog文件从index中去掉

法3：配自动清理参数
[mysqld]
expire_logs_days=N
表示只保存N天的binlog，默认值是0，表示不删除
```

## 其他相关问题
### 增量备份怎么做
通常MySQL不做增量备份，因为MySQL复制本身就是实时在做增量，从库开binlog，在从库上备份binlog即可
Oracle增量备份还是有用的，万一page发生crash，需要把所有日志重做一遍

### row格式的binlog回放
一个sql插了3条记录，其实插了3次，对应3个write_rows,解析这个东西，变相执行3个sql

一个sql删了3条记录，对应的单个delete_rows，回放的时候先根据主键回放，没有主键就找一个索引来回放，如果一个索引没有，会scan全表

如果表中有10w条记录，一个索引没有，你去删全表的话，每条记录删的时候都会扫10w次，复杂度是O(10w^2)次,但因为记录越来越少，最后会扫描10w + （10w * 10w-1）/2 次，所以为什么每张表必须要有主键，这里又是一种体现，有主键回放速度会快很多，特别是delete和update

注意：没主键是不要扯什么row_id，binlog是server层的东西，和row_id没关系

**tips:**

MySQL5.6推出下面这个参数来指定scan算法可以部分解决无主键表导致的复制延迟问题，其基本思路是对于在一个ROWS EVENT中的所有前镜像收集起来，然后在一次扫描全表时，判断HASH中的每一条记录进行更新
slave_rows_search_algorithms，默认值是table_scan，index_scan，另一个hash_scan可配，具体参考官网文档

### flash back
二进制日志能实现一个非常好的功能，用来挽救数据，实现flash back，oracle中还要用到undo

对于insert的event，如果要flash back，就搞成delete，delete搞成insert，update交换前后项即可

听说8.0会支持这个工具，但是现在每家互联网公司都开源自己的工具，实现flashback，但是一定要用row格式的binlog_format

### binlog_cache
binlog默认写入binlog_cache中

**binlog生成的过程**
|步骤|操作|
|:-:|:-:|
|1|binlog被write到各session对应的文件句柄缓存中，也就是标准io缓存|
|2|binlog从每个session私有的缓存中flush到公共缓存中，即操作系统缓存中|
|3|binlog从内存中sync到文件系统，持久化|

第一步session之间互相看不到
第二步每个session之间互相可以看到
在这之前只要机器发生crash，则日志就相对应的丢失了

特殊情况：遇到大事务时，binlog很大，cache放不下就会落盘

```
(root@172.16.0.10) [(none)]> show global status like 'binlog_cache%';
+-----------------------+-------+
| Variable_name         | Value |
+-----------------------+-------+
| Binlog_cache_disk_use | 0     |	-- 记录使用临时文件记录binlog日志的次数(监控项)
| Binlog_cache_use      | 1     |	-- 记录使用缓冲写binlog日志的次数
+-----------------------+-------+	
2 rows in set (0.01 sec)

(root@172.16.0.10) [(none)]> show variables like 'binlog_cache_size';
+-------------------+-------+
| Variable_name     | Value |
+-------------------+-------+
| binlog_cache_size | 32768 |
+-------------------+-------+
1 row in set (0.00 sec)

默认为32K，sessioin级的内存变量，勿设置太大
```

- 生产环境中，我们一般把sync_binlog设为1，让binlog绕过缓存直接落盘，以此来保证数据完整性，所以上面这块binlog_cache内容了解即可
- cache写不下落盘，然后再写binlog，就是两次写磁盘，这样会变慢。若如果参数Binlog_cache_disk_use次数很多，须考虑调大binlog_cache_size,或者检查业务中是否存在大事务(oltp场景尽量大事务拆小事务)
