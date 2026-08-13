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


# 写在前面<br>
一个月前整了个基于opencode的玩具LLM源码审计系统[https://wsxk.github.io/ai_audit/](https://wsxk.github.io/ai_audit/)<br>
业界也主要在探究这部分内容,玩具审计系统已经不足以支撑实际使用了，还是需要整点好的<br>
btw,llm辅助二进制逆向挖洞这个方向已经被pass掉了（还无法完整分析大型程序）<br>

# 前期调研<br>
[https://github.com/purpleroc/llm_code_audit/blob/main/llm-code-audit-report.md](https://github.com/purpleroc/llm_code_audit/blob/main/llm-code-audit-report.md)<br>
这篇文档，虽然是chatgpt写的，但是实际上某种程度上也说明了当前llm辅助源码的探究方向。<br>

[https://x.com/sujingshen/article/2048278721107530231](https://x.com/sujingshen/article/2048278721107530231)<br>
这篇文章里提到当前AI系统






<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>