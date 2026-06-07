# Troubler 事件留痕协议

> 当审计发现涉及「变更追溯」「决策溯源」「过程留痕」「审计合规」时加载本模块。
> 融合：OWASP MCP Top 10 审计要求 + Anthropic 长程 Harness + ADR 架构决策记录 +
> claude-mem/claude-hooks-sdk/vibe-logger + Lár/TraceMind 事件溯源。

---

## 触发条件

- 审计任务要求追溯「谁在什么时候做了什么、为什么」
- 被审项目缺乏变更记录、决策文档、session log
- 用户要求建立/检查留痕机制
- 多 Agent 协作需要跨 session 记忆传递

---

## 一、问题定义：Vibe Coding 的留痕真空

2025 年多次重大事故的根因相同——**聊天记录就是开发流程**：

| 事件 | 日期 | 后果 |
|------|------|------|
| Replit Agent 删生产数据库 | 2025.07 | 1200+ 公司数据丢失 |
| AWS Kiro 删除重建环境 | 2025.12 | 13 小时 AWS 中断 |
| Claude Code 执行 `rm -rf ~/` | 2025 | 清空用户主目录 |

**共同根因**：
- 无 ticket、无 PR、无 trace——不知道谁改了什么、为什么
- AI Agent 没有「后果感知」——不知道操作不可逆
- 没有留痕就没有 accountability

---

## 二、留痕五层模型

```
Layer 5: 决策留痕 — ADR（为什么做这个选择？）
Layer 4: 会话留痕 — Session Log（这次会话做了什么？）
Layer 3: 操作留痕 — Tool Audit（每个工具调用？参数？结果？）
Layer 2: 变更留痕 — Git + diff（代码到底改了什么？）
Layer 1: 产物留痕 — Build hash + 产物快照（构建出来的是什么？）
```

Troubler 的四阶段审计天然覆盖 Layer 1-2（产物审讯）。本模块补齐 Layer 3-5。

---

## 三、Layer 5 — 决策留痕（ADR）

### ADR 模板（Nygard + MADR 融合，Agent 可读）

```markdown
# ADR-{NNNN}: {决策标题}

- **日期**: YYYY-MM-DD
- **状态**: Proposed | Accepted | Deprecated | Superseded by ADR-{XXXX}
- **决策者**: {human / agent / agent+human}
- **领域**: {data-layer | security | dev-tooling | architecture | design-system}
- **关联**: relates-to ADR-{XXXX}, issue #{N}

## 背景
{为什么需要做这个决策？什么约束和驱动力？}

## 决策
{具体、可验证的决策陈述。现在时/将来时。}

## 考虑过的方案
### 方案 A: {描述}
- **优点**: ...
- **缺点**: ...
- **为何拒绝**: ...

### 方案 B（选定）: {描述}
- **优点**: ...
- **缺点**: ...

## 后果
### 正面
- {获得的能力/优化的指标}

### 负面
- {付出的代价/引入的风险}

### 中性
- {需要记住的事实}

## 验证方式
{如何确认决策被正确实施？}

## 参考
- ADR-{XXXX}, {链接/论文/文档}
```

### ADR 强制时机

以下场景 **必须先写 ADR 再执行**：
- 选择技术栈/框架/库
- 架构模式变更（单页→微前端、CSR→SSR）
- 引入新的全局依赖
- 放弃社区标准方案
- 涉及数据迁移的变更

### 检查清单

```
□ ADR 目录存在？(.docs/adr/ 或 docs/decisions/)
□ 至少有 5 个 ADR？
□ 状态字段标准化？（Proposed/Accepted/Deprecated/Superseded）
□ 每个 ADR 至少含 2 个被拒绝的方案？
□ ADR 按序号递增？（ADR-0001, ADR-0002...）
□ 被废弃的 ADR 引用替代者？
```

---

## 四、Layer 4 — 会话留痕（Session Log）

### claude-progress.txt 格式（对齐 Anthropic Harness）

```markdown
# Session Log — {项目名}

## 2026-06-07 11:00 — Troubler Session
**操作**: pqa-app 全链路审计 #1
**发现**: 4 FAIL (F1-F4), 6 WARN
**产物**: /audit-reports/pqa-app-2026-06-07.md
**Hash 变更**: CSS C7qV7tpC→CnfFauKP (build 验证通过)
**状态**: ✅ 审计关闭

## 2026-06-07 11:30 — Troubler Session
**操作**: quiz-app 全链路审计 #2
**发现**: 6 FAIL (F1-F6), 8 WARN
**产物**: /audit-reports/quiz-app-2026-06-07.md
**状态**: ❌ 不通过 — 待修复
```

### 强制字段

每行 session 记录至少包含：
```
**操作**: {一句话描述}
**产物**: {产出文件路径}
**变更**: {改了哪些文件/hash 是否轮转}
**状态**: {✅通过 / ❌不通过 / ⚠️有条件通过 / 🔄进行中}
```

### Session Handoff 协议

当 Agent A 的 session 结束、Agent B 接手时：

```
Agent A 写入 → claude-progress.txt（上次 session 摘要 + 当前状态）
Agent B 读取 → claude-progress.txt + git log + 关键文件 diff
Agent B 确认 → 理解上次的产出和当前待办
Agent B 继续 → 从上次的断点开始
```

---

## 五、Layer 3 — 操作留痕（Tool Audit Log）

### 结构化事件格式（对齐 OWASP MCP08）

每个工具调用一条 JSON 事件：

```json
{
  "event_id": "evt_20260607_001",
  "timestamp": "2026-06-07T11:00:00.000Z",
  "trace_id": "trace_abc123",
  "session_id": "sess_1a17b0bf",
  "agent_id": "troubler",
  "agent_role": "auditor",
  "tool_name": "Bash",
  "tool_params": {
    "command_hash": "sha256:abc...",
    "command_summary": "grep user-scalable dist/index.html"
  },
  "tool_result": {
    "exit_code": 0,
    "key_finding": "user-scalable=no found in dist/index.html"
  },
  "decision_context": "Phase 3 社区校准 — 检查缩放锁定",
  "risk_level": "low",
  "reversibility": "read_only",
  "user_identity": "mino (audit task #2)",
  "correlation_id": "audit-2-phase3-check1"
}
```

### 必记录字段（OWASP MCP08 对齐）

| 字段 | 说明 |
|------|------|
| `timestamp` | ISO 8601 UTC |
| `agent_id` | 哪个 Agent 执行 |
| `session_id` | 哪个 Session |
| `tool_name` | 调用的工具名 |
| `tool_params` | 参数摘要（不记录敏感信息） |
| `tool_result.exit_code` | 成功/失败 |
| `decision_context` | 为什么调用这个工具 |
| `risk_level` | low / medium / high / critical |
| `reversibility` | read_only / reversible / irreversible |

### 不可逆操作分级

| 级别 | 示例 | 要求 |
|------|------|------|
| **Read Only** | grep, ls, cat, git log | 无需确认 |
| **Reversible** | git commit, npm install, file write | 记录即可 |
| **Irreversible** | git push --force, rm -rf, DB delete, deploy to prod | 必须人工确认 + 双签 |

---

## 六、社区工具速查

### 6.1 vibe-logger（fladdict）

```
语言: Python / TypeScript
安装: pip install vibelogger / npm install vibelogger
用途: AI Native 结构化日志

核心字段:
  correlation_id — 跨操作追踪
  operation — 操作名称
  context — 上下文信息
  human_note — 人类备注（给 AI 的指令）
  ai_todo — AI 待办事项

集成（CLAUDE.md）:
  ### Project Logging
  * Use vibelogger library for all logging
  * Check ./logs/<project>/ for debugging data
```

### 6.2 claude-mem（thedotmack）

```
安装: npx claude-mem install
用途: 跨 session 持久记忆

5 个生命周期钩子:
  SessionStart → 注入上次摘要
  PreToolUse → 记录操作意图
  PostToolUse → 记录操作结果
  SessionStop → 生成观察 + 压缩
  Manual → /remember 手动记录

3 层渐进披露:
  L1: 搜索索引（关键词 → observation ID）
  L2: 时间线（observation ID → 摘要）
  L3: 详情（摘要 → 完整上下文）

隐私: <private> 标签排除敏感内容
```

### 6.3 agent-recorder

```
安装: npm install -g agent-recorder
用途: MCP 代理 — 透明记录所有 MCP 工具调用

特点:
  - 不记录 prompt 和 chain-of-thought（隐私优先）
  - SQLite 存储 + JSON/JSONL 导出
  - TUI 会话浏览器
  - 实时 tailing

命令:
  agent-recorder sessions list
  agent-recorder sessions view <id>
  agent-recorder sessions tail <id>
```

### 6.4 claude-hooks-sdk

```
安装: npm install claude-hooks-sdk
用途: 拦截所有 10 个 Claude Code 钩子事件

10 个事件:
  PreToolUse, PostToolUse, Notification, SessionStart,
  SessionEnd, Stop, SubagentStop, PreCompact,
  PreCommand, PostCommand

内置:
  - 会话上下文注入（session ID / name 自动带入）
  - 文件变更追踪
  - Todo 进度提取
  - 事件关联（transaction ID / prompt ID / git metadata）
  - JSONL 录制和回放
```

---

## 七、Troubler 审计场景的留痕检查清单

### 被审项目留痕健康度评分

| 检查项 | 权重 | 评分 |
|--------|------|------|
| Git 仓库存在且 commit 信息有意义 | 15% | 0-1 |
| commit 粒度合理（非「fix stuff」） | 10% | 0-1 |
| ADR 目录存在且 ≥ 5 个决策记录 | 20% | 0-1 |
| 有 session/progress log（如 claude-progress.txt） | 15% | 0-1 |
| 构建产物可追溯（hash 可对应到源码 commit） | 20% | 0-1 |
| 不可逆操作有确认机制 | 10% | 0-1 |
| 关键决策可追溯到决策者和时间 | 10% | 0-1 |

**评分标准**：
- ≥ 0.8 — 留痕健康
- 0.5–0.8 — 需要补强
- < 0.5 — 留痕严重不足（vibe coding 风险）

### 审计发现分类（留痕相关）

```
🚫 FAIL — 完全缺失（如无 git、无 ADR、无 session log）
💡 WARN — 存在但不规范（如 commit 信息模糊、ADR 缺状态字段）
👍 PRAISE — 留痕规范（如 ADR 完整 + session log 详细 + commit 有意义）
```

---

## 八、快速留痕启动脚本

```bash
#!/bin/bash
# init-trail.sh — 为项目初始化留痕基础设施
set -e
PROJECT_DIR="${1:-.}"
cd "$PROJECT_DIR"

echo "=== 初始化事件留痕系统 ==="

# 1. ADR 目录
mkdir -p docs/adr
cat > docs/adr/ADR-0001-record-architecture-decisions.md << 'EOF'
# ADR-0001: 使用 ADR 记录架构决策

- **日期**: $(date +%Y-%m-%d)
- **状态**: Accepted
- **决策者**: Troubler-init

## 背景
项目需要记录架构决策的上下文和理由。

## 决策
使用 Michael Nygard 格式的 ADR，存储在 docs/adr/ 目录下。

## 后果
### 正面
- 每个重大决策有可追溯的记录
- 新成员/AI Agent 可以理解决策上下文

### 负面
- 需要额外维护成本
EOF
echo "  ✓ ADR 目录 + ADR-0001 创建"

# 2. Session log
cat > claude-progress.txt << 'EOF'
# Session Log — $(basename "$PROJECT_DIR")

> 初始化于 $(date '+%Y-%m-%d %H:%M')
> 格式：**操作** / **产物** / **变更** / **状态**

EOF
echo "  ✓ claude-progress.txt 创建"

# 3. 留痕规则（CLAUDE.md 追加）
if [ -f CLAUDE.md ]; then
  cat >> CLAUDE.md << 'EOF'

## 留痕规则
- 每次 session 结束后更新 claude-progress.txt
- 重大架构决策 → 写 ADR 到 docs/adr/
- 不可逆操作前 → 人工确认（记录确认者和时间）
- commit message 格式: `type(scope): 描述 [ADR-NNNN]`
EOF
  echo "  ✓ CLAUDE.md 留痕规则追加"
fi

echo ""
echo "=== 留痕系统初始化完成 ==="
echo "  docs/adr/            — 架构决策记录"
echo "  claude-progress.txt  — 会话进度日志"
echo "  CLAUDE.md            — 留痕规则（AI 自动遵守）"
```

---

## 九、与 Troubler 四阶段的集成

```
Phase 0（架构审讯）:
  检查 ADR 目录 → 评估决策记录完整性 → 追踪技术选型理由

Phase 1（边界审讯）:
  检查 session log → 追踪上次变更 → 评估 handoff 质量

Phase 2（产物审讯）:
  检查 build hash → git log → 产物-源码对应关系 → 变更可追溯性

Phase 3（社区校准）:
  检查 commit 粒度 → ADR 格式标准化 → 对标社区留痕最佳实践
```

---

> **核心原则**：每次 AI 操作必须能回答三个问题——**谁做的？什么时候？为什么？** 留痕不是负担，是 vibe coding 时代的唯一防线。没有留痕的 AI 开发 = 没有刹车的自动驾驶。
