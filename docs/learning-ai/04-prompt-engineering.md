# 04 — Prompt 工程：Agent 的灵魂配方

> **前置要求：** 完成 [03-framework-startup.md](03-framework-startup.md)
> **学习目标：** 理解 Prompt 的三要素设计、版本化管理机制、组装源码流程，掌握高质量 Prompt 的编写方法论
> **预计时间：** 2-3 天

---

## 1. 先来一个类比：你是怎么给新同事布置任务的？

想象你让一个新同事写一份市场分析报告。你可以说：

> ❌ "写个报告。"

结果：他完全不知道写什么，交了份乱七八糟的东西。

你也可以说：

> ✅ "写一份 **2026 年 Q1 新能源汽车市场分析报告**，包含三部分：
> ① 市场大盘数据（销量、增长率、市场份额）
> ② 前 5 名竞争对手的动态分析
> ③ 我们的机会点和建议。
> 参考附件 A 的原始数据，用**专业但不生硬**的语言，
> **1000 字以内**，**周五下班前**交。"

前者什么都不清晰。后者给了：**角色定位**（市场分析师）、**任务目标**（市场分析报告）、**格式约束**（三部分结构）、**数据来源**（附件 A）、**风格要求**（专业但不生硬）、**质量约束**（1000 字）、**交付时间**（周五）。

**给 LLM 写 Prompt 和这个一模一样。** Prompt 就是你给 AI 的"任务描述书"。写得越好，AI 输出越符合预期。

### 1.1 Prompt Engineering 不是什么

- 不是编程语言——没有标准语法，没有编译器
- 不是一成不变的公式——同样的 Prompt 在不同模型上效果不同
- 不是"越复杂越好"——有时候简洁直接比长篇大论更有效

**Prompt Engineering 是一门实践艺术：** 通过反复试验和迭代，找到让 LLM 输出高质量结果的最有效指令组合。

---

## 2. agentUniverse 的 Prompt 系统架构

### 2.1 核心组件

```
Prompt 系统 = Prompt(模板定义) + AgentPromptModel(数据容器) + PromptManager(注册管理)
```

### 2.2 AgentPromptModel：三要素数据模型

```python
# agentuniverse/prompt/prompt_model.py:15-23
class AgentPromptModel(BaseModel):
    """Prompt 内容的数据容器——三个字段，各司其职"""

    introduction: Optional[str] = None    # "你是谁" —— 角色设定
    target: Optional[str] = None          # "做什么" —— 任务目标
    instruction: Optional[str] = None     # "怎么做" —— 行为约束
```

这三个字段的设计不是随意的，它们对应了人类给下属布置任务时的自然结构：

| 字段 | 对应 | 示例 |
|------|------|------|
| `introduction` | "你是一位资深 Python 工程师" | 角色设定 |
| `target` | "帮我审查这段代码的安全漏洞" | 任务目标 |
| `instruction` | "1. 先看 SQL 注入 2. 再看 XSS 3. 输出 JSON" | 行为约束 |

### 2.3 YAML 中 Prompt 的两种定义方式

**方式一：独立 Prompt YAML（推荐，支持版本管理）**

```yaml
# intelligence/agentic/prompt/simple_qa_prompt.yaml
name: 'simple_qa_prompt'
introduction: |
  你是一个乐于助人且友好的问答助手。
target: |
  以友好和对话的方式回答用户的问题。
instruction: |
  1. 清晰简洁 - 直接回答问题，不绕弯子
  2. 准确无误 - 如果不确定，请诚实地说"我不确定"
  3. 友好亲切 - 使用"您"的敬语，语气温暖
  4. 语言适应 - 用与问题相同的语言回答
  5. 乐于助人 - 如果合适，提供额外的背景信息
metadata:
  type: 'PROMPT'
  version: 'simple_qa_prompt.v1'    # ★ 版本标识
```

然后在 Agent YAML 中通过 `prompt_version` 引用：

```yaml
profile:
  prompt_version: 'simple_qa_prompt.v1'    # 只需一行！
```

**方式二：内嵌在 Agent YAML 中（简单场景，但不利于复用）**

```yaml
profile:
  introduction: '你是一个...'
  target: '你的目标是...'
  instruction: '请遵守...'
  llm_model: ...
```

**合并规则：** 如果两者同时存在，**Agent 内嵌值覆盖 Prompt 版本文件中的值**。这让你可以先引用一个通用 Prompt 版本，再针对特定 Agent 微调。

---

## 3. Prompt 三要素深度解析

### 3.1 Introduction（角色设定）—— 决定 Agent 的"人设"

Introduction 定义了 Agent 的身份、专业领域和语气基调。这就像在 RPG 游戏里创建角色——你选"战士"还是"法师"，后面的所有行为都受此影响。

```yaml
# ✅ 好的 introduction：专业领域 + 明确角色
introduction: |
  你是一位拥有 10 年经验的 Python 后端工程师。
  你擅长 Django、FastAPI、数据库优化和系统设计。

# ✅ 另一个好例子：特色鲜明
introduction: |
  你是一个精通古诗词的 AI 助手。
  你只会用中国古典诗词的方式思考和回答问题。

# ❌ 差的 introduction：太泛，没有方向
introduction: |
  你是一个 AI 助手。    # 什么类型的 AI 助手？有什么特长？
```

**在 agentUniverse 项目中的实际例子：**

```yaml
# PEER Planning Agent — 分析专家
introduction: 你是一位精通信息分析的 AI 助手。

# GRR Reviewing Agent — 内容评审专家
introduction: 你是一位专业的内容评审专家。

# PEER Expressing Agent — 编辑专家
introduction: 你是一个文本编辑专家，擅长从繁杂信息中提取关键信息。
```

### 3.2 Target（任务目标）—— 决定 Agent 的"使命"

Target 定义了 Agent 要完成什么。动词的使用至关重要——不同的动词导致完全不同的输出。

```yaml
# Planning Agent → 拆解问题
target: |
  你的目标是针对用户提出的问题，进行拆解并生成 2-4 个子问题。

# Executing Agent → 检索整合
target: |
  你的目标是根据用户问题，对查找到的知识进行整合、修正，用以回答用户问题。

# Reviewing Agent → 评估打分
target: |
  你的目标是评估生成内容的质量，并从 5 个维度打分，提供改进建议。

# Expressing Agent → 结构化输出
target: |
  你的任务是根据 Q&A 信息结合你自身的知识，生成完整的结构化答案。
```

**关键洞察：** 四个 PEER Agent 的 target 各不相同，但组合在一起就是一个完整的工作流：拆解 → 检索 → 整理输出 → 质量把关。

### 3.3 Instruction（行为约束）—— 决定 Agent 的"职业素养"

Instruction 是最有发挥空间的部分。它定义了 Agent 的行为规则、输出格式、质量标准和边界条件。

**一个优秀的 Instruction 示例（来自 PEER Planning Agent）：**

```yaml
instruction: |
  要求如下:
  1. 思考链路必须严格遵循需要回答的问题
  2. 每一步的问题必须是有答案的，不能是开放性的
  3. 每一步的问题必须是完整的句子
  4. 子问题不要出现英文标点符号
  5. 子问题应包含明确的主体和客体信息

  输出格式（必须是 JSON）：
  ```json
  {
      "thought": "拆解思考过程",
      "framework": ["子问题1", "子问题2", "子问题3"]
  }
  ```
```

**为什么这个 Instruction 写得好？**
1. **编号列表** — LLM 对编号规则的遵循度显著高于自然段
2. **具体约束** — 不只是"拆解问题"，而是精确到"不能是开放性的""不要英文标点"
3. **输出格式示例** — 明确的 JSON Schema，LLM 可以直接模仿
4. **thought 字段** — 要求 LLM 展示推理过程（Chain-of-Thought），提高最终输出质量

**另一个优秀示例（GRR Reviewing Agent）：**

```yaml
instruction: |
  评分维度：
  1. 准确性 - 内容是否准确回应了用户需求 (权重 30%)
  2. 完整性 - 是否没有遗漏重要信息 (权重 25%)
  3. 逻辑性 - 结构是否清晰，推理是否合理 (权重 20%)
  4. 语言质量 - 表达是否流畅自然 (权重 15%)
  5. 实用性 - 是否对用户有实际价值 (权重 10%)

  评分标准：
  - 80-100分: 优秀，仅需微调
  - 60-79分: 良好，需要部分改进
  - 40-59分: 及格，需要较大改进
  - 0-39分: 不及格，需要重新生成

  请输出 JSON：
  {
      "score": 75,
      "output": "评审总结",
      "suggestion": "改进建议（具体到某一段或某一句）"
  }
```

**亮点：** 明确了评分维度的权重、各等级的含义和阈值，让 LLM 的打分有据可依，不是"凭感觉打分"。

---

## 4. 占位符系统：Prompt 中的动态变量

### 4.1 框架自动注入的变量

```yaml
instruction: |
  今天的日期是: {date}              # ★ 自动注入，格式: YYYY-MM-DD
  之前的对话: {chat_history}         # ★ 从 Memory 加载
  背景信息: {background}             # ★ Tool/Knowledge 执行结果
  用户问题: {input}                  # ★ 用户原始输入
```

这些占位符由 `Agent.pre_parse_input()` 在每次请求时自动填充：

```python
# agentuniverse/agent/agent.py:164-181
def pre_parse_input(self, input_object) -> dict:
    agent_input = dict()
    agent_input['chat_history'] = input_object.get_data('chat_history') or ''
    agent_input['background'] = input_object.get_data('background') or ''
    agent_input['image_urls'] = input_object.get_data('image_urls') or []
    agent_input['date'] = datetime.now().strftime('%Y-%m-%d')
    agent_input['session_id'] = input_object.get_data('session_id') or ''
    agent_input['agent_id'] = self.agent_model.info.get('name', '')
    return agent_input
```

### 4.2 多 Agent 模式的特有占位符

PEER/GRR 等多 Agent 模式通过占位符在 Agent 之间传递信息——这些占位符就是**多 Agent 之间的数据管道**：

| 占位符 | 含义 | 谁填充 | 谁使用 |
|--------|------|--------|--------|
| `{input}` | 用户原始输入 | 用户 | 所有 Agent |
| `{planning_result}` | Planning Agent 的输出 | PeerWorkPattern | Executing Agent |
| `{executing_result}` | Executing Agent 的输出 | PeerWorkPattern | Expressing Agent |
| `{expressing_result}` | Expressing Agent 的输出 | PeerWorkPattern | Reviewing Agent |
| `{reviewing_result}` | Reviewing Agent 的输出 | PeerWorkPattern | 下一轮 Planning Agent |
| `{background}` | Tool/Knowledge 执行结果 | Planner | 需要上下文的 Agent |

### 4.3 自定义占位符

你可以在 Agent Template 的 `parse_input()` 方法中注入自定义变量：

```python
def parse_input(self, input_object: InputObject, agent_input: dict) -> dict:
    agent_input['input'] = input_object.get_data('input')
    agent_input['user_name'] = input_object.get_data('user_name', '访客')
    agent_input['user_level'] = input_object.get_data('user_level', 'beginner')
    return agent_input
```

然后在 Prompt 中使用 `{user_name}` 和 `{user_level}` —— 实现个性化回复。

---

## 5. Prompt 版本化管理

### 5.1 为什么需要版本化？

想象你的 Prompt 就像一个 API 的 handler。你不能直接改线上的 handler 而不留备份。Prompt 版本化解决了：

- **A/B 测试** — `v1` 和 `v2` 同时存在，通过改 Agent YAML 一行切换
- **快速回滚** — 新版本效果不好？一行 `prompt_version: 'xxx.v1'` 切回去
- **多语言管理** — `demo_planning_agent.cn` vs `demo_planning_agent.en`
- **实验隔离** — 新 Prompt 在开发环境打磨，不影响生产

### 5.2 版本命名约定

```yaml
# 版本号约定
metadata:
  version: 'simple_qa_prompt.v1'         # {name}.{version_tag}

# 多语言约定
metadata:
  version: 'demo_planning_agent.cn'       # {name}.{language}
```

### 5.3 版本合并机制（重要！）

```python
# agentuniverse/prompt/prompt_model.py:25-33
def __add__(self, other: 'AgentPromptModel') -> 'AgentPromptModel':
    """合并两个 Prompt 模型 — self 的值优先于 other"""
    merged = AgentPromptModel()
    for key in ['introduction', 'target', 'instruction']:
        value = getattr(self, key, None)        # 先取当前对象的值
        if value is None:
            value = getattr(other, key, None)   # 没有才用 other 的
        setattr(merged, key, value)
    return merged

# 调用顺序:
# Agent 内嵌 Prompt + Prompt 版本文件 = 最终 Prompt
# (内嵌优先)       (版本文件)
```

**合并优先级：Agent 内嵌 > Prompt 版本文件。** 也就是说你可以引用一个通用 Prompt 版本，然后在特定 Agent 上微调某个字段。

---

## 6. Prompt 组装源码全链路

### 6.1 流程总览

```
YAML (Prompt 或 Agent Profile)
      │
      ▼
AgentPromptModel (Pydantic 对象)
  introduction: "..."
  target: "..."
  instruction: "..."
      │
      ├─ 纯文本模式 (legacy): generate_template() → 拼成一个字符串
      │
      └─ Chat 模式 (推荐): generate_chat_template() → 组装为 Message 列表
              │
              ▼
         ChatPrompt (多条 Message)
           Message(type='system', content="...")
           Message(type='human', content="...")
              │
              ▼
         ChatPrompt.as_langchain() → ChatPromptTemplate
              │
              ▼
         发送给 LLM API
```

### 6.2 ChatPrompt 模式：区分 System 和 Human 消息

这是一个巧妙的设计：

```python
# agentuniverse/prompt/prompt_model.py:21-23
_message_type_mapping = {
    'introduction': 'system',      # "你是谁" → SYSTEM（让 LLM 理解为角色设定）
    'target': 'system',            # "做什么" → SYSTEM
    'instruction': 'human',        # "怎么做" → HUMAN（关键！）
}
```

**为什么 `instruction`（行为约束）映射为 HUMAN 而不是 SYSTEM？**

这是基于大量实践经验的一个洞察：**LLM 对 "user 说的话" 的遵循度往往高于 "system 说的话"。** 把最重要的行为约束（instruction）放在 HUMAN 角色位置，LLM 会更加严格地遵守。这是一个"巧思"而非 bug。

### 6.3 Prompt 组装的触发点

在 `ReActPlanner.handle_prompt()` 中：

```python
# react_planner/react_planner.py:108-147 (简化)
def handle_prompt(self, agent_model, planner_input):
    # 1. 从 Agent Model 提取 profile 中的 Prompt 定义
    profile = agent_model.profile
    profile_prompt_model = AgentPromptModel(
        introduction=profile.get('introduction'),
        target=profile.get('target'),
        instruction=profile.get('instruction'),
    )

    # 2. 如果配置了 prompt_version，加载版本化 Prompt
    prompt_version = profile.get('prompt_version')
    if prompt_version:
        version_prompt = PromptManager().get_instance_obj(prompt_version)
        version_prompt_model = AgentPromptModel(
            introduction=getattr(version_prompt, 'introduction', ''),
            target=getattr(version_prompt, 'target', ''),
            instruction=getattr(version_prompt, 'instruction', ''),
        )
        # ★ 合并：内嵌优先
        profile_prompt_model = profile_prompt_model + version_prompt_model

    # 3. 调用 Prompt 构建最终消息
    prompt = Prompt().build_prompt(profile_prompt_model, self.prompt_assemble_order)
    return prompt
```

---

## 7. 长文本压缩：当上下文太长怎么办

当 `background`（工具/知识库返回的结果）太长时，直接塞进 Prompt 会超出 LLM 的上下文窗口。agentUniverse 提供了三种策略：

```python
# agentuniverse/prompt/enum.py
class PromptProcessEnum(Enum):
    TRUNCATE = 'truncate'        # 直接截断
    STUFF = 'stuff'              # LLM 摘要压缩
    MAP_REDUCE = 'map_reduce'    # 分段摘要 + 合并摘要
```

在 Agent YAML 中配置：

```yaml
profile:
  llm_model:
    name: 'qwen_llm'
    prompt_processor:
      type: 'truncate'           # 或 'stuff' 或 'map_reduce'
      llm: 'qwen_llm'
      summary_prompt_version: 'prompt_processor.summary_cn'
      combine_prompt_version: 'prompt_processor.combine_cn'
```

| 策略 | 原理 | Token 成本 | 适用场景 |
|------|------|-----------|---------|
| `truncate` | 直接切掉超长部分 | 零 | 上下文完整性要求不高 |
| `stuff` | LLM 把长文本摘要为短文本 | 中等 | 文本结构简单 |
| `map_reduce` | 分段摘要 → 再汇总摘要 | 较高 | 文本很长且结构复杂 |

---

## 8. 编写高质量 Prompt 的六原则

### 原则 1：角色越具体，输出越专业

```
❌ "你是一个 AI 助手"
✅ "你是一位拥有 10 年经验的金融分析师，专精于宏观经济趋势和二级市场研究"
```

LLM 对"有明确身份"的指令遵循度远高于泛泛的角色。

### 原则 2：目标用精确的动作动词

```
❌ "帮助用户了解天气"
✅ "给定城市名称，返回今日天气的五个维度：温度、湿度、风速、降水概率、空气质量"
```

### 原则 3：规则用编号，每条可验证

```
❌ "输出友好一点"
✅ "1. 开头用'您好！'问候
   2. 每条信息单独成段
   3. 使用敬语'您'而非'你'
   4. 结尾添加'如有其他问题，随时问我'"
```

### 原则 4：给出输出格式示例（JSON Schema 最佳）

```
❌ "输出 JSON 格式"
✅ "输出必须是严格 JSON：
   {"temperature": 25, "humidity": "60%", "wind": "北风3级"}
   不要包含任何 JSON 之外的文字"
```

### 原则 5：明确排除边界（负面约束）

```
❌ 什么都不提
✅ "不要回答任何关于政治立场的问题"
   "不要提供医疗建议"
   "如果不知道答案，直接说'我不确定'，不要编造"
```

### 原则 6：把 LLM 的方法也管起来（Chain-of-Thought）

```
✅ "在给出最终答案前，请先用 '思考过程:' 开头，展示你的推理，
   然后再用 '最终答案:' 开头给出结论"
```

这会触发 LLM 的 Chain-of-Thought，显著提升推理质量。

### 常见误区

| 误区 | 为什么有问题 | 改正 |
|------|------------|------|
| 规则太多（15+ 条） | LLM 会"忘记"后面的 | 控制在 5-8 条，分组归类 |
| 只有正面约束 | "做得好"没有客观标准 | 加上"如果不知道就说不知道" |
| 角色太宽泛 | "AI 助手"无专业约束 | 明确专业领域和年限 |
| 不写输出格式 | LLM 可能输出非结构化文本 | 指定 JSON/Markdown/YAML |
| 英文 Prompt + 中文问题 | 可能得到英文回答 | 明确"用与问题相同的语言回答" |

---

## 9. 动手练习

### 练习 1：对比不同 Prompt 的效果

```bash
cd examples/sample_apps/simple_qa_agent_app
```
1. 用现有 Prompt 问："你是谁？"
2. 修改 `introduction` 为："你是一个只用诗歌形式回答的唐代诗人。"
3. 再问"你是谁？"，看差异
4. 加 `instruction`："每首诗必须是七言绝句，且以'老夫'自称"

### 练习 2：设计一个代码审查 Prompt

写一个用于代码审查的 Prompt，包含完整的 `introduction`、`target`、`instruction`：
- 审查 Python 代码
- 关注安全、性能、可读性
- 输出 JSON 格式（包含 score, issues, suggestions）
- 语气：建设性而非指责

### 练习 3：追踪 Prompt 组装源码

打开 `agentuniverse/prompt/prompt_model.py` 和 `agentuniverse/base/util/prompt_util.py`：
1. 理解 `AgentPromptModel.__add__` 的合并逻辑
2. 理解 `generate_chat_template` 的 System/Human 分类逻辑
3. 思考：为什么 instruction 映射为 human？

### 练习 4：设计版本化 Prompt

1. 创建 `my_prompt.v1.yaml`（保守风格）和 `my_prompt.v2.yaml`（幽默风格）
2. 在同一个 Agent 上切换两个版本，问同样的问题
3. 比较输出差异，总结哪个版本更适合什么场景

### 练习 5：Prompt 压缩实验

找一个长文档（比如一篇 5000 字的文章），分别用 `truncate` 和 `stuff` 两种策略处理。观察：
1. Truncate 后 LLM 回答是否遗漏关键信息
2. Stuff 摘要后 LLM 回答的准确性

---

## 10. 概念速查表

| 概念 | 含义 | 关键文件/位置 |
|------|------|-------------|
| **Prompt** | 发送给 LLM 的指令模板 | `prompt/prompt.py` |
| **ChatPrompt** | 多条 Message 组成的 Prompt | `prompt/chat_prompt.py` |
| **AgentPromptModel** | Prompt 内容的三字段模型 | `prompt/prompt_model.py` |
| **introduction** | 角色设定（→ system 消息） | — |
| **target** | 任务目标（→ system 消息） | — |
| **instruction** | 行为约束（→ human 消息，巧思！） | — |
| **prompt_version** | 版本标识，支持 A/B 和回滚 | Agent YAML `profile.prompt_version` |
| **PromptManager** | Prompt 注册管理器 | `prompt/prompt_manager.py` |
| **generate_chat_template** | 将 Prompt 模型转为 Message 列表 | `base/util/prompt_util.py` |
| **PromptProcessEnum** | 长文本压缩策略 | `prompt/enum.py` |
| **{date} / {chat_history} / {input}** | 框架自动注入的占位符 | `agent.py:164-181` |

---

## 下一步

你已经理解了 Prompt 的完整系统——三要素设计、版本化管理、组装流程、编写技巧。

下一步进入 **[05-react-pattern.md](05-react-pattern.md)**——ReAct 模式，理解 Agent 如何通过"思考→行动→观察"循环自主解决问题。
