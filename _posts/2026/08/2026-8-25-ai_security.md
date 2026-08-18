---
layout: post
tags: [AI]
title: "LLM 源码审计 研究"
author: wsxk
date: 2026-08-25
comments: true
---

- [写在前面](#写在前面)
- [前期调研](#前期调研)
  - [评测标准调研](#评测标准调研)
- [1. LLM源码审计](#1-llm源码审计)
  - [1.1 静态分析工具](#11-静态分析工具)
  - [1.2 代码检索mcp工具](#12-代码检索mcp工具)
  - [1.3 动态验证技术](#13-动态验证技术)
  - [1.4 RAG与知识库](#14-rag与知识库)


# 写在前面<br>
一个月前整了个基于opencode的玩具LLM源码审计系统[https://wsxk.github.io/ai_audit/](https://wsxk.github.io/ai_audit/)<br>
业界也主要在探究这部分内容,玩具审计系统已经不足以支撑实际使用了，还是需要整点好的<br>
btw,llm辅助二进制逆向挖洞这个方向已经被pass掉了（还无法完整分析大型程序）<br>

# 前期调研<br>
[https://github.com/purpleroc/llm_code_audit/blob/main/llm-code-audit-report.md](https://github.com/purpleroc/llm_code_audit/blob/main/llm-code-audit-report.md)<br>
这篇文档，虽然是chatgpt写的，但是实际上某种程度上也说明了当前llm辅助源码的探究方向。<br>

[https://x.com/sujingshen/article/2048278721107530231](https://x.com/sujingshen/article/2048278721107530231)<br>
这篇文章里提到当前AI系统上下文1M tokens级别，虽然能够容纳20-40万行代码，但是 **context rot（上下文衰减）问题严重** ，大模型经常忽略细节，且装入上下文后并不等于推理完成。<br>
即便用了RAG加强代码检索，也无法解决状态追踪、隐含依赖发现、缺失检测等关键审计问题。<br>
当前流行的agentic模式，是让AI根据需要读代码，也只能沿着搜索路径走，无法系统性遍历整个代码库。<br>

## 评测标准调研<br>

[https://github.com/scaleapi/SWE-bench_Pro-os](https://github.com/scaleapi/SWE-bench_Pro-os)<br>
似乎看起来是有点不错的benchmark<br>


# 1. LLM源码审计<br>
目前看下来，`LLM源码审计`中可以用到的技术有： `RAG与知识库`，`代码检索mcp工具`、`静态分析工具`、`动态验证技术`四种。<br>
值得一提的是，目前还没有一个统一的分析，说明这些技术使用后能够让模型识别源码问题的效率更高，精度更准。<br>

## 1.1 静态分析工具<br>

[https://github.com/opengrep/opengrep](https://github.com/opengrep/opengrep)<br>

## 1.2 代码检索mcp工具<br>
[https://github.com/colbymchenry/codegraph](https://github.com/colbymchenry/codegraph)<br>


## 1.3 动态验证技术<br>
容器、或者给agent提供一个可运行验证的环境+skill（使用方法）+mcp（使用工具）即可<br>

## 1.4 RAG与知识库<br>
[https://github.com/nashsu/llm_wiki](https://github.com/nashsu/llm_wiki)<br>





<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>