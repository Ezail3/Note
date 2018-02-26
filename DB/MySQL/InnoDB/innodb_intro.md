---
# InnoDB存储引擎结构介绍
---

## Ⅰ、InnoDB发展史
|时间|事件|备注|
|:-:|:-:|:-:|
|1995|由Heikki Tuuri创建Innobase Oy公司，开发InnoDB存储引擎|Innobase开始做的是数据库，希望卖掉该公司|
|1996|MySQL 1.0 发布||
|2000|MySQL3.23版本发布||
|2001|InnoDB存储引擎集成到MySQL数据库|以插件方式集成|
|2006|Innobase被Oracle公司收购（InnoDB作为开源产品，性能和功能很强大）|InnoDB在被收购后的，MySQL中的InnoDB版本没有改变|
|2010|MySQL5.5版本InnoDB存储引擎称为默认存储引擎|MySQL被Sun收购，Sun被Oracle收购，使得MySQL和InnoDB重新在一起配合开发|
|至今|其他存储引擎已经不再得到Oracle官方的后续开发||

## Ⅱ、InnoDB重要特性一览
- Fully ACID（InnoDB默认的Repeat Read隔离级别就支持）
- Row-level Locking（支持行锁）
- Multi-version concurrency control（MVCC）（支持多版本并发控制）
- Foreign key support（支持外键）
- Automatic deadlock detection（死锁自动检测）
- High performance、High scalability、High availability（高性能，高扩展，高可用）

## Ⅲ、物理存储结构
### 主要组成部分

**表空间文件：**

- 独立表空间
- 共享表空间
- undo表空间(MySQL5.6开始)

**重做日志文件：**

- 物理逻辑日志
- 没有Oracle的归档重做日志

### 细说表空间文件

**表空间的概念：**

- 表空间是一个逻辑存储的概念
- 表空间可以由多个文件组成
- 支持裸设备（可以直接使用O_DIRECT 方式绕过缓存，直接写入磁盘）

**表空间的分类：**

①共享表空间(最早只有这个)
- 存储元数据信息
- 存储Change Buffer信息
- 存储Undo信息
- 一开始所有的表和索引的信息都是存储在共享表空间中,随后InnoDB对其做了改进，可以使用独立的表空间

②独立表空间
- innodb-file-per-table=1 （开启支持每个表一个独立的表空间）
- 每张用户表对应一个独立的 ibd文件
- 分区表可以对应多个ibd文件

③undo表空间
- MySQL5.6版本支持独立的Undo表空间
- innodb_undo_tablespaces(该值8.0开始将会被剔除，不可修改，默认写死，为2)

④临时表空间
- MySQL5.7增加了临时表空间（ibtmp1）
- innodb_temp_data_file_path

### 看下数据目录
```
[root@VM_0_5_centos data3306]# ll ib*
-rw-r----- 1 mysql mysql    16285 Feb  4 18:15 ib_buffer_pool
-rw-r----- 1 mysql mysql 79691776 Feb 24 10:53 ibdata1          #共享表空间
-rw-r----- 1 mysql mysql 50331648 Feb 24 10:53 ib_logfile0      #重做日志
-rw-r----- 1 mysql mysql 50331648 Feb  4 15:06 ib_logfile1    
-rw-r----- 1 mysql mysql 12582912 Feb 24 10:14 ibtmp1           #临时表空间

[root@VM_0_5_centos data3306]# cd test
[root@VM_0_5_centos test]# ll
total 228
-rw-r----- 1 mysql mysql       65 Feb  4 13:21 db.opt           #记录默认字符集和字符集排序规则
-rw-r----- 1 mysql mysql  8554 Feb  1 15:45 abc.frm             #表结构文件
-rw-r----- 1 mysql mysql 98304 Feb 24 10:53 abc.ibd             #独立表空间
```

## Ⅳ、逻辑存储结构
