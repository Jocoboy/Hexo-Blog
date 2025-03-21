---
title: C#进阶语法
date: 2025-03-20 14:57:10
categories:
- Language
tags:
- c#
---

c#进阶语法，包含委托与事件、动态类型与泛型、信号量等。

<!--more-->

## 前言

c#中的一些进阶语法使用记录，包含委托(Delegate)与事件(Event)、动态类型(Dynamic Types)与泛型(Generics)、信号量(Semaphore/SemaphoreSlim)等。 

## 委托与事件

### 委托(Delegate)

在C#中，委托(Delegate)是一种类型安全的函数指针，用于封装一个或多个方法。委托允许你将方法作为参数传递、存储方法引用，并支持事件和回调机制。委托是C#中实现事件驱动编程和多播委托的基础。

下面是一个委托定义、实例化和调用的示例:

```c#
public class Program
{
    // 定义委托
    public delegate int MathOperation(int a, int b);

    // 定义符合委托签名的方法
    public static int Add(int a, int b) => a + b;
    public static int Subtract(int a, int b) => a - b;

    public static void Main()
    {
        // 实例化委托
        MathOperation operation = Add;

        // 调用委托
        int result = operation(10, 5);
        Console.WriteLine($"Addition Result: {result}"); // 输出: Addition Result: 15

        // 重新指向另一个方法
        operation = Subtract;
        result = operation(10, 5);
        Console.WriteLine($"Subtraction Result: {result}"); // 输出: Subtraction Result: 5
    }
}
```

C#支持匿名方法和Lambda表达式，简化了委托的使用，上面的委托实例化可以简写为：

```c#
MathOperation operation = delegate(int a, int b) { return a + b; }; // 匿名方法
MathOperation operation = (a, b) => a + b; // Lambda表达式
```

委托可以指向多个方法，称为多播委托。通过+=和-=运算符，可以添加或移除方法。

```c#
public class Program
{
    // 定义委托
    public delegate void DisplayMessage(string message);

    // 定义符合委托签名的方法
    public static void ShowMessage(string message) => Console.WriteLine($"Message: {message}");
    public static void ShowWarning(string message) => Console.WriteLine($"Warning: {message}");

    public static void Main()
    {
        // 实例化多播委托
        DisplayMessage display = ShowMessage;
        display += ShowWarning; // 添加第二个方法

        // 调用多播委托（会依次调用所有方法）
        display("Hello, World!");
    }
}
```

C#提供了几种内置的泛型委托类型，避免了手动定义委托。

- Action: 表示没有返回值的方法，最多支持16个参数。示例：

    ```c#
    Action<string> print = message => Console.WriteLine(message);
    print("Hello, Action!");
    ```
- Func: 表示有返回值的方法，最后一个泛型参数是返回值类型。示例：

    ```c#
    Func<int, int, int> add = (a, b) => a + b;
    int result = add(10, 5); 
    ```

- Predicate: 表示返回布尔值的方法，通常用于条件判断。

    ```c#
    Predicate<int> isEven = num => num % 2 == 0;
    bool result = isEven(10); 
    ```

委托的应用场景有很多，例如事件处理、回调机制(将方法作为参数传递给其他方法，用于异步编程或延迟执行)、LINQ查询等。

### 事件(Event)

委托是C#事件的基础。在C#中，事件(Event)是一种特殊的委托，用于实现发布-订阅模式(观察者模式)。事件允许一个类(发布者)通知其他类(订阅者)某些事情发生了。事件的核心是委托，它定义了事件处理方法的签名。

下面是一个事件的定义、触发、订阅和取消的示例：

```c#
public class Publisher
{
    // 定义委托
    public delegate void EventHandler(string message);

    // 定义事件
    public event EventHandler OnMessageReceived;

    // 触发事件的方法
    public void SendMessage(string message)
    {
        Console.WriteLine($"Publisher: Sending message - {message}");
        OnMessageReceived?.Invoke(message); // 触发事件
    }
}

public class Subscriber
{
    // 事件处理方法
    public void HandleMessage(string message)
    {
        Console.WriteLine($"Subscriber: Received message - {message}");
    }
}

public class Program
{
    public static void Main()
    {
        // 创建发布者和订阅者
        Publisher publisher = new Publisher();
        Subscriber subscriber = new Subscriber();

        // 订阅事件
        publisher.OnMessageReceived += subscriber.HandleMessage;

        // 发布者发送消息，触发事件
        publisher.SendMessage("Hello, World!");

        // 取消订阅事件
        publisher.OnMessageReceived -= subscriber.HandleMessage;

        // 再次发送消息，不会触发事件
        publisher.SendMessage("This message will not be received.");
    }
}
```

C#提供了标准的EventHandler委托和EventArgs类，推荐使用它们来定义事件，下面是一个使用示例(调用方式同上)：

```c#
using System;

public class Publisher
{
    // 定义事件
    public event EventHandler<CustomEventArgs> OnMessageReceived;

    // 触发事件的方法
    public void SendMessage(string message)
    {
        Console.WriteLine($"Publisher: Sending message - {message}");
        OnMessageReceived?.Invoke(this, new CustomEventArgs { Message = message });
    }
}

public class CustomEventArgs : EventArgs
{
    public string Message { get; set; }
}

public class Subscriber
{
    // 事件处理方法
    public void HandleMessage(object sender, CustomEventArgs e)
    {
        Console.WriteLine($"Subscriber: Received message - {e.Message}");
    }
}
```

事件的使用场景有很多，例如用户界面事件、异步编程事件(如文件下载完成事件)、自定义通知机制(如状态更新)等。

## 动态类型与泛型

### 动态类型(Dynamic Types)

dynamic是c#中的一个动态类型，属性的访问是在运行时解析的，因此需要确保运行时对象具有你访问的属性。

可以通过下面的方式创建一个dynamic对象：

```c#
dynamic item = new { Name = "Alice", Age = 25 };
```

要对`List<dynamic>`使用Where语句，可通过反射检查属性是否存在，确保dynamic对象具有需要访问的属性：

```c#
List<dynamic> dynamicList = new List<dynamic>
{
    new { Name = "Alice", Age = 25 },
    new { Name = "Bob", Age = 30 },
    new { Name = "Charlie", Age = 20 }
};

dynamicList = dynamicList.Where(item =>
{
    var property = item.GetType().GetProperty("Name");
    return property != null && (string)property.GetValue(item) != "Charlie";
}).ToList();
```

dynamic是提供用于在运行时指定动态行为的基类。必须继承此类；不能直接对其进行实例化。通常用到的是它的继承类ExpandoObject。

`System.Dynamic.ExpandoObject`是一个动态对象，允许在运行时添加或删除属性。它可以像字典一样操作(实现了 `IDictionary<string, object>`)，也可以像普通对象一样访问属性。

可以通过下面的方式创建一个ExpandoObject对象：

```c#
dynamic item = new ExpandoObject();
item.Name = "Alice";
item.Age = 25;
Console.WriteLine(((IDictionary<string, object>)item)["Name"]);
```

要对`List<ExpandoObject>`使用Where语句，通常需要将其视为`IDictionary<string, object>`，然后根据键值对进行过滤。

```c#
List<ExpandoObject> expandoList = new List<ExpandoObject>();

dynamic item1 = new ExpandoObject();
item1.Name = "Alice";
item1.Age = 25;
expandoList.Add(item1);

dynamic item2 = new ExpandoObject();
item2.Name = "Bob";
item2.Age = 30;
expandoList.Add(item2);

dynamic item3 = new ExpandoObject();
item3.Name = "Charlie";
item3.Age = 20;
expandoList.Add(item3);

expandoList = expandoList.Where(item =>
                            ((IDictionary<string, object>)item).ContainsKey("Name") &&
                            (string)((IDictionary<string, object>)item)["Name"] != "Charlie"
                        ).ToList();
```

### 泛型(Generics)

在C#中，泛型允许你编写可以处理多种类型的代码，而无需为每种类型重复编写逻辑。泛型在编译时确定类型，并提供类型安全和性能优势。

泛型(Generics)和动态类型(Dynamic Types)是两种不同的特性，用于处理类型不确定的场景。它们的主要区别在于类型检查的时机、安全性、性能和使用场景等。以下是他们的主要区别：

| 区别 | 泛型(Generics) | 动态类型(Dynamic Types) | 
| ----------- | ----------- | ----------- |
| 类型检查时机 | 编译时 | 运行时 | 
| 类型安全 | 是 | 否 | 
| 性能 | 高性能(避免装箱和拆箱) | 较低(运行时解析类型) | 
| 灵活性 | 较低(类型在编译时确定) | 高(类型在运行时确定) | 
| 适用场景 | 类型安全的通用代码、集合类、高性能场景 | 动态语言交互、未知类型处理、灵活性需求 | 

## 信号量Semaphore与SemaphoreSlim

Semaphore和SemaphoreSlim是.NET中用于控制并发访问的同步原语，他们的主要区别如下：

| 区别 | Semaphore | SemaphoreSlim | 
| ----------- | ----------- | ----------- |
| 跨进程支持 | 支持跨进程同步，可以用于不同进程之间的线程同步 | 仅支持单进程内的线程同步 | 
| 性能 | 性能相对较低，因为它依赖于操作系统内核对象 | 性能更高，因为它完全基于.NET运行时 | 
| 异步支持 | 不支持异步操作 | 支持异步操作 |
| 适用场景 | 适用于需要跨进程同步的场景 | 适用于单进程内需要高性能同步的场景 |

### Semaphore

Semaphore用于限制可同时访问某一资源或资源池的线程数。下面是一个简单的使用示例：

```c#
// 创建一个信号量，初始计数为0，最大计数为3
Semaphore semaphore = new Semaphore(0, 3);
semaphore.WaitOne();
// 临界区代码
semaphore.Release();
```

- Semaphore第一个参数是初始计数initialCount，代表未执行Release方法之前允许通过的信号量
- Semaphore第二个参数是最大计数maximumCount，代表执行Release方法之后允许通过的最大信号量
- 释放之前的数量加上释放数量releaseCount应当不超过最大计数，即Release(releaseCount) + releaseCount <= maximumCount，其中Release(releaseCount)初始值为initialCount

以下示例展示了信号量Semaphore是如何控制并发访问的：

```c#
public class Program
{
    private static Semaphore _semaphore;
    private static int _padding;

    private static void Worker(object num)
    {
        Console.WriteLine("线程 {0} 开始并等待进入信号量.", num);
        _semaphore.WaitOne();

        int padding = Interlocked.Add(ref _padding, 100);

        Console.WriteLine("线程 {0} 进入信号量.", num);

        Thread.Sleep(1000 + padding);

        Console.WriteLine("线程 {0} 从信号量中释放.", num);

        try
        {
            Console.WriteLine("线程 {0} 释放之前信号量中的可用资源数: {1}",
            num, _semaphore.Release());
        }
        catch (SemaphoreFullException)
        {
            Console.WriteLine("警告: 释放数量超过信号量最大计数！");
        }
    }

    static void Main(string[] args)
    {
        _semaphore = new Semaphore(initialCount: 1, maximumCount: 4);

        for (int i = 1; i <= 5; i++)
        {
            Thread t = new Thread(new ParameterizedThreadStart(Worker));

            t.Start(i);
        }

        Thread.Sleep(500);

        int releaseCount = 4;
        Console.WriteLine($"主线程尝试释放{releaseCount}个资源");

        try
        {
            _semaphore.Release(releaseCount);
        }
        catch (SemaphoreFullException)
        {
            Console.WriteLine("警告: 释放数量超过信号量最大计数！");
        }

        Console.WriteLine("主线程退出");
    }
}
```

控制台输出内容如下：

> 线程 2 开始并等待进入信号量.
线程 5 开始并等待进入信号量.
线程 3 开始并等待进入信号量.
线程 1 开始并等待进入信号量.
线程 4 开始并等待进入信号量.
线程 2 进入信号量.
主线程尝试释放4个资源
主线程退出
线程 3 进入信号量.
线程 5 进入信号量.
线程 4 进入信号量.
线程 1 进入信号量.
线程 2 从信号量中释放.
线程 2 释放之前信号量中的可用资源数: 0
线程 5 从信号量中释放.
线程 5 释放之前信号量中的可用资源数: 1
线程 1 从信号量中释放.
线程 1 释放之前信号量中的可用资源数: 2
线程 3 从信号量中释放.
线程 3 释放之前信号量中的可用资源数: 3
线程 4 从信号量中释放.
警告: 释放数量超过信号量最大计数！

### SemaphoreSlim

SemaphoreSlim是对可同时访问资源或资源池的线程数加以限制的Semaphore的轻量替代。下面是一个简单的使用示例：

```c#
// 创建一个信号量，初始计数为0，最大计数为3
SemaphoreSlim semaphoreSlim = new SemaphoreSlim(0, 3);
await semaphoreSlim.WaitAsync();
// 临界区代码
semaphoreSlim.Release();
```

以下示例展示了信号量SemaphoreSlim是如何控制并发访问的：

```c#
public class Program
{
    private static SemaphoreSlim _semaphoreSlim;
    private static int _padding;

    private static void Worker()
    {
        Console.WriteLine("线程 {0} 开始并等待进入信号量.", Task.CurrentId);
        _semaphoreSlim.Wait();

        int semaphoreCount;

        Interlocked.Add(ref _padding, 100);

        Console.WriteLine("线程 {0} 进入信号量.", Task.CurrentId);

        Thread.Sleep(1000 + _padding);

        try
        {
            Console.WriteLine("线程 {0} 从信号量中释放.", Task.CurrentId);
            semaphoreCount = _semaphoreSlim.Release();
            Console.WriteLine("线程 {0} 释放之前信号量中的可用资源数: {1}", Task.CurrentId, semaphoreCount);
        }
        catch (SemaphoreFullException)
        {
            Console.WriteLine("警告: 释放数量超过信号量最大计数！");
        }
    }

    static void Main(string[] args)
    {
        _semaphoreSlim = new SemaphoreSlim(1, 4);
        Task[] tasks = new Task[5];

        for (int i = 0; i <= 4; i++)
        {
            tasks[i] = Task.Run(Worker);
        }

        Thread.Sleep(500);

        int releaseCount = 4;
        Console.WriteLine($"主线程尝试释放{releaseCount}个资源");
        try
        {
            _semaphoreSlim.Release(releaseCount);
        }
        catch (SemaphoreFullException)
        {
            Console.WriteLine("警告: 释放数量超过信号量最大计数！");
        }

        Task.WaitAll(tasks);

        Console.WriteLine("主线程退出");
    }
}
```

控制台输出内容与Semaphore类似，此处不再展示。

SemaphoreSlim还可用于异步锁，避免线程阻塞，下面是一个简单示例：

```c#
private readonly SemaphoreSlim _semaphoreSlim = new SemaphoreSlim(1, 1);  // 使用SemaphoreSlim实现异步锁，初始化时设置最大并发数为 1

public async Task DoWork()
{
    try
    {
        await _semaphoreSlim.WaitAsync(); // 等待锁
        // TODO
    }
    finally
    {
        _semaphoreSlim.Release(); // 释放锁
    }
}
```

## 参考文档

- [C#编程指南](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide)

- [.NET API参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/)

- [System.Dynamic.ExpandoObject类使用说明](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/runtime-libraries/system-dynamic-expandoobject)
