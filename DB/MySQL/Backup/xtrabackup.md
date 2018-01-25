---
# MySQL物理备份基本操作
---

## xtrabackup介绍

- xtrabackup只能备份innodb引擎的数据，不能备份表结构，percona开源的，强烈推荐最新版本(旧版本bug多)
- innobackupex可以备份myisam和innodb两种引擎的数据和表结构，一般用这个
- 备份时，默认读取MySQL配置文件(datadir)

## xtrabackup安装使用

### 安装
```
[root@VM_0_5_centos src]# yum install perl-DBD-MySQL
不安装这个备份会报错：Failed to connect to MySQL server: DBI connect

[root@VM_0_5_centos src]# cd /usr/local/src
[root@VM_0_5_centos src]# wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.7/binary/tarball/percona-xtrabackup-2.4.7-Linux-x86_64.tar.gz
[root@VM_0_5_centos src]# tar zxvf percona-xtrabackup-2.4.7-Linux-x86_64.tar.gz -C ..
添加环境变量
[root@VM_0_5_centos src]# cd ..
[root@VM_0_5_centos src]# ln -s percona-xtrabackup-2.4.7-Linux-x86_64/ xtrabackup
[root@VM_0_5_centos src]# echo "PATH=/usr/local/xtrabackup/bin:$PATH" >> /etc/profile
[root@VM_0_5_centos src]# source /etc/profile
```

### 玩一手
```
[root@VM_0_5_centos src]# innobackupex --compress --compress-threads=8 --stream=xbstream -S /tmp/mysql.sock --parallel=4 /data/backup > backup.xbstream
建议用-S连接，默认走socket，不用-S可能报连不上

常用参数：throttle
指定备份时用到的iops是多少，限制速度
```

8个压缩线程，4个备份线程

输出内容（简化）
```
180122 19:47:53 innobackupex: Starting the backup operation

IMPORTANT: Please check that the backup run completes successfully.
           At the end of a successful backup run innobackupex
           prints "completed OK!".

180122 19:47:53  version_check Connecting to MySQL server with DSN 'dbi:mysql:;mysql_read_default_group=xtrabackup;mysql_socket=/tmp/mysql.sock' as 'root'  (using password: YES).
180122 19:47:53  version_check Connected to MySQL server
180122 19:47:53  version_check Executing a version check against the server...
180122 19:47:53  version_check Done.
180122 19:47:53 Connecting to MySQL server host: localhost, user: root, password: set, port: not set, socket: /tmp/mysql.sock
Using server version 5.7.20-log
innobackupex version 2.4.7 based on MySQL server 5.7.13 Linux (x86_64) (revision id: 05f1fcf)
xtrabackup: uses posix_fadvise().

# 连接数据库并做两次版本检查

xtrabackup: cd to /mdata/mysql_test_data
xtrabackup: open files limit requested 0, set to 100001
xtrabackup: using the following InnoDB configuration:
xtrabackup:   innodb_data_home_dir = .
xtrabackup:   innodb_data_file_path = ibdata1:12M:autoextend
xtrabackup:   innodb_log_group_home_dir = ./
xtrabackup:   innodb_log_files_in_group = 2
xtrabackup:   innodb_log_file_size = 50331648
InnoDB: Number of pools: 1
180122 19:47:53 >> log scanned up to (10304795)

# 读取配置文件，寻找对应的文件及日志位置

xtrabackup: Generating a list of tablespaces
InnoDB: Allocated tablespace ID 51 for dump_test/dump_inno, old maximum was 0
xtrabackup: Starting 4 threads for parallel data files transfer
180122 19:47:53 [04] Compressing and streaming ./ibdata1
180122 19:47:53 [03] Compressing and streaming ./dump_test/dump_inno.ibd
180122 19:47:53 [03]        ...done
180122 19:47:53 [03] Compressing and streaming ./test/test.ibd
180122 19:47:53 [02] Compressing and streaming ./test/sbtest1.ibd
180122 19:47:53 [03]        ...done

...

180122 19:47:54 >> log scanned up to (10304795)
180122 19:47:54 Executing FLUSH NO_WRITE_TO_BINLOG TABLES...
180122 19:47:54 Executing FLUSH TABLES WITH READ LOCK...
180122 19:47:54 Starting to backup non-InnoDB tables and files
180122 19:47:54 [01] Compressing and streaming ./dump_test/dump_inno.frm to <STDOUT>
180122 19:47:54 [01]        ...done
180122 19:47:54 [01] Compressing and streaming ./dump_test/db.opt to <STDOUT>
180122 19:47:54 [01]        ...done

...

180122 19:47:55 Finished backing up non-InnoDB tables and files

# 拷贝数据

180122 19:47:55 [00] Compressing and streaming xtrabackup_binlog_info
180122 19:47:55 [00]        ...done

# 获取二进制文件日志点

180122 19:47:55 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '10304786'
180122 19:47:55 >> log scanned up to (10304795)
xtrabackup: Stopping log copying thread.
180122 19:47:55 Executing UNLOCK TABLES
180122 19:47:55 All tables unlocked

# 停止拷贝，释放锁

180122 19:47:55 [00] Compressing and streaming ib_buffer_pool to <STDOUT>
180122 19:47:55 [00]        ...done
180122 19:47:55 Backup created in directory '/data/backup'
MySQL binlog position: filename 'bin.000006', position '154'
180122 19:47:55 [00] Compressing and streaming backup-my.cnf
180122 19:47:55 [00]        ...done
180122 19:47:55 [00] Compressing and streaming xtrabackup_info
180122 19:47:55 [00]        ...done
xtrabackup: Transaction log of lsn (10304786) to (10304795) was copied.
180122 19:47:55 completed OK!
180122 19:47:55 [00]        ...done
180122 19:47:55 Backup created in directory '/data/backup'
MySQL binlog position: filename 'bin.000006', position '154'
180122 19:47:55 [00] Compressing and streaming backup-my.cnf
180122 19:47:55 [00]        ...done
180122 19:47:55 [00] Compressing and streaming xtrabackup_info
180122 19:47:55 [00]        ...done
xtrabackup: Transaction log of lsn (10304786) to (10304795) was copied.
180122 19:47:55 completed OK!

# 生成各种文件，备份结束
```

## Ⅲ、xtrabackup原理分析

### xtrabackup全备步骤
|-|操作|解析|
|:-:|:-:|:-:|
|step1|Connecting to MySQL server host|连接登录|
|step2|using the following InnoDB configuration|读相关配置文件|
|step3|start xtrabackup_log|启用日志文件，记录redo的lsn，同时持续扫描redo log，将新产生的redo拷贝到xtrabackup_logfile|
|step4|copy innodb tables .ibd、.ibdata1、undo logs|拷贝innodb表的独立表空间、共享表空间、undo日志|
|step5|flush no_write_to_binlog tables、flush tables with read lock|强制将commit log刷入redo防止数据丢失(5.6之前没有)，锁表|
|step6|copy non-innodb tables .MYD、.MYI、.opt、misc files和innodb tables .frm、.opt、misc files|拷贝myisam表相关内容和innodb表的表结构文件|
|step7|Get binary log position|获取二进制日志位置点，写入到xtrabackup_binlog_info文件|
|step8|flush no_write_to_binlog engine logs|将redo刷盘|
|step9|stopping log copying thread|停止拷贝|
|step10|unlock tables|释放锁|
|step11|completed OK|生成各种文件，备份结束|

**tips：**

①简单点说：一个线程备份redo，贯穿整个过程始终，另外的线程备份表空间文件，直到completed OK，备份成功

②5.6之前的xtrabackup有丢数据的风险，强烈建议使用最新版本

③和mysqldump、mydumper相比，xtrabackup备份的是结束时间点的数据(二进制文件位置点不一样)，所以物理备份除了本身恢复块之外，同步也快，因为不用拉数据，做一个一小时的备份，逻辑备份需要做一个小时的数据同步，物理备份不需要

④备份过程中遇到myisam还是会阻塞，数据一致性需求

## Ⅳ、xtrabackup备份恢复

### 查看备份文件
由于我这里用的是流文件的方式备份的，所以要先打开流文件
```
[root@VM_0_5_centos backup]# xbstream -x < backup.xbstream
[root@VM_0_5_centos backup]# ll
total 2792
drwxr-x--- 2 root root    4096 Jan 23 15:52 abc
-rw-r----- 1 root root     417 Jan 23 15:52 backup-my.cnf.qp
-rw-r--r-- 1 root root 1822257 Jan 23 15:51 backup.xbstream
drwxr-x--- 2 root root    4096 Jan 23 15:52 dump_test
-rw-r----- 1 root root     370 Jan 23 15:52 ib_buffer_pool.qp
-rw-r----- 1 root root  969374 Jan 23 15:52 ibdata1.qp
drwxr-x--- 2 root root    4096 Jan 23 15:52 mysql
drwxr-x--- 2 root root    4096 Jan 23 15:52 performance_schema
drwxr-x--- 2 root root   12288 Jan 23 15:52 sys
drwxr-x--- 2 root root    4096 Jan 23 15:52 test
-rw-r----- 1 root root     102 Jan 23 15:52 xtrabackup_binlog_info.qp
-rw-r----- 1 root root     115 Jan 23 15:52 xtrabackup_checkpoints
-rw-r----- 1 root root     494 Jan 23 15:52 xtrabackup_info.qp
-rw-r----- 1 root root     391 Jan 23 15:52 xtrabackup_logfile.qp

看到很多qp文件，是因为备份时做了压缩，我们需要将其解压
[root@VM_0_5_centos backup]# for f in `find ./ -iname "*\.qp"`; do qpress -dT4 $f  $(dirname $f) && rm -f $f; done
[root@VM_0_5_centos backup]# ll
total 14152
drwxr-x--- 2 root root     4096 Jan 23 15:55 abc
-rw-r--r-- 1 root root      427 Jan 23 15:55 backup-my.cnf
-rw-r--r-- 1 root root  1822257 Jan 23 15:51 backup.xbstream
drwxr-x--- 2 root root     4096 Jan 23 15:55 dump_test
-rw-r--r-- 1 root root      413 Jan 23 15:55 ib_buffer_pool
-rw-r--r-- 1 root root 12582912 Jan 23 15:55 ibdata1
drwxr-x--- 2 root root     4096 Jan 23 15:55 mysql
drwxr-x--- 2 root root    12288 Jan 23 15:55 performance_schema
drwxr-x--- 2 root root    12288 Jan 23 15:55 sys
drwxr-x--- 2 root root     4096 Jan 23 15:55 test
-rw-r--r-- 1 root root       15 Jan 23 15:55 xtrabackup_binlog_info
-rw-r----- 1 root root      115 Jan 23 15:52 xtrabackup_checkpoints
-rw-r--r-- 1 root root      521 Jan 23 15:55 xtrabackup_info
-rw-r--r-- 1 root root     2560 Jan 23 15:55 xtrabackup_logfile

可以看到，除了备份表空间等，还生成了4个文件
```

看下4个文件
```
[root@VM_0_5_centos backup]# cat xtrabackup_binlog_info           # 记录binlog文件名和position
bin.000006	154
------
[root@VM_0_5_centos backup]# cat xtrabackup_checkpoints           # 记录备份过程中checkpoint、lsn信息
backup_type = full-backuped
from_lsn = 0
to_lsn = 10304786
last_lsn = 10304795
compact = 0
recover_binlog_info = 0
------
[root@VM_0_5_centos backup]# cat xtrabackup_info                  # 整个备份过程中的信息
uuid = 48febc78-0012-11e8-b724-525400a4dac1
name = 
tool_name = innobackupex
tool_command = --compress --compress-threads=8 --stream=xbstream -S /tmp/mysql.sock --parallel=4 ./
tool_version = 2.4.7
ibbackup_version = 2.4.7
server_version = 5.7.20-log
start_time = 2018-01-23 15:51:51
end_time = 2018-01-23 15:51:56
lock_time = 0
binlog_pos = filename 'bin.000006', position '154'
innodb_from_lsn = 0
innodb_to_lsn = 10304786
partial = N
incremental = N
format = xbstream
compact = N
compressed = compressed
encrypted = N
------
xtrabackup_logfile                                                # 持续备份的redo，直接看不了
```

### 恢复一手瞅瞅
```
step1: 应用日志，将backup恢复
[root@VM_0_5_centos mdata]# innobackupex --apply-log backup

step2：将恢复好的数据拷贝到datadir,直接move也行
[root@VM_0_5_centos mdata]# innobackupex --copy-back backup

step3：修改文件属主
[root@VM_0_5_centos mdata]# chown -R mysql:mysql mysql_test_data

step4：启动数据库
/etc/init.d/mysql.server start
Starting MySQL. SUCCESS! 
```

**tips：**

- 日志应用完成后，backup文件中会多出一个文件：xtrabackup_binlog_pos_innodb，记录的是innodb表当前的binlog position，而xtrabackup_binlog_info记录的是整个实例当前的binlog position

- 般情况下，这两个位置点是一样的，但备份时两种引擎都存在时,则xtrabackup_binlog_info > xtrabackup_binlog_pos_innodb 

- 所以我们一般用xtrabackup_binlog_info中的binlog position

 根据之前分析的备份流程可知，先备份innodb表，所以这块也不难理解


## Ⅴ、其他相关问题

### 增量备份
--incremental-history-name=name 可使用改参数做增量备份

但非常不建议用这个增量备份功能，性能特别差

若昨天全备100G，今天更新了30G，做增量要扫描100G文件才知道哪些页改动了，再去备份，线上很难接受

percona有个参数可以监控哪些页改动了，所以不用去扫之前的所有备份的表空间，但用的也比较少

要做增量，用二进制日志的机制来做即可

### 指定库表备份
同样不推荐这种玩法，强烈建议完整备份

备份原理是备份所有表空间(ibd)，不完整备份的话，可能会遇到各种问题

比如备份了a库,没备份b库，用这个备份恢复后在b库下面创建一个和之前同名的表就创建不了

### 远程备份
```
innobackupex --compress --compress-threads=8 --stream=xbstream --user=root --parallel=4 ./ | ssh root@192.168.1.192 "xbstream -x -C /data/www/mysql/backup"
```
