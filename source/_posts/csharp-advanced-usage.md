---
title: C#进阶语法
date: 2025-03-20 14:57:10
categories:
- Language
tags:
- c#
---

c#进阶语法，包含委托与事件、动态类型等。

<!--more-->

## 前言

c#中的一些进阶语法使用记录，包含委托(Delegate)与事件(Event)、动态类型(dynamic)等。 

## 委托(Delegate)与事件(Event)

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

## 动态类型(dynamic)

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

## 参考文档

- [C#编程指南](https://learn.microsoft.com/zh-cn/dotnet/csharp/programming-guide)

- [System.Dynamic.ExpandoObject类使用说明](https://learn.microsoft.com/zh-cn/dotnet/fundamentals/runtime-libraries/system-dynamic-expandoobject)