---
title: Jenkins+gitlab前后端项目自动化部署
date: 2023-02-27 13:52:40
tags: Jenkins
categories: Jenkins
top: 3
---

## 前言

通过Jenkins与GitLab联动，当gitlab仓库中代码发生变动（增、删、改），自动触发Jenkins自动构建发布，实现自动化运维。

思路：
1、安装部署gitlab、安装部署Jenkins
2、jenkins安装功能插件
3、安装git工具
4、Jenkins job配置构建触发器
5、gitlab仓库配置webhooks

#### 1.配置gitlab的密钥

首页->管理员->凭据->系统->全局凭证

点击新增按钮

![凭证](/images/access.jpg)

#### 2.首页创建任务(项目)

##### 2.1 输入任务名称

随便取名称，最好是跟你前后端项目名称相关联，选择第一个，构建自动风格的软件项目。

找到源码管理，选择 <mark>Git</mark> 选项输入你的git仓库地址，在Credentials下面输入框选择第一步创建的全局凭证，填写需要触发的git分支（如develop分支）如图所示

![凭证](/images/2.jpg)

##### 2.2 构建触发器

找到 Build when a change is pushed to GitLab. GitLab webhook URL 选项，点击勾选，复制 “webhook URL”后面的地址（在gitlab上配置webhooks会用到），找到 <mark>高级</mark>按钮选项，点击按钮往下拉，找到Secrt token ，在右下角点击 <mark>“Generate”</mark>生成token，

![凭证](/images/3.jpg)

复制token以及webhook地址到gitlab上配置。

![凭证](/images/4.jpg)

gitlab上点击项目选择左侧菜单栏的 "设置->Web钩子"

![凭证](/images/jenkins/11.jpg)

点击新增Web钩子，然后回到jenkins界面

#### 3.配置自动化构建脚本

###### 3.1 找到 Build Steps 下拉框选择 执行shell 

```
cd /data/jenkins
sh build.sh
```

##### 在服务器的 /data/jenkins 目录下编写shell脚本 build.sh

```
mkdir -p /data/jenkins
cd /data/jenkins
touch build.sh
```

```
#!/bin/sh
cd /home/skyinfor/AI-workflow
rm -rf dist
git checkout .
git pull origin develop
npm i
npm run build
sudo cp -rf dist/*  /你的nginx或者apache 根目录下(根据自己web服务器的根目录自行设置,如 /usr/local/nginx/html/)
```

最终效果如图所示

![凭证](/images/jenkins/12.jpg)


###### 后端API项目同理（正常来说只需改改shell脚本构建步骤，git仓库地址即可）！