# 使用 LINQ 对集合进行快速处理

## 背景
以往我们对一个集合进行操作时，尤其是查询、筛选和统计等，往往要写大量的循环语句。而在 C# 中，我们可以使用 LINQ 大大简化这一点。

## LINQ
LINQ 全称为 .NET Language Integrated Query，专用于对集合进行各种查询操作，包含于 System.Linq，并作为 IEnumerable 接口的扩展方法出现。

## 适用场景
LINQ 可用于任何继承自 IEnumerable 接口的对象，如 List、Array 等等

## 实例类型定义
```csharp
class Data1
{
    public A { get; set; }
    public B { get; set; }
    public C { get; set; }
}

class Date2
{
    public A { get; set; }
    public X { get; set; }
}

List<Data1> list = new List<Data1>();
List<Date2> reflist = new List<Date2>();
```

## 数据获取和选择
```csharp
from i in list select new { A = i.A, B = i.B };
list.Select(i => new { A = i.A, B = i.B });
```
以上两句作用相同，都是从 list 中找出所有元素，并生成一个新的 IEnumerable<T>，我们称为 list'，list' 中只包含 A 和 B 两个成员，称我们选择 A 和 B 这两个成员。

## 数据筛选
```csharp
from i in list where <condition> select <object>;
list.Where(i => <condition>).Select(i => <object>);
```
使用 where 对列表进行筛选，使得结果中仅包含满足特定条件的元素。

## 数据排序
```csharp
from i in list where <condition> orderby <key> ascending select <object>;
list.Where(i => <condition>).OrderBy(i => <key>).Select(i => <object>);

from i in list where <condition> orderby <key> descending select <object>;
list.Where(i => <condition>).OrderByDescending(i => <key>).Select(i => <object>);
```
使用 orderby 使得集合中按照 key 进行排序。

## 数据分组
```csharp
from i in list group i by <key>;
list.GroupBy(i => <key>);
```
使用 groupby 将集合中的所有元素按照 key 进行分组，生成一个 IEnumerable\<IGrouping\<T1, T2>>。
如果必须引用某个组的结果，可使用 into 关键字创建能被进一步查询的标识符，然后就可以使用该标识符进行进一步的操作。
```csharp
from i in list group i by <key> into <identifier> ...;
```

## 数据联接
```csharp
from i in list 
join j in reflist on <condition>
...
```
在 LINQ 中，不必像在 SQL 中那样频繁使用 join，因为 LINQ 中的外键在对象模型中表示为包含项集合的属性。

## 其他
除了以上的基本操作之外，我们还可以利用 LINQ 进行数据的拼接、转换、运算和统计等等。  
例如，可以使用 Max() 方法求最大值，使用 Cast<T>() 方法进行数据类型转换，使用 Sum() 方法求和，以及使用 Concat() 方法进行集合的拼接等等。

## 查询语法和方法组语法
你可能已经发现了上述所有例子中都写了两种语法。其中第一种成为查询语法，使用一种类 SQL 语言 -- LINQ 语言对数据进行操作；另外一种成为方法组语法，来自于 IEnumerable 的扩展方法。  
LINQ 的本质便是 IEnumerable 的扩展方法，而它的查询语法可以认为是一种语法糖，在编译期会被翻译成方法组语法进行调用。