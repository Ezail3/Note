---
# 备份基础简单介绍
---

## Ⅰ、备份类型

这里介绍三种全量备份

**热备(hot backup)**

- 在线备份
- 对应用基本无影响(应用程序读写不会阻塞，但是性能还是会又下降，所以尽量不要在主上做备份，在从库上做)

**冷备(cold backup)**

- 备份数据文件 ，需要停机
- 备份datadir目录下的所有文件

只拷贝这个目录可以吗？实际undo、redo、binlog可以配置不同的目录，可能不在datadir下

特殊情况：
    create table zz(a int) data directory = '/tmp/'
有这个情况怎么办呢？备份时候解析每个表的data directory？所以 不建议用这东西建表

**tips：**

redo、undo、binlog放hdd上，数据放ssd？没必要，现在ssd很便宜，顺序性也不差，三四块盘做个raid蛮好的，没必要分开
话说回来，冷备机会不多，要停机，不可接受

**温备(warm backup)**

- 针对myisam的备份(myisam不支持热备)，备份时候实例只读不可写
- 对应用影响很大
- 通常加一个读锁

## Ⅱ、MySQL热备工具

**ibbackup**

- 官方备份工具
- 收费
- 物理备份

**xtrabackup**

- 开源社区备份工具
- 开源免费，上面那东西的免费版本（老版本有问题，备份出来的数据可能有问题）
- 物理备份  

**mysqldump**

- 官方自带备份工具
 开源免费
- 逻辑备份(速度慢)

## Ⅲ、逻辑备份vs物理备份
|-|逻辑|物理|
|:-:|:-:|:-:|
|备份方式|备份数据库逻辑内容|备份数据库物理文件|
|优点|备份文件相对较小，只备份表中的数据与结构|恢复速度比较快(物理文件恢复基本已经完成恢复)|
|缺点|恢复速度较慢(需要重建索引，存储过程等)|备份文件相对较大(备份表空间，包含数据与索引，碎片)|
|对业务影响|缓冲池污染(把所有数据读一遍，读到bp中)，I/O负载加大|I/O负载加大|
|代表工具|mysqldump|ibbackup、xtrabackup|

**tips:**

一般从以下几个维度考虑备份方式
备份速度 、恢复速度 、备份大小 、对业务影响