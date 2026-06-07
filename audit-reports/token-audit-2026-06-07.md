# 四项目 Token 效能审计报告

> Troubler 审计 #6 | 2026-06-07
> 模块：08-TOKEN-AUDIT

---

## 审计排序（浪费量从高到低）

| 排名 | 项目 | 浪费度 | 核心问题 |
|------|------|--------|---------|
| 🥇 | **quiz-app** | 🔴 严重 | 无 CLAUDE.md + 3.2MB JSON 五副本 + Android 构建缓存 |
| 🥈 | **pqa-app** | 🔴 严重 | 无 CLAUDE.md + 748KB data.js + 4MB video 三副本 |
| 🥉 | **Troubler** | 🟡 需优化 | Rules 2142 行 + 4 Skills 非全必要 |
| 4 | **shop-analyzer** | 🟢 健康 | 仅缺 CLAUDE.md |

---

## 逐项目诊断

### 🥇 quiz-app — 🔴 57/100

| 维度 | 结果 | 说明 |
|------|------|------|
| CLAUDE.md | ❌ **不存在** | 每次对话 AI 从零探索，340MB Android 构建目录也在扫描范围 |
| 文件地图 | ❌ 无 | AI 盲目搜索 10+ 目录 |
| Skills | 0 | — |
| Rules | 0 | — |
| Agents | 0 | — |
| 大文件风险 | ❌ **3.2MB JSON × 5 副本** | public/ + ios/App/App/public/ + android 构建缓存 ×3 |
| Build 产物在 git | ❌ **8MB .dex + 4.5MB .apk** | Android build/ 目录不应提交 |

**Token 浪费 Top 3**：

1. 🚫 **无 CLAUDE.md — 每次对话浪费 2000-5000 token 探索**
   AI 每次都要全项目搜索确定「这是什么项目、文件在哪、用了什么技术栈」

2. 🚫 **3.2MB quiz-data.json — AI 可能尝试读它**
   3.2MB ≈ 800K token。如果 AI 在探索阶段读了它 = 一次吃掉全部上下文

3. 🚫 **Android build 产物在 git 中 — 污染文件列表**
   340MB 的 android/app/build/ 让 `find` 和 `ls` 输出极其庞大

**修复**：
```
1. 创建 CLAUDE.md（~300 token，含文件地图 + 技术栈 + 约束速查）
2. echo "android/app/build/" >> .gitignore && git rm -r --cached android/app/build/
3. 在 CLAUDE.md 标注 "quiz-data.json 3.2MB — 不要读取此文件"
4. 删除多余的 quiz-data.json 副本（ios/android 各自保留时由 cap sync 自动复制）
```

---

### 🥈 pqa-app — 🔴 62/100

| 维度 | 结果 | 说明 |
|------|------|------|
| CLAUDE.md | ❌ **不存在** | 虽然有 design_guide + verify + GUARDRAILS，但 AI 不知道从哪读起 |
| 文件地图 | ❌ 无 | 1524 行 MD 散落在多文件中，AI 随机探索 |
| Skills | 0 | — |
| Rules | 0（不在 .claude/ 中） | 实际规则分散在 TROUBLER_FRAMEWORK / GUARDRAILS |
| Agents | 0 | — |
| 大文件风险 | ⚠️ 748KB data.js + 4MB video ×3 | 视频文件在 public/ + ios/ + 根目录 |
| 截图在 git | ⚠️ 110KB ×3 screenshots | 不应在 git 中 |

**Token 浪费 Top 3**：

1. 🚫 **无 CLAUDE.md — 同上，每次探索 2000-5000 token**

2. 💡 **748KB data.js + 4MB video ×3 副本**
   data.js 是必须的（运行时数据），但 AI 探索时可能误读。video 三副本浪费磁盘+污染文件列表

3. 💡 **1524 行 MD 文档无人导航**
   design_guide(159) + GUARDRAILS(472) + TROUBLER_FRAMEWORK(267) + verify(109) — 丰富的文档，但 AI 不知道优先级

**修复**：
```
1. 创建 CLAUDE.md，指向已有文档的阅读顺序
2. 删除根目录和 ios 下的重复 video（只保留 public/）
3. screenshots/ 加入 .gitignore
```

---

### 🥉 Troubler — 🟡 78/100

| 维度 | 结果 | 说明 |
|------|------|------|
| CLAUDE.md | ✅ **237 行** | ≤300，合格 |
| 文件地图 | ❌ 无 | Rules 2142 行没有地图导航 |
| Skills | ⚠️ **4 个** | agent-browser/download-anything/myagents-cli/pptx — 审计时不一定全用 |
| Rules | ⚠️ **7 个 / 2142 行** | 每次基础加载 ~4000+ token |
| Agents | 0 | — |
| 审计报告 | ⚠️ 6 个 MD 文件 | 可能被意外加载 |

**Token 浪费 Top 3**：

1. 💡 **7 个 Rules 2142 行 — 全量预加载**
   虽然渐进披露应该只加载 frontmatter，但 #14882 bug 可能导致全文加载。压缩后可省 30-50%

2. 💡 **4 个 Skills — 不一定全需要**
   `agent-browser` / `download-anything` / `pptx` 在审计任务中几乎用不到。关掉可省 ~400 token/session

3. 💡 **无文件地图 — AI 需遍历 7 个 Rules 才发现目标模块**
   加一个 Rules 索引（哪个规则管什么）→ 省 AI 遍历成本

**修复**：
```
1. CLAUDE.md 加文件地图（40行）→ 省 AI 自主探索
2. 不用的 Skills 去掉或 disable（保留 myagents-cli，其余按需启用）
3. Rules 加索引表（哪个规则管什么场景）
```

---

### 🏅 shop-analyzer — 🟢 85/100

| 维度 | 结果 | 说明 |
|------|------|------|
| CLAUDE.md | ❌ **不存在** | 唯一的问题 |
| 源文件 | ✅ 1196 行 | 最精简 |
| 大文件 | ✅ 无 | 仅 package-lock.json |
| Skills | 0 | — |
| Rules | 0 | — |
| dist | ✅ 干净 | 仅 index + CSS + JS |

**Token 浪费 Top 1**：

1. 💡 **仅缺 CLAUDE.md** — 加一个 150 token 的轻量 CLAUDE.md（技术栈 + 文件地图 + 约束速查 + 已知事实）即可满分

**修复**：
```markdown
# shop-analyzer
## 项目地图
src/utils/ ← 3工具 | src/components/ ← 6视图 | App.jsx ← 路由 | App.css ← 样式

## 约束
触控≥48px / 对比度≥4.5:1 / 字号≥14px / 无硬编码颜色 / npm run ci

## 已知事实
✅ pushState / ✅ user-scalable / ✅ 39 tests / ✅ ESLint / ⚠️ token 硬编码残留
```

---

## 汇总表

| 项目 | 评分 | CLAUDE.md | 大文件风险 | 文件地图 | Top 浪费 |
|------|------|-----------|-----------|---------|---------|
| quiz-app | 🔴 57 | ❌ 无 | 🚫 3.2MB JSON ×5 + Android构建 | ❌ | 5000 token/次 探索 |
| pqa-app | 🔴 62 | ❌ 无 | ⚠️ 748KB + 12MB video | ❌ | 3000 token/次 探索 |
| Troubler | 🟡 78 | ✅ 237行 | — | ❌ | Rules 2142 行预加载 |
| shop-analyzer | 🟢 85 | ❌ 无 | ✅ — | ❌ | 150 token 就够修 |

---

## 修复优先级

```
1. quiz-app:   创建 CLAUDE.md + gitignore Android/build + 标注 JSON 不读     (省 ~4000 token/会话)
2. pqa-app:    创建 CLAUDE.md + 删除 video 副本 + screenshots gitignore      (省 ~2500 token/会话)
3. Troubler:   Rules 索引 + 精简 Skills + 文件地图                          (省 ~1000 token/会话)
4. shop-analyzer: 创建 150 token 轻量 CLAUDE.md                            (省 ~500 token/会话)

全部修完 → 四项目合计省 ~8000 token/会话
```
