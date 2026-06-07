# CC（指挥官）Token 效能审计报告

> Troubler 审计 #8 | 2026-06-07
> 目标：/Users/1234/CC

---

## 审计结论：🔴 严重浪费（42/100）

---

## 每次会话预加载

```
CLAUDE.md:          203行 / 10KB          → ~2500 tokens
Skills frontmatter: 57个                  → ~1750 tokens  ← 🔴 最大黑洞
Skills 命中加载:    1-3个/次              → ~3000-9000 tokens (全文)
Playwright MCP 日志: 35个 / 620KB          → 不加载，但污染 ls/find
temp PNG:           25个 / 1.7MB          → 同上
Rules/Agents:       0                     → —
─────────────────────────────────────────
总计每次会话: ~6000-14000 tokens (不含对话)
```

---

## Top 3 Token 浪费

### 🥇 57 个 Skills — ~1750 token 每次预加载 🔴

57 个 Skills 的 name+description 全部预加载。其中 **48 个是投资分析专用**，每类场景只用 3-5 个。

**分类**：

| 场景 | Skills 数 | 实际每次用 |
|------|----------|-----------|
| Edge 研究管道 | 10 | 2-3 |
| 选股/筛选 | 12 | 1-2 |
| 技术分析 | 8 | 1-2 |
| 下单/仓位 | 6 | 1 |
| 宏观/市场 | 5 | 0-1 |
| 股息/税务 | 3 | 0-1 |
| 通用(mcp/cli/ppt) | 9 | 2-3 |
| 其他 | 4 | 0 |

**修复**：按场景拆分 Skills。每个 session 只加载当前场景的 5-8 个，其余懒加载。预期省 ~1500 token/session。

### 🥈 CLAUDE.md 203 行 / 10KB — ~2500 token 🔴

203 行无文件地图，AI 需要全量读取 + 再自行探索。

**修复**：加文件地图 + 约束速查（30行），其余内容压缩。203→100行。省 ~1200 token。

### 🥉 Playwright 日志 35 个 / temp PNG 25 个 — 污染文件列表

620KB 日志 + 1.7MB 临时截图。AI `ls`/`find` 时遍历这些文件 → 浪费。

**修复**：`.gitignore` 加 `.playwright-mcp/` + `myagents_files/temp/`。或在 CLAUDE.md 标注「不读这些目录」。

---

## 评分卡

| 维度 | 状态 | 说明 |
|------|------|------|
| CLAUDE.md 行数 | 🔴 203行 | >300 可接受但缺文件地图 |
| Skills 数量 | 🔴 57个 | 🚫 严重超标 (>10) |
| Rules 行数 | N/A | 无 |
| Agents 数量 | ✅ 0 | — |
| 大文件/日志污染 | 🔴 60个/2.3MB | 污染文件列表 |
| 文件地图 | ❌ 无 | — |
| data-schema | N/A | — |

---

## 修复优先级

| # | 操作 | 节省 |
|---|------|------|
| 🥇 | Skills 分场景加载：会话启用5-8个，其余懒加载 | ~1500/session |
| 🥈 | CLAUDE.md 压缩 + 文件地图 (203→100行) | ~1200/session |
| 🥉 | gitignore 屏蔽 .playwright-mcp + temp PNG | 省 ls/find 污染 |

**全修 → 省 ~2700 token/session。当前为全工作区最严重的 Token 浪费。**
