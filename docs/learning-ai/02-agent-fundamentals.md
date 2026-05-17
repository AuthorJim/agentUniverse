# 02 — Agent 基础概念：当一个函数学会了思考

> **前置要求：** 完成 [01-python-for-nodejs-developers.md](01-python-for-nodejs-developers.md)
> **学习目标：** 理解 AI Agent 的本质、agentUniverse 的六维 Agent 模型、一次请求从入口到输出的完整链路
> **预计时间：** 2-3 天

---

## 1. 从你最熟悉的东西开始：函数

### 1.1 一个普通函数

你写过无数这样的代码：

```python
def add(a: int, b: int) -> int:
    return a + b
```

- **输入**：a, b
- **逻辑**：加法（硬编码，永不改变）
- **输出**：a + b

函数的本质是**确定性的**——同样的输入，永远得到同样的输出。你完全掌控它的行为。

### 1.2 那什么是 Agent？

**Agent 就是一个函数，但它的"逻辑"不是写死的代码，而是一个 LLM（大语言模型）。**

```
传统函数:   输入 → [硬编码逻辑] → 输出（确定性的）
Agent:      输入 → [LLM 思考 + 调用工具 + 检索知识] → 输出（非确定性的）
```

同样的输入问两次，Agent 可能给出不同的回答。它可以：
- 自己决定"我回答不了，先搜一下"
- 自己判断"搜到的信息不够，换个关键词再搜"
- 自己意识到"之前的推理有误，重新分析"

**这就是 Agent 和函数最根本的区别：Agent 有自主决策能力，函数没有。**

### 1.3 用你熟悉的 Node.js 后端架构类比

你每天都在和这套架构打交道：

```
Client Request → Router → Controller → Service → Repository → Database
                                         ↓
                                    Response ← Client
```

一个 AI Agent 的请求流程高度相似，只是"Service 层"变成了 LLM：

```
User Input → Agent.run()
              ├── 1. 加载 Memory（≈ Redis 中读 Session）
              ├── 2. 组装 Prompt（≈ 渲染 HTML 模板）
              ├── 3. LLM 推理（≈ 调用决策引擎）
              ├── 4. 如需 Tool → 调用工具（≈ 调用外部微服务 API）
              ├── 5. 如需 Knowledge → RAG 检索（≈ Elasticsearch 查询）
              ├── 6. LLM 再次推理（基于工具/检索结果）
              └── 7. 返回最终结果
```

| Agent 组件 | 后端映射 | 说明 |
|-----------|---------|------|
| Agent.run() | Controller + Service | 请求入口，协调各组件 |
| LLM | 智能决策引擎 | "大脑"，但不可预测 |
| Tool | 外部微服务 API | 让 Agent 能"做事"（搜网络、执行代码、查数据库） |
| Knowledge (RAG) | Elasticsearch 全文检索 | 私有文档的语义搜索 |
| Memory | Redis Session | 跨轮对话的上下文保持 |
| Prompt | 配置模板 | 定义 Agent 的"人设"和任务 |

---

## 2. Agent 的核心循环：Observe → Think → Act

所有 AI Agent 都遵循同一个循环（这被称为 **ReAct 模式**——第 05 章会深入）：

```
┌──────────────────────────────────────────────┐
│                                              │
│   ① Observe（观察）                           │
│   - 读取用户输入                              │
│   - 回顾对话历史（Memory）                    │
│   - 获取工具返回的结果                        │
│          ↓                                   │
│   ② Think（思考）                             │
│   - LLM 分析当前状态                          │
│   - 决定下一步: 用工具？还是直接回答？          │
│          ↓                                   │
│   ③ Act（行动）                               │
│   - 调用工具（搜索、计算、查数据库）           │
│   - 或者输出最终答案                          │
│          ↓                                   │
│   判断: 任务完成了吗？                        │
│   - 没完成 → 回到 ①                          │
│   - 完成了 → 输出结果                        │
│                                              │
└──────────────────────────────────────────────┘
```

### 2.1 用一个实际例子走一遍

假设用户问 Agent："今天北京天气怎么样？"（Agent 有一个 `google_search` 工具）

```
═══════════════════════════════════════════════
Round 1
═══════════════════════════════════════════════
① Observe: 用户问"今天北京天气怎么样"（没有历史对话）
② Think:   我需要实时天气数据，得用 google_search 工具
③ Act:     调用 google_search("北京 天气 2026-05-17")
   结果:    "北京今日晴，15°C~25°C，北风3级"
→ 判断: 还没完成，需要整理成自然语言

═══════════════════════════════════════════════
Round 2
═══════════════════════════════════════════════
① Observe: 搜索结果返回了天气数据
② Think:   数据够了，可以整理成友好回复
③ Act:     直接输出最终答案
   "北京今天晴天，气温15到25摄氏度，北风3级，适合出行。"
→ 判断: 完成！
```

**对于简单问题（Agent 没有工具或不需要工具），可能 1 轮就结束了。**

---

## 3. agentUniverse 的 Agent 六维模型

在 agentUniverse 框架中，一个 Agent 由**六个维度**组成。这就是你看到的每个 Agent YAML 配置的骨架：

```
Agent = Profile + Plan + Action + Memory + Prompt + WorkPattern
```

### 3.1 六维全景

| 维度 | 一句话 | 技术类比 | YAML 位置 |
|------|--------|---------|----------|
| **Profile** | 绑定哪个 LLM，温度多少 | 选数据库引擎 | `profile.llm_model` |
| **Plan** | 怎么思考（ReAct？顺序？） | Controller 路由策略 | `plan.planner` |
| **Action** | 能用什么工具、查什么知识库 | 微服务调用清单 | `action.tool` + `action.knowledge` |
| **Memory** | 记住对话历史 | Redis Session | `memory` |
| **Prompt** | 系统指令（"你是谁""怎么做"） | HTML 模板 | `profile.prompt_version` |
| **WorkPattern** | 多 Agent 协作模式（单 Agent 不用） | 工作流引擎 | `work_pattern` |

### 3.2 以 simple_qa_agent 为例解剖

```yaml
# examples/sample_apps/simple_qa_agent_app/
#   intelligence/agentic/agent/agent_instance/simple_qa_agent.yaml

info:
  name: 'simple_qa_agent'                    # Agent 名称
  description: '一个简单的问答助手'

profile:                                     # ① Profile: 大脑配置
  prompt_version: 'simple_qa_prompt.v1'      #  引用 Prompt 模板
  introduction: '你是一个乐于助人的问答助手'   #  内嵌 Prompt（会与版本合并）
  target: '以友好的方式回答用户问题'
  instruction: '清晰简洁、准确无误...'
  llm_model:                                 #  LLM 配置
    name: 'qwen_llm'                         #  引用 LLM 实例
    model_name: 'qwen2.5-72b-instruct'
    temperature: 0.7                         #  随机度(0=确定, 1=创造)
    max_tokens: 1000

action:                                      # ③ Action: 工具和知识库
  tool: []                                   #  没有工具
  knowledge: []                              #  没有知识库

# memory:                                    # ④ Memory: 被注释了
#   name: 'demo_memory'

plan:                                        # ② Plan: 思考策略
  planner:
    name: 'react_planner'                    #  使用 ReAct 模式

metadata:                                    # 框架元数据
  type: 'AGENT'
  module: 'agentuniverse.agent.template.react_agent_template'
  class: 'ReActAgentTemplate'
```

**从这份配置你能得出什么结论？** 这是一个"极简"Agent：有大脑(LLM)、有人设(Prompt)、用 ReAct 模式思考——但没有工具、没有知识库、没有记忆。本质上是一个聊天机器人。

---

## 4. 源码穿透：agent.run() 的完整旅程

现在跟踪 agentUniverse 中 `agent.run()` 的一次完整调用，看看六个维度如何被串联起来。

### 4.1 入口：Agent.run()

```python
# agentuniverse/agent/agent.py:110-129 (简化)
@trace_agent
def run(self, **kwargs) -> OutputObject:
    """Agent 实例的运行入口"""

    # 第 1 步：校验输入 — 必须包含 input_keys 定义的字段
    self.input_check(kwargs)

    # 第 2 步：包装用户输入为 InputObject（统一数据结构）
    input_object = InputObject(kwargs)

    # 第 3 步：预处理输入 — 注入 chat_history、date、session_id 等
    agent_input = self.pre_parse_input(input_object)

    # 第 4 步：★ 核心执行 — 交给 Planner
    planner_result = self.execute(input_object, agent_input)

    # 第 5 步：解析 Planner 返回的结果
    agent_result = self.parse_result(planner_result)

    # 第 6 步：校验输出
    self.output_check(agent_result)

    # 第 7 步：包装为 OutputObject 返回
    return OutputObject(agent_result)
```

**用后端术语理解：**
- `input_check` = 请求参数校验（express-validator）
- `pre_parse_input` = 给 request 对象附加上下文（middleware）
- `execute` = 核心业务逻辑（service 层）
- `parse_result` = 格式化返回（response serializer）
- `output_check` = 返回结果校验

### 4.2 核心：Agent.execute() → Planner.invoke()

```python
# agentuniverse/agent/agent.py:147-159
def execute(self, input_object: InputObject, agent_input: dict) -> dict:
    """将执行委托给 Planner"""

    # 1. 从 PlannerManager（单例）获取 Planner 实例
    planner_base: Planner = PlannerManager().get_instance_obj(
        self.agent_model.plan.get('planner').get('name')
    )

    # 2. 调用 Planner.invoke()
    planner_result = planner_base.invoke(
        self.agent_model,   # Agent 的完整配置（六个维度都在里面）
        agent_input,        # 预处理后的输入
        input_object        # 原始输入
    )
    return planner_result
```

**Planner 是真正的"大脑调度器"。** Agent 类只负责前置校验和后置处理，中间的思考逻辑全部委托给 Planner。

### 4.3 ReActPlanner.invoke()：ReAct 循环的落地

```python
# agentuniverse/agent/plan/planner/react_planner/react_planner.py:46-77
def invoke(self, agent_model, planner_input, input_object) -> dict:
    """ReAct 模式的核心执行"""

    # ① 处理 Memory（加载历史对话）
    memory = self.handle_memory(agent_model, planner_input)

    # ② 获取 LLM 实例（大脑）
    llm = self.handle_llm(agent_model)

    # ③ 获取已注册的 Tool，转为 LangChain 格式
    tools = self.acquire_tools(agent_model.action)

    # ④ 组装 Prompt（系统指令 + 工具描述 + 用户输入）
    prompt = self.handle_prompt(agent_model, planner_input)

    # ⑤ 创建 ReAct Agent（LangChain 的）
    agent = create_react_agent(llm.as_langchain(), tools, prompt.as_langchain())

    # ⑥ 包装为 AgentExecutor（控制循环次数）
    agent_executor = AgentExecutor(
        agent=agent, tools=tools,
        max_iterations=15,    # ★ 最多 15 轮循环！
        verbose=True,
        handle_parsing_errors=True
    )

    # ⑦ 执行！这里就是 Observe→Think→Act 循环
    return agent_executor.invoke(input=planner_input, ...)
```

### 4.4 完整调用关系图

```
用户调用 agent.run(input="...")
  │
  ▼
Agent.run()                              ← 入口：7 步流水线
  ├── input_check()                       ← 参数校验
  ├── pre_parse_input()                   ← 注入 {date}, {chat_history}, {session_id}
  │
  ├── execute()                           ← 委托给 Planner
  │   └── ReActPlanner.invoke()           ← 核心调度
  │       ├── handle_memory()             ← 从 Memory Storage 加载对话历史
  │       ├── handle_llm()                ← 从 LLMManager 获取 LLM 实例
  │       ├── acquire_tools()             ← 从 ToolManager 获取工具列表
  │       ├── handle_prompt()             ← 组装 Prompt（模板 + 变量替换）
  │       └── AgentExecutor.invoke()      ← ★ 执行 ReAct 循环
  │           └── LLM API 调用            ← 真正的 AI 推理
  │
  ├── parse_result()                      ← 提取最终输出
  └── OutputObject                        ← 包装返回
```

---

## 5. 各维度深入理解

### 5.1 LLM（大脑）：Agent 的推理引擎

agentUniverse 支持的 LLM 厂商（都在 `agentuniverse/llm/default/` 下）：

```
qwen_openai_style_llm.py      # 通义千问 (Qwen)
deepseek_openai_style_llm.py  # DeepSeek
openai_openai_style_llm.py    # OpenAI GPT
claude_openai_style_llm.py    # Anthropic Claude
gemini_openai_style_llm.py    # Google Gemini
kimi_openai_style_llm.py      # Moonshot Kimi
ollama_llm.py                 # 本地 Ollama
vllm_llm.py                   # vLLM 推理服务
aws_bedrock_llm.py            # AWS Bedrock
...
```

**temperature 参数的含义（这不是比喻，是真的效果）：**
- `0.0 ~ 0.3`：严谨、确定（数学题、代码、事实查询）
- `0.4 ~ 0.7`：平衡（一般对话、问答）
- `0.8 ~ 1.0`：创造性强（写诗、脑暴、创意文案）
- `> 1.0`：几乎随机（极少使用，除非故意要"发疯"）

### 5.2 Prompt（人设）：告诉 Agent 它是谁、做什么、怎么做

```yaml
# intelligence/agentic/prompt/simple_qa_prompt.yaml
name: 'simple_qa_prompt'
introduction: '你是一个乐于助人且友好的问答助手。'     # 角色设定
target: '以友好和对话的方式回答用户的问题。'           # 任务目标
instruction: |                                       # 行为约束
  1. 清晰简洁 - 直接回答问题
  2. 准确无误 - 如果不确定，请明说
  3. 友好亲切 - 使用温暖的语气
  4. 语言适应 - 用与问题相同的语言回答
  5. 乐于助人 - 提供额外背景信息
metadata:
  type: 'PROMPT'
  version: 'simple_qa_prompt.v1'    # 支持版本号！
```

**Prompt 版本化**是 agentUniverse 的一个特色：你可以有 `simple_qa_prompt.v1`、`simple_qa_prompt.v2`、`simple_qa_prompt.cn`（中文）、`simple_qa_prompt.en`（英文），然后在 Agent YAML 中通过 `prompt_version` 一行切换。这就像管理代码分支一样管 Prompt。

### 5.3 Tool（手脚）：让 Agent 能"做事"

纯 LLM 只能聊天，不能干实事。Tool 是 Agent 和外部世界的桥梁：

```python
class Tool(ComponentBase):
    name: str             # 工具名（LLM 用来决定调用哪个）
    description: str       # ★ 这是给 LLM 看的文档！
    
    def execute(self, **kwargs):
        """真正干活的代码（子类必须实现）"""
        pass
```

**关键理解：`description` 字段是写给 LLM 看的。** LLM 通过阅读 Tool 的描述来决定"什么时候该用这个工具"。description 写得越清晰，LLM 调用工具的准确率越高。

### 5.4 Memory（记忆）：跨轮对话的上下文

没有 Memory 的 Agent = 金鱼记忆（7 秒就忘）：

```
无 Memory:
  用户: "我叫小明"     → Agent: "你好小明！"
  用户: "我叫什么？"   → Agent: "我不知道你的名字。"  ← 忘了

有 Memory:
  用户: "我叫小明"     → Agent: "你好小明！" [存入 Memory]
  用户: "我叫什么？"   → Agent: "你叫小明。"  ← 记住了
```

agentUniverse 的 Memory 系统有三层：**Memory（管理器）→ MemoryStorage（存哪里）→ MemoryCompressor（超长时压缩摘要）**。

### 5.5 Knowledge/RAG（知识库）：私有文档注入

LLM 的知识截止于训练日期，且不知道你公司的内部文档。RAG（检索增强生成）在查询 LLM 之前先从你的文档库中检索相关内容，注入到 Prompt 中。

### 5.6 Plan（规划器）：决定思考策略

不同的 Plan 策略适合不同场景：

| Planner | 策略 | 适用场景 |
|---------|------|---------|
| ReActPlanner | Thought-Action-Observation 循环 | 需要调用工具的通用场景 |
| RagPlanner | 检索优先，轻推理 | 知识库问答 |
| PeerPlanner | 四角色流水线 | 复杂分析报告 |
| ExecutingPlanner | 信息整合 | 知识整理 |
| ReviewingPlanner | 评分反馈 | 质量评审 |

---

## 6. Token 是什么？为什么它很重要？

Token 是 LLM 处理文本的最小单位。不是"字"也不是"词"，而是介于两者之间的一种切分：

```
中文: 1 个汉字 ≈ 1-2 tokens
英文: 1 个单词 ≈ 1-3 tokens
代码: 取决于语言，Python 每行 ≈ 5-15 tokens
```

1000 tokens ≈ 750 个英文单词 ≈ 400-500 个中文字。

Token 的关键约束：
- **Context Window**（上下文窗口）：LLM 一次最多能处理的 token 总数。比如 `qwen2.5-72b` 的 context window 是 128K tokens。
- **Memory 的 `max_tokens`**：控制多少对话历史注入 Prompt，超出会被裁剪。
- **`max_tokens` 参数**：控制 LLM 输出最多生成多少 tokens。

**类比：** Context Window = 你一次能记住的信息上限；max_tokens = 你一次最多能说多少话。

---

## 7. 动手练习

### 练习 1：画出六维模型图

不看文档，凭记忆画出 Agent 的六维模型，标注每个维度的作用和在 YAML 中的位置。

### 练习 2：追踪 agent.run() 源码

打开 `agentuniverse/agent/agent.py`，找到 `run()` 方法（约第 110 行），逐行标注每个步骤的作用。

### 练习 3：运行 simple_qa_agent

```bash
cd examples/sample_apps/simple_qa_agent_app
# 1. 编辑 config/custom_key.toml，配置你的 API Key
# 2. 启动
python bootstrap/intelligence/server_application.py
# 3. 通过 curl 或浏览器向 Agent 提问
```

### 练习 4：改配置看变化

1. 把 `temperature` 从 `0.7` 改为 `0.05`，感受回复的变化
2. 把 `max_tokens` 改为 `50`，观察回复被截断的效果
3. 修改 `instruction`，加一条"所有回复必须以'叮咚！'开头"，看效果

### 练习 5：手动模拟 ReAct

不用代码，写出以下场景的 ReAct 循环（每轮的 Thought → Action → Observation）：

> 用户: "帮我算 123 * 456，然后用中文告诉我结果"
> Agent 有 `python_runner` 工具。

---

## 8. 概念速查表

| 概念 | 一句话 | 项目位置 |
|------|--------|---------|
| **Agent** | LLM + Tool + Memory + Prompt 的封装体 | `agent/agent.py` |
| **LLM** | 大语言模型，Agent 的大脑 | `llm/` |
| **ReAct** | Reasoning + Acting 推理行动循环 | `react_planner/react_planner.py` |
| **Planner** | 控制 Agent 的思考策略 | `agent/plan/planner/` |
| **Prompt** | Agent 的系统指令和人设 | `prompt/prompt.py` |
| **Tool** | Agent 可调用的外部功能 | `agent/action/tool/tool.py` |
| **Knowledge** | RAG 知识注入 | `agent/action/knowledge/knowledge.py` |
| **Memory** | 对话历史 + 裁剪机制 | `agent/memory/memory.py` |
| **Token** | LLM 的文本计量单位 | — |
| **Temperature** | LLM 输出的随机性(0=确定, 1=创造) | `profile.llm_model.temperature` |
| **ComponentBase** | 所有组件的基类 | `base/component/component_base.py` |
| **AgentManager** | 全局 Agent 注册表（单例） | `agent/agent_manager.py` |

---

## 下一步

你已经理解了 Agent 的本质和六维模型。下一章 **[03-framework-startup.md](03-framework-startup.md)** 深入框架启动流程——`AgentUniverse().start()` 背后发生了什么、YAML 如何变成内存中的实例。
