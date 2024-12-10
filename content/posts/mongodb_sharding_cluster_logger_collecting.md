---
title: 'Loki Stack收集MongoDB日志最佳实践(上)'
tags: ['MongoDB', '分片集群', 'kind', 'helm', 'loki', 'fluent bit']
categories: ['数据库', '日志管理', '实战', 'k8s']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
date: 2024-12-06T14:21:59+08:00
---

## 前戏

> 老花：小白，我们今天来聊聊收集容器日志的那些事儿。

> 小白：好啊，老花。我听说有好几种方法可以收集容器日志，你能给我讲讲吗？

## 常见收集容器日志的方法

老花：当然可以。最常见的有两种方法：`Sidecar`和`DaemonSet`。

- **Sidecar**：俗称边车, 在每个需要收集日志的容器旁边运行一个`Sidecar`容器，专门负责日志收集。这种方法的好处是可以为每个应用提供定制化的日志收集配置，但缺点是资源消耗较大，因为每个应用都需要额外的容器。

- **DaemonSet**：在`Kubernetes`集群的每个节点上运行一个`Pod`，负责收集该节点上所有容器的日志。优点是资源消耗较小，因为每个节点只需要一个`Pod`。缺点是配置不够灵活，因为所有容器共享同一个日志收集配置。

> 小白：我明白了，那我们通常用哪种方法呢？

老花：这取决于你的具体需求。如果你需要高度定制化的日志收集策略，可能会选择`Sidecar`。如果你更关心资源消耗，`DaemonSet`可能更适合。

## 日志收集系统优缺点

老花：不管是哪种收集方式, 都需要暴露日志路径给收集器。日志除了收集, 通常还有轮转、转换、存储和可视化查询等组件整合在一起, 组成强大日志收集系统。

我们来聊聊几个常见的日志收集系统：

采集、转换、扭转:

- **Loki**：轻量级，易于水平扩展，但不支持全文搜索。
- **Fluentd/Fluent Bit**：Fluent Bit 轻量级，插件丰富，Fluentd 插件丰富，支持多种数据源和目的地，灵活的配置，适合大规模部署。
- **Logstash**：功能强大，但资源消耗大，配置复杂。
- **Filebeat**：轻量级，易于配置，但功能不如 Logstash。
- 网易和阿里开源的采集组件也很强大。

存储:

- **Elasticsearch/ OpenSearch**，搜索能力强，但成本高；
- **ClickHouse（CK）**，查询速度快，成本较低。

套件:

- **ELK Stack（Elasticsearch, Logstash, Kibana）**：功能强大，但资源消耗大，配置复杂。
- **EFK Stack（Elasticsearch, Fluent, Kibana）**：功能强大，`fluent-bit`资源消耗低, 插件不如`fluentd`丰富，配置容易。
- **Loki on grafana**：采集组件可以使用`promtail`或者`fluent-bit`, 不需要额外的存储, 还自带可视化界面 UI, 多租户, 读写分离, 但不支持全文搜索。
- 各种采集器+ `Kafka` + `ClickHouse` +`ClickVisual` 等等。

> 小白：听起来每个系统都有它的优缺点，我们应该怎么选择呢？

老花：确实，选择哪个系统取决于你的具体需求，比如预算、资源、搜索需求等。

## Loki 接入 MongoDB 分片集群日志最佳实践

> 小白：老花，我们公司正在用 MongoDB 分片集群，你觉得用 `Loki` 来收集日志怎么样？

老花：这是个好问题。[`Loki`](https://grafana.com/docs/loki/latest/get-started/loki_architecture_components.svg) 可以很好地集成 MongoDB 分片集群的日志。我们可以部署 `Promtail`或者`Fluent-bit` 等采集器 作为 `DaemonSet` 来收集 `MongoDB` 的日志，然后将它们发送到 `Loki`。

> 小白：那 `Promtail`或者`Fluent-bit` 怎么知道哪些日志文件需要收集呢？

老花：它们会使用 `Kubernetes` 的 `API` 来发现节点上的 `Pods`，并根据 `Pod` 的注解或者其他配置来决定是否收集该 `Pod` 的日志。

> 小白：那听起来不错，我们可以试试。

### Loki Stack 部署

老花：我来帮你整理一下 `Loki` 的安装步骤，这里我们使用`Fluent Bit`作为采集器。我们仍然使用[前文](https://zhu733756.github.io/posts/mongodb_sharding_cluster_deploy_k8s_with_kind_guide/)创建的`mongodb-sharded` 来部署`helm`应用。

#### 准备工作

**参考链接**:

- https://github.com/grafana/helm-charts/blob/main/charts/fluent-bit/README.md

**导入仓库**:

```bash
$ helm repo add grafana https://grafana.github.io/helm-charts
$ helm repo update
```

**导入镜像**:

```bash
$ docker pull grafana/fluent-bit-plugin-loki:2.1.0-amd64
$ kind load docker-image grafana/fluent-bit-plugin-loki:2.1.0-amd64 --name mongodb-sharded

$ docker pull grafana/loki:2.6.1
$ kind load docker-image grafana/loki:2.6.1 --name mongodb-sharded

$ docker pull grafana/grafana:10.3.3
$ docker pull quay.io/kiwigrid/k8s-sidecar:1.19.2
$ kind load docker-image grafana/grafana:10.3.3 --name mongodb-sharded
$ kind load docker-image quay.io/kiwigrid/k8s-sidecar:1.19.2--name mongodb-sharded
```

**安装 Fluent-bit**：

```bash
$ helm upgrade --install  fluent-bit grafana/fluent-bit --create-namespace --namespace=loki
$ kubectl get pods -n loki
NAME                               READY   STATUS    RESTARTS   AGE
fluent-bit-fluent-bit-loki-hll2q   1/1     Running   0          30s
fluent-bit-fluent-bit-loki-m4gz4   1/1     Running   0          30s
fluent-bit-fluent-bit-loki-zf4tp   1/1     Running   0          30s
```

**安装 Loki**：

```bash
$ helm upgrade --install loki grafana/loki-stack --set fluent-bit.enabled=false,promtail.enabled=false,grafana.enabled=true,grafana.service.type=NodePort --create-namespace --namespace=loki

$ kubectl get pods -n loki
NAME                               READY   STATUS    RESTARTS   AGE
fluent-bit-fluent-bit-loki-hll2q   1/1     Running   0          18m
fluent-bit-fluent-bit-loki-m4gz4   1/1     Running   0          18m
fluent-bit-fluent-bit-loki-zf4tp   1/1     Running   0          18m
loki-0                             1/1     Running   0          16m
loki-grafana-66f56c89f7-gl5zh      2/2     Running   0          7m48s
```

### Loki Stack 问题排查

#### fluent-bit 写入报错

查看日志:

```bash
$ kubectl  -n loki logs -f fluent-bit-fluent-bit-loki-hll2q
level=warn caller=client.go:288 id=0 component=client host=fluent-bit-loki:3100 msg="error sending batch, will retry" status=-1 error="Post \"http://fluent-bit-loki:3100/api/prom/push\": context deadline exceeded"

$ kubectl -n loki get svc
$ kubectl -n loki get svc
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
loki              ClusterIP   10.0.52.138   <none>        3100/TCP   25m
loki-grafana      ClusterIP   10.0.82.18    <none>        80/TCP     16m
loki-headless     ClusterIP   None          <none>        3100/TCP   25m
loki-memberlist   ClusterIP   None          <none>        7946/TCP   25m
```

显然, 这里的`service`有问题, 我们需要改成`loki`:

```bash
$ kubectl edit cm fluent-bit-fluent-bit-loki -n loki
[Output]
    Name grafana-loki
    Match *
    Url http://loki:3100/api/prom/push
    TenantID ""
    BatchWait 1
    BatchSize 1048576
```

发现报错依然存在, 显然这个镜像是不支持热加载的, 现在我们手动重启`pod`或者也可以在开始安装的时候直接指定配置:

```bash
helm upgrade --install  fluent-bit grafana/fluent-bit --create-namespace --namespace=loki --set loki.serviceName="loki"
```

#### grafana dashboard 访问

`kind`是部署在`docker`中的, 我们先难道容器`ip`:

```bash
 $ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED      STATUS      PORTS                                                 NAMES
d75686260f85   kindest/node:v1.25.3   "/usr/local/bin/entr…"   8 days ago   Up 8 days                                                         mongodb-sharded-worker3
8627b5cd41f9   kindest/node:v1.25.3   "/usr/local/bin/entr…"   8 days ago   Up 8 days   0.0.0.0:31000->31000/tcp, 127.0.0.1:38823->6443/tcp   mongodb-sharded-control-plane
b0a00cb36381   kindest/node:v1.25.3   "/usr/local/bin/entr…"   8 days ago   Up 8 days                                                         mongodb-sharded-worker
aff9e82d00be   kindest/node:v1.25.3   "/usr/local/bin/entr…"   8 days ago   Up 8 days

$ docker inspect 8627b5cd41f9 |grep IPAddress
            "SecondaryIPAddresses": null,
            "IPAddress": "",
                    "IPAddress": "172.18.0.4",
```

这说明在`docker`中的`30776`暴露了出来:

```bash
$ kubectl  -n loki get svc
NAME              TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
loki              ClusterIP   10.0.52.138   <none>        3100/TCP       49m
loki-grafana      NodePort    10.0.82.18    <none>        80:30776/TCP   40m
loki-headless     ClusterIP   None          <none>        3100/TCP       49m
loki-memberlist   ClusterIP   None          <none>        7946/TCP       49m
```

理论上`172.18.0.4:30776`是可以通的。

为了让我们的虚拟机可以在主机上访问, 我们翻看下之前暴露的端口:

```bash
$ cat cluster.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31000  # 将主机 31000 端口映射到容器的 31000 端口
    hostPort: 31000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
```

因此, 我们要把`grafana`的`NodePort`改成`31000`:

```bash
$ kubectl edit svc loki-grafana  -n loki
```

恭喜你, 可以在浏览器打开`31000`端口了~

下面获取登录密码:

```bash
$ kubectl get  secret loki-grafana  -n loki -ojson | jq .data
{
  "admin-password": "SWhZcldQaTBkUkdjZ1EyTzVFTml3anZUY085WElSb09aWHQxTjZIMA==",
  "admin-user": "YWRtaW4=",
  "ldap-toml": ""
}

$ kubectl get  secret loki-grafana  -n loki -ojson | jq -r '.data."admin-password"' | base64 -d
```

然后我们就可以登录`grafana`页面了~

添加`datasource`:`http://loki:3100`, 点击`explore`就可以查询了:

## 小尾巴

> 小白: 太好了, 那我们开始查询容器日志吧!

老花: 休息会儿, 让我下期帮你介绍下`loki`语法哈~ 在此之前, 你可以使用[这个模板](https://grafana.com/grafana/dashboards/18042-logging-dashboard-via-loki-v2/)和[这个模板](https://grafana.com/grafana/dashboards/20010-loki-kubernetes-logs/)摸索哦~
