---
# 通用表空间与页压缩
---

## 通用表空间的使用

**需求：**

指定存储路径创建一张表

### 方案一：指定data directory
```
[root@VM_0_5_centos mdata]# mkdir /mdata/general
[root@VM_0_5_centos mdata]# chown mysql.mysql /mdata/general/

(root@localhost) [(none)]> create table test.test_ger (a int) data directory='/mdata/general';
Query OK, 0 rows affected (0.06 sec)

[root@VM_0_5_centos general]# tree
.
`-- test                     #库名
    `-- test_ger.ibd         #表空间文件

1 directory, 1 file

进入datadir下再看
[root@VM_0_5_centos test]# cd /mdata/data3306/test
[root@VM_0_5_centos test]# ll t*
-rw-r----- 1 mysql mysql 8554 Feb 27 15:00 test_ger.frm
-rw-r----- 1 mysql mysql   32 Feb 27 15:00 test_ger.isl
[root@VM_0_5_centos test]# cat test_ger.isl       一个文本文件，内容就是idb文件的路径，做了一个链接
/mdata/general/test/test_ger.ibd
```

### 方案二：通用表空间
```
第一步：创建通用表空间
(root@localhost) [(none)]> create tablespace ger_space add datafile '/mdata/general/ger_space.ibd' file_block_size=16384;
Query OK, 0 rows affected (0.03 sec)

(root@localhost) [(none)]> create tablespace ger2_space add datafile 'ger2_space.ibd' file_block_size=16384;
Query OK, 0 rows affected (0.02 sec)

[root@VM_0_5_centos data3306]# cd /mdata/data3306
[root@VM_0_5_centos data3306]# ll ger*
-rw-r----- 1 mysql mysql 65536 Feb 27 16:17 ger2_space.ibd
-rw-r----- 1 mysql mysql    28 Feb 27 16:16 ger_space.isl
[root@VM_0_5_centos data3306]# cat ger_space.isl 
/mdata/general/ger_space.ibd       # ibd真实路径

综上：
datafile指定存储路径后，在datadir下会产生一个isl文件，该文件的内容为General space的ibd文件的路径

如果datafile不指定路径，则ibd文件默认存储在datadir目录下，且不需要isl文件了

(root@localhost) [(none)]> select * from information_schema.innodb_sys_tablespaces where name='ger_space'\G
*************************** 1. row ***************************
         SPACE: 114
          NAME: ger_space
          FLAG: 2089
   FILE_FORMAT: Barracuda
    ROW_FORMAT: Compressed
     PAGE_SIZE: 16384       # 页大小为16k
 ZIP_PAGE_SIZE: 0                       
    SPACE_TYPE: General
 FS_BLOCK_SIZE: 4096
     FILE_SIZE: 65536
ALLOCATED_SIZE: 69632
1 row in set (0.00 sec)

第二步：创建表
(root@localhost) [test]> create table test_ger(a int) tablespace=ger_space;
Query OK, 0 rows affected (0.03 sec)

(root@localhost) [test]> create table test2_ger(a int) tablespace=ger_space;
Query OK, 0 rows affected (0.04 sec)

[root@VM_0_5_centos test]# cd /mdata/data3306/test
[root@VM_0_5_centos test]# ll
total 28
-rw-r----- 1 mysql mysql   65 Feb 27 15:00 db.opt
-rw-r----- 1 mysql mysql 8554 Feb 27 17:02 test2_ger.frm
-rw-r----- 1 mysql mysql 8554 Feb 27 16:37 test_ger.frm

[root@VM_0_5_centos data3306]# cd ..
[root@VM_0_5_centos data3306]# ll ger*
-rw-r----- 1 mysql mysql 28 Feb 27 16:37 ger_space.isl
[root@VM_0_5_centos data3306]# cat ger_space.isl 
/mdata/general/ger_space.ibd
```

**小结：**

- 通过使用General Space，一个表空间可以对应多张表
- 当对表进行alter等操作时，还是和原来一样，无需额外语法指定表空间位置
- 可以简单的理解为把多个表的ibd文件合并在一起了

**tips：**
①删除通用表空间：
```
drop tablespace xxx;
若表空间中有数据，会报错，删除失败
ERROR 1529 (HY000): Failed to drop TABLESPACE xxx
```

②file_block_size可指定表空间块大小，其实就是page_size
但是注意，如果设置了innodb_page_size,且大小不是file_block_size，那么在创建表的时候会报错,如下：
```
(root@localhost) [test]> create table test_ger(a int) tablespace=ger_space;
ERROR 1478 (HY000): InnoDB: Tablespace `ger_space` uses block size 8192 and cannot contain a table with physical page size 16384

这块不太理解，既然无法创建表，那应该在创建general space时就应该报错啊，后续涉及压缩表时可以使用，想不通！
```

③使用通用表空间须注意备份工具对这块是否支持

















