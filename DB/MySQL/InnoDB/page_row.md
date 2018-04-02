---
# 页与记录相关内容
---

## Ⅰ、无主键的一个小测试
### 1.1 表上存在唯一键
```
(root@localhost) [test]> show create table test_key\G
*************************** 1. row ***************************
       Table: test_key
Create Table: CREATE TABLE `test_key` (
  `a` int(11) DEFAULT NULL,
  `b` int(11) NOT NULL,
  `c` int(11) NOT NULL,
  UNIQUE KEY `b` (`b`),
  UNIQUE KEY `c` (`c`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

(root@localhost) [test]> select *, _rowid from test_key;
+------+---+---+--------+
| a    | b | c | _rowid |
+------+---+---+--------+
|    1 | 2 | 3 |      2 |
|    4 | 5 | 6 |      5 |
|    7 | 8 | 9 |      8 |
+------+---+---+--------+
3 rows in set (0.00 sec)

可以看出，b列被作为了主键

(root@localhost) [test]> show create table test_key2\G
*************************** 1. row ***************************
       Table: test_key2
Create Table: CREATE TABLE `test_key2` (
  `a` varchar(4) DEFAULT NULL,
  `b` varchar(4) NOT NULL,
  `c` varchar(4) NOT NULL,
  UNIQUE KEY `b` (`b`),
  UNIQUE KEY `c` (`c`),
  UNIQUE KEY `a` (`a`)
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

(root@localhost) [test]> select *, _rowid from test_key2;
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
这里_rowid只有当key类型为id时才有效

换俩办法看即可
法1：
(root@localhost) [test]> select * from information_schema.columns where table_name='test_key2' and column_key='pri'\G
*************************** 1. row ***************************
           TABLE_CATALOG: def
            TABLE_SCHEMA: test
              TABLE_NAME: test_key2
             COLUMN_NAME: b
        ORDINAL_POSITION: 2
          COLUMN_DEFAULT: NULL
             IS_NULLABLE: NO
               DATA_TYPE: varchar
CHARACTER_MAXIMUM_LENGTH: 4
  CHARACTER_OCTET_LENGTH: 4
       NUMERIC_PRECISION: NULL
           NUMERIC_SCALE: NULL
      DATETIME_PRECISION: NULL
      CHARACTER_SET_NAME: latin1
          COLLATION_NAME: latin1_swedish_ci
             COLUMN_TYPE: varchar(4)
              COLUMN_KEY: PRI
                   EXTRA: 
              PRIVILEGES: select,insert,update,references
          COLUMN_COMMENT: 
   GENERATION_EXPRESSION: 
1 row in set (0.00 sec)

法2：
(root@localhost) [test]> desc test_key2;
+-------+------------+------+-----+---------+-------+
| Field | Type       | Null | Key | Default | Extra |
+-------+------------+------+-----+---------+-------+
| a     | varchar(4) | YES  | UNI | NULL    |       |
| b     | varchar(4) | NO   | PRI | NULL    |       |
| c     | varchar(4) | NO   | UNI | NULL    |       |
+-------+------------+------+-----+---------+-------+
3 rows in set (0.00 sec)
看到b列被作为主键
```

### 1.2 表上无唯一键
当表中未显式指定主键，且没有非空唯一键时，系统会自定义一个主键（6个字节，int型，全局，隐藏）
```
(root@localhost) [test]> show create table test_key3\G                                                    
*************************** 1. row ***************************
       Table: test_key3
Create Table: CREATE TABLE `test_key3` (
  `a` int(11) DEFAULT NULL,
  `b` int(11) DEFAULT NULL,
  `c` int(11) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=latin1
1 row in set (0.00 sec)

(root@localhost) [test]> select * from test_key3;
+------+------+------+
| a    | b    | c    |
+------+------+------+
|    1 |    2 |    3 |
|    4 |    5 |    6 |
|    7 |    8 |    9 |
+------+------+------+
3 rows in set (0.00 sec)

(root@localhost) [test]> select *, _rowid from test_key3;
ERROR 1054 (42S22): Unknown column '_rowid' in 'field list'
(root@localhost) [test]> select * from information_schema.columns where table_name='test_key3' and column_key='pri'\G
Empty set (0.00 sec)

(root@localhost) [test]> desc test_key3;                                                                  
+-------+---------+------+-----+---------+-------+
| Field | Type    | Null | Key | Default | Extra |
+-------+---------+------+-----+---------+-------+
| a     | int(11) | YES  |     | NULL    |       |
| b     | int(11) | YES  |     | NULL    |       |
| c     | int(11) | YES  |     | NULL    |       |
+-------+---------+------+-----+---------+-------+
3 rows in set (0.00 sec)
由上可见，这种情况下_rowid我们是看不了的，对我们透明
```

**tips：**
假设有两张表都使用了系统定义的主键，则系统定义的主键的id并不是表内单调递增的，而是全局递增

该系统的rowid是定义在ibdata1.ibd中的sys_rowid中，全局自增

6个字节表示的数据量为 2^48 ，通常意义上是够用的
