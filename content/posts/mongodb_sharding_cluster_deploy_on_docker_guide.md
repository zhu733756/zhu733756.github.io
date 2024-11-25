---
title: 'MongoDB分片集群Sharded Cluster容器化部署实战'
date: 2024-11-25T12:21:37+08:00
tags: ['MongoDB', '分片集群']
categories: ['数据库', '实战', 'docker']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
---

## 前戏

> 上回说了 MongoDB 的高可用架构有两种, 副本集合分片集群, 这一回我们来探索,分片集群怎么部署吧!

> 小白: 你好，老花！我想学习如何部署 MongoDB 的分片集群，你能帮帮我吗？

老花: 当然可以！MongoDB 的分片集群（Sharded Cluster）是一种分布式数据库架构，可以帮你处理大量数据和高吞吐量请求。接下来，我会一步步带你完成部署。

## docker-compose 部署实战

### 第一步：准备工作

在我们开始之前，你需要确保已经克隆了这个 GitHub 仓库，并且切换到了`with-keyfile-auth`这个文件夹。这个文件夹包含了我们需要的带认证的 docker-compose 配置文件。

```bash
git clone https://github.com/minhhungit/mongodb-cluster-docker-compose.git
cd mongodb-cluster-docker-compose/with-keyfile-auth
```

### 第二步：创建密钥文件

为了设置认证，我们需要一个密钥文件。我已经帮你创建了一个，但如果你自己想要创建一个，可以按照以下步骤操作：

在 Linux 上，你可以使用以下命令：

```bash
openssl rand -base64 756 > mongodb-keyfile
chmod 400 mongodb-keyfile
```

创建好`mongodb-keyfile`后，记得替换文件夹`with-keyfile-auth/mongodb-build/auth/`中的文件，然后进行下一步。

### 第三步：启动所有容器

更新或者安装 docker-compose:

```bash
pip install docker-compose -U
```

现在，我们来启动所有的 Docker 容器。切换到`with-keyfile-auth`目录下，然后运行：

```
docker-compose up -d
```

> Tip: 如果镜像无法拉取, 可以配置一些国内源:

```
cat /etc/docker/daemon.json
{
    "registry-mirrors": [
        "https://docker.m.daocloud.io",
        "https://dockerproxy.com",
        "https://docker.mirrors.ustc.edu.cn",
        "https://docker.nju.edu.cn"
    ]
}

docker pull docker.m.daocloud.io/mongo:6.0.2
docker tag docker.m.daocloud.io/mongo:6.0.2 mongo:6.0.2
docker-compose up -d
```

预期的输出如下, 可以看到目前我们已经启动三个分片副本集合,

```
docker-compose ps

Name              Command               State                   Ports
mongo-config-01   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27119->27017/tcp,:::27119->27017/tcp
mongo-config-02   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27120->27017/tcp,:::27120->27017/tcp
mongo-config-03   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27121->27017/tcp,:::27121->27017/tcp
router-01         docker-entrypoint.sh mongo ...   Up      0.0.0.0:27117->27017/tcp,:::27117->27017/tcp
router-02         docker-entrypoint.sh mongo ...   Up      0.0.0.0:27118->27017/tcp,:::27118->27017/tcp
shard-01-node-a   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27122->27017/tcp,:::27122->27017/tcp
shard-01-node-b   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27123->27017/tcp,:::27123->27017/tcp
shard-01-node-c   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27124->27017/tcp,:::27124->27017/tcp
shard-02-node-a   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27125->27017/tcp,:::27125->27017/tcp
shard-02-node-b   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27126->27017/tcp,:::27126->27017/tcp
shard-02-node-c   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27127->27017/tcp,:::27127->27017/tcp
shard-03-node-a   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27128->27017/tcp,:::27128->27017/tcp
shard-03-node-b   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27129->27017/tcp,:::27129->27017/tcp
shard-03-node-c   docker-entrypoint.sh mongo ...   Up      0.0.0.0:27130->27017/tcp,:::27130->27017/tcp
```

### 第四步：初始化副本集

接下来，我们需要初始化配置服务器和分片的副本集。运行以下命令：

```bash
docker-compose exec configsvr01 bash "/scripts/init-configserver.js"
docker-compose exec shard01-a bash "/scripts/init-shard01.js"
docker-compose exec shard02-a bash "/scripts/init-shard02.js"
docker-compose exec shard03-a bash "/scripts/init-shard03.js"
```

### 第五步：初始化路由器

等待几秒，让配置服务器和分片选举出主节点后，我们就可以初始化路由器了：

```bash
docker-compose exec router01 sh -c "mongosh < /scripts/init-router.js"
```

### 第六步：设置认证

现在，我们需要设置认证。默认的管理员账户是`your_admin`和`your_password`，你可以在`/scripts/auth.js`文件中修改它们。

```bash
#!/bin/bash
mongosh <<EOF
use admin;
db.createUser({user: "admin", pwd: "123456", roles:[{role: "root", db: "admin"}]});
exit;
EOF
```

运行以下命令来设置认证：

```bash
docker-compose exec configsvr01 bash "/scripts/auth.js"
docker-compose exec shard01-a bash "/scripts/auth.js"
docker-compose exec shard02-a bash "/scripts/auth.js"
docker-compose exec shard03-a bash "/scripts/auth.js"
```

### 第七步：启用分片并设置分片键

最后，我们需要启用分片并设置分片键。首先，使用以下命令连接到路由器：

```bash
docker-compose exec router01 mongosh --port 27017 -u "your_admin" --authenticationDatabase admin
```

然后，整个初始化就完成了!

接下来，你可以开始向集群中插入数据，使用[上一篇博客](https://zhu733756.github.io/posts/mongodb_sharding_cluster_and_replicaset/#%e5%88%86%e7%89%87%e9%9b%86)的脚本来测试分片集群数据了!

这里我们是有 hash 分片来测试:

```
mongos> use admin
mongos> db.runCommand( { enablesharding : "test" } )
mongos> use test
mongos> db.vast.ensureIndex( { id: 1 } )
mongos> use admin
mongos> db.adminCommand( {shardCollection: "test.vast", key : {id: "hashed"} } )
mongos> use test
mongos> for(i=0;i<20000;i++){ db.vast.insert({"id":i,"name":"woshishuaige","age":i,"date":new Date()}); }
mongos> db.vast.stats()
```

这里为了比较清晰的显示, 我们可以写一个 js 函数来格式化输出:

```
mongos> function processStats(stats) {
    // 遍历每个shard的统计信息
    for (let shardName in stats) {
        // 提取每个shard的统计数据
        let shardStats = stats[shardName];

        // 打印shard名称
        print("Shard: " + shardName);

        // 提取并打印对象数据和对象占用的大小
        print("Objects: " + shardStats.objects);
        print("Data Size: " + shardStats.dataSize + " bytes");
        print("Average Object Size: " + shardStats.avgObjSize + " bytes");
        print("\n");
    }
}

mongos> processStats(db.stats().raw);
Shard: rs-shard-02/shard02-a:27017,shard02-b:27017,shard02-c:27017
Objects: 6708
Data Size: 509808 bytes
Average Object Size: 76 bytes


Shard: rs-shard-01/shard01-a:27017,shard01-b:27017,shard01-c:27017
Objects: 6817
Data Size: 518092 bytes
Average Object Size: 76 bytes


Shard: rs-shard-03/shard03-a:27017,shard03-b:27017,shard03-c:27017
Objects: 6475
Data Size: 492100 bytes
Average Object Size: 76 bytes
```

看起来哈希分片后的数据还是比较均衡的, 你可以比较下这两种分片的运行效率!

测试完毕, 可以使用以下命令进行回收哦~

```
docker-compose down
```

## 后戏

> 小白: 太棒了，老花！我已经按照你的步骤成功部署了 MongoDB 的分片集群。接下来我该做些什么？

老花: 其实, 这个过程还是挺简单的, 我这里还有一套更简单的部署方式。你知道 `Kubernetes` 吗? 只需要一个 yaml 文件, 就能部署服务, 并且你可以找下开源的 `chart` 进行部署! 下一篇, 我们探索快速搭建 `k8s` 测试环境, 以及怎么在 Kubernetes 上 快速构建分片集群吧! 记得关注哦~
