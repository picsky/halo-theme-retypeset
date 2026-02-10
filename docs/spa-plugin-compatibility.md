# SPA 页面过渡方案 — Halo 插件兼容性设计

## 背景

本主题实现了一个轻量级 Client Router，将传统 MPA（多页应用）导航转为 SPA（单页应用）式导航：拦截链接点击 → fetch 新页面 HTML → 替换 `<body>` → View Transitions API 动画。

这带来了丝滑的页面过渡体验，但也打破了 Halo 插件的核心假设：**每次导航都是全新的 document**。

### MPA vs SPA 的差异

| 维度 | 传统 MPA | 我们的 SPA |
|------|---------|-----------|
| `window` 对象 | 每次导航全新创建 | 始终同一个 |
| `DOMContentLoaded` | 每次导航触发 | 只在首次加载触发 |
| `window.load` | 每次导航触发 | 只在首次加载触发 |
| `<head>` | 每次全部重新加载 | 保留不变，仅合并新增 |
| `<body>` | 每次全部重新加载 | 替换为新内容 |
| 全局变量 | 不跨页面保留 | 始终保留 |
| 外部脚本 | 每次重新下载执行 | 只加载一次，保留在内存中 |

---

## 四层防护架构

```
SPA 导航发生
    │
    ▼
┌─── Layer 1: Head 合并 ───────────────────────────────┐
│ 新页面 <head> 中的脚本和样式表                         │
│ • 按 src/href 去重                                    │
│ • 已存在 → 跳过（不重复加载）                         │
│ • 不存在 → 添加到 <head>（首次加载）                  │
│ 覆盖: TemplateHeadProcessor 注入的 CSS/JS             │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─── Layer 2: 智能脚本激活 ────────────────────────────┐
│ 新 <body> 中的脚本（插件通过内容处理器注入的）         │
│ • 外部脚本 + 已加载 → 跳过（库在内存中）             │
│ • 外部脚本 + 未加载 → 克隆执行（首次加载）           │
│ • 内联脚本 → 始终克隆执行（页面特定逻辑）            │
│ 覆盖: ReactivePostContentHandler 注入的脚本           │
│ 覆盖: <halo:footer /> 注入的脚本                      │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─── Layer 3: 插件重渲染钩子 ──────────────────────────┐
│ 主动调用已在内存中的插件渲染 API                       │
│ • Vditor: window.vditorRender.render()                │
│ • KaTeX auto-render: window.renderMathInElement()     │
│ • MathJax: window.MathJax.typeset()                   │
│ • Mermaid: window.mermaid.run()                       │
│ 覆盖: 使用全局单例守卫的插件                          │
└───────────────────────────────────────────────────────┘
    │
    ▼
┌─── Layer 4: Web Component 自动升级 ──────────────────┐
│ <halo:comment> 等自定义元素                            │
│ • 插入 DOM 时自动触发 connectedCallback               │
│ • 无需特殊处理                                        │
│ 覆盖: 基于 Web Component 的 Halo 组件                 │
└───────────────────────────────────────────────────────┘
```

---

## Halo 插件的四种注入模式

### 模式 A：`TemplateHeadProcessor` — 向 `<head>` 注入

**机制**：插件在服务端通过 Thymeleaf 模板处理器向 `<head>` 添加 `<script>`、`<link>` 标签。

**典型插件**：plugin-katex（注入 `katex.min.css` + `katex.min.js`）

**我们的处理**：Layer 1 Head 合并。按 `src`/`href` 去重，首次遇到时添加，后续跳过。

**兼容性**：✅ 自动兼容

### 模式 B：`ReactivePostContentHandler` — 向文章内容注入

**机制**：插件在服务端修改文章 HTML 内容，在正文前后插入 `<script>`、`<link>`、`<div>` 等标签。

**典型插件**：
- **Vditor**（注入 `render-utils.js`、`render.js` 等外部脚本 + 标记 div）
- **plugin-katex**（在文章末尾注入内联渲染脚本）

**我们的处理**：

- **内联脚本**（如 plugin-katex 的 `renderMath()` 调用）→ Layer 2 始终克隆执行 ✅
- **外部脚本 + 无全局守卫** → Layer 2 首次克隆执行，后续跳过 + Layer 2 内联脚本仍会触发 ✅
- **外部脚本 + 有全局守卫**（如 Vditor 的 `window.vditorPjax`）→ Layer 2 跳过 + Layer 3 主动调用渲染 API ✅

**兼容性**：✅ 绝大多数自动兼容。使用全局守卫且暴露渲染 API 的插件需在 Layer 3 添加一行调用。

### 模式 C：`ElementTagPostProcessor` — 服务端修改 HTML 标签

**机制**：插件在服务端 Thymeleaf 渲染阶段修改 HTML 标签属性（如替换 `<img src>`）。

**典型插件**：halo-plugin-webp-cloud（将图片 src 替换为 WebP Cloud 代理 URL）

**我们的处理**：无需处理。服务端已完成所有修改，Client Router fetch 到的 HTML 就是最终结果。

**兼容性**：✅ 天然兼容，零冲突

### 模式 D：Web Component 自定义元素

**机制**：插件通过 Halo 的自定义标签（如 `<halo:comment>`）注入 Web Component。

**典型插件**：Halo 评论组件

**我们的处理**：Layer 4。当 Web Component 元素被插入 DOM（body 替换）时，浏览器的 `CustomElementRegistry` 自动调用 `connectedCallback`，触发组件初始化。

**兼容性**：✅ 自动兼容

---

## 已验证插件清单

| 插件 | 注入模式 | 全局守卫 | 兼容方式 | 状态 |
|------|---------|---------|---------|------|
| [halo-plugin-vditor](https://github.com/justice2001/halo-plugin-vditor) | B（外部脚本） | `window.vditorPjax` | Layer 2 跳过 + Layer 3 主动调用 `vditorRender.render()` | ✅ |
| [plugin-katex](https://github.com/halo-sigs/plugin-katex) | A + B（head CSS/JS + body 内联脚本） | 无 | Layer 1 去重 + Layer 2 内联脚本克隆执行 | ✅ |
| [halo-plugin-webp-cloud](https://github.com/webp-sh/halo-plugin-webp-cloud) | C（纯服务端标签处理） | 无 | 天然兼容 | ✅ |
| Halo 评论组件 | D（Web Component） | 无 | Layer 4 自动升级 | ✅ |

---

## 如何为新插件添加支持

### 步骤 1：判断插件类型

1. 查看插件的 Java 源码，确认注入模式（A/B/C/D）
2. 如果是模式 C 或 D → **无需任何处理**
3. 如果是模式 A → 检查是否只注入 CSS/JS 文件 → **大概率自动兼容**
4. 如果是模式 B → 进入步骤 2

### 步骤 2：检查脚本类型

- **纯内联脚本**（如 plugin-katex）→ **自动兼容**，无需额外处理
- **外部脚本 + 无全局守卫** → **自动兼容**
- **外部脚本 + 有全局守卫** → 进入步骤 3

### 步骤 3：添加重渲染钩子

在 `src/main.ts` 的 `reInitPlugins()` 函数中添加调用：

```typescript
function reInitPlugins(): void {
  const win = window as unknown as Record<string, unknown>;

  // ... 已有插件 ...

  // 新插件: 检查全局对象是否存在，调用其渲染 API
  if (win.newPluginGlobal && typeof (win.newPluginGlobal as Record<string, unknown>).render === 'function') {
    (win.newPluginGlobal as { render: () => void }).render();
  }
}
```

### 步骤 4：构建验证

```bash
pnpm build
```

部署后验证：从列表页 SPA 导航到含该插件内容的文章页，检查功能是否正常。

---

## 无法覆盖的边界场景

以下场景我们的方案**无法自动处理**：

1. **使用全局守卫 + 不暴露任何公开渲染 API 的插件**
   - 症状：SPA 导航后插件功能失效
   - 解决：联系插件作者暴露 re-render API，或关闭页面过渡动画

2. **依赖 `DOMContentLoaded` / `window.load` 且无 `readyState` 降级的外部脚本**
   - 症状：首次加载正常，SPA 导航后不触发
   - 解决：在 `reInitPlugins()` 中手动调用其初始化函数

3. **在 `<head>` 中注入的内联脚本进行 DOM 操作**
   - 我们的 Head 合并不处理内联 `<script>`（只处理 `<script src>` 和 `<link>`）
   - 通常 head 内联脚本只做配置/变量声明，不影响功能

### 最终兜底

用户可以在主题设置 → 高级 → 关闭「启用页面过渡动画」，回退到传统 MPA 导航，100% 兼容所有插件。

---

## 相关源码

| 文件 | 职责 |
|------|------|
| `src/components/client-router.ts` | Client Router 核心：链接拦截、页面获取、Head 合并、智能脚本激活、DOM 替换、View Transitions |
| `src/main.ts` → `reInitPlugins()` | Layer 3 插件重渲染钩子 |
| `src/main.ts` → `initPageComponents()` | 页面切换后的组件重初始化（TOC、图片缩放、插件重渲染） |
| `settings.yaml` → `advanced.enable_transition` | 过渡动画开关（兜底方案） |
