---
# Flashback工具
---

## Ⅰ、背景
早先操作数据误操作后，我们一般通过全量备份+binlog的方式来实现恢复(前滚)

有时只想撤销一个几分钟前的操作，采用这种方式就会显得很笨重

大家都知道Oracle有个叫做flashback的功能，很遗憾MySQL官方并没有提供类似的工具

但姜老师的innosql中实现了这个功能，而且还兼容官方MySQL，目前支持到5.7版本

## Ⅱ、玩两手
关注微信公众号：InsideMySQL，找姜老师要下压缩包即可，直接解压，开箱即用，方便快捷
```
[root@VM_0_5_centos flashback]# ./mysqlbinlog -V
./mysqlbinlog Ver 3.4-InnoSQL for Linux at x86_64
[root@VM_0_5_centos flashback]# ./mysqlbinlog --help |grep flashback
  -B, --flashback     Flashback data to start_postition or start_datetime.
  -E, --fb-event=name only flashback this type of
flashback                         FALSE
```

演示闪回功能
```
(root@localhost) [test]> select version();
+------------+
| version()  |
+------------+
| 5.7.20-log |
+------------+
1 row in set (0.08 sec)

(root@localhost) [test]> show variables like 'binlog_format';
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| binlog_format | ROW   |
+---------------+-------+
1 row in set (0.00 sec)

(root@localhost) [test]> select * from flashback;
+------+------+------+------+
| a    | b    | c    | d    |
+------+------+------+------+
|    1 |    2 |    3 |    4 |
|    2 |    3 |    4 |    5 |
|    3 |    4 |    5 |    6 |
|    4 |    5 |    6 |    7 |
+------+------+------+------+
4 rows in set (0.00 sec)

(root@localhost) [test]> show master status;
+------------+----------+--------------+------------------+-------------------+
| File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------+----------+--------------+------------------+-------------------+
| bin.000001 |     3028 |              |                  |                   |
+------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)

误删数据
(root@localhost) [test]> delete from flashback;
Query OK, 4 rows affected (0.01 sec)

(root@localhost) [test]> select * from flashback;
Empty set (0.00 sec)

[root@VM_0_5_centos flashback]# ./mysqlbinlog --base64-output=decode-rows -v bin.000001 --start-position=3028
截取重点部分如下：
### DELETE FROM `test`.`flashback`
### WHERE
###   @1=1
###   @2=2
###   @3=3
###   @4=4
### DELETE FROM `test`.`flashback`
### WHERE
###   @1=2
###   @2=3
###   @3=4
###   @4=5
### DELETE FROM `test`.`flashback`
### WHERE
###   @1=3
###   @2=4
###   @3=5
###   @4=6
### DELETE FROM `test`.`flashback`
### WHERE
###   @1=4
###   @2=5
###   @3=6
###   @4=7
可以看到删除了4条记录

来看下flashback的核心参数-B，同样截取重点
### INSERT INTO `test`.`flashback`
### SET
###   @1=1
###   @2=2
###   @3=3
###   @4=4
### INSERT INTO `test`.`flashback`
### SET
###   @1=2
###   @2=3
###   @3=4
###   @4=5
### INSERT INTO `test`.`flashback`
### SET
###   @1=3
###   @2=4
###   @3=5
###   @4=6
### INSERT INTO `test`.`flashback`
### SET
###   @1=4
###   @2=5
###   @3=6
###   @4=7
可以看到删除变成插入了

[root@VM_0_5_centos flashback]# ./mysqlbinlog -B -v bin.000001 --start-position=3028 |mysql -S /tmp/mysql3306.sock
(root@localhost) [test]> select * from flashback;
+------+------+------+------+
| a    | b    | c    | d    |
+------+------+------+------+
|    1 |    2 |    3 |    4 |
|    2 |    3 |    4 |    5 |
|    3 |    4 |    5 |    6 |
|    4 |    5 |    6 |    7 |
+------+------+------+------+
4 rows in set (0.00 sec)
恢复成功

tips:
恢复的时候不要加--base64-output=decode-rows，导不进去，没反应
```

## Ⅲ、关于flashback回放的位置点
假设Master宕机切到了Slave，Master恢复后，可能需要将部分数据Flashback掉(宕机前最后一部分未传过去的binlog)，Flashback掉的位置很关键，这个位置一般 以Slave上SQL线程最终回放完的位置为准

## Ⅳ、相关小结
- flashback是基于binlog的逆操作(逻辑)，Oracle的闪回是基于undo做的(物理)

- 使用flashback，binlog_format必须为row，这个之前binlog章节有简单提到过

- binlog_row_image必须设为full

- flashback仅支持DML操作的闪回，不支持ddl

- 同一事务中的DML语句不仅闪回，语句执行顺序也会倒过来(有兴趣的可以测试，篇幅原因只贴了一个delete操作)

**tips：**

①DDL的闪回功能，商业版InnoSQL是支持的，修改了MySQL源码，将删除的库或者表保存在回收站(Recycle Bin Tablespace)

②这里我们分析的是出事之后的挽回，那我们最好的办法就是尽量不要惹事，这里推荐一个参数sql_safe_updates，默认是off的，开启此参数，执行的sql中存在不带where条件的delete和update就会报错1175

最后祝大家，永远不用flashback，一路平安！！！
