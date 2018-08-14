View
=============

Iris支持开箱即用的5个模板引擎，开发人员仍然可以使用任何外部golang模板引擎，
因为`context/context#ResponseWriter()` 是 `io.Writer`。

这五个模板引擎都具有通用API的共同特征，如布局，模板功能，特定于派对的布局，部分渲染等。

- 标准的html，它的模板解析器就是 golang.org/pkg/html/template/
- Django，它的模板解析器就是 github.com/flosch/pongo2
- Pug(Jade), 它的模板解析器就是 github.com/Joker/jade
- Handlebars, 它的模板解析器就是 github.com/aymerick/raymond
- Amber,它的模板解析器就是 github.com/eknkc/amber

```go
// file: main.go
package main

import "github.com/kataras/iris"

func main() {
    app := iris.New()
    // Load all templates from the "./views" folder
    // where extension is ".html" and parse them
    // using the standard `html/template` package.
    app.RegisterView(iris.HTML("./views", ".html"))

    // Method:    GET
    // Resource:  http://localhost:8080
    app.Get("/", func(ctx iris.Context) {
        // Bind: {{.message}} with "Hello world!"
        ctx.ViewData("message", "Hello world!")
        // Render template file: ./views/hello.html
        ctx.View("hello.html")
    })

    // Method:    GET
    // Resource:  http://localhost:8080/user/42
    app.Get("/user/{id:long}", func(ctx iris.Context) {
        userID, _ := ctx.Params().GetInt64("id")
        ctx.Writef("User ID: %d", userID)
    })

    // Start the server using a network address.
    app.Run(iris.Addr(":8080"))
}
```

```html
<!-- file: ./views/hello.html -->
<html>
<head>
    <title>Hello Page</title>
</head>
<body>
    <h1>{{.message}}</h1>
</body>
</html>
```

**Template functions**

```go
package main

import "github.com/kataras/iris"

func main() {
    app := iris.New()

    // - standard html  | iris.HTML(...)
    // - django         | iris.Django(...)
    // - pug(jade)      | iris.Pug(...)
    // - handlebars     | iris.Handlebars(...)
    // - amber          | iris.Amber(...)
    tmpl := iris.HTML("./templates", ".html")

    // built'n template funcs are:
    //
    // - {{ urlpath "mynamedroute" "pathParameter_ifneeded" }}
    // - {{ render "header.html" }}
    // - {{ render_r "header.html" }} // partial relative path to current page
    // - {{ yield }}
    // - {{ current }}

    // register a custom template func.
    tmpl.AddFunc("greet", func(s string) string {
        return "Greetings " + s + "!"
    })

    // register the view engine to the views, this will load the templates.
    app.RegisterView(tmpl)

    app.Get("/", hi)

    // http://localhost:8080
    app.Run(iris.Addr(":8080"))
}

func hi(ctx iris.Context) {
    // render the template file "./templates/hi.html"
    ctx.View("hi.html")
}
```

```html
<!-- file: ./templates/hi.html -->
<b>{{greet "kataras"}}</b> <!-- will be rendered as: <b>Greetings kataras!</b> -->
```

**嵌套**

View引擎也支持捆绑（https://github.com/jteeuwen/go-bindata）模板文件,
`go-bindata`给你两个功能 `Assset` 和 `AssetNames`，这些可以使用`.Binary`函数设置到每个模板引擎。

```go
package main

import "github.com/kataras/iris"

func main() {
    app := iris.New()
    // $ go get -u github.com/jteeuwen/go-bindata/...
    // $ go-bindata ./templates/...
    // $ go build
    // $ ./embedding-templates-into-app
    // html files are not used, you can delete the folder and run the example
    app.RegisterView(iris.HTML("./templates", ".html").Binary(Asset, AssetNames))
    app.Get("/", hi)

    // http://localhost:8080
    app.Run(iris.Addr(":8080"))
}

type page struct {
    Title, Name string
}

func hi(ctx iris.Context) {
    //                      {{.Page.Title}} and {{Page.Name}}
    ctx.ViewData("Page", page{Title: "Hi Page", Name: "iris"})
    ctx.View("hi.html")
}
```

这里可以找到一个真实的例子： https://github.com/kataras/iris/tree/master/_examples/view/embedding-templates-into-app.

**Reload**

在每个请求上启用模板的自动重新加载。在开发人员处于开发模式时很有用，因为他们不需要在每个模板编辑时重新启动他们的应用程序。

```go
pugEngine := iris.Pug("./templates", ".jade")
pugEngine.Reload(true) // <--- set to true to re-build the templates on each request.
app.RegisterView(pugEngine)
```

**Examples**

- Overview
- Hi
- A simple Layout
- Layouts: `yield` and `render` tmpl funcs
- The `urlpath` tmpl func
- The `url` tmpl func
- Inject Data Between Handlers
- Embedding Templates Into App Executable File

您也可以通过使用`context#ResponseWriter`来提供quicktemplate文件，请查看iris/_examples/http_responsewriter/quicktemplate示例。