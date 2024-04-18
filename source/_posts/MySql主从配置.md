---
title: MySql主从配置
date: 2024-03-27 13:22:27
tags: MySql
categories: 数据库
top: 9
---

## 操作系统：Centos7.9（可支持docker部署）

<mark>注意集群中各服务器的时间需要相同，可配置向同一 NTP 服务器校时<mark>

| master节点IP   |   slave节点IP
|   ---          |     ---
| 192.168.6.185  |  192.168.6.121

#### docker部署(可选)：

```
docker pull centos:7.9.2009

#需要开启特权模式否则容器内无法执行systemctl
docker run -it -d --name="mysqlMaster" --privileged=true  -v /sys/fs/cgroup:/sys/fs/cgroup -p 3306:3306 centos:7.9.2009 /usr/sbin/init

#进入容器
docker exec -it mysqlMaster bash

#安装必要依赖
yum -y install wget
yum -y install libaio
yum -y install numactl
```
- docker 部署方式，在slave机器上重复执行以上操作

### 一、Mysql Master

1.下载安装包

```
百度网盘链接: https://pan.baidu.com/s/1Y-dYeEbA5TzAxkdc_4ruvw  密码: jvmm
服务器下载：wget http://pdpublic.mingdao.com/private-deployment/offline/common/mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
```

2.解压至安装目录


```
tar -zxvf mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.42-linux-glibc2.12-x86_64 /usr/local/mysql
```

3.创建 mysql 用户

```
useradd -U -M -s /sbin/nologin mysql
```

4.创建数据、日志目录并授予权限 


```
mkdir -p /data/mysql/ /data/logs/mysql
chown -R mysql.mysql /usr/local/mysql/ /data/mysql/ /data/logs/mysql
```

5.配置 systemd 管理文件

```
cat > /etc/systemd/system/mysql.service <<EOF
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Service]
User=mysql
Group=mysql
Type=forking
PIDFile=/usr/local/mysql/mysqld.pid
TimeoutSec=0
PermissionsStartOnly=true
ExecStart=/usr/local/mysql/bin/mysqld --daemonize --log-error=/data/logs/mysql/mysqld.log --datadir=/data/mysql --socket=/usr/local/mysql/mysql.sock --character-set-server=utf8 --pid-file=/usr/local/mysql/mysqld.pid --server-id=1 --log-bin=mysql-bin --max_connections=2000 --slow_query_log=1 --slow_query_log_file=/data/logs/mysql/mysql-slow.log
LimitNOFILE=5000
Restart=on-failure
RestartPreventExitStatus=1
PrivateTmp=false
[Install]
WantedBy=multi-user.target
EOF
```

6.初始化

```
/usr/local/mysql/bin/mysqld --initialize --datadir=/data/mysql/ --user=mysql --log-error=/data/logs/mysql/mysqld.log
```

7.启动 mysql 并加入开机自启动

```
systemctl daemon-reload
systemctl enable mysql
systemctl start mysql
```

8.修改 mysql 密码

```
/usr/local/mysql/bin/mysql -h127.0.0.1 -uroot -p$(grep 'temporary password' /data/logs/mysql/mysqld.log  | awk '{print $NF}')
set PASSWORD = PASSWORD('123456');
GRANT ALL ON *.* to root@'%' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
quit;
```

- 命令中修改的 root 用户密码为 123456，实际部署时注意替换


## 二、Mysql Slave

1. 下载 mysql 安装包

```
百度网盘链接: https://pan.baidu.com/s/1Y-dYeEbA5TzAxkdc_4ruvw  密码: jvmm
服务器下载：wget http://pdpublic.mingdao.com/private-deployment/offline/common/mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
```

2.解压至安装目录

```
tar -zxvf mysql-5.7.42-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.7.42-linux-glibc2.12-x86_64 /usr/local/mysql
```

3.创建 mysql 用户

```
useradd -U -M -s /sbin/nologin mysql
```

4.创建数据、日志目录并授予权限

```
mkdir -p /data/mysql/ /data/logs/mysql
chown -R mysql.mysql /usr/local/mysql/ /data/mysql/ /data/logs/mysql
```

5.配置 systemd 管理文件


```
cat > /etc/systemd/system/mysql.service <<EOF
[Unit]
Description=MySQL Server
Documentation=man:mysqld(8)
Documentation=http://dev.mysql.com/doc/refman/en/using-systemd.html
After=network.target
After=syslog.target
[Service]
User=mysql
Group=mysql
Type=forking
PIDFile=/usr/local/mysql/mysqld.pid
TimeoutSec=0
PermissionsStartOnly=true
ExecStart=/usr/local/mysql/bin/mysqld --daemonize --log-error=/data/logs/mysql/mysqld.log --datadir=/data/mysql --socket=/usr/local/mysql/mysql.sock --character-set-server=utf8 --pid-file=/usr/local/mysql/mysqld.pid --server-id=2 --log-bin=mysql-bin --max_connections=2000 --slow_query_log=1 --slow_query_log_file=/data/logs/mysql/mysql-slow.log
LimitNOFILE=5000
Restart=on-failure
RestartPreventExitStatus=1
PrivateTmp=false
[Install]
WantedBy=multi-user.target
EOF
```

6.初始化

```
/usr/local/mysql/bin/mysqld --initialize --datadir=/data/mysql/ --user=mysql --log-error=/data/logs/mysql/mysqld.log
```

7.启动 mysql 并加入开机自启动

```
systemctl daemon-reload
systemctl enable mysql
systemctl start mysql
```

8.修改 mysql 密码

```
/usr/local/mysql/bin/mysql -h127.0.0.1 -uroot -p$(grep 'temporary password' /data/logs/mysql/mysqld.log  | awk '{print $NF}')
set PASSWORD = PASSWORD('123456');
GRANT ALL ON *.* to root@'%' IDENTIFIED BY '123456';
FLUSH PRIVILEGES;
quit;
```
- 命令中修改的 root 用户密码为 123456，实际部署时注意替换


## 三、配置主从

1.登录 master 节点，配置主从同步用户

```
/usr/local/mysql/bin/mysql --socket=/usr/local/mysql/mysql.sock -uroot -p123456
grant replication slave on *.* to 'repl'@"%" identified by "123456";
flush privileges;
show master status;
```

- show master status 执行显示如下：
```
如：
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |     1265 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
```


2.登录 slave 节点，配置启用主从

```
/usr/local/mysql/bin/mysql --socket=/usr/local/mysql/mysql.sock -uroot -p123456
change master to master_host="192.168.6.185",master_port=3306,master_user="repl",master_password="123456",master_log_file="mysql-bin.000001",master_log_pos=1265;
start slave;
```

- change master 语句中 master_host 的地址注意替换为部署时实际的 master 服务器IP
- change master 语句中 master_log_file , master_log_pos 的值为在 master 节点执行 show master status; 看到的输出，如有不同请以实际部署时为准


3.检查主从同步状态

```
show slave status\G

# 输出结果中 Slave_IO_Running 与 Slave_SQL_Running 均为 Yes 代表主从同步正常
```
