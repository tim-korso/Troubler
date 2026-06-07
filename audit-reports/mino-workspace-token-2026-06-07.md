# mino 工作区 Token 效能审计报告

> Troubler 审计 #7 | 2026-06-07
> 目标：/Users/1234/.myagents/projects/mino

---

## 审计结论：🟡 需要优化（68/100）

---

## Token 消耗分析

### 每次会话基础预加载

```
🟢 CLAUDE.md:          150 行 / 6.4KB  → ~1600 tokens
🟢 Rules:              312 行 (4文件)   → ~800 tokens  
🟡 Memory:             751 行 / 40KB   → ~2000 tokens (5个日文件+6个topic)
🔴 Skills frontmatter: 20 个           → ~830 tokens
🟡 Skills 命中加载:    1-3 个/次       → ~3000-9000 tokens (全文)
─────────────────────────────────────────
总计每次会话: ~5000-12000 tokens (不含对话)
```

---

## Top 3 Token 浪费

### 🥇 20 个 Skills — ~830 token 每次预加载

每次对话，20 个 SKILL.md 的 `name + description` 全部加载进上下文。但这 20 个中，大部分跟当前审计/开发任务无关：

**几乎不会触发（可关）：**
- `docx/pptx/xlsx/pdf` — 4 个文档技能，审计时从不触发
- `agent-browser` — 浏览器自动化，很少用
- `download-anything` — 下载技能
- `add-community-extension` — GitHub 扩展管理
- `full-auto-dev` — 全自动开发管道

**偶尔触发（保留）：**
- `frontend-design` / `web-design-guidelines` / `vercel-*` — UI 开发时
- `owasp-security` — 安全审计时
- `karpathy-guidelines` — 编码规范

**每次审计必用（保留）：**
- `task-alignment` / `task-implement` — 任务执行
- `myagents-cli` — MyAgents 操作
- `github` — 代码管理
- `skill-creator` — 创建技能时

**修复**：关闭 8 个低频 Skills → 省 ~350 token/session。

### 🥈 Memory 751 行 / 40KB — ~2000 token 每次预加载

5 个日文件（6/3-6/7）+ 6 个 topic 文件。每天都会增长。CLAUDE.md 可能引用了 Memory 加载机制——全部加载到上下文。

**修复**：
- 日文件压缩：只保留最近 3 天 + 摘要（-50%）
- Topic 文件去重：合并相似 topic
- 加 Memory 加载上限：≤ 300 行

### 🥉 ikebana/.local-packages/ — ~20MB Capacitor 框架在 git 中

`Capacitor.xcframework` (4MB+4MB)、`.abi.json` (1.1MB) 等 SPM 包不应在扫描范围内。

**修复**：`.gitignore` 加 `ikebana/.local-packages/`

---

## 详细评分

| 维度 | 状态 | 说明 |
|------|------|------|
| CLAUDE.md 行数 | ✅ 150行 | ≤300，OK |
| Skills 数量 | 🔴 20个 | >10，6个低频可关 |
| Rules 行数 | ✅ 312行 | OK |
| Memory 大小 | 🟡 40KB/751行 | 日增长，需压缩 |
| 大文件扫描风险 | 🟡 ikebana 20MB | .gitignore不完整 |
| Agent 描述 | ✅ 精简 | OK |
| 文件地图 | ✅ 有 | CLAUDE.md 含地图 |

---

## 修复优先级

| 优先级 | 操作 | 预期节省 |
|--------|------|---------|
| 🥇 | 关闭 8 个低频 Skills | ~350 token/session |
| 🥈 | Memory 压缩到 ≤300行 | ~1000 token/session |
| 🥉 | gitignore 加 ikebana/.local-packages | 避免 AI 误扫 |
| 4 | data-schema.md 覆盖大文件 | 避免 AI 误读 |

**全部修完：每次会话省 ~1500 token**
