---
# MySQL物理备份介绍
---

## xtrabackup介绍

- xtrabackup只能备份innodb引擎的数据，不能备份表结构，percona开源的，强烈推荐最新版本(旧版本bug多)
- innobackupex可以备份myisam和innodb两种引擎的数据和表结构，一般用这个
- 备份时，默认读取MySQL配置文件(datadir)

## xtrabackup安装使用

### 安装
```
yum install perl-DBD-MySQL
不安装这个备份会报错：Failed to connect to MySQL server: DBI connect

cd /usr/local/src
wget https://www.percona.com/downloads/XtraBackup/Percona-XtraBackup-2.4.7/binary/tarball/percona-xtrabackup-2.4.7-Linux-x86_64.tar.gz
tar zxvf percona-xtrabackup-2.4.7-Linux-x86_64.tar.gz -C ..
添加环境变量
cd ..
ln -s percona-xtrabackup-2.4.7-Linux-x86_64/ xtrabackup
echo "PATH=/usr/local/xtrabackup/bin:$PATH" >> /etc/profile
source /etc/profile
```

### 玩一手
```
innobackupex --compress --compress-threads=8 --stream=xbstream -S /tmp/mysql.sock --parallel=4 /tmp > backup.xbstream
建议用-S连接，默认走socket，不用-S可能报连不上
```

压缩线程8，备份线程4

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

180122 19:47:55 Executing FLUSH NO_WRITE_TO_BINLOG ENGINE LOGS...
xtrabackup: The latest check point (for incremental): '10304786'
180122 19:47:55 >> log scanned up to (10304795)
xtrabackup: Stopping log copying thread.

180122 19:47:55 Executing UNLOCK TABLES
180122 19:47:55 All tables unlocked
180122 19:47:55 [00] Compressing and streaming ib_buffer_pool to <STDOUT>
180122 19:47:55 [00]        ...done
180122 19:47:55 Backup created in directory '/tmp/'
MySQL binlog position: filename 'bin.000006', position '154'
180122 19:47:55 [00] Compressing and streaming backup-my.cnf
180122 19:47:55 [00]        ...done
180122 19:47:55 [00] Compressing and streaming xtrabackup_info
180122 19:47:55 [00]        ...done
xtrabackup: Transaction log of lsn (10304786) to (10304795) was copied.
180122 19:47:55 completed OK!
```
