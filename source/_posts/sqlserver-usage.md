---
title: SQL Server常用SQL、T-SQL语句
date: 2024-12-04 10:02:40
categories:
  - Database
tags:
  - SQL Server
---

SQL Server数据库的适用场景，以及一些常用SQL、T-SQL语句。

<!--more-->

## 前言

Microsoft SQL Server(又称 MS SQL)是一种关系数据库管理系统(RDBMS)。应用程序和工具连接到SQL Server实例或数据库，并使用Transact-SQL(T-SQL)进行通信。

## 适用场景

与MySQL相比，SQL Server数据库有以下优势：
- 高可用性解决方案：SQL Server提供了一套成熟的高可用性解决方案，如Always - On可用性组。它允许数据库管理员在不同的服务器之间配置数据库副本，实现自动故障转移。
- 强大的查询优化器：SQL Server拥有一个先进的查询优化器，它能够根据查询的复杂程度、数据分布以及索引情况自动生成高效的执行计划。例如，在处理包含多个表连接、子查询和复杂条件的大型企业级查询时，SQL Server的查询优化器能够通过多种优化策略（如选择合适的连接算法、索引使用等）来减少查询响应时间。
- 集成的身份认证和权限管理：SQL Server支持Windows身份认证和SQL Server身份认证两种方式。在企业环境中，Windows身份认证可以与企业的活动目录集成，方便用户管理和权限控制。
- 数据加密功能：SQL Server提供了透明数据加密（TDE）功能，可以对整个数据库进行加密，包括数据文件和日志文件。这在数据存储和传输过程中，有效保护了数据的安全性。
- 与.NET集成：SQL Server与.NET 框架配合紧密，通过ADO.NET等技术，开发人员可以轻松地在各种.NET应用程序中访问和操作数据库。这种紧密的集成使得企业在构建基于微软技术栈的应用系统时，选择SQL Server能够减少技术整合的复杂性。

## 数据库连接

.NET项目配置文件中数据库连接字符串如下，

```json
{
    "ConnectionStrings": {
        "SqlserverConnection": "Data Source=DESKTOP-Q4ORSFH\\SQLEXPRESS;Initial Catalog=your-database;Persist Security Info=True;User ID=sa;Password=your-password"
    }
}
```

备注：SQL Server默认监听的端口号为1433，因此连接字符串可以不指定端口号

### GUI方式

SQL Server Management Studio（SSMS）是微软为SQL Server数据库提供的一款功能强大、集成度高的管理工具。以免费版为例，选择SQL Server身份认证方式登录，默认登录名为sa。

### CLI方式

sqlcmd是SQL Server自带的命令行工具，它允许用户在命令提示符或批处理文件中执行T-SQL语句和脚本。通过sqlcmd可以连接到本地或远程的SQL Server实例，执行数据库操作，如查询数据、创建数据库对象、执行存储过程等。

以SQL Server身份认证方式连接到数据库，

`sqlcmd -S localhost -U sa -P your-password -d your-database`

以Windows身份认证方式连接到数据库，

`sqlcmd -S localhost -E -d your-database` 

`sqlcmd -S localhost -T -d your-database`

## T-SQL

T - SQL（Transact - SQL）是微软为SQL Server数据库管理系统开发的一种编程语言。它是SQL的扩展，用于在SQL Server环境中进行数据定义、数据操纵、数据控制以及事务处理等操作。T - SQL不仅包含了标准SQL的命令，还增加了许多用于增强功能和编程便利性的扩展语法。

### 数据定义语言（DDL）

#### CREATE 

创建数据库表

```sql
CREATE TABLE [dbo].[table_name] (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    [column_name] VARCHAR(50)
);
```

#### ALTER

在表中添加新的列

```sql
ALTER TABLE [dbo].[table_name]
ADD [column_name] VARCHAR(20);
```

#### DROP 

```sql
DROP TABLE [dbo].[table_name];
```

### 数据操纵语言（DML）

#### INSERT 

在表中插入数据

```sql
INSERT INTO [dbo].[table_name] ([column_name]) VALUES ('xxx');
```

#### Update

更新表中数据

```sql
UPDATE [dbo].[table_name]
SET [column_name] = 'xxx'
WHERE Id = 1;
```

#### DELETE

```sql
DELETE FROM [dbo].[table_name]
```

### 数据查询语言（DQL）

#### SELECT

```sql
SELECT [column_name] FROM [dbo].[table_name]
```

### 数据控制语言（DCL）

#### 用户权限管理

授予用户查询权限

```sql
GRANT SELECT ON [dbo].[table_name] TO [user_name];
```

撤销用户查询权限

```sql
REVOKE SELECT ON [dbo].[table_name] TO [user_name];
```

#### 事务控制

可以使用BEGIN TRANSACTION、COMMIT TRANSACTION和ROLLBACK TRANSACTION语句来控制事务


```sql
BEGIN TRANSACTION;
UPDATE [dbo].[table_name]
SET [column_name] = 'xxx'
WHERE Id = 1;
DELETE FROM [dbo].[table_name]
WHERE Id = 2;
COMMIT TRANSACTION;
```

### 存储过程和函数

#### 存储过程

T - SQL支持存储过程的创建和调用。存储过程是一组预编译的T - SQLx语句，可以在数据库中存储并反复调用。

```sql
CREATE PROCEDURE [dbo].[GetAllInfos]
AS
BEGIN
    SELECT * FROM [dbo].[table_name];
END;

-- 执行存储过程
EXEC [dbo].[GetAllInfos];
```

#### 自定义函数

用户自定义函数可以返回一个值，并且可以在查询语句中使用。

```sql
CREATE FUNCTION [dbo].[AddNumbers](@num1 INT, @num2 INT)
RETURNS INT
AS
BEGIN
    RETURN @num1 + @num2;
END;

-- 执行函数
SELECT [dbo].[AddNumbers](3, 5) AS Result;
```

### 流程控制

T - SQL 包含多种流程控制语句，用于编写复杂的程序逻辑。

#### IF ELSE

```sql
DECLARE @Variable DECIMAL(10,2);
SELECT @Variable = [column_name] FROM [dbo].[table_name] WHERE Id = 1;
IF @Variable > 10000
BEGIN
    ...
END
ELSE
BEGIN
    ...
END
```

#### WHILE

```sql
DECLARE @Index INT;
SET @Index = 1;
WHILE @Index > 0
BEGIN
    -- 执行一些更新操作
    SET @Index = @Index + 1;
END
```

## 参考文档

- [SQL Server官方中文文档](https://learn.microsoft.com/zh-cn/sql/sql-server)

- [sqlcmd使用参考](https://learn.microsoft.com/zh-cn/sql/tools/sqlcmd/sqlcmd-utility)