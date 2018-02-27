---
# InnoDB页压缩技术
---

## Ⅰ、上节回顾
### 创建表报错
```
(root@localhost) [(none)]> create tablespace ger_space add datafile 'ger_space.ibd' file_block_size=8192;
Query OK, 0 rows affected (0.02 sec)

(root@localhost) [(none)]> create database test;
Query OK, 1 row affected (0.01 sec)

(root@localhost) [(none)]> create table test_ger(a int) tablespace=ger_space;
(root@localhost) [(none)]> use test;
Database changed
(root@localhost) [test]> create table test_ger(a int) tablespace=ger_space;
ERROR 1478 (HY000): InnoDB: Tablespace `ger_space` uses block size 8192 and cannot contain a table with physical page size 16384
```
### 解决
```
(root@localhost) [test]> create table test_ger(a int) tablespace=ger_space row_format=compressed, key_block_size=8;
Query OK, 0 rows affected (0.14 sec)

amazing!!!(～﹃～)~zZ
```
这里用到一个压缩page的技术，本章我们讨论一下InnoDB的页压缩

## Ⅱ、传统压缩方式(from 5.5)
### 原理
- 基于页的压缩，每个表的页大小可以不同，压缩算法是L777
- 当用户获取数据时，如果压缩的页没有在Innodb_Buffer_Pool缓冲池里，那么会从磁盘加载进去，并且会在Innodb_Buffer_Pool缓冲池里开辟一个新的未压缩的16KB的数据页来解压缩，为了减少磁盘I/O以及对页的解压操作，在缓冲池里同时存在着被压缩的和未压缩的页
- 为了给其他需要的数据页腾出空间，缓冲池里会把未压缩的数据页踢出去，而保留压缩的页在内存中，如果未压缩的页在一段时间内没有被访问，那么会直接刷入磁盘中，因此缓冲池中可能有压缩和未压缩的页，也可能只有压缩页

### 玩两手
```
直接创建
(root@localhost) [test]> create table comps_test(a int) row_format=compressed, key_block_size=4; 
Query OK, 0 rows affected (0.04 sec)

对已存在的表启用压缩，并且页大小为4k，
alter table xxxxx
engine=innodb
row_format=compressed,key_block_size=4

可以设置为1 2 4 8 16
操作须知：
指定row_format=compressed，则可忽略key_block_size的可选项是1k的值，这时使用默认innodb页的一半，即8kb
key_block_size的可选项是1k，则可忽略row_format=compressed，会自动启用压缩
0代表默认压缩页的值，Innodb页的一半
key_block_size的值只能小于等于innodb page size，若指定了一个大于innodb page size的值，mysql会忽略这个值然后产生一个警告，这时key_block_size的值是Innodb页的一半
若设置了innodb_strict_mode=ON，那么指定一个不合法的key_block_size的值是返回报错
```

**tips：**

虽然SQL语法中写的是row_format=compressed，但是压缩是针对页的，而不是记录，即读页的时候解压，写页的时候压缩，并不会在读取或写入单个记录（row）时就进行解压或压缩操作

### 细说key_block_size
- key_block_size的可选项是1k，2k，4k，8k，16k（是页大小，不是比例）
- 不是将原来innodb_page_size页大小的数据压缩成key_block_size的页大小，因为有些数据可能不能压缩，或者压缩不到那么小
- 压缩是将原来的页的数据通过压缩算法压缩到一定的大小，然后用key_block_size大小的页去存放
- 比如原来的innodb_page_size大小是16K，现在的key_block_size设置为8K,某表的数据大小是24k,原先使用2个16k的页存放,压缩后数据从24k变为18k，由于现在的key_block_size=8k，所以需要3个8K的页存放压缩后的18k数据，多余的空间可以留给下次插入或者更新
- 压缩比和设置的key_block_size没有关系，压缩比看数据本身和算法，key_block_size仅仅是设置存放压缩数据的页大小

**tips：**

不解压也能插入数据，通过在剩余空间直接存放redo log，然后页空间存放满后，再解压，利用redo log更新完成后，最后再压缩存放（此时就没有redo log 了）以此来减少解压和压缩的次数

### 重点须知
并不是key_block_size越小，压缩比越高，只是页的大小发生了修改

压缩过程，16k的页压8k，先判断能不能压，可以就存为8k，压完大于8k，比如12k，这时候就会存为两个8k

16k压到8k成功率在80%~90%，但是再压就不能保证了

### 查看压缩比

## Ⅲ、透明页压缩
