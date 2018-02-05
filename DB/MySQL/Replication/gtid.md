---
# 基于GTID的复制
---

## Ⅰ、GTID的介绍
- global transaction id identifier 全局事务id
- gtid = server_uuid + transaction_id
- server_uuid是全局唯一的，5.6开始才有，表示当前实例的uuid，保存在数据目录中的auto.conf文件中
- transaction_id是自增的
- gtid的作用是替代filename+ + position

主：show master status;
```
(root@localhost) [test]> show master status;
+------------+----------+--------------+------------------+----------------------------------------+
| File       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set                      |
+------------+----------+--------------+------------------+----------------------------------------+
| bin.000006 |      408 |              |                  | d565cde8-0573-11e8-89b2-525400a4dac1:1 |
+------------+----------+--------------+------------------+----------------------------------------+
1 row in set (0.00 sec)
Executed_Gtid_set：server_id:1-xxxx     表示产生了xxx个事务

从：show master status;
Retrieved_Gtid_Set: d565cde8-0573-11e8-89b2-525400a4dac1:1
Executed_Gtid_Set: d565cde8-0573-11e8-89b2-525400a4dac1:1
也能看到事务号
```

**tips：**

如果做了A和B做了双主，B上一直在同步A上数据，这时候在B上写入一个事务

A上看下Executed_Gtid_set，会发现又两个值

一个是自己做主当前的事务号，一个是同步的从上的事务号

## Ⅱ、GTID的意义
之前的复制基于(file,pos)，当主从发生宕机，切换的时候有问题

slave保存的是原master上的(file,pos)，无法直接指向新master上的(file,pos)

mha通过relay log来判断(非常有技术性)

gtid实现了真正的全局唯一位置(所有机器上都是统一的)

更容易进行failover操作

举例：

a是master，b c d是slave，a挂了，b做主，c d做change master

此时c d 上的pos却还是a上面的pos，和b没有对应关系，文件名，文件大小，position完全不一样，change不起来

使用gtid的话，b上保存这c和d回放的位置G_a、G_b(b是通过选举出来的，保存着最多的日志)

## Ⅲ、gtid配置
```
[mysqld]
log_bin
gtid_mode = ON
log_slave_updates = 1          5.6必须开，5.7可以不开
enforce-gtid-consistency = 1
```

**tips：**

- MySQL5.6必须开启参数log_slave_updates,5.7.6开始无需配置
- MySQL5.6升级到gtid模式需要停机重启
- MySQL5.7.6版本开始可以在线升级gtid模式
- 5.6中gtid用的比较少，最重要的原因在于gtid要么开要么不开，不能做到非gtid升级到gtid
- gtid是一切高可用基础(gr,mha),强烈建议打开，5.6就有了，很成熟了

```
5.7的gtid_mode可选值
ON                      完全打开GTID，如果打开状态的备库接受到不带GTID的事务，则复制中断
ON_PERMISSIVE           可以认为是打开gtid前的过渡阶段，主库在设置成该值后会产生GTID，同时备库依然容忍带GTID和不带GTID的事务
OFF_PERMISSIVE          可以认为是关闭GTID前的过渡阶段，主库在设置成该值后不再生成GTID,备库在接受到带GTID和不带GTID事务都可以容忍
                        主库在关闭GTID时，执行事务会产生一个Anonymous_Gtid事件，会在备库执行：set @@session.gtid_next='anonymous'
OFF                     彻底关闭GTID，如果关闭状态的备库收到带GTID的事务，则复制中断
之前只有ON和OFF
```

## Ⅳ、简单说下搭建过程
大同小异，全备+binlog

开启gtid后，mysqldump备份单库时会报warning，意思是gtid包含所有事务，只备份了单库，忽略即可

用mydumper备份，看下metadata文件，找到gitd：xxxxxx:x-xxx

这玩意等同于mysqdump备份文件中set @@global.gtid_purged='xxxx:x-xxx';

表示这部分gtids对应的事务已经在备份中了，slave在还原备份后复制时，需要跳过这些gtids

```
reset master;   清空@@GLOBAL.GTID_EXECUTED，不然执行下一步会宝座
SET @@GLOBAL.GTID_PURGED = '找出来的位置'
以上操作mysqldump出来的文件导入无需操作，mydumper要手动，因为myloader不执行这个

最后一把change master送给大家
change master to master_host='127.0.0.1', master_port=3306, master_user='rpl', master_password='123', MASTER_AUTO_POSITION=1;
start slave;
```

**tips:**

- binlog文件中会有两个关于gtid的event——Previous_gtids和Gtid
- 通过扫描binlog中的gtid值，可以知道gtid与filename-pos的对应关系，如果binlog很大，扫描量也很大，所以用Previous_gtid来记录之前一个binlog文件中最大的gtid
- 如果要找的gtid比previous_gtids大，就扫描当前文件，反之扫之前的文件，依次类推
- binlog在routate的时候，是知道当前最大gitd的，将该值，写入下个binlog的文件头，即previous_gtids

## Ⅴ、GTID复制中处理报错小技巧
这里模拟一个1062错误即可，不演示

报错会告诉你对应的gtid

操作步骤如下：
- 我们将gtid_next指向报错的gtid

报错中没有gtid，则用Retrieved_Gtid_Set和Executed_Gtid_Set对比一下就知道哪个事务执行出错了

```
(root@localhost) [(none)]> set gtid_next='xxxxxx:xxxx';  # 设置为之前失败的那个GTID的值
Query OK, 0 rows affected (0.00 sec)
```

- 执行一个空事务

```
(root@localhost) [(none)]> begin;commit;
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

- 将gtid_next还原为automatic

```
(root@localhost) [(none)]> set gtid_next="automatic";
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> stop slave;
Query OK, 0 rows affected (0.01 sec)
(root@localhost) [(none)]> start slave;
Query OK, 0 rows affected (0.07 sec)
```

该操作类似于sql_slave_skip_counter，只是跳过错误，不能保证数据一致性，需要人工介入，固强烈建议从机开启read_only=1

## Ⅵ、GTID的限制
- 在开启GTID后，不能在一个事物中使用创建临时表的语句，需要使得 autocommit=1;才可以。5.7开始直接创建临时表已经可以创建了
- 在开启GTID后，不能使用create table select ... 的语法来创建表了，因为这其实是多个事物了，GTID没法对应
