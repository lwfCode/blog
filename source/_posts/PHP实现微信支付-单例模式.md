---
title: PHP实现微信支付(单例模式)
date: 2023-03-28 11:20:05
tags: PHP
categories: 后端
top: 13
---
### 前言

随着移动支付的发展，微信支付逐渐成为了人们日常支付的一种主要方式。作为一个PHP开发者，必须要在项目中集成微信支付

主要思路：
1、获取微信支付用户授权(需要申请appid以及appsecret，设置授权回调页面和商户号)
2、生成支付订单
3、统一下单
4、支付成功回调
5、处理支付结果

###### 微信支付涉及金额操作，从发起支付生成订单到最终的回调，必须要经过严格的签名验证和参数校验等处理逻辑。
---

#### 一、请求参数

1.1准备工作

```
#安装curl请求类
composer require gaoming13/http-curl
```

```
// 商户配置
protected $option = [
    'appid' => '',
    'key' => '',
    'mch_id' => ""
];
//订单参数
$postArr = [
        'appid' => $this->option['appid'], //appid
        'body' => $reason, //商品说明  
        'mch_id' => $this->option['mch_id'], //商户号
        'nonce_str' => md5($trade_no), //
        'notify_url' => $callback_url, //回调地址
        'openid' => $openid, //付款用户openid   
        'out_trade_no' => $trade_no,
        'spbill_create_ip' => $this->getClientIp(), //客户端ip
        'total_fee' => $money, //支付金额
        'trade_type' => 'JSAPI', //支付类型
        'spbill_create_ip' => $this->getClientIp() //客户端ip
    ];
```

#### 二、签名

```
public function sign($arr)
{
    if(!empty($arr['sign'])) unset($arr['sign']);

    ksort($arr);
    $query = urldecode(http_build_query($arr)) . '&key=' . $this->option['key'];
    return strtoupper(md5($query));
}
```

#### 三、数组转XML

```
/**
 * 制作XML
 *
 * @param array $data
 * @return string
*/
protected function buildXML(array $data) : string
{
    $result = '<xml>';
    foreach ($data as $key => $value) {
        $result = $result . "\r\n" . '<' . $key . '>' . $value . '</' . $key . '>';
    }
    return $result . '</xml>';
}
```

#### 四、统一下单

```
/**
 * 创建微信订单|支持H5，微信小程序
 *
 * @param [type] $trade_no
 * @param [type] $reason
 * @param [type] $openid
 * @param [type] $money
 * @param [type] $callback_url
 * @return void
*/
public function CreateWeChatOrder($trade_no, $reason,  $money, $openid, $callback_url = null)
{
    $postArr = [
        'appid' => $this->option['appid'],
        'body' => $reason,
        'mch_id' => $this->option['mch_id'],
        'nonce_str' => md5($trade_no),
        'notify_url' => $callback_url,
        'openid' => $openid,
        'out_trade_no' => $trade_no,
        'spbill_create_ip' => $this->getClientIp(),
        'total_fee' => $money,
        'trade_type' => 'JSAPI',
        'spbill_create_ip' => $this->getClientIp()
    ];

    // 签名
    $postArr['sign'] = $this->sign($postArr);

    // 生成xml
    $postData = $this->buildXML($postArr);

    // 发送数据并解析
    list($body) = HttpCurl::request('https://api.mch.weixin.qq.com/pay/unifiedorder', 'POST', $postData);

    $xmlString = simplexml_load_string($body, 'SimpleXMLElement', LIBXML_NOCDATA);
    $response = json_decode(json_encode($xmlString), true);


    if ($response['return_code'] == 'SUCCESS') {
        $data = [
            'appId' => $this->option['appid'],
            'timeStamp' => strval(time()),
            'nonceStr' => md5($trade_no),
            'package' => 'prepay_id=' . $response['prepay_id'],
            'signType' => 'MD5'
        ];
        ksort($data);
        $dataQurey = urldecode(http_build_query($data)) . '&key=' . $this->option['key'];
        $data['paySign'] = strtoupper(md5($dataQurey));

        return $data;
    } else {
        return false;
    }
}
```
最终伪代码如下：

```
<?php
namespace lwfdeveloper\wxpay;

use Gaoming13\HttpCurl\HttpCurl;

class WeChatPay
{
    
    //保存例实例在此属性中
    private static $_instance;

    // 商户配置
    protected $option = [
        'appid' => '',
        'key' => '',
        'mch_id' => ""
    ];


    public function __construct($appid = null, $key = null ,$mch_id = null)
    {
        if (empty($appid) || empty($key) || empty($mch_id)) {
            return false;
        }
        $this->option['appid'] = $appid;
        $this->option['key'] = $key;
        $this->option['mch_id'] = $mch_id;
    }


    /**
     * 静态方法，单例统一访问入口
     * @param $options
     * @return RsaCrypt
     */
    public static function getInstance($options = [])
    {
        if (is_null ( self::$_instance ) || isset ( self::$_instance )) {
            self::$_instance = new self ($options['appid'],$options['key'],$options['mch_id']);
        }
        return self::$_instance;
    }


    /**
     * 创建微信订单|支持H5，微信小程序
     *
     * @param [type] $trade_no
     * @param [type] $reason
     * @param [type] $openid
     * @param [type] $money
     * @param [type] $callback_url
     * @return void
     */
    public function CreateWeChatOrder($trade_no, $reason,  $money, $openid, $callback_url = null)
    {
        $postArr = [
            'appid' => $this->option['appid'],
            'body' => $reason,
            'mch_id' => $this->option['mch_id'],
            'nonce_str' => md5($trade_no),
            'notify_url' => $callback_url,
            'openid' => $openid,
            'out_trade_no' => $trade_no,
            'spbill_create_ip' => $this->getClientIp(),
            'total_fee' => $money,
            'trade_type' => 'JSAPI',
            'spbill_create_ip' => $this->getClientIp()
        ];

        // 签名
        $postArr['sign'] = $this->sign($postArr);

        // 生成xml
        $postData = $this->buildXML($postArr);

        // 发送数据并解析
        list($body) = HttpCurl::request('https://api.mch.weixin.qq.com/pay/unifiedorder', 'POST', $postData);

        $xmlString = simplexml_load_string($body, 'SimpleXMLElement', LIBXML_NOCDATA);
        $response = json_decode(json_encode($xmlString), true);


        if ($response['return_code'] == 'SUCCESS') {
            $data = [
                'appId' => $this->option['appid'],
                'timeStamp' => strval(time()),
                'nonceStr' => md5($trade_no),
                'package' => 'prepay_id=' . $response['prepay_id'],
                'signType' => 'MD5'
            ];
            ksort($data);
            $dataQurey = urldecode(http_build_query($data)) . '&key=' . $this->option['key'];
            $data['paySign'] = strtoupper(md5($dataQurey));

            return $data;
        } else {
            return false;
        }
    }


    /**
     * 创建App微信订单
     *
     * @param [type] $trade_no
     * @param [type] $reason
     * @param [type] $money
     * @param [type] $callback_url
     * @return void
     */
    public function CreateWeChatAppOrder($trade_no, $reason, $money, $callback_url = null)
    {
        $postArr = [
            'appid' => $this->option['appid'],
            'body' => $reason,
            'mch_id' => $this->option['mch_id'],
            'nonce_str' => md5($trade_no),
            'notify_url' => $callback_url,
            'out_trade_no' => $trade_no,
            'spbill_create_ip' => $this->getClientIp(),
            'total_fee' => $money,
            'trade_type' => 'APP'
        ];

        // 签名
        $postArr['sign'] = $this->sign($postArr);

        // 生成xml
        $postData = $this->buildXML($postArr);

        // 发送数据并解析
        list($body) = HttpCurl::request('https://api.mch.weixin.qq.com/pay/unifiedorder', 'POST', $postData);

        $xmlString = simplexml_load_string($body, 'SimpleXMLElement', LIBXML_NOCDATA);
        $response = json_decode(json_encode($xmlString), true);

        if ($response['return_code'] == 'SUCCESS') {
            // 接收微信返回的数据，传给APP！
            $data = [
                'appid' => $this->option['appid'],
                'timestamp' => time(),
                'noncestr' => md5($trade_no),
                'package' => 'Sign=WXPay',
                'partnerid' => $this->option['mch_id'],
                'prepayid' => $response['prepay_id']
            ];
            ksort($data);
            $dataQurey = urldecode(http_build_query($data)) . '&key=' . $this->option['key'];
            $data['sign'] = strtoupper(md5($dataQurey));
            $data['trade_no'] = $trade_no;

            return $data;
        } else {
            return false;
            // return '订单创建失败：' . $response['return_msg'];
        }
    }


    /**
     * 制作XML
     *
     * @param array $data
     * @return string
     */
    protected function buildXML(array $data) : string
    {
        $result = '<xml>';
        foreach ($data as $key => $value) {
            $result = $result . "\r\n" . '<' . $key . '>' . $value . '</' . $key . '>';
        }
        return $result . '</xml>';
    }


    /**
     * 签名
     */
    public function sign($arr)
    {
        if(!empty($arr['sign'])) unset($arr['sign']);

        ksort($arr);
        $query = urldecode(http_build_query($arr)) . '&key=' . $this->option['key'];
        return strtoupper(md5($query));
    }


    /**
     * 获取客户端IP地址
     * @return array|false|string
     */
    public function getClientIp()
    {
        $cip = 'unknown';

        if($_SERVER['REMOTE_ADDR']){
            $cip = $_SERVER['REMOTE_ADDR'];
        }elseif (getenv("REMOTE_ADDR")){
            $cip = getenv("REMOTE_ADDR");
        }

        return $cip;
    }


    /**
     * 私有克隆函数，防止外办克隆对象
     */
    private function __clone() {}
}
```

#### 五、示例调用

```
$options = [
    'appid' => 'you wechat appid',
    'key' => 'you key',
    'mch_id' => 'you wechat mch_id'
];

$weChatPay = \lwfdeveloper\wxpay\WeChatPay::getInstance($options);//实例化单例类

/** 订单号*/
$orderId = rand(000000, 999999) . date('YmdHisu') . rand(000000, 999999); 
$openid = 'create wechat pay openid';
$reason = "create wechat pay order";

/** 支付金额*/
$amount = 100;
$callbackUrl = "http://api.develop.cn/api/wechat/callback";

#H5，微信小程序内发起支付
$response = $weChatPay->CreateWeChatOrder($orderId, $reason, $amount, $openid, $callbackUrl);
var_dump($response);

#App内发起微信支付
$responseApp = $weChatPay->CreateWeChatAppOrder($orderId, $reason, $amount, $callbackUrl);
var_dump($responseApp);

```
开箱即用：
```
composer require lwfdeveloper/wxpay
```
最后附上github地址：[微信支付](https://github.com/lwfdeveloper/wxpay/)

