# 12 — 实战：构建自己的 Agent 应用

> **前置要求：** 完成前面所有章节
> **学习目标：** 综合运用所学，从零搭建一个完整的 Agent 应用
> **预计时间：** 3-5 天

---

## 1. 项目脚手架

agentUniverse 提供了标准项目模板 `examples/sample_standard_app/`，包含完整的目录结构和配置文件。

### 1.1 目录结构

```
my_agent_app/
├── config/
│   ├── config.toml            # 框架配置（扫描路径、服务端口等）
│   ├── custom_key.toml        # API Keys（不提交 git）
│   └── log_config.toml        # 日志配置
│
├── intelligence/
│   └── agentic/
│       ├── agent/              # ★ Agent YAML 放这里
│       │   └── agent_instance/
│       ├── llm/                # ★ LLM YAML 放这里
│       ├── prompt/             # ★ Prompt YAML 放这里
│       ├── tool/               # ★ Tool Python+YAML 放这里
│       ├── knowledge/          # ★ Knowledge (RAG) 放这里
│       │   ├── store/
│       │   ├── reader/
│       │   ├── rag_router/
│       │   └── doc_processor/
│       └── memory/             # ★ Memory 配置放这里
│
├── bootstrap/
│   └── intelligence/
│       └── server_application.py  # ★ 启动入口
│
└── pyproject.toml              # ★ 项目依赖
```

### 1.2 启动入口（最小化）

```python
# bootstrap/intelligence/server_application.py
from agentuniverse.agent_serve.web.web_booster import start_web_server
from agentuniverse.base.agentuniverse import AgentUniverse

class ServerApplication:
    @classmethod
    def start(cls):
        AgentUniverse().start()
        start_web_server()

if __name__ == "__main__":
    ServerApplication.start()
```

---

## 2. 实战项目一：技术问答 Agent（难度：★☆☆）

### 2.1 目标

创建一个能回答编程问题的 Agent，绑定 StackOverflow 搜索和 Python 代码执行。

### 2.2 步骤

**第 1 步：创建 LLM 配置**

```yaml
# intelligence/agentic/llm/qwen_llm.yaml
name: 'qwen_llm'
model_name: 'qwen2.5-72b-instruct'
api_key: '${DASHSCOPE_API_KEY}'
temperature: 0.7
max_tokens: 2000
metadata:
  type: 'LLM'
  module: 'agentuniverse.llm.default.qwen_openai_style_llm'
  class: 'QWenOpenAIStyleLLM'
```

**第 2 步：创建 Prompt**

```yaml
# intelligence/agentic/prompt/tech_qa_prompt.yaml
name: 'tech_qa_prompt'
introduction: |
  你是一位资深的软件工程师，擅长 Python、JavaScript 和系统设计。
target: |
  准确、专业地回答技术问题，必要时提供代码示例。
instruction: |
  1. 先分析问题的核心
  2. 提供原理说明
  3. 给出可运行的代码示例
  4. 指出常见的陷阱和最佳实践
  5. 使用 Markdown 格式（代码用 ``` 包裹）
metadata:
  type: 'PROMPT'
  version: 'tech_qa_prompt.v1'
```

**第 3 步：创建 Agent**

```yaml
# intelligence/agentic/agent/agent_instance/tech_qa_agent.yaml
info:
  name: 'tech_qa_agent'
  description: '技术问答 Agent'
profile:
  prompt_version: 'tech_qa_prompt.v1'
  llm_model:
    name: 'qwen_llm'
    model_name: 'qwen2.5-72b-instruct'
    temperature: 0.3
    max_tokens: 3000
action:
  tool: []
  knowledge: []
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

**第 5 步：启动测试**

```python
from agentuniverse.base.agentuniverse import AgentUniverse
from agentuniverse.agent.agent_manager import AgentManager

AgentUniverse().start(config_path='config/config.toml', core_mode=True)

agent = AgentManager().get_instance_obj('tech_qa_agent')
result = agent.run(input='Python 中的装饰器是什么？')
print(result.get_data('output'))
```

---

## 3. 实战项目二：RAG 文档问答 Agent（难度：★★☆）

### 3.1 目标

将你的项目文档（如 README.md、API docs）向量化，构建一个能回答项目问题的 Agent。

### 3.2 关键步骤

**配置 Knowledge 组件：**

```yaml
# intelligence/agentic/knowledge/project_knowledge.yaml
name: 'project_knowledge'
description: '项目文档知识库'
stores: ['project_chroma_store']
insert_processors: ['doc_splitter']
readers:
  txt: 'default_txt_reader'
  md: 'default_txt_reader'
  pdf: 'default_pdf_reader'
metadata:
  type: 'KNOWLEDGE'
  module: 'agentuniverse.agent.action.knowledge.knowledge'
  class: 'Knowledge'
```

**配置 Store：**

```yaml
# intelligence/agentic/knowledge/store/project_chroma_store.yaml
name: 'project_chroma_store'
description: '项目文档 ChromaDB 存储'
collection_name: 'project_docs'
embedding_model: 'default_openai_embedding'
persist_directory: './db/project_docs.db'
metadata:
  type: 'STORE'
  module: 'agentuniverse.agent.action.knowledge.store.chroma_store'
  class: 'ChromaStore'
```

**导入文档：**

```python
from agentuniverse.agent.action.knowledge.knowledge_manager import KnowledgeManager

knowledge = KnowledgeManager().get_instance_obj('project_knowledge')
knowledge.insert_knowledge(source_path='./README.md')
knowledge.insert_knowledge(source_path='./docs/api-reference/')
```

---

## 4. 实战项目三：PEER 多 Agent 分析报告（难度：★★★）

### 4.1 目标

使用 PEER 模式构建一个行业分析报告 Agent。

### 4.2 四个 Agent 的 YAML 配置要点

```yaml
# demo_planning_agent.yaml
info:
  name: 'industry_planning_agent'
profile:
  prompt_version: 'industry_planning_prompt.cn'
  llm_model:
    name: 'deepseek_llm'
    model_name: 'deepseek-v3'
    temperature: 0.1
action:
  tool: []    # Planning Agent 不需要工具
metadata:
  module: 'agentuniverse.agent.template.planning_agent_template'
  class: 'PlanningAgentTemplate'

# demo_executing_agent.yaml
info:
  name: 'industry_executing_agent'
profile:
  llm_model:
    name: 'deepseek_llm'
    temperature: 0.2
action:
  tool: ['google_search_tool']  # ★ 绑定搜索工具
metadata:
  module: 'agentuniverse.agent.template.executing_agent_template'
  class: 'ExecutingAgentTemplate'
```

### 4.3 组装 PEER Work Pattern

```yaml
# intelligence/agentic/agent/agent_instance/demo_peer_agent.yaml
info:
  name: 'demo_peer_agent'
  description: 'PEER 模式行业分析 Agent'
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

## 5. 常见问题排查

### 5.1 "Component name 'xxx' is already registered"

原因：同名组件被重复扫描。检查 `config.toml` 中的 `CORE_PACKAGE` 路径是否有重叠。

### 5.2 "Can not find xxx" / ImportError

原因：包路径不存在或拼写错误。检查：
1. 目录下是否有 `__init__.py`
2. `config.toml` 的 `CORE_PACKAGE` 路径是否正确
3. `metadata.module` 是否指向正确的 Python 模块

### 5.3 Agent 启动后没有响应

排查步骤：
1. 检查 `custom_key.toml` 中 API Key 是否正确
2. 检查 LLM YAML 中的 `api_key: '${VAR_NAME}'` 变量名是否匹配
3. 检查 Agent 的 `action.tool` 引用的 Tool 名是否已注册
4. 查看日志输出（loguru 会打印启动过程中的警告和错误）

### 5.4 Tool 没被 LLM 调用

原因通常是 Tool 的 `description` 写得不清楚。LLM 以为不需要用这个工具。

---

## 6. 从开发到部署

### 6.1 开发环境

```bash
# 本地开发
poetry install
python bootstrap/intelligence/server_application.py
# → http://localhost:8888
```

### 6.2 生产环境

```bash
# 使用 Gunicorn（多进程）
# 在 config.toml 中设置：
[GUNICORN]
activate = 'true'
gunicorn_config_path = './gunicorn_config.toml'
```

### 6.3 Docker 部署

项目提供了 Dockerfile 模板在 `image_build/` 目录：

```dockerfile
# image_build/Dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY . .
RUN pip install poetry && poetry install
CMD ["python", "bootstrap/intelligence/server_application.py"]
```

---

## 7. 学习路线完成检查清单

回顾整个学习路径，检查你是否掌握了以下内容：

- [ ] **Python 基础**：能阅读和修改 agentUniverse 的 Python 源码
- [ ] **Agent 概念**：能解释 Agent ≠ 函数，理解六维模型
- [ ] **框架启动**：能解释 scan → register 的完整流程
- [ ] **Prompt 工程**：能为 Agent 编写清晰有效的 Prompt
- [ ] **ReAct 模式**：能解释 Thought-Action-Observation 循环
- [ ] **Tool 系统**：能编写自定义 Tool 并绑定到 Agent
- [ ] **Memory 系统**：能配置对话记忆并理解裁剪机制
- [ ] **RAG 知识库**：能搭建完整的文档检索流水线
- [ ] **GRR 模式**：能解释三 Agent 的反馈迭代
- [ ] **IS 模式**：能解释双 Agent 的分步检查
- [ ] **PEER 模式**：能解释四 Agent 的流水线协作
- [ ] **实战能力**：能从零搭建自定义 Agent 应用

---

## 下一步

完成本学习路径后，你可以：

1. **深入源码**：阅读 `agentuniverse/` 下的核心源码，理解框架的设计哲学
2. **贡献社区**：在 GitHub Issues 上参与讨论，提交 PR
3. **阅读论文**：[PEER: Expertizing Domain-Specific Tasks with a Multi-Agent Framework](https://arxiv.org/abs/2407.06985)
4. **探索更多**：研究 agentUniverse 的可观测性、MCP Server、gRPC 等高级特性

恭喜你完成了 agentUniverse 系统学习路径！
