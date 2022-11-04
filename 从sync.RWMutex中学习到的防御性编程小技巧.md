> 本文的代码可以在这里找到：[Github](https://github.com/zbh255/my_example/blob/main/GO/concurrentPrograming/rwLock_test.go)
>
> 防御性编程的一些介绍: https://zh.wikipedia.org/wiki/%E9%98%B2%E5%BE%A1%E6%80%A7%E7%BC%96%E7%A8%8B

> 最近在研究标准库锁的实现原理，在`sync.RWMutex`看到`RLocker`的实现，这个方法将读锁转型为`Locker`接口派生出去，底层共用`RWMutex`，它这里使用了一个私有的类型别名(`rlocker`)包装了原来的`RWMutex`，这样就可以防止外部方法的断言，但也是"防君子不防小人的方法"，本文记录了一些博主无聊的想法。

`src/sync/rwmutex.go`

```go
...
func (rw *RWMutex) RLocker() Locker {
	return (*rlocker)(rw)
}

type rlocker RWMutex

func (r *rlocker) Lock()   { (*RWMutex)(r).RLock() }
func (r *rlocker) Unlock() { (*RWMutex)(r).RUnlock() }
```

比如以下的类型断言就会`panic`

```go
func TestSafeConversion(t *testing.T) {
	lock := sync.RWMutex{}
	lock.Lock()
	readLock := lock.RLocker()
	readWriteLock := readLock.(*sync.RWMutex) # panic
	readWriteLock.Unlock()
}
```

`OutPut`

```go
panic: interface conversion: sync.Locker is *sync.rlocker, not *sync.RWMutex [recovered]
```

---

但真的无法转换吗？其实不然，以上是`safe`的实现方式，go中提供的`unsafe`就可以做到

我们知道Go runtime对有类型信息的`interface`的定义如下。

`src/runtime/runtime2.go`

```go
type iface struct {
	tab  *itab
	data unsafe.Pointer
}
```

在我的机器上(`darwin64`)，`iface struct`其实就是两个8字节的指针，`*itab`是接口元数据的指针，`data`存储的是指向实例的指针，关于`go interface`可以看看这篇:[文章](https://zhuanlan.zhihu.com/p/136949285)，本文就不在展开了，因为这不是重点。

所以我们只要得到`data`并将它转换为我们需要的类型，所以我们构造一个具有两个8字节指针的`iface`

```go
type Interface struct {
	_iface uintptr
	data   unsafe.Pointer
}
```

接下来我们就可以大展身手了，将`Locker`的指针转换为`sync.RWMutex`的指针

```go
func TestUnsafeConversion(t *testing.T) {
	lock := sync.RWMutex{}
	lock.Lock()
	readLock := lock.RLocker()
	readWriteLock := (*sync.RWMutex)((*Interface)(unsafe.Pointer(&readLock)).data)
	readWriteLock.Unlock()
}
```

跑一下`go test`，哈哈，`Lock&UnLock`成功。

```go
=== RUN   TestUnsafeConversion
--- PASS: TestUnsafeConversion (0.00s)
PASS
```