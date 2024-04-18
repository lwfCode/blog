---
title: PHP安装mcrypt扩展
date: 2023-05-28 11:00:23
tags: PHP
categories: 后端
top: 12
---
### 前言

自从服务器升级 php 7.2 后，在composer下载包时候老是提示 call to undefined function mcrypt_module_open()，为此查阅诸多资料皆无果。

最后还是看官方手册得出结论：php 7.2的扩展有变动；mcrypt 扩展从 php 7.1.0 开始废弃；自 php 7.2.0 起，会移到 pecl ，不过还好，安装过程不复杂。

---
环境：Linux Centos7

- yum 安装依赖包：

```
yum install libmcrypt libmcrypt-devel mcrypt mhash
```

- 在 php 官网下载 mcrypt包，[php扩展官网](http://pecl.php.net/package/mcrypt)


```
wget  http://pecl.php.net/get/mcrypt-1.0.1.tgz
tar xf mcrypt-1.0.1.tgz
cd mcrypt-1.0.1
```

- 编译安装 mcrypt


```
 /usr/local/php/bin/phpize
 ./configure --with-php-config=/usr/local/php/bin/php-config  && make && make install
```

最后在php.ini中加上扩展即可

```
extension=mcrypt.so
/etc/init.d/php-fpm restart
```
至此大功告成，composer下载包正常，真香!