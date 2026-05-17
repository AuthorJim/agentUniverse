# 02 — Agent 基础概念：什么是 AI Agent？

> **前置要求：** 完成 [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md)
> **学习目标：** 理解 AI Agent 的核心概念、agentUniverse 的六维 Agent 模型、一次请求的完整生命周期
> **预计时间：** 2-3 天

---

## 1. 什么是 Agent？从一个最简单的类比对全栈工程师

### 1.1 你还记得"函数"吗？

你写过的代码里有无数个函数：

```python
def add(a, b):
    return a + b
```

- **输入**：a, b
- **逻辑**：加法
- **输出**：a + b

函数是**确定性的**——同样的输入永远得到同样的输出。

### 1.2 那什么是 Agent？

**Agent 就是一个函数，但它的"逻辑"不是写死的代码，而是一个 LLM（大语言模型）。**

```
传统函数：  输入 → [硬编码逻辑] → 输出
Agent：     输入 → [LLM 思考 + 调用工具] → 输出
```

和函数最大的不同：Agent 的输出**不是确定性的**。同样的问题问两次，可能得到不同的回答。

### 1.3 用 Node.js 后端架构类比

你熟悉的典型后端请求流程：

```
Client Request → Router → Controller → Service → Repository → Database
                                         ↓
                                    Response ← Client
```

一个 AI Agent 的请求流程：

```
User Input → Agent.run()
              ├── 1. 加载 Memory（之前的对话上下文）
              ├── 2. 组装 Prompt（系统指令 + 对话历史 + 用户问题）
              ├── 3. LLM 推理（大脑思考）
              ├── 4. 如果需要，调用 Tool（比如搜索引擎）
              ├── 5. 如果需要，查询 Knowledge（RAG 检索）
              ├── 6. LLM 再次推理（基于工具结果）
              └── 7. 返回最终结果
```

**对比理解：**
- Agent 的 `run()` 方法 ≈ Controller + Service 的合体
- LLM ≈ 一个不可预测但极度聪明的决策引擎
- Tool ≈ 外部微服务调用（类似 HTTP 请求第三方 API）
- Knowledge ≈ 内部搜索引擎（类似 Elasticsearch）
- Memory ≈ Session 存储（类似 Redis）

---

## 2. Agent 的核心循环：Observe → Think → Act

所有 AI Agent 本质上都遵循同一个循环模式：

```
┌─────────────────────────────────────────┐
│                                         │
│   ① Observe（观察）                      │
│   - 读取用户输入                         │
│   - 回顾对话历史（Memory）               │
│   - 获取工具返回的结果                   │
│         ↓                               │
│   ② Think（思考）                        │
│   - LLM 分析当前状态                     │
│   - 决定下一步做什么                     │
│         ↓                               │
│   ③ Act（行动）                          │
│   - 调用工具（搜索、计算、查询数据库）    │
│   - 或者给出最终回答                     │
│         ↓                               │
│   判断：任务完成了吗？                   │
│   - 没完成 → 回到 ①                      │
│   - 完成了 → 输出结果                    │
│                                         │
└─────────────────────────────────────────┘
```

这就是 **ReAct（Reasoning + Acting）** 模式的本质——也是 agentUniverse 最核心的 Agent 运行范式。

### 2.1 用一个实际例子走一遍

假设你问 Agent："今天北京天气怎么样？"

```
第 1 轮：
  ① Observe：用户问"今天北京天气怎么样"
  ② Think：我需要查询天气数据。我有个 google_search 工具可以用。
  ③ Act：调用 google_search("北京 今天 天气 2026年")
  → 判断：还没完成，需要看搜索结果

第 2 轮：
  ① Observe：搜索结果返回"北京今日晴，15°C~25°C，北风3级"
  ② Think：我已经有了天气数据，可以整理成自然语言回复。
  ③ Act：输出"北京今天晴天，气温15到25摄氏度，北风3级..."
  → 判断：完成了！
```

这个多轮循环过程在 agentUniverse 中由 **Planner（规划器）** 控制。

---

## 3. agentUniverse 的 Agent 六维模型

在 agentUniverse 框架中，一个 Agent 由**六个维度**组成。这就是你看到的每个 Agent YAML 配置文件的骨架：

```
Agent = Profile + Plan + Action + Memory + Prompt + WorkPattern
```

### 3.1 六维总览

| 维度 | 一句话解释 | 技术类比 | YAML 配置位置 |
|------|----------|---------|-------------|
| **Profile** | 绑定哪个 LLM，设什么参数 | 选择用什么数据库引擎 | `profile.llm_model` |
| **Plan** | 用什么策略思考（ReAct？顺序执行？） | Controller 的路由逻辑 | `plan.planner` |
| **Action** | 能用什么工具、查询什么知识库 | 微服务调用的 client list | `action.tool` + `action.knowledge` |
| **Memory** | 记住对话历史 | Session / Redis | `memory.name` |
| **Prompt** | 系统指令模板（"你是谁"、"怎么做"） | HTML template | `profile.prompt_version` |
| **WorkPattern** | 多 Agent 协作模式（单体 Agent 可忽略） | 工作流引擎 | `work_pattern` |

### 3.2 以 simple_qa_agent 为例

```yaml
# examples/sample_apps/simple_qa_agent_app/intelligence/agentic/agent/
# agent_instance/simple_qa_agent.yaml

info:
  name: 'simple_qa_agent'                    # Agent 的名字

profile:                                     # ① Profile：绑定 LLM
  prompt_version: 'simple_qa_prompt.v1'      # 引用 Prompt 模板
  introduction: '你是一个乐于助人的问答助手'
  target: '以友好的方式回答用户问题'
  instruction: '清晰简洁、准确无误、友好亲切...'
  llm_model:                                 # LLM 配置
    name: 'qwen_llm'                         # 引用已注册的 LLM 实例
    model_name: 'qwen2.5-72b-instruct'       # 具体模型名
    temperature: 0.7                         # 随机程度（0=确定，1=有创造性）
    max_tokens: 1000                         # 最大输出长度

action:                                      # ③ Action：工具 + 知识库
  tool: []                                   # 没有工具（纯对话 Agent）
  knowledge: []                              # 没有知识库

# memory:                                    # ④ Memory：注释掉了，没有记忆
#   name: 'demo_memory'

metadata:                                    # 元数据（框架用于注册组件）
  type: 'AGENT'
  module: 'agentuniverse.agent.template.react_agent_template'
  class: 'ReActAgentTemplate'                # ② Plan：使用 ReAct 模板（内建 Planner）
```

**看完这个配置，你能得出什么结论？**

这是一个"最简化"的 Agent：
- 有 LLM 大脑（qwen）
- 有系统指令（prompt）
- 没有工具、没有知识库、没有记忆
- 用 ReAct 模式思考
- 就是一个纯粹的"聊天机器人"

---

## 4. 源码跟踪：agent.run() 到底做了什么

让我们跟踪 agentUniverse 中 `agent.run()` 的完整源码链路，看看六个维度是如何协同工作的。

### 4.1 入口：Agent.run()

```python
# agentuniverse/agent/agent.py:110-129
@trace_agent
def run(self, **kwargs) -> OutputObject:
    """Agent 实例运行入口"""

    # 步骤 1：检查输入（必须包含 input_keys 定义的字段）
    self.input_check(kwargs)

    # 步骤 2：包装用户输入为 InputObject
    input_object = InputObject(kwargs)

    # 步骤 3：预处理输入（添加 chat_history, date, session_id 等）
    agent_input = self.pre_parse_input(input_object)

    # 步骤 4：执行核心逻辑（交给 Planner）
    planner_result = self.execute(input_object, agent_input)

    # 步骤 5：解析结果
    agent_result = self.parse_result(planner_result)

    # 步骤 6：检查输出
    self.output_check(agent_result)

    # 步骤 7：包装并返回
    output_object = OutputObject(agent_result)
    return output_object
```

**用后端术语理解：**
- `input_check` = 请求参数校验（类似 express-validator）
- `pre_parse_input` = 给 request 对象附加上下文（类似 middleware）
- `execute` = 核心业务逻辑（类似 service 层）
- `parse_result` = 格式化输出（类似 response serializer）
- `output_check` = 返回结果校验

### 4.2 核心：Agent.execute() → Planner.invoke()

```python
# agentuniverse/agent/agent.py:147-159
def execute(self, input_object: InputObject, agent_input: dict) -> dict:
    """执行 Agent 实例"""

    # 1. 获取 Planner 实例（比如 ReActPlanner）
    planner_base: Planner = PlannerManager().get_instance_obj(
        self.agent_model.plan.get('planner').get('name')
    )

    # 2. 调用 Planner 的 invoke 方法
    planner_result = planner_base.invoke(
        self.agent_model,   # Agent 的完整配置
        agent_input,        # 预处理后的用户输入
        input_object        # 原始输入
    )
    return planner_result
```

**Planner 是真正的"大脑调度器"。** Agent 只负责组装和转发，Planner 负责决定"怎么思考"。

### 4.3 ReActPlanner.invoke()：ReAct 循环的实现

```python
# agentuniverse/agent/plan/planner/react_planner/react_planner.py:46-77
def invoke(self, agent_model, planner_input, input_object) -> dict:
    """ReAct 模式的核心执行"""

    # 1. 处理 Memory（加载历史对话）
    memory = self.handle_memory(agent_model, planner_input)

    # 2. 获取 LLM 实例
    llm = self.handle_llm(agent_model)

    # 3. 获取已注册的 Tool（搜索、代码执行等）
    tools = self.acquire_tools(agent_model.action)

    # 4. 组装 Prompt（系统指令 + 工具描述 + 用户输入）
    prompt = self.handle_prompt(agent_model, planner_input)

    # 5. 创建 ReAct Agent（LangChain 的 AgentExecutor）
    agent = create_react_agent(llm.as_langchain(), tools, prompt.as_langchain())
    agent_executor = AgentExecutor(
        agent=agent,
        tools=tools,
        max_iterations=15,    # 最多循环 15 轮
        verbose=True
    )

    # 6. 执行！（这里就是 Observe→Think→Act 循环）
    return agent_executor.invoke(input=planner_input, ...)
```

**关键参数 `max_iterations=15`：** 如果 Agent 在 15 轮循环内还没给出最终答案，就会强制停止。这是一个**安全阀**，防止无限循环。

### 4.4 调用关系图

```
用户调用
  │
  ▼
Agent.run()                       ← 入口（6 步校验-执行-解析流水线）
  │
  ├─ pre_parse_input()            ← 附加 chat_history, date, session_id
  │
  ├─ execute()                    ← 委托给 Planner
  │   │
  │   └─ Planner.invoke()         ← ReActPlanner / RagPlanner / PeerPlanner...
  │       │
  │       ├─ handle_memory()      ← 从 Memory 加载历史对话
  │       ├─ handle_llm()         ← 获取 LLM 实例
  │       ├─ handle_prompt()      ← 组装 Prompt（系统指令 + 历史 + 用户输入）
  │       ├─ acquire_tools()      ← 获取可用的工具列表
  │       └─ AgentExecutor.invoke() ← 执行 Observe→Think→Act 循环
  │           │
  │           └─ LLM 调用        ← 真正的 AI 推理（HTTP API 调用）
  │
  ├─ parse_result()               ← 提取并格式化输出
  │
  └─ OutputObject                 ← 包装返回结果
```

---

## 5. 关键概念深入

### 5.1 LLM（大语言模型）— Agent 的"大脑"

LLM 就是 Agent 的推理引擎。agentUniverse 支持的所有 LLM 供应商：

```
agentuniverse/llm/default/
├── qwen_openai_style_llm.py     # 通义千问（Qwen）
├── deepseek_openai_style_llm.py # DeepSeek
├── openai_openai_style_llm.py   # OpenAI GPT 系列
├── claude_openai_style_llm.py   # Anthropic Claude
├── gemini_openai_style_llm.py   # Google Gemini
├── kimi_openai_style_llm.py     # Moonshot Kimi
├── ollama_llm.py                # 本地 Ollama 模型
├── vllm_llm.py                  # vLLM 推理框架
└── ...
```

配置 LLM 只需要一个 YAML 文件：

```yaml
# 来自 simple_qa_agent_app/intelligence/agentic/llm/qwen_llm.yaml
name: 'qwen_llm'
model_name: 'qwen2.5-72b-instruct'
api_key: '${DASHSCOPE_API_KEY}'    # 引用 custom_key.toml 中的变量
temperature: 0.7
max_tokens: 1000
metadata:
  type: 'LLM'
  module: 'agentuniverse.llm.default.qwen_openai_style_llm'
  class: 'QWenOpenAIStyleLLM'
```

**temperature 参数的含义：**
- `0.0 ~ 0.3`：严谨、确定性强（适合数学、代码、事实查询）
- `0.4 ~ 0.7`：平衡（适合一般对话）
- `0.8 ~ 1.0`：创造性高（适合写诗、脑暴）
- `> 1.0`：几乎随机（极少使用）

### 5.2 Prompt（提示词）— 定义 Agent 的"人设"

Prompt 告诉 LLM "你是谁"、"你要做什么"、"你应该怎么做"。

```yaml
# 来自 simple_qa_agent_app/intelligence/agentic/prompt/simple_qa_prompt.yaml
name: 'simple_qa_prompt'
introduction: |
  你是一个乐于助人且友好的问答助手。
target: |
  以友好和对话的方式回答用户的问题。
instruction: |
  1. 清晰简洁 - 直接回答问题
  2. 准确无误 - 如果不确定，请明说
  3. 友好亲切 - 使用温暖的语气
  4. 语言适应 - 用与问题相同的语言回答
  5. 乐于助人 - 提供额外背景信息
metadata:
  type: 'PROMPT'
  version: 'simple_qa_prompt.v1'    # 支持版本化管理
```

**Prompt 的三大组成部分：**

| 部分 | 含义 | 影响 |
|------|------|------|
| `introduction` | 角色设定（"你是谁"） | 决定 Agent 的行为风格 |
| `target` | 目标（"你要完成什么"） | 决定输出的方向 |
| `instruction` | 约束规则（"你怎么做"） | 决定输出的质量和格式 |

**提示词版本化：** agentUniverse 支持 `prompt_version` 字段，让你可以像管代码一样管 Prompt——A/B 测试不同版本的 prompt，出问题可以快速回滚。

### 5.3 Tool（工具）— 给 Agent 装上"手脚"

纯 LLM 只能对话，不能"做事"。Tool 让 Agent 可以和外部世界交互。

```python
# Tool 的本质是一个类，继承 Tool 基类
# 来自 agentuniverse/agent/action/tool/tool.py:48
class Tool(ComponentBase):
    """工具基类"""

    name: str
    description: str    # ★ 这个 description 至关重要

    def run(self, **kwargs):
        """执行工具逻辑（子类必须实现）"""
        pass
```

**关键理解：`description` 字段是给 LLM 看的**

LLM 通过阅读 Tool 的 `description` 来决定"什么时候该调用这个工具"。所以 description 写得越清晰，LLM 调用工具的准确率越高。

比如一个搜索工具的 description：
```yaml
name: 'google_search'
description: |
  当你需要查找实时信息、新闻或最新数据时使用此工具。
  输入：搜索关键词字符串
  输出：相关的搜索结果摘要
```

### 5.4 Memory（记忆）— Agent 的"上下文"

没有 Memory，每个问题都是独立的新对话——Agent 是"金鱼记忆"。有了 Memory，Agent 能记住之前聊过什么。

agentUniverse 的 Memory 系统有三层：

```
Memory（记忆管理器）
  ├── MemoryStorage（存储后端） — 存哪里？（RAM / ChromaDB / SQLite）
  ├── MemoryCompressor（压缩器） — 太长了怎么办？（摘要压缩）
  └── Message（消息对象） — 一条条对话记录
```

**核心参数：**

```python
# 来自 agentuniverse/agent/memory/memory.py:40-49
class Memory(ComponentBase):
    memory_key: str = 'chat_history'   # 在 Prompt 中的变量名
    max_tokens: int = 2000             # 最多保留多少 token 的历史
    memory_storages: list = ['ram_memory_storage']  # 存储后端列表
    memory_compressor: str = None      # 超长时的压缩策略
```

`max_tokens=2000` 意味着当历史对话超过 2000 个 token 时，最早的对话会被**压缩或丢弃**——这叫做"记忆裁剪（pruning）"。

**Token 是什么？**
Token 是 LLM 处理文本的最小单位。一个中文字约 1-2 个 token，一个英文词约 1-3 个 token。1000 token ≈ 750 个英文单词 ≈ 400-500 个中文字。

### 5.5 Knowledge（知识库/RAG）— Agent 的"专业资料"

LLM 的知识截止于训练数据日期。Knowledge 让 Agent 能查询你的私有文档。

```
知识注入流程（RAG = Retrieval-Augmented Generation）：

离线阶段（构建知识库）：
  PDF/TXT/JSON → Reader(读取) → DocProcessor(切分) → Embedding(向量化) → Store(存储)

在线阶段（查询时）：
  用户问题 → QueryParaphraser(改写) → RagRouter(路由) → Store(向量检索) → 注入 Prompt
```

**为什么需要 RAG？** 因为 LLM 的知识是"冻结的"——GPT-4 不知道 2024 年之后的事，也不知道你公司的内部文档。RAG 把相关知识"塞进" Prompt，让 LLM 能回答它本不知道的问题。

---

## 6. 一次完整请求的详细生命周期

以 `simple_qa_agent` 为例，用户输入 "法国的首都是什么？"：

```
时间线：
────────────────────────────────────────────────────────────────────

T0: 用户调用
    agent.run(input='法国的首都是什么？')

T1: Agent.run() 入口
    ├─ input_check()：验证 input 字段存在 ✅
    ├─ InputObject({'input': '法国的首都是什么？'})
    └─ pre_parse_input()：补充字段
        agent_input = {
            'input': '法国的首都是什么？',
            'chat_history': '',         # 无记忆，空
            'background': '',           # 无背景知识，空
            'date': '2026-05-17',
            'session_id': '',
            'agent_id': 'simple_qa_agent',
        }

T2: Agent.execute() → ReActPlanner.invoke()
    ├─ handle_memory()：无 memory 配置 → 返回 None
    ├─ handle_llm()：获取 qwen_llm 实例
    ├─ acquire_tools()：tool 列表为空 → 返回 []
    └─ handle_prompt()：组装 Prompt

T3: Prompt 组装结果（简化版）
    """你是一个乐于助人且友好的问答助手。
    你的目标：以友好和对话的方式回答用户的问题。
    规则：
    1. 清晰简洁
    2. 准确无误
    3. 友好亲切
    ...

    用户问题：法国的首都是什么？
    """

T4: AgentExecutor.invoke() — ReAct 循环开始

    Round 1:
      Thought: 这是一个简单的事实性问题，我不需要工具就能回答。
      Action: 直接回答
      Observation: 巴黎是法国的首都。

T5: 循环结束 → 返回结果
    {'output': '法国的首都是巴黎。它以埃菲尔铁塔、卢浮宫...'}

T6: Agent.parse_result()
    agent_result = {'output': '法国的首都是巴黎...'}

T7: OutputObject 包装返回
    OutputObject({'output': '法国的首都是巴黎...'})
```

**对于没有工具和知识库的简单 Agent，ReAct 循环实际上就只有一轮。**

---

## 7. 有工具的 Agent：ReAct 多轮循环实战

如果我们给 Agent 加上一个 `google_search` 工具，问"今天北京天气怎么样？"：

```
Round 1:
  Thought: 我需要查询今天北京的天气，这是实时信息，我应该用 google_search。
  Action: google_search("北京 今天 天气 2026-05-17")
  Observation: [搜索结果] 北京今日晴，15°C~25°C

Round 2:
  Thought: 我有了天气数据，可以整理回复了。
  Action: 直接输出
  Final Answer: "北京今天晴天，气温15到25摄氏度..."
```

**这就是 ReAct 的核心价值：** LLM 自己决定要不要用工具、用什么工具、怎么用、什么时候该结束。

---

## 8. 动手练习

### 练习 1：画出你理解的 Agent 架构图

不看文档，凭记忆画出 Agent 的六维模型，并标注每个维度的作用。

### 练习 2：阅读源码中的 run() 方法

打开 `agentuniverse/agent/agent.py`，找到 `run()` 方法（第 110 行），逐行标注每个步骤的作用。

### 练习 3：运行并观察 simple_qa_agent

```bash
cd examples/sample_apps/simple_qa_agent_app

# 1. 检查配置
cat intelligence/agentic/agent/agent_instance/simple_qa_agent.yaml

# 2. 如果没有 API Key，先配置
# 编辑 config/custom_key.toml，设置 DASHSCOPE_API_KEY = "你的key"

# 3. 启动服务
python bootstrap/intelligence/server_application.py

# 4. 向 Agent 提问（通过 curl 或浏览器访问 http://localhost:8888）
```

### 练习 4：修改配置观察变化

1. 把 `temperature` 从 `0.7` 改为 `0.1`，再次提问，观察回复的变化
2. 修改 `instruction`，给 Agent 加一条限制："不要回答任何关于政治的问题"
3. 试试把 `max_tokens` 调到 `50`，看 Agent 的回答会被截断成什么样

### 练习 5：追踪 ReAct 思维过程

在你自己的话语中，描述下面这个场景中 Agent 的 ReAct 思考过程：

> 用户问："帮我算一下 123 * 456，然后用中文告诉我结果"
> Agent 有一个 `python_runner` 工具（可以执行 Python 代码）

写出每一轮 Round 的 Thought、Action、Observation。

---

## 9. 概念速查表

| 概念 | 一句话 | 在本项目的具体体现 |
|------|--------|------------------|
| **Agent** | LLM + 工具 + 记忆 + 提示词的封装体 | `agentuniverse/agent/agent.py` |
| **LLM** | 大语言模型，Agent 的大脑 | `agentuniverse/llm/` |
| **ReAct** | 推理-行动循环模式 | `react_planner/react_planner.py` |
| **Planner** | 控制 Agent 的思考策略 | `agent/plan/planner/` |
| **Prompt** | Agent 的系统指令和人设 | `prompt/prompt.py` |
| **Tool** | Agent 可调用的外部功能 | `agent/action/tool/tool.py` |
| **Knowledge** | RAG，私有知识注入 | `agent/action/knowledge/knowledge.py` |
| **Memory** | 对话历史和上下文保持 | `agent/memory/memory.py` |
| **Token** | LLM 处理文本的最小计量单位 | `max_tokens` 参数 |
| **Temperature** | 控制 LLM 输出的随机性 | `profile.llm_model.temperature` |
| **ComponentBase** | 所有组件的统一基类 | `base/component/component_base.py` |
| **AgentManager** | 全局 Agent 注册表（单例） | `agent/agent_manager.py` |
| **YAML Config** | 声明式组件配置 | 所有 `.yaml` 文件 |

---

## 下一步

你已经理解了：
- Agent 的本质（LLM 驱动的非确定性函数）
- agentUniverse 的六维模型（Profile + Plan + Action + Memory + Prompt + WorkPattern）
- agent.run() 的完整源码链路
- ReAct 循环的工作原理

下一步进入 **03-framework-startup.md**，学习 agentUniverse 框架是如何启动的——它如何扫描 YAML、注册组件、以及 AgentManager 等单例管理器的工作原理。
