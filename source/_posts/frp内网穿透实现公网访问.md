---
title: frp内网穿透实现公网访问
date: 2026-03-15 11:03:54
tags: fpc
categories: 网络
top: 37
---

### 内网穿透实现公网访问
FRP 内网穿透核心原理是：公网服务器跑 frps（服务端）、内网设备跑 frpc（客户端），通过配置建立隧道，把内网服务暴露到公网。

#### 一、准备工作

- 一台有公网 IP 的云服务器（阿里云 / 腾讯云 / 华为云等，2 核 2G 即可）
- 内网设备（电脑 / NAS / 树莓派等，需访问的服务已运行）
- 下载 FRP：https://github.com/fatedier/frp/releases

安装解压

```bash
# 以 Linux 64 位为例
mkdir -p /home/skyinfor
wget https://github.com/fatedier/frp/releases/download/v0.44.0/frp_0.44.0_linux_amd64
tar -zxvf frp_0.44.0_linux_amd64.tar.gz
mv frp_0.44.0_linux_amd64 /home/skyinfor/frp
```

#### 二、配置 frps（服务端）

编辑 frps.ini：

```ini
[common]
bind_port = 7000
dashboard_port = 7500
dashboard_user = admin
dashboard_pwd = admin123  # 登录 frp 管理界面的用户名密码,默认 admin/admin123
```

启动服务端：
```bash
#前台运行（测试用）
./frps -c ./frps.ini

#后台运行（推荐）
nohup ./frps -c frps.ini > /dev/null 2>&1 &
```
#### 开放端口（安全组 + 防火墙）

- 云服务器控制台安全组：放行 7000、7500（及后续业务端口）
- 系统防火墙（如 ufw）：

```bash
ufw allow 7000/tcp
ufw allow 7500/tcp
ufw reload
```

##### 设置开机自启（Systemd）

sudo vim /etc/systemd/system/frps.service
```
[Unit]
Description=FRP Server
After=network.target

[Service]
ExecStart=/home/skyinfor/frp/frps -c /home/skyinfor/frp/frps.ini  # 启动服务端，指定配置文件，请自行修改为实际路径
Restart=always

[Install]
WantedBy=multi-user.target
```

##### 启用 frps 服务
```bash
sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
```

#### 二、配置 frpc（客户端） 内网设备

1、安装与解压（同服务端，下载对应系统版本）
2、编辑 frpc.ini：
```ini
[common]
server_addr = 你的服务器公网IP
server_port = 7000

[ssh_5081]
type = tcp
local_ip = 127.0.0.1
local_port = 22     # 本地服务端口
remote_port = 5081  # 公网访问端口，可自定义，避免冲突

[web_80]
type = tcp
local_ip = 127.0.0.1
local_port = 80  # 本地服务端口
remote_port = 5082  # 公网访问端口，可自定义，避免冲突
```

3、启动客户端：
```bash
#前台运行（测试用）
./frpc -c ./frpc.ini

#linux后台运行（推荐）
nohup ./frpc -c frpc.ini > /dev/null 2>&1 &

#Windows
frpc.exe -c frpc.ini
```

###### Linux开机自启（Systemd）
sudo vim /etc/systemd/system/frpc.service

```
[Unit]
Description=FRP Client (frpc)  
After=network.target 

[Service]
User=root
Group=root
Type=simple
WorkingDirectory=/home/skyinfor/frp/ 
ExecStart=/home/skyinfor/frp/frpc -c /home/skyinfor/frp/frpc.ini 
Restart=always  
RestartSec=3  
#StandardOutput=/data/log/frpc/stdout.log  # 标准输出日志（替换为你的日志路径）
#StandardError=/data/log/frpc/stderr.log  # 错误输出日志（替换为你的日志路径）

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable frpc
sudo systemctl start frpc
```

#### 三、验证访问
- 公网访问：`ssh -p 5081 root@你的服务器公网IP`（默认 22 端口）
- 公网访问：`http://你的服务器公网IP:5082`（默认 80 端口）

#### 四、常见问题

- 连接失败：检查服务器安全组、防火墙是否放行端口
- 访问慢：可能是网络问题，尝试更换服务器位置或优化网络
- 无法访问服务：确认内网服务正常运行；local_port 正确配置
