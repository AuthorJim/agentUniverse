# 05 — ReAct 模式：让 Agent 学会"思考→行动→观察"

> **前置要求：** 完成 [04-prompt-engineering.md](04-prompt-engineering.md)
> **学习目标：** 深入理解 ReAct 循环的原理、agentUniverse 的源码实现、LLM 如何自主决定使用工具
> **预计时间：** 2-3 天

---

## 1. 为什么纯 LLM 不够用？

### 1.1 普通 LLM 调用：一问一答，不会思考

```
用户: "2026 年 5 月 17 日的黄金价格是多少？"
LLM:  "抱歉，我的训练数据截止于 2024 年，无法提供 2026 年的实时数据。"
```

LLM 的知识是"冻结"的——它只能基于训练数据回答。不会自己查资料，不会自己验证，不会自己纠错。

### 1.2 有 ReAct 的 Agent：自己会查

```
用户: "2026 年 5 月 17 日的黄金价格是多少？"
Agent:
  Thought: 我需要查询实时金价，用 google_search 工具。
  Action: google_search("2026年5月17日 黄金价格")
  Observation: COMEX 黄金期货 $2,845/盎司...

  Thought: 数据有了，可以整理回复了。
  Final Answer: "2026年5月17日，COMEX黄金期货价格为 $2,845/盎司..."
```

**这就是 ReAct 的核心价值：LLM 自己判断什么时候需要工具、用哪个工具、怎么解读结果、什么时候结束。框架只负责"给工具"和"把结果传回去"。**

---

## 2. ReAct = Reasoning + Acting

ReAct 是 "Reasoning"（推理）和 "Acting"（行动）的合成词。每个循环包含三步：

```
┌────────────────────────────────────────────┐
│  Thought（思考）：分析现状，决定下一步       │
│      ↓                                     │
│  Action（行动）：调用工具 OR 输出最终答案     │
│      ↓                                     │
│  Observation（观察）：接收工具返回的结果      │
│      ↓                                     │
│  判断：完成了吗？没完成 → 回到 Thought       │
└────────────────────────────────────────────┘
```

### 2.1 ReAct 的 Prompt 格式（标准模板）

ReAct 使用一种特殊的格式引导 LLM 输出：

```
Thought: [你当前的推理过程]
Action: [工具名称]
Action Input: [工具参数 JSON]
Observation: [工具返回结果 — 由框架注入]
...（多轮循环）...
Thought: 我现在有足够信息了
Final Answer: [最终自然语言回答]
```

### 2.2 一个完整的双轮例子

```
用户输入: "帮我算 123 * 456，用中文告诉我结果"
Agent 有 python_runner 工具

═══════════════════════════════════════════
Round 1
═══════════════════════════════════════════

Thought: 我需要计算 123 × 456。我可以用 python_runner 执行 Python 代码。
Action: python_runner
Action Input: {"input": "print(123 * 456)"}

→ 框架执行 python_runner.execute(input="print(123 * 456)")
→ Observation: 56088

═══════════════════════════════════════════
Round 2
═══════════════════════════════════════════

Thought: 我得到了计算结果 56088，可以整理成中文回答。
Final Answer: 123 乘以 456 的结果是 56088（五万六千零八十八）。
```

---

## 3. agentUniverse 中的 ReAct 实现

### 3.1 代码分层架构

```
Agent 层: ReActAgentTemplate (agent/template/react_agent_template.py)
  │  Agent 模板，使用 ReAct 专用 Planner
  ▼
Planner 层: ReActPlanner (agent/plan/planner/react_planner/react_planner.py)
  │  组装 Prompt + Tool + LLM，交给 LangChain 的 AgentExecutor
  ▼
执行层 (LangChain): AgentExecutor
  │  控制 Thought → Action → Observation 的循环执行
  ▼
LLM API: 真正的 AI 推理
```

### 3.2 ReActPlanner.invoke() 源码穿透

```python
# agentuniverse/agent/plan/planner/react_planner/react_planner.py:46-77
def invoke(self, agent_model, planner_input, input_object) -> dict:
    """ReAct 核心执行 — 7 个步骤"""

    # ① 加载 Memory — 获取历史对话
    memory = self.handle_memory(agent_model, planner_input)

    # ② 获取 LLM — 从 LLMManager 取实例
    llm = self.handle_llm(agent_model)

    # ③ 获取所有 Tool — 从 ToolManager 取，转为 LangChain 格式
    tools = self.acquire_tools(agent_model.action)

    # ④ 组装 Prompt — 注入工具描述 + 用户输入
    prompt = self.handle_prompt(agent_model, planner_input)

    # ⑤ 创建 ReAct Agent（LangChain）
    agent = create_react_agent(
        llm.as_langchain(),
        tools,
        prompt.as_langchain(),
        stop_sequence=['\nObservation'],  # ★ 关键：告诉 LLM 在 Action 后停止
        bind_params=agent_model.llm_params()
    )

    # ⑥ 包装为 AgentExecutor（添加循环控制 + 安全阀）
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,                  # 打印详细日志（调试神器）
        handle_parsing_errors=True,    # 解析出错自动重试
        max_iterations=15              # ★ 最多 15 轮！安全阀
    )

    # ⑦ 执行！
    return agent_executor.invoke(
        input=planner_input,
        memory=memory.as_langchain() if memory else None,
        chat_history=planner_input.get('chat_history'),
        ...
    )
```

### 3.3 create_react_agent() 的三个关键设计

```python
def create_react_agent(llm, tools, prompt, ...):
    # ① 注入工具描述到 Prompt — 告诉 LLM 有哪些工具、怎么用
    prompt = prompt.partial(
        tools=render_text_description(list(tools)),    # 工具描述文本
        tool_names=", ".join([t.name for t in tools]), # 工具名列表
    )

    # ② 给 LLM 绑定 stop word — 生成 Action 后自动停止
    llm_with_stop = llm.bind(stop=["\nObservation"], **bind_params)

    # ③ 构建处理链
    agent = (
        RunnablePassthrough.assign(
            # ★ 把之前的步骤格式化为 scratchpad
            agent_scratchpad=lambda x: format_log_to_str(x["intermediate_steps"]),
        )
        | prompt              # 注入 Prompt（含工具描述）
        | llm_with_stop       # 调用 LLM（遇 Observation 就停）
        | output_parser       # 解析 LLM 输出 → 提取 Thought / Action
    )
    return agent
```

**三个关键设计的解释：**

1. **工具描述注入（`tools` 占位符）** — Prompt 模板中的 `{tools}` 和 `{tool_names}` 被替换为实际的工具信息。这样 LLM 才知道"我有哪些武器可用"。

2. **stop word `\nObservation`** — LLM 生成到 `Action: xxx` 后自动停止，因为框架会负责执行工具并注入 `\nObservation: 结果`。LLM 不需要（也不应该）自己编造 Observation。

3. **agent_scratchpad** — 把之前的所有循环步骤（Thought → Action → Observation）格式化后塞进新 Prompt，让 LLM 看到"我之前做了什么"。这样 LLM 不会重复调用同一个工具或忘记之前的推理。

### 3.4 `max_iterations=15`：为什么需要安全阀？

```python
AgentExecutor(..., max_iterations=15)
```

如果 LLM 陷入"死循环"——比如反复调用同一个工具但得不到有效结果——15 轮后 AgentExecutor 会强制终止并返回当前状态。防止：
- LLM 幻觉导致无限循环
- Token 成本失控（每轮调用都要花钱）
- 用户请求超时

---

## 4. Tool 与 ReAct 的交互机制

### 4.1 Tool 如何被 LLM 发现？

框架将所有 Tool 的描述拼接成一个字符串，注入到 Prompt 中：

```python
# react_planner.py 中的 acquire_tools 和 handle_prompt
tools_str = ''
for tool in tools:
    tools_str += f"tool name: {tool.name}\ntool description: {tool.description}\n"

planner_input['tools'] = tools_str
planner_input['tool_names'] = '|'.join([t.name for t in tools])
```

**所以 Tool 的 `description` 是给 LLM 看的——它是决定 LLM 会不会用你的 Tool 的最关键因素。**

### 4.2 Tool 执行链路

```
LLM 输出: Action: google_search
          Action Input: {"query": "比特币价格"}

框架解析:
  → 提取 Action = "google_search"
  → 提取 Action Input = '{"query": "比特币价格"}'

Tool 执行:
  → tool.langchain_run('{"query": "比特币价格"}')
  → parse_and_check_json_markdown() → {'query': '比特币价格'}
  → tool.run(query='比特币价格')
  → tool.execute(query='比特币价格')
  → 返回: "比特币当前价格 $87,234"

框架处理:
  → 格式化为: \nObservation: 比特币当前价格 $87,234
  → 追加到 agent_scratchpad
  → 进入下一轮循环
```

---

## 5. ReAct 的局限和应对

| 局限 | 原因 | agentUniverse 的缓解方式 |
|------|------|------------------------|
| **Tool 选择失误** | LLM 理解不准确 | `handle_parsing_errors=True` 自动重试 |
| **无限循环** | LLM 死磕一个问题 | `max_iterations=15` 强制终止 |
| **解析失败** | LLM 输出格式不符合预期 | `output_parser` + 自动重试 |
| **幻觉** | LLM 编造 Observation | stop word `\nObservation` 防止 LLM 编造 |
| **Token 消耗大** | 多轮循环每轮都调用 LLM | 合理设置 `max_iterations` 和 `max_tokens` |

---

## 6. agentUniverse 中的不同 Planner

ReAct 不是唯一的 Planner，agentUniverse 还有：

| Planner | 适用场景 | 与 ReAct 的关系 |
|---------|---------|---------------|
| `ReActPlanner` | 通用推理 + 工具调用 | 基础范式 |
| `RagPlanner` | 知识库问答 | ReAct 变体（偏检索，轻推理） |
| `PeerPlanner` | PEER 多 Agent | 每个步骤是一个子 Agent |
| `ExecutingPlanner` | 知识整合 | 侧重信息整理而非工具调用 |
| `ReviewingPlanner` | 质量评审 | 侧重评分和反馈 |

---

## 7. 动手练习

### 练习 1：追踪 ReActPlanner.invoke() 源码

打开 `agentuniverse/agent/plan/planner/react_planner/react_planner.py`，逐行阅读 `invoke()` 方法。标注每个步骤的作用和输入输出。

### 练习 2：运行 react_agent_app 观察循环

```bash
cd examples/sample_apps/react_agent_app
# 配置 API Key 后启动
python bootstrap/intelligence/server_application.py
```
问一个需要工具的复杂问题，观察控制台输出的 `verbose` 日志（每轮 Thought 和 Action）。

### 练习 3：修改 max_iterations 看效果

把 `max_iterations` 从 15 改为 2，问需要多步工具调用的问题。观察第 2 轮后被强制终止时的输出。

### 练习 4：手动模拟 ReAct

不用代码，用纸笔模拟以下场景的完整 ReAct 循环：

> 用户: "巴黎人口多少？把这个数字的平方根算出来，用中文回答。"
> Agent 有: google_search 和 python_runner

写出每一轮的 Thought → Action → Observation，直到 Final Answer。

### 练习 5：理解 stop word 机制

在 `create_react_agent()` 中，`stop=["\nObservation"]` 的作用是什么？如果去掉这个 stop word，会发生什么？试着推理。

---

## 8. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **ReAct** | Reasoning + Acting 循环 | 本文核心 |
| **Thought** | LLM 的推理步骤 | ReAct Prompt 格式 |
| **Action** | 调用的工具名 + 参数 | — |
| **Observation** | 工具返回结果（框架注入） | — |
| **ReActPlanner** | agentUniverse 的 ReAct 实现 | `react_planner/react_planner.py` |
| **create_react_agent()** | 构建 ReAct 执行链 | `react_planner.py:150` |
| **AgentExecutor** | LangChain 的循环控制器 | `react_planner.py:70` |
| **max_iterations** | 最大循环轮数（安全阀，默认 15） | — |
| **agent_scratchpad** | 历史步骤的格式化字符串 | `format_log_to_str()` |
| **stop_sequence** | LLM 停止标记（`\nObservation`） | `create_react_agent()` |
| **ReActAgentTemplate** | ReAct 模式的 Agent 模板 | `agent/template/react_agent_template.py` |

---

## 下一步

你已经完全理解了 ReAct 循环。下一步进入 **[06-tool-system.md](06-tool-system.md)**——学习如何为 Agent 编写自定义 Tool，让 Agent 能调用任何你需要的功能。
