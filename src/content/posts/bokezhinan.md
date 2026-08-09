---
title: 博客使用指南
slug: blog-guide
published: 2026-08-09 00:00:00
updated: 2026-08-09 00:00:00
description: 从文章写作到布局配置，再到内部链接，一份覆盖 Firefly 博客日常使用的完整指南。
image: api
category: 博客指南
tags: [Firefly, 博客, 指南, 布局, Markdown]
draft: false
pinned: true                                  # 置顶
---

本站基于 [Firefly](https://github.com/CuteLeaf/Firefly) 主题搭建。本文整理了日常使用中最常用的三块内容：文章写作、布局配置和内部链接，方便随时查阅。

## 一、文章的 Front-matter

每篇文章顶部使用 YAML 格式的 Front-matter 描述元数据，常见字段如下：

```yaml
---
title: 我的第一篇博客文章        # 文章标题（必填）
published: 2026-08-09 12:00:00   # 发布时间
updated: 2026-08-09 12:00:00     # 更新时间
description: 这是文章的一句话摘要  # 列表卡片和搜索引擎的描述
image: ./cover.jpg               # 文章封面图，填 "api" 启用随机封面
slug: hello-world                # 自定义 URL 路径
category: 博客指南               # 分类
tags: [Firefly, 博客]            # 标签数组
draft: false                     # 是否为草稿
pinned: true                     # 是否置顶
---
```

> 提示：`published` 和 `updated` 使用 `YYYY-MM-DD HH:mm:SS` 格式；`tags` 既可写成 `[A, B]` 数组，也可写成 YAML 列表。

## 二、文章文件的放置位置

所有文章统一放在 `src/content/posts/` 目录下，支持两种形式：

```
src/content/posts/
├── post-1.md            # 单文件形式
└── post-2/              # 目录形式（适合带本地封面图的文章）
    ├── cover.png
    └── index.md
```

- **单文件**：适合纯文字、封面用网络图片或随机封面的文章。
- **目录形式**：当文章需要引用本地图片资源时使用，封面路径写相对路径 `./cover.png` 即可。

## 三、自定义文章 URL（Slug）

Slug 是文章 URL 路径的自定义部分。如果不设置 `slug`，系统会使用文件名作为 URL。

### 示例 1：使用文件名作为 URL

```yaml
---
title: 我的第一篇博客文章
published: 2026-08-09
---
```

- 文件：`src/content/posts/my-first-blog-post.md`
- URL：`/posts/my-first-blog-post`

### 示例 2：自定义 Slug

```yaml
---
title: 我的第一篇博客文章
published: 2026-08-09
slug: hello-world
---
```

- 文件：`src/content/posts/my-first-blog-post.md`
- URL：`/posts/hello-world`

### 示例 3：中文文件名配合 Slug

```yaml
---
title: 如何使用 Firefly 博客主题
published: 2026-08-09
slug: how-to-use-firefly-blog-theme
---
```

- 文件：`src/content/posts/如何使用Firefly博客主题.md`
- URL：`/posts/how-to-use-firefly-blog-theme`

### Slug 使用建议

1. **使用英文和连字符**：`my-awesome-post` 而不是 `my awesome post`
2. **保持简洁**：避免过长的 slug
3. **具有描述性**：让 URL 能够反映文章内容
4. **避免特殊字符**：只使用字母、数字和连字符
5. **保持一致**：整个博客使用相似的命名模式

### 注意事项

- Slug 一旦发布，建议不要随意更改，以免影响 SEO 和已存在的链接
- 多篇文章使用相同的 slug，后面的会覆盖前面的
- Slug 会自动转换为小写

## 四、布局系统概览

Firefly 的布局由两个维度共同决定：**侧边栏布局** 和 **文章列表布局**，二者可在配置文件中自由组合。

### 4.1 侧边栏布局

侧边栏用于展示导航、分类、标签、统计等辅助内容，支持两种模式。

#### 单侧边栏（position: "left" / "right"）

- 侧边栏固定在页面某一侧，阅读区更宽敞
- 适合传统博客风格、强调导航与分类的站点
- 配置位于 `src/config/sidebarConfig.ts`：

```ts
export const sidebarLayoutConfig: SidebarLayoutConfig = {
  enable: true,
  position: "left",                    // 左侧边栏
  showBothSidebarsOnPostPage: true,    // 文章详情页是否显示双侧边栏
};
```

> `showBothSidebarsOnPostPage` 适用于「主页用单侧栏，但文章详情页想用对侧栏目录」的场景。

#### 双侧边栏（position: "both"）

- 左右两侧同时存在侧边栏，主内容居中
- 最大化利用屏幕空间，适合宽屏与信息密集型博客

```ts
export const sidebarLayoutConfig: SidebarLayoutConfig = {
  enable: true,
  position: "both",                    // 双侧边栏
};
```

### 4.2 文章列表布局

文章列表是首页和归档页的核心，支持列表与网格两种模式，配置位于 `src/config/siteConfig.ts`。

#### 列表模式（defaultMode: "list"）

- 单列纵向排列，封面图可配置在左侧或右侧
- 摘要展示更充分，适合深度阅读

```ts
export const siteConfig: SiteConfig = {
  postListLayout: {
    defaultMode: "list",
    coverPosition: "right",            // 封面位置："right" 或 "left"
  },
};
```

#### 网格模式（defaultMode: "grid"）

- 列数根据浏览器宽度自适应
- 信息密度高，适合快速浏览

```ts
export const siteConfig: SiteConfig = {
  postListLayout: {
    defaultMode: "grid",
    grid: {
      masonry: true,                   // 开启瀑布流
      columnWidth: 320,                // 卡片最小宽度(px)
    },
  },
};
```

#### 瀑布流（Masonry）

网格模式内置智能瀑布流，自动将卡片放置到最短的列，消除垂直方向的空白间隙，尤其适合图文混合、卡片高度不一致的场景。

### 4.3 布局组合参考

| 侧边栏模式 | 文章列表模式 | 适用场景 |
|---|---|---|
| 单侧边栏 | 列表模式 | 摄影、设计、生活类博客，强调图片和沉浸感 |
| 单侧边栏 | 网格模式 | 技术、笔记类博客，平衡阅读与检索效率 |
| 双侧边栏 | 列表模式 | 需要展示大量侧边栏信息的站点 |
| 双侧边栏 | 网格模式 | 极客风格，追求最高信息密度 |

### 4.4 响应式行为

布局系统会随屏幕尺寸自动调整：

1. **网格列数自动减少**：由 `columnWidth` 和容器宽度决定，屏幕越窄列数越少
2. **列表 → 网格**：屏幕宽度小于 380px 时，列表模式自动切换为网格模式
3. **双侧 → 单侧**：屏幕宽度小于 1280px 时，根据 `tabletSidebar` 配置显示单侧边栏，文章目录导航切换为浮动目录

## 五、Wiki Link 内部链接

Firefly 支持在 Markdown、MDX 中使用 Obsidian 风格的 `[[...]]` 内部链接，链接目标填写文章的 slug 或文件路径（无需扩展名）。

### 5.1 文章链接卡片

`[[slug]]` 单独成段时，会自动读取目标文章的标题、描述、发布时间、分类、标签和封面，渲染为链接卡片：

```
[[firefly]]

[[guide/index]]
```

### 5.2 行内链接

`[[slug]]` 出现在正文中间时，渲染为普通链接，链接文字自动使用目标文章的标题：

```
请参阅 [[firefly]] 了解主题特性。
```

### 5.3 自定义显示标题

在 `|` 后填写显示文字。行内链接会用它替换文章标题；单独成段时仍渲染为卡片，卡片标题使用自定义文字：

```
请参阅 [[firefly|主题介绍]] 了解主题特性。

[[firefly|Firefly 主题介绍]]
```

> 例外：如果 `|` 后的文字只是把链接目标又抄了一遍（如 `[[guide/index|index]]`），会被当作无效别名忽略，仍显示文章标题。

### 5.4 链接目标的三种写法

把 `src/content/posts` 目录作为 Obsidian 仓库打开时，「仓库根目录」即为此目录，也是 Firefly 解析链接路径的起点。链接目标按以下顺序匹配：

| 写法 | 示例 | 在 Obsidian 中 |
|---|---|---|
| frontmatter 的 `slug` | `[[blog-guide]]` | 不支持 |
| 文件路径（相对仓库根目录） | `[[guide/firefly-layout-system]]` | 需改设置（推荐） |
| 裸文件名（仓库内唯一时） | `[[firefly-layout-system]]` | 默认即是 |

- **slug 写法**：Obsidian 不读取 frontmatter 的 `slug`，按 slug 写的链接在 Obsidian 里点不动，只在构建站点后能跳转。
- **文件路径写法（推荐）**：在 Obsidian `设置 → 文件与链接 → 内部链接类型` 中选择「基于仓库根目录的绝对路径」，插入的链接会自带目录，与 Firefly 路径完全一致。
- **裸文件名写法**：只要文件名在仓库内唯一即可生效，无需任何设置；重名时会失效，构建日志会给出提示。

### 5.5 链接到文章标题

在 slug 后添加 `#标题` 即可链接到文章内的某个标题。带锚点的链接始终渲染为普通链接：

```
[[code-examples#语法高亮|查看代码块语法高亮]]

[[guide/firefly-layout-system#相关链接|firefly-layout-system]]
```

### 5.6 链接到本页标题

省略文章 slug，只填写标题即可链接到当前文章的对应章节：

```
[[#布局系统概览|跳转到布局系统]]
```

### 5.7 不支持的情况

- 附件嵌入语法 `![[image.png]]` 不会被转换，会按原文显示
- 行内代码和代码块中的 `[[firefly]]` 不会被转换

## 相关链接

- [Firefly 官方文档](https://docs-firefly.cuteleaf.cn/)
- [Firefly GitHub 仓库](https://github.com/CuteLeaf/Firefly)
- [侧边栏配置文档](https://docs-firefly.cuteleaf.cn/config/sidebarConfig-usage/)
- [站点配置文档](https://docs-firefly.cuteleaf.cn/config/siteConfig-usage/)
