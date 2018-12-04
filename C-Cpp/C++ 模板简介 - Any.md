# Any

你有没有过需要在 C++ 中使用类似 `object` 这种通用类型的需求呢？C++ 中如何类型安全地保存一个什么类型都有可能是的变量？（这个大概算类型擦除？）

## 导入

如果你写过一些与运行时状态有关的东西，比如一个列表中的数据可能是任意一种数据，或者比如 SQL 数据表中的一行可能是很多的类型，这个时候你需要用一个 `vector` 保存，如果在其他语言里，你可以：

```c#
List<object> row = new List<object> { 1, 1.0, "123", DateTime.Now };
```

然后我们可以 `(DateTime) row[3]` 来获取我们想要的数据（你肯定会有一个表说明这个 `row` 第几个元素是什么类型的数据，或者我们判断 `row[3] is DateTime` 来判断第 4 个元素是不是 `DateTime` 以便使用。如何在 C++ 中实现一个 `object`？你可能想到了 `void*`，没错 `void*` 可以存储任意类型的数据，但是很遗憾不直接支持 `is` 或者 `instanceof` 这种判断是不是某个类型的方式。下面我们介绍如何存储任意类型的数据以及如何判断是不是某个类型的数据。

## 实现

下面是使用 `RTTI` 的实现方法（使用 `type_index` 来比较是否是同一个类型）：

```cpp
#include <memory>
#include <typeindex>
using namespace std;

class object_t {
	class bad_cast : exception { };

	struct base {
		virtual unique_ptr<base> clone() const = 0;
	};

	template<typename T>
	struct derived : base {
		T value;

		template<typename U>
		derived(U&& value) : value(forward<U>(value)) {}

		unique_ptr<base> clone() const override {
			return unique_ptr<base>(new derived<T>(value));
		}
	};

	unique_ptr<base> clone() const {
		if (!empty()) return ptr->clone();
		else return nullptr;
	}

	unique_ptr<base> ptr;
	type_index index;

public:
	object_t();
	object_t(const object_t &that);
	object_t(object_t &&that);

	template <typename U, typename enable_if<!is_same<typename decay<U>::type, object_t>::value, bool>::type = true>
	object_t(U&& value) : ptr(new derived<typename decay<U>::type>(forward<U>(value))), index(type_index(typeid(typename decay<U>::type))) {}

	bool empty() const;

	template<typename U>
	bool is() const {
		return index == type_index(typeid(U));
	}

	type_index type() const {
		return index;
	}

	template <typename U>
	U cast() const {
		if (!is<U>())
		{
			throw bad_cast();
		}
		auto d = dynamic_cast<derived<U>*> (ptr.get());
		return d->value;
	}

	object_t& operator= (const object_t& b) {
		if (ptr != b.ptr) {
			ptr = b.clone();
			index = b.index;
		}
		return *this;
	}
};
```

有没有方法不使用 `RTTI` 的那一套呢？现在是通过类型对应某种唯一的值相等就可以判断类型是否一致。如何为每一个类型提供一个唯一的整数值呢？很简单，我们实现这样的一个小东西：

```cpp
template <typename T>
struct type_identifier {
    static void func() {}
};
```

那么对于任意一个类型 `T`，其 `&type_identifier<T>::func` 函数都是唯一的不同的函数，因此这些函数的地址也就都不一样，那么我们就可以拿这个地址作为类型的唯一标识符。那么我们就可以保存 `func` 的地址而不是 `type_index`。