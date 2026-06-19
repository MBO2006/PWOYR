# PowerDoc-Agent

**垂直领域文档自动化平台**，支持私有化部署。

从一份完整的 Word 报告出发，提取出所有可变内容（地名、公司名、技术参数等），生成带 `{{占位符}}` 的模板，然后通过 Web 页面填写新值，一键生成新文档。

---

## 核心流程

```
原始报告.docx                     ← 你从客户/同事拿到的完整报告
       │
       ▼  [Phase 1 & 2 · CLI]
  模板生成引擎
       │  1. 解压 docx (ZIP → XML)
       │  2. 合并碎 run (merge_runs)
       │  3. 替换为 {{占位符}}
       │  4. 重打包为 .docx
       │
       ▼
模板文件 (my_template.docx)       ← 一次性工作，每类报告做一次
       │
       ▼  [Phase 3 · Web UI]
  Web 填写界面 (src/api/)
       │  1. 自动解析模板中的 {{占位符}}
       │  2. 网页展示变量列表（分组/分类）
       │  3. 用户修改值 → 点"生成文档"
       │  4. 替换占位符 → 输出新 .docx
       │
       ▼
生成文档.docx                      ← 可以直接用了
```

---

## 项目状态

### ✅ Phase 1: 文档解析基础

- docx 解压/重打包（ZIP 操作）
- XML 部件识别（document.xml + header*.xml，共 35 个）
- 段落级 run 合并（merge_runs），消除 Word/WPS 随机拆分 run 导致的文本断裂

### ✅ Phase 2: 模板生成引擎

- 变量 schema 定义（70 个变量，涵盖地区/项目/公司/技术参数等）
- 文本替换引擎：长值优先、上下文锚定、空格归一化、占位符防嵌套
- 跨 run 段落级拼接替换
- 关键词匹配 + manual placeholder 注入（覆盖 81 个变量 key）

### ✅ Phase 3: Web 界面

- FastAPI 后端：自动扫描模板占位符、展示变量、接收填值、生成文档
- 网页前端：变量分组导航、输入框/textarea、一键下载
- 支持两轮替换：`{{key}}` 精确替换 + 无占位符字段的原文回退替换
- 模板上传热加载

### 🚧 Phase 4: LLM Agent（未开始）

---

## 快速开始

### 1. 安装依赖

```bash
pip install -r requirements.txt
```

### 2. 生成模板（Phase 2）

```bash
# 从完整报告生成带 {{占位符}} 的模板
python -m src.generate_template "原始报告.docx" "my_template.docx"
```

### 3. 启动 Web 界面（Phase 3）

```bash
# 方式一：启动脚本（推荐）
python run.py

# 方式二：直接启动 uvicorn
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000
```

打开浏览器访问 `http://localhost:8000`，你将看到：

```
┌─────────────────────────────────────┐
│ ⚡ PowerDoc-Agent   [模板已加载]     │
├──────────┬──────────────────────────┤
│ 变量分类  │  地区                    │
│          │  ┌─ A ─────────────────┐ │
│ 地区 ✓   │  │ {{A}}  县级市名称    │ │
│ 项目 ✓   │  │ [北流市____________] │ │
│ 公司 ✓   │  └────────────────────┘ │
│ 电网数据  │  ┌─ D18 ──────────────┐ │
│ ...      │  │ {{D18}} 额定功率    │ │
│          │  │ [300MW____________] │ │
│ 共 101 项 │  └────────────────────┘ │
│          │  [🚀 生成文档]           │
└──────────┴──────────────────────────┘
```

### 4. 下载生成的文档

填写变量后点击"生成文档"→ 下载 `.docx` 文件，直接用 Word 打开。

---

## 项目结构

```
powerdoc-agent/
├── src/
│   ├── core/
│   │   ├── merge_runs.py         # 段落级 run 合并引擎
│   │   └── replace_variables.py  # 变量替换引擎
│   ├── generate_template.py      # 模板生成主入口（CLI）
│   ├── api/
│   │   ├── main.py               # FastAPI Web 后端
│   │   ├── templates/
│   │   │   └── index.html        # Web 界面模板
│   │   ├── static/
│   │   │   └── style.css         # 界面样式
│   │   └── README.md             # API 文档
│   └── __init__.py
├── variables/
│   └── schema.json               # 变量定义（70 个变量）
├── test/
│   ├── test_baseline.py          # 合并前基线测试
│   └── test_merge.py             # 合并后效果测试
├── run.py                        # 一键启动脚本
├── requirements.txt              # Python 依赖
├── readme.md                     # 本文件
├── 汇总.md                       # 零基础版详细讲解
├── my_template.docx              # 生成的模板文件
└── 北流市XXX...报告(完整版).docx  # 原始报告示例
```

---

## 变量定义（schema.json）

`variables/schema.json` 定义了从原始报告中提取出的所有可变内容。

### 变量结构

```jsonc
{
  "var_id": "D19",            // 变量 ID，全局唯一
  "value": "300MW/600MWh",    // 当前值
  "type": "atomic",           // 变量类型
  "category": "项目",          // 语义分类
  "description": "总装机容量"  // 说明
}
```

### 变量类型

| type | 含义 | 自动替换 | Web 界面展示 |
|------|------|----------|-------------|
| `atomic` | 单一值 | ✅ | 输入框 |
| `composite` | 多个子值 | ✅ | 多个输入框 |
| `paragraph` | 整段文字 | ⚠️ 关键词匹配 | 文本框（textarea） |

### 变量 ID 命名规则

| 前缀 | 语义域 | 示例 |
|------|--------|------|
| A | 地区 | A（县级市）、A1（地级市）、A2（相邻地区） |
| B | 用电/电力 | B1（用电量）、B4（电池容量）、B6（盈余电量） |
| C | 项目/附件/引用 | C（项目核心词）、C1-C3（附件图）、C4-C5（项目列表） |
| D | 电网数据/技术参数 | D2（变电容量）、D18（额定功率）、D19（总装机容量） |
| E | 线路 | E（线路名称集合） |
| F | 公司/机构 | F1（公司）、F2（调度中心）、F4（地调） |
| G | 时期 | G1（当前规划期）、G2（上一期）、G3（下一期） |
| H | 站点 | H（变电站名称集合） |
| J | 仿真 | J1（仿真模型） |

---

## 替换引擎设计

1. **长值优先** — 按搜索值长度降序，避免短值先替换破坏长值
2. **上下文锚定** — 短数值（纯数字/小数）带后缀匹配，防止碰撞误伤
3. **空格归一化** — 处理 WPS 排版插入的空格（如"中 心"→"中心"）
4. **占位符防嵌套** — 替换后立即掩码 `{{...}}`，后续规则不会污染已替换区域
5. **两轮策略** — 先段落级拼接（仅跨 run 匹配），再单 run 级替换
6. **关键词注入** — 对未精确匹配的段落变量，用关键词在文档中搜索并注入 `{{var_id}}`

---

## Web 界面功能

| 功能 | 说明 |
|------|------|
| 变量展示 | 101 个字段按 22 个分类分组展示 |
| 三种输入类型 | 输入框（atomic）、多输入框（composite）、文本框（paragraph） |
| 分类导航 | 左侧固定导航，点击跳转 |
| 快捷键 | `Ctrl+Enter` 快速提交 |
| 模板上传 | 可上传新的模板文件 |
| 两轮替换 | `{{key}}` 精确替换 + 原文回退替换 |
| 临时清理 | 启动时自动清理 1 小时前的生成文件 |

---

## 技术栈

| 组件 | 技术 |
|------|------|
| 模板生成 | Python 3.12+, lxml, zipfile |
| Web 框架 | FastAPI + Uvicorn |
| 前端 | Jinja2 + 原生 CSS |
| 数据格式 | JSON (schema.json) |
| 文档格式 | .docx (ZIP + XML) |

---

## 已知边界

- **段落变量精确匹配**：约 28 个 paragraph 变量的 schema 文本与文档实际内容有差异（空格/前缀/换行），回退替换可能不成功，需人工对照修正 schema 或直接在 Word 中修改
- **短值变量**：2 个 sub-value（D19_mwh, H_jiantao）因值包含在更长值内，未自动替换
- **目录（TOC）**：原样保留，未转为 Word 自动目录
- **`_CONTEXT_ANCHORS` 和 `_SPACE_NORMALIZED`**：绑定当前北流项目，通用化需重构 schema

---

## 开发

```bash
# 安装依赖
pip install -r requirements.txt

# CLI 模式生成模板
python -m src.generate_template "报告.docx" "模板.docx"

# Web 模式（热重载）
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000

# 跑测试
python test/test_baseline.py
python test/test_merge.py
```

查看 [`src/api/README.md`](src/api/README.md) 获取 API 开发文档。  
零基础用户请阅读 [`汇总.md`](汇总.md) 获取逐文件详解。
