---
title: "MongoDB 高可用架构: 副本(ReplicaSet)/分片(Sharding)集群是什么回事"
date: 2024-11-22T17:00:29+08:00
tags: ["MongoDB", "高可用架构"]
categories: ["数据库", "理论"]
series: ["MongoDB 知识汇总"]                    
author: ["zhu733756"]
---

## 前戏

> **小白**：嗨，老花，我听说 MongoDB 挺火的，但我对它不是很了解。你能给我讲讲 MongoDB 是啥，它和 MySQL 这样的 SQL 数据库有啥不同吗？

**老花**：当然可以！MongoDB 是一种 NoSQL 数据库，它以文档存储的形式存储数据，这使得它在处理大量半结构化数据时非常灵活和高效。与 MySQL 这样的关系型数据库相比，MongoDB 不需要预定义的模式，支持动态字段，这对于快速发展和频繁变更的数据模型来说是一个很大的优势。

以下是 MySQL 和 MongoDB 进行增删改查操作的基本 SQL 语句和命令的对比：

### MySQL CRUD

```sql
INSERT INTO table_name (column1, column2, column3, ...)  VALUES (value1, value2, value3, ...);
DELETE FROM table_name WHERE condition;
UPDATE table_name SET column1 = value1, column2 = value2, ... WHERE condition;
SELECT column1, column2, ... FROM table_name WHERE condition;
```

### MongoDB CRUD

```javascript
db.collection_name.insert({
  field1: value1,
  field2: value2,
  field3: value3,
  ...
});

db.collection_name.insertMany([
  {
    field1: value1,
    field2: value2,
    field3: value3,
    ...
  },
  {
    field1: value1,
    field2: value2,
    field3: value3,
    ...
  },
  ...
]);

db.collection_name.remove({
  condition: value
});

db.collection_name.deleteMany({
  condition: value
});

db.collection_name.update(
  { <filter> },
  { $set: { <field1>: <value1>, <field2>: <value2>, ... } },
  { multi: <boolean> }
);

db.collection_name.find({
  field: value
});

```

在 MongoDB 中，`<filter>` 是查询条件，`<field1>` 和 `<value1>` 是要更新的字段和值，`multi` 参数是一个布尔值，如果设置为 `true`，则更新所有匹配的文档，否则只更新第一个匹配的文档。

请注意，MongoDB 的查询和更新操作使用了不同的操作符来指定条件和更新内容，例如 `$set` 用于指定更新字段的值。MongoDB 的查询语言非常强大，支持多种操作符来进行复杂的查询和数据聚合。

MongoDB 在处理大量非结构化数据、需要灵活性和可扩展性的场景下，还提供了丰富的查询语言和索引机制，使开发人员可以方便地进行数据查询和分析。而 MySQL 在需要强一致性和事务支持的应用场景下表现更优。

## MongoDB 的数据类型和编程特点

> **小白**：MongoDB 的数据类型和编程特点是怎样的？

**老花**：MongoDB 支持多种数据类型，包括字符串、数字、数组、对象等，这使得它能够存储复杂的数据结构。在编程方面，MongoDB 提供了多种语言的驱动程序，如 Python、Java、Node.js 等，这让开发者能够轻松地在他们选择的语言中使用 MongoDB。

## 正戏: MongoDB 高可用架构探索

> **小白**：听起来很不错！我们知道 Mysql 有主从复制, Redis 有哨兵和 Cluster 模式, 那 MongoDB 又有哪些高可用部署模式有哪些呢？

**老花**：MongoDB 主要有两种高可用部署模式：副本集和分片集。

### 副本集

> **小白**：副本集是什么？它是怎么工作的？

**老花**：副本集是一组维护相同数据集的 MongoDB 服务器。这样数据存储在多个节点上, 通过冗余存储来保障高可用, 这种方式还是挺常见的。

- 写请求在主节点上进行, 读请求可以负载均衡;
- 当主节点发生故障, 能选出新的主节点, 继续服务;
- 主节点通过 oplog 日志复制到从节点 。

#### Oplog（操作日志）

MongoDB 中的 Oplog 是一个特殊的有上限的集合，它保存所有修改数据库中存储的数据的操作的滚动记录。主节点（Primary）上应用数据库操作后，这些操作会被记录到主节点的 Oplog 上。从节点（Secondary）会以异步的方式复制并应用这些操作。

#### 数据同步过程

数据同步主要包含两个步骤：initial sync（全量同步）和 replication（增量同步）。Initial sync 用于在新节点加入副本集时，从源成员拷贝全量数据到目标成员。Replication 则是在 initial sync 之后，从主节点不断重放 Oplog 来同步增量数据
。

#### 主从同步机制

- 写操作在主节点: 当用户发送写操作（如插入、更新或删除）到主节点时，主节点处理这些操作并将它们记录在其 Oplog 中。
- Oplog 复制到从节点：从节点定期轮询主节点的 Oplog。Oplog 包含了所有写操作的顺序记录。从节点读取 Oplog 条目并将相同的操作应用到它们自己的数据集上，保持与主节点数据的一致性。
- 数据一致性：通过基于 Oplog 的复制，从节点逐渐与主节点的数据保持一致。这个过程确保了从节点上的数据与主节点上的数据保持一致。

### 分片集

> **小白**：分片集又是什么呢？

**老花**：分片集是一种分布式部署模式，有几种不同的节点角色：

- **分片服务器（Shard Server）**：一组存储实际数据的服务器的副本集, 一般由三个主从节点组成。一个分片集群上可以有多个分片, 每一个分片上的数据保持一致。
- **配置服务器（Config Server）**：一组存储集群的元数据和配置信息的副本集, 一般包含三个主从副本。
- **查询路由器（Mongos）**：客户端连接的接口，负责将请求路由到正确的分片服务器。

其实, 分片集就是把两个副本集串联起来了, 换汤不换药啦。

举个例子说明, 有一个两个分片的高可用分片集群:

- 2 个 Mongos 保证无状态查询路由器高可用
- 3 个 Config Server: 保证配置服务器高可用
- 2 个分片副本集 shard1 和 shard2, 每一个分片副本集都包含三个主从副本: shard1-{1,2,3} 和 shard2-{1,2,3}, 这两个副本集的主从节点各自同步数据

假设现在有 100G 的数据, 开启分片后, 通过 mongos 写入数据, 比较均匀的话, 就会两个分片副本集各自 50Gi 左右。

这就是数据分片的核心思想了。

需要注意的是, 分片集群并不是配置了就开启了分片, 老花在使用的 4.0.3 和 4.2.1 都是需要手动对数据库开启数据分片的哦!

```
mongos> use admin
mongos> db.runCommand( { enablesharding : "test" } )
mongos> use test
mongos> db.vast.ensureIndex( { id: 1 } )
mongos> use admin
mongos> db.runCommand( {shardcollecteion: "test.vast", key : {id: 1} } )
mongos> use test
mongos> for(i=0;i<20000;i++){ db.vast.insert({"id":i,"name":"woshishuaige","age":i,"date":new Date()}); }
mongos> db.vast.stats()
```

> **小白**：那么问题来了, 怎么保证数据的均匀?

**老花**：问的好! 这不得不说分片键这个概念了，他就是用来干这个活的! 常用的分片键主要有[范围分片](https://www.mongodb.com/docs/manual/core/ranged-sharding/)和[哈希分片](https://www.mongodb.com/docs/manual/images/sharding-hash-based.bakedsvg.svg), 都是顾名思义, 你懂的。

构建哈希分片:
```
sh.shardCollection( "database.collection", { <field> : "hashed" } )
```

构建范围分片:
```
sh.shardCollection( "database.collection", { <shard key> } )
```

使用范围去分数据是可能导致数据倾斜的, 一般来说, 使用哈希函数去分数据就相对比较均衡点了，但也不是绝对的哈。均衡之道, 存乎一心。这个过程就是: mongos 问 config server 拿到分片的元数据和分片键, 就知道往哪个分片写了, 找到这个分片的主节点, 开始灌水就是。

## 后戏

> **小白**：现在我对 MongoDB 有了更深的理解，那接下来你会介绍什么呢？

**老花**：接下来的文章，我会详细介绍如何部署这些高可用集群，包括具体的配置步骤和最佳实践。这样你就可以自己动手搭建一个高可用的 MongoDB 集群了！

> **小白**：太好了，我已经迫不及待想要学习如何部署了！谢谢你，老花！

**老花**：不客气啦！记得关注下一篇文章哦！
