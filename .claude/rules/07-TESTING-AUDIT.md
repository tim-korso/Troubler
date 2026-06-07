# Troubler 测试可靠性审计模块

> 当审计任务涉及「测试策略」「测试覆盖率」「测试有效性」「CI 质量门禁」时加载本模块。
> 融合：Kent Dodds Testing Trophy + 7 大 Agentic 测试用例 + TDD 技能方法论 +
> 金字塔/蜂巢模型 + RL 自适应测试 + RAG 测试生成 + 突变测试。

---

## 触发条件

- 审计项目的测试策略和测试套件质量
- 评估测试覆盖率是否有效（vs 虚假覆盖率）
- 检查 CI/CD 质量门禁
- AI 生成的测试代码可靠性审查
- Flaky test 检测与根因分析

---

## 一、测试分层模型 — 选型审计

### 三种模型对照

| 模型 | 起源 | 适合项目 | 核心比例 |
|------|------|---------|---------|
| **金字塔** | Mike Cohn (2009) | 传统后端/全栈 | 70% Unit / 20% Integration / 10% E2E |
| **奖杯** | Kent Dodds (2018) | 现代前端 SPA | Static 基底 + 大量 Integration + 少量 E2E |
| **蜂巢** | Spotify | 微服务 | 核心 Integration/Contract + 少量 Unit + 极少 E2E |
| **E2E-Heavy (2025)** | Kent Dodds 重新评估 | SSR 应用 | E2E 最大比例（Playwright 成本已接近 Integration） |

### 审计检查：项目选对模型了吗？

```
□ 前端 SPA（React/Vue 纯客户端）→ 奖杯模型
□ SSR 应用（Next.js / Remix）→ 2025 E2E-Heavy 模型
□ 微服务后端 → 蜂巢模型
□ 传统单体 → 金字塔模型
□ 移动端（React Native / Capacitor）→ 奖杯 + E2E
□ 管线/脚本（如视频管道）→ 自定义：输入不变→输出不变的幂等测试
```

---

## 二、测试策略审计（4 层 + 1）

### Layer 0: Static（零运行时成本）

```
□ TypeScript 严格模式开启？
□ ESLint 配置存在且无 disabled rules？
□ Prettier / formatter 配置存在？
□ CI 中 lint 步骤存在且阻塞合并？
```

### Layer 1: Unit（纯逻辑）

```
□ 工具函数有测试？（parser, formatter, calculator）
□ 边界值有覆盖？（空输入/极大值/负值/null/undefined）
□ 测试的是行为还是实现细节？
  反模式: expect(component.state.loading).toBe(true)
  正确: expect(screen.getByText('加载中...')).toBeInTheDocument()
□ 无「测试 getter/setter」的充数测试？
```

### Layer 2: Integration（组件协作）

```
□ 关键用户流程有集成测试？
  - 登录→首页→详情→返回
  - 输入→搜索→结果→点击
□ 网络请求是 mock 还是真实？
  - Mock（MSW / nock）→ 集成测试
  - 真实请求 → E2E 测试
□ 组件渲染测试用的是 @testing-library 而非 enzyme/shallow？
```

### Layer 3: E2E（端到端）

```
□ 核心业务流程有 E2E 覆盖？
□ E2E 工具是 Playwright / Cypress / Detox？
□ E2E 在 CI 中运行？有截图/录像失败时的诊断产物？
□ E2E 运行时间合理？（< 15 min 为佳，> 30 min 需优化）
```

### Layer X: 突变测试（测试的测试）

```
□ 是否跑过突变测试（Stryker / Mutmut / PIT）？
□ 突变测试通过率 > 80%？
  → 高通过率 = 测试能抓 bug（好）
  → 低通过率 = 测试是摆设（差）
□ AI 生成的测试 → 必须跑突变测试验证质量
```

---

## 三、AI 生成测试的专项审计

### 审计清单（对齐 2025 研究 + TDD 技能方法论）

```
□ AI 生成的测试代码经过人工审查？（不是 merge 即跑）
□ 测试覆盖的是「行为」还是「实现巧合」？
  → AI 倾向复制代码逻辑作为「预期值」— 这是循环论证
□ 同一个功能生成 3 次测试，结果一致吗？
  → 不一致 → AI 随机性风险
□ AI 生成的测试中，有语义相同的重复测试吗？
  → 用 embedding 相似度检测（同义测试 = 浪费 CI 时间）
□ 测试有 flaky 检测吗？（至少跑 5 次确认稳定）
  → 2025 研究：AI 生成测试 flaky rate 约 8.3%
```

### AI 测试验证管道

```
生成 → 执行 5 次（检测 flaky）→ 突变测试（检测有效性）
  → 覆盖率增量检测（新增了独特覆盖？）
  → 语义去重（与已有测试重复？）
  → 人工审查（关键场景）
```

---

## 四、测试可靠性评分卡（8 维度）

| 维度 | 权重 | 0 分 | 0.5 分 | 1 分 |
|------|------|------|--------|------|
| 分层选择正确性 | 15% | 无分层 | 分层存在但比例错误 | 分层与项目类型匹配 |
| Static 层覆盖 | 10% | 无 lint/TS | 有 lint 但非严格 | TS strict + ESLint + CI |
| Unit 测试有效性 | 15% | 纯 mock 测试/无断言 | 有测试但覆盖率 <50% | 关键逻辑全部覆盖 + 边界值 |
| Integration 测试 | 20% | 无 | 少量（<5 条关键流程） | 所有关键流程 + MSW/nock |
| E2E 测试 | 15% | 无 | 有但不在 CI 中 | CI 中运行 + 失败截图/录像 |
| 突变测试通过率 | 10% | 未跑过 | <60% | ≥80% |
| Flaky 率 | 10% | >20% | 10-20% | <5% |
| CI 质量门禁 | 5% | 无门禁 | 仅 lint | lint + test + build + coverage |

**评分标准**：
- ≥ 0.8 — 测试体系可靠
- 0.5–0.8 — 部分可靠，需补强
- < 0.5 — 测试体系不可靠

---

## 五、Flaky Test 审计

### 检测方法

```
□ CI 中同一测试最近 10 次运行，失败/通过交替？
  → flaky 嫌疑
□ 测试依赖外部服务（API/数据库）且无 mock/retry？
  → flaky 来源
□ 测试依赖时间（Date.now() / setTimeout）且无 fake timers？
  → flaky 来源
□ 测试依赖 DOM 状态（无 waitFor / findBy）？
  → flaky 来源
□ 测试依赖执行顺序（共享 mutable state）？
  → flaky 来源
```

### Flaky 严重度分级

```
Critical: 阻断 CI merge，每次需要手动 re-run → 必须修
High:     偶尔 flaky（>10% 运行），但 CI 不阻断 → 本周修
Medium:   罕见 flaky（<5%），有重试机制掩盖 → 记录 tech debt
Low:      本地 flaky，CI 不 flaky → 记录，暂不修
```

---

## 六、测试覆盖率的正确读法

### 覆盖率陷阱

```
□ 100% 行覆盖率 ≠ 测试好
  → 可能只是「执行了代码」但没「验证了行为」
  → 检查：是否有断言？断言是否验证了有意义的输出？

□ 高覆盖率 + 低突变分数 = 测试是摆设
  → 注释掉任意一行代码，测试应该失败
  → 如果不失败 → 这行代码没有被真正测试

□ 覆盖率是下限，不是上限
  → 50% 覆盖率一定不够
  → 但 95% 覆盖率不一定够（要看覆盖了什么）
```

### 覆盖率健康指标

```
□ 关键业务逻辑覆盖率 ≥ 90%？
□ UI 渲染（JSX/HTML）覆盖率 ≥ 70%？
□ 工具函数覆盖率 ≥ 95%？
□ 覆盖率在最近 3 个月是否持续下降？
  → 下降 = 新代码没有测试
□ 覆盖率报告是否区分新增代码 vs 存量代码？
```

---

## 七、测试策略与 Troubler 四阶段的集成

```
Phase 0（架构审讯）:
  测试分层模型选对了吗？→ 对照项目类型检查

Phase 1（边界审讯）:
  每个边界有测试吗？
  - 空输入 → 有测试？
  - 网络失败 → 有测试？
  - localStorage 配额满 → 有测试？
  - 并发操作 → 有测试？

Phase 2（产物审讯）:
  测试产物检查：
  - npm test / pytest 通过？
  - 覆盖率报告存在？
  - 突变测试报告存在？
  - CI 最近 5 次全部绿？

Phase 3（社区校准）:
  - 测试分层模型与社区推荐对比
  - Flaky 率 < 行业平均 10%？
  - 自研测试框架 vs 行业标准工具？
```

---

## 八、快速审计脚本

```python
#!/usr/bin/env python3
"""测试体系快速审计 — 输入项目信息，输出健康度评分"""
import subprocess, sys, json, os

def audit(project_path: str) -> dict:
    fails, warns, praises = [], [], []
    os.chdir(project_path)

    # 1. Static layer
    tsconfig = os.path.exists("tsconfig.json")
    eslint = os.path.exists("eslint.config.js") or os.path.exists(".eslintrc.js")
    if tsconfig and eslint: praises.append("👍 Static 层完整 (TS + ESLint)")
    elif not tsconfig: warns.append("💡 缺少 TypeScript")
    elif not eslint: warns.append("💡 缺少 ESLint")

    # 2. Test runner
    pkg = json.load(open("package.json")) if os.path.exists("package.json") else {}
    scripts = pkg.get("scripts", {})
    test_cmd = scripts.get("test", "")
    if not test_cmd: fails.append("🚫 无 test script")

    # 3. Run tests
    if test_cmd:
        r = subprocess.run(["npm", "test"], capture_output=True, text=True)
        if r.returncode != 0:
            fails.append(f"🚫 测试不通过 (exit {r.returncode})")
        else: praises.append("👍 测试全部通过")

    # 4. Coverage
    if "coverage" in str(scripts) or "--coverage" in test_cmd:
        praises.append("👍 有覆盖率配置")
    else: warns.append("💡 无测试覆盖率")

    # 5. E2E
    has_playwright = "playwright" in str(pkg.get("devDependencies", {}))
    has_cypress = "cypress" in str(pkg.get("devDependencies", {}))
    if has_playwright or has_cypress: praises.append("👍 有 E2E 测试")
    else: warns.append("💡 无 E2E 测试")

    # Conclusion
    n_fails = len(fails)
    if n_fails == 0: conclusion = "✅ 测试体系基本可靠" if len(warns) <= 2 else "⚠️ 有条件可靠"
    elif n_fails <= 1: conclusion = "⚠️ 有条件可靠"
    else: conclusion = "❌ 测试体系不可靠"

    return {"conclusion": conclusion, "fails": fails, "warns": warns, "praises": praises}


if __name__ == "__main__":
    path = sys.argv[1] if len(sys.argv) > 1 else "."
    result = audit(path)
    for key in ["praises", "fails", "warns"]:
        for item in result.get(key, []): print(f"  {item}")
    print(f"\n结论: {result['conclusion']}")
```

---

## 九、审计报告模板

```markdown
# {项目} 测试体系审计报告

> Troubler 审计 #{n} | {日期}

## 审计结论：{可靠 / 有条件可靠 / 不可靠}

### 测试分层评估
| 层级 | 状态 | 覆盖率 | 说明 |
|------|------|--------|------|
| Static | ✅/❌ | — | |
| Unit | | X% | |
| Integration | | X% | |
| E2E | | X% | |

### 测试有效性
- 突变测试通过率: X%
- Flaky 率: X%
- 覆盖率健康度: 上升/持平/下降

### FAIL/WARN 清单
```

---

> **核心原则**：不是跑了的测试就是好测试。覆盖率骗人，突变测试不骗人。AI 生成的测试默认不可信——跑 5 次测 flaky + 跑突变测有效性，两项都通过才算数。
