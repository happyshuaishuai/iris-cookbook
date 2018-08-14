MVC
=============

Iris拥有一流的MVC（模型视图控制器）模式，你不会在Go世界的任何其他地方找到这些东西，

Iris Web框架以最快的执行速度支持Request数据，模型，持久性数据和绑定。

**特点**

支持所有的HTTP方法，例如，如果是GET请求，那么控制器应该有一个名为`Get()`的函数,
在同一个控制器里你可以定义多个方法来代表多种请求。

通过BeforeActivation自定义事件回调，每个控制器，将自定义 controller's struct's 方法作为具有自定义路径的处理程序（即使使用正则表达式参数化路径）

```go
import (
    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
)

func main() {
    app := iris.New()
    mvc.Configure(app.Party("/root"), myMVC)
    app.Run(iris.Addr(":8080"))
}

func myMVC(app *mvc.Application) {
    // app.Register(...)
    // app.Router.Use/UseGlobal/Done(...)
    app.Handle(new(MyController))
}

type MyController struct {}

func (m *MyController) BeforeActivation(b mvc.BeforeActivation) {
    // b.Dependencies().Add/Remove
    // b.Router().Use/UseGlobal/Done // and any standard API call you already know

    // 1-> Method
    // 2-> Path
    // 3-> The controller's function name to be parsed as handler
    // 4-> Any handlers that should run before the MyCustomHandler
    b.Handle("GET", "/something/{id:long}", "MyCustomHandler", anyMiddleware...)
}

// GET: http://localhost:8080/root
func (m *MyController) Get() string { return "Hey" }

// GET: http://localhost:8080/root/something/{id:long}
func (m *MyController) MyCustomHandler(id int64) string { return "MyCustomHandler says Hey" }
```

通过定义服务的依赖关系在你的 controller struc 内部持久化数据（请求之间共享数据）或者`Singleton`控制范围。

共享控制器之间的依赖关系或在父MVC应用程序上注册它们，并能够在Controller内的 `BeforeActivation` 可选事件回调中修改每个控制器的依赖关系，

```go
func(c *MyController) BeforeActivation(b mvc.BeforeActivation) {
	b.Dependencies().Add/Remove(...) 
}
```

作为控制器的字段访问 `Context` i.e `Ctx iris.Context`或通过方法传参，i.e `func(ctx iris.Context, otherArguments...)`

Controller struct 中的 Models（在方法中设置并有视图呈现），在同一请求生命周期中，您可以在控制器的方法中返回 models，
在请求生命周期中设置一个字段，并将该字段返回给另一个方法。

MVC有自己的`Router`，它是 `iris/router.Party` 类型，标准的 iris api，`Controllers`可以被注册到任何 `Party`,
包括子域名，Party's 按预期的开始和完成 handlers 工作着。

可选项 `BeginRequest(ctx)` 在方法执行之前上演初始化操作，用于调用中间件或许多方法使用相同的数据集合。

可选项 `EndRequest(ctx)` 在执行方法之后执行终止操作。

继承，递归`mvc.SessionController`，`Session `、`*sessions.Session`和`Manager *sessions.Sessions`作为其 `BeginRequest`填充的嵌入字段。
这只是个栗子，你可以使用通过管理器的`Start`返回的`sessions.Session`作为MVC应用程序的动态依赖。

`mvcApp.Register(sessions.New(sessions.Config{Cookie: "iris_session_id"}).Start)`

通过控制器方法的输入参数访问动态路径参数，不需要绑定，当您使用Iris的默认语法来解析来自控制器的处理程序时，您需要使用`By`来为方法添加后缀,
区分大小写。

`mvc.New(app.Party("/user")).Handle(new(user.Controller))`

- `func(*Controller) Get() - GET:/user`
- `func(*Controller) Post() - POST:/user`
- `func(*Controller) GetLogin() - GET:/user/login`
- `func(*Controller) PostLogin() - POST:/user/login`
- `func(*Controller) GetProfileFollowers() - GET:/user/profile/followers`
- `func(*Controller) PostProfileFollowers() - POST:/user/profile/followers`
- `func(*Controller) GetBy(id int64) - GET:/user/{param:long}`
- `func(*Controller) PostBy(id int64) - POST:/user/{param:long}`

`mvc.New(app.Party("/profile")).Handle(new(profile.Controller))`

- `func(*Controller) GetBy(username string) - GET:/profile/{param:string}`

`mvc.New(app.Party("/assets")).Handle(new(file.Controller))`

- `func(*Controller) GetByWildard(path string) - GET:/assets/{param:path}`

接收器支持的类型 `int`、`int64`、`bool`、`string`

通过输出参数响应，可选 i.e

```go
func(c *ExampleController) Get() string |
                                (string, string) |
                                (string, int) |
                                int |
                                (int, string) |
                                (string, error) |
                                error |
                                (int, error) |
                                (any, bool) |
                                (customStruct, error) |
                                customStruct |
                                (customStruct, int) |
                                (customStruct, string) |
                                mvc.Result or (mvc.Result, error)
```

其中`mvc.Result`是一个只包含该函数的接口：`Dispatch(ctx iris.Context)`

**使用Iris MVC进行代码重用**

通过创建彼此独立的组件，开发人员能够在其他应用程序中快速轻松地重用组件，
对于具有不同数据的另一个应用程序，可以为一个应用程序重构相同（或类似）的视图，因为视图只是处理数据如何显示给用户。

如果您不熟悉后端Web开发，请首先阅读有关MVC架构模式的内容，一个好的开始就是维基百科文章。

**快速MVC教程第1部分**

这个栗子和  https://github.com/kataras/iris/blob/master/_examples/hello-world/main.go 一样。

似乎你必须编写额外的代码是不值得的，但请记住，这个例子没有使用像模型这样的iris mvc功能，
持久性或View引擎？也不是Session，这对于学习来说非常简单，可能你永远不会在你的应用程序中的任何地方使用简单的控制器。

在我的个人笔记本电脑上，在每个20MB吞吐量的“/hello”路径上使用MVC的这个例子JSON花费了大约2MB，
它适用于大多数应用，但你可以选择最适合你的Iris，低级处理程序：在大型应用程序上更小的代码库更易于维护。

```go
package main

import (
    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"

    "github.com/kataras/iris/middleware/logger"
    "github.com/kataras/iris/middleware/recover"
)

func main() {
    app := iris.New()
    // Optionally, add two built'n handlers
    // that can recover from any http-relative panics
    // and log the requests to the terminal.
    app.Use(recover.New())
    app.Use(logger.New())

    // Serve a controller based on the root Router, "/".
    mvc.New(app).Handle(new(ExampleController))

    // http://localhost:8080
    // http://localhost:8080/ping
    // http://localhost:8080/hello
    // http://localhost:8080/custom_path
    app.Run(iris.Addr(":8080"))
}

// ExampleController serves the "/", "/ping" and "/hello".
type ExampleController struct{}

// Get serves
// Method:   GET
// Resource: http://localhost:8080
func (c *ExampleController) Get() mvc.Result {
    return mvc.Response{
        ContentType: "text/html",
        Text:        "<h1>Welcome</h1>",
    }
}

// GetPing serves
// Method:   GET
// Resource: http://localhost:8080/ping
func (c *ExampleController) GetPing() string {
    return "pong"
}

// GetHello serves
// Method:   GET
// Resource: http://localhost:8080/hello
func (c *ExampleController) GetHello() interface{} {
    return map[string]string{"message": "Hello Iris!"}
}

// BeforeActivation called once, before the controller adapted to the main application
// and of course before the server ran.
// After version 9 you can also add custom routes for a specific controller's methods.
// Here you can register custom method's handlers
// use the standard router with `ca.Router` to do something that you can do without mvc as well,
// and add dependencies that will be binded to a controller's fields or method function's input arguments.
func (c *ExampleController) BeforeActivation(b mvc.BeforeActivation) {
    anyMiddlewareHere := func(ctx iris.Context) {
        ctx.Application().Logger().Warnf("Inside /custom_path")
        ctx.Next()
    }
    b.Handle("GET", "/custom_path", "CustomHandlerWithoutFollowingTheNamingGuide", anyMiddlewareHere)

    // or even add a global middleware based on this controller's router,
    // which in this example is the root "/":
    // b.Router().Use(myMiddleware)
}

// CustomHandlerWithoutFollowingTheNamingGuide serves
// Method:   GET
// Resource: http://localhost:8080/custom_path
func (c *ExampleController) CustomHandlerWithoutFollowingTheNamingGuide() string {
    return "hello from the custom handler without following the naming guide"
}

// GetUserBy serves
// Method:   GET
// Resource: http://localhost:8080/user/{username:string}
// By is a reserved "keyword" to tell the framework that you're going to
// bind path parameters in the function's input arguments, and it also
// helps to have "Get" and "GetBy" in the same controller.
//
// func (c *ExampleController) GetUserBy(username string) mvc.Result {
//     return mvc.View{
//         Name: "user/username.html",
//         Data: username,
//     }
// }

/* Can use more than one, the factory will make sure
that the correct http methods are being registered for each route
for this controller, uncomment these if you want:

func (c *ExampleController) Post() {}
func (c *ExampleController) Put() {}
func (c *ExampleController) Delete() {}
func (c *ExampleController) Connect() {}
func (c *ExampleController) Head() {}
func (c *ExampleController) Patch() {}
func (c *ExampleController) Options() {}
func (c *ExampleController) Trace() {}
*/

/*
func (c *ExampleController) All() {}
//        OR
func (c *ExampleController) Any() {}



func (c *ExampleController) BeforeActivation(b mvc.BeforeActivation) {
    // 1 -> the HTTP Method
    // 2 -> the route's path
    // 3 -> this controller's method name that should be handler for that route.
    b.Handle("GET", "/mypath/{param}", "DoIt", optionalMiddlewareHere...)
}

// After activation, all dependencies are set-ed - so read only access on them
// but still possible to add custom controller or simple standard handlers.
func (c *ExampleController) AfterActivation(a mvc.AfterActivation) {}

*/
```



Movies MVC Application
-----------------------------

Iris有一个非常强大和极快的MVC支持，你可以从方法函数返回任何类型的任何值，它将按预期发送到客户端。

- 如果为`string` 它将是body
- 如果`string`是第二个输出参数，那么它就是content type
- 如果是`int`那么它是状态代码
- 如果`error`而不是`nil`那么（任何类型）响应将被省略，并且将呈现带有400错误请求的错误文本。
- 如果是`(int, error)`和`error`不是`nil`，那么响应结果将是错误的文本，状态代码为`int`
- 如果是`custom struct`或`interface{}`或`slice`或`map`，那么它将呈现为json，除非跟随字符串内容类型。
- 如果`mvc.Result`然后它执行其`Dispatch`函数，那么可以使用好的设计模式在需要的地方拆分模型的逻辑。

没有什么能阻止您使用自己喜欢的文件夹结构。 Iris是一个低级Web框架，它有MVC一流的支持，但它不限制你的文件夹结构，这是你的选择。

结构取决于您自己的需求。我们无法告诉您如何设计自己的应用程序，但您可以自由地仔细查看下面的一个用例示例

**数据模型层**

让我们从`Movie`模型开始

```go
// file: datamodels/movie.go

package datamodels

// Movie is our sample data structure.
// Keep note that the tags for public-use (for our web app)
// should be kept in other file like "web/viewmodels/movie.go"
// which could wrap by embedding the datamodels.Movie or
// declare new fields instead butwe will use this datamodel
// as the only one Movie model in our application,
// for the shake of simplicty.
type Movie struct {
    ID     int64  `json:"id"`
    Name   string `json:"name"`
    Year   int    `json:"year"`
    Genre  string `json:"genre"`
    Poster string `json:"poster"`
}
```

**数据源/数据存储层**

之后，我们继续为我们的`Movies`创建一个简单的内存存储

```go
// file: datasource/movies.go

package datasource

import "github.com/kataras/iris/_examples/mvc/overview/datamodels"

// Movies is our imaginary data source.
var Movies = map[int64]datamodels.Movie{
    1: {
        ID:     1,
        Name:   "Casablanca",
        Year:   1942,
        Genre:  "Romance",
        Poster: "https://iris-go.com/images/examples/mvc-movies/1.jpg",
    },
    2: {
        ID:     2,
        Name:   "Gone with the Wind",
        Year:   1939,
        Genre:  "Romance",
        Poster: "https://iris-go.com/images/examples/mvc-movies/2.jpg",
    },
    3: {
        ID:     3,
        Name:   "Citizen Kane",
        Year:   1941,
        Genre:  "Mystery",
        Poster: "https://iris-go.com/images/examples/mvc-movies/3.jpg",
    },
    4: {
        ID:     4,
        Name:   "The Wizard of Oz",
        Year:   1939,
        Genre:  "Fantasy",
        Poster: "https://iris-go.com/images/examples/mvc-movies/4.jpg",
    },
    5: {
        ID:     5,
        Name:   "North by Northwest",
        Year:   1959,
        Genre:  "Thriller",
        Poster: "https://iris-go.com/images/examples/mvc-movies/5.jpg",
    },
}
```

**Repositories**

可以直接访问“数据源”并可以直接操作数据的层。

可选（因为你可以在`Service`中也有），但是对于那个例子，我们需要创建一个`Repository`，一个存储库处理“低级”，直接访问`Movies`数据源。
保留"a Repository",它是一个`interface`，因为它可以有所不同，这取决于您的应用程序开发的状态。
列如在生产中将使用一些真正的SQL查询或您用于查询数据的任何其他内容。

```go
// file: repositories/movie_repository.go

package repositories

import (
    "errors"
    "sync"

    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
)

// Query represents the visitor and action queries.
type Query func(datamodels.Movie) bool

// MovieRepository handles the basic operations of a movie entity/model.
// It's an interface in order to be testable, i.e a memory movie repository or
// a connected to an sql database.
type MovieRepository interface {
    Exec(query Query, action Query, limit int, mode int) (ok bool)

    Select(query Query) (movie datamodels.Movie, found bool)
    SelectMany(query Query, limit int) (results []datamodels.Movie)

    InsertOrUpdate(movie datamodels.Movie) (updatedMovie datamodels.Movie, err error)
    Delete(query Query, limit int) (deleted bool)
}

// NewMovieRepository returns a new movie memory-based repository,
// the one and only repository type in our example.
func NewMovieRepository(source map[int64]datamodels.Movie) MovieRepository {
    return &movieMemoryRepository{source: source}
}

// movieMemoryRepository is a "MovieRepository"
// which manages the movies using the memory data source (map).
type movieMemoryRepository struct {
    source map[int64]datamodels.Movie
    mu     sync.RWMutex
}

const (
    // ReadOnlyMode will RLock(read) the data .
    ReadOnlyMode = iota
    // ReadWriteMode will Lock(read/write) the data.
    ReadWriteMode
)

func (r *movieMemoryRepository) Exec(query Query, action Query, actionLimit int, mode int) (ok bool) {
    loops := 0

    if mode == ReadOnlyMode {
        r.mu.RLock()
        defer r.mu.RUnlock()
    } else {
        r.mu.Lock()
        defer r.mu.Unlock()
    }

    for _, movie := range r.source {
        ok = query(movie)
        if ok {
            if action(movie) {
                loops++
                if actionLimit >= loops {
                    break // break
                }
            }
        }
    }

    return
}

// Select receives a query function
// which is fired for every single movie model inside
// our imaginary data source.
// When that function returns true then it stops the iteration.
//
// It returns the query's return last known "found" value
// and the last known movie model
// to help callers to reduce the LOC.
//
// It's actually a simple but very clever prototype function
// I'm using everywhere since I firstly think of it,
// hope you'll find it very useful as well.
func (r *movieMemoryRepository) Select(query Query) (movie datamodels.Movie, found bool) {
    found = r.Exec(query, func(m datamodels.Movie) bool {
        movie = m
        return true
    }, 1, ReadOnlyMode)

    // set an empty datamodels.Movie if not found at all.
    if !found {
        movie = datamodels.Movie{}
    }

    return
}

// SelectMany same as Select but returns one or more datamodels.Movie as a slice.
// If limit <=0 then it returns everything.
func (r *movieMemoryRepository) SelectMany(query Query, limit int) (results []datamodels.Movie) {
    r.Exec(query, func(m datamodels.Movie) bool {
        results = append(results, m)
        return true
    }, limit, ReadOnlyMode)

    return
}

// InsertOrUpdate adds or updates a movie to the (memory) storage.
//
// Returns the new movie and an error if any.
func (r *movieMemoryRepository) InsertOrUpdate(movie datamodels.Movie) (datamodels.Movie, error) {
    id := movie.ID

    if id == 0 { // Create new action
        var lastID int64
        // find the biggest ID in order to not have duplications
        // in productions apps you can use a third-party
        // library to generate a UUID as string.
        r.mu.RLock()
        for _, item := range r.source {
            if item.ID > lastID {
                lastID = item.ID
            }
        }
        r.mu.RUnlock()

        id = lastID + 1
        movie.ID = id

        // map-specific thing
        r.mu.Lock()
        r.source[id] = movie
        r.mu.Unlock()

        return movie, nil
    }

    // Update action based on the movie.ID,
    // here we will allow updating the poster and genre if not empty.
    // Alternatively we could do pure replace instead:
    // r.source[id] = movie
    // and comment the code below;
    current, exists := r.Select(func(m datamodels.Movie) bool {
        return m.ID == id
    })

    if !exists { // ID is not a real one, return an error.
        return datamodels.Movie{}, errors.New("failed to update a nonexistent movie")
    }

    // or comment these and r.source[id] = m for pure replace
    if movie.Poster != "" {
        current.Poster = movie.Poster
    }

    if movie.Genre != "" {
        current.Genre = movie.Genre
    }

    // map-specific thing
    r.mu.Lock()
    r.source[id] = current
    r.mu.Unlock()

    return movie, nil
}

func (r *movieMemoryRepository) Delete(query Query, limit int) bool {
    return r.Exec(query, func(m datamodels.Movie) bool {
        delete(r.source, m.ID)
        return true
    }, limit, ReadWriteMode)
}
```

**Services**

可以访问“repositories”和“models”（甚至是“datamodels”，如果是简单应用程序）的函数的层。它应该包含大部分域逻辑。

我们需要一个服务来与我们的存储库在“high-level”和store/retrieve `Movies`中进行通信，这将在下面的Web控制器上使用。

```go
// file: services/movie_service.go

package services

import (
    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
    "github.com/kataras/iris/_examples/mvc/overview/repositories"
)

// MovieService handles some of the CRUID operations of the movie datamodel.
// It depends on a movie repository for its actions.
// It's here to decouple the data source from the higher level compoments.
// As a result a different repository type can be used with the same logic without any aditional changes.
// It's an interface and it's used as interface everywhere
// because we may need to change or try an experimental different domain logic at the future.
type MovieService interface {
    GetAll() []datamodels.Movie
    GetByID(id int64) (datamodels.Movie, bool)
    DeleteByID(id int64) bool
    UpdatePosterAndGenreByID(id int64, poster string, genre string) (datamodels.Movie, error)
}

// NewMovieService returns the default movie service.
func NewMovieService(repo repositories.MovieRepository) MovieService {
    return &movieService{
        repo: repo,
    }
}

type movieService struct {
    repo repositories.MovieRepository
}

// GetAll returns all movies.
func (s *movieService) GetAll() []datamodels.Movie {
    return s.repo.SelectMany(func(_ datamodels.Movie) bool {
        return true
    }, -1)
}

// GetByID returns a movie based on its id.
func (s *movieService) GetByID(id int64) (datamodels.Movie, bool) {
    return s.repo.Select(func(m datamodels.Movie) bool {
        return m.ID == id
    })
}

// UpdatePosterAndGenreByID updates a movie's poster and genre.
func (s *movieService) UpdatePosterAndGenreByID(id int64, poster string, genre string) (datamodels.Movie, error) {
    // update the movie and return it.
    return s.repo.InsertOrUpdate(datamodels.Movie{
        ID:     id,
        Poster: poster,
        Genre:  genre,
    })
}

// DeleteByID deletes a movie by its id.
//
// Returns true if deleted otherwise false.
func (s *movieService) DeleteByID(id int64) bool {
    return s.repo.Delete(func(m datamodels.Movie) bool {
        return m.ID == id
    }, 1)
}
```

**View Models**

```go
import (
    "github.com/kataras/iris/_examples/mvc/overview/datamodels"

    "github.com/kataras/iris/context"
)

type Movie struct {
    datamodels.Movie
}

func (m Movie) IsValid() bool {
    /* do some checks and return true if it's valid... */
    return m.ID > 0
}
```

Iris能够将任何自定义数据结构转换为HTTP Response,因此从理论上讲，如果确实需要，则允许使用以下内容：

```go
// Dispatch completes the `kataras/iris/mvc#Result` interface.
// Sends a `Movie` as a controlled http response.
// If its ID is zero or less then it returns a 404 not found error
// else it returns its json representation,
// (just like the controller's functions do for custom types by default).
//
// Don't overdo it, the application's logic should not be here.
// It's just one more step of validation before the response,
// simple checks can be added here.
//
// It's just a showcase,
// imagine the potentials this feature gives when designing a bigger application.
//
// This is called where the return value from a controller's method functions
// is type of `Movie`.
// For example the `controllers/movie_controller.go#GetBy`.
func (m Movie) Dispatch(ctx context.Context) {
    if !m.IsValid() {
        ctx.NotFound()
        return
    }
    ctx.JSON(m, context.JSON{Indent: " "})
}
```

但是，因为 Movie structure 不包含任何敏感数据，我们将使用“datamodels”作为唯一的一个模型包，
客户可以看到它的所有字段，我们不需要任何额外的功能或验证。

**Controllers**

处理Web请求，在服务端和客户端之间架起桥梁。

而最重要的是，在iris里，`MovieService`是与`Controller`通信的。
我们通常将所有与http相关的东西存储在名为“web”的不同文件夹中，所以所有控制器都可以在“web/controllers”内，
请注意，“通常”您也可以使用其他设计模式，这取决于您。

```go
// file: web/controllers/movie_controller.go

package controllers

import (
    "errors"

    "github.com/kataras/iris/_examples/mvc/overview/datamodels"
    "github.com/kataras/iris/_examples/mvc/overview/services"

    "github.com/kataras/iris"
)

// MovieController is our /movies controller.
type MovieController struct {
    // Our MovieService, it's an interface which
    // is binded from the main application.
    Service services.MovieService
}

// Get returns list of the movies.
// Demo:
// curl -i http://localhost:8080/movies
//
// The correct way if you have sensitive data:
// func (c *MovieController) Get() (results []viewmodels.Movie) {
//     data := c.Service.GetAll()
//
//     for _, movie := range data {
//         results = append(results, viewmodels.Movie{movie})
//     }
//     return
// }
// otherwise just return the datamodels.
func (c *MovieController) Get() (results []datamodels.Movie) {
    return c.Service.GetAll()
}

// GetBy returns a movie.
// Demo:
// curl -i http://localhost:8080/movies/1
func (c *MovieController) GetBy(id int64) (movie datamodels.Movie, found bool) {
    return c.Service.GetByID(id) // it will throw 404 if not found.
}

// PutBy updates a movie.
// Demo:
// curl -i -X PUT -F "genre=Thriller" -F "poster=@/Users/kataras/Downloads/out.gif" http://localhost:8080/movies/1
func (c *MovieController) PutBy(ctx iris.Context, id int64) (datamodels.Movie, error) {
    // get the request data for poster and genre
    file, info, err := ctx.FormFile("poster")
    if err != nil {
        return datamodels.Movie{}, errors.New("failed due form file 'poster' missing")
    }
    // we don't need the file so close it now.
    file.Close()

    // imagine that is the url of the uploaded file...
    poster := info.Filename
    genre := ctx.FormValue("genre")

    return c.Service.UpdatePosterAndGenreByID(id, poster, genre)
}

// DeleteBy deletes a movie.
// Demo:
// curl -i -X DELETE -u admin:password http://localhost:8080/movies/1
func (c *MovieController) DeleteBy(id int64) interface{} {
    wasDel := c.Service.DeleteByID(id)
    if wasDel {
        // return the deleted movie's ID
        return iris.Map{"deleted": id}
    }
    // right here we can see that a method function can return any of those two types(map or int),
    // we don't have to specify the return type to a specific type.
    return iris.StatusBadRequest
}
```

web/middleware 里的一个中间件

```go
// file: web/middleware/basicauth.go

package middleware

import "github.com/kataras/iris/middleware/basicauth"

// BasicAuth middleware sample.
var BasicAuth = basicauth.New(basicauth.Config{
    Users: map[string]string{
        "admin": "password",
    },
})
```

`main.go`

```go
// file: main.go

package main

import (
    "github.com/kataras/iris/_examples/mvc/overview/datasource"
    "github.com/kataras/iris/_examples/mvc/overview/repositories"
    "github.com/kataras/iris/_examples/mvc/overview/services"
    "github.com/kataras/iris/_examples/mvc/overview/web/controllers"
    "github.com/kataras/iris/_examples/mvc/overview/web/middleware"

    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
)

func main() {
    app := iris.New()
    app.Logger().SetLevel("debug")

    // Load the template files.
    app.RegisterView(iris.HTML("./web/views", ".html"))

    // Register our controllers.
    // mvc.New(app.Party("/movies")).Handle(new(controllers.MovieController))
    // But,
    // You can also split the code you write to configure an mvc.Application
    // using the `mvc.Configure` method, as shown below.
    mvc.Configure(app.Party("/movies"), movies)

    // http://localhost:8080/movies
    // http://localhost:8080/movies/1
    app.Run(
        // Start the web server at localhost:8080
        iris.Addr("localhost:8080"),
        // disables updates:
        iris.WithoutVersionChecker,
        // skip err server closed when CTRL/CMD+C pressed:
        iris.WithoutServerError(iris.ErrServerClosed),
        // enables faster json serialization and more:
        iris.WithOptimizations,
    )
}

// note the mvc.Application, it's not iris.Application.
func movies(app *mvc.Application) {
    // Add the basic authentication(admin:password) middleware
    // for the /movies based requests.
    app.Router.Use(middleware.BasicAuth)

    // Create our movie repository with some (memory) data from the datasource.
    repo := repositories.NewMovieRepository(datasource.Movies)
    // Create our movie service, we will bind it to the movie app's dependencies.
    movieService := services.NewMovieService(repo)
    app.Register(movieService)

    // serve our movies controller.
    // Note that you can serve more than one controller
    // you can also create child mvc apps using the `movies.Party(relativePath)` or `movies.Clone(app.Party(...))`
    // if you want.
    app.Handle(new(controllers.MovieController))
}
```

MVC loves Websockets
-----------------------------

```go
package main

import (
    "sync/atomic"

    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
    "github.com/kataras/iris/websocket"
)

func main() {
    app := iris.New()
    // load templates.
    app.RegisterView(iris.HTML("./views", ".html"))

    // render the ./views/index.html.
    app.Get("/", func(ctx iris.Context) {
        ctx.View("index.html")
    })

    mvc.Configure(app.Party("/websocket"), configureMVC)
    // Or mvc.New(app.Party(...)).Configure(configureMVC)

    // http://localhost:8080
    app.Run(iris.Addr(":8080"))
}

func configureMVC(m *mvc.Application) {
    ws := websocket.New(websocket.Config{})
    // http://localhost:8080/websocket/iris-ws.js
    m.Router.Any("/iris-ws.js", websocket.ClientHandler())

    // This will bind the result of ws.Upgrade which is a websocket.Connection
    // to the controller(s) served by the `m.Handle`.
    m.Register(ws.Upgrade)
    m.Handle(new(websocketController))
}

var visits uint64

func increment() uint64 {
    return atomic.AddUint64(&visits, 1)
}

func decrement() uint64 {
    return atomic.AddUint64(&visits, ^uint64(0))
}

type websocketController struct {
    // Note that you could use an anonymous field as well, it doesn't matter, binder will find it.
    //
    // This is the current websocket connection, each client has its own instance of the *websocketController.
    Conn websocket.Connection
}

func (c *websocketController) onLeave(roomName string) {
    // visits--
    newCount := decrement()
    // This will call the "visit" event on all clients, except the current one,
    // (it can't because it's left but for any case use this type of design)
    c.Conn.To(websocket.Broadcast).Emit("visit", newCount)
}

func (c *websocketController) update() {
    // visits++
    newCount := increment()

    // This will call the "visit" event on all clients, including the current
    // with the 'newCount' variable.
    //
    // There are many ways that u can do it and faster, for example u can just send a new visitor
    // and client can increment itself, but here we are just "showcasing" the websocket controller.
    c.Conn.To(websocket.All).Emit("visit", newCount)
}

func (c *websocketController) Get( /* websocket.Connection could be lived here as well, it doesn't matter */ ) {
    c.Conn.OnLeave(c.onLeave)
    c.Conn.On("visit", c.update)

    // call it after all event callbacks registration.
    c.Conn.Wait()
}
```

MVC loves Sessions
-----------------------------

```go
// +build go1.9

package main

import (
    "fmt"
    "time"

    "github.com/kataras/iris"
    "github.com/kataras/iris/mvc"
    "github.com/kataras/iris/sessions"
)

// VisitController handles the root route.
type VisitController struct {
    // the current request session,
    // its initialization happens by the dependency function that we've added to the `visitApp`.
    Session *sessions.Session

    // A time.time which is binded from the MVC,
    // order of binded fields doesn't matter.
    StartTime time.Time
}

// Get handles
// Method: GET
// Path: http://localhost:8080
func (c *VisitController) Get() string {
    // it increments a "visits" value of integer by one,
    // if the entry with key 'visits' doesn't exist it will create it for you.
    visits := c.Session.Increment("visits", 1)
    // write the current, updated visits.
    since := time.Now().Sub(c.StartTime).Seconds()
    return fmt.Sprintf("%d visit from my current session in %0.1f seconds of server's up-time",
        visits, since)
}

func newApp() *iris.Application {
    app := iris.New()
    sess := sessions.New(sessions.Config{Cookie: "mysession_cookie_name"})

    visitApp := mvc.New(app.Party("/"))
    // bind the current *session.Session, which is required, to the `VisitController.Session`
    // and the time.Now() to the `VisitController.StartTime`.
    visitApp.Register(
        // if dependency is a function which accepts
        // a Context and returns a single value
        // then the result type of this function is resolved by the controller
        // and on each request it will call the function with its Context
        // and set the result(the *sessions.Session here) to the controller's field.
        //
        // If dependencies are registered without field or function's input arguments as
        // consumers then those dependencies are being ignored before the server ran,
        // so you can bind many dependecies and use them in different controllers.
        sess.Start,
        time.Now(),
    )
    visitApp.Handle(new(VisitController))

    return app
}

func main() {
    app := newApp()

    // 1. open the browser (no in private mode)
    // 2. navigate to http://localhost:8080
    // 3. refresh the page some times
    // 4. close the browser
    // 5. re-open the browser and re-play 2.
    app.Run(iris.Addr(":8080"), iris.WithoutVersionChecker)
}
```

A Singleton Controller
-----------------------------

```go
package main

import (
	"fmt"
	"sync/atomic"

	"github.com/kataras/iris"
	"github.com/kataras/iris/mvc"
)

func main() {
	app := iris.New()
	mvc.New(app.Party("/")).Handle(&globalVisitorsController{visits: 0})

	// http://localhost:8080
	app.Run(iris.Addr(":8080"))
}

type globalVisitorsController struct {
	// When a singleton controller is used then concurent safe access is up to the developers, because
	// all clients share the same controller instance instead.
	// Note that any controller's methods
	// are per-client, but the struct's field can be shared across multiple clients if the structure
	// does not have any dynamic struct field dependencies that depend on the iris.Context
	// and ALL field's values are NOT zero, at this case we use uint64 which it's no zero (even if we didn't set it
	// manually ease-of-understand reasons) because it's a value of &{0}.
	// All the above declares a Singleton, note that you don't have to write a single line of code to do this, Iris is smart enough.
	//
	// see `Get`.
	visits uint64
}

func (c *globalVisitorsController) Get() string {
	count := atomic.AddUint64(&c.visits, 1)
	return fmt.Sprintf("Total visitors: %d", count)
}

```

----

More folder structure guidelines can be found at the [https://github.com/kataras/iris/tree/master/_examples/#structuring](https://github.com/kataras/iris/tree/master/_examples/#structuring) section.

Follow the examples below

- [Hello world](https://github.com/kataras/iris/blob/master/_examples/mvc/hello-world/main.go) **UPDATED**
- [Session Controller](https://github.com/kataras/iris/blob/master/_examples/mvc/session-controller/main.go) **UPDATED**
- [Overview - Plus Repository and Service layers](https://github.com/kataras/iris/tree/master/_examples/mvc/overview) **UPDATED**
- [Login showcase - Plus Repository and Service layers](https://github.com/kataras/iris/tree/master/_examples/mvc/login) **UPDATED**
- [Singleton](https://github.com/kataras/iris/tree/master/_examples/mvc/singleton) **NEW**
- [Websocket Controller](https://github.com/kataras/iris/tree/master/_examples/mvc/websocket) **NEW**
- [Register Middleware](https://github.com/kataras/iris/tree/master/_examples/mvc/middleware) **NEW**
- [Vue.js Todo MVC](https://github.com/kataras/iris/tree/master/_examples/tutorial/vuejs-todo-mvc) **NEW**

