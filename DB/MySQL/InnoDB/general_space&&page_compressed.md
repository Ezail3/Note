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
