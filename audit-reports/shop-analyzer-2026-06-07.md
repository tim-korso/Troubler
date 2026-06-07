# shop-analyzer 全链路审计报告

> Troubler 审计 #4 | 2026-06-07
> 项目：/Users/1234/.myagents/projects/mino/workspace/shop-analyzer
> 审计范围：React 19 + Vite 6，Dark Luxe 购物成本分析 App，14 源文件

---

## 审计结论：**有条件通过** ⚠️

- 产物审讯：**通过** ✅ — 构建完整
- 架构边界：**通过** ✅ — pushState / localStorage / 数据流
- 社区校准：**有条件通过** ⚠️ — 2 FAIL，3 WARN

---

## Phase 0 — 架构审讯

### 数据流图

```
用户输入（链接/手动）→ parser.js（URL解析）
  → HomeView（输入组件）
  → App.jsx（状态管理 + 视图路由）
  → storage.js（addItem → localStorage 持久化）
  → AnalyzeView / DetailView / TrackView（展示）

AI 估算: ai-estimate.js → Playwright MCP → 抓取网页 → LLM 分析 → 自动填入天数

路由: pushState + history.back() + popstate 监听
```

### 边界清单

| # | 边界 | 传递方式 |
|---|------|---------|
| B1 | 用户输入 → URL 解析 | parser.js 正则 |
| B2 | localStorage ↔ App state | try/catch + JSON 序列化 |
| B3 | App state → 子组件 | React props drilling |
| B4 | Vite build → dist/ | 标准 Vite React 构建 |
| B5 | Playwright MCP → AI 估算 | MCP 工具调用 |
| B6 | 路由切换 | pushState + popstate |

---

## Phase 1 — 边界审讯

### B1：URL 解析
- ✅ 正则匹配主流电商 URL
- ⚠️ 无法解析的 URL → 静默 fallback 到手动输入（UX 友好但用户可能困惑）

### B2：localStorage
- ✅ loadItems() try/catch → 损坏数据返回 `[]`
- ✅ saveItems() 直接写入（可能因配额满而静默失败）
- ⚠️ saveItems 无 try/catch — localStorage 配额满时抛异常

### B3/B4/B6：React + Vite + Router
- ✅ pushState + history.back() + popstate 全部实现
- ✅ viewStack 与 browser history 双向同步
- ✅ replaceState 用于 tab 切换

### B5：Playwright MCP
- ⚠️ 外部依赖 — MCP 服务不可用时 AI 估算失效
- ✅ 有 fallback（手动输入天数）

---

## Phase 2 — 产物审讯

| 检查项 | 结果 | 证据 |
|--------|------|------|
| 构建通过 | ✅ | npm run build exit 0, 372ms |
| CSS > 0 字节 | ✅ | 8,086 bytes |
| JS 大小合理 | ✅ | 212KB (React 19 标准范围) |
| 关键 CSS 类名在产物中 | ✅ | cost-breakdown, result-card, track-btn 等命中 |
| tokens.css + App.css 均已 import | ✅ | main.jsx → tokens.css, App.jsx → App.css |
| user-scalable=no | ✅ | maximum-scale=3.0, user-scalable=yes |
| pushState 路由 | ✅ | 4 处调用 (pushState/back/popstate/replaceState) |

**产物审讯结论：通过 ✅**

---

## Phase 3 — 社区校准

### FAIL（必须修复，2 项）

#### F1. 🚫 --text-muted (#5a5a6e) 对比度 2.9:1 不达标

- **症状**：辅助文字/标签在深色背景上几乎看不见
- **根因**：`#5a5a6e` 在 `#0a0a0f` 背景上对比度仅 2.9:1
- **标准**：WCAG 2.1 AA ≥ 4.5:1
- **证据**：`python3` 精确计算 luminance 后对比度为 2.89:1
- **影响范围**：
  - `.tab-btn` (Tab 导航)
  - `.url-input::placeholder` (占位文字)
  - `.recent-title` (分类标题)
  - `.cost-card .label` (分摊标签)
  - `.detail-label` (详情标签)
  - `.history-meta` (历史元数据)
  - `.history-daily .unit` (单位文字)
- **修复**：将 `--text-muted` 从 `#5a5a6e` 改为 `#8a8aa0`（对比度 5.1:1）

#### F2. 🚫 触控目标 44px < 48px（2 处）

- **症状**：返回按钮和删除按钮触控区域偏小
- **根因**：`.header-back` (L10) 和 `.history-delete` (L171) 均为 44×44px
- **标准**：design_guide.md 明确「触控 ≥ 48px」
- **证据**：grep 确认两处 `width: 44px; height: 44px`
- **修复**：44→48px

### WARN（建议修复，3 项）

#### W1. 💡 字号 12-13px 低于 --font-small: 14px

- 12px 在 design_guide 中被允许（「标签: 12px」），但 detail-label / history-meta / cost-card .label 等 7 处使用了 12px 作为标签文字
- 13px（recent-daily span / recent-monthly / result-platform / ai-estimate-btn）低于 14px 最小值
- **建议**：统一标签为 12px（符合 design_guide），非标签辅助文字提升到 13-14px

#### W2. 💡 `#fff` 硬编码

- `App.css:152` `.track-btn.stop { color: #fff; }` — 应使用 `var(--text-primary)` 或定义 `--text-on-danger` token
- **仅有 1 处**，影响极小

#### W3. 💡 saveItems 无 try/catch 保护

- `storage.js:9` — localStorage 写入未包裹 try/catch，配额满时未捕获异常
- loadItems 已正确 try/catch，saveItems 应一致

---

## 历史案例对照

| 历史案例 | 状态 |
|----------|------|
| A-1（数据加载降级） | ✅ loadItems try/catch + `[]` fallback |
| A-2（大文件阻塞） | ✅ 纯 Vite build，无同步大文件 |
| A-3（JS 语法） | ✅ 构建通过 |
| A-5（pushState） | ✅ 4 处完整实现 |
| A-6（缩放） | ✅ maximum-scale=3.0 |
| A-10（设计 token） | ✅ 全部使用 CSS 变量（仅 1 处 #fff） |
| A-11（无 emoji） | ✅ 零 emoji |
| A-12（触控） | ❌ 2 处 44px < 48px |

---

## 量化总结

```
Phase 0 架构审讯:  6 边界 ✓
Phase 1 边界审讯:  5/6 通过, 1 WARN (saveItems)
Phase 2 产物审讯:  7/7 通过 ✓
Phase 3 社区校准:  FAIL: 2, WARN: 3

综合: 有条件通过 ⚠️
```

---

> 整体代码质量较高——14 源文件零外部 UI 库、pushState 完整、设计 token 几乎全部遵守。
> F1（对比度 2.9:1）是唯一真正影响用户体验的问题——深色背景下 muted 文字几乎不可读。
