---
title: Redis数据库常用CLI命令及应用场景
date: 2024-08-14 10:58:38
categories:
- Database
tags:
- Redis
- NoSQL
- ABP
---

Redis数据库的使用场景、使用方法、重要概念，一些常用的CLI命令，以及应用场景。

<!--more-->

## 前言

Redis(Remote Dictionary Server)是一个高性能的(key/value)分布式内存数据库，基于内存运行并支持持久化的NoSQL数据库，常用于数据缓存、充当消息队列等。Redis不仅仅支持简单的key-value类型的数据(如string)，同时还提供list，set，sorted set，hash等数据结构的存储。

## 安装配置

Redis在Mac/Linux系统下可通过命令行直接安装，而在Windows系统下需要通过WSL安装一个Linux系统来安装Redis，或者通过Docker下载Redis镜像通过镜像运行Redis，也可以使用传统EXE安装包安装(不推荐)。Redis可通过CLI、API、GUI(如RedisInsight)三种方式使用。

### 通过EXE安装包启动

启动Redis

`redis-server`

报错
> Could not create server TCP listening socket *:6379: bind: 在一个非套接字上尝试了一个操作 。

Redis安装目录下，依次输入

`redis-cli.exe`

`shutdown`

`exit`

`redis-server.exe redis.windows.conf`

### 通过Docker启动

拉取Redis官方镜像

`docker pull redis`

启动Redis，设置端口映射(主机:容器)，设置数据持久化挂载，并启用AOF持久化

`docker run --name my-redis -p 6379:6379 -v /path/to/redis/data:/data -d redis redis-server --appendonly yes`

使用自定义配置文件启动

```bash
docker run \
  --name my-redis \
  -v /path/to/redis.conf:/usr/local/etc/redis/redis.conf \
  -v /path/to/data:/data \
  -d redis \
  redis-server /usr/local/etc/redis/redis.conf
```

> 注：/path/to/redis.conf为主机(HOST)上的Redis配置文件路径，/usr/local/etc/redis/redis.conf为容器内部的默认Redis配置文件路径。容器启动时会使用主机上的配置文件替代容器内的默认配置，对主机配置文件的修改会实时生效到容器内(无需重建容器)

## CLI常用命令

### 数据结构

#### key-value

设置键值对

`SET [key] [value]`

获取键值

`GET [key]`

删除键值对

`DELETE [key]`

判断键值对是否存在

`EXISTS [key]`

获取所有键值对

`KEYS *`

获取所有以xx结尾的键值对

`KEYS *xx`

游标方式获取所有以xx:开头的前n个键值对(非阻塞查询)

`SCAN 0 MATCH xx:* COUNT n`

删除所有键值对

`FLUSHALL`

设置键值对过期时间

`EXPIRE [key] [seconds]`

获取键值对过期时间

`TTL [key]`

设置键值对并设置过期时间

`SETEX [key] [seconds] [value]`

设置不存在的键值对(若已存在则忽略执行)

`SETNX [key] [value]`

#### List

创建列表并添加元素

`LPUSH [list] [value, ...]`

`RPUSH [list] [value, ...]`

获取整个列表

`LRANGE [list] 0 -1`

删除列表元素

`LPOP [list] [count]`

`RPOP [list] [count]`

获取列表长度

`LLEN [list]`

仅保留索引为n到m的部分列表

`LTRIME [list] n m`

#### Set

创建集合并添加元素

`SADD [set] [value, ...]`

获取集合中的元素

`SMEMBERS [set]`

判断元素是否在集合中

`SISMEMBER [set] [value]`

删除集合中的元素

`SREM [set] [value]`

#### SortedSet

创建有序集合并添加元素

`ZADD [sortedset] [(score value), ...]`

获取有序集合全部元素

`ZRANGE [sortedset] 0 -1 WITHSCORES`

获取有序集合某个元素权值

`ZSCORE [sortedset] [value]`

获取有序集合某个元素排名

`ZRANK [sortedset] [value]`

`ZREVRANK [sortedset] [value]`

删除有序集合某个元素

`ZREM [sortedset] [value]`

#### Hash

创建哈希集合

`HSET [hash] [key] [value]`

获取哈希集合中的元素

`HGET [hash] [key]`

获取哈希集合中的全部元素

`HGETALL [hash]`

删除哈希集合中的某个元素

`HDEL [hash] [key]`

判断哈希集合中的某个元素是否存在

`HEXISTS [hash] [key]`

获取哈希集合中的键

`HKEYS [hash]`

获取哈希集合长度

`HLEN [hash]`

### 发布订阅模式

订阅频道

`SUBSCRIBE [channel]`

发布频道消息

`PUBLISH [channel] [message]`

#### Stream

向消息队列中添加一条消息

`XADD [stream] * [key] [value]`

获取消息队列中消息数量

`XLEN [stream]`

获取消息队列中的所有消息

`XRANGE [stream] - +`

删除消息队列中的某条消息

`XDEL [stream] [id]`

删除消息队列中的所有消息

`XTRIM [stream] MAXLEN 0`

读取消息队列中索引为i开始的n条消息，如不存在阻塞x秒

`XRED COUNT n BLOCK x STREAMS [stream] i`

创建消费者组

`XGROUP CREATE [stream] [group] [id]`

获取消费者组信息

`XINFO GROUPS [stream]`

向消费者组中添加消费者

`XGROUP CREATECONSUMER [stream] [group] [consumer]`

读取消费者组中某消费者的最新n条消息，如不存在阻塞x秒

`XREDGROUP [group] [consumer] COUNT n BLOCK x STREAMS [stream] >`

### 事务

注: Redis中的事务概念与传统事务概念不同，不保证原子性


标记一个事务开始

`MULTI`

提交事务

`EXEC`

### 持久化

Redis有RDB(Redis Database)和AOF(Append Only File)两种数据持久化方式。

#### RDB

RDB相当于数据快照，可在redis.confg文件中配置自动触发，例如<u>save 60 10000</u>代表60s内如果有10000次修改则触发一次快照，也可通过CLI中的save命令手动触发。

#### AOF

AOF相当于使用日志记录操作命令，可在redis.confg文件中配置参数<u>appendonly yes</u>自动触发。

### 主从复制

Redis主从复制是指将Redis主节点数据复制到从节点，数据的复制是单向的。

假设主节点端口号为6379，从节点端口号为6380。在从节点的redis.confg文件中配置参数<u>replicaof 127.0.0.1 6379</u>启用主从复制功能。启动从节点6380服务后，使用CLI命令

`redis-cli -p 6380`

`info replication`

即可看到当前节点角色已从master变为slave。

### 哨兵模式

Redis主从复制存在一个问题，如果主节点宕机了，需要手动去将从节点设置为主节点。为了实现主节点的自动故障转移，Redis引入了一个独立的进程来监视主节点，通过发布订阅模式通知从节点变更为主节点。

创建配置文件sentinel.conf并配置参数<u>sentinel monitor master 127.0.0.1 6379 1</u>(1代表故障转移需要同意的哨兵的个数)，使用CLI命令

`redis-sentinel sentinel.conf`

即可启动哨兵进程。

## 应用场景

### Redis分布式缓存

Redis最常用的应用场景是数据缓存。Redis将高频访问数据从磁盘(数据库)移至内存，响应时间从毫秒级降至微秒级。1个Redis实例(16GB内存)可替代数百台数据库服务器的缓存负载。Redis作为独立缓存层，可以保护后端系统免受流量冲击。

ABP中集成了Redis分布式缓存，下面是一个使用示例。

Web层WebModule配置如下

```c#
private void ConfigureRedis(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();

    Configure<AbpDistributedCacheOptions>(options =>
    {
        options.KeyPrefix = "ABPDemo:"; // 可选：设置缓存键前缀
    });

    // 配置Redis分布式缓存
    context.Services.AddStackExchangeRedisCache(options =>
    {
        options.Configuration = configuration["Redis:Configuration"];
    });

    context.Services.AddSingleton<IConnectionMultiplexer>(_ => ConnectionMultiplexer.Connect(configuration["Redis:Configuration"]));
}
```

Application层中使用示例如下

```c#
public class StudentAppService : ABPDemoAppService, IStudentAppService
{
    ...
    private readonly IDistributedCache<StudentCacheItem> _distributeCache;

    public StudentAppService(..., IDistributedCache<StudentCacheItem> distributedCache)
    {
        ...
        _distributeCache = distributedCache;
    }

    [Authorize(Roles = ABPDemoRoles.Admin)]
    public async Task<StudentCacheItem> GetStudentFromCacheAsync(Guid id, CancellationToken cancellationToken)
    {
        var key = $"Student:{id}"; // 缓存Key
        return await _distributeCache.GetOrAddAsync(
            key, 
            async () =>  // 如果缓存不存在，从数据库加载
            {
                var student = await _studentRepository.GetAsync(id, false, cancellationToken);
                var studentCacheItem = ObjectMapper.Map<Student, StudentCacheItem> (student);
                return studentCacheItem;
            },
            () => new DistributedCacheEntryOptions
            {
                AbsoluteExpiration = DateTimeOffset.Now.AddHours(1) // 1小时后过期
            },
            null, false, cancellationToken
        );
    }
}
```

有些场景需要手动清除Redis缓存，为此引入RedisCacheManager，开发环境中Redis缓存清除可以使用`KEYS`。

```c#
public class RedisCacheManager :  ITransientDependency
{
    private readonly IConnectionMultiplexer _redis;

    public RedisCacheManager(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task ClearByPrefixAsync(string keyPrefix)
    {
        var db = _redis.GetDatabase();
        var server = _redis.GetServer(_redis.GetEndPoints().First());

        var keys = new List<RedisKey>();
        await foreach (var key in server.KeysAsync(pattern: $"{keyPrefix}*"))
        {
           keys.Add(key);
        }

        批量删除
        if (keys.Count != 0)
        {
           await db.KeyDeleteAsync(keys.ToArray());
        }
    }
}
```

KEYS是Redis的一个命令，用于模式匹配查询键(如`KEYS user:*`)，但会阻塞整个Redis服务直到扫描完成，在生产环境中推荐使用`SCAN`。

```c#
public class RedisCacheManager :  ITransientDependency
{
    private readonly IConnectionMultiplexer _redis;

    public RedisCacheManager(IConnectionMultiplexer redis)
    {
        _redis = redis;
    }

    public async Task ClearByPrefixAsync(string keyPrefix)
    {
        var db = _redis.GetDatabase();

        #region 使用 SCAN 迭代查询（避免 KEYS 阻塞）
        var script = @"
                        local keys = redis.call('SCAN', 0, 'MATCH', ARGV[1], 'COUNT', 1000)
                        for i, key in ipairs(keys[2]) do
                            redis.call('DEL', key)
                        end
                        return keys[1]";

        try
        {
            // 递归扫描直到返回的游标为 0
            long cursor;
            do
            {
                var result = (RedisResult[])await db.ScriptEvaluateAsync(script, values: new RedisValue[] { $"{keyPrefix}*" });

                cursor = (long)result[0];
            } while (cursor != 0);
        }
        catch { }
        #endregion
    }
}
```

### Redis分布式锁

Redis可以作为分布式锁，替代基于数据库的锁方案。Redis拥有内存级操作和单命令原子性。Redis基于内存的原子操作(如`SETNX`)，获取/释放锁的延迟通常在毫秒级，远优于基于数据库的锁方案。Redis的`SET key value NX PX 30000`可原子性实现「加锁+过期时间设置」，避免多命令竞态条件。

ABP中同样集成了Redis分布式锁，下面是一个使用示例。

Web层WebModule配置如下

```c#
private void ConfigureRedis(ServiceConfigurationContext context)
{
    var configuration = context.Services.GetConfiguration();

    // 配置Redis分布式锁
    context.Services.AddSingleton<IDistributedLockProvider>(sp =>
    {
        var connection = ConnectionMultiplexer.Connect(configuration["Redis:Configuration"]);

        return new RedisDistributedSynchronizationProvider(connection.GetDatabase());
    });
}
```

Application层中使用示例如下

```c#
public class StudentAppService : ABPDemoAppService, IStudentAppService
{
    ...
    private readonly IAbpDistributedLock _abpDistributedLock;

    public StudentAppService(..., IAbpDistributedLock abpDistributedLock)
    {
        ...
        _abpDistributedLock = abpDistributedLock;
    }

    [Authorize(Roles = ABPDemoRoles.Admin)]
    public async Task UpdateStudentLevelWithLockAsync(Guid id, StudentLevelType level, CancellationToken cancellationToken)
    {
        // 定义锁的名称
        var lockName = $"Student:{id}:UpdateLock";
        // 尝试获取锁
        await using var handle = await _abpDistributedLock.TryAcquireAsync(lockName, TimeSpan.Zero, cancellationToken);
        if (handle != null)
        {
            // 临界区代码
            var student = await _studentRepository.GetAsync(id, false, cancellationToken);
            student.StudentLevel = level;
            await _studentRepository.UpdateAsync(student, true, cancellationToken);
        }
    }
}
```

## 参考文档

[Redis CLI命令官方文档](https://redis.io/docs/latest/commands/)