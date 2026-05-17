# agentUniverse AI Agent 系统学习路径

> 面向全栈工程师（Node.js 背景），从零开始系统学习 AI Agent。
> 以 agentUniverse 项目为实战基地，每章结合源码理解概念。

## 学习路线图

| 阶段 | 主题 | 文档 | 状态 |
|------|------|------|------|
| 准备 | Python 入门（Node.js → Python） | [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md) | ✅ 已完成 |
| 01 | Agent 基础概念 | [02-agent-fundamentals.md](02-agent-fundamentals.md) | ✅ 已完成 |
| 02 | 框架启动流程 & 组件系统 | [03-framework-startup.md](03-framework-startup.md) | ✅ 已完成 |
| 03 | Prompt 工程 & 版本管理 | [04-prompt-engineering.md](04-prompt-engineering.md) | ✅ 已完成 |
| 04 | ReAct 模式：推理与行动 | [05-react-pattern.md](05-react-pattern.md) | ✅ 已完成 |
| 05 | Tool 系统：给 Agent 装上手脚 | [06-tool-system.md](06-tool-system.md) | ✅ 已完成 |
| 06 | Memory 系统：让 Agent 记住上下文 | [07-memory-system.md](07-memory-system.md) | ✅ 已完成 |
| 07 | RAG：领域知识注入 | [08-rag-knowledge.md](08-rag-knowledge.md) | ✅ 已完成 |
| 08 | 多 Agent 协作：GRR 模式 | [09-grr-pattern.md](09-grr-pattern.md) | ✅ 已完成 |
| 09 | 多 Agent 协作：IS 模式 | [10-is-pattern.md](10-is-pattern.md) | ✅ 已完成 |
| 10 | 多 Agent 协作：PEER 模式 | [11-peer-pattern.md](11-peer-pattern.md) | ✅ 已完成 |
| 11 | 实战：构建自己的 Agent 应用 | [12-building-your-own.md](12-building-your-own.md) | ✅ 已完成 |

## 如何使用

1. **按顺序阅读**：每篇文档都建立在前一篇基础上。
2. **对照源码**：每个概念都会引用 `agentuniverse/` 目录下的真实代码。
3. **动手实践**：每章末尾有练习任务，使用 `examples/` 下的示例代码。

## 前置条件

- 基本的编程能力（理解函数、类、REST API、数据库等概念）
- 完成 [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md)
- 本地已 clone 项目并安装依赖：`poetry install`
