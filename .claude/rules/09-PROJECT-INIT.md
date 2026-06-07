# Troubler 新项目初始化审计

> 任何新项目 / 新工作区启动 AI 开发前，必须先过 Troubler 初始化审计。
> 本模块是「先明确规则约束再开工」的强制执行协议。

---

## 触发条件

- 新项目创建 / 新工作区初始化
- 用户说「新建项目」「初始化」「开新坑」
- 首次在项目中启用 AI Agent 开发
- **默认：所有新项目必须执行本审计**

---

## 一、为什么必须初始化审计

### 无规则开发的成本

```
无规则 → AI 随机探索 → 每次对话 5000 token 浪费
       → 硬编码颜色 → 回头整改 → 3000 token
       → 缺 data-schema → AI 读 3.2MB JSON → 上下文炸
       → 无 verify.md → 修完没验证 → 返工 5000 token

总计：一次返工 = 10+ 次正常对话的 token
```

### 有规则的成本

```
有规则 → 初始化审计一次 → 创建 4 个文件 → 500 token
       → 每次对话 AI 看 CLAUDE.md → 直达目标 → 省 4000 token/次

ROI: 一次 500 token 投入 → 每次对话省 4000 token → 第 1 次对话就回本
```

---

## 二、初始化审计清单（8 项必过）

### Phase 0: 项目骨架

```
□ 1. CLAUDE.md 存在且含以下内容:
     □ 技术栈（框架/语言/构建工具）
     □ 文件地图（src/ 目录树，不超 20 行）
     □ 约束速查（触控/对比度/字号/编码参数）
     □ 已知事实（上次审计结论，新项目可留空）
     □ 禁止清单（硬编码颜色/emoji/user-scalable=no/...）

□ 2. .gitignore 存在且覆盖:
     □ node_modules/
     □ dist/ 或 build/
     □ .DS_Store
     □ 构建产物（android/app/build/ 等）
     □ 大文件（> 10MB 的视频/数据文件）

□ 3. 大文件检测 + data-schema.md:
     □ 扫描 > 100KB 的 JSON/JS/CSV
     □ 每个创建 data-schema.md（含结构+行数+访问方式+不读警告）
```

### Phase 1: 设计约束

```
□ 4. design_guide.md 存在（或 tokens.css 定义完整）
     □ 颜色变量（--color-primary 等，≥ 5 个）
     □ 字号变量（--font-body 等，≥ 3 个）
     □ 间距变量（--space-xs 到 --space-xl）
     □ 触控标准（≥ 44px 或 ≥ 48px）
     □ 禁止清单（硬编码/emoji/缩放锁定）

□ 5. verify.md 存在且含:
     □ 自动检查（构建 + 产物完整性）
     □ 手动检查（功能流程 + 视觉）
     □ 验收结论三选一（通过/有条件通过/不通过）
```

### Phase 2: 质量门禁

```
□ 6. 测试基础设施:
     □ npm test 脚本存在
     □ 至少 1 个测试文件
     □ CI 脚本（npm run ci = lint + test + build）

□ 7. 留痕机制:
     □ claude-progress.txt 存在（可选，大项目建议）
     □ docs/adr/ 目录存在（可选，架构变更项目建议）
```

### Phase 3: Token 效能

```
□ 8. 效能检查:
     □ CLAUDE.md < 300 行
     □ Skills ≤ 10 个
     □ data-schema.md 覆盖所有大文件
     □ 无构建产物/缓存文件在 git 中
```

---

## 三、快速初始化命令

```bash
#!/bin/bash
# troubler-init.sh — 新项目初始化审计
PROJECT_DIR="${1:-.}"
cd "$PROJECT_DIR"
echo "=== Troubler 初始化审计: $(basename "$PWD") ==="

PASS=0; FAIL=0

# 1. CLAUDE.md
[ -f CLAUDE.md ] && echo "✓ CLAUDE.md" && ((PASS++)) || { echo "✗ 缺 CLAUDE.md"; ((FAIL++)); }

# 2. .gitignore
grep -q "node_modules" .gitignore 2>/dev/null && echo "✓ .gitignore" && ((PASS++)) || { echo "✗ .gitignore 不完整"; ((FAIL++)); }

# 3. 大文件 + data-schema
BIG_FILES=$(find . -not -path "./node_modules/*" -not -path "./.git/*" -type f -size +100k \( -name "*.json" -o -name "*.js" -o -name "*.csv" \) 2>/dev/null)
if [ -z "$BIG_FILES" ]; then
  echo "✓ 无大文件" && ((PASS++))
else
  MISSING=0
  for f in $BIG_FILES; do
    dir=$(dirname "$f")
    [ ! -f "$dir/data-schema.md" ] && echo "✗ $(basename "$f")($(du -h "$f" | cut -f1)) 缺 data-schema.md" && ((MISSING++))
  done
  [ $MISSING -eq 0 ] && echo "✓ 大文件都有 data-schema" && ((PASS++)) || ((FAIL++))
fi

# 4. design_guide 或 tokens.css
[ -f design_guide.md ] || [ -f src/tokens.css ] && echo "✓ 设计约束" && ((PASS++)) || { echo "✗ 缺 design_guide.md"; ((FAIL++)); }

# 5. verify.md
[ -f verify.md ] && echo "✓ verify.md" && ((PASS++)) || { echo "✗ 缺 verify.md"; ((FAIL++)); }

# 6. test
grep -q '"test"' package.json 2>/dev/null && echo "✓ test script" && ((PASS++)) || { echo "✗ 缺 test script"; ((FAIL++)); }

# 7. CLAUDE.md 行数
LINES=$(wc -l < CLAUDE.md 2>/dev/null || echo 999)
[ $LINES -le 300 ] && echo "✓ CLAUDE.md $LINES 行(≤300)" && ((PASS++)) || { echo "✗ CLAUDE.md $LINES 行(>300)"; ((FAIL++)); }

echo ""
echo "=== $PASS 通过 / $FAIL 失败 ==="
[ $FAIL -eq 0 ] && echo "✅ 初始化审计通过 — 可以开工" || echo "❌ $FAIL 项不通过 — 先修再开工"
```

---

## 四、最小可行骨架生成

如果项目完全从零开始，一键生成所有初始化文件：

```bash
troubler-init my-new-project
```

自动创建：
```
my-new-project/
├── CLAUDE.md              ← 技术栈 + 文件地图 + 约束速查 + 禁止清单
├── .gitignore             ← node_modules / dist / .DS_Store
├── design_guide.md        ← 颜色/字号/间距/触控/禁止清单模板
├── verify.md              ← 自动检查 + 手动检查 + 验收结论模板
├── claude-progress.txt    ← 会话日志
└── public/
    └── data-schema.md     ← 如有大文件则自动生成
```

---

## 五、与 Troubler 审计流程的集成

```
新项目启动 →
  Step 1: Troubler 初始化审计（本模块）
    → 8 项清单全绿 → 开工
    → 有红色 → 补到全绿再开工
  
  Step 2: 首次 AI 开发 →
    AI 读 CLAUDE.md → 看到文件地图 → 直达目标文件
    AI 读 data-schema.md → 理解数据 → 不读全量 JSON
  
  Step 3: 首次部署前 →
    Troubler 四阶段审计（01-AUDIT）
    → Phase 0-3 全部
  
  Step 4: 每次修复后 →
    Troubler 验证审计
    → 只验上次的 FAIL 项 → 确认修复 → 更新已知事实
```

---

> **核心原则**：初始化 500 token → 每次会话省 4000 → 第一次对话就回本。不允许任何项目在无规则约束的情况下开始 AI 开发。
