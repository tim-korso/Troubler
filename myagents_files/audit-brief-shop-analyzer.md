## 审计任务 #4：shop-analyzer 全链路审计

### 目标
对 mino 工作区新建的购物成本分析 App 执行完整四阶段审计。

### 项目位置
- 代码：`/Users/1234/.myagents/projects/mino/workspace/shop-analyzer/`
- Spec：`/Users/1234/.myagents/projects/mino/.task/0607-shop-analyzer/spec.md`
- Plan：`/Users/1234/.myagents/projects/mino/.task/0607-shop-analyzer/plan.md`
- 设计系统：`/Users/1234/.myagents/projects/mino/workspace/shop-analyzer/design_guide.md`
- 验收契约：`/Users/1234/.myagents/projects/mino/workspace/shop-analyzer/verify.md`

### 项目背景
- React 19 + Vite 6，单页 App，Dark Luxe 美学
- 功能：粘贴购物链接 → AI 分析价格 → 日均/月均分摊 → 使用追踪
- 14 个源文件，零外部 UI 库，CSS 8KB，JS 212KB
- localStorage 持久化，Playwright MCP 抓取网页
- 无后端，纯前端

### 审计重点

**Phase 0 — 架构审讯**：数据流图 + 边界清单
**Phase 1 — 边界审讯**：localStorage 边界、URL 解析边界、Playwright MCP 边界
**Phase 2 — 产物审讯**：构建产物完整性（dist/ CSS/JS hash）
**Phase 3 — 社区校准**：
- 触控 ≥ 48px？
- 颜色对比度 ≥ 4.5:1（深色背景下的 #8888a0 文字）？
- 无 user-scalable=no？
- 无 emoji？
- pushState 路由正常？
- 字号 ≥ 14px？

### 历史案例对照
- A-1（数据加载）: localStorage 加载失败时的降级处理
- A-2（数据阻塞）: 无大文件同步加载（纯 Vite build）
- A-3（JS 语法）: 构建已通过
- A-5（pushState）: 验证返回按钮
- A-6（缩放）: 验证 viewport
- A-10（设计 token）: 验证无硬编码颜色
- A-11（无 emoji）: 全组件扫描
- A-12（触控）: ≥ 48px

### 输出要求
1. 四阶段审计结论（通过/不通过/有条件通过）
2. FAIL/WARN 清单
3. 报告写 `/Users/1234/Troubler/audit-reports/shop-analyzer-2026-06-07.md`
4. 完成后回复 mino（session: bce106c3-d030-4a...）
