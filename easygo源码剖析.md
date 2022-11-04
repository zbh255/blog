> `mailru/easygo`是一个简单实现的事件库，正好最近在研究怎么实现一个高性能的HTTP Server，而事件库是不可或缺的一部分，本文记录了我对`easygo`源码的研究。
>
> 我第一次写这种类型的文章，所以细节之处可能不完善，还望海涵。
>
> ==注==：在开始之前我假设你已经对Epoll和Go语言底层有一定的了解。

## 流程分析

> 在源码分析的路上，我们往往需要一些示例代码来引导我们，但是`easygo`并没有示例代码，所以我自己根据注释和理解写了一些简单的示例来理解`easygo`的流程，`easygo`抽象了多个IO多路复用的机制，本文只会讲述`epoll`的抽象

`示例代码`

```go
import (
	"errors"
	"fmt"
	"github.com/mailru/easygo/netpoll"
	"io"
	"net"
	"os"
	"os/signal"
	"runtime"
	"syscall"
	"time"
)

// N_READY_CONNECTIONS n worker thread
var N_READY_CONNECTIONS = (runtime.NumCPU() - 1) * 8

// ready ok connection channel
var readyConn = make(chan net.Conn,N_READY_CONNECTIONS)


func main() {
	poller, err := netpoll.New(nil)
	if err != nil {
		panic(err)
	}
	listener, err := net.Listen("tcp", "0.0.0.0:8080")
	if err != nil {
		panic(err)
	}
	acceptDesc, err := netpoll.HandleListener(listener, netpoll.EPOLLIN)
	if err != nil {
		panic(err)
	}
	cancel := initHandleGoroutine()
	// add accept goroutine cancel
	acceptCancel := make(chan struct{})
	cancel = append(cancel,acceptCancel)
	// signal handle
	signalChan := make(chan os.Signal,1)
	signal.Notify(signalChan, syscall.SIGINT, syscall.SIGTERM, syscall.SIGHUP)
	go func() {
		<-signalChan
		for _,v := range cancel{
			v <- struct{}{}
		}
		// close readyConn chan write pipe
		close(readyConn)
		// close acceptConn chan write pipe
		close(acceptCancel)
		// check old fd is handle ok?
		for {
			time.Sleep(time.Millisecond * 10)
			if len(readyConn) == 0 && len(acceptCancel) == 0 {
				os.Exit(0)
			}
		}
	}()

	acceptDone := make(chan struct{},255)
	err = poller.Start(acceptDesc, func(event netpoll.Event) {
		acceptDone <- struct{}{}
	})
	if err != nil {
		panic(err)
	}

	go func() {
		for {
			select {
			case <-acceptDone:
				conn,err := listener.Accept()
				if err != nil {
					fmt.Println("accept error : ",err)
					continue
				}
				readDesc, err := netpoll.HandleRead(conn)
				if err != nil {
					fmt.Println("handle read event error : ",err)
				}
				// on read
				err = poller.Start(readDesc, func(event netpoll.Event) {
					readyConn <- conn
				})
				if err != nil {
					fmt.Println("poller read start error : ",err)
				}
				// on close
				closeDesc, err := netpoll.Handle(conn,netpoll.EPOLLRDHUP)
				err = poller.Start(closeDesc, func(event netpoll.Event) {
					err := poller.Stop(readDesc)
					if err != nil {
						fmt.Println("poller read stop err : ", err)
					}
					err = poller.Stop(closeDesc)
					if err != nil {
						fmt.Println("poller close stop err : ", err)
					}
					err = readDesc.Close()
					if err != nil {
						fmt.Println("read desc close conn err : ", err)
					}
					err = closeDesc.Close()
					if err != nil {
						fmt.Println("close desc close conn err : ", err)
					}
				})
				if err != nil {
					fmt.Println("poller close start error : ",err)
				}
			case <-acceptCancel:
				return
			}
		}
	}()

	// hang
	select {}
}

func initHandleGoroutine() []chan struct{} {
	cancel := make([]chan struct{},0,N_READY_CONNECTIONS/8)
	for i := 0; i < N_READY_CONNECTIONS / 8; i++ {
		nCancel := make(chan struct{})
		cancel = append(cancel,nCancel)
		go func() {
			runtime.LockOSThread()
			for {
				select {
				case conn := <- readyConn:
					buffer := make([]byte,256)
					_, err := conn.Read(buffer)
					if err == io.EOF || errors.Is(err,net.ErrClosed) {
						continue
					}
					if err != nil {
						fmt.Println("read err : ",err)
						continue
					}
					// reset buffer len
					buffer = buffer[:0]
					buffer = append(buffer, "HTTP/1.1 200 OK\r\nServer: easygo\r\nContent-Type: text/plain\r\nDate: "...)
					buffer = time.Now().AppendFormat(buffer, "Mon, 02 Jan 2006 15:04:05 GMT")
					buffer = append(buffer, "\r\nContent-Length: 12\r\n\r\nHello World!"...)
					// write
					_, err = conn.Write(buffer)
					if err == io.EOF || errors.Is(err,net.ErrClosed) {
						continue
					}
					if err != nil {
						fmt.Println("write err : ",err)
						continue
					}
				case <- nCancel:
					return
				}
			}
		}()
	}
	return cancel
}
```

示例代码实现了一个简单的Htpp Echo服务器，从代码中可以看出`easygo`的实现非常简易，它实现了一个单Reactor模式的事件循环，统一处理所有的就绪事件。

easygo事件的注册和处理也不太方便，使用起来还是挺麻烦的。

easygo只响应就绪事件，对`Non-Block`连接的`read&write`则运用了go runtime和go net包的能力来处理

easygo中定义了一些通用的事件类型来屏蔽底层IO多路复用机制的差异

`netpoll/netpoll.go`

```go
const (
	EventRead  Event = 0x1
	EventWrite       = 0x2
)

const (
	// EventHup is indicates that some side of i/o operations (receive, send or
	// both) is closed.
	// Usually (depending on operating system and its version) the EventReadHup
	// or EventWriteHup are also set int Event value.
	EventHup Event = 0x10

	EventReadHup  = 0x20
	EventWriteHup = 0x40

	EventErr = 0x80

	// EventPollerClosed is a special Event value the receipt of which means that the
	// Poller instance is closed.
	EventPollerClosed = 0x8000
)
```

## 代码分析

我们先来看看`easygo`定义的一些通用数据类型

`netpoll/handle.go`

```go
// Desc is a network connection within netpoll descriptor.
// It's methods are not goroutine safe.
type Desc struct {
	file  *os.File
	event Event
}
```

其实注释已经说的很明白了`Desc`是netpoll中为了兼容不同的IO多路复用机制而抽象的网络连接。

- `Desc.file`所记录的是原来文件的副本，虽然跟原来的fd指向一样的文件对象，但不要去读取这个fd做一些记录的操作
- `Desc.event`则记录了创建`Desc`时指定的事件类型，`easygo`中不同的IO多路复用机制会以不同的角度来对待它们

`netpoll/handle.go`

```go
// filer describes an object that has ability to return os.File.
type filer interface {
	// File returns a copy of object's file descriptor.
	File() (*os.File, error)
}
```

filer接口主要用于断言一些实例有无实现返回文件副本的接口，比如实例代码中的实际返回的`net.TCPConn`和`net.TCPListener`就实现了这个接口

`net/tcpsock.go`

```go
func (c *conn) File() (f *os.File, err error) {
	f, err = c.fd.dup()
	if err != nil {
		err = &OpError{Op: "file", Net: c.fd.net, Source: c.fd.laddr, Addr: c.fd.raddr, Err: err}
	}
	return
}
```

`net.TCPConn`和`net.TCPListener`的File()方法逻辑差不多，主要是使用`dup`系统调用为原fd派生一个副本并检查错误。

所以`easygo`创建`Desc`的方法逻辑很简单，断言实例有无实现filer接口，并将文件副本与`event`赋值给一个新的Desc并返回

`netpoll/handle.go`

```go
func handle(x interface{}, event Event) (*Desc, error) {
	f, ok := x.(filer)
	if !ok {
		return nil, ErrNotFiler
	}

	// Get a copy of fd.
	file, err := f.File()
	if err != nil {
		return nil, err
	}

	return &Desc{
		file:  file,
		event: event,
	}, nil
}
```

因为dup系统调用返回的副本并不继承原有fd的属性，`net.TCPConn`和`net.TCPListener`的File方法设置了`close-on-exec`(执行时关闭)的属性，但是并没有把副本fd设置为非阻塞的IO模式，所以作者在可导出的Handle()函数中重新设置了这个属性

`src/internal/poll/fd_unix.go`

```go
func dupCloseOnExecOld(fd int) (int, string, error) {
	syscall.ForkLock.RLock()
	defer syscall.ForkLock.RUnlock()
	newfd, err := syscall.Dup(fd)
	if err != nil {
		return -1, "dup", err
	}
	syscall.CloseOnExec(newfd)
	return newfd, "", nil
}
```

`netpoll/handle.go`

```go
func Handle(conn net.Conn, event Event) (*Desc, error) {
	desc, err := handle(conn, event)
	if err != nil {
		return nil, err
	}

	// Set the file back to non blocking mode since conn.File() sets underlying
	// os.File to blocking mode. This is useful to get conn.Set{Read}Deadline
	// methods still working on source Conn.
	//
	// See https://golang.org/pkg/net/#TCPConn.File
	// See /usr/local/go/src/net/net.go: conn.File()
	if err = setNonblock(desc.fd(), true); err != nil {
		return nil, os.NewSyscallError("setnonblock", err)
	}

	return desc, nil
}
```

`easygo`中还提供了许多方便创建Desc的函数。

`netpoll/handle.go`

```go
...
func HandleRead(conn net.Conn) (*Desc, error) {
	return Handle(conn, EventRead|EventEdgeTriggered)
}
...
func HandleReadOnce(conn net.Conn) (*Desc, error) {
	return Handle(conn, EventRead|EventOneShot)
}
...
```

`Poller`接口是描述`easygo`事件轮询器的实现，这个接口分别被抽象epoll和kqueue的类型实现，我接下来会分析epoll抽象的实现。

`netpoll/netpoll.go`

```go
type Poller interface {
	// Start adds desc to the observation list.
	//
	// Note that if desc was configured with OneShot event, then poller will
	// remove it from its observation list. If you will be interested in
	// receiving events after the callback, call Resume(desc).
	//
	// Note that Resume() call directly inside desc's callback could cause
	// deadlock.
	//
	// Note that multiple calls with same desc will produce unexpected
	// behavior.
	Start(*Desc, CallbackFn) error

	// Stop removes desc from the observation list.
	//
	// Note that it does not call desc.Close().
	Stop(*Desc) error

	// Resume enables observation of desc.
	//
	// It is useful when desc was configured with EventOneShot.
	// It should be called only after Start().
	//
	// Note that if there no need to observe desc anymore, you should call
	// Stop() to prevent memory leaks.
	Resume(*Desc) error
}
```

首先就是epoller的New函数，New函数负责创建一个底层是Epoll的实例，供上层调用，我们来分析这个New函数的行为

`netpoll/netpoll_epoll.go`

```go
func New(c *Config) (Poller, error) {
	cfg := c.withDefaults()

	epoll, err := EpollCreate(&EpollConfig{
		OnWaitError: cfg.OnWaitError,
	})
	if err != nil {
		return nil, err
	}

	return poller{epoll}, nil
}
```

```go
cfg := c.withDefaults()
```

上面的方法返回了底层轮询器在`epoll_wait`和`close`系统调用出错时的回调函数。

```go
epoll, err := EpollCreate(&EpollConfig{
		OnWaitError: cfg.OnWaitError,
	})
```

EpollCreate是创建epoll示例的方法，具体逻辑如以下代码，我们来分析以下他的流程

- `fd, err := unix.EpollCreate1(0)`,使用epoll_create1系统调用创建epoll实例
- `r0, _, errno := unix.Syscall(unix.SYS_EVENTFD2, 0, 0, 0)`,使用eventfd2系统调用创建一个fd，这个fd是线程安全的，在Epoll轮询器的实现中是用于Close的事件通知的，关于eventfd2系统调用的行为和原理可以阅读参考资料中这篇文章:[^1]

```go
func EpollCreate(c *EpollConfig) (*Epoll, error) {
	config := c.withDefaults()

	fd, err := unix.EpollCreate1(0)
	if err != nil {
		return nil, err
	}

	r0, _, errno := unix.Syscall(unix.SYS_EVENTFD2, 0, 0, 0)
	if errno != 0 {
		return nil, errno
	}
	eventFd := int(r0)

	// Set finalizer for write end of socket pair to avoid data races when
	// closing Epoll instance and EBADF errors on writing ctl bytes from callers.
	err = unix.EpollCtl(fd, unix.EPOLL_CTL_ADD, eventFd, &unix.EpollEvent{
		Events: unix.EPOLLIN,
		Fd:     int32(eventFd),
	})
	if err != nil {
		unix.Close(fd)
		unix.Close(eventFd)
		return nil, err
	}

	ep := &Epoll{
		fd:        fd,
		eventFd:   eventFd,
		callbacks: make(map[int]func(EpollEvent)),
		waitDone:  make(chan struct{}),
	}

	// Run wait loop.
	go ep.wait(config.OnWaitError)

	return ep, nil
}
```

EpollCtl对应`epoll_ctl`系统调用，为epoll的兴趣列表添加一个要监听的事件，而这里把`eventfd2`系统调用返回的`fd`添加作用是为了监听是否有关闭的信号，关于具体响应关闭的代码可以在后文的`ep.wait`和`ep.Close`分析中可以看到，在这里就暂时这么理解。

```go
err = unix.EpollCtl(fd, unix.EPOLL_CTL_ADD, eventFd, &unix.EpollEvent{
		Events: unix.EPOLLIN,
		Fd:     int32(eventFd),
	})
```

这段代码则做一些初始化的工作

- 存储epoll fd
- 存储`event2fd fd`提供给`ep.wait`中判断事件的来源
- 初始化存储`Poller.Start`注册的回调函数的Map
- 初始化接收在其他goroutine运行的`ep.wait`退出的信号

```go
ep := &Epoll{
		fd:        fd,
		eventFd:   eventFd,
		callbacks: make(map[int]func(EpollEvent)),
		waitDone:  make(chan struct{}),
	}
```

启动一个后台的`goroutine`自动等待事件。

返回创建完成的Epoll实例

```go
// Run wait loop.
go ep.wait(config.OnWaitError)
return ep, nil
```

接下来我们来看分析创建完一个Poller之后，`easygo`怎么把事件添加到Epoll实例的。

作者将`*Epoll`作为`poller`的匿名字段，这样有助于抽象底层不同的IO多路复用机制，虽然poller类型实现了Poller接口，但根据编译条件的不同，poller实现的细节却不一样，以下是实现Epoll和Kqueue轮询器的对比

```go
// poller implements Poller interface.
type poller struct {
	*Epoll
}
```

```go
type poller struct {
	*Kqueue
}
```

不过，在此我们只关注Epoll的抽象，我们来看看底层为epoll的poller的Start方法的实现

```go
func (ep poller) Start(desc *Desc, cb CallbackFn) error {
	err := ep.Add(desc.fd(), toEpollEvent(desc.event),
		func(ep EpollEvent) {
			var event Event

			if ep&EPOLLHUP != 0 {
				event |= EventHup
			}
			if ep&EPOLLRDHUP != 0 {
				event |= EventReadHup
			}
			if ep&EPOLLIN != 0 {
				event |= EventRead
			}
			if ep&EPOLLOUT != 0 {
				event |= EventWrite
			}
			if ep&EPOLLERR != 0 {
				event |= EventErr
			}
			if ep&_EPOLLCLOSED != 0 {
				event |= EventPollerClosed
			}

			cb(event)
		},
	)
	if err == nil {
		if err = setNonblock(desc.fd(), true); err != nil {
			return os.NewSyscallError("setnonblock", err)
		}
	}
	return err
}
```

在进入`ep.Add`之前主要做的是

- 获取要添加到Epoll兴趣列表的Fd
- `toEpollEvent`对`easygo`中通用的Flags转换成Epoll对应的Flags
- `func(ep EpollEvent)`则在正式调用事件对应的回调函数之前将Epoll对应的Flags转换成通用的Flags

`netpoll/netpoll_epoll.go`

```go
func toEpollEvent(event Event) (ep EpollEvent) {
	if event&EventRead != 0 {
		ep |= EPOLLIN | EPOLLRDHUP
	}
	if event&EventWrite != 0 {
		ep |= EPOLLOUT
	}
	if event&EventOneShot != 0 {
		ep |= EPOLLONESHOT
	}
	if event&EventEdgeTriggered != 0 {
		ep |= EPOLLET
	}
	return ep
}
```

关于事件对应的回调函数的定义

`netpoll/netpoll.go`

```go
// CallbackFn is a function that will be called on kernel i/o event
// notification.
type CallbackFn func(Event)
```

我们来看看，`ep.Add`内部做了什么。

- 使用配置好的Fd和Flags初始化一个EpollEvent
- 写锁保证同时只有一个线程往epoll中添加事件和添加回调函数
	- 这里加锁的原因，前文提到的后台的goroutine等待到事件之后会加读锁来读取对应Fd的回调函数
	- 所以如果写Map不加写锁的话就会产生数据竞争
- 检查对应Fd的回调函数是否已经被注册，已经注册则返回错误，没有注册则注册
- 使用`epoll_ctl`系统调用往Epoll中添加初始化好的Epoll事件

`netpoll/epoll.go`

```go
func (ep *Epoll) Add(fd int, events EpollEvent, cb func(EpollEvent)) (err error) {
	ev := &unix.EpollEvent{
		Events: uint32(events),
		Fd:     int32(fd),
	}

	ep.mu.Lock()
	defer ep.mu.Unlock()

	if ep.closed {
		return ErrClosed
	}
	if _, has := ep.callbacks[fd]; has {
		return ErrRegistered
	}
	ep.callbacks[fd] = cb

	return unix.EpollCtl(ep.fd, unix.EPOLL_CTL_ADD, fd, ev)
}
```

关于其他的`Mod`和`Del`方法的逻辑和Add方法差不多，相信读者知道了Add方法的逻辑也能知道Mod和Del中做了什么，它们也是使用了epoll_ctl系统调用。

接下来，我们就可以来分析在后台运行的`ep.wait`做了什么了。

`netpoll/epoll.go`

```go
func (ep *Epoll) wait(onError func(error)) {
	defer func() {
		if err := unix.Close(ep.fd); err != nil {
			onError(err)
		}
		close(ep.waitDone)
	}()

	events := make([]unix.EpollEvent, maxWaitEventsBegin)
	callbacks := make([]func(EpollEvent), 0, maxWaitEventsBegin)

	for {
		n, err := unix.EpollWait(ep.fd, events, -1)
		if err != nil {
			if temporaryErr(err) {
				continue
			}
			onError(err)
			return
		}

		callbacks = callbacks[:n]

		ep.mu.RLock()
		for i := 0; i < n; i++ {
			fd := int(events[i].Fd)
			if fd == ep.eventFd { // signal to close
				ep.mu.RUnlock()
				return
			}
			callbacks[i] = ep.callbacks[fd]
		}
		ep.mu.RUnlock()

		for i := 0; i < n; i++ {
			if cb := callbacks[i]; cb != nil {
				cb(EpollEvent(events[i].Events))
				callbacks[i] = nil
			}
		}

		if n == len(events) && n*2 <= maxWaitEventsStop {
			events = make([]unix.EpollEvent, n*2)
			callbacks = make([]func(EpollEvent), 0, n*2)
		}
	}
}
```

这里的代码主要做一些清理的工作。

- 关闭epoll fd，有错误是调用传入的处理错误的回调函数
- 关闭`ep.waitDone`管道，相当于通知`wait`方法已经完成了它工作，这个`channel`是由`Close`方法读取的

```go
defer func() {
		if err := unix.Close(ep.fd); err != nil {
			onError(err)
		}
		close(ep.waitDone)
	}()
```

这里的代码主要做一些初始化数据的工作

- 初始化提供给epoll_wait告知哪些事件就绪的切片，这里作者定义了初始最多处理1024个就绪事件
- 初始化保存回调函数的切片，容量跟存储就绪事件的切片一样，也是1024

```go
events := make([]unix.EpollEvent, maxWaitEventsBegin)
callbacks := make([]func(EpollEvent), 0, maxWaitEventsBegin)
```

这里则是事件循环。

- `epoll_wait`系统调用负责等待就绪的事件，错误处理逻辑负责处理暂时的系统调用错误，如果错误不是暂时的则调用注册的错误处理回调函数，并return退出程序触发defer

- 根据返回的就绪事件的数量，读取就绪事件对应的Fd

	- 检查是不是用于响应Close事件的通过`eventfd2`系统调用创建的Fd就绪，就绪的话则解锁读锁并return返回

	- 加读锁来从map中读取就绪fd对应的回调函数

	- 执行所有就绪事件的回调函数

	- 根据就绪事件的大小调整存储就绪事件和回调函数的Slice大小

	- `netpoll`子包中定义`events`和`callbacks`初始容量和最大容量

	- ```go
		const (
			maxWaitEventsBegin = 1024
			maxWaitEventsStop  = 32768
		)
		```

---

```go
for {
		n, err := unix.EpollWait(ep.fd, events, -1)
		if err != nil {
			if temporaryErr(err) {
				continue
			}
			onError(err)
			return
		}

		callbacks = callbacks[:n]

		ep.mu.RLock()
		for i := 0; i < n; i++ {
			fd := int(events[i].Fd)
			if fd == ep.eventFd { // signal to close
				ep.mu.RUnlock()
				return
			}
			callbacks[i] = ep.callbacks[fd]
		}
		ep.mu.RUnlock()

		for i := 0; i < n; i++ {
			if cb := callbacks[i]; cb != nil {
				cb(EpollEvent(events[i].Events))
				callbacks[i] = nil
			}
		}

		if n == len(events) && n*2 <= maxWaitEventsStop {
			events = make([]unix.EpollEvent, n*2)
			callbacks = make([]func(EpollEvent), 0, n*2)
		}
	}
```

接下来，我们来分析Close的逻辑，以下是可导出的Close方法的定义，我们来看看它的内部做了什么。

`netpoll/epoll.go`

```go
func (ep *Epoll) Close() (err error) {}
```

以下的代码片段主要是将退出信号通知到在后台运行的`ep.wait`，`Create`方法执行时创建的`eventfd`，写入对应的`uint64`值就可以触发epoll的已读事件。

加锁是因为避免多个线程同时调用Close产生`data-race`，和`closed`字段配合保证同时只有一个线程进入这段代码，且`wirte eventfd`只会被执行一次

```go
ep.mu.Lock()
{
    if ep.closed {
        ep.mu.Unlock()
        return ErrClosed
    }
    ep.closed = true

    if _, err = unix.Write(ep.eventFd, closeBytes); err != nil {
        ep.mu.Unlock()
        return
    }
}
ep.mu.Unlock()
```

closeBytes的定义，按照小端字节序来读取，这个值就是1

```go
var closeBytes = []byte{1, 0, 0, 0, 0, 0, 0, 0}
```

这里的代码主要做一些清理的工作

- `<-ep.waitDone`等待`wait`执行完成退出
- `unix.Close`关闭之前创建的用于通知`wait goroutine`的`eventfd`
- 执行所有Fd注册的回调函数，给回调函数传入的Flags改为表示Epoll已经关闭，然后置空回调函数Map
	- 由于置空也是一个写操作，所以需要上写锁保证置空操作的安全

```go
<-ep.waitDone

if err = unix.Close(ep.eventFd); err != nil {
    return
}

ep.mu.Lock()
// Set callbacks to nil preventing long mu.Lock() hold.
// This could increase the speed of retreiving ErrClosed in other calls to
// current epoll instance.
// Setting callbacks to nil is safe here because no one should read after
// closed flag is true.
callbacks := ep.callbacks
ep.callbacks = nil
ep.mu.Unlock()

for _, cb := range callbacks {
    if cb != nil {
        cb(_EPOLLCLOSED)
    }
}

return
```

到此，`easygo`中关于`epoll`相关的代码主要的已经分析完毕，我们来总结一下。

- `easygo`实现是一个单Reactor监听就绪事件的模型
- `easygo`使用统一的Poller接口和具体的poller实现来抽象底层不同IO多路复用机制的差异
- `epoll`轮询器的实现中，使用线程安全的`eventfd2`来往后台goroutine通知Close事件
- `easygo`跟后台负责等待事件的goroutine交互的操作会有锁开销，比如往回调函数map中写数据，检查`ep.closed`属性

## 参考资料

1. [eventfd(2) 结合 select(2) 源码分析](https://www.cnblogs.com/shuqin/p/11700682.html)
2. [eventfd系统调用简介](https://evian-zhang.github.io/introduction-to-linux-x86_64-syscall/src/filesystem/eventfd-eventfd2.html)
3. [[转][译]百万级WebSockets和Go语言](https://colobu.com/2017/12/13/A-Million-WebSockets-and-Go/)
4. 《Linux高性能服务器编程》6、9章节
5. 《Linux/UNIX》系统编程手册 63、43章节

[^1]:[eventfd(2) 结合 select(2) 源码分析](https://www.cnblogs.com/shuqin/p/11700682.html)

