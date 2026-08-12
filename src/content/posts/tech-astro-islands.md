---
title: Astro 7 架构解析：岛屿架构如何让博客又快又灵活
slug: tech-astro-islands
published: 2026-08-12
updated: 2026-08-12
description: 本站基于 Astro 7 搭建，零 JS 默认输出、岛屿架构按需水合，本文拆解其核心设计理念与实际应用。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - Astro
  - 架构
  - 性能
  - SSG
draft: false
pinned: false
---

> Astro 的核心理念：默认零 JavaScript，只在需要交互的地方加水。

## 为什么选择 Astro

传统 SSR 框架（Next.js、Nuxt）虽然功能强大，但默认会向客户端发送大量 JavaScript 运行时。对于一个以内容为主的博客来说，这些 JS 大部分是多余的——文章页面不需要 React 的虚拟 DOM，不需要 hydration，不需要状态管理。

Astro 的做法截然不同：

1. **默认零 JS**：页面在构建时渲染为纯 HTML + CSS，不发送任何客户端框架代码
2. **岛屿架构**：只在需要交互的组件处"加水"（hydration），其余部分保持静态
3. **多框架共存**：可以在同一个页面混用 React、Vue、Svelte、Solid 组件

本站正是基于这一理念搭建，实现了极快的首屏加载速度。

## 岛屿架构详解

### 什么是岛屿

想象一片海洋（静态 HTML 页面），海面上零星分布着几个岛屿（交互式组件）。每个岛屿是独立的，互不干扰，只有它们需要被"激活"（hydrate）。

```astro
---
// 服务端渲染，零客户端 JS
import PostCard from "@/components/PostCard.astro";
import Search from "@/components/Search.svelte"; // 岛屿
---

<PostCard title="Hello" />          <!-- 纯静态 -->
<Search client:load />               <!-- 岛屿：立即水合 -->
```

### 水合指令

Astro 提供了多种水合指令，控制岛屿何时"激活"：

| 指令 | 行为 | 适用场景 |
|------|------|----------|
| `client:load` | 页面加载后立即水合 | 需要立即可交互的组件（搜索框） |
| `client:idle` | 浏览器空闲时水合 | 非紧急交互组件（设置面板） |
| `client:visible` | 组件进入视口时水合 | 页面下方组件（评论区） |
| `client:media` | 匹配媒体查询时水合 | 响应式组件 |
| `client:only` | 跳过服务端渲染 | 纯客户端组件 |

本站的使用策略：
- **主题切换**：`client:load`（首屏即用）
- **显示设置面板**：`client:idle`（不阻塞首屏）
- **动态信息流**：`client:visible`（滚到才加载）

## 内容集合（Content Collections）

Astro 7 的内容集合提供了类型安全的内容管理：

```typescript
// src/content/config.ts
import { defineCollection, z } from "astro:content";

const blog = defineCollection({
  type: "content",
  schema: z.object({
    title: z.string(),
    published: z.coerce.date(),
    description: z.string(),
    image: z.string().optional(),
    tags: z.array(z.string()).default([]),
    draft: z.boolean().default(false),
  }),
});

export const collections = { blog };
```

这带来了三个好处：
1. **类型安全**：frontmatter 字段在构建时校验
2. **自动补全**：IDE 中获取字段提示
3. **查询 API**：通过 `getCollection` 类型安全地查询内容

## SSG 构建流程

本站采用静态生成（SSG）模式：

```text
Markdown/MDX → Astro 编译 → 静态 HTML → CDN 分发
```

构建产物是纯静态文件，可以部署到任何静态托管平台：
- Cloudflare Pages
- Vercel
- Netlify
- GitHub Pages

没有服务器运行时，没有数据库查询，没有 SSR 延迟。页面加载速度只受网络和 CDN 影响。

## 性能数据

得益于 Astro 的设计，本站的关键性能指标：

| 指标 | 数值 |
|------|------|
| 首屏 JS 体积 | < 50KB（gzip） |
| LCP | < 1.5s |
| CLS | ~0 |
| Lighthouse 性能分 | 95+ |

对比传统 React SSR 博客（动辄 200KB+ JS），Astro 的优势在内容站点上非常明显。

## 结语

Astro 7 的岛屿架构证明了一件事：不是所有页面都需要是一个"应用"。对于博客、文档、营销页这类内容驱动的站点，少即是多——少一点 JavaScript，多一点用户体验。

本站正是这一理念的实践。
