---
title: 文档型数据库MongoDB使用介绍
date: 2025-11-10 13:17:44
categories:
  - Database
tags:
  - MongoDB
  - NoSQL
---

MongoDB文档型NoSQL数据库中的使用特性、适用场景，以及命令行工具Mongo Shell、操作符表达式等使用介绍。

<!--more-->

## 前言

MongoDB是一个开源的、面向文档的NoSQL数据库，它在设计上与传统的关系型数据库（如 MySQL、Oracle）有很大不同。它使用类似JSON的文档模型存储数据，将数据存储为BSON（Binary JSON）格式的"文档"，而不是行和列。这种格式使得数据存储非常灵活，表达非常自然和强大。MongoDB旨在为WEB应用提供可扩展的高性能数据存储解决方案。MongoDB是一个介于关系数据库和非关系数据库之间的产品，是非关系数据库当中功能最丰富，最像关系数据库的。

## 使用特性

MongoDB与关系型数据库的核心概念对比如下:

| 概念 | MongoDB（文档数据库） | 关系型数据库（如 MySQL） | 
| ----------- | ----------- | ----------- |
| 数据库 | Database | Database | 
| 表/集合 | Collection | Table | 
| 行/文档 | Document | Row | 
| 列/字段 | Field | Column | 
| 主键 | _id（默认自动创建） | Primary Key | 
| 索引 | index | index | 
| 表关联 | 引用（$lookup）或嵌入式文档 | Join（表连接） | 
| Schema | 动态模式（无模式） | 预定义，严格的模式 |


MongoDB与传统关系型数据库相比，具有以下优势：
- 灵活的文档模式（无模式）：同一个集合（Collection）中的文档（Document）不需要具有相同的结构（字段），每个文档可以有自己的独特字段。这对于快速迭代的开发非常友好，可以在不关闭数据库的情况下，随时为数据添加新字段，极大地提高了开发效率。
- 现代化的存储格式：文档以一种类似于JSON的格式存储，称为BSON。它支持丰富的数据类型，如字符串、数字、日期、数组，甚至嵌套的其他文档。这种格式对于现代编程语言来说非常自然，减少了数据在应用层和数据库层之间转换的复杂性。
- 强大的查询语音：MongoDB提供了丰富的查询操作符，用于执行读取、更新、删除和聚合操作。它几乎能实现SQL的所有功能，语法同样强大且直观。
- 高性能优化方案：支持多种索引（单字段、复合、多键、地理空间、全文等），可以极大地加快查询速度。支持内存映射，将内存管理工作交给操作系统，利用系统缓存来优化性能。提供强大的聚合框架，可以对数据进行复杂的转换和分析，类似于SQL中的 GROUP BY和聚合函数，但更灵活。
- 高可用性与可扩展性：MongoDB通过复制集提供高可用性。一个复制集由多个节点组成（通常一主多从），主节点故障时，从节点会自动选举出新的主节点，确保服务不中断。当数据量巨大或吞吐量要求极高时，MongoDB支持分片（横向扩展）。它将一个集合的数据分布式地存储在多台机器（分片）上，形成一个数据库集群，从而突破单机性能瓶颈。


## 适用场景

MongoDB的文档模型和无模式特性适用于以下场景：
- 内容管理系统：文章、评论、标签等数据非常适合用文档模型来存储。
- 社交网络：用户档案、动态、好友列表等，其中包含大量非结构化和半结构化数据。
- 物联网：海量的设备传感器数据，通常是以时间序列的形式写入，MongoDB能够高效地处理这种写入密集型操作。
- 实时分析：聚合管道非常适合用于生成实时报表和分析数据。
- 产品目录：不同品类的商品拥有完全不同的属性，MongoDB的无模式特性可以完美应对这种多变性。
- 移动应用：其灵活的schema非常适合快速迭代的移动应用后端。

## 数据库连接

一个标准的MongoDB连接字符串格式如下：

```
mongodb://[username:password@]host1[:port1][,host2[:port2],...[,hostN[:portN]]][/[database][?options]]
```

示例：

- 本地无认证方式

```
mongodb://localhost:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.5.9
```

- 带认证的数据库

```
mongodb://myUser:myPassword@localhost:27017/myAppDatabase
```

- 复制集（集群）

```
mongodb://host1:27017,host2:27017,host3:27017/myAppDatabase?replicaSet=myReplicaSet
```

- Atlas 云数据库

```
mongodb+srv://myUser:myPassword@cluster0.abcde.mongodb.net/myAppDatabase
```

### GUI方式

MongoDB Compass是MongoDB官方提供的免费、图形化界面（GUI）工具，用于与MongoDB数据库进行交互。

### CLI方式

MongoDB Shell是MongoDB提供的命令行工具，允许用户与MongoDB数据库进行交互、执行命令和操作数据库。下载完MongoCompass后可前往[MongoDB Shell下载地址](https://www.mongodb.com/try/download/shell)自行下载。

#### 常用命令

启动Shell

`mongosh`

此时会连接到本地MongoDB服务器，连接成功后，可以执行各种MongoDB数据库操作

```shell
test> show dbs # 显示数据库列表
test> use <database_name> # 切换到指定数据库
<database_name> > db.<collection_name>.insertOne({ ... }) # 向指定集合插入文档
<database_name> > db.<collection_name>.find() # 查询指定文档
<database_name> > db.<collection_name>.updateOne({ ... }) # 向指定集合更新文档
<database_name> > db.<collection_name>.deleteOne({ ... }) # 向指定集合删除文档
```

退出Shell

```shell
<database_name> > quit
```

示例：

- 插入文档

```shell
mydatabase> db.mycollection.insertOne({ name: "Alice", age: 30 })
{
  acknowledged: true,
  insertedId: ObjectId('69118ecdae2964f29e63b112')
}
```

- 查询文档

```shell
mydatabase> db.mycollection.find()
[
    { _id: ObjectId('667cd8789a69705686ed70f2'), name: 'Alice', age: 31 }
]
```

- 更新文档

```shell
mydatabase> db.mycollection.updateOne({ name: "Alice" }, { $set: { age: 31 } })
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}
```

- 删除文档

```shell
mydatabase> db.mycollection.deleteOne({ name: "Alice" })
{ 
    acknowledged: true, deletedCount: 1 
}
```

## 操作符表达式

MongoDB的操作符表达式是其查询语言的核心，可以构建复杂且强大的查询、更新、聚合等操作。这些操作符以$符号开头。

### 查询操作符

用于find()、update()等方法中，定位符合条件的文档。

- 比较操作符：例如\$eq、\$gt、\$lt、\$in等

```shell
db.<collection_name>.find({ age: { $lt: 25 } })
```

- 逻辑操作符：例如\$and、\$or、\$not、\$nor

```shell
db.<collection_name>.find({ $and: [ { status: "A" }, { age: { $lt: 30 } } ] })
```

- 元素操作符：例如\$exists(匹配具有指定字段的文档)、\$type(匹配字段是指定 BSON 类型的文档)

```shell
db.<collection_name>.find({ phone: { $exists: true } })
```

- 数组操作符：\$all(匹配包含数组中指定的所有元素的数组)、\$elemMatch(匹配数组中的元素能同时满足所有指定条件的)、\$size(匹配数组大小等于指定值的文档)

```shell
db.<collection_name>.find({ scores: { $elemMatch: { $gt: 80， $lt: 90 } } })
```

### 更新操作符

用于update()和findAndModify()等方法中，修改文档的内容。

- 字段更新操作符：例如\$set(设置字段的值)、\$unset(删除指定字段)、\$rename(重命名字段)、\$inc(将字段的值增加指定的数量)等

```shell
db.<collection_name>.update({}, { $set: { status: "A"， modified: ISODate() } })
```

- 数组更新操作符：例如\$\[\<identifier>](过滤后的位置操作符，与arrayFilters一起使用，更新所有符合过滤条件的数组元素)、\$pop、\$pull、\$push

```shell
db.<collection_name>.update({}, { $set: { “grades.$[elem].score”: 100 } }, { arrayFilters: [ { “elem.score”: { $lte: 80 } } ] })
```

### 聚合管道操作符

用于aggregate()方法中，对数据进行转换和计算。它们在聚合管道的各个阶段中使用。

- 阶段操作符：例如\$match、\$group、\$sort、\$limit、\$skip、\$lookup(对同一数据库中的另一个集合执行左外连接)、\$project(重塑文档流，例如添加、删除、重命名字段，类似于投影)

- 表达式操作符(常用于\$group)：\$sum、\$avg、\$firt、\$last、\$min、\$max

示例：获取所有用户，并列出他们发表的所有文章。

```js
db.users.aggregate([
  {
    $lookup: {
      from: "posts",
      localField: "_id",       // users 的 _id
      foreignField: "authorId", // posts 的 authorId
      as: "userPosts"          // 这个数组会包含用户的所有文章
    }
  },
  {
    $project: {
      name: 1,
      email: 1,
      "userPosts.title": 1,    // 只投影出文章的 title 和 date
      "userPosts.createdAt": 1
    }
  }
])
```

## 参考文档

- [MongoDB官方中文文档](https://www.mongodb.com/zh-cn/docs/manual/)

- [MongoDB Shell使用参考](https://www.mongodb.com/zh-cn/docs/mongodb-shell/)