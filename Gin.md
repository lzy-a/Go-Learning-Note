# Gin

### html渲染

```go
type Person struct {
    Name string
    Age  int
}

func main() {
    r := gin.Default()

    // Define the template
    var html = template.Must(template.New("example").Parse(`
    <html>
    <head>
      <title>Example</title>
    </head>
    <body>
      <h1>Hello, {{.Name}}!</h1>
      <p>Your age is {{.Age}}.</p>
    </body>
    </html>
    `))

    // Set the template for the router
    r.SetHTMLTemplate(html)

    // Define the route
    r.GET("/", func(c *gin.Context) {
        // Create a Person struct
        person := Person{Name: "John", Age: 30}

        // Render the template with the Person struct
        c.HTML(http.StatusOK, "example", person)
    })

    // Start the server
    r.Run(":8080")
}

```

`gin.Default()`是Gin框架的一个函数，它返回一个默认的路由引擎。在这个应用程序中，我们使用`gin.Default()`来创建一个新的路由引擎。`c.HTML()`是 Gin 框架中的一个方法，用于返回一个 HTML 格式的响应。它的第一个参数是状态码，第二个参数是要返回的 HTML 内容。

### 从文件加载html模板

```go
func main() {
	router := gin.Default()
	router.LoadHTMLGlob("templates/**/*")
	router.GET("/posts/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "posts/index.tmpl", gin.H{
			"title": "Posts",
		})
	})
	router.GET("/users/index", func(c *gin.Context) {
		c.HTML(http.StatusOK, "users/index.tmpl", gin.H{
			"title": "Users",
		})
	})
	router.Run(":8080")
}
```

### HTTP2 server 推送

```go
package main

import (
	"html/template"
	"log"

	"github.com/gin-gonic/gin"
)

var html = template.Must(template.New("https").Parse(`
<html>
<head>
  <title>Https Test</title>
  <script src="/assets/app.js"></script>
</head>
<body>
  <h1 style="color:red;">Welcome, Ginner!</h1>
</body>
</html>
`))

func main() {
	r := gin.Default()
	r.Static("/assets", "./assets")
	r.SetHTMLTemplate(html)

	r.GET("/", func(c *gin.Context) {
		if pusher := c.Writer.Pusher(); pusher != nil {
			// 使用 pusher.Push() 做服务器推送
			if err := pusher.Push("/assets/app.js", nil); err != nil {
				log.Printf("Failed to push: %v", err)
			}
		}
		c.HTML(200, "https", gin.H{
			"status": "success",
		})
	})

	// 监听并在 https://127.0.0.1:8080 上启动服务
	r.RunTLS(":8080", "./testdata/server.pem", "./testdata/server.key")
}
```

这段代码是一个使用gin框架实现的https服务器，主要功能是在浏览器中访问https://127.0.0.1:8080时，返回一个html页面，页面中包含一个js文件的引用，同时服务器会将这个js文件推送给客户端。

其中，html变量是一个html模板，模板中包含了一个h1标签和一个script标签，script标签引用了/assets/app.js这个js文件。

在main函数中，首先创建了一个gin的默认引擎r，然后通过r.Static()方法将/assets目录下的文件映射到了/assets路由下，这样当浏览器请求/assets/app.js时，服务器就会返回/assets目录下的app.js文件。

接着，通过r.SetHTMLTemplate()方法将html模板设置为默认的html模板。

然后，通过r.GET()方法将/路由映射到了一个回调函数中，这个回调函数中首先判断是否支持服务器推送，如果支持，则通过pusher.Push()方法将/assets/app.js这个js文件推送给客户端。

最后，通过c.HTML()方法将html模板渲染成html页面并返回给客户端。

最后一行代码r.RunTLS()是启动https服务器的代码，其中第一个参数是服务器监听的地址和端口，第二个参数是服务器证书的路径，第三个参数是服务器私钥的路径。

`pusher.Push()`用于在客户端请求资源之前将资源推送到客户端，这可以提高网页的性能。当客户端收到服务器推送的资源时，它会将这些资源缓存起来，以便在后续的页面加载中使用。当客户端需要使用这些资源时，它会从缓存中获取，而不是再次向服务器请求。这可以减少页面加载时间和网络带宽的使用。

### JSONP

JSONP（JSON with Padding）是 JSON 的一种“使用模式”，可让网页在不跨域的情况下从其他来源获取数据。JSONP 的实现方式是，网页通过添加一个<script>元素，向服务器请求 JSON 数据，服务器收到请求后，将数据放在一个指定名字的回调函数里传回来。由于<script>元素的src属性不受同源策略的限制，所以这种做法不受跨域限制。

```go
func main() {
	r := gin.Default()

	r.GET("/JSONP", func(c *gin.Context) {
		data := map[string]interface{}{
			"foo": "bar",
		}
		
		// /JSONP?callback=x
		// 将输出：x({\"foo\":\"bar\"})
		c.JSONP(http.StatusOK, data)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

在代码中，`c.JSONP(http.StatusOK, data)`表示返回一个 JSONP 格式的响应，其中`http.StatusOK`表示状态码为 200，`data`是要返回的数据。当客户端请求`/JSONP?callback=x`时，将输出：`x({\"foo\":\"bar\"})`，其中x是回调函数的名字，`{\"foo\":\"bar\"}`是要返回的 JSON 数据。

### Multipart/Urlencoded 绑定

```go
package main

import (
	"github.com/gin-gonic/gin"
)

type LoginForm struct {
	User     string `form:"user" binding:"required"`
	Password string `form:"password" binding:"required"`
}

func main() {
	router := gin.Default()
	router.POST("/login", func(c *gin.Context) {
		// 你可以使用显式绑定声明绑定 multipart form：
		// c.ShouldBindWith(&form, binding.Form)
		// 或者简单地使用 ShouldBind 方法自动绑定：
		var form LoginForm
		// 在这种情况下，将自动选择合适的绑定
		if c.ShouldBind(&form) == nil {
			if form.User == "user" && form.Password == "password" {
				c.JSON(200, gin.H{"status": "you are logged in"})
			} else {
				c.JSON(401, gin.H{"status": "unauthorized"})
			}
		}
	})
	router.Run(":8080")
}
```

在 Gin 框架中，可以使用 `ShouldBind `方法将请求中的数据绑定到一个结构体中。在这个例子中，使用了一个 `LoginForm `结构体来存储请求中的` user` 和 `password `参数。这个结构体中的字段使用了` form `标签来指定对应的请求参数名，使用了 `binding `标签来指定这个参数是必须的。在 main函数中，使用了 `c.ShouldBind(&form)`将请求中的数据绑定到 `form` 变量中。如果绑定成功，就可以通过 `form.User`和 `form.Password`来获取请求中的参数值。如果请求中的参数值符合预期，就可以返回一个 JSON 格式的响应。

### Multipart/Urlencoded 表单

```go
func main() {
	router := gin.Default()

	router.POST("/form_post", func(c *gin.Context) {
		message := c.PostForm("message")
		nick := c.DefaultPostForm("nick", "anonymous")

		c.JSON(200, gin.H{
			"status":  "posted",
			"message": message,
			"nick":    nick,
		})
	})
	router.Run(":8080")
}
```

gin框架中的`PostForm`方法，用于获取`POST`请求中的表单数据。在这个例子中，`message`和`nick`都是表单中的字段名，其中`message`是必填项，而`nick`是可选项，如果没有填写则默认为`anonymous`。

### Query 和 post form

```go
func main() {
	router := gin.Default()

	router.POST("/post", func(c *gin.Context) {

		id := c.Query("id")
		page := c.DefaultQuery("page", "0")
		name := c.PostForm("name")
		message := c.PostForm("message")

		fmt.Printf("id: %s; page: %s; name: %s; message: %s", id, page, name, message)
	})
	router.Run(":8080")
}
```

- `c.Query("id")`用于获取HTTP请求中的query参数id的值
- `c.DefaultQuery("page", "0")`用于获取HTTP请求中的query参数page的值，如果该参数不存在则返回默认值0
- `c.PostForm("name") `用于获取HTTP请求中的form参数name的值
- `c.PostForm("message")`用于获取HTTP请求中的form参数message的值

其中，query参数是在URL中的参数，例如`http://localhost:8080/post?id=123&page=1`，而form参数是在请求体中的参数，例如表单提交。

### SecureJSON

[json劫持](https://cloud.tencent.com/developer/article/1554938)。JSON 劫持的实现原理是攻击者构造一个 JSON 数据，然后通过 script 标签的 src 属性，将 JSON 数据作为参数传递给一个服务端的接口，服务端返回一个 JSON 数据，攻击者可以通过将一个JSON数组或对象作为响应返回给受害者的网站，并在其中嵌入一个构造函数，来劫持受害者网站的数据。当受害者网站将这个JSON数组或对象作为响应返回给前端时，前端的浏览器会自动执行这个构造函数，从而将数据发送到攻击者的服务器中。

使用 SecureJSON 防止 json 劫持。

```go
func main() {
	r := gin.Default()

	// 你也可以使用自己的 SecureJSON 前缀
	// r.SecureJsonPrefix(")]}',\n")

	r.GET("/someJSON", func(c *gin.Context) {
		names := []string{"lena", "austin", "foo"}

		// 将输出：while(1);["lena","austin","foo"]
		c.SecureJSON(http.StatusOK, names)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

### 优雅的重启和关闭

```go
router := gin.Default()
router.GET("/", handler)
// [...]
endless.ListenAndServe(":4242", router)
```

优雅通常指的是一种优雅的退出或重启方式，即在退出或重启时，能够保证已有的请求能够处理完毕，而不是直接强制退出或重启。在web服务中，优雅的退出或重启方式可以避免因为强制退出或重启而导致的请求失败或数据丢失等问题。

这里使用的endless库是一个封装了http.Server的库，它的原理是在启动http.Server之前，先创建一个stopChan和doneChan，然后在启动http.Server之后，使用goroutine来监听os.Signal，当接收到os.Interrupt或os.Kill信号时，向stopChan发送信号，然后等待doneChan接收到信号后，关闭http.Server。在这个过程中，endless库会不断地接收新的请求，直到stopChan接收到信号后，停止接收新的请求，等待已有的请求处理完成后，关闭http.Server。这样就实现了优雅的重启和停止。

### 记录日志

```go
func main() {
    // 禁用控制台颜色，将日志写入文件时不需要控制台颜色。
    gin.DisableConsoleColor()

    // 记录到文件。
    f, _ := os.Create("gin.log")
    gin.DefaultWriter = io.MultiWriter(f)

    // 如果需要同时将日志写入文件和控制台，请使用以下代码。
    // gin.DefaultWriter = io.MultiWriter(f, os.Stdout)

    router := gin.Default()
    router.GET("/ping", func(c *gin.Context) {
        c.String(200, "pong")
    })

    router.Run(":8080")
}
```

gin.DefaultWriter = io.MultiWriter(f)将日志写入文件中。

### 自定义中间件

```go
func Logger() gin.HandlerFunc {
	return func(c *gin.Context) {
		t := time.Now()

		// 设置 example 变量
		c.Set("example", "12345")

		// 请求前

		c.Next()

		// 请求后
		latency := time.Since(t)
		log.Print(latency)

		// 获取发送的 status
		status := c.Writer.Status()
		log.Println(status)
	}
}

func main() {
	r := gin.New()
	r.Use(Logger())

	r.GET("/test", func(c *gin.Context) {
		example := c.MustGet("example").(string)

		// 打印："12345"
		log.Println(example)
	})

	// 监听并在 0.0.0.0:8080 上启动服务
	r.Run(":8080")
}
```

定义一个返回gin.HandlerFunc类型的函数，然后在路由处理函数中使用r.Use()方法将其注册到gin引擎中。

### cookie

```go
func main() {

    router := gin.Default()

    router.GET("/cookie", func(c *gin.Context) {

        cookie, err := c.Cookie("gin_cookie")

        if err != nil {
            cookie = "NotSet"
            c.SetCookie("gin_cookie", "test", 3600, "/", "localhost", false, true)
        }

        fmt.Printf("Cookie value: %s \n", cookie)
    })

    router.Run()
}
```

`c.SetCookie`是一个用于设置cookie的方法，它是Gin框架中的一部分。在这个代码示例中，

c.SetCookie 用于设置一个名为 "gin_cookie" 的cookie。这是它的参数列表：

```go
c.SetCookie(name string, value string, maxAge int, path string, domain string, secure bool, httpOnly bool)
```

- name: "gin_cookie"，cookie的名称。
- value: "test"，cookie的值。
- maxAge: 3600，cookie的最大存活时间（以秒为单位）。
- path: "/"，cookie的路径。
- domain: "localhost"，cookie的域名。
- secure: false，表示这个cookie是否仅在HTTPS连接上发送。
- httpOnly: true，表示这个cookie是否仅能通过HTTP(S)请求访问，而不能通过客户端脚本（如JavaScript）访问。

### 重定向

```go
//
r.GET("/test", func(c *gin.Context) {
	c.Redirect(http.StatusMovedPermanently, "http://www.google.com/")
})
//post重定向
r.POST("/test", func(c *gin.Context) {
	c.Redirect(http.StatusFound, "/foo")
})
//路由重定向
r.GET("/test", func(c *gin.Context) {
    c.Request.URL.Path = "/test2"
    r.HandleContext(c)
})
r.GET("/test2", func(c *gin.Context) {
    c.JSON(200, gin.H{"hello": "world"})
})
```

