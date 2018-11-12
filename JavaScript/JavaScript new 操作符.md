# JavaScript new 操作符

## `new` 原理

我们可以在 ES5 [官方文档](https://www.ecma-international.org/ecma-262/5.1/#sec-13.2.2)中看到其对 `new` 创建对象过程的定义与约束：

> [13.2.2](https://www.ecma-international.org/ecma-262/5.1/#sec-13.2.2) [[Construct]]
>
> When the [[Construct]] internal method for a Function object F is called with a possibly empty list of arguments, the following steps are taken:
>
> 1. Let _obj_ be a newly created native ECMAScript object.
> 2. Set all the internal methods of _obj_ as specified in [8.12](https://www.ecma-international.org/ecma-262/5.1/#sec-8.12).
> 3. Set the [[Class]] internal property of _obj_ to `"Object"`.
> 4. Set the [[Extensible]] internal property of _obj_ to **true**.
> 5. Let _proto_ be the value of calling the [[Get]] internal property of _F_ with argument `"prototype"`.
> 6. If [Type](https://www.ecma-international.org/ecma-262/5.1/#sec-8)(_proto_) is Object, set the [[Prototype]] internal property of _obj_ to*proto*.
> 7. If [Type](https://www.ecma-international.org/ecma-262/5.1/#sec-8)(_proto_) is not Object, set the [[Prototype]] internal property of _obj_ to the standard built-in Object prototype object as described in [15.2.4](https://www.ecma-international.org/ecma-262/5.1/#sec-15.2.4).
> 8. Let _result_ be the result of calling the [[Call]] internal property of _F_, providing _obj_ as the **this** value and providing the argument list passed into [[Construct]] as _args_.
> 9. If [Type](https://www.ecma-international.org/ecma-262/5.1/#sec-8)(_result_) is Object then return _result_.
> 10. Return _obj_.

若执行 `new Foo()` ，其过程可简单描述为：

1. 创建新对象 `o` ；
2. 给新对象的内部属性赋值，关键是给 `[[Prototype]]` 属性赋值，构造原型链。若构造函数的原型为 `Object` 类型，则指向构造函数的原型，否则执行 `Object` 对象的原型；
3. 内部 `this` 指向 `o` ，执行构造函数；
4. 若构造函数显式返回对象，则返回该对象，否则返回 `o` 。

## `new` 实现

根据上两节的内容，我们可以尝试自己实现 `new` ：

```javascript
function my_new() {
  // 创建一个空的对象
  let o = {};
  // 获得构造函数
  let constructor = [].shift.call(arguments);
  // 链接到原型
  let prototype =
    typeof constructor.prototype === "object"
      ? constructor.prototype
      : Object.prototype;
  o.__proto__ = prototype;
  // 绑定 this，执行构造函数
  let result = constructor.apply(o, arguments);
  // 确保 new 出来的是个对象
  return typeof result === "object" ? result : o;
}

function Foo(name) {
  this.name = name;
}

const obj = my_new(Foo, "Jiahonzheng");
console.log(obj); // {name: "Jiahonzheng"}
```

## `instanceof` 原理

我们可以通过 `instanceof` 判断某个对象的类型，其内部机制是通过判断对象的原型链中是否能找到类型的 `prototype` 。

值得注意的是，`null instanceof Object` 返回 `false` ，所以 `null` 不是 `Object` 类型，尽管 `typeof null` 返回 `"Object"` 。

PS：`typeof null` 返回 `"Object"` 情况的出现是因为：在 JS 的最初版本中，使用的是 32 位系统，为了性能考虑使用低位存储变量的类型信息，`000` 开头代表的是对象，而 `null` 所有数据位为零，故 `typeof` 错误地将它判断为 `object` 。虽然现在内部类型判断代码已经改变，但是这个 Bug 仍在流传。

## `instanceof` 实现

```javascript
function instanceOf(left, right) {
  // 获得类型的原型
  let prototype = right.prototype;
  // 获得对象的原型
  left = left.__proto__;
  // 判断对象的类型是否等于类型的原型
  while (true) {
    if (left === null) return false;
    if (prototype === left) return true;
    left = left.__proto__;
  }
}

const Foo = function() {};
const o = new Foo();

console.log(instanceOf(o, Foo)); // true

// 重写 Foo 原型
Foo.prototype = {};
console.log(instanceOf(o, Foo)); // false
```

## 注意

- 当构造函数为箭头函数时，无法使用 `new` 调用，原因是箭头函数无 `[[Construct]]` 方法。

- 对于实例对象来说，都是通过 `new` 生成的。

- 推荐使用字面量的方式创建对象，因为在使用 `new Object()` 的方式创建对象，需要通过作用域链层层查找到 `Object` ，而在使用字面量时，则无此问题。

- `new` 运算符优先级问题：

  ```javascript
  function Foo() {
    return this;
  }
  Foo.func = function() {
    console.log("1");
  };
  Foo.prototype.func = function() {
    console.log("2");
  };

  new Foo.func(); // new (Foo.func());   -> 1
  new Foo().func(); // (new Foo()).func(); -> 2
  ```
