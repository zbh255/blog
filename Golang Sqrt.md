# Golang Sqrt

`参考文章`

> - https://zhuanlan.zhihu.com/p/74728007
> - https://blog.csdn.net/qq_42637568/article/details/86761108

---

> 本文主要是关于`Golang`中平方根计算的实现方法，由博主收集而来，原理本文不在赘述，主要是实现，文末还有相关的性能测试比较这几种方法的性能

`标准库(std lib)`

```go
package test

import (
	"math"
	"testing"
)

func TestSqrt(t *testing.T) {
	t.Log(math.Sqrt(500))
}
```

`牛顿迭代法`

```go
func Crsqrt(number float64) float64 {
	i := number / 2
	a := 0.0000000000001
	count := 0
	for math.Abs(math.Pow(i, 2) -number) > a {
		i = i - (math.Pow(i,2) - number)/(2*i)
		count++
	}
	return i
}
```

`卡马克魔数`

```go
func Qrsqrt(number float32) float32 {
	y := number
	x2 := number * 0.5
	var halfs float32 = 1.5
	i := *(*uint32)(unsafe.Pointer(&y))
	i = 0x5f3759df - (i >> 1)
	y = *(*float32)(unsafe.Pointer(&i))
	y = y * (halfs - (x2 * y * y))
	return y * number
}

```

----

`OutPut`

```shell
    sqrt_test.go:90: 22.360679774997898 # 牛顿迭代法
    sqrt_test.go:91: 22.350563 # 卡马克算法
    sqrt_test.go:93: 22.360679774997898 # 标准库
```

##### 性能测试

```go
func BenchmarkSqrt(b *testing.B) {
	b.Run("Std Lib", func(b *testing.B) {
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			math.Sqrt(500)
		}
	})
	b.Run("CarMock", func(b *testing.B) {
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			Qrsqrt(500)
		}
	})
	b.Run("NewtonIter", func(b *testing.B) {
		b.ResetTimer()
		for i := 0; i < b.N; i++ {
			Crsqrt(500)
		}
	})
}
```

> 在我的机器`Cpu`i7 8705G上的结果

```go
BenchmarkSqrt
BenchmarkSqrt/Std_Lib
BenchmarkSqrt/Std_Lib-8         	1000000000	         0.305 ns/op
BenchmarkSqrt/CarMock
BenchmarkSqrt/CarMock-8         	1000000000	         0.566 ns/op
BenchmarkSqrt/NewtonIter
BenchmarkSqrt/NewtonIter-8      	 1839075	       650 ns/op
```

##### 一些总结

> 卡马克算法本身是用来算平方根的倒数的，乘与原数得出来的结果会有误差，比如输入`4`得出:`1.9966143`
>
> 牛顿迭代法则可以控制精度，但算法本身是`O(logn)`的复杂度，速度远比不上标准库