# iteration-planning

一个 Claude Code 技能，用于处理清晰需求的迭代改动。

## 适用场景

- 用户需求已清晰（知道要改什么）
- 改动琐碎但数量多
- 需要有条理、不遗漏、有顺序的执行

**不适用：**
- 模糊需求 → 用 brainstorming
- 单一简单改动 → 直接对话处理

## 安装

### 方法一：直接下载

```bash
# 克隆仓库
git clone https://github.com/meetz/iteration-planning.git

# 创建符号链接到 Claude skills 目录
ln -s $(pwd)/iteration-planning ~/.claude/skills/iteration-planning
```

### 方法二：手动安装

1. 下载 `SKILL.md` 文件
2. 放到 `~/.agents/skills/iteration-planning/SKILL.md`
3. 创建符号链接：
   ```bash
   ln -s ~/.agents/skills/iteration-planning ~/.claude/skills/iteration-planning
   ```

## 使用

### 触发方式

- 直接调用：`/iteration-planning`
- 自动触发：说"我有一堆改动要做"、"帮我整理改动清单"

### 流程

1. 你陈述改动内容（可口语化）
2. 整理成结构化清单（改动项 + 位置 + 类型）
3. 按模块分组 + 按依赖顺序排序
4. 确认清单（你核对，补充遗漏）
5. 逐项执行（每项完成前确认）

## License

MIT