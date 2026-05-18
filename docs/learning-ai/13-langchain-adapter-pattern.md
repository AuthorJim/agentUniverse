# 13 — LangChain 适配：把它当"翻译官"而不是"主人"

> **前置要求：** 完成 [05-react-pattern.md](05-react-pattern.md) 理解 ReAct 概念
> **学习目标：** 理解 agentUniverse 为什么用 LangChain、怎么用、以及适配器模式的精髓
> **预计时间：** 1-2 天

---

## 0. 先搞清楚立场：谁是谁的谁？

很多初学者学 LangChain 时的心态是"我要学 LangChain 来搭 Agent"。但 agentUniverse 的态度是相反的：

```
❌ 错误心态：agentUniverse 是 LangChain 的"插件"
✅ 正确心态：LangChain 是 agentUniverse 的"底层实现引擎"
```

打个比方：你买了一台特斯拉（agentUniverse），但它的电机是松下生产的（LangChain）。你开车时操作的是特斯拉的方向盘和踏板，不需要知道松下的电机内部怎么转。但如果你想改装这台车，你就得理解松下电机的接口。

agentUniverse 的所有对外 API 都是自己的风格（YAML 配置、ComponentBase、Manager 单例），**LangChain 只在"运行时"被调用，而且是通过一层翻译/适配来调用的**。

---

## 1. 核心设计模式：适配器（Adapter）

### 1.1 全项目统一的方法名：`as_langchain()`

在 agentUniverse 代码库里，你会在各组件里反复看到一个方法：

```python
def as_langchain(self):
    ...
```

这不是巧合。整个框架里，凡是需要和 LangChain 交互的组件，都有 `as_langchain()` 方法。它做一件事：**把 agentUniverse 自己的对象翻译成 LangChain 认识的对象**。

| agentUniverse 组件 | as_langchain() 返回值 | LangChain 类型 |
|---|---|---|
| `Tool` | `LangchainTool(name, func, description)` | `langchain.tools.Tool` |
| `LLM` / `OpenAIStyleLLM` | `LangchainOpenAI(llm)` | `BaseLanguageModel` (ChatOpenAI 子类) |
| `ChatMemory` | `AuConversationTokenBufferMemory(...)` | `BaseChatMemory` |
| `ChatPrompt` | `ChatPromptTemplate.from_messages(...)` | `ChatPromptTemplate` |
| `Embedding` | `OpenAIEmbeddings(...)` | `LCEmbeddings` |
| `Knowledge` | `LangchainTool(name, func, desc)` | `langchain.tools.Tool` |
| `Agent` | `LangchainTool(name, func, desc)` | `langchain.tools.Tool` |

注意最后三个：Knowledge 和 Agent 都可以被"降维"成 Tool。Knowledge 变成 Tool 后可以被 ReAct Agent 调用（查知识库就是一个工具调用），Agent 变成 Tool 后可以嵌套（一个 Agent 把另一个 Agent 当工具用）。

### 1.2 类比：电源适配器

```
     agentUniverse 的世界              LangChain 的世界
    ┌─────────────────────┐          ┌─────────────────┐
    │  Tool (aU 格式)     │          │                 │
    │  ├─ name: "search"  │          │  LangchainTool  │
    │  ├─ execute()       │─as_langchain()→│  ├─ name: ...   │
    │  └─ input_keys      │          │  ├─ func: ...   │
    │                     │          │  └─ description  │
    └─────────────────────┘          └─────────────────┘
```

就像你去日本旅游，带了个 110V→220V 的电源适配器。你的 MacBook（agentUniverse）还是那个 MacBook，只是通过适配器接入了不同的"插座标准"（LangChain API）。

---

## 2. 为什么选 LangChain？它到底解决了什么问题？

### 2.1 LLM 调用不只是一个 HTTP 请求

表面上看，调 LLM 很简单：

```python
import openai
response = openai.ChatCompletion.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "你好"}]
)
```

但实际上，框架需要考虑：

- **多供应商**：OpenAI、Qwen、Claude、Gemini、DeepSeek、WenXin、Ollama...每家的 SDK 不一样
- **流式输出**：SSE 事件解析、delta 拼接、中断处理
- **Tool Calling**：Function calling 格式不统一，有的 vendor 用 JSON，有的用特殊 token
- **Token 计数**：不同模型的 tokenizer 不一样（tiktoken、sentencepiece...）
- **重试/超时**：网络波动、限流、降级
- **异步支持**：同步/异步要同时支持

如果 agentUniverse 自己全写一遍，起码要维护上万行胶水代码。

### 2.2 LangChain 提供的"统一接口"

LangChain 把这些差异封装成了统一的抽象：

```
agentUniverse 只需面对：
  BaseLanguageModel.invoke(messages)    → 同步调用
  BaseLanguageModel.ainvoke(messages)   → 异步调用
  BaseLanguageModel.bind(stop=[...])    → 配置参数
  BaseLanguageModel.stream(messages)    → 流式

不需要关心：
  - 底下是 OpenAI 还是 Qwen 还是 Claude
  - 流式怎么解析 SSE
  - Token 怎么算
```

**LangChain 就是 LLM 界的 JDBC**——你写 `Connection.execute(sql)`，不用管连的是 MySQL 还是 PostgreSQL。

---

## 3. 源码逐层解剖

### 3.1 LLM 适配：最核心的一层

入口在 `agentuniverse/llm/openai_style_llm.py:175`：

```python
# OpenAIStyleLLM 类
def as_langchain(self) -> BaseLanguageModel:
    """将 aU LLM 实例转换为 Langchain OpenAI 对象"""
    return LangchainOpenAIStyleInstance(llm=self)
```

`LangchainOpenAIStyleInstance` 继承自 LangChain 的 `ChatOpenAI`（`langchain_instance.py:22`）：

```python
class LangchainOpenAI(ChatOpenAI):
    llm: Optional[LLM] = None    # ← 持有 aU LLM 的引用

    def __init__(self, llm: LLM):
        # 把 aU LLM 的属性"翻译"成 LangChain 的初始化参数
        init_params['model_name'] = llm.model_name
        init_params['temperature'] = llm.temperature
        init_params['max_tokens'] = llm.max_tokens
        init_params['openai_api_key'] = llm.openai_api_key
        super().__init__(**init_params)   # ← 调 ChatOpenAI 的构造
        self.llm = llm                     # ← 保留引用

    def _generate(self, messages, stop, ...):
        # 实际调用时，转发给 aU 自己的 LLM
        llm_output = self.llm.call(messages=message_dicts, **params)
        ...
```

**关键设计：双重身份**

`LangchainOpenAI` 既是 LangChain 体系里的 `ChatOpenAI`（可以被 LangChain 的 AgentExecutor、Chain 等直接使用），内部又委托给 agentUniverse 自己的 `LLM.call()`。这就像一个会双语的翻译官——对 LangChain 说 LangChain 的语言，对内执行 aU 的逻辑。

### 3.2 Tool 适配：函数签名翻译

`agentuniverse/agent/action/tool/tool.py:137`：

```python
def as_langchain(self) -> LangchainTool:
    return LangchainTool(
        name=self.name,
        func=self.langchain_run,        # ← 注意：不是 self.execute
        description=self.description
    )
```

为什么用 `langchain_run` 而不是 `execute`？

因为 LangChain 调用 Tool 时传的参数格式不同。LangChain 会把 LLM 的 `Action Input` 作为第一个字符串参数传入，而 aU 自己的 `execute()` 用的是关键字参数。`langchain_run` 做了参数格式转换：

```python
def langchain_run(self, *args, **kwargs):
    # LangChain 传过来的可能是JSON字符串，需要解析
    parse_result = parse_and_check_json_markdown(args[0], self.input_keys)
    return self.execute(**parse_result)
```

这层翻译很简单，但很关键——**两边约定的接口不同，`langchain_run` 就是那个"翻译层"**。

### 3.3 Memory 适配：继承 + 扩展

`agentuniverse/agent/memory/chat_memory.py:40`：

```python
def as_langchain(self) -> BaseChatMemory:
    if self.type == MemoryTypeEnum.SHORT_TERM:
        return AuConversationTokenBufferMemory(
            llm=self.llm.as_langchain(),
            max_token_limit=self.max_tokens,
            messages=self.messages
        )
    elif self.type == MemoryTypeEnum.LONG_TERM:
        return AuConversationSummaryBufferMemory(...)
```

`AuConversationSummaryBufferMemory` 和 `AuConversationTokenBufferMemory` 在 `memory/langchain_instance.py` 中定义，它们**直接继承** LangChain 的 `ConversationSummaryBufferMemory` 和 `ConversationTokenBufferMemory`，然后：

1. 重写 `build_memory()` — 从 aU 的 `Message` 列表初始化记忆
2. 重写 `predict_new_summary()` — 用 aU 的 Prompt 管理来做摘要压缩
3. 扩展 `load_memory_str` — 支持字符串格式输出

这是适配器模式的另一种变体：**直接继承 + 局部改造**，而不是从头包装。

### 3.4 Prompt 适配：模板语法翻译

`agentuniverse/prompt/chat_prompt.py:33`：

```python
def as_langchain(self) -> ChatPromptTemplate:
    return ChatPromptTemplate.from_messages(
        Message.as_langchain_list(self.messages)
    )
```

aU 的 `Message` 是自定义类（有 type、content 等），先转成 LangChain 的 `BaseMessage` 列表，再传进 `ChatPromptTemplate.from_messages()`。`{variable}` 占位符语法两者一致，所以这步主要是对象格式转换。

### 3.5 拼在一起：ReAct Agent 的完整翻译链

现在回看 `react_agent_template.py:62-78` 的 `customized_execute`，你就清楚了：

```python
def customized_execute(self, ...):
    # 第 1 步：把所有 aU 组件翻译成 LangChain 版
    lc_tools = [t.as_langchain() for t in tools]           # Tool 翻译
    lc_llm = llm.as_langchain()                             # LLM 翻译
    lc_prompt = prompt.as_langchain()                       # Prompt 翻译
    lc_memory = memory.as_langchain() if memory else None   # Memory 翻译

    # 第 2 步：用翻译后的组件，调 LangChain 的原生 ReAct
    agent = self.create_react_agent(lc_llm, lc_tools, lc_prompt)
    agent_executor = AgentExecutor(agent=agent, tools=lc_tools, ...)
    res = agent_executor.invoke(input=agent_input, ...)
```

翻译完成后，后续全是 LangChain 原生的 `AgentExecutor` 在跑。agentUniverse 只负责"翻译"和"传结果"。

---

## 4. LCEL 管道语法速成

agentUniverse 的 ReAct 模板中用到了 LangChain 的 LCEL（LangChain Expression Language）。核心就两点：`|` 管道和 `RunnablePassthrough`。

### 4.1 `|` 管道的本质

LangChain 重载了 Python 的 `|` 运算符，把它变成了数据管道：

```python
result = step_a | step_b | step_c
# 等价于：
# data = step_a.invoke(input)
# data = step_b.invoke(data)
# data = step_c.invoke(data)
```

管道中的数据是一个 `dict`，每个步骤可以读取、修改、新增字段。

### 4.2 `RunnablePassthrough.assign()` — 不改变原数据，只追加字段

```python
RunnablePassthrough.assign(
    agent_scratchpad=lambda x: format_log_to_str(x["intermediate_steps"]),
)
```

`RunnablePassthrough` 字面意思是"可运行的直通器"——它什么都不做，只是把数据原样传给下一步。`.assign()` 的意思是"在传递前，往 dict 里追加一个新字段"。就像快递员在包裹上多贴了一张便利贴。

### 4.3 `llm.bind(stop=[...])` — 给 LLM 加停止条件

```python
llm_with_stop = llm.bind(stop=["\nObservation"])
```

相当于对 LLM 说："你可以一直生成文字，但一旦出现 `\nObservation` 就立刻闭嘴。"为什么？因为 ReAct 中 `Observation` 是工具调用的结果，应该由系统注入，不能让 LLM 自己编。这是一个**防幻觉机制**。

### 4.4 完整管道再看一遍

```python
agent = (
    RunnablePassthrough.assign(                              # ①
        agent_scratchpad=lambda x: format_log_to_str(        #   往数据包加"历史日志"字段
            x["intermediate_steps"]
        ),
    )
    | prompt              # ② 填入 Prompt 模板
    | llm_with_stop       # ③ 调 LLM（遇 Observation 就停）
    | output_parser       # ④ 解析 Thought/Action
)
```

数据流向：

```
{input, tools, intermediate_steps}
  → ① 追加 agent_scratchpad 字段
  → {input, tools, intermediate_steps, agent_scratchpad}
  → ② 填入模板 → ChatPromptValue
  → ③ LLM 生成 → AIMessage
  → ④ 解析 → {thought, action, action_input}
```

### 4.5 AgentExecutor 的完整循环

`AgentExecutor` 调用 `agent`（上面那个管道）一次得到 Action，执行工具拿到 Observation，把 Observation 追加到 `intermediate_steps`，再调 `agent`...如此循环，直到 LLM 输出 `Final Answer` 或达到 `max_iterations`。

---

## 5. 一个完整的翻译调用链（图解）

```
用户输入
  │
  ▼
┌─────────────────────────────────────────────────────┐
│  agentUniverse 的世界                                 │
│                                                       │
│  ReactAgentTemplate.customized_execute()              │
│    │                                                  │
│    ├─→ ToolManager().get_instance_obj(name)           │
│    │     └─→ tool.as_langchain()     ═══════════╗    │
│    │                                              ║    │
│    ├─→ llm.as_langchain()             ═══════════╣    │
│    │                                              ║    │
│    ├─→ prompt.as_langchain()          ═══════════╣    │
│    │                                              ║    │
│    └─→ memory.as_langchain()          ═══════════╣    │
│                                                     ║    │
└─────────────────────────────────────────────────────┘  ║
                                                         ║
              ╔═══════════════════════════════════════════╝
              ║  (翻译完成，进入 LangChain 的世界)
              ║
┌─────────────╨──────────────────────────────────────────┐
│  LangChain 的世界                                       │
│                                                         │
│  AgentExecutor.invoke()                                 │
│    │                                                    │
│    ├─→ agent.invoke(input)  ← LCEL 管道                 │
│    │     ├─ RunnablePassthrough.assign(...)              │
│    │     ├─ prompt (ChatPromptTemplate)                  │
│    │     ├─ llm_with_stop (ChatOpenAI.bind(stop=...))    │
│    │     └─ output_parser (ReActSingleInputOutputParser) │
│    │                                                    │
│    ├─→ tool.run(action_input)  ← 调用翻译后的工具        │
│    │     └─ 内部执行 agentUniverse 的 Tool.execute()     │
│    │                                                    │
│    └─→ 循环直到 Final Answer 或达到 max_iterations      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

关键洞察：**`as_langchain()` 不是在"创建新对象"，而是在给原来的 aU 对象穿上一件 LangChain 能识别的外套。** Tool 还是那个 Tool，LLM 还是那个 LLM，只是外面套了一层 LangChain 接口。

---

## 6. Node.js 视角：这跟 Express 中间件是一个道理

你在 Node.js 里用过 Express 中间件：

```js
app.use(cors());       // cors 中间件
app.use(bodyParser()); // body 解析中间件
app.get('/api', handler);  // 业务处理
```

每个中间件只知道 `(req, res, next)` 这一套接口，不管前一个和后一个中间件内部干了什么。LangChain 的组件抽象就是同样的思路：

- Express 有 `(req, res, next)`
- LangChain 有 `.invoke(input) → output`
- agentUniverse 的 `as_langchain()` 就是把你自己的业务代码包装成符合这个签名的函数

---

## 7. 总结：为什么这种设计是务实的

agentUniverse **没有**自己从头写 ReAct 循环、LLM 调用的 SSE 解析、Token 计数、Tool Calling 格式处理...

但它也**没有**把整个框架绑死在 LangChain 上。证据：

1. 所有 LangChain 依赖都通过 `as_langchain()` 隔离 —— 理论上可以换 `as_llamaindex()` 或 `as_crewai()`
2. 框架定义层（YAML、ComponentBase、Manager）完全不依赖 LangChain
3. LLM 的 `call()` 和 `acall()` 是 agentUniverse 自己的接口，LangChain 只是其中一个"执行通道"

这就像一个优秀的架构师会说："我不信任任何第三方框架能活 10 年，所以我的核心设计不依赖于其中任何一个。但我也不会蠢到自己去实现底层协议。"

---

## 8. 延伸阅读

- [LangChain Expression Language (LCEL) 官方文档](https://python.langchain.com/docs/concepts/lcel/)
- 本项目 05-react-pattern.md — 了解 ReAct 循环的业务逻辑
- `agentuniverse/llm/langchain_instance.py` — LLM 适配器的完整实现
- `agentuniverse/agent/memory/langchain_instance.py` — Memory 适配器（继承 + 重写）
- `agentuniverse/agent/template/react_agent_template.py:101-136` — `create_react_agent` 管道组装
