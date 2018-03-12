---
# 初识后台线程
---

## Ⅰ、看下所有后台线程
```
(root@localhost) [(none)]> select name from performance_schema.threads where name like 'thread/innodb%' order by name;
+----------------------------------------+
| name                                   |
+----------------------------------------+
| thread/innodb/buf_dump_thread          |    # dump bp
| thread/innodb/dict_stats_thread        |    # 元数据
| thread/innodb/io_ibuf_thread           |    # change buffer
| thread/innodb/io_log_thread            |    # 写重做日志
| thread/innodb/io_read_thread           |    # 异步读
| thread/innodb/io_read_thread           |
| thread/innodb/io_read_thread           |
| thread/innodb/io_read_thread           |
| thread/innodb/io_write_thread          |    # 异步写
| thread/innodb/io_write_thread          |
| thread/innodb/io_write_thread          |
| thread/innodb/io_write_thread          |
| thread/innodb/page_cleaner_thread      |    # 刷新脏页，innodb_page_cleaners，建议和io_write_thread设置一样
| thread/innodb/srv_error_monitor_thread |    # 监控
| thread/innodb/srv_lock_timeout_thread  |    # 监控
| thread/innodb/srv_master_thread        |    # 之前很关键，脏页刷新之前在这个线程里，5.6开始移出去了，基本上现在就是刷新重做日志的
| thread/innodb/srv_monitor_thread       |    # 监控
| thread/innodb/srv_purge_thread         |    # 回收undo
| thread/innodb/srv_worker_thread        |    # page_cleaner的子线程
| thread/innodb/srv_worker_thread        |
| thread/innodb/srv_worker_thread        |
+----------------------------------------+
21 rows in set (0.00 sec)
```

异步读写由下面两个参数控制

innodb_read_io_threads    不建议调整，读取基本上很少是异步，调大没太大意义

innodb_write_io_threads   可以调大一点，因为数据库写入都是异步的，默认是4，调成8或者16，特别是ssd的情况下
