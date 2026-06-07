# Troubler 多并发决策协议

> 当审计任务需要「多 Agent 协调」「任务拆解与路由」「并行审查决策」「架构选择」时加载本模块。
> 融合：Google DeepMind Agent Scaling（量化预测） + Anthropic 5 工作流模式 +
> Autono 自适应决策 + 社区 orchestrator/agent-army/A2A 协议。

---

## 触发条件

- 审计任务需要多 Agent 并行执行
- 用户说「全面审查」「深度审计」「多视角」
- 被审项目 > 5000 LOC 或 > 3 个子系统
- 需要决定任务拆解粒度 / 并行度 / Agent 数量的架构选择

---

## 一、架构选择决策树（DeepMind 量化框架）

> 核心原则：**没有普遍最优的 Agent 架构。架构选择由任务属性决定。**
> DeepMind 2025 量化模型可解释 51%+ 性能方差，预测最优架构准确率 ~87%。

### Step 1: 任务分类

先回答三个问题：

```
Q1: 任务可以拆解吗？
    → 是：子任务独立？（独立→续Q2）还是依赖链？（串行→选 Single Agent）
    → 否：选 Single Agent

Q2: 子任务之间的噪音水平？
    → 低噪音（结构化任务，如代码检查、数据对比）→ 选 Centralized MAS
    → 高噪音（信息矛盾，如多渠道调研、开放式分析）→ 选 Decentralized MAS

Q3: 精度要求？
    → 精度优先 > 速度 → Centralized MAS（Orchestrator + Workers）
    → 速度优先 > 精度 → Independent MAS（并行无通信）
    → 两者都要（不限资源）→ Hybrid MAS（去中心辩论 + 中心审阅）
```

### Step 2: 架构-任务匹配表

| 任务类型 | 推荐架构 | 预期收益 | 风险 |
|---------|---------|---------|------|
| **严格串行**（步骤间强依赖） | Single Agent | 0% | MAS 反而 -70% |
| **可分解 + 低噪**（代码检查、lint、测试） | Centralized MAS | +80.8% | Orchestrator 单点故障 |
| **可分解 + 高噪**（多渠道调研、信息验证） | Decentralized MAS | +9.2% | 辩论循环放大错误 |
| **混合办公**（日常开发运维） | Single / 轻量 MAS | 中性 | 协调成本可能抵消收益 |
| **极限品质**（安全审计、合规） | Hybrid MAS | 最高精度 | 资源消耗 ×3-5 |

### Step 3: 能力饱和检查（Capability Saturation Paradox）

```
单 Agent 当前准确率 > 45%？
  → 加 Agent 可能反而降性能！
  → 优化单 Agent（更好的 prompt / 更多工具）比加 Agent 更有效
```

### Step 4: 规模预测公式

```
最优 Agent 数 = 独立子任务数 × (1 - 耦合系数)
  where 耦合系数 = 依赖边数 / 总子任务数

实际并行度 = min(最优 Agent 数, CPU 核数-2, 16)

边际收益递减点 = 最优 Agent 数的 2 倍
  → 超过这个数量，通信成本 > 并行收益
```

---

## 二、Anthropic 5 工作流模式 — 选型指南

### 模式选择决策矩阵

```
                    任务可拆解?
                   /          \
                 否            是
                 |              |
            选 Single       子任务独立?
            Agent 模式       /         \
                          是           否（依赖链）
                          |              |
                    子任务可预定义?    选 Prompt Chaining
                    /          \       （串行链 + 关卡验证）
                  是            否
                  |              |
            选 Parallelization  选 Orchestrator-Workers
            （Sectioning 或 Voting）（运行时动态分配）

需要迭代优化品质?
  → 叠加 Evaluator-Optimizer 循环
```

### 各模式详解

#### 模式 1: Prompt Chaining（串行链）

```
[Step A] → gate → [Step B] → gate → [Step C]

适用：步骤间有严格依赖、前置输出是后续输入
Troubler 用例：审计 Phase 0 → Phase 1 → Phase 2 → Phase 3 串行管道
关卡设计：每步输出验证 → 不通过则回退/修正
```

#### 模式 2: Routing（智能路由）

```
输入 → 分类器 → 路由到专业处理器
                ├── 前端代码 → 前端审查 Agent
                ├── 后端代码 → 后端审查 Agent
                ├── 配置文件 → 配置审查 Agent
                └── 文档 → 文档审查 Agent

Troubler 用例：按被审项目类型路由到不同审计规则
```

#### 模式 3: Parallelization（并行化）

**子模式 A — Sectioning（分区并行）：**
```
大任务拆成独立子任务 → 并行执行 → 汇总
├── Agent 1: 审计 src/components/
├── Agent 2: 审计 src/hooks/
├── Agent 3: 审计 src/styles/
└── Agent 4: 审计 public/
```

**子模式 B — Voting（投票并行）：**
```
同一任务 → 3 个 Agent 独立判断 → 多数投票
├── Agent A（证据导向）：只看数据
├── Agent B（逻辑导向）：只看逻辑链
└── Agent C（来源导向）：只追踪引用

→ 3/3 一致 = 高置信度
→ 2/3 一致 = 中置信度
→ 1/3 = 发起 Delphi 共识轮
```

#### 模式 4: Orchestrator-Workers（编排-工人）

```
Orchestrator（主控 Agent）
  ├── 接收任务 → 动态拆解
  ├── 根据子任务类型 → 路由到专业 Worker
  ├── 收集 Worker 输出 → 交叉验证 → 合并
  └── 输出最终报告

Worker 不通信（避免 N² 消息爆炸）
Worker 不知道其他 Worker 的存在（保证独立审查）
```

#### 模式 5: Evaluator-Optimizer（评估-优化）

```
Generator → Evaluator → 未达标 → Generator（重试）
                ↓
              达标 → 输出

Troubler 用例：
- 审计报告 → 自检 Agent → 发现遗漏 → 补充审查
- 最大迭代次数：3（防无限循环）
- 优化方向：覆盖率、精确度、可操作性
```

---

## 三、多 Agent 并发协调协议

### 3.1 初始任务拆解（Task Decomposition）

```
输入：一个审计任务
输出：TaskGraph（DAG），每个节点包含：
  - task_id: 唯一标识
  - task_type: 架构/边界/产物/社区/安全/无障碍/性能
  - dependencies: [前置 task_id]
  - agent_type: 执行的 Agent 类型
  - priority: critical/high/medium/low
  - expected_output: 结构化 schema
  - timeout: 最大执行时间（秒）
```

**拆解规则：**
```
1. 每个 audit phase 至少一个节点
2. 独立子项目 → 独立节点
3. 共享依赖 → 先执行依赖，再并行无依赖节点
4. 高冲突风险 → 加 Voting 节点（2-3 个独立 Agent）
```

### 3.2 并发执行调度

```
Wave 0: 所有无依赖节点 → 并行启动
Wave 1: 依赖已完成的节点 → 并行启动
...
Wave N: 最后一组节点 → 并行启动

每波内的 Agent 数 ≤ min(CPU核数-2, 独立节点数, 16)
```

### 3.3 协调通信模式

| 模式 | 通信方式 | 适用场景 |
|------|---------|---------|
| **零通信（Independent）** | 无 | 纯独立子任务，Agent 不知彼此存在 |
| **汇聚通信（Centralized）** | Orchestrator ← Worker | 需要中心审阅/去重/一致性检查 |
| **辩论通信（Decentralized）** | 全连接（P2P） | 需要 Agent 间相互挑战 |
| **事件总线（Event-Driven）** | 发布/订阅 | 需要异步通知和增量更新 |

**Troubler 默认：汇聚通信 **。Orchestrator 分配任务 → Workers 返回结果 → Orchestrator 合并去重。

### 3.4 失败处理

```
Agent 超时 → 重试 1 次 → 仍失败 → 降级（由另一个 Agent 覆盖）
Agent 返回空 → 记录为 No Finding（不是通过）
Agent 间矛盾 → 升级到 Human-in-the-Loop 或第三 Agent 仲裁
Orchestrator 失败 → 取所有已完成 Worker 的结果，不做合并
```

---

## 四、自适应决策（Autono 模式）

### 4.1 适时放弃策略（Timely Abandonment）

```
对一个子任务：
  Round 1: 执行 → 无结果？→ 换个角度重试
  Round 2: 执行 → 仍无结果？→ 评估放弃代价
    放弃代价 = 缺失此结果的审计覆盖率损失
    继续代价 = 额外的 token × 时间
    
  放弃代价 < 继续代价 → 放弃，标注 No Finding
  放弃代价 > 继续代价 → 最后重试 1 次
```

### 4.2 不确定性感知路由

```
对每个 Worker 的输出标注不确定性：
  LOW uncertainty  → 直接采用
  MEDIUM uncertainty → 加入 Voting 队列
  HIGH uncertainty → 派发额外独立 Agent 验证
```

### 4.3 记忆传递（Memory Transfer）

```
Worker Agent 的发现 → 写入共享上下文（结构化 JSON）
  → 后续 Worker 可以读取（避免重复工作）
  → Orchestrator 可以引用（交叉验证）

关键原则：只共享事实发现（What），不共享判断（So What）
  → Why: 避免偏见级联传播
```

---

## 五、多模型路由策略

### 5.1 按任务类型选择模型

| 任务类型 | 推荐模型 | 理由 |
|---------|---------|------|
| 架构决策、安全审计、跨源验证 | Claude Opus | 推理深度最高 |
| 代码检查、产物验证、模式匹配 | Claude Sonnet | 性价比最佳 |
| 批量格式化、简单 grep、标准检查 | Claude Haiku | 低成本重复工作 |

### 5.2 三模型共识（Critical Decision Only）

仅对影响审计结论的 FAIL/WARN 判断启用：

```
Agent A（Claude Opus）→ 独立审查 →
Agent B（Claude Sonnet） → 独立审查 → 3 路对比
Agent C（不同视角 prompt）→ 独立审查 →

  3/3 一致 → FAIL/WARN 确认
  2/3 一致 → 标注 「2/3 共识」，记录分歧
  1/3 → 降级一级（FAIL→WARN, WARN→INFO）
```

---

## 六、Troubler 审计场景的实际决策模板

### 场景 A：标准单项目审计（当前默认）

```
选型：Single Agent 串行（Phase 0→1→2→3）
理由：审计步骤间有强依赖（架构 → 边界 → 产物 → 校准）
并发：无
模型：Claude Sonnet
```

### 场景 B：深度全面审计（用户说「全面审查」）

```
选型：Centralized MAS（Orchestrator + 5 Workers）
Worker 1: 架构+数据流（Phase 0+1）
Worker 2: 产物检查（Phase 2）— 独立
Worker 3: 社区校准（Phase 3）— 独立
Worker 4: 安全审计（OWASP/注入/密钥）
Worker 5: 无障碍+设计一致性（a11y + tokens）

Orchestrator: 合并 → 去重 → 一致性检查 → 输出报告
并行度: 5
模型: Opus（Orchestrator + Worker 4,5）+ Sonnet（Worker 1,2,3）
```

### 场景 C：多项目并行审计

```
选型：Independent MAS（每个项目一个 Agent）
Agent A: pqa-app 审计
Agent B: quiz-app 审计
Agent C: video-pipeline 审计

并行度: 3
模型: Sonnet
后处理: Orchestrator 汇总对比，提取跨项目模式
```

### 场景 D：高冲突风险判断

```
选型：Voting MAS（3 Agent）
同一判断 → 3 个不同视角的 Agent →
  证据导向 / 逻辑导向 / 来源导向

用于：审计结论与修复方有争议时
```

---

## 七、决策协议日志格式

每次关键决策记录：

```json
{
  "decision_id": "audit-3-decision-001",
  "context": "审计 quiz-app — 是否启用多 Agent 并行",
  "task_properties": {
    "decomposability": "medium",
    "sequentiality": "low",
    "noise_level": "low",
    "accuracy_requirement": "high"
  },
  "architecture_selected": "Single Agent 串行",
  "reasoning": "步骤间有 Phase 0→1→2→3 依赖链，拆解后协调成本 > 收益",
  "model": "sonnet",
  "parallel_degree": 1,
  "fallback": "若超出 60 分钟则拆为 2 session"
}
```

---

## 八、快速决策脚本

```python
#!/usr/bin/env python3
"""Troubler 架构选择器 — 输入任务属性，输出推荐架构"""
import sys, json

def recommend(task: dict) -> dict:
    decomp = task.get("decomposability", "low")
    noise = task.get("noise_level", "low")
    sequential = task.get("sequentiality", "high")
    accuracy = task.get("accuracy_requirement", "medium")
    budget = task.get("budget_constraint", "normal")
    single_accuracy = task.get("single_agent_accuracy", 0.3)

    # Rule 1: Capability Saturation
    if single_accuracy > 0.45:
        return {"arch": "Single Agent", "reason": "能力饱和—加 Agent 可能降性能", "agents": 1}

    # Rule 2: Sequential dependency
    if sequential == "high":
        return {"arch": "Single Agent / Prompt Chaining", "reason": "强依赖链—MAS 协调成本 -70%", "agents": 1}

    # Rule 3: Decomposable + Low noise
    if decomp == "high" and noise == "low":
        return {"arch": "Centralized MAS", "reason": "可分解低噪音—预期 +80.8%", "agents": "3-5"}

    # Rule 4: Decomposable + High noise
    if decomp == "high" and noise == "high":
        return {"arch": "Decentralized MAS", "reason": "高噪音环境—Agent 辩论对撞", "agents": "3-4"}

    # Rule 5: Maximum quality
    if accuracy == "critical" and budget == "unlimited":
        return {"arch": "Hybrid MAS", "reason": "双保险—辩论+中心审阅", "agents": "5-7"}

    # Default
    return {"arch": "Single Agent", "reason": "任务简单—不需要 MAS 协调成本", "agents": 1}

if __name__ == "__main__":
    task = json.loads(sys.argv[1]) if len(sys.argv) > 1 else {}
    result = recommend(task)
    print(json.dumps(result, indent=2, ensure_ascii=False))
```

**用法**：
```bash
echo '{"decomposability":"high","noise_level":"low","sequentiality":"low"}' | python3 decide.py
# → {"arch": "Centralized MAS", "reason": "可分解低噪音—预期 +80.8%", "agents": "3-5"}
```

---

> **核心原则**：不加 Agent 是默认选择。只有任务属性数据证明 MAS 有净收益时，才拆 Agent。DeepMind 的量化框架告诉我们——多 Agent 的收益不是靠直觉猜的，是靠任务属性算的。
