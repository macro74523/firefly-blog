---
title: TailwindCSS v4 样式系统：原子化 CSS 在博客中的实践
slug: tech-tailwindcss-system
published: 2026-08-12
updated: 2026-08-12
description: 本站使用 TailwindCSS v4 构建样式系统，CSS 变量主题、暗色模式、响应式布局一站式搞定，本文分享配置与使用经验。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - TailwindCSS
  - CSS
  - 样式
  - 设计系统
draft: false
pinned: false
---

> TailwindCSS 的核心思想：不要写 CSS，而是用工具类组合出 UI。

## 为什么用 TailwindCSS

传统 CSS 写法有一个痛点：命名。每个元素都需要起一个 class 名，然后到 CSS 文件里写对应的样式。随着项目增长，class 名越来越多，CSS 文件越来越大，死代码越来越多。

TailwindCSS 用另一种方式解决这个问题：提供一组有限的、约定俗成的工具类，直接在 HTML 中组合使用。

```html
<!-- 传统 CSS -->
<div class="card">
  <h2 class="card-title">标题</h2>
</div>

<!-- TailwindCSS -->
<div class="rounded-lg bg-white p-4 shadow-md dark:bg-neutral-800">
  <h2 class="text-lg font-bold text-neutral-900 dark:text-neutral-100">标题</h2>
</div>
```

## CSS 变量主题系统

本站的样式系统建立在 CSS 变量之上，TailwindCSS 负责消费这些变量：

```css
:root {
  --primary: oklch(0.55 0.15 250);
  --card-bg: oklch(1 0 0);
  --content-meta: oklch(0.5 0 0);
  --btn-regular-bg: oklch(0.95 0 0);
  --hue: 250;
}

.dark {
  --card-bg: oklch(0.2 0 0);
  --content-meta: oklch(0.6 0 0);
  --btn-regular-bg: oklch(0.25 0 0);
}
```

在 TailwindCSS 中使用这些变量：

```html
<button class="bg-(--primary) text-(--btn-content)">
  点击我
</button>

<div class="bg-(--card-bg) text-(--btn-content)">
  卡片内容
</div>
```

`bg-(--primary)` 是 TailwindCSS v4 的任意值语法，直接引用 CSS 变量。

## 暗色模式

本站的暗色模式通过 `dark:` 前缀实现，配合 `class` 策略：

```html
<div class="bg-white dark:bg-neutral-800 text-neutral-900 dark:text-neutral-100">
  这段文字在两种模式下都清晰可读
</div>
```

切换暗色模式时，只需在 `<html>` 上添加或移除 `dark` 类：

```javascript
function applyTheme(theme) {
  if (theme === 'dark') {
    document.documentElement.classList.add('dark');
  } else {
    document.documentElement.classList.remove('dark');
  }
}
```

本站支持三种模式：亮色、暗色、跟随系统，通过循环切换实现。

## 主题色动态调节

本站的一个特色功能是主题色动态调节。用户可以通过滑块调整 hue 值，整个站点的主题色实时变化：

```javascript
function setHue(hue) {
  document.documentElement.style.setProperty('--hue', `${hue}`);
}
```

CSS 变量使用 `oklch` 色彩空间，只需改变 hue 分量就能保持一致的亮度和饱和度：

```css
:root {
  --primary: oklch(0.55 0.15 var(--hue));
}
```

这比 RGB 色彩空间更适合做主题色调节，因为 `oklch` 的感知均匀性更好。

## 响应式设计

TailwindCSS 的响应式前缀让移动端适配变得简单：

```html
<!-- 移动端 9x9，桌面端 11x11 -->
<button class="h-9 w-9 md:h-11 md:w-11">
  按钮
</button>

<!-- 移动端单列，桌面端双列 -->
<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
  <div>项目1</div>
  <div>项目2</div>
</div>
```

本站的断点策略：
- 默认：移动端优先
- `sm:` (640px)：大手机
- `md:` (768px)：平板
- `lg:` (1024px)：桌面
- `xl:` (1280px)：大屏

## 常用工具类组合

本站高频使用的工具类模式：

### 卡片基础

```html
<div class="card-base rounded-lg p-4 transition-colors">
  <!-- card-base 是自定义类，包含背景色、边框等基础样式 -->
</div>
```

### 按钮系列

```html
<!-- 普通按钮 -->
<button class="btn-regular rounded-lg px-3 py-1.5 active:scale-95 transition-transform">
  按钮
</button>

<!-- 图标按钮 -->
<button class="btn-plain rounded-lg h-9 w-9 flex items-center justify-center active:scale-90">
  <Icon name="settings" />
</button>
```

### 浮动面板

```html
<div class="float-panel float-panel-closed absolute transition-all right-4 z-50">
  面板内容
</div>
```

### 文字截断

```html
<p class="truncate">很长的文字会被截断...</p>
<p class="line-clamp-2">最多显示两行...</p>
```

## 自定义工具类

对于重复使用的样式组合，可以在 CSS 中定义自定义工具类：

```css
@layer components {
  .card-base {
    @apply rounded-lg bg-(--card-bg) border border-(--border-color);
  }

  .btn-regular {
    @apply bg-(--btn-regular-bg) text-(--btn-content) transition-colors;
  }

  .btn-plain {
    @apply text-(--btn-content) transition-colors;
  }
}
```

这样在 HTML 中就可以用 `card-base` 替代一长串工具类。

## 与 Astro `<style>` 共存

Astro 组件的 `<style>` 标签是作用域隔离的，与 TailwindCSS 配合使用：

```astro
---
// 组件逻辑
---

<div class="card-base rounded-lg p-4">
  <!-- 使用 TailwindCSS 工具类 -->
</div>

<style>
  /* 作用域样式，处理 TailwindCSS 不方便的场景 */
  .custom-animation {
    animation: shimmer 1.4s ease-in-out infinite;
  }

  @keyframes shimmer {
    0% { background-position: -200% 0; }
    100% { background-position: 200% 0; }
  }
</style>
```

原则：
- **布局、间距、颜色**：优先用 TailwindCSS 工具类
- **动画、伪元素、复杂选择器**：用 `<style>` 作用域样式

## 构建优化

TailwindCSS v4 使用 Lightning CSS 引擎，构建速度极快。生产构建时会自动清除未使用的工具类（tree-shaking），最终 CSS 体积通常只有 10-20KB（gzip）。

```bash
# 构建后的 CSS 体积
dist/_astro/*.css  15.3 KB │ gzip: 3.2 KB
```

## 结语

TailwindCSS v4 让本站的样式开发效率提升了数倍。无需在 HTML 和 CSS 文件之间反复跳转，无需纠结 class 命名，直接在 HTML 中用工具类表达样式意图。

配合 CSS 变量主题系统，实现了暗色模式、动态主题色、响应式布局等复杂功能，而最终 CSS 体积控制在极小的范围内。
