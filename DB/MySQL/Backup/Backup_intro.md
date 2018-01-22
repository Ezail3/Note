---
# 备份基础介绍
---

## Ⅰ、备份类型

这里介绍三种全量备份

**热备(hot backup)**

- 在线备份
- 对应用基本无影响(应用程序读写不会阻塞，但是性能还是会又下降，所以尽量不要在主上做备份，在从库上做)

**冷备(cold backup)**

- 备份数据文件 ，需要停机
- 备份datadir目录下的所有文件(只拷贝这个目录可以吗？实际undo、redo、binlog可以配置不同的目录，可能不在datadir下)

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

