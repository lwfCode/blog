---
title: 海康api接口鉴权示例
date: 2025-02-21 11:21:45
tags: PHP
categories: 后端
top: 8
---

#### 一、设备接入

```
1. 打开设备，打开网络设置，连接wifi网络。
2. 开启ISUP协议，配置相关信息。
3. 打开国标，配置国标协议信息。
4. 保存设置并退出，设备将按照新设置进行工作。
```

#### 二、API鉴权

```
#接口加密鉴权
public static function makeSignature(string $str) {
    $sk = "you secret";
    $sign = hash_hmac('sha256', $str, $sk, true);
    return base64_encode($sign);
}

public static function makeSignStr(string $url, array $header = []) {
    $str = "POST\n"
        . $header['Accept'] . "\n";
    !empty($header['Content-MD5']) && $str .= $header['Content-MD5'] . "\n";
    $str .= $header['Content-Type'] . "\n"
        . "x-ca-key:{$header['X-Ca-Key']}\n"
        . "x-ca-timestamp:{$header['X-Ca-Timestamp']}\n";
    $str .= $url;
    return $str;
}

public static function signature(string $url, array $header = []) {
    $str = self::makeSignStr($url, $header);
    return self::makeSignature($str);
}

public static function getHeaders(array $headers){
    $data = [];
    foreach($headers as $k => $v) {
        $data[] = $k.":".$v;
    }
    return $data;
}
```

#### 二、API请求

```
public function requestApi()
{
    $appKey = "you appkey";
    $headers = [
        "Accept" => "*/*",
        "Content-Type" => "application/json",
        "X-Ca-Key" => $appKey,
        "X-Ca-Signature" => "",
        "X-Ca-Signature-Headers" => "x-ca-key,x-ca-timestamp",
        "X-Ca-Timestamp" => time()."000"
    ];
    $url = '/artemis/api/brecorder/v3/deviceApiService/device/control';
    $headers['X-Ca-Signature'] = self::signature($url, $headers);
    $headers = self::getHeaders($headers);
    $data = [
        "batchDeviceIds" => ['设备编号'],
        "command" => 2
    ];
    $apiUrl = "https://192.167.1.10:4443/artemis/api/brecorder/v3/deviceApiService/device/control";
    list($body, $header) = HttpCurl::request($apiUrl, "post", json_encode($data), $headers);

    return json(json_decode($body,true));
}
```

返回最终结果如下：

```
{
    "code": "0",
    "msg": "SUCCESS",
    "data": null
}
```

