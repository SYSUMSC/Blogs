# Golang 字符串拼接的正确方式

## 背景

在很多场景中，我们会进行字符串拼接操作。

我们可能使用如下的操作：

```go
package main

import "fmt"

func main() {
	src := []string{
		"A",
		"B",
		"C",
	}

	var str string

	for _, s := range src {
		str += s
	}

	fmt.Println(str)
}
```

与许多支持 `string` 类型的语言一样，golang 中的 `string` 类型也是只读且不可变的。因此，上述拼接字符串的方式，会导致大量的 `string` 创建、销毁和内存分配。如果在业务中，拼接的字符串比较多，这显然不是一个正确的方式。

## 使用 `bytes.Buffer`

在 Go 1.10 之前，我们还可以通过 `bytes.Buffer` 来拼接字符串。

```go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	src := []string{
		"A",
		"B",
		"C",
	}

	var b bytes.Buffer
	for _, s := range src {
		fmt.Fprint(&b, s)
	}

	fmt.Println(b.String())
}
```

这里使用了 `var b bytes.Buffer` 存放最终拼接好的字符串，一定程度上避免第一种方法中每进行一次拼接操作就重新申请新的内存空间存放中间字符串的问题。

但注意到，在使用 `b.String()` 会有一次 `[]byte` 到 `string` 类型转换，而该过程需要进行一次内存分配和内容拷贝。

## 使用 `strings.Builder`

在 Go 1.10 以后，我们可以使用性能更强的 `strings.Builder` 完成字符串的拼接操作。

```Go
package main

import (
	"bytes"
	"fmt"
)

func main() {
	src := []string{
		"A",
		"B",
		"C",
	}

	var b strings.Builder
	for _, s := range src {
		fmt.Fprint(&b, s)
	}

	fmt.Println(b.String())
}
```

## Benchmark

这里，我们比较 `bytes.Buffer` 与 `strings.Builder` 的性能差异，下面是 `Benchmark` 代码。

```go
package test

import (
	"bytes"
	"fmt"
	"strings"
	"testing"
)

func BenchmarkBuffer(b *testing.B) {
	var buf bytes.Buffer
	for i := 0; i < b.N; i++ {
		fmt.Fprint(&buf, "😊")
		_ = buf.String()
	}
}

func BenchmarkBuilder(b *testing.B) {
	var builder strings.Builder
	for i := 0; i < b.N; i++ {
		fmt.Fprint(&builder, "😊")
		_ = builder.String()
	}
}
```

执行 `go test` 后，我们得到以下的测试结果。

```bash
Jiahonzheng$ go test -bench=. -benchmem
goos: darwin
goarch: amd64
BenchmarkBuffer-8    	  500000	     89135 ns/op	 1004339 B/op	       2 allocs/op
BenchmarkBuilder-8   	20000000	        70.1 ns/op	      21 B/op	       0 allocs/op
PASS
ok  	_/Users/Jiahonzheng/Desktop/test	46.101s
```

我们发现，二者间的性能差距还是挺大的，下面我们看一下标准库是如何实现 `strings.Builder` 方法。

## `strings.Builder` 源码解析

我们可以在 `strings/builder.go` 找到 `strings.Builder` 的实现，下面是摘录后的关键代码。

```go
// A Builder is used to efficiently build a string using Write methods.
// It minimizes memory copying. The zero value is ready to use.
// Do not copy a non-zero Builder.
type Builder struct {
	addr *Builder // of receiver, to detect copies by value
	buf  []byte // 1
}

// Write appends the contents of p to b's buffer.
// Write always returns len(p), nil.
func (b *Builder) Write(p []byte) (int, error) {
	b.copyCheck() 
	b.buf = append(b.buf, p...) // 2
	return len(p), nil
}

// String returns the accumulated string.
func (b *Builder) String() string {
	return *(*string)(unsafe.Pointer(&b.buf)) // 3
}

```

1. 与 `bytes.Buffer` 思路类似，既然 `string` 在构建过程中，会不断地被销毁和重建，那么就通过底层使用一个 `buf []byte` 来存放字符串的内容，从而尽量避免这个问题。
2. 对于写操作，就是简单地将 `byte` 写入到 `buf` 。
3. 为了解决 `bytes.Buffer` 存在的 `[]byte` 到 `string` 类型转换和内存拷贝问题，这里使用了一个 `unsafe.Pointer` 的指针转换操作，实现了直接将 `buf []byte` 转换为 `string` 类型，同时避免了内存申请、分配和销毁的问题。

## 一种引申的最佳实践方式

一般 Golang 标准库中使用的方式都是会逐步被推广的，成为某些场景下的最佳实践方式。

在 `strings.Builder` 使用到的 `*(*string)(unsafe.Pointer(&b.buf))` 可在其他的场景下使用，比如：如何在不进行内存分配的情况下，比较 `string` 和 `[]byte` 是否相等？

```go
func unsafeEqual(a string, b []byte) bool {
    bStr := *(*string)(unsafe.Pointer(&b))
    return a == bStr
}
```

 

