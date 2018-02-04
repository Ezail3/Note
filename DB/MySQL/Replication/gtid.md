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

change master to master_host='127.0.0.1', master_port=3306, master_user='rpl', master_password='123', MASTER_AUTO_POSITION=1;
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
