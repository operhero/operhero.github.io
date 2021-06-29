---
layout:     post
title:      "读MoreEffectiveC#"
subtitle:   "" 
date:   2021-06-28
author: "qijun"
tags:
    - reading notes
---

### 泛型
C#早期使用Object来代表任意类型，运行时强制转换存在安全问题，类型判断影响效率。
```csharp
public interface IComparable
{
    int CompareTo(object other);
}

public class Customer : IComparable
{
    public string Name;
    public int CompareTo(object right)
    {
        if(!(right is Customer))
        {
            throw new ArgumentException("Argument not a customer", "right");
            Customer rightCustomer = (Customer)right;
            return Name.CompareTo(rightCustomer.Name);
        }
    }
}
```
使用泛型能够保证编译器类型安全，并提高应用程序的执行效率。

```csharp
/// 泛型类, Example<int> e = new Example<int>();
public class Example<T> where T : new() // 泛型约束：struct、class、new()、NameOfBaseClass\NameOfInterface
{
    private T t;
}

/// 泛型方法, ExampleFunc.DoSomething(1);
public static ExampleFunc
{
    public static T DoSomething(T t)
    {
        Console.Writeline(t.GetType());
    }
}
```

使用泛型方法比泛型类更简单，不用显示的指明类型参数。有两种情况必须使用泛型类：
1. 类本身需要存放类型参数对象作为其内部状态。（例如List<T>）
2. 类实现了泛型接口

需要在到类型参数为值类型还是引用类型的情况下,用default为对象赋初值
```csharp
///之所以会用到default关键字，
///是因为需要在不知道类型参数为值类型还是引用类型的情况下，为对象实例赋初值。
///考虑以下代码：
class TestDefault<T>
{
    public T foo()
    {
        T t = null; //???
        return t;
    }
}

///如果我们用int型来绑定泛型参数，那么T就是int型，
///那么注释的那一行就变成了 int t = null；显然这是无意义的。
///为了解决这一问题，引入了default关键字：
class TestDefault<T>
{
    public T foo()
    {
        return default(T);
    }
}
```

### C#中的多线程
使用线程池而不是创建线程。如使用ThreadPool.QueueUserWorkItem<br />
当需要检查完成情况、跟踪进度、暂停或取消任务时，使用BackgroundWorker

优先使用lock().需要在不同上下文中释放锁，或者需要给锁加个超时时间时，使用Monitor.Enter(lockTarget)/Monitor.TryEnter(lockTarget, 1000ms)和Monitor.Exit(lockTarget)<br />
未规避死锁问题，规定lock(this)或lock(someclass)都不如为类创建一个同步对象
```csharp
private object syncHandle = new object();

public void IncrementTotal()
{
    lock(syncHandle)
    {
        //代码省略
    }
}
```

如果程序不需要经常锁定，一种减少同步对象创建的写法：
```csharp
private object syncHandle;

private object GetSyncHandle()
{
    // CompareExchange首先比较syncHandle和null,若为null则创建一个新对象并指派给syncHandle
    System.Threading.Interlocked.CompareExchange(ref syncHandle, new object(), null);
    return syncHandle
}

public void IncrementTotal()
{
    lock(GetSyncHandle())
    {
        //代码省略
    }
}
```

System.Threading.Interlocked.Increment(ref v);原子自增<br />
System.Threading.Interlocked.Decrement(ref v);原子自减<br />
System.Threading.Interlocked.Exchange(ref T location, T value);原子替换并返回原始值<br />
System.Threading.Interlocked.CompareExchange(ref T location, T value, T comparand);原子比较，若相等则赋新值；不关相等与否都返回原始值<br />
ReaderWriteLockSlim(ReaderWriterLock的改进版);读写锁
