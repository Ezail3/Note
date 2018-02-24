---
# 多源复制与级联复制
---

## 一、多源复制

### 多源复制的应用场景

- 多个数据库实例的数据需要合并统计分析

- 多个实例的数据放到一台机器备份

### 多源复制的限制

- MySQL5.7.6开始才支持多源复制

- 一个从库最多支持64个主库

- 每多一个主，从上会多创建两个slave线程

- 每个主库server_id必须不一致

- 多源复制只能有一个复制是半同步

- 尽量保证每个主同步过来的库名不要一样

### 部署
在语法层面上，只是在 原来的change master的基础上，增加了for channel 'channel_name'

前面的备份还原步骤略过，只写重要步骤

**step1:**

slave端配置复制的库，不然会有冲突
[mysqld]
replicate_do_db=db1
replicate_do_db=db2

**step2:**

```
第一个主：
mysql> change master to master_host='127.0.0.1', master_port=3006, master_user='rpl', master_password='123', master_auto_position=1 for channel 'ch1'; -- 多源复制 channel ：ch1

mysql> start slave for channel 'ch1'; -- 启动ch1
Query OK, 0 rows affected (0.03 sec)

mysql> show slave status for channel 'ch1'\G

第二个主类似
```

## 二、级联复制
