---
title: Biome + TypeScript：极速 Lint 与类型安全的工程化实践
slug: tech-biome-typescript
published: 2026-08-12
updated: 2026-08-12
description: 本站使用 Biome 替代 ESLint + Prettier，配合 TypeScript 严格模式，实现极速格式化、Lint 检查和类型安全。
image: "https://jpg.macro.wang/20260812122404861.jpg"
category: 技术栈
tags:
  - Biome
  - TypeScript
  - Lint
  - 工程化
draft: false
pinned: false
---

> 好的工具应该像空气一样：你感受不到它的存在，但它一直在保护你。

## 为什么选 Biome

前端工程化工具链的传统选择是 ESLint + Prettier：
- ESLint 负责代码质量检查（未使用变量、潜在 bug）
- Prettier 负责代码格式化（缩进、引号、换行）

但这个组合有几个问题：
1. **速度慢**：ESLint + Prettier 分别解析 AST，大型项目动辄几十秒
2. **配置复杂**：两个工具的规则可能冲突，需要 `eslint-config-prettier` 关闭冲突项
3. **依赖多**：插件、解析器、配置包加起来几十个 node_modules

Biome（前身 Rome）用一个工具解决了所有问题：

| 工具 | 速度 | 功能 |
|------|------|------|
| ESLint + Prettier | 慢（JS 实现） | Lint + Format |
| Biome | 快 10-30 倍（Rust 实现） | Lint + Format + Import 排序 |

## 配置

本站的 Biome 配置（`biome.json`）：

```json
{
  "$schema": "https://biomejs.dev/schemas/2.0.0/schema.json",
  "files": {
    "ignore": ["dist/**", "node_modules/**", ".astro/**"]
  },
  "formatter": {
    "indentStyle": "tab",
    "indentWidth": 2,
    "lineWidth": 100
  },
  "javascript": {
    "formatter": {
      "quoteStyle": "double",
      "semicolons": "always",
      "trailingCommas": "all"
    }
  },
  "linter": {
    "rules": {
      "recommended": true
    }
  }
}
```

关键配置：
- **缩进**：Tab（与 Astro 默认一致）
- **引号**：双引号（与 Astro/TypeScript 生态一致）
- **分号**：始终使用
- **尾逗号**：全部添加（方便 git diff）
- **规则集**：使用 recommended 预设

## npm 脚本

本站的 package.json 中定义了相关命令：

```json
{
  "scripts": {
    "check": "astro check",
    "type-check": "tsc --noEmit",
    "format": "biome format --write src",
    "lint": "biome check --write src"
  }
}
```

工作流：
1. **写代码**：IDE 自动用 Biome 格式化
2. **提交前**：`pnpm lint` 检查并修复问题
3. **CI**：`pnpm check && pnpm type-check && pnpm build` 全量验证

## TypeScript 严格模式

本站启用了 TypeScript 严格模式：

```json
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "exactOptionalPropertyTypes": true
  }
}
```

`strict: true` 包含了以下检查：
- `noImplicitAny`：禁止隐式 any
- `strictNullChecks`：严格的 null 检查
- `strictFunctionTypes`：严格的函数类型检查
- `strictBindCallApply`：严格的 bind/call/apply 检查
- `strictPropertyInitialization`：严格的属性初始化检查

### 实际受益的例子

```typescript
// 没有 strictNullChecks 时，这段代码编译通过但运行时崩溃
function getUserName(user: User | null) {
  return user.name; // 运行时可能 user 为 null
}

// 有 strictNullChecks 时，编译器报错
function getUserName(user: User | null) {
  if (!user) return "匿名";
  return user.name; // 编译器知道这里 user 不是 null
}
```

## 类型安全的配置系统

本站的配置文件全部有类型定义：

```typescript
// src/types/config.ts
export interface SiteConfig {
  title: string;
  subtitle: string;
  lang: string;
  themeColor: {
    defaultMode: LIGHT_DARK_MODE;
    defaultHue: number;
  };
  navbar: {
    title?: string;
    widthFull?: boolean;
    menuAlign?: "left" | "center" | "right";
  };
  postListLayout: {
    defaultMode: "list" | "grid";
    mobileDefaultMode?: "list" | "grid";
    allowSwitch: boolean;
  };
}

// src/config/siteConfig.ts
export const siteConfig: SiteConfig = {
  title: "Firefly",
  subtitle: "一个博客",
  lang: "zh-CN",
  // ... IDE 自动补全，类型检查
};
```

这带来了三个好处：
1. **IDE 补全**：写配置时自动提示字段名和类型
2. **编译检查**：拼错字段名或类型不对时立即报错
3. **重构安全**：修改类型定义后，所有不符合的地方都会标红

## Astro 诊断

Astro 自带的检查工具 `astro check` 能检测：
- 模板中的类型错误
- 组件 props 类型不匹配
- 导入路径有效性

```bash
pnpm check
# Result (194 files):
# - 0 errors
# - 0 warnings
# - 0 hints
```

本站要求所有提交前 `pnpm check` 必须 0 错误。

## 常见 Lint 规则

Biome 的 recommended 规则集涵盖了很多常见问题：

### 未使用变量

```typescript
// ❌ Biome 报错
const unused = "this variable is never used";

// ✅ 修复
// 删除或使用下划线前缀
const _unused = "intentionally unused";
```

### 使用 const 代替 let

```typescript
// ❌ Biome 建议
let count = 0; // 从未重新赋值

// ✅ 修复
const count = 0;
```

### 禁止 console.log

```typescript
// ❌ 生产代码中的 console
console.log("debug info");

// ✅ 使用专门的 logger 或删除
```

### Import 排序

Biome 自动排序 import：

```typescript
// 排序前
import { onMount } from "svelte";
import type { Config } from "@/types";
import { siteConfig } from "@/config";
import { z } from "zod";

// 排序后（Biome 自动处理）
import type { Config } from "@/types";
import { z } from "zod";
import { onMount } from "svelte";
import { siteConfig } from "@/config";
```

## 与 IDE 集成

Biome 提供了 VSCode 扩展，实现：
- 保存时自动格式化
- 实时 Lint 提示
- 快速修复（Quick Fix）

```json
// .vscode/settings.json
{
  "editor.defaultFormatter": "biomejs.biome",
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "quickfix.biome": "explicit"
  }
}
```

## 性能对比

在 200 个文件的项目上：

| 工具 | 格式化 | Lint | 总计 |
|------|--------|------|------|
| ESLint + Prettier | 8.2s | 12.5s | 20.7s |
| Biome | 0.4s | 0.6s | 1.0s |

Biome 快了约 20 倍，这意味着：
- 保存文件时格式化几乎无延迟
- CI 流水线大幅加速
- 开发体验更流畅

## 团队协作保障

Biome 的确定性格式化消除了代码风格争议：
- 同样的输入永远产生同样的输出
- 不需要在 PR review 中讨论缩进和引号
- git diff 更干净，只有真正的逻辑变更

```bash
# 提交前一键修复
pnpm lint
# Checked 194 files in 0.6s. Fixed 3 files.
```

## 结语

Biome + TypeScript 是本站工程化的基石。Biome 用 Rust 实现的极速 Lint + Format 保证了代码质量，TypeScript 严格模式在编译时消灭了大量潜在 bug。

这套工具链的特点是"零摩擦"：配置简单、速度快到无感、规则合理不啰嗦。开发者可以专注于业务逻辑，而不是与工具搏斗。
