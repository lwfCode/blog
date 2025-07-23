---
title: Go语言实现发送邮箱验证码
date: 2024-05-11 09:58:03
tags: Go
categories: gin
top: 21
---

## 前言

实现发送邮箱验证码前置条件
1、注册163邮箱账号
2、开通IMAP/SMTP、POP3/SMTP服务
3、获取授权密码


```
go get github.com/jordan-wright/email
go get github.com/joho/godotenv
```
### 具体实现代码

1、route.go

```
func Router() *gin.Engine {

	r := gin.Default()

	r.Use(cors.Default())

	v1Verison := "/v1/api"
	/** 无需授权 */
	noAuth := r.Group(v1Verison)
	{
		//文件上传
		noAuth.POST("/send", service.SendCode)
	}
	return r
}
```

2.service/email.go

```
func SendCode(c *gin.Context) {
	json := make(map[string]interface{})
	c.BindJSON(&json)

	email, ok := json["email"]
	if !ok {
		common.Response(c, 403, "请传入email", nil)
		return
	}

	/** 根据业务逻辑需要判断邮箱是否被注册 */
	
    //生成code
	code := helper.GetRandCode()
	err := helper.SendEmailCode(email.(string), code)
	if err != nil {
		log.Printf("[Send Code 发送失败]:%v\n", err)
		common.Response(c, 403, "发送失败", nil)
		return
	}

	//发送成功存入redis中，有效期为30分钟
	err = models.CacheSet(define.CACHE_PREFIX+email.(string), code, time.Second*1800)
	if err != nil {
		log.Printf("[Cache Set Code ERROR]:%v\n", err)
	}

	common.Response(c, 200, "success", nil)
}
```

3.helper.go

```
func SendEmailCode(toUserEmail, code string) error {
	e := email.NewEmail()
	e.From = "go-im <你的163邮箱账号>"
	e.To = []string{toUserEmail}
	e.Subject = "【Go-IM】您好，验证码已发送，30分钟内有效。"
	e.HTML = []byte("您的验证码：<b>" + code + "</b>")

    //你的授权密码
	MailPwd, err := define.GetMailPwd()
	if err != nil {
		log.Printf("auth error")
		return err
	}
	return e.SendWithTLS("smtp.163.com:465",
		smtp.PlainAuth("", "你的163邮箱账号", MailPwd, "smtp.163.com"),
		&tls.Config{InsecureSkipVerify: true, ServerName: "smtp.163.com"})
}

// 生成随机6位验证码
func GetRandCode() string {
	code := fmt.Sprintf("%06v", rand.New(rand.NewSource(time.Now().UnixNano())).Int31n(1000000))
	return code
}
```

4.define.go
```
func GetMailPwd() (string, error) {
	err := godotenv.Load(".env")
	if err != nil {
		log.Printf("load env failed!")
		return "", err
	}
	MailPwd := os.Getenv("MailPwd")

	return MailPwd, err
}
```

.env
```
MailPwd="你的163邮箱授权密码"
```

#### 最终结果

![alt text](/images/file/email.png)

