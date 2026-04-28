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

按模块或业务线自动分组，每组标注依赖关系：

```markdown
## 改动清单

### 支付模块（依赖：无）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 1 | 新增日志记录参数 | 新增 | scan_controller.ts:handleScan():20 | 待执行 |

### 订单模块（依赖：支付模块完成）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 2 | 订单状态同步 | 修改 | order_service.ts:updateStatus() | 待执行 |

### 用户模块（依赖：无）
| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 3 | 用户积分更新 | 新增 | user_service.ts:addPoints() | 待执行 |
```

**排序规则：**
- 按模块/业务线分组
- 无依赖的模块 → 排前面，可并行执行
- 有依赖的模块 → 排后面，等待前置模块
- 同模块内：底层 → 上层

**并行执行提示：**

如果多个模块无依赖关系，标注可并行：
```
支付模块 + 用户模块 → 无依赖，可并行执行
订单模块 → 依赖支付模块，需等待
```

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

文档特点：
- **精确性** — 文件路径、函数名、行号精确标注
- **结构化** — 表格形式，便于 AI 解析
- **可执行性** — 包含改动前后代码对比，按文档可精确复现
- **完整性** — 覆盖所有改动项，无遗漏

---

#### 1. 改动清单文档

**文件：** `CHANGELOG.md`

**用途：** AI 快速定位所有改动项，按清单逐项执行

**内容：**
```markdown
# 改动清单

**项目：** xxx
**日期：** 2026-04-28
**触发方向：** 被扫接口加日志

---

## 改动项

| # | 改动项 | 类型 | 位置 | 状态 |
|---|--------|------|------|------|
| 1 | 新增日志记录参数 | 新增 | scan_controller.ts:handleScan():20 | 已完成 |
| 2 | 参数结构新增logId | 修改 | scan_service.ts:process() | 已完成 |

---

## 执行顺序

1. #1 — 无依赖，优先执行
2. #2 — 依赖 #1 的 logId
3. #3 — 依赖 #2 的参数结构
```

---

#### 2. 影响范围分析文档

**文件：** `IMPACT_ANALYSIS.md`

**用途：** AI 理解代码结构和依赖关系，确保改动不遗漏调用方

**关键信息（AI 必读）：**
- 调用链路径（从入口到底层）
- 上游调用方列表（改动返回值需同步）
- 下游依赖列表（改动参数需同步）
- 边界条件（异常处理、并发场景）

**内容：**
```markdown
# 影响范围分析

## 代码结构

### scan_controller.ts
- handleScan() — 入口，处理被扫请求
- 参数：ScanRequest { merchantId, amount, deviceId }
- 返回：ScanResponse { orderId, status }

### scan_service.ts
- process() — 处理逻辑
- 调用 payment_gateway.scan()

---

## 调用链

```
api/routes.ts:scanRoute()
  → scan_controller.ts:handleScan()
    → scan_service.ts:process()
      → payment_gateway.scan()
```

---

## 上游调用方

| 文件 | 函数 | 影响说明 |
|------|------|----------|
| api/routes.ts | scanRoute() | 需传递 logId |
| tests/scan_controller_test.ts | test_handleScan | 需更新测试数据 |

---

## 下游依赖

| 文件 | 函数 | 影响说明 |
|------|------|----------|
| scan_service.ts | process() | 参数新增 logId |

---

## 边界条件

- 并发场景：同一订单重复扫描 → 无影响（logId 由调用方生成）
- 异常处理：payment_gateway 超时 → 日志已记录，无影响

---

## 改动后代码说明

### handleScan() 改动后

```typescript
// 改动前
function handleScan(req: ScanRequest): ScanResponse {
  const result = scanService.process(req);
  return result;
}

// 改动后
function handleScan(req: ScanRequest): ScanResponse {
  logger.info({ logId: req.logId, merchantId: req.merchantId }, "scan request received");
  const result = scanService.process(req);
  return result;
}
```

### ScanRequest 改动后

```typescript
// 改动前
interface ScanRequest {
  merchantId: string;
  amount: number;
  deviceId: string;
}

// 改动后
interface ScanRequest {
  merchantId: string;
  amount: number;
  deviceId: string;
  logId: string; // 新增
}
```
```

---

#### 3. 执行记录文档

**文件：** `EXECUTION_LOG.md`

**用途：** AI 回溯执行状态，确认哪些已完成、哪些阻塞

**内容：**
```markdown
# 执行记录

## 2026-04-28 iteration-planning 执行

**总项数：** 4
**已完成：** 4
**耗时：** 15 分钟

---

## 执行明细

| 时间 | # | 改动项 | 结果 |
|------|---|--------|------|
| 14:01 | 1 | 新增日志记录参数 | ✅ 已完成 |
| 14:03 | 2 | 参数结构新增logId | ✅ 已完成 |
| 14:05 | 3 | 调用时传递logId | ✅ 已完成 |
| 14:08 | 4 | 新增测试用例 | ✅ 已完成 |

---

## 遗留事项

无
```

---

#### 4. 可执行代码文档

**文件：** `IMPLEMENTATION_GUIDE.md`

**用途：** AI 执行代码实现的核心依据，防止偏差

**关键要素（必须包含）：**
- **精确位置** — 文件路径 + 函数名 + 行号
- **改动前代码** — 原始代码片段（完整上下文）
- **改动后代码** — 目标代码片段（完整上下文）
- **具体步骤** — 每一步操作指令
- **验证方法** — 如何确认改动正确

**AI 执行要求：**
- Read 文档后，必须对比改动前代码与当前文件代码是否一致
- 按"具体步骤"逐条执行，不跳步、不漏步
- 执行后按"验证方法"确认结果

```markdown
# 实现指南

本文档包含所有改动项的完整实现细节，可按文档重新实现。

---

## #1 新增日志记录参数

**位置：** `src/pay/scan_controller.ts:handleScan() 第20行`

**改动前代码：**
```typescript
function handleScan(req: ScanRequest): ScanResponse {
  validateRequest(req);
  const result = scanService.process(req);
  return result;
}
```

**改动后代码：**
```typescript
function handleScan(req: ScanRequest): ScanResponse {
  validateRequest(req);
  logger.info({ logId: req.logId, merchantId: req.merchantId }, "scan request received");
  const result = scanService.process(req);
  return result;
}
```

**具体步骤：**
1. 在 `validateRequest(req)` 后插入一行
2. 使用 `logger.info()` 记录日志
3. 日志内容：`{ logId, merchantId }`
4. 日志消息：`"scan request received"`

**验证方法：**
- 运行 `npm test`，确认测试通过
- 手动调用接口，确认日志输出正确

---

## #2 参数结构新增 logId

**位置：** `src/pay/scan_service.ts:ScanRequest interface`

**改动前代码：**
```typescript
interface ScanRequest {
  merchantId: string;
  amount: number;
  deviceId: string;
}
```

**改动后代码：**
```typescript
interface ScanRequest {
  merchantId: string;
  amount: number;
  deviceId: string;
  logId: string; // 新增：日志追踪ID
}
```

**具体步骤：**
1. 在 `ScanRequest` interface 最后新增一行
2. 字段名：`logId`，类型：`string`
3. 添加注释说明用途

**验证方法：**
- TypeScript 编译无错误
- 下游调用方已适配

---

## #3 调用时传递 logId

**位置：** `api/routes.ts:scanRoute()`

**改动前代码：**
```typescript
router.post('/scan', (req, res) => {
  const scanReq = {
    merchantId: req.body.merchantId,
    amount: req.body.amount,
    deviceId: req.body.deviceId
  };
  const result = handleScan(scanReq);
  res.json(result);
});
```

**改动后代码：**
```typescript
router.post('/scan', (req, res) => {
  const logId = generateLogId(); // 新增
  const scanReq = {
    merchantId: req.body.merchantId,
    amount: req.body.amount,
    deviceId: req.body.deviceId,
    logId // 新增
  };
  const result = handleScan(scanReq);
  res.json(result);
});
```

**具体步骤：**
1. 在 `scanReq` 构建前调用 `generateLogId()` 生成 logId
2. 将 `logId` 加入 `scanReq` 对象
3. 确认 `generateLogId()` 函数已存在（如不存在需新增）

**验证方法：**
- 调用接口，确认返回结果包含 logId
- 日志中 logId 与请求一致

---

## #4 新增测试用例

**位置：** `tests/scan_controller_test.ts`

**新增代码：**
```typescript
describe('handleScan with logId', () => {
  it('should log request with logId', () => {
    const req = {
      merchantId: 'M001',
      amount: 100,
      deviceId: 'D001',
      logId: 'LOG-001'
    };
    const result = handleScan(req);
    expect(result.status).toBe('success');
    // 验证日志输出
    expect(logger.info).toHaveBeenCalledWith(
      { logId: 'LOG-001', merchantId: 'M001' },
      'scan request received'
    );
  });
});
```

**具体步骤：**
1. 新增 describe block
2. 构造带 logId 的测试请求
3. 验证返回结果正确
4. 验证日志输出正确

**验证方法：**
- `npm test` 全部通过
```

---

## 输出目录结构

```
docs/iteration-{YYYY-MM-DD}/
├── CHANGELOG.md          # 改动清单（AI 定位所有改动项）
├── IMPACT_ANALYSIS.md    # 影响范围分析（AI 理解依赖关系）
├── EXECUTION_LOG.md      # 执行记录（AI 回溯执行状态）
└── IMPLEMENTATION_GUIDE.md # 可执行代码文档（AI 执行核心依据）
```

**多模块/多业务线时：**

如果涉及多个模块或业务线，文档按模块分开：

```
docs/iteration-{YYYY-MM-DD}/
├── CHANGELOG.md          # 总览清单
├── 支付模块/
│   ├── IMPACT_ANALYSIS.md
│   ├── IMPLEMENTATION_GUIDE.md
│   └── EXECUTION_LOG.md
├── 订单模块/
│   ├── IMPACT_ANALYSIS.md
│   ├── IMPLEMENTATION_GUIDE.md
│   └── EXECUTION_LOG.md
└── 用户模块/
    ├── IMPACT_ANALYSIS.md
    ├── IMPLEMENTATION_GUIDE.md
    └── EXECUTION_LOG.md
```

---

## 注意事项

1. **不要添加未被要求的改动** — 只做用户清单里的内容
2. **理解代码后再拆项** — 不凭猜测拆分
3. **分析影响要全面** — 上游、下游、边界、风险
4. **文档要足够详细** — 能按文档重新实现
5. **阻塞时立即汇报** — 不要自行跳过或绕过