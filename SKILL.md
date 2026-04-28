---
name: iteration-planning
description: 处理清晰方向的迭代改动。当用户说"我有一堆改动要做"、"帮我整理改动清单"、"这些改动帮我梳理一下"、"某个模块/接口改动xxx"时使用。也适用于：方向清晰但细节待细化、改动琐碎数量多、涉及多个模块或业务线、需要清单化执行、需要分析影响范围、需要输出执行文档的场景。不要用于完全模糊需求（那种用 brainstorming）或单一简单改动。
---

# Iteration Planning

从清晰方向出发，细化需求、分析影响、整理清单、逐项执行，最后输出完整文档。

## 适用场景

- **方向清晰但细节待细化**（用户知道改什么模块/接口，但不确定具体改动点）
- 改动琐碎但数量多
- 需要有条理、不遗漏、有顺序的执行
- 需要分析影响范围、调用链、边界条件
- 需要输出持久化文档（可作为执行依据）

**不适用：**
- 完全模糊需求 → 用 brainstorming
- 单一简单改动 → 直接对话处理

---

## 流程

### Step 1: 收集方向 + 检索代码

让用户陈述改动方向，可以口语化：如"被扫接口里改动xxx"、"支付回调加个日志"。

**主动检索理解代码：**

1. **定位文件** — 找到相关文件
2. **Read 理解代码** — 整个文件或相关函数，理解：
   - 入口函数、调用链
   - 参数结构、返回值处理
   - 依赖的下游服务/模块
   - 边界条件、异常处理
3. **告知用户** — 输出代码结构概览，确认范围正确

**输出示例：**

```
找到被扫接口: src/pay/scan_controller.ts

代码结构：
- handleScan() — 入口函数
  - 参数校验 → ScanRequest
  - 调用 scanService.process() → 下游支付服务
  - 返回 ScanResponse → 包含订单号、状态

调用链：
scan_controller → scan_service → payment_gateway

依赖关系：
- 依赖 payment_gateway 的 scan API
- 依赖 order_service 创建订单

确认这是你要改的范围？
```

### Step 2: 分析影响范围

基于代码理解，分析改动影响：

| 分析维度 | 内容 |
|----------|------|
| **调用链** | 改动函数的上游调用方、下游依赖 |
| **数据流** | 入参变化是否影响下游、返回值变化是否影响上游 |
| **边界条件** | 异常处理、并发场景、边界值 |
| **模块依赖** | 改动后是否需要其他模块配合改动 |

**输出示例：**

```
影响范围分析：

上游调用方：
- api/routes.ts — 路由层调用 handleScan()
- 测试用例 scan_controller_test.ts

下游依赖：
- scan_service.ts — 如果改参数，需同步修改
- payment_gateway — 外部服务，接口不变则无影响

边界条件：
- 并发场景：同一订单重复扫描
- 异常处理：payment_gateway 超时

潜在风险：
- 参数结构调整 → 需同步 3 个调用方
- 返回值新增字段 → 上游需适配
```

### Step 3: 拆具体改动项

把用户方向 + 影响分析 → 拆成具体改动项：

**拆分原则：**
- 每项改动只做一件事
- 明确位置（文件:函数:行号）
- 明确类型（新增/修改/删除）
- 标注依赖关系

**输出示例：**

```
改动项拆分：

#1 [新增] src/pay/scan_controller.ts:handleScan() 第20行
    新增日志记录参数

#2 [修改] src/pay/scan_service.ts:process() 
    参数结构新增 logId 字段

#3 [修改] api/routes.ts:scanRoute()
    调用时传递 logId

#4 [新增] tests/scan_controller_test.ts
    新增日志相关测试用例
```

### Step 4: 整理清单 + 排序

把改动项整理成结构化清单：

**多模块/多业务线自动分类：**

按模块或业务线自动分组（根据项目实际命名），每组标注依赖关系：

```markdown
## 改动清单

### {模块名1}（依赖：无）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 1 | ... | 新增 | {文件路径}:{函数名}:{行号} | 待执行 |

### {模块名2}（依赖：{模块名1}完成）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 2 | ... | 修改 | {文件路径}:{函数名} | 待执行 |

### {模块名3}（依赖：无）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 3 | ... | 新增 | {文件路径}:{函数名} | 待执行 |
```

**模块命名规则：**
- 根据项目实际代码结构命名（如 `auth`、`payment`、`order`）
- 或根据业务线命名（如 `线上业务`、`线下业务`）
- 或根据服务命名（如 `用户服务`、`商品服务`）

**并行执行提示：**

如果多个模块无依赖关系，标注可并行：
```
{模块名1} + {模块名3} → 无依赖，可并行执行
{模块名2} → 依赖 {模块名1}，需等待
```

**排序规则：**
- 按模块/业务线分组
- 无依赖的模块 → 排前面，可并行执行
- 有依赖的模块 → 排后面，等待前置模块
- 同模块内：底层 → 上层

### Step 5: 确认清单

向用户展示清单 + 影响分析，询问：

```
清单已整理完毕，共 N 项改动。

影响范围：
- 涉及 X 个文件
- 影响上游 Y 个调用方
- 需同步下游 Z 个模块

请核对：
1. 改动项完整吗？
2. 执行顺序正确吗？
3. 影响分析符合预期吗？

确认后我开始逐项执行。
```

等待用户确认、补充、调整。

### Step 6: 逐项执行

**逐项确认模式（默认）：**

每次执行前：
```
即将执行 #1: 新增日志记录参数
位置: scan_controller.ts:handleScan():20
类型: 新增

确认执行？
```

用户确认后执行，完成后汇报：
```
#1 已完成 ✓
下一项: #2 参数结构新增logId
继续？
```

如遇阻塞 → 标记原因，询问处理方式。

### Step 7: 输出文档

**文档目的：给 AI 执行用，防止代码实现偏差**

执行结束后，生成结构化文档到项目目录（如 `docs/iteration-{date}/`）。

**混合格式方案：**

| 文件 | 格式 | 用途 |
|------|------|------|
| `CHANGELOG.json` | JSON | AI 定位改动项，字段精确，解析不出错 |
| `IMPLEMENTATION_GUIDE.json` | JSON | AI 执行核心依据，改动前后代码精确对比 |
| `IMPACT_ANALYSIS.md` | Markdown | 代码结构说明，人类和 AI 都能看 |
| `EXECUTION_LOG.json` | JSON | 执行状态追溯，字段精确 |

---

#### 1. 改动清单（JSON）

**文件：** `CHANGELOG.json`

**用途：** AI 快速定位所有改动项，JSON 格式确保字段不漏

**JSON Schema：**

```json
{
  "changelog": {
    "project": "{项目名}",
    "date": "{YYYY-MM-DD}",
    "trigger_direction": "{触发方向：用户原始描述}",
    "items": [
      {
        "id": 1,
        "description": "{改动项描述}",
        "type": "新增|修改|删除",
        "location": {
          "file": "{文件路径}",
          "function": "{函数名}",
          "line": {行号或null}
        },
        "status": "待执行|已完成|阻塞",
        "depends_on": {依赖的改动项ID或null}
      }
    ],
    "execution_order": [
      {
        "id": 1,
        "reason": "无依赖，优先执行"
      },
      {
        "id": 2,
        "reason": "依赖 #1 的 {具体依赖内容}"
      }
    ]
  }
}
```

**AI 执行要求：**
- 读取 `items` 数组，按 `execution_order` 顺序执行
- 每项执行时，用 `location.file` + `location.function` + `location.line` 精确定位
- 执行后更新 `status` 字段

---

#### 2. 影响范围分析（Markdown）

**文件：** `IMPACT_ANALYSIS.md`

**用途：** AI 理解代码结构和依赖关系，确保改动不遗漏调用方

**保持 Markdown 格式的原因：**
- 代码结构说明需要代码块展示
- 调用链需要树形结构展示
- 人类和 AI 都需要能直接阅读

**内容结构：**

```markdown
# 影响范围分析

## 代码结构

### {文件名}
- {函数名}() — {用途说明}
- 参数：{参数结构}
- 返回：{返回值结构}

---

## 调用链

{入口函数}()
  → {中层函数}()
    → {底层函数}()

---

## 上游调用方

| 文件 | 函数 | 影响说明 |
|------|------|----------|
| {文件路径} | {函数名} | {需同步的改动} |

---

## 下游依赖

| 文件 | 函数 | 影响说明 |
|------|------|----------|
| {文件路径} | {函数名} | {需同步的改动} |

---

## 边界条件

- {场景1}：{处理方式}
- {场景2}：{处理方式}
```

**AI 执行要求：**
- 执行改动前，先阅读"上游调用方"和"下游依赖"
- 确认改动参数时，下游已同步
- 确认改动返回值时，上游已适配
```

---

#### 3. 执行记录（JSON）

**文件：** `EXECUTION_LOG.json`

**用途：** AI 回溯执行状态，JSON 格式确保字段不漏

**JSON Schema：**

```json
{
  "execution_log": {
    "session_id": "{YYYY-MM-DD-HH-MM}",
    "total_items": 4,
    "completed_items": 4,
    "blocked_items": 0,
    "duration_minutes": 15,
    "items": [
      {
        "id": 1,
        "start_time": "{HH:MM}",
        "end_time": "{HH:MM}",
        "result": "已完成|阻塞|跳过",
        "error_message": {错误信息或null},
        "rollback_needed": false
      }
    ],
    "pending_items": [],
    "blocked_reasons": []
  }
}
```

**AI 执行要求：**
- 执行过程中实时更新 `items[].result`
- 阻塞时填写 `error_message`
- 如果需要回滚，设置 `rollback_needed: true`

---

#### 4. 可执行代码指南（JSON）

**文件：** `IMPLEMENTATION_GUIDE.json`

**用途：** AI 执行代码实现的**核心依据**，JSON 格式确保字段齐全、不漏步骤

**这是最关键的文档，必须包含：**
- 精确位置（文件路径、函数名、行号）
- 改动前代码（完整上下文）
- 改动后代码（完整上下文）
- 具体步骤（数组，不漏任何一步）
- 验证方法（数组）

**JSON Schema：**

```json
{
  "implementation_guide": {
    "items": [
      {
        "id": 1,
        "description": "{改动项描述}",
        "location": {
          "file": "{文件路径}",
          "function": "{函数名}",
          "line": {行号}
        },
        "change_type": "新增|修改|删除",
        "before_code": "{改动前的完整代码片段}",
        "after_code": "{改动后的完整代码片段}",
        "diff_context": "{改动位置的上下文：前后各5行代码}",
        "steps": [
          "{步骤1：具体操作}",
          "{步骤2：具体操作}",
          "{步骤3：具体操作}"
        ],
        "verify_methods": [
          "{验证方法1}",
          "{验证方法2}"
        ],
        "risks": [
          "{潜在风险1}",
          "{潜在风险2}"
        ],
        "rollback_steps": [
          "{回滚步骤1}",
          "{回滚步骤2}"
        ]
      }
    ]
  }
}
```

**字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `id` | ✓ | 改动项ID，对应 CHANGELOG.json |
| `location.file` | ✓ | 文件路径（相对路径或绝对路径） |
| `location.function` | ✓ | 函数名 |
| `location.line` | ✓ | 行号（精确插入位置） |
| `before_code` | ✓ | 改动前的完整代码（包含上下文，不少于5行） |
| `after_code` | ✓ | 改动后的完整代码（包含上下文，不少于5行） |
| `diff_context` | ✓ | 改动位置的上下文（前后各5行，用于定位） |
| `steps` | ✓ | 具体操作步骤数组（每一步必须明确） |
| `verify_methods` | ✓ | 验证方法数组 |
| `risks` | ○ | 潜在风险数组（可选） |
| `rollback_steps` | ○ | 回滚步骤数组（可选） |

**AI 执行要求（严格执行）：**

1. **定位代码**
   - Read `location.file`
   - 找到 `location.function`，定位到 `location.line`
   - 对比 `before_code` 与当前文件代码是否一致
   - 如果不一致，停止执行，询问用户

2. **执行改动**
   - 按 `steps` 数组顺序执行，不跳步、不漏步
   - 每执行一步，确认无误后继续

3. **验证结果**
   - 按 `verify_methods` 数组验证
   - 所有验证通过后，更新 EXECUTION_LOG.json

4. **遇到问题**
   - 停止执行，填写 `error_message`
   - 如果需要回滚，按 `rollback_steps` 执行

---

## 输出目录结构

```
docs/iteration-{YYYY-MM-DD}/
├── CHANGELOG.json          # 改动清单（JSON，AI定位）
├── IMPLEMENTATION_GUIDE.json # 可执行代码指南（JSON，AI执行核心依据）
├── IMPACT_ANALYSIS.md      # 影响范围分析（Markdown，人类和AI都能看）
└── EXECUTION_LOG.json      # 执行记录（JSON，AI回溯状态）
```

**多模块/多业务线时：**

```
docs/iteration-{YYYY-MM-DD}/
├── CHANGELOG.json          # 总览清单（包含所有模块的改动项）
├── {模块名1}/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
├── {模块名2}/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
└── {模块名3}/
    ├── IMPLEMENTATION_GUIDE.json
    ├── IMPACT_ANALYSIS.md
    └── EXECUTION_LOG.json
```

---

## JSON vs Markdown 使用原则

| 场景 | 用 JSON | 用 Markdown |
|------|---------|-------------|
| AI 需要精确解析 | ✓ 改动清单、执行依据、执行记录 | ✗ |
| 需要展示代码块 | ✗ 转义麻烦，难以阅读 | ✓ 影响范围分析 |
| 字段必须齐全 | ✓ JSON 缺字段会报错 | ✗ |
| 人类直接阅读 | ✗ 需要渲染 | ✓ |

---

## 注意事项

1. **不要添加未被要求的改动** — 只做用户清单里的内容
2. **理解代码后再拆项** — 不凭猜测拆分
3. **分析影响要全面** — 上游、下游、边界、风险
4. **文档要足够详细** — 能按文档重新实现
5. **阻塞时立即汇报** — 不要自行跳过或绕过