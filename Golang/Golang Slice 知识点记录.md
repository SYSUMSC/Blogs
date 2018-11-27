# Golang slice 知识点记录

slice 是 Go 中一种基本的数据结构，我们可以通过这种结构来管理数据集合，在开发中不定长度表示的数组全部都是 slice 。

## 结构定义

我们可以在 Golang 的[运行库](https://golang.org/src/runtime/slice.go)实现中查看 slice 结构定义。

```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

按照上述的定义，我们可以通过以下的代码解析出 `array` 、`len` 和 `cap`。

代码地址：https://play.golang.org/p/zIJv2h4RPBa

```go
package main

import (
	"fmt"
	"unsafe"
)

type Slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}

func main() {
	s := make([]int, 0, 5)
	p := (*Slice)(unsafe.Pointer(&s))
	fmt.Printf("%+v\n", *p)
}
```

运行结果：

```
{array:0x452000 len:0 cap:5}
```

根据结果，我们可以解析到 `array` 地址为 `0x452000` ，`len` 值为 `0` ，`cap` 值为 `5` 。

我们可以通过 `p` 修改 `s` 的 `len` 和 `cap` 。

代码地址：https://play.golang.org/p/xSfSKgSejxI

```go
p.len = 2
p.cap = 4
fmt.Printf("len: %d, cap: %d\n", len(s), cap(s)) // len: 2, cap: 4
```

## array 指针的指向

在一个切片或数组 `base` 的基础上，创建新的切片 `s = base[begin:end]` ，则新切片的 `array` 指针指向的是 `base[begin]` 数据的内存地址。注意， `begin` 和 `end` 的默认值分别为 `0` 和 `len(base)` 。

代码地址：https://play.golang.org/p/YXDOVebbxmZ

```go
package main

import (
	"fmt"
	"unsafe"
)

var sizeOfInt = int(unsafe.Sizeof(int(0)))

func main() {
	base := [5]int{0, 1, 2, 3, 4}

	fmt.Printf("size of int: %d \n", sizeOfInt)
	fmt.Printf("array addr: %p \n", &base)

	s1 := base[1:4]
	fmt.Println("array:", s1)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(s1), cap(s1), *(*unsafe.Pointer)(unsafe.Pointer(&s1)))

	s2 := s1[1:]
	fmt.Println("array:", s2)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(s2), cap(s2), *(*unsafe.Pointer)(unsafe.Pointer(&s2)))
}
```

输出结果如下：

```
size of int: 4
array addr: 0x452000
array: [1 2 3]
len:3 cap: 4 array ptr: 0x452004
array: [2 3]
len:2 cap: 3 array ptr: 0x452008
```

`s1` 的 `array` 指针指向的是 `base[1]` 的地址。

`s2` 的 `array` 指针指向的是 `s1[1]` 的地址，即 `base[2]` 的地址。

## append

我们可以通过 `append` 向 slice 追加元素。

代码地址：https://play.golang.org/p/INCV7N-iWhs

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	arr := make([]int, 1)
	fmt.Println("array:", arr)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(arr), cap(arr), *(*unsafe.Pointer)(unsafe.Pointer(&arr)))

	for i := 0; i < 4; i++ {
		arr = append(arr, i)
		fmt.Println("array:", arr)
		fmt.Printf("len:%d cap: %d array ptr: %v \n", len(arr), cap(arr), *(*unsafe.Pointer)(unsafe.Pointer(&arr)))
	}
}
```

输出结果如下：

```
array: [0]
len:1 cap: 1 array ptr: 0x416020
array: [0 0]
len:2 cap: 2 array ptr: 0x416050
array: [0 0 1]
len:3 cap: 4 array ptr: 0x416070
array: [0 0 1 2]
len:4 cap: 4 array ptr: 0x416070
array: [0 0 1 2 3]
len:5 cap: 8 array ptr: 0x45a020
```

从结果中，我们发现，当 `len` 与 `cap` 相等时，slice 会发生自动扩容，且扩容后 `array` 指针发生了变化。 值得注意的是，当 slice 未发生自动扩容时，`array` 指针保持原有的值。

在对 slice 进行 `append` 操作时，容量的增长规则是：

- 首先判断，如果新申请容量大于 2 倍的旧容量，则最终容量就是新申请的容量
- 否则判断，如果旧切片的长度小于 1024，则最终容量就是旧容量的两倍
- 否则判断，如果旧切片长度大于等于 1024，则最终容量从旧容量开始循环增加原来的 1/4，即直到最终容量大于等于新申请的容量
- 如果最终容量计算值溢出，则最终容量就是新申请容量

在实际使用中，我们最好事先设置好 slice 的 `cap` ，这里可以避免在使用 `append` 时出现反复重新分配、复制之前数据的问题，减少不必要的性能消耗。

## len 和 cap 的计算

在一个切片或数组 `base` 的基础上，创建新的切片 `s = base[begin:end]` ，则新切片的 `len` 的值为 `end - begin`，`cap` 值为 `cap(base) - begin` 。

代码地址：https://play.golang.org/p/O6Sh6RNyfdx

```go
package main

import (
	"fmt"
	"unsafe"
)

func main() {
	arr := make([]int, 5, 8)

	for i := 0; i < 5; i++ {
		arr[i] = i
	}

	fmt.Println("array:", arr)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(arr), cap(arr), *(*unsafe.Pointer)(unsafe.Pointer(&arr)))

	// s := base[begin:end]
	// default value:
	//	begin: 0
	//	end:   len(base)
	// len(s): end - begin
	// cap(s): cap(base) - begin

	// len(s1): 4 - 2 = 2
	// cap(s1): 8 - 2 = 6
	s1 := arr[2:4]
	fmt.Println("array:", s1)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(s1), cap(s1), *(*unsafe.Pointer)(unsafe.Pointer(&s1)))

	// len(s1): 2 - 1 = 1
	// cap(s1): 6 - 1 = 5
	s1 = s1[1:]
	fmt.Println("array:", s1)
	fmt.Printf("len:%d cap: %d array ptr: %v \n", len(s1), cap(s1), *(*unsafe.Pointer)(unsafe.Pointer(&s1)))
}
```

输出结果如下：

```
array: [0 1 2 3 4]
len:5 cap: 8 array ptr: 0x452000
array: [2 3]
len:2 cap: 6 array ptr: 0x452008
array: [3]
len:1 cap: 5 array ptr: 0x45200c
```

## 参考资料

- [深入理解 go 的 slice 和到底什么时候该用 slice](https://segmentfault.com/a/1190000005812839)
- [深入解析 Go 中 Slice 底层实现](https://halfrost.com/go_slice/)
- [[译]如何避免 golang 的坑](https://gocn.vip/article/171)
