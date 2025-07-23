---
title: flask上传文件
date: 2025-03-22 15:21:46
tags: Python
categories: Python
top: 25
---


### 一、Flask框架安装
Flask 是一个用 Python 编写的轻量级 Web 应用框架，通过flask写一个简单的上传文件接口。

在开始构建文件上传功能之前，首先需要确保已经安装了 Flask 及其相关依赖。以下是环境搭建的步骤：


> 1. 安装Python
> 2. 安装pip包
> 3. 安装flask
> 4. 创建flask应用
> 5. 处理文件上传逻辑

- 安装flask
```
 pip install Flask
```
- 该命令将通过 pip 包管理器安装 Flask


### 二、处理文件上传逻辑

##### file.py

导入模块包
```
from flask import Blueprint
from utils.common import *
#引入装饰器为上传文件接口设置鉴权
from middleware.middleware import ApiAuthRequired
import os,time,string,random

file_blueprint = Blueprint('file_blueprint', __name__)

#设置文件保存目录
staticFile = 'static/storage/uploads'
```


设置允许上传的文件类型
```
def allowed_file(filename):
    ALLOWED_EXTENSIONS = {'png', 'jpg', 'jpeg', 'gif', 'pdf', 'txt','doc','docx','xls','xlsx'}
    return '.' in filename and filename.rsplit('.', 1)[1].lower() in ALLOWED_EXTENSIONS
```

创建文件保存目录地址
```
def get_upload_folder():
    current_time = time.localtime()
    folder_name = time.strftime("%Y-%m-%d", current_time)

    app_root = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))

    BASE_UPLOAD_FOLDER = os.path.join(app_root, staticFile)
    # 创建日期文件夹
    upload_folder = os.path.join(BASE_UPLOAD_FOLDER, folder_name)
    if not os.path.exists(upload_folder):
        os.makedirs(upload_folder)
    return upload_folder
```


文件上传
```
@file_blueprint.route('/upload', methods=['POST'])
@ApiAuthRequired
def upload():

    if 'file' not in request.files:
        return response({},400,'error no file part')
    
    file = request.files['file']
    
    if file.filename == '':
        return response({},400,'No selected file')
    
    if file and allowed_file(file.filename):
        upload_folder = get_upload_folder()
        random_string = ''.join(random.choices(string.ascii_letters + string.digits, k=12))
        filename = os.path.join(upload_folder, random_string+file.filename)
        file.save(filename)
        
        file_url = os.path.relpath(filename, start=staticFile)

        return response({"filePath": '/'+staticFile+'/'+file_url})
    else:
        return response({},400,'不允许上传的文件类型')
```


装饰器鉴权如下：
```
import json
from flask import request
from functools import wraps
def ApiAuthRequired(f):
    @wraps(f)
    def wrapper(*args, **kwargs):
        if not request.headers.get('token'):
            data = {
                "code": 401,
                "msg": "token不存在，未登录"
            }
            json_data = json.dumps(data)
            return json_data,401
        return f(*args, **kwargs)
    return wrapper


```


公共返回函数：
`from utils.common import *`
```
def response(data = {},code = 1,msg="success"):
    return jsonify({
        "code":code,
        "msg":msg,
        "data":data
    })
```

### 三、最终实现效果如下

![alt text](/images/file/flask_upload.jpg)