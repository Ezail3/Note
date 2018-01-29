---
# 常见函数解析
---

## Ⅰ、ifnull和isnull判断空
```
(root@localhost) [(none)]> select ifnull(null,1), ifnull(0,1), ifnull('',1), hex('');
+----------------+-------------+--------------+---------+
| ifnull(null,1) | ifnull(0,1) | ifnull('',1) | hex('') |
+----------------+-------------+--------------+---------+
|              1 |           0 |              |         |
+----------------+-------------+--------------+---------+
1 row in set (0.00 sec)

(root@localhost) [(none)]> select hex(null),hex(''),hex(' ');
+-----------+---------+----------+
| hex(null) | hex('') | hex(' ') |
+-----------+---------+----------+
| NULL      |         | 20       |
+-----------+---------+----------+
1 row in set (0.00 sec)
这里和oracle不一样的是，MySQL中''和null不是一回事

(root@localhost) [(none)]> select isnull(null),isnull(0),isnull(1/0);
+--------------+-----------+-------------+
| isnull(null) | isnull(0) | isnull(1/0) |
+--------------+-----------+-------------+
|            1 |         0 |           1 |
+--------------+-----------+-------------+
1 row in set (0.00 sec)
这里注意:1/0在MySQL中不会报错
```

## Ⅱ、时间函数
```
(root@localhost) [(none)]> select now(),sysdate(),sleep(2),now(),sysdate()\G
*************************** 1. row ***************************
    now(): 2018-01-29 11:06:12
sysdate(): 2018-01-29 11:06:12
 sleep(2): 0
    now(): 2018-01-29 11:06:12
sysdate(): 2018-01-29 11:06:14
1 row in set (2.04 sec)
```
now是sql开始执行的时间,sysdate是函数本身开始执行的时间

案例：
```
(root@172.16.0.10) [employees]> desc salaries;
+-----------+---------+------+-----+---------+-------+
| Field     | Type    | Null | Key | Default | Extra |
+-----------+---------+------+-----+---------+-------+
| emp_no    | int(11) | NO   | PRI | NULL    |       |
| salary    | int(11) | NO   |     | NULL    |       |
| from_date | date    | NO   | PRI | NULL    |       |
| to_date   | date    | NO   |     | NULL    |       |
+-----------+---------+------+-----+---------+-------+
4 rows in set (0.00 sec)

(root@172.16.0.10) [employees]> desc select * from salaries where emp_no=10001 and from_date>now();
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
| id | select_type | table    | partitions | type  | possible_keys  | key     | key_len | ref  | rows | filtered | Extra       |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | range | PRIMARY,emp_no | PRIMARY | 7       | NULL |    1 |   100.00 | Using where |
+----+-------------+----------+------------+-------+----------------+---------+---------+------+------+----------+-------------+
1 row in set, 1 warning (0.00 sec)

(root@172.16.0.10) [employees]> desc select * from salaries where emp_no=10001 and from_date>sysdate();
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
| id | select_type | table    | partitions | type | possible_keys  | key     | key_len | ref   | rows | filtered | Extra       |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
|  1 | SIMPLE      | salaries | NULL       | ref  | PRIMARY,emp_no | PRIMARY | 4       | const |   17 |    33.33 | Using where |
+----+-------------+----------+------------+------+----------------+---------+---------+-------+------+----------+-------------+
1 row in set, 1 warning (0.01 sec)
```

综上：建议统一规范用now，可以当作常数用，而sysdate是一直变的，会导致索引用不上

**tips:**

- 5.7的desc很好用，可以看到filtered
- 5.6想要filtered得用explain extented

--sysdate-is-now 可以强制把sysdate改为now

[root@VM_0_5_centos ~]# mysqld --verbose --help |grep sysdate

--sysdate-is-now    Non-default option to alias SYSDATE() to NOW() to make it

sysdate-is-now 

## Ⅲ、补位与空格
```
左右补位：
(root@localhost) [(none)]> select rpad('a',5,'0'), lpad('a',5,'0');
+-----------------+-----------------+
| rpad('a',5,'0') | lpad('a',5,'0') |
+-----------------+-----------------+
| a0000           | 0000a           |
+-----------------+-----------------+
1 row in set (0.00 sec)

去除空格
(root@localhost) [(none)]> select ltrim(' A'),length(ltrim(' A')),rtrim('A '),length(rtrim('A ')),trim(' A '),length(trim(' A '));
+-------------+---------------------+-------------+---------------------+-------------+---------------------+
| ltrim(' A') | length(ltrim(' A')) | rtrim('A ') | length(rtrim('A ')) | trim(' A ') | length(trim(' A ')) |
+-------------+---------------------+-------------+---------------------+-------------+---------------------+
| A           |                   1 | A           |                   1 | A           |                   1 |
+-------------+---------------------+-------------+---------------------+-------------+---------------------+
1 row in set (0.04 sec)
```

## Ⅳ、字符拼接
```
(root@localhost) [(none)]> select concat('a','bc');
+------------------+
| concat('a','bc') |
+------------------+
| abc              |
+------------------+
1 row in set (0.00 sec)

(root@localhost) [(none)]> set sql_mode='pipes_as_concat';
Query OK, 0 rows affected (0.00 sec)

(root@localhost) [(none)]> select 'a'||'bc';
+-----------+
| 'a'||'bc' |
+-----------+
| abc       |
+-----------+
1 row in set (0.00 sec)
```

**tips:**

sql_mode一定要设置一把，不然拼起来会出错，是数字，不信试试
