---
title: PHP的RSA工具类
date: 2024-08-03 14:59:21
tags: PHP
categories: 后端
top: 22
---

### 前言

目前常见的算法加密有对称加密和非对称加密以及哈希算法。

在日常开发工程中，基本上与第三方对接，大多用RSA非对称加密方式，通过公钥私钥加密解密方式来进行数据的传输。

### 在PHP中使用RSA加密，需要安装openssl扩展

```
<?php

/**
 * RSA工具类
 */
class RSA
{
    /**
     * 签名算法
     */
    const KEY_ALGORITHM       = OPENSSL_KEYTYPE_RSA;
    const SIGNATURE_ALGORITHM = OPENSSL_ALGO_MD5;
    const EN_DE_ALGORITHM     = OPENSSL_PKCS1_PADDING;

    /**
     * 字节数
     */
    const PRIVATE_KEY_BITS = 1024;

    const PUBLIC_TYPE  = 'pub';
    const PRIVATE_TYPE = 'pri';


    /**
     * 私钥加签
     * @param string $dataStr 数据字符串
     * @param string $privateKey 私钥
     * @return string
     */
    public static function sign($dataStr, $privateKey)
    {
        $dataStr = self::str2utf8($dataStr);
        $privateKeyId = openssl_get_privatekey($privateKey);
        openssl_sign($dataStr, $sign, $privateKeyId, self::SIGNATURE_ALGORITHM);
        openssl_free_key($privateKeyId);
        return base64_encode($sign);
    }

    /**
     * 公钥验签
     * @param string $dataStr 加签原数据字符串
     * @param string $publicKey 公钥
     * @param string $sign 签名
     * @return bool
     */
    public static function verifySign($dataStr, $publicKey, $sign)
    {
        $dataStr     = self::str2utf8($dataStr);
        $publicKeyId = openssl_get_publickey($publicKey);
        return (boolean) openssl_verify($dataStr, base64_decode($sign), $publicKeyId, self::SIGNATURE_ALGORITHM);
    }

    /**
     * 公钥加密
     * @param string $dataStr 加签原数据字符串
     * @param string $publicKey 公钥
     * @return string
     */
    public static function encryptByPublicKey($dataStr, $publicKey)
    {
        $dataStr     = self::str2utf8($dataStr);
        $publicKeyId = openssl_get_publickey($publicKey);
        $data        = "";

        $dataArray = str_split($dataStr, self::PRIVATE_KEY_BITS / 8 - 11);
        foreach ($dataArray as $value) {
            openssl_public_encrypt($value,$encryptedTemp, $publicKeyId,self::EN_DE_ALGORITHM);
            $data .= $encryptedTemp;
        }
        openssl_free_key($publicKeyId);
        return base64_encode($data);
    }

    /**
     * 私钥加密
     * @param string $dataStr 加签原数据字符串
     * @param string $privateKey 私钥
     * @return string
     */
    public static function encryptByPrivateKey($dataStr, $privateKey)
    {
        $dataStr      = self::str2utf8($dataStr);
        $privateKeyId = openssl_get_privatekey($privateKey);
        $data         = "";

        $dataArray = str_split($dataStr, self::PRIVATE_KEY_BITS / 8 - 11);
        foreach ($dataArray as $value) {
            openssl_private_encrypt($value,$encryptedTemp, $privateKeyId,self::EN_DE_ALGORITHM);
            $data .= $encryptedTemp;
        }
        openssl_free_key($privateKeyId);
        return base64_encode($data);
    }

    /**
     * 公钥解密
     * @param string $encryptData 加密数据字符串
     * @param string $publicKey 公钥
     * @return string
     */
    public static function decryptByPublicKey($encryptData, $publicKey) {
        $decrypted   = "";
        $decodeStr   = base64_decode($encryptData);
        $publicKeyId = openssl_get_publickey($publicKey);

        $enArray = str_split($decodeStr, self::PRIVATE_KEY_BITS / 8);

        foreach ($enArray as $value) {
            openssl_public_decrypt($value,$decryptedTemp, $publicKeyId,self::EN_DE_ALGORITHM);
            $decrypted .= $decryptedTemp;
        }
        openssl_free_key($publicKeyId);
        return $decrypted;
    }

    /**
     * 私钥解密
     * @param string $encryptData 加密数据字符串
     * @param string $private 私钥
     * @return string
     */
    public static function decryptByPrivateKey($encryptData, $private) {
        $decrypted    = "";
        $decodeStr    = base64_decode($encryptData);
        $privateKeyId = openssl_get_privatekey($private);

        $enArray = str_split($decodeStr, self::PRIVATE_KEY_BITS / 8);

        foreach ($enArray as $value) {
            openssl_private_decrypt($value,$decryptedTemp, $privateKeyId,self::EN_DE_ALGORITHM);
            $decrypted .= $decryptedTemp;
        }
        openssl_free_key($privateKeyId);
        return $decrypted;
    }

    /**
     * 公私钥转为字符串格式
     * @param string $key 公私钥
     * @return string
     */
    public static function key2str($key)
    {
        $key = preg_replace('/-----.*-----/','', $key);
        $key = preg_replace('/[\n\s]/','', $key);
        return is_string($key) ? $key : '';
    }


    /**
     * 将字符串编码转为 utf8
     * @param $str
     * @return string
     */
    private static function str2utf8($str)
    {
        $encode = mb_detect_encoding($str, array('ASCII', 'UTF-8', 'GB2312', 'GBK', 'BIG5'));
        $str = $str ? $str : mb_convert_encoding($str, 'UTF-8', $encode);
        $str = is_string($str) ? $str : '';
        return $str;
    }
}

```

使用：
```
<?php

use RSA;

class Debug{
    
    const PRIVATE_KEY = "-----BEGIN RSA PRIVATE KEY-----
你的私钥内容
-----END RSA PRIVATE KEY-----";
    
    //公钥
    const PUBLIC_KEY = "-----BEGIN PUBLIC KEY-----
你的公钥内容
-----END PUBLIC KEY-----";
    
    public function test()
    {
        $dataStr = "测试";
        $privateKey = self::PRIVATE_KEY;
        $response = RSA::sign($dataStr,$privateKey);
        var_dump($response,"++++++加密结果");
        
        
        $publicKey = self::PUBLIC_KEY;   
        $result = RSA::decryptByPublicKey($response,$publicKey);
        var_dump($result,"++++++解密结果");
    }
    
    
}
```