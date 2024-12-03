---
title: '深入了解MongoDB日志系统'
tags: ['MongoDB', '分片集群', '日志轮转', 'oplog']
categories: ['数据库', '日志管理', '理论']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
date: 2024-12-03T08:55:11+08:00
---

## 前戏

> 小白: 嘿，老花，我最近在研究 MongoDB 的日志系统，你能给我介绍一下 MongoDB 有哪些日志吗？

老花: 当然可以，小白。MongoDB 主要有四种日志：系统日志、Journal 日志、主从日志（oplog）和慢查询日志
。让我一一解释给你听。

首先, 我们看下不同组件的配置文件描述信息。

## 不同组件的配置文件

### shardsvr

下面我们演示从之前部署的集群上获取一个数据节点`shardsvr`的启动配置:

```bash
$ kubectl -n mongodb-sharded exec -it mongodb-sharded-shard0-data-0 -- bash

$ ps -ef
UID          PID    PPID  C STIME TTY          TIME CMD
1001           1       0  2 Nov30 ?        00:46:47 /opt/bitnami/mongodb/bin/mongod --config=/opt/bitnami/mongodb/conf/mongodb.conf

$ cat /opt/bitnami/mongodb/conf/mongodb.conf
# mongod.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where and how to store data.
storage:
  dbPath: /bitnami/mongodb/data/db
  directoryPerDB: false

# where to write logging data.
systemLog:
  destination: file
  quiet: false
  logAppend: true
  logRotate: reopen
  path: /opt/bitnami/mongodb/logs/mongodb.log
  verbosity: 0

# network interfaces
net:
  port: 27017
  unixDomainSocket:
    enabled: true
    pathPrefix: /opt/bitnami/mongodb/tmp
  ipv6: false
  bindIpAll: true
  #bindIp:

# replica set options
replication:
  replSetName: mongodb-sharded-shard-0
  enableMajorityReadConcern: true

# sharding options
sharding:
  clusterRole: shardsvr

# process management options
processManagement:
   fork: false
   pidFilePath: /opt/bitnami/mongodb/tmp/mongodb.pid

# set parameter options
setParameter:
   enableLocalhostAuthBypass: false

# security options
security:
  authorization: enabled
  keyFile: /opt/bitnami/mongodb/conf/keyfile
```

这个 MongoDB 配置文件包含了多个部分，每个部分都用于设置 MongoDB 实例的不同配置选项。以下是对每个部分的详细分析：

#### 存储设置 (`storage`)

- `dbPath`: 数据库文件存储路径，设置为`/bitnami/mongodb/data/db`。
- `directoryPerDB`: 是否为每个数据库创建一个单独的目录。这里设置为`false`，意味着所有数据库文件都存储在`dbPath`指定的目录下。

#### 日志设置 (`systemLog`)

- `destination`: 日志输出目的地，设置为`file`，表示日志将写入文件。
- `quiet`: 设置为`false`，表示不启用安静模式，即会输出日志信息。
- `logAppend`: 设置为`true`，表示日志信息会追加到现有日志文件末尾，而不是覆盖。
- `logRotate`: 设置为`reopen`，表示在日志轮转时会重新打开日志文件。
- `path`: 日志文件路径，设置为`/opt/bitnami/mongodb/logs/mongodb.log`。
- `verbosity`: 日志详细程度，设置为`0`，这是默认的日志级别，只包括信息性消息。

#### 网络设置 (`net`)

- `port`: MongoDB 实例监听的端口，设置为`27017`。
- `unixDomainSocket`: 启用 UNIX 域套接字，设置为`true`，并指定路径前缀为`/opt/bitnami/mongodb/tmp`。
- `ipv6`: 是否启用 IPv6 支持，设置为`false`。
- `bindIpAll`: 设置为`true`，表示绑定到所有 IPv4 地址（`0.0.0.0`）。如果启动时启用了`net.ipv6`，则也会绑定到所有 IPv6 地址（`::`）。
- `bindIp`: 此设置被注释掉了，意味着将使用`bindIpAll`的设置。

#### 复制集设置 (`replication`)

- `replSetName`: 指定复制集的名称，设置为`mongodb-sharded-shard-0`。
- `enableMajorityReadConcern`: 设置为`true`，表示启用大多数读关注，这确保读取操作返回已提交的文档。

#### 分片设置 (`sharding`)

- `clusterRole`: 设置为`shardsvr`，表示这个 MongoDB 实例作为分片集群中的一个分片。

#### 进程管理设置 (`processManagement`)

- `fork`: 设置为`false`，表示不以守护进程模式运行。
- `pidFilePath`: 指定进程 ID 文件的路径，设置为`/opt/bitnami/mongodb/tmp/mongodb.pid`。

#### 参数设置 (`setParameter`)

- `enableLocalhostAuthBypass`: 设置为`false`，表示禁用本地主机认证绕过。

#### 安全设置 (`security`)

- `authorization`: 设置为`enabled`，表示启用角色基于访问控制。
- `keyFile`: 指定用于副本集成员之间认证的密钥文件路径，设置为`/opt/bitnami/mongodb/conf/keyfile`。

这个配置文件为 MongoDB 实例提供了一个全面的配置，包括存储、日志、网络、复制集、分片、进程管理和安全设置。这些设置确保了 MongoDB 实例可以按照预期的方式运行，并且可以在分片集群中作为分片进行操作。

### configsvr

```bash
$ kubectl -n mongodb-sharded exec -it mongodb-sharded-configsvr-2 -- bash
$ cat /opt/bitnami/mongodb/conf/mongodb.conf
# mongod.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where and how to store data.
storage:
  dbPath: /bitnami/mongodb/data/db
  directoryPerDB: false

# where to write logging data.
systemLog:
  destination: file
  quiet: false
  logAppend: true
  logRotate: reopen
  path: /opt/bitnami/mongodb/logs/mongodb.log
  verbosity: 0

# network interfaces
net:
  port: 27017
  unixDomainSocket:
    enabled: true
    pathPrefix: /opt/bitnami/mongodb/tmp
  ipv6: false
  bindIpAll: true
  #bindIp:

# replica set options
replication:
  replSetName: mongodb-sharded-configsvr
  enableMajorityReadConcern: true

# sharding options
sharding:
  clusterRole: configsvr

# process management options
processManagement:
   fork: false
   pidFilePath: /opt/bitnami/mongodb/tmp/mongodb.pid

# set parameter options
setParameter:
   enableLocalhostAuthBypass: false

# security options
security:
  authorization: enabled
  keyFile: /opt/bitnami/mongodb/conf/keyfile
```

除了`replSetName`和`clusterRole`不同外, 其余配置与`shardsvr`基本相同。

### mongos

```bash
$ kubectl -n mongodb-sharded exec -it mongodb-sharded-mongos-9cffc5c76-k7ps2  bash
$ cat /opt/bitnami/mongodb/conf/mongos.conf
# mongod.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where to write logging data.
systemLog:
  destination: file
  quiet: false
  logAppend: true
  logRotate: reopen
  path: /opt/bitnami/mongodb/logs/mongodb.log
  verbosity: 0

# network interfaces
net:
  port: 27017
  unixDomainSocket:
    enabled: true
    pathPrefix: /opt/bitnami/mongodb/tmp
  ipv6: false
  bindIpAll: true
  #bindIp:

# sharding options
sharding:
  configDB: mongodb-sharded-configsvr/mongodb-sharded-configsvr-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017

security:
  keyFile: /opt/bitnami/mongodb/conf/keyfile

# process management options
processManagement:
   fork: false
   pidFilePath: /opt/bitnami/mongodb/tmp/mongodb.pid

# set parameter options
setParameter:
   enableLocalhostAuthBypass: false
```

配置了`configDB`, 用于查询元数据信息和目标数据节点。

下面我们正式开始今天的主题:

## MongoDB 有哪些日志?

### 系统日志

这是`MongoDB`中非常重要的日志，它记录了 MongoDB 启动和停止的操作，以及服务器在运行过程中发生的任何异常或者调试信息。你可以通过在启动`mongod`时指定`logpath`参数来配置系统日志。

举个例子:

```yaml
systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log
```

这个配置定义了系统日志路径是`/var/log/mongodb/mongod.log`, 采用的方式是追加。

在某些低版本, 例如`4.0.3`, 你可能需要定期自行做日志轮转, 以避免单个日志文件长期膨胀, 从而更好地配合日志采集系统哦~

下面是老花常用的的定时脚本:

```bash
default_expiry=604800
default_sleep=300
max_file_size=104857600 # 100M
pwd=123456
port=27017

log_path="/var/log/mongo/mongod.log"
run_cnt=1

while true; do
  echo "[rotate_logs] Log rotation agent started at $(date +"%Y-%m-%d %H:%M:%S"), cnt: $run_cnt"
  enable_rotate_logs=false
  if [[ ! -f "$log_path" ]]; then
    echo "[rotate_logs] Log file not found: $log_path"
    sleep 5
    continue
  fi

  file_size=$(stat -c %s "$log_path")
  if ((file_size > max_file_size)); then
    enable_rotate_logs=true
  fi

  for file in "$log_path".*; do
    if [[ ! -f "$file" ]]; then
      continue
    fi

    last_modified=$(stat -c %Y "$file")
    now=$(date +%s)
    age=$((now - last_modified))
    if ((age >= expiry)); then
      rm "$file"
      echo "[rotate_logs] remove expiry file $file"
    fi
  done

  date_hm=$(date +%H:%M)
  if [[ "$enable_rotate_logs" = true ]]; then
    ret=$(mongo admin -u admin -p $pwd --port $port --quiet --eval "db.adminCommand( { logRotate : 1 } ).ok")
    if [[ "$ret" == "1" ]]; then
      echo "[rotate_logs] rotate logs finished"
    fi
  fi

  run_cnt=$(($run_cnt + 1))
  sleep $sleep_time
done
```

### Journal 日志

Journal 日志：这个日志功能是`MongoDB`中非常重要的一个功能，它保证了数据库服务器在意外断电、自然灾害等情况下数据的完整性。它通过预写式的`wal`日志为`MongoDB`增加了额外的可靠性保障。

```bash
$ cat /opt/bitnami/mongodb/conf/mongodb.conf
# mongod.conf
# for documentation of all options, see:
#   http://docs.mongodb.org/manual/reference/configuration-options/

# where and how to store data.
storage:
  dbPath: /bitnami/mongodb/data/db
  directoryPerDB: false
```

这样我们可以看到`/bitnami/mongodb/data/db/`数据目录的分布如下:

```bash
tree /bitnami/mongodb/data/db/
.
├── diagnostic.data
│   ├── ...
│   └── ...
├── journal
│   ├── ...
│   ├── ...
│   └── ...
├── local
│   ├── ....
│   └── ....
├── _mdb_catalog.wt  // mongodb元数据信息，包含集合和wt表的对应，文档创建参数和索引信息。
├── mongod.lock // mongod 实例启动的lock文件，防止多个实例读取一个文件
├── sizeStorer.wt // 存储占用空间信息，文档大小，文档数等信息，每次文档修改都会更新该文件的cache信息, 每1000次操作刷新一次。。
├── storage.bson
├── WiredTiger
├── WiredTigerLAS.wt
├── WiredTiger.lock
├── WiredTiger.turtle
└── WiredTiger.wt

$ ls -al journal/ -al
total 204808
drwx------ 2 1001 1001      4096 Nov 28 01:30 .
drwxr-xr-x 4 1001 1001      4096 Dec  2 02:03 ..
-rw------- 1 1001 1001 104857600 Dec  2 02:03 WiredTigerLog.0000000006
-rw------- 1 1001 1001 104857600 Nov 28 01:30 WiredTigerPreplog.0000000001
```

`WiredTigerLog.0000000006` 是`MongoDB WiredTiger`存储引擎使用的预写式日志`（write-ahead log，WAL）`文件的名称。这个文件名表示了日志文件的序列编号，用于标识日志文件。

日志文件命名规则：WiredTiger 的日志文件名前缀为 WiredTigerLog，紧接着是一个由十个数字组成的序列号，例如 WiredTigerLog.0000000006。这个序列号从 0000000001 开始，每次创建新的日志文件时递增。

序列号是唯一的，并且用于区分不同的日志文件。它表示日志文件的创建顺序，但并不直接关联到日志文件中记录的数据量或者日志文件的大小。这些日志文件记录了数据库的所有写操作，包括数据的插入、更新和删除等。如果 MongoDB 服务器发生故障，这些日志文件可以用于恢复未持久化到磁盘的数据，确保数据的完整性和一致性。WiredTiger 日志文件通常有一个大小限制（例如 100MB），当当前日志文件达到这个大小时，就会创建一个新的日志文件，文件名的序列号会增加，而旧的日志文件会被标记为可以进行归档或删除。

`WiredTigerLog.0000000006`文件就是第六个创建的日志文件，它包含了从上一个日志文件结束到当前文件满或达到指定条件之间的所有写操作记录。这些记录对于数据库的故障恢复至关重要。

`WiredTiger`会在以下条件下将缓冲的日志记录同步到磁盘：

- 每 100 毫秒一次（可以通过 storage.journal.commitIntervalMs 设置）。
- 当 WiredTiger 创建新的日志文件时。

### 主从日志（oplog）

`oplog` 是 `MongoDB` 中的一个固定集合，记录了所有数据库操作的日志，用于实现数据库的复制功能。这是副本集自行进行数据同步的重要组件。

`oplog`存储在`local`数据库的`oplog.rs`集合中。

```bash
$ mongosh admin -u root -p 123456

$ show databases;
admin   132.00 KiB
config  528.00 KiB
local   908.00 KiB
$ use local
switched to db local
$  db.oplog.rs.find().limit(1).pretty()
[
  {
    op: 'n',
    ns: '',
    o: { msg: 'initiating set' },
    ts: Timestamp({ t: 1732757305, i: 1 }),
    v: Long('2'),
    wall: ISODate('2024-11-28T01:28:25.240Z')
  }
]
```

如果报权限不够, 你可以试试创建超级管理员:

```
db.createRole({role:"sysadmin", roles:[], privileges:[{resource:{anyResource:true},actions:["anyAction"]}]});
db.updateUser("root",{roles:[{role:"sysadmin", db: "admin"},{role:"root", db: "admin"}]})
```

### 慢查询日志

慢查询日志记录了数据库的查询和索引使用情况，特别是那些执行时间较长的查询，可以帮助我们优化数据库性能。

我们可以在启动配置中进行配置:

```yaml
operationProfiling:
  mode: slowOp # 记录所有慢查询
  slowOpThresholdMs: 100 # 100毫秒作为慢查询的阈值
```

也可以设置运行时生效:

```bash
db.setProfilingLevel(1, { slowms: 200 })
db.getProfilingStatus()
```

在 MongoDB 中慢查询功能（Profiling）设置有三个级别，分别代表如下含义：

- 0：代表关闭，不收集任何慢查询
- 1：收集慢查询数据，默认收集超过 100 毫秒的慢查询
- 2：收集任何操作记录数据

注意, 以上配置是针对单个节点设置的, 如果要统一的话, 最好是每个节点逐个下发哦~

下面进行慢日志查询验证:

```bash
$ db.getProfilingStatus()
{
  was: 1,
  slowms: 0,
  sampleRate: 1,
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1733107668, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
      keyId: Long('0')
    }
  },
  operationTime: Timestamp({ t: 1733107667, i: 2 })
}

$ use test
$ db.vast.insert({x:2})
{
  acknowledged: true,
  insertedIds: { '0': ObjectId('674d1fd2ce510a83fafe6912') }
}

$ db.system.profile.find()
[
  {
    op: 'insert',
    ns: 'test.vast',
    command: {
      insert: 'vast',
      documents: [ { x: 2, _id: ObjectId('674d1fd2ce510a83fafe6912') } ],
      ordered: true,
      lsid: { id: UUID('2579214d-6a2e-4f26-926f-9785755cd458') },
      txnNumber: Long('2'),
      '$clusterTime': {
        clusterTime: Timestamp({ t: 1733107650, i: 1 }),
        signature: {
          hash: Binary.createFromBase64('AAAAAAAAAAAAAAAAAAAAAAAAAAA=', 0),
          keyId: 0
        }
      },
      '$readPreference': { mode: 'primaryPreferred' },
      '$db': 'test'
    },
    ninserted: 1,
    keysInserted: 1,
    numYield: 0,
    locks: {
      ReplicationStateTransition: { acquireCount: { w: Long('4') } },
      Global: { acquireCount: { r: Long('2'), w: Long('2') } },
      Database: { acquireCount: { r: Long('1'), w: Long('2') } },
      Collection: { acquireCount: { w: Long('2') } },
      Mutex: { acquireCount: { r: Long('7') } }
    },
    flowControl: { acquireCount: Long('1') },
    readConcern: { level: 'local', provenance: 'implicitDefault' },
    storage: { data: { txnBytesDirty: Long('864') } },
    responseLength: 241,
    protocol: 'op_msg',
    cpuNanos: 539222,
    millis: 0,
    totalOplogSlotDurationMicros: 151,
    ts: ISODate('2024-12-02T02:47:46.451Z'),
    client: '127.0.0.1',
    appName: 'mongosh 2.3.2',
    allUsers: [ { user: 'root', db: 'admin' } ],
    user: 'root@admin'
  }
]
```

## 后记

> 小白：MongoDB 日志系统看起来还是挺多的, 我得好好琢磨了~

老花：哈哈, 其实这些日志换其他的数据库产品也是都有的, 知识配置可能有差别哦~ 学习还是得注意方法论~

> 小白: 下一期, 咱们继续讲解什么知识? 

老花: 可能是监控, 可能是日志采集, 也可能是其他数据库产品哦~ 敬请期待!

