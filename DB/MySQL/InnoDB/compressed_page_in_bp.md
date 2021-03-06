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
```
(root@localhost) [(none)]> show engine innodb status\G
...
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 127372
Buffer pool size   8191
Free buffers       7759
Database pages     632
Old database pages 253
Modified db pages  0
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s
Pages read 597, created 35, written 42
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 632, unzip_LRU len: 4          压缩页在bp中的长度是4
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
...
```

## Ⅱ、伙伴算法
- 磁盘中存放压缩页（row_format=compressed），假设key_block_size=8K，Buffer Pool的页大小是16K
- 向Free List中申请空闲的页，如果没有空闲页，则向LRU List申请页，如果LRU满了，则找LRU中最后的一个页，如果最后的页是脏页，则做flush操作，最后得到一个空白的页（16K）
- 该16k的空白页，就给8K的压缩页使用，这样就多出一个8K的空间 ，该空间会移到8K的Free List中去
- 如果有一个4K的压缩页，就把8K的Free list中的空白页给他用，然后多余的4K的空间移到4K的Free List中去

通过上述方式，不同大小的页可以在同一个Buffer Pool中使用（可以简单的认为Free List是按照页大小来进行划分的）
不能根据页大小来划分缓冲池，缓冲池中页的大小就是固定的大小（等于innodb_page_size ）
LRU List和Flush List不需要按照页大小划分，都是统一的innodb_page_size大小




```
                                         ^  + 
                                         |  |
                                     read|  |Update 
                                         |  | 
            +--------+                +--+--v---+
            |        | decompressed   |     16K |   
            |   4K   +---------------->         | 
Buffer Pool |        |                |         |        
            |   redo <---+update      |         | 
            |        | no need        |         | 
	    +--+--^--+ decompressed   +---------+ 
	       |  |
+-------------------------------------------------------------+ 
               |  |
  flush to disk|  |read from disk
               |  | 
	     +-v--+-+ Disk 
	     |      | 
	     |  4K  | 
	     |      | 
	     +------+
```

- 压缩页在内存中保留
- 被压缩的页需要在Buffer Pool中解压
- 原来的压缩页保留在Buffer Pool中
- 缺点是压缩页占用了Buffer Pool的空间，对于热点数据来说，相当于内存小了，可能造成性能下降（热点空间变小）
- 所以在开启了压缩后，Buffer Pool的空间要相应增大
- 如果启用压缩后节省的磁盘IO能够 抵消 掉Buffer； Pool“空间变小”所带来的性能下降，那整体性能还是会上涨；
- 启用压缩的前提是，内存尽可能的大
- 压缩页保留的原因是为了在更新数据的时候，将redo添加到压缩页的空闲部分，如果要刷回磁盘，可以直接将该压缩页刷回去，如果该页被写满，则做一次 reorganize操作（在此之前也要做解压），真的写满了才做分裂

1. 保留压缩页是为了更快的刷回磁盘
2. 解压的页是为了更快的查询
透明压缩则没有上述压缩页的问题，因为压缩是文件系统层的，对MySQL是透明的
