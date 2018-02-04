---
# 半同步复制
---

## Ⅰ、认识半同步
我们目前MySQL默认的复制模式是异步复制，主不关心从的数据到哪里了，主宕了，做切换，如果从落后太多，就会导致丢失的数据太多，从5.5版本开始，MySQL引入了半同步复制

简单理解：一个事务提交时，日志至少要保证有一个从接收到，那么它的提交才能继续

到5.7版本，在原来半同步的基础上又出了一种半同步，叫无损复制

### semi-syncreplication

- 至少有一个从机收到binlog再返回
- 减少数据丢失风险
- 不能完全避免数据丢失
- MySQL5.5版本开始支持

### lossless semi-syncreplication

- 二进制日志先写远端
- 可保证数据完全不丢失
- MySQL5.7版本开始支持

## Ⅱ、配置半同步复制
semi-sync replication
```
step1：
在线加载插件安装
(root@localhost) [(none)]> install plugin rpl_semi_sync_master soname 'semisync_master.so';
(root@localhost) [(none)]> install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

写入配置文件
[mysqld]
plugin-dir=/usr/local/mysql/lib/plugin
plugin-load="rpl_semi_sync_master:semisync_master.so;rpl_semi_sync_slave:semisync_slave.so"

(root@localhost) [(none)]> show plugins;
截取一段
+----------------------------+----------+--------------------+--------------------+---------+
| rpl_semi_sync_master       | ACTIVE   | REPLICATION        | semisync_master.so | GPL     |
| rpl_semi_sync_slave        | ACTIVE   | REPLICATION        | semisync_slave.so  | GPL     |
+----------------------------+----------+--------------------+--------------------+---------+

可以看到已经装好了，还多了一些参数
(root@localhost) [(none)]> show variables like 'rpl%';
+-------------------------------------------+------------+
| Variable_name                             | Value      |
+-------------------------------------------+------------+
| rpl_semi_sync_master_enabled              | OFF        |
| rpl_semi_sync_master_timeout              | 10000      |
| rpl_semi_sync_master_trace_level          | 32         |
| rpl_semi_sync_master_wait_for_slave_count | 1          |
| rpl_semi_sync_master_wait_no_slave        | ON         |
| rpl_semi_sync_master_wait_point           | AFTER_SYNC |
| rpl_semi_sync_slave_enabled               | OFF        |
| rpl_semi_sync_slave_trace_level           | 32         |
| rpl_stop_slave_timeout                    | 31536000   |
+-------------------------------------------+------------+
9 rows in set (0.00 sec)

step2：
(root@localhost) [(none)]> set global rpl_semi_sync_master_enabled = 1;
(root@localhost) [(none)]> set global rpl_semi_sync_slave_enabled = 1;

tips:配置文件里都配上哈，主从都搞好

重启一下复制
主上执行
(root@localhost) [(none)]> show global status like 'rpl%';
+--------------------------------------------+-------+
| Variable_name                              | Value |
+--------------------------------------------+-------+
| Rpl_semi_sync_master_clients               | 1     |
| Rpl_semi_sync_master_net_avg_wait_time     | 0     |
| Rpl_semi_sync_master_net_wait_time         | 0     |
| Rpl_semi_sync_master_net_waits             | 0     |
| Rpl_semi_sync_master_no_times              | 0     |
| Rpl_semi_sync_master_no_tx                 | 0     |
| Rpl_semi_sync_master_status                | ON    |
| Rpl_semi_sync_master_timefunc_failures     | 0     |
| Rpl_semi_sync_master_tx_avg_wait_time      | 0     |
| Rpl_semi_sync_master_tx_wait_time          | 0     |
| Rpl_semi_sync_master_tx_waits              | 0     |
| Rpl_semi_sync_master_wait_pos_backtraverse | 0     |
| Rpl_semi_sync_master_wait_sessions         | 0     |
| Rpl_semi_sync_master_yes_tx                | 0     |
| Rpl_semi_sync_slave_status                 | OFF   |
+--------------------------------------------+-------+
15 rows in set (0.03 sec)
```

这里面有一个超时，默认10s

主发给从，从没接收到，主提交不了，如果超时了，会切为异步

相关参数：rpl_semi_sync_master_timeout


**测试一把：**

把从停掉，主上insert，会被hang住，金融行业要超时时间设置很大
