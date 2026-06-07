# 全项目优化报告 — 2026-06-07

> 娜娜（mino）基于 Ethan 方法论 + MyAgents 官网全站内容，对所有项目执行的系统性优化。

---

## 方法论来源

### Ethan 的核心主张（80 亿 token / $5000 烧出来的教训）

| 主张 | 内容 |
|------|------|
| **放手，别微观管理** | AI 比你强。对齐意图和目标，让 AI 找最佳实践。你做收益/代价判断，不做技术判断。 |
| **建立「系统」** | AI 的 context 有限 → 全局不一致。解法：设计系统文档 + 架构规范文档，AI 开发前必读。 |
| **Cross Code Review** | 同一对话 review 自己 = 屎山。空 context + 多模型 + 多视角并行 review。 |
| **verify.md > 执行** | 「没有 verify.md，AI 干一万小时也是耍流氓。」长程任务真正的瓶颈是验收，不是执行。 |
| **Vibe Coding 的尽头是管理学** | 放手 = 目标管理，设计系统 = 制度建设，三视角审查 = 质量管理。 |
| **3 倍 token 建流程 = 省 10 倍 token 修 bug** | 通过系统设计把「人」从开发链路里抽出来。 |

### 社区可用 Skills（来自 MyAgents_skills）
- **task-alignment** — 模糊想法 → 四份契约文档（alignment/task/verify/progress）
- **task-implement** — UserProxy Agent，读取契约，拆解→派发→独立验收
- **download-anything** — 视频/电子书/论文/网盘 1800+ 站点
- **ultra-research** — 多 AI 并行深度研究

---

## 执行的优化

### 1. mino（本工作区）
- ✅ 安装 `task-implement` skill（已有 task-alignment）
- 现在可用 skills：task-alignment, task-implement, download-anything, ultra-research, agent-browser, docx, pdf, pptx, xlsx, github, myagents-cli, skill-creator

### 2. Loser/pqa-app（爸妈版信息验证 App）
- ✅ 创建 `design_guide.md` — CSS 变量/颜色/字号/间距/触控目标/禁止清单。AI 每次 UI 改动前必读。
- ✅ 创建 `verify.md` — 三层验收（guardrails 自动检查 → 产物完整性 → 真机截图+像素分析）
- ✅ 更新 `CLAUDE.md` — 新增 Ethan 方法论章节 + 约束（AI UI 改动前必读 design_guide、部署前必跑 guardrails + verify、代码后必须 Cross Code Review）
- ✅ 已有资产：TROUBLER_FRAMEWORK.md（18 个案例）、GUARDRAILS.md（防护清单+脚本）、PROJECT_TEMPLATE.md（新项目模板）

### 3. AICode/quiz-app（题库 App）
- ✅ 创建 `design_guide.md` — CSS 变量/颜色/字号/间距/触控目标/禁止清单
- ✅ 创建 `verify.md` — 三层验收
- ✅ 更新 `CLAUDE.md` — 新增 Ethan 方法论 + 约束

### 4. Troubler（审计 Agent）
- ✅ 创建 `CLAUDE.md` — 身份定义、工作流、约束
- ✅ 创建 `.claude/rules/01-AUDIT.md` — 审计四阶段 + 通用弹药库 + 审计节奏
- ⚠️ 需要配置 Troubler session 运行以接收后续审计任务

### 5. CC（指挥官）
- 已有完善的 CLAUDE.md（调度模型 + Opus 时钟 + 合流策略），无需大改。
- 💡 建议：在调度 Agent 时要求子 Agent 先读目标项目的 design_guide.md

---

## 防浪费速查（汤姆自用）

### 每次 UI 改动前
```
1. AI 读 design_guide.md（颜色/字号/间距/禁止清单）
2. 验收清单：颜色用 var(--color-xxx)？字号用 var(--font-xxx)？没硬编码？
```

### 每次部署前
```
1. guardrails-check.sh 全绿
2. 构建产物检查：dist/ 有 CSS？hash 变了？数据大小正常？
3. 独立 Agent 跑 verify.md（不是开发 Agent 自己验）
4. 真机截图 + 像素分析
```

### 每次长程任务前
```
1. /task-alignment → 输出四份契约文档
2. /task-implement → UserProxy Agent 执行
3. verify.md 独立验收（自己写的绝不自己验）
```

### Cross Code Review 模板
```
"请以首次看到这段代码的视角，从以下三个角度审查：
1. 代码质量：逻辑错误、边界条件、异常处理、AI 臆造 API
2. 架构一致性：是否符合项目架构约束？有没有绕过已有模块？
3. 对抗性测试：用破坏性输入测试边界"
三个 Agent 空 context 启动，互不知道对方，并行跑。
```

---

> **一句话总结**：汤姆浪费的钱不是模型的问题，是没有「系统」的问题。Ethan 初期也跟你一样——同一对话 review → 屎山、没设计规范 → CSS 地狱、没 verify → 验收黑洞。解法不是换模型，是建流程。
> 
> 本次优化给每个项目补上了 Ethan 方法论的核心机制：设计系统、验收契约、Cross Review 约束。
