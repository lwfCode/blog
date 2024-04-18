---
title: Linux-Centos 7.9 源码编译安装PHP的RabbitMQ扩展
date: 2024-03-26 14:02:07
tags: RabbitMQ
categories: 后端
top: 9
---


#### 1.RabbitMQ源码下载

###### 1.1 下载，解压rabbitmq-c源码

```
wget -c https://github.com/alanxz/rabbitmq-c/archive/v0.9.0.tar.gz
tar -zxvf v0.9.0.tar.gz
```

#### 2.RabbitMq 配置，安装，编译

2.1 安装
```
cd rabbitmq-c-0.9.0/
mkdir build && cd build #这一步是在rabbitmq-c的根目录下创建一个build子目录
```


```

cmake -DCMAKE_INSTALL_PREFIX=/usr/local/rabbitmq-c ..
# 这一步是真正的build rabbitmq-c库的，注意，不要漏掉点 '.'
cmake --build .  --target install
```

2.2 库软链


```
ln -s /usr/local/rabbitmq-c/lib64 /usr/local/rabbitmq-c/lib
```

2.3 下载，解压amqp


```
wget -c https://pecl.php.net/get/amqp-1.9.4.tgz
tar -zxvf amqp-1.9.4.tgz
cd amqp-1.9.4
```

2.4 生成confingure文件

ps:请自行查询自己服务器php安装位置
```
/usr/local/php/bin/phpize
```
2.5 配置检查，编译，安装


```
./configure --with-php-config=/usr/local/php/bin/php-config --with-amqp --with-librabbitmq-dir=/usr/local/rabbitmq-c
make -j4
make install
```

2.6 安装完成(显示如下)

```
Installing shared extensions:     /usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/
```
#### 3.查看扩展文件
 
[root@tasdffa amqp-1.9.4]# ls /usr/local/php/lib/php/extensions/no-debug-non-zts-20170718/

amqp.so  memcached.so  mongodb.so pdo_pgsql.so  swoole.so  redis.so

3.1 将扩展加入php.ini


```
echo "extension=amqp.so" >> /usr/local/php/etc/php.ini
```

3.2 重启php，查看phpinfo();

```
service php-fpm restart
或
/bin/systemctl restart php-fpm.service
```

搜索 amqp 查看插件是否安装完成
end