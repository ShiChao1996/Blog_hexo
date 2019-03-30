---
title: Go net/dial.go (II)
cover: https://image.littlechao.top/20180301160118000011.jpg
author: 
  nick: Lovae
date: 2018-3-2
categories: go sdk
# 首页每篇文章的子标题
subtitle: Go package net/dial.go 阅读笔记

---
[上一篇文章](https://www.littlechao.top/#/index/article?_id=5a9823ceebc087002a3e7259) 我们大致分析了dial.go中的代码，起主要的功能就是为真正发起连接做一些准备，起到了应用层的作用（DNS解析等）。但是一个连接完整的连接还需要更深层次的网络协议来完成协作，所以我们接着上篇来分析，由于篇(懒)幅原因，只将`dialTcp`作为传输层的例子。。。话不多说，上代码：

```go
func dialTCP(ctx context.Context, net string, laddr, raddr *TCPAddr) (*TCPConn, error) {
	if testHookDialTCP != nil { //testHookDialTCP 是语言开发者为了测试留的钩子函数，不用管
		return testHookDialTCP(ctx, net, laddr, raddr)
	}
	return doDialTCP(ctx, net, laddr, raddr)
}
```

> 注意现在所在文件是在`tcpsock_posix.go` 这部分是**传输层**的内容了。



来看`doDialTCP`:

```go
func doDialTCP(ctx context.Context, net string, laddr, raddr *TCPAddr) (*TCPConn, error) {
	fd, err := internetSocket(ctx, net, laddr, raddr, syscall.SOCK_STREAM, 0, "dial")

	for i := 0; i < 2 && (laddr == nil || laddr.Port == 0) && (selfConnect(fd, err) || spuriousENOTAVAIL(err)); i++ {
		if err == nil {
			fd.Close()
		}
		fd, err = internetSocket(ctx, net, laddr, raddr, syscall.SOCK_STREAM, 0, "dial")
	}

	if err != nil {
		return nil, err
	}
	return newTCPConn(fd), nil
}
```

参数里的ctx自然不言而喻了，是为了控制请求超时取消请求释放资源的；`laddr`是 local address ， `raddr`是指 remote address；返回值这里会得到 `TCPConn`。代码不长，就是调用了 `internetSocket`得到一个文件描述符，并用其新建一个conn返回。但这里我想多说几句，因为不难发现， `internetSocket`可能会被调用多次，为什么呢？

首先我们需要知道 Tcp 有一个极少使用的机制，叫`simultaneous connection`（同时连接）。正常的连接是：A主机 dial B主机，B主机 listen。 而同时连接则是： A 向 B dial 同时 B 向 A dial，那么 A 和 B 都不需要监听。

我们知道，当 传入 dial  函数的参数`laddr`==`raddr`时，内核会拒绝dial。但如果传入的`laddr`为nil，kernel 会自动选择一个本机端口，这时候有可能会使得新的`laddr`==`raddr`,这个时候，kernel不会拒绝dial，并且这个dial会成功，原因是就`simultaneous connection`，这可能是kernel的bug。所以会判断是否是 `selfConnect`或者`spuriousENOTAVAIL`(spurious error not avail)来判断上一次调用`internetSocket`返回的 err 类型，在特定的情况下重新尝试`internetSocket`.关于这个问题的讨论参见[这里](https://stackoverflow.com/questions/4949858/how-can-you-have-a-tcp-connection-back-to-the-same-port)。



好了，我们接下来看看`internetSocket`，该函数在`ipsock_posix.go`文件，到了**网络层**的范围了。

```go
func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
	if (runtime.GOOS == "windows" || runtime.GOOS == "openbsd" || runtime.GOOS == "nacl") && mode == "dial" && raddr.isWildcard() {
		raddr = raddr.toLocal(net) 
      // 如果 raddr 是零地址，把它转化成当前系统对应的零地址格式(local system address 127.0.0.1 or ::1)
	}
	family, ipv6only := favoriteAddrFamily(net, laddr, raddr, mode)
	return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}
```

（sotype 和 proto 是生成 socket 文件d的系统调用时用的）首先判断了运行系统的类型，`favoriteAddrFamily`返回了当前 dial 最合适的地址族，主要是判断应该用ipv4还是ipv6或者都用，其返回值 family 有两种可能值：`AF_INET`和`AF_INET6`，都是int类型，感兴趣的朋友可以参见[这里](https://stackoverflow.com/questions/1593946/what-is-af-inet-and-why-do-i-need-it)。



让我们接着关注`socket`,该函数在`sock_posix.go`文件，意味着接下来将是更加底层的系统调用了。

```go
// socket returns a network file descriptor that is ready for
// asynchronous I/O using the network poller.
func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr) (fd *netFD, err error) {
	s, err := sysSocket(family, sotype, proto)
	if err != nil {
		return nil, err
	}
	if err = setDefaultSockopts(s, family, sotype, ipv6only); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}
	if fd, err = newFD(s, family, sotype, net); err != nil {
		poll.CloseFunc(s)
		return nil, err
	}

	// This function makes a network file descriptor for the
	// following applications:
	//
	// - An endpoint holder that opens a passive stream
	//   connection, known as a stream listener
	//
	// - An endpoint holder that opens a destination-unspecific
	//   datagram connection, known as a datagram listener
	//
	// - An endpoint holder that opens an active stream or a
	//   destination-specific datagram connection, known as a
	//   dialer
	//
	// - An endpoint holder that opens the other connection, such
	//   as talking to the protocol stack inside the kernel
	//
	// For stream and datagram listeners, they will only require
	// named sockets, so we can assume that it's just a request
	// from stream or datagram listeners when laddr is not nil but
	// raddr is nil. Otherwise we assume it's just for dialers or
	// the other connection holders.

	if laddr != nil && raddr == nil {
		switch sotype {
		case syscall.SOCK_STREAM, syscall.SOCK_SEQPACKET:
			if err := fd.listenStream(laddr, listenerBacklog); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		case syscall.SOCK_DGRAM:
			if err := fd.listenDatagram(laddr); err != nil {
				fd.Close()
				return nil, err
			}
			return fd, nil
		}
	}
	if err := fd.dial(ctx, laddr, raddr); err != nil {
		fd.Close()
		return nil, err
	}
	return fd, nil
}
```

这段代码隐含了大量细节，首先看最上面函数的注释，返回值是一个使用了[`network poller`](https://segmentfault.com/a/1190000003063859)的**异步I/O**的文件描述符。前面三个 if 里，先创建了一个 socket，然后设置基本参数，再 new 一个文件描述符，其中包含了大量的系统调用和底层细节，这里先跳过。我想说的在下面。

socket 这个函数可以为一下几种应用创建一个文件描述符：

* 一个打开了 被动的、流式的 连接的终端，通常叫`stream listener`
* 一个打开了 没有具体目的地的、数据报格式的 连接的终端，通常叫` datagram listener`
* 一个打开了 主动的、有明确目的地的、数据报格式的 连接的终端，通常叫`dialer`
* 一个打开了其他连接的终端，比如与内核中的协议栈通信

> 通常可以认为当 `laddr`不为空但`raddr`为空时的 request 是来自stream or datagram listeners。否则就是来自 dialers 或者其他系统连接。

所以一个dialer和listener的区别就是 laddr， 也就是dialer在一定情况下可以当做listener，到这里就可以解释之前tcp的`simultaneous connection`同时连接了。



接下来调用了fd的dial函数，这里才真正通过socket开始发送连接请求。

(待续)

