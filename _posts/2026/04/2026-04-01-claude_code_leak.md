---
layout: post
tags: [AI]
title: "claude code leak分析"
author: wsxk
date: 2026-04-01
comments: true
---

- [1. 起因](#1-起因)
- [2. claude code 代码分析](#2-claude-code-代码分析)
  - [2.1 map文件还原源码](#21-map文件还原源码)
  - [2.2 claude code封号机制逆向](#22-claude-code封号机制逆向)
- [3. 后续](#3-后续)
  - [3.1 claude code gateway](#31-claude-code-gateway)
  - [3.2 claude rust](#32-claude-rust)


# 1. 起因<br>
[https://www.npmjs.com/package/@anthropic-ai/claude-code?activeTab=code](https://www.npmjs.com/package/@anthropic-ai/claude-code?activeTab=code)中泄露了claude-code 的源代码映射文件`cli.js.map`<br>
PS: claude-code的人已经更新并删除了npm包仓库中的`cli.js.map`文件，所以现在点进去网站什么东西都看不到。<br>
`cli.js.map`文件十分重要，它是`JavaScript Source Map 文件`,一般情况下，发布到npm中的`cli.js`并不是程序员开发的原始版本，其发布往往经过如下几个步骤:<br>
```
1. typescript -> javascript
2. 多个文件合并
3. 代码压缩、混淆，通常去除了符号
4. Babel 转译：Babel是JavaScript的编译器，通过Babel可以将新特性代码翻译成符合各个老式浏览器适配的代码。
```
虽然理论上可以还原这些步骤，通常情况还是很难做到的，需要资深逆向人员进行长时间的还原尝试才有可能做到（而且还原效果不敢保证）。<br>
而`cli.js.map`文件，可以理解为开发人员在生成`cli.js`时的调试文件，用于**把压缩/编译后的 cli.js，映射回原始源码，方便调试、报错定位、逆向分析。**<br>
😀，我合理怀疑打包上传npm的步骤已经由ai代劳了，而ai没有关注源码泄露的事情，直接把`cli.js.map`上传到插件包网站了。<br>
PS: claude code的源码还保留在[https://github.com/instructkr/claw-code](https://github.com/instructkr/claw-code)<br>


# 2. claude code 代码分析<br>
## 2.1 map文件还原源码<br>
AI代劳（AI还是太好用了）<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260401000319.png)
顺便附上ai写的提取脚本:<br>
```js
import fs from "node:fs";
import path from "node:path";

const cwd = process.cwd();
const mapPath = path.join(cwd, "cli.js.map");
const outputRoot = path.join(cwd, "claude_src");

function ensureInsideOutput(resolvedPath) {
  const relative = path.relative(outputRoot, resolvedPath);
  if (relative.startsWith("..") || path.isAbsolute(relative)) {
    throw new Error(`Refusing to write outside output directory: ${resolvedPath}`);
  }
}

function normalizeSourcePath(sourcePath, index) {
  const cleaned = sourcePath.replace(/\\/g, "/");
  const withoutPrefix = cleaned.replace(/^(\.\.\/)+/, "");
  const safePath = withoutPrefix || `unknown/source_${index}.txt`;
  return safePath;
}

function main() {
  if (!fs.existsSync(mapPath)) {
    throw new Error(`Source map not found: ${mapPath}`);
  }

  fs.mkdirSync(outputRoot, { recursive: true });

  const raw = fs.readFileSync(mapPath, "utf8");
  const map = JSON.parse(raw);
  const sources = Array.isArray(map.sources) ? map.sources : [];
  const contents = Array.isArray(map.sourcesContent) ? map.sourcesContent : [];

  let written = 0;
  let skipped = 0;

  for (let i = 0; i < sources.length; i += 1) {
    const sourcePath = sources[i];
    const sourceContent = contents[i];

    if (typeof sourcePath !== "string" || typeof sourceContent !== "string") {
      skipped += 1;
      continue;
    }

    const relativePath = normalizeSourcePath(sourcePath, i);
    const destination = path.resolve(outputRoot, relativePath);
    ensureInsideOutput(destination);

    fs.mkdirSync(path.dirname(destination), { recursive: true });
    fs.writeFileSync(destination, sourceContent, "utf8");
    written += 1;
  }

  const summaryPath = path.join(outputRoot, "_extraction_summary.txt");
  const summary = [
    `map: ${mapPath}`,
    `output: ${outputRoot}`,
    `sources: ${sources.length}`,
    `written: ${written}`,
    `skipped: ${skipped}`,
    "",
    "Top-level restored directories:",
    ...Array.from(
      new Set(
        sources
          .filter((entry) => typeof entry === "string")
          .map((entry, index) => normalizeSourcePath(entry, index).split("/")[0]),
      ),
    ).sort(),
    "",
  ].join("\n");

  fs.writeFileSync(summaryPath, summary, "utf8");

  process.stdout.write(
    [
      `Restored ${written} source files to ${outputRoot}`,
      `Skipped ${skipped} entries without inline content`,
      `Summary: ${summaryPath}`,
    ].join("\n"),
  );
}

main();
```

## 2.2 claude code封号机制逆向<br>
前人的研究:<br>
[https://bytedance.larkoffice.com/docx/E2JudVzf7oCNfhxyxaQcZIW1n0g](https://bytedance.larkoffice.com/docx/E2JudVzf7oCNfhxyxaQcZIW1n0g)<br>
让我震惊的是codex的逆向分析，和前人的研究结果非常类似，把我震惊到了！<br>
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260401235826.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260401235847.png)
![](https://raw.githubusercontent.com/wsxk/wsxk_pictures/main/2025-9-25/20260401235903.png)
上报远端的字符分类竟然一致，难以置信了说是:<br>

# 3. 后续<br>
## 3.1 claude code gateway<br>
在人们发现claude code是在前端收集的信息来判断账号是否异常时，很快啊，有人马上就写了claude code的gateway来让claude code官方认为账号是安全稳定的，链接如下：<br>
[https://github.com/motiful/cc-gateway](https://github.com/motiful/cc-gateway)<br>

## 3.2 claude rust<br>
有人用rust语言重写了一边 claude code cli:<br>
[https://github.com/Kuberwastaken/claurst](https://github.com/Kuberwastaken/claurst)<br>


<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-C22S5YSYL7"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-C22S5YSYL7');
</script>