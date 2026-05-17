# 05 — ReAct 模式：思考→行动→观察

> **前置要求：** 完成 [04-prompt-engineering.md](04-prompt-engineering.md)
> **学习目标：** 深入理解 ReAct 循环的工作原理、源码实现，以及它为什么是 AI Agent 的核心范式
> **预计时间：** 2-3 天

---

## 1. 为什么需要 ReAct？普通 LLM 调用的局限

### 1.1 普通 LLM 调用：一问一答

```
用户: "2026年5月17日的黄金价格是多少？"
LLM:  "抱歉，我的知识截止于2024年，无法提供2026年的实时金价。"
```

LLM 的知识是"冻结的"——它只能基于训练数据回答，无法获取实时信息。

### 1.2 有工具的 Agent：自己查

```
用户: "2026年5月17日的黄金价格是多少？"
Agent:
  Thought: 我需要查询最新的黄金价格数据。
  Action: google_search("2026年5月17日 黄金价格")
  Observation: 搜索结果: COMEX黄金期货 $2,845/盎司...

  Thought: 我已经有了数据，可以整理回答了。
  Final Answer: "2026年5月17日，COMEX黄金期货价格为 $2,845/盎司..."
```

**这就是 ReAct 的核心价值：LLM 自己决定什么时候需要调用工具、调用哪个工具、怎么解读结果。**

---

## 2. ReAct 是什么？

**ReAct = Reasoning（推理）+ Acting（行动）**

它是一个循环模式，Agent 在每一轮中：
1. **Thought（推理）**：分析当前情况，决定下一步
2. **Action（行动）**：调用工具或输出最终答案
3. **Observation（观察）**：接收工具返回的结果或结束

### 2.1 完整的 ReAct 循环

```
┌──────────────────────────────────────────────────────────────┐
│ 用户输入: "今天北京天气如何？"                                    │
├──────────────────────────────────────────────────────────────┤
│                                                                │
│  Round 1:                                                      │
│    Thought: 我需要查询北京今天的天气，用 google_search 工具。    │
│    Action: google_search("北京 天气 2026-05-17")               │
│    Observation: "北京今日晴，15°C~25°C，北风3级"               │
│    → 还没完成，进入 Round 2                                     │
│                                                                │
│  Round 2:                                                      │
│    Thought: 我有天气数据了，整理成自然语言回复。                  │
│    Action: Final Answer                                        │
│    "北京今天天气晴朗，气温15到25摄氏度，北风3级..."              │
│    → 完成！                                                    │
│                                                                │
└──────────────────────────────────────────────────────────────┘
```

### 2.2 ReAct 的 Prompt 格式（标准模板）

ReAct 使用了一种特殊的 Prompt 格式来引导 LLM 按照 Thought → Action → Observation 的模式输出：

```
Thought: [你当前的推理]
Action: [工具名称]
Action Input: [工具参数]
Observation: [工具返回结果]
...（重复）...
Thought: 我现在可以给出最终答案了
Final Answer: [最终回答]
```

---

## 3. agentUniverse 中的 ReAct 实现

### 3.1 源码架构

```
Agent 层
  └─ ReActAgentTemplate (agent/template/react_agent_template.py)
       │  使用 ReAct 专用 Planner
       ▼
Planner 层
  └─ ReActPlanner (agent/plan/planner/react_planner/react_planner.py)
       │  组装 Prompt + Tool + LLM，交给 LangChain AgentExecutor
       ▼
执行层 (LangChain)
  └─ AgentExecutor
       │  执行 Thought → Action → Observation 循环
       ▼
LLM API
```

### 3.2 ReActPlanner.invoke() 完整源码走读

```python
# agentuniverse/agent/plan/planner/react_planner/react_planner.py:46-77
def invoke(self, agent_model: AgentModel, planner_input: dict,
           input_object: InputObject) -> dict:

    # 步骤 1：加载 Memory（对话历史）
    memory: Memory = self.handle_memory(agent_model, planner_input)

    # 步骤 2：获取 LLM 实例（大脑）
    llm: LLM = self.handle_llm(agent_model)

    # 步骤 3：获取所有注册的 Tool 并转为 LangChain 工具格式
    tools = self.acquire_tools(agent_model.action)

    # 步骤 4：组装 Prompt（系统指令 + 工具描述 + 用户输入）
    prompt: Prompt = self.handle_prompt(agent_model, planner_input)

    # 步骤 5：创建 ReAct Agent
    agent = create_react_agent(
        llm.as_langchain(),     # LLM
        tools,                   # 工具列表
        prompt.as_langchain(),  # Prompt 模板
        stop_sequence=['\nObservation'],  # 停止标记
        bind_params=agent_model.llm_params()
    )

    # 步骤 6：包装为 AgentExecutor（添加循环控制）
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        verbose=True,                      # 打印详细日志
        handle_parsing_errors=True,        # 解析错误时自动重试
        max_iterations=15                  # ★ 最多 15 轮！
    )

    # 步骤 7：执行
    return agent_executor.invoke(
        input=planner_input,
        memory=memory.as_langchain() if memory else None,
        chat_history=planner_input.get('chat_history'),
        config=self.get_run_config(agent_model, input_object)
    )
```

### 3.3 create_react_agent() 函数详解

```python
# react_planner/react_planner.py:150-184（简化）
def create_react_agent(llm, tools, prompt, output_parser=None, stop_sequence=True, bind_params=None):
    # 1. 给 Prompt 注入工具描述（告诉 LLM 有哪些工具可用）
    prompt = prompt.partial(
        tools=render_text_description(list(tools)),    # 工具描述
        tool_names=", ".join([t.name for t in tools]), # 工具名称列表
    )

    # 2. 给 LLM 绑定 stop word（遇到 \nObservation 就停止输出）
    llm_with_stop = llm.bind(stop=["\nObservation"], **(bind_params or {}))

    # 3. 构建 Agent 执行链
    agent = (
        RunnablePassthrough.assign(
            # 把之前的执行步骤格式化为 scratchpad
            agent_scratchpad=lambda x: format_log_to_str(x["intermediate_steps"]),
        )
        | prompt                # 注入 Prompt
        | llm_with_stop         # 调用 LLM
        | output_parser         # 解析 LLM 输出（提取 Thought/Action）
    )
    return agent
```

**三个关键设计：**

1. **工具描述注入**：Prompt 中的 `{tools}` 和 `{tool_names}` 占位符被替换为实际的工具信息，告诉 LLM 有什么工具、怎么用。

2. **stop word `\nObservation`**：LLM 输出到 `Action: xxx` 后停止，由框架执行工具并将结果以 `\nObservation: xxx` 格式追加回 Prompt。这样 LLM 不需要生成 Observation（那是框架的工作），只需要生成 Thought 和 Action。

3. **agent_scratchpad**：自动格式化之前的执行历史（Thought → Action → Observation），让 LLM 看到"之前发生了什么"。

### 3.4 max_iterations：安全阀

```python
AgentExecutor(..., max_iterations=15)
```

这意味着 Agent 最多执行 15 轮 Thought-Action-Observation 循环。如果 15 轮后还没给出 Final Answer，AgentExecutor 会强制返回。这是防止 LLM "死循环"（反复调用同一个工具得不到结果）的关键保护。

---

## 4. ReAct 循环的完整执行过程

以下以 "帮我查一下比特币价格，然后乘以2告诉我" 为例，展示 AgentExecutor 内部的执行流程：

```
══════════════════════════════════════════════════════════
Round 1
══════════════════════════════════════════════════════════

→ 当前 Prompt（发送给 LLM）:
  """
  你可以使用以下工具：
  - google_search: 搜索实时信息
  - python_runner: 执行 Python 代码

  用户问题：帮我查一下比特币价格，然后乘以2告诉我

  {agent_scratchpad}   ← 第一轮是空的
  """

← LLM 输出:
  """
  Thought: 我需要先查询比特币的当前价格，然后乘以2。
  Action: google_search
  Action Input: "比特币 当前价格 2026年5月"
  """

→ 框架解析：提取 Action=google_search, Input="比特币..."
→ 框架执行：google_search.run("比特币 当前价格 2026年5月")
→ Observation: "比特币当前价格 $87,234"

══════════════════════════════════════════════════════════
Round 2
══════════════════════════════════════════════════════════

→ 当前 Prompt:
  """
  ...（同上，工具描述）...

  {agent_scratchpad}
  Thought: 我需要先查询比特币的当前价格，然后乘以2。
  Action: google_search
  Action Input: "比特币 当前价格 2026年5月"
  Observation: 比特币当前价格 $87,234
  """

← LLM 输出:
  """
  Thought: 我有了比特币价格 $87,234。现在需要用 python_runner 计算乘以2。
  Action: python_runner
  Action Input: print(87234 * 2)
  """

→ 框架执行：python_runner.run("print(87234 * 2)")
→ Observation: "174468"

══════════════════════════════════════════════════════════
Round 3
══════════════════════════════════════════════════════════

→ 当前 Prompt:
  """
  ...（工具描述）...

  {agent_scratchpad}
  ...Round 1 and Round 2 history...
  Observation: 174468
  """

← LLM 输出:
  """
  Thought: 我得到了结果，可以输出最终答案。
  Final Answer: 比特币当前价格为 $87,234，乘以2后为 $174,468。
  """

→ 检测到 Final Answer → 退出循环
→ 返回: {'output': '比特币当前价格为 $87,234，乘以2后为 $174,468。'}
```

---

## 5. Tool 与 ReAct 的交互

### 5.1 工具如何被 LLM 发现？

```python
# react_planner.py:108-147
def handle_prompt(self, agent_model, planner_input):
    # ★ 获取所有工具并拼接描述字符串
    tools_str = ''
    for tool in self.acquire_tools(agent_model.action):
        tools_str += f"tool name: {tool.name} tool description: {tool.description}\n"

    # ★ 注入到 Prompt 的 {tools} 和 {tool_names} 占位符
    planner_input['tools'] = tools_str
    planner_input['tool_names'] = '|'.join([t.name for t in tools])
```

**工具的 `description` 决定了 LLM 会不会调用它。**如果 description 写得不清楚，LLM 可能根本不会用。

### 5.2 Tool 的 langchain_run 方法

```python
# agentuniverse/agent/action/tool/tool.py
def langchain_run(self, *args, callbacks=None, **kwargs):
    """LangChain 调用入口"""
    # ReAct 模式传入的是 JSON 格式的字符串
    parse_result = parse_and_check_json_markdown(args[0], self.input_keys)
    return self.execute(**parse_result)
```

**关键理解：** LLM 生成的 `Action Input` 是一个 JSON 字符串，框架解析后传给 `tool.execute(**params)`。

---

## 6. ReActAgentTemplate：Agent 模板层

### 6.1 继承关系

```
ComponentBase → Agent → AgentTemplate → ReActAgentTemplate
```

```python
# agentuniverse/agent/template/react_agent_template.py:37-52
class ReActAgentTemplate(AgentTemplate):
    """ReAct 模式的 Agent 模板"""

    agent_names: Optional[list[str]] = None       # 子 Agent 列表
    stop_sequence: Optional[list[str]] = None      # 自定义停止序列
    max_iterations: Optional[int] = None           # 最大迭代次数

    def input_keys(self) -> list[str]:
        return ['input']

    def output_keys(self) -> list[str]:
        return ['output']

    def parse_input(self, input_object: InputObject, agent_input: dict) -> dict:
        agent_input['input'] = input_object.get_data('input')
        tools_context = self.build_tools_context()
        ...
```

### 6.2 Agent 如何绑定到 ReAct 模板

在 YAML 中：

```yaml
# simple_qa_agent.yaml
metadata:
  type: 'AGENT'
  module: 'agentuniverse.agent.template.react_agent_template'
  class: 'ReActAgentTemplate'
```

`class: 'ReActAgentTemplate'` 意味着实例化时使用这个类。但 Planner 的绑定是隐式的——ReActAgentTemplate 内部 `execute()` 方法调用的是 `PlannerManager().get_instance_obj('react_planner')`（如果没有显式配置 Planner 则需要框架默认注入）。

---

## 7. ReAct 的局限与改进

### 7.1 局限性

| 问题 | 说明 | 缓解方式 |
|------|------|---------|
| **Tool 选择失误** | LLM 可能选错工具 | 提高 Tool description 质量 |
| **解析失败** | LLM 输出格式不符合预期 | `handle_parsing_errors=True` 自动重试 |
| **无限循环** | LLM 反复调用同一个工具 | `max_iterations=15` 强制终止 |
| **幻觉** | 错误理解工具返回结果 | 在 Prompt 中强调"基于 Observation 回答" |
| **Token 成本** | 多轮循环消耗大量 token | 合理设置 `max_tokens` 和迭代上限 |

### 7.2 与多模式 Planner 的关系

agentUniverse 不只 ReAct 一种 Planner，还有：

| Planner | 适用场景 | 与 ReAct 的区别 |
|---------|---------|---------------|
| `ReActPlanner` | 通用推理+工具调用 | Thought-Action-Observation 循环 |
| `RagPlanner` | 知识库查询 | 偏重检索，轻推理 |
| `PeerPlanner` | PEER 多 Agent | 每个步骤是一个子 Agent |
| `ExecutingPlanner` | 知识整合 | 侧重信息整理而非工具调用 |
| `ReviewingPlanner` | 质量评审 | 侧重评分和反馈生成 |

---

## 8. 动手练习

### 练习 1：追踪 ReActPlanner.invoke() 源码

打开 `agentuniverse/agent/plan/planner/react_planner/react_planner.py`，逐行阅读 `invoke()` 方法。在代码旁边标注每个步骤的作用。

### 练习 2：理解 create_react_agent 流水线

画出 `create_react_agent` 中的链式处理流程：
```
Input → RunnablePassthrough.assign(agent_scratchpad) → prompt → llm_with_stop → output_parser
```
每一步的输入和输出是什么？

### 练习 3：调试 react_agent_app

```bash
cd examples/sample_apps/react_agent_app
```

1. 启动服务
2. 问一个需要工具的问题（比如"帮我查一下最新的AI新闻"）
3. 观察控制台输出的 verbose 日志（每轮 Thought 和 Action）
4. 数一数总共执行了几轮

### 练习 4：修改 max_iterations

在 ReAct Agent 的 YAML 配置中，将 `plan.planner.max_iterations` 改为 `2`。然后问一个需要多步工具调用的问题，观察 Agent 在第 2 轮后被强制终止时的输出。

### 练习 5：手动模拟

不用代码，用纸笔模拟以下场景的 ReAct 循环：

- 用户问："巴黎的人口是多少？把这个数字的平方根算出来。"
- Agent 有 `google_search` 和 `python_runner` 两个工具

写出每一轮的：Thought → Action → Observation，直到最终输出。

---

## 9. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **ReAct** | Reasoning + Acting 循环模式 | 本文全篇 |
| **Thought** | LLM 的推理步骤 | ReAct Prompt 格式 |
| **Action** | 调用的工具名称 | — |
| **Observation** | 工具返回结果 | 由框架注入 |
| **ReActPlanner** | agentUniverse 的 ReAct 实现 | `react_planner/react_planner.py` |
| **create_react_agent()** | 构建 ReAct 执行链 | `react_planner.py:150` |
| **AgentExecutor** | LangChain 的循环控制器 | `react_planner.py:70` |
| **max_iterations** | 最大循环轮数（安全阀） | 默认 15 |
| **agent_scratchpad** | 历史步骤的格式化字符串 | `format_log_to_str()` |
| **stop_sequence** | LLM 输出停止标记 | `"\nObservation"` |
| **ReActAgentTemplate** | ReAct 模式的 Agent 模板 | `agent/template/react_agent_template.py` |

---

## 下一步

你已经理解了 ReAct 模式——Agent 如何通过"思考→行动→观察"循环自主解决问题。

下一步进入 **06-tool-system.md**，学习 Tool 系统——如何为 Agent 编写自定义工具、工具的注册流程、以及 Tool 与 ReAct 的深度交互。
