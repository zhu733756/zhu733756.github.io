---
title: 'OneAPI: 大模型接口管理和分发系统'
tags: ['ollama', 'deepseek', '接口管理']
categories: ['实战', 'oneapi', '大模型']
series: ['llm']
author: ['zhu733756']
date: 2025-02-13T18:23:49+08:00
---

## 前戏

> 今天我们聊聊大模型接口管理和分发系统 OneApi, 并使用 ollama 对接大模型 deepseek-r1:7b。

## 🌟 项目简介

> https://github.com/songquanpeng/one-api

One API 是一个开源的 OpenAI 接口管理和分发系统，支持多种大模型，包括 Azure、Anthropic Claude、Google PaLM 2 & Gemini、智谱 ChatGLM、百度文心一言、讯飞星火认知、阿里通义千问、360 智脑以及腾讯混元。该项目旨在通过标准的 OpenAI API 格式访问所有的大模型，开箱即用。

## 📚 架构设计

`oneapi`的架构图比较简单:

![oneapi](/posts/llm_api/oneapi.png)

用户使用`key`访问`oneapi`的接口时，根据用户的身份和权限，调用对应的`model`模块进行模型调用。

## 🚀 功能特点

> https://github.com/songquanpeng/one-api?tab=readme-ov-file#%E5%8A%9F%E8%83%BD

- 支持多种大模型, 这里不一一列举了。
- 支持令牌管理
- 支持配额和兑换码管理
- 支持额度和用户激励
- 支持用户管理, 支持飞书、github、微信授权登录
- ...

## 🛠️ 部署指南

### 基于 Docker 进行部署

官方推荐:

```
# 使用 SQLite 的部署命令：
$ docker run --name one-api -d --restart always -p 3000:3000 -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api

# 使用 MySQL 的部署命令:
$ docker run --name one-api -d --restart always -p 3000:3000 -e SQL_DSN="root:123456@tcp(localhost:3306)/oneapi" -e TZ=Asia/Shanghai -v /home/ubuntu/data/one-api:/data justsong/one-api
```

由于 latest 当前没有 arm64 架构的镜像, 本人使用正式版部署:

```
docker run --name one-api -d --restart always -p 3000:3000 -e TZ=Asia/Shanghai -v /Applications/Docker-Home/oneapi/data:/data justsong/one-api:v0.6.10
```

查看部署状态:

```bash
$ docker ps
```

### 基于源码部署

#### 构建前端

```
cd one-api/web/default
npm install
npm run build

```

#### 构建后端

```
cd ../..
go mod download
go build -ldflags "-s -w" -o one-api

chmod u+x one-api
./one-api --port 3000 --log-dir ./logs
```

## 本地大模型 deepseek-r1 7b 实战

### 下载 ollama

> https://ollama.com/

### 安装 deepseek-r1:7b

设置环境变量运行远程访问:

```bash
$ launchctl setenv OLLAMA_HOST "0.0.0.0"
$ launchctl setenv OLLAMA_ORIGINS "*"
$ ollama run deepseek-r1:7b
```

### 配置令牌

使用 `root/123456` 登录 `http://<host>:3000/`添加渠道和令牌:

![channel](/posts/llm_api/oneapi-channel.png)

可以点击测试, 然后使用下面的代码进行测试:

```python
import openai

# 设置您的 API 密钥
openai.api_key = "sk-xxxx"

# 设置自定义的 API 请求地址
openai.api_base = "http://<one-api>:3000/v1"

# 设置对话的 prompt
messages = [
    {"role": "system", "content": ""},
    {"role": "user", "content": "1+1=？"}
]
# 使用 OpenAI API 进行聊天
response = openai.ChatCompletion.create(
    model="qwen:0.5b-chat-v1.5-q4_1",
    messages=messages,
    max_tokens=100,
    n=1,
    stop=None,
    temperature=0.5,
)

# 输出回复
assistant_message = response.choices[0].message
print(assistant_message.content)
```

## 小尾巴

前面我们介绍过`RAG`的概念, 指的是`检索增强和生成`的意思, 为了解决`llm`胡编乱造, 一般会投喂特定领域的知识库, 让后让大模型组织回答:

![rag](/posts/llm_api/rag.png)

`oneapi` 打通了用户管理和模型管理的两个维度, 让我们可以管理用户和模型, 用户想用哪款 `llm` 回答就用哪款。
