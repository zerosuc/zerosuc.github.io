---
title: "gin请求"
date: 2019-09-01T13:51:10-08:00
draft: false
categories: ["gin"]
tags: ["gin"]
---

## 1. Get 请求参数

使用 Get 请求传参时，类似于这样 `http://localhost:8080/user/save?id=11&name=zhangsan`。

如何获取呢？

### 1.1 普通参数

request url: `http://localhost:8080/user/save?id=11&name=zhangsan`

```go
r.GET("/user/save", func(ctx *gin.Context) {
		id := ctx.Query("id")
		name := ctx.Query("name")
		ctx.JSON(200, gin.H{
			"id":   id,
			"name": name,
		})
	})
```

如果参数不存在，就给一个默认值：

```go
r.GET("/user/save", func(ctx *gin.Context) {
		id := ctx.Query("id")
		name := ctx.Query("name")
		address := ctx.DefaultQuery("address", "北京")
		ctx.JSON(200, gin.H{
			"id":      id,
			"name":    name,
			"address": address,
		})
	})
```

判断参数是否存在：

```go
r.GET("/user/save", func(ctx *gin.Context) {
		id, ok := ctx.GetQuery("id")
		address, aok := ctx.GetQuery("address")
		ctx.JSON(200, gin.H{
			"id":      id,
			"idok":    ok,
			"address": address,
			"aok":     aok,
		})
	})
```

id 是数值类型，上述获取的都是 string 类型，根据类型获取：

```go
type User struct {
	Id   int64  `form:"id"`
	Name string `form:"name"`
}
r.GET("/user/save", func(ctx *gin.Context) {
		var user User
		err := ctx.BindQuery(&user)
		if err != nil {
			log.Println(err)
		}
		ctx.JSON(200, user)
})
```

也可以：

```go
r.GET("/user/save", func(ctx *gin.Context) {
		var user User
		err := ctx.ShouldBindQuery(&user)
		if err != nil {
			log.Println(err)
		}
		ctx.JSON(200, user)
	})
```

区别：

```go
type User struct {
	Id      int64  `form:"id"`
	Name    string `form:"name"`
	Address string `form:"address" binding:"required"`
}
```

当 bind 是必须的时候，ShouldBindQuery 会报错，开发者自行处理，状态码不变。

BindQuery 则报错的同时，会将状态码改为 400。所以一般建议是使用 Should 开头的 bind。

### 1.2 数组参数

请求 url：`http://localhost:8080/user/save?address=Beijing&address=shanghai`

```go
r.GET("/user/save", func(ctx *gin.Context) {
		address := ctx.QueryArray("address")
		ctx.JSON(200, address)
	})
```

```go
r.GET("/user/save", func(ctx *gin.Context) {
		address, ok := ctx.GetQueryArray("address")
		fmt.Println(ok)
		ctx.JSON(200, address)
	})
```

```go
r.GET("/user/save", func(ctx *gin.Context) {
		var user User
		err := ctx.ShouldBindQuery(&user)
		fmt.Println(err)
		ctx.JSON(200, user)
	})
```

### 1.3 map 参数

请求 url：`http://localhost:8080/user/save?addressMap[home]=Beijing&addressMap[company]=shanghai`

```go
r.GET("/user/save", func(ctx *gin.Context) {
		addressMap := ctx.QueryMap("addressMap")
		ctx.JSON(200, addressMap)
	})
```

```go
r.GET("/user/save", func(ctx *gin.Context) {
		addressMap, _ := ctx.GetQueryMap("addressMap")
		ctx.JSON(200, addressMap)
	})
```

map 参数 bind 并没有支持

## 2. Post 请求参数

post 请求一般是表单参数和 json 参数

### 2.1 表单参数

![1724075529901](https://zerosuc.github.io/posts/gin/1724075529901.png)

```go
r.POST("/user/save", func(ctx *gin.Context) {
		id := ctx.PostForm("id")
		name := ctx.PostForm("name")
		address := ctx.PostFormArray("address")
		addressMap := ctx.PostFormMap("addressMap")
		ctx.JSON(200, gin.H{
			"id":         id,
			"name":       name,
			"address":    address,
			"addressMap": addressMap,
		})
	})
```

```go
r.POST("/user/save", func(ctx *gin.Context) {
		var user User
		err := ctx.ShouldBind(&user)
		addressMap, _ := ctx.GetPostFormMap("addressMap")
		user.AddressMap = addressMap
		fmt.Println(err)
		ctx.JSON(200, user)
	})
```

### 2.2json 参数

```json
{
  "id": 1111,
  "name": "zhangsan",
  "address": ["beijing", "shanghai"],
  "addressMap": {
    "home": "beijing"
  }
}
```

```go
r.POST("/user/save", func(ctx *gin.Context) {
		var user User
		err := ctx.ShouldBindJSON(&user)
		fmt.Println(err)
		ctx.JSON(200, user)
	})
```

其他类型参数注入 xml，yaml 等和 json 道理一样

## 3. 路径参数

请求 url：`http://localhost:8080/user/save/111`

```go
r.POST("/user/save/:id", func(ctx *gin.Context) {
		ctx.JSON(200, ctx.Param("id"))
	})
```

## 4. 文件参数

```go
r.POST("/user/save", func(ctx *gin.Context) {
		form, err := ctx.MultipartForm()
		if err != nil {
			log.Println(err)
		}
		files := form.File
		for _, fileArray := range files {
			for _, v := range fileArray {
				ctx.SaveUploadedFile(v, "./"+v.Filename)
			}

		}
		ctx.JSON(200, form.Value)
	})
```

## 5.客户端请求头

```go
router.GET("/", func(c *gin.Context) {
  // 首字母大小写不区分  单词与单词之间用 - 连接
  // 用于获取一个请求头
  fmt.Println(c.GetHeader("User-Agent"))
  //fmt.Println(c.GetHeader("user-Agent"))
  //fmt.Println(c.GetHeader("user-AGent"))

  // Header 是一个普通的 map[string][]string
  fmt.Println(c.Request.Header)
  // 如果是使用 Get方法或者是 .GetHeader,那么可以不用区分大小写，并且返回第一个value
  fmt.Println(c.Request.Header.Get("User-Agent"))
  fmt.Println(c.Request.Header["User-Agent"])
  // 如果是用map的取值方式，请注意大小写问题
  fmt.Println(c.Request.Header["user-agent"])
  // 自定义的请求头，用Get方法也是免大小写
  fmt.Println(c.Request.Header.Get("Token"))
  fmt.Println(c.Request.Header.Get("token"))
  c.JSON(200, gin.H{"msg": "成功"})
})

```

### 6.服务器响应头

```go
// 设置响应头
router.GET("/res", func(c *gin.Context) {
  c.Header("Token", "sdsafdsafdfdwwwxxxx")
  c.Header("Content-Type", "application/text; charset=utf-8")
  c.JSON(0, gin.H{"data": "xxxxxxxx"})
})
```
