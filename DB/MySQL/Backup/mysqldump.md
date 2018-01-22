---
# mysqldump详解
---

## Ⅰ、mysqldump的简单使用与注意点

### 重点
--single-transaction 必须加（一个事物中导出数据，确保产生一致性的备份数据）

my.cnf中配上下面配置
```
[mysqldump]
single-transcation
master-data=2
```

### 日常选项
只备份innodb，用不了几个参数，记住下面几个即可，其他的没什么卵用
-A 备份所有的database
-B 备份哪几个数据库
-R 备份存储过程(-- routines)
-E 备份定时任务(-- events)
-d 备份表结构
-w 备份过滤数据
--triggers 备份触发器
--master-data=2 在备份文件中以注释的形式记录备份开始时binlog的position，默认值是1，不注释

常用的几种：
mysqldump --single-transaction -B test a > backup.sql    备份test库和a库
mysqldump --single-transaction test a > backup.sql       备份test库下的a表
mysqldump --single-transaction test a -w "c=12"> backup.sql

