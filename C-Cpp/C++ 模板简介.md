# C++ 模板简介

~~由于我太菜了 mq 大佬不要 D 我~~

首先推荐 [C++ 官方模板介绍](https://en.cppreference.com/w/cpp/language/template_specialization)



1. SFINAE（很多内容被 Concepts 取代）
2. Meta programming（很多内容被 constexpr 函数取代）
3. Tuple
4. Any

~~（看完上面介绍你就觉得全部讲的毫无用处了）~~



建议你先了解一下模式匹配与模板偏特化是什么东西，否则你可能 4 篇文章都看不懂。

## 模式匹配

模式匹配指的是，检查某一个词汇序列是否满足给定的一些模板。对于函数式编程语言来说（当然 C++ 并不是函数式编程语言），函数的参数就可以进行模式匹配，比如：

```haskell
fac :: Int -> Int -- 定义函数 fac 的类型，你可以理解为第一个参数是 Int，返回值是 Int
fac 0 = 1 -- 规定第一个参数为 0 的时候返回值是 1
fac n = n * fac (n - 1) -- 否则对于其他的参数 n，等于 n - 1 的阶乘的值乘上 n
```

上述代码是一个简单的模式匹配的例子，我指定了某个参数为什么值的时候函数的返回值是什么。我们再来看一个例子：

```haskell
len :: [a] -> Int -- 定义函数 len 的类型，你可以理解为第一个参数为元素类型任意的列表（数组），返回值是 Int
len [] = 0 -- 如果列表是空的列表，那么显然其长度为 0
len (x:[]) = 1 -- 如果列表只有一个元素，那么显然其长度为 
len (x:xs) = 1 + len xs
```

`len` 函数的参数我们分了 3 种情况讨论，而这三种情况都是相互排斥的。我们考虑 C++ 中模板偏特化的模仿：

```cpp
template <typename T, T... List>
struct len;

template <typename T>
struct len<T> { static const size_t value = 0; };

template <typename T, typename X>
struct len<T, X> { static const size_t value = 1; };

template <typename T, typename X, typename... XS>
struct len<T, X, XS...> { static const size_t value = 1 + len<T, XS...>::value; };
```

这样我们就模仿实现了 Haskell 版本的代码，你可以看到前两行是类型声明，之后 3 个 `len` 一一对应。也就是说，模板的偏特化实际上就是在做模式匹配。