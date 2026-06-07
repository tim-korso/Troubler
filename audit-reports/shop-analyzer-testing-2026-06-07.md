# shop-analyzer 测试体系审计报告

> Troubler 审计 #5 | 2026-06-07
> 项目：/Users/1234/.myagents/projects/mino/workspace/shop-analyzer
> 模块：07-TESTING-AUDIT

---

## 审计结论：**❌ 不可靠**

测试体系基本为零。14 个源文件，零测试文件，零 CI。

---

## 测试分层评估

| 层级 | 状态 | 说明 |
|------|------|------|
| **Static** | ❌ 缺失 | 无 TypeScript、无 ESLint、无 Prettier |
| **Unit** | ❌ 缺失 | 无 test script、零测试文件 |
| **Integration** | ❌ 缺失 | 无测试框架、无 MSW/nock |
| **E2E** | ❌ 缺失 | 无 Playwright/Cypress |
| **Mutation** | ❌ 未跑 | 没有可突变的测试 |
| **CI** | ❌ 缺失 | 无 GitHub Actions / 质量门禁 |

---

## 当前状态

```
package.json scripts:
  dev:     vite --host 0.0.0.0
  build:   vite build
  preview: vite preview
  test:    无 ❌

依赖:
  devDependencies: @vitejs/plugin-react + vite (仅 2 个)
  测试框架: 无
  lint: 无

验证手段:
  verify.md 中 9 条手动检查项 (全人工、全手动)
  无自动化验证
```

---

## 评分卡

| 维度 | 权重 | 得分 | 说明 |
|------|------|------|------|
| 分层选择正确性 | 15% | 0 | 无分层 |
| Static 层 | 10% | 0 | 无 TS/ESLint |
| Unit 测试 | 15% | 0 | 零测试 |
| Integration 测试 | 20% | 0 | 零测试 |
| E2E 测试 | 15% | 0 | 零测试 |
| 突变测试 | 10% | 0 | 未跑 |
| Flaky 率 | 10% | N/A | 无测试 |
| CI 门禁 | 5% | 0 | 无 CI |

**总分: 0/1.0** → ❌ 不可靠

---

## 风险分析

| 风险 | 概率 | 影响 |
|------|------|------|
| **storage.js 回归** — 改代码后 localStorage 读写静默出错 | 中 | 高（数据丢失） |
| **parser.js 回归** — URL 解析改坏后无法分析链接 | 中 | 中 |
| **UI 回归** — 改组件后页面白屏/布局错乱 | 高 | 高 |
| **构建失败** — 改代码后 build 不通过但无人知道 | 低 | 中（有 verify.md 手动检查） |

当前唯一的防线是 verify.md 的 9 条手动检查 + Troubler 审计。无自动化。

---

## 修复建议（按投资回报排序）

### 🥇 零成本立即加（已有 Vite → vitest 零额外依赖）

```bash
npm install -D vitest @testing-library/react @testing-library/jest-dom jsdom
```

**package.json 加**：
```json
"scripts": {
  "test": "vitest run",
  "test:watch": "vitest"
}
```

**vite.config.js 加**：
```js
/// <reference types="vitest" />
export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: './src/test/setup.js',
  }
})
```

### 🥈 第一批测试（最高价值，30 分钟）

```
src/utils/storage.test.js   — localStorage CRUD + 边界(空/损坏/配额满)
src/utils/parser.test.js    — URL解析 + 边界(非URL/空字符串/null)
src/components/HomeView.test.js — 输入链接→分析按钮启用
```

### 🥉 第二批（中等价值，1 小时）

```
src/components/AnalyzeView.test.js — 价格展示/分摊计算
src/components/TrackView.test.js   — 追踪开始/停止/天数计算
src/components/HistoryView.test.js — 列表渲染/空状态/删除
src/App.test.js                     — 路由切换(pushState/返回)
```

### 🏅 第三批（锦上添花）

```
- ESLint + Prettier (Static 层)
- Playwright E2E (完整用户流程)
- GitHub Actions CI (lint + test + build 门禁)
```

---

## 一句话

> shop-analyzer 代码干净（审计 #4 结论），但测试为零。Vite 生态下加测试的成本极低——`npm i vitest` 就是全部额外依赖，30 分钟可覆盖最危险的 3 个文件。
