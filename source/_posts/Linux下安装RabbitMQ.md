---
title: Linux下安装RabbitMQ
date: 2023-05-26 14:33:42
tags: RabbitMQ
categories: 消息队列
top: 8
---

## Linux下安装 RabbitMQ安装

#### 1.RabbitMQ概念

##### RabbitMQ是流行的开源消息队列系统，很多概念就不说了，官网，百度一大堆，此处略...

#### 2.RabbitMQ必须要安装的依赖

2.1 先安装erlang

<mark>ps:提示没有yum源请自行百度安装<mark>

    curl -s https://packagecloud.io/install/repositories/rabbitmq/erlang/script.rpm.sh | sudo bash

    yum install -y erlang

#### 3.RabbitMQ安装

导入将于2018年12月1日起使用的新PackageCloud密钥(GMT)

    rpm --import https://packagecloud.io/rabbitmq/rabbitmq-server/gpgkey

完成RabbitMQ的前置条件配置==>执行云存储库快速脚本

```

curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash

yum install rabbitmq-server -y
```

又或者下载rabbitmq-server-3.8.9-1.el7.noarch.rpm到服务器

<mark>ps:我自己是没用到这个步骤，看你自己选择<mark>

    yum install -y rabbitmq-server-3.8.9-1.el7.noarch.rpm

#### 3.php扩展 php-amqplib安装 （看个人选择，我是选择原生mq，没用php-amqp扩展）

    composer require php-amqplib/php-amqplib

#### 4.安装rabbitmq插件

(1)查找rabbitmq：whereis rabbitmq

(2)列出rabbitmq执行文件：ll /usr/sbin/ | grep 'rabbit'

(3)检查是否启动成功\:ps -ef | grep rabbitmq

(4)查看所有插件rabbitmq-plugins list

4.1 命令

启动：

方法一 rabbitmq-server -detached

方法二 systemctl start rabbitmq-server

方法三 service rabbitmq-server start

停止：rabbitmqctl stop

状态：rabbitmqctl status

#### 5.WEB管理

(1)开启web插件，启用管理平台插件后，可以可视化管理RabbitMQ

    rabbitmq-plugins enable rabbitmq_management

(2)关闭管控台

    rabbitmq-plugins disable rabbitmq_management

访问：<http://host:15672/> （这里如果是云服务器，需自行开启 15672，5672端口）
如果您安装的是3.x系统以上版本，则无法使用默认用户登陆，请参考步骤 6


rabbitmq默认端口（如果用其它协议，还有其它口，参照rabbitmq官网说明）

client端通信口：5672

管理口：15672

server间内部通信口：25672

erlang发现口：4369

#### 6.rabbitmq的用户管理

查看所有用户rabbitmqctl list\_users

添加一个用户rabbitmqctl add\_user leiwenfeng

配置权限，授权远程访问（也可以登录后，可视化配置）

    rabbitmqctl set_permissions -p "/" leiwenfeng "." "." ".*"

查看用户权限

    rabbitmqctl list_user_permissions leiwenfeng

设置tag（设置用户为超级管理员，Tag可以为administrator,monitoring,management）

    rabbitmqctl set_user_tags leiwenfeng administrator

删除用户（安全起见，删除默认用户）

```
rabbitmqctl delete_user guest（3.x系统后默认用户不允许登陆，此步骤看个人）
```

创建完成后，重启RabbitMQ

    systemctl restart rabbitmq-server

设置开机自启

    chkconfig rabbitmq-server on systemctl enable rabbitmq-server.service

== End
