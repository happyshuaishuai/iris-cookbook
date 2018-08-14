# Hosts

**监听服务**

你可以启动服务监听任何类型的 `net.Listener` 甚至 `http.Server` 实例， 最后通过 `Run`方法初始化服务。

Go开发人员启动服务最常用方法是传递“hostname：ip”形式的网络地址。在iris里，我们使用 `iris.Addr` 这是一个 `iris.Runner` 类型。

```go
// 监听 TCP 的网络地址 0.0.0.0:8080
app.Run(iris.Addr(":8080"))
```

有时您在应用程序的其他位置创建了标准的 `net/http` 服务，并希望使用它来为 Iris Web 应用程序提供服务

```go
// 与之前相同，但使用自定义的 `http.Server`，也可能在其他地方使用
app.Run(iris.Server(&http.Server{Addr:":8080"}))
```

最高级的用法是创建自定义或标准的 `net.Listener` 并将其传递给 `app.Run`

```go
// 使用自定义的 net.Listener
l, err := net.Listen("tcp4", ":8080")
if err != nil {
    panic(err)
}
app.Run(iris.Listener(l))
```

一个更完整的示例，使用仅限unix的套接字文件功能

```go
package main

import (
    "os"
    "net"
    "github.com/kataras/iris"
)

func main() {
    app := iris.New()

    // UNIX socket
    if errOs := os.Remove(socketFile); errOs != nil && !os.IsNotExist(errOs) {
        app.Logger().Fatal(errOs)
    }

    l, err := net.Listen("unix", socketFile)

    if err != nil {
        app.Logger().Fatal(err)
    }

    if err = os.Chmod(socketFile, mode); err != nil {
        app.Logger().Fatal(err)
    }

    app.Run(iris.Listener(l))
}
```

UNIX和BSD主机可以优先考虑重用端口功能

```go
package main

import (
	/*
	 * Package tcplisten 提供各种定制的 TCP net.Listener
	 * 性能相关选项：
	 *     - SO_REUSEPORT 在多CP​​U服务器上此选项允许线性扩展服务器性能
	 *		 See https://www.nginx.com/blog/socket-sharding-nginx-release-1-9-1/ for details.
	 *
	 *	   - TCP_DEFER_ACCEPT This option expects the server reads from the accepted connection before writing to them.
	 *
	 *     - TCP_FASTOPEN. See https://lwn.net/Articles/508865/ for details.
	 */
	"github.com/valyala/tcplisten"
	"github.com/kataras/iris"
)

// go get github.com/valyala/tcplisten

func main() {
    app := iris.New()

    app.Get("/", func(ctx iris.Context) {
        ctx.HTML("<h1>Hello World!</h1>")
    })

    listenerCfg := tcplisten.Config{
        ReusePort:   true,
        DeferAccept: true,
        FastOpen:    true,
    }

    l, err := listenerCfg.NewListener("tcp", ":8080")
    if err != nil {
        app.Logger().Fatal(err)
    }

    app.Run(iris.Listener(l))
}
```

**HTTP/2 and Secure**

如果您已签名文件密钥，则可以基于这些认证密钥使用 `iris.TLS` 服务 `https`

```go
// TLS using files
app.Run(iris.TLS("127.0.0.1:443", "mycert.cert", "mykey.key"))
```

你的应用程序准备好生产时应该使用的方法是`iris.AutoTLS`，它可以通过https://letsencrypt.org免费提供自动认证启动安全服务器

```go
// Automatic TLS
app.Run(iris.AutoTLS(":443", "example.com", "admin@example.com"))
```

**Any iris.Runner**

有时候你需要一些非常特别的监听方式并不像 `net.Listener` 这类型的，你可以通过`iris.Raw`做到这一点，但你的为此方法负责。

```go
// Using any func() error,
// the responsibility of starting up a listener is up to you with this way,
// 为了简单起见，我们使用 net/http package 的函数 ListenAndServe 
app.Run(iris.Raw(&http.Server{Addr:":8080"}).ListenAndServe)
```

**Host 配置器**

上述的所有监听形式都接受最后一个可选参数 `func(*iris.Supervisor)` 这用于为通过这些函数传递的特定 hosts 添加配置程序。

例如，假设我们要添加一个在服务器关闭时触发的回调

```go
app.Run(iris.Addr(":8080", func(h *iris.Supervisor) {
    h.RegisterOnShutdown(func() {
        println("server terminated")
    })
}))
```

您甚至可以在`app.Run`方法之前执行此操作，但区别在于这些 hosts 配置程序将执行到您可能用于为Web应用程序提供服务的所有 hosts（通过app.NewHost我们会在一分钟内看到）

```go
app := iris.New()
app.ConfigureHost(func(h *iris.Supervisor) {
    h.RegisterOnShutdown(func() {
        println("server terminated")
    })
})
app.Run(iris.Addr(":8080"))
```

在`Run`方法之后，`Application＃Hosts`字段可以访问为您的应用程序提供服务的所有 hosts。

但最常见的情况是您可能需要在`app.Run`方法之前访问 hosts，有两种获取访问 host supervisor,的方法，请参阅下文。

我们已经看到了如何通过`app.Run`或`app.ConfigureHost`的第二个参数配置所有应用程序的 hosts。
还有一种方法更适合简单场景，使用`app.NewHost`创建新主机并使用其`Serve`或`Listen`功能之一通过`iris#Raw` Runner启动应用程序。

请注意，这种方式需要额外导入`net/http`包。

```go
h := app.NewHost(&http.Server{Addr:":8080"})
h.RegisterOnShutdown(func(){
    println("server terminated")
})

app.Run(iris.Raw(h.ListenAndServe))
```

**多Hosts**

您可以使用多个服务器为您的 Iris Web 应用程序提供服务，`iris.Router`与`net/http/Handler`功能兼容，
因为你可以理解，它可以在任何`net/http`服务器上使用，但是有一种更简单的方法，通过使用`app.NewHost`，
它是还会复制所有主机配置程序，并关闭`app.Shutdown`上附加到特定Web应用程序的所有主机。

```go
app := iris.New()
app.Get("/", indexHandler)

// 在不同的goroutine中运行，以便不阻塞主要的“goroutine”。
go app.Run(iris.Addr(":8080"))
// 启动第二个服务器，它正在监听tcp 0.0.0.0:9090，
// 没有“go”关键字，因为我们想要在最后一次服务器运行时阻止。
app.NewHost(&http.Server{Addr:":9090"}).ListenAndServe()
```

**优雅的关机**

让我们继续学习如何捕获CONTROL+C/COMMAND+C或unix kill命令并优雅地关闭服务器。
	- 正确关闭CONTROL+C/COMMAND+C或者当发送的kill命令是ENABLED BY-DEFAULT时。

为了手动管理应用程序中断时要执行的操作，我们必须使用选项WithoutInterruptHandler禁用默认行为并注册新的中断处理程序（全局，跨所有可能的主机）。

```go
package main

import (
    "context"
    "time"

    "github.com/kataras/iris"
)


func main() {
    app := iris.New()

    iris.RegisterOnInterrupt(func() {
        timeout := 5 * time.Second
        ctx, cancel := context.WithTimeout(context.Background(), timeout)
        defer cancel()
        // close all hosts
        app.Shutdown(ctx)
    })

    app.Get("/", func(ctx iris.Context) {
        ctx.HTML(" <h1>hi, I just exist in order to see if the server is closed</h1>")
    })

    app.Run(iris.Addr(":8080"), iris.WithoutInterruptHandler)
}
```