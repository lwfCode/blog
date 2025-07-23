---
title: Python+Flask+DockerFile部署构建
date: 2025-03-22 17:21:46
tags: Python
categories: Python
top: 26
---

### 前言
使用Flask框架写完API服务后，利用Docker容器化部署是非常常见的操作，它可以帮你轻松管理和扩展你的应用。

假设目录结构如下：

```
app/
├── config
├── database
├── static
├── routes
├── logs
├── service
└── docker/
    └── Dockerfile
└── shell/
    └── build.sh
    └── python_3.9-slim.tar.gz #本地上传等基础镜像包
├── app.py
├── .env
├── requirements.txt
```



#### 一、创建 Flask 应用并确保其可以在本地运行

首先，确保你有一个可以在本地运行的 Flask 应用，如下所示：
```
# app.py
from flask import Flask

app = Flask(__name__)

@app.route('/')
def hello_world():
    return 'Hello, World!'

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8090)

```

#### 二、编写DockerFile以构建Docker镜像，包含Flask应用及其依赖

接下来，你需要编写一个DockFile来定义如何构建docker镜像，比如：

###### docker/DockerFile
```
#基础镜像
FROM python:3.9-slim

#创建工作目录
WORKDIR /app

#拷贝所有文件
COPY . .

#安装所需依赖
RUN pip install --no-cache-dir -r requirements.txt
EXPOSE 8090

CMD ["gunicorn", "-w", "4", "-b", "0.0.0.0:8090", "app:app"]
```

#### 二、构建镜像


==重要：构建时要指定 Dockerfile 路径，并以项目根目录为构建上下==
```
docker build -t api-flask -f docker/Dockerfile .
```

- -f docker/Dockerfile: 指定 Dockerfile 路径。

- . 是构建上下文（表示当前目录，必须包含你的 app.py 和 requirements.txt）。

#### 三、启动容器

```
docker run -it -d --name="api" --restart=always -p 8090:8090 api-flask:latest
```


#### 四、通过shell脚本一键构建 (推荐)

创建构建shell脚本，如下所示 (路径：shell/buil.sh)：
###### build.sh
```
#!/bin/bash

# === 配置 ===
IMAGE_NAME="python:3.9-slim"
IMAGE_TAR="python_3.9-slim.tar.gz"
BUILD_IMAGE_NAME="web-api:latest"

echo "🚀 检查本地镜像是否存在：$IMAGE_NAME"

# 检查本地是否已存在 base 镜像
if ! docker image inspect "$IMAGE_NAME" > /dev/null 2>&1; then
  echo "🔍 未找到本地镜像 $IMAGE_NAME"

  # 尝试从 tar 加载
  if [ -f "$IMAGE_TAR" ]; then
    echo "📦 加载本地镜像包：$IMAGE_TAR"
    docker load -i "$IMAGE_TAR"
  else
    echo "⚠️ 本地也没有镜像包，稍后将从远程仓库拉取镜像 $IMAGE_NAME"
  fi
else
  echo "✅ 本地已存在镜像 $IMAGE_NAME"
fi

# 构建镜像
echo "🔧 开始构建镜像 $BUILD_IMAGE_NAME ..."
if [ -d ../docker ]; then
  cd ..
else
  echo "❌ 上一级目录中没有 docker 文件夹以及dockerFile文件"
  exit 1
fi

docker build -t web-api -f docker/Dockerfile .
docker run -it -d --name="web-api" --restart=always -p 8090:8090 web-api:latest

if [ $? -eq 0 ]; then
  echo "🎉 镜像构建成功：$BUILD_IMAGE_NAME"
else
  echo "❌ 镜像构建失败，请检查 Dockerfile 或依赖。"
fi

```

#### 五、最终构建如下

```
cd /shell
chmod +x build.sh
./build.sh
```
启动构建DockerFile并安装flask依赖
![alt text](/images/file/build_1.jpg)

最终启动成功服务器docker上监听8090端口，api服务部署成功
![alt text](/images/file/build_2.jpg)

#### 问题
DockFile中第一步骤，拉取 FROM python:3.9-slim 镜像有时无法拉取问题

解决方案：
```
1、切换国内镜像源，如淘宝、华为、清华等镜像源等

sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://registry.docker-cn.com",
    "https://mirror.baidubce.com",
    "https://docker.mirrors.ustc.edu.cn"
  ]
}
EOF
#重启docker
sudo systemctl daemon-reexec
sudo systemctl restart docker

2、本地上传镜像包进行docker load
docker load -i python_3.9-slim.tar.gz
```
PS:Shell脚本构建无需关心镜像问题，自动判断本地应用目录中的基础镜像包。


