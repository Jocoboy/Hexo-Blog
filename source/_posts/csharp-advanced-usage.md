---
title: C#进阶语法
date: 2025-03-20 14:57:10
categories:
- Language
tags:
- c#
---

c#进阶语法，包含委托与事件、动态类型与泛型、并发访问控制等。

<!--more-->

## 前言

c#中的一些进阶语法使用记录，包含委托(Delegate)与事件(Event)、动态类型(Dynamic Types)与泛型(Generics)、并发访问控制(信号量、并发字典)等。 

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

## 并发访问控制

### 信号量Semaphore与SemaphoreSlim

Semaphore和SemaphoreSlim是.NET中用于控制并发访问的同步原语，他们的主要区别如下：

| 区别 | Semaphore | SemaphoreSlim | 
| ----------- | ----------- | ----------- |
| 跨进程支持 | 支持跨进程同步，可以用于不同进程之间的线程同步 | 仅支持单进程内的线程同步 | 
| 性能 | 性能相对较低，因为它依赖于操作系统内核对象 | 性能更高，因为它完全基于.NET运行时 | 
| 异步支持 | 不支持异步操作 | 支持异步操作 |
| 适用场景 | 适用于需要跨进程同步的场景 | 适用于单进程内需要高性能同步的场景 |

#### Semaphore

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

#### SemaphoreSlim

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

### 并发字典ConcurrentDictionary

在C#中，线程安全集合是一种特殊的数据结构，能够在多线程环境下安全地访问和修改集合元素。线程安全集合通常是通过加锁或者使用其他同步机制来实现线程安全。C#中提供了多种线程安全集合，包括ConcurrentDictionary、ConcurrentQueue、ConcurrentStack、ConcurrentBag等。

ConcurrentDictionary是C#中用于处理多线程并发访问的线程安全字典。它的设计目标是高效地支持多个线程同时读写数据，而无需显式加锁。它的核心特性使其在高并发场景下表现出色，以下是它的主要特性：

- 线程安全：允许多个线程同时读取和写入数据，而不会导致数据损坏或抛出异常。它通过内部锁和无锁技术(如CAS操作)来实现高效的并发访问
- 原子操作：提供了一系列原子操作方法(TryAdd、TryUpdate、TryRemove、AddOrUpdate、GetOrAdd等)，确保在多线程环境下的操作是线程安全的，避免了手动加锁的复杂性
- 高性能：针对高并发场景进行了优化，使用了分区锁(lock striping)技术，将字典分成多个段(segments)，每个段使用独立的锁。这样可以减少锁竞争，提高并发性能
- 无锁读取：大多数情况下读取操作是无锁的(lock-free)，这意味着多个线程可以同时读取数据而不会阻塞。这使得读取操作非常高效

#### 应用场景

ConcurrentDictionary的应用场景很多，例如缓存系统、共享数据存储、统计和聚合数据、任务调度和状态管理、并发日志记录、分布式锁或资源管理、实时数据处理等。

##### 统计和聚合数据

在并发任务中，ConcurrentDictionary可以用于统计或聚合数据，例如计数、求和等。

下面这个示例使用了常规字典Dictionary对字符串列表进行统计，由于Dictionary是线程不安全的，统计结果将会不可预知。

```c#
var dict = new Dictionary<string, int>();
string[] words = { "apple", "banana", "apple", "orange", "banana", "apple" };

// 模拟多线程环境
Parallel.ForEach(words, word =>
{
    if (dict.ContainsKey(word))
    {
        dict[word] += 1;
    }
    else
    {
        dict[word] = 1;
    }
});

foreach (var kvp in dict)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

上面的代码运行后有概率会出现以下错误提示：

>1.`Object reference not set to an instance of an object`.
>2.`Operations that change non-concurrent collections must have exclusive access. A concurrent update was performed on this collection and corrupted its state. The collection's state is no longer correct.`

更改非并发集合的操作必须具有独占访问权限。对非并发集合执行并发更新将会损坏其状态，集合的状态将不再正确。可以使用lock关键字手动添加互斥锁避免竞态条件，也可以直接使用ConcurrentDictionary。

```c#
var concurrentDict = new ConcurrentDictionary<string, int>();
string[] words = { "apple", "banana", "apple", "orange", "banana", "apple" };

// 模拟多线程环境
Parallel.ForEach(words, word =>
{
    concurrentDict.AddOrUpdate(word, 1, (key, oldValue) => oldValue + 1);
});

foreach (var kvp in concurrentDict)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

##### 并发日志记录

在并发环境中，ConcurrentDictionary可以用于记录日志或事件。

```c#
var log = new ConcurrentDictionary<DateTime, string>();

void LogEvent(string message)
{
    log.TryAdd(DateTime.Now, message);
}

// 模拟多线程环境
Parallel.For(0, 10, i =>
{
    LogEvent($"Event {i} occurred");
});

foreach (var kvp in log)
{
    Console.WriteLine($"{kvp.Key}: {kvp.Value}");
}
```

上述代码输出的日志记录个数是不确定的，因为同一时间只会记录一条记录。

##### 分布式锁或资源管理

ConcurrentDictionary可以用于实现简单的分布式锁或资源管理。

```c#
var resourceLocks = new ConcurrentDictionary<int, object>();
var lockObj = new object();

void AccessResource(int resourceId)
{
    var resourceLock = resourceLocks.GetOrAdd(resourceId, id => new object());
    lock(resourceLock)
    {
        Console.WriteLine($"Resource {resourceId} is being accessed by thread {Task.CurrentId}");
        Task.Delay(1000).Wait(); // 模拟资源访问
    }
}

// 多线程访问资源
Parallel.For(1, 10, i =>
{
    AccessResource(i % 3); // 重复访问相同的资源
});
```

上述代码对于同一资源的访问赋予了独占访问权限，若使用普通锁(将resourceLock改为lockObj)，那么某个进程访问资源池中的某个资源时，将会导致资源池中的其他资源也无法被访问。

## 参考文档

- [C#编程指南](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide)

- [.NET API参考文档](https://learn.microsoft.com/zh-cn/dotnet/api/)

- [System.Dynamic.ExpandoObject类使用说明](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/runtime-libraries/system-dynamic-expandoobject)
