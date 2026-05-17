# 03 — 框架启动流程与组件系统

> **前置要求：** 完成 [02-agent-fundamentals.md](02-agent-fundamentals.md)
> **学习目标：** 理解 `AgentUniverse().start()` 的完整启动链路、组件注册机制、单例管理器模式
> **预计时间：** 2-3 天

---

## 1. 先看全貌：start() 的 10 个步骤

每次你运行一个 agentUniverse 应用，入口总是这行代码：

```python
AgentUniverse().start()
```

这一行背后发生了什么？10 个大步骤：

```
AgentUniverse().start()
│
├── 步骤 1：显示 Banner
│
├── 步骤 2：加载 config.toml（主配置文件）
│
├── 步骤 3：加载 custom_key.toml（API Keys，优先加载以解析 ${VAR} 占位符）
│
├── 步骤 4：重新加载 config.toml（这次 ${VAR} 能被正确解析）
│
├── 步骤 5：初始化日志系统（loguru）
├── 步骤 6：初始化 OpenTelemetry（可观测性）
├── 步骤 7：初始化 gRPC / Gunicorn（如果配置了）
├── 步骤 8：加载扩展模块（Extension Modules）
├── 步骤 9：开启 Monitor
│
└── 步骤 10：★ 扫描并注册所有组件 ★  ← 本章重点
    ├── scan()：扫描所有 YAML 文件
    └── register()：实例化并登记到对应的 Manager
```

**用 Node.js 的思维理解：** 这整个流程类似于：

```javascript
// 伪 Node.js 对比
async function start() {
    console.log(banner);                    // 1. Banner
    const config = loadConfig('config.toml'); // 2-4. 加载配置
    initLogger(config.log);                  // 5. 日志
    initTelemetry(config.otel);              // 6. 可观测性
    initHttpServer(config.server);           // 7. HTTP 服务
    loadExtensions(config.extensions);       // 8. 扩展
    startMonitor(config.monitor);            // 9. 监控
    await scanAndRegisterComponents();       // 10. ★ 扫描注册
}
```

---

## 2. 配置文件系统

### 2.1 双层配置架构

agentUniverse 有两类配置文件：**TOML** 和 **YAML**。

| 文件类型 | 格式 | 用途 | 何处指定 |
|---------|------|------|---------|
| 框架配置 | TOML | 应用名、扫描路径、服务端口、数据库等 | `config.toml` |
| API Keys | TOML | 各 LLM 厂商的密钥 | `custom_key.toml` |
| 组件配置 | YAML | 每个 Agent/LLM/Tool/Prompt 的具体参数 | 各自目录下的 `.yaml` |

```
项目根目录/
├── config/
│   ├── config.toml           ← 框架配置（TOML）
│   ├── custom_key.toml       ← API 密钥（TOML，不提交 git）
│   └── log_config.toml       ← 日志配置（TOML）
│
└── intelligence/agentic/     ← 组件配置（YAML）
    ├── agent/
    │   └── agent_instance/
    │       └── simple_qa_agent.yaml
    ├── llm/
    │   └── qwen_llm.yaml
    └── prompt/
        └── simple_qa_prompt.yaml
```

### 2.2 config.toml 结构详解

```toml
# 来自 simple_qa_agent_app/config/config.toml

[BASE_INFO]
appname = 'simple_qa_agent_app'          # ★ 应用名，很重要

[CORE_PACKAGE]
# ★ 核心：告诉框架去哪里扫描组件
default = ['simple_qa_agent_app.intelligence.agentic']  # 默认扫描路径
agent = ['simple_qa_agent_app.intelligence.agentic.agent']  # Agent 专用路径
llm = ['simple_qa_agent_app.intelligence.agentic.llm']      # LLM 专用路径
prompt = ['simple_qa_agent_app.intelligence.agentic.prompt'] # Prompt 专用路径
knowledge = []
tool = []
memory = []

[SUB_CONFIG_PATH]
log_config_path = './log_config.toml'
custom_key_path = './custom_key.toml'

[DB]
system_db_uri = ''             # 空 = 自动创建本地 SQLite

[GUNICORN]
activate = 'false'             # 不启用 Gunicorn

[GRPC]
activate = 'false'             # 不启用 gRPC
```

**`CORE_PACKAGE.default` vs 各类型专属路径的关系：**

- `default`：**兜底路径**。如果某个类型的专属路径为空，就用 default。
- 各类型的专属路径（`agent`, `llm`, `tool`...）：**覆盖 default**。如果配置了，就用这个。

每种组件类型的最终扫描路径 = （该类型的专属路径 或 default）+ 系统内建路径。

### 2.3 custom_key.toml：密钥管理

```toml
# custom_key.toml 示例
[KEY_LIST]
DASHSCOPE_API_KEY = 'sk-xxxxxxxxxxxxxxxx'
OPENAI_API_KEY = 'sk-yyyyyyyyyyyyyyyy'
ANTHROPIC_API_KEY = 'sk-ant-zzzzzzzzzzzzzz'
```

在 YAML 中通过 `${变量名}` 引用：

```yaml
# llm/qwen_llm.yaml
api_key: '${DASHSCOPE_API_KEY}'    # 框架启动时会被替换为实际值
```

**注意：** `custom_key.toml` 不应该提交到 git。查看 `.gitignore` 确认。

### 2.4 为什么 config.toml 要加载两次？

阅读源码 `agentuniverse.py:78-88`：

```python
# agentuniverse/base/agentuniverse.py:64-90（简化）
def start(self, config_path=None, core_mode=False):
    # 第一次加载 config.toml
    configer = Configer(path=config_path).load()

    # ★ 立即加载 custom_key.toml（为了解析 ${VAR} 占位符）
    custom_key_configer_path = ...   # 从 config.toml 读取 custom_key_path
    CustomKeyConfiger(custom_key_configer_path)

    # ★ 重新加载 config.toml —— 这次 ${DASHSCOPE_API_KEY} 等变量可被解析
    configer = Configer(path=config_path).load()
    app_configer = AppConfiger().load_by_configer(configer)
```

**为什么？** 因为 `config.toml` 里的某些字段可能也包含 `${变量名}` 引用（尽管不常见）。先加载密钥，再重新解析配置，保证所有占位符都能被替换。

**对比全栈经验：** 这类似于先加载 `.env` 文件，再读取 `process.env`。

---

## 3. 组件扫描：从 Python 包到 YAML 文件

### 3.1 扫描流程

`__scan_and_register()` 是启动过程中最核心的方法（`agentuniverse.py:143`）。

```
__scan_and_register(app_configer)
│
├── 1. 为每种组件类型确定扫描路径列表
│     component_package_map = {
│       ComponentEnum.AGENT:    [用户路径] + [系统内建路径],
│       ComponentEnum.LLM:      [用户路径] + [系统内建路径],
│       ComponentEnum.TOOL:     [用户路径] + [系统内建路径],
│       ...
│     }
│
├── 2. 对每种组件类型，逐个路径 scan()
│     └── 遍历包路径 → 查找所有 *.yaml → 解析 → 生成 ComponentConfiger 列表
│
└── 3. 对每种组件类型，调用 __register()
      └── 遍历 ComponentConfiger 列表 → 实例化 → 注册到 Manager
```

### 3.2 scan() 方法详解

```python
# agentuniverse/base/agentuniverse.py:225-270（简化）
def scan(self, package_list, config_type_enum, component_enum):
    component_configer_list = []

    for package_name in package_list:
        # ① 将 Python 包名转为文件系统路径
        #    例如 'simple_qa_agent_app.intelligence.agentic.agent'
        #    → /path/to/project/intelligence/agentic/agent/
        package_path = self.__package_name_to_path(package_name)
        path = Path(package_path)

        # ② 递归查找该目录下所有 .yaml 文件
        config_files = path.rglob('*.yaml')

        for config_file in config_files:
            # ③ 将 YAML 文件加载为 Configer 对象
            configer = Configer(path=config_file_str).load()

            # ④ 将 Configer 转为 ComponentConfiger 对象
            component_configer = ComponentConfiger().load_by_configer(configer)

            # ⑤ 检查 metadata.type 是否匹配
            if component_configer.get_component_config_type() == component_enum.value:
                component_configer_list.append(component_configer)

    return component_configer_list
```

**这 5 个步骤的本质：**

| 步骤 | 做什么 | 关键方法 |
|------|--------|---------|
| ① | Python 包名 → 文件路径 | `importlib.util.find_spec()` |
| ② | 递归找 `.yaml` 文件 | `Path.rglob('*.yaml')` |
| ③ | YAML → Configer 对象 | `Configer.load()` |
| ④ | Configer → ComponentConfiger | `load_by_configer()` |
| ⑤ | 按 `metadata.type` 过滤 | `get_component_config_type()` |

### 3.3 关键理解：metadata 块的作用

每个 YAML 文件的 `metadata` 块是框架识别组件的"身份证"：

```yaml
metadata:
  type: 'AGENT'                                            # ① 什么类型？
  module: 'agentuniverse.agent.template.react_agent_template'  # ② Python 模块路径
  class: 'ReActAgentTemplate'                               # ③ 对应哪个 Python 类
```

- `type` 决定这个组件归哪个 Manager 管（AgentManager? LLMManager? ToolManager?）
- `module` + `class` 决定实例化时使用哪个 Python 类

---

## 4. 组件注册：从 YAML 到可用的实例

### 4.1 __register() 方法详解

```python
# agentuniverse/base/agentuniverse.py:272-336（简化版）
def __register(self, component_enum, component_configer_list):
    # 1. 获取该组件类型对应的 Manager 类
    component_manager_clz = ComponentConfigerUtil.get_component_manager_clz_by_type(component_enum)

    for component_configer in component_configer_list:
        # 2. 获取该组件类型对应的 Configer 类并加载
        configer_clz = ComponentConfigerUtil.get_component_config_clz_by_type(component_enum)
        configer_instance = configer_clz().load_by_configer(component_configer.configer)

        # 3. 获取该配置对应的 Python 类
        #    根据 metadata.module + metadata.class 动态导入
        component_clz = ComponentConfigerUtil.get_component_object_clz_by_component_configer(configer_instance)

        # 4. 实例化
        component_instance = component_clz().initialize_by_component_configer(configer_instance)

        # 5. 注册到 Manager 的实例池
        component_manager_clz().register(
            component_instance.get_instance_code(),   # 如 'simple_qa_agent_app.agent.simple_qa_agent'
            component_instance                        # 实例对象
        )
```

### 4.2 组件实例编码（Instance Code）公式

每个注册的组件都有一个**全局唯一标识**：

```python
# agentuniverse/base/component/component_base.py:29-32
def get_instance_code(self) -> str:
    appname = ApplicationConfigManager().app_configer.base_info_appname
    return f'{appname}.{self.component_type.value.lower()}.{self.name}'
```

**格式：** `{appname}.{type}.{name}`

**实际例子：**

| 组件 | App Name | Type | Name | Instance Code |
|------|----------|------|------|---------------|
| simple_qa_agent | `simple_qa_agent_app` | `AGENT` | `simple_qa_agent` | `simple_qa_agent_app.agent.simple_qa_agent` |
| qwen_llm | `simple_qa_agent_app` | `LLM` | `qwen_llm` | `simple_qa_agent_app.llm.qwen_llm` |
| simple_qa_prompt | `simple_qa_agent_app` | `PROMPT` | `simple_qa_prompt` | `simple_qa_agent_app.prompt.simple_qa_prompt` |

**这个编码就是组件的"服务名"——** 在代码中通过 `AgentManager().get_instance_obj('simple_qa_agent')` 获取实例时，内部就是按这个 Instance Code 查找。

---

## 5. 单例模式与 Manager 系统

### 5.1 @singleton 装饰器回顾

```python
# agentuniverse/base/annotation/singleton.py
def singleton(cls):
    instances = {}

    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance
```

**@singleton 做了什么？** 把类变成了一个**工厂函数**——第一次调用时创建实例，之后永远返回同一个实例。

```python
@singleton
class AgentManager:
    def __init__(self):
        self.pool = {}

# 使用
mgr1 = AgentManager()   # 第一次：创建新实例
mgr2 = AgentManager()   # 之后：返回同一个实例
assert mgr1 is mgr2     # True —— 是同一个对象
```

**对比 JavaScript：**

```javascript
// Node.js 模块级单例（常见模式）
class AgentManager {
    constructor() { this.pool = {}; }
}
// 模块导出实例，不是类
module.exports = new AgentManager();
// 所有 import 这个模块的地方都拿到同一个实例
```

### 5.2 ComponentManagerBase：统一的实例池

```python
# agentuniverse/base/component/component_manager_base.py:21-75（简化）
class ComponentManagerBase:
    def __init__(self, component_type: ComponentEnum):
        # ★ 核心数据结构：实例池
        self._instance_obj_map: dict[str, ComponentTypeVar] = {}
        self._component_type = component_type

    def register(self, component_instance_name: str, component_instance_obj):
        """注册组件：名字 → 实例"""
        if component_instance_name in self._instance_obj_map:
            # 如果已存在，优先保留用户配置的（覆盖系统内建的）
            if is_system_builtin(component_instance_obj):
                return   # 不覆盖用户的配置
            return       # 不重复注册
        self._instance_obj_map[component_instance_name] = component_instance_obj

    def get_instance_obj(self, component_instance_name: str, new_instance=True):
        """获取组件：根据名字返回实例"""
        appname = ApplicationConfigManager().app_configer.base_info_appname
        instance_code = f'{appname}.{self._component_type.value.lower()}.{component_instance_name}'
        instance = self._instance_obj_map.get(instance_code)
        if new_instance and instance:
            return instance.create_copy()    # ★ 返回深拷贝，不污染原始实例
        return instance
```

### 5.3 具体 Manager 一览

每种 ComponentEnum 类型都有对应的 Manager：

```python
# Agent 类型 → AgentManager
@singleton
class AgentManager(ComponentManagerBase):
    def __init__(self):
        super().__init__(ComponentEnum.AGENT)
```

所有 Manager 列表：

| Manager | 管理的组件类型 | 文件 |
|---------|-------------|------|
| `AgentManager` | AGENT | `agent/agent_manager.py` |
| `LLMManager` | LLM | `llm/llm_manager.py` |
| `ToolManager` | TOOL | `agent/action/tool/tool_manager.py` |
| `PlannerManager` | PLANNER | `agent/plan/planner/planner_manager.py` |
| `MemoryManager` | MEMORY | `agent/memory/memory_manager.py` |
| `PromptManager` | PROMPT | `prompt/prompt_manager.py` |
| `KnowledgeManager` | KNOWLEDGE | `agent/action/knowledge/knowledge_manager.py` |
| `StoreManager` | STORE | `agent/action/knowledge/store/store_manager.py` |
| `EmbeddingManager` | EMBEDDING | `agent/action/knowledge/embedding/embedding_manager.py` |
| `WorkPatternManager` | WORK_PATTERN | `agent/work_pattern/work_pattern_manager.py` |
| ... | ... | ... |

**所有的 Manager 都是 @singleton**，全局只有一个实例，内部维护一个 `{instance_code: instance}` 字典。

### 5.4 获取组件的标准调用方式

```python
from agentuniverse.agent.agent_manager import AgentManager

# 按名称获取 Agent
agent = AgentManager().get_instance_obj('simple_qa_agent')
agent.run(input='hello')

# 内部发生了什么：
# 1. AgentManager() 返回单例
# 2. get_instance_obj('simple_qa_agent') 拼接 instance_code
#    → 'simple_qa_agent_app.agent.simple_qa_agent'
# 3. 在 _instance_obj_map 中查找
# 4. 返回 create_copy()（深拷贝，线程安全）
```

---

## 6. 用户配置 vs 系统内建的覆盖规则

### 6.1 两套扫描路径

```
组件扫描路径 = 用户路径（来自 config.toml CORE_PACKAGE）
              + 系统内建路径（来自 AgentUniverse.__system_default_*_package）
```

```python
# agentuniverse/base/agentuniverse.py:149-150
# Agent 的扫描路径 = 用户路径 + 系统内建路径
core_agent_package_list = (
    (app_configer.core_agent_package_list or app_configer.core_default_package_list)
    + self.__system_default_agent_package  # ['agentuniverse.agent.default']
)
```

### 6.2 覆盖规则

如果用户配置的组件和系统内建组件 **同名（相同的 instance_code）**：

```python
# component_manager_base.py:33-37
if component_instance_name in self._instance_obj_map.keys():
    if is_system_builtin(component_instance_obj):
        # 这是系统内建组件，实例池里已经有了同名组件（用户的）
        # → 跳过，保留用户的
        return
    # 这不是系统内建组件，但实例池里已经有了（重复注册）
    # → 警告并跳过
    LOGGER.warn(f"'{component_instance_name}' already exists.")
    return
```

**优先级：用户配置 > 系统内建**

---

## 7. 完整启动链路串讲

现在把所有碎片串起来，跟着一条完整的启动链路走一遍：

```
┌──────────────────────────────────────────────────────────────────┐
│ 启动入口                                                          │
│ AgentUniverse().start()                                           │
│ agentuniverse/base/agentuniverse.py                               │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 配置加载阶段                                                       │
│                                                                    │
│ Configer('config.toml').load()         ← 第 1 次加载               │
│ CustomKeyConfiger('custom_key.toml')   ← 解析 API Key              │
│ Configer('config.toml').load()         ← 第 2 次加载（能解析${}了）│
│ AppConfiger().load_by_configer()       ← 应用配置                  │
│ init_loggers()                          ← 日志初始化               │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ __scan_and_register(app_configer)                                 │
│                                                                    │
│ 为每种组件类型准备扫描路径:                                         │
│   AGENT  → ['simple_qa_agent_app.intelligence.agentic.agent',     │
│              'agentuniverse.agent.default']                       │
│   LLM    → ['simple_qa_agent_app.intelligence.agentic.llm',       │
│              'agentuniverse.llm.default']                         │
│   PROMPT → ['simple_qa_agent_app.intelligence.agentic.prompt',    │
│              'agentuniverse.agent', 'agentuniverse.base.util']    │
│   ...                                                              │
│                                                                    │
│ 对每种类型：                                                        │
│   ┌──────────────────────────────────────────────┐                │
│   │ scan() - 遍历所有路径，找到所有 .yaml 文件     │                │
│   │   path.rglob('*.yaml')                        │                │
│   │   → 解析每个 YAML → ComponentConfiger          │                │
│   │   → 按 metadata.type 过滤                     │                │
│   └──────────────────┬───────────────────────────┘                │
│                      │                                             │
│                      ▼                                             │
│   ┌──────────────────────────────────────────────┐                │
│   │ __register() - 实例化并登记到 Manager          │                │
│   │                                                │                │
│   │ 对于每个 ComponentConfiger:                    │                │
│   │  ① 找到对应的 Configer 类 → 加载配置            │                │
│   │  ② 找到对应的 Python 类 → 实例化                │                │
│   │  ③ 调用 initialize_by_component_configer()     │                │
│   │  ④ 生成 instance_code                          │                │
│   │  ⑤ 注册到 Manager 的 _instance_obj_map          │                │
│   └──────────────────────────────────────────────┘                │
└──────────────────────────────────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│ 结果：所有 Manager 的实例池已填充完毕                               │
│                                                                    │
│ AgentManager._instance_obj_map = {                                │
│   'simple_qa_agent_app.agent.simple_qa_agent': <Agent 实例>,      │
│ }                                                                  │
│ LLMManager._instance_obj_map = {                                  │
│   'simple_qa_agent_app.llm.qwen_llm': <LLM 实例>,                 │
│ }                                                                  │
│ PromptManager._instance_obj_map = {                                │
│   'simple_qa_agent_app.prompt.simple_qa_prompt': <Prompt 实例>,   │
│ }                                                                  │
│                                                                    │
│ 现在随时可以通过 Manager 获取任何组件：                              │
│   AgentManager().get_instance_obj('simple_qa_agent').run(...)      │
└──────────────────────────────────────────────────────────────────┘
```

---

## 8. 动手练习

### 练习 1：跟踪 startup 源码

打开 `agentuniverse/base/agentuniverse.py`，找到 `start()` 方法。在每一步旁边标注它的作用。

### 练习 2：画图

画出以下关系图：
- `AgentUniverse` → `__scan_and_register()` → `scan()` → `__register()`
- `__register()` → `ComponentManagerBase.register()` → `_instance_obj_map`
- `AgentManager.get_instance_obj()` → `get_instance_code()` → `instance_code 公式`

### 练习 3：验证你的理解

```bash
cd examples/sample_apps/simple_qa_agent_app

# 1. 找一个 YAML 文件，看它的 metadata 块
cat intelligence/agentic/agent/agent_instance/simple_qa_agent.yaml | grep -A 5 metadata

# 2. 推算出这个组件的 instance_code 是什么？
#    提示：appname 在 config/config.toml 的 [BASE_INFO] 里

# 3. 在 config/config.toml 中，找到 CORE_PACKAGE.agent 的值
#    这个路径下的所有 .yaml 文件会被扫描
```

### 练习 4：阅读 Manager 源码

1. 阅读 `agentuniverse/base/component/component_manager_base.py`，重点关注 `register()` 和 `get_instance_obj()` 方法
2. 阅读 `agentuniverse/agent/agent_manager.py`，理解它继承 `ComponentManagerBase` 后做了什么
3. 思考：为什么 `get_instance_obj()` 默认返回 `create_copy()` 而不是原始实例？这和线程安全有什么关系？

### 练习 5：假设题

假设你新建了一个 Agent YAML 文件：

```yaml
# intelligence/agentic/agent/agent_instance/my_agent.yaml
info:
  name: 'my_custom_agent'
metadata:
  type: 'AGENT'
  module: 'agentuniverse.agent.template.react_agent_template'
  class: 'ReActAgentTemplate'
```

但 `config.toml` 中的 `CORE_PACKAGE.agent` 配置的是另一个路径。启动后能否通过 `AgentManager().get_instance_obj('my_custom_agent')` 获取到它？为什么？

---

## 9. 概念速查表

| 概念 | 含义 | 关键文件 |
|------|------|---------|
| **AgentUniverse** | 框架启动入口 | `base/agentuniverse.py` |
| **start()** | 启动方法，执行 10 个步骤 | `agentuniverse.py:64` |
| **Configer** | TOML/YAML 文件加载器 | `base/config/configer.py` |
| **AppConfiger** | 应用级配置容器 | `base/config/application_configer/` |
| **CustomKeyConfiger** | API Key 解析器 | `base/config/custom_configer/` |
| **ComponentEnum** | 22 种组件类型的枚举 | `base/component/component_enum.py` |
| **ComponentConfiger** | 组件配置的通用容器 | `base/config/component_configer/` |
| **ComponentManagerBase** | 所有 Manager 的基类 | `base/component/component_manager_base.py` |
| **ComponentConfigerUtil** | 根据 ComponentEnum 查找对应的 Configer/Manager/Python 类 | `base/component/component_configer_util.py` |
| **@singleton** | 简易单例装饰器 | `base/annotation/singleton.py` |
| **instance_code** | `{appname}.{type}.{name}` 唯一标识 | `component_base.py:29` |
| **_instance_obj_map** | Manager 内部的实例池（dict） | `component_manager_base.py:28` |
| **CORE_PACKAGE** | 扫描路径配置段 | `config.toml` |
| **metadata** | YAML 中的组件身份证 | 每个 `.yaml` 文件底部 |

---

## 下一步

你已经理解了：
- `AgentUniverse().start()` 的 10 步启动流程
- `scan()` + `__register()` 如何将 YAML 配置变成内存中的实例
- `{appname}.{type}.{name}` 的 Instance Code 公式
- `@singleton` + `ComponentManagerBase` 构建的统一组件注册框架

下一步进入 **04-prompt-engineering.md**，学习 Prompt 系统——提示词是如何设计、组装、版本化管理的，以及它如何影响 Agent 的行为。
