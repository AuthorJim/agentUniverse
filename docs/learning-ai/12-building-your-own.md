# 12 — 实战：从零构建你的 Agent 应用

> **前置要求：** 完成前 11 章全部内容
> **学习目标：** 综合运用所有知识点，从零搭建完整的 Agent 应用（单 Agent → 多 Agent → 带 RAG）
> **预计时间：** 3-5 天

---

## 1. 项目脚手架回顾

在前面的学习过程中，你已经见过这套标准目录结构很多次了。现在是时候自己创建了。

```
my_agent_app/
├── pyproject.toml
├── config/
│   ├── config.toml            # 框架配置
│   ├── custom_key.toml        # API Keys（不提交）
│   ├── custom_key.toml.sample # Key 模板（提交）
│   └── log_config.toml        # 日志配置
├── intelligence/
│   └── agentic/
│       ├── agent/agent_instance/   # Agent YAML
│       ├── llm/                    # LLM YAML
│       ├── prompt/                 # Prompt YAML
│       ├── tool/                   # Tool Python + YAML
│       ├── knowledge/              # RAG 组件
│       │   ├── store/
│       │   ├── reader/
│       │   └── doc_processor/
│       └── memory/                 # Memory YAML
├── bootstrap/
│   └── intelligence/
│       └── server_application.py   # 启动入口
```

> 详细脚手架搭建指南见项目根目录的 `scaffold_init_guide.md`。

---

## 2. 实战项目一：技术问答 Agent（难度 ★☆☆）

### 2.1 目标

创建一个能回答编程问题的 Agent。这个项目练习的是 **最基础的 Agent 搭建流程——LLM + Prompt + Agent YAML，不需要 Tool 和 Knowledge。**

### 2.2 步骤

**第 1 步：创建 LLM 配置**

```yaml
# intelligence/agentic/llm/deepseek_llm.yaml
name: 'deepseek_llm'
model_name: 'deepseek-chat'
api_key: '${DEEPSEEK_API_KEY}'
temperature: 0.3          # 技术问答要求严谨，温度偏低
max_tokens: 2000
metadata:
  type: 'LLM'
  module: 'agentuniverse.llm.default.deepseek_openai_style_llm'
  class: 'DeepSeekOpenAIStyleLLM'
```

**第 2 步：创建 Prompt**

```yaml
# intelligence/agentic/prompt/tech_qa_prompt.yaml
name: 'tech_qa_prompt'
introduction: |
  你是一位资深的全栈软件工程师，精通 Python、JavaScript、
  Go 和系统设计。你拥有 10 年的实战经验。
target: |
  准确、专业地回答技术问题。如果合适，提供可运行的代码示例。
instruction: |
  1. 先分析问题的核心——用户真正想问什么
  2. 提供原理说明（为什么这样做）
  3. 给出可运行的代码示例（用 ``` 包裹）
  4. 指出常见的陷阱和最佳实践
  5. 如果问题有争议，说明不同选择的权衡
  6. 使用 Markdown 格式组织回答
  7. 如果不知道答案，诚实说明，不要编造
metadata:
  type: 'PROMPT'
  version: 'tech_qa_prompt.v1'
```

**第 3 步：创建 Agent**

```yaml
# intelligence/agentic/agent/agent_instance/tech_qa_agent.yaml
info:
  name: 'tech_qa_agent'
  description: '技术问答 Agent — 回答编程相关问题'
profile:
  prompt_version: 'tech_qa_prompt.v1'
  llm_model:
    name: 'deepseek_llm'
    model_name: 'deepseek-chat'
    temperature: 0.3
    max_tokens: 3000
action:
  tool: []
  knowledge: []
plan:
  planner:
    name: 'react_planner'
metadata:
  type: 'AGENT'
  module: 'agentuniverse.agent.template.react_agent_template'
  class: 'ReActAgentTemplate'
```

**第 4 步：配置 config.toml**

```toml
[BASE_INFO]
appname = 'my_tech_qa_app'

[CORE_PACKAGE]
default = ['my_tech_qa_app.intelligence.agentic']
agent = ['my_tech_qa_app.intelligence.agentic.agent']
llm = ['my_tech_qa_app.intelligence.agentic.llm']
prompt = ['my_tech_qa_app.intelligence.agentic.prompt']

[SUB_CONFIG_PATH]
custom_key_path = './custom_key.toml'
log_config_path = './log_config.toml'
```

**第 5 步：启动入口**

```python
# bootstrap/intelligence/server_application.py
from agentuniverse.base.agentuniverse import AgentUniverse
from agentuniverse.agent.agent_manager import AgentManager

AgentUniverse().start(config_path='config/config.toml', core_mode=True)
agent = AgentManager().get_instance_obj('tech_qa_agent')
result = agent.run(input='Python 中的装饰器是什么？')
print(result.get_data('output'))
```

**第 6 步：配置 API Key 并运行**

```bash
# 1. 编辑 config/custom_key.toml
# 2. 运行
python bootstrap/intelligence/server_application.py
```

---

## 3. 实战项目二：带工具的研发助手（难度 ★★☆）

### 3.1 目标

在项目一的基础上，给 Agent 加上 `python_runner`（执行 Python 代码）和自定义的计算工具。让 Agent 不仅能回答问题，还能帮你跑代码、做计算。

### 3.2 关键步骤

**新增 Tool：**

```python
# intelligence/agentic/tool/calculator_tool.py
from agentuniverse.agent.action.tool.tool import Tool

class CalculatorTool(Tool):
    """安全的四则运算工具"""
    
    def execute(self, expression: str) -> str:
        import re
        # 安全过滤：只允许数字、运算符、括号、空格
        if not re.match(r'^[\d+\-*/().%\s]+$', expression):
            return 'ERROR: 只允许数字和 + - * / ( ) % 运算符'
        try:
            result = eval(expression)
            return str(result)
        except Exception as e:
            return f'ERROR: 计算失败 — {e}'
```

**更新 Agent YAML：**
```yaml
action:
  tool: ['python_runner', 'calculator_tool']    # ★ 绑定工具
```

**测试：**
```python
agent.run(input='帮我算一下 (1234 * 5678) / 2 等于多少？')
```

---

## 4. 实战项目三：RAG 文档问答 Agent（难度 ★★☆）

### 4.1 目标

将你的项目文档（README.md、API docs 等）向量化存储，构建一个能回答"这个项目怎么用"的 Agent。

### 4.2 关键步骤

**Knowledge 配置：**
```yaml
# intelligence/agentic/knowledge/project_knowledge.yaml
name: 'project_knowledge'
description: '项目文档知识库'
stores: ['project_chroma_store']
insert_processors: ['doc_splitter']
readers:
  md: 'default_txt_reader'
  txt: 'default_txt_reader'
metadata:
  type: 'KNOWLEDGE'
  module: 'agentuniverse.agent.action.knowledge.knowledge'
  class: 'Knowledge'
```

**Store 配置：**
```yaml
# intelligence/agentic/knowledge/store/project_chroma_store.yaml
name: 'project_chroma_store'
collection_name: 'project_docs'
embedding_model: 'default_openai_embedding'
persist_directory: './db/project_docs.db'
metadata:
  type: 'STORE'
  module: 'agentuniverse.agent.action.knowledge.store.chroma_store'
  class: 'ChromaStore'
```

**导入文档并测试：**
```python
from agentuniverse.agent.action.knowledge.knowledge_manager import KnowledgeManager

knowledge = KnowledgeManager().get_instance_obj('project_knowledge')
knowledge.insert_knowledge(source_path='./README.md')
knowledge.insert_knowledge(source_path='./docs/')

# 通过 Agent 查询
agent = AgentManager().get_instance_obj('doc_qa_agent')
result = agent.run(input='这个项目的 API 限流策略是什么？')
```

---

## 5. 实战项目四：PEER 行业分析 Agent（难度 ★★★）

### 5.1 目标

使用 PEER 模式构建一个能生成行业分析报告的 Agent 系统。需要配置 4 个子 Agent 的 Prompt 和工具。

### 5.2 四 Agent YAML 骨架

```yaml
# planning_agent.yaml
info:
  name: 'industry_planning_agent'
profile:
  llm_model:
    name: 'deepseek_llm'
    temperature: 0.1         # 规划要严谨
action:
  tool: []                   # Planning 一般不需要工具
metadata:
  module: 'agentuniverse.agent.template.planning_agent_template'
  class: 'PlanningAgentTemplate'

# executing_agent.yaml
info:
  name: 'industry_executing_agent'
profile:
  llm_model:
    name: 'deepseek_llm'
    temperature: 0.2
action:
  tool: ['google_search_tool']  # ★ Executing 需要搜索工具
metadata:
  module: 'agentuniverse.agent.template.executing_agent_template'
  class: 'ExecutingAgentTemplate'

# expressing_agent.yaml — 类似结构
# reviewing_agent.yaml — 类似结构
```

**PEER Agent 组装：**
```yaml
# demo_peer_agent.yaml
info:
  name: 'industry_peer_agent'
  description: 'PEER 模式行业分析'
profile:
  llm_model:
    name: 'default_qwen_llm'
plan:
  planner:
    name: 'peer_planner'
    retry_count: 3
    jump_step: 'planning'
    eval_threshold: 70
action:
  tool: ['google_search_tool']
work_pattern:
  name: 'peer_work_pattern'
metadata:
  type: 'AGENT'
  module: 'agentuniverse.agent.template.peer_agent_template'
  class: 'PeerAgentTemplate'
```

---

## 6. 常见问题排障指南

### 6.1 "Component name 'xxx' is already registered"
**原因：** 同名组件被重复扫描（用户路径和系统路径都扫描到了）  
**解决：** 检查 `config.toml` 的 `CORE_PACKAGE` 路径是否与系统路径重叠，或给组件改名

### 6.2 "Can not find xxx" / ImportError
**原因：** 包路径或模块路径错误  
**排查：**
1. 目录下有没有 `__init__.py`？
2. `config.toml` 的扫描路径是否正确？
3. YAML 的 `metadata.module` 是否指向正确的 Python 模块？

### 6.3 Agent 启动后没有响应
**排查顺序：**
1. `custom_key.toml` 中 API Key 是否正确？
2. YAML 中 `${VAR_NAME}` 变量名是否和 custom_key.toml 中的一致？
3. LLM 能正常访问吗？（curl 测试一下 API 端点）
4. 查看 loguru 输出的日志（启动时打印的警告和错误）

### 6.4 Tool 没被 LLM 调用
**原因：** Tool 的 `description` 写得不清楚，LLM 不知道什么时候该用它  
**解决：** 重写 description，明确说明：
- 什么场景下使用此工具
- 需要什么输入参数
- 返回什么格式的结果

---

## 7. 完整学习路线检查清单

回顾 12 章的全部内容，确保你掌握了：

- [ ] **Python 基础** — 能独立看懂、修改 agentUniverse 源码
- [ ] **Agent 六维模型** — 能解释 Profile/Plan/Action/Memory/Prompt/WorkPattern
- [ ] **框架启动流程** — 能画出 scan → register 的完整链路
- [ ] **Prompt 工程** — 能为 Agent 编写清晰有效的三要素 Prompt
- [ ] **ReAct 模式** — 能逐轮还原 Thought-Action-Observation 循环
- [ ] **Tool 系统** — 能从零编写自定义 Tool 并绑定到 Agent
- [ ] **Memory 系统** — 能配置对话记忆，理解裁剪和压缩机制
- [ ] **RAG 知识库** — 能搭建完整的文档→向量→检索流水线
- [ ] **GRR 模式** — 能解释三 Agent 的生成→评审→改写循环
- [ ] **IS 模式** — 能解释双 Agent 的 checkpoint 式分步协作
- [ ] **PEER 模式** — 能解释四角色的完整推理流水线和 jump_step 机制
- [ ] **实战能力** — 能从零搭建至少一个自定义 Agent 应用

---

## 8. 下一步：学完这套文档后

1. **深入源码：** 打开 `agentuniverse/` 下的核心源码，你会发现现在读起来像读注释一样顺畅
2. **贡献社区：** 在 [agentUniverse GitHub Issues](https://github.com/agentuniverse-ai/agentUniverse/issues) 上参与讨论
3. **阅读论文：** [PEER: Expertizing Domain-Specific Tasks with a Multi-Agent Framework](https://arxiv.org/abs/2407.06985)
4. **探索高级特性：** OpenTelemetry 可观测性、MCP Server、gRPC 服务、工作流引擎

---

## 结语

从 Python 语法零基础，到理解单 Agent 的六维模型，再到掌握 GRR/IS/PEER 三种多 Agent 协作模式——这 12 章的旅程覆盖了从"会用 API"到"理解 Agent 框架设计"的完整跨越。

你作为 Node.js 全栈工程师的已有知识——分布式架构、微服务通信、中间件模式、DI 容器——这些不是负担，而是理解 AI Agent 系统的**加速器**。Agent 框架本质上就是在这些后端基础设施之上，加了一层 LLM 驱动的智能决策层。

现在，去构建你自己的 Agent 应用吧。
