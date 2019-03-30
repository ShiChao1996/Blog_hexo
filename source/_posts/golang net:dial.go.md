---
title: Go net/dial.go（I）
cover: https://image.littlechao.top/20180301160118000011.jpg
author: 
  nick: Lovae
tags: 
    - golang 
    - net
date: 2018-3-1
categories: go sdk
# 首页每篇文章的子标题
subtitle: Go package net/dial.go 阅读笔记

---
> 实际上dial.go这个文件中并没有实际发起连接的部分，基本上是在为真正发起连接做一系列的准备，比如：解析网络类型、从addr解析ip地址。。。实际发起连接的函数在`tcpsock_posix.go`、`udpsock_posix.go`。。。

首先看一下最主要的类型：

```go
type Dialer struct {
	Timeout time.Duration
  
	Deadline time.Time

  	LocalAddr Addr //真正dial时的本地地址，兼容各种类型(TCP、UDP...),如果为nil，则系统自动选择一个地址

	DualStack bool // 双协议栈，即是否同时支持ipv4和ipv6.当network值为tcp时，dial函数会向host主机的v4和v6地址都发起连接

	FallbackDelay time.Duration // 当DualStack为真，ipv6会延后于ipv4发起，此字段即为延迟时间，默认为300ms

	KeepAlive time.Duration 

	Resolver *Resolver

  	Cancel <-chan struct{} // 用于取消dial
}
```

Dial是最主要的函数，看一下源码注释：

```go
// Dial connects to the address on the named network.
//
// Known networks are "tcp", "tcp4" (IPv4-only), "tcp6" (IPv6-only),
// "udp", "udp4" (IPv4-only), "udp6" (IPv6-only), "ip", "ip4"
// (IPv4-only), "ip6" (IPv6-only), "unix", "unixgram" and
// "unixpacket".
//
// For TCP and UDP networks, the address has the form "host:port".
// The host must be a literal IP address, or a host name that can be
// resolved to IP addresses.
// The port must be a literal port number or a service name.
// If the host is a literal IPv6 address it must be enclosed in square
// brackets, as in "[2001:db8::1]:80" or "[fe80::1%zone]:80".
// The zone specifies the scope of the literal IPv6 address as defined
// in RFC 4007.
// The functions JoinHostPort and SplitHostPort manipulate a pair of
// host and port in this form.
// When using TCP, and the host resolves to multiple IP addresses,
// Dial will try each IP address in order until one succeeds.
//
// Examples:
//	Dial("tcp", "golang.org:http")
//	Dial("tcp", "192.0.2.1:http")
//	Dial("tcp", "198.51.100.1:80")
//	Dial("udp", "[2001:db8::1]:domain")
//	Dial("udp", "[fe80::1%lo0]:53")
//	Dial("tcp", ":80")
//
// For IP networks, the network must be "ip", "ip4" or "ip6" followed
// by a colon and a literal protocol number or a protocol name, and
// the address has the form "host". The host must be a literal IP
// address or a literal IPv6 address with zone.
// It depends on each operating system how the operating system
// behaves with a non-well known protocol number such as "0" or "255".
//
// Examples:
//	Dial("ip4:1", "192.0.2.1")
//	Dial("ip6:ipv6-icmp", "2001:db8::1")
//	Dial("ip6:58", "fe80::1%lo0")
//
// For TCP, UDP and IP networks, if the host is empty or a literal
// unspecified IP address, as in ":80", "0.0.0.0:80" or "[::]:80" for
// TCP and UDP, "", "0.0.0.0" or "::" for IP, the local system is
// assumed.
//
// For Unix networks, the address must be a file system path.
```

> 从注释可以看出，Dial 支持多种网络类型；支持ipv4、ipv6；还支持用host名代替ip地址。



```go
func Dial(network, address string) (Conn, error) {
	var d Dialer
	return d.Dial(network, address)
}

func DialTimeout(network, address string, timeout time.Duration) (Conn, error) {
	d := Dialer{Timeout: timeout}
	return d.Dial(network, address)
}

func (d *Dialer) Dial(network, address string) (Conn, error) {
	return d.DialContext(context.Background(), network, address)
}
```

以上前两个是导出的主要函数，都调用了d.dial()，d.DialContext()。d.DialContext()可以传入一个context，如果context的生命周期在connect完成之前结束，那么会立即返回错误。如果context在连接建立完成之后结束，则不会影响连接。另外如果addr是一组ip地址的话，会把当前剩下的所有时间均分到每个ip上去尝试连接。只要有一个成功，就会立即返回成功的连接并取消其他尝试。具体看代码(有删减)：

```go
func (d *Dialer) DialContext(ctx context.Context, network, address string) (Conn, error) {
	...
  	deadline := d.deadline(ctx, time.Now()) 
  	//d.deadline() 比较d.deadline、ctx.deadline、now+timeout，返回其中最小.如果都为空，返回0
  	...
  	subCtx, cancel := context.WithDeadline(ctx, deadline) //设置新的超时context
  	defer cancel()
  	...
	// Shadow the nettrace (if any) during resolve so Connect events don't fire for DNS lookups.
	resolveCtx := ctx
	...//给resolveCtx带上一些value

	addrs, err := d.resolver().resolveAddrList(resolveCtx, "dial", network, address, d.LocalAddr) // 解析IP地址，返回值是一个切片

	dp := &dialParam{
		Dialer:  *d,
		network: network,
		address: address,
	}

	var primaries, fallbacks addrList
	if d.DualStack && network == "tcp" { //表示同时支持ipv4和ipv6
		primaries, fallbacks = addrs.partition(isIPv4) // 将addrs分成两个切片，前者包含ipv4地址，后者包含ipv6地址
	} else {
		primaries = addrs
	}

	var c Conn
	if len(fallbacks) > 0 {//有ipv6的情况，v4和v6一起dial
		c, err = dialParallel(ctx, dp, primaries, fallbacks)
	} else {
		c, err = dialSerial(ctx, dp, primaries)
	}
	if err != nil {
		return nil, err
	}
	...
	return c, nil
}
```

从上面代码看到，DialContext最终调用的是`dialParallel`和`dialSerial`,先看dialParallel，该函数将v4地址和v6地址分开，先尝试v4地址组，在dialer.fallbackDelay 时间后开始尝试v6地址组，每一组都是调用dialSerial(),让两组竞争：

```go
func dialParallel(ctx context.Context, dp *dialParam, primaries, fallbacks addrList) (Conn, error) {
	if len(fallbacks) == 0 {
		return dialSerial(ctx, dp, primaries)
	}

	type dialResult struct {
		Conn
		error
		primary bool
		done    bool
	}
	results := make(chan dialResult) // unbuffered

	startRacer := func(ctx context.Context, primary bool) {
		ras := primaries // ras 意思是 remote addresses
		if !primary {
			ras = fallbacks
		}
		c, err := dialSerial(ctx, dp, ras)
      	...
		results <- dialResult{Conn: c, error: err, primary: primary, done: true}
	}

	var primary, fallback dialResult

	// Start the main racer.
	primaryCtx, primaryCancel := context.WithCancel(ctx)
	defer primaryCancel()
	go startRacer(primaryCtx, true)	//先尝试ipv4地址组

	// Start the timer for the fallback racer.
	fallbackTimer := time.NewTimer(dp.fallbackDelay())
	defer fallbackTimer.Stop()

	for {
		select {
		case <-fallbackTimer.C: // ipv6延迟时间到，开始尝试ipv6地址组
			fallbackCtx, fallbackCancel := context.WithCancel(ctx)
			defer fallbackCancel()
			go startRacer(fallbackCtx, false)

		case res := <-results: //表示至少有一组已经建立连接
			if res.error == nil { //
				return res.Conn, nil
			}
			if res.primary {
				primary = res
			} else {
				fallback = res
			}
			if primary.done && fallback.done {//同时建立连接，抛弃
				return nil, primary.error
			}
			if res.primary && fallbackTimer.Stop() {
				// If we were able to stop the timer, that means it
				// was running (hadn't yet started the fallback), but
				// we just got an error on the primary path, so start
				// the fallback immediately (in 0 nanoseconds).
				fallbackTimer.Reset(0)
			}
		}
	}
}
```

继续看`dialSerial`：

```go
func dialSerial(ctx context.Context, dp *dialParam, ras addrList) (Conn, error) {
	var firstErr error // The error from the first address is most relevant.

	for i, ra := range ras { // ra => remote address
		select {
		case <-ctx.Done(): //表示
			return nil, &OpError{Op: "dial", Net: dp.network, Source: dp.LocalAddr, Addr: ra, Err: mapErr(ctx.Err())}
		default:
		}

		deadline, _ := ctx.Deadline()
		partialDeadline, err := partialDeadline(time.Now(), deadline, len(ras)-i)
        // 这里表示前 i 个IP地址的连接失败，然后将剩下的时间均分到剩余的IP地址
		...//判断是否超时并处理
      
		dialCtx := ctx
        dialCtx, cancel := context.WithDeadline(ctx, partialDeadline)
		defer cancel()

		c, err := dialSingle(dialCtx, dp, ra)// 对单个IP地址发起连接
		if err == nil {
			return c, nil
		}
		if firstErr == nil {
			firstErr = err
		}
	}

	if firstErr == nil {
		firstErr = &OpError{Op: "dial", Net: dp.network, Source: nil, Addr: nil, Err: errMissingAddress}
	}
	return nil, firstErr
}
```

最终所有的对单个IP地址发起链接的任务是由dialSingle分配的（此处简单看下就好），该函数解决了兼容不同网络类型的问题：

```go
func dialSingle(ctx context.Context, dp *dialParam, ra Addr) (c Conn, err error) {
	trace, _ := ctx.Value(nettrace.TraceKey{}).(*nettrace.Trace)
	if trace != nil {
		raStr := ra.String()
		if trace.ConnectStart != nil {
			trace.ConnectStart(dp.network, raStr)
		}
		if trace.ConnectDone != nil {
			defer func() { trace.ConnectDone(dp.network, raStr, err) }()
		}
	}
	la := dp.LocalAddr
	switch ra := ra.(type) {
	case *TCPAddr:
		la, _ := la.(*TCPAddr)
		c, err = dialTCP(ctx, dp.network, la, ra)
	case *UDPAddr:
		la, _ := la.(*UDPAddr)
		c, err = dialUDP(ctx, dp.network, la, ra)
	case *IPAddr:
		la, _ := la.(*IPAddr)
		c, err = dialIP(ctx, dp.network, la, ra)
	case *UnixAddr:
		la, _ := la.(*UnixAddr)
		c, err = dialUnix(ctx, dp.network, la, ra)
	default:
		return nil, &OpError{Op: "dial", Net: dp.network, Source: la, Addr: ra, Err: &AddrError{Err: "unexpected address type", Addr: dp.address}}
	}
	if err != nil {
		return nil, &OpError{Op: "dial", Net: dp.network, Source: la, Addr: ra, Err: err} // c is non-nil interface containing nil pointer
	}
	return c, nil
}
```

到此，dial.go基本就这么多内容，真正通过socket建立连接的部分下篇再写吧(其实是偷懒)。

#### (待续)

