# iteration-planning

[English](README.md) | [简体中文](README.zh-CN.md)

一个 Claude Code 技能，从清晰方向出发，细化需求、分析影响、整理清单、逐项执行，最后输出完整文档。

**核心目的：防止 AI 执行代码时出现偏差**

---

## 解决什么问题

| 问题 | 这个技能怎么解决 |
|------|------------------|
| 方向清晰但细节模糊 | AI 主动检索代码，理解结构，帮你拆成具体改动点 |
| 改动琐碎容易遗漏 | 分析影响范围（上游调用方、下游依赖），确保不漏 |
| 不确定执行顺序 | 按依赖关系排序，标注哪些可并行执行 |
| 多模块改动混乱 | 自动按模块分组，每个模块独立文档 |
| AI 实现容易偏差 | 输出 JSON 文档（改动前后代码对比 + 具体步骤），防止偏差 |
| 执行过程不可追溯 | 输出执行记录，可回溯每项状态 |

---

## 适用场景

| 适用 | 不适用 |
|------|--------|
| 方向清晰但细节待细化（如"支付接口改动xxx"） | 完全模糊需求 → 用 brainstorming |
| 改动琐碎但数量多（10+ 个小改动） | 单一简单改动 → 直接对话处理 |
| 涉及多个模块或多个业务线 | |
| 需要分析调用链、边界条件 | |
| 需要输出文档给 AI 执行 | |

---

## 流程

```
Step 1: 收集方向 + 检索代码
        用户说改动方向 → AI 检索代码 → Read 理解结构

Step 2: 分析影响范围
        调用链、上游调用方、下游依赖、边界条件、风险点

Step 3: 拆具体改动项
        方向 + 影响分析 → 具体改动点（文件:函数:行号）

Step 4: 整理清单 + 排序
        按模块分组 + 依赖顺序 + 标注可并行执行

Step 5: 确认清单
        用户核对、补充、调整

Step 6: 逐项执行
        每项确认后执行，执行完汇报

Step 7: 输出文档
        4 类文档（JSON + Markdown），给 AI 执行用
```

---

## 输出文档

执行结束后生成到 `docs/iteration-{日期}/`：

| 文件 | 格式 | 用途 | 为什么用这个格式 |
|------|------|------|------------------|
| `CHANGELOG.json` | JSON | AI 定位改动项 | JSON 缺字段会报错，不会漏 |
| `IMPLEMENTATION_GUIDE.json` | JSON | AI 执行核心依据 | 字段直接存路径/行号，解析不出错 |
| `IMPACT_ANALYSIS.md` | Markdown | 代码结构说明 | 代码块展示，人类和 AI 都能看 |
| `EXECUTION_LOG.json` | JSON | 执行状态追溯 | 字段精确，不漏状态 |

---

### CHANGELOG.json 结构

改动清单，AI 定位所有改动项：

```json
{
  "changelog": {
    "project": "payment-service",
    "date": "2026-04-28",
    "trigger_direction": "被扫接口加日志",
    "items": [
      {
        "id": 1,
        "description": "新增日志记录参数",
        "type": "新增",
        "location": {
          "file": "scan_controller.ts",
          "function": "handleScan",
          "line": 20
        },
        "status": "已完成",
        "depends_on": null
      }
    ],
    "execution_order": [
      { "id": 1, "reason": "无依赖，优先执行" },
      { "id": 2, "reason": "依赖 #1 的 logId" }
    ]
  }
}
```

---

### IMPLEMENTATION_GUIDE.json 结构

**这是核心文档**，AI 执行代码的精确依据：

```json
{
  "implementation_guide": {
    "items": [
      {
        "id": 1,
        "description": "新增日志记录参数",
        "location": {
          "file": "scan_controller.ts",
          "function": "handleScan",
          "line": 20
        },
        "change_type": "新增",
        "before_code": "function handleScan(req) {\n    validateRequest(req);\n    process(req);\n    return result;\n}",
        "after_code": "function handleScan(req) {\n    validateRequest(req);\n    logger.info({ logId: req.logId }, \"scan request received\");\n    process(req);\n    return result;\n}",
        "diff_context": "改动位置前后各5行代码...",
        "steps": [
          "在 validateRequest(req) 后插入一行",
          "新增 logger.info({ logId: req.logId }, \"scan request received\")",
          "确认不影响原有逻辑"
        ],
        "verify_methods": [
          "运行项目测试，确认测试通过",
          "手动调用接口，确认日志输出正确"
        ],
        "risks": ["日志格式需与监控系统兼容"],
        "rollback_steps": ["删除新增的 logger.info 行"]
      }
    ]
  }
}
```

**字段说明：**

| 字段 | 必填 | 说明 |
|------|------|------|
| `location.file` | ✓ | 文件路径 |
| `location.function` | ✓ | 函数名 |
| `location.line` | ✓ | 行号（精确插入位置） |
| `before_code` | ✓ | 改动前完整代码（不少于5行上下文） |
| `after_code` | ✓ | 改动后完整代码（不少于5行上下文） |
| `steps` | ✓ | 具体操作步骤数组（每一步必须明确） |
| `verify_methods` | ✓ | 验证方法数组 |
| `risks` | ○ | 潜在风险（可选） |
| `rollback_steps` | ○ | 回滚步骤（可选） |

---

### 多模块输出结构

涉及多个模块时，文档按模块分开：

```
docs/iteration-2026-04-28/
├── CHANGELOG.json          # 总览清单（所有模块的改动项）
├── 支付模块/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
├── 订单模块/
│   ├── IMPLEMENTATION_GUIDE.json
│   ├── IMPACT_ANALYSIS.md
│   └── EXECUTION_LOG.json
└── 用户模块/
    ├── IMPLEMENTATION_GUIDE.json
    ├── IMPACT_ANALYSIS.md
    └── EXECUTION_LOG.json
```

---

## 安装

```bash
# 克隆仓库
git clone https://github.com/meetz7/iteration-planning.git

# 创建符号链接到 Claude skills 目录
ln -s $(pwd)/iteration-planning ~/.claude/skills/iteration-planning
```

---

## 使用

### 触发方式

- 直接调用：`/iteration-planning`
- 自动触发：说以下任意一句
  - "我有一堆改动要做"
  - "某个模块/接口改动xxx"
  - "帮我整理改动清单"
  - "这些改动帮我梳理一下"

---

### 使用示例

**场景：支付接口加日志，订单模块同步状态**

```
用户：支付接口加个日志记录请求参数，订单模块同步状态

AI 执行：
┌─────────────────────────────────────────────────────────┐
│ Step 1: 检索代码                                         │
│   - 找到支付接口：src/pay/scan_controller.ts             │
│   - 找到订单模块：src/order/order_service.ts             │
│   - Read 理解代码结构、调用链                            │
├─────────────────────────────────────────────────────────┤
│ Step 2: 分析影响                                         │
│   - 支付接口：上游 2 个调用方，下游依赖 payment_gateway   │
│   - 订单模块：上游 3 个调用方，依赖支付接口的返回值       │
│   - 边界条件：并发场景、异常处理                         │
├─────────────────────────────────────────────────────────┤
│ Step 3: 拆改动项                                         │
│   - 支付模块：#1 新增日志、#2 参数加 logId、#3 测试       │
│   - 订单模块：#4 状态同步、#5 调用方适配、#6 测试         │
├─────────────────────────────────────────────────────────┤
│ Step 4: 整理清单                                         │
│   - 支付模块（无依赖）→ 可立即执行                       │
│   - 订单模块（依赖支付模块完成）→ 等待                   │
│   - 标注：支付模块可与其他无依赖模块并行                  │
├─────────────────────────────────────────────────────────┤
│ Step 5: 确认清单                                         │
│   - 展示清单，用户核对                                   │
│   - 用户补充："还需要加个日志开关"                       │
│   - 更新清单                                             │
├─────────────────────────────────────────────────────────┤
│ Step 6: 逐项执行                                         │
│   - 即将执行 #1: 新增日志记录参数                        │
│   - 确认执行？ → 用户确认 → 执行 → 完成 ✓                │
│   - 下一项 #2...                                         │
├─────────────────────────────────────────────────────────┤
│ Step 7: 输出文档                                         │
│   - docs/iteration-2026-04-28/                           │
│     ├── CHANGELOG.json                                   │
│     ├── 支付模块/IMPLEMENTATION_GUIDE.json               │
│     ├── 订单模块/IMPLEMENTATION_GUIDE.json               │
│     ...                                                  │
└─────────────────────────────────────────────────────────┘
```

---

### 如何用输出文档执行代码

**场景：你把文档给另一个 AI（或另一个会话），让它执行代码实现**

```
另一个 AI 执行流程：

1. Read CHANGELOG.json
   - 解析 items[] → 获取所有改动项
   - 解析 execution_order[] → 获取执行顺序

2. Read IMPLEMENTATION_GUIDE.json
   - 对每个 item：
     a. 解析 location → 定位文件、函数、行号
     b. 对比 before_code 与当前文件代码 → 确认一致
     c. 按 steps[] 逐条执行 → 不跳步、不漏步
     d. 按 verify_methods[] 验证 → 确认正确

3. 更新 EXECUTION_LOG.json
   - 记录每项执行结果

4. 如果遇到问题
   - 停止执行，填写 error_message
   - 如需回滚，按 rollback_steps[] 执行
```

**为什么用 JSON 不会出错：**

| 风险 | Markdown 可能出错 | JSON 不会出错 |
|------|-------------------|---------------|
| 漏字段 | 表格解析可能漏"具体步骤"或"验证方法" | JSON 缺字段会报错 |
| 位置不精确 | `位置: scan_controller.ts:handleScan():20` → AI 可能解析偏差 | `{ "file": "...", "function": "...", "line": 20 }` → 精确 |
| 执行顺序混乱 | 列表解析可能跳项或顺序错误 | `{ "execution_order": [1, 2, 3] }` → 数组顺序固定 |

---

## 注意事项

| 注意点 | 说明 |
|--------|------|
| 不要添加未要求的改动 | 只做清单里的内容，不"顺便优化" |
| 理解代码后再拆项 | 不凭猜测拆分，先 Read 理解 |
| 分析影响要全面 | 上游、下游、边界、风险都要分析 |
| 文档要足够详细 | before_code 和 after_code 要有完整上下文（不少于5行） |
| 阻塞时立即汇报 | 不自行跳过或绕过 |

---

## 适用人群

- 使用 Claude Code 的开发者
- 有迭代改动需求的团队
- 需要 AI 执行代码但担心偏差的场景
- 多模块、多业务线的复杂改动

---

## License

MIT