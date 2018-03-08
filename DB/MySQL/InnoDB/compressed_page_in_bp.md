---
# 缓冲池中的压缩页相关分析

## Ⅰ、看看buffer pool中的压缩页嘛
```
(root@localhost) [(none)]> SELECT                                                                                                                                                            
    ->     table_name,
    ->     space,
    ->     page_number,
    ->     index_name,
    ->     compressed,
    ->     compressed_size
    -> FROM
    ->     information_schema.innodb_buffer_page_lru
    -> WHERE
    ->     compressed = 'yes'
    -> LIMIT 5;
+------------+-------+-------------+------------+------------+-----------------+
| table_name | space | page_number | index_name | compressed | compressed_size |
+------------+-------+-------------+------------+------------+-----------------+
| NULL       |   168 |         502 | NULL       | YES        |            4096 |
| NULL       |   168 |         505 | NULL       | YES        |            4096 |
| NULL       |   168 |         508 | NULL       | YES        |            4096 |
| NULL       |   168 |         510 | NULL       | YES        |            4096 |
| NULL       |   168 |         513 | NULL       | YES        |            4096 |
+------------+-------+-------------+------------+------------+-----------------+
5 rows in set (0.00 sec)

(root@localhost) [(none)]> SELECT 
    ->     table_id, name, space, row_format, zip_page_size
    -> FROM
    ->     information_schema.INNODB_SYS_TABLES
    -> WHERE
    ->     space = 168;
+----------+----------------+-------+------------+---------------+
| table_id | name           | space | row_format | zip_page_size |
+----------+----------------+-------+------------+---------------+
|      173 | sbtest/sbtest1 |   168 | Compressed |          4096 |
+----------+----------------+-------+------------+---------------+
1 row in set (0.00 sec)

(root@localhost) [(none)]> show create table sbtest.sbtest1\G
*************************** 1. row ***************************
       Table: sbtest1
Create Table: CREATE TABLE `sbtest1` (
  `id` int(10) unsigned NOT NULL AUTO_INCREMENT,
  `k` int(10) unsigned NOT NULL DEFAULT '0',
  `c` char(120) NOT NULL DEFAULT '',
  `pad` char(60) NOT NULL DEFAULT '',
  PRIMARY KEY (`id`),
  KEY `k_1` (`k`)
) ENGINE=InnoDB AUTO_INCREMENT=10000001 DEFAULT CHARSET=latin1 MAX_ROWS=1000000 ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=4
1 row in set (0.00 sec)
```
一路摸下来，果然是个压缩表

## Ⅱ、压缩页在bp中的存储
这块其实之前章节中 已简单提过，这里详细分析一下
