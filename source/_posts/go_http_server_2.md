---
title: Go http server (II)
cover: /images/http/go_web_cover.jpg
author: 
  nick: Lovae
date: 2018-06-01

# 首页每篇文章的子标题
subtitle: Go package net/http/server.go 阅读笔记（II）
categories: HTTP
---
## GO http server (II) Server.Handler

>  上一篇里讨论了 go 官方库里提供的 http 服务框架，使用者需要关心的是 Server 的 handler 域。当 Server 调用 Serve 函数时 Server.Handler 为 nil，则默认使用 http.DefaultServeMux 作为 handler。

### DefaultServeMux

来看一下它的定义和描述：

```go
// ServeMux is an HTTP request multiplexer.
// It matches the URL of each incoming request against a list of registered
// patterns and calls the handler for the pattern that
// most closely matches the URL.
```

简单的说，它就是一个路由分发器。

##### 路由注册

```go
type ServeMux struct {
	mu    sync.RWMutex
	m     map[string]muxEntry
  	//路由规则，一个string对应一个mux实例对象，map的key就是注册的路由表达式(string类型的)
	hosts bool // whether any patterns contain hostnames
}

var defaultServeMux ServeMux
var DefaultServeMux = &defaultServeMux

type muxEntry struct { // 代表着一个 路由-处理函数 组合
	explicit bool //表示 patern 是否已经被明确注册过了
	h        Handler
	pattern  string //路由表达式
}
```

之前提到过，Server.Handler 需要有路由功能，并且可以执行路由对应的处理函数。当注册路由时，调用`mux.Handle`:

```go
func (mux *ServeMux) Handle(pattern string, handler Handler) {
	mux.mu.Lock()
	defer mux.mu.Unlock()

	if pattern == "" {
		panic("http: invalid pattern " + pattern)
	}
	if handler == nil {
		panic("http: nil handler")
	}
	if mux.m[pattern].explicit {
		panic("http: multiple registrations for " + pattern)
	}

	if mux.m == nil {
		mux.m = make(map[string]muxEntry)
	}
	mux.m[pattern] = muxEntry{explicit: true, h: handler, pattern: pattern}

	if pattern[0] != '/' {
		mux.hosts = true
	}

  	// 以下是很有用的功能:当pattern == “/tree/”时,
  	// 会插入一条永久的重定向到“/tree”,注意最后的斜杠。
  	// 当然前提是在这之前没有“/tree”这条路由
	n := len(pattern)
	if n > 0 && pattern[n-1] == '/' && !mux.m[pattern[0:n-1]].explicit {
		//如果包含host，
		path := pattern
		if pattern[0] != '/' {
			// In pattern, at least the last character is a '/', so
			// strings.Index can't be -1.
			path = pattern[strings.Index(pattern, "/"):]
		}
		url := &url.URL{Path: path}
		mux.m[pattern[0:n-1]] = muxEntry{h: RedirectHandler(url.String(), StatusMovedPermanently), pattern: pattern}
	}
}
```

代码挺多，其实主要就做了一件事，向`DefaultServeMux`的`map[string]muxEntry`中增加对应的路由规则和`handler`。注意这里每条路由并没有包含我们常说的 GET、POST 等等区别，主要有两个原因：一是为了简洁，很多开发者偏好不同的处理方法，官方库只提供最基本的功能；二是不直接和请求方法绑定起来便于写 RESTful API。

但是这里还要注意路径结尾的`/`,这时候该路径为一个子树，如果能完全匹配到其子路由，那么也能匹配到这个子树，不过路由越长，优先级越大；如果不能完全匹配到其子路由，会匹配到这个子树的路由。比如有一个根路由`/`、`/example/`和 `/example/1`，那么访问`/example/2`时，会匹配到`/example/`，访问`/nothing`会匹配到`/`。

##### 处理路由请求

注册好路由，并且没有使用别的 handler 时，DefaultServerMux 的 ServeHTTP 就会在接收到 request 时被调用。

```go
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

ServeHTTP 主要从之前注册好的路由表中获取对应的 handler：

```go
func (mux *ServeMux) ServeHTTP(w ResponseWriter, r *Request) {
   ...
   h, _ := mux.Handler(r) // 匹配和 request 最接近的路由，拿到对应的 handler
   h.ServeHTTP(w, r)
}

func (mux *ServeMux) Handler(r *Request) (h Handler, pattern string) {
   ...
   host := stripHostPort(r.Host)
   path := cleanPath(r.URL.Path)
   if path != r.URL.Path {
      _, pattern = mux.handler(host, path)
      url := *r.URL
      url.Path = path
      return RedirectHandler(url.String(), StatusMovedPermanently), pattern
      //注意这里的重定向 handler
   }

   return mux.handler(host, r.URL.Path)
}

func (mux *ServeMux) handler(host, path string) (h Handler, pattern string) {
	mux.mu.RLock()
	defer mux.mu.RUnlock()

	// Host-specific pattern takes precedence over generic ones
	if mux.hosts {
		h, pattern = mux.match(host + path)
	}
	if h == nil {
		h, pattern = mux.match(path) // match 做的是字符串匹配的工作
	}
	if h == nil {
		h, pattern = NotFoundHandler(), "" 
	}
	return
}
```

在没有找到匹配的路由时，返回 NotFoundHandler， 默认只是写入 404 not found，但通常我们会自定义它，然后返回一个专门的好看的 404 页面。

如果需要重定向，则会通过返回的 redirectHandler 调用 Redirect：

```go
func Redirect(w ResponseWriter, r *Request, url string, code int) {
   if u, err := parseURL(url); err == nil {
      // If url was relative, make absolute by
      // combining with request path.
      // The browser would probably do this for us,
      // but doing it ourselves is more reliable.

      // NOTE(rsc): RFC 2616 says that the Location
      // line must be an absolute URI, like
      // "http://www.google.com/redirect/",
      // not a path like "/redirect/".
      // Unfortunately, we don't know what to
      // put in the host name section to get the
      // client to connect to us again, so we can't
      // know the right absolute URI to send back.
      // Because of this problem, no one pays attention
      // to the RFC; they all send back just a new path.
      // So do we.
      if u.Scheme == "" && u.Host == "" {
         oldpath := r.URL.Path
         if oldpath == "" { // should not happen, but avoid a crash if it does
            oldpath = "/"
         }

         // no leading http://server
         if url == "" || url[0] != '/' {
            // make relative path absolute
            olddir, _ := path.Split(oldpath)
            url = olddir + url
         }

         var query string
         if i := strings.Index(url, "?"); i != -1 {
            url, query = url[:i], url[i:]
         }

         // clean up but preserve trailing slash
         trailing := strings.HasSuffix(url, "/")
         url = path.Clean(url)
         if trailing && !strings.HasSuffix(url, "/") {
            url += "/"
         }
         url += query
      }
   }

   w.Header().Set("Location", hexEscapeNonASCII(url))
   w.WriteHeader(code)

   // RFC 2616 recommends that a short note "SHOULD" be included in the
   // response because older user agents may not understand 301/307.
   // Shouldn't send the response for POST or HEAD; that leaves GET.
   if r.Method == "GET" {
      note := "<a href=\"" + htmlEscape(url) + "\">" + statusText[code] + "</a>.\n"
      fmt.Fprintln(w, note)
   }
}
```

可以看到，DefaultServerMux 只有一个最基本的路由功能，是一个最简单的 HTTP 服务框架。可是这通常不能满足我们的需求，于是我们可以根据我们自己的需要自定义一个简单通用的 HTTP Server 框架。