---
title: Go http server (I)
cover: /images/http/go_web_cover.jpg
author: 
  nick: Lovae
date: 2018-5-29

# 首页每篇文章的子标题
subtitle: Go package net/http/server.go 阅读笔记（I）
categories: HTTP

---
## Go http server (I) 源码阅读

> 这个系列会写三到四篇文章，第一篇是 go sdk 里 net/http/server.go 的阅读笔记，之后会写一下如何利用 server.go 的接口自定义一个简易通用的 HTTP server 框架。

### example

先从一个简单的例子开始吧：

```go
package main

import (
	"net/http"
	"fmt"
	"log"
)

//开启web服务
func test() {
	http.HandleFunc("/", sayHello)
	err := http.ListenAndServe(":9090", nil) // 注意这里第二个参数为 nil
	if err != nil {
		log.Fatal("ListenAndServer:", err)
	}
}

func sayHello(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintf(w, "Hello Guest!")
}

func main() {
	test()
}
```

运行代码，此时浏览器访问`localhost:9090`就会看到输出 “Hello Guest!”，其实访问`localhost:9090/`+任意字符串，都能得到结果。这段代码先用`http.HandleFunc`注册了一个处理函数，然后调用`http.ListenAndServe`监听端口，当有请求到来时，会根据访问路径找到并执行对应的处理函数。

我们通常还能看到另一种写法：

```go
package main

import (
	"net/http"
	"fmt"
	"log"
)

//开启web服务
func test() {
  http.Handle("/", &handler{})
	err := http.ListenAndServe(":9090", nil) //
	if err != nil {
		log.Fatal("ListenAndServer:", err)
	}
}

func sayHello(w http.ResponseWriter, r *http.Request) {...}

func main() {
	test()
}

type handler struct{}

func (h *handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	sayHello(w, r)
}
```

这段代码效果一样。区别就是`http.HandleFunc`和`http.Handle`需要的第二个参数，前者要一个`func (w http.ResponseWriter, r *http.Request)`函数，后者要一个实现了该函数的结构体。

```go
func (mux *ServeMux) HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	mux.Handle(pattern, HandlerFunc(handler))
}

func Handle(pattern string, handler Handler) { DefaultServeMux.Handle(pattern, handler) }

func HandleFunc(pattern string, handler func(ResponseWriter, *Request)) {
	DefaultServeMux.HandleFunc(pattern, handler)
}
```

可以看到，两个函数都会调用`mux.handle`

```go
func (mux *ServeMux) Handle(pattern string, handler Handler)
```

第二个参数是Handler，是一个接口:

```go
type Handler interface {
	ServeHTTP(ResponseWriter, *Request)
}
```

现在回到上面的`HandleFunc`,注意这个：`HandlerFunc(handler)`,这里很容易让人误以为HandlerFunc是一个函数并且包装了传入的handler，再返回一个`Handler`类型。而实际上这里是**类型转换**，来看HandlerFunc的定义：

```go
type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

虽然`HandlerFunc`的类型是一个函数，但它是一种类型，因为是以`type`来定义而不是`func`，并且实现了`ServeHTTP(w ResponseWriter, r *Request)`，在这个函数里，它又调用了自身。这个细节是十分重要的，因为这一步关乎到当路由规则匹配时，相应的响应方法是否会被调用的问题！这里的类型转换用法使一个函数自身实现了一个接口，就不用每次都要先写一个本身无用结构体，再用结构体实现接口。请仔细体会这种技巧！

。。。有点扯偏了，这里记住 Handler 这个接口是 go 语言 HTTP 服务最最最重要的接口，官方库和第三方库都按照这个接口来扩展。

### Server

来看一下 Server 这个结构体吧, 这里我只列出了几个核心的域：

```go
type Server struct {
	Addr      string      // TCP address to listen on, ":http" if empty
	Handler   Handler     // handler to invoke, http.DefaultServeMux if nil
	TLSConfig *tls.Config // optional TLS config, used by ServeTLS and ListenAndServeTLS

	listeners  map[net.Listener]struct{}
	onShutdown []func()
}
```

#### handler

这里主要关注 Handler，这个 Handler 就是刚刚的那个接口，可以在创建 Server 时传入，也可以在调用 Server.ListenAndServe 时传入：

```go
func ListenAndServe(addr string, handler Handler) error {
	server := &Server{Addr: addr, Handler: handler}
	return server.ListenAndServe()
}
```

 这个 handler 是在建立连接后收到客户端请求时用到：

```go
func (c *conn) serve(ctx context.Context) { // conn 指当前连接
    ...
	for {
		w, err := c.readRequest(ctx)
		...
		serverHandler{c.server}.ServeHTTP(w, w.req)
		...
	}
}

type serverHandler struct {
	srv *Server
}

func (sh serverHandler) ServeHTTP(rw ResponseWriter, req *Request) {
	handler := sh.srv.Handler
	if handler == nil {
		handler = DefaultServeMux
	}
	if req.RequestURI == "*" && req.Method == "OPTIONS" {
		handler = globalOptionsHandler{}
	}
	handler.ServeHTTP(rw, req)
}
```

从 serverHandler 的 ServeHTTP 函数可以看到，当 server.handler==nil 时，使用内部全局变量，也就是前面提到过的 DefaultServeMux。也就是说，我们在收到请求时通过这个 handler 来执行自己的逻辑代码，所以这个 handler 必须包含路由功能，并且能够执行路由对应的处理函数。同时我们用的第三方 HTTP server 框架(echo、beego…)也是通过自定义 handler 来实现功能扩展。这也是 Handler 这个接口是最最重要的接口的原因。

关于 DefaultServeMux 和 自定义的 handler，会在之后详细讨论。接下来回到 Server 本身。

#### Server.Serve

在主函数中可以调用 http.ListenAndServe 或者 http.Serve 来开始 HTTP 服务， 原理都一样：

```go
func (srv *Server) ListenAndServe() error {
	addr := srv.Addr
	if addr == "" {
		addr = ":http"
	}
	ln, err := net.Listen("tcp", addr)
	if err != nil {
		return err
	}
	return srv.Serve(tcpKeepAliveListener{ln.(*net.TCPListener)})
}
```

仔细看下 srv.Serve 的实现：

```go
func (srv *Server) Serve(l net.Listener) error {
	defer l.Close()
    
	var tempDelay time.Duration // how long to sleep on accept failure

	if err := srv.setupHTTP2_Serve(); err != nil {// 如果设置了 http2，就使用 http2 服务，
		return err
	}

	baseCtx := context.Background() // base is always background, per Issue 16220
	ctx := context.WithValue(baseCtx, ServerContextKey, srv)
	for {
		rw, e := l.Accept() // 这里会等待新的连接的建立，会阻塞在这里。
		if e != nil {
			select {
			case <-srv.getDoneChan():
				return ErrServerClosed
			default:
			}
			if ne, ok := e.(net.Error); ok && ne.Temporary() {
				if tempDelay == 0 {
					tempDelay = 5 * time.Millisecond
				} else {
					tempDelay *= 2
				}
				if max := 1 * time.Second; tempDelay > max {
					tempDelay = max
				}
				srv.logf("http: Accept error: %v; retrying in %v", e, tempDelay)
				time.Sleep(tempDelay)
				continue
			}
			return e
		}
		tempDelay = 0
		c := srv.newConn(rw)
		c.setState(c.rwc, StateNew) // before Serve can return
		go c.serve(ctx)
	}
}
```

这里要详细解释一下的就是 Accept 返回的 error 了。有以下几种可能：

* Accept 的时候 Server 由于某种原因停止了
* 收到系统信号产生中断，当然如果 返回的是 EINTR 表示可以重新调用
* 之前断掉的连接在短时间被重用了，此时该连接处于 **TIME_WAIT** 状态，新连接暂时不可用。可参考[这里](https://blog.littlechao.top/2018/04/05/tcp_faq/)

对于暂时性的错误，可以稍等一会儿，所以会出现 sleep。如果成功拿到 conn，先标记连接状态，然后创建新 goroutine 开始对连接服务。

#### conn.serve

> 这里由于 HTTPS 和 HTTP2 本身比较复杂，主要讨论 HTTP1.1 的实现。

```go
func (c *conn) serve(ctx context.Context) {
	c.remoteAddr = c.rwc.RemoteAddr().String()
	ctx = context.WithValue(ctx, LocalAddrContextKey, c.rwc.LocalAddr())
	defer func() {
		if err := recover(); err != nil && err != ErrAbortHandler {
			const size = 64 << 10
			buf := make([]byte, size)
			buf = buf[:runtime.Stack(buf, false)]
			c.server.logf("http: panic serving %v: %v\n%s", c.remoteAddr, err, buf)
		}
		if !c.hijacked() {
			c.close()
			c.setState(c.rwc, StateClosed)
		}
	}()

	if tlsConn, ok := c.rwc.(*tls.Conn); ok {
		// 处理 https
	}

	// HTTP/1.x from here on.

	ctx, cancelCtx := context.WithCancel(ctx)
	c.cancelCtx = cancelCtx
	defer cancelCtx()

	c.r = &connReader{conn: c}
	c.bufr = newBufioReader(c.r)
	c.bufw = newBufioWriterSize(checkConnErrorWriter{c}, 4<<10)

	for { // 同一个连接有多个请求，循环处理
		w, err := c.readRequest(ctx) // 读取请求，会阻塞
		if c.r.remain != c.server.initialReadLimitSize() {
			// If we read any bytes off the wire, we're active.
			c.setState(c.rwc, StateActive)
		}
		if err != nil {
			// handle error
			return
		}

		// Expect 100 Continue support
		req := w.req
		if req.expectsContinue() {
			if req.ProtoAtLeast(1, 1) && req.ContentLength != 0 {
				// Wrap the Body reader with one that replies on the connection
				req.Body = &expectContinueReader{readCloser: req.Body, resp: w}
			}
		} else if req.Header.get("Expect") != "" {
			w.sendExpectationFailed()
			return
		}

		c.curReq.Store(w)

		if requestBodyRemains(req.Body) { // 支持管线化，处理当前请求时可能还在接收请求
			registerOnHitEOF(req.Body, w.conn.r.startBackgroundRead)
		} else {
			if w.conn.bufr.Buffered() > 0 {
				w.conn.r.closeNotifyFromPipelinedRequest()
			}
			w.conn.r.startBackgroundRead()
		}

		serverHandler{c.server}.ServeHTTP(w, w.req) // 这里就是之前提到的，自定义处理的入口
		w.cancelCtx()
		if c.hijacked() {
			return
		}
		w.finishRequest() // 把数据 flush 到网络层，此次请求在应用层结束
		if !w.shouldReuseConnection() {
			if w.requestBodyLimitHit || w.closedRequestBodyEarly() {
				c.closeWriteAndWait() // 发送 TCP FIN ，关闭连接
			}
			return
		}
		
		...
        
		if d := c.server.idleTimeout(); d != 0 { // 设置空闲超时，超时后关闭连接
			c.rwc.SetReadDeadline(time.Now().Add(d))
			if _, err := c.bufr.Peek(4); err != nil {
				return
			}
		}
		c.rwc.SetReadDeadline(time.Time{})
	}
}
```

这里代码比较复杂，包含了比较完整的 HTTP、HTTPs、HTTP2 协议的实现，建议了解了协议的内容再来看具体实现。代码协议的细节部分代码就不详细谈了，我们需要理解的是 创建 listener，从 Accept 拿到连接，等待并读取到 request，用 handler 处理 request 并把结果或错误信息写到 response 的过程。

需要注意的是，我们所讨论的是 go 语言官方库的 HTTP 的实现，这里的发送和接收数据都是指的发给下层传输层和从传输层接收，也就是调用 socket 接口，一定要分清楚各个层次。
