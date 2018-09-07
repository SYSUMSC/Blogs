# JavaScript this 绑定规则

在 JavaScript 中，`this` 的绑定规则有4种，规则间存在着不同的优先级。

## 默认绑定

在非严格模式下，默认绑定会将 `this` 指向全局对象。

```javascript
function foo() {
    console.log(this.a);
}
var a = 2;
foo(); // 2
```

在严格模式下，默认绑定会将 `this` 指向 `undefined` ，此时对于上述代码，会报 `TypeError: this is undefined` 的类型错误。

## 隐式绑定

当函数被调用时，若函数引用具有上下文对象，则隐式绑定会将 `this` 绑定到这个上下文对象。

对象属性引用链中，只有最顶层或者说最后一层会影响函数的调用位置，故此情况下，隐式绑定会将 `this 绑定到最后一层对象。

一个最常见的 `this` 绑定问题就是被隐式绑定的函数会丢失绑定对象，也就是说它会应用默认绑定，从而将 `this` 绑定到全局对象或者 `undefined` 上，取决于是否是严格模式。

```javascript
function foo() {
    console.log(this.a);
}
var a = "Oops, global"
var obj2 = {
    a: 2,
    foo: foo
};
var obj1 = {
    a: 42,
    obj2: obj2
};
var bar = obj2.foo; //函数别名
obj2.foo(); // 42 - obj2是foo被调用时的上下文对象
obj1.obj2.foo(); // 42 - obj2是对象属性引用链的最后一层
bar(); // "Oops, global" - 隐式丢失
```

由于 `bar` 是 `obj.foo` 的一个引用，故在调用 `bar` 时，相当于不带任何修饰的 `foo` 调用，故应用了默认绑定，即发生了隐式丢失。

一种更微妙、更常见且更出乎意料的隐式丢失情况，发生在传入回调函数时：

```javascript
function foo() {
    console.log(this.a);
}
function doFoo(fn) {
    // fn是foo的引用
    fn();
}
var obj = {
    a: 2,
    foo: foo
};
var a = "Oops, global";
doFoo(obj.foo); // "Oops, global"
```

正如我们所见到的一样，回调函数丢失 `this` 绑定是非常常见的，除此之外，有种情况 `this` 的行为会出乎我们意料：调用回调函数的函数可能会修改 `this` 。

## 显式绑定

我们可以通过 `apply` 、`call` 和 `bind` 将函数中的 `this` 绑定到指定对象。

```javascript
function foo() {
    console.log(this.a);
}
var a = "Oops, global";
var obj = {
    a: 2
};

foo.call(obj); // 2
```

### 被忽略的 this

如果我们把 `null` 或 `undefined` 作为 `this` 的绑定对象传入 `call` 、`apply` 或者是 `bind` ，这些值在调用时会被忽略，实际应用的是默认绑定规则。

一种非常常见的做法是使用 `apply(..)` 来“展开”一个数组，并当作参数传入一个函数。类似地，`bind(..)` 可以对参数进行柯里化（预先设置一些参数）。

```javascript
function foo(a, b) {
    console.log("a: " + a + ", b: " + b);
}

// 把数组“展开”成参数
foo.apply(null, [2, 3]); // a: 2, b: 3

// 使用 bind(..) 进行柯里化
var bar = foo.bind(null, 2);
bar(3); // a: 2, b: 3
```

在 ES6 中，可以用 `...` 操作符代替 `apply(..)` 来“展开”数组，`foo(…[1, 2])` 和 `foo(1, 2)` 是一样的，这样可以避免不必要的 `this` 绑定。

然而，总是使用 `null` 来忽略 `this` 绑定可能产生一些副作用，如果某个函数确实使用了 `this` ，那默认绑定规则就会把 `this` 绑定到全局对象。

## new 绑定

在 JavaScript 中，构造函数只是一些使用 `new` 操作符时被调用的函数，它们并不属于某个类，也不会实例化一个类。实际上，它们甚至都不能说是一种特殊的函数类型，它们只是被 `new` 操作符调用的普通函数而已。

> 不存在所谓的“构造函数”，只有对函数的“构造调用”。

使用 `new` 来调用函数时，会自动执行以下操作：

+ 创建一个全新的对象。
+ 这个新对象会被执行 [[Prototype]] 连接。
+ 这个新对象会绑定到函数调用的 `this` 。
+ 如果函数没有返回其他对象，那么 `new` 表达式中的函数调用会自动返回这个新对象。

```javascript
function foo1() {
    this.a = 1;
}
function foo2() {
    this.a = 2;
    return {
        a: 3
    };
}
var a = new foo1();
var b = new foo2();
console.log(a); // {a: 1}
console.log(b); // {a: 3}
```

## 箭头函数

箭头函数中的 `this` 只取决于它外面的第一个不是箭头函数的函数的 `this` 。

```javascript
function foo() {
    return () => {
        return () => {
          	console.log(this.a);  
        };  
    };
}
var a = 2;
console.log(foo()()()); // 2 
```

箭头函数的 `this` 一旦绑定了上下文，就不会被任何代码改变。

## 判断 this 指向

1. 函数是否在 `new` 中调用（new 绑定）？如果是的话，`this` 绑定的是新创建的对象。
2. 函数是否通过 `call` 、`apply` 或 `bind` 显示绑定？如果是的话，`this` 绑定的是指定的对象。
3. 函数是否在某个上下文对象中调用（隐式绑定）？如果是的话，`this` 绑定的是那个上下文对象。
4. 如果都不是的话，则使用默认绑定。如果在严格模式，则绑定到 `undefined` ，否则绑定到全局对象。