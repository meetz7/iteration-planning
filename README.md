# iteration-planning

一个 Claude Code 技能，从清晰方向出发，细化需求、分析影响、整理清单、逐项执行，最后输出完整文档。

## 适用场景

- **方向清晰但细节待细化**（如"某个模块/接口改动xxx"）
- 改动琐碎但数量多
- **涉及多个模块或多个业务线**
- 需要分析影响范围、调用链、边界条件
- 需要输出持久化文档（给 AI 执行用，防止实现偏差）

**不适用：**
- 完全模糊需求 → 用 brainstorming
- 单一简单改动 → 直接对话处理

## 流程

```
Step 1: 收集方向 + 检索代码
        用户说改动方向 → 检索代码 → 理解结构

Step 2: 分析影响范围
        调用链、上游下游、边界条件、风险点

Step 3: 拆具体改动项
        方向 + 影响 → 具体改动点（文件:函数:行号）

Step 4: 整理清单 + 排序
        按模块/业务线分组 + 依赖顺序 + 并行执行提示

Step 5: 确认清单
        用户核对，补充，调整

Step 6: 逐项执行
        每项确认后执行

Step 7: 输出文档
        4类文档（给 AI 执行用，防止偏差）
```

## 输出文档

**文档目的：给 AI 执行用，防止代码实现偏差**

执行结束后生成到 `docs/iteration-{日期}/`：

| 文件 | 用途 |
|------|------|
| `CHANGELOG.md` | AI 定位所有改动项 |
| `IMPACT_ANALYSIS.md` | AI 理解代码结构和依赖关系 |
| `EXECUTION_LOG.md` | AI 回溯执行状态 |
| `IMPLEMENTATION_GUIDE.md` | AI 执行核心依据（精确位置+改动前后代码） |

**多模块/多业务线时：**

文档按模块分开，每个模块独立子目录（根据项目实际命名）：

```
docs/iteration-{日期}/
├── CHANGELOG.md      # 总览清单
├── {模块名1}/
│   ├── IMPACT_ANALYSIS.md
│   ├── IMPLEMENTATION_GUIDE.md
├── {模块名2}/
│   ├── IMPACT_ANALYSIS.md
│   ├── IMPLEMENTATION_GUIDE.md
```

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
- 自动触发：说"我有一堆改动要做"、"某个模块改动xxx"、"帮我整理改动清单"

### 示例

```
用户：{模块1}加个日志，{模块2}同步状态

技能执行：
1. 检索 {模块1}、{模块2} 代码
2. Read 理解两个模块的代码结构、调用链
3. 分析影响：{模块1}无依赖，{模块2}依赖{模块1}
4. 拆改动项：{模块1} 2 项、{模块2} 3 项
5. 按模块分组，标注并行执行：{模块1}和其他无依赖模块可并行
6. 整理清单，确认后逐项执行
7. 输出文档：总览清单 + 每个模块独立文档目录
```

## License

MIT