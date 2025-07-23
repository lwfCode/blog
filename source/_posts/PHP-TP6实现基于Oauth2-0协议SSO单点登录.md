---
title: PHP+TP6实现基于Oauth2.0协议SSO单点登录
date: 2024-11-22 15:31:31
tags: PHP
categories: SSO
top: 23
---

#### 一、SSO简介

##### 1.1 单点登录含义

单点登录(Single sign on)，英文名称缩写SSO，SSO的意思就是在多系统的环境中，登录单方系统，就可以在不用再次登录的情况下访问相关受信任的系统，也就是说只要登录一次单体系统就可以。

#### 1.2 单点登录角色

单点登录一般包括下面三种角色：

*   用户(多个)

*   认证中心(一个)

*   Web应用(多个)

PS：这里所说的web应用可以理解为SSO Client，认证中心可以说是SSO Server。

#### 二、Oauth2.0协议

#### 2.1 OAuth2.0简介
Oauth2.0是一种开放协议, 允许用户让第三方应用以安全且标准的方式获取该用户在某一网站，移动或者桌面应用上存储的秘密的资源（如用户个人信息，照片，视频，联系人列表），而无需将用户名和密码提供给第三方应用。

官网:https://oauth.net/2/

#### 2.2、OAuth2.0角色

- 资源所有者(Resource Owner)： 能够许可受保护资源访问权限的实体。当资源所有者是个人时，它作为最终用户被提及。
- 用户代理(User Agent)： 指的的资源拥有者授权的一些渠道。一般指的是浏览器、APP
- 客户端(Client) 使用资源所有者的授权代表资源所有者发起对受保护资源的请求的应用程序。术语“客户端”并非特指任何特定的的实现特点（例如：应用程序是否在服务器、台式机或其他设备上执行）。
授权服务器(Authorization Server)： 在成功验证资源所有者且获得授权后颁发访问令牌给客户端的服务器。
- 授权服务器和资源服务器之间的交互超出了本规范的范围。授权服务器可以和资源服务器是同一台服务器，也可以是分离的个体。一个授权服务器可以颁发被多个资源服务器接受的访问令牌。
- 资源服务器(Resource Server)： 托管受保护资源的服务器，能够接收和响应使用访问令牌对受保护资源的请求。

#### 2.3 Oauth2.0四种授权模式

##### 1、客户端模式（Client Credentials）

指客户端以自己的名义，而不是以用户的名，向“服务提供商”进行认证。严格的说，客户端模式并不属于OAuth框架所要解决的问题。在这种模式中，用户直接向客户端注册，客户端以自己的名义要求“服务提供商”提供服务，其实不存在授权问题。


| 参数     | 说明  |
| -----    | ---  |
| client_id | 客户端id （必须）|
| client_secret | 客户端密钥 （必须）|
| grant_type  |  使用的授权模式，（必须），固定值：clent_credentislas  |



**示例：**

```
https://oauth.example.com/token?grant_type=client_credentials&client_id=CLIENT_ID&client_secret=SECRET
```


服务提供者验证通过以后，直接返回令牌。这种方式给出的令牌，是针对第三方应用的，而不是针对用户的。因此这就要求我们对client完全的信任，而client本身也是安全的。所以这种模式一般用来提供给我们完全信任的服务器端服务。

##### 2、密码模式（Resource Owner Password Credentials）

使用用户名/密码作为授权方式从授权服务器上获取令牌，一般不支持刷新令牌。这种方式风险很大，用户向客户端提供自己 的用户名和密码。客户端使用这些信息，向“服务提供商”索要授权。这种方式通常用于可信任的应用程序，比如用户拥有自己的资源服务器并且信任应用程序直接使用其凭据。


| 参数 | 说明 |
| --- | --- |
| client_id | 客户端id 必须|
| clent_secret |  客户端密钥，必须|
| grant_type|  授权模式，必须，固定“password”|
| username |  资源拥有者用户账号，必须|
| password |  资源拥有者密码，必须|


**示例：**

https://oauth.example.com/token?client_id=CLIENT_ID&client_secret=SECRET&grant_type=password&username=USERNAME&password=PASSWORD

密码模式说明：

密码模式的优点是他的实现相对简单，适用于那些已经具有高度信任度的应用程序，例如移动应用程序或第一方Web应用程序。然而，密码模式也存在一些安全风险，因为他要求客户端直接处理用户的凭据，这可能会增加密码泄露的风险。因此，在使用密码模式时，必须非常小心的保护客户端的安全性，例如使用安全的存储和传输机制来处理用户凭据。

<!-- 流程图： -->

<!-- ![image](/images/sso/1.jpg) -->

##### 3、隐式授权模式（Implicit Grant）

首先用户访问页面时，会重定向到认证服务器，接着认证服务器给用户一个认证页面，等待用户授权，用户填写信息完成授权后，认证服务器返回token。

操作步骤说明：

【步骤1】【步骤2】用户访问客户端，需要使用服务提供商的数据（用户信息），客户端通过重定向跳转到服务提供商的页面

【步骤3】用户选择是否给予 客户端授权访问 服务提供商（用户信息）数据的权限

【步骤4】用户给予授权后，授权系统通过重定向（redirect_ui）并携带 访问令牌（access_token）跳转回客户端。

【步骤5】客户端携带access_token向资源服务器发出请求资源的请求

【步骤6】【步骤7】服务提供商的资源服务器返回数据给客户端使用

关键步骤：

步骤2：客户端申请认证的url,包含一下参数


| 参数 | 说明 |
| --- | --- |
| client_id | 客户端id，必选 |
| response_type | 授权类型，必选，此处值为 “token” |
| redirect_uri | 重定向uri，接受或拒绝请求后的跳转网址，可选 |
| scope | 申请授权范围，可选 |
| state | 客户端当前状态，可以指定任意值 |


示例：A网站提供一个链接，用户点击后就会跳转到B网站，授权用户数据给A网站使用。下面就是A网站跳转B网站的一个示意链接：

https://www.b.com/oauth/authorize?client_id=client_id&response_type=token&scope=app&redirect_uri=http://xx.xx/callback

步骤3：用户跳转后，B网站会要求用户登录，然后询问是否同意给予A网站授权（此过程也可以自动授权，用户无感知）

步骤4：授权服务器将授权码将令牌（access_token）以Hash的形式存放在重定向uri的fargment中发送给浏览器。认证服务器回应客户端的URI,包含一下参数：


| 参数 | 说明 |
| --- | --- |
| access_token| 访问令牌，必须 |
| token_type | 令牌类型，必选 |
| expires_in | 过期时间，单位为秒 |
| scope |  权限范围|
| state |  指定参数值，客户端请求包含此参数，服务端会返回原始值|

示例：

https://server.example.com/xxx#access_token=ACCESS_TOKEN&state=xyz&token_type=example&expires_in=3600

或

https://www.a.com/callback#token=ACCESS_TOKEN

##### 4、授权码模式（Authrization Code) 推荐使用

第三方应用先申请一个授权码code,然后通过code获取令牌Access Token

授权码模式是功能最完全，流程最严密安全的授权模式。他的特点就是通过客户端的后台服务器，与“服务提供商”的认证服务器进行互动，access_token不会经过浏览器或移动端的App，而是直接从服务端去校验，这样就最大限度的减少了令牌泄露的风险。

接入平台的第三方应用，必须首先联系平台管理员或技术支持注册应用，提供应用名称、访问地址等。注册完成之后会获得一个客户端安全凭证，安全凭证包括APPKEY和APPSECRET。

```
APPKEY 用于标识客户端身份。
APPSECRET 用于验证客户端身份的密钥。
```

<!-- 流程图：

![image](/images/sso/2.jpg) -->

**第一步**：用户访问第三方应用，第三方应用需要引导未登录用户重定向访问以下API


```
GET {{SSO地址}}/api/oauth/authorize?client_id=YOUR_CLIENT_ID&response_type=code&state=snsapi_login&
&redirect_uri=YOUR_REDIRECT_URI&id=YOUR_ID
```


```
client_id：客户端身份标识APPKEY
client_secret：客户端密钥APPSECRET
response_type：code  (固定值)
redirect_uri：客户端回调地址（需urlcode编码）
state:snsapi_login (固定值)
id:平台用户唯一标识 (可选)

```

**注意事项：**

1、client_id必须与在平台注册应用时获取的APPKEY一致。
2、redirect_uri 必须为标准http协议格式且是在平台应用注册时填写的回调地址。

**第二步**：在登录页面输入用户的账号名、密码，如果登录成功自动跳转到第三步。

**第三步**：登录验证成功后，统一认证服务会自动跳转到页面：


```
YOUR_REDIRECT_URI?code=CODE

如下所示：
http://192.167.10.24/api/oauth/callback?code=b3b20b5da14e6bf50540d18b50e28962c92abe70&state=scope
```

**第四步**：第三方应用获取到code之后，需要调用以下API换取Access Token：

```
POST {{SSO地址}}/api/oauth/getAccessToken?client_id=YOUR_CLIENT_ID
&client_secret=YOUR_CLIENT_SECRET
&grant_type=authorization_code
&redirect_uri=YOUR_REDIRECT_URI
&code=CODE
```

- **client_id：客户端身份标识APPKEY**
- **client_secret：客户端密钥APPSECRET**
- **grant_type：固定值 authorization_code** 
- **redirect_uri：客户端回调地址**
- **code：获取到的授权码**

如果Access Token获取成功，则返回以下格式的数据


```
{
    "access_token": "8e09d299442693e63dc06fead3ee057b1211ae1e",
    "expires_in": 3600,
    "token_type": "Bearer",
    "scope": null,
    "refresh_token": "7719292d90ebfad4bf1baf734d6bb972a331bb9b"
}
```
进入第五步。
如果Access Token获取不成功，则返回错误响应数据，结束。


```
{
"error_description":"非法的Client_ID",
"error":"invalid_client"
}

{
"error_description":"未验证通过的客户端身份",
"error":"unauthorized_client"
}

{
"error_description":"非法的redirect_url",
"error":"invalid_redirect_url"
}

```

**第五步**：使用返回的access_token访问如下接口获取用户信息


```
POST {{SSO地址}}/api/oauth/getUserInfo

需要在请求头中传入token参数，传参方式如下：
参数名：Authorization
参数值：Bearer YOUR_ACCESS_TOKEN

```
返回正确如下：

```
{
    "code": 200,
    "message": "success",
    "data": {
        "accountId": "用户唯一标识",
        "mobile": "手机号码",
        "email": "邮箱",
        "createTime": "创建时间",
        "projectId": "平台组织唯一id",
        "companyName": "平台名称",
        "fullname": "用户名称",
        "contactPhone": "工作电话",
        "jobNumber": "工号",
        "departments": [ //加入部门列表
            {
                "departmentId": "766b3670-84d8-481e-8cdf-9e263658c1bd",
                "IsMainDepartment": 0,
                "IsManager": 0,
                "accountId": "417724ac-2a30-4e31-b0e3-4e1411e7f66e",
                "departmentName": "技术支持"
            }
        ],
        "roles": [], //加入角色列表
        "user_token": //用户权限token "0e404a0c10ba0be0660600920d50f500904103d091081016"
    }
}
```


返回错误如下：

```
{
    "error": "invalid_token",
    "error_description": "The access token provided is invalid"
}
```

**第六步**：第三方应用获取到用户信息之后，就可以进行本地登录了，分配用户角色以访问系统页面，结束。

#### 本次以第四种授权码方式为例：

##### PHP+TP6+Oauth2.0协议实现单点登录伪代码如下：

mysql表结构：

```
CREATE TABLE `oauth_access_tokens` (
  `access_token` varchar(40) NOT NULL,
  `client_id` varchar(80) NOT NULL,
  `user_id` varchar(255) DEFAULT NULL,
  `expires` timestamp NOT NULL,
  `scope` varchar(2000) DEFAULT NULL,
  PRIMARY KEY (`access_token`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `oauth_authorization_codes` (
  `authorization_code` varchar(40) NOT NULL,
  `client_id` varchar(80) NOT NULL,
  `user_id` varchar(255) DEFAULT NULL,
  `redirect_uri` varchar(2000) DEFAULT NULL,
  `expires` timestamp NOT NULL,
  `scope` varchar(2000) DEFAULT NULL,
  PRIMARY KEY (`authorization_code`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `oauth_clients` (
  `client_id` varchar(80) NOT NULL,
  `client_secret` varchar(80) NOT NULL,
  `redirect_uri` varchar(2000) NOT NULL,
  PRIMARY KEY (`client_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `oauth_refresh_tokens` (
  `refresh_token` varchar(40) NOT NULL,
  `client_id` varchar(80) NOT NULL,
  `user_id` varchar(255) DEFAULT NULL,
  `expires` timestamp NOT NULL,
  `scope` varchar(2000) DEFAULT NULL,
  PRIMARY KEY (`refresh_token`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `oauth_scopes` (
  `scope` text,
  `is_default` tinyint(1) DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

INSERT INTO `test`.`oauth_clients` (`client_id`, `client_secret`, `redirect_uri`) VALUES ('you_client', 'you_client_secret', 'https://www.xxx.com/api/oauth/callback');
```

##### OAuth2 第三方类

```
下载地址链接: https://pan.baidu.com/s/1Kna9PRkbF4bH9ZYu31H_0A 提取码: b222
```


##### Route.php


```
<?php
use think\facade\Route;

Route::group('oauth', function () {

    /** callback */
    Route::any('/callback', 'api/mdy.OauthController/callback');

    /** 授权*/
    Route::any('/authorize', 'api/mdy.OauthController/authorize');

    /** 获取access_token */
    Route::post('/getAccessToken', 'api/mdy.OauthController/getAccessToken');

    /** 获取用户基本信息 */
    Route::post('/getUserInfo', 'api/mdy.OauthController/getUserInfo');
    
})->middleware(['ApiAuth']);
```


##### OauthController.php

```

<?php
namespace app\api\controller;

use app\BaseController;
use Gaoming13\HttpCurl\HttpCurl;
use app\api\service\Auth;

class OauthController
{
    private $authService;
    private $storage;
    private $userId;
    public $isAuthorized = true;
    public $config = [];
    public $headerJson = "Content-Type:application/json";


    public function __construct()
    {
        $this->config = [
            "dns" => config('sso.dns'),
            "host" => config('sso.host'),
            "port" => config('sso.port'),
            "dbname" => config('sso.dbname')
        ];
        $dns = "{$this->config['dns']}:host={$this->config['host']};port={$this->config['port']};dbname={$this->config['dbname']};charset=utf8";
        $this->storage = new \OAuth2\Storage\Pdo(
            array('dsn' => $dns, 'username' => config('sso.username'), 'password' => config('sso.password'))
        );
        $this->authService =new \OAuth2\Server($this->storage);

    }


    public function callback()
    {
        $code = $this->request->param('code') ?? '';
        $state = $this->request->param('state') ?? '';

        if (empty($code)){
            throw new \Exception("code empty!");
        }
        
        /** 测试地址 */
        $url = "http://192.167.10.24/api/oauth/getAccessToken";
        $redirect_url = "http://192.167.10.24/api/oauth/callback";
        $requestParams = [
            "client_id" => "you client_id",
            "client_secret" => "you client_secret",
            "grant_type"  => "authorization_code",
            "code" => $code,
            "redirect_uri" => urlencode($redirect_url)
        ];
        list($body,$header) =  HttpCurl::request($url,'POST',
            json_encode($requestParams),
            [$this->headerJson]);

        $response = json_decode($body,true);
        return json($response);
    }


    /**
     * OAuth2.0认证接口
     * @return \think\response\Json
     */
    public function authorize()
    {
        \OAuth2\Autoloader::register();

        $this->authService->addGrantType(new \OAuth2\GrantType\ClientCredentials($this->storage));
        $this->authService->addGrantType(new \OAuth2\GrantType\AuthorizationCode($this->storage));
        $request = \OAuth2\Request::createFromGlobals();

        $response = new \OAuth2\Response();
        $state = $this->request->param('state') ?: "snsapi_login";

        /** 唯一标识，用于鉴别哪个用户授权登录 */
        $this->userId = $this->request->param('id');
  
        if (!$this->authService->validateAuthorizeRequest($request, $response)) {
            $response->send();die;
        }
        $is_authorized = $this->isAuthorized;
        $this->authService->handleAuthorizeRequest($request, $response, $is_authorized,$this->userId);
        if ($is_authorized) {
            $code = substr($response->getHttpHeader('Location'), strpos($response->getHttpHeader('Location'), 'code=')+5, 40);
            $this->authService->handleAuthorizeRequest($request, $response, $is_authorized,$userId)->send();
            return false;
        }
        $response->send();die;
    }


    /**
     * @return \think\response\Json
    */
    public function getAccessToken()
    {
        \OAuth2\Autoloader::register();
        //$this->authService = new \OAuth2\Server($this->storage);
        $request = \OAuth2\Request::createFromGlobals();
        $request->request = array_merge($request->query,$request->request);
        $this->authService->addGrantType(new \OAuth2\GrantType\ClientCredentials($this->storage));
        $this->authService->addGrantType(new \OAuth2\GrantType\AuthorizationCode($this->storage));
        $this->authService->handleTokenRequest($request)->send();die;

    }


    /**
     * 获取用户基本信息
     * @return \think\response\Json|void
    */
    public function getUserInfo()
    {
        $request = \OAuth2\Request::createFromGlobals();

        if (!$this->authService->verifyResourceRequest($request)) {
            $this->authService->getResponse()->send();die;
        }else{
            $token = $this->authService->getAccessTokenData($request);
            if (isset($token['user_id'])){
                $user = Auth::getUserInfo($token['user_id']);
                $cache = new \app\dao\Redis();
                $session = $cache::getAccountData($token['user_id']);
                $user['user_token'] = $session['mdId'] ?? '';
            }
        }
        return json(['code' => 200, 'message' => "success", "data" => $user ?? []]);
    }
}

```
######  最终请求：

发起授权：

```
https://xxx.com/v1/api/auth/authorize?client_id=642bfd5-4f01-4708-a54f-aa7a2c77e0c5&response_type=code%20&redirect_uri=https%3A%2F%2Fwww.xxxx.com%3A1883%2Fv1%2Fapi%2Foauth%2Fcallback&id=8b51227f-29c5-4815-b2b3-f3b5fe2881e2

```

###### 获取access_token:

![image](/images/sso/sso_1.jpg)

###### 获取用户信息：

![image](/images/sso/sso_2.jpg)