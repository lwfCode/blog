---
title: Linux环境下安装Docker
date: 2024-03-27 13:30:01
tags: Docker
categories: Docker
top: 3
---
## 前沿

Docker是一种虚拟化容器技术，基于镜像，可以秒级启动各种容器。每一种容器都是一个完整的运行环境，容器之间互相隔离。


#### 1.下载 docker 安装包

```
百度网盘下载链接: https://pan.baidu.com/s/1E_s-GR_GH65f3pcKQMzQzg  密码: wbhi
服务器支持互联网下载：wget http://pdpublic.mingdao.com/private-deployment/offline/common/docker-20.10.16.tgz
```
#### 2.解压，并将文件移动到二进制文件目录

```
tar -zxvf docker-20.10.16.tgz

mv -f docker/* /usr/bin/
```

#### 3.创建 docker 配置文件

- 默认 docker 数据目录在 /data/docker，如果需要修改默认数据目录，请修改配置文件中的 data-root 值

```
mkdir -p  /etc/docker/

cat > /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": ["https://uvlkeb6d.mirror.aliyuncs.com"],
  "data-root": "/data/docker",
  "max-concurrent-downloads": 10,
  "exec-opts": ["native.cgroupdriver=cgroupfs"],
  "storage-driver": "overlay2",
  "default-address-pools":[{"base":"172.80.0.0/16","size":24}]
}
EOF
```

#### 4.配置 systemd 管理 docker

```
cat > /etc/systemd/system/docker.service <<EOF
[Unit]
Description=Docker
After=network-online.target
Wants=network-online.target
[Service]
Type=notify
ExecStart=/usr/bin/dockerd
ExecReload=/bin/kill -s HUP \$MAINPID
LimitNOFILE=102400
LimitNPROC=infinity
LimitCORE=0
TimeoutStartSec=0
Delegate=yes
KillMode=process
Restart=on-failure
StartLimitBurst=3
StartLimitInterval=60s
[Install]
WantedBy=multi-user.target
EOF
```

#### 5.启动 docker

```
systemctl daemon-reload && systemctl start docker && systemctl enable docker

#查看是否启动
systemctl status docker
```




