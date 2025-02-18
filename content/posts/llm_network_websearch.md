---
title: '离线大模型联网的N个插件尝试'
tags: ['page assist', 'openweb ui', 'anything llm', 'chatbox']
categories: ['实战', '大模型联网插件']
series: ['llm']
author: ['zhu733756']
date: 2025-02-18T09:32:54+08:00
---

## 前戏

> 今天我们一起看下几个大模型联网插件, 让你的离线大模型打通实时联网能力任督二脉。

## 架构介绍

一个典型的 RAG 流程图如下:

![rag简易流程时序图](/posts/llm_api/rag.png)

在这个流程中有一个联网查询过程, 插件通过调用实时爬虫将搜索到的文档丢给大模型, 然后总结输出回答。

## 离线大模型部署

> https://ollama.com/

## 联网插件哪家强?

### Page Assist

Page Assist 是一款专为开发者设计的开源浏览器扩展工具，旨在通过简单的配置和操作，让用户直接调用本地或离线运行的大模型。它支持多种主流浏览器（如 Chrome、Edge、Firefox 等），并提供了丰富的插件功能，帮助用户实现跨网页交互、文档解析、搜索管理等功能。

本文使用 Edge 浏览器演示, 毕竟不用使用魔法就能安装。

安装以后, 我们在右上角的配置设置选择语言模式为中文, 然后配置本地的 ollama 地址:

![page_assist_ollama](/posts/llm_websearch/page_assist_ollama.png)

接着配置联网插件, 这里我们使用百度。

![page_assist_search](/posts/llm_websearch/page_assist_search.png)

问下大模型 page assist 是什么?

![page_assist_qa](/posts/llm_websearch/page_assist_qa.png)

### Open WebUI

> https://github.com/open-webui/open-webui

#### 功能特点

🎉 无缝安装：支持通过 Docker 或 Kubernetes（kubectl、kustomize 或 helm）进行安装，支持 :ollama 和 :cuda 标签的镜像。

🔗 Ollama/OpenAI API 集成：无缝集成 OpenAI 兼容 API，支持与 LMStudio、GroqCloud、Mistral、OpenRouter 等更多服务的连接。

🛡️ 细粒度权限与用户组：管理员可以创建详细的用户角色和权限，确保安全的用户环境。

📱 响应式设计：支持桌面、笔记本和移动设备。

📱 进阶 Web 应用（PWA）：支持移动端的离线访问，提供类似原生应用的体验。

✍️ Markdown 和 LaTeX 支持：支持 Markdown 和 LaTeX，提升交互体验。

🎤 视频/语音通话：支持免提语音和视频通话功能。

🛠️ 模型构建器：通过 Web UI 创建 Ollama 模型，支持自定义角色和代理。

🐍 Python 函数调用工具：支持在工具工作区中直接运行 Python 函数。

📚 本地 RAG 集成：支持文档检索增强生成（RAG），可以直接在聊天中加载文档。

🔍 网络搜索：支持通过多种搜索引擎（如 SearXNG、Google PSE 等）将结果注入聊天中。

🌐 网页浏览：支持通过 URL 将网页内容集成到聊天中。

🎨 图像生成集成：支持通过 AUTOMATIC1111 API 或 OpenAI 的 DALL-E 等工具生成图像。

🌐 多模型对话：支持同时与多个模型对话。

🔐 基于角色的访问控制（RBAC）：确保只有授权用户可以访问 Ollama 和模型。

🌐 多语言支持：支持国际化（i18n），并欢迎更多语言贡献。

🧩 插件支持：支持通过 Pipelines 插件框架集成自定义逻辑和 Python 库。

#### 安装

使用 pip 安装:

```bash
$ pip install open-webui
```

使用 docker 安装:

```bash
$ docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/Applications/Docker-Home/open-webui/data -e OLLAMA_BASE_URL=http://127.0.0.1:11434 --name open-webui --restart always ghcr.nju.edu.cn/open-webui/open-webui:main
```

#### 登录 ui 开启大模型联网功能

访问地址: http://127.0.0.1:3000, 配置模型和搜索功能:

![open_webui_websearch](/posts/llm_websearch/open_webui_websearch.png)

测试一下:

![open_webui_qa](/posts/llm_websearch/open_webui_qa.png)

### Anything LLM

> https://github.com/Mintplex-Labs/anything-llm

#### 功能特点

🎉 一站式解决方案：支持多种文档类型（PDF、TXT、DOCX 等），并提供简单易用的聊天界面。

🌐 多模态支持：支持开源和闭源的 LLM。

👥 多用户支持：Docker 版本支持多用户实例和权限管理。

🤖 自定义 AI 代理：在工作空间内支持网页浏览、代码运行等功能。

🌐 嵌入式聊天小部件：支持将聊天功能嵌入到网站中（仅限 Docker 版本）。

📖 多文档类型支持：支持多种文档格式。

🌐 云部署就绪：支持 100% 云部署。

📊 成本与时间优化：内置管理大文档的优化措施。

🔍 开发者 API：支持自定义集成。

🌐 支持多种 LLM 和向量数据库：支持 OpenAI、Azure OpenAI、Anthropic、Google Gemini Pro 等。

#### docker 安装

```bash
export STORAGE_LOCATION=/Applications/Docker-Home/anythingllm/data && \
mkdir -p $STORAGE_LOCATION && \
touch "$STORAGE_LOCATION/.env" && \
docker run -d -p 3001:3001 \
--cap-add SYS_ADMIN \
-v ${STORAGE_LOCATION}:/app/server/storage \
-v ${STORAGE_LOCATION}/.env:/app/server/.env \
-e STORAGE_DIR="/app/server/storage" \
mintplexlabs/anythingllm
```

#### 登录 ui 开启大模型联网功能

Anything LLM 支持自动嗅探 ollama 服务, 部署后, 打开 `http://192.168.137.112:3001/` 登录 ui 界面:

![anything-workspace](/posts/llm_websearch/anything_workspace.png)

在设置中打开网页搜索, 这里选择免费的 `DuckDuckGo`:

![anything-websearch-settings](/posts/llm_websearch/anything_websearch_settings.png)

确认聊天大模型的配置选项:

![anything-websearch-chat](/posts/llm_websearch/anything_websearch_chat.png)

测试一下:

![anything-websearch-qa](/posts/llm_websearch/anything_websearch_qa.png)

## 选用支持联网的模型

这里以`chat box` 为例说明:

### 安装 chat box

> https://web.chatboxai.app/

安装后配置 ollama 域名地址和模型:

![chatbox_ollama](/posts/llm_websearch/chatbox_ollama.png)

可惜了, 离线的 llm 联网功能不支持:

![chatbox_ollama](/posts/llm_websearch/chatbox_network_error.png)

### 第三方联网模型

可以使用第三方集成了搜索引擎的模型和 api:

> https://www.mkeai.com/tutorials/detail/168.html

## 小尾巴

今天介绍了几款开源高星的开源项目, 觉得挺有意思的, 可以选择 `page assist` 作为浏览器插件
来支持联网功能, 也可以选择 `open-webui` 这个项目支持网络查询, 至于 `anything llm `由于搜索是按次数收费的, 免费的`DuckDuckGo`对中文不友好, 可以按需酌情使用。

当然啦, 也可以使用支持联网的模型来搞。
