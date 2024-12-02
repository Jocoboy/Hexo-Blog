---
title: Oracle常用SQL、PL/SQL语句
date: 2024-11-29 15:48:14
categories:
  - Database
tags:
  - Oracle
---

Oracle数据库的适用场景，以及一些常用SQL、PL/SQL语句。

<!--more-->

## 前言

Oracle是由Oracle公司开发和维护的商业数据库管理系统，使用Oracle数据库通常需要购买许可证，价格因版本、功能、用户数量等因素而异。它主要面向对数据安全、性能和稳定性要求极高的大型企业、金融机构、电信公司等，这些企业愿意为高端的数据库解决方案支付较高的成本。Oracle在数据库技术和Java之间进行了大量的技术整合，Java是Oracle生态系统中企业级应用开发的核心语言，在企业资源规划（ERP）系统、客户关系管理（CRM）系统等大型企业应用中具有重要地位。

## 适用场景

Oracle数据库有以下性能特点和功能特性：
- 处理复杂查询：Oracle在处理复杂的企业级查询和大规模数据操作方面表现卓越。
- 并发处理：Oracle采用多版本并发控制（MVCC）机制来处理高并发事务，同时还具备强大的锁机制来确保数据的一致性和完整性。在高并发的 OLTP（联机事务处理）环境下，如银行的网上交易系统，大量用户同时进行账户查询、转账等操作时，Oracle 能够有效地协调并发事务，避免数据冲突。
- 数据存储和读写性能：Oracle在存储结构上有多种存储选项，如自动存储管理（ASM）等，可以根据不同的应用场景和数据类型优化存储性能。
- 数据类型和扩展性：Oracle支持丰富的内置数据类型，包括基本数据类型以及复杂的数据类型（如XMLTYPE用于处理XML数据）。在扩展性方面，Oracle 提供了诸如分区表、索引组织表等功能来提高数据管理的效率。
- 备份和恢复功能：Oracle提供了一套完整的备份和恢复解决方案，包括物理备份和逻辑备份。它还支持闪回技术，可以在一定程度上快速恢复数据到某个时间点，这在应对数据误操作等情况时非常有用。
- 安全机制：Oracle具有高度复杂的安全体系，包括用户认证、授权、角色管理、数据加密等多个层面。它可以通过细粒度的权限控制，对不同用户和角色访问不同的数据对象和操作进行严格的限制。

## 常用SQL语句

### 模式

在Oracle数据库中，模式是一个逻辑概念，它是一组数据库对象（如表、视图、存储过程、函数、序列等）的集合。可以将模式看作是一个用户所拥有的对象的容器，每个模式都与一个数据库用户相关联。

#### 使用CREATE SCHEMA语句直接创建

```sql
-- 查询所有用户
SELECT USERNAME, USER_ID, ACCOUNT_STATUS FROM DBA_USERS;
-- 查询当前用户
SELECT USER FROM DUAL;
-- 授权创建模式的系统权限给用户
GRANT CREATE SCHEMA TO test_user;
-- 创建模式并授权所有者
CREATE SCHEMA new_schema AUTHORIZATION test_user;
```

#### 通过创建用户隐式创建模式

当创建一个新用户时，如果这个用户开始创建数据库对象（如创建表、视图等），Oracle会自动为这个用户创建一个与用户同名的模式来存放这些对象。


在Oracle容器数据库（CDB）架构中，包含一个根容器（CDB$ROOT）和多个可插拔数据库（PDB）。公用用户是可以在CDB的公共部分或者多个PDB中访问的用户。这些用户用于管理跨越多个PDB的公共资源或执行通用的管理任务。创建公用用户的方式如下：

```sql
CREATE USER C##test_user IDENTIFIED BY test_password;
GRANT CREATE SESSION TO C##test_user;
```

创建本地用户则需要切换当前会话到指定的PDB中

```sql
-- 在容器数据库（CDB）中查看可插拔数据库（PDB）
SELECT name FROM v$pdbs;
-- 换到目标 PDB
ALTER SESSION SET CONTAINER = pdb1; 
-- PDB中创建本地用户
CREATE USER new_schema_user IDENTIFIED BY schema_password;
GRANT CREATE SESSION TO new_schema_user;
```

### 表空间

表空间是Oracle数据库中用于存储数据库对象（如表、索引等）的逻辑存储区域，后续创建的数据库对象可以指定存放在表空间里。

#### 表空间操作

创建表空间

```sql
CREATE TABLESPACE tablespace_name
DATAFILE 'datafile_path' SIZE size_value;
```

例如创建名为WORKHUB的表空间，并设置大小为50M
```sql
CREATE TABLESPACE "WORKHUB"
DATAFILE 'WORKHUB.dbf' SIZE 50M 
```

修改表所在空间

```sql
ALTER TABLE table_name MOVE TABLESPACE new_tablespace_name;
```

修改用户在特定表空间中的配额

```sql
ALTER USER user_name QUOTA new_quota_size ON tablespace_name;
```

表空间参数组合使用

```sql
CREATE TABLE schema_name.table_name (
    ...
)
TABLESPACE "WORKHUB" -- 定义了表空间的名称为 “WORKHUB”
LOGGING -- 启用日志记录功能
NOCOMPRESS -- 数据不进行压缩存储
PCTFREE 10 -- 每个数据块中，会预留 10% 的空间作为空闲区域
INITRANS 1 -- 每个数据块初始分配的事务入口数量为1
STORAGE ( -- 表空间存储相关参数的详细设置
  INITIAL 65536  -- 指定了表空间的初始大小为 65536 * 8KB = 512MB
  NEXT 1048576 -- 当表空间需要扩展时，每次扩展增加的大小为1048576 * 8KB = 8GB
  MINEXTENTS 1 -- 表示表空间最初创建时至少包含的扩展次数为1
  MAXEXTENTS 2147483645 -- 表空间最多可以扩展的次数为无限
  BUFFER_POOL DEFAULT -- 指定该表空间使用默认的缓冲池
)
PARALLEL 1 -- 表示对这个表空间中数据的并行处理程度，最多可以启用 1 个并行执行服务器来协助处理任务
NOCACHE -- 指定了该表空间中的数据块在被读取到内存（缓冲池）后，不会被缓存起来用于后续的重复访问
DISABLE ROW MOVEMENT -- 禁止行移动功能
;
```

## PL/SQL语句

### 概述

PL/SQL（Procedural Language/Structured Query Language）是Oracle数据库系统的过程化编程语言。它是一种块结构语言，将 SQL语句的强大数据处理能力与过程化编程语言的流程控制结构相结合。这使得开发人员可以在数据库内部编写复杂的业务逻辑，而不仅仅是执行简单的查询操作。由于PL/SQL程序是在数据库服务器内部执行，减少了数据在客户端和服务器之间的传输，从而提高了性能。特别是对于复杂的数据库操作和大量的数据处理，这种优势更加明显。

### 基本结构

#### 块结构

PL/SQL程序由块（Block）组成，每个块都有一个声明部分、执行部分和可选的异常处理部分。声明部分用于定义变量、常量、游标等；执行部分包含了要执行的SQL语句和PL/SQL语句，用于实现具体的业务逻辑；异常处理部分用于处理程序执行过程中可能出现的错误。例如：

```sql
DECLARE
    -- 声明部分，定义变量
    v_count NUMBER;
BEGIN
    -- 执行部分，查询并赋值
    SELECT COUNT(*) INTO v_count FROM employees;
    DBMS_OUTPUT.PUT_LINE('员工总数为：' || v_count);
EXCEPTION
    -- 异常处理部分，处理可能的错误
    WHEN NO_DATA_FOUND THEN
        DBMS_OUTPUT.PUT_LINE('未找到数据。');
END;
```

#### 控制结构

条件语句包括IF - THEN - ELSE语句用于根据条件执行不同的代码块。例如：

```sql
IF condition THEN
    -- 条件为真时执行的语句
ELSIF another_condition THEN
    -- 另一个条件为真时执行的语句
ELSE
    -- 所有条件为假时执行的语句
END IF;
```

循环语句有LOOP、WHILE - LOOP和FOR - LOOP等多种循环结构。例如，使用FOR - LOOP来遍历一个查询结果集

```sql
FOR i IN (SELECT column_name FROM table_name) LOOP
    -- 对每一行数据进行操作
END LOOP;
```

### 存储过程

存储过程：是一组预编译的PL/SQL语句，存储在数据库中，可以被调用以执行特定的任务。存储过程可以接受参数，并且可以包含复杂的业务逻辑和数据库操作。例如，使用存储过程向employees表中插入一条新员工记录

```sql
CREATE OR REPLACE PROCEDURE insert_employee(
    p_name VARCHAR2,
    p_salary NUMBER
) AS
BEGIN
    INSERT INTO employees (name, salary) VALUES (p_name, p_salary);
    COMMIT;
END;
```

### 存储函数

存储函数与存储过程类似，但函数必须返回一个值。函数可以用于计算并返回一个结果，这个结果可以在SQL语句中使用。例如，使用存储函数计算员工的年薪

```sql
CREATE OR REPLACE FUNCTION calculate_annual_salary(
    p_salary NUMBER
) RETURN NUMBER AS
BEGIN
    RETURN p_salary * 12;
END;
```

## 命令行工具

### SQL*Plus

SQL*Plus是Oracle数据库提供的一个命令行界面的工具，用于与Oracle数据库进行交互。它允许用户输入和执行SQL语句、PL/SQL块以及执行各种数据库管理和操作任务。比如查询数据、创建表、修改数据库对象结构等。

#### 连接数据库

方式一：使用Easy Connection Identifier连接

```shell
sqlplus system/root1234@"localhost:1521/FREEPDB1"
```

方式二：使用Full Connection Identifier连接。首先需要编辑tnsnames.ora文件(以23 ai个人免费版为例，对应目录为`C:\app\your-username\product\23ai\dbhomeFree\network\admin`)，添加以下配置

```
FREEPDB1 = 
 (DESCRIPTION=
   (ADDRESS=(PROTOCOL=TCP)(HOST=localhost)(PORT=1521))
   (CONNECT_DATA=
      (SERVICE_NAME=FREEPDB1)
    )
 )
```

```shell
sqlplus system/root1234@FREEPDB1
```

### SQLcl

SQLcl是Oracle推出的一款现代化的命令行工具，它是基于Java开发的，在功能上可以看作是SQL*Plus的增强版，提供了更加简洁易用、功能丰富的交互界面，并且融入了很多新的特性来提升开发和管理数据库的效率。

### 配置

SQLcl需要Java 11及以上版本的JDK，Oracle在使用过JDK后就会将JDK的配置写到配置文件中，若是Oracle的环境变量配置在JDK的变量前时将会被Oracle的配置信息加载覆盖掉。若遇到Java版本切换不生效的问题，可将PATH路径中的`C:\Program Files\Common Files\Oracle\Java\javapath`置于JDK变量之后。

SQL*Plus和SQLcl可执行文件一般位于`%ORACLE_HOME%/bin`目录下，以23 ai个人免费版为例，对应目录为`C:\app\your-username\product\23ai\dbhomeFree\bin`。

## 参考文档

[Oracle数据库官方文档](https://docs.oracle.com/en/database/oracle/index.html)

[Oracle数据库SQL语法参考(23ai)](https://docs.oracle.com/en/database/oracle/oracle-database/23/sqlrf/index.html)

[Oracle数据库命令行工具SQLcl使用参考](https://docs.oracle.com/en/database/oracle/sql-developer-command-line/24.1/sqcug/working-sqlcl.html)

[Oracle数据库创建示例模式(23ai)](https://docs.oracle.com/en/database/oracle/oracle-database/23/comsc/installing-sample-schemas.html)

[Oracle数据库SQL报错帮助](https://docs.oracle.com/en/error-help/db/)

