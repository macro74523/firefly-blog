---
title: Svelte 5 岛屿实践：用最少的代码实现最活的交互
slug: tech-svelte-islands
published: 2026-08-12
updated: 2026-08-12
description: 本站交互组件全部基于 Svelte 5 Runes 语法构建，编译时优化、无虚拟 DOM、极小体积，本文分享实际组件中的实践经验。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - Svelte
  - 前端
  - 交互
  - 编译
draft: false
pinned: false
---

> Svelte 的哲学：框架不应该存在于运行时，而应该在编译时消失。

## 为什么选 Svelte 做岛屿

Astro 支持多种框架作为岛屿，本站选择了 Svelte 5，原因有三：

1. **编译时优化**：Svelte 在构建时将组件编译为原生 DOM 操作，没有虚拟 DOM 的运行时开销
2. **极小体积**：单个岛屿组件的水合代码通常只有几 KB
3. **语法简洁**：Svelte 的单文件组件（SFC）写法非常直观，学习成本低

## Svelte 5 Runes 语法

Svelte 5 引入了 Runes，替代了之前的 `export let` 和 `$:` 语法：

### 响应式状态

```svelte
<script lang="ts">
// 旧语法
// let count = 0;
// $: doubled = count * 2;

// 新语法（Runes）
let count = $state(0);
let doubled = $derived(count * 2);

function increment() {
  count++;
}
</script>

<button onclick={increment}>
  {count} × 2 = {doubled}
</button>
```

### 副作用

```svelte
<script lang="ts">
let hue = $state(180);

// $effect 替代了旧的 $: 副作用
$effect(() => {
  document.documentElement.style.setProperty('--hue', `${hue}`);
});
</script>
```

本站的显示设置面板（DisplaySettingsIntegrated.svelte）大量使用了 Runes：

```svelte
let wallpaperMode = $state(backgroundWallpaper.mode);
let overlayOpacity = $state(getDefaultOverlayOpacity());
let bannerSettingsIsDefault = $derived(
  (!isWavesSwitchable || wavesEnabled === defaultWavesEnabled) &&
  (!isGradientSwitchable || gradientEnabled === defaultGradientEnabled)
);
```

## 实际组件：主题切换

本站的主题切换组件（LightDarkSwitch.svelte）是一个典型的 Svelte 岛屿：

```svelte
<script lang="ts">
let mode = $state(LIGHT_MODE);
let displayedMode = $state(LIGHT_MODE);

function cycleScheme() {
  const cycleOrder = [LIGHT_MODE, DARK_MODE, SYSTEM_MODE];
  const nextIndex = (cycleOrder.indexOf(mode) + 1) % cycleOrder.length;
  mode = cycleOrder[nextIndex];
  setTheme(mode);
  updateDisplayedMode();
}
</script>

<button onclick={cycleScheme}>
  {#if mode === LIGHT_MODE}
    <SunIcon />
  {:else if mode === DARK_MODE}
    <MoonIcon />
  {:else}
    <SystemIcon />
  {/if}
</button>
```

这个组件编译后的客户端代码不到 3KB，却实现了完整的主题循环切换逻辑。

## 实际组件：布局切换

布局切换按钮（LayoutSwitchButton.svelte）展示了 Svelte 与 localStorage 的配合：

```svelte
<script lang="ts">
let currentLayout = $state<"list" | "grid">("list");
let isSwitching = $state(false);

function switchLayout() {
  if (!mounted || isSmallScreen || isSwitching) return;
  isSwitching = true;
  currentLayout = currentLayout === "list" ? "grid" : "list";
  localStorage.setItem("postListLayout", currentLayout);

  // 通过 CustomEvent 通知页面其他部分
  window.dispatchEvent(new CustomEvent("layoutChange", {
    detail: { layout: currentLayout }
  }));

  setTimeout(() => isSwitching = false, 500);
}
</script>
```

关键设计：
- **事件驱动**：通过 `CustomEvent` 解耦组件与页面逻辑
- **防抖**：`isSwitching` 状态防止快速连点
- **持久化**：`localStorage` 保存用户偏好

## 条件渲染与列表

Svelte 的条件渲染语法比 React 的三元表达式更清晰：

```svelte
{#if loading}
  <SkeletonLoader />
{:else if failed}
  <ErrorMessage />
{:else}
  {#each entries as entry (entry.id)}
    <DynamicItem {entry} />
  {/each}
{/if}
```

动态信息流组件（DynamicFeed.svelte）正是用这种方式管理了加载骨架屏、空状态、错误状态和正常列表四种 UI。

## 生命周期与 Swup 集成

Astro 使用 Swup 做页面切换，Svelte 组件需要处理重新挂载的场景：

```svelte
<script lang="ts">
import { onMount } from "svelte";

onMount(() => {
  // 组件首次挂载
  const storedTheme = getStoredTheme();
  mode = storedTheme;

  // 监听 Swup 页面切换
  const handleContentReplace = () => {
    mode = getStoredTheme();
  };

  if (window.swup?.hooks) {
    window.swup.hooks.on("content:replace", handleContentReplace);
  }

  return () => {
    // 清理函数
  };
});
</script>
```

## 样式隔离

Svelte 的 `<style>` 标签默认是作用域隔离的，不会泄漏到全局：

```svelte
<style>
  .theme-switch-btn {
    /* 只作用于当前组件 */
  }

  /* 需要全局时用 :global() */
  :global(.dark) .theme-switch-btn {
    /* 暗色模式下的样式 */
  }
</style>
```

这与 Astro 的 `<style>` 行为一致，两者配合使用时不会冲突。

## 性能对比

同样实现一个主题切换按钮：

| 框架 | 客户端 JS（gzip） | 运行时开销 |
|------|-------------------|------------|
| React 19 | ~45KB | 虚拟 DOM diff |
| Vue 3 | ~35KB | 响应式代理 |
| Svelte 5 | ~3KB | 编译时消除 |

对于只需要一个按钮的岛屿，Svelte 的体积优势极为明显。

## 结语

Svelte 5 + Astro 的组合，是当前内容站点最轻量的交互方案之一。Runes 语法让响应式更加显式和可预测，编译时优化消除了运行时开销。

本站的所有交互组件——主题切换、布局切换、显示设置、动态信息流——都基于这一组合构建，总体客户端 JS 体积控制在 50KB 以内。
