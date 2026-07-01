---
title: 基于PGSQL咨询锁函数实现的分布式锁
date: 2024-08-30 19:45:42
categories:
- Database
tags:
- PostgreSQL
- Concurrency Control
- .NET
- C#
---

基于PGSQL咨询锁函数实现的一个分布式锁基础设施。

<!--more-->

## 前言

PostgreSQL提供了分布式锁服务，可以用于多个数据库实例(会话)之间协调访问资源。通过使用pg_advisory_lock等咨询锁函数，可以实现一个由应用定义其含义的分布式锁基础设施。


## 基本概念

### 乐观锁与悲观锁

锁的一种宏观分类方式是悲观锁和乐观锁。乐观锁和悲观锁是并发控制的一种机制，用于多线程或多进程环境下对共享资源的访问管理，以防止数据不一致或竞态条件。悲观锁与乐观锁并不是特指某个锁，而是在并发情况下的两种不同策略。

#### 悲观锁

悲观锁是一种对资源持有较悲观态度的锁定方式。它假设数据在并发访问时极有可能发生冲突，因此每次访问数据时都会先加锁，以确保其他线程不能访问此数据直到锁被释放。

悲观锁常见的实现方式是数据库中的行级锁、表级锁或行级锁等。一旦线程获得锁，其它尝试获取锁的线程都会被阻塞，直到锁被释放。

- 适用场景：在高并发、数据竞争激烈的场景中使用，如金融交易、库存管理等。
- 缺点：
    - 可能导致系统吞吐量降低，因为锁定机制会阻止其他线程并发访问资源
    - 容易产生死锁，如果锁的持有和释放管理不当，会导致系统无法继续运行

#### 乐观锁

乐观锁则持相对乐观的态度，假设并发操作冲突的可能性较小，因此不会主动加锁，而是进行数据版本检查来决定是否提交操作。

乐观锁一般通过版本号或时间戳等机制来实现。在数据读取时，获取当前版本号；在数据更新时，检查版本号是否与之前读取时的一致。如果一致，表示没有其他并发操作修改过数据，可以提交；否则，操作失败回滚。 

- 适用场景：适用于读操作多、写操作少的场景，如一些阅读类应用、CMS系统等。因为这些场景下，冲突发生的概率较低，乐观锁可以提高系统的并发性。
- 缺点：
    - 在并发冲突频繁的场景下，可能会导致大量重试操作，影响性能
    - 需要开发人员显式管理版本控制机制，增加开发复杂度

##### CAS

比较并替换(Compare-and-Swap)是乐观锁实现的基础。CAS操作包括三个步骤：读取内存值、比较内存值与预期值、如果相等则更新内存值。CAS锁可以有效地解决传统锁机制中的性能问题和死锁问题，是并发编程中常用的同步手段之一。

### 线程锁、进程锁与分布式锁

#### 线程锁

线程锁也被称为互斥锁(Mutex)，主要用于控制同一进程中的多个线程对共享资源的访问。

在C#中，可以使用lock关键字来实现互斥锁。

```c#
private Object lockObject = new Object();
...
    ...
    // 在需要保护共享资源的代码块中使用lock
    lock (lockObject)
     {
        // 访问和修改共享资源的代码
     }
    ...
...
```

#### 进程锁

进程锁是用于控制同一台机器上的多个进程对共享资源的访问。进程锁可以是系统级的，如文件锁；也可以是用户级的，如信号量。

#### 分布式锁

分布式锁是用于控制分布式系统中的多个节点对共享资源的访问。由于分布式系统中的节点可能位于不同的物理机甚至不同的地理位置，因此分布式锁的实现比线程锁和进程锁要复杂得多。分布式锁需要在网络中的多个节点之间进行协调，以保证锁的唯一性和一致性。

### PGSQL咨询锁

PostgreSQL提供了一种由应用定义其含义的锁，这种锁被称为咨询锁(Advisory Lock)。咨询锁是一种悲观锁、分布式锁。咨询锁用一个long类型的数值或两个int类型的数值标识一把锁，long类型标识的锁和int类型标识的锁互相独立。

咨询锁有两个锁定级别：会话级和事务级。
- 会话级锁定会持续到显式释放或会话结束，不受会话中事务的影响
- 事务级锁定不能显式释放，会持续锁定到事务结束

不论哪个级别的锁定都是可重入的，即同一个线程在持有锁的情况下，可以多次获取该锁而不会造成死锁。

咨询锁有两种锁定模式：独占和共享。
- 独占锁定和其它的独占锁定或共享锁定都互斥
- 共享锁定只和独占锁定互斥, 共享锁定之间不互斥

锁定模式不受锁定级别影响，同一把锁的会话级独占锁定和事务级独占锁定会正确的互斥。

#### 实现方式

PGSQL一共提供了21个咨询锁函数，抛开标识类型不谈，lock和unlock分别代表获取锁定和释放锁定。以_shared结尾的代表锁定是共享的而非独占的，带_xact_关键字代表锁定级别是事务级而非会话级的，带_try_关键字代表锁定是不可等待的。

##### 定义锁定模式

定义一个枚举作为区分锁定模式的参数。

```c#
/// <summary>
/// 锁定模式
/// </summary>
public enum LockMode
{
    /// <summary>
    /// 独占
    /// </summary>
    Exclusive = 0,

    /// <summary>
    /// 共享
    /// </summary>
    Shared = 1,
}
```

##### 定义咨询锁接口

会话级锁继承IDisposable接口，以便数据库连接等资源的释放。
```c#
public interface ISessionLock : IDisposable
{
}
```

定义不同标识类型、不同会话级别下的咨询锁接口。
```c#
public interface IAdvisoryLock
{
    /// <summary>
    /// 对long类型的数值标识的锁进行会话级别的锁定
    /// </summary>
    Task<ISessionLock> LockAsync(long k, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default);

    /// <summary>
    /// 对由两个int类型的数值标识的锁进行会话级别的锁定
    /// </summary>
    Task<ISessionLock> LockAsync(int m, int n, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default);

    /// <summary>
    /// 对long类型的数值标识的锁进行事务级别的锁定
    /// </summary>
    Task<bool> XactLockAsync(long k, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default);

    /// <summary>
    /// 对由两个int类型的数值标识的锁进行事务级别的锁定
    /// </summary>
    Task<bool> XactLockAsync(int m, int n, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default);
}
```

##### 实现会话级锁的显示释放

由于会话级锁定会持续到显式释放或会话结束，需要实现显示释放。

```c#
public class PGSQLSessionLock : ISessionLock
{
    private DatabaseFacade database;
    private long? kid;
    private int? mid;
    private int? nid;
    private bool isShare;
    private bool disposedValue;

    public PGSQLSessionLock(DatabaseFacade database, long kid, bool isShare)
    {
        this.database = database;
        this.kid = kid;
        this.isShare = isShare;
    }
    public PGSQLSessionLock(DatabaseFacade database, int mid, int nid, bool isShare)
    {
        this.database = database;
        this.mid = mid;
        this.nid = nid;
        this.isShare = isShare;
    }

    private void ReleaseLock()
    {
        if (isShare)
        {
            if (kid != null)
            {
                database.ExecuteSqlRaw("select pg_advisory_unlock_shared({0})", new object[] { kid.Value });
            }
            else
            {
                database.ExecuteSqlRaw("select pg_advisory_unlock_shared({0},{1})", new object[] { mid.Value, nid.Value });
            }
        }
        else
        {
            if (kid != null)
            {
                database.ExecuteSqlRaw("select pg_advisory_unlock({0})", new object[] { kid.Value });
            }
            else
            {
                database.ExecuteSqlRaw("select pg_advisory_unlock({0},{1})", new object[] { mid.Value, nid.Value });
            }
        }

    }

    protected virtual void Dispose(bool disposing)
    {
        if (!disposedValue)
        {
            if (disposing)
            {
            }

            ReleaseLock();
            database = null;
            disposedValue = true;
        }
    }

    ~PGSQLSessionLock()
    {
        Dispose(disposing: false);
    }

    public void Dispose()
    {
        Dispose(disposing: true);
        GC.SuppressFinalize(this);
    }
}
```

##### 实现咨询锁

实现不同锁定标识、不同锁定级别下的咨询锁函数，每个咨询锁函数定义了锁定标识、锁定模式、是否可等待参数。

```c#
public class PGSQLAdvisoryLock : IAdvisoryLock, ISingletonDependency
{
    IDbContextProvider<ABPDemoDbContext> _dbContextProvider;

    public PGSQLAdvisoryLock(IDbContextProvider<ABPDemoDbContext> dbContextProvider)
    {
        _dbContextProvider = dbContextProvider;
    }

    public async Task<ISessionLock> LockAsync(long k, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default)
    {
        var database = (await _dbContextProvider.GetDbContextAsync()).Database;

        var locked = await InternalLockAsync(new object[] { k }, lockMode, waiting, false, database, cancellationToken);

        ISessionLock result = locked ? new PGSQLSessionLock(database, k, lockMode == LockMode.Shared) : null;

        return result;
    }

    public async Task<ISessionLock> LockAsync(int m, int n, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default)
    {
        var database = (await _dbContextProvider.GetDbContextAsync()).Database;

        var locked = await InternalLockAsync(new object[] { m, n }, lockMode, waiting, false, database, cancellationToken);

        ISessionLock result = locked ? new PGSQLSessionLock(database, m, n, lockMode == LockMode.Shared) : null;

        return result;
    }

    public async Task<bool> XactLockAsync(long k, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default)
    {
        var database = (await _dbContextProvider.GetDbContextAsync()).Database;

        return await InternalLockAsync(new object[] { k }, lockMode, waiting, true, database, cancellationToken);
    }

    public async Task<bool> XactLockAsync(int m, int n, LockMode lockMode, bool waiting, CancellationToken cancellationToken = default)
    {
        var database = (await _dbContextProvider.GetDbContextAsync()).Database;

        return await InternalLockAsync(new object[] { m, n }, lockMode, waiting, true, database, cancellationToken);
    }

    private async Task<bool> InternalLockAsync(object[] parameters, LockMode lockMode, bool waiting, bool isXact, DatabaseFacade database, CancellationToken cancellationToken)
    {
        bool locked;

        var xact = isXact ? "_xact" : "";

        var param = parameters.Length == 1 ? "{0}" : "{0},{1}";

        var mode = lockMode switch
        {
            LockMode.Exclusive => "",
            LockMode.Shared => "_shared",
            _ => throw new NotImplementedException($"LockMode.{lockMode} is not implemented.")
        };

        if (waiting)
        {
            await database.ExecuteSqlRawAsync($"select pg_advisory{xact}_lock{mode}({param})", parameters);

            locked = true;
        }
        else
        {
            locked = (await database.SqlQueryRaw<bool>($"select pg_try_advisory{xact}_lock{mode}({param})", parameters).ToListAsync()).Single();
        }
        return locked;
    }
}
```

#### 应用方式

##### 在应用中定义和使用

首先定义一个锁枚举，具体名称由应用的功能派生，比如定义一个更新学生信息的锁StudentUpdate。

```c#
public static class Locks
{
    public const long StudentUpdate = 10001;
}
```

然后置于并发操作上下文(替换操作之前)即可。

```c#
private readonly IAdvisoryLock _advisoryLock;

public StudentAppService(..., IAdvisoryLock advisoryLock)
{
   ...
    _advisoryLock = advisoryLock;
}

...
var student = await _studentRepository.GetAsync(x => x.Name == input.Name, false, cancellationToken);

await _advisoryLock.LockAsync(Locks.StudentUpdate, LockMode.Exclusive, true, cancellationToken);

student.Name = input.Name;
student.StudentLevel = input.StudentLevel;

await _studentRepository.UpdateAsync(student, false, cancellationToken);

return ObjectMapper.Map<Student,StudentDto>(student);
...
```

##### 验证锁是否生效

在LockAsync之后打上断点，调用API命中断点之后，使用SQL语句查看锁

```sql
SELECT * FROM pg_locks t1
JOIN  pg_stat_activity t2
ON t1.pid  = t2.pid
ORDER BY t1.pid;
```

{% asset_img pgsql_pg_lock.png 命中断点后数据库锁的情况 %}

使用SQL语句再次获取该锁定标识对应的锁

```sql
SELECT pg_advisory_lock(10001);
```

{% asset_img pgsql_pg_lock_waiting.png 获取该锁定标识对应的锁 %}

可以看到该锁定标识对应的资源已被阻塞，将等待直到该资源变成可用。点击继续跳过断点之后即可再次成功获取。

## 参考文档

- [PGSQL中文文档-咨询锁函数](http://www.postgres.cn/docs/12/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS)