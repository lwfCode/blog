---
title: NPS搭建内网穿透代理服务器
date: 2025-12-19 10:52:39
tags: NPS
categories: 网络
top: 33
---

NPS是一个基于Go语言开发的内网穿透代理服务器，它可以将内网中的服务暴露到公网，实现远程访问。

## 服务端
```
#下载nps服务端程序
wget https://github.com/ehang-io/nps/releases/download/v0.26.10/linux_amd64_server.tar.gz
```
### 安装
```
sudo ./nps install
```
### 修改配置文件

```
vim /etc/nps/conf/nps.conf

#修改默认的http 、https 的端口

http_proxy_port=980 （用非80端口，自定义）
https_proxy_port=9443（用非443端口，自定义）
```
### 启动
```
sudo nps start
```
### 调试（可选）
```
sudo nps 

输出如下没有err错误代表启动成功：
2025/12/19 09:34:11.999 [I] [nps.go:202]  the version of server is 0.26.10 ,allow client core version to be 0.26.0
2025/12/19 09:34:12.108 [I] [connection.go:36]  server start, the bridge type is tcp, the bridge port is 8024
2025/12/19 09:34:12.131 [I] [server.go:200]  tunnel task  start mode：httpHostServer port 0
2025/12/19 09:34:12.131 [I] [connection.go:71]  web management start, access port is 8080
2025/12/19 09:34:12.131 [I] [connection.go:62]  start https listener, port is 9443
2025/12/19 09:34:12.131 [I] [connection.go:53]  start http listener, port is 980
```

## 访问NPS web管理后台

默认账号密码：admin/123

![alt text](/images/nps/nps_web.png)
![alt text](/images/nps/nps_web_admin.png)


## 客户端：

#下载nps客户端程序

```
wget https://github.com/ehang-io/nps/releases/download/v0.26.10/linux_amd64_client.tar.gz
```

### npc放在 /data/skyinfor/nps目录(自定义)下

#解压
```
tar -zxvf linux_amd64_client.tar.gz -C /data/skyinfor/nps
```

```
#编辑linux的systemctl配置文件来管理
vim /etc/systemd/system/npc.service

[Unit]
Description=NPS Client Service
After=network.target           
Wants=network.target

[Service]
Type=simple                    
User=root                      
Group=root
WorkingDirectory=/data/skyinfor/nps
ExecStart=/data/skyinfor/nps/npc -server=公网服务器ip:8024 -vkey=你的密钥
ExecReload=/bin/kill -HUP $MAINPID
ExecStop=/bin/kill -TERM $MAINPID
Restart=always                
RestartSec=5                 
TimeoutSec=30                 

[Install]
WantedBy=multi-user.target
```

# 重载配置
```
systemctl daemon-reload
```
# 启动npc服务
```
systemctl start npc
```

# 停止npc服务
```
systemctl stop npc
```

# 重启npc服务
```
systemctl restart npc
```
# 查看服务状态（确认是否运行正常）
```
systemctl status npc
```
# 设置开机自启
```
systemctl enable npc
```
# 关闭开机自启
```
systemctl disable npc
```
# 查看服务日志（排查启动失败问题）
```
journalctl -u npc -f  # -f 实时查看日志
```

#### 客户端启动如下

![alt text](/images/nps/nps_clien.png)

#### 在NPS web管理后台添加客户端获取密钥

![alt text](/images/nps/nps1.png)

#### 新增TCP隧道，把内网服务器的ssh服务映射到公网服务器的8001端口

![alt text](/images/nps/nps2.png)

###### 用公网ip+端口，进行ssh链接本地内网服务器

![alt text](/images/nps/nps3.png)








