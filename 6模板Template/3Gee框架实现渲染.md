[toc]

# 一、渲染

> 主要分为以下三步：
>
> 1. 设置自定义函数
> 2. 加载所有模板到Engine
> 3. 渲染模板



### 1. 使用效果

```go
// 1. 设置自定义函数
r.SetFuncMap(template.FuncMap{
		"FormatAsDate": FormatAsDate,
	})
//2. 模板加载：将template/下所有模板文件加载
r.LoadHTMLGlob("templates/*") //当前问价路径是main.go所在文件夹

//3. 渲染模板
r.GET("/", func(c *gee.Context) {
    c.HTML(http.StatusOK, "css.html", nil) //输出HTML类型的输出
})
```



> engine中添加了2个模板属性，funcMap用来存储自定义函数， htmlTemplate存储解析后的模板
>
> ```go
> type Engine struct {
>     ....
>     funcMap template.FuncMap
>     htmlTemplate *template.Template
> }
> ```

### 2. 模板自定义函数

在main中设置好自定义函数，注册到模板中



所有自定义函数临时存储在engine.funcMap中， 之后在模板加载时再用

```go
func (e *Engine) SetFuncMap(funcMap template.FuncMap) {
	e.funcMap = funcMap
}
```

main.go中的自定义函数

```go
func FormatAsDate(t time.Time) string {
	year, month, day := t.Date()
	return fmt.Sprintf("%d-%02d-%02d", year, month, day)
}
```



### 3. 模板加载

将所有模板加载好，存储在engine.htmlTemplate变量中， 之后如果需要某个模板直接从中取出来。

- 注意要先Funcs加载自定义函数， 再解析模板

```go
// 【2. 加载所有模板到engine】
// 加载|所有|模板到engine以便后面Context渲染到io.Writer
func (e *Engine) LoadHTMLGlob(pattern string) { //传入模板对应文件路径
	// 1. 注册自定义函数 2. 解析模板到htmlTemplate
	e.htmlTemplates = template.Must(template.New("").Funcs(e.funcMap).ParseGlob(pattern))
}
```



### 4. 模板渲染

指定需要加载的模板名字， 从engine.htmlTemplate中获取模板进行渲染。



1. **Context添加engine属性， 以便获取engine.htmlTemplate**

```go
type Context struct {
    .....
    engine *Engine 
}
```

2. **模板渲染**

渲染指定的模板： `c.engine.htmlTemplates.ExecuteTemplate(c.Writer, name, data)`

```go
func (c *Context) HTML(code int, name string, data interface{}) {
	c.SetHeader("Content-Type", "text/html")
	c.StatusCode = code

	// 将data数据传递给模板，渲染模板到c.Writer
	if err := c.engine.htmlTemplates.ExecuteTemplate(c.Writer, name, data); err != nil {
		c.Fail(500, err.Error())
	}
}
```











# 二、静态服务器

**1）要做的事情**

​	原生http库已经实现了找到文件后，如何将文件返回。因此，Gee框架的工作就是，**解析请求，将请求URL映射到文件**



**2）文件服务器的流程**

1. 解析请求：请求/v1/index/file.js
2. 映射到文件：找到这个地址对应的文件
3. 将文件返回给客户端

**Gee框架只需要执行前两部**，最后一步net/http 已经完成了



### 效果

> 将请求prefix中的`/assets`(参数一)替换成`/usr/geektutu/blog/static`（参数二）

​	用户访问`localhost:9999/assets/js/geektutu.js`，最终返21回`/usr/geektutu/blog/static/js/geektutu.js`

```go
r := gee.New()
r.Static("/assets", "/usr/geektutu/blog/static")
// 或相对路径 r.Static("/assets", "./static")
r.Run(":9999")
```



### 实现

实现路径映射到对应的静态文件，

```go
// 【静态服务器】
// 将请求URL与实际文件路径联系起来
// @relatetivePath:是请求URL路径，需要注册对应handler在路由表中
// @root：是文件实际在文件夹中的位置
func (group *RouterGroup) Static(relativePath, root string) {
	// 创建fileServer
	handler := group.createStaticHandler(relativePath, http.Dir(root))
	// 注册请求URL的路径
	reqPath := relativePath + "/*filepath"
	group.GET(reqPath, handler) //没有加分组前缀是因为GET会加上
}

// 创建一个fileServer的handler
func (group *RouterGroup) createStaticHandler(relativePath string, fs http.FileSystem) HandlerFunc {
	// 构建fileServer
	// 需要去掉请求路径前缀，比如请求的是/static，实际上注册的是 /v1/static/....
	fileServer := http.StripPrefix(group.prefix+relativePath, http.FileServer(fs))
	// 返回一个handler
	return func(c *Context) {
		// 测试文件是否能够打开
		filePath := c.Param("filepath")
		if _, err := fs.Open(filePath); err != nil {

		}
		fileServer.ServeHTTP(c.Writer, c.Req)
	}
}
```



