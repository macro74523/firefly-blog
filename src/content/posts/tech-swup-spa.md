---
title: Swup SPA 切换：让静态博客拥有应用般的流畅体验
slug: tech-swup-spa
published: 2026-08-12
updated: 2026-08-12
description: 本站使用 Swup 实现无刷新页面切换，配合过渡动画和脚本重载，让 SSG 博客拥有 SPA 般的流畅体验。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - Swup
  - SPA
  - 过渡动画
  - 用户体验
draft: false
pinned: false
---

> 静态站点最大的痛点：每次点击链接都要白屏刷新。Swup 解决了这个问题。

## 为什么需要 Swup

传统的多页站点（MPA）每次点击链接都会触发完整的页面刷新：
1. 浏览器发送请求
2. 等待服务器响应
3. 清空当前页面
4. 解析新 HTML
5. 重新加载 CSS 和 JS
6. 渲染新页面

这个过程会导致白屏闪烁，用户体验断裂。SPA（单页应用）通过客户端路由解决了这个问题，但传统 SPA 需要重写整个架构。

Swup 提供了一个中间方案：在 MPA 之上添加一层客户端路由拦截，只替换页面内容部分，保留头部导航、侧边栏等公共元素，实现无刷新切换。

## 工作原理

```text
用户点击链接
    ↓
Swup 拦截点击事件
    ↓
fetch 获取新页面 HTML
    ↓
解析新 HTML，提取 [data-swup] 区域内容
    ↓
旧内容淡出 → 替换为新内容 → 新内容淡入
    ↓
更新页面标题、滚动位置
    ↓
重新执行内联脚本
```

整个过程不会刷新页面，不会重新加载 CSS 和 JS 外部资源，只替换内容区域。

## Astro 集成

本站使用 `@swup/astro` 集成 Swup：

```javascript
// astro.config.mjs
import swup from '@swup/astro';

export default defineConfig({
  integrations: [
    swup({
      theme: false,
      animationClass: 'transition-',
      containers: ['#swup'],
      cache: true,
      preload: true,
      accessibility: true,
      globalInstance: true,
    }),
  ],
});
```

关键配置：
- `containers: ['#swup']`：只替换 id 为 swup 的容器内容
- `animationClass: 'transition-'`：过渡动画的 CSS 类前缀
- `globalInstance: true`：将 Swup 实例挂载到 window，方便全局访问
- `cache: true`：缓存已访问的页面，加速后退
- `preload: true`：鼠标悬停时预加载目标页面

## 过渡动画

Swup 通过 CSS 类控制过渡动画：

```css
/* 页面切换动画基类 */
html.is-animating .transition-main {
  transition: opacity 300ms ease, transform 300ms ease;
}

/* 离开动画 */
html.is-animating .transition-main {
  opacity: 0;
  transform: translateY(-8px);
}

/* 进入动画 */
html.is-leaving .transition-main {
  opacity: 1;
  transform: translateY(0);
}
```

本站的过渡效果：
- **离开**：内容向上淡出（8px 位移 + 300ms 透明度过渡）
- **进入**：内容从下方淡入
- **缓动**：ease-out，让动画结束时减速

## 脚本处理

Astro 的 `<script is:inline>` 默认只在首次加载时执行。Swup 切换页面后，这些脚本不会重新运行，导致依赖它们的组件失效。

解决方案有两种：

### 方案一：监听 Swup 事件

```javascript
// 在全局脚本中监听
document.addEventListener('swup:contentReplaced', () => {
  // 重新初始化日历、侧边栏等组件
  initCalendar();
  initSidebar();
});

// 或者通过 Swup hooks
window.swup.hooks.on('content:replace', () => {
  // 重新执行
});
```

### 方案二：data-swup-ignore-script

对于不需要重复执行的脚本，添加 `data-swup-ignore-script` 属性：

```html
<script is:inline data-swup-ignore-script>
  // 只执行一次的脚本
</script>
```

本站混合使用两种方案：
- **日历组件**：需要重新初始化 → 监听 `contentReplaced`
- **分析脚本**：只需执行一次 → `data-swup-ignore-script`

## Svelte 组件与 Swup

Svelte 岛屿组件在 Swup 切换时可能被销毁重建，需要正确处理状态同步：

```svelte
<script lang="ts">
import { onMount } from "svelte";

onMount(() => {
  // 从 localStorage 恢复状态
  mode = getStoredTheme();

  // 监听 Swup 页面切换
  const handleContentReplace = () => {
    // 重新读取存储的状态
    mode = getStoredTheme();
    updateDisplayedMode();
  };

  if (window.swup?.hooks) {
    window.swup.hooks.on("content:replace", handleContentReplace);
  } else {
    // Swup 尚未就绪，等待
    document.addEventListener("swup:enable", () => {
      window.swup?.hooks.on("content:replace", handleContentReplace);
    });
  }
});
</script>
```

关键点：
- Swup 可能比组件晚加载，需要处理 `swup:enable` 事件
- 页面切换后从 localStorage 重新读取状态，确保一致性
- 清理函数中移除事件监听，防止内存泄漏

## 预加载优化

Swup 支持鼠标悬停预加载，用户还没点击就开始获取页面：

```javascript
swup({
  preload: true,
});
```

工作流程：
1. 用户鼠标悬停在链接上
2. Swup 在后台 fetch 目标页面
3. 用户点击链接
4. 页面已经缓存，瞬间切换

这让页面切换感觉几乎是零延迟的。

## 缓存管理

Swup 维护一个页面缓存，但某些情况下需要手动清除：

```javascript
// 用户切换主题后，清除缓存
window.swup.cache.clear();

// 或者特定页面
window.swup.cache.delete('/about/');
```

本站在以下场景清除缓存：
- 主题色变化
- 壁纸模式切换
- 布局模式切换

因为这些设置会影响页面渲染，缓存的旧 HTML 不再有效。

## 滚动位置管理

Swup 默认在页面切换时重置滚动位置到顶部。本站的自定义行为：

```javascript
// 切换到新页面时滚动到顶部
window.swup.hooks.on('visit:start', () => {
  window.scrollTo({ top: 0 });
});

// 后退时恢复之前的滚动位置
window.swup.hooks.on('history:popstate', (visit) => {
  // 恢复滚动位置
});
```

## 常见问题

### 问题：切换后 JS 失效

**原因**：内联脚本未重新执行
**解决**：监听 `swup:contentReplaced` 事件，手动重新初始化

### 问题：切换后样式丢失

**原因**：动态添加的 class 被清除
**解决**：在 `content:replace` 钩子中重新添加

```javascript
window.swup.hooks.on('content:replace', () => {
  // 重新应用主题
  const theme = localStorage.getItem('theme');
  document.documentElement.classList.toggle('dark', theme === 'dark');
});
```

### 问题：页面切换闪烁

**原因**：过渡动画时间设置不当
**解决**：确保离开动画完成后再替换内容

## 性能数据

| 指标 | 无 Swup | 有 Swup |
|------|---------|---------|
| 首次加载 | 正常 | 正常（+3KB JS） |
| 页面切换 | 800-1500ms | 200-400ms |
| 重复访问 | 800-1500ms | <100ms（缓存） |
| 白屏闪烁 | 有 | 无 |

## 结语

Swup 让静态博客拥有了 SPA 般的流畅体验，同时保留了 MPA 的简洁架构和 SEO 优势。3KB 的体积换来的是质的体验提升。

本站的所有页面切换都经过 Swup 处理，配合过渡动画和预加载，实现了接近原生的流畅感受。
