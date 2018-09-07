# JavaScript 立即函数的易错点

在阅读 JavaScript 代码时，我们时常能够见到[立即函数](http://macr.ae/article/immediate-functions.html)的身影，一方面立即函数能够实现模块的隔离，另一方面它为私有成员变量的存在提供了可能。

一般来说，立即函数表现为以下的形式。

```javascript
// 常见形式
(function() {})()
(function() {}())
```

在一些工程化项目中，我们还可以看到下面这些“奇葩”的形式。

```javascript
// “奇葩”形式
(function() {})()
(function() {}())
void function() {} ()
!function() {}()
+function() {}()
-function() {}()
~function() {}()
```

但是，如果我们将立即函数写成如下的形式，则是**错误**的。

```javascript
// 错误形式
function () {}()
```

我们知道，在 JavaScript 中，函数是一等对象，可以通过 `new Function([arg1[, arg2[, ...argN]],] functionBody)` 来构造，但通常我们更多的是使用**函数表达式**或**函数声明式**语句。

```javascript
// 函数表达式
let fun = function() {}

// 函数声明式
function fun() {}
```

当我们使用 `function() {}()` 的形式时，解析器会根据前面的 `function() {}` 认为这是**函数声明式**语句。由于**函数声明式语句后面不能有括号**，故解析器抛出语法错误。

当我们使用 `(function() {})()` 的形式时，将函数体包围的括号会使得解析器认为这是**函数表达式**语句，故使用此形式，不会抛出语法错误。

至于其他较为“奇葩”的形式，也正是遵循这样的思路：通过符号的使用，使得解析器认为是**函数表达式**语句，从而实现立即函数。