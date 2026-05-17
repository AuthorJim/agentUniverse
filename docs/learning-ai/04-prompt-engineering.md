# 04 — Prompt 工程：Agent 的灵魂

> **前置要求：** 完成 [03-framework-startup.md](03-framework-startup.md)
> **学习目标：** 理解 Prompt 的组成部分、版本化管理、组装流程，掌握为 Agent 编写高质量 Prompt 的方法
> **预计时间：** 2-3 天

---

## 1. 什么是 Prompt Engineering？

### 1.1 最简单的类比：你如何给下属布置任务？

想象你让一个新同事写一份报告。你可以说：

> ❌ "写个报告" → 他完全不知道写什么

你也可以说：

> ✅ "写一份 2026 年第一季度市场分析报告，包含三个部分：① 市场大盘数据，② 竞争对手动态，③ 我们的机会点。参考附件 A 的数据，用专业但不生硬的语言，1000 字以内，周五前交。"

前者什么都不清晰，后者给了：角色定位、任务目标、格式约束、数据来源、风格要求、字数限制、截止日期。

**给 LLM 写 Prompt 和这个一模一样。** Prompt 就是你给 AI 的"任务描述"。写得越好，AI 输出越符合预期。

### 1.2 Prompt Engineering 不是什么

- 不是编程语言（没有标准语法）
- 不是一成不变的公式
- 不是"越复杂越好"

**Prompt Engineering 是一门实践艺术**——通过反复试验和迭代，找到让 LLM 输出高质量结果的最有效指令。

---

## 2. agentUniverse 的 Prompt 系统架构

### 2.1 核心组件

```
Prompt 系统 = Prompt(模板定义) + AgentPromptModel(数据模型) + PromptManager(注册管理)
```

```python
# agentuniverse/prompt/prompt_model.py:15-23
class AgentPromptModel(BaseModel):
    """Prompt 内容的数据容器"""

    introduction: Optional[str] = None    # 角色设定
    target: Optional[str] = None          # 任务目标
    instruction: Optional[str] = None     # 行为约束
```

### 2.2 YAML 中 Prompt 的两种定义方式

**方式一：独立的 Prompt YAML 文件（推荐，支持版本化）**

```yaml
# intelligence/agentic/prompt/simple_qa_prompt.yaml
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
metadata:
  type: 'PROMPT'
  version: 'simple_qa_prompt.v1'    # ★ 版本标识
```

然后 Agent YAML 通过 `prompt_version` 引用：

```yaml
# agent YAML 中
profile:
  prompt_version: 'simple_qa_prompt.v1'    # 引用上面的 Prompt
```

**方式二：内嵌在 Agent YAML 中（简单场景）**

```yaml
# agent YAML 中直接写
profile:
  introduction: |
    你是一个...
  target: |
    你的目标是...
  instruction: |
    请遵守以下规则...
  llm_model: ...
```

**两种方式的优先级：** 如果两者都存在（Agent YAML 有 `introduction/target/instruction`，又有 `prompt_version`），两个版本会**合并**（`AgentPromptModel.__add__`），Agent 内嵌的值覆盖 Prompt 文件中的值。

### 2.3 Prompt 的本质是一个字符串模板

不管上面哪种方式，最终 prompt 会变成一个**带占位符的字符串模板**：

```
你是 X 角色。
你的任务是 Y。

规则：
1. ...
2. ...

当前日期：{date}
历史对话：{chat_history}
用户问题：{input}
```

框架运行时，把 `{变量名}` 替换为实际值，然后发送给 LLM。

---

## 3. Prompt 三要素深度解析

### 3.1 Introduction（角色设定）— "你是谁"

**Introduction 定义了 Agent 的身份、专业领域和语气风格。**

```yaml
# 示例 1：通用助手（来自 simple_qa_prompt.yaml）
introduction: |
  你是一个乐于助人且友好的问答助手。

# 示例 2：专业分析师（来自 PEER planning_agent/cn.yaml）
introduction: |
  你是一位精通信息分析的ai助手。

# 示例 3：内容评审专家（来自 GRR reviewing_agent/cn.yaml）
introduction: |
  你是一位专业的内容评审专家。

# 示例 4：文本编辑专家（来自 PEER expressing_agent/cn.yaml）
introduction: |
  你是一个文本编辑专家，你非常擅长从繁杂的信息中提取关键信息。
```

**书写原则：**
- 明确专业领域（"精通信息分析" vs 泛泛的"AI 助手"）
- 设定语气基调（"友好"、"专业"、"严谨"）
- 限制范围（"不要回答政治问题"等排除性声明也可放在 introduction）

### 3.2 Target（任务目标）— "你要做什么"

**Target 定义了 Agent 要完成的具体任务。**

```yaml
# 拆解问题（planning agent）
target: |
  你的目标是针对用户提出的问题，进行拆解并生成2-4个子问题。

# 整合知识回答问题（executing agent）
target: |
  你的目标是根据用户提供的问题，对查找到的知识进行整合、修正，用来回答用户问题。

# 评估内容质量（reviewing agent）
target: |
  你的目标是评估生成内容的质量，并提供改进建议。

# 生成结构化答案（expressing agent）
target: |
  你的任务是根据背景信息提供的q&a信息结合你自身的知识，
  对用户提出的具体问题，生成一个完整的结构化问题答案。
```

**书写原则：**
- 一句话说清楚目标
- 使用具体的动词：拆解、整合、评估、生成、提取
- 可以包含量化约束：生成 2-4 个子问题

### 3.3 Instruction（行为约束）— "你怎么做"

**Instruction 定义了 Agent 的行为规则、输出格式和质量标准。这是最有发挥空间的部分。**

```yaml
# 来自 PEER planning_agent 的 instruction（简化）
instruction: |
  要求如下:
  1. 思考链路必须严格遵循需要回答的问题
  2. 每一步的问题必须是有答案的，不能是开放性的
  3. 每一步的问题必须是完整的句子
  4. 子问题不要出现英文标点符号
  5. 子问题应包含明确的主体和客体信息

  输出格式：
  ```json
  {
      "thought": "拆解思考过程",
      "framework": ["子问题1", "子问题2", "子问题3"]
  }
  ```
```

```yaml
# 来自 GRR reviewing_agent 的 instruction（简化）
instruction: |
  评分维度：
  1. 准确性 - 内容是否准确回应了用户需求
  2. 完整性 - 是否没有遗漏重要信息
  3. 逻辑性 - 结构是否清晰
  4. 语言质量 - 表达是否流畅
  5. 实用性 - 是否对用户有实际价值

  评分标准：
  - 80-100分: 优秀
  - 60-79分: 良好
  - 40-59分: 及格
  - 0-39分: 不及格

  输出 JSON：
  {
      "score": 75,
      "output": "评审总结",
      "suggestion": "改进建议"
  }
```

**书写原则：**
- 用编号列表组织规则（LLM 对编号列表的遵循度更高）
- 给反面例子（"不可以出现 XXX、ABC 等不明确的词语"）
- 指定输出格式（纯文本、JSON、Markdown）
- 提供格式示例（JSON schema 示例）

---

## 4. 占位符系统：Prompt 中的动态变量

### 4.1 框架自动注入的变量

```yaml
instruction: |
  今天的日期是: {date}              # ★ 自动注入当前日期
  之前的对话: {chat_history}         # ★ 自动注入 Memory 历史
  背景信息是: {background}           # ★ 自动注入 Knowledge/Tool 结果
  用户问题: {input}                  # ★ 自动注入用户输入
```

这些占位符由 `Agent.pre_parse_input()` 自动填充：

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
    self.parse_input(input_object, agent_input)
    return agent_input
```

### 4.2 自定义占位符

你可以在 Prompt 中定义任意占位符，然后在 Agent 模板的 `parse_input()` 方法中填充：

```python
# 你的 Agent Template 中
def parse_input(self, input_object: InputObject, agent_input: dict) -> dict:
    agent_input['input'] = input_object.get_data('input')
    agent_input['user_name'] = input_object.get_data('user_name', '用户')
    agent_input['user_level'] = input_object.get_data('user_level', 'beginner')
    return agent_input
```

然后在 Prompt 中使用 `{user_name}` 和 `{user_level}`。

### 4.3 多 Agent 模式的特有占位符

PEER/GRR 等多 Agent 模式会自动传递特定占位符：

| 占位符 | 含义 | 谁填充 |
|--------|------|--------|
| `{input}` | 用户原始输入 | 用户 |
| `{expressing_result}` | Expressing Agent 的输出 | 框架（传给 Reviewing） |
| `{background}` | Tool/Knowledge 执行结果 | Planner |
| `{executing_result}` | Executing Agent 的输出 | 框架 |
| `{planning_result}` | Planning Agent 的输出 | 框架 |

**这些占位符就是多 Agent 之间传递信息的"管道"。**

---

## 5. Prompt 版本化管理

### 5.1 为什么需要版本化？

- **A/B 测试：** 同一 Agent 绑定不同版本 Prompt（v1 vs v2），对比效果
- **快速回滚：** 新版本 Prompt 效果不好，直接切回旧版本
- **多语言：** `cn.yaml`（中文版）、`en.yaml`（英文版）通过不同版本号管理
- **实验迭代：** 正在打磨一个新 Prompt，不影响生产环境

### 5.2 版本命名约定

```yaml
# Prompt YAML 的 metadata
metadata:
  type: 'PROMPT'
  version: 'simple_qa_prompt.v1'       # 格式：{name}.{version_tag}

# 或者多语言版本约定
metadata:
  type: 'PROMPT'
  version: 'demo_planning_agent.cn'     # .cn / .en 标识语言
```

### 5.3 在 Agent 中切换版本

```yaml
# Agent YAML 中
profile:
  prompt_version: 'simple_qa_prompt.v2'    # 只需改这一个字段
```

### 5.4 版本合并机制

```python
# agentuniverse/prompt/prompt_model.py:25-33
def __add__(self, other):
    """合并两个 AgentPromptModel（版本 + 内嵌）"""
    merged_object = AgentPromptModel()
    for key in set(self.__dict__.keys()).union(other.__dict__.keys()):
        value = getattr(self, key, None)     # 先取当前对象的值
        if value is None:
            value = getattr(other, key, None) # 当前没有才用 other 的
        setattr(merged_object, key, value)
    return merged_object
```

**合并优先级：Agent 内嵌 > Prompt 版本文件**

也就是说，如果 Agent YAML 的 `profile.introduction` 有值，它会**覆盖** `prompt_version` 引用的文件中的 `introduction`。

---

## 6. Prompt 组装源码全流程

让我们跟踪从 YAML 到发给 LLM 的最终 Prompt 字符串的完整链路。

### 6.1 流程总览

```
YAML 文件（Prompt / Agent Profile）
      │
      ▼
AgentPromptModel（Pydantic 数据对象）
  introduction: "你是一个..."
  target: "你的目标是..."
  instruction: "请遵守..."
      │
      ├─ 方式 A：generate_template() → 合并为纯文本字符串
      │
      └─ 方式 B：generate_chat_template() → 组装为 Message 列表
              │
              ▼
         ChatPrompt（多条 Message）
           Message(type='system', content="...")
           Message(type='human', content="...")
              │
              ▼
         ChatPrompt.as_langchain() → ChatPromptTemplate
              │
              ▼
         LLM API 调用
```

### 6.2 方式 A：纯文本 Prompt（legacy Prompt 类）

```python
# agentuniverse/base/util/prompt_util.py:1-15
def generate_template(agent_prompt_model: AgentPromptModel,
                       prompt_assemble_order: list[str]) -> str:
    """将 AgentPromptModel 按顺序拼接为纯文本字符串"""
    values = []
    for attr in prompt_assemble_order:     # ['introduction', 'target', 'instruction']
        value = getattr(agent_prompt_model, attr, None)
        if value is not None:
            values.append(value)

    return "\n".join(values)     # 用换行符拼接
```

**结果：**
```
你是一个乐于助人的问答助手。

以友好的方式回答用户问题。

1. 清晰简洁
2. 准确无误
...
```

### 6.3 方式 B：ChatPrompt（多消息 Prompt）—— 推荐方式

```python
# agentuniverse/base/util/prompt_util.py:17-40
def generate_chat_template(agent_prompt_model: AgentPromptModel,
                            prompt_assemble_order: list[str]) -> list[Message]:
    """将 AgentPromptModel 组装为 Message 列表（区分 system 和 human 角色）"""
    message_list = []
    for attr in prompt_assemble_order:
        value = getattr(agent_prompt_model, attr, None)
        if value is not None:
            # ★ introduction 和 target 标记为 SYSTEM 消息
            # ★ instruction 标记为 HUMAN 消息
            message_list.append(
                Message(
                    type=agent_prompt_model.get_message_type(attr),
                    content=value
                )
            )

    # ★ 合并所有 system 消息为一条，放在列表最前面
    system_messages = '\n'.join(...)
    message_list.insert(0, Message(type='system', content=system_messages))

    return message_list
```

**关键设计：三种 prompt 属性对应不同消息角色**

```python
# prompt_model.py:21-23
_message_type_mapping = {
    'introduction': 'system',      # ★ 角色设定 → SYSTEM
    'target': 'system',            # ★ 目标任务 → SYSTEM
    'instruction': 'human',        # ★ 行为约束 → HUMAN
}
```

**为什么 instruction 映射为 HUMAN 而不是 SYSTEM？**

这是 OpenAI/Anthropic 等模型的一条**含蓄规则**：LLM 对 "user 说的话" 的遵从度有时候高于 "system 说的话"。把 `instruction`（最重要的行为约束）放在 HUMAN 位置，可以提高模型的遵循率。

### 6.4 Prompt 组装的调用触发点

在 `ReActPlanner.handle_prompt()` 中：

```python
# react_planner/react_planner.py:108-147
def handle_prompt(self, agent_model, planner_input):
    # 1. 从 Agent Model 提取 profile 中的 prompt 定义
    profile = agent_model.profile
    profile_prompt_model = AgentPromptModel(
        introduction=profile.get('introduction'),
        target=profile.get('target'),
        instruction=profile.get('instruction')
    )

    # 2. 如果配置了 prompt_version，加载版本化 Prompt
    prompt_version = profile.get('prompt_version')
    version_prompt = PromptManager().get_instance_obj(prompt_version)
    if version_prompt:
        version_prompt_model = AgentPromptModel(
            introduction=getattr(version_prompt, 'introduction', ''),
            target=getattr(version_prompt, 'target', ''),
            instruction=getattr(version_prompt, 'instruction', '')
        )
        # ★ 合并两个 Prompt 模型
        profile_prompt_model = profile_prompt_model + version_prompt_model

    # 3. 组装最终 Prompt
    prompt = Prompt().build_prompt(profile_prompt_model, self.prompt_assemble_order)
    return prompt
```

---

## 7. 长文本处理：Prompt 压缩策略

当 `background`（工具/知识库返回的结果）太长时，直接塞进 Prompt 会超出 LLM 的上下文窗口（token limit）。agentUniverse 提供了三种压缩策略：

```python
# agentuniverse/prompt/enum.py
class PromptProcessEnum(Enum):
    TRUNCATE = 'truncate'        # 直接截断
    STUFF = 'stuff'              # LLM 摘要压缩
    MAP_REDUCE = 'map_reduce'    # 分片分别摘要，再合并摘要
```

**在 Agent YAML 中配置：**

```yaml
profile:
  llm_model:
    name: 'qwen_llm'
    prompt_processor:           # ★ Prompt 压缩配置
      type: 'truncate'          # 或 'stuff' 或 'map_reduce'
      llm: 'qwen_llm'           # 用于摘要的 LLM
      summary_prompt_version: 'prompt_processor.summary_cn'
      combine_prompt_version: 'prompt_processor.combine_cn'
```

| 策略 | 原理 | 适用场景 |
|------|------|---------|
| `truncate` | 直接切掉超长的部分 | 对上下文完整性要求不高的场景 |
| `stuff` | 让 LLM 把长文本摘要成短文本 | 文本结构简单，一次摘要即可 |
| `map_reduce` | 分段分别摘要，再汇总摘要 | 文本很长且结构复杂 |

---

## 8. 编写高质量 Prompt 的实践经验

### 8.1 五原则

**原则 1：角色越具体，输出越专业**

```
❌ "你是一个AI助手"
✅ "你是一位拥有10年经验的金融分析师，专精于宏观经济趋势预测"
```

**原则 2：目标用动作动词**

```
❌ "帮助用户了解天气"
✅ "给定城市名称，返回今日天气的四个维度：温度、湿度、风速、降水概率"
```

**原则 3：规则用编号，越具体越好**

```
❌ "输出友好一点"
✅ "1. 开头用'您好！'问候
   2. 每条信息单独成段
   3. 使用敬语'您'而非'你'
   4. 回复结尾添加'如有其他问题，随时问我'"
```

**原则 4：给出输出格式示例**

```
❌ "输出JSON格式"
✅ "输出必须是严格的JSON：
   {"temperature": 25, "humidity": "60%", "wind": "北风3级", "rain": "10%"}
   不要包含任何JSON之外的文字。"
```

**原则 5：明确排除边界**

```
❌ 不提不能做什么
✅ "不可以出现XXX、ABC等不明确的词语"
   "不要延伸这个问题"
   "不要回答任何与医学建议相关的问题"
```

### 8.2 常见误区

| 误区 | 为什么有问题 | 更好的做法 |
|------|------------|-----------|
| 规则太多太散 | LLM 容易"忘记"后面的规则 | 控制在 5-8 条，分组归类 |
| 正面指令含糊 | "做得好"没有标准 | "准确率 > 95%"有标准 |
| 角色太宽泛 | "AI 助手"没有专业约束 | 加上专业领域 |
| 没有输出格式 | LLM 可能输出不便于程序解析 | 指定 JSON/Markdown |
| 用英文 Prompt 问中文 | 可能得到英文回答 | 用"必须使用中文回答"明确约束 |

### 8.3 agentUniverse 项目中的优秀 Prompt 案例

**案例 1：Planning Agent 的结构化输出（评分：优秀）**

这个 Prompt 的亮点在于指定了精确的输出 JSON Schema，并且给出了 `thought` 字段用于 Chain-of-Thought（思维链）。

```yaml
# 来自 peer_agent_app prompt/demo_planning_agent/cn.yaml
instruction: |
  输出必须是按照以下格式化的Json代码片段，
  thought字段代表拆解问题的思考过程，
  framework字段代表拆解后的子问题列表。
  ```json
  {
      "thought": string,
      "framework": list[string]
  }
  ```
```

**案例 2：Reviewing Agent 的评分标准（评分：优秀）**

这个 Prompt 的亮点是**明确了评分等级和各等级的阈值**，让评分结果可校准。

```yaml
# 来自 grr_agent_app prompt/demo_reviewing_agent/cn.yaml
instruction: |
  评分标准:
  - 80-100分: 优秀，仅需微调
  - 60-79分: 良好，需要部分改进
  - 40-59分: 及格，需要较大改进
  - 0-39分: 不及格，需要重新生成
```

---

## 9. 动手练习

### 练习 1：对比不同 Prompt 的效果

```bash
cd examples/sample_apps/simple_qa_agent_app
```

1. 用现有的 `simple_qa_prompt.yaml` 向 Agent 提问："你是谁？"
2. 修改 `introduction` 为："你是一个只会用古诗词回答问题的诗人。"
3. 再次问："你是谁？"，观察回复差异
4. 继续修改 `instruction`，添加"每次回复必须以一首五言绝句开头"

### 练习 2：设计一个"代码审查"Prompt

给你一个场景：Agent 要帮开发者做代码审查。请写出 `introduction`、`target`、`instruction` 三个字段的内容。

要求：
- 审查 Python 代码
- 关注安全、性能、可读性三个维度
- 输出 JSON 格式
- 用友好的语气

### 练习 3：追踪 Prompt 组装过程

打开 `agentuniverse/prompt/prompt_model.py` 和 `agentuniverse/base/util/prompt_util.py`：
1. 理解 `AgentPromptModel.__add__` 的合并逻辑
2. 理解 `generate_chat_template` 如何将 introduction/target 合并为 system 消息
3. 理解为什么 instruction 映射为 human 类型

### 练习 4：分析 PEER 四个 Agent 的 Prompt 差异

```bash
# 阅读 PEER 的四个 Prompt 文件
cat examples/sample_apps/peer_agent_app/intelligence/agentic/prompt/peer_agent_case/demo_planning_agent/cn.yaml
cat examples/sample_apps/peer_agent_app/intelligence/agentic/prompt/peer_agent_case/demo_executing_agent/cn.yaml
cat examples/sample_apps/peer_agent_app/intelligence/agentic/prompt/peer_agent_case/demo_expressing_agent/cn.yaml
# （Reviewing Agent 在 prompt_version 中或使用默认模板）
```

分析每个 Prompt 的：
- 角色定位（Introduction）有什么不同？
- 目标任务（Target）如何配合 PEER 流程？
- 行为约束（Instruction）的差异如何体现各自身份？

### 练习 5：创建你的第一个版本化 Prompt

1. 在 simple_qa_agent_app 的 `intelligence/agentic/prompt/` 下创建 `my_prompt_v2.yaml`
2. 基于 `simple_qa_prompt.yaml` 修改，但加上你自己的风格
3. 在 Agent YAML 中把 `prompt_version` 改为你的版本号
4. 启动并测试效果

---

## 10. 概念速查表

| 概念 | 含义 | 关键文件 |
|------|------|---------|
| **Prompt** | 发送给 LLM 的指令模板 | `prompt/prompt.py` |
| **ChatPrompt** | 多条 Message 组成的 Prompt | `prompt/chat_prompt.py` |
| **AgentPromptModel** | Prompt 内容的数据模型（introduction + target + instruction） | `prompt/prompt_model.py` |
| **introduction** | 角色设定 → SYSTEM 消息 | — |
| **target** | 任务目标 → SYSTEM 消息 | — |
| **instruction** | 行为约束 → HUMAN 消息（巧思: 提高遵循率） | — |
| **prompt_version** | Prompt 版本号，用于引用和切换 | Agent YAML 的 `profile.prompt_version` |
| **PromptManager** | Prompt 的注册管理器（单例） | `prompt/prompt_manager.py` |
| **prompt_assemble_order** | 组装顺序: `['introduction', 'target', 'instruction']` | `planner.py:46` |
| **generate_chat_template** | 将 Prompt 模型转为 Message 列表 | `base/util/prompt_util.py` |
| **PromptProcessEnum** | 长文本压缩策略（truncate / stuff / map_reduce） | `prompt/enum.py` |
| **可观测变量** | `{input}`, `{date}`, `{chat_history}`, `{background}` 等占位符 | `agent.py:164-181` |

---

## 下一步

你已经理解了：
- Prompt 的三要素设计（introduction / target / instruction）
- agentUniverse 的 Prompt 组装全流程
- 版本化管理和合并优先级
- 长文本压缩策略
- 高质量 Prompt 的编写原则

下一步进入 **05-react-pattern.md**，学习 ReAct 模式——Agent 如何通过"思考→行动→观察"的循环来解决复杂问题。
