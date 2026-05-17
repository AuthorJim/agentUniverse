# agentUniverse AI Agent 系统学习路径

> 面向全栈工程师（Node.js 背景），从零开始系统学习 AI Agent。
> 以 agentUniverse 项目为实战基地，每章结合源码理解概念。

---

## 你在哪里，要去哪里？

假设你是一个经验丰富的 Node.js 全栈工程师——你能写 Express API、设计数据库 Schema、部署微服务。现在你想搞懂 AI Agent 是怎么回事，而且不只停留在"调个 API"的层面，而是真正理解：**Agent 的大脑（LLM）、手脚（Tool）、记忆（Memory）、专业知识（RAG）、以及多个 Agent 怎么像一支工程团队一样协作（PEER/GRR/IS）。**

这就是这套文档要做的事。

和你看过的那些浮光掠影的"十分钟入门"不同，这套文档把你丢进一个真实的、生产级的开源多 Agent 框架——**agentUniverse**（蚂蚁集团开源）——让你在实战中学习。每个概念都直接对应到源码中的真实文件和实际运行逻辑。

---

## 学习路线图：从独奏到交响乐

整套文档按照一条清晰的线索展开——**先学会操作单个 Agent，再学会指挥多个 Agent 协同作战**：

```
┌───────────────────────────────────────────────────────────┐
│                                                           │
│  准备阶段        单体 Agent         多 Agent 协作          │
│  ─────────      ──────────        ──────────────         │
│                                                           │
│  Python      →  Agent 是什么  →  GRR（3人创作组）         │
│  速成           框架怎么启动      IS（执行+监督二人组）    │
│                  Prompt 怎么写    PEER（4人专业团队）      │
│                  ReAct 怎么工作                            │
│                  Tool/Memory/RAG                           │
│                                                           │
│  ──────────────────────────────────────────────────────→  │
│                    复杂度递增                              │
└───────────────────────────────────────────────────────────┘
```

| 阶段 | 章节 | 主题 | 核心问题 |
|------|------|------|---------|
| **准备** | [01](01-python-for-nodejs-developers.md) | Python 速成（Node.js → Python） | 怎么读懂 Python 源码？ |
| **单体 Agent** | [02](02-agent-fundamentals.md) | Agent 基础概念 | Agent 到底是什么？ |
| | [03](03-framework-startup.md) | 框架启动 & 组件系统 | start() 背后发生了什么？ |
| | [04](04-prompt-engineering.md) | Prompt 工程 & 版本管理 | 怎么给 AI 写"任务说明书"？ |
| | [05](05-react-pattern.md) | ReAct 推理模式 | Agent 怎么"思考"？ |
| | [06](06-tool-system.md) | Tool 系统 | 怎么给 Agent 装"手脚"？ |
| | [07](07-memory-system.md) | Memory 系统 | Agent 怎么"记住"上下文？ |
| | [08](08-rag-knowledge.md) | RAG 知识注入 | Agent 怎么读你的私有文档？ |
| **多 Agent** | [09](09-grr-pattern.md) | GRR 模式 | 3 个 Agent 怎么像 CI/CD 一样协作？ |
| | [10](10-is-pattern.md) | IS 模式 | 执行+监督二人组怎么工作？ |
| | [11](11-peer-pattern.md) | PEER 模式 | 4 个 Agent 怎么像一支工程团队？ |
| **实践** | [12](12-building-your-own.md) | 从零搭建应用 | 怎么把学到的全用上？ |

---

## 怎么学效果最好？

**1. 按顺序，别跳。** 每篇文档都建立在前一篇的基础上。跳过中间章节就像盖楼跳过第三层——后面的你会看不懂。

**2. 打开源码对照着读。** 每个概念都会标注对应的源文件和行号。不要让代码停在屏幕上——clone 项目，边读文档边看源码。

**3. 跑起来。** 每章末尾有动手练习。文档可以告诉你"是什么"，但只有亲手运行、修改、调试，你才能理解"为什么"。

**4. 用你已有的知识做桥梁。** 作为 Node.js 工程师，你脑子里已经有了一套完整的后端架构模型。每当你遇到一个新概念，问自己："这在我的 Node.js 世界里是什么？"

---

## 开始前的准备

- 基本的编程能力（函数、类、REST API、数据库等概念——你显然已经有了）
- Python 环境（3.10+，推荐用 `pyenv` 管理版本）
- 已完成 [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md)（或至少读完 Python 语法基础）
- 本地已 clone 项目：

```bash
git clone https://github.com/agentuniverse-ai/agentUniverse.git
cd agentUniverse
poetry install
```

---

## 开始之前：先学会怎么读

**强烈建议在开始学习之前，先花 20 分钟阅读这篇文章：**

- [如何高效阅读这套文档 —— 从"读过就忘"到"真正掌握"](reading-methods/01-how-to-read-this-guide.md) — 基于认知科学的学习方法，让你 12 章的知识留存率从 10% 提升到 50%+

## 补充阅读

- [pyproject.toml 完全指南（Node.js 工程师视角）](python-ecosystem/pyproject-toml-guide.md) — 读完 01 后建议看这篇，理解 Python 的包管理生态

---

准备好了吗？从 [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md) 开始你的 AI Agent 之旅。
