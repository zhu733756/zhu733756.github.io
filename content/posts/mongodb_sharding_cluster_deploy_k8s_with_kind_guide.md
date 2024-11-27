---
title: 'MongoDB分片集群在k8s上的部署实战(一)'
date: 2024-11-27T19:21:37+08:00
tags: ['MongoDB', '分片集群', 'kind']
categories: ['数据库', '实战', 'k8s']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
---

## 前戏

> 小白：你好，老花！我对在 Kubernetes 上使用 Helm 部署 MongoDB Sharded 集群很感兴趣，但我对 Kind 和 Helm 不太熟悉，你能详细教我一下吗？

老花：当然可以，小白！我们先从 Kind 和 Helm 的安装开始，然后详细介绍 Helm 中的每个角色配置，最后解释 Helm 应用是如何运行起来的。

## Kind 快速构建集群

Kind 是一个使用 Docker 容器作为节点来运行本地 Kubernetes 集群的工具。可以通过以下步骤安装 Kind:

### 安装 Docker

Kind 需要 Docker 来运行 Kubernetes 集群，所以首先确保你已经安装了 Docker。

### 安装 Kind

可以使用以下两种方式中一种下载:

```bash
> sudo apt-get install kind
> go install sigs.k8s.io/kind@v0.25.0
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
```

或者使用代理:

```json
{
	"proxies": {
		"default": {
			"httpProxy": "http://xxxx:xx",
			"httpsProxy": "https://xxxx:xx",
			"noProxy": "docker.m.daocloud.io,127.0.0.0/8"
		}
	}
}
```

### 创建`Kind`集群:

#### 配置`docker`和环境

下面的命令修改了`docker`运行时, 并配置了`ulimits`上限:

```bash
> cat /etc/docker/daemon.json
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://registry.cn-hangzhou.aliyuncs.com",
    "https://dockerhub.icu",
    "https://docker.chenby.cn",
    "https://docker.1panel.live",
    "https://docker.awsl9527.cn",
    "https://docker.anyhub.us.kg",
    "https://dhub.kubesre.xyz",
    "https://docker.13140521.xyz"
  ],
  "exec-opts": [
    "native.cgroupdriver=systemd"
  ],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-ulimits": {
    "nofile": {
      "name": "nofile",
      "hard": 65536,
      "soft": 65536
    }
  }
}

> systemctl daemon-reload && systemctl restart docker
```

查看 `cgroup` 是否生效:

```
> docker info |grep -i cgroup
Cgroup Driver: systemd
Cgroup Version: 1
```

执行`swapoff`:

```
swapoff -a
```

开启转发:

```
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv4.ip_forward=1
```

开启`cgroup v2`(高版本推荐):

```
sudo nano /etc/default/grub
GRUB_CMDLINE_LINUX="quiet splash systemd.unified_cgroup_hierarchy=1"
sudo update-grub
sudo reboot
```

#### 正常模式

```yaml
cat cluster.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31000  # 将主机 31000 端口映射到容器的 31000 端口
    hostPort: 31000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
- role: worker
```

```bash
kind create cluster --config cluster.yaml --name mongodb-sharded
```

#### 国内模式

参考[博客](https://zhuanlan.zhihu.com/p/60464867):

```yaml
cat cluster.yaml

kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 31000  # 将主机 31000 端口映射到容器的 31000 端口
    hostPort: 31000
    listenAddress: "0.0.0.0" # Optional, defaults to "0.0.0.0"
    protocol: tcp # Optional, defaults to tcp
- role: worker
kubeadmConfigPatches:
- |
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: ClusterConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
  nodeRegistration:
    kubeletExtraArgs:
      pod-infra-container-image: registry.aliyuncs.com/google_containers/pause:3.1
- |
  apiVersion: kubeadm.k8s.io/v1beta3
  kind: InitConfiguration
  metadata:
    name: config
  networking:
    serviceSubnet: 10.0.0.0/16
  imageRepository: registry.aliyuncs.com/google_containers
```

#### 成功创建集群

其他配置: 参考[官方文档](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)

```bash
kind create cluster --config cluster.yaml --name mongodb-sharded --image kindest/node:v1.25.3
Creating cluster "mongodb-sharded" ...
 ✓ Ensuring node image (kindest/node:v1.25.3) 🖼
 ✓ Preparing nodes 📦 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
 ✓ Joining worker nodes 🚜
Set kubectl context to "kind-mongodb-sharded"
You can now use your cluster with:

kubectl cluster-info --context kind-mongodb-sharded

Have a nice day! 👋

> kubectl get po -n kube-system
NAME                                                    READY   STATUS    RESTARTS      AGE
coredns-c676cc86f-4fz5s                                 1/1     Running   1 (58m ago)   71m
coredns-c676cc86f-gh6bc                                 1/1     Running   1 (58m ago)   71m
etcd-mongodb-sharded-control-plane                      1/1     Running   0             58m
kindnet-l26fb                                           1/1     Running   1 (58m ago)   71m
kindnet-x26gq                                           1/1     Running   1 (58m ago)   71m
kube-apiserver-mongodb-sharded-control-plane            1/1     Running   0             58m
kube-controller-manager-mongodb-sharded-control-plane   1/1     Running   1 (58m ago)   71m
kube-proxy-8nxr2                                        1/1     Running   8 (60m ago)   71m
kube-proxy-9c2km                                        1/1     Running   8 (59m ago)   71m
kube-scheduler-mongodb-sharded-control-plane            1/1     Running   1 (58m ago)   71m
```

## 后记

> 小白: 经过几个小时的来回折腾, 咱们终于用上`Kind`创建的集群了~ 

老花: 测试不易,求关注~ 下篇我们将使用`helm`部署高可用分片集群,敬请关注~
