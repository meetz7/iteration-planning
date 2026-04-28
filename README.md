# iteration-planning

一个 Claude Code 技能，从清晰方向出发，细化需求、分析影响、整理清单、逐项执行，最后输出完整文档。

## 适用场景

- **方向清晰但细节待细化**（如"被扫接口里改动xxx"）
- 改动琐碎但数量多
- 需要分析影响范围、调用链、边界条件
- 需要输出持久化文档（可作为执行依据）

**不适用：**
- 完全模糊需求 → 用 brainstorming
- 单一简单改动 → 直接对话处理

## 流程

```
Step 1: 收集方向 + 检索代码
        用户说"xx接口改动xxx" → 检索代码 → 理解结构

Step 2: 分析影响范围
        调用链、上游下游、边界条件、风险点

Step 3: 拆具体改动项
        方向 + 影响 → 具体改动点（文件:函数:行号）

Step 4: 整理清单 + 排序
        结构化清单 + 依赖顺序

Step 5: 确认清单
        用户核对，补充，调整

Step 6: 逐项执行
        每项确认后执行

Step 7: 输出文档
        4类文档（可作为执行依据）
```

## 输出文档

执行结束后生成到 `docs/iteration-{日期}/`：

| 文件 | 内容 |
|------|------|
| `CHANGELOG.md` | 改动清单（改动项、位置、状态） |
| `IMPACT_ANALYSIS.md` | 影响范围分析 + 改动前后代码对比 |
| `EXECUTION_LOG.md` | 执行记录（时间、结果） |
| `IMPLEMENTATION_GUIDE.md` | 可执行代码文档（按文档可重新实现） |

## 安装

```bash
# 克隆仓库
git clone https://github.com/meetz7/iteration-planning.git

# 创建符号链接到 Claude skills 目录
ln -s $(pwd)/iteration-planning ~/.claude/skills/iteration-planning
```

## 使用

### 触发方式

- 直接调用：`/iteration-planning`
- 自动触发：说"我有一堆改动要做"、"被扫接口里改动xxx"

### 示例

```
用户：被扫接口里加个日志，记录请求参数

技能执行：
1. 检索找到 scan_controller.ts
2. Read 理解代码结构、调用链
3. 分析影响：上游 3 个调用方、下游依赖 scan_service
4. 拆改动项：#1 新增日志、#2 参数加 logId、#3 调用方传递、#4 测试
5. 整理清单，确认后执行
6. 输出 4 类文档到 docs/iteration-2026-04-28/
```

## License

MIT