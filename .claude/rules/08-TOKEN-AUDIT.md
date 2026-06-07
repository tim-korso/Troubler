# Troubler Token 效能审计模块

> 当审计任务涉及「Token 消耗」「上下文效率」「AI 成本优化」时加载本模块。
> 融合：Anthropic 3亿缓存实践 + context-budget + token-efficiency +
> context-optimization + 渐进披露 + 模型分级路由。

---

## 触发条件

- 用户说「token 花太快」「成本太高」「省 token」
- 审计项目的 CLAUDE.md / skills / MCP / agents 配置
- 新项目初始化时的上下文效率评估

---

## 一、Token 消耗模型

### 上下文 = 输入 token 的四大组成

```
每次对话回合消耗 = 
  System Layer（固定）
    ├── 工具定义 (MCP schemas ~500 token/工具)
    ├── 系统指令 (固定 ~2000 token)
    └── 模型配置
  + Project Layer（固定，有缓存）
    ├── CLAUDE.md (~500-5000 token)
    ├── Rules (.claude/rules/*.md)
    ├── Skills (frontmatter ~100 token/skill, 加载后 ~3000/skill)
    └── Memory 文件
  + Conversation Layer（递增，有缓存）
    ├── 历史消息
    ├── 文件读取内容
    └── 工具调用结果
  + 当前 Prompt（变量）
    └── 你的提问
```

### 缓存 TTL

| 环境 | TTL | 缓存价格 |
|------|-----|---------|
| Claude Code 订阅 | 1 小时 | 10% |
| Claude API | 5 分钟 | 10% |
| 子 Agent | 5 分钟 | 10% |
| DeepSeek API | 查询官方 | 缓存命中 $0.028/M |

---

## 二、Token 效能评分卡（8 维度）

### 配置层审计

| # | 检查项 | 标准 | 超标判断 |
|---|--------|------|---------|
| 1 | **CLAUDE.md 行数** | ≤ 300 行 | > 300 行 → 每轮多烧 1000+ token |
| 2 | **Skills 数量** | ≤ 10 个 | > 10 → 检查是否全部必要 |
| 3 | **MCP 工具数** | ≤ 20 个 | > 20 → 每个工具 ≈500 token schema |
| 4 | **Agent 描述长度** | ≤ 30 词 | > 30 词 → 浪费 |

### 行为层审计

| # | 检查项 | 标准 | 超标判断 |
|---|--------|------|---------|
| 5 | **探索浪费率** | < 20% | 全项目 ls/find/grep 占总 token > 20% |
| 6 | **输出浪费** | < 10% | "Sure!", "Great!" 等客套话占输出 > 10% |
| 7 | **重读率** | < 15% | 未修改文件被重新读取的比例 |
| 8 | **缓存利用率** | > 60% | 命中率 < 60% → 配置或习惯有问题 |

### 评分标准

```
≥ 6/8 达标 → 🟢 Token 效能健康
3-5/8 达标 → 🟡 需要优化
< 3/8 达标 → 🔴 严重浪费
```

---

## 三、四层省 Token 策略

### Layer 1: 输入层（最省，最直接）

```
策略                         节省量      难度
─────────────────────────────────────────────
CLAUDE.md 文件地图替代全文      50-80%     ★☆☆
  → 200 token 地图省 2000 token 盲目搜索

MCP 工具精简                  30-50%     ★★☆
  → 关掉不用的 MCP 服务器
  → 优先用内置工具替代 MCP 工具

Agent 描述精简到一行           10-20%     ★☆☆
  → "代码审查员" → 一行够

Skills 渐进披露               20-40%     ★★☆
  → frontmatter 精准（触发条件写清楚）
  → 不命中的 skill 只消耗 ~100 token

子 Agent 隔离大任务            30-60%     ★★☆
  → 独立上下文，查完即释放
  → 不污染主对话

data-schema 替代全量数据读取   99%+       ★☆☆
  → 大 JSON/CSV/数据文件 → 写 data-schema.md
  → AI 读 200 字节 schema 而非 3.2MB 数据
  → 强制规则 T-1: 超过 100KB 的数据文件必须有 data-schema.md
```

### Layer 2: 处理层

```
策略                         节省量      难度
─────────────────────────────────────────────
模型分级路由                  50-90%     ★★☆
  → 简单 grep/ls → Haiku 或 DeepSeek V3
  → 中等推理 → Sonnet
  → 复杂架构决策 → Opus / R1

目标+地图+约束 prompt         60-80%     ★☆☆
  → "审计社区校准，文件在 App.css" 
  → vs "帮我全面检查代码"

工具调用预算                  20-40%     ★★☆
  → 设定每任务工具调用上限
  → 到 80% 时警告，到 100% 强制停止
```

### Layer 3: 输出层

```
策略                         节省量      难度
─────────────────────────────────────────────
禁止客套话                    10-20%      ★☆☆
  → "Sure!", "Great question!", "Let me know!"
  → 全部禁止

结构化输出                    40-70%      ★★☆
  → JSON/表格 替代 散文
  → 输出 token 价格 = 输入 × 5

一次完成原则                  20-30%      ★★☆
  → 测试绿了就停
  → 禁止给已通过测试的代码抛光

不再读未修改文件              15-25%      ★★☆
  → 读过的文件如果没改 → 不重读
```

### Layer 4: 会话层

```
策略                         节省量      难度
─────────────────────────────────────────────
任务切换 = 新 Session          30-50%     ★☆☆
  → 不要 /compact
  → session handoff: 写文件 → 新 session → 读文件

不闲置 > 1 小时               缓存不过期   ★☆☆
  → 空闲前写好 handoff 文件
  → 回来时开新 session 读文件

大文档放 Projects              20-40%     ★★☆
  → Projects 独立缓存
  → 不随对话每次重传

中途不改配置                  10-20%     ★☆☆
  → 编辑 CLAUDE.md = 全缓存失效
  → 切换模型 = 全缓存失效
```

---

## 四、缓存杀手检测清单

```
□ 对话中途切过模型？（包括 Opus plan 模式）
□ 对话中途编辑过 CLAUDE.md？
□ 对话中途加/删过 MCP 服务器？
□ 对话中途改过 Skills 配置？
□ 上次消息 > 1 小时前？
□ 用了 /clear 后还继续同一 session？
□ 用了 /compact 后丢失了精确数据？

任何一项为「是」→ 缓存已失效 → 重启 session
```

---

## 五、快速审计命令

```bash
#!/bin/bash
# token-audit.sh — 扫描当前项目 Token 效能
echo "=== Token 效能审计 ==="

# 1. CLAUDE.md 行数
LINES=$(wc -l < CLAUDE.md 2>/dev/null || echo 0)
echo "  CLAUDE.md: ${LINES} 行 $([ $LINES -le 300 ] && echo '✓' || echo '⚠️ >300')"

# 2. Skills 数量
SKILLS=$(ls .claude/skills/ 2>/dev/null | wc -l)
echo "  Skills: ${SKILLS} 个 $([ $SKILLS -le 10 ] && echo '✓' || echo '⚠️ >10')"

# 3. MCP 工具数（需要 mcp list 支持）
MCP_COUNT=$(myagents mcp list 2>/dev/null | grep -c "✓\|enabled" 2>/dev/null || echo "?")
echo "  MCP 活跃: ${MCP_COUNT}"

# 4. Agent 描述
echo "  Agent 描述:"
find .claude/agents -name "*.md" 2>/dev/null | while read f; do
  DESC=$(head -3 "$f" | grep "description" | head -1)
  WORDS=$(echo "$DESC" | wc -w)
  echo "    $(basename $f): ${WORDS} 词 $([ $WORDS -le 30 ] && echo '✓' || echo '⚠️ >30')"
done

# 5. Rules 行数
RULES_LINES=$(cat .claude/rules/*.md 2>/dev/null | wc -l)
echo "  Rules 总行数: ${RULES_LINES}"

echo ""
echo "=== 估算 ==="
SKILL_TOKENS=$((SKILLS * 100))
RULES_TOKENS=$((RULES_LINES * 2))
CLAUDE_TOKENS=$((LINES * 3))
TOTAL=$((SKILL_TOKENS + RULES_TOKENS + CLAUDE_TOKENS))
echo "  每次会话基础消耗: ~${TOTAL} tokens (不含 MCP 工具)"
```

---

## 六、CLAUDE.md 精确化改造指南

### 改造前（典型反模式）

```markdown
# 项目说明
这是一个基于 React + Vite 构建的前端项目。我们使用 TypeScript 进行类型检查，
ESLint 进行代码规范检查，Playwright 进行 E2E 测试。
项目采用 Feature-Sliced Design 架构，组件放在 components 目录下...
（3000 token 的散文，AI 每次都要读）
```

### 改造后（文件地图 + 约束速查）

```markdown
## 项目地图
```
src/
├── utils/     ← 3 个工具(localStorage/parser/ai)
├── components/ ← 6 个视图
├── App.jsx    ← 路由+状态
└── App.css    ← 全部样式(~200行)
```

## 约束速查
- 触控≥48px / 对比度≥4.5:1 / 字号≥14px
- npm run ci = lint + test + build
- 构建验证: ls dist/assets/ → grep hash

## 已知事实
✅ pushState / ✅ user-scalable / ❌ text-muted对比度

## 禁止
- 硬编码颜色 / emoji图标 / user-scalable=no
```

**效果**: 3000 token → 300 token。90% 削减。

---

## 七、集成到 Troubler 审计

```
Phase 0（架构审讯）:
  → 检查 CLAUDE.md 行数 + Skills 数量 + MCP 工具数
  → 检查项目是否有文件地图

Phase 1（边界审讯）:
  → 检查缓存杀手触发频率
  → 检查探索浪费率（ls/find/grep 占总 token 比）

Phase 2（产物审讯）:
  → 运行 /context 检查实际消耗
  → 检查 /compact 使用频率 vs session handoff
  → 检查子 Agent 使用模式

Phase 3（社区校准）:
  → 对标 Anthropic 3 亿缓存实践
  → 对标 context-budget 最佳实践
  → 对标 token-efficiency 40-60% 输出节省
```

---

> **核心原则**：Token 不是省出来的——是设计出来的。90% 的 Token 浪费来自「AI 不知道往哪看」。200 token 的文件地图 = 省 2000 token 的盲目探索。
