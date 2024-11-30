---
title: '实战 | 用golang撸一个极简MongoDB测试客户端?'
tags: ['MongoDB', '分片集群', 'mclient', '客户端']
categories: ['数据库', '实战', 'golang', 'demo']
series: ['MongoDB 知识汇总']
author: ['zhu733756']
date: 2024-11-29T20:40:12+08:00
---

## 前戏

> 小白：你好，老花！最近有个刁民让我测试部署的分片集群是否可用, 除了上官方上下载`mongosh`, 还有其他思路?

老花：当然有的! 我们可以用`golang`直接撸一个! 我们先整理一个功能清单:

```bash
mclient flags:
--uri mongodb 连接地址
--user 测试创建随机用户,并返回用户和密码
--insert 测试写入数据, 打印写入的随机条数
--read 测试插入随机数据, 并读取改数据
--remove  测试插入随机数据, 并删除该数据, 返回删除条数

```

## 搭建`golang`环境

1. 下载 Go 语言 SDK 工具包：访问 Go 官网下载适合`Linux`的安装包。

2. 解压安装包到指定目录，例如`/root/go`。

3. 配置环境变量：

- 使用`root`权限编辑`/etc/profile`文件。
- 添加`export GOROOT=/root/go`和`export PATH=$PATH:$GOROOT/bin`。
- 执行`source /etc/profile`刷新配置。

4. 测试配置是否生效，运行`go version`。

5. 配置 Go 语言开发工具
   可以选择多种开发工具，如 Visual Studio Code、GoLand 等，并安装相应的 Go 语言插件以支持智能提示、编译运行等功能
   。

6. 配置国内代理

   由于国内网络环境的特殊性，配置国内代理可以加速 Go 模块的下载。可以通过设置环境变量 GOPROXY 来实现：

   ```bash
    go env -w GOPROXY=https://goproxy.cn,direct
    go env -w GOPROXY=https://mirrors.aliyun.com/goproxy/
   ```

## mclient 功能实现

### 连接数据库

```go
client, err := mongo.NewClient(options.Client().ApplyURI(uri))
if err != nil {
  log.Fatalf("Failed to create MongoDB client: %v", err)
}

// 设置连接超时
ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
defer cancel()

// 连接到MongoDB集群
err = client.Connect(ctx)
if err != nil {
  log.Fatalf("Failed to connect to MongoDB cluster: %v", err)
}
defer client.Disconnect(ctx)

// 确认连接成功，发送ping命令
err = client.Ping(ctx, readpref.Primary())
if err != nil {
  log.Fatalf("Failed to ping MongoDB cluster: %v", err)
}

fmt.Println("Connected to MongoDB cluster successfully!")
```

### 创建随机用户

```go
func createRandomUser(client *mongo.Client) {
	collection := client.Database("admin").Collection("system.users")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	rand.Seed(time.Now().UnixNano())
	username := fmt.Sprintf("user%d", rand.Int())
	password := fmt.Sprintf("password%d", rand.Int())

	user := bson.M{"user": username, "pwd": password, "roles": []bson.M{{"role": "readWrite", "db": "testdb"}}}
	_, err := collection.InsertOne(ctx, user)
	if err != nil {
		log.Fatalf("Failed to create user: %v", err)
	}
	fmt.Printf("User created with username: %s and password: %s\n", username, password)
}
```

### 插入随机数据

```go
func insertRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	result, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}
	fmt.Printf("Inserted %d documents with IDs: %v\n", len(result.InsertedIDs), result.InsertedIDs)
}
```

### 读取和删除数据

```go
func insertAndReadRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	result, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}

	var results []bson.M
	cur, err := collection.Find(ctx, bson.M{})
	if err != nil {
		log.Fatalf("Failed to read documents: %v", err)
	}
	defer cur.Close(ctx)

	for cur.Next(ctx) {
		var elem bson.M
		err := cur.Decode(&elem)
		if err != nil {
			log.Fatal(err)
		}
		results = append(results, elem)
	}

	if err := cur.Err(); err != nil {
		log.Fatal(err)
	}

	fmt.Println("Documents read from database:", results)
}

func insertAndRemoveRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	result, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}

	filter := bson.M{"name": bson.M{"$in": []string{"hahahaa0"}}}

	deleteResult, err := collection.DeleteMany(ctx, filter)
	if err != nil {
		log.Fatalf("Failed to delete documents: %v", err)
	}
	fmt.Printf("Deleted %d documents\n", deleteResult.DeletedCount)
}
```

### 完整代码

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"math/rand"
	"time"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
	"go.mongodb.org/mongo-driver/mongo/readpref"
)

// 定义命令行参数
var uri string
var createUserFlag bool
var insertFlag bool
var readFlag bool
var removeFlag bool
var numDocuments int

func init() {
	// 默认URI设置为localhost，用户可以通过命令行参数--uri来覆盖
	flag.StringVar(&uri, "uri", "mongodb://localhost:27017", "MongoDB connection URI")
	flag.BoolVar(&createUserFlag, "user", false, "Test creating a random user")
	flag.BoolVar(&insertFlag, "insert", false, "Test inserting random data")
	flag.BoolVar(&readFlag, "read", false, "Test inserting and reading random data")
	flag.BoolVar(&removeFlag, "remove", false, "Test inserting and removing random data")
	flag.IntVar(&numDocuments, "num", 1, "Number of random documents to insert")
}

func main() {
	flag.Parse() // 解析命令行参数

	// 创建MongoDB客户端
	client, err := mongo.NewClient(options.Client().ApplyURI(uri))
	if err != nil {
		log.Fatalf("Failed to create MongoDB client: %v", err)
	}

	// 设置连接超时
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// 连接到MongoDB集群
	err = client.Connect(ctx)
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB cluster: %v", err)
	}
	defer client.Disconnect(ctx)

	// 确认连接成功，发送ping命令
	err = client.Ping(ctx, readpref.Primary())
	if err != nil {
		log.Fatalf("Failed to ping MongoDB cluster: %v", err)
	}

	fmt.Println("Connected to MongoDB cluster successfully!")

	if createUserFlag {
		createRandomUser(client)
	}

	if insertFlag {
		insertRandomDocuments(client, numDocuments)
	}

	if readFlag {
		insertAndReadRandomDocuments(client, numDocuments)
	}

	if removeFlag {
		insertAndRemoveRandomDocuments(client, numDocuments)
	}
}

func createRandomUser(client *mongo.Client) {
	collection := client.Database("admin").Collection("system.users")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	rand.Seed(time.Now().UnixNano())
	username := fmt.Sprintf("user%d", rand.Int())
	password := fmt.Sprintf("password%d", rand.Int())

	user := bson.M{"user": username, "pwd": password, "roles": []bson.M{{"role": "readWrite", "db": "testdb"}}}
	_, err := collection.InsertOne(ctx, user)
	if err != nil {
		log.Fatalf("Failed to create user: %v", err)
	}
	fmt.Printf("User created with username: %s and password: %s\n", username, password)
}

func insertRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	_, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}
	fmt.Printf("Inserted %d documents with IDs: %v\n", len(result.InsertedIDs), result.InsertedIDs)
}

func insertAndReadRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	_, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}

	var results []bson.M
	cur, err := collection.Find(ctx, bson.M{})
	if err != nil {
		log.Fatalf("Failed to read documents: %v", err)
	}
	defer cur.Close(ctx)

	for cur.Next(ctx) {
		var elem bson.M
		err := cur.Decode(&elem)
		if err != nil {
			log.Fatal(err)
		}
		results = append(results, elem)
	}

	if err := cur.Err(); err != nil {
		log.Fatal(err)
	}

	fmt.Println("Documents read from database:", results)
}

func insertAndRemoveRandomDocuments(client *mongo.Client, num int) {
	collection := client.Database("testdb").Collection("testdata")
	ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
	defer cancel()

	var documents []interface{}
	for i := 0; i < num; i++ {
		documents = append(documents, bson.M{"name": fmt.Sprintf("hahahaa%d", i), "age": rand.Intn(100)})
	}

	_, err := collection.InsertMany(ctx, documents)
	if err != nil {
		log.Fatalf("Failed to insert documents: %v", err)
	}

	filter := bson.M{"name": bson.M{"$in": []string{"hahahaa0"}}}

	deleteResult, err := collection.DeleteMany(ctx, filter)
	if err != nil {
		log.Fatalf("Failed to delete documents: %v", err)
	}
	fmt.Printf("Deleted %d documents\n", deleteResult.DeletedCount)
}
```

## 编译二进制

### 编译命令

- 编译单个文件：`go build -o myapp myapp.go`。
- 编译模块：`go build -mod=readonly -o myapp ./...`。
- 编译交叉平台：`GOOS=windows GOARCH=amd64 go build -o myapp_windows_amd64 myapp.go`。

### 编译优化

- 优化代码：避免不必要的包依赖，使用 Go 的性能分析工具（如 pprof），优化数据结构和算法。
- 优化编译选项：使用`-gcflags="all=-N -l"`禁用垃圾回收和内联优化，使用`-buildmode=pie`生成位置无关可执行文件。
- 优化构建工具：使用 Makefile 或构建系统（如 Bazel、Maven）自动化编译过程，使用并行编译提高编译速度。

## 测试效果

使用[上一篇博客](https://zhu733756.github.io/posts/mongodb_sharding_cluster_deploy_with_helm_guide/)部署的集群测试, 拿到`mongos`的`ip`: `10.244.2.2`:

```bash
$ kubectl get po -owide -n mongodb-sharded
NAME                                     READY   STATUS    RESTARTS   AGE   IP           NODE                      NOMINATED NODE   READINESS GATES
mongodb-sharded-configsvr-0              1/1     Running   0          35h   10.244.1.3   mongodb-sharded-worker    <none>           <none>
mongodb-sharded-configsvr-1              1/1     Running   0          35h   10.244.3.5   mongodb-sharded-worker2   <none>           <none>
mongodb-sharded-configsvr-2              1/1     Running   0          35h   10.244.2.6   mongodb-sharded-worker3   <none>           <none>
mongodb-sharded-mongos-9cffc5c76-k7ps2   1/1     Running   0          35h   10.244.2.2   mongodb-sharded-worker3   <none>           <none>
mongodb-sharded-shard0-data-0            1/1     Running   0          35h   10.244.2.4   mongodb-sharded-worker3   <none>           <none>
mongodb-sharded-shard0-data-1            1/1     Running   0          35h   10.244.3.7   mongodb-sharded-worker2   <none>           <none>
mongodb-sharded-shard0-data-2            1/1     Running   0          35h   10.244.2.8   mongodb-sharded-worker3   <none>           <none>
mongodb-sharded-shard1-data-0            1/1     Running   0          35h   10.244.3.3   mongodb-sharded-worker2   <none>           <none>
mongodb-sharded-shard1-data-1            1/1     Running   0          35h   10.244.1.5   mongodb-sharded-worker    <none>           <none>
mongodb-sharded-shard1-data-2            1/1     Running   0          35h   10.244.1.7   mongodb-sharded-worker    <none>           <none>
```

编译:

```go
$ go build -o  mclient client.go
```

拷贝二进制到容器中:

```bash
$ docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED        STATUS        PORTS                                                 NAMES
d75686260f85   kindest/node:v1.25.3   "/usr/local/bin/entr…"   36 hours ago   Up 36 hours                                                         mongodb-sharded-worker3
8627b5cd41f9   kindest/node:v1.25.3   "/usr/local/bin/entr…"   36 hours ago   Up 36 hours   0.0.0.0:31000->31000/tcp, 127.0.0.1:38823->6443/tcp   mongodb-sharded-control-plane
b0a00cb36381   kindest/node:v1.25.3   "/usr/local/bin/entr…"   36 hours ago   Up 36 hours                                                         mongodb-sharded-worker
aff9e82d00be   kindest/node:v1.25.3   "/usr/local/bin/entr…"   36 hours ago   Up 36 hours                                                         mongodb-sharded-worker2

$ docker cp mclient d75686260f85:mclient
Successfully copied 12.1MB to d75686260f85:mclient

$ docker exec -it d75686260f85 bash
$ chmod a+x mclient
$./mclient --help
Usage of ./mclient:
  -insert
        Test inserting random data
  -num int
        Number of random documents to insert (default 1)
  -read
        Test inserting and reading random data
  -remove
        Test inserting and removing random data
  -uri string
        MongoDB connection URI (default "mongodb://localhost:27017")
  -user
        Test creating a random user
```

### 连接测试

```bash
$ ./mclient --uri mongodb://root:123456@10.244.2.2:27017/admin
Connected to MongoDB cluster successfully!
```

### 插入测试

```bash
$./mclient --uri mongodb://root:123456@10.244.2.2:27017/admin --insert
Connected to MongoDB cluster successfully!
Inserted 1 documents with IDs: [ObjectID("6749bc5feef8327a8288b504")]
```

### 读取测试

```bash
$./mclient --uri mongodb://root:123456@10.244.2.2:27017/admin --read
Connected to MongoDB cluster successfully!
Documents read from database: [map[_id:ObjectID("6749bc5feef8327a8288b504") age:60 name:hahahaa0] map[_id:ObjectID("6749bc66080d166723679c56") age:10 name:hahahaa0]]
```

### 删除测试

```bash
$./mclient --uri mongodb://root:123456@10.244.2.2:27017/admin --remove
Connected to MongoDB cluster successfully!
Deleted 3 documents
```

### 创建随机用户

```bash
$./mclient --uri mongodb://root:123456@10.244.2.2:27017/admin --user
Connected to MongoDB cluster successfully!
User created with username: user5888017645476970789 and password: password6708713425938649173
```

## 后记

> 小白：真是总有刁民`gank`朕~这下可帮我解决了大麻烦

老花：哈哈, 平常心就好~文中只是一个`demo`, 可以按需修改哦~

> 小白：下一期我们继续分享数据库生态哪些内容?

老花: 这个看心情哈~ 有可能是日志, 有可能监控~ see you!
