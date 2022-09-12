# GeeNote
【笔记】从零实现框架Gee

第一次看这种开源项目的源码，学习它的思路和写法，感觉还是收获颇多的。对我而言，Gee-web 让我理解了Web 框架的工作原理，学习了如何通过引入上下文和中间件实现框架的接口和扩展。一些语言使用方式，例如 Go test等，也了解到怎么将项目代码专业化。
对整个项目印象最为深刻的还是接口型函数的广泛使用，它的使用更加灵活，可读性也更好，方便传入函数作为参数。Handler 就是一个典型的接口型函数，而 HandleFunc 则是能够将普通的函数类型/结构体进行转换。
