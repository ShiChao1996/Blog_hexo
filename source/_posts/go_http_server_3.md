---
title: Go http server (III)
cover: /images/http/go_web_cover.jpg
author: 
  nick: Lovae
date: 2018-06-02

# 首页每篇文章的子标题
subtitle: 利用官方库和一些三方库来定制一个简易合用的 HTTP Server 框架
categories: HTTP
---
## GO http server (III) 组建简易 HTTP Server 框架

> 上篇提到 DefaultServerMux 作为默认的 HTTP Server 框架太过简单，缺少很多功能。这篇我们利用官方库和一些三方库来定制一个简易合用的 HTTP Server 框架。完整代码见[这里](https://github.com/TechcatsLab/apix)

### Router

首先要有 router 模块，这里我使用第三方 gorilla 框架的最小化路由模块 mux，它的作用和 DefaultServerMux 差不多，只不过支持了 RESTful API。

在添加路由和对应 handler 时，很可能我们写的处理函数有 bug，导致没有往 response 里写入内容就返回，这会造成客户端阻塞等待，所以当出现错误提前返回时，需要一个默认的错误处理函数，给客户端返回默认错误信息。

```go
import (
	"net/http"

	"github.com/gorilla/mux"
)
type Router struct {
   router     *mux.Router
   ctxPool    sync.Pool
   errHandler func(w http.responseWriter, r *http.request) 
}
```

很多时候，执行路由对应 handler 时我们并不想直接操作 http.responseWriter 和 *http.request，并且希望有一些简单的封装，提供更多的功能。再者，这两个对象并不能很好的携带中间件处理过程中产生的一些参数。所以我们会定义一个 Context （下一节）来封装它们。每一个请求都应该有一个 Context，为了方便的管理，使用 sync.Pool 做一个 context 池。

创建新的 Router：

```go
// NewRouter returns a router.
func NewRouter() *Router {
   r := &Router{
      router:     mux.NewRouter(),
      errHandler: func(_ *Context) {},
   }

   r.ctxPool.New = func() interface{} {
      return NewContext(nil, nil)
   }

   r.router.NotFoundHandler = http.NotFoundHandler()
   r.router.MethodNotAllowedHandler = MethodNotAllowedHandler()

   return r
}
```

router 注册路由，由于使用 gorilla.mux，调用其 HandleFunc ，返回 router 本身，在调用 Method 即可指定请求方法。不过我们还可以在自己的 handler 执行之前，提供一些钩子，这里我们可以添加一些 filter 函数，以便功能扩展。

```go
type FilterFunc func(*Context) bool

func (rt *Router) Get(pattern string, handler HandlerFunc, filters ...FilterFunc) {
   rt.router.HandleFunc(pattern, rt.wrapHandlerFunc(handler, filters...)).Methods("GET")
}

// Post adds a route path access via POST method.
func (rt *Router) Post(pattern string, handler HandlerFunc, filters ...FilterFunc) {
   rt.router.HandleFunc(pattern, rt.wrapHandlerFunc(handler, filters...)).Methods("POST")
}

// Wraps a HandlerFunc to a http.HandlerFunc.
func (rt *Router) wrapHandlerFunc(f HandlerFunc, filters ...FilterFunc) http.HandlerFunc {
   return func(w http.ResponseWriter, r *http.Request) {
      c := rt.ctxPool.Get().(*Context)
      defer rt.ctxPool.Put(c)
      c.Reset(w, r)

      if len(filters) > 0 {
         for _, filter := range filters {
            if passed := filter(c); !passed {
               c.LastError = errFilterNotPassed
               return
            }
         }
      }

      if err := f(c); err != nil {
         c.LastError = err
         rt.errHandler(c)
      }
   }
}
```

### Context

前面提到可以用一个 Context 包装 http.responseWriter 和 *http.request，并且提供一些额外的功能。额外的功能如 validator，用来对请求做参数验证。这个 validator 我们可以直接用一个第三方库，也可以做成 Interface 以便升级。

另外我们可能需要 Context 能够携带额外的信息，所以可以加一个 map 用来存储。

```go
type Context struct {
   responseWriter http.ResponseWriter
   request        *http.Request
   Validator      *validator.Validate
   store          map[string]interface{}
}
```

不要忘了在 Router 里面我们是用一个线程安全的池来管理 context ，也就是每次用完 context 需要还回去来避免临时分配带来的开销。所以别忘了还回去之前需要把 context 重置成原来的样子。

```go
func (c *Context) Reset(w http.ResponseWriter, r *http.Request) {
   c.responseWriter = w
   c.request = r
   c.store = make(map[string]interface{})
}
```

### Server

有了 router 和 context，我们还需要封装一个 server。首先定义一个 EntryPoiont 结构体，当然名字随意。非常确认的是我们需要用到 http 包的 Server，还可以加上可能用到的 net.Listener。另外，我们需要方便的添加一些即插即用的工具，所以需要中间件，这里我使用第三方库 negroni 。然后我们可能需要一个通知关闭所有连接的机制，用一个 channel 可以做到。所以 EntryPoint 大致如下：

```go
type Entrypoint struct {
   server        *http.Server
   listener      net.Listener
   middlewares   []negroni.Handler
}
```

#### negroni

其实 negroni 的核心代码也很简单，就只是把多个 middleware 串起来使其能够串行调用。

```go
type Negroni struct {
   middleware middleware
   handlers   []Handler
}

type middleware struct {
	handler Handler
	next    *middleware
}

type Handler interface {
	ServeHTTP(rw http.ResponseWriter, r *http.Request, next http.HandlerFunc)
}
```

关键就是 Handler 接口，所有第三方实现的中间件要和 negroni 一起用的话，都要实现它，并且每个中间件执行完自己的功能后，要去调用 next 触发下一个中间件的执行。

添加中间件：

```go
func (n *Negroni) Use(handler Handler) {
   if handler == nil {
      panic("handler cannot be nil")
   }

   n.handlers = append(n.handlers, handler)
   n.middleware = build(n.handlers)
}

func build(handlers []Handler) middleware {
	var next middleware

	if len(handlers) == 0 {
		return voidMiddleware()
	} else if len(handlers) > 1 {
		next = build(handlers[1:])
	} else {
		next = voidMiddleware()
	}

	return middleware{handlers[0], &next}
}
```

添加中间件的时候，递归地调用 build ，把所有 middlewares 串起来。必然的，negroni 实现了 http.Handler 接口，这使得 Negroni 可以当做 http.Handler 传给 Server.Serve()

```go
func (n *Negroni) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
   n.middleware.ServeHTTP(NewResponseWriter(rw), r)
}

func (m middleware) ServeHTTP(rw http.ResponseWriter, r *http.Request) {
	m.handler.ServeHTTP(rw, r, m.next.ServeHTTP)
}
```

#### 整合 router

当所有中间件执行完了以后，应该把 context 传给 router 去执行对应的路由，所以把 router 作为最后一个中间件传到 negroni 。

```go
func (ep *Entrypoint) buildRouter(router http.Handler) http.Handler {
	n := negroni.New()

	for _, mw := range ep.middlewares {
		n.Use(mw)
	}

	n.Use(negroni.Wrap(http.HandlerFunc(router.ServeHTTP)))

	return n
}
```

当然在启动 Server.Serve() 之前，还要把 ep.buildRouter 返回的对象赋给 ep.Server.Handler，使这个对象代替 DefaultServerMux。

```go
func (ep *Entrypoint) prepare(router http.Handler) error {
   var (
      err       error
      listener  net.Listener
   ）

   listener, err = net.Listen("tcp", ep.configuration.Address)
   if err != nil {
      return err
   }

   ep.listener = listener
   ep.server = &http.Server{
      Addr:      ep.configuration.Address,
      Handler:   ep.buildRouter(router),
   }

   return nil
}
```

接下来就可以调用 start 跑起服务：

```go
func (ep *Entrypoint) Start(router http.Handler) error {
   if router == nil {
      return errNoRouter
   }

   if err := ep.prepare(router); err != nil {
      return err
   }

   go ep.startServer()

   fmt.Println("Serving on:", ep.configuration.Address)

   return nil
}
```

### 中间件封装

有的时候有一些现成的中间件，但是不能直接放到 negroni 里面用，就需要我们给它加一层封装。

例如，我们要做 jwt 验证，使用第三方的 *jwtmiddleware.JWTMiddleware，但是有的路径我们不需要 token，需要跳过 jwt 中间件。不方便改别人的代码，可以这样封装来代替原来的 *jwtmiddleware.JWTMiddleware：

```go
type Skipper func(path string) bool

// JWTMiddleware is a wrapper of go-jwt-middleware, but added a skipper func on it.
type JWTMiddleware struct {
   *jwtmiddleware.JWTMiddleware
   skipper Skipper
}
```

使用 *jwtmiddleware.JWTMiddleware 作为一个匿名变量，这样可以在自定义的 JWTMiddleware 上直接调用 *jwtmiddleware.JWTMiddleware 的函数。然后用 handler 函数覆盖原有的 HandlerWithNext 函数，这样就能通过调用时传入的 skipper 函数判断是否需要跳过 jwt：

```go
func (jm *JWTMiddleware) handler(w http.ResponseWriter, r *http.Request, next http.HandlerFunc) {
   path := r.URL.Path
   if skip := jm.skipper(path); skip {
      next(w, r)
      return
   }

   jm.HandlerWithNext(w, r, next)
}
```

最后用 negroni 包装一下，使它能够直接被 negroni 使用：

```go
func NegroniJwtHandler(key string, skipper Skipper, signMethod *jwt.SigningMethodHMAC, errHandler func(w http.ResponseWriter, r *http.Request, err string)) negroni.Handler {
   if signMethod == nil {
      signMethod = jwt.SigningMethodHS256
   }
   jm := jwtmiddleware.New(jwtmiddleware.Options{
      ValidationKeyGetter: func(token *jwt.Token) (interface{}, error) {
         return []byte(key), nil
      },
      SigningMethod: signMethod,
      ErrorHandler:  errHandler,
   })

   if skipper == nil {
      skipper = defaulSkiper
   }

   JM := JWTMiddleware{
      jm,
      skipper,
   }

   return negroni.HandlerFunc(JM.handler)
}
```

### 总结

目前为止我们实现了一个简易通用的 HTTP server 框架，虽然功能还不是很完善，不过好在可扩展性比较高，我们可以在此基础上任意扩展，可以添加上缓存、数据库、监控等等模块。

如果有兴趣的话，可以去看看 echo 的实现，其实也是大同小异。

最后，再放一遍项目[地址](https://github.com/TechCatsLab/apix)，还有一些别的库，欢迎 star 和 pr 啦！