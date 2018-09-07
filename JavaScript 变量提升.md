# JavaScript 变量提升

**变量提升是一个将变量声明或者函数声明提升到作用域起始处的过程**，即变量声明 `var` 和函数声明 `function fun() {..}` 在会发生变量提升过程。

但对于 ES2015 引入的 `let` ，变量提升是不能准确描述其变量初始化过程和可用性判断的，即 **`let` 变量提升是不同寻常的**。

ES2015 为 `let` 提供了一个不同的改进机制，它要求了更严格的变量声明方式（即在定义变量前是无法访问它的），从而在结果上保证了更好的代码质量。

在本篇博文中，我们一起深入了解这个过程的更多细节。

## 变量的生命周期

当引擎使用变量时，它们的生命周期包含以下阶段：

+ **声明阶段**，这一阶段在作用域中注册了一个变量。
+ **初始化阶段**， 这一阶段分配了内存并在作用域中让内存与变量建立了一个绑定，变量会被自动初始化为 `undefined` 。
+ **赋值阶段**，这一阶段为已初始化的变量分配具体的一个值。


一个变量在通过**声明阶段**后，它还是处于**未初始化**的状态，因为此时它仍为进入到**初始化阶段**。

![网络不给力](https://jiahonzheng-blog.oss-cn-shenzhen.aliyuncs.com/JavaScript%20%E5%8F%98%E9%87%8F%E6%8F%90%E5%8D%87%20-%201.jpg)

注意，按照变量的生命周期过程，**声明阶段**与我们通常所说的**变量声明**是不同的术语。简单来讲，引擎处理变量声明需要经过完善的这 3 个阶段：声明阶段、初始化阶段和赋值阶段。

## `var` 变量的生命周期

稍微熟悉下变量的三大生命周期阶段，现在让我们用它们来描述引擎是如何处理 `var` 变量的。

![网络不给力](https://jiahonzheng-blog.oss-cn-shenzhen.aliyuncs.com/JavaScript%20%E5%8F%98%E9%87%8F%E6%8F%90%E5%8D%87%20-%202.jpg)

假设一个场景，当 JavaScript 遇到了一个函数作用域，其中包含了 `var variable` 的语句，则在任何语句执行之前，这个变量就已经通过了**声明阶段**和**初始化阶段**（对于 `var`  来说，该两阶段不存在任何间隙）。

同时，`var variable` 在函数作用域中的位置并不会影响它的声明和初始化阶段的**优先进行**。

在声明和初始化阶段后，赋值阶段之前，变量的值为 `undefined` ，且已经可以被使用了。

在赋值阶段 `varibale = 'some value'` ，赋值语句使得变量得到新的赋值。

对于 `var` ，**变量提升**指 `var` 变量的**声明阶段**和**初始化阶段**得到提升，并且这两阶段之间没有任何的间隙。

下面是一个可供参考的范例：

```javascript
function multiplyByTen(number) {
    console.log(ten); // undefined
    var ten;
    ten = 10; // 赋值阶段
    console.log(ten); // 10
    return number * 10;
}
multiplyByTen(4); // 40
```

当 JavaScript 开始执行 `multiplyByTen(4)` 时进入到函数作用域中，变量 `ten` 在第一个语句之前就完成了声明和初始化阶段，并且值为 `undefined` ，故在调用 `console.log(ten)` 时打印为 `undefined` 。

语句 `ten = 10` 为变量赋于 `10` ，故在赋值后，`console.log(ten)` 打印了正确的 `10` 值。

## 函数的生命周期

![网络不给力](https://jiahonzheng-blog.oss-cn-shenzhen.aliyuncs.com/JavaScript%20%E5%8F%98%E9%87%8F%E6%8F%90%E5%8D%87%20-%203.jpg)

对于 `function` ，**声明、初始化和赋值阶段**在封闭的函数作用域的开头，便立即执行，其提升优先级比 `var` 和 `let` 提升优先级高。 `funName()` 可以在作用域中的任意位置被调用，与其声明语句所在位置无关。

值得注意的是，函数声明会被提升，但是函数表达式不会被提升。

```javascript
foo(); // 3 
// 这里不出现 TypeError 的原因是：
// 重复的声明会被忽略，所以 var 提升时，不会把已有的 foo 初始化为 undefined

bar(); // ReferenceError
// 出现 Reference 的原因是：
// 具名函数表达式的函数名只在函数内部有效

function foo() {
    console.log(1);
}

var foo = function bar() {
    console.log(2);
}

function foo() {
    console.log(3);
}

foo(); // 2
```



## `let` 变量的生命周期

**`let` 的提升是只针对声明阶段的提升**。

![网络不给力](https://jiahonzheng-blog.oss-cn-shenzhen.aliyuncs.com/JavaScript%20%E5%8F%98%E9%87%8F%E6%8F%90%E5%8D%87%20-%204.jpg)

现在让我们研究这样一个场景，当解释器进入了一个包含 `let variable` 语句的块级作用域中，这个变量立即通过了**声明阶段**，并在作用域内注册了它的名称。

然后解释器继续解析块语句。

如果这时尝试访问 `variabl` ，JavaScript 将会抛出 `ReferenceError: variable is not defined` ，因为这个变量的状态依然是未初始化的。

在声明阶段结束到初始化阶段开始， `variable` 处于**临时死区**。

当解释器到达语句 `let variable` 时，此时变量通过了**初始化阶段**，现在变量状态为已初始化的，并且具有 `undefined` 的值，同时变量也离开了临时死区。

之后当到达赋值语句 `variable = 'some value'` 时，变量通过了**赋值阶段**。

如果 JavaScript 遇到了 `let variable = 'some value'` ，那么变量会在这一个条语句中完成初始化和赋值阶段。

## 结论

至此，我们知道变量提升分为三种：

+ `var` 只有**声明阶段**和**初始化阶段**被提升。
+ `function` 的**声明阶段**、**初始化阶段**和**赋值阶段**都被提升。
+ `let` 只有**声明阶段**被提升。

## 参考资料

+ [JavaScript variables lifecycle: why let is not hoisted](https://dmitripavlutin.com/variables-lifecycle-and-why-let-is-not-hoisted/)
+ [我用了两个月的时间才理解 let](https://zhuanlan.zhihu.com/p/28140450?utm_source=wechat_session&utm_medium=social#showWechatShareTip)