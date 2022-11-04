## 引言

我之前一直想写一篇关于`sync/atomic`源码分析的文章，但奈何之前的这一方面的知识太浅，现在学到了多一点知识，希望能通过此篇文章串联起来。本人才疏学浅，难免有错误之处，欢迎各位赐教。

本篇文章分两个部分，第一部分分析`atomic`基本过程的实现，覆盖Load/Store/CompareAndSwap/Swap，第二部分则探究`atomic.Value`的实现和问题。

本文基于GO 1.17分析，之后的版本一些底层数据类型的表示可能会变更，比如`interface{}`对应的`eface`

## 修改记录(Modify)

- 2022 05/29 更新内容，修复把`FP`描述为`SP`错误的描述

## 一些概念

### 数据长度

|     Byte     |         Word         |   Machine Word   | Double Word | Quard Word  | Cache Line              |
| :----------: | :------------------: | :--------------: | :---------: | :---------: | ----------------------- |
|     8Bit     |     2Byte-16Bit      |   4Byte/8Byte    | 4Byte-32Bit | 8Byte-64Bit | ~64Byte                 |
| 寻址基本单位 | 16位机器下寄存器的字 | 机器字，也称字长 |    双字     |    四字     | 缓存行的大小，一般为64B |
|      B       |          w           |  w/word lenght   |  dw/l/long  |    q/wq     |                         |

### 调用规约

要读懂下面的go汇编代码需要一点儿关于调用规约和go汇编的知识，这些知识将在这里简单讲解。

在Go 1.17之前，函数参数都是通过栈内存传递的，1.17有通过寄存器传参的实验版本，不过这不在本文的讨论范围

#### What is ? (什么是?)

什么是调用规约，`wikipedia`中有一处解释：[Link](https://stackoverflow.com/questions/513832/how-do-i-compare-strings-in-java)，我这里引用部分内容

> - 极微参数或复杂参数独立部分的分配顺序
> - 参数是如何被传递的（放置在堆栈上，或是寄存器中，亦或两者混合）
> - 被调用者应保存调用者的哪个[寄存器](https://zh.wikipedia.org/wiki/寄存器)
> - 调用[函数](https://zh.wikipedia.org/wiki/子程序)时如何为任务准备[堆栈](https://zh.wikipedia.org/wiki/堆栈)，以及任务完成如何恢复

我个人的理解是函数调用一般需要两个角色，即`caller`(调用者)，`callee`(被调用者)，调用者即调用函数的角色，被调用者则是那个函数内部的过程，调用规约即它们之间的一些约定，比如参数怎么传入，返回值由谁接收？

#### GO调用规约

栈区的内存都是从高到低增长的，`FP`虚拟寄存器是栈帧指针(Frame Pointer)，所以在汇编代码中，对此寄存器Add是缩栈，Sub是扩栈。

```ASN.1
       FP        FP      ▲High
        │         │      │
        │         │      │
        │         │      │
        │         │      │
        │         │      │
        │         │      │
    Sub 8     Add 8      │
        │         │      │
        │         │      │
        │         │      │
        │         │      │
        │         │      │
        │         ▼      │
────────┼────────────────┴────────
        │                ▼Low
        │
        ▼
```

在go调用规约中，函数参数是从右到左依次入栈，返回值的内存由caller(调用者)负责准备，比如以下函数中的栈帧布局。

```go
...
func CompareAndSwapUint64(addr *uint64, old, new uint64) (swapped bool)
```

```json
┌──────────┐
│ swapped  ├─────►Caller
├──────────┤
│   new    ├─────►Callee
├──────────┤
│   old    ├─────►Callee
├──────────┤
│   addr   ├─────►Callee
└──────────┘
```

有了这些知识，相信你就可以看得懂接下来的内容了。

## API

`sync/atomic`包看起来函数很多，其实只是对不同类型的包装，但是底层的实现很少，根据不同的数据长度划分了不同的函数，我们这里只解析`x86_64`下的64位原子操作

`sync/atomic/doc.go`中的函数声明，以下的代码中给出了部分声明。

```go
...
func SwapInt64(addr *int64, new int64) (old int64)
...
func SwapUint32(addr *uint32, new uint32) (old uint32)
...
func CompareAndSwapInt32(addr *int32, old, new int32) (swapped bool)
...
func CompareAndSwapInt64(addr *int64, old, new int64) (swapped bool)
...
func AddInt32(addr *int32, delta int32) (new int32)
...
func AddInt64(addr *int64, delta int64) (new int64)
...
func LoadInt32(addr *int32) (val int32)
...
func LoadInt64(addr *int64) (val int64)
...
func StoreInt32(addr *int32, val int32)
...
func StoreInt64(addr *int64, val int64)
...
```

`sync/atomic/asm.s`中是以上go代码中声明的函数的汇编实现，事实上`sync/atomic`中除了`atomic.Value`之外的源码都在`runtime/internal/atomic`中

我们这里摘取`sync/atomic/asm.s`中部分的声明

```assembly
TEXT ·SwapUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xchg64(SB)
...
TEXT ·CompareAndSwapInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Cas64(SB)
...
TEXT ·AddUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Xadd64(SB)
...
TEXT ·LoadInt64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Load64(SB)
...
TEXT ·StoreUint64(SB),NOSPLIT,$0
	JMP	runtime∕internal∕atomic·Store64(SB)
...
```

本篇文章分析`x86_64`下的实现，`x86_64`下的go汇编实现在`runtime/internal/atomic/atomic_amd64.s`下

### Cas64

我们先来分析`qw`数据长度下的Cas函数`Cas64`

```assembly
TEXT ·Cas64(SB), NOSPLIT, $0-25
	// 将指针值(addr)移动到BX虚拟寄存器中
	MOVQ	ptr+0(FP), BX
	// 将要比较的老值移动到AX虚拟寄存器中
	// 根据不同的数据长度，会选择不同的物理寄存器
	// 四字的情况下就是%rax
	MOVQ	old+8(FP), AX
	// 将要比较的新值移动到CX寄存器中
	MOVQ	new+16(FP), CX
	// x86的Lock指令前缀
	LOCK
	// 相当于cmpxchg new,ptr
	// cpu会将位于寄存器%rax中的旧值跟ptr中的值进行比较，如果相等则设置为源操作数提供的值
	// Intel x86手册中规定了cmpxchg的源操作数不能为内存地址
	CMPXCHGQ	CX, 0(BX)
	// cmp系列的指令比较完成之后会设置标志位(Flags)
	// SETEQ对应x86_64下的SETE指令，该指令会检查EFlags，并将检查的结果写入到目标操作数
	SETEQ	ret+24(FP)
	// 返回到调用者
	RET
```

关于`Lock`前缀的作用Intel的x86手册中这样描述：

> Causes the processor’s LOCK# signal to be asserted during execution of the accompanying instruction (turns the instruction into an atomic instruction). In a multiprocessor environment, the LOCK# signal ensures that the processor has exclusive use of any shared memory while the signal is asserted.

我英语有点渣，谷歌翻译了一下大致意思

> 使处理器的 LOCK# 信号在伴随指令的执行期间被断言（将指令转换为原子指令）。在多处理器环境中，LOCK# 信号确保处理器在信号被断言时独占使用任何共享内存。

也就是说`cpu`保证`Lock`后一条指令执行的原子性，指令执行的期间，只有执行指令的一个CPU能够访问指定位置的共享内存，其它`Cpu`对该共享内存的访问将会被阻塞。

关于锁缓存还是锁总线？`x86_64`中只有当处理器支持缓存锁定的时候，且数据没有跨`Cache Line`时才可以进行缓存锁定，否则锁定内存总线。

`Lock`前缀也不是都能加在所有指令之上的，只能添加在部分有read/write行为的指令之上，Intel x86手册中列出了能够添加的指令。

``` assembly
ADD, ADC, AND, BTC, BTR, BTS, CMPXCHG, CMPXCH8B, CMPXCHG16B, DEC, INC, NEG, NOT, OR, SBB, SUB, XOR, XADD, and XCHG
```

### Xadd64

`Xadd64`同样也是用于`qw`数据长度的，以下是其中之一的go函数声明和源码

```go
...
func AddUint64(addr *uint64, delta uint64) (new uint64)
...
```

```assembly
TEXT ·Xadd64(SB), NOSPLIT, $0-24
	// 将指针(addr)移动到BX虚拟寄存器
	MOVQ	ptr+0(FP), BX
	// 将要增量的值(delta)移动到AX虚拟寄存器
	MOVQ	delta+8(FP), AX
	MOVQ	AX, CX
	// x86的Lock指令前缀
	LOCK
	// 将(addr)指针对应的值增量delta
	// xadd指令会将目标操作数和源操作数的值互换，然后再对目标操作数增量
	// 所以该指令结束之后AX虚拟寄存器中存储的是未自增前的旧值
	XADDQ	AX, 0(BX)
	// 将旧值和新值相加,结果存储在目标操作数，也就是AX虚拟寄存器中
	ADDQ	CX, AX
	// 将相加的结果(新值)移动到调用者的栈中
	MOVQ	AX, ret+16(FP)
	RET
```

### Xchg64

不知各位发现没有，在`Xchg64`中，并没有使用`Lock`前缀，这是因为`xchg`指令带有隐式的`Lock`，所以无需显示声明。

同样附上go函数声明和go asm的源码。

```go
func SwapUint64(addr *uint64, new uint64) (old uint64)
```

```assembly
TEXT ·Xchg64(SB), NOSPLIT, $0-24
	// 将指针(addr)移动到BX虚拟寄存器
	MOVQ	ptr+0(FP), BX
	// 将要交换的值(new)移动到AX虚拟寄存器
	MOVQ	new+8(FP), AX
	// xchg会交换源操作数和目标操作数的值
	// 交换完毕后AX中的是旧值，addr指针对应的内存中的是新值
	XCHGQ	AX, 0(BX)
	// 将旧值移动到调用者用于存储返回值的栈区中
	MOVQ	AX, ret+16(FP)
	RET
```

一点补充——Intel x86手册中对`xchg`行为的描述：

> Exchanges the contents of the destination (first) and source (second) operands. The operands can be two general- purpose registers or a register and a memory location. If a memory operand is referenced, the processor’s locking protocol is automatically implemented for the duration of the exchange operation, regardless of the presence or absence of the LOCK prefix or of the value of the IOPL. (See the LOCK prefix description in this chapter for more information on the locking protocol.)

我英语渣，附上谷歌翻译的结果

> 交换目标（第一个）和源（第二个）操作数的内容。操作数可以是两个通用寄存器或一个寄存器和一个内存位置。如果引用了内存操作数，则处理器的锁定协议将在交换操作期间自动实现，无论是否存在 LOCK 前缀或 IOPL 的值。 （有关锁定协议的更多信息，请参阅本章中的 LOCK 前缀描述。）

### Store64

`Store`不同长度的`asm`函数比其它的操作要多一点，`Store`中也是用了`xchg`，跟上文`Xchg64`的源码基本一样，没啥可分析的，唯一的不同是直接将新值存入，不用将旧值返回给调用者。

```go
...
func StoreUint64(addr *uint64, val uint64)
...
```

```assembly
TEXT ·Store64(SB), NOSPLIT, $0-16
	MOVQ	ptr+0(FP), BX
	MOVQ	val+8(FP), AX
	XCHGQ	AX, 0(BX)
	RET
```

### Load64

x86下`Loadxx`的实现跟其它体系结构有很大部分，`Load`系列的函数过程的是由go代码实现的，附上声明和代码

```go
// LoadUint64 atomically loads *addr.
func LoadUint64(addr *uint64) (val uint64)

// LoadPointer atomically loads *addr.
func LoadPointer(addr *unsafe.Pointer) (val unsafe.Pointer)
```

```go
//go:nosplit
//go:noinline
func Loadp(ptr unsafe.Pointer) unsafe.Pointer {
	return *(*unsafe.Pointer)(ptr)
}

//go:nosplit
//go:noinline
func Load64(ptr *uint64) uint64 {
	return *ptr
}
```

> - `go:nosplit`指示禁止编译器在此函数内插入栈检查的指令
> - `go:noinline`指示编译器禁止将此函数内联

那么问题来了，为什么写操作都是用`Lock`指令前缀来保证原子性和互斥性，为什么读操作就不使用带`Lock`的访存模式？

初步猜想编译器可能做了一些优化和转换，后来查看编译的汇编代码中只是一条内联的`Mov`指令，很显然这个猜想不正确。

```assembly
0x0033 00051 (source_test.go:15)        MOVQ    "".cmpV(SB), DX
```

Stackoverflow上有个讨论探讨了这个问题：[Link](https://stackoverflow.com/questions/55787091/does-golang-atomic-load-have-a-acquire-semantics)

> 大致意思是go标准的访存满足了x86的强内存序的要求，实现了满足一致性的原子操作，`Mov`指令访问对齐的内存不会访问到中间状态，可以观察到顺序一致性原子的写入，`Load`函数也可以避免编译器对其进行`非法`的重排序和优化。听不懂的读者只要记住`Load`也可以用来同步变量的访问就可以了。
>
> 内存模型的知识并不是本篇的内容，所以我不打算讲更多，如果想要了解内存模型，可以看看以下几篇文章
>
> - [[译]GO内存模型](https://www.cnblogs.com/XiaoXiaoShuai-/p/15778776.html)
> - [[译]更新GO内存模型](https://colobu.com/2021/07/13/Updating-the-Go-Memory-Model/)
> - [[译]编程语言内存模型](https://colobu.com/2021/07/11/Programming-Language-Memory-Models/)

## Value

### 定义的一些类型

#### ifaceWords

```go
type ifaceWords struct {
	typ  unsafe.Pointer
	data unsafe.Pointer
}
```

该结构本质是对空接口(interface{})底层数据结构的再描述，用于类型转换，空接口(interface)的底层表示在`runtime/runtime2.go`中

```go
...
type eface struct {
	_type *_type
	data  unsafe.Pointer
}
```

#### Value

```go
...
// A Value must not be copied after first use.
type Value struct {
	v interface{}
}
```

`sync`包中的公开数据结构首次使用后都不可以复制，注释也有写，没什么可说的。

内部的空接口被用于保存每次Strore/Swap/CompareAndSwap操作传入的`val`。

### Load

```go
...
func (v *Value) Load() (val interface{}) {
	vp := (*ifaceWords)(unsafe.Pointer(v))
	typ := LoadPointer(&vp.typ)
    // 没有存入过数据和有goroutine正在第一次存入数据直接返回nil
	if typ == nil || uintptr(typ) == ^uintptr(0) {
		// First store not yet completed.
		return nil
	}
    // 否则原子加载指针并返回
	data := LoadPointer(&vp.data)
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	vlp.typ = typ
	vlp.data = data
	return
}
```

### Store

```go
...
func (v *Value) Store(val interface{}) {
    // 存入的空接口值不能为nil
	if val == nil {
		panic("sync/atomic: store of nil value into Value")
	}
    // 加载原值和要存入值的eface指针
	vp := (*ifaceWords)(unsafe.Pointer(v))
	vlp := (*ifaceWords)(unsafe.Pointer(&val))
	for {
        // 加载eface描述类型元数据的指针
        // Value用eface类型的元数据的指针描述当前有无goroutine正在存入数据
        // type == 0表示第一次存入，type == MaxUint则表示有goroutine已经禁止抢占并正在存入数据
        // type != Maxuint则表示没有goroutine正在存入数据
		typ := LoadPointer(&vp.typ)
        // 第一次存入的时候空接口必定为nil
        // 所以这里是应对第一次存入的情况
		if typ == nil {
			// 禁止抢占当前M
			runtime_procPin()
            // 比较旧值是否为nil，为nil则交换将MaxUint与&vp.typ中的指针值交换
			if !CompareAndSwapPointer(&vp.typ, nil, unsafe.Pointer(^uintptr(0))) {
                // 当有两个以上goroutine都是第一次存入数据就会有一个CAS操作失败
                // 此时解除抢占并尝试自旋
				runtime_procUnpin()
				continue
			}
			// Value中用MaxUint描述是否有goroutine在存入数据
            // 因为MaxUint并不是一个合法的指针，所以调用者传入的正常数据中不可能会有数值等于此的指针值
            // 存入完毕之后则其它goroutine可以观察到没有goroutine正在存入数据
			StorePointer(&vp.data, vlp.data)
			StorePointer(&vp.typ, vlp.typ)
            // 解除禁用抢占
			runtime_procUnpin()
			return
		}
        // 有goroutine正在第一次存入数据的情况下则自旋重试
		if uintptr(typ) == ^uintptr(0) {
			continue
		}
		// 检查要存入数据的类型是否跟Value中的相等
		if typ != vlp.typ {
			panic("sync/atomic: store of inconsistently typed value into Value")
		}
        // 不是第一次存入数据的情况下则直接存入
		StorePointer(&vp.data, vlp.data)
		return
	}
}
```

### Other

`Value`还有`Swap`和`CompareAndSwap`这两个方法，它们的逻辑跟上文的`Store`差不多，基本是检查状态、禁止抢占、自旋、使用对应的原子指令，相信你看懂了`Store`的逻辑也能试着去分析它们，我这里就不在费篇幅赘述了。

### 关于一些问题的论述

#### CAS ABA

什么是`ABA`问题？wikipedia上有个条目解释的很清楚，我这里引用一小节：[Link](https://en.wikipedia.org/wiki/ABA_problem)

> In [multithreaded](https://en.wikipedia.org/wiki/Thread_(computer_science)) [computing](https://en.wikipedia.org/wiki/Computer_science), the **ABA problem** occurs during synchronization, when a location is read twice, has the same value for both reads, and "value is the same" is used to indicate "nothing has changed". However, another thread can execute between the two reads and change the value, do other work, then change the value back, thus fooling the first thread into thinking "nothing has changed" even though the second thread did work that violates that assumption.

我英语渣，附上谷歌翻译的结果

> 在[多线程](https://en.wikipedia.org/wiki/Thread_(computer_science)) [计算](https://en.wikipedia.org/wiki/Computer_science)中，**ABA问题**发生在同步过程中，当一个位置被读取两次时，两次读取的值相同，用“值相同”表示“没有任何变化”。但是，另一个线程可以在两次读取之间执行并更改值，执行其他工作，然后将值更改回来，从而欺骗第一个线程认为“没有任何变化”，即使第二个线程确实违反了该假设。

可见，ABA问题的前提都是其中一个线程在执行`lock cmpxchg`之类的指令之前，被调度器中断执行，换入其它线程执行其它线程的代码。

那么`atomic.Value`中会不会有ABA问题呢？其实也会有，因为`Store`等方法中也会有自旋等操作，这些操作都是可以被调度器抢占的，读者可能会看到`Store`中第一次写入调用了`runtime_procPin`禁止抢占，但这个函数只是禁止抢占当前`M`而已，对其他的`M`不起作用，如果goroutine在不同的`M`上执行，相当于两个独立的线程使用原子操作访问共享内存，而这些线程在执行`lock cmpxchg`以外的指令，或者两条指令之间也可能被操作系统的线程调度器中断执行。

#### 具体问题和解决方法

网上很多资料都指出了一些问题，比如在使用`CAS`实现无锁队列的时候，重新执行的线程可能观察不到队列头之外数据的变化，导致这个线程认为队列并没有变化，此时可能有一些隐藏问题。

像`JAVA`之类的语言提供了基于乐观并发控制的思想解决ABA问题的atomic API，不过具体的实现并不是本篇的内容。

## 总结

在这篇文章中，我分析了`sync/atomic`包中，一些底层原子操作的实现，之后我还分析了`atomic.Value`的实现，`atomic.Value`在2014第一次提交的时候只有Load&Store两个方法，Swap/CompareAndSwap都是1.17版本添加的，具体可以看看这个：[issue](https://github.com/golang/go/issues/39351)，事实上`atomic.Value`并不是开始就存在于atomic包中的，这里引用来自这篇博客的部分内容：[Go 语言标准库中 atomic.Value 的前世今生](https://blog.betacat.io/post/golang-atomic-value-exploration/)

> 我在`golang-dev`邮件列表中翻到了14年的[这段讨论](https://groups.google.com/forum/#!msg/golang-dev/SBmIen68ys0/WGfYQQSO4nAJ)，有用户报告了`encoding/gob`包在多核机器上（80-core）上的性能问题，认为`encoding/gob`之所以不能完全利用到多核的特性是因为它里面使用了大量的互斥锁（mutex），如果把这些互斥锁换成用`atomic.LoadPointer/StorePointer`来做并发控制，那性能将能提升20倍。
>
> 针对这个问题，有人提议在已有的`atomic`包的基础上封装出一个`atomic.Value`类型，这样用户就可以在不依赖 Go 内部类型`unsafe.Pointer`的情况下使用到`atomic`提供的原子操作。

我还浅析了一些CAS带来的问题，至此，这已经是全部的内容。

## 参考资料

- [英特尔® 64 位和 IA-32 架构开发人员手册：卷 2A/B/C](https://www.intel.cn/content/www/cn/zh/search.html?ws=text#q=%E8%8B%B1%E7%89%B9%E5%B0%94%C2%AE%2064%20%E4%BD%8D%E5%92%8C%20IA-32%20%E6%9E%B6%E6%9E%84%E5%BC%80%E5%8F%91%E4%BA%BA%E5%91%98%E6%89%8B%E5%86%8C%EF%BC%9A%E5%8D%B7%202&sort=relevancy)

- [GO Asm示例](https://colobu.com/goasm/)
- 《GO语言高级编程》第三章-汇编语言
- [Go assembly language complementary reference](https://quasilyte.dev/blog/post/go-asm-complementary-reference/)
- [虚拟寄存器对应的x86体系结构真实寄存器的映射](https://github.com/golang/go/blob/master/src/cmd/asm/internal/arch/arch.go#L131)
- 《现代操作系统》原书第三版 —— 第二章 进程与线程
- 《现代体系结构上的UNIX系统》—— 第二部分-第8章-多处理器系统概述
- [x86 Assembly/Data Transfer](https://en.wikibooks.org/wiki/X86_Assembly/Data_Transfer#Data_swap_based_on_comparison)
- [X86汇编语言/X86架构及寄存器解释](https://zh.wikibooks.org/wiki/X86%E7%B5%84%E5%90%88%E8%AA%9E%E8%A8%80/X86%E6%9E%B6%E6%9E%84%E5%8F%8A%E5%AF%84%E5%AD%98%E5%99%A8%E8%A7%A3%E9%87%8A#%E9%80%9A%E7%94%A8%E5%AF%84%E5%AD%98%E5%99%A8%EF%BC%88GPR%EF%BC%89_-_32%E4%BD%8D%E5%91%BD%E5%90%8D%E7%BA%A6%E5%AE%9A)