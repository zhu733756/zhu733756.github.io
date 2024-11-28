---
title: 'MongoDB分片集群在k8s上的部署实战(二)'
date: 2024-11-28T09:21:37+08:00
tags: ['MongoDB', '分片集群', 'kind', 'helm']
categories: ['数据库', '实战', 'k8s']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
---

## 前戏

> 小白：你好，老花！前面我们已经用`kind`创建了一个`k8s`集群, 接下来我们怎么部署`sharded cluster`?

老花：当然可以，小白！我们先从 Helm 的安装开始，然后详细介绍 Helm 中的每个角色配置，最后解释 Helm 应用是如何运行起来的。

### 安装 helm

老花：Helm 是 Kubernetes 的包管理器，它帮助我们管理 Kubernetes 应用。安装 Helm 的步骤如下：

1. 下载 Helm：访问 Helm 的[官方 GitHub 页面](https://github.com/helm/helm/releases)，下载最新版本的 Helm。
2. 解压并移动 Helm 到你的 PATH 中：

   ```bash
   tar -zxvf helm-v3.0.0-linux-amd64.tar.gz
   cd linux-amd64
   sudo mv helm /usr/local/bin/
   ```

3. 验证 Helm 是否安装成功：

   ```bash
   helm version
   ```

老花：Helm 也安装好了，我们可以开始部署 MongoDB Sharded 集群了。再安装之前, 我们先了解 Helm 的基本命令。

### Helm 常见命令汇总

- `helm init`：初始化 Tiller（在 Helm 3 中不再需要）。
- `helm repo add <repo_name> <repository_url>`：添加一个新的 Helm 仓库。
- `helm repo list`：列出所有已添加的 Helm 仓库。
- `helm repo update`：更新本地仓库的缓存。
- `helm search repo <keyword>`：在所有仓库中搜索 Chart。
- `helm search hub <keyword>`：在 Helm Hub 中搜索 Chart。
- `helm install <release_name> <chart>`：安装一个 Helm Chart，并给它一个发布名称。
- `helm install --namespace <namespace> <release_name> <chart>`：指定命名空间安装 Helm Chart。
- `helm list`：列出所有的 Helm 发布。
- `helm status <release_name>`：获取指定 Helm 发布的状态信息。
- `helm upgrade <release_name> <new_chart>`：升级一个现有的 Helm 发布到新版本的 Chart。
- `helm rollback <release_name> <revision>`：将 Helm 发布回滚到指定的版本。
- `helm uninstall <release_name>`：卸载一个 Helm 发布。
- `helm history <release_name>`：查看一个 Helm 发布的更新历史。
- `helm package <chart_dir>`：将 Helm Chart 打包成 `tgz` 文件。
- `helm show chart <chart>`：查看一个 Helm Chart 的详细信息。
- `helm get values <release_name>`：查看指定 Helm 发布的配置值。
- `helm get all <release_name>`：查看指定 Helm 发布的所有配置和值。
- `helm dependency list <chart>`：列出一个 Helm Chart 的依赖。
- `helm dependency update <chart_dir>`：更新一个 Helm Chart 目录中的依赖。
- `helm lint <chart_dir>`：对一个 Helm Chart 进行 lint 检查。

### MongoDB Helm Chart 中的每个角色配置

[前文](https://zhu733756.github.io/posts/mongodb_sharding_cluster_and_replicaset/#%e5%88%86%e7%89%87%e9%9b%86)提到过, MongoDB Sharded 集群主要包括以下几个角色：

1. **Config Server**：配置服务器，负责存储集群的元数据。
2. **Mongos**：路由器，负责处理客户端请求并路由到正确的分片。
3. **Shard**：数据分片，负责存储实际的数据。

我们将使用 Bitnami 的 MongoDB Sharded Helm Chart 来部署这些角色。

### 部署 MongoDB Sharded 集群

首先，我们需要添加 Bitnami 的 Helm 仓库：

```bash
helm repo add stable https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

然后，我们可以使用以下命令部署 MongoDB Sharded 集群:

> Tip: 生产环境最少要设置副本数为 3, 除非有高可用云盘托底

> Tip: `Kind`的集群创建在`docker`环境中, 我们要将宿主机的镜像导入到docker中的镜像中, 可以使用 `kind load docker-image`

```bash
> docker pull  bitnami/mongodb-sharded:8.0.3-debian-12-r0

> kind load docker-image  docker.io/bitnami/mongodb-sharded:8.0.3-debian-12-r0 --name mongodb-sharded

> helm -n mongodb-sharded install mongodb-sharded bitnami/mongodb-sharded --create-namespace --set architecture=sharded --set shards=2 --set shardsvr.dataNode.replicaCount=3 --set configsvr.replicaCount=3 --set mongos.replicaCount=1 --set auth.rootPassword="123456"

NAME: mongodb-sharded
LAST DEPLOYED: Thu Nov 28 09:28:05 2024
NAMESPACE: mongodb-sharded
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
CHART NAME: mongodb-sharded
CHART VERSION: 9.0.3
APP VERSION: 8.0.3
```

> 小白：这个命令中的`--set architecture=sharded`是什么意思？

老花：这个参数指定了 MongoDB 的架构为分片架构。除了这个参数，我们还可以设置其他参数，比如：

- `shards`：设置分片的数量。
- `shardsvr.dataNode.replicaCount`：设置每个数据节点角色的副本数量。
- `configsvr.replicaCount`：设置配置服务器的副本数量。
- `mongos.replicaCount`：设置路由器的副本数量。
- `auth.rootPassword`: 设置
- 更多参数参考这里[values.yaml](https://github.com/bitnami/charts/blob/main/bitnami/mongodb-sharded/values.yaml)

### Helm 应用是如何运行起来的

> 小白：Helm 应用是如何在 Kubernetes 上运行起来的？

老花：当你运行 Helm install 命令时，Helm 会根据 chart 中的模板和你的配置参数生成 Kubernetes 的 manifest 文件。然后，Kubernetes 会根据这些 manifest 文件创建 Pods、Services、Deployments 等资源。

1. **创建 Namespace**：Helm 会在 Kubernetes 中创建一个新的 Namespace，用于隔离部署的应用。
2. **部署 Config Server**：Helm 会创建 Config Server 的 Deployment 和 Service。
3. **部署 Shards**：Helm 会创建 Shards 的 StatefulSet 和 Service。
4. **部署 Mongos**：Helm 会创建 Mongos 的 Deployment 和 Service。
5. **初始化集群**：Helm 会执行初始化脚本以及结合容器内部的一些脚本来配置 MongoDB Sharded 集群。

> 小白：原来是这样，那我怎么检查集群的状态呢？

## 集群可用性检测和测试

### 检查集群状态

老花：你可以通过以下命令检查 Pods 的状态：

```bash
> kubectl get po -n mongodb-sharded
NAME                                     READY   STATUS    RESTARTS   AGE
mongodb-sharded-configsvr-0              1/1     Running   0          6m43s
mongodb-sharded-configsvr-1              1/1     Running   0          5m56s
mongodb-sharded-configsvr-2              1/1     Running   0          5m30s
mongodb-sharded-mongos-9cffc5c76-k7ps2   1/1     Running   0          6m43s
mongodb-sharded-shard0-data-0            1/1     Running   0          6m42s
mongodb-sharded-shard0-data-1            1/1     Running   0          4m35s
mongodb-sharded-shard0-data-2            1/1     Running   0          3m56s
mongodb-sharded-shard1-data-0            1/1     Running   0          6m43s
mongodb-sharded-shard1-data-1            1/1     Running   0          4m36s
mongodb-sharded-shard1-data-2            1/1     Running   0          3m59s
```

确保所有的 Pods 都处于 Running 状态。

### 测试集群

> 小白：我怎么测试集群是否正常工作？

老花：你可以通过连接到 Mongos Pod 来测试集群：

```bash
 kubectl exec -it mongodb-sharded-mongos-9cffc5c76-k7ps2  -n mongodb-sharded bash --  mongosh admin -u root -p 123456
Current Mongosh Log ID: 6747ca1e1c6d17dd2dfe6910
Connecting to:          mongodb://<credentials>@127.0.0.1:27017/admin?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.3.2
Using MongoDB:          8.0.3
Using Mongosh:          2.3.2
mongosh 2.3.4 is available for download: https://www.mongodb.com/try/download/shell

For mongosh info see: https://www.mongodb.com/docs/mongodb-shell/
```

然后，你可以执行一些基本的 MongoDB 命令来测试集群，比如创建数据库、插入数据等：

```javascript
use testdb
testdb> db.createCollection("testcol")
testdb> db.testcol.insert({"testKey": "testValue"})
testdb> db.testcol.find()
[ { _id: ObjectId('6747ca421c6d17dd2dfe6911'), testKey: 'testValue' } ]
testdb> db.stats()
{
  raw: {
    'mongodb-sharded-shard-1/mongodb-sharded-shard1-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017,mongodb-sharded-shard1-data-1.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017,mongodb-sharded-shard1-data-2.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017': {
      db: 'testdb',
      collections: Long('1'),
      views: Long('0'),
      objects: Long('1'),
      avgObjSize: 45,
      dataSize: 45,
      storageSize: 20480,
      indexes: Long('1'),
      indexSize: 20480,
      totalSize: 40960,
      scaleFactor: Long('1'),
      fsUsedSize: 73270255616,
      fsTotalSize: 156865380352,
      ok: 1
    },
    'mongodb-sharded-shard-0/mongodb-sharded-shard0-data-0.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017,mongodb-sharded-shard0-data-1.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017,mongodb-sharded-shard0-data-2.mongodb-sharded-headless.mongodb-sharded.svc.cluster.local:27017': {
      db: 'testdb',
      collections: Long('0'),
      views: Long('0'),
      objects: Long('0'),
      avgObjSize: 0,
      dataSize: 0,
      storageSize: 0,
      indexes: Long('0'),
      indexSize: 0,
      totalSize: 0,
      scaleFactor: Long('1'),
      fsUsedSize: 0,
      fsTotalSize: 0,
      ok: 1
    }
  },
  db: 'testdb',
  collections: 1,
  views: 0,
  objects: 1,
  avgObjSize: 45,
  dataSize: 45,
  storageSize: 20480,
  indexes: 1,
  indexSize: 20480,
  totalSize: 40960,
  scaleFactor: 1,
  fsUsedSize: 73270255616,
  fsTotalSize: 156865380352,
  ok: 1,
  '$clusterTime': {
    clusterTime: Timestamp({ t: 1732758105, i: 1 }),
    signature: {
      hash: Binary.createFromBase64('cS19dYQ9GQKmux8od1BUyi7RpUc=', 0),
      keyId: Long('7442135956880097286')
    }
  },
  operationTime: Timestamp({ t: 1732758104, i: 1 })
}
```

当然了, 你可以使用[上一篇博客](https://zhu733756.github.io/posts/mongodb_sharding_cluster_deploy_on_docker_guide/#%E7%AC%AC%E4%B8%83%E6%AD%A5%E5%90%AF%E7%94%A8%E5%88%86%E7%89%87%E5%B9%B6%E8%AE%BE%E7%BD%AE%E5%88%86%E7%89%87%E9%94%AE)中的命令去验证啦~

## 后记

> 小白：老花，我还有一个问题。如果我想在生产环境中部署 `MongoDB` 分片集群，有什么特别的注意事项吗？

老花：当然有。在生产环境中，你需要考虑更多的因素，比如：

- **高可用性**：确保你的集群配置了足够的副本数来提供高可用性。
- **安全性**：配置 TLS/SSL 来加密数据传输，并使用 RBAC 来管理用户权限。
- **监控和日志**：设置监控和日志收集来跟踪集群的性能和问题。
- **备份和恢复**：定期备份数据，并确保你可以从备份中恢复数据。

记得关注下一篇文章，我会带来更多关于 `Kubernetes` 和 `MongoDB` 的实用指南。
