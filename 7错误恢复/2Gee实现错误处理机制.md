



# Gee实现错误处理机制



## 错误处理要做什么？

当出现空指针异常、数组越界等如果没有错误处理可能导致系统宕机：panic终止导致系统运行。

**错误处理要实现的事情：**

1. 对panic进行异常捕捉，将跟踪panic的信息记录在日志里面，方便定位

2. 向用户发送Internal Server Error的信息

## 错误处理的实现



### Recovery中间件

通过中间件Recover实现

- 在执行handler之前，执行recover里面的内容
- recover捕获handler运行过程中的错误
- 并打印错误出现的各个func





1. 下面在执行handler之后添加了defer func(){ recover...}, 防止panic导致程序直接终止。

```go
func Recovery() HandlerFunc {
	return func(c *Context) {
		defer func() {
			if err := recover(); err != nil {
				message := fmt.Sprintf("%s", err)
				log.Printf("%s\n\n", trace(message))
				c.Fail(http.StatusInternalServerError, "Internal Server Error")
			}
		}()

		c.Next()
	}
}
```



2. `trace()` 追踪panic出现的function

```go
// print stack trace for debug
func trace(message string) string {
	var pcs [32]uintptr
	n := runtime.Callers(3, pcs[:]) // skip first 3 caller

	var str strings.Builder
	str.WriteString(message + "\nTraceback:")
	for _, pc := range pcs[:n] {
		fn := runtime.FuncForPC(pc)
		file, line := fn.FileLine(pc)
		str.WriteString(fmt.Sprintf("\n\t%s:%d", file, line))
	}
	return str.String()
}
```





### 统一注册所有中间件

在创建Engine前时就注册好recovery中间件等所有需要的全局中间件

- 如Log，Recovery等等

```go
func Default() *Engine {
	engine := New()
	engine.Use(Logger(), Recovery())
	return engine
}
```



