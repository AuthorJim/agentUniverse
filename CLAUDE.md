# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

**Setup:**
```bash
pip install poetry
poetry install                    # install all dependencies
poetry install --extras store_ext # with optional pymilvus
pre-commit install                # enable pre-commit hooks (black, ruff)
```

**Testing:**
```bash
pytest                                  # run all tests
pytest tests/test_agentuniverse/unit/   # unit tests only
pytest tests/test_agentuniverse/unit/test_academic_paper_fragmenter.py  # single file
pytest -k "test_name_pattern"           # filter by test name
```

**Linting & Formatting:**
```bash
ruff check .                  # lint
ruff check --fix .            # lint with auto-fix
ruff format .                 # format
black --line-length=120 .     # alternative formatter
pre-commit run --all-files    # run all pre-commit checks
```

**Type checking:**
```bash
mypy agentuniverse            # mypy configured in pyproject.toml
```

**Build:**
```bash
poetry build         # build sdist + wheel
```

The project uses Poetry for dependency management, ruff for linting/formatting, black (120-char lines), mypy, and pytest (via unittest classes — no pytest-specific fixtures needed). Tests use `unittest.TestCase` patterns, not pytest function-style tests. Python 3.10+ required.

## Overview

agentUniverse is an open-source (Apache 2.0) multi-agent framework by AntGroup for building LLM-powered applications. Python 3.10+. Installed via `pip install agentUniverse` (PyPI package `agentUniverse`).

**Three-pillar architecture:**
1. **Single Agent** — composable agent with profile, action, plan, memory, prompt
2. **Multi-Agent Collaboration** — work patterns (PEER, GRR, IS, DOE) + visual workflow engine
3. **Domain Knowledge** — RAG pipeline with stores, readers, doc processors, embeddings

## Core Architecture

### Single Agent (`agentuniverse/agent/agent.py`)

The `Agent` base class assembles from these configurable dimensions (set via YAML or `AgentModel`):

| Dimension | Purpose | Key Files |
|-----------|---------|-----------|
| `profile` | LLM model binding (name, model_name, temperature, etc.) | `agent_model.py:22` (`llm_params()`) |
| `action` | Tools (`tool`) and knowledge bases (`knowledge`) | `agent/action/tool/`, `agent/action/knowledge/` |
| `plan` | Planner/strategy (e.g., ReAct, simple sequential) | `agent/plan/planner/` |
| `memory` | Conversation memory (ChatMemory, message storage) | `agent/memory/` |
| `prompt` | Prompt templates with versioning (`prompt_version`) | `prompt/` |
| `work_pattern` | Multi-agent collaboration pattern binding | `agent/work_pattern/` |

Agents are registered via YAML configs scanned at startup. The YAML `metadata` block specifies `module` + `class`; the rest defines `info`, `profile`, `action`, etc.

### Multi-Agent Collaboration

Two complementary systems:

**Work Patterns** (`agentuniverse/agent/work_pattern/`) — Opinionated multi-agent orchestration patterns:

- **PEER** (`peer_work_pattern.py`) — Plan → Execute → Express → Review. 4 specialised agents. Iterative refinement loop with scoring (`eval_threshold`). Best for complex reasoning: event analysis, industry reports.
- **GRR** (`grr_work_pattern.py`) — Generate → Review → Rewrite. 3 agents. Iterative until quality score met. Best for content generation.
- **IS** (`is_work_pattern.py`) — Implementation + Supervision. 2 agents. Checkpoint-based execution with correction feedback. Best for goal-aligned tasks.

All patterns support `invoke()` (sync) and `async_invoke()` (async). Patterns are `ComponentBase` subclasses, registered and managed as `WORK_PATTERN` components.

**Workflow Engine** (`agentuniverse/workflow/`) — Visual graph-based orchestration:
- `graph/` — DAG execution engine
- `node/` — Node types: `agent_node`, `llm_node`, `tool_node`, `knowledge_node`, `condition_node`, `start_node`, `end_node`
- Used by the visual agentic workflow platform (magent-ui)

### Domain Knowledge Injection (`agentuniverse/agent/action/knowledge/`)

RAG pipeline components:

| Component | Directory | Purpose |
|-----------|-----------|---------|
| `Reader` | `reader/` | Load data from files (PDF, JSON, TXT, etc.) |
| `DocProcessor` | `doc_processor/` | Split, clean, transform documents |
| `Embedding` | `embedding/` | Generate embeddings for documents |
| `Store` | `store/` | Vector storage (ChromaDB, Milvus, etc.) |
| `RagRouter` | `rag_router/` | Route queries to appropriate stores |
| `QueryParaphraser` | `query_paraphraser/` | Paraphrase/rewrite queries |
| `Knowledge` | `knowledge.py` | Orchestrates the full RAG pipeline |

## Key Directories

```
agentuniverse/
├── agent/               # Agent classes, templates, work patterns, memory, context
│   ├── agent.py         # Base Agent class
│   ├── agent_manager.py # Singleton agent registry
│   ├── agent_model.py   # AgentModel (pydantic config container)
│   ├── template/        # Agent templates (PeerAgentTemplate, RAGAgentTemplate, ReActAgentTemplate, etc.)
│   ├── default/         # Pre-built default agent instances
│   ├── work_pattern/    # PEER, GRR, IS pattern implementations
│   ├── action/          # Tools, toolkits, knowledge (RAG) components
│   ├── memory/          # Memory, ChatMemory, message storage/compression
│   ├── plan/            # Planner strategies (ReAct, etc.)
│   └── context/         # Context management (managers, stores, routers)
├── llm/                 # LLM abstraction layer
│   ├── llm.py           # Base LLM class
│   ├── llm_manager.py   # Singleton LLM registry
│   ├── default/         # Vendor-specific implementations (OpenAI, Qwen, DeepSeek, Claude, Gemini, etc.)
│   ├── llm_channel/     # API channel implementations (official SDKs)
│   ├── openai_style_llm.py      # OpenAI-compatible base
│   └── langchain_instance.py    # LangChain wrapping
├── base/                # Framework core
│   ├── agentuniverse.py # AgentUniverse class — startup, scanning, registration
│   ├── config/          # Config loading (TOML, YAML), Configer, custom_key handling
│   ├── component/       # ComponentBase, ComponentEnum, manager bases, registration
│   ├── annotation/      # Decorators (@singleton, @trace_agent, etc.)
│   ├── util/            # Logging (loguru), monitoring (OpenTelemetry), system utils
│   ├── context/         # Framework-level context management
│   └── tracing/         # OpenTelemetry tracing setup
├── prompt/              # Prompt management with versioning
│   ├── prompt.py        # Base Prompt class
│   ├── chat_prompt.py   # ChatPrompt (system + human messages)
│   ├── prompt_manager.py# Singleton prompt registry
│   └── prompt_model.py  # AgentPromptModel
├── workflow/            # Visual workflow engine (graph + nodes)
│   ├── workflow.py      # Workflow class
│   ├── graph/           # DAG graph execution
│   └── node/            # Node types (agent, llm, tool, knowledge, condition, start, end)
├── agent_serve/         # Serving layer
│   ├── web/             # Flask/Gunicorn HTTP servers, request_task, MCP server
│   └── service.py       # Service definitions for API exposure
└── database/            # SQLAlchemy DB wrappers
```

Other top-level dirs:
- `examples/` — Sample apps (`sample_standard_app/`, `sample_apps/`, `startup_app/`)
- `agentuniverse_connector/` — External connector package
- `agentuniverse_extension/` — Extension package
- `agentuniverse_product/` — Visual platform product package (magent-ui integration)
- `docs/` — Guidebook (EN/ZH/JP), API reference
- `tests/` — Test suite

## Component System

All components extend `ComponentBase` and are registered via `ComponentEnum` (in `base/component/component_enum.py`). Types include: `AGENT`, `LLM`, `TOOL`, `TOOLKIT`, `KNOWLEDGE`, `PLANNER`, `MEMORY`, `PROMPT`, `WORKFLOW`, `WORK_PATTERN`, `SERVICE`, `EMBEDDING`, `DOC_PROCESSOR`, `READER`, `STORE`, `RAG_ROUTER`, `QUERY_PARAPHRASER`, `MEMORY_COMPRESSOR`, `MEMORY_STORAGE`, `LLM_CHANNEL`, `LOG_SINK`, `SQLDB_WRAPPER`.

Components are discovered by package scanning (configured in `config.toml` `CORE_PACKAGE` section) and instantiated from YAML configs. Each YAML file declares its `metadata.type` and `metadata.module`/`metadata.class`.

## Configuration

### Main Config (`config/config.toml`)
```toml
[BASE_INFO]           # appname (used as namespace prefix)
[PACKAGE_PATH_INFO]   # ROOT_PACKAGE placeholder
[CORE_PACKAGE]        # Scan paths per component type (agent, llm, tool, knowledge, etc.)
[SUB_CONFIG_PATH]     # Paths to custom_key.toml, log_config.toml
[DB]                  # system_db_uri (SQLAlchemy, defaults to local SQLite)
[GUNICORN]            # activate + gunicorn_config_path
[GRPC]                # activate + port + max_workers
[MONITOR]             # activate + dir
[EXTENSION_MODULES]   # class_list for hooks
```

### API Keys (`config/custom_key.toml`)
Define `[KEY_LIST]` entries. Key variable names follow convention:
- `OPENAI_API_KEY`, `DEEPSEEK_API_KEY`, `DASHSCOPE_API_KEY` (Qwen), `ANTHROPIC_API_KEY` (Claude)
- `KIMI_API_KEY`, `ZHIPU_API_KEY`, `BAICHUAN_API_KEY`, `GOOGLE_API_KEY` (Gemini)
- `QIANFAN_AK`/`QIANFAN_SK` (WenXin), `OLLAMA_BASE_URL`
- Search: `SERPER_API_KEY`, `BING_SUBSCRIPTION_KEY`, `SEARCHAPI_API_KEY`

Referenced in YAML via `${VAR_NAME}` syntax (e.g., `api_key: '${DASHSCOPE_API_KEY}'`).

### Default LLM (`intelligence/agentic/llm/default_llm.toml`)
```toml
[DEFAULT]
default_llm = 'qwen2.5-72b-instruct'
```

### Agent YAML Config Pattern
```yaml
info:
  name: 'agent_name'
  description: '...'
profile:
  prompt_version: prompt_name.version
  llm_model:
    name: 'llm_instance_name'    # references a registered LLM component
    model_name: 'specific-model'
    temperature: 0.1
memory:
  name: 'memory_instance_name'
action:
  tool: ['tool_name_1']
  knowledge: ['kb_name_1']
metadata:
  type: 'AGENT'
  module: 'agentuniverse.agent.template.some_template'
  class: 'SomeAgentTemplate'
```

### LLM YAML Config Pattern
```yaml
name: 'qwen_llm'
description: '...'
model_name: 'qwen2.5-72b-instruct'
max_tokens: 2000
api_key: '${DASHSCOPE_API_KEY}'
temperature: 0.1
metadata:
  type: 'LLM'
  module: 'agentuniverse.llm.default.qwen_openai_style_llm'
  class: 'QWenOpenAIStyleLLM'
```

Supported LLM vendors (pre-built in `agentuniverse/llm/default/`): Qwen, DeepSeek, OpenAI, Claude, Gemini, Kimi, Zhipu (ChatGLM), Baichuan, WenXin, Ollama, vLLM, AWS Bedrock, Doubao.

## Running the Framework

### Minimal Startup
```python
from agentuniverse.base.agentuniverse import AgentUniverse
from agentuniverse.agent.agent_manager import AgentManager

AgentUniverse().start(config_path='config/config.toml', core_mode=True)

# Run an agent
agent = AgentManager().get_instance_obj('demo_peer_agent')
agent.run(input='Your question here')
```

`AgentUniverse().start()` does: load config → load custom keys → scan YAML files → register components → init web/gRPC/OTEL if configured.

### Start as Web Server
```python
from agentuniverse.base.agentuniverse import AgentUniverse
from agentuniverse.agent_serve.web.web_booster import start_web_server

AgentUniverse().start()
start_web_server()  # Flask + optional Gunicorn
```

### Visual Workflow Platform
```bash
pip install magent-ui ruamel.yaml
python examples/sample_apps/workflow_agent_app/bootstrap/platform/product_application.py
```

### Key Examples
| Example | Path | Description |
|---------|------|-------------|
| Standard Project Scaffold | `examples/sample_standard_app/` | Template for new projects |
| PEER Multi-Agent | `examples/sample_apps/peer_agent_app/` | PEER pattern financial event analysis |
| RAG Agent | `examples/sample_apps/rag_app/` | Knowledge base + RAG |
| ReAct Agent | `examples/sample_apps/react_agent_app/` | ReAct pattern agent |
| Discussion Group | `examples/sample_apps/discussion_group_app/` | Multi-turn multi-agent discussion |
| Simple QA Agent | `examples/sample_apps/simple_qa_agent_app/` | Minimal single-agent example |
| Workflow Agent | `examples/sample_apps/workflow_agent_app/` | Visual workflow platform |

## Work Pattern Details

### PEER (`peer_work_pattern.py:17`)
Four agent roles: `planning` (PlanningAgentTemplate) → `executing` (ExecutingAgentTemplate) → `expressing` (ExpressingAgentTemplate) → `reviewing` (ReviewingAgentTemplate). Parameters: `retry_count`, `jump_step` (skip to a specific phase), `eval_threshold`. The reviewing agent assigns a score; if below threshold, the loop repeats from the specified `jump_step`.

### GRR (`grr_work_pattern.py:15`)
Three agent roles: `generating` → `reviewing` → `rewriting`. Parameters: `retry_count` (default 2), `eval_threshold` (default 60). Iterates until review score meets threshold.

### IS (`is_work_pattern.py:14`)
Two agent roles: `implementation` + `supervision`. Parameters: `checkpoint_count` (default 3), `max_corrections` (default 2). Execution happens at checkpoints; supervision monitors alignment and triggers corrections.

### DOE (Data-fining/Opinion-inject/Express)
Referenced in README/docs but not yet a standalone work pattern class in the codebase. Described as 3 agents for data-intensive tasks: Data-fining (precision computation), Opinion-inject (merge data + expert opinions), Express (format output).

## Conventions
- Config files use `${VAR_NAME}` placeholder syntax resolved from `custom_key.toml`
- Components identified by `{appname}.{component_type}.{name}` instance codes
- All managers are singletons (`@singleton` decorator)
- Agents support both `run()` (sync) and `async_run()` (async)
- YAML config `metadata` block is required for component discovery
- Use `api_key: '${ENV_VAR}'` pattern in YAML; actual keys in `custom_key.toml` (not committed)
