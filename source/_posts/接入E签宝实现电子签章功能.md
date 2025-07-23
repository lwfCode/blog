---
title: E签宝实现电子签章功能
date: 2025-06-25 10:45:53
tags: PHP
categories: 后端
top: 25
---

### 前言

在互联网高速发展的今天，个人电子签章正以每年35%的增速渗透到大众生活中。从线上合同签署到平台认证，这种新型身份标识技术让原本需要线下奔波的手续变得轻松便捷。据统计，2023年已有超过50%的电子签名用户选择线上平台办理手续，其中85%的使用者反馈节省了至少3天的办理时间。现在创业者注册公司、个体户签署供货协议，都能通过可靠的线上平台完成全套流程。


#### 电子签章的必备条件

- 注册E签宝平台并进行实名认证
- 创建沙箱应用环境
- 开发测试
- 创建正式应用
- 订购套餐
- 应用正式上线
- 线上运维

##### 流程图

![alt text](/images/engin/duijie.jpg)

E签宝官网地址：https://open.esign.cn


#### 创建沙箱应用

![alt text](/images/engin/shaxiang.jpg)

#### 配置应用

添加webhook（回调地址），e签宝所有通知数据均推送此地址，同时记录应用ID与应用Secret

![alt text](/images/engin/12.jpg)

#### 开发对接

对接e签名开发流程大致如下：

```
flowchart LR
    上传本地合同到e签宝 --> 获取文件状态 --> 获取拖章签名页面URL/根据关键字搜索定位签章坐标 --> 开启签章流程 --> 获取完成签章文件url
    
```

##### 步骤一：上传本地合同到e签宝

代码片段如下：

```
public function createByFile($params,$contentType = "application/octet-stream")
{
    $filePath = $params['filePath'];
    $save = 'storage/ensign';
    $parsedUrl = parse_url($filePath);
    $path = $parsedUrl['path'] ?? '';

    $extension = pathinfo($path, PATHINFO_EXTENSION);
    $fileName = "电子签章盖章合同.".$extension;
    $response = getImage($filePath,$save,$fileName);
    if (isset($response,$response['save_path'])) {
        $filePath = public_path() . $response['save_path'];
    }else{
        throw new \Exception("文件上传失败!");
    }
    $postUrl = "/v3/files/file-upload-url";
    $postAllUrl = self::API . $postUrl;
    $fileName = basename($filePath);
    $md5 = EsignHttpCfgHelper::getContentBase64Md5($filePath);

    $reqBodyObj = [
        "contentMd5" => $md5,
        "contentType" => $contentType,
        "fileName" => $fileName,
        "fileSize" => filesize($filePath)
    ];
    $reqBodyData = json_encode($reqBodyObj, JSON_UNESCAPED_UNICODE);
    $contentMD5 = EsignHttpCfgHelper::doContentMD5($reqBodyData);
    $reqSignature = $this->getSign($contentMD5,$postUrl);
    $timeStamp = round(microtime(true) * 1000);
    $accept = "*/*";
    $contentTypeHeader = "application/json";
    $headersArr = [
        "X-Tsign-Open-App-Id: ".self::APP_ID,
        "X-Tsign-Open-Auth-Mode: Signature",
        "X-Tsign-Open-Ca-Timestamp: $timeStamp",
        "Accept: $accept",
        "Content-Type: $contentTypeHeader",
        "X-Tsign-Open-Ca-Signature: $reqSignature",
        "Content-MD5: $contentMD5"
    ];

    $result = EsignHttpCfgHelper::sendPost($postAllUrl, $reqBodyData, $headersArr);
    $result = json_decode($result,true);
    if (isset($result['data']['fileUploadUrl'])){
        $binaryData = file_get_contents($filePath);
        EsignHttpCfgHelper::upLoadFileHttp($result['data']['fileUploadUrl'],$md5,$binaryData,$contentType);
    }else{
        //记录失败日志
    }
    return $result;
}



public function getSign($contentMD5,$postUrl,$method = "POST",$contentType = "application/json")
{
    $accept = "*/*";
    $date = "";
    $headers = "";

    $plaintext = $method . "\n" .
        $accept . "\n" .
        $contentMD5 . "\n" .
        $contentType . "\n" .
        $date . "\n";

    if ($headers === "") {
        $plaintext .= $postUrl;
    } else {
        $plaintext .= $headers . "\n" . $postUrl;
    }

    $reqSignature = EsignHttpCfgHelper::doSignatureBase64($plaintext, self::APP_SECRET);
    return $reqSignature;
}

```


##### 步骤二：获取文件状态

在操作e签宝文件之前必须要查询文件是否已被平台处理过

代码片段如下：

```
public function getFileStatus($params)
{
    $app['app_key'] = self::APP_KEY;
    $app['sign'] = self::SIGN;
    $data = WorkSheet::getRowById($params['rowid'],$app,self::WORK_SHEET,$this->host);

    $data = $data['data'] ?? [];
    if (empty($data)) throw new \Exception("本条记录数据不存在!");

    $fileId = $data['fileId'];
    $api = self::API."/v3/files/".$fileId;
    $timeStamp = round(microtime(true) * 1000);
    $accept = "*/*";
    $reqSignature = $this->getSign("","/v3/files/".$fileId,"GET","");
    $headersArr = [
        "X-Tsign-Open-App-Id: ".self::APP_ID,
        "X-Tsign-Open-Auth-Mode: Signature",
        "X-Tsign-Open-Ca-Timestamp: $timeStamp",
        "Accept: $accept",
        "X-Tsign-Open-Ca-Signature: $reqSignature"
    ];
    $result = EsignHttpCfgHelper::sendGet($api,$headersArr);
    $result = json_decode($result,true);

    return $result;
}

示例：
/**
 * 注：fileStatus：2 或 5 代表可用
 * {
 * "code": 0,
 * "message": "成功",
 * "data": {
 * "fileId": "********",
 * "fileName": "测试.pdf",
 * "fileSize": null,
 * "fileStatus": 2,
 * "fileDownloadUrl": "https://esignoss.esign.cn/11182/****.pdf?Expires=1749694469&OSSAccessKeyId=****",
 * "fileTotalPageCount": 26,
 * "pageWidth": null,
 * "pageHeight": null
 * }
 * }
 */
```

##### 步骤三：获取拖章页面URL

这一步骤为E签宝推送数据到webhook的回调地址，根据 “action” 字段为“GET_SEAL_POSITION” 则为拖章回调内容
如：
```
{
    "action": "GET_SEAL_POSITION",
    "components": [
        {
            "componentPosition": {
                "componentPageNum": 1,
                "componentPositionX": 478.24,
                "componentPositionY": 695.18
            },
            "componentSize": {
                "componentHeight": 100,
                "componentWidth": 100
            },
            "componentType": 6,
            "fileId": "c7fb4fbda4304be9a5bc80264c10adc6",
            "normalSignField": {
                "sealSpecs": 1,
                "showSignDate": 0,
                "signFieldStyle": 1
            },
            "signerRole": ""
        },
        {
            "componentPosition": {
                "componentPageNum": 1,
                "componentPositionX": 534.53,
                "componentPositionY": 719.21
            },
            "componentSize": {
                "componentHeight": 118,
                "componentWidth": 74
            },
            "componentType": 6,
            "fileId": "c7fb4fbda4304be9a5bc80264c10adc6",
            "normalSignField": {
                "showSignDate": 0,
                "signFieldStyle": 2
            },
            "signerRole": ""
        }
    ],
    "customBizNum": "1cdfca0d-fe1f-44ee-216c-dd4e0002a4ce",
    "timestamp": 1750753533975
}
```

代码片段如下：

```
public function selPosition($components,$customBizNum)
{
    //骑章定位（可选）
    $arr = [
        [
            "normalSignFieldConfig" => [
                "signFieldStyle" => 2,
                'autoSign' => true,
                "signFieldPosition" => [
                    "acrossPageMode" => "ALL",
                    "positionY" => 300
                ]
            ]
        ]
    ];
    $filedArr = [];

    foreach ($components as $item){
        $signFiled = $item['normalSignField']['signFieldStyle'] ?? '';
        $fileId = $item['fileId'];
        /** 骑章 */
        if ($signFiled === 2){
          
        }else{
            $tmp = [
                "fileId" => $fileId,
                "customBizNum" => $customBizNum,
                'normalSignFieldConfig' => [
                    'autoSign' => true,
                    "signFieldStyle" => 1,
                    "signFieldPosition" => [
                        "positionPage" => $item['componentPosition']['componentPageNum'],
                        "positionX" => $item['componentPosition']['componentPositionX'],
                        "positionY" => $item['componentPosition']['componentPositionY'],
                    ]
                ]
            ];
            array_push($filedArr,$tmp);
        }
    }

    //记录日志
    
    //拖章完成后，开启签章流程
    return $this->startSignFile([
        'fileId' => $fileId,
        "signFiledsArr" => $filedArr
    ]);
}
```

##### 步骤四：开启签章

```
public function startSignFile($params)
{
    $fileId = $params['fileId'];
    $postUrl = "/v3/sign-flow/create-by-file";
    $postAllUrl = self::API."/v3/sign-flow/create-by-file";
    $reqBodyObj = [
        "docs" => [
            ['fileId' => $fileId]
        ],
        'signFlowConfig' => [
            "signFlowTitle" => '电子签章',
            "autoFinish" => true,
            "notifyUrl" => "你的接口地址/v1/skws/esign/callback"
        ],
        'signers' => [
            [
                'signerType' => 1,
                'signFields' => $params['signFiledsArr']
            ],
        ]
    ];

    $reqBodyData = json_encode($reqBodyObj, JSON_UNESCAPED_UNICODE);
    $contentMD5 = EsignHttpCfgHelper::doContentMD5($reqBodyData);
    $reqSignature = $this->getSign($contentMD5,$postUrl);
    $headersArr = self::getHeader($reqSignature,$contentMD5);
    $result = EsignHttpCfgHelper::sendPost($postAllUrl, $reqBodyData, $headersArr);
    $result = json_decode($result,true);

    return $result;
}
```

##### 步骤五：下载完成签章文件URL

开启签章流程后，e签宝异步处理，对已完成签章的文件通过webhook方式推送至回调地址

代码片段如下：

```
public function downloadFile($params)
{
    $signFlowId = $params['signFlowId'];
    $api = self::API."/v3/sign-flow/{$signFlowId}/file-download-url";
    $timeStamp = round(microtime(true) * 1000);
    $accept = "*/*";
    $reqSignature = $this->getSign("","/v3/sign-flow/{$signFlowId}/file-download-url","GET","");
    $headersArr = [
        "X-Tsign-Open-App-Id: ".self::APP_ID,
        "X-Tsign-Open-Auth-Mode: Signature",
        "X-Tsign-Open-Ca-Timestamp: $timeStamp",
        "Accept: $accept",
        "X-Tsign-Open-Ca-Signature: $reqSignature"
    ];
    $result = EsignHttpCfgHelper::sendGet($api,$headersArr);
    $result = json_decode($result,true);

    return $result;
}
```

##### 其他代码片段：

一、回调函数：

```
public function callback($params)
{
    //记录日志
    writeLog(json_encode($params),"Esign_callback","Esign_callback");
    $action = $params['action'] ?? '';
    switch ($action){
        /** 拖章定位*/
        case "GET_SEAL_POSITION":
            $components = $params['components']  ?? [];
            if (!empty($components)){
               $this->selPosition($components,$params['customBizNum']);
            }
            break;
        /** 完成签章 */
        case "SIGN_FLOW_COMPLETE":
            $signFlowId = $params['signFlowId']  ?? '';
            $signFlowStatus = $params['signFlowStatus'] ?? '';
            if (!empty($signFlowId) && !empty($signFlowStatus) && $signFlowStatus == "2"){
                $this->downloadFile(['signFlowId' => $signFlowId]);
            }
            break;
    }
    return ['code' => 200,'msg' => 'success'];
}
```

二、工具类


```
<?php
namespace  app\service\esign\tools;

class EsignHttpCfgHelper
{
    public static $connectTimeout = 15;//15 second
    public static $readTimeout = 15;//15 second
    public static $uploadReadTimeout = 60;
    public static $uploadConnectTimeout = 60;

    public static function doContentMD5($data)
    {
        return base64_encode(md5($data, true));
    }

    public static function getContentBase64Md5($filePath)
    {
        $md5file = md5_file($filePath,true);
        $contentBase64Md5 = base64_encode($md5file);
        return $contentBase64Md5;
    }

    public static function doSignatureBase64($plaintext, $secret)
    {
        return base64_encode(hash_hmac('sha256', $plaintext, $secret, true));
    }

    public static function sendGet($url, $headers)
    {
        $ch = curl_init($url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_HTTPGET, true);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);

        $response = curl_exec($ch);
        if (curl_errno($ch)) {
            echo 'CURL错误: ' . curl_error($ch);
        }
        curl_close($ch);

        return $response;
    }


    public static function sendPost($url, $postData, $headers)
    {
        $ch = curl_init($url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_POST, true);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $postData);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);

        $response = curl_exec($ch);
        if (curl_errno($ch)) {
            echo 'CURL错误: ' . curl_error($ch);
        }
        curl_close($ch);

        return $response;
    }

    public static function upLoadFileHttp($uploadUrls, $contentMd5, $fileContent,$ContenType){
        $header = array(
            'Content-Type:'.$ContenType,
            'Content-Md5:' . $contentMd5
        );

        $curl_handle = curl_init();
        curl_setopt($curl_handle, CURLOPT_URL, $uploadUrls);
        curl_setopt($curl_handle, CURLOPT_FILETIME, true);
        curl_setopt($curl_handle, CURLOPT_FRESH_CONNECT, false);
        // curl_setopt($curl_handle, CURLOPT_HEADER, true); // 输出HTTP头 true
        curl_setopt($curl_handle, CURLOPT_RETURNTRANSFER, true);

        if (self::$uploadReadTimeout) {
            curl_setopt($curl_handle, CURLOPT_TIMEOUT, self::$uploadReadTimeout);
        }
        if (self::$uploadConnectTimeout) {
            curl_setopt($curl_handle, CURLOPT_CONNECTTIMEOUT, self::$uploadConnectTimeout);
        }

        curl_setopt($curl_handle, CURLOPT_SSL_VERIFYPEER, false);
        curl_setopt($curl_handle, CURLOPT_SSL_VERIFYHOST, false);

        curl_setopt($curl_handle, CURLOPT_HTTPHEADER, $header);
        curl_setopt($curl_handle, CURLOPT_CUSTOMREQUEST, 'PUT');

        curl_setopt($curl_handle, CURLOPT_POSTFIELDS, $fileContent);
        if (is_array($header) && 0 < count($header)) {
            curl_setopt($curl_handle, CURLOPT_HTTPHEADER, $header);
        }
        $curlRes = curl_exec($curl_handle);
        #$httpCode = curl_getinfo($curl_handle, CURLINFO_HTTP_CODE);
        curl_close($curl_handle);
        return $curlRes;
    }

    public static function sendPut($url, $contentMd5, $binaryData, $contentType = 'application/octet-stream')
    {
        $headers = [
            "Content-MD5: $contentMd5",
            "Content-Type: $contentType"
        ];

        $ch = curl_init($url);
        curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
        curl_setopt($ch, CURLOPT_CUSTOMREQUEST, "PUT");
        curl_setopt($ch, CURLOPT_POSTFIELDS, $binaryData);
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
        curl_setopt($ch, CURLOPT_TIMEOUT, 30);

        $response = curl_exec($ch);
        if (curl_errno($ch)) {
            echo 'CURL错误: ' . curl_error($ch);
        }
        curl_close($ch);

        return $response;
    }

}

```

类常量

```
const APP_ID = "xxx";
const APP_SECRET = "xxxx";
const API = "https://smlopenapi.esign.cn";
```

#### 最终效果如下：

##### 待签章文件

![alt text](/images/engin/1.jpg)

##### 拖章页面

![alt text](/images/engin/555.jpg)

##### 已完成签章文件

![alt text](/images/engin/2.jpg)