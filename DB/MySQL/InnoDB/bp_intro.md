---
# 缓冲池工作原理浅析
---

## Ⅰ、缓冲池介绍
innodb存储引擎缓冲池(buffer pool) ，类似于oracle的sga，里面放着数据页 、索引页 、change buffer 、自适应哈希 、锁(5.5之前)等内容

```
                                  Clients
                                     ^  +
                           read data |  |
              +----------------------+  |
              |                         |
	      |          +--------------+
              |          | write data
              |          |
              |          |
          +--+--+-----+--v--+---------------+ +--------+ +--------+     +--------+
          |  |  |     | Buffer              | | Buffer | | Buffer |     | Buffer |
          | 16K | 16K | 16K |...... Pool 1  | | Pool 2 | | Pool 3 | ... | Pool N | On Memory
          | p:5 |     | p:8 |               | |        | |        |     |        |
          +--^--+-----+--+--+---------------+ +--------+ +--------+     +--------+
             |           |
    read page| from disk |
             |           |
+----------------------------------------------------------------------------------------------------+
             |           |
             |      flush|to disk
             +-----+     |
                   |     |
                +--+--+--v--+-----+------+-----+
                |     |     |     |      |     |
 space id = 196 | 16K | 16K | 16K |......| 16K | a.ibd       +-------+
                | p:5 | p:8 |     |      |     |                     |
                +-----+-----+-----+------+-----+                     |
                                                                     |
                +-----+-----+-----+------+-----+                     |
                |     |     |     |      |     |                     |
 space id = 169 | 16K | 16K | 16K |......| 16K | xx.ibd    +---------+ On Disk
                |     |     |     |      |     |                     |
                +-----+-----+-----+------+-----+                     |
                                                                     |
                +-----+-----+-----+------+-----+                     |
                |     |     |     |      |     |                     |
 space id = 0   | 16K | 16K | 16K |......| 16K | ibdata1.ibd +-------+
                |     |     |     |      |     |
                +-----+-----+-----+------+-----+
```
综上所示：
- 每次读写数据都是通过Buffer Pool
- 当Buffer Pool中没有用户所需要的数据时才去硬盘中获取
- 通过innodb_buffer_pool_size进行设置总容量，该值设置的越大越好

## Ⅱ、缓冲池性能问题

### 性能线性扩展

假设服务器72核，ht超线程后，144个逻辑核，跑测试按道理144个核应该跑满，如果跑不满，就说明并发有瓶颈，我加核了，却用不上，性能上不去

5.1之前这个问题经常被吐槽，现在不存在这个问题了

1G空间下面如果有65536个page，对这些page进行管理，每次都要对bp加锁(latch),如果bp大了，就有瓶颈，这里说的锁是bp的latch，和数据库的lock不是一回事

qps达到1w，每秒钟要获得至少1w次latch(就看bp的latch，不谈释放和唤醒latch)，开销比较大

核比较多，latch或者并发设计的不好，性能则不能线性扩展 ，而这个bp对于扩展性非常重要，所有的热点的page都在里面，每次访问这些page都要获得bp的latch

### 如何提升上述缓冲池性能问题

调整innodb_buffer_pool_instances参数，设置为cpu的数量

默认5.5为1，5.6和5.7是8

假设开始这个值是1，现调整为4，原来1个bp管理65536个页，现在4个bp，每个bp管理16384个页，拆成4个分片，将热点打散，latch变少了，并发性能提升了，这是非常常见的内核层对并发调优的手段，经测试，不调整与调整后性能相差30%

**tips：**

设置多个缓冲池的时候，必须满足每个池子大于1G才生效，否则，即使my.cnf中设置了innodb_buffer_pool_instances，重启看看是没用的

## Ⅲ、buffer pool中热点数据的管理
### buffer pool的组成
```
+----+      +--------------+
|    |------>     LRU      |
|free|      |              |
|    <------|include(flush)|
+----+      +--------------+

Free List  干净的page
LRU List   包括LRU和unzip_LRU
Flush List 根据oldest_lsn进行排序
```

缓冲池中的热点是以page为单位来管理，并不是三种List加起来等于总的bp大小，而是Free List + LRU List(Flush List是包含在LRU list里面的)
- Free List 放空白的page

buffer Pool刚启动时，有一个个16K的空白的页，这些页就存放（链表串联）在Free List中

- LRU List   包括LRU和unzip_LRU

当读取一个数据页的时候，就从Free List中取出一个页，存入数据，并将该页读到LRU List中

当Free List给一个页给LRU List时，这个过程中需要一个并发控制，也就是之前说的latch，假设现在有两个线程都读到磁盘上这个页，则都需要问Free List来申请空闲页，谁先来先给谁，latch就是对这三个List进行并发控制访问的

- Flush List 包含脏页（数据经过修改，但是未刷入磁盘的页），根据oldest_lsn进行排序

假设被读到的页，马上被更新，这个页就叫脏页，会被放入到Flush List列表中，但只是放了一个指针，而不是实际的页（只要修改过，就放入，不管修改几次）

**tips：**

Flush list 中存放的不是一个页，而是页的指针（page number）

**小结：**

LRU List存放的是所有已经使用的页，里面既有干净页也有脏页，Flush List中只有指向脏页的指针

### 查看buffer pool的状态

**方法1：show engine innodb status\G**
```
...
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 137428992
Dictionary memory allocated 303387
Buffer pool size   8192		#缓冲池中共8192个page
Free buffers       7772	        #空白页(Free List),线上很可能是0	
Database pages     420		#在使用的页(LRU List)
Old database pages 0		#LRU中教冷的page		
Modified db pages  0		#脏页
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 0, not young 0
0.00 youngs/s, 0.00 non-youngs/s	#yongs表示old变为new
Pages read 368, created 52, written 322
0.00 reads/s, 0.00 creates/s, 0.00 writes/s
No buffer pool page gets since the last printout
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 420, unzip_LRU len: 0
I/O sum[0]:cur[0], unzip sum[0]:cur[0]
...
```


**方法2：看两张元数据表**
