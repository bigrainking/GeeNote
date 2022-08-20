

[toc]

>1. 将Router独立出来，方便之后添加内容
>2. Context封装Request Response，添加会话各个属性的快速访问

# Main.go

下面main函数中可以看到构造响应时候非常简单：

下面有三种响应： JSON、String、HTML

```go
import "Gee/day2-context/gee" //其余import被省略
func main() {
	r := gee.New()
	r.GET("/", func(c *gee.Context) {
		c.HTML(http.StatusOK, "<h1>Hello Gee</h1>")
	})
	r.GET("/hello", func(c *gee.Context) {
		// expect /hello?name=geektutu
		c.String(http.StatusOK, "hello %s, you're at %s\n", c.Query("name"), c.Path, c.Method)
	})

	r.POST("/login", func(c *gee.Context) {
		// 构造返回的JSON数据
		c.JSON(http.StatusOK, gee.H{
			// 第一条数据是 "username":获取request中上传的username ： 通过本次会话的Context获取
			"username": c.PostForm("username"), //c.PostForm("username")获取请求request中的username
			"password": c.PostForm("password"),
		})
	})

	r.Run(":9999")
}
```





# Context.go具体实现

**Context实现了**

1. 对request.Method req.Path的快速访问， (目前只包含了这两个属性request、ResponseWriter) ： 对外函数

2. 获取Query，PostForm参数 ： 对外函数

3. 快速构造JSON、String、HTML、DATA响应的函数 ： 对外函数

   为了方便构造JSON响应更简洁，创建了 `type H [string]interface`

```go
```



# Router路由器

从上一节的gee.go中将Router完全抽离出来, 为以后添加

Router包含了下面2个函数：

- addRouter(method, pattern, HandlerFunc)添加路由条目：将HandlerFun - method+Pattern 的keyvalue条目添加到路由表中
- handle(c *Context)路由查询： 对进来的请求查找路由表，如果有则直接调用

```go
package gee

// 将router单独拎出来
import (
	"net/http"
)

// 路由器
type Router struct {
	handlers map[string]HandlerFunc
}

// 构造函数：创建一个Router
func newRouter() *Router { //供内部使用需要小写
	return &Router{handlers: make(map[string]HandlerFunc)}
}

func (router *Router) addRouter(method, pattern string, handlerfunc HandlerFunc) {
	key := method + "-" + pattern
	router.handlers[key] = handlerfunc
}

// 提供给ServeHTTP的路由选择函数
func (router *Router) Handle(c *Context) {
	// 处理进来的request ： 类似switch case，这里借用路由表一个道理
	key := c.Method + "-" + c.Path
	// 如果该请求路径注册过路由，则执行处理函数
	if HandlerFunc, ok := router.handlers[key]; ok {
		HandlerFunc(c)
	} else {
		// 构造返回内容
		c.String(http.StatusNotFound, "404 NOT FOUND！")
	}
}
```



# Gee.go框架入口

Router被抽离出去之后，所有的函数变为了对Router封装的调用

**创建实例**

- Engine：由于Router被抽离出去，路由表直接引用Router

**对外函数**

- GET(pattern, HandlerFunc(*Context)) : 原来的request Response变为Context（原来HandlerFunc重新定义）
- POST同上

- ServeHTTP(ResponseWriter, Requets):实现Handler接口的ServeHTTP函数。 因此参数不会变
  - 【创建context】request进来的地方，产生请求就创建Context对象
  - 调用路由选择函数： 在Router中封装的handler根据请求调用对应的HanderFunc



```go
package gee

// Gee改造ServeHTTP
import (
	"net/http"
)

type HandlerFunc func(c *Context)

// 1. 实例结构体 Engine实现ServeHTTP接口
type Engine struct {
	// 实例的路由表，存储所有路由映射
	router *Router //value是存储的路由，之后通过传入的r ,w 调用
}

// 构造函数
func New() *Engine {
	return &Engine{router: newRouter()}
}

// 2. 实现ServeHTTP Func
func (engine *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// 构造Context对象
	c := newContext(w, r)
	// 调用Router的路由器
	engine.router.Handle(c) //将当前会话的请求传入到router进行路由选择
}

func (engine *Engine) Run(addr string) error {
	return http.ListenAndServe(addr, engine)
}

func (engine *Engine) GET(pattern string, handlerfunc HandlerFunc) {
	// 绑定方法是：将HandlerFunc添加到engine的路由表中
	engine.router.addRouter("GET", pattern, handlerfunc)
}

func (engine *Engine) POST(pattern string, handlerfunc HandlerFunc) {
	engine.router.addRouter("POST", pattern, handlerfunc)
}
```



# 测试

```shell
  curl -i http://localhost:9999/
HTTP/1.1 200 OK
Content-Type: text/html
Date: Fri, 19 Aug 2022 13:51:53 GMT
Content-Length: 18

<h1>Hello Gee</h1>%                                                                          curl -i "http://localhost:9999/hello?name=geektutu"
HTTP/1.1 200 OK
Content-Type: text/plain
Date: Fri, 19 Aug 2022 13:52:21 GMT
Content-Length: 52

hello [geektutu /hello GET], you're at %!s(MISSING)
 curl -i "http://localhost:9999/login" -X POST -d 'username=geektutu&password=1234'
HTTP/1.1 200 OK
Content-Type: application/json
Date: Fri, 19 Aug 2022 13:52:42 GMT
Content-Length: 42

{"password":"1234","username":"geektutu"}
```

