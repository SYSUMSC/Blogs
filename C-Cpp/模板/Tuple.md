# Tuple

## 导入

我们可以通过 `std::tuple` 构造一个复合类型，比如可以使得函数拥有“多个”返回值：

```cpp
std::tuple<int, int> divide(int a, int b) {
    return make_tuple(a / b, a % b);
}
```

然后我们可以通过 `get<>` 函数来获得 `tuple` 内存储的值：

```cpp
auto result = divide(4, 3);
int quotient = std::get<0>(result); // 获取 tuple 的第 0 个数
int remainder = std::get<1>(result); // 获取 tuple 的第 1 个数
```

还可以：

```cpp
int quotient, remainder;
std::tie(quotient, remainder) = divide(4, 3);
```

或者在 C++17 中：

```cpp
auto [ quotient, remainder ] = divide(4, 3);
```

是不是更有多返回值的感觉？

## 实现

首先我们要知道我们如何实现一个 `tuple`。借鉴之前实现一系列模板元编程得到的基本函数的思想，我们只令 `tuple<head, tail...>` 这个类型只存 `head` 类型的变量，剩余类型的变量在 `tuple<tail...>` 中保存。那么如何实现这样的分开保存呢？一种方法是在 `tuple` 类内存两个变量，一个是 `head` 类型的变量即当前值，第二个是 `tuple<tail...>` 类型的变量存剩余值。这样我们在 `get` 的时候就直接递归拿值即可（比较朴素的方法）。但是这样的话 `get` 函数的性能就会下降，因为在我们只需要特定的值的情况下， `get` 却可能需要递归比较多层，而`get`函数本来要获取的值的下标识编译期确定的，我们却在运行期做了遍历 `tuple` 的工作，不符合我们的零开销策略（不过加上 `constexpr` 之后这种方法的开销就降为零了）。



接下来我们介绍另一种方法实现，就是**继承**，如果我们存在 `tuple<Args...>`，我们要找特定的值，遍历一次其祖先，和第一种方法一样。为了能够避免遍历，我们在模板参数中添加一个 `N` 表示当前的部分 `tuple` 是原 `tuple` 的第几个元素开始的，新的类我们称作 `tuple_impl`。那么 `tuple_impl<N, head, tail...>` 的父类就理所当然的是 `tuple_impl<N + 1, tail...>`，即父类是下一个元素开始的子 `tuple`。那么这么实现有什么好处？我们在实现 `get` 函数的时候就可以这么写：

```cpp
template <std::size_t N, typename _Head, typename... _Tail>
constexpr const _Head &get_helper(const tuple_impl<N, _Head, _Tail...> &t) {
    return tuple_impl<N, _Head, _Tail...>::head(t);
}

template <std::size_t N, typename... Args>
constexpr const tuple_element_t<N, tuple<Args...>> &
get(const tuple<Args...> &t) {
    return get_helper<N>(t);
}
```

你可以看到，`get`函数调用了`get_helper<N>`函数，`tuple<Args...>`类是`tuple_impl<0, Args...>` 的子类，`t` 传入 `get_helper<N>` 后变成了 `tuple_impl<N, Args2...>`，而这个 `Args2` 是我们交由编译器自动推导的（`get_helper<N>` 的参数我们只给了 `N` 而没有给 `_Head` 和 `_Tail`）。也就是说我们利用了继承的机制后，我们可以直接拿到 `tuple<Args...>` 的第 N 个父类 `tuple_impl<N, Args2...>`！从而直接获得第 `N` 个元素，从而避免了递归调用（而且由于我们没有虚函数和虚继承，所以这个是零开销的）。



下面是完整的 `tuple` 实现，完成了大部分功能（实现了`tie` 和 `get`）。

```cpp
template <size_t N, typename... _Elements>
struct tuple_impl;

template <std::size_t N, typename _Head>
struct tuple_impl<N, _Head> {

    explicit tuple_impl(const _Head &head) : value(head) {}

    template <typename _UHead>
    void assign(const tuple_impl<N, _UHead> &in) {
        head(*this) = tuple_impl<N, _UHead>::head(in);
    }

    static constexpr _Head &head(tuple_impl &t) {
        return t.value;
    }

    static constexpr const _Head &head(const tuple_impl &t) {
        return t.value;
    }

    _Head value;
};

template <std::size_t N, typename _Head, typename... _Tail>
struct tuple_impl<N, _Head, _Tail...>
    : public tuple_impl<N + 1, _Tail...> {

    typedef tuple_impl<N + 1, _Tail...> base_type;

    explicit tuple_impl(const _Head &head, const _Tail &... tail)
        : base_type(tail...), value(head) {}
    
    template <typename... _UElements>
    void assign(const tuple_impl<N, _UElements...> &in) {
        head(*this) = tuple_impl<N, _UElements...>::head(in);
        tail(*this).assign(tuple_impl<N, _UElements...>::tail(in));
    }

    static _Head &head(tuple_impl &t) {
        return t.value;
    }

    static const _Head &head(const tuple_impl &t) {
        return t.value;
    }

    static base_type &tail(tuple_impl &t) {
        return t;
    }

    static const base_type &tail(const tuple_impl &t) {
        return t;
    }

    _Head value;
};

template <typename... _Elements>
struct tuple : public tuple_impl<0, _Elements...> {

    typedef tuple_impl<0, _Elements...> base_type;

    constexpr tuple(const _Elements &... elements)
        : base_type(elements...) {}

    tuple &operator=(const tuple &in) {
        this->assign(in);
        return *this;
    }

    template <typename... _UElements>
    tuple &operator=(const tuple<_UElements...> &in) {
        this->assign(in);
        return *this;
    }
};

template <std::size_t N, typename... _Elements>
constexpr tuple<_Elements &...> tie(_Elements &... args) {
    return tuple<_Elements &...>(args...);
}

template <std::size_t N, typename... _Elements>
struct tuple_element;

template <std::size_t N, typename _Head, typename... _Tail>
struct tuple_element<N, tuple<_Head, _Tail...>> {
    using type = typename tuple_element<N - 1, tuple<_Tail...>>::type;
};

template <typename _Head, typename... _Tail>
struct tuple_element<0, tuple<_Head, _Tail...>> {
    using type = _Head;
};

template <std::size_t N, typename... _Elements>
using tuple_element_t = typename tuple_element<N, _Elements...>::type;

template <std::size_t N, typename _Head, typename... _Tail>
constexpr _Head &get_helper(tuple_impl<N, _Head, _Tail...> &t) {
    return tuple_impl<N, _Head, _Tail...>::head(t);
}

template <std::size_t N, typename _Head, typename... _Tail>
constexpr const _Head &get_helper(const tuple_impl<N, _Head, _Tail...> &t) {
    return tuple_impl<N, _Head, _Tail...>::head(t);
}

template <std::size_t N, typename... Args>
constexpr tuple_element_t<N, tuple<Args...>> &
get(tuple<Args...> &t) {
    return get_helper<N>(t);
}

template <std::size_t N, typename... Args>
constexpr const tuple_element_t<N, tuple<Args...>> &
get(const tuple<Args...> &t) {
    return get_helper<N>(t);
}
```