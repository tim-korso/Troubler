# Troubler 投资策略审计模块

> 当审计任务涉及「投资策略」「交易系统」「择时模型」「量化因子」时加载本模块。
> 融合：AQR 196 策略量化框架 + Goldman 抄底清单 + AlphaAgent 多 Agent +
> FinDebate 辩论协议 + FINSABER 长期回测 + ConsensusAI 多人格投票。

---

## 触发条件

- 审计任何投资/交易策略的可靠性
- 验证 AI 生成的交易信号
- 评估「抄底」「择时」「因子选股」等方法论
- 对回测结果做压力测试和过拟合检测

---

## 一、策略分类学

先归类，再审计：

| 类型 | 定义 | 典型失效模式 |
|------|------|------------|
| **趋势跟踪** | 顺势而为，追涨杀跌 | 震荡市假突破反复止损 |
| **均值回归/抄底** | 逆势买入，赌反弹 | 熊市中接飞刀（AQR: -18% 危机表现） |
| **因子选股** | 按因子（价值/动量/质量）排序选股 | 因子拥挤 → 集体踩踏 |
| **事件驱动** | 财报/并购/政策事件交易 | 事件已被定价、反应过慢 |
| **AI 黑箱** | LLM 直接输出买卖信号 | FINSABER: 牛市太保守，熊市太激进 |
| **混合策略** | 多策略叠加 | 过拟合、信号冲突 |

---

## 二、审计四阶段（对齐 Troubler 标准）

### Phase 0 — 策略架构审讯

画出策略的完整决策链：

```
信号触发 → 确认过滤 → 仓位计算 → 执行 → 持仓管理 → 退出
```

对每个环节追问：
```
□ 信号来源是什么？（价格/基本面/情绪/AI 生成）
□ 信号之间有矛盾时怎么处理？
□ 仓位怎么算？（固定比例/凯利公式/ATR 自适应）
□ 退出条件明确吗？（止盈/止损/时间退出）
□ 有冷却期吗？（防 overtrading）
```

### Phase 1 — 边界审讯

```
对策略的每个假设追问：
□ 如果趋势持续 3 年不回头，策略会怎样？（牛市压力测试）
□ 如果市场跌 40%，策略会怎样？（熊市压力测试）
□ 如果波动率突然翻倍，策略会怎样？（恐慌压力测试）
□ 如果流动性枯竭（无法成交），策略会怎样？
□ 策略的最大回撤是多少？你确定吗？（回测最大 ≠ 未来最大）
□ 策略依赖的参数有多少？每个参数变化 ±20%，结果怎么变？
```

**AQR 铁律**：
- 抄底策略 + 趋势策略 → 天然对冲（相关系数 -0.14）
- 单一策略的 Sharpe > 1.0 且样本外 > 0.5 → 大概率过拟合
- 参数 < 3 个 → 更可能样本外有效

### Phase 2 — 回测产物审讯

不看策略描述，看回测报告：

```
□ 样本内/样本外各自多久？比例至少 50/50
□ 交易笔数 > 30？< 30 无统计意义
□ 胜率 × 盈亏比 > 1.2？（＜ 1.0 = 长期必亏）
□ 最大回撤在样本外是否 ≤ 样本内 × 1.5？
□ Sharpe > 2.0？大概率过拟合（AQR：多数真策略 ≤ 1.0）
□ 扣除交易成本了吗？（滑点+佣金+冲击成本 ≥ 0.3%/笔）
□ 基准是什么？策略跑赢基准是选股能力还是暴露了某种 beta？
```

**过拟合检测清单**：
```
□ 参数 ≥ 5 个 → 红灯
□ 交易规则 ≥ 10 条 → 红灯
□ "在 X 条件下做 A，在 Y 条件下做 B" 的分支 > 3 → 黄灯
□ 没有样本外测试 → 红灯
□ 样本外表现显著弱于样本内（Sharpe 降 > 40%）→ 过拟合确认
```

### Phase 3 — 社区校准

```
□ 策略的理论基础是什么？有学术文献支持吗？
□ AQR/Goldman/JPMorgan 对此类策略的最新研究怎么说？
□ 策略与其他已知策略的相关性？
  如果 > 0.7 → 没有提供增量价值
□ 策略的「优势」是否可持续？
  - 数据优势（独有数据）→ 可持续
  - 速度优势（低延迟）→ 军备竞赛
  - 模型优势（更好的 AI）→ 衰减中
```

---

## 三、多 Agent 辩论协议（FinDebate 模式）

对重大策略决策启动 5 Agent 辩论：

```
Round 1: 独立分析
  Bull Agent    → 「为什么这个策略会赚钱？」（找所有支持证据）
  Bear Agent    → 「为什么这个策略会亏钱？」（找所有反对证据）
  Quant Agent   → 回测数据的统计有效性（过拟合/样本偏差/幸存者偏差）
  Macro Agent   → 当前宏观环境是否适合这个策略？
  Risk Agent    → 极端情境下的最大损失估算

Round 2: 交叉质询
  Bull ↔ Bear 对撞 → 找出假设分歧点
  Quant 检查双方的证据质量
  Macro 提供外部约束
  Risk 评估每个分歧的下行风险

Round 3: 共识形成
  5/5 一致 → 策略可靠（罕见）
  4/5 一致 → 策略可行，标注少数意见
  3/5 一致 → 有条件可行，需降低仓位
  < 3/5   → 策略不可靠，不应执行
```

---

## 四、抄底策略专项审计

### Goldman Sachs 抄底决策树

```
抄底信号触发 →
  Q1: 经济在衰退吗？
    是 → ❌ 禁止抄底
    否 → Q2

  Q2: S&P 500 跌幅 ≥ 5%？
    是 → Q3（Goldman 84% 胜率阈值）
    否 → 等待

  Q3: VIX < 35？
    是 → Q4
    否 → 仓位减半

  Q4: 多因子确认
    □ RSI(14) ≤ 35？
    □ 成交量 ≥ 1.5× 20日均值？
    □ Fear & Greed ≤ 30？
    □ 200-SMA 仍然向上（长期趋势完好）？

  4/4 → 正常仓位抄底
  3/4 → 半仓抄底
  < 3/4 → 等待更多确认
```

### 仓位管理（ATR 自适应）

```
单笔风险 = 账户净值 × 2%（最多）
止损距离 = ATR(14) × 2.5
仓位大小 = 单笔风险 / 止损距离

分批建仓:
  T0: 1/3 仓位（信号触发）
  T+1: 1/3 仓位（如果价格回落 > 0.5× ATR）
  T+2: 1/3 仓位（如果再次确认信号仍有效）

退出条件（满足任一）:
  - 价格触及止损
  - RSI(14) ≥ 70（超买）
  - 价格偏离 MA20 > 2× ATR（过度延伸）
  - 持仓时间 > 20 个交易日（时间止损）
```

---

## 五、回测质量评分卡

| 维度 | 权重 | 0 分 | 0.5 分 | 1 分 |
|------|------|------|--------|------|
| 样本外比例 | 20% | 无样本外 | < 30% | ≥ 50% |
| 交易笔数 | 15% | < 30 | 30-100 | > 100 |
| 参数数量 | 15% | > 10 | 5-10 | < 5 |
| Sharpe（样本外） | 15% | < 0 | 0-1.0 | 1.0-2.0（> 2.0 = 0.5 分，可能过拟合） |
| 扣除成本 | 10% | 未扣除 | 部分扣除 | 全额扣除（滑点+佣金+冲击） |
| 压力测试 | 10% | 无 | 仅历史最大回撤 | 2008/2020/2022 情景测试 |
| 基准对比 | 10% | 无基准 | 简单基准 | 风险因子归因（Fama-French） |
| 策略逻辑 | 5% | 无理论依据 | 有依据但模糊 | 学术文献支持 |

**评分标准**：
- ≥ 0.8 — 高质量回测，可信任
- 0.5–0.8 — 有参考价值，需更多验证
- < 0.5 — 不可信任，大概率过拟合

---

## 六、AI 生成策略的额外审查

针对 LLM 直接生成的交易策略（对齐 FINSABER 发现）：

```
□ 策略是否在牛市样本外测试过？
   → LLM 倾向在牛市中过度保守（踏空），验证这一条

□ 策略是否在熊市样本外测试过？
   → LLM 倾向在熊市中过度激进（接飞刀），验证这一条

□ 策略的决策逻辑能否用 3 句话解释清楚？
   → 不能 → 黑箱风险

□ 同样的 prompt 跑 3 次，策略一致吗？
   → 不一致 → LLM 随机性风险

□ 策略是否避免了 "look-ahead bias"？
   → LLM 可能无意中使用训练数据中的未来信息
```

---

## 七、审计报告模板

```markdown
# {策略名称} 投资策略审计报告

> Troubler 审计 #{n} | {日期}

## 审计结论：{可靠 / 有条件可靠 / 不可靠}

### Phase 0 — 策略架构
决策链路 + 信号来源

### Phase 1 — 边界压力测试
熊市/牛市/恐慌/流动性 四种情景

### Phase 2 — 回测审计
| 维度 | 得分 | 说明 |
回测质量总分: X/10

### Phase 3 — 社区校准
对标 AQR/Goldman/JPMorgan 研究结论

### 多 Agent 辩论结果
| Agent | 判断 | 核心论据 |
| Bull | |
| Bear | |
| Quant | |
| Macro | |
| Risk | |
共识: X/5

### FAIL/WARN 清单
🚫 过拟合 / 无样本外 / 回测质量 < 0.5
💡 参数过多 / 未扣成本 / 缺乏压力测试
```

---

## 八、快速审计脚本

```python
#!/usr/bin/env python3
"""投资策略快速审计 — 输入回测数据，输出健康度评分"""
import sys, json

def audit(backtest: dict) -> dict:
    """
    backtest = {
      "sharpe_in_sample": float, "sharpe_out_sample": float,
      "n_trades": int, "n_params": int, "max_dd_in": float, "max_dd_out": float,
      "win_rate": float, "avg_win_loss_ratio": float,
      "costs_deducted": bool, "benchmark_compared": bool,
      "stress_tested": bool
    }
    """
    fails, warns = [], []
    score = 0

    # 过拟合检测
    oos_ratio = backtest.get("sharpe_out_sample", 0) / max(backtest.get("sharpe_in_sample", 0.01), 0.01)
    if oos_ratio < 0.6:
        fails.append(f"🚫 过拟合: 样本外 Sharpe 仅为样本内的 {oos_ratio:.0%}")
    elif oos_ratio < 0.8:
        warns.append(f"💡 样本外衰减: {oos_ratio:.0%}")

    # 交易笔数
    if backtest.get("n_trades", 0) < 30:
        fails.append("🚫 交易 < 30 笔，无统计意义")
    elif backtest.get("n_trades", 0) < 100:
        warns.append(f"💡 仅 {backtest['n_trades']} 笔交易")

    # 参数
    if backtest.get("n_params", 0) > 10:
        fails.append(f"🚫 {backtest['n_params']} 个参数 → 严重过拟合风险")
    elif backtest.get("n_params", 0) > 5:
        warns.append(f"💡 {backtest['n_params']} 个参数偏多")

    # 盈亏比
    ev = backtest.get("win_rate", 0) * backtest.get("avg_win_loss_ratio", 1)
    if ev < 1.0:
        fails.append(f"🚫 期望值 {ev:.2f} < 1.0，长期必亏")
    elif ev < 1.2:
        warns.append(f"💡 期望值 {ev:.2f} 偏低")

    # 成本
    if not backtest.get("costs_deducted"):
        warns.append("💡 未扣除交易成本")

    # 基准
    if not backtest.get("benchmark_compared"):
        warns.append("💡 未对比基准")

    # 压力测试
    if not backtest.get("stress_tested"):
        warns.append("💡 未做压力测试")

    # 结论
    n_fails = len(fails)
    if n_fails == 0:
        conclusion = "✅ 可靠" if len(warns) <= 2 else "⚠️ 有条件可靠"
    elif n_fails <= 1:
        conclusion = "⚠️ 有条件可靠"
    else:
        conclusion = "❌ 不可靠"

    return {"conclusion": conclusion, "fails": fails, "warns": warns, "fail_count": n_fails, "warn_count": len(warns)}


if __name__ == "__main__":
    # 示例
    sample = {"sharpe_in_sample": 2.5, "sharpe_out_sample": 0.8, "n_trades": 45, 
              "n_params": 7, "win_rate": 0.55, "avg_win_loss_ratio": 1.8,
              "costs_deducted": False, "benchmark_compared": True, "stress_tested": False}
    result = audit(sample)
    print(json.dumps(result, indent=2, ensure_ascii=False))
```

---

> **核心原则**：回测好 ≠ 策略好。AQR 用 196 种策略证明——>60% 的抄底策略跑输买入持有。审计的使命不是确认策略能赚钱，是确认策略的赚钱逻辑经得起样本外、压力测试和学术研究的交叉检验。
