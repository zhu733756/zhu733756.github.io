---
title: 'MongoDB备份与恢复实战(一)'
tags: ['MongoDB', '高可用架构', '全量备份']
categories: ['数据库', '实战', '备份与恢复', '分布式']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
date: 2024-12-13T17:45:36+08:00
---

## 前戏

> 小白: 嘿, 老花。今天能聊聊 MongoDB 的数据备份与恢复, 特别是涉及到用户和用户权限数据的部分。这可是个技术活儿。

老花: 好的, 在回答你的问题前, 我们先思考几个问题。

## 关于备份恢复的几个问题

我们先搞清楚要备份的数据有哪些?

- 生产数据
- 用户
- 用户权限

备份是追求准确性还是性能?

- 从`master`备份, 可能会影响写数据, 但数据是全的
- 从`slave`备份, 不影响读写, 单如果主从同步延迟, 可能会丢数据

备份的集群模式是分片还是副本?

- 分片模式: 需要从多个节点上同时备份, 以提高性能, 注意, 需要备份`configsvr`。
- 副本模式: 只需要考虑从一个节点上备份。

要做增量备份还是全量备份?

恢复到其他实例? 管理员账号要调过还是覆盖? (这个只能业务处理了)

## 备份恢复工具简介

### mongodump

`mongodump`是官方提供的一个备份工具, 它支持一些参数:

- -h, --host：这个参数用来指定远程 MongoDB 数据库的地址。如果你不设置这个参数, mongodump 默认会尝试连接到本地的 MongoDB 实例。
- --port：这个参数用来指定远程 MongoDB 数据库的端口号。默认情况下, mongodump 会连接到端口 27017, 这是 MongoDB 的默认端口。
- -u, --username：如果你的数据库设置了认证, 这个参数就非常重要了。它用来指定连接远程数据库的用户名。
- -p, --password：与用户名相对应, 这个参数用来输入对应用户名的密码, 以验证身份。
- -d, --db：这个参数让你指定想要备份的特定数据库。
- -c, --collection：如果你只想备份数据库中的某个特定集合, 这个参数就能派上用场。
- -o, --out：这个参数用来指定备份文件的输出目录。备份完成后, 所有数据都会存储在这个目录下。
- -q, --query：这个参数允许你指定查询条件, 从而只备份满足特定条件的数据。
- -j, --numParallelCollections：这个参数用来设置并行转储的集合数, 可以提高备份效率。默认情况下, mongodump 会并行转储 4 个集合。
- --gzip：如果你想要节省存储空间, 可以使用这个参数。它会将备份文件压缩成 gzip 格式。
- --oplog：这个参数允许你使用 oplog（操作日志）进行时间点快照, 这对于需要精确到特定时间点的数据备份非常有用。`
- --authenticationDatabase：这个参数用来指定进行用户认证的数据库。有时候, 认证数据库和你要备份的数据库不是同一个。

#### 全量备份

全量备份有两个思路, 挨个`db`进行备份, 或者通过`--oplog`来可以顺带做`oplog`日志的备份。

#### 增量备份

`--query`和`--oplog`对于做增量备份十分有用。但要考虑的是, 可以根据时间过滤, 单这个时间点必须在`oplog`日志的覆盖范围内。

### mongorestore

`mongorestore`也是官方提供的一个恢复工具, 它的参数基本跟`mongodump`类似。值得一提的是, 区别`--oplog`, 这里提供了`--oplogReplay`来进行恢复。

#### 全量恢复

全量恢复有两个思路, 挨个`db`进行恢复, 或者通过`--oplogReplay`来可以顺带做`oplog`日志的备份。

#### 增量恢复

`--query`和`--oplogReplay`对于做增量恢复十分有用。

### 全量备份实战

进入一个`shard pod`:

```bash
$ kubectl -n mongodb-sharded exec -it mongodb-sharded-shard0-data-1 -- bash
```

#### 备份一个库的数据

```bash
$ mongodump -u root -p 123456 --authenticationDatabase admin --db test -o /tmp/
2024-12-13T10:28:17.339+0000    writing test.vast to /tmp/test/vast.bson
2024-12-13T10:28:17.346+0000    done dumping test.vast (2 documents)

$  ls /tmp/test/ -al
total 16
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:28 .
drwxrwsrwx 3 root 1001 4096 Dec 13 10:28 ..
-rw-r--r-- 1 1001 1001   58 Dec 13 10:28 vast.bson
-rw-r--r-- 1 1001 1001  171 Dec 13 10:28 vast.metadata.json

$ bsondump /tmp/test/vast.bson
{"_id":{"$oid":"674d1f2dce510a83fafe6911"},"d":{"$numberInt":"1"}}
{"_id":{"$oid":"674d1fd2ce510a83fafe6912"},"x":{"$numberInt":"2"}}
2024-12-13T10:31:31.310+0000    2 objects found
```

#### 备份全库(包括用户和权限)

```bash
mongodump -u root -p 123456 --authenticationDatabase admin  --oplog  -o /tmp/
2024-12-13T10:32:44.277+0000    writing admin.system.users to /tmp/admin/system.users.bson
2024-12-13T10:32:44.281+0000    done dumping admin.system.users (1 document)
2024-12-13T10:32:44.281+0000    writing admin.system.roles to /tmp/admin/system.roles.bson
2024-12-13T10:32:44.283+0000    done dumping admin.system.roles (1 document)
2024-12-13T10:32:44.284+0000    writing admin.system.version to /tmp/admin/system.version.bson
2024-12-13T10:32:44.288+0000    done dumping admin.system.version (3 documents)
2024-12-13T10:32:44.289+0000    writing test.vast to /tmp/test/vast.bson
2024-12-13T10:32:44.297+0000    done dumping test.vast (2 documents)
2024-12-13T10:32:44.301+0000    writing captured oplog to
2024-12-13T10:32:44.313+0000            dumped 1 oplog entry

$ ls /tmp/ -al
total 20
drwxrwsrwx 4 root 1001 4096 Dec 13 10:32 .
drwxr-xr-x 1 root root 4096 Dec 12 01:13 ..
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:32 admin
-rw-r--r-- 1 1001 1001  103 Dec 13 10:32 oplog.bson
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:32 test

$ ls /tmp/admin/ -al
total 32
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:32 .
drwxrwsrwx 4 root 1001 4096 Dec 13 10:32 ..
-rw-r--r-- 1 1001 1001  159 Dec 13 10:32 system.roles.bson
-rw-r--r-- 1 1001 1001  297 Dec 13 10:32 system.roles.metadata.json
-rw-r--r-- 1 1001 1001  563 Dec 13 10:32 system.users.bson
-rw-r--r-- 1 1001 1001  297 Dec 13 10:32 system.users.metadata.json
-rw-r--r-- 1 1001 1001  530 Dec 13 10:32 system.version.bson
-rw-r--r-- 1 1001 1001  181 Dec 13 10:32 system.version.metadata.json

$ bsondump /tmp/admin/system.users.bson
{"_id":"admin.root","userId":{"$binary":{"base64":"ExJtYuIXSregLHw4+YZoOg==","subType":"04"}},"user":"root","db":"admin","credentials":{"SCRAM-SHA-1":{"iterationCount":{"$numberInt":"10000"},"salt":"QShjpsVWB2TumHDd0r5nAg==","storedKey":"1Wjjeih24FfYqCjL1wAZd6Hpw34=","serverKey":"fJZDV6zXe7CuokX2UnjDuH30EKc="},"SCRAM-SHA-256":{"iterationCount":{"$numberInt":"15000"},"salt":"FRuQe4o4EG0HHRARFAXSu05J9Ey5dMupkTrGzQ==","storedKey":"zaxpklLF7c97ScV1uq7BYLxLnik0Dek/NdclOfSXbhA=","serverKey":"CiHJkzGcoHMmjVpL8C7al3GFjiytAaWBMLE+aVgmlUs="}},"roles":[{"role":"sysadmin","db":"admin"},{"role":"root","db":"admin"}]}
```

#### 只备份用户和角色

```bash
 mongodump -u root -p 123456 --authenticationDatabase admin --db admin  -o /tmp/
```

### 全量恢复实战

#### 恢复一个库的数据

```bash
$ mongorestore -u root -p 123456 --authenticationDatabase admin --db test /tmp/test/
2024-12-13T10:40:20.830+0000    The --db and --collection flags are deprecated for this use-case; please use --nsInclude instead, i.e. with --nsInclude=${DATABASE}.${COLLECTION}
2024-12-13T10:40:20.830+0000    building a list of collections to restore from /tmp/test dir
2024-12-13T10:40:20.830+0000    reading metadata for test.vast from /tmp/test/vast.metadata.json
2024-12-13T10:40:20.831+0000    restoring to existing collection test.vast without dropping
2024-12-13T10:40:20.831+0000    restoring test.vast from /tmp/test/vast.bson
2024-12-13T10:40:20.844+0000    finished restoring test.vast (0 documents, 0 failures)
2024-12-13T10:40:20.844+0000    Failed: test.vast: error restoring from /tmp/test/vast.bson: (NotWritablePrimary) not primary
2024-12-13T10:40:20.844+0000    0 document(s) restored successfully. 0 document(s) failed to restore
```

看来需要找主节点才会恢复:

```bash
mongosh > rs.isMaster().primary
mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017
```

这个节点已经是主节点, 看来只能使用`--host`:

```bash
$ mongorestore --host mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017  -u root -p 123456 --authenticationDatabase admin  --drop --db test /tmp/test/

2024-12-13T10:48:13.753+0000    The --db and --collection flags are deprecated for this use-case; please use --nsInclude instead, i.e. with --nsInclude=${DATABASE}.${COLLECTION}
2024-12-13T10:48:13.754+0000    building a list of collections to restore from /tmp/test dir
2024-12-13T10:48:13.754+0000    reading metadata for test.vast from /tmp/test/vast.metadata.json
2024-12-13T10:48:13.756+0000    dropping collection test.vast before restoring
2024-12-13T10:48:13.773+0000    restoring test.vast from /tmp/test/vast.bson
2024-12-13T10:48:13.859+0000    finished restoring test.vast (2 documents, 0 failures)
2024-12-13T10:48:13.859+0000    no indexes to restore for collection test.vast
2024-12-13T10:48:13.859+0000    2 document(s) restored successfully. 0 document(s) failed to restore.
```

`--drop`会删除重复的文档, 在某些场景有用

#### 全量恢复所有数据

这里`admin`和`test`数据库都在, 我们并不需要使用`--oplogReplay`

```bash
$ ls /tmp/ -al
total 16
drwxrwsrwx 4 root 1001 4096 Dec 13 10:39 .
drwxr-xr-x 1 root root 4096 Dec 12 01:13 ..
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:36 admin
drwxr-sr-x 2 1001 1001 4096 Dec 13 10:39 test

$ mongorestore --host mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017  -u root -p 123456 --authenticationDatabase admin --drop  /tmp/
2024-12-13T10:49:59.440+0000    preparing collections to restore from
2024-12-13T10:49:59.451+0000    reading metadata for test.vast from /tmp/test/vast.metadata.json
2024-12-13T10:49:59.453+0000    dropping collection test.vast before restoring
2024-12-13T10:49:59.509+0000    restoring test.vast from /tmp/test/vast.bson
2024-12-13T10:49:59.548+0000    finished restoring test.vast (2 documents, 0 failures)
2024-12-13T10:49:59.548+0000    restoring users from /tmp/admin/system.users.bson
2024-12-13T10:49:59.617+0000    restoring roles from /tmp/admin/system.roles.bson
2024-12-13T10:49:59.730+0000    no indexes to restore for collection test.vast
2024-12-13T10:49:59.730+0000    2 document(s) restored successfully. 0 document(s) failed to restore.
```

#### 全量恢复用户和角色

```bash
$ mongorestore --host mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017  -u root -p 123456 --authenticationDatabase admin --drop --db admin  /tmp/admin
2024-12-13T11:02:45.039+0000    The --db and --collection flags are deprecated for this use-case; please use --nsInclude instead, i.e. with --nsInclude=${DATABASE}.${COLLECTION}
2024-12-13T11:02:45.039+0000    building a list of collections to restore from /tmp/admin dir
2024-12-13T11:02:45.042+0000    restoring users from /tmp/admin/system.users.bson
2024-12-13T11:02:45.067+0000    restoring roles from /tmp/admin/system.roles.bson
2024-12-13T11:02:45.100+0000    0 document(s) restored successfully. 0 document(s) failed to restore.
```

### 增量备份恢复实战

我们给`test2`插入一条数据:

```bash
test2> db.vast.insert({dd:1})
DeprecationWarning: Collection.insert() is deprecated. Use insertOne, insertMany, or bulkWrite.
{
  acknowledged: true,
  insertedIds: { '0': ObjectId('675c14d9206b963152fe6911') }
}
```

我们现在增量备份这条数据:
```bash
$ rm -rf /tmp/*
$ mongodump   -u root   -p 123456   --host mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017   --authenticationDatabase admin   -d local   -c "oplog.rs"   --query '{"ts": {"$gte":{"$timestamp":{"t":1734090347,"i":1}}}}'   --out /tmp/
2024-12-13T11:33:58.039+0000    writing local.oplog.rs to /tmp/local/oplog.rs.bson
2024-12-13T11:33:58.044+0000    done dumping local.oplog.rs (1 document)
```

如何获取oplog的起始时间:
```bash
rs.printReplicationInfo()
actual oplog size
'4047.9306640625 MB'
---
configured oplog size
'4047.9306640625 MB'
---
log length start to end
'1333042 secs (370.29 hrs)'
---
oplog first event time
'Thu Nov 28 2024 01:28:25 GMT+0000 (Coordinated Universal Time)'
---
oplog last event time
'Fri Dec 13 2024 11:45:47 GMT+0000 (Coordinated Universal Time)'
---
now
'Fri Dec 13 2024 11:45:49 GMT+0000 (Coordinated Universal Time)'
```

## 小尾巴

老花: 本期就将这么多了, 接下来我们继续讲这个话题~