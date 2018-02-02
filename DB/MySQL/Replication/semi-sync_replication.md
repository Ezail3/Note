---
# 半同步复制
---

## Ⅰ、认识半同步
我们目前MySQL默认的复制模式是异步复制，主不关心从的数据到哪里了，主宕了，做切换，如果从落后太多，就会导致丢失的数据太多，从5.5版本开始，MySQL引入了半同步复制

简单理解：一个事务提交时，日志至少要保证有一个从接收到，那么它的提交才能继续

### semi-syncreplication

- 至少有一个从机收到binlog再返回
- 减少数据丢失风险
- 不能完全避免数据丢失
- MySQL5.5版本开始支持

### lossless semi-syncreplication

- 二进制日志先写远程
- 可保证数据完全不丢失
- MySQL5.7版本开始支持



semi-sync replication
step1：加载插件
[mysqld]
plugin-dir=/usr/local/mysql/lib/plugin
plugin-load="rpl_semi_sync_master:semisync_master.so;rpl_semi_sync_slave:semisync_slave.so"

在线装
mysql>install plugin rpl_semi_sync_master soname 'semisync_master.so';
mysql>install plugin rpl_semi_sync_slave soname 'semisync_slave.so';

show plugins;
可以看到已经装好了，还多了一些参数
show variables like 'rpl%';

step2：配置
set global rpl_semi_sync_master_enabled = 1;
set global rpl_semi_sync_slave_enabled = 1;
tips:配置文件里都配上哈，主从都搞好
重启一下复制
主上执行show global status like 'rpl%';

这里面有一个超时，默认10s
主发给从，从没接收到，主提交不了，如果超时了，会切为异步
相关参数：rpl_semi_sync_master_timeout

测试：
把从停掉
主上insert，会被hang住
金融行业要超时时间设置很大
