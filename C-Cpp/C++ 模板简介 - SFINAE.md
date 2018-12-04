# SFINAE, 类型检查, Concepts

​	SFINAE 机制是组成 C++ 模板机制及类型安全的相当重要的基础。全称是 Substitution failure is not an error。大概的意思就是只要找到了可用的原型（比如函数模板、类模板等）就不会编译错误。SFINAE 可以被用来进行模式匹配。在尝试本篇代码时请打开 C++17。

https://en.cppreference.com/w/cpp/language/sfinae

## 导入

​	为什么我们需要类型安全？除了能够保证用户调用我们编写的函数时传错参数之外，我们还可以避免这个情况：

```cpp
struct A {};
vector<A> v;
sort(v.begin(), v.end());
```

你可以看到一大坨一大坨的信息（真的很多，你试着编译一下就知道有多少（我可以告诉你就因为没有为 A 添加小于号运算符，产生了 200 行的编译错误信息）。

如果我们采用了这篇文章中的机制，我们可以将编译错误信息限制到 30 行以内（友好多了）。

~~不过如果你打开了 C++20，那么编译错误信息就会相当好看，然后本篇博客就被废掉了~~

## SFINAE

### 纯模板参数

我们看下面的一个例子：

```cpp
template <typename T>
struct A;

template <>
struct A<int>
{
    typedef int value_type;
};

template <class T, class U = typename A<T>::value_type>
void func(T);
```

如果我们调用 `func(int)`，那么上面的代码就可以编译，但是调用`func(double)`时，就会报错：

```
test.cpp: In function ‘int main()’:
test.cpp:18:13: error: no matching function for call to ‘func(double)’
     func(0.0);
             ^
test.cpp:11:6: note: candidate: template<class T, class U> void func(T)
 void func(T t)
      ^
test.cpp:11:6: note:   template argument deduction/substitution failed:
test.cpp:10:20: error: invalid use of incomplete type ‘struct A<double>’
 template <class T, class U = typename A<T>::value_type>
                    ^
test.cpp:2:8: note: declaration of ‘struct A<double>’
 struct A;
        ^
```

意思是我们在调用了 `func(double)` 时，`func`的完整的类型其实是`func<double, typename A<double>::value_type>`，也就是说 `func` 的类型依赖于 `A<double>::value_type`，但是我们知道我们只定义了 `A<int>::value_type`，而并未对其他的模板参数特化，也就是说 `A<double>` 其实是一个不完整的类型，显然我们不可以调用不完整的类型，因此编译失败。

### 函数参数（模板相关）

我们再来看另一种例子：

```cpp
struct A { typedef int typeA; };
struct B { typedef int typeB; };
struct C { typedef int typeC; };
template <typename T> void func(typename T::typeA) { cout << 1; }
template <typename T> void func(typename T::typeB) { cout << 2; }
template <typename T> void func(T) { cout << 3; }

int main()
{
    func<A>(1); // 输出 1，匹配到了第一个 func（只要找到一个匹配的即可）
    func<B>(2); // 输出 2，由于第一个 func 不能匹配，看第二个 func，匹配到了
    func<C>(3); // 编译失败，因为 C 既没有 typeA，也没有 typeB，两个 func 都不能匹配，编译失败
    func<int>(4); // 输出 3
}
```

看到上面的例子中 func 能匹配到相应的函数，这是因为匹配条件是唯一不冲突的（因此定义的顺序是没有关系的，因为不会产生歧义），我们再来看：

```cpp
template <typename T> void func(typename T::typeA) { cout << 1; }
template <typename T> void func(typename T::typeB) { cout << 2; }
template <typename T> void func(int) { cout << 3; }
```

如果我们的`func`函数是这么定义的，那么可以让`func<C>(3)`编译通过。但是`func<A>(1)`和`func<B>(2)`都将会编译失败，因为这两个函数调用既可以匹配前两条，又可以匹配第三条。所以会产生歧义从而编译失败。

### 其他

下面是 C++ Reference 上提到的一个例子：

```cpp
template <int I> void div(char(*)[I % 2 == 0 ? 1 : -1] = 0) {
    // this overload is selected when I is even
}

template <int I> void div(char(*)[I % 2 == 1 ? 1 : -1] = 0) {
    // this overload is selected when I is odd
}
```

这个例子很有趣，`div` 函数分成了两份，一份只在 `I`为偶数的情况下调用，一份只在`I`为奇数的情况下调用。首先这两个函数利用了函数参数类型的不一致从而避免了调用的歧义，其次再利用两个钟必有一个参数需要维度为负数的数组会编译失败的性质，根据 SFINAE 原则，选取那个不会编译失败的函数进行调用。从而区分开来了两个函数。多说一下：

​	所以你可能会想我们为什么要费这么大劲这么写两个 `div` 来区分 `I` 的奇偶性？而不是用 `if` 判断？这就涉及零开销的问题了，因为 `I` 的奇偶性我们在编译期就可以知道，那么判断 `I` 的时间如果能在编译时完成，如果再到运行时每次判断一下，就会造成运行时的额外开销。（很多 C++ 程序编译的时候都有跑编译的服务器集群跑的）

​	我们可以抽取一下使得这段代码更加易于阅读（需要启用 C++11）：

```cpp
template <int I> void div(typename std::enable_if<(I % 2 == 0)>::type * = 0) {
    
}
template <int I> void div(typename std::enable_if<(I % 2 == 1)>::type * = 0) {
    
}
```

​	我们在参数中使用了 `enable_if` 这个结构体代替了声明一个数组。`enable_if` 的模板参数为真时 type 才存在，否则不存在（就像之前的 `A::value_type` 是否存在一样）。然后参数中我们定义了 `type` 的指针，省略了这个参数的参数名，并且为其添加了默认值 0 使得我们不需要为其传值。至此你应该也能理解之前我们声明 `char` 数组时后面的 `=0` 是什么意思。我们之后详细介绍 `enable_if` 的内容。

​	我们可以使用模板的偏特化来模仿上面的例子（函数不支持模板的偏特化，所以只能用结构体内的静态函数代替）

```cpp
template <int I, bool = I % 2>
struct div;

template <int I, true>
struct div
{
    static void work() {
        
    }
};

template <int I, false>
struct div
{
    static void work() {
        
    }
};
```

## 应用

### 限定参数是特定的原生类型

我们花了很多篇幅介绍了 SFINAE 是什么，那么它能做什么？我们了解一下 `<type_traits>` 模板库中的函数。

假如我们现在有如下需求：

```cpp
template <typename T> T div(T t) { return t / 2; }
```

我们希望这个函数的参数是整型（`bool`, `int`, `long`, `unsigned` 等），而不希望是浮点型或者其他类型的变量传入（否则就不是向下取整的除以 2）。要怎么做呢？

```cpp
template <typename T>
typename std::enable_if<std::is_integral<T>::value, T>::type div(T t) {
    return t / 2;
}

template <typename T>
typename std::enable_if<std::is_floating_point<T>::value, T>::type div(T t) {
    return std::floor(t / 2);
}

// 对于既不是整型，又不是浮点数的类型，就会因为匹配不到两个函数从而编译失败
```

首先我们先介绍一个帮助模板 `is_integral<T>`，其模板参数为整型时，`is_integral<T>::value` 为真，否则为假。由于我们要利用 SFINAE 来实现类型检查，所以我们要在函数的某个地方插入一些代码使得模板参数 T 不为整型时这个函数将会编译失败从而让编译器不选择这个函数。纵观函数，我们发现返回值相当适合用来判断，我们之前介绍了 `enable_if` 的用法，其原理就是当模板参数中的布尔值为假时其 `type` 不存在从而导致编译失败进而阻止编译器采用该函数。那如果布尔值为真呢？`type` 就是 `T`。也就是说我们绕了一圈，最后回到了 `T`。最后我们只要将其模板参数中的布尔值令为 `is_integral_v<T>` ，就可以在 `T` 为整型的情况下 `type` 存在且为  `T` 从而不改变该函数的真实返回类型。

### enable_if

我们之前不断地提到了 `enable_if` 这个模板，怎么实现的呢？相信你通过之前的描述能够自己想出来怎么实现的，这里给出一种普通的实现方式：

```cpp
template <bool Cond, class T = void> struct enable_if {};
template <class T> struct enable_if<true, T> { typedef T type; };
```

那么如何利用这个 `enable_if` 就要发挥你的想象啦

### 判断是否存在某个函数

你可能想在 C++ 中使用类似接口的东西，比如这样：

```cpp
struct counter_base { virtual void count() = 0; };
struct counter : public counter_base {
    virtual void count() override {
        // do something
    }
};
```

然后你就可以这么干：

```cpp
void count(counter_base &i) { i.count(); }
```

这样如果我们调用了 `foo(counter())`，那么 `i.count()` 将会调用 `counter::count`。同时我们可以确保传进来的变量 `i` 确实有 `count()` 这个函数。



但是！如果 `foo` 这个函数调用的地方实在是太多了，多到居然虚函数居然会影响程序性能，以至于你被迫不这么干的时候，你要怎么做呢？大概就是：

```cpp
template <typename T> void count(T &i) { i.count(); }
```

这样完全可以，我们确保了 `T` 确实有 `count`，否则会编译失败。

但是如果我们哪天添加了一个需求：允许 `count<int>(var)`，然后使 `var++`来表示一次计数，你现在的程序就失效了。那么我们要怎么办呢？~~解决提出问题的人~~

那么我们就需要使用 SFINAE 了，考虑如何判断一个函数是否存在。我们只能通过调用对象实例的 `count` 函数才能知道是否存在，那么这个并不能使用类型的 SFINAE 检查，因为我们目前还没有一个工具可以以布尔值的形式得到一个函数是否存在，也就是说 `enable_if` 还无法使用。考虑 `decltype` 关键字，我们知道 `decltype` 关键字能得到一个表达式的类型，同时在表达式不合法时编译失败。那么我们可以考虑通过表达式检查模板类型 `T` 是否有 `count` 函数。比如一个函数的参数类型或返回类型中使用 `decltype(i.count())`，就可以出现这个函数编译失败的情况。那么接下来我们如何判断函数是否编译失败？答案就是 SFINAE。参考之前 `div` 参数的写法，我们就可以得到如下的程序：

```cpp
#include <bits/stdc++.h>
using namespace std;

struct counter {
    void count() {
        std::cout << "count";
    }
};

template <typename T>
struct has_count {
    
    template <typename K>
    static std::true_type test(decltype(std::declval<K>().count()) *);

    template <typename K>
    static std::false_type test(...); // 使用 ... 就可以区分开两个函数而不会产生歧义

    using type = decltype(test<T>(nullptr)); // 通过获得函数的返回类型来判断使用了哪个函数
};

template <typename T>
enable_if_t<has_count<T>::type::value> foo(T &t) {
    t.count();
}

template <typename T>
enable_if_t<is_integral_v<T>> foo(T &i) {
    std::cout << "int";
}

int main() {
    counter c; foo(c);
    int i; foo(i);
}
```

`test` 利用了 SFINAE，`declval<K>()` 表示拿到一个编译期的 `K` 的实例，这样我们就可以调用 `count` 函数，由于我们调用 `count` 函数是在编译期（`decltype` 的计算是在编译期，所以括号内的值是编译期计算的）调用的，所以可能产生一个 SFINAE 的编译错误，如果 `K` 没有 `count` 函数，那么编译期就会选中第二个 `test` 函数。那么我们怎么知道编译器选择了哪一个函数呢？我们可以通过函数返回值得到。首先第二个 test 函数的返回类型就是 `false_type`，而第一个 `test` 函数的返回类型就是 true_type。这样我们通过 `decltype(test<T>(blablabla))` 就可以得到 `true_type` 或者 `false_type` 从而区分开两个函数。

注意 `test` 函数必须要有模板 `K` 才能启用 SFINAE，如果 `declval<K>` 写成 `declval<T>` 是不行的，因为依赖了 `test` 本身以外的模板参数。



### 判断是否存在运算符

我们最开始提到了 `sort` 函数默认情况下将调用 `less` 比较器进而调用比较对象的小于运算符，如果小于运算符不存在将会造成大量的编译错误信息。那么我们如何实现判断运算符是否存在？或者判断比较器是否可用？和函数判断一样的：

```cpp
template <typename A, typename B, typename OperT>
struct has_operator {

    template <typename X, typename Y, typename Oper>
    static std::true_type test(decltype(std::declval<Oper>()(std::declval<X>(), std::declval<Y>())) *);

    template <typename X, typename Y, typename Oper>
    static std::false_type test(...);

    using type = decltype(test<A, B, OperT>(nullptr));
    static constexpr bool value = type::value;
};

static_assert(has_operator<int, int, std::less<>>::value, "failed");
```

我们知道了如何判断是否存在某种运算符，那么就可以做很多事情了：判断一个模板参数类型是不是 callable 的，或者判断 T 是不是迭代器（支持 `++` 等）。



上面的例子还能被修改成检查运算符范围类型的。你可以想想怎么做。

### 判断是否是基类

`std::is_base_of<Base, Derived>` 可以判断 `Derived` 是不是 `Base` 的子类。如何实现呢？和判断是否有函数、运算符一样，我们使用两个函数来表示。我们可以利用的性质是：`Derived` 是 `Base` 的子类，所以 `Derived*` 可以传进 `Base*` 参数的函数中，那么事情就变得简单了：

```cpp
template <typename Base, typename Derived>
struct is_base_of {

    template <typename X>
    static std::true_type test(Base *);

    template <typename X>
    static std::false_type test(...);

    using type = decltype(test<Base>(std::declval<Derived*>()));
    static constexpr bool value = type::value;
};
```



## Concepts

Concepts 真正地将我们从以上晦涩难懂拐弯抹角的代码（而且编译器也很累啊）中解救出来，我们看一些例子：

```cpp
template <typename T>
concept bool EqualityComparable = requires(T a, T b) {
    { a == b } -> bool
};

void f(EqualityComparable);

template <typename T>
void f(T) requires EqualityComparable<T>;
```

上面几行代码就表示 `f` 需要一个具有相等运算符，而且运算符返回类型为 `bool`。`HasCount` 就可以检查 `T` 是否有 `count` 函数。

下面是一些其他的例子：

```cpp
template <int T> concept Even = T % 2 == 0;
template <int T> concept Odd = T % 2 == 1;
template <Even I> void div();
template <Odd I> void div();


template <typename T>
concept bool HasCount = requires(T a) {
    { a.count() } -> void
};
```

