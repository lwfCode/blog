---
title: Docker环境下配置土拨鼠喇叭喊话
date: 2025-01-06 15:39:14
tags: Docker
categories: Docker
top: 24
---
### 前言

鉴于项目中多次使用喇叭配合摄像头喊话需求，特此记录。

###### 项目同款喇叭商品购买链接（小播鼠网络广播音箱）

```
20WIP音柱（网线）
https://item.taobao.com/item.htm?_u=4agrpjt81e4&id=573575299818&skuId=4759108821855&spm=a1z09.2.0.0.70a62e8dTjtzMh
```

#### 一、必备条件

```
1、云服务器一台
2、公网ip
3、公司/个人具备二次开发能力
```

#### 二、启动喇叭服务

#### （一）docker方式（推荐）

#### （二）宿主机安装需要满足以下安装包要求


```
glibc-2.17-106.el7_2.4.x86_64
libstdc++-4.8.5-16.el7_4.2.x86_64
glibc-2.17-106.el7_2.4.x86_64
libgcc-4.8.5-16.el7_4.2.x86_64
glibc-2.17-106.el7_2.4.x86_64
zlib-1.2.7-17.el7.x86_64
openssl-libs-1.0.2k-8.el7.x86_64
systemd-libs-219-42.el7_4.4.x86_64
libicu-50.1.2-15.el7.x86_64
pcre2-utf16-10.23-2.el7.x86_64
glibc-2.17-106.el7_2.4.x86_64
glib2-2.54.2-2.el7.x86_64
krb5-libs-1.15.1-8.el7.x86_64
libcom_err-1.42.9-10.el7.x86_64
krb5-libs-1.15.1-8.el7.x86_64
libcap-2.22-8.el7.x86_64
glibc-2.17-106.el7_2.4.x86_64
libselinux-2.5-11.el7.x86_64
xz-libs-5.1.2-12alpha.el7.x86_64
libgcrypt-1.5.3-12.el7_1.1.x86_64
libgpg-error-1.12-3.el7.x86_64
glibc-2.17-106.el7_2.4.x86_64
elfutils-libs-0.163-3.el7.x86_64
pcre-8.32-17.el7.x86_64
krb5-libs-1.15.1-8.el7.x86_64
keyutils-libs-1.5.8-3.el7.x86_64
libattr-2.4.46-12.el7.x86_64
elfutils-libelf-0.163-3.el7.x86_64
bzip2-libs-1.0.6-13.el7.x86_64
```


###### 2.1 启动docker

```
docker pull centos:7.9.2009

docker run -it -d --name="speaker" --net=host centos:7.9.2009
```

###### 2.2 拷贝服务到容器内

media文件下载链接: https://pan.baidu.com/s/1WGqBbDCzxOXLP8FGw9BXkg 提取码: pxxi
下载地址：

Adore文件下载链接: https://pan.baidu.com/s/1UfNnEu4LigiytepiY0p-aA 提取码: w5b2

windows系统下调试工具（喇叭配置助手）：
链接: https://pan.baidu.com/s/1ZJoacqdF60RmBz_V4aooXQ 提取码: awuw
```
docker cp Adore.tar.gz speaker:/data
docker cp media.tar.gz speaker:/data

docker exec -it speaker bash

cd /data

tar xvf Adore.tar.gz
tar xvf media.tar.gz
```

###### 2.3 修改media服务的config.json文件

- 修改LocalIP 地址
- 扫码获取并且修改SEED_List 以及 SQR_List
- ASKEY随便生成一个32位长度的字符串
- 保证“ 23022、3333、3000-3002、6000-6008 、6081-6082 "端口不被占用

客户端现场配置如下：

```
媒体服务IP地址如：101.91.127.12
媒体服务端口号：3333
设备二维码：是点击扫码配置自动获取的/ 也可以通过 微信扫码SEED 码扫出来
设备ip地址：是你在现场跟摄像头同一个内网 如 192.168 的地址 （ip地址无冲突）
默认网关：你的现场ip地址一致的网关地址
设备dns：114.114.114.114
开启DHCP：不要打勾
开启语音调试：调试打开，调试完毕后建议关闭，防止一直在提示媒体服务器连接成功。


PS: 喊话功能 需要 扫设备上的2个码，一个SEED码，一个SQR码，需要注册到云服务器上才能连通喇叭服务，现场网络需要能连上互联网，否则无法使用喊话功能。
```
如图所示：
![alt text](/images/speak/1.jpg)

扫码获取SEED码跟SQR码
如图所示：
![alt text](/images/speak/2.jpg)

config.json配置
```
{
    "AryLen": 20,
    "Check_": 3000,
    "Check_len": 10,
    "DPort": 3333,
    "DataPort": 3002,
    "EnableRedis": false,
    "LocalIP": "101.91.127.12",
    "ManagePort": 6007,
    "Netcross_Port0": 6000,
    "Netcross_Port1": 6001,
    "OPort0": 6000,
    "OPort1": 6001,
    "DataPortBack2": 6081,
    "DataPortBack3": 6082,
    "Pastcmd": 6008,
    "BjPort": 6004,
    "PackSize": 2048,
    "RedisIP": "127.0.0.1",
    "RedisPort": 6379,
    "PS1":6003,
    "PS2":6006,
    "ASKEY":"21df2afe8ea0fbce546aec82134149ck",
    "SEED_List": [      
        "6b7219ff05f644890090bf96839010a8",
    ],
    "SQR_List": [ 
        "cfe8faad4f49de4b40aa9399d853ae86",
    ],
    "TSWait": 3000
}

```

###### 2.4 启动服务
```
#启动/关闭/重启等命令
./control start|restart|stop|status|tail

#启动
./control start
查看启动log
./control tail
```

##### 看到如下图则表示连接喇叭到服务器成功
如图所示：
![alt text](/images/speak/3.jpg)


#### 三、启动websocket服务


```
cd /data/Adore
vi config.ini

#修改wbport端口号为23202，Meida_port为3002
[Settings]
wbport=23202
Meida_port=3002
Meida_ip=127.0.0.1
askey=21df2afe8ea0fbce546aec8198237b

#启动
./control start
```

通过nginx反向代理websocket服务


```
location ^~/horn {
 proxy_pass http://101.91.127.12:23202;
 proxy_http_version 1.1;
 proxy_set_header Upgrade $http_upgrade;
 proxy_set_header Connection "upgrade";
}

```

测试连接是否正常

![alt text](/images/speak/4.jpg)


#### 四、附上前端代码片段

##### 父组件 index.vue

```

<div class="btns" style="top:110px" v-if="camera_data.horn_seed">
  <div class="btn" :class="lbActive == 3 ? 'active' : ''" @click="lbActive = 3">
    点击喊话
  </div>
  <div class="btn" :class="lbActive == 4 ? 'active' : ''" @click="lbActive = 4">
    停止喊话
  </div>
  <CallHorn :lbActive="lbActive" :camera_data="camera_data"></CallHorn>
</div>
```

##### 子组件 callHorn.vue


```
<template>
<div></div>
</template>

<script>
import MP3Recorder from "@/utils/horn/recordmp3.js";
export default {
props: ["camera_data","lbActive"],
data() {
    return {
        ws:null,
        recorder:null,
        url:"wss://api.ai-safer.cn/horn",
        horn_seed:''
    };
},
watch: {
    camera_data(nval, oval) {
       
    },
    lbActive(nval,oval){
        if(nval == 3){
            this.funOntime()
        }else{
            this.funOnStop()
        }
    }
},
created() {},
mounted() {
  this.init();
},
methods: {
    init() {
        var that = this
        var recorder = new MP3Recorder({  
            debug:true,  
            funOk: function () {  
                console.log("funOk初始化成功...")
            },  
            funCancel: function (msg) {  
                console.log(msg,"funCancel...")
                that.recorder = null;  
            }  
        },that); 
        this.recorder = recorder
    },
    funOntime(){
        console.log('开启广播...')
        this.ws = new WebSocket(this.url);
        setTimeout(() => {
            this.startCall()
        },1000)
    },
    startCall(){
        console.log(this.camera_data.horn_seed,'=======喇叭序列号')
        var info =  {
            "Meport": 6002,  //跨服端口号
            "Umagic": 89686, //快服句柄随机数
            "Umask": "84d3def493da487b96ed12744ad44c7a", //快服填随机字符串
            "plevel": 9,//该用户内播放等级1~9 9是最高优先级
            "sskey": "496beaa0dfa8d67262f364ce49fdcf78",
            "ulevel": 600,//用户间等级 1~600   不同用户间同时推流一个音响 等级高的会切掉等级低的
            "snlist": [this.camera_data.horn_seed], //被播放的设备序列号
            "cmd": "PLAYLIST",//固定值
            "Meip": "127.0.0.1" // 您的节点服务器IP地址 
    }
    let length = JSON.stringify(info).length
    let data = length+"\n{\"Meport\":6002,\"Umagic\":89686,\"Umask\":\"84d3def493da487b96ed12744ad44c7a\",\"plevel\":9,\"sskey\":\"496beaa0dfa8d67262f364ce49fdcf78\",\"ulevel\":999,\"snlist\":[\""+this.camera_data.horn_seed+"\"],\"cmd\":\"PLAYLIST\",\"Meip\":\"127.0.0.1\"}";
    this.ws.send(data);//这里 发送的是 信息头;
        
    this.recorder.Ontime(2);
    },
    funOnStop(){
        console.log('关闭广播...')
        this.recorder.stop(); 
        this.ws.close();
    }
},
};
</script>

<style lang="scss"></style>
  
```

附上recordmp3.js下载链接: https://pan.baidu.com/s/1WcGns_pDkoFGGFko1BumNw 提取码: nhwj