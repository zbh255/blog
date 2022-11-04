## 本文的目的

> 最近在写一个日志库，而日志库避免不了要打印调用者所在的源码文件、行号之类的信息，本文是记录博主优化Caller调用的性能时的一些想法和实践，还有的是其他的一些开源日志库是怎么优化这一块的。

## 函数定义和步骤性能分析

`runtime.Caller`的函数定义如下

```go
func Caller(skip int) (pc uintptr, file string, line int, ok bool)
```

至于各种返回值和参数的含义不在赘述，这里引用来自：[pkg.go.dev](https://pkg.go.dev/runtime#Caller)的释义

> Caller reports file and line number information about function invocations on the calling goroutine's stack. The argument skip is the number of stack frames to ascend, with 0 identifying the caller of Caller. (For historical reasons the meaning of skip differs between Caller and Callers.) The return values report the program counter, file name, and line number within the file of the corresponding call. The boolean ok is false if it was not possible to recover the information.

`runtime.Caller`的源码主要调用了`callers`和`CallersFrames(rpc).Next()`

```go
rpc := make([]uintptr, 1)
n := callers(skip+1, rpc[:])
if n < 1 {
  return
}
frame, _ := CallersFrames(rpc).Next()
return frame.PC, frame.File, frame.Line, frame.PC != 0
```

>  通过`pprof`的性能分析可以看到，`runtime.callers`占用了`runtime.Caller`函数`59%`左右的时间，callers函数的逻辑是找出当前的G的函数调用栈指针，Next则负责遍历查找该指针对应的文件Path、函数名、行号

![](https://blog-xiao-hui-1257821917.file.myqcloud.com/%E6%96%87%E7%AB%A0%E7%B4%A0%E6%9D%90/20220323/%E6%88%AA%E5%B1%8F2022-03-23%20%E4%B8%8A%E5%8D%8812.03.00.png)

## 性能优化

### 1.控制调用

>  很多库都实现了让使用者来决定是否在日志中输出调用信息

比如标准库`log`，标准库通过一些Flag来控制是否输出调用信息，标准库提供的`Flag`，用于控制输出

```go
const (
	Ldate         = 1 << iota     // the date in the local time zone: 2009/01/23
	Ltime                         // the time in the local time zone: 01:23:23
	Lmicroseconds                 // microsecond resolution: 01:23:23.123123.  assumes Ltime.
	Llongfile                     // full file name and line number: /a/b/c/d.go:23
	Lshortfile                    // final file name element and line number: d.go:23. overrides Llongfile
	LUTC                          // if Ldate or Ltime is set, use UTC rather than the local time zone
	Lmsgprefix                    // move the "prefix" from the beginning of the line to before the message
	LstdFlags     = Ldate | Ltime // initial values for the standard logger
)
```

zerolog则粒度更细，需要用户手动触发Caller()方法才会输出文件名和行号

`github.com/rs/zerolog#add-file-and-line-number-to-log`

```go
log.Logger = log.With().Caller().Logger()
log.Info().Msg("hello world")

// Output: {"level": "info", "message": "hello world", "caller": "/go/src/your_project/some_file:21"}
```

### 2.缓存

`callers`函数很难优化其性能，以我的能力是做不到了。但是可以优化`Next`和`CallersFrames`函数调用次数——通过将获取的调用栈指针作为`key`保存下来，`runtime.Frame`作为`value`保存下来，构建一个简单的cache来减少这里个函数的调用次数和内存分配。

在我编写的日志库中，我使用了以上两种方法来减少cpu时间的开销。

`github.com/zbh255/bilog/blob/main/caller.go`

```go
var (
	cache           = make(map[uintptr]runtime.Frame, 16)
	concurrentCache sync.Map
)

func CallerOfConcurrentCache(skip int) (file string, line int) {
	rpc := make([]uintptr, 1)
	n := runtime.Callers(skip+1, rpc[:])
	if n < 1 {
		return
	}
	frameI, ok := concurrentCache.Load(rpc[0])
	if ok {
		file = frameI.(runtime.Frame).File
		line = frameI.(runtime.Frame).Line
	} else {
		frame, _ := runtime.CallersFrames(rpc).Next()
		concurrentCache.Store(rpc[0], frame)
		file = frame.File
		line = frame.Line
	}
	return
}
```

使用`sync.Map`的原因是它比较适合读多写少的场景，存储的Frame写一次读多次，比如`access_log`的场景之类的高频日志很友好，接下来我们跑个测试对比以下有无缓存的cpu时间开销。

`github.com/zbh255/bilog/blob/main/caller_test.go`

```go
func BenchmarkCaller(b *testing.B) {
	b.Run("Default", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			_, _ = Caller(3)
		}
	})
	b.Run("CallerCached", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			_, _ = CallerOfCache(3)
		}
	})
	b.Run("CallerConcurrentCache", func(b *testing.B) {
		b.ReportAllocs()
		for i := 0; i < b.N; i++ {
			_,_ = CallerOfConcurrentCache(3)
		}
	})
}
```

`OutPut`

```shell
goos: darwin
goarch: amd64
pkg: github.com/zbh255/bilog
cpu: Intel(R) Core(TM) i7-8705G CPU @ 3.10GHz
BenchmarkCaller
BenchmarkCaller/Default
BenchmarkCaller/Default-8         	 1636160	       821.2 ns/op	     216 B/op	       2 allocs/op
BenchmarkCaller/CallerCached
BenchmarkCaller/CallerCached-8    	 2840467	       393.9 ns/op	       8 B/op	       1 allocs/op
BenchmarkCaller/CallerConcurrentCache
BenchmarkCaller/CallerConcurrentCache-8         	 3037758	       381.0 ns/op	       8 B/op	       1 allocs/op
PASS
```

可以看到，耗时和分配都减少了很多。

---

## 参考文章和资料

[golang日志组件使用runtime.Caller性能问题分析]:https://cloud.tencent.com/developer/article/1385947
[go语言最全优化技巧总结，值得收藏！]:https://z.itpub.net/article/detail/524C10CE8F59C6F8B7A252435345C5DA
[godoc]:https://pkg.go.dev/runtime#Caller

- [golang日志组件使用runtime.Caller性能问题分析 ](https://cloud.tencent.com/developer/article/1385947)
- [go语言最全优化技巧总结，值得收藏！](https://z.itpub.net/article/detail/524C10CE8F59C6F8B7A252435345C5DA)
- [godoc](https://pkg.go.dev/runtime#Caller)

