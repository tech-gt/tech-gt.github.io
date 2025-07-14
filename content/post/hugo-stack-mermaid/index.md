---
title: "让 Hugo Stack 主题完美支持 Mermaid 流程图"
date: 2023-07-18
categories:
  - Hugo
  - 技术分享
tags:
  - hugo
  - mermaid
  - 博客搭建
  - 可视化
---

在日常写博客的过程中，流程图、时序图等可视化表达越来越重要。Mermaid 作为一款轻量级的图表语法工具，非常适合用来在 Markdown 里画图。但 Hugo Stack 主题默认并不支持 Mermaid 渲染，本文就来分享一下我是如何让它完美支持 Mermaid 的。

## 问题分析

Hugo 的 Goldmark 渲染器默认不会把 `mermaid` 代码块当作图表处理，而 Stack 主题本身也没有内置 Mermaid 的支持。网上的教程五花八门，有的让你魔改主题，有的让你用第三方插件，但都不太优雅。

## 解决思路

我的目标是：
- 既能用代码块的方式写 Mermaid，也能用短代码（shortcode）
- 尽量不动主题源码，升级主题时不容易冲突
- 配置简单，易于维护

## 配置步骤

### 1. 配置 Goldmark 支持 Mermaid 代码块

编辑 `config/_default/markup.toml`，如下内容：

```toml
[goldmark.extensions.passthrough.delimiters]
block = [['\\[', '\\]'], ['$$', '$$'], ['```mermaid', '```']]
inline = [['\\(', '\\)']]
```

这样 Hugo 就不会转义 mermaid 代码块内容。

### 2. 新建 Mermaid 短代码

在 `layouts/shortcodes/mermaid.html` 新建如下内容：

```html
<div class="mermaid">
{{- .Inner -}}
</div>
```

以后写博客时可以这样用：
<pre class="no-mermaid">
&#123;&#123;&lt; mermaid &gt;&#125;&#125;
graph TD
    A[开始] --> B{判断}
    B -->|是| C[执行]
    B -->|否| D[结束]
    C --> D
&#123;&#123;&lt; /mermaid &gt;&#125;&#125;
</pre>
### 3. 全局引入 Mermaid 脚本

在 `layouts/partials/head/custom.html` 新建如下内容：

```html
<!-- Mermaid Diagram Support -->
<script type="module">
  import mermaid from 'https://cdn.jsdelivr.net/npm/mermaid@10/dist/mermaid.esm.min.mjs';
  mermaid.initialize({ 
    startOnLoad: true,
    theme: 'default',
    flowchart: {
      useMaxWidth: true,
      htmlLabels: true
    }
  });
</script>
```

这样每个页面都会自动加载 Mermaid。

## 踩过的坑

- **重复定义 Goldmark 配置**：一开始我在 `markup.toml` 里多次写了 `[goldmark.extensions.passthrough.delimiters]`，导致 Hugo 报错。正确做法是合并到同一个表里。
- **主题覆盖问题**：Stack 主题有自己的 head 模板，直接加在 `head.html` 里可能会被覆盖。用 `custom.html` 是最保险的。
- **浏览器缓存**：有时候改完配置没生效，多刷新几次或者清空缓存就好了。

## 最终效果

现在无论是用代码块还是短代码，Mermaid 图表都能在博客里完美显示了！

效果如下：

{{< mermaid >}}
graph TD
    A[开始] --> B{判断}
    B -->|是| C[执行]
    B -->|否| D[结束]
    C --> D
{{< /mermaid >}}

