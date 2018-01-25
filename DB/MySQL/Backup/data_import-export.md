---
# 数据导入导出
---

## Ⅰ、传统姿势
这种方法需要提前设置一个参数

- secure_file_priv

|value|meaning|
|:-:|:-:|
|NULL|不允许导入导出|
|''|可以导入到任何地址|
|'/tmp'|导入到具体地址|

这个参数是只读参数，只能修改my.cnf后重启

做这种操作需要file权限

### 导出
```
select * into outfile 'xxx.data' from xxx    #sql语句随便写
fields terminated by 'string'                指定分隔符，默认tab
lines terminated by 'string'                 指定结束符，默认换行

测试一把
(root@localhost) [test]> select * from data_load;
+------+------+
| a    | b    |
+------+------+
|    1 |    2 |
|    2 |    3 |
|    3 |    4 |
+------+------+
3 rows in set (0.00 sec)

(root@localhost) [test]> select * into outfile '/tmp/data_load.data' from data_load;
Query OK, 3 rows affected (0.01 sec)

[root@VM_0_5_centos tmp]# cat data_load.data 
1	2
2	3
3	4
```

导出的数据和mysqldump不太一样，打开来看会发现里面是每个列的数据，tab来分割

### 导入
```
create table xxx like xxx;
数据文件里不是sql语句，用loaddata导入,不用额外解析insert，比较快
load data infile 'xxx' into table xxx
分隔符和结束符不一样的时候要调整

也测一把
(root@localhost) [test]> create table data_load2 like data_load;
Query OK, 0 rows affected (0.06 sec)

(root@localhost) [test]> load data infile '/tmp/data_load.data' into table data_load2;
Query OK, 3 rows affected (0.01 sec)
Records: 3  Deleted: 0  Skipped: 0  Warnings: 0

(root@localhost) [test]> select * from data_load2;
+------+------+
| a    | b    |
+------+------+
|    1 |    2 |
|    2 |    3 |
|    3 |    4 |
+------+------+
3 rows in set (0.00 sec)

玩个好玩的
(root@localhost) [test]> create table wanwan(a int, b int, c int);
Query OK, 0 rows affected (0.05 sec)

(root@localhost) [test]> load data infile '/tmp/data_load.data' into table wanwan (a,b) set c=a+b;
Query OK, 3 rows affected, 3 warnings (0.35 sec)
Records: 3  Deleted: 0  Skipped: 0  Warnings: 3

(root@localhost) [test]> select * from wanwan;
+------+------+------+
| a    | b    | c    |
+------+------+------+
|    1 |    2 |    3 |
|    2 |    3 |    5 |
|    3 |    4 |    7 |
+------+------+------+
3 rows in set (0.00 sec)

tips：ignore N lines 可指定不到如前N行
```

注意：

地理空间的数据类型，列比较特殊，geometry的列，导出的话是一个经度和纬度，导入的时候就要用set a = geometry(a,b);

### load data的缺点

- 针对文本文件，会有限制，比如一篇博客，有很多字符，逗号，tab，等等

其实可以搞成十六进制，再导入进去，但是比较烦

## Ⅱ、新式玩法
上面这种常用于异构数据的导入导出，如果一张表非常大，导起来可能会有点慢

myisam表flush一下就可以随意copy，innodb不行？ 那是innodb的信息由数据字典维护，保存在共享表空间中，没法把它sync到磁盘上

5.6版本开始支持独立表空间导入与导出（透明表空间传输），类似于xtrabackup备份，因为5.6有个新语法flush table ... for export将数据字典sync到disc

要求：两张表结构一样，这样表空间才能互相传输

操作步骤：
```
1、目标服务器：alter table t discard tablespace;                 删除表空间文件
2、源服务器：flush tables t for export;                          锁成只读
   show processlist;    
   waiting for table metadata lock                              加了元数据锁
3、把源实例的表空间拷贝一份到目标实例
4、源服务器：unlock tables;    释放锁
5、调整文件用户权限
6、目标服务器：alter table t import tablespace;                   导入

傻瓜式操作，简单快捷，不演示了，真的有点累啊！
```

不希望有warning的话，还有个cfg文件需要处理一下
import并不是秒级别的，和表空间大小有关，需要修改元数据，tablespace里面有space id和page no，两个ibd文件的space id是完全一样的，import的时候需要修改一下space id

**缺点：**

- 可能数据量比较大的话，和xtrabackup一样会造成一定的io飙升

**限制：**

- 两个实例都必须开启独立表空间，innodb_file_per_table
- 迁移的两个实例的innodb_page_size必须一致，并且mysql server版本建议一致

这个特性并没有得到广泛应用

## Ⅲ、好东西
5.7有个更好的东西，用的比较多(订单，快递保留三个月数据)

基于分区表透明表空间传输
```
alter table t1 discard p2,p3 tablespace;
alter table t1 import p2,p3 tablespace;
```
