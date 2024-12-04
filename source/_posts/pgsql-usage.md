---
title: PostgreSQL常用SQL语句
date: 2024-08-12 22:54:01
categories: 
- Database
tags:
- PostgreSQL
---

PostgreSQL数据库中的一些常用SQL。

<!--more-->

## 前言

PostgreSQL相较于MySQL而言，具有更强的事务处理能力，能够处理更复杂的数据操作和高并发访问。同时其提供了更丰富的索引类型，能够进行更细粒度的查询和优化。此外还支持多种数据类型和函数库，可以处理更复杂的应用程序。

## 数据库连接

.NET项目配置文件中数据库连接字符串如下，

```json
{
    "ConnectionStrings": {
        "PgsqlConnection": "Host=localhost;Database=your-database;Username=postgres;Password=your-password;Persist Security Info=True;"
    }
}
```

备注：PGSQL默认监听的端口号为5432，因此连接字符串可以不指定端口号

## PGSQL常用语句


### 扩展安装

安装uuid生成函数扩展

```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
```


注: []标注的内容必须带双引号""

### SELECT

获取uuid
```sql
SELECT uuid_generate_v4();
```

获取东八时(北京时间)
```sql
SELECT now() AT TIME ZONE ('CCT');
```

类型转换
```sql
--写法一
SELECT t."Id"::varchar;
--写法二
SELECT cast(t."Id" AS varchar);
```


### INSERT


```sql
INSERT INTO public.[table_name] 
([column_name], ...) 
VALUES(column_value, ...);
```

### UPDATE

```sql
UPDATE public.[table_name]  
SET [column_name]  = column_value
WHERE ...
```

#### FOR UPDATE(行级锁)

更新表时跳过被锁定的行

注: `[column_a_id]`代表外键列，依赖于表a

```sql
UPDATE public.[table_b]    
SET [column_3]  = column_value
FROM 
(
        	SELECT b.*, a.[column_1]
        	FROM [table_b] b
        	JOIN [table_a] a ON b.[column_a_id] = a."Id"
        	WHERE ...
        	AND NOT EXISTS 
        	(
        		SELECT *
        		FROM [table_b] b2
        		WHERE b2.[column_a_id] = b.[column_a_id] 
                AND b2.[column_3] = column_value
        	) 
        	ORDER BY a.[column_2]
        	LIMIT 1 FOR UPDATE OF b SKIP LOCKED
) AS t
where  t."Id" = b."Id"  
returning t.[column_1]
```

### TRUNCATE/DROP/DELETE

TRUNCATE常用于清理数据，可重置自增键，不破坏表结构，但无法回滚也无法携带WHERE语句
```sql
TRUNCATE TABLE [table_name] CASCADE
```

DROP会删除表结构及其依赖的索引、外键约束，无法回滚
```sql
DROP TABLE [table_name]
```

DELETE可以携带WHERE语句，可以回滚，但不会重置自增键也不会释放表或索引占用的空间
```sql
DELETE FROM [table_name]
WHERE ...
```

### ALTER

更新表字段排序规则
```sql
ALTER TABLE [table_name]  ALTER COLUMN [column_name] 
SET DATA TYPE CHARACHTER VARING(50) COLLATE "zh-x-icu"
```


### LEFT JOIN(视图)

注: `[column_a_id]`代表外键列，依赖于表a；`[column_c_id]`代表外键列，依赖于表c

```sql
CREATE OR REPLACE VIEW public.[view_name] 
AS SELECT 
	a.[column_name],
    ...
    c.[column_name],
    ...
    t.[column_name],
    ...
FROM [table_a] a
LEFT JOIN ( 
    		SELECT 
    		b.[column_name],
            ...
            FROM [table_b] b
		  ) t 
ON a."Id" = t.[column_a_id]
LEFT JOIN [table_c] c
ON a.[column_c_id] = c."Id"
WHERE ...
```

c#中LINQ写法，
```c#
from l in leftSet
join r in rightSet on l.Id equals r.Id into groupJoin
from r in groupJoin.DefaultIfEmpty() 
select new { ... };
```

### 函数与存储过程


#### 存储函数

生成uuid
```sql
CREATE OR REPLACE FUNCTION public.uuid_generate_v4()
RETURNS uuid LANGUAGE c
PARALLEL SAFE STRICT
AS '$libdir/uuid-ossp', 
$function$uuid_generate_v4$function$;
```


#### 存储过程

```sql
CREATE OR REPLACE PROCEDURE public.[procedure_name](IN column_1_variable column_1_type) LANGUAGE plpgsql
AS $procedure$

BEGIN 

	UPDATE [table_a]  a
	SET [column_2] = column_value,
	WHERE a.[column_1] = column_variable;

END;

$procedure$;
```

### 序列

创建自增序列
```sql
CREATE SEQUENCE task_id_seq
START WITH 1
INCREMENT BY 1
NO MINVALUE
NO MAXVALUE
CACHE 1;
```


### 窗口函数和聚合函数

在PostgreSQL中，窗口函数（Window Functions）是对一组行进行计算的函数，比如求和、平均、排名等。它们可以用于创建复杂的分析查询，并且可以对一系列的行而不仅仅是组内的行进行操作。

聚合函数（Aggregate Functions）则是用于合并多个值，返回单一的值的函数，如COUNT、SUM、AVG、MIN、MAX等。

例如，分组计算平均值，窗口函数可以带出额外信息`[column_1]`
```sql
SELECT
    [column_1],
    [column_2],
    AVG([column_2]) OVER (PARTITION BY [column_3]) AS avg_value
FROM
    [table_a]    
WHERE
    [column_2] > avg_value;
```

而聚合函数只能操作组内成员
```sql
SELECT
    [column_3]
    AVG([column_2]) AS avg_value
FROM
    [table_a]   
GROUP BY 
    [column_3]
```

组合用法，计算组内排名
```sql
SELECT
    [column_1],
    [column_2],
    RANK() OVER (PARTITION BY [column_3] ORDER BY [column_2] DESC) AS rank
FROM
    [table_a] 
```

### 事务隔离级别

PGSQL数据库事务的隔离级别有以下四种:
- 读未提交(READ UNCOMMITTED)
- 读已提交(READ COMMITTED)
- 重复读(REPEATABLE READ)
- 串行化(SERIALIZABLE)

对于并发事务，我们不希望发生的行为如下：
- 脏读：一个事务读取了另一个未提交的事务写入的数据
- 重复读：一个事务重新读取前面读取过的数据时，发现该数据已改变
- 幻读：当某个事务在读取某个范围内的记录时，另外一个事务中又在该范围插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行

对于不同事务隔离级别，脏读、重复读、幻读的可能性如下

| ISOLATION LEVEL     | 脏读     | 重复读     |幻读
| -------- | -------- | -------- | -------- |
| READ UNCOMMITTED | YES | YES | YES |
| READ COMMITTED | NO | YES | YES |
| REPEATABLE READ | NO | NO | YES |
| SERIALIZABLE | NO | NO | NO |

```c#
// ABP中配置数据库事务隔离级别为读已提交
...
Configure<AbpUnitOfWorkOptions>(options =>
{
    options.IsolationLevel = IsolationLevel.ReadCommitted;
});
...
```

### 进程与会话

查看进程
```sql
SELECT * FROM pg_locks t1
JOIN  pg_stat_activity t2
ON t1.pid  = t2.pid
ORDER BY t1.pid;
```

关闭进程
```sql
SELECT pg_terminate_backend(PID);
```

查看会话
```sql
SELECT * FROM pg_stat_activity WHERE datname = 'your_database';
```

关闭当前会话
```sql
DISCARD ALL;
```

## 参考文档

[PGSQL官方文档](https://www.postgresql.org/docs/)