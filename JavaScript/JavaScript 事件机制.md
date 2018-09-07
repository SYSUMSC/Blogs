# JavaScript 事件机制

通俗地来说， JavaScript 事件机制描述的是事件在 DOM 里面的传递顺序，以及我们可以对这些事件做出如何的响应。

假设我们具有一个 `ul` 元素，其包括很多 `li` 元素。当我们点击任何一个 `li` 时，其实我们也点击了 `ul` ，因为 `ul` 把所有的 `li` 元素给“包装”了。

## 简单范例

在接下来的博文中，我们通过以下范例对事件机制进行介绍。

```html
<!DOCTYPE html>
<html>
<body>
  <ul id="list">
    <li id="list_item">
      <a id="list_item_link" target="_blank" href="https://google.com">
        Google
      </a>
    </li>
  </ul>
</body>
</html>
```

## 三个阶段

JavaScript 事件触发有三个阶段。

+ `CAPTURING_PHASE`，即捕获阶段
+ `AT_TARGET`，即目标阶段
+ `BUBBLING_PHASE`，即冒泡阶段

我们可通过事件对象的 `eventPhase` 属性，得知事件处于哪个阶段。`eventPhase` 为一个正整数，其定义可在 [Event interface](https://www.w3.org/TR/DOM-Level-2-Events/events.html#Events-interface) 查阅到。

```c
const unsigned short CAPTURING_PHASE = 1;
const unsigned short AT_TARGET       = 2;
const unsigned short BUBBLING_PHASE  = 3;
```

DOM 的事件在传播时，会从根节点开始往下传递到 `target` ，若注册了事件监听器，则监听器处于捕获阶段。。

`target` 就是触发事件的具体对象，这时注册在 `target` 上的事件监听器处于目标阶段。

最后，事件再往上从 `target` 一路逆向传递到根节点，若注册了事件监听器，则监听器处于冒泡阶段。

## 注册事件

通常我们使用 `addEventListener` 注册事件，该函数有一个 `useCapture` 参数，该参数接收一个布尔值，默认值为 `false` ，代表注册事件为冒泡事件。若想注册事件为捕获事件，则将 `useCapture` 设置为 `true` 。

## target 和 currentTarget

在了解上述的事件传递的三个阶段后，我们来梳理事件对象中容易混淆的两个属性：`target` 和 `currentTarget` 。

+ `target` 是触发事件的某个具体的对象，只会出现在事件机制的目标阶段，即“谁触发了事件，谁就是 `target` ”。
+ `currentTarget` 是绑定事件的对象。

## 两个有用的结论

### 先捕获，再冒泡

```javascript
const get = (id) => document.getElementById(id);
const $list = get('list');
const $list_item = get('list_item');
const $list_item_link = get('list_item_link');
  
// list 的捕获
$list.addEventListener('click', (e) => {
  console.log('list capturing', e.eventPhase);
}, true);
  
// list 的冒泡
$list.addEventListener('click', (e) => {
  console.log('list bubbling', e.eventPhase);
}, false);
  
// list_item 的捕获
$list_item.addEventListener('click', (e) => {
  console.log('list_item capturing', e.eventPhase);
}, true);
  
// list_item 的冒泡
$list_item.addEventListener('click', (e) => {
  console.log('list_item bubbling', e.eventPhase);
}, false);
  
// list_item_link 的捕获
$list_item_link.addEventListener('click', (e) => {
  console.log('list_item_link capturing', e.eventPhase);
}, true);
  
// list_item_link 的冒泡
$list_item_link.addEventListener('click', (e) => {
  console.log('list_item_link bubbling', e.eventPhase);
}, false);
```

在我们点击超链接后，`console` 输出以下结果：

```
list capturing
1
list_item capturing
1
list_item_link capturing
2
list_item_link bubbling
2
list_item bubbling
3
list bubbling
3
```

从实验中，我们得出这样的一个结论：**先捕获，后冒泡** 。

### 在 target 注册的监听器，不分捕获和冒泡

既然我们得出了“先捕获，后冒泡”的结论，那么无论 `addEventListener` 的注册顺序如何改变，最终效果应该是一样的。理想很丰满，现实很骨感。

我们将前面的实验代码更改一下：

```javascript
const get = (id) => document.getElementById(id);
const $list = get('list');
const $list_item = get('list_item');
const $list_item_link = get('list_item_link');
  
// list 的捕获
$list.addEventListener('click', (e) => {
  console.log('list capturing', e.eventPhase);
}, true);
  
// list 的冒泡
$list.addEventListener('click', (e) => {
  console.log('list bubbling', e.eventPhase);
}, false);
  
// list_item 的捕获
$list_item.addEventListener('click', (e) => {
  console.log('list_item capturing', e.eventPhase);
}, true);
  
// list_item 的冒泡
$list_item.addEventListener('click', (e) => {
  console.log('list_item bubbling', e.eventPhase);
}, false);
  
// list_item_link 的冒泡
$list_item_link.addEventListener('click', (e) => {
  console.log('list_item_link bubbling', e.eventPhase);
}, false);
  
// list_item_link 的捕获
$list_item_link.addEventListener('click', (e) => {
  console.log('list_item_link capturing', e.eventPhase);
}, true);
```

再次点击超链接，`console` 输出以下结果：

```
list capturing
1
list_item capturing
1
list_item_link bubbling
2
list_item_link capturing
2
list_item bubbling
3
list bubbling
3
```

可以发现，`list_item_link` 先执行了在冒泡阶段的 `listener` ，随后才执行捕获阶段的 `listener` 。

原因是当事件传递到 `target` 时，不管 `addEventListner` 的 `useCapture` 是 `true` 和 `false` ，`e.eventPhase` 均为 `AT_TARGET` ，自然就没有捕获与冒泡的区别，故执行顺序取决于 `addEventListen` 的注册顺序。

由上面的实验，我们得出第二个结论：**在 target 注册的监听器，不分捕获和冒泡** 。

## 取消事件传递

我们可以通过 `e.stopPropagation` 中断事件的向下或向上传递。

在前面的实验代码中，我们给 `list` 的捕获阶段监听器添加中断事件传播的方法。

```javascript
// list 的捕获
$list.addEventListener('click', (e) => {
  console.log('list capturing');
  e.stopPropagation();
}, true);
```

则在点击超链接后，会输出以下结果：

```
list capturing
```

可见，事件传播被中断了，剩下的 `listener` 不能接收到事件。

不过，需要注意：**`stopPropagation` 不能阻止同一节点的其他 `listener` 的执行** 。

比如说：

```javascript
// list 的捕获
$list.addEventListener('click', (e) => {
  console.log('list capturing');
  e.stopPropagation();
}, true);

// list 的捕获2
$list.addEventListener('click', (e) => {
  console.log('list capturing 2');
}, true);
```

则输出结果为：

```
list capturing
list capturing 2
```

若想让同一节点的其他 `listener` 不被执行，我们可以使用 `e.stopImmediatePropagation` 方法。

## 取消预设行为

我们可以使用 `e.preventDefault` 取消默认行为。

改写实验代码：

```javascript
// list_item_link 的冒泡
$list_item_link.addEventListener('click', (e) => {
  e.preventDefault();
}, false);
```

这样，当我们点击超链接时，就不会执行原本的默认行为（新开分页或跳转）。很多人会将 `e.stioPropagation` 和 `e.preventDefault` 混淆，事实上，`e.preventDefault` 与事件传递没有任何关系，并不会影响事件的向下或向上传播。

这里有个特别值得注意的地方，来自 W3C 。

> Once preventDefault has been called it will remain in effect throughout the remainder of the event’s propagation.

上面这句话的意思是，只要调用了 `preventDefault` 方法，在之后传递新的事件里面也会有效果。

我们来看一个范例：

```javascript
// list 的冒泡
$list.addEventListener('click', (e) => {
  console.log('list bubbling', e.eventPhase);
  e.preventDefault();
}, true);
```

结果是超链接的默认行为没有被执行，注意到：**不管是在捕获阶段还是在冒泡阶段，只要使用了 `preventDefault` 方法，即可取消默认行为的执行** 。

## 事件代理

当我们想在 `ul` 节点添加 1000 个 `li` ，若在每个 `li` 添加 `eventListener` ，则新建了 1000 个 `function` 。但通过事件传播机制，我们可以在 `ul` 注册 `eventListener` 。

这样的好处有亮点：

+ 节省内存
+ 不需要给子节点注销事件

## 参考资料

+ [DOM 的事件傳遞機制：捕獲與冒泡](https://blog.techbridge.cc/2017/07/15/javascript-event-propagation/)