---
layout: post
title: "Linux I/O Models and GO Network Model Part II: Go Netpoller"
description: ""
category: articles
tags: [Linux, GO]
---
# Linux 的 I/O 模型以及 Go 的网络模型实现
## 第二部分：Go netpoller 实现原理分析
在 Go 中，所有的 I/O 都是阻塞的。Go 建议用户实现阻塞的 I/O 接口，并通过使用 goroutines 和 channel 实现并发。比如在 net/http 中的 HTTP server，每接受一个 connection，都会启动一个goroutines：


```
net/http/server.go:2802

func (srv *Server) Serve(l net.Listener) error {
    ...
    for {
        rw, e := l.Accept()
        ...
        tempDelay = 0
        c := srv.newConn(rw)
        c.setState(c.rwc, StateNew) // before Serve can return
        go c.serve(ctx) // 启动一个goroutines处理连接
    }

```

这种直观的方式在一定程度上简化了 I/O 模型的复杂度。不过这种阻塞只发生在 goroutines 层面，如果在底层OS也采用阻塞方式，效率就未免太低了。试想如果Go全部采用阻塞模型，将上层 blocking I/O 转化为 OS blocking I/O，也就是说一个 thread 对应一个 blocking I/O，那么所有连接的 thread 都会陷入 syscall 来等待 I/O 成功，这对整个系统的消耗可想而知。为了解决这个问题，Go采用 netpoller 将 OS asynchronous I/O 转换为 goroutines blocking I/O。

### Go netpoller
Go netpoller 的实现原理说来也简单，Go 程序在启动的时候会创建一个 M（这里涉及 Go 的调度模型：[The Go scheduler](http://morsmachine.dk/go-scheduler)）去跑监控函数：

```
runtime/proc.go:110

func main() {
    ...

    if GOARCH != "wasm" { // no threads on wasm yet, so no sysmon
        systemstack(func() {
            newm(sysmon, nil) // 监控函数
        })
    }

    // Lock the main goroutine onto this, the main OS thread,
    // during initialization. Most programs won't care, but a few
    // do require certain calls to be made by the main thread.
    // Those can arrange for main.main to run in the main thread
    // by calling runtime.LockOSThread during initialization
    // to preserve the lock.
    lockOSThread()
    ...
}
```

在 sysmon 中会轮询调用 netpoll 监听底层 fd，如果 fd 准备好，poller 就会唤醒被阻塞的 G（不同的平台有不同的接口实现，我们在这只讨论 Linux epoll ）：

```
runtime/proc.go:4315

func sysmon() {
    ...
    for {
       if netpollinited() && lastpoll != 0 && lastpoll+10*1000*1000 < now {
            atomic.Cas64(&sched.lastpoll, uint64(lastpoll), uint64(now))
            gp := netpoll(false) // non-blocking - returns list of goroutines
            if gp != nil {
                // Need to decrement number of idle locked M's
                // (pretending that one more is running) before injectglist.
                // Otherwise it can lead to the following situation:
                // injectglist grabs all P's but before it starts M's to run the P's,
                // another M returns from syscall, finishes running its G,
                // observes that there is no work to do and no other running M's
                // and reports deadlock.
                incidlelocked(-1)
                injectglist(gp)
                incidlelocked(1)
            }
        }
        ...
    }
}
```

在 Linux 平台下 netpoll 会调用`epollwait`等待就绪的 fd：

```
runtime/netpoll_epoll.go:61

func netpoll(block bool) *g {
    ...
retry:
    n := epollwait(epfd, &events[0], int32(len(events)), waitms)
    ...
}
```

### Go epoll创建和注册
上一节简单介绍了 Go netpoller 的运行机制，那么 Go epoll 是如何创建和监听 fd 的呢？看过上一部分的读者应该会记得，调用 Linux epoll 分为三步：

1. 调用`epoll_create`在内核中创建 context
2. 调用`epoll_ctl`增加或删除 fd
3. 调用`epoll_wait`等待事件发生

在 Go epoll 中，实际调用步骤是一样的。以一个简单的 tcp 连接为例，客户端代码如下：

```
func main() {
  // 创建socket连接
  conn, _ := net.Dial("tcp", "127.0.0.1:8081")
  for { 
    reader := bufio.NewReader(os.Stdin)
    fmt.Print("Text to send: ")
    text, _ := reader.ReadString('\n')
    fmt.Fprintf(conn, text + "\n")
    message, _ := bufio.NewReader(conn).ReadString('\n')
    fmt.Print("Message from server: "+message)
  }
}
```
`net.Dial`最终会调用 net/dial.go 的`DialContext`：

```
net/dial.go:341

func (d *Dialer) DialContext(ctx context.Context, network, address string) (Conn, error) {
    ...
    var c Conn
    if len(fallbacks) > 0 {
        c, err = dialParallel(ctx, dp, primaries, fallbacks)
    } else {
        c, err = dialSerial(ctx, dp, primaries)
    }
    ...
}

net/dial.go:489

func dialSerial(ctx context.Context, dp *dialParam, ras addrList) (Conn, error) {
    ...
    c, err := dialSingle(dialCtx, dp, ra)
    ...
}

net/dial.go:532

func dialSingle(ctx context.Context, dp *dialParam, ra Addr) (c Conn, err error) {
    ...
    case *TCPAddr:
        la, _ := la.(*TCPAddr)
        c, err = dialTCP(ctx, dp.network, la, ra)
    ...
}

net/tcpsock_posix.go:61

func doDialTCP(ctx context.Context, net string, laddr, raddr *TCPAddr) (*TCPConn, error) {
    fd, err := internetSocket(ctx, net, laddr, raddr, syscall.SOCK_STREAM, 0, "dial")
    ...
}

net/ipsock_posix.go:136

func internetSocket(ctx context.Context, net string, laddr, raddr sockaddr, sotype, proto int, mode string) (fd *netFD, err error) {
    ...
    return socket(ctx, net, family, sotype, proto, ipv6only, laddr, raddr)
}

net/sock_posix.go:18

func socket(ctx context.Context, net string, family, sotype, proto int, ipv6only bool, laddr, raddr sockaddr) (fd *netFD, err error) {
    ...
    if fd, err = newFD(s, family, sotype, net); err != nil {
        poll.CloseFunc(s)
        return nil, err
    }
    
    if err := fd.dial(ctx, laddr, raddr); err != nil {
        fd.Close()
        return nil, err
    }
    ...
}
```

经过上述一系列调用，代码会执行到`fd.dial`，最终调用`pollDesc.init`实现 epoll 的初始化及注册：


```
internal/poll/fd_poll_runtime.go:35

func (pd *pollDesc) init(fd *FD) error {
    serverInit.Do(runtime_pollServerInit)
    ctx, errno := runtime_pollOpen(uintptr(fd.Sysfd))
    if errno != 0 {
        if ctx != 0 {
            runtime_pollUnblock(ctx)
            runtime_pollClose(ctx)
        }
        return syscall.Errno(errno)
    }
    pd.runtimeCtx = ctx
    return nil
}
```

注意`runtime_pollServerInit`和`runtime_pollOpen`这两个函数，它们是使用`go:linkname`链接过来的，实际调用的是：

```
runtime/netpoll.go:85

//go:linkname poll_runtime_pollServerInit internal/poll.runtime_pollServerInit
func poll_runtime_pollServerInit() {
    netpollinit()
    atomic.Store(&netpollInited, 1)
}

//go:linkname poll_runtime_pollOpen internal/poll.runtime_pollOpen
func poll_runtime_pollOpen(fd uintptr) (*pollDesc, int) {
    pd := pollcache.alloc()
    lock(&pd.lock)
    if pd.wg != 0 && pd.wg != pdReady {
        throw("runtime: blocked write on free polldesc")
    }
    if pd.rg != 0 && pd.rg != pdReady {
        throw("runtime: blocked read on free polldesc")
    }
    pd.fd = fd
    pd.closing = false
    pd.seq++
    pd.rg = 0
    pd.rd = 0
    pd.wg = 0
    pd.wd = 0
    unlock(&pd.lock)

    var errno int32
    errno = netpollopen(fd, pd)
    return pd, int(errno)
}
```

先看`poll_runtime_pollServerInit`。在 Linux 平台下，它通过`netpollinit`调用`epollcreate`或`epollcreate1`实现`epoll`初始化：

```
runtime/netpoll_epoll.go:25

func netpollinit() {
    epfd = epollcreate1(_EPOLL_CLOEXEC)
    if epfd >= 0 {
        return
    }
    epfd = epollcreate(1024)
    if epfd >= 0 {
        closeonexec(epfd)
        return
    }
    println("runtime: epollcreate failed with", -epfd)
    throw("runtime: netpollinit failed")
}
```

而`poll_runtime_pollOpen`则通过`netpollopen`调用`epollctl`注册监听 fd：

```
runtime/netpoll_epoll.go:43

func netpollopen(fd uintptr, pd *pollDesc) int32 {
    var ev epollevent
    ev.events = _EPOLLIN | _EPOLLOUT | _EPOLLRDHUP | _EPOLLET
    *(**pollDesc)(unsafe.Pointer(&ev.data)) = pd
    return -epollctl(epfd, _EPOLL_CTL_ADD, int32(fd), &ev)
}
```

而整个流程的最后是调用`epollwait`监听fd的就绪状态，这个我们在开头时就讨论过。实际上这整个流程是和 Linux epoll 的调用步骤完全对应的。
