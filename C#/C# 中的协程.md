# Coroutine in C&#35;

## 简介
C# 中有个关键字 yield，其作用是在迭代器块中用于向枚举数对象提供值或发出迭代结束信号。  
形式可以为以下之一：
```csharp
yield return <expression>;
yield break
```

分别有什么作用呢？我肯可以看这个例子：
```csharp
using System;
using System.Collections;

namespace YieldTest
{
    public class MyCollection
    {
        public IEnumerable Test()
        {
            yield return "test1";
            yield return "test2";
            yield return "test3";
            yield return "test4";
            Console.WriteLine("hhhh1");
            yield return "test5";
            yield break;
            Console.WriteLine("hhhh2");
        }
    }

    public class YieldTest
    {
        public static void Main(string[] args)
        {
            var c = new MyCollection();
            foreach (var i in c.Test())
            {
                Console.WriteLine(i);
            }
        }
    }
}
```
运行后，会产生以下输出：
```
test1
test2
test3
test4
hhhh1
test5
```
上面的例子实现了一个可迭代（可以用 foreach）的数据集合，返回 IEnumerable。  
在遇到 `yield return` 时，会进行下面几件事情：
1. 记录下当前执行到的代码位置
2. 将代码控制权返回到外部，并将 `yield return` 后面的值作为迭代的当前值。
3. 执行下一个循环时从刚才记录的代码位置后面继续执行代码。
4. 如果遇到了 `yield break` 则终止本次迭代。

以上东西还可以用 `IEnumerator` 这么实现：
```csharp
using System;
using System.Collections;

namespace YieldTest
{
    public class MyCollection
    {
        public IEnumerator Test()
        {
            yield return "test1";
            yield return "test2";
            yield return "test3";
            yield return "test4";
            Console.WriteLine("hhhh1");
            yield return "test5";
            yield break;
            Console.WriteLine("hhhh2");
        }
    }

    public class YieldTest
    {
        public static void Main(string[] args)
        {
            var c = new MyCollection();
            var i = c.Test();
            while (i.MoveNext())
            {
                Console.WriteLine(i.Current);
            }
        }
    }
}
```

## 协程实现

有了上面的东西，我们就可以用这个来实现协程了：
```csharp
using System.Collections;
using System.Collections.Generic;

namespace Com.Coroutine
{
    public class Coroutine
    {
        internal IEnumerator m_Routine;
        internal IEnumerator Routine
        {
            get { return m_Routine; }
        }

        internal Coroutine()
        {
        }

        internal Coroutine(IEnumerator routine)
        {
            this.m_Routine = routine;
        }

        internal bool MoveNext()
        {
            var routine = m_Routine.Current as Coroutine;
            if (routine != null)
            {
                if (routine.MoveNext())
                {
                    return true;
                }
                else if (m_Routine.MoveNext())
                {
                    return true;
                }
                else
                {
                    return false;
                }
            }
            else if (m_Routine.MoveNext())
            {
                return true;
            }
            else
            {
                return false;
            }
        }
    }

    // use this as a template for functions like WaitForSeconds()
    public class WaitForCount : Coroutine
    {
        int count = 0;
        public WaitForCount(int count)
        {
            this.count = count;
            this.m_Routine = Count();
        }

        IEnumerator Count()
        {
            while (--count >= 0)
            {
                System.Console.WriteLine(count);
                yield return true;
            }
        }
    }

    // use this as the base class for enabled coroutines
    public class CoroutineManager
    {

        internal List<Coroutine> m_Coroutines = new List<Coroutine>();

        // just like Unity's MonoBehaviour.StartCoroutine
        public Coroutine StartCoroutine(IEnumerator routine)
        {
            var coroutine = new Coroutine(routine);
            m_Coroutines.Add(coroutine);
            return coroutine;
        }

        // call this every frame
        public void ProcessCoroutines()
        {
            for (int i = 0; i < m_Coroutines.Count; )
            {
                var coroutine = m_Coroutines[i];
                if (coroutine.MoveNext())
                {
                    ++i;
                }
                else if (m_Coroutines.Count > 1)
                {
                    m_Coroutines[i] = m_Coroutines[m_Coroutines.Count - 1];
                    m_Coroutines.RemoveAt(m_Coroutines.Count - 1);
                }
                else
                {
                    m_Coroutines.Clear();
                    break;
                }
            }
        }
    }
}
```