---
title:  understand_go_context
cover: /images/go/cancelCtx.png
author: 
  nick: Lovae
date: 2018-7-15
categories: go sdk
tags: 
    - golang
subtitle: 深入理解 golang context

---
# 深入理解 Go Context

## 什么是 Context

Context 的最常见但也是最不准确的翻译是 ‘上下文’（因为程序里通常只需要上文），其实译为 ‘语境’ 更为合适，意思是当前说话的环境。最直观的作用是提供一些必要的信息：

> ...
>
> 唐僧：“悟空~”
>
> 
>
> question：唐僧的“悟空” 表达了怎样的心理？
>
> answer：。。。去你的

Context 的概念本身比较宽泛，从系统角度说，线程/进程 的切换时，需要先保存当前寄存器和栈指针，然后载入下一个 进程/线程 需要的寄存器和栈。寄存器和栈就是进程/线程的 Context。

在不同编程语言中，也有不同体现：

如 c 语言的 errno (摘自某呼)：

> 注意过 errno 这个全局变量的朋友会发现,这个全局变量其实有可能不是一个真正的变量.它返回了一个本地线程的存储空间.它实际上是每个线程有一份.这里,其实 C 语言运行时已经悄悄变成了多份,而对应当前线程的实例用本地线程保存,它就是一个 context

又例如 Javascript 在浏览器中运行就有浏览器作为环境提供 window 对象，而在 node.js 环境下面运行就没有 window 对象。



照此看来，Context 好像就是一个 ‘全局变量’，那为什么不直接声明全局变量，非要用 Context 这个生涩的概念呢？

* 在软件工程中，对全局变量基本持否定态度，一是是代码变得耦合，二是暴露了多余的信息，三是全局变量在多线程环境下使用锁浪费 CPU 资源。不过它有很好的效果：间接的提升了某些变量的作用域，保证了这些数据的生命周期
* 于是出现了 *不那么全局的全局变量* ，例如 *线程局部* 的全局变量(可以做到线程安全)或者 *包局部* 的全局变量。很多语言的 this ，其实也是如此。
* 另外还有匿名形式的 *闭包* 局部的全局变量

再结合轮子哥说的：

> 每一段程序都有很多外部变量。只有像 Add 这种简单的函数才是没有外部变量的。一旦你的一段程序有了外部变量，这段程序就不完整，不能独立运行。你为了使他们运行，就要给所有的外部变量一个一个写一些值进去。这些值的集合就叫 Context

那么我们可以认为，Context 就是把一些信息打包聚合到一起，形成一个模块交互的语境，各个模块**像传递包裹一样取用**它，而不是通过全局变量来访问它。

## Go 语言里的 Context

### Context 的使用

Go 语言的 Context 在携带信息的基础上，增加了非常实用的功能，设计也非常简洁巧妙。标准库提供了*可携带 value 的 Context*、*可取消的 Context* 和 *可超时的 Context* 。

#### 携带 value 的 Context

前面提到 Context 最基本的作用是携带语境中的一些信息，比如一些参数。但是问题来了，所有参数都要放到 Context 吗？哪些应该、哪些不应该？如果一个函数如下：

```go
func a(key string, value interface, id int){
    ...
}
```

如果把参数全都放到 context：

```go
func a(ctx context.Context){
    ...
}
```

前者我们可以一目了然的从函数签名中获取或猜出一些关于这个函数的大概信息，而后者只看函数签名获得不了什么信息，需要仔细的从代码里读。很明显，前者可读性更高。一个良好的 API 设计，应该从函数签名就清晰的理解函数的逻辑。

使用 Context 携带参数会让接口定义更加模糊。那么什么样的信息应该放到 Context 里呢？官方注释如下：

```go
Use context values only for request-scoped data that transits processes and API boundaries, not for passing optional parameters to functions.
```

也就是说，应该保存 Request 范畴的值：

- 任何关于 Context 自身的都是 Request 范畴的（这俩同生共死）
- 从 Request 数据衍生出来，并且随着 Request 的结束而终结

。。。好像这句话说了和没说差不多？在处理请求的时候，难道不是所有的信息都来自 Request ？

其实通常来说, Context.Value 应该是 **告知性质** 的东西，而不是 **控制性质** 的东西。

##### 哪些不是控制性质的？

- Request ID
  - 只是给每个 RPC 调用一个 ID，而没有实际意义
  - 这就是个数字/字符串，反正你也不会用其作为逻辑判断
  - 一般也就是日志的时候需要记录一下
    - 而 `logger` 本身不是 Request 范畴，所以 `logger` 不应该在 `Context` 里
    - 非 Request 范畴的 `logger` 应该只是利用 `Context` 信息来修饰日志
- User ID ，比如可以在 jwt 中间件解析出 userID 然后带在 Context 里再传给 controller。
- Incoming Request ID

##### 显然是控制性质的：

* 数据库连接
  - 显然会非常严重的影响逻辑
  - 因此这应该在函数参数里，明确表示出来
* ...

> **关于 可携带 value 的 Context，还有一个值得注意的地方是：Context 本身是不可变的（immutable），让一个 Context 携带新的参数并不是一个 “setter” 来修改 Context 值，而是通过“包含”的形式，生成一个新的 Context 包含原有 Context，形成链式结构。在下面实现的时候继续讨论。**

#### 可取消 和 可超时的 Context

**为什么要取消（超时的本质也是取消，只不过通过计时器触发取消操作）？**

这和 Go 语言的 goroutine 有关。当你在 c 程序中 fork 一个新的进程，你会得到一个 PID，你可以通过这个 PID 向它发送信号来停止它的运行。

可是当你启动一个 goroutine 时，你并不会得到一个这个‘线程‘的 ID，那么要如何才能关掉它呢？答案就是 *可取消的 Context*。

官方示例：

```go
package main

import (
    "context"
    "fmt"
    "time"
)

func main() {
    d := time.Now().Add(50 * time.Millisecond)
    ctx, cancel := context.WithDeadline(context.Background(), d)

    // Even though ctx will be expired, it is good practice to call its
    // cancelation function in any case. Failure to do so may keep the
    // context and its parent alive longer than necessary.
    defer cancel()

    select {
    case <-time.After(1 * time.Second):
        fmt.Println("overslept")
    case <-ctx.Done():
        fmt.Println(ctx.Err())
    }
}
```

#### 几点问题

当你搜索关于 Go Context 的博客的时候，通常你会看到一些规则：

> * 不要将 Context 放入结构体, Context 应该作为第一个参数传入，命名为 ctx.
> * 即使函数允许, 也不要传入nil 的 Context. 如果不知道用哪种 Context，可以使用 context.TODO().
> * 使用 context 的 Value 相关方法,只应该用于在程序和接口中传递和请求相关数据，不能用它来传递一些可选的参数
> * 相同的 Context 可以传递给在不同的 goroutine; Context 是并发安全的.

可是有几点问题：

* 为什么不应该放在结构体？

  最开始已经说明了，Context 最基本的作用，是对一些 *不那么全局的全局变量* 的打包，把它放到结构体，其生存周期和作用域是无法控制的，相当于把它变成了它所在包的一个全局变量。理想情况下，`Context` 存在于调用栈（Call Stack） 中，所以通过参数传递。

* 为什么 HTTP 包的 Request 结构体持有 context？

  Request 本身就是一堆参数的集合，只不过参数太多单独写成结构体了而已，这堆参数在请求结束时或者读写超时时（conn readTimeout/writeTimeout）就应该释放，需要一个可超时的 Context 来协助。那为什么不把请求参数都放在 Context 呢，这个问题前面已经讨论过了，可读性是非常重要的。

* 为什么是并发安全的？

  Context 本身的实现是不可变的（immutable），既然不可变，那当然是线程安全的。并且通过 Context.Done() 返回的通道可以协调 goroutine 的行为。

### Go 的 Context 实现

在标准库里，Context 是一个接口：

```go
type Context interface {
   Deadline() (deadline time.Time, ok bool)

   Done() <-chan struct{}

   Err() error
   
   Value(key interface{}) interface{}
}
```

而我们常用的 context.Background() 返回的是一个最基本的全局 context：background，是一个什么功能也没有的 emptyCtx：

```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
   return
}
func (*emptyCtx) Done() <-chan struct{} {
   return nil
}
func (*emptyCtx) Err() error {
   return nil
}
func (*emptyCtx) Value(key interface{}) interface{} {
   return nil
}

var (
   background = new(emptyCtx)
   todo       = new(emptyCtx)
)
```

其他所有的 Context 都应该衍生自这两个基本的 ctx，生成新的 context 的方式是找一个 ‘父亲’ ，然后复制它，再结合 value 或者 timer 生成新的 context。

#### withValue

```go
func WithValue(parent Context, key, val interface{}) Context {
   if key == nil {
      panic("nil key")
   }
   if !reflect.TypeOf(key).Comparable() {
      panic("key is not comparable")
   }
   return &valueCtx{parent, key, val} // 返回的是一个指针
}

type valueCtx struct {
   Context // 注意这里使用匿名域
   key, val interface{}
}
```

每次添加 value 不是改变了context ，而是在原有的 context 基础上重新生成一个，形成了一条链。获取 value 的时候是逆序的：

```go
func (c *valueCtx) Value(key interface{}) interface{} {
   if c.key == key {
      return c.val
   }
   return c.Context.Value(key)
}
```

先看最后一个节点的键值对，如果不是，那么沿着链往上查找:

![](/images/go/value_ctx_chain.png)

#### withCancel

由于可超时的 Context 是基于可取消的 Context 实现的，所以这里只讨论 cancelCtx：

```go

type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

type cancelCtx struct {
   Context

   mu       sync.Mutex            // 由于多个线程都可能执行 ctx.Cancel()，要加锁
   done     chan struct{}         // created lazily, closed by first cancel call
   children map[canceler]struct{} // 由于需要在父节点取消时取消其所有字节点，所以记录其所有可取消子节点
   err      error                 // set to non-nil by the first cancel call
}

func (c *cancelCtx) Done() <-chan struct{} {
   c.mu.Lock()
   if c.done == nil {
      c.done = make(chan struct{})
   }
   d := c.done
   c.mu.Unlock()
   return d
}
```

生成一个新的 可取消 Context 的时候，需要传入一个父 Context 节点，并且通过父节点找到祖先节点里面最近的一个可取消的 Context 节点，然后把自己记录在那个**祖先**节点的 children 里面，这样在祖先被 cancel 的时候，新的这个 Context 也会被取消。不过为什么是祖先节点而不是父节点呢？因为可能有如下情况(图中箭头方向代表生长方向)：

![](/images/go/cancelCtx.png)

其父节点可能不是可取消的，所以没法记录 children，所以不难理解代码了：

```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
   c := newCancelCtx(parent) // 生成一个新的可取消节点
   propagateCancel(parent, &c) // 找到可取消祖先并记录自己到祖先的 children
   return &c, func() { c.cancel(true, Canceled) }
}

// newCancelCtx returns an initialized cancelCtx.
func newCancelCtx(parent Context) cancelCtx {
   return cancelCtx{Context: parent}
}

// propagateCancel arranges for child to be canceled when parent is.
func propagateCancel(parent Context, child canceler) {
   if parent.Done() == nil {
       return // 这里尤其注意，parent.Done() 返回 nil，表示整个链上都没有可取消/可超时的 context。因为新的 Context 在包含父节点的时候，都是采用匿名字段，也就是说，如果新的 Context 本身没有某个函数，但是它的匿名字段上有那个函数，那么该函数是可以直接被新的 Context 调用的。如此就可以一直追溯到 background 节点，而正好这个根节点是有 Done() 这个函数，并且返回 nil。另外，不可能出现中间一个可取消 context 调用 Done() 返回 nil，看实现便知。
   }
   if p, ok := parentCancelCtx(parent); ok {
      p.mu.Lock()
      if p.err != nil {
         // parent has already been canceled
         child.cancel(false, p.err)
      } else {
         if p.children == nil {
            p.children = make(map[canceler]struct{})
         }
         p.children[child] = struct{}{}
      }
      p.mu.Unlock()
   } else {  // 没想通的是这里，什么情况会走到这步呢？
      go func() {
         select {
         case <-parent.Done():
            child.cancel(false, parent.Err())
         case <-child.Done():
         }
      }()
   }
}

func parentCancelCtx(parent Context) (*cancelCtx, bool) {
   for { // 沿着父节点往上找，直到找到一个 可取消的/可超时的 祖先节点
      switch c := parent.(type) {
      case *cancelCtx:
         return c, true
      case *timerCtx:
         return &c.cancelCtx, true
      case *valueCtx:
         parent = c.Context
      default:
         return nil, false
      }
   }
}
```

知道如何注册 cancelCtx，那么具体 cancel 的实现也很简单了，就是先取消自己，然后根据 children 递归遍历并取消所有可取消子节点。代码就不贴了，有兴趣自己看一遍完整源码比较合适。

最后再放一张图，更清楚的理解它们的关系：

![](/images/go/ctx_relation.png)



### reference

* [国外大佬关于 context 的演讲](https://www.youtube.com/watch?v=-_B5uQ4UGi0)
* [逼呼-为什么那么多框架都设计有Context](https://www.zhihu.com/question/269905592)

