+++
title = "Gin"
date = "2026-06-02T00:00:00+08:00"
draft = false
weight = 6
+++

# 安装

```go
go get -u github.com/gin-gonic/gin
```

## 简单使用

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/", func(c *gin.Context) {
		// 3、返回值
		c.String(http.StatusOK,"hello world")
	})
	// 4、开启监听
	g.Run(":8000")
}
```

# 获取请求参数

## Handler处理器

路由需要传入两个参数，一个为路径，另一个为路由执行的方法，我们叫做它处理器 `Handler` ，而且，该参数是可变长参数。也就是说，可以传入多个 handler，形成一条 handler chain 。

同时对 handler 该函数有着一些要求，该函数需要传入一个 `Gin.Context` **指针**，同时要通过该指针进行值得处理。Handler 函数可以对前端返回 `字符串，Json，Html` 等多种格式或形式文件。

以`GET`为例，其内部实现如下：

```go
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}
```

## 获取GET请求参数

指的是URL中`?`后面携带的参数，例如：`/user/search?username=ANY&age=18`。

获取请求的参数的方法如下：

1.  **c.Query()**

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/user/search", func (c *gin.Context) {
		// 获取值
		name := c.Query("name")
		age := c.Query("age")
		// 3、返回值
		c.String(http.StatusOK, "name: "+name+" age: "+age)
	})
	// 4、开启监听
	g.Run(":8000")
}
```

等同于

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Search(c *gin.Context) {
		// 获取值
		name := c.Query("name")
		age := c.Query("age")
		// 3、返回值
		c.String(http.StatusOK, "name: "+name+" age: "+age)
	}

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/user/search", Search)
	// 4、开启监听
	g.Run(":8000")
}
```

![image.png](assets/获取请求参数-1.png)

2.  **c.DefaultQuery()**

没有设置参数的值设置一个默认值。

如下：

```go
username := c.DefaultQuery("username", "joker")
```

3.  **c.QueryArray()**

使用QueryArray()获取多个值。

```go
ids = c.QueryArrary("ids")
```

请求：http://127.0.0.1:8000/query?ids=1,2,3,4,5

4.  **c.QueryMap()**

使用QueryMap()获取Map类型的数据

```go
user = c.QueryMap("name")
```

请求：http://127.0.0.1:8000/query?user[name]=joker&user[age]=18

## 获取Post请求参数

获取Post请求参数的常用函数：

```go
func (c *Context) PostForm(key string) string
func (c *Context) DefaultPostForm(key, defaultValue string) string
func (c *Context) GetPostForm(key string) (string, bool)
```
```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Handler(c *gin.Context) {
    //获取name参数, 通过PostForm获取的参数值是String类型。
    name := c.PostForm("name")

    // 跟PostForm的区别是可以通过第二个参数设置参数默认值
    name := c.DefaultPostForm("name", "sockstack")

    //获取id参数, 通过GetPostForm获取的参数值也是String类型,
    // 区别是GetPostForm返回两个参数，第一个是参数值，第二个参数是参数是否存在的bool值，可以用来判断参数是否存在。
    id, ok := c.GetPostForm("id")
    if !ok {
        // 参数不存在
    }
}

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/user/search", Handler)
	// 4、开启监听
	g.Run(":8000")
}
```

## 获取URL路径参数

获取路径中的参数用`context.Param`。在定义路由的时候用`/:`后面的符号是一个占位符，我们可以对该值进行传值。如下：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/usr/:name", func(c *gin.Context) {
		// 获取值
		name := c.Param("name")
		// 3、返回值
		c.String(http.StatusOK,name)
	})
	// 4、开启监听
	g.Run(":8000")
}
```

然后我们在浏览器上输入如下地址`http://127.0.0.1:8000/usr/ANY`，那么我们获取到的name就是joker，并返回给我们，如下：

![image.png](assets/获取请求参数-2.png)

## 请求参数绑定到`struct`对象

前面获取参数的方式都是一个个参数的读取，比较麻烦，Gin框架支持将请求参数自动绑定到一个struct对象，这种方式支持Get/Post请求，也支持http请求body内容为json/xml格式的参数。

例子：将请求参数绑定到User struct对象

```go
// User 结构体定义
type User struct {
    Name  string `json:"name" form:"name"`
    Email string `json:"email" form:"email"`
}
```

通过定义struct字段的标签，定义请求参数和struct字段的关系。

struct标签说明：

| 标签 | 说明 |
| --- | --- |
| json:"name" | 数据格式为json格式，并且json字段名为name |
| form:"name" | 表单参数名为name |

**提示：**可以根据自己的需要选择支持的数据类型，例如需要支持json数据格式，可以这样定义字段标签: json:"name"

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type User struct {
	Name string `form:"name"`
	Age  int    `form:"age"`
}

func main() {
    // 1、创建路由
	g := gin.Default()
    g.POST("/user/:id", func(c *gin.Context) {
        // 初始化user struct
        u := User{}
        // 通过ShouldBind函数，将请求参数绑定到struct对象， 处理json请求代码是一样的。
        // 如果是post请求则根据Content-Type判断，接收的是json数据，还是普通的http请求参数
        if c.ShouldBind(&u) == nil {
            // 绑定成功， 打印请求参数
            log.Println(u.Name)
            log.Println(u.Age)
    
        }
        // http 请求返回一个字符串 
        c.String(200, "Success")
    })
}
```

`ShouldBind`有一系列函数，大致就是把前面的方式绑定到结构体的方式，如：`ShouldBindUri()`、`ShouldBindQuery()`等等，用法和`ShouldBind`类似，这里就不展开介绍了

**提示：**如果通过http请求body传递json格式的请求参数，并且通过post请求的方式提交参数，则需要将Content-Type设置为application/json, 如果是xml格式的数据，则设置为application/xml

## 获取请求头信息

获取请求头的常用函数：

-   `func (c *Context) GetHeader(key string) string`

例子：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func Handler(c *gin.Context) {
    //获取请求头Host的值
    host := c.GetHeader("Host")
    //控制台输出host的值
    fmt.Println(host)
}

func main() {
    g := gin.Defalut()
    g.GET("/user", Handler)
}
```

## 获取客户端IP

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)
func main() {
    g := gin.Default()
    g.GET("/user", func(c *gin.Context) {
        // 获取用户IP
        ip := c.ClientIP()
    })
}
```

# 路由

#### 普通路由

```go
r.GET("/index", func(c *gin.Context) {...})
r.GET("/login", func(c *gin.Context) {...})
r.POST("/login", func(c *gin.Context) {...})
```

此外，还有一个可以匹配所有请求方法的`Any`方法如下：

```go
r.Any("/test", func(c *gin.Context) {...})
```

为没有配置处理函数的路由添加处理程序，默认情况下它返回404代码，下面的代码为没有匹配到路由的请求都返回`views/404.html`页面。

```go
r.NoRoute(func(c *gin.Context) {
		c.HTML(http.StatusNotFound, "views/404.html", nil)
	})
```

#### 路由组

我们可以将拥有共同URL前缀的路由划分为一个路由组。习惯性一对`{}`包裹同组的路由，这只是为了看着清晰，你用不用`{}`包裹功能上没什么区别。

```go
func main() {
	r := gin.Default()
	userGroup := r.Group("/user")
	{
		userGroup.GET("/index", func(c *gin.Context) {...})
		userGroup.GET("/login", func(c *gin.Context) {...})
		userGroup.POST("/login", func(c *gin.Context) {...})
	}
	shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
	}
	r.Run()
}
```

路由组也是支持嵌套的，例如：

```go
shopGroup := r.Group("/shop")
	{
		shopGroup.GET("/index", func(c *gin.Context) {...})
		shopGroup.GET("/cart", func(c *gin.Context) {...})
		shopGroup.POST("/checkout", func(c *gin.Context) {...})
		// 嵌套路由组
		xx := shopGroup.Group("xx")
		xx.GET("/oo", func(c *gin.Context) {...})
	}
```

通常我们将路由分组用在划分业务逻辑或划分API版本时。

​  

**结合简单项目实现：**

目录结构：

![image.png](assets/路由-1.png)

```go
package router

import (
	"any.com/service"
	"github.com/gin-gonic/gin"
)

func InitApi(g *gin.Engine) {
	// 请求：/api/v1
	api := g.Group("/api")
	v1 := api.Group("/v1")
	v1.GET("ping", service.Ping)

}
```
```go
package service

import "github.com/gin-gonic/gin"

func Ping(c *gin.Context) {
	c.JSON(200, gin.H{
		"message": "pong",
	})
}
```
```go
package main

import (
	"any.com/router"
	"github.com/gin-gonic/gin"
)

func main() {
	g := gin.Default()
	router.InitApi(g)
	g.Run(":8080")
}
```

#### 路由原理

Gin框架中的路由使用的是[httprouter](https://github.com/julienschmidt/httprouter)这个库。

其基本原理就是构造一个路由地址的前缀树。

#### Handler处理器

路由需要传入两个参数，一个为路径，另一个为路由执行的方法，我们叫做它处理器 `Handler` ，而且，该参数是可变长参数。也就是说，可以传入多个 handler，形成一条 handler chain 。

同时对 handler 该函数有着一些要求，该函数需要传入一个 `Gin.Context` **指针**，同时要通过该指针进行值得处理。

Handler 函数可以对前端返回 字符串，Json，Html 等多种格式或形式文件。

我们已`GET`为例，其内部实现如下：

```go
func (group *RouterGroup) GET(relativePath string, handlers ...HandlerFunc) IRoutes {
	return group.handle(http.MethodGet, relativePath, handlers)
}
```

  

​

# 请求数据参数绑定

为了能够更方便的获取请求相关参数，提高开发效率，我们可以基于请求的`Content-Type`识别请求数据类型并利用反射机制自动提取请求中`QueryString`、`form表单`、`JSON`、`XML`等参数到结构体中。

在Go中，通过反射解析数据都是存放在结构体中，所以我们先定义一个结构体用来接受数据，如下：

```go
type Login struct{
    User string `form:"username" json:"username" xml:"username" binding:"required"`
    Password string `form:"username" json:"username" xml:"username" binding:"required"`
}
```

其中 ：

-   form：会去解析form表单数据
-   json：会去解析json格式数据
-   xml：会去解析xml格式数据
-   binding：required表示设置的参数是必须参数，如果没有传就会报错

### 解析JSON数据

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Login struct{
	User string `form:"username" json:"username" xml:"username" binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则
	g.POST("/loginJSON", func(context *gin.Context) {
		var login Login
		// 对参数进行绑定
		if err := context.ShouldBindJSON(&login);err != nil{
			// 如果有错误返回JSON数据
			context.JSON(304,gin.H{"status": err.Error()})
		}
		// 如果没有报错，取值并返回
		context.JSON(200,gin.H{
			"status": http.StatusOK,
			"username": login.User,
			"password": login.Password,
		})
	})
	g.Run(":8000")
}

```

测试看结果：

（1）、正常请求

![image.png](assets/请求数据参数绑定-1.png)

（2）、错误请求

![image.png](assets/请求数据参数绑定-2.png)

### 解析Form表单

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Login struct{
	User string `form:"username" json:"username" xml:"username" binding:"required"`
	Password string `form:"password" json:"password" xml:"password" binding:"required"`
}

func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则
	g.POST("/loginFORM", func(context *gin.Context) {
		var login Login
		// 对参数进行绑定
		if err := context.ShouldBind(&login);err != nil{
			// 如果有错误返回JSON数据
			context.JSON(304,gin.H{"status": err.Error()})
		}
		// 如果没有报错，取值并返回
		context.JSON(200,gin.H{
			"status": http.StatusOK,
			"username": login.User,
			"password": login.Password,
		})
	})
	g.Run(":8000")
}

```

  

![image.png](assets/请求数据参数绑定-3.png)

  

### 解析URL数据

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Login struct{
	User string `form:"username" json:"username" xml:"username" uri:"username" binding:"required"`
	Password string `form:"password" json:"password" xml:"password" uri:"password" binding:"required"`
}

func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则
	g.GET("/:username/:password", func(context *gin.Context) {
		var login Login
		// 对参数进行绑定
		if err := context.ShouldBindUri(&login);err != nil{
			// 如果有错误返回JSON数据
			context.JSON(304,gin.H{"status": err.Error()})
		}
		// 如果没有报错，取值并返回
		context.JSON(200,gin.H{
			"status": http.StatusOK,
			"username": login.User,
			"password": login.Password,
		})
	})
	g.Run(":8000")
}

```

![image.png](assets/请求数据参数绑定-4.png)

  

### 解析queryString数据

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type Login struct{
	User string `form:"username" json:"username" xml:"username" uri:"username" binding:"required"`
	Password string `form:"password" json:"password" xml:"password" uri:"password" binding:"required"`
}

func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则
	g.GET("/login", func(context *gin.Context) {
		var login Login
		// 对参数进行绑定
		if err := context.ShouldBind(&login);err != nil{
			// 如果有错误返回JSON数据
			context.JSON(304,gin.H{"status": err.Error()})
		}
		// 如果没有报错，取值并返回
		context.JSON(200,gin.H{
			"status": http.StatusOK,
			"username": login.User,
			"password": login.Password,
		})
	})
	g.Run(":8000")
}

```

![image.png](assets/请求数据参数绑定-5.png)

# 入门操作

### 安装

```go
go get -u github.com/gin-gonic/gin
```

### 使用

#### String

`c.String()` 第一个参数是code，第二个参数是格式化字符串，第三个开始的若干参数支持任何数据类型。

例子：简单输出

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	// 1、创建路由
	g := gin.Default()
	// 2、绑定路由规则，执行视图函数
	g.GET("/", func(c *gin.Context) {
		// 3、返回值，c.String() 第一个参数是code，第二个参数是格式化字符串，第三个开始的若干参数支持任何数据类型。
		c.String(http.StatusOK, "尊敬的用户：%s", "ANY")
	})
	// 4、开启监听
	g.Run(":8000")
}
```

浏览器访问：

![image.png](assets/入门操作-1.png)

#### Json

Gin 使用 encoding/json 作为默认的 json 包，可以序列化 map 类型的对象。

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	g := gin.Default()
	g.GET("/", func(c *gin.Context) {
        // 创建一个map
		user := make(map[string]interface{})
		user["name"] = "zhangsan"
		user["age"] = 18
		user["gender"] = "man"
        // 序列化返回
		c.JSON(http.StatusOK, user)
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

浏览器访问：

![image.png](assets/入门操作-2.png)

  

`gin.H` 是 `map[string]interface{}` 的一种快捷方式。

示例代码：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	g := gin.Default()
	g.GET("/", func(c *gin.Context) {
		c.JSON(http.StatusOK, gin.H{
			"name": "ANY",
			"age":  18,
			"sex":  "男",
			"hobby": []string{
				"football",
				"basketball",
			},
		})
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

浏览器访问：

![image.png](assets/入门操作-3.png)

​  

还可以序列化 struct 类型的对象，并且可以使用 tag 标签修改响应结果的字段名。

示例代码：

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type user struct {
	Name string `json:"name"`
	Age  int
	Sex  string `json:"sex"`
}

func main() {
	g := gin.Default()
	g.GET("/", func(c *gin.Context) {
		u := user{
			Name: "zhangsan",
			Age:  18,
			Sex:  "男",
		}
		c.JSON(http.StatusOK, u)
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

![image.png](assets/入门操作-4.png)

#### JSONP

使用 JSONP 向不同域的服务器请求数据。如果查询参数存在回调，则将回调添加到响应体中。

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

type user struct {
	Name string `json:"name"`
	Age  int
	Sex  string `json:"sex"`
}

func main() {
	g := gin.Default()
	g.GET("/user", func(c *gin.Context) {
		c.JSONP(http.StatusOK, user{"小明", 18, "男"})
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

![image.png](assets/入门操作-5.png)

#### PureJSON

通常，JSON 使用 unicode 替换特殊 HTML 字符，例如 < 变为 \\ u003c。如果要按字面对这些字符进行编码，则可以使用 PureJSON。Go 1.6 及更低版本无法使用此功能。

示例

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	g := gin.Default()
	g.GET("/user", func(c *gin.Context) {
		c.PureJSON(http.StatusOK, gin.H{
			"html": "<h1>ANY</h1>",
		})
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

![image.png](assets/入门操作-6.png)

#### SecureJSON

使用 SecureJSON 防止 json 劫持。如果给定的结构是数组值，则默认预置 "`while(1),`" 到响应体。

也可以使用自己的 SecureJSON 前缀，`r.SecureJsonPrefix(")]}',\n")`

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main(){
    r := gin.Default()
    r.GET("/user",func(c *gin.Context){
        arr := [...]string{"a", "b", "c"}
        c.SecureJSON(http.StatusOK, arr)
   })
    _ = r.Run(":8081')
}
```

![image.png](assets/入门操作-7.png)

#### AsciiJSON

使用 AsciiJSON 生成具有转义的非 ASCII 字符的 ASCII-only JSON。

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main(){
    r := gin.Default()
    r.GET("/user",func(c *gin.Context){
    c.AsciiJSON(http.StatusOK, gin.H{
            "name": "张三",
            "email": "zs@gmail.com",
            "age": 20,
        })
   })
    _ = r.Run(":8081')
}
```

![image.png](assets/入门操作-8.png)

#### XML

输出XML 格式

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	g := gin.Default()
	g.GET("/user", func(c *gin.Context) {
		c.XML(http.StatusOK, gin.H{
			"name": "ANY",
			"age":  18,
		})
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

![image.png](assets/入门操作-9.png)

#### YAML

输出yaml格式

```go
package main

import (
	"github.com/gin-gonic/gin"
	"net/http"
)

func main() {
	g := gin.Default()
	g.GET("/user", func(c *gin.Context) {
		c.YAML(http.StatusOK, gin.H{
			"name": "ANY",
			"age":  18,
		})
	})
	// 开启监听
	_ = g.Run(":8080")
}
```

浏览器请求会直接下载文件

文件内容为：

![image.png](assets/入门操作-10.png)

# 文件导出接口demo

```go
func (z *BusinessZP) ExportZPDeviceList(req *dto.GetZPDeviceListReq) (err error) {
	log := z.Log
	fileAbsPath, err := z.SaveData2FileWithStream(req)
	if err != nil {
		err = errors.WithMessage(err, "export data to file failed")
		log.Errorf("%+v", err)
		return err
	}
	defer func() {
		if len(fileAbsPath) > 0 {
			os.Remove(fileAbsPath)
		}
	}()
	fileName := filepath.Base(fileAbsPath)
	err = tools.GinDownloadFile(z.Ctx, fileAbsPath, fileName)
	if err != nil {
		err = errors.WithMessage(err, "download file failed")
		return err
	}
	return nil
}

func (z *BusinessZP) SaveData2FileWithStream(req *dto.GetZPDeviceListReq) (string, error) {
	var MaxQueryCount int64 = 10000
	var MaxCanExport int64 = 800000
	req.SetMaxPageSize(MaxQueryCount)
	tmpDir := config.BusinessConfig.TmpDir
	_, err := os.Stat(tmpDir)
	if err != nil {
		tmpDir = os.TempDir()
	}
	fileName := fmt.Sprintf("%s_%s.csv", "zp_device_info", time.Now().Format("20060102150405"))
	filePath := filepath.Join(tmpDir, fileName)

	file, err := os.Create(filePath)
	if err != nil {
		return "", errors.WithStack(err)
	}
	defer file.Close()
	// 设置文件10MB 缓冲区
	bufferWriter := bufio.NewWriterSize(file, 10*1024*1024)
	writer := csv.NewWriter(bufferWriter)
	defer writer.Flush()

	// 写入表头
	if err := writer.Write([]string{
		"字节设备ID", "主机名", "设备类型", "省份(字节)", "运营商(字节)", "省份(ECDN)", "运营商(ECDN)",
		"验收状态", "实时状态", "运营商是否一致", "总带宽", "CPU", "内存", "总磁盘容量", "注册时间", "更新时间",
	}); err != nil {
		return filePath, errors.WithStack(err)
	}
	req.PageIndex = 1
	req.PageSize = MaxQueryCount
	for {
		select {
		case <-z.Ctx.Done(): // 校验是否导出取消
			return filePath, errors.Wrap(z.Ctx.Err(), "导出数据被取消")
		default:
		}
		res, err := z.GetZPDeviceList(req)
		if err != nil {
			return filePath, err
		}
		if res.Total > MaxCanExport {
			return filePath, errors.New("导出数据量过大，请缩小查询范围")
		}
		var totalBandwidth string
		var (
			cpu  string
			mem  string
			disk string
		)
		ispStatus := map[int]string{
			0: "否",
			1: "是",
		}
		for _, item := range res.Items {
			// 转换为string类型
			if item.TotalBandwidth >= 0 {
				totalBandwidth = fmt.Sprintf("%d", item.TotalBandwidth)
			}
			if item.CPU >= 0 {
				cpu = fmt.Sprintf("%d", item.CPU)
			}
			if item.Memory >= 0 {
				mem = fmt.Sprintf("%d", item.Memory)
			}
			if item.TotalDiskCapacity >= 0 {
				disk = fmt.Sprintf("%d", item.TotalDiskCapacity)
			}
			if err = writer.Write([]string{
				item.DeviceId,
				item.ProviderDeviceId,
				item.Type,
				item.Province,
				item.Isp,
				item.EcdnProvince,
				item.EcdnIsp,
				item.AcceptanceStatus,
				item.RealTimeStatus,
				ispStatus[item.IspStatus],
				totalBandwidth,
				cpu,
				mem,
				disk,
				item.CreatedAt.Format("2006-01-02 15:04:05"),
				item.UpdatedAt.Format("2006-01-02 15:04:05"),
			}); err != nil {
				return filePath, err
			}
		}
		// 刷新缓冲区
		writer.Flush()
		if err = writer.Error(); err != nil {
			return filePath, errors.WithStack(err)
		}
		if int64(len(res.Items)) < MaxQueryCount {
			break
		}
		req.PageIndex++
	}
	return filePath, nil
}
```

# 中间件

Gin框架允许开发者在处理请求的过程中，加入用户自己的钩子（Hook）函数。

这个钩子函数就叫中间件，中间件适合处理一些公共的业务逻辑，比如登录认证、权限校验、数据分页、记录日志、耗时统计等。

#### 默认中间件

gin自带默认有这些中间件

```go

func BasicAuth(accounts Accounts) HandlerFunc
func BasicAuthForRealm(accounts Accounts, realm string) HandlerFunc
func Bind(val interface{}) HandlerFunc //拦截请求参数并进行绑定
func ErrorLogger() HandlerFunc       //错误日志处理
func ErrorLoggerT(typ ErrorType) HandlerFunc //自定义类型的错误日志处理
func Logger() HandlerFunc //日志记录
func LoggerWithConfig(conf LoggerConfig) HandlerFunc
func LoggerWithFormatter(f LogFormatter) HandlerFunc
func LoggerWithWriter(out io.Writer, notlogged ...string) HandlerFunc
func Recovery() HandlerFunc
func RecoveryWithWriter(out io.Writer) HandlerFunc
func WrapF(f http.HandlerFunc) HandlerFunc //将http.HandlerFunc包装成中间件
func WrapH(h http.Handler) HandlerFunc //将http.Handler包装成中间件

```

去除中间件：

```go
//去除默认全局中间件
r := gin.New()//不带中间件
```

`gin.Default()`默认使用了`Logger`和`Recovery`中间件，其中：

-   `Logger`中间件将日志写入`gin.DefaultWriter`，即使配置了`GIN_MODE=release`。
-   `Recovery`中间件会recover任何`panic`。如果有panic的话，会写入500响应码。

如果不想使用上面两个默认的中间件，可以使用`gin.New()`新建一个没有任何默认中间件的路由。

**​**  

**gin中间件中使用goroutine**

当在中间件或`handler`中启动新的`goroutine`时，**不能使用**原始的上下文（c \*gin.Context），必须使用其只读副本（`c.Copy()`）。

#### 自定义中间件

Gin中的中间件必须是一个`gin.HandlerFunc`类型

```go
func MymiddleWare1(c *gin.Context) {
	fmt.Println("进入中间件1")
	c.Set("user1", "zhangsan")
	start := time.Now()
	c.Next()
	cost := time.Since(start)
	fmt.Println("中间1件结束，耗时: ", cost)
}

func MymiddleWare2(c *gin.Context) {
	fmt.Println("进入中间件2")
	c.Set("user2", "lisi")
	start := time.Now()
	c.Next()
	cost := time.Since(start)
	fmt.Println("中间2件结束，耗时: ", cost)
}
```

其中最关键的一点是`ctx.Next()`，调用这个的作用就是处理后续的处理函数。

#### 注册中间件

在gin框架中，可以为每个路由添加任意数量的中间件。

##### 全局中间件

定义全局中间件的话就在主函数中用`Use()`方法加载中间件。为了代码规范，建议绑定路由规则都包含在`{}`中，如下：

```go
func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、全局加载中间件
	g.Use(MyMiddleWare1, MyMiddleWare2)
	// 2、绑定路由规则
	g.GET("/middleware", myFunc)
	g.Run(":8000")
}

func myFunc(c *gin.Context){
	// 获取中间件中设置的值
	user1 := c.MustGet("user1")
	fmt.Println("中间件1中的值：",user1)
    user2 := c.MustGet("user2")
    fmt.Println("中间件2中的值：",user1)
	time.Sleep(time.Second*5)
}
```
```go
进入中间件1
进入中间件2
中间件1中的值：zhangsan
中间件2中的值：lisi
中间1件结束，耗时:  5.0007657s
中间2件结束，耗时： 5.0002031s
```

##### 局部中间件

局部中间件也就是为某个路由单独注册中间件，只需要在绑定路由的时候在路径后面跟上中间件函数即可。

如下：

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"time"
)

func main(){
	// 1、创建路由
	g := gin.Default()
	// 2、加载中间件
	g.Use(MyMiddleWare())
	// 2、绑定路由规则
	g.GET("/test2", MyMiddleWare1, test2)
	g.Run(":8000")
}

func test2(ctx *gin.Context){
	// 获取中间件中设置的值
	ret := ctx.MustGet("name")
	fmt.Println("中间件中的值：",ret)
	time.Sleep(time.Second*5)
}
```

然后访问如下：

```go
进入中间件1
中间件1中的值：zhangsan
中间1件结束，耗时:  5.0007657s
```

我们可以看到执行了两遍中间件，其中在执行函数之前是先执行的全局中间件，然后再是局部中间件，再返回的时候先执行局部中间件，再执行全局中间件。

流程图如下：

![image.png](assets/中间件-1.png)

##### 为路由组注册中间件

为路由组注册中间件有以下两种写法。

写法1：

```go
shopGroup := r.Group("/shop", middleWare1())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

写法2：

```go
shopGroup := r.Group("/shop")
shopGroup.Use(StatCost())
{
    shopGroup.GET("/index", func(c *gin.Context) {...})
    ...
}
```

# gorm

## 介绍

官网：[https://gorm.io/zh\_CN/docs/models.html](https://gorm.io/zh_CN/docs/models.html)

优秀笔记：[gorm使用](https://www.cnblogs.com/qidaii/articles/17807120.html#1-gorm-%E5%9F%BA%E6%9C%AC%E4%BB%8B%E7%BB%8D)

GORM是一个开源的Go语言ORM库，它提供了一种近乎自然的数据库操作方式。开发者可以通过简单的Go代码来定义模型，然后GORM会自动处理数据库的创建、查询、更新和删除等操作。这不仅提高了开发效率，也使得代码更加清晰易读。

**特点：**

-   **简洁的API设计**：GORM的API设计直观易懂，即使是ORM新手也能快速上手。
-   **强大的功能**：支持事务、关联、迁移、预加载等高级功能，满足复杂应用需求。
-   **高性能**：GORM底层使用原生SQL，性能优越。
-   **社区活跃**：GORM拥有一个活跃的社区，不断有新功能和改进被加入。

## 安装

```go
// 获取gorm包
go get -u gorm.io/gorm
// 安装数据库驱动，数据库类型有：MySQL, PostgreSQL, SQLite, SQL Server 和 TiDB
go get -u gorm.io/driver/mysql
```

## 简单配置使用

先创建模型文件：

`touch models/user.go` 创建user.go文件

```go
package models

import (
	"gorm.io/gorm"
)

// User 模型定义
type User struct {
	gorm.Model
	Name     string `gorm:"type:varchar(100);not null"`
	Password string `gorm:"type:varchar(100);not null"`
    Email    string `gorm:"type:varchar(100);not null"`
}

// 注意默认情况下gorm会同时帮我们创建id、created_at、updated_at、deleted_at
// 这三个字段

```

整合到main中

```go
package main

import (
	"any.com/models"
	"fmt"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"log"
)

func main() {

	dsn := "root:123@tcp(192.168.9.11)/mydb?charset=utf8mb4&parseTime=True&loc=Local"
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		log.Fatalf("failed to connect database: %v", err)
	}

	// 检查数据库连接是否成功
	if err := db.Exec("SELECT 1").Error; err != nil {
		log.Fatalf("failed to connect database: %v", err)
	}
	log.Println("Successfully connected to the database!")

	// 执行数据库迁移
	//migrate(db)

	// 创建一个user实例
	user := models.User{
		Name:     "any",
		Password: "123456",
		Email:    "any@example.com",
	}
	result := db.Create(&user)
	if result.Error != nil {
		log.Fatalf("failed to create user: %v", result.Error)
	}
	log.Printf("User has been created successfully. ID: %d \n", user.ID)

	// 查询数据
	selectDB(db)
}

// 数据库迁移
func migrate(db *gorm.DB) {
	if err := db.AutoMigrate(&models.User{}); err != nil {
		log.Fatalf("failed to migrate database: %v \n", err)
	}
	log.Println("User table has been created successfully!")
}

func selectDB(db *gorm.DB) {
	// 查询数据 先查询单条数据
	var user models.User
	// 查第一条件记录  主键盘升序
	db.First(&user)
	log.Printf("First user: %#v\n", user)
	fmt.Println(user.Name, user.Password, user.Email)
}
```

执行结果：

```go
2024/12/27 18:27:30 Successfully connected to the database!
2024/12/27 18:27:30 User has been created successfully. ID: 2 
2024/12/27 18:27:30 First user: models.User{Model:gorm.Model{ID:0x1, CreatedAt:time.Date(2024, time.December, 27, 18, 26, 14, 798000000, time.Local), UpdatedAt:time.Date(2024, time.December, 27, 18, 26, 14, 798000000, time.Local), DeletedAt:gorm.DeletedAt{Time:time.Date(1, time.January, 1, 0, 0, 0, 0, time.UTC), Valid:false}}, Name:"any", Password:"123456", Email:"any@example.com"}
any 123456 any@example.com
```

## 整合gin使用

### 整体目录结构

![image.png](assets/gorm-1.png)

yaml配置文件：`config.yaml`

```go
mysql:
  host: 127.0.0.1
  port: 3306
  user: root
  password: 123
  database: gin_demo
  max-idle-conns: 10
  max-open-conns: 100
  log-mode: ""
  log-zap: false
```

全局变量声明文件：`global/global.go`

```go
package global

import "gorm.io/gorm"

var (
    // 声明全局DB全局变量
	DB *gorm.DB
)
```

### 创建数据库连接

数据库连接配置`initialize/dbConn.go`

```go
package initialize

import (
	"any.com/global"
	"fmt"
	"github.com/spf13/viper"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"
	"log"
)

// 使用viper模块，从配置文件中获取dsn配置信息
// 此处我是为了方便直接把获取配置的方法，写到初始化模块中，但是实际开发中，一般专门创建一个配置模块(config)，然后调用此方法获取配置信息
func getDsn() string {
	viper.SetConfigFile("config") // 指定配置文件名称
	viper.SetConfigType("yaml")   // 文件类型，默认是json
	viper.AddConfigPath("..")     // 添加配置文件所在的路径
	err := viper.ReadInConfig()
	if err != nil {
		log.Panic(err)
	}
	//return "root:123@tcp(192.168.9.20)/gin_demo?charset=utf8mb4&parseTime=True&loc=Local"
	dsn := fmt.Sprintf("%s:%s@tcp(%s)/%s?charset=utf8mb4&parseTime=True&loc=Local",
		viper.GetString("mysql.user"),
		viper.GetString("mysql.password"),
		viper.GetString("mysql.host"),
		viper.GetString("mysql.database"))
	return dsn
}

// 数据库配置初始化
func DBInit() {
	dsn := getDsn()
	var err error
	global.DB, err = gorm.Open(mysql.New(mysql.Config{
		DSN:               dsn,
		DefaultStringSize: 256, // string 类型字段的默认长度
	}), &gorm.Config{
		Logger:      logger.Default.LogMode(logger.Info),
		PrepareStmt: true,
	})
	if err != nil {
		log.Panic(err)
	}
	setPool(global.DB)

}

// 创建连接池
func setPool(db *gorm.DB) {
	sqlDB, err := db.DB()
	if err != nil {
		log.Panicln(err)
		return
	}
	// 设置最大连接数
	sqlDB.SetMaxIdleConns(5)
	// 设置一个连接最长存活时间，防止连接过多造成资源浪费
	sqlDB.SetConnMaxLifetime(1)
	sqlDB.SetMaxOpenConns(10)
}

```

### 创建模型

模型文件：`models/user.go`

```go
package models

import (
	"gorm.io/gorm"
)

// User 模型定义
type User struct {
	gorm.Model
	Name     string `json:"name" gorm:"type:varchar(100);not null"`
	Password string `json:"password" gorm:"type:varchar(100);not null"`
	Age      uint8  `json:"age"`
	Sex      string `json:"sex" gorm:"type:varchar(100);not null"`
	Birthday string `json:"birthday"`
	Email    string `json:"email" gorm:"type:varchar(100)"`
}

// 注意默认情况下gorm会同时帮我们创建id、created_at、updated_at、deleted_at 这三个字段
```

### 数据库迁移配置

迁移配置方法文件：`initialize/gorm.go`

```go
package initialize

import (
	"any.com/global"
	"any.com/models"
	"log"
)

func RegisterTables() {
	err := global.DB.Migrator().AutoMigrate(models.User{})
	if err != nil {
		log.Panic(err)
		return
	}
}
```

### 路由及后端逻辑文件

路由配置文件：`route/user.go`

```go
package router

import (
	"any.com/service"
	"github.com/gin-gonic/gin"
)

func InitUserRouter(g *gin.Engine) {
	g.POST("/user/add", service.Add)
	g.GET("/user/list", service.List)
}
```

后端逻辑：`service/user.go`

```go
package service

import (
	"any.com/global"
	"any.com/models"
	"github.com/gin-gonic/gin"
	"net/http"
)

// 添加用户数据
func Add(c *gin.Context) {
	var user models.User
    // 获取到接口传来的参数并绑定
	err := c.ShouldBindJSON(&user)
	if err != nil {
		c.JSON(200, gin.H{
			"code": 400,
			"msg":  "参数错误",
		})
		return
	}
    // 写入数据库
	global.DB.Create(&user)
	c.JSON(http.StatusOK, gin.H{
		"code": 200,
		"msg":  "添加成功!",
	})
}

// 数据查询
func List(c *gin.Context) {
	var users []models.User
    // 查询数据
	global.DB.Find(&users)
	c.JSON(http.StatusOK, gin.H{
		"code": 200,
		"data": users,
	})
}
```

### main文件

```go
package main

import (
	"any.com/initialize"
	_ "any.com/initialize"
	"any.com/router"
	"github.com/gin-gonic/gin"
)

func main() {
	// 数据库连接初始化
	initialize.DBInit()
	// 创建表
	initialize.RegisterTables()

	// 启动gin
	g := gin.Default()
	router.InitUserRouter(g)
	g.Run(":8080")
}
```

### 测试

添加接口：127.0.0.1:8080/user/add

![image.png](assets/gorm-2.png)

查询接口：127.0.0.1:8080/user/list

![image.png](assets/gorm-3.png)

# swagger

官网：[https://pkg.go.dev/github.com/swaggo/gin-swagger#section-readme](https://pkg.go.dev/github.com/swaggo/gin-swagger#section-readme)

## 安装

```go
// 下载包
go get -u github.com/swaggo/swag/cmd/swag

// 安装
go install github.com/swaggo/swag/cmd/swag@latest

go get -u github.com/swaggo/gin-swagger
go get -u github.com/swaggo/files

swag init
```

在包含 `main.go` 文件（默认情况下）的项目根目录运行 `swag init` 命令，将会解析 swag 注释并生成 `docs/` 目录以及 `/docs/docs.go`、`docs/swagger.json`、`docs/swagger.yaml` 三个文件。

```bash
$ swag init -h # 查看 init 子命令使用方法
NAME:
   swag init - Create docs.go

USAGE:
   swag init [command options] [arguments...]

OPTIONS:
   --quiet, -q                            不在控制台输出日志 (default: false)
   --generalInfo value, -g value          API 通用信息所在的 Go 源文件路径，如果是相对路径则基于 API 解析目录 (default: "main.go")
   --dir value, -d value                  API 解析目录，多个目录可用逗号分隔 (default: "./")
   --exclude value                        解析扫描时排除的目录，多个目录可用逗号分隔
   --propertyStrategy value, -p value     结构体字段命名规则，三种：snake_case，camelCase，PascalCase (default: "camelCase")
   --output value, -o value               所有生成文件的输出目录（swagger.json, swagger.yaml and docs.go）(default:"./docs")
   --outputTypes value, --ot value        生成文件的输出类型（docs.go, swagger.json, swagger.yaml）三种：go,json,yaml (default: "go,json,yaml")
   --parseDependency, --pd                解析依赖目录中的 Go 文件 (default: false)
   --markdownFiles value, --md value      指定 API 的描述信息所使用的 Markdown 文件所在的目录，默认禁用
   --parseInternal                        解析 internal 包中的 Go 文件 (default: false)
   --generatedTime                        输出时间戳到输出文件 `docs.go` 顶部 (default: false)
   --parseDepth value                     依赖项解析深度 (default: 100)
   --requiredByDefault                    默认情况下，为所有字段设置 `required` 验证 (default: false)
   --instanceName value                   设置文档实例名 (default: "swagger")
   --parseGoList                          通过 'go list' 解析依赖关系 (default: true)
   --tags value, -t value                 逗号分隔的标签列表，用于过滤指定标签生成 API 文档。特殊情况下，如果标签前缀是 '!' 字符，那么带有该标记的 API 将被排除
   --help, -h                             显示帮助信息 (default: false)
   ...
```

`swag fmt` 命令可以格式化 swag 注释。

```bash
$ swag fmt -h # 查看 fmt 子命令使用方法
NAME:
   swag fmt - format swag comments

USAGE:
   swag fmt [command options] [arguments...]

OPTIONS:
   --dir value, -d value          API 解析目录，多个目录可用逗号分隔 (default: "./")
   --exclude value                解析扫描时排除的目录，多个目录可用逗号分隔
   --generalInfo value, -g value  API 通用信息所在的 Go 源文件路径，如果是相对路径则基于 API 解析目录 (default: "main.go")
   --help, -h                     显示帮助信息 (default: false)
```

## 结合gin使用

### Swagger UI

完整代码如下：

```go
package main

import (
	"net/http"

	"github.com/gin-gonic/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"

	"gin-swag/docs" // 当前包名为 gin-swag
)

// @title           Swagger Example API
// @version         1.0

// @schemes         http
// @host            localhost:8080
// @BasePath        /api/v1

// @tag.name        example
// @tag.description 示例接口

// Helloworld godoc
//
// @Summary     该操作的简短摘要
// @Description 操作行为的详细说明
// @Tags        example
// @Accept      json
// @Produce     json
// @Success     200 {string} string "Hello World!"
// @Router      /example/helloworld [get]
func Helloworld(g *gin.Context) {
	g.JSON(http.StatusOK, "Hello World!")
}

func main() {
	// 会覆盖上面注释部分 title 属性的设置
	docs.SwaggerInfo.Title = "Swag Example API"

	r := gin.Default()
	v1 := r.Group("/api/v1")
	{
		eg := v1.Group("/example")
		{
			eg.GET("/helloworld", Helloworld)
		}
	}

	// Swagger 文档接口地址
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

| 注释 | 说明 |
| --- | --- |
| @title | 必填，应用程序的名称。 |
| @version | 必填，提供应用程序 API 的版本。 |
| @schemes | 用空格分隔的请求传输协议。 |
| @host | 运行 API 的主机（主机名或 IP 地址）。 |
| @BasePath | 运行 API 的基本路径。 |
| @tag.name | 标签的名称。 |
| @tag.description | 标签的描述。 |
| ​ | ​ |

还有一部分注释代表了 API 操作，其含义如下：

| 注释 | 说明 |
| --- | --- |
| @Summary | 该操作的简短摘要。 |
| @Description | 操作行为的详细说明。 |
| @Tags | 该 API 操作的标签列表，多个标签以逗号分隔。 |
| @Accept | API 可以接收的参数 MIME 类型列表。 |
| @Produce | API 可以生成的参数 MIME 类型列表。 |
| @Success | 成功响应。 |
| @Router | 路由路径定义。 |

使用 swag 根据注释生成 Swagger 文档，在项目根目录下（`.`）执行 `swag init`，将得到新的目录结构：

可以发现 `swag init` 生成的三个文件 `docs.go`、`swagger.json`、`swagger.yaml` 默认都在 `docs/` 目录下。

其中 `swagger.json`、`swagger.yaml` 正是符合 OpenAPI 2.0 规范的 JSON 和 YAML 接口文档，例如 `swagger.yaml` 内容如下：

```yaml
basePath: /api/v1
host: localhost:8080
info:
  contact: {}
  title: Swagger Example API
  version: "1.0"
paths:
  /example/helloworld:
    get:
      consumes:
      - application/json
      description: 操作行为的详细说明
      produces:
      - application/json
      responses:
        "200":
          description: Hello World!
          schema:
            type: string
      summary: 该操作的简短摘要
      tags:
      - example
schemes:
- http
swagger: "2.0"
tags:
- description: 示例接口
  name: example
```

​  

执行 `go run main.go` 启动服务，访问 `http://localhost:8080/swagger/index.html` 即可查看 Swagger UI 交互式文档界面。

![image.png](assets/swagger-1.png)

  

实际项目使用实例：

用户登录接口

```go
//@author: [xxx](https://github.com/xxx)
//@author: [xxx](https://github.com/xxx)
//@function: Login
//@description: 用户登录
//@param: u *model.SysUser
//@return: err error, userInter *model.SysUser
func (userService *UserService) Login(u *system.SysUser) (userInter *system.SysUser, err error) {
    ...
}
```

### ReDoc 风格

也许相较于 Swagger UI 多年不变的界面风格，你更喜欢 ReDoc 风格的 UI，那么 [go-redoc](https://link.juejin.cn/?target=https%3A%2F%2Fgithub.com%2Fmvrilo%2Fgo-redoc) 是一个比较不错的选择。

在 gin 中使用 go-redoc 非常简单，只需要将如下套路代码加入到我们的 `main.go` 文件中即可。

```go
package main

import (
	"net/http"
	"path"
	"runtime"

	"github.com/gin-gonic/gin"
	"github.com/mvrilo/go-redoc"
	ginRedoc "github.com/mvrilo/go-redoc/gin"
	swaggerFiles "github.com/swaggo/files"
	ginSwagger "github.com/swaggo/gin-swagger"

	"gin-swag/docs" // 当前包名为 gin-swag
)

// @title           Swagger Example API
// @version         1.0

// @schemes         http
// @host            localhost:8080
// @BasePath        /api/v1

// @tag.name        example
// @tag.description 示例接口

// Helloworld godoc
//
// @Summary     该操作的简短摘要
// @Description 操作行为的详细说明
// @Tags        example
// @Accept      json
// @Produce     json
// @Success     200 {string} string "Hello World!"
// @Router      /example/helloworld [get]
func Helloworld(g *gin.Context) {
	g.JSON(http.StatusOK, "Hello World!")
}

func main() {
	// 会覆盖上面注释部分 title 属性的设置
	docs.SwaggerInfo.Title = "Swag Example API"

	_, filename, _, _ := runtime.Caller(0)
	doc := redoc.Redoc{
		Title:       "ReDoc Example API",
		Description: "ReDoc Example API Description",
		SpecFile:    path.Join(path.Dir(filename), "docs/swagger.json"),
		SpecPath:    "/swagger.json",
		DocsPath:    "/redoc",
	}

	r := gin.Default()
	v1 := r.Group("/api/v1")
	{
		eg := v1.Group("/example")
		{
			eg.GET("/helloworld", Helloworld)
		}
	}

	// Swagger 文档接口地址
	r.GET("/swagger/*any", ginSwagger.WrapHandler(swaggerFiles.Handler))

	// 注册 Redoc 文档接口地址
	r.Use(ginRedoc.New(doc))

	if err := r.Run(":8080"); err != nil {
		panic(err)
	}
}
```

执行 `go run main.go` 启动服务，访问 [http://localhost:8080/redoc](https://link.juejin.cn/?target=http%3A%2F%2Flocalhost%3A8080%2Fredoc) 即可查看 Redoc UI。

![image.png](assets/swagger-2.png)
