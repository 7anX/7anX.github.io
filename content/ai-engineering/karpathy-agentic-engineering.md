---
title: "像 Karpathy 一样用编程 Agent"
date: 2026-06-30
draft: false
tags: ["AI Agent", "Karpathy", "Agentic Engineering", "Claude Code"]
categories: ["ai-engineering"]
description: "从 Karpathy 的 Agentic Engineering 实践中提炼出的编程 Agent 使用方法论。"
ShowToc: true
TocOpen: true
---

Karpathy 在 2025 年 12 月经历了一次转变：从 80% 手写代码变成 80% 让 Agent 写。他把这种新的工作方式叫做 **Agentic Engineering**——不是随便让 AI 生成代码不管质量，而是像项目经理一样指挥 Agent 团队，同时保持专业标准。

以下是从他的实践中提炼出的方法论，以及在 Claude Code 中的具体落地方式。

## 核心原则

1. **人负责"做什么"和"好不好"，Agent 负责"怎么做"**
2. **严格限制范围** — 每次只让 Agent 改一个小范围的东西
3. **人写指令文件，Agent 写代码** — 你编程的对象变了

> "你可以外包你的思考，但永远不能外包你的理解。"

## 实践 1：写好 CLAUDE.md

Karpathy 的 [AutoResearch](https://github.com/karpathy/autoresearch) 项目里，人唯一维护的文件是 `program.md`——一份给 Agent 的指令。Agent 根据这个文件决定做什么、怎么做。

在 Claude Code 里等价物是 `CLAUDE.md`，放在项目根目录：

```markdown
# 约定
- Go 1.23, 错误处理用 errors.Join
- 测试用 table-driven 风格
- commit message 用中文

# 架构
- pkg/scanner/ — 核心扫描逻辑
- internal/config/ — 配置加载

# 红线
- 不引入 CGO 依赖
- 所有对外请求必须带 timeout
```

写一次，每次对话 Claude 都会自动读取，省去反复解释项目背景。

## 实践 2：分解任务，限制范围

AutoResearch 之所以有效：Agent 只改一个文件 (`train.py`)，每次实验只跑 5 分钟。范围小 = 可控 + 可审查。

同样的道理：

```
❌ "帮我重构整个认证模块"

✅ 拆开：
  1. "把 token 校验逻辑抽成独立函数"
  2. "给新函数写单元测试"
  3. "把调用方改成用新函数"
```

每步做完可以审查，方向不对随时调整。

## 实践 3：写代码 → 审查 → 验证

Karpathy 的模式是 Agent 做完实验后检查是否改进，没改进就丢弃。

Claude Code 里的等价流程：

```
1. 描述需求 → Claude 写代码
2. /code-review        ← 静态审查，找 bug
3. /code-review --fix  ← 自动修复问题
4. /verify             ← 实际跑起来确认
5. 没问题 → commit
```

- `/code-review` 是让 Claude 审查当前 git diff，报告问题
- `/verify` 是让 Claude 真正运行项目，观察行为是否符合预期

不要只写不查。

## 实践 4：并行工作

Karpathy 在多个显示器上同时跑多个 Agent，各自干不同的 feature。

Claude Code 里可以：
- 同时开多个 Claude Code 终端窗口，各自做独立任务
- 用 Agent 工具并行派发子任务
- 用 Workflow 编排更复杂的流程

关键是任务之间**不能有依赖**，否则并行没意义。

## 实践 5：积累经验到 Memory

Karpathy 会不断迭代 `program.md`，把发现写进去让 Agent 下次更聪明。

Claude Code 里对应的是 Memory 系统。遇到值得记住的东西就说：

- "记住：这个项目部署用 `make deploy-prod`"
- "记住：xxx 库的 v3 有内存泄漏，用 v2"
- "记住：我习惯先写接口定义再写实现"

下次对话会自动加载这些上下文。

## 总结

| Karpathy 做的 | 你在 Claude Code 里做的 |
|---|---|
| 维护 `program.md` | 维护 `CLAUDE.md` |
| Agent 只改一个文件 | 每次只给一个小任务 |
| 实验后检查是否改进 | `/code-review` + `/verify` |
| 多 Agent 并行 | 多终端 / Agent 工具 |
| 迭代指令文件 | 积累 Memory |

核心就一句话：**你从写代码的人变成管 Agent 的人。**

---

参考：
- [karpathy/autoresearch](https://github.com/karpathy/autoresearch)
- [Karpathy at AI Ascent 2026 (Sequoia)](https://www.youtube.com/watch?v=96jN2OCIfLs)
- [Forbes: AI Agents Wrote 80% Of Karpathy's Code](https://www.forbes.com/sites/josipamajic/2026/03/22/ai-agents-wrote-80-of-karpathys-code-junior-developers-are-paying-the-price/)
