---
title: .NET Core 实现定时后台任务
date: 2025-01-06 15:35:14
categories:
- Framework
tags:
- BackgroundJob
- .NET
- ASP.NET Core
---

.NET Core实现定时后台任务的几种方法。

<!--more-->

## 前言

在开发应用时，定时任务是一个常见的需求。它可以自动化周期性的操作，例如定期同步数据、定时清理缓存、发送通知等。

## 应用场景

- 数据处理
    - 数据同步：定时从外部数据源（如数据库、API等）同步数据到本地系统
    - 数据备份：定期备份数据库或文件系统中的数据，以防止数据丢失
- 系统维护
    - 缓存更新：定时刷新缓存数据，确保缓存中的数据是最新的
    - 日志清理：定时清理日志文件，防止日志文件占用过多磁盘空间
- 任务调度
    - 定时任务调度：根据预设的时间表或条件，自动执行特定的任务。例如，每天凌晨自动执行数据备份任务
    - 任务队列处理：定时从任务队列中取出任务并执行，适用于需要异步处理的场景
- 用户交互
    - 定时通知：定时向用户发送通知或提醒，例如发送邮件、短信或应用内通知

## 实现方式

### 基于IHostedService接口实现自定义后台服务

定时后台任务使用System.Threading.Timer类。在StartAsync上使用计时器执行DoWork任务，在StopAsync上禁用计时器，并在Dispose上处置服务容器时处置计时器。

Timer不等待先前的DoWork执行完成。使用Interlocked.Increment以原子操作的形式将执行计数器递增，这可确保多个线程不会并行更新executionCount。

#### 具体实现

首先创建自定义的后台服务类，

```c#
public class TimedHostedService : IHostedService, IDisposable
{
    private int executionCount = 0;
    private readonly ILogger<TimedHostedService> _logger;
    private Timer _timer;

    public TimedHostedService(ILogger<TimedHostedService> logger)
    {
        _logger = logger;
    }

    public Task StartAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Timed Hosted Service running.");

        _timer = new Timer(DoWork, null, TimeSpan.Zero,
            TimeSpan.FromSeconds(5));

        return Task.CompletedTask;
    }

    private void DoWork(object state)
    {
        var count = Interlocked.Increment(ref executionCount);

        _logger.LogInformation(
            "Timed Hosted Service is working. Count: {Count}", count);
    }

    public Task StopAsync(CancellationToken stoppingToken)
    {
        _logger.LogInformation("Timed Hosted Service is stopping.");

        _timer?.Change(Timeout.Infinite, 0);

        return Task.CompletedTask;
    }

    public void Dispose()
    {
        _timer?.Dispose();
    }
}
```

然后在Startup中注册服务。

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddHostedService<TimedHostedService>();
}
```

### 使用BackgroundService

BackgroundService是用于实现长时间运行的IHostedService的基类，是IHostedService的一个简单实现。

StartAsync应仅限于短期任务，因为托管服务是按顺序运行的，在StartAsync运行完成之前不会启动其他服务。长期任务应放置在ExecuteAsync中。

#### 具体实现

首先创建服务类，

```c#
public interface IWorkService
{
    Task TaskWorkAsync(CancellationToken cancellationToken);
}
```

```c#
public class WorkService : IWorkService
{
    private int executionCount = 0;
    private readonly ILogger<WorkService> _logger;
    private DateTime nextDateTime;

    public WorkService(ILogger<WorkService> logger)
    {
        _logger = logger;
    }

    public async Task TaskWorkAsync(CancellationToken cancellationToken)
    {
        while (!cancellationToken.IsCancellationRequested)
        {
            var StartHour = 0;
            var StartMinute = 0;
            var StartSecond = 0;
            var IntervalMinute = 1440;

            // 计算下一个时间节点
            var now = DateTime.Now;
            nextDateTime = new DateTime(now.Year, now.Month, now.Day, StartHour, StartMinute, StartSecond).AddMinutes(IntervalMinute);
            if(executionCount == 0)
            {
                nextDateTime = FirstDateTime;
            }
            else
            {
                nextDateTime = nextDateTime.AddMinutes(IntervalMinute);
            }
            
            if(nextDateTime < now)
            {
                var delay = nextDateTime.AddDays(1) - now;
                await Task.Delay(delay, cancellationToken);
            }
            else
            {
                var delay = nextDateTime - now;
                await Task.Delay(delay, cancellationToken);
                await DoWork();

                var count = Interlocked.Increment(ref executionCount);
                _logger.LogInformation("Timed Hosted Service is working. Count: {Count}", count);
            }

        }
    }

    private async Task DoWork(){}
}
```

然后创建后台服务调用类，

```c#
public class CustomBackgroundService : BackgroundService
{
    private readonly IServiceProvider _services;

    public CustomBackgroundService(IServiceProvider services)
    {
        _services = services;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        using var scope = _services.CreateScope();
        //获取服务类
        var taskWorkService = scope.ServiceProvider.GetRequiredService<IWorkService>();
        //执行服务类的定时任务
        await taskWorkService.TaskWorkAsync(stoppingToken);
    }
}
```

最后在Startup中注册后台服务，并添加主机服务。

```c#
public void ConfigureServices(IServiceCollection services)
{
    services.AddScoped<IWorkService, WorkService>();
    services.AddHostedService<CustomBackgroundService>();
}
```

## 参考文档

- [如何实现IHostedService接口](https://learn.microsoft.com/zh-cn/dotnet/core/extensions/timer-service)

- [如何在ASP.NET Core中使用托管服务实现后台任务](https://learn.microsoft.com/zh-cn/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1&tabs=visual-studio)