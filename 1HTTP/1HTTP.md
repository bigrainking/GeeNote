

[toc]

> 创建我们自己的实例 ListenAndServe的第二个参数Handler
>
> 拦截所有HTTP请求 填入路由表
>
> 实例创建路由表 key=pattern+method value=逻辑处理
>
> Get/POST()用户绑定pattern与逻辑处理函数

# 一、基础介绍

## 1 Go原生代码实现HTTP服务

1. 我们设置了两个路由器 indexHandler helloHandler 分别绑定在 路径 /  /hello上。

​	根据不同的请求， 将会调用不同的路由

​	HandleFunc(路径， handler)实现了路由和Handler的映射

2. `http.ListenAndServe(":8080", nil)` 中的第二个参数代表了处理 端口8080所有请求的 一个实例， 而设置为 nil表示当前使用标准库中的默认实例

   也就是说： 传入了任何请求，都交给这个默认实例处理

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

func main() {
	// 注册hander到Addr
	http.HandleFunc("/", indexHandler)
	http.HandleFunc("/hello", helloHandler)
	// 监听端口，并启动服务
	log.Fatal(http.ListenAndServe(":8080", nil))
}

// 自定义handler处理器
func indexHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "url = %q", req.URL.Path)
}

func helloHandler(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "url = %q", req.URL.Path)
}
```

运行

```shell
 curl http://localhost:8080/
url = "/"%                                                                                   
 curl http://localhost:8080/hello
url = "/hello"%
```



## 2 自己构造一个简单的hander

> 上面的 `http.ListenAndServe(":8080", nil)` 第二个参数就是我们框架的入手点。
>
> 如果我们自己实现了一个实例，那么所有请求将会走我们设计的实例



**1. 第二个参数是什么呢？**

从下面可以看到是Handler接口， 只要我们实现了这个接口，就实现了一个实例，所有的request就会走这个实例

```go
package http

type Handler interface {
    ServeHTTP(w ResponseWriter, r *Request)
}

func ListenAndServe(address string, h Handler) error
```



**2. 实现一个我们自己的实例**

1. 下面实现了一个实例，通过实现Handler接口。 定义Engine结构体，实现函数ServeHTTP()

2. **框架的第一步** ： 将所有请求引入到我们自己的处理逻辑中。 

   之前的HandleFunc()只能针对具体的路由写处理逻辑

   通过创建下面的Engine实例，拦截了所有的HTTP请求(从下面可以看到/, /hello都使用这个engine实例)

```go
package main

import (
	"fmt"
	"log"
	"net/http"
)

// 自定义实例，实现第二个参数的hander，让所有request进入该实例

// 结构体对象
type Engine struct{}

// 实现ServerHTTP
func (e *Engine) ServeHTTP(w http.ResponseWriter, req *http.Request) {
	switch req.URL.Path {
	case "/":
		fmt.Fprintf(w, "url = %q", req.URL.Path)
	case "/hello":
		fmt.Fprintf(w, "url = %q", req.URL.Path)
	default:
		fmt.Fprintf(w, "404 not found!")
	}
}

func main() {
	engine := new(Engine) //创建一个实例对象
	log.Fatal(http.ListenAndServe(":8080", engine))
}
```

测试：

```shell
 curl http://localhost:8080/
url = "/"%                                                                                
 curl http://localhost:8080/hello
url = "/hello"%
```





# 二、实现Gee框架的HTTP服务

## 2.1 整体架构

```shell
.
├── gee #Gee框架
│   ├── gee.go
│   └── go.mod
├── go.mod
└── main.go
```

## 2.2 Main.go使用Gee框架



```go
package main

// 使用New()创建 gee 的实例，使用 GET()方法添加路由，最后使用Run()启动Web服务。
// 这里的路由，只是静态路由，不支持/hello/:name这样的动态路由，动态路由我们将在下一次实现。

import (
	"fmt"
	"log"
	"net/http"

	"Gee/day1-httpbase/base3/gee"
)

func main() {
	// 创建一个Gee实例
	r := gee.New()
	// Get方法添加路由
	r.GET("/", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})
	r.GET("/hello", func(w http.ResponseWriter, req *http.Request) {
		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
	})

	// 启动服务器
	log.Fatal(r.Run(":9999"))
}

```



## 2.3 Gee框架

创建一个处理所有请求的实例

1. **HandlerFunc()** 提供给框架用户的路由映射的处理方法，用户实现函数，自定义逻辑处理

   `type HandlerFunc func(w http.ResponseWriter, r *http.Request)`

2. 在实例 `Engine`中创建了一张**路由表 `Router`** ，将用户自定义的路由与请求路由映射起来

   Key是请求路由+请求方法 ： "GET-/" "GET-/hello"

   Value是逻辑处理代码 ： 

   ```go
   //func的类型对应HandlerFunc
   func(w http.ResponseWriter, req *http.Request) {
   		fmt.Fprintf(w, "URL.Path = %q\n", req.URL.Path)
   	}
   ```

3. **用户绑定路由** ：

   提供 `(*Engine) GET(patter *string*, handler HandlerFunc)` 函数， 将逻辑处理函数与路径 请求方法绑定，注册到 Engine实例中

4. **启动监听 Run()**:

   gee.Run() 是对http.ListenAndServe()的封装， 其中第二个参数Engine，实现的ServeHTTP功能是：

   根据用户请求的*http.Request，在Engine的路由表中寻找是否有对应的value=>HanderFunc

   - 如果找到对应的HandlerFunc,则调用HandlerFunc将req,w传入， 进行处理
   - 没有找到对应的路由器，则返回 404





```go
package gee

// 目的：实现一个Handler 处理所有通过端口的请求
import (
	"fmt"
	"net/http"
)

type HandlerFunc func(w http.ResponseWriter, r *http.Request)

// 1. 实例结构体 Engine实现ServeHTTP接口
type Engine struct {
	// 实例的路由表，存储所有路由映射
	router map[string]HandlerFunc //value是存储的路由，之后通过传入的r ,w 调用
}

// 构造函数
func New() *Engine {
	return &Engine{router: make(map[string]HandlerFunc)}
}

// 2. 实现ServeHTTP Func
func (engine *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	// 处理进来的request ： 类似switch case，这里借用路由表一个道理
	key := r.Method + "-" + r.URL.Path
	// 如果该请求路径注册过路由，则执行处理函数
	if HandlerFunc, ok := engine.router[key]; ok {
		HandlerFunc(w, r)
	} else {
		fmt.Fprint(w, "404 NOT FOUNT!")
	}
}

// 3. 封装ListenAndServe():
// 让默认走用户自定义的Engine
func (engine *Engine) Run(addr string) error {
	return http.ListenAndServe(addr, engine)
}

// 4. 提供给用户的路由映射处理函数：绑定patter和路由
// 类似HandleFunc：绑定pattern 路由
func (engine *Engine) GET(pattern string, handlerfunc HandlerFunc) {
	// 绑定方法是：将HandlerFunc添加到engine的路由表中
	engine.addRouter("GET", pattern, handlerfunc)
}
func (engine *Engine) POST(pattern string, handlerfunc HandlerFunc) {
	// 绑定方法是：将HandlerFunc添加到engine的路由表中
	engine.addRouter("POST", pattern, handlerfunc)
}
//添加路由表内容
func (engine *Engine) addRouter(method, pattern string, handlerfunc HandlerFunc) {
	key := method + "-" + pattern
	engine.router[key] = handlerfunc
}
```





# 三、参考文献

[极客兔兔](https://geektutu.com/post/gee-day1.html#%E6%A0%87%E5%87%86%E5%BA%93%E5%90%AF%E5%8A%A8Web%E6%9C%8D%E5%8A%A1)













