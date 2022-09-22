---
layout:     post
title:      "SynchronizationContext(同步上下文)"
subtitle:   "SynchronizationContext"
date:   2022-09-21
author: "qijun"
tags:
- DotnetCore
---

### 问题产生
当异步操作需要将结果传递给指定线程执行，需要一个中介来实现。  
列如UI程序需要ui线程执行异步结果显示；单线程模型服务器需要由主线程执行异步结果操作逻辑。  

### 基础概念
工作单元(Task)应列入上下文(Context)，而不是某个特定线程(Thread);  
微软为开发者提供了SynchronizationContext,在线程与线程通讯中充当传输者的角色;  
不是每个线程都附加SynchronizationContext这个对象,对于WinForm程序，WindowsFormsSynchronizationContext将所有异步操作回归UI线程执行;  
每一个运行的Task都有相应的TaskScheduler来为其进行调度，而SynchronizationContext是作为TaskScheduler的一个一般抽象，它的出现主要是针对具有UI线程的应用。所以，在不同的应用中，都有各自的SynchronizationContext的实现，比如WinForm、WPF等。    
当await一个异步操作时，就会捕获这个SynchronizationContext或者TaskScheduler，具体逻辑如下面代码所示：如果当前SynchronizationContext不为空则就是它，否则，查看TaskScheduler是否符合条件。  
```c#
object scheduler = SynchronizationContext.Current;
if (scheduler is null && TaskScheduler.Current != TaskScheduler.Default)
{
    scheduler = TaskScheduler.Current;
}
```

### Default SynchronizationContext
```c#
// Decompiled with JetBrains decompiler
// Type: System.Threading.SynchronizationContext
// Assembly: System.Private.CoreLib, Version=6.0.0.0, Culture=neutral, PublicKeyToken=7cec85d7bea7798e
// MVID: 52CD8444-76EC-4091-8389-09409447BE01
// Assembly location: C:\Program Files\dotnet\shared\Microsoft.NETCore.App\6.0.8\System.Private.CoreLib.dll


#nullable enable
namespace System.Threading
{
  public class SynchronizationContext
  {
    private bool _requireWaitNotification;


    #nullable disable
    private static int InvokeWaitMethodHelper(
      SynchronizationContext syncContext,
      IntPtr[] waitHandles,
      bool waitAll,
      int millisecondsTimeout)
    {
      return syncContext.Wait(waitHandles, waitAll, millisecondsTimeout);
    }


    #nullable enable
    public static SynchronizationContext? Current => Thread.CurrentThread._synchronizationContext;

    protected void SetWaitNotificationRequired() => this._requireWaitNotification = true;

    public bool IsWaitNotificationRequired() => this._requireWaitNotification;

    public virtual void Send(SendOrPostCallback d, object? state) => d(state);

    public virtual void Post(SendOrPostCallback d, object? state) => ThreadPool.QueueUserWorkItem<(SendOrPostCallback, object)>((Action<(SendOrPostCallback, object)>) (s => s.d(s.state)), (d, state), false);

    public virtual void OperationStarted()
    {
    }

    public virtual void OperationCompleted()
    {
    }

    [CLSCompliant(false)]
    public virtual int Wait(IntPtr[] waitHandles, bool waitAll, int millisecondsTimeout) => SynchronizationContext.WaitHelper(waitHandles, waitAll, millisecondsTimeout);

    [CLSCompliant(false)]
    protected static int WaitHelper(IntPtr[] waitHandles, bool waitAll, int millisecondsTimeout)
    {
      if (waitHandles == null)
        throw new ArgumentNullException(nameof (waitHandles));
      return WaitHandle.WaitMultipleIgnoringSyncContext((Span<IntPtr>) waitHandles, waitAll, millisecondsTimeout);
    }

    public static void SetSynchronizationContext(SynchronizationContext? syncContext) => Thread.CurrentThread._synchronizationContext = syncContext;

    public virtual SynchronizationContext CreateCopy() => new SynchronizationContext();
  }
}

```

注意到Post方法，默认的是调用ThreadPool

### 自定义同步上下文
继承SynchronizationContext，实现异步方法回归主线程执行：
```c#
using System;
using System.Collections.Concurrent;
using System.Threading;

namespace ET
{
    public class ThreadSynchronizationContext : SynchronizationContext
    {
        // 线程同步队列,发送接收socket回调都放到该队列,由poll线程统一执行
        private readonly ConcurrentQueue<Action> queue = new ConcurrentQueue<Action>();

        private Action a;

        public void Update()
        {
            while (true)
            {
                if (!this.queue.TryDequeue(out a))
                {
                    return;
                }

                try
                {
                    a();
                }
                catch (Exception e)
                {
                    Log.Error(e);
                }
            }
        }

        public override void Post(SendOrPostCallback callback, object state)
        {
            this.Post(() => callback(state));
        }
		
        public void Post(Action action)
        {
            this.queue.Enqueue(action);
        }
    }
}
```

通过重写Post方法，将异步执行完毕调用封装成委托，存入队列，由主线程调用

```c#
private readonly ThreadSynchronizationContext threadSynchronizationContext = new ThreadSynchronizationContext();
```
创建自己的上下文，并设置到SynchronizationContext
```c#
SynchronizationContext.SetSynchronizationContext(this.threadSynchronizationContext);
```
这样所有await async执行完异步方法后，通过传递的上下文threadSynchronizationContext，向队列中传入回调，由主线程执行

###关于上下文在异步方法中的传递
异步调用遇到await关键字时，自动将当前线程的上下文传递给异步方法。但你可以使用ConfigureAwait来改变这一行为
```c#
using System;
using System.Collections.Concurrent;
using System.IO;
using System.Threading;
using System.Threading.Tasks;

namespace Test2
{
    public class MySynchronizationContext : SynchronizationContext
    {
        // 线程同步队列,发送接收socket回调都放到该队列,由poll线程统一执行
        private readonly ConcurrentQueue<Action> queue = new ConcurrentQueue<Action>();

        private Action a;

        public void Update()
        {
            while (true)
            {
                if (!this.queue.TryDequeue(out a))
                {
                    return;
                }

                try
                {
                    a();
                }
                catch (Exception e)
                {
                    Console.WriteLine(e);
                }
            }
        }

        public override void Post(SendOrPostCallback callback, object state)
        {
            this.Post(() => callback(state));
        }
		
        public void Post(Action action)
        {
            this.queue.Enqueue(action);
        }
    }

    class Program
    {
        private static int loopCount = 0;
        private static MySynchronizationContext ctx;
        
        static void Main(string[] args)
        {
            ctx = new MySynchronizationContext();
            SynchronizationContext.SetSynchronizationContext(ctx);
            Console.WriteLine($"主线程: {Thread.CurrentThread.ManagedThreadId}");

            Crontine();

            while (true)
            {
                ctx.Update();
                Thread.Sleep(1);
                
                ++loopCount;
                if (loopCount % 10000 == 0)
                {
                    Console.WriteLine($"loop count: {loopCount}");
                }
            }
        }
        

        private static async void Crontine()
        {
            await WaitTimeAsync(0);
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
            await WaitTimeAsync(0);
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
            await WaitTimeAsync(0);
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
            await WaitTimeAsync(0).ConfigureAwait(false);
            Console.WriteLine("ConfigureAwait(false)之后...");
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
            await WaitTimeAsync(0);
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
            await WaitTimeAsync(0);
            Console.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}, WaitTimeAsync finsih loopCount的值是: {loopCount}");
        }
        
        private static Task WaitTimeAsync(int waitTime)
        {
            TaskCompletionSource<bool> tcs = new TaskCompletionSource<bool>();
            Thread thread = new Thread(()=>WaitTime(waitTime, tcs));
            thread.Start();
            return tcs.Task;
        }
        
        /// <summary>
        /// 在另外的线程等待
        /// </summary>
        private static void WaitTime(int waitTime, TaskCompletionSource<bool> tcs)
        {
            Thread.Sleep(waitTime);
            // 将tcs扔回主线程执行
            //ctx.Post(o=>tcs.SetResult(true), null);
            tcs.SetResult(true);
        }
    }
}
```
结果：
```c#
主线程: 1
当前线程: 1, WaitTimeAsync finsih loopCount的值是: 0
当前线程: 1, WaitTimeAsync finsih loopCount的值是: 1
当前线程: 1, WaitTimeAsync finsih loopCount的值是: 2
ConfigureAwait(false)之后...
当前线程: 8, WaitTimeAsync finsih loopCount的值是: 2
当前线程: 10, WaitTimeAsync finsih loopCount的值是: 3
当前线程: 11, WaitTimeAsync finsih loopCount的值是: 4

```
默认情况下，当前同步上下文在await关键字处被捕获，await后面的执行代码会列入到该同步上下文中执行，即调用上下文的Post方法

对于异步方法
```c#
static async Task<int> fun()
{
    await AsyncFun();

    Run();
}
 
```
执行代码类似于
```c#
Task t = AsyncFun();
var currentContext = SynchronizationContext.Current;
 
if (null  == currentContext)
{
    t.ContinueWith((te) => { Run(); }, TaskScheduler.Current);
}
else
{
    t.ContinueWith((te) => { Run(); }, TaskScheduler.FromCurrentSynchronizationContext());
}
```
函数会先获取SynchronizationContext.Current对象，这个对象默认情况下，在控制台程序中为null，在GUI程序中不为null  
ConfigureAwait 提供了一种途径避免 SynchronizationContext 捕获
而TaskScheduler.Current默认使用线程池来执行异步回调


### 参考
[实现自己的异步方法](https://www.jianshu.com/p/d109af185469)  
[C#中的ExecutionContext 和 SynchronizationContext](https://zhuanlan.zhihu.com/p/378386442)  
[SynchronizationContext(同步上下文)综述](https://www.cnblogs.com/BigBrotherStone/p/12240731.html)



