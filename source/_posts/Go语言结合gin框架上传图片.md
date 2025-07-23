---
title: Go语言结合gin框架上传图片
date: 2024-05-11 09:39:58
tags: Go
categories: gin
top: 20
---

## 前言

通过Go+gin框架，实现文件上传功能

```
1、限制文件大小不超过10M
2、仅支持.png,.jpg,.gif,.jpeg,.pptx,.xlsx,.docx格式文件
```

#### 具体实现代码

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
		noAuth.POST("/file/upload", service.FileUpload)
	}
	return r
}
```

2、service/file.go

```
func FileUpload(c *gin.Context) {
	file, err := c.FormFile("file")
	if err != nil {
		common.Response(c, -1, "请选择文件上传!", nil)
		return
	}
	extName := path.Ext(file.Filename)

	allowExt := map[string]bool{
		".jpg":  true,
		".png":  true,
		".gif":  true,
		".jpeg": true,
		".docx": true,
		".xlsx": true,
		".pptx": true,
	}
	if _, ok := allowExt[extName]; !ok {
		common.Response(c, -1, "仅支持png,jpg,jpeg,docx,xlsx,gig,pptx类型文件!", nil)
		return
	}
	day := time.Now().Format("2006-01-02")
	dir := "./static/file/" + day
	e := os.MkdirAll(dir, 0755)
	if e != nil {
		fmt.Println(e, "+++++")
		common.Response(c, -1, "创建文件目录出错，上传文件失败!", nil)
		return
	}

	fileMaxSize := 10 << 20
	if int(file.Size) > fileMaxSize {
		common.Response(c, -1, "文件不允许大于10M!", nil)
		return
	}
	fileName := strconv.FormatInt(time.Now().Unix(), 10) + extName
	dst := path.Join(dir, fileName)

	c.SaveUploadedFile(file, dst)
	response := gin.H{
		"file": dst,
	}
	common.Response(c, 200, "上传成功!", response)
}
```

#### 结果
![alt text](/images/file/image.png)

![alt text](/images/file/image-1.png)