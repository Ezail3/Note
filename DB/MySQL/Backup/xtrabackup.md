---
# MySQL物理备份介绍
---

## xtrabackup介绍

- xtrabackup只能备份innodb存储引擎的数据，不能备份表结构，percona开源的，强烈推荐最新版本(旧版本bug多)
- innobackupex可以备份多种引擎的数据和表结构，一般用这个
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

输出内容（简化）
```
```
