---
title: MySQL常用CLI命令及SQL语句
date: 2024-11-29 14:43:02
categories:
  - Database
tags:
  - MySQL
---

MySQL 数据库中的一些常用 SQL、CLI 命令，以及配置文件。

<!--more-->

## 前言

MySQL 采用多种存储引擎，如 InnoDB 和 MyISAM 等。InnoDB 是 MySQL 默认的存储引擎，支持事务处理、行级锁和外键约束。MyISAM 存储引擎则更侧重于性能，适合以读为主的应用场景。MySQL 在简单的查询操作和高并发的读场景下，性能表现较好，但在处理复杂的嵌套查询和大规模数据写入时，性能可能会受到一定影响。MySQL 广泛应用于互联网行业的中小型应用、网站开发和云计算环境，因为它易于安装、配置和维护，能够满足大多数网站的基本数据存储和查询需求，同时拥有庞大的用户社区和丰富的文档资源及第三方工具和插件支持。

## 数据库连接

.NET 项目配置文件中数据库连接字符串如下，

```json
{
  "ConnectionStrings": {
    "MysqlConnection": "Server=localhost;userid=root;password=your-password;database=your-database;port=3306;"
  }
}
```

## MySQL 常用 CLI 命令

### mysqld

mysqld，也称为 MySQL Server，是 MySQL 数据库系统中的核心组件。它是一个服务守护进程（daemon），负责管理数据库的访问和操作。在 Linux 系统中，服务通常以“d”结尾，代表守护进程。mysqld 作为服务器端程序，它处理来自客户端程序的网络连接请求，并管理对数据库的访问。

#### 常用命令

初始化 mysql 服务，初始化数据目录，但不生成随机密码(设置数据库空密码)，同时指定运行 mysqld 服务器的用户名为 root，端口号为 3306

`mysqld --initialize-insecure --user=root --port=3306 --console`

安装 mysql 服务，命名为 MySQL80，并设置默认配置文件

`mysqld --install MySQL80 --defaults-file=C:\Program Files\MySQL\MySQL Server 8.0\my.ini`

启动 MySQL80 服务

`net start ANWISE-MySQL`

#### 配置文件

my.ini 配置文件内容如下，

```ini
[mysql]
default-character-set=utf8mb4
# 配置免密登录 (可选)
user=root
password=your_passowrd
[mysqld]
default_authentication_plugin=mysql_native_password
port=3306
basedir=C:\Program Files\MySQL\MySQL Server 8.0
datadir=C:\Program Files\MySQL\MySQL Server 8.0\data
max_connections=200
character-set-server=utf8mb4
default-storage-engine=InnoDB
log_timestamps=SYSTEM
innodb_page_size=64K
max_allowed_packet=1G
# 配置sql_mode (可选)
sql_mode='STRICT_TRANS_TABLES,NO_ZERO_IN_DATE,NO_ZERO_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USER,NO_ENGINE_SUBSTITUTION'
```

### mysql

mysql 是 MySQL 自带的命令行客户端程序，用于交互式输入 SQL 语句或以批处理模式从文件执行它们。

#### 常用命令

登录 mysql

`mysql -u root -P 3306`

修改 mysql 密码

```shell
mysql>ALTER USER 'root'@'localhost'IDENTIFIED WITH mysql_native_password BY 'your_passowrd';
mysql>flush privileges;
mysql>quit;
```

查看所有数据库

```shell
mysql>show databases;
```

执行SQL文件(需要在控制台中将字符编码设置为UTF-8)

`chcp 65001`

`mysql -uroot -p[your_passowrd] -D[your_databse] < [your_script].sql`


## MySQL 常用 SQL

### CREATE/GRANT

CREATE 可用于创建库名和用户名，配合GRANT可以在创建用户的同时赋予权限。

```sql
-- 创建数据库名
CREATE DATABASE `demo` CHARACTER SET 'utf8mb4' COLLATE 'utf8mb4_unicode_ci';
-- 创建用户并赋予权限
CREATE USER test@'localhost' IDENTIFIED BY 'test';
GRANT ALL ON demo.* TO 'test'@'localhost';                
```

### SELECT

获取数据库中所有 TRUNCATE 语句

```sql
SELECT
	CONCAT('TRUNCATE TABLE ', table_schema, '.', TABLE_NAME, ';')
FROM
	INFORMATION_SCHEMA.TABLES
WHERE
	table_schema IN ('db_name');
```

### TRUNCATE/DROP/DELETE

TRUNCATE 只能操作表，将表中数据全部删除，在功能上和不带 WHERE 子句的 DELETE 语句相同，但是 TRUNCATE 会释放表空间，且不能回滚事务。TRUNCATE 一般会配合禁用/启动外键约束的语句使用。

```sql
-- 禁用外键约束
SET FOREIGN_KEY_CHECKS=0;
TRUNCATE TABLE [table_name]
-- 开启外键约束
SET FOREIGN_KEY_CHECKS=1;
```

DROP 将删除表的结构，以及被依赖的约束、触发器、索引。DROP 执行速度最快。

```sql
DROP TABLE [table_name]
```

DELETE 将表中数据全部删除，但是不会释放表空间，可以回滚事务。DELETE 执行速度最慢。

```sql
DELETE FROM TABLE [table_name]
```

### 存储函数

举例，对 int 类型的字段值进行转换

```sql
-- 创建存储函数
DELIMITER  //
DROP FUNCTION IF EXISTS convert_function //
CREATE FUNCTION convert_function([var_name] INT) RETURNS INT
BEGIN
	DECLARE res INT;
	IF [var_name]  = 100 THEN
		SET res = 0;
	END IF;
    RETURN res;
END;

-- 执行存储函数
SELECT convert_function(100);
```

### 存储过程

mysql支持对类型为json的字段值进行操作，例如修改类型为**json**的字段中的某个属性值，需要调用JSON_SET和JSON_EXTRACT

```sql
UPDATE [table_name] t
SET t.json = JSON_SET(t.json, '$.myProperty', convert_function(JSON_EXTRACT(t.json, '$.myProperty')));
```

举例，修改类型为**json数组**的字段中的某个属性值

```sql
-- 创建存储过程
DELIMITER //
DROP PROCEDURE IF EXISTS update_json_array //
CREATE PROCEDURE update_json_array()
BEGIN
    DECLARE done INT DEFAULT FALSE;
    DECLARE json_index INT DEFAULT 0;
    DECLARE json_length INT;
    DECLARE current_id char(36);
    DECLARE current_json JSON;
    DECLARE cursor_json CURSOR FOR
        SELECT id, json FROM [table_name];
    DECLARE CONTINUE HANDLER FOR NOT FOUND SET done = TRUE;

    OPEN cursor_json;
    read_loop: LOOP
        FETCH cursor_json INTO current_id, current_json;
        IF done THEN
            LEAVE read_loop;
        END IF;
        SET json_length = JSON_LENGTH(current_json);
        WHILE json_index < json_length DO
            -- 更新json数组中每个对象的属性值
            SET current_json = JSON_SET(current_json,
                CONCAT('$[', json_index, '].myProperty'), convert_function(JSON_EXTRACT(current_json, CONCAT('$[', json_index, '].myProperty'))));
            SET json_index = json_index + 1;
        END WHILE;

        -- 更新表中的记录
        UPDATE [table_name]
        SET files = current_json
        WHERE id = current_id;
        SET json_index = 0;
    END LOOP;
    CLOSE cursor_json;
END //
DELIMITER ;

-- 执行存储过程
CALL update_json_array();
```

### 全局参数修改

>Error Code: 1418. This function has none of DETERMINISTIC, NO SQL, or READS SQL DATA in its declaration and binary logging is enabled (you *might* want to use the less safe log_bin_trust_function_creators variable)

出现如上报错，需要修改`log_bin_trust_function_creators`的值为true，`log_bin_trust_function_creators`控制是否可以信任存储函数创建者，不会创建写入二进制日志引起不安全事件的存储函数。

```sql
SET global log_bin_trust_function_creators=TRUE;
```

## 参考文档

- [MySQL 官方中文文档](https://mysql.net.cn/doc/refman/8.0/en/)
