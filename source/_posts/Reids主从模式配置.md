---
title: Reids主从模式配置
date: 2024-03-27 13:26:42
tags: Redis
categories: 数据库
top: 8
---

## 操作系统：Linux Centos7.9（可支持docker部署）

<mark>注意集群中各服务器的时间需要相同，可配置向同一 NTP 服务器校时<mark>

| master节点IP   |   slave节点IP
|   ---          |     ---
| 192.168.6.185  |  192.168.6.121

#### docker部署(可选)：

```
docker pull centos:7.9.2009

#需要开启特权模式否则容器内无法执行systemctl
docker run -it -d --name="RedisMaster" --privileged=true  -v /sys/fs/cgroup:/sys/fs/cgroup -p 6379:6379 centos:7.9.2009 /usr/sbin/init

#安装必要依赖
yum -y install wget
yum -y install libaio
yum -y install numactl

docker exec -it RedisMaster bash
```
- docker 部署方式，在slave机器上重复执行以上操作，区分修改一下容器名称

一、Redis Master

1.下载redis安装包

```
百度网盘链接: https://pan.baidu.com/s/1OQvHGQ3BF9ZsFd51vAbCrA  密码: i2mi

服务器下载：wget http://pdpublic.mingdao.com/private-deployment/offline/common/redis-5.0.14-glibc2.17.tar.gz
```

2.解压到安装目录

```
tar -zxvf redis-5.0.14-glibc2.17.tar.gz
mv redis-5.0.14-glibc2.17 /usr/local/redis
```

3.调整内核参数

```
echo 'net.core.somaxconn = 32768' >> /etc/sysctl.conf
echo 'vm.overcommit_memory = 1' >> /etc/sysctl.conf
sysctl -p
```

4.创建 redis 数据目录

```
mkdir -p /data/redis
```

5.修改配置文件


```
cat > /usr/local/redis/redis.conf <<EOF
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /usr/local/redis/redis.pid
loglevel notice
logfile /usr/local/redis/redis.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 0 0 0
hz 10
requirepass 123456
masterauth 123456
maxmemory 16gb
maxmemory-policy allkeys-lru
maxclients 100000
EOF
```
- redis 认证与主从认证的密码为 123456，实际部署时注意替换

6.创建 redis 用户授权

```
useradd -U -M -s /sbin/nologin redis
chown -R redis.redis /usr/local/redis/ /data/redis/
```

7.配置 systemd 管理

```
cat > /etc/systemd/system/redis.service <<EOF
[Unit]
Description=Redis
[Service]
User=redis
Group=redis
TasksMax=infinity
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=0
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
ExecStop=/usr/bin/kill \$MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

8.启动 redis 服务并加入开机自启动

```
systemctl start redis
systemctl enable redis
```

## ==二、Redis Slave==

1.下载 redis 安装包

```
百度网盘链接: https://pan.baidu.com/s/1OQvHGQ3BF9ZsFd51vAbCrA  密码: i2mi

服务器下载：wget http://pdpublic.mingdao.com/private-deployment/offline/common/redis-5.0.14-glibc2.17.tar.gz
```
2.解压到安装目录

```
tar -zxvf redis-5.0.14-glibc2.17.tar.gz
mv redis-5.0.14-glibc2.17 /usr/local/redis
```

3.调整内核参数

```
echo 'net.core.somaxconn = 32768' >> /etc/sysctl.conf
echo 'vm.overcommit_memory = 1' >> /etc/sysctl.conf
sysctl -p
```
4.创建 redis 数据目录

```
mkdir -p /data/redis
```

5.修改配置文件


```
cat > /usr/local/redis/redis.conf <<EOF
bind 0.0.0.0
protected-mode yes
port 6379
tcp-backlog 511
timeout 0
tcp-keepalive 300
daemonize no
supervised no
pidfile /usr/local/redis/redis.pid
loglevel notice
logfile /usr/local/redis/redis.log
databases 16
save 900 1
save 300 10
save 60 10000
stop-writes-on-bgsave-error yes
rdbcompression yes
rdbchecksum yes
dbfilename dump.rdb
dir /data/redis
slave-serve-stale-data yes
slave-read-only yes
repl-diskless-sync no
repl-diskless-sync-delay 5
repl-disable-tcp-nodelay no
slave-priority 100
lua-time-limit 5000
slowlog-log-slower-than 10000
slowlog-max-len 128
latency-monitor-threshold 0
notify-keyspace-events ""
hash-max-ziplist-entries 512
hash-max-ziplist-value 64
list-max-ziplist-size -2
list-compress-depth 0
set-max-intset-entries 512
zset-max-ziplist-entries 128
zset-max-ziplist-value 64
hll-sparse-max-bytes 3000
activerehashing yes
client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 0 0 0
hz 10
requirepass 123456
masterauth 123456
maxmemory 16gb
maxmemory-policy allkeys-lru
maxclients 100000
slaveof 192.168.6.185 6379
EOF
```

- redis 认证与主从认证的密码为 123456，实际部署时注意替换
- redis master ip地址注意在实际部署过程中进行替换

6.创建 redis 用户授权

```
useradd -U -M -s /sbin/nologin redis
chown -R redis.redis /usr/local/redis/ /data/redis/
```

7.配置 systemd 管理

```
cat > /etc/systemd/system/redis.service <<EOF
[Unit]
Description=Redis
[Service]
User=redis
Group=redis
TasksMax=infinity
LimitNOFILE=infinity
LimitNPROC=infinity
LimitCORE=0
ExecStart=/usr/local/redis/bin/redis-server /usr/local/redis/redis.conf
ExecStop=/usr/bin/kill \$MAINPID
Restart=on-failure
[Install]
WantedBy=multi-user.target
EOF
```

8.启动 redis 服务并加入开机自启动

```
systemctl start redis
systemctl enable redis
```

查看同步状态

```
/usr/local/redis/bin/redis-cli -a 123456 info replication
```
显示如下：

```
Master节点：

# Replication
role:master
connected_slaves:1
slave0:ip=192.168.6.121,port=6379,state=online,offset=92638,lag=1
master_replid:c40383b82a707d5d7e1373cb05bcaa435ea75298
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:92638
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:92638

Slave节点：

# Replication
role:slave
master_host:192.168.6.185
master_port:6379
master_link_status:up
master_last_io_seconds_ago:2
master_sync_in_progress:0
slave_repl_offset:92680
slave_priority:100
slave_read_only:1
connected_slaves:0
master_replid:c40383b82a707d5d7e1373cb05bcaa435ea75298
master_replid2:0000000000000000000000000000000000000000
master_repl_offset:92680
second_repl_offset:-1
repl_backlog_active:1
repl_backlog_size:1048576
repl_backlog_first_byte_offset:1
repl_backlog_histlen:92680
```




