---
title: '从 RAG 到 Deep Research: 大模型又出大招?'
tags: ['RAG', 'Deep Research']
categories: ['实战', '大模型', 'RAG', 'Deep Research']
series: ['llm']
author: ['zhu733756']
date: 2025-02-24T15:41:59+08:00
---

## 前戏

> 随着人工智能技术的飞速发展, 大语言模型（LLM）逐渐成为研究和应用的热点。从最初的检索增强生成（RAG）技术到如今的深度研究（Deep Research）, LLM 技术经历了多次重要的迭代与演进。本文将带您一探究竟, 回顾这一技术的发展历程。

## RAG 技术的兴起与发展

### RAG 的起源与原理

检索增强生成（Retrieval-Augmented Generation, RAG）技术最早于 2021 年被提出。

RAG 的核心思想, 是通过检索外部知识库中的相关信息, 增强语言模型的生成能力。RAG 技术的出现, 为 LLM 的发展奠定了重要的基础, 一定程度上解决了大模型由于私域知识不足, 往往胡编乱造的现象。

### RAG 的范式演变

最初, RAG 主要被用于 LLM 的预训练阶段, 随后逐渐扩展到微调与推理任务。

2024 年, RAG 技术出现了多种新的范式:

- Naive RAG: 最基础的检索增强生成方法, 直接将检索到的信息与模型输入结合。
- Advanced RAG: 引入更复杂的检索策略和生成机制, 提升模型性能。
- Modular RAG: 将 RAG 系统模块化, 使其更灵活地适应不同任务。
- Graph RAG: 融合知识图谱, 进一步增强模型对知识的结构化理解和推理能力。
- Agentic RAG: 作为最新的范式, Agentic RAG 通过集成自主 AI 代理, 实现了动态管理检索策略、迭代细化上下文理解, 并适应性地调整工作流程。

### LLM 技术的快速迭代

#### 模型架构的创新

随着 RAG 技术的不断发展, LLM 本身也在架构上进行了多次创新。例如, DeepSeek-V3 采用了基于 MoE（Mixture of Experts）架构的设计, 总参数量达到 671B。这种架构通过激活部分参数, 既提升了模型性能, 又降低了计算成本。此外, DeepSeek-V3 还引入了无辅助损失的负载均衡策略和多 token 预测训练目标, 进一步优化了模型的训练效率。

#### 训练技术的突破

在训练技术方面, LLM 的发展同样令人瞩目。例如, FP8 混合精度训练技术被首次应用于超大规模模型的训练中。这种低精度训练方案不仅显著提升了训练速度, 还降低了 GPU 内存占用, 为模型的高效训练提供了有力支持。

## 从 RAG 到 Deep Research 的深度演进

### 什么是 Deep Research?

> Deep Research 是近期由 OpenAI 提出的, 详见https://openai.com/index/introducing-deep-research/。

Deep Research, 顾名思义, 就是深度研究的意思。 它旨在通过多步骤的检索和分析, 逐步解决复杂问题。它模拟人类的研究过程, 能够根据新发现调整研究计划, 支持跨多文档、多学科的查询, 同时强调深度思考和复杂任务处理, 能够进行多轮迭代检索, 直到达到研究目的。整个探索过程, 就好像在做研究一样。

举个例子, 假设你正在研究一个历史人物的生平, 具体问题是: "爱德华·G·罗宾逊（Edward G. Robinson）在哪里上的大学？"

RAG 的处理方式:

- 直接根据问题从知识库中检索相关信息, 再通过大模型进行回答。
- 如果知识库中没有直接提到爱德华·G·罗宾逊的大学信息, RAG 可能无法给出准确答案。

Deep Research 的处理方式:

- 分解问题: 先检索"爱德华·G·罗宾逊是谁", 确定他是著名的演员。
- 进一步检索: 根据第一步的结果, 检索"爱德华·G·罗宾逊的教育背景"。
- 整合信息: 最终找到答案"爱德华·G·罗宾逊在纽约大学学习过"。

## 有哪些开源实现?

以下是目前一些流行的 Deep Research 开源解决方案, 这些项目旨在复现或替代 OpenAI 的 Deep Research 功能, 提供免费且开源的深度研究工具:

### Deep Searcher

简介：Deep Searcher 是一个基于 Deep Research 思路的本地部署开源方案, 适用于企业级场景。

功能:

- 支持多种 LLM（如 DeepSeek、OpenAI）。
- 支持多种 Embedding 模型（如 OpenAI Embedding、Pymilvus 模型）。
- 支持离线和在线文档加载（如 PDF、Markdown、通过 FireCrawl 获取的网页内容）。
- 集成向量数据库（如 Milvus、Zilliz Cloud）。

> https://github.com/zilliztech/deep-searcher。

### Open Deep Research

简介：这是一个开源复现版的 Deep Research, 专注于利用网络数据完成复杂的多步骤研究任务。

功能：

- 使用 Firecrawl 搜索和提取数据, 支持多源数据整合。
- 基于强大的推理模型（如 GPT-4o、Anthropic 等）进行深度分析。
- 提供统一的 API 和 Next.js 应用框架, 支持实时数据输入和服务器端渲染。

> https://github.com/nickscamara/open-deep-research。

### node-DeepResearch

简介：由 Jina AI 提供的开源深度研究工具, 专注于通过迭代搜索提供精准答案。

功能：

- 基于 Node.js 架构, 结合 Jina Reader 进行网页搜索和阅读。
- 通过循环迭代深入调查复杂问题, 直到找到答案。
- 支持 OpenAI 和谷歌 Gemini 等推理模型。

> https://github.com/jina-ai/node-DeepResearch。

### OpenDeepResearcher

简介：这是一个开源的 AI 研究工具, 模拟人类的研究过程, 通过迭代搜索和信息整合生成研究报告。

功能：

- 用户输入主题后, AI 自动进行网络搜索、页面查看和信息提取。
- 支持多轮迭代搜索, 直到信息足够充分或达到预设次数上限。
- 最终生成详细的研究报告。

> https://github.com/mshumer/OpenDeepResearcher。

### Open Deep Research

简介：Hugging Face 团队快速复现的开源版本, 能够自主浏览网页、处理文件并进行计算。

功能：

- 自主浏览网页, 滚动页面, 处理文件。
- 在 GAIA 基准测试中表现良好, 准确率 55%。

> https://github.com/huggingface/smolagents/tree/main/examples/open_deep_research

### deep-research

简介：这是一个基于 OpenAI Deep Research 概念的开源项目, 允许用户调整研究的广度和深度。

功能：

- 支持并行运行多个研究线程。
- 根据新发现不断迭代, 直到达到研究目标。
- 运行时间可从 5 分钟到 5 小时自动调整。

> https://github.com/dzhng/deep-research。

## 小尾巴

> 从 RAG 到 Deep Research 的深度演进, 是 LLM 技术在近期的快速迭代趋势。本文探讨了一些开源 Deep Research 实现, 后面有机会我们跑一下实例。
