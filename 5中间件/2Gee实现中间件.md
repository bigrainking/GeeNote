

# 一、设计

上一节已经大概了解了如何实现，下面说明一些细节问题：

- 之前所有中间件是放在Context中的，实际上，可以注册到Engine这个最大RouterGroup中，也可以注册到下面的小RouterGroup中（/v1等等）

  > 但是由于作用在小的RouterGroup中还不如直接在RouterGroup对应handler中调用函数更加直接，因此，中间件仅仅作用在最大的Engine这个RouterGroup上

- 如何将中间件注册到Context.handlers中？

  

# 二、实现

## 1. 中间件组件的写法

中间件的通用模式如下面的Log()

#### Log.go

> 设计Gee框架中默认的一个中间件：
>
> 开发人员自己定义的中间件格式都应该是
>
> ```go
> func Log() HandlerFunc {
> 	return func(c *Context) {
>         // Next前要执行的内容...
> 		c.Next()
> 	// Next后要执行的...
>     }
> }
> ```

**Log.go**

```go
// 这是一个中间件
func Log() HandlerFunc {
	return func(c *Context) {
		// Next前
		t := time.Now()

		c.Next()

		// Next后
		log.Printf("[%d] %s in %v", c.StatusCode, c.Req.RequestURI, time.Since(t))
	}
}
```



## 2. gee.go

#### 1. 中间件注册函数

1. 【注册中间件】外部调用(*RouterGroup).Use() : 添加所有的中间件到 RouterGroup.middlewares

#### 2. Req对应所有中间件写入到Context.handlers中

1. 【添加到Context】 ServeHTTP(w, Req) : **匹配Req所在的RouterGroup中的中间件**，并添加到会话Context.handlers中

> Req所在Group可能有多个，比如 /v1/admin/gee 在/，同时在 /v1/admin 这两个RouterGroup中，
>
> 因此下面遍历engine的所有group，并且一旦发现有相同前缀的group就append到c.handlers中

2. 【调用handle】： Req需要用到的中间件已经放置在Context中，因此调用handle(c) 执行

```go
func (e *Engine) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	c := newContext(w, r)
	// 找到r请求所在的group，将group里面的中间件都添加到Context.handlers中
	for _, group := range e.groups {
		// 通过比较RouterGroup与请求的前缀得到
		if strings.HasPrefix(r.URL.Path, group.prefix) {
			c.handlers = append(c.handlers, group.middlewares...) //将所有RouterGroup中的中间件拿出来
		}
	}
	e.router.handle(c)
}
```



## 3. Context.go

> Context的变化：
>
> - 属性
>
>   - handlers []HandlerFunc:存储中间件组件and路由函数
>
>   - index ： handlers的索引，初始值为-1
>
> - 功能函数：
>
>   - Next():实现按序执行中间件和路由函数的主要函数



```go
type Context struct {
	....
	// 中间件middleware
	handlers []HandlerFunc //指针类型不能调用函数，因此存储的是HandlerFunc，以便执行HandlerFunc(c)
	index    int
}

func (c *Context) Next() {
	c.index++
	fmt.Println(c.index)
	for c.index < len(c.handlers) {
		c.handlers[c.index](c)
		c.index++
	}
}
```



**答疑时间：**

- 1. handlers []HandlerFunc //为啥不是指针捏handlers []*HandlerFunc

  - 因为在Next需要调用函数
  - `c.handlers[c.index](c)` c.handlers[index]必须存储的HandlerFunc(Context)函数类型
  - 指针类型无法执行函数

## 4. Router.go

ServeHTTP()中调用handle执行业务函数：

1. 【将业务函数插入到Context.handlers一起】路由查询找到Req对应handlerFunc并插入到Context中
2. 【执行c.Next()执行所有中间件与业务函数】调用c.Next()

上述过程与上一节说到的H`andleHTTPReq`函数作用一样

```go
func (r *router) handle(c *Context) {
	if node, params := r.getRoute(c.Method, c.Path); node == nil {
		c.String(http.StatusNotFound, "404 Note FOUND!")
	} else {
		c.Params = params 
		key := c.Method + "-" + node.pattern
		

		// 中间件执行部分
		// 1. 将业务函数添加到handlers中
		c.handlers = append(c.handlers, r.handlers[key])
		// 2. 调用Next ： 就应该放在这里，如果前缀树中找到了Req才执行c.Next
		c.Next()
	}
}
```





