---
title: Pagefind 静态搜索：零服务器实现全文检索
slug: tech-pagefind-search
published: 2026-08-12
updated: 2026-08-12
description: 本站使用 Pagefind 实现全文搜索，构建时索引、客户端检索，零服务器成本，毫秒级响应。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - Pagefind
  - 搜索
  - 静态站点
  - 全文检索
draft: false
pinned: false
---

> 搜索功能是内容站点的刚需，但传统方案要么需要服务器，要么依赖第三方服务。Pagefind 提供了第三条路。

## 搜索方案对比

博客需要搜索功能时，通常有这几种选择：

| 方案 | 优点 | 缺点 |
|------|------|------|
| Algolia | 功能强大、速度快 | 需要注册、有用量限制、数据需上传 |
| Lunr.js | 纯客户端、无依赖 | 索引体积大、中文支持差 |
| ElasticSearch | 功能最全 | 需要服务器、运维成本高 |
| **Pagefind** | **零服务器、索引小、中文友好** | **需要静态构建** |

Pagefind 的独特之处：在构建时为静态 HTML 创建索引，索引文件随站点一起部署到 CDN，客户端直接加载索引进行搜索。

## 工作原理

```text
构建阶段：
  HTML 页面 → Pagefind 爬取 → 提取文本 → 建立倒排索引 → 写入 pagefind/ 目录

运行时：
  用户输入关键词 → 客户端加载索引分片 → 本地检索 → 返回匹配结果
```

关键设计：
1. **索引分片**：索引被切成多个小文件，按需加载，不是一次性下载全部
2. **倒排索引**：经典搜索算法，通过词项快速定位文档
3. **增量加载**：先加载索引摘要，搜索时再加载具体分片

## 构建集成

本站在 `pnpm build` 时自动执行 Pagefind 索引：

```json
{
  "scripts": {
    "build": "astro build && pagefind --site dist"
  }
}
```

构建流程：
1. Astro 生成静态 HTML 到 `dist/`
2. Pagefind 扫描 `dist/` 中的 HTML 文件
3. 提取可搜索内容（标题、正文、元数据）
4. 生成索引文件到 `dist/pagefind/`
5. 随站点一起部署

## 内容标记

Pagefind 默认会索引所有 HTML 中的文本，但可以通过 `data-pagefind-*` 属性精细控制：

```html
<!-- 标记为可搜索内容 -->
<article data-pagefind-body>
  <h1 data-pagefind-meta="title">文章标题</h1>
  <time data-pagefind-meta="date">2026-08-12</time>
  <div data-pagefind-body>
    文章正文内容...
  </div>
</article>

<!-- 排除某些内容 -->
<nav data-pagefind-ignore>
  导航栏不需要被索引
</nav>

<!-- 自定义过滤器 -->
<article data-pagefind-body data-pagefilter="category:tech">
  技术文章
</article>
```

本站的标记策略：
- **文章页面**：`data-pagefind-body` 标记正文区域
- **导航/侧边栏**：`data-pagefind-ignore` 排除
- **元数据**：标题、日期、分类、标签都通过 `data-pagefind-meta` 索引

## 搜索 UI

本站的搜索组件（Search.svelte）是一个 Svelte 岛屿：

```svelte
<script lang="ts">
import { onMount } from "svelte";

let query = $state("");
let results = $state<SearchResult[]>([]);
let pagefind: any = null;

onMount(async () => {
  // 动态加载 Pagefind
  pagefind = await import('/pagefind/pagefind.js');
  await pagefind.init();
});

async function search() {
  if (!query.trim() || !pagefind) {
    results = [];
    return;
  }
  const searchResult = await pagefind.search(query);
  results = await Promise.all(
    searchResult.results.slice(0, 10).map(r => r.data())
  );
}
</script>

<input
  type="text"
  bind:value={query}
  oninput={search}
  placeholder="搜索文章..."
/>

{#each results as result}
  <a href={result.url}>
    <h3>{result.meta.title}</h3>
    <p>{result.excerpt}</p>
  </a>
{/each}
```

关键设计：
- **动态导入**：只在需要搜索时才加载 Pagefind JS
- **实时搜索**：输入时即时检索，无需按回车
- **结果限制**：只显示前 10 条，避免渲染过多 DOM

## 中文搜索支持

Pagefind 内置了对中文的支持，不需要额外配置分词器：

```text
输入：Astro 岛屿
索引：自动按字符和词组建立索引
结果：匹配包含"Astro"或"岛屿"的文章
```

它使用 Unicode 分段算法处理 CJK 字符，效果比 Lunr.js 的简单分词好很多。

## 索引体积

本站的索引体积：

| 内容 | 体积 |
|------|------|
| pagefind.js（搜索引擎） | ~10KB（gzip） |
| 索引分片（10篇文章） | ~30KB |
| 索引摘要 | ~2KB |

总计约 42KB，而且：
- 搜索引擎 JS 只在打开搜索框时加载
- 索引分片按需加载，不搜索不下载
- 搜索"Astro"只加载包含该词的分片

## 高级特性

### 搜索过滤器

```javascript
// 只搜索技术分类
const result = await pagefind.search("Astro", {
  filters: { category: ["技术栈"] }
});
```

### 搜索权重

通过 `data-pagefind-weight` 调整内容权重：

```html
<h1 data-pagefind-weight="10">标题权重更高</h1>
<p>正文权重默认</p>
```

### 结果摘要

Pagefind 自动生成搜索结果摘要，高亮匹配关键词：

```javascript
const result = await pagefind.search("Astro");
const data = await result.results[0].data();
// data.excerpt 包含高亮的摘要 HTML
```

## 性能数据

| 指标 | 数值 |
|------|------|
| 索引构建时间（10篇文章） | <1s |
| 搜索响应时间 | <50ms |
| 索引总体积 | ~42KB |
| 首次搜索加载时间 | <200ms |
| 后续搜索 | <50ms（索引已缓存） |

## 与 Swup 的配合

本站使用 Swup 做页面切换，搜索组件需要特殊处理：

```svelte
<script lang="ts">
onMount(() => {
  // Pagefind 只加载一次
  if (!window.__pagefind) {
    import('/pagefind/pagefind.js').then(pf => {
      pf.init();
      window.__pagefind = pf;
      pagefind = pf;
    });
  } else {
    pagefind = window.__pagefind;
  }
});
</script>
```

将 Pagefind 实例缓存在 `window` 上，Swup 页面切换后不会重复加载。

## 结语

Pagefind 是静态站点搜索的最佳选择之一。零服务器成本、毫秒级响应、中文友好、索引体积小，完美契合 Astro SSG 架构。

本站的搜索功能完全由 Pagefind 驱动，用户可以在搜索框中输入关键词，实时获得全文检索结果，而这一切都不需要任何服务器端支持。
