# Golang 方法、接口与匿名组合

我们知道 Golang 是一门面向对象的语言，通过在 `struct` 和 `interface` 上使用组合和多态来实现继承关系，而使用组合和多态的方式包括了**方法**、**接口**与**匿名组合**。

## 方法

在方法调用上，我们需要遵循以下规则。

- 类型 `T` 的可调用方法集：接收者为 `T` 和 `*T` 的方法
- 类型 `*T` 的可调用方法集：接收者为 `T` 和 `*T` 的方法
- 当方法的接收者为 `T` 时，即便是通过类型 `*T` 调用，该方法也操作的是对应接收者的副本，

当我们使用一个可寻址的类型 `T` 的值调用接收者为 `*T` 的方法时，Golang 将自动获取这个值的地址（指针），这也从另一个方面证实了规则一。

对于不可寻址的类型 `T` 的值，可正常调用接收者为 `T` 的方法，但在调用接收者为 `*T` 的方法时，会出现编译错误。

代码地址：https://play.golang.org/p/OVr6zhOrLCQ

```go
package main

import (
	"fmt"
)

type User struct {
	Name  string
	Count int
}

func (u User) SayHello() {
	u.Count++
	fmt.Printf("SayHello: %s\n", u.Name)
}

func (u *User) SayHelloPtr() {
	u.Count++
	fmt.Printf("SayHelloPtr: %s\n", u.Name)
}

func main() {
	u1 := User{
		Name:  "u1",
		Count: 0,
	}

	// 不可寻址的类型 T 的值，可调用接收者为 T 的方法
	User{
		Name:  "temp",
		Count: 0,
	}.SayHello()

	// 不可寻址的类型 T 的值，不可调用接收者为 *T 的方法
	// User{
	//	Name:  "temp",
	//	Count: 0,
	// }.SayHelloPtr()

	// T 类型的值可以调用接收者为 T 的方法
	u1.SayHello()
	fmt.Printf("User: %+v\n", u1)
	// T 类型的值可以调用接收者为 *T 的方法
	u1.SayHelloPtr()
	fmt.Printf("User: %+v\n", u1)

	u2 := &User{
		Name:  "u2",
		Count: 0,
	}
	// *T 类型的值可以调用接收者为 T 的方法
	u2.SayHello()
	fmt.Printf("User: %+v\n", u2)
	// *T 类型的值可以调用接收者为 *T 的方法
	u2.SayHelloPtr()
	fmt.Printf("User: %+v\n", u2)
}
```

程序输出结果

```
SayHello: u1
User: {Name:u1 Count:0}
SayHelloPtr: u1
User: {Name:u1 Count:1}
SayHello: u2
User: &{Name:u2 Count:0}
SayHelloPtr: u2
User: &{Name:u2 Count:1}
```

可见，第 28 和 31 行验证了规则一；第 39 和 42 行验证了规则二；第 40 行的输出结果验证了规则三。

## 接口

在接口调用上，我们需要遵循以下规则。

- 类型 `T` 的可调用方法集：接收者为 `T` 的方法
- 类型 `*T` 的可调用方法集：接收者为 `T` 和 `*T` 的方法

代码地址：https://play.golang.org/p/96RfgUC-Lox

```go
package main

import (
	"fmt"
)

type Hello interface {
	SayHello()
}

func SayHello(h Hello) {
	h.SayHello()
}

type User struct {
	Name  string
	Count int
}

func (u *User) SayHello() {
	u.Count++
	fmt.Printf("SayHello: %s\n", u.Name)
}

func main() {
	u := User{
		Name:  "u",
		Count: 0,
	}

	// 类型 T 的可调用方法集：接收者为 T 的方法
	// User 没有实现接收者为 User 的 SayHello 方法，故 User 未实现 Hello 接口
	// 若注释第 34 行，将发生编译错误
	// SayHello(u)

	// 类型 *T 的可调用方法集：接收者为 T 和 *T 的方法
	// User 实现了接收者为 *User 的 SayHello 方法，故 User 实现了 Hello 接口
	// 无编译错误
	SayHello(&u)

	fmt.Printf("User: %+v\n", u)
}
```

程序输出结果

```
SayHello: u
User: {Name:u Count:1}
```

## 匿名组合

结构体类型可以包含匿名或者嵌入字段（嵌入类型的名字充当了匿名组合的字段名）。

```go
type Admin struct {
	User
	Name  string
	Level int
}
```

### 数据和方法提升

在匿名组合中，可能会发生数据和方法提升的行为。

- 当我们嵌入一个类型时，这个内部类型的数据和方法就变成了外部类型的数据和方法，但是当内部方法被调用时，其接收者为内部类型，而非外部类型
- 当外部类型具有与内部类型同名的数据或方法时，对应的内部类型的数据或方法不会得到提升，但可通过字段名访问该数据或方法

代码地址：https://play.golang.org/p/e1YdyTSpcHD

```go
package main

import (
	"fmt"
)

type Admin struct {
	User
	Name string
}

func (a *Admin) SayHello() {
	a.User.Count++
	fmt.Printf("SayHello: %s\n", a.Name)
}

type User struct {
	Name  string
	Count int
}

func (u *User) SayHello() {
	u.Count++
	fmt.Printf("SayHello: %s\n", u.Name)
}

func main() {
	a := Admin{
		User: User{
			Name:  "Admin1-User",
			Count: 0,
		},
		Name: "Admin1",
	}

	// 内部类型数据没有得到提升
	fmt.Printf("Admin Name: %s\n", a.Name)
	// 通过字段名访问内部类型数据
	fmt.Printf("Admin User Name: %s\n", a.User.Name)

	// 内部类型方法没有得到提升
	a.SayHello()
	fmt.Printf("Admin: %+v\n", a)

	// 通过字段名访问内部类型方法
	a.User.SayHello()
	fmt.Printf("Admin: %+v\n", a)
}
```

程序输出结果

```
Admin Name: Admin1
Admin User Name: Admin1-User
SayHello: Admin1
Admin: {User:{Name:Admin1-User Count:1} Name:Admin1}
SayHello: Admin1-User
Admin: {User:{Name:Admin1-User Count:2} Name:Admin1}
```

### 方法

根据上述章节的规则描述，我们可以推导出在方法调用时，匿名组合所对应的规则。

- 类型 `S` 包含匿名字段 `T`
  - 类型 `S` 的可调用方法集：接收者为 `T` 和 `*T` 的方法
  - 类型 `*S` 的可调用方法集：接收者为 `T` 和 `*T` 的方法
- 类型 `S` 包含匿名字段 `*T`
  - 类型 S 的可调用方法集：接收者为 `T` 和 `*T` 的方法
  - 类型 `*S` 的可调用方法集：接收者为 `T` 和 `*T` 的方法

### 接口

根据上述章节的规则描述，我们可以推导出在接口调用时，匿名组合所对应的规则。

- 类型 `S` 包含匿名字段 `T`
  - 类型 `S` 的可调用方法集：接收者为 `T` 的方法
  - 类型 `*S` 的可调用方法集：接收者为 `T` 和 `*T` 的方法
- 类型 `S` 包含匿名字段 `*T`
  - 类型 S 的可调用方法集：接收者为 `T` 和 `*T` 的方法
  - 类型 `*S` 的可调用方法集：接收者为 `T` 和 `*T` 的方法

### 示例代码

我们可以通过以下例子验证匿名组合在方法调用和接口调用上的规则。

代码地址：https://play.golang.org/p/IBxh1FnzN8s

```go
package main

import (
	"fmt"
)

type Hello interface {
	SayHello()
}

func SayHello(h Hello) {
	h.SayHello()
}

type Admin struct {
	User
	Name string
}

type User struct {
	Name  string
	Count int
}

func (u *User) SayHello() {
	u.Count++
	fmt.Printf("SayHello: %s\n", u.Name)
}

func main() {
	a := Admin{
		User: User{
			Name:  "Admin1-User",
			Count: 0,
		},
		Name: "Admin1",
	}

	// 方法调用
	// 类型 S 的可调用方法集：接收者为 T 和 *T 的方法
	a.SayHello()
	fmt.Printf("Admin: %+v\n", a)

	// 方法调用
	// 类型 *S 的可调用方法集：接收者为 T 和 *T 的方法
	(&a).SayHello()
	fmt.Printf("Admin: %+v\n", a)

	// 接口调用
	// 类型 S 包含匿名字段 T
	// 	类型 S 的可调用方法：接收者为 T 的方法
	// User 没有实现接收者为 User 的 SayHello 方法
	// 若注释第 54 行，将发生编译错误
	// SayHello(a)
	// fmt.Printf("Admin: %+v\n", a)

	// 接口调用
	// 类型 S 包含匿名字段 T
	// 	类型 *S 的可调用方法：接收者为 T 和 *T 的方法
	// User 实现接收者了为 *User 的 SayHello 方法
	// 无编译错误
	SayHello(&a)
	fmt.Printf("Admin: %+v\n", a)
}
```

程序输出结果

```
SayHello: Admin1-User
Admin: {User:{Name:Admin1-User Count:1} Name:Admin1}
SayHello: Admin1-User
Admin: {User:{Name:Admin1-User Count:2} Name:Admin1}
SayHello: Admin1-User
Admin: {User:{Name:Admin1-User Count:3} Name:Admin1}
```

## 相关资料

- [Is Go an Object Oriented language?](https://spf13.com/post/is-go-object-oriented/)
- [Go-方法调用与接口](https://ninokop.github.io/2017/10/29/Go-%E6%96%B9%E6%B3%95%E8%B0%83%E7%94%A8%E4%B8%8E%E6%8E%A5%E5%8F%A3/)
