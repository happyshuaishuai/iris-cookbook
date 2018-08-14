Routing
=============

**Handlers**

请求处理

```go
// 响应一个HTTP请求并进行处理
// 返回头与数据写入 Context.ResponseWriter()
// 返回请求完成的信号
// 之后使用上下文是无效的，同时完成处理程序调用	
//
// 根据客户端HTTP软件、HTTP协议版本以及客户端和IRIS服务器之间的任何中介
// 在写入 context.ResponseWriter() 之后，不可能读取 Context.Request().Body
// 谨慎的处理程序应该首先读取 Context.Request().Body
//
// 除了读取 body 外，处理程序不应修改所提供的 Context 。
//
// If Handler panics, the server (the caller of Handler) assumes that
// the effect of the panic was isolated to the active request.
// It recovers the panic, logs a stack trace to the server error log
// and hangs up the connection.

type Handler func(Context)
```

一旦 Handler 被注册后，我们可以使用返回的 [`Route`](https://godoc.org/github.com/kataras/iris/core/router#Route) 实例为处理程序注册命名，以便在代码或模板中更容易查找。

有关更多信息，请查看'路由和反向查找'部分。

**API**

支持所有HTTP方法，开发人员还可以为不同方法注册相同路径的处理

第一个参数是HTTP方法，第二个参数是路由的请求路径，当用户从服务器请求该特定资源路径时,
第三个可变参数应包含一个或多个由注册表顺序执行的 iris.Handler 。

```go
app := iris.New()

app.Handle("GET", "/contact", func(ctx iris.Context) {
    ctx.HTML("<h1> Hello from /contact </h1>")
})
```

为了让开发人员更轻松，iris为所有HTTP方法提供函数。

```go
app := iris.New()

// Method: "GET"
app.Get("/", handler)

// Method: "POST"
app.Post("/", handler)

// Method: "PUT"
app.Put("/", handler)

// Method: "DELETE"
app.Delete("/", handler)

// Method: "OPTIONS"
app.Options("/", handler)

// Method: "TRACE"
app.Trace("/", handler)

// Method: "CONNECT"
app.Connect("/", handler)

// Method: "HEAD"
app.Head("/", handler)

// Method: "PATCH"
app.Patch("/", handler)

// register the route for all HTTP Methods
app.Any("/", handler)

func handler(ctx iris.Context){
    ctx.Writef("Hello from method: %s and path: %s", ctx.Method(), ctx.Path())
}
```

**Grouping Routes**

由路径前缀分组的一组路由可以（可选）共享相同的中间件处理程序和模板布局, 也可以多组嵌套。

`.Party`用于分组路由，开发人员可以声明无限数量的（嵌套）组。

```go
app := iris.New()

users := app.Party("/users", myAuthMiddlewareHandler)

// http://localhost:8080/users/42/profile
users.Get("/{id:int}/profile", userProfileHandler)
// http://localhost:8080/users/messages/1
users.Get("/inbox/{id:int}", userMessageHandler)
```

也可以使用接收子路由器（Party）的功能编写相同的内容。

```go
app := iris.New()

app.PartyFunc("/users", func(users iris.Party) {
    users.Use(myAuthMiddlewareHandler)

    // http://localhost:8080/users/42/profile
    users.Get("/{id:int}/profile", userProfileHandler)
    // http://localhost:8080/users/messages/1
    users.Get("/inbox/{id:int}", userMessageHandler)
})
```
***************************************************************************************
Context
-----------------------------

`iris.Context` [源代码](https://github.com/kataras/iris/blob/master/context/context.go)可以在这里找到。使用具有自动补全功能的IDE编辑器保持记录将对您有所帮助。

```go
// Context is the midle-man server's "object" for the clients.
//
// A New context is being acquired from a sync.Pool on each connection.
// The Context is the most important thing on the iris's http flow.
//
// Developers send responses to the client's request through a Context.
// Developers get request information from the client's request a Context.
//
// This context is an implementation of the context.Context sub-package.
// context.Context is very extensible and developers can override
// its methods if that is actually needed.
type Context interface {
    // ResponseWriter returns an http.ResponseWriter compatible response writer, as expected.
    ResponseWriter() ResponseWriter
    // ResetResponseWriter should change or upgrade the Context's ResponseWriter.
    ResetResponseWriter(ResponseWriter)

    // Request returns the original *http.Request, as expected.
    Request() *http.Request

    // SetCurrentRouteName sets the route's name internally,
    // in order to be able to find the correct current "read-only" Route when
    // end-developer calls the `GetCurrentRoute()` function.
    // It's being initialized by the Router, if you change that name
    // manually nothing really happens except that you'll get other
    // route via `GetCurrentRoute()`.
    // Instead, to execute a different path
    // from this context you should use the `Exec` function
    // or change the handlers via `SetHandlers/AddHandler` functions.
    SetCurrentRouteName(currentRouteName string)
    // GetCurrentRoute returns the current registered "read-only" route that
    // was being registered to this request's path.
    GetCurrentRoute() RouteReadOnly

    // AddHandler can add handler(s)
    // to the current request in serve-time,
    // these handlers are not persistenced to the router.
    //
    // Router is calling this function to add the route's handler.
    // If AddHandler called then the handlers will be inserted
    // to the end of the already-defined route's handler.
    //
    AddHandler(...Handler)
    // SetHandlers replaces all handlers with the new.
    SetHandlers(Handlers)
    // Handlers keeps tracking of the current handlers.
    Handlers() Handlers

    // HandlerIndex sets the current index of the
    // current context's handlers chain.
    // If -1 passed then it just returns the
    // current handler index without change the current index.rns that index, useless return value.
    //
    // Look Handlers(), Next() and StopExecution() too.
    HandlerIndex(n int) (currentIndex int)
    // HandlerName returns the current handler's name, helpful for debugging.
    HandlerName() string
    // Next calls all the next handler from the handlers chain,
    // it should be used inside a middleware.
    //
    // Note: Custom context should override this method in order to be able to pass its own context.Context implementation.
    Next()
    // NextHandler returns(but it is NOT executes) the next handler from the handlers chain.
    //
    // Use .Skip() to skip this handler if needed to execute the next of this returning handler.
    NextHandler() Handler
    // Skip skips/ignores the next handler from the handlers chain,
    // it should be used inside a middleware.
    Skip()
    // StopExecution if called then the following .Next calls are ignored.
    StopExecution()
    // IsStopped checks and returns true if the current position of the Context is 255,
    // means that the StopExecution() was called.
    IsStopped() bool

    //  +------------------------------------------------------------+
    //  | Current "user/request" storage                             |
    //  | and share information between the handlers - Values().     |
    //  | Save and get named path parameters - Params()              |
    //  +------------------------------------------------------------+

    // Params returns the current url's named parameters key-value storage.
    // Named path parameters are being saved here.
    // This storage, as the whole Context, is per-request lifetime.
    Params() *RequestParams

    // Values returns the current "user" storage.
    // Named path parameters and any optional data can be saved here.
    // This storage, as the whole Context, is per-request lifetime.
    //
    // You can use this function to Set and Get local values
    // that can be used to share information between handlers and middleware.
    Values() *memstore.Store
    // Translate is the i18n (localization) middleware's function,
    // it calls the Get("translate") to return the translated value.
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/miscellaneous/i18n
    Translate(format string, args ...interface{}) string

    //  +------------------------------------------------------------+
    //  | Path, Host, Subdomain, IP, Headers etc...                  |
    //  +------------------------------------------------------------+

    // Method returns the request.Method, the client's http method to the server.
    Method() string
    // Path returns the full request path,
    // escaped if EnablePathEscape config field is true.
    Path() string
    // RequestPath returns the full request path,
    // based on the 'escape'.
    RequestPath(escape bool) string

    // Host returns the host part of the current url.
    Host() string
    // Subdomain returns the subdomain of this request, if any.
    // Note that this is a fast method which does not cover all cases.
    Subdomain() (subdomain string)
    // RemoteAddr tries to parse and return the real client's request IP.
    //
    // Based on allowed headers names that can be modified from Configuration.RemoteAddrHeaders.
    //
    // If parse based on these headers fail then it will return the Request's `RemoteAddr` field
    // which is filled by the server before the HTTP handler.
    //
    // Look `Configuration.RemoteAddrHeaders`,
    //      `Configuration.WithRemoteAddrHeader(...)`,
    //      `Configuration.WithoutRemoteAddrHeader(...)` for more.
    RemoteAddr() string
    // GetHeader returns the request header's value based on its name.
    GetHeader(name string) string
    // IsAjax returns true if this request is an 'ajax request'( XMLHttpRequest)
    //
    // There is no a 100% way of knowing that a request was made via Ajax.
    // You should never trust data coming from the client, they can be easily overcome by spoofing.
    //
    // Note that "X-Requested-With" Header can be modified by any client(because of "X-"),
    // so don't rely on IsAjax for really serious stuff,
    // try to find another way of detecting the type(i.e, content type),
    // there are many blogs that describe these problems and provide different kind of solutions,
    // it's always depending on the application you're building,
    // this is the reason why this ``IsAjax`` is simple enough for general purpose use.
    //
    // Read more at: https://developer.mozilla.org/en-US/docs/AJAX
    // and https://xhr.spec.whatwg.org/
    IsAjax() bool

    //  +------------------------------------------------------------+
    //  | Response Headers helpers                                   |
    //  +------------------------------------------------------------+

    // Header adds a header to the response writer.
    Header(name string, value string)

    // ContentType sets the response writer's header key "Content-Type" to the 'cType'.
    ContentType(cType string)
    // GetContentType returns the response writer's header value of "Content-Type"
    // which may, setted before with the 'ContentType'.
    GetContentType() string

    // StatusCode sets the status code header to the response.
    // Look .GetStatusCode too.
    StatusCode(statusCode int)
    // GetStatusCode returns the current status code of the response.
    // Look StatusCode too.
    GetStatusCode() int

    // Redirect redirect sends a redirect response the client
    // accepts 2 parameters string and an optional int
    // first parameter is the url to redirect
    // second parameter is the http status should send, default is 302 (StatusFound),
    // you can set it to 301 (Permant redirect), if that's nessecery
    Redirect(urlToRedirect string, statusHeader ...int)

    //  +------------------------------------------------------------+
    //  | Various Request and Post Data                              |
    //  +------------------------------------------------------------+

    // URLParam returns the get parameter from a request , if any.
    URLParam(name string) string
    // URLParamInt returns the url query parameter as int value from a request,
    // returns an error if parse failed.
    URLParamInt(name string) (int, error)
    // URLParamInt64 returns the url query parameter as int64 value from a request,
    // returns an error if parse failed.
    URLParamInt64(name string) (int64, error)
    // URLParams returns a map of GET query parameters separated by comma if more than one
    // it returns an empty map if nothing found.
    URLParams() map[string]string

    // FormValue returns a single form value by its name/key
    FormValue(name string) string
    // FormValues returns all post data values with their keys
    // form data, get, post & put query arguments
    //
    // NOTE: A check for nil is necessary.
    FormValues() map[string][]string
    // PostValue returns a form's only-post value by its name,
    // same as Request.PostFormValue.
    PostValue(name string) string
    // FormFile returns the first file for the provided form key.
    // FormFile calls ctx.Request.ParseMultipartForm and ParseForm if necessary.
    //
    // same as Request.FormFile.
    FormFile(key string) (multipart.File, *multipart.FileHeader, error)

    //  +------------------------------------------------------------+
    //  | Custom HTTP Errors                                         |
    //  +------------------------------------------------------------+

    // NotFound emits an error 404 to the client, using the specific custom error error handler.
    // Note that you may need to call ctx.StopExecution() if you don't want the next handlers
    // to be executed. Next handlers are being executed on iris because you can alt the
    // error code and change it to a more specific one, i.e
    // users := app.Party("/users")
    // users.Done(func(ctx context.Context){ if ctx.StatusCode() == 400 { /*  custom error code for /users */ }})
    NotFound()

    //  +------------------------------------------------------------+
    //  | Body Readers                                               |
    //  +------------------------------------------------------------+

    // SetMaxRequestBodySize sets a limit to the request body size
    // should be called before reading the request body from the client.
    SetMaxRequestBodySize(limitOverBytes int64)

    // UnmarshalBody reads the request's body and binds it to a value or pointer of any type
    // Examples of usage: context.ReadJSON, context.ReadXML.
    UnmarshalBody(v interface{}, unmarshaler Unmarshaler) error
    // ReadJSON reads JSON from request's body and binds it to a value of any json-valid type.
    ReadJSON(jsonObject interface{}) error
    // ReadXML reads XML from request's body and binds it to a value of any xml-valid type.
    ReadXML(xmlObject interface{}) error
    // ReadForm binds the formObject  with the form data
    // it supports any kind of struct.
    ReadForm(formObject interface{}) error

    //  +------------------------------------------------------------+
    //  | Body (raw) Writers                                         |
    //  +------------------------------------------------------------+

    // Write writes the data to the connection as part of an HTTP reply.
    //
    // If WriteHeader has not yet been called, Write calls
    // WriteHeader(http.StatusOK) before writing the data. If the Header
    // does not contain a Content-Type line, Write adds a Content-Type set
    // to the result of passing the initial 512 bytes of written data to
    // DetectContentType.
    //
    // Depending on the HTTP protocol version and the client, calling
    // Write or WriteHeader may prevent future reads on the
    // Request.Body. For HTTP/1.x requests, handlers should read any
    // needed request body data before writing the response. Once the
    // headers have been flushed (due to either an explicit Flusher.Flush
    // call or writing enough data to trigger a flush), the request body
    // may be unavailable. For HTTP/2 requests, the Go HTTP server permits
    // handlers to continue to read the request body while concurrently
    // writing the response. However, such behavior may not be supported
    // by all HTTP/2 clients. Handlers should read before writing if
    // possible to maximize compatibility.
    Write(body []byte) (int, error)
    // Writef formats according to a format specifier and writes to the response.
    //
    // Returns the number of bytes written and any write error encountered.
    Writef(format string, args ...interface{}) (int, error)
    // WriteString writes a simple string to the response.
    //
    // Returns the number of bytes written and any write error encountered.
    WriteString(body string) (int, error)
    // WriteWithExpiration like Write but it sends with an expiration datetime
    // which is refreshed every package-level `StaticCacheDuration` field.
    WriteWithExpiration(body []byte, modtime time.Time) (int, error)
    // StreamWriter registers the given stream writer for populating
    // response body.
    //
    // Access to context's and/or its' members is forbidden from writer.
    //
    // This function may be used in the following cases:
    //
    //     * if response body is too big (more than iris.LimitRequestBodySize(if setted)).
    //     * if response body is streamed from slow external sources.
    //     * if response body must be streamed to the client in chunks.
    //     (aka `http server push`).
    //
    // receives a function which receives the response writer
    // and returns false when it should stop writing, otherwise true in order to continue
    StreamWriter(writer func(w io.Writer) bool)

    //  +------------------------------------------------------------+
    //  | Body Writers with compression                              |
    //  +------------------------------------------------------------+
    // ClientSupportsGzip retruns true if the client supports gzip compression.
    ClientSupportsGzip() bool
    // WriteGzip accepts bytes, which are compressed to gzip format and sent to the client.
    // returns the number of bytes written and an error ( if the client doesn' supports gzip compression)
    //
    // This function writes temporary gzip contents, the ResponseWriter is untouched.
    WriteGzip(b []byte) (int, error)
    // TryWriteGzip accepts bytes, which are compressed to gzip format and sent to the client.
    // If client does not supprots gzip then the contents are written as they are, uncompressed.
    //
    // This function writes temporary gzip contents, the ResponseWriter is untouched.
    TryWriteGzip(b []byte) (int, error)
    // GzipResponseWriter converts the current response writer into a response writer
    // which when its .Write called it compress the data to gzip and writes them to the client.
    //
    // Can be also disabled with its .Disable and .ResetBody to rollback to the usual response writer.
    GzipResponseWriter() *GzipResponseWriter
    // Gzip enables or disables (if enabled before) the gzip response writer,if the client
    // supports gzip compression, so the following response data will
    // be sent as compressed gzip data to the client.
    Gzip(enable bool)

    //  +------------------------------------------------------------+
    //  | Rich Body Content Writers/Renderers                        |
    //  +------------------------------------------------------------+

    // ViewLayout sets the "layout" option if and when .View
    // is being called afterwards, in the same request.
    // Useful when need to set or/and change a layout based on the previous handlers in the chain.
    //
    // Note that the 'layoutTmplFile' argument can be setted to iris.NoLayout || view.NoLayout
    // to disable the layout for a specific view render action,
    // it disables the engine's configuration's layout property.
    //
    // Look .ViewData and .View too.
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/view/context-view-data/
    ViewLayout(layoutTmplFile string)

    // ViewData saves one or more key-value pair in order to be passed if and when .View
    // is being called afterwards, in the same request.
    // Useful when need to set or/and change template data from previous hanadlers in the chain.
    //
    // If .View's "binding" argument is not nil and it's not a type of map
    // then these data are being ignored, binding has the priority, so the main route's handler can still decide.
    // If binding is a map or context.Map then these data are being added to the view data
    // and passed to the template.
    //
    // After .View, the data are not destroyed, in order to be re-used if needed (again, in the same request as everything else),
    // to clear the view data, developers can call:
    // ctx.Set(ctx.Application().ConfigurationReadOnly().GetViewDataContextKey(), nil)
    //
    // If 'key' is empty then the value is added as it's (struct or map) and developer is unable to add other value.
    //
    // Look .ViewLayout and .View too.
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/view/context-view-data/
    ViewData(key string, value interface{})

    // GetViewData returns the values registered by `context#ViewData`.
    // The return value is `map[string]interface{}`, this means that
    // if a custom struct registered to ViewData then this function
    // will try to parse it to map, if failed then the return value is nil
    // A check for nil is always a good practise if different
    // kind of values or no data are registered via `ViewData`.
    //
    // Similarly to `viewData := ctx.Values().Get("iris.viewData")` or
    // `viewData := ctx.Values().Get(ctx.Application().ConfigurationReadOnly().GetViewDataContextKey())`.
    GetViewData() map[string]interface{}

    // View renders templates based on the adapted view engines.
    // First argument accepts the filename, relative to the view engine's Directory,
    // i.e: if directory is "./templates" and want to render the "./templates/users/index.html"
    // then you pass the "users/index.html" as the filename argument.
    //
    // Look: .ViewData and .ViewLayout too.
    //
    // Examples: https://github.com/kataras/iris/tree/master/_examples/view/
    View(filename string) error

    // Binary writes out the raw bytes as binary data.
    Binary(data []byte) (int, error)
    // Text writes out a string as plain text.
    Text(text string) (int, error)
    // HTML writes out a string as text/html.
    HTML(htmlContents string) (int, error)
    // JSON marshals the given interface object and writes the JSON response.
    JSON(v interface{}, options ...JSON) (int, error)
    // JSONP marshals the given interface object and writes the JSON response.
    JSONP(v interface{}, options ...JSONP) (int, error)
    // XML marshals the given interface object and writes the XML response.
    XML(v interface{}, options ...XML) (int, error)
    // Markdown parses the markdown to html and renders to client.
    Markdown(markdownB []byte, options ...Markdown) (int, error)

    //  +------------------------------------------------------------+
    //  | Serve files                                                |
    //  +------------------------------------------------------------+

    // ServeContent serves content, headers are autoset
    // receives three parameters, it's low-level function, instead you can use .ServeFile(string,bool)/SendFile(string,string)
    //
    // You can define your own "Content-Type" header also, after this function call
    // Doesn't implements resuming (by range), use ctx.SendFile instead
    ServeContent(content io.ReadSeeker, filename string, modtime time.Time, gzipCompression bool) error
    // ServeFile serves a view file, to send a file ( zip for example) to the client you should use the SendFile(serverfilename,clientfilename)
    // receives two parameters
    // filename/path (string)
    // gzipCompression (bool)
    //
    // You can define your own "Content-Type" header also, after this function call
    // This function doesn't implement resuming (by range), use ctx.SendFile instead
    //
    // Use it when you want to serve css/js/... files to the client, for bigger files and 'force-download' use the SendFile.
    ServeFile(filename string, gzipCompression bool) error
    // SendFile sends file for force-download to the client
    //
    // Use this instead of ServeFile to 'force-download' bigger files to the client.
    SendFile(filename string, destinationName string) error

    //  +------------------------------------------------------------+
    //  | Cookies                                                    |
    //  +------------------------------------------------------------+

    // SetCookie adds a cookie
    SetCookie(cookie *http.Cookie)
    // SetCookieKV adds a cookie, receives just a name(string) and a value(string)
    //
    // If you use this method, it expires at 2 hours
    // use ctx.SetCookie or http.SetCookie if you want to change more fields.
    SetCookieKV(name, value string)
    // GetCookie returns cookie's value by it's name
    // returns empty string if nothing was found.
    GetCookie(name string) string
    // RemoveCookie deletes a cookie by it's name.
    RemoveCookie(name string)
    // VisitAllCookies takes a visitor which loops
    // on each (request's) cookies' name and value.
    VisitAllCookies(visitor func(name string, value string))

    // MaxAge returns the "cache-control" request header's value
    // seconds as int64
    // if header not found or parse failed then it returns -1.
    MaxAge() int64

    //  +------------------------------------------------------------+
    //  | Advanced: Response Recorder and Transactions               |
    //  +------------------------------------------------------------+

    // Record transforms the context's basic and direct responseWriter to a ResponseRecorder
    // which can be used to reset the body, reset headers, get the body,
    // get & set the status code at any time and more.
    Record()
    // Recorder returns the context's ResponseRecorder
    // if not recording then it starts recording and returns the new context's ResponseRecorder
    Recorder() *ResponseRecorder
    // IsRecording returns the response recorder and a true value
    // when the response writer is recording the status code, body, headers and so on,
    // else returns nil and false.
    IsRecording() (*ResponseRecorder, bool)

    // BeginTransaction starts a scoped transaction.
    //
    // You can search third-party articles or books on how Business Transaction works (it's quite simple, especially here).
    //
    // Note that this is unique and new
    // (=I haver never seen any other examples or code in Golang on this subject, so far, as with the most of iris features...)
    // it's not covers all paths,
    // such as databases, this should be managed by the libraries you use to make your database connection,
    // this transaction scope is only for context's response.
    // Transactions have their own middleware ecosystem also, look iris.go:UseTransaction.
    //
    // See https://github.com/kataras/iris/tree/master/_examples/ for more
    BeginTransaction(pipe func(t *Transaction))
    // SkipTransactions if called then skip the rest of the transactions
    // or all of them if called before the first transaction
    SkipTransactions()
    // TransactionsSkipped returns true if the transactions skipped or canceled at all.
    TransactionsSkipped() bool

    // Exec calls the framewrok's ServeCtx
    // based on this context but with a changed method and path
    // like it was requested by the user, but it is not.
    //
    // Offline means that the route is registered to the iris and have all features that a normal route has
    // BUT it isn't available by browsing, its handlers executed only when other handler's context call them
    // it can validate paths, has sessions, path parameters and all.
    //
    // You can find the Route by app.GetRoute("theRouteName")
    // you can set a route name as: myRoute := app.Get("/mypath", handler)("theRouteName")
    // that will set a name to the route and returns its RouteInfo instance for further usage.
    //
    // It doesn't changes the global state, if a route was "offline" it remains offline.
    //
    // app.None(...) and app.GetRoutes().Offline(route)/.Online(route, method)
    //
    // Example: https://github.com/kataras/iris/tree/master/_examples/routing/route-state
    //
    // User can get the response by simple using rec := ctx.Recorder(); rec.Body()/rec.StatusCode()/rec.Header().
    //
    // Context's Values and the Session are kept in order to be able to communicate via the result route.
    //
    // It's for extreme use cases, 99% of the times will never be useful for you.
    Exec(method string, path string)

    // Application returns the iris app instance which belongs to this context.
    // Worth to notice that this function returns an interface
    // of the Application, which contains methods that are safe
    // to be executed at serve-time. The full app's fields
    // and methods are not available here for the developer's safety.
    Application() Application
}
```

The [examples](https://github.com/kataras/iris/tree/master/_examples) will give you the direction.
*********************************************************************************************************************************
Path Parameters
-----------------------------

**Dynamic Path Parameters**

Iris拥有您遇到过的最简单，最强大的路由进程。

与此同时，Iris有自己的interpeter（是一种编程语言），用于路由的路径语法及其动态路径参数解析和评估。我们称它们为“macros”用于快捷方式。

怎么样？它计算了它的需求，如果不需要任何特殊的正则表达式，那么它只是用低级路径语法注册路由，
否则它预先编译正则表达式并添加必要的中间件。这意味着与其他路由器或Web框架相比，您的性能成本为零。

标准 macro types 路由路径参数

```go
+------------------------+
| {param:string}         |
+------------------------+
string type
anything

+------------------------+
| {param:int}            |
+------------------------+
int type
only numbers (0-9)

+------------------------+
| {param:long}           |
+------------------------+
int64 type
only numbers (0-9)

+------------------------+
| {param:boolean}        |
+------------------------+
bool type
only "1" or "t" or "T" or "TRUE" or "true" or "True"
or "0" or "f" or "F" or "FALSE" or "false" or "False"

+------------------------+
| {param:alphabetical}   |
+------------------------+
alphabetical/letter type
letters only (upper or lowercase)

+------------------------+
| {param:file}           |
+------------------------+
file type
letters (upper or lowercase)
numbers (0-9)
underscore (_)
dash (-)
point (.)
no spaces ! or other character

+------------------------+
| {param:path}           |
+------------------------+
path type
anything, should be the last part, more than one path segment,
i.e: /path1/path2/path3 , ctx.Params().Get("param") == "/path1/path2/path3"
```

如果缺少类型，则参数的类型默认为字符串, `{param} == {param:string}`

如果在该类型上找不到函数，则使用 `string` macro 类型的函数。

除了 iris 提供基本类型和一些默认的“macro funcs”，你也可以自己注册！

注册命名路径参数函数

```go
app.Macros().Int.RegisterFunc("min", func(argument int) func(paramValue string) bool {
    // [...]
    return true 
    // -> true means valid, false means invalid fire 404 or if "else 500" is appended to the macro syntax then internal server error.
})
```

在 `func(argument ...)` 你可以有任何标准类型, 它将在服务器启动之前进行验证，
因此不关心那里的任何性能成本，它在服务运行时的唯一事情是返回 `func(paramValue string) bool.`

```go
{param:string equal(iris)} , "iris" will be the argument here:
app.Macros().String.RegisterFunc("equal", func(argument string) func(paramValue string) bool {
    return func(paramValue string){ return argument == paramValue }
})
```

Example Code:

```go
app := iris.New()
// you can use the "string" type which is valid for a single path parameter that can be anything.
app.Get("/username/{name}", func(ctx iris.Context) {
    ctx.Writef("Hello %s", ctx.Params().Get("name"))
}) // type is missing = {name:string}

// Let's register our first macro attached to int macro type.
// "min" = the function
// "minValue" = the argument of the function
// func(string) bool = the macro's path parameter evaluator, this executes in serve time when
// a user requests a path which contains the :int macro type with the min(...) macro parameter function.
app.Macros().Int.RegisterFunc("min", func(minValue int) func(string) bool {
    // do anything before serve here [...]
    // at this case we don't need to do anything
    return func(paramValue string) bool {
        n, err := strconv.Atoi(paramValue)
        if err != nil {
            return false
        }
        return n >= minValue
    }
})

// http://localhost:8080/profile/id>=1
// this will throw 404 even if it's found as route on : /profile/0, /profile/blabla, /profile/-1
// macro parameter functions are optional of course.
app.Get("/profile/{id:int min(1)}", func(ctx iris.Context) {
    // second parameter is the error but it will always nil because we use macros,
    // the validaton already happened.
    id, _ := ctx.Params().GetInt("id")
    ctx.Writef("Hello id: %d", id)
})

// to change the error code per route's macro evaluator:
app.Get("/profile/{id:int min(1)}/friends/{friendid:int min(1) else 504}", func(ctx iris.Context) {
    id, _ := ctx.Params().GetInt("id")
    friendid, _ := ctx.Params().GetInt("friendid")
    ctx.Writef("Hello id: %d looking for friend id: ", id, friendid)
}) // this will throw e 504 error code instead of 404 if all route's macros not passed.

// http://localhost:8080/game/a-zA-Z/level/0-9
// remember, alphabetical is lowercase or uppercase letters only.
app.Get("/game/{name:alphabetical}/level/{level:int}", func(ctx iris.Context) {
    ctx.Writef("name: %s | level: %s", ctx.Params().Get("name"), ctx.Params().Get("level"))
})

// let's use a trivial custom regexp that validates a single path parameter
// which its value is only lowercase letters.

// http://localhost:8080/lowercase/anylowercase
app.Get("/lowercase/{name:string regexp(^[a-z]+)}", func(ctx iris.Context) {
    ctx.Writef("name should be only lowercase, otherwise this handler will never executed: %s", ctx.Params().Get("name"))
})

// http://localhost:8080/single_file/app.js
app.Get("/single_file/{myfile:file}", func(ctx iris.Context) {
    ctx.Writef("file type validates if the parameter value has a form of a file name, got: %s", ctx.Params().Get("myfile"))
})

// http://localhost:8080/myfiles/any/directory/here/
// this is the only macro type that accepts any number of path segments.
app.Get("/myfiles/{directory:path}", func(ctx iris.Context) {
    ctx.Writef("path type accepts any number of path segments, path after /myfiles/ is: %s", ctx.Params().Get("directory"))
})

app.Run(iris.Addr(":8080"))
}
```

路径参数名称应仅包含字母。不允许使用“_”和数字等符号。

最后，不要将 `ctx.Params()` 和 `ctx.Values()` 混淆，
Path parameter's values goes to ctx.Params() and context's local storage that can be used to communicate between handlers and middleware(s) goes to ctx.Values().
******************************************************************************************************************************************************************
Reverse routes
-----------------------------

**路由与反向查找**

正如前面章节所提到的 Handlers，Iris提供一些handler祖册方法并返回一个 [`Route`](https://godoc.org/github.com/kataras/iris/core/router#Route) 实例

**路由命名**

路由命名很简单，我们只需要掉用 `Name` 字段定义其名称并返回 `*Route`

```go
package main

import (
	"github.com/kataras/iris"
)

func main() {
	app := iris.New()

	h := func(cxt iris.Context) {
		ctx.HTML("<b>Hi</b1>")
	}

	home := app.get("/", h)

	home.Name = "home"

	app.Get("/about", h).Name = "about"
	app.Get("/page/{id}", h).Name = "page"

	app.Run(iris.Addr(":8080"))
}
```

**路由反查 AKA 从路由名称生成URLs**

当我们为特定 path 注册 handlers 时，我们能够根据传递给Iris的结构化数据创建URL。
在上面的例子中，我们命名了三个路由，其中一个甚至带参数，如果我们使用的时默认`html/template` 视图引擎，
我们可以使用一个简单的操作来反查路由（并生成实际的URL）

```go
Home: {{ urlpath "home" }}
About: {{ urlpath "about" }}
Page 17: {{ urlpath "page" "17" }}
```

上面的代码将生成以下输出：

```go
Home: http://localhost:8080/ 
About: http://localhost:8080/about
Page 17: http://localhost:8080/page/17
```

**在代码中使用路由名称**

我们可以使用以下 function/methods 来处理命名的路由（及路由参数）

- [`GetRoutes`](https://godoc.org/github.com/kataras/iris/core/router#APIBuilder.GetRoutes)函数获取所有的注册路由
- [`GetRoute(routeName string)`](https://godoc.org/github.com/kataras/iris/core/router#APIBuilder.GetRoute)方法按名称检索路由
- [`URL(routeName string, paramValues ...interface{})`](https://godoc.org/github.com/kataras/iris/core/router#RoutePathReverser.URL)方法根据提供的参数生成url字符串
- [`Path(routeName string, paramValues ...interface{}`](https://godoc.org/github.com/kataras/iris/core/router#RoutePathReverser.Path)方法，根据提供的值生成部份URL（没有主机和协议）

**栗子**

有关详情请移步[kataras/iris/_examples/view/template_html_4](https://github.com/kataras/iris/tree/master/_examples/view/template_html_4)
***********************************************************************************************************************************************************************
Middleware
-----------------------------

当我们在Iris中讨论中间件时，我们讨论的是HTTP求情生命周期中，代码运行之前或之后。
例如，日志记录中间件可能会将传入的请求详细信息写入日志，然后调用 handler code，
在编写有关日志响应的详细信息之前，关于中间件的一个很酷的事情是这些单元非常灵活和可重用。

中间件只是Handler的一种形式 `func(ctx iris.Context)`，当调用 `ctx.Next()` 的时候中间件正在运行，这可以用于登录验证，
假如用户已经登录执行 `ctx.Next()`, 否则响应相关错误返回。

**编写中间件**

```go
package mian

import (
	"github.com/kataras/iris"
)

func mian() {
	app := iris.New()

    app.Get("/", before, mainHandler, after)

    app.Run(iris.Addr(":8080"))
}

func before(cxt iris.Context) {
	shareInformation := "this is a sharable information between handlers"

	requestPath := cxt.Path()

	println("Before the mainHandler: " + requestPath)

	ctx.Values().GetString("info")

	// 执行下一个处理程序
	ctx.Next()
}

func after(cxt iris.Context) {
	println("After the mainHandler")	
}

func mainHandler(cxt iris.Context) {
	println("Inside mainHandler")

	// 在 'before' 处理之前得到 info
	info := cxt.Values().GetString("info")

	cxt.HTML("<h1>Response</h1>")
	cxt.HTML("<br/> Info: " + info)

	// 执行 'after'
	ctx.Next()
}
```

```
$ go run main.go # and navigate to the http://localhost:8080
Now listening on: http://localhost:8080
Application started. Press CTRL+C to shut down.
Before the mainHandler: /
Inside mainHandler
After the mainHandler
```

**全局中间件**

```go
package main

import "github.com/kataras/iris"

func main() {
    app := iris.New()
    
    // 最先执行
    app.Use(before)
    
    // 最后执行
    app.Done(after)

   	// 注册路由
    app.Get("/", indexHandler)
    app.Get("/contact", contactHandler)

    app.Run(iris.Addr(":8080"))
}

func before(ctx iris.Context) {
     // [...]
}

func after(ctx iris.Context) {
    // [...]
}

func indexHandler(ctx iris.Context) {
    
    ctx.HTML("<h1>Index</h1>")

    // 执行 `Done` 注册的 after
    ctx.Next() 
}

func contactHandler(ctx iris.Context) {
    
    ctx.HTML("<h1>Contact</h1>")

     // 执行 `Done` 注册的 after
    ctx.Next() 
}
```

**探索**

你可以去看一下一些实用的源代码进行学习

| Middleware | Example |
| -----------|-------------|
| [basic authentication](basicauth) | [iris/_examples/authentication/basicauth](https://github.com/kataras/iris/tree/master/_examples/authentication/basicauth) |
| [Google reCAPTCHA](recaptcha) | [iris/_examples/miscellaneous/recaptcha](https://github.com/kataras/iris/tree/master/_examples/miscellaneous/recaptcha) |
| [localization and internationalization](i18n) | [iris/_examples/miscellaneous/i81n](https://github.com/kataras/iris/tree/master/_examples/miscellaneous/i18n) |
| [request logger](logger) | [iris/_examples/http_request/request-logger](https://github.com/kataras/iris/tree/master/_examples/http_request/request-logger) |
| [profiling (pprof)](pprof) | [iris/_examples/miscellaneous/pprof](https://github.com/kataras/iris/tree/master/_examples/miscellaneous/pprof) |
| [recovery](recover) | [iris/_examples/miscellaneous/recover](https://github.com/kataras/iris/tree/master/_examples/miscellaneous/recover) |

一些真正帮助您完成特定任务的中间件

| Middleware | Description | Example |
| -----------|--------|-------------|
| [jwt](https://github.com/iris-contrib/middleware/tree/master/jwt) | Middleware checks for a JWT on the `Authorization` header on incoming requests and decodes it. | [iris-contrib/middleware/jwt/_example](https://github.com/iris-contrib/middleware/tree/master/jwt/_example) |
| [cors](https://github.com/iris-contrib/middleware/tree/master/cors) | HTTP Access Control. | [iris-contrib/middleware/cors/_example](https://github.com/iris-contrib/middleware/tree/master/cors/_example) |
| [secure](https://github.com/iris-contrib/middleware/tree/master/secure) | Middleware that implements a few quick security wins. | [iris-contrib/middleware/secure/_example](https://github.com/iris-contrib/middleware/tree/master/secure/_example/main.go) |
| [tollbooth](https://github.com/iris-contrib/middleware/tree/master/tollboothic) | Generic middleware to rate-limit HTTP requests. | [iris-contrib/middleware/tollbooth/_examples/limit-handler](https://github.com/iris-contrib/middleware/tree/master/tollbooth/_examples/limit-handler) |
| [cloudwatch](https://github.com/iris-contrib/middleware/tree/master/cloudwatch) |  AWS cloudwatch metrics middleware. |[iris-contrib/middleware/cloudwatch/_example](https://github.com/iris-contrib/middleware/tree/master/cloudwatch/_example) |
| [new relic](https://github.com/iris-contrib/middleware/tree/master/newrelic) | Official [New Relic Go Agent](https://github.com/newrelic/go-agent). | [iris-contrib/middleware/newrelic/_example](https://github.com/iris-contrib/middleware/tree/master/newrelic/_example) |
| [prometheus](https://github.com/iris-contrib/middleware/tree/master/prometheus)| Easily create metrics endpoint for the [prometheus](http://prometheus.io) instrumentation tool | [iris-contrib/middleware/prometheus/_example](https://github.com/iris-contrib/middleware/tree/master/prometheus/_example) |
| [casbin](https://github.com/iris-contrib/middleware/tree/master/casbin)| An authorization library that supports access control models like ACL, RBAC, ABAC | [iris-contrib/middleware/casbin/_examples](https://github.com/iris-contrib/middleware/tree/master/casbin/_examples) |
******************************************************************************************************************
Wrapping the Router
-----------------------------

非常罕见，您可能永远不需要它，但无论如何您都需要它。

有时您需要覆盖或决定是否将在传入请求上执行路由，如果您以前有过使用 `net/http` 和其他Web框架的经验，
那么这个函数将对您很熟悉（它具有 net/http 中间件的形式，但它不接受下一个处理，而是接受路由作为要执行的功能）

```go
// WrapperFunc is used as an expected input parameter signature
// for the WrapRouter. It's a "low-level" signature which is compatible
// with the net/http.
// It's being used to run or no run the router based on a custom logic.
type WrapperFunc func(w http.ResponseWriter, r *http.Request, firstNextIsTheRouter http.HandlerFunc)

// WrapRouter adds a wrapper on the top of the main router.
// Usually it's useful for third-party middleware
// when need to wrap the entire application with a middleware like CORS.
//
// Developers can add more than one wrappers,
// those wrappers' execution comes from last to first.
// That means that the second wrapper will wrap the first, and so on.
//
// Before build.
func WrapRouter(wrapperFunc WrapperFunc)
```

Iris的路由根据 `HTTP Method` 搜索其路由，路由装配器可以覆盖该行为并执行自定义代码。

```go
package main

import (
    "net/http"
    "strings"

    "github.com/kataras/iris"
)

// In this example you'll just see one use case of .WrapRouter.
// You can use the .WrapRouter to add custom logic when or when not the router should
// be executed in order to execute the registered routes' handlers.
//
// To see how you can serve files on root "/" without a custom wrapper
// just navigate to the "file-server/single-page-application" example.
//
// This is just for the proof of concept, you can skip this tutorial if it's too much for you.


func main() {
    app := iris.New()

    app.OnErrorCode(iris.StatusNotFound, func(ctx iris.Context) {
        ctx.HTML("<b>Resource Not found</b>")
    })

    app.Get("/", func(ctx iris.Context) {
        ctx.ServeFile("./public/index.html", false)
    })

    app.Get("/profile/{username}", func(ctx iris.Context) {
        ctx.Writef("Hello %s", ctx.Params().Get("username"))
    })

    // serve files from the root "/", if we used .StaticWeb it could override
    // all the routes because of the underline need of wildcard.
    // Here we will see how you can by-pass this behavior
    // by creating a new file server handler and
    // setting up a wrapper for the router(like a "low-level" middleware)
    // in order to manually check if we want to process with the router as normally
    // or execute the file server handler instead.

    // use of the .StaticHandler
    // which is the same as StaticWeb but it doesn't
    // registers the route, it just returns the handler.
    fileServer := app.StaticHandler("./public", false, false)

    // wrap the router with a native net/http handler.
    // if url does not contain any "." (i.e: .css, .js...)
    // (depends on the app , you may need to add more file-server exceptions),
    // then the handler will execute the router that is responsible for the
    // registered routes (look "/" and "/profile/{username}")
    // if not then it will serve the files based on the root "/" path.
    app.WrapRouter(func(w http.ResponseWriter, r *http.Request, router http.HandlerFunc) {
        path := r.URL.Path
        // Note that if path has suffix of "index.html" it will auto-permant redirect to the "/",
        // so our first handler will be executed instead.

        if !strings.Contains(path, ".") { 
            // if it's not a resource then continue to the router as normally. <-- IMPORTANT
            router(w, r)
            return
        }
        // acquire and release a context in order to use it to execute
        // our file server
        // remember: we use net/http.Handler because here we are in the "low-level", before the router itself.
        ctx := app.ContextPool.Acquire(w, r)
        fileServer(ctx)
        app.ContextPool.Release(ctx)
    })

    // http://localhost:8080
    // http://localhost:8080/index.html
    // http://localhost:8080/app.js
    // http://localhost:8080/css/main.css
    // http://localhost:8080/profile/anyusername
    app.Run(iris.Addr(":8080"))

    // Note: In this example we just saw one use case,
    // you may want to .WrapRouter or .Downgrade in order to bypass the iris' default router, i.e:
    // you can use that method to setup custom proxies too.
    //
    // If you just want to serve static files on other path than root
    // you can just use the StaticWeb, i.e:
    //                          .StaticWeb("/static", "./public")
    // ________________________________requestPath, systemPath
}
```

这里没有太多要说的，它只是一个函数包装器，它接受本机响应编写和请求以及下一个处理程序，
即Iris的路由本身，无论是否被调用它是否被执行，它是整个中间件路由。