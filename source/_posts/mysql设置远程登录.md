---
title: mysql设置远程登录
date: 2025-01-01 10:45:14
tags: MySql
categories: 数据库
top: 22
---

### 1.流程描述

| 步骤     | 说明  |
| -----    | ---  |
| 1 | 修改mysql配置文件|
| 2 | 重启mysql服务|
| 3  |  创建或修改用户权限  |
| 4 |  测试远程连接  |

### 2.修改mysql配置文件

MySQL的配置文件通常位于/etc/mysql/my.cnf或/etc/my.cnf

首先，使用以下命令打开配置文件

```
sudo vi /etc/mysql/my.cnf
```
找到 [mysqld] 打开注释或者添加以下配置，允许所有IP远程访问

```
bind-address = 0.0.0.0
```

### 3.重启MySQL服务

```
sudo systemctl restart mysql
```

### 4.创建或修改用户权限

登录到MySQL控制台,使用以下命令输入密码进入mysql终端

```
mysql -uroot -h 127.0.0.1 -p
```

假设我们要设置一名为root的用户，密码为abcd@1234，并允许从任何IP登录

```
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'abcd@1234' WITH GRANT OPTION;
```
刷新权限
```
FLUSH PRIVILEGES;

#查看mysql用户表记录
select host,user from mysql.user;
```

如下图所示

![alt text](/images/mysql/auth.jpg)

### 5.测试远程连接

![alt text](/images/mysql/remote_debug.jpg)