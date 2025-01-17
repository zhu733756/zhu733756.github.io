---
title: '使用 Milvus 构建图片搜索引擎'
tags: ['milvus', 'kind', 'towhee', '图片搜索']
categories: ['实战', '大模型', 'RAG']
series: ['llm']
author: ['zhu733756']
date: 2025-01-13T18:46:43+08:00
---

## 前戏

> 在当今数字化时代，图片数据呈爆炸式增长，如何从海量图片中快速准确地找到相似图片成为了一个重要问题。`Milvus` 作为一个高性能的向量数据库，为解决这一问题提供了强大的支持。本文将详细介绍如何使用 `Milvus` 构建一个图片搜索引擎，让你能够轻松地在图片海洋中找到“知音”。

## Milvus 简介

Milvus 是专为处理海量向量数据而设计的数据库，它构建在 Faiss、HNSW、DiskANN、SCANN 等流行的向量搜索库之上，能够高效地在包含数百万、数十亿甚至数万亿向量的数据集上进行相似性搜索。Milvus 支持数据分片、流式数据摄取、动态 Schema、结合向量和标量数据的搜索等多种高级功能，采用共享存储架构，计算节点具有存储和计算分解及横向扩展能力，非常适合用于构建图片搜索引擎。

## 图片搜索引擎架构

> https://github.com/milvus-io/bootcamp/blob/master/applications/image/reverse_image_search/README.md

上面的例子使用 `towhee` 通过 `ResNet50` 提取图像特征，并使用 `Milvus` 构建了一个可以执行反向图像搜索的系统。

`Towhee` 由四大模块构成：`算子`、`流水线`、`数据处理 API` 和`执行引擎`。

> 算子是基础组件，可为神经网络模型、数据处理方法或 Python 函数。流水线由多个算子组成，形成有向无环图，实现复杂功能如特征提取等。数据处理 API 提供多种数据转换接口，用于构建数据处理管道，处理非结构化数据。执行引擎负责流水线实例化、任务调度、资源管理及性能优化，有轻量级本地引擎和高性能的 Nvidia Triton 引擎。

系统架构如下:

![架构图](/posts/llm_milvus/towhee_arch.png)

## 数据源

> https://drive.google.com/file/d/1n_370-5Stk4t0uDV1QqvYkcvyV8rbw0O/view?usp=sharing

## 环境准备

### 安装 Milvus

你可以选择在本地安装 Milvus Lite，也可以在 Docker 或 Kubernetes 上部署高性能的 Milvus 服务器。可以参考[这篇文章](https://zhu733756.github.io/posts/llm_deploy_milvus_on_kubernetes/)。

### 服务部署

> https://github.com/milvus-io/bootcamp/blob/master/applications/image/reverse_image_search/docker-compose.yaml

这个 `docker-compose.yaml` 文件是一个用于部署`Milvus` 和相关组件的配置文件。主要包括:

- etcd
- minio
- milvus-standalone
- mysql
- webserver
- webclient

我们可以把它缩减成这样:

- mysql
- webserver
- webclient

> 详见: https://github.com/zhu733756/cloud-database-tools/blob/master/milvus/img-search/docker/docker-compose.yaml

需要修改这几个参数:

```
environment:
   MILVUS_HOST: '172.18.0.4'
   MILVUS_PORT: '30774'
   MYSQL_HOST: '172.16.238.11'
```

配置 `Milvus` 这个 `CR` 为 `NodePort`:

```
$ kubectl  get milvus -oyaml |grep NodePort -C 10
proxy:
   paused: false
   replicas: 1
   serviceType: NodePort
$ kubectl  get svc my-release-milvus
NAME                TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                          AGE
my-release-milvus   NodePort   10.0.234.236   <none>        19530:30774/TCP,9091:30738/TCP   11d
```

获取`Milvus`的外网地址:

```
$ docker ps | head -2
CONTAINER ID   IMAGE                  COMMAND                  CREATED       STATUS       PORTS                                                 NAMES
0f822511a237   kindest/node:v1.25.3   "/usr/local/bin/entr…"   13 days ago   Up 9 hours                                                         milvus-cluster-worker

$ docker inspect 0f822511a237  |grep IPAddress
"IPAddress": "172.18.0.4",
```

## 开始操作

### api 接口

![webserver api](/posts/llm_milvus/image-search-server-api.png)

### 插入数据

![insert data](/posts/llm_milvus/image-search-insert-data.png)

### 搜索图片

![results](/posts/llm_milvus/image-search-results.png)

## 核心代码走读

### 图片上传

`do_upload`实现了向量和元数据的存储:

```python
@app.post('/img/upload')
async def upload_images(image: UploadFile = File(None), url: str = None, table_name: str = None):
    # Insert the upload image to Milvus/MySQL
    try:
        # Save the upload image to server.
        if image is not None:
            content = await image.read()
            img_path = os.path.join(UPLOAD_PATH, image.filename)
            with open(img_path, "wb+") as f:
                f.write(content)
        elif url is not None:
            img_path = os.path.join(UPLOAD_PATH, os.path.basename(url))
            urlretrieve(url, img_path)
        else:
            return {'status': False, 'msg': 'Image and url are required'}, 400
        vector_id = do_upload(table_name, img_path, MODEL, MILVUS_CLI, MYSQL_CLI)
        LOGGER.info(f"Successfully uploaded data, vector id: {vector_id}")
        return "Successfully loaded data: " + str(vector_id)
    except Exception as e:
        LOGGER.error(e)
        return {'status': False, 'msg': e}, 400

def do_upload(table_name: str, img_path: str, model: Resnet50, milvus_client: MilvusHelper, mysql_cli: MySQLHelper):
    try:
        if not table_name:
            table_name = DEFAULT_TABLE
        if not milvus_client.has_collection(table_name):
            milvus_client.create_collection(table_name)
            milvus_client.create_index(table_name)
        feat = model.resnet50_extract_feat(img_path)
        ids = milvus_client.insert(table_name, [feat])
        mysql_cli.create_mysql_table(table_name)
        mysql_cli.load_data_to_mysql(table_name, [(str(ids[0]), img_path.encode())])
        return ids[0]
    except Exception as e:
        LOGGER.error(f"Error with upload : {e}")
        sys.exit(1)
```

### 搜索图片

```python
@app.post('/img/search')
async def search_images(image: UploadFile = File(...), topk: int = Form(TOP_K), table_name: str = None):
    # Search the upload image in Milvus/MySQL
    try:
        # Save the upload image to server.
        content = await image.read()
        img_path = os.path.join(UPLOAD_PATH, image.filename)
        with open(img_path, "wb+") as f:
            f.write(content)
        paths, distances = do_search(table_name, img_path, topk, MODEL, MILVUS_CLI, MYSQL_CLI)
        res = dict(zip(paths, distances))
        res = sorted(res.items(), key=lambda item: item[1])
        LOGGER.info("Successfully searched similar images!")
        return res
    except Exception as e:
        LOGGER.error(e)
        return {'status': False, 'msg': e}, 400
```

## 小尾巴

通过以上步骤，我们成功地使用 `Milvus` 构建了一个图片搜索引擎。`Milvus` 强大的向量搜索能力使得我们能够在海量图片中快速找到与查询图片相似的结果。

无论是用于图像检索、内容推荐还是其他需要图片相似性搜索的场景，`Milvus` 都是一个值得信赖的选择。希望本文能够帮助你更好地理解和应用 `Milvus`，让你在图片搜索的世界中畅游无阻。

> 感谢你的时间, 欢迎关注一波。
