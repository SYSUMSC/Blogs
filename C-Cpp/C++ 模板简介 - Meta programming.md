# 编译期的整数操作

​	模板元编程是一个挺有意思~~（但是毫无卵用）~~的东西。比如我们可以实现编译期的快速排序。（但是 `constexpr` 函数基本把这部分给废掉了）

## 导入

​	下面是一个例子：

```cpp
template <int A, int B>
struct add { static constexpr int value = A + B; };

cout << add<1, 2>::value;
```

输出 3，但是这个 3 是在编译期计算出来的而不是运行时。不过实际上。。

```cpp
template <typename T>
constexpr T add(T a, T b) { return a + b; }
```

就可以实现一样的东西，而且看起来更直观。

那么我们学这个有什么用呢？~~（纯粹好玩）~~

## 基本类型

### integer_constant

我们希望一些运算的工作放在编译期完成，那么我们就需要一些东西来存储或表示我们的运算结果。我们从导入中提到的例子能看到，我们表示一个编译期的整数，都是在模板的参数中表示。那么我们就知道了应该如何表示一个编译期的常量：

```cpp
template <typename T, T v>
struct integral_constant {
    static constexpr T value = v;
    typedef T value_type;
    constexpr operator T() const noexcept { return value; }
    constexpr T operator()() const noexcept { return value; }
};

template <int v>
using integer_constant = integral_constant<int, v>;
```

这样我们就能在模板参数中表示一个整型的常量，常量的值就在模板参数中，比如：`integer_constant<5>` 这个类型就能表示 5。

### integer_sequence

函数式编程中另一个重要的数据结构就是列表，我们再来看如何表示一个列表：

```cpp
template <typename T, T... values>
struct integral_sequence { };

template <int ... values>
using integer_sequence = integral_sequence<int, values...>;
```

这里利用了可变模板参数来实现列表。这样我们就可以通过 `integer_sequence<1, 2, 3, 4, 5>` 来表示列表：`[1, 2, 3, 4, 5]`。

## 基本函数

现在有了基本数据类型，我们就可以在这上面构建一系列的函数了。我们已经知道了模式匹配，那么我们就可以照猫画虎地写出相应的基本函数：

### length

```cpp
template <typename integer_sequence>
struct length;

template <int... values>
struct length<integer_sequence<values...>> : integral_constant<size_t, sizeof...(values)> {};

template <typename T>
constexpr size_t length_v = length<T>::value;
```

### push_front

```cpp
template <int V, typename integer_sequence>
struct push_front;

template <int V, int... values>
struct push_front<V, integer_sequence<values...>> {
	using type = integer_sequence<V, values...>;
};

template <int V, typename T>
using push_front_t = typename push_front<V, T>::type;

////////

push_front_t<1, integer_sequence<2, 3, 4, 5>> === integer_sequence<1, 2, 3, 4, 5>;
```

### empty

```cpp
template <typename integer_sequence>
struct empty;

template <>
struct empty<integer_sequence<>> : std::true_type {};

template <int... values>
struct empty<integer_sequence<values...>> : std::false_type {};

template <typename T>
constexpr bool empty_v = empty<T>::type::value;

////////

empty_v<integer_sequence<>> == true;
empty_v<integer_sequence<1, 2, 3, 4, 5>> == false;
```

### conditional

这个 `if` 像一个三目表达式。函数式语言中 `if` 只是一个表达式，必须有 `else` 子句。

```cpp
template <bool Cond, typename Then, typename Else>
struct conditional;

template <typename Then, typename Else>
struct conditional<true, Then, Else> { using type = Then; };

template <typename Then, typename Else>
struct conditional<false, Then, Else> { using type = Else; };

template <bool Cond, typename Then, typename Else>
using conditional_t = typename conditional<Cond, Then, Else>::type;

///////

conditional<true, true_type, false_type> === true_type;
conditional<false, false_type, true_type> === true_type;
```

### map

```cpp
template <template <int> typename Mapper, typename integer_sequence>
struct map;

template <template <int> typename Mapper>
struct map<Mapper, integer_sequence<>> {
	using type = integer_sequence<>;
};

template <template <int> typename Mapper, int head, int... tail>
struct map<Mapper, integer_sequence<head, tail...>> {
	using type = typename push_front<Mapper<head>::value, typename map<Mapper, integer_sequence<tail...>>::type>::type;
};

template <template <int> typename Mapper, typename T>
using map_t = typename map<Mapper, T>::type;

/////////
template <int T> struct increment { static constexpr int value = T + 1; };
map_t<increment, integer_sequence<1, 2, 3, 4, 5>> === integer_sequence<2, 3, 4, 5, 6>;
```

### filter

```cpp
filter f [] = []
filter f (x:[]) = if (f x) [x] else []
filter f (x:xs) = if (f x) [x] ++ (filter f xs) else (filter f xs)
```

```cpp
template <template <int> typename Filter, typename integer_sequence>
struct filter;

template <template <int> typename Filter>
struct filter<Filter, integer_sequence<>> {
	using type = integer_sequence<>;
};

template <template <int> typename Filter, int head, int... tail>
struct filter<Filter, integer_sequence<head, tail...>> {
	using type = typename conditional<Filter<head>::value,
		typename push_front<head, typename filter<Filter, integer_sequence<tail...>>::type>::type,
		typename filter<Filter, integer_sequence<tail...>>::type
		>::type;
};

template <template <int> typename Filter, typename T>
using filter_t = typename filter<Filter, T>::type;

/////////
template <int T> struct is_even { static constexpr bool value = T % 2 == 0; };
filter_t<is_even, integer_sequence<1, 2, 3, 4, 5>> === integer_sequence<2, 4>;
```

### print

```cpp
void print(integer_sequence<> seq) {
    std::cout << std::endl;
}

template <int head>
void print(integer_sequence<head> seq) {
    std::cout << head << std::endl;
}

template <int head, int... tail>
void print(integer_sequence<head, tail...> seq) {
    std::cout << head << ' ';
    print(integer_sequence<tail...>());
}

//////// for integer_sequence<1, 2, 3, 4, 5> outputs "1 2 3 4 5\n"
```

### concat

```cpp
template <typename I1, typename I2>
struct concat;

template <typename I1, typename I2>
using concat_t = typename concat<I1, I2>::type;

template <typename I1>
struct concat<I1, integer_sequence<>> {
    using type = I1;
};

template <typename I1, int head, int... tail>
struct concat<I1, integer_sequence<head, tail...>> {
    using type = concat_t<push_back_t<head, I1>, integer_sequence<tail...>>;
};
```

### reverse

```cpp
template <typename integer_sequence>
struct reverse;

template <>
struct reverse<integer_sequence<>> {
    using type = integer_sequence<>;
};

template <int head, int... tail>
struct reverse<integer_sequence<head, tail...>> {
    using type = typename push_back<head, typename reverse<integer_sequence<tail...>>::type>::type;
};

template <typename T>
using reverse_t = typename reverse<T>::type;

/////////
reverse_t<integer_sequence<1, 2, 3, 4, 5>> === integer_sequence<5, 4, 3, 2, 1>;
```

### reduce

```cpp
template <int e, template <int, int> typename F, typename integer_sequence>
struct reduce;

template <int e, template <int, int> typename F>
struct reduce<e, F, integer_sequence<>> {
    static constexpr int value = e;
};

template <int e, template <int, int> typename F, int head, int... tail>
struct reduce<e, F, integer_sequence<head, tail...>> {
    static constexpr int value = F<head, reduce<e, F, integer_sequence<tail...>>::value>::value;
};

////////// samples

template <int A, int B> struct add {
    static const vlaue = A + B;
};

cout << reduce<0, add, integer_sequence<1, 2, 3, 4, 5>>::value;
// outputs 15
```

### format

我在这里插一句，已经不知道该写在哪里了。。。我们可以利用 C++ 的可变模板参数列表实现一个简易的字符串格式化工具函数，比如 `format("Welcome! {}. Current time is {}:{}:{}.", sir, hour, minute, second)`从而实现对 `String.format` 的模仿。这个应该很容易想出来怎么写：

```cpp
template <typename StreamT>
void format0(StreamT &ss, const wchar_t *fmt)
{
    ss << fmt;
}

template <typename StreamT, typename T, typename... Args>
void format0(StreamT &ss, const wchar_t *fmt, const T &arg0, Args... args)
{
    if (!(*fmt))
        return;
    if (*(fmt + 1) && *fmt == L'{' && *(fmt + 1) == L'}')
    {
        ss << arg0;
        format0(ss, fmt + 2, args...);
    }
    else
    {
        ss << *fmt;
        format0(ss, fmt + 1, arg0, args...);
    }
}

template <typename... Args>
std::wstring format(const wchar_t *fmt, Args... args)
{
    std::owstringstream ss;
    format0(ss, fmt, args...);
    return ss.str();
}
```

## 排序

### 插入排序

接下来我们实现一个编译期的插入排序。首先我们看 Haskell 的版本。首先是插入函数

```haskell
insert :: Ord a => a -> [a] -> [a]
insert x [] = [x]
insert x (y:ys) = if x < y then x:y:ys else y:insert x ys
```

那么实现插入排序就很简单了：

```haskell
insertionSort :: Ord a => [a] -> [a]
insertionSort [] = []
insertionSort (x:xs) = insert x (insertionSort xs)
```

我们对应实现 C++ 版本：

```cpp
template <template <int, int> typename Compare, int x, typename integer_sequence>
struct insert;

template <template <int, int> typename Compare, int x>
struct insert<Compare, x, integer_sequence<>> {
    using type = integer_sequence<x>;
};

template <template <int, int> typename Compare, int x, int head, int... tail>
struct insert<Compare, x, integer_sequence<head, tail...>> {
    using type = typename conditional<Compare<x, head>::value,
                                      integer_sequence<x, head, tail...>,
                                      typename push_front<head, typename insert<Compare, x, integer_sequence<tail...>>::type>::type>::type;
};

template <template <int, int> typename Compare, typename integer_sequence>
struct insertion_sort;

template <template <int, int> typename Compare>
struct insertion_sort<Compare, integer_sequence<>> {
    using type = integer_sequence<>;
};

template <template <int, int> typename Compare, int head, int... tail>
struct insertion_sort<Compare, integer_sequence<head, tail...>> {
    using type = typename insert<Compare, head, typename insertion_sort<Compare, integer_sequence<tail...>>::type>::type;
};
```

测试样例：

```cpp
template <int A, int B>
struct lessThan {
    static const bool value = A < B;
};

int main() {
    using s = integer_sequence<7, 5, 4, 6, 0>;
    print(typename insertion_sort<lessThan, s>::type());
}
```

### 快速排序

我们先看 Haskell 版本的快速排序：

```haskell
quicksort [] = []
quicksort (x:xs) = quicksort small ++ (x : quicksort large)
    where small = [y | y <- xs, y < x]
          large = [y | y <- xs, y >= x]
```

这个快速排序很简单，每次取列表的第一个元素作为中值。

```cpp
template <template <int, int> typename Compare, typename integer_sequence>
struct quick_sort;

template <template <int, int> typename Compare>
struct quick_sort<Compare, integer_sequence<>> {
    using type = integer_sequence<>;
};

template <template <int, int> typename Compare, int head, int... tail>
struct quick_sort<Compare, integer_sequence<head, tail...>> {
    template <int y>
    using partial_compare_t = Compare<y, head>;

    template <int y>
    struct not_partial_compare_t {
        static constexpr bool value = !partial_compare_t<y>::value;
    };

    using type = concat_t<
        typename quick_sort<Compare, filter_t<partial_compare_t, integer_sequence<tail...>>>::type,
        push_front_t<head, typename quick_sort<Compare, filter_t<not_partial_compare_t, integer_sequence<tail...>>>::type>>;
};
```

