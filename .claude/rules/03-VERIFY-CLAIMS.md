# Troubler 置信度验证模块

> 当审计任务涉及「主张验证」「信息准确性」「置信度评分」时加载本模块。
> 融合：AutoVerifier(6层管道) + DelphiAgent(多Agent共识) + LoCal(双评估) + HybridRAG(校准) + Anthropic RSP(5层验证)。

---

## 触发条件

以下任一项命中即加载本模块：
- 被审项目包含主张/断言/结论的置信度标注
- 被审项目使用「置信度」「可靠性」「证据等级」等标签
- 用户要求验证某个主张的准确性
- 审计报告中需要对发现做置信度分级

---

## 一、主张分类学（Claim Taxonomy）

任何主张首先归类：

| 类型 | 定义 | 验证方式 | 示例 |
|------|------|---------|------|
| **可检验事实** | 可被实验/数据直接验证 | 查原始数据 | 「每天摄入钙补充剂增加心梗风险20%」 |
| **因果主张** | A 导致 B | 查 RCT/自然实验 | 「种子油不引起炎症」 |
| **统计主张** | 含具体数字/百分比 | 查原始论文 | 「2777人血液检测」 |
| **框架性论述** | 分析框架/前提假设 | 查逻辑一致性 | 「信息市场有三条断线」 |
| **观点/价值判断** | 不可证伪 | 标注为主观 | 「爸妈应该自己判断」 |

---

## 二、证据来源分级（5-Tier Provenance）

对齐 AutoVerifier 五级来源体系：

| 级别 | 来源类型 | 置信度权重 | 颜色 | 判定标准 |
|------|---------|-----------|------|---------|
| **L1** | 实验数据 / RCT 原始结果 | 1.0 | 🟢 深绿 | 可直接追溯到原始论文表格/附图 |
| **L2** | 模拟/建模结果 / Meta分析 | 0.8 | 🟢 浅绿 | 方法学描述清晰，参数可复现 |
| **L3** | 理论推导 / 机制解释 | 0.6 | 🟡 | 逻辑链完整，无跳跃 |
| **L4** | 引用其他工作 / 综述 | 0.4 | 🟠 | 可追溯到被引文献 |
| **L5** | 作者断言 / 个人经验 | 0.2 | 🔵 | 无外部可验证来源 |

**判定规则**：
- 同一主张有多级来源 → 取最高级
- 来源之间有矛盾 → 降一级
- 来源不可追溯 → 最高 L4

---

## 三、验证管道（6-Layer Verification Pipeline）

### Layer 1: 主张提取与分解

```
输入：一段文本
输出：(Subject, Predicate, Object, Confidence) 四元组列表

对每个主张：
  □ 主张可原子化吗？（一条主张 = 一个可验证断言）
  □ 主张含数值吗？→ 标注数字精度
  □ 主张含因果箭头吗？→ 标注因果方向
  □ 主张有时效性吗？→ 标注时间范围
```

**检测脚本**：
```python
# 提取文本中所有带数字的主张
import re
claims = re.findall(r'([^。.]*?\d+[^。.]*?[。.])', text)
# 标注每个主张的类型和来源
```

### Layer 2: 文档内验证（Intra-Document NLI）

对每个主张，在原文中找证据：

```
□ 主张在文档内的来源是什么？（段落/图表编号）
□ 数据是否被正确引用？（数值对上？方向对上？）
□ 结论是否超出数据支持范围？（overclaim 检测）
□ 同一文档内不同位置的说法一致吗？（内部一致性）
```

**检测**：
- 主张中的数字 vs 引文中的数字 → 精确匹配
- 主张中的因果方向 vs 引文的结论 → 方向一致性
- 「所有」「从未」「完全」等绝对化词汇 → overclaim 红灯

### Layer 3: 跨源验证（Cross-Source Triangulation）

```
对每个主张：
  □ 是否有 ≥2 个独立来源支持？
  □ 来源是否独立？（不同团队/不同方法/不同数据）
  □ 来源之间有矛盾吗？
  □ 矛盾的根因是什么？（方法差异？人群差异？时间差异？）
```

**多源权重计算**：
```
独立来源数 ≥3 且一致 → 置信度 +1 级
独立来源数 =2 且一致 → 置信度 不变
独立来源数 =1         → 置信度 -1 级
来源之间有矛盾        → 置信度 -2 级，标注「争议」
```

### Layer 4: 外部信号验证（External Signal Corroboration）

```
□ 该主张是否与已知科学共识一致？
□ 该主张是否被权威机构（WHO/FDA/CDC）背书？
□ 该主张是否被 Google Fact Check API 收录？
□ 该主张的反对意见是否有可靠来源？
```

**Google Fact Check API 查询模板**：
```bash
curl "https://factchecktools.googleapis.com/v1alpha1/claims:search?query=URL_ENCODED_CLAIM&key=API_KEY"
# 返回已有的核查结果
```

### Layer 5: 逻辑与因果一致性（Logical + Causal Check）

对齐 LoCal 框架的双评估：

```
逻辑等价性检查：
  □ 主张的前提成立吗？
  □ 从前提能必然推出结论吗？
  □ 有无偷换概念/范畴错误？

因果一致性检查：
  □ 如果前提不成立，结论还成立吗？（反事实测试）
  □ 因果方向正确吗？（A→B 还是 B→A？）
  □ 有无遗漏混淆变量？
```

**反事实测试模板**：
```
「如果 [前提] 不成立，[结论] 还会成立吗？」
→ 会：前提不是原因 → 因果主张有问题
→ 不会：前提可能是原因 → 通过
```

### Layer 6: 综合评分与报告（Scored Assessment）

```
最终置信度 = 来源等级 × 内部一致性 × 跨源一致性 × 逻辑通过率

输出：
  ✅ Supported       — ≥0.8 综合分 → 可以依赖
  ⚠️ Needs Review    — 0.4-0.8 分 → 需要更强证据
  ❌ Likely Flawed    — <0.4 分 → 不应依赖
  🔮 Unverifiable    — 无法验证（观点类）
```

---

## 四、多 Agent 共识验证协议（Delphi Protocol）

当主张涉及重大决策，启动多 Agent 独立验证：

```
Round 1: 独立判断
  Agent A (证据导向) → 只看数据，忽略论述
  Agent B (逻辑导向) → 只看逻辑链，忽略权威
  Agent C (来源导向) → 只追踪引用链

Round 2: 分歧对撞
  三个 Agent 的分歧点 → 每个分歧追问 3 层：
  「你为什么这么判断？」
  「你的证据是什么？」
  「对方 Agent 的证据你看了吗？」

Round 3: 共识形成
  → 3/3 一致 = 高置信度
  → 2/3 一致 = 中置信度，记录少数意见
  → 1/3 或无共识 = 低置信度，标注「不确定」
```

---

## 五、置信度校准（Calibration）

### 校准检查（对齐 Hybrid RAG 的 Temperature Scaling）

验证完成后对验证者自身做校准：

```
□ 过去 10 次验证的准确率是多少？
□ 高置信度判断的准确率 vs 低置信度的准确率？
□ 是否有「过度自信」倾向？（Overconfidence Index）
   公式：高置信判断中实际正确的比例 / 高置信判断的比例
   理想值：≥ 0.9（90% 高置信判断是正确的）
```

### ECE（Expected Calibration Error）简化版

```
将判断按置信度分桶：[0-20%], [20-40%], [40-60%], [60-80%], [80-100%]
对每个桶：|准确率 - 平均置信度|
ECE = Σ(桶样本比例 × 桶误差)
理想值：< 0.05
```

---

## 六、与 Troubler 四阶段的集成

```
审计任务到达 →
  Phase 0（架构审讯）:
    识别所有主张 → 分类 → 标注来源等级
  Phase 1（边界审讯）:
    对每个主张追问：「这个主张什么情况下是错的？」
  Phase 2（产物审讯）:
    验证主张中的数字 → 追踪引用链 → 检查 NLI
  Phase 3（社区校准）:
    跨源对比 → Google Fact Check → 科学共识检查
  →
  输出置信度报告（✅/⚠️/❌/🔮）
```

---

## 七、结构化输出模板

```markdown
## 置信度验证报告

### 主张清单
| # | 主张（简化） | 类型 | 来源等级 | 内部一致 | 跨源一致 | 逻辑通过 | 综合分 | 结论 |
|---|------------|------|---------|---------|---------|---------|--------|------|
| 1 | ... | 可检验事实 | L1 | ✓ | 2源一致 | ✓ | 0.92 | ✅ |
| 2 | ... | 因果主张 | L3 | ⚠️ | 1源 | ✗ | 0.35 | ❌ |

### 分歧记录
| 主张 | Agent A | Agent B | Agent C | 共识 |
|------|---------|---------|---------|------|
| ... | L2 | L3 | L2 | 2/3 = L2 |

### 校准指标
- Overconfidence Index: 0.92（达标）
- ECE: 0.038（达标）
- 不可验证主张数/总主张数: 12%
```

---

## 八、快速验证脚本

```python
#!/usr/bin/env python3
"""快速主张验证 — 对单条主张跑 6 层管道"""
import sys, json

claim = sys.argv[1] if len(sys.argv) > 1 else input("主张: ")

# Layer 1: 分解
print("=== L1: 主张分解 ===")
has_number = bool(__import__('re').search(r'\d+', claim))
has_causal = any(w in claim for w in ['导致','引起','增加','降低','因为','所以'])
print(f"  含数值: {has_number}  |  含因果: {has_causal}")

# Layer 2: 需要人工提供原文
print("=== L2: 文档内验证 ===")
source = input("  原文段落（回车跳过）: ").strip()
if source:
    # 简单 NLI: 数字匹配
    nums_claim = set(__import__('re').findall(r'\d+\.?\d*', claim))
    nums_source = set(__import__('re').findall(r'\d+\.?\d*', source))
    match = nums_claim & nums_source
    print(f"  数字匹配: {len(match)}/{len(nums_claim)} ({100*len(match)/max(len(nums_claim),1):.0f}%)")

# Layer 3-5: 人工辅助
print("=== L3-L5: 交叉验证 ===")
independent_sources = input("  独立来源数: ").strip()
logic_ok = input("  逻辑链完整? (y/n): ").strip().lower() == 'y'

# Layer 6: 评分
score = 0.5  # base
if has_number: score += 0.15
if independent_sources.isdigit() and int(independent_sources) >= 2: score += 0.2
if logic_ok: score += 0.15

label = "✅ Supported" if score >= 0.8 else ("⚠️ Needs Review" if score >= 0.4 else "❌ Likely Flawed")
print(f"\n综合分: {score:.2f} → {label}")
```

---

> **核心原则**：置信度不是感觉，是证据等级 × 多源一致 × 逻辑完备的综合。未经交叉验证的主张最高只能给 L4。
