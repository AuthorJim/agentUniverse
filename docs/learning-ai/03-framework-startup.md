# 03 — 框架启动：从一个 `start()` 看整个宇宙的诞生

> **前置要求：** 完成 [02-agent-fundamentals.md](02-agent-fundamentals.md)
> **学习目标：** 理解 `AgentUniverse().start()` 的完整 10 步启动链路、YAML 扫描机制、组件注册原理、单例 Manager 系统
> **预计时间：** 2-3 天

---

## 1. 类比先行：`start()` 到底在做什么？

每次运行 agentUniverse 应用，入口总是这行代码：

```python
AgentUniverse().start()
```

如果你是 Node.js 工程师，这行代码等价于一次性完成以下所有初始化：

```javascript
// 伪 Node.js 对照 —— 这一行 Python 等于下面所有这些
async function bootstrap() {
    console.log(banner);                           // 1. 打 Logo
    require('dotenv').config();                    // 2-4. 加载配置 + 环境变量
    winston.configure(require('./log_config'));    // 5. 初始化日志
    await initOpenTelemetry();                     // 6. 初始化可观测性
    await startHttpServer();                       // 7. 启动 HTTP 服务
    await loadExtensions();                        // 8. 加载扩展/插件
    startMonitor();                                // 9. 启动监控
    await scanAndRegisterAllComponents();          // 10. ★ 扫描所有配置，注册所有组件
}
```

**`start()` 就是一个把所有初始化逻辑打包好的 `bootstrap()` 函数。** 下面我们拆开每个步骤。

---

## 2. `start()` 的 10 个步骤

```python
# agentuniverse/base/agentuniverse.py:64-90 (简化)
def start(self, config_path=None, core_mode=False):
    """框架启动入口"""

    # 步骤 1: 显示 Banner（就是那个 ASCII Art 的 agentUniverse logo）
    self.__banner()

    # 步骤 2: 第 1 次加载 config.toml（主配置，但 ${VAR} 占位符还没解析）
    configer = Configer(path=config_path).load()

    # 步骤 3: 加载 custom_key.toml（API Keys）
    custom_key_path = configer.value.get('SUB_CONFIG_PATH', {}).get('custom_key_path')
    CustomKeyConfiger(path=custom_key_path)

    # 步骤 4: ★ 重新加载 config.toml —— 这次 ${VAR} 能被正确解析了
    configer = Configer(path=config_path).load()
    app_configer = AppConfiger().load_by_configer(configer)

    # 步骤 5: 初始化日志系统（基于 log_config.toml）
    self.__init_loggers(app_configer)

    # 步骤 6: 初始化 OpenTelemetry（可观测性）
    self.__init_otel(app_configer)

    # 步骤 7: 初始化数据库连接、gRPC、Gunicorn（如果配置了）
    self.__init_system(app_configer, core_mode)

    # 步骤 8: 加载扩展模块（ConfigExtension / YamlFuncExtension）
    self.__init_extension_modules(app_configer)

    # 步骤 9: 启动监控模块
    self.__init_monitor(app_configer)

    # 步骤 10: ★★★ 扫描并注册所有组件（本章最核心的部分）
    self.__scan_and_register(app_configer)
```

### 2.1 为什么 config.toml 要加载两次？

注意步骤 2 和步骤 4——同一个文件加载了两遍。这是 agentUniverse 的关键设计：

```
第 1 次加载 config.toml:
  目的: 找到 custom_key.toml 的路径
  此时: ${DASHSCOPE_API_KEY} 这种占位符还是原始字符串，无法解析

加载 custom_key.toml:
  解析出: DASHSCOPE_API_KEY = 'sk-xxxxxx'

第 2 次加载 config.toml:
  现在: ${DASHSCOPE_API_KEY} → 'sk-xxxxxx' 能正确替换了
  所有配置项都得到最终值
```

**Node.js 对照：** 这就像先 `require('dotenv').config()` 加载 `.env`，然后才能通过 `process.env.VAR` 访问环境变量。顺序不能错。

---

## 3. 配置文件系统：两层架构

agentUniverse 有两类配置文件，格式不同、用途不同：

### 3.1 TOML（框架级配置）

TOML 文件控制框架自身的行为——就像 `config/default.json` + `.env`：

```
config/
├── config.toml           # 主配置（appname、扫描路径、服务开关）
├── custom_key.toml       # API Keys（≈ .env，绝对不能提交 git！）
└── log_config.toml       # 日志配置（≈ winston config）
```

### 3.2 YAML（组件级配置）

YAML 文件定义每个具体的 Agent、LLM、Tool、Prompt 的参数：

```
intelligence/agentic/
├── agent/agent_instance/
│   └── simple_qa_agent.yaml    # Agent 配置
├── llm/
│   └── qwen_llm.yaml           # LLM 配置
└── prompt/
    └── simple_qa_prompt.yaml   # Prompt 配置
```

### 3.3 config.toml 结构详解

```toml
# 来自 simple_qa_agent_app/config/config.toml

[BASE_INFO]
appname = 'simple_qa_agent_app'          # ★ 应用名，很重要（instance_code 的前缀）

[CORE_PACKAGE]
# ★★★ 最关键的一段：告诉框架去哪里扫描组件 YAML
default = ['simple_qa_agent_app.intelligence.agentic']  # 默认扫描路径
agent = ['simple_qa_agent_app.intelligence.agentic.agent']  # Agent 专用路径
llm = ['simple_qa_agent_app.intelligence.agentic.llm']      # LLM 专用路径
prompt = ['simple_qa_agent_app.intelligence.agentic.prompt'] # Prompt 专用路径
knowledge = []
tool = []
memory = []

[SUB_CONFIG_PATH]
custom_key_path = './custom_key.toml'    # API Key 文件路径
log_config_path = './log_config.toml'    # 日志配置路径

[DB]
system_db_uri = ''             # 空 = 自动创建本地 SQLite

[GUNICORN]
activate = 'false'             # 开发期间不启用 Gunicorn（类似不用 PM2）

[GRPC]
activate = 'false'             # 不启用 gRPC

[MONITOR]
activate = 'false'             # 不启用监控

[EXTENSION_MODULES]
class_list = ['config.config_extension.ConfigExtension']  # 扩展类列表
```

**`CORE_PACKAGE` 的路径覆盖规则：**
- `default` 是**兜底路径**：某类型的专属路径为空时，用 default
- 各类型的专属路径（`agent`, `llm`, `tool`...）**覆盖** default
- 最终扫描路径 = 用户配置（专属路径 or default）+ 系统内建路径

### 3.4 custom_key.toml 和 ${VAR} 占位符

```toml
# custom_key.toml
[KEY_LIST]
DASHSCOPE_API_KEY = 'sk-xxxxxxxxxxxxxxxx'
OPENAI_API_KEY = 'sk-yyyyyyyyyyyyyyyy'
```

在 YAML 中引用：
```yaml
# llm/qwen_llm.yaml
api_key: '${DASHSCOPE_API_KEY}'    # 启动时被替换为实际值
```

这种模式和 Node.js 中 `process.env.VAR` 本质相同，区别在于 agentUniverse 做的是**启动时的字符串替换**而非运行时读取。

---

## 4. 组件扫描：从 Python 包路径到内存实例

### 4.1 `__scan_and_register()` 总览

这是 `start()` 中最核心的方法：

```
__scan_and_register(app_configer)
│
├── 1. 为每种组件类型确定扫描路径列表
│     AGENT  → [用户配置路径] + [agentuniverse.agent.default]
│     LLM    → [用户配置路径] + [agentuniverse.llm.default]
│     TOOL   → [用户配置路径] + [agentuniverse.agent.action.tool.default]
│     ... (共 22 种组件类型)
│
├── 2. 对每种组件类型 → scan() → 遍历路径找 *.yaml
│
└── 3. 对每种组件类型 → __register() → 实例化 → 存到 Manager
```

### 4.2 scan() 方法的 5 个步骤

```python
# agentuniverse/base/agentuniverse.py:225-270 (简化)
def scan(self, package_list, config_type_enum, component_enum):
    component_configer_list = []

    for package_name in package_list:
        # ① 将 Python 包名转为文件系统路径
        #    'simple_qa_agent_app.intelligence.agentic.agent'
        #    → /path/to/project/intelligence/agentic/agent/
        package_path = self.__package_name_to_path(package_name)

        # ② 递归查找该目录下所有 .yaml 文件
        for config_file in Path(package_path).rglob('*.yaml'):

            # ③ 将 YAML 文件加载为 Configer 对象
            configer = Configer(path=str(config_file)).load()

            # ④ 将 Configer 转为 ComponentConfiger 对象
            component_configer = ComponentConfiger().load_by_configer(configer)

            # ⑤ 检查 metadata.type 是否匹配目标类型
            if component_configer.get_component_config_type() == component_enum.value:
                component_configer_list.append(component_configer)

    return component_configer_list
```

| 步骤 | 操作 | 输入 | 输出 |
|------|------|------|------|
| ① | 包名→文件路径 | `'xxx.intelligence.agentic.agent'` | `/path/to/project/intelligence/agentic/agent/` |
| ② | 递归找文件 | 目录路径 | `.yaml` 文件列表 |
| ③ | YAML→对象 | YAML 文本 | Configer 对象 |
| ④ | Configer→ComponentConfiger | Configer | ComponentConfiger（含 metadata） |
| ⑤ | 类型过滤 | ComponentConfiger | 只保留匹配类型的 |

### 4.3 metadata 块：组件的身份证

每个 YAML 文件的 `metadata` 块是框架识别组件的唯一依据：

```yaml
metadata:
  type: 'AGENT'                                               # 组件类型
  module: 'agentuniverse.agent.template.react_agent_template' # Python 模块路径
  class: 'ReActAgentTemplate'                                 # 对应的 Python 类名
```

- `type` → 决定这个组件归哪个 Manager 管（AgentManager? LLMManager?）
- `module` + `class` → 决定实例化时使用哪个 Python 类

---

## 5. 组件注册：从 YAML 到 `_instance_obj_map`

### 5.1 `__register()` 方法

```python
# agentuniverse/base/agentuniverse.py:272-336 (简化)
def __register(self, component_enum, component_configer_list):
    # 1. 获取该组件类型对应的 Manager 类
    manager_clz = ComponentConfigerUtil.get_component_manager_clz_by_type(component_enum)

    for component_configer in component_configer_list:
        # 2. 获取该组件类型对应的 Configer 类并加载
        configer_clz = ComponentConfigerUtil.get_component_config_clz_by_type(component_enum)
        configer_instance = configer_clz().load_by_configer(component_configer.configer)

        # 3. 根据 metadata.module + metadata.class 动态导入 Python 类
        component_clz = ComponentConfigerUtil.get_component_object_clz_by_component_configer(
            configer_instance
        )

        # 4. 实例化
        component_instance = component_clz().initialize_by_component_configer(configer_instance)

        # 5. 注册到 Manager 的实例池
        manager_instance = manager_clz()
        manager_instance.register(
            component_instance.get_instance_code(),  # key
            component_instance                      # value
        )
```

### 5.2 Instance Code 公式

每个注册的组件都有一个**全局唯一标识**：

```
{appname}.{component_type}.{name}
```

实际例子：

| 组件 | appname | type | name | Instance Code |
|------|---------|------|------|---------------|
| simple_qa_agent | `simple_qa_agent_app` | `AGENT` | `simple_qa_agent` | `simple_qa_agent_app.agent.simple_qa_agent` |
| qwen_llm | `simple_qa_agent_app` | `LLM` | `qwen_llm` | `simple_qa_agent_app.llm.qwen_llm` |
| simple_qa_prompt | `simple_qa_agent_app` | `PROMPT` | `simple_qa_prompt` | `simple_qa_agent_app.prompt.simple_qa_prompt` |

当你调用 `AgentManager().get_instance_obj('simple_qa_agent')` 时，框架内部就在做这个拼接 + 查找。

---

## 6. 单例 Manager 系统

### 6.1 `@singleton` 装饰器

agentUniverse 中所有 Manager 类都用 `@singleton` 装饰：

```python
# agentuniverse/base/annotation/singleton.py (简化)
def singleton(cls):
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

# 使用
@singleton
class AgentManager(ComponentManagerBase):
    def __init__(self):
        super().__init__(ComponentEnum.AGENT)
```

效果：`AgentManager()` 无论调用多少次，返回的都是同一个实例。内部维护一个 `_instance_obj_map` 字典作为所有 Agent 组件的注册表。

### 6.2 ComponentManagerBase：统一的实例池

```python
# agentuniverse/base/component/component_manager_base.py:21-75 (简化)
class ComponentManagerBase:
    def __init__(self, component_type: ComponentEnum):
        self._instance_obj_map: dict[str, ComponentTypeVar] = {}  # ★ 核心数据结构
        self._component_type = component_type

    def register(self, component_instance_name: str, component_instance_obj):
        """注册：名字 → 实例"""
        if component_instance_name in self._instance_obj_map:
            # 已存在 → 用户配置优先于系统内建
            if is_system_builtin(component_instance_obj):
                return   # 不覆盖用户配置
            return         # 不重复注册
        self._instance_obj_map[component_instance_name] = component_instance_obj

    def get_instance_obj(self, component_instance_name: str, new_instance=True):
        """获取：根据名字返回实例"""
        appname = ApplicationConfigManager().app_configer.base_info_appname
        instance_code = f'{appname}.{self._component_type.value.lower()}.{component_instance_name}'
        instance = self._instance_obj_map.get(instance_code)
        if new_instance and instance:
            return instance.create_copy()    # ★ 返回深拷贝，不污染原始实例！
        return instance
```

**为什么 `get_instance_obj()` 默认返回 `create_copy()`？**

线程安全。如果多个请求同时获取同一个 Agent，它们拿到的是不同的实例副本，各自修改不会互相影响。

### 6.3 所有 Manager 一览

每种组件类型都有对应的 Manager：

| Manager | 管理的组件 | 文件位置 |
|---------|-----------|---------|
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

**全部都是 @singleton。** 全局只有一个实例，内部一个 `{instance_code: instance}` 字典。

---

## 7. 用户配置 vs 系统内建的优先级

框架的扫描路径 = 用户路径 + 系统内建路径：

```python
# agentuniverse/base/agentuniverse.py:149-150
core_agent_package_list = (
    user_paths                                # 你 config.toml 里配的
    + system_default_paths                    # agentuniverse 自带的
)
```

**覆盖规则：** 如果同名（相同的 instance_code），用户配置优先于系统内建：

```python
# component_manager_base.py:33-37
if component_instance_name in self._instance_obj_map:
    if is_system_builtin(component_instance_obj):
        return   # 系统内建组件，但用户已经配了同名的 → 跳过
    LOGGER.warn(f"'{component_instance_name}' already registered.")
    return
```

这让你可以**覆盖框架内置的默认 Agent/LLM/Prompt，而不需要修改框架源码。**

---

## 8. 完整启动链路全景图

```
AgentUniverse().start()
│
├── 步骤 1: Banner ───────────────────────── 打印 ASCII 艺术 Logo
│
├── 步骤 2-4: 配置加载 ─────────────────────
│   Configer('config.toml')       ← 第 1 次（${VAR} 还是原始字符串）
│   CustomKeyConfiger()           ← 解析 API Keys
│   Configer('config.toml')       ← 第 2 次（${VAR} 能解析了）
│   AppConfiger()                 ← 应用级配置对象
│
├── 步骤 5-9: 基础设施初始化 ───────────────
│   __init_loggers()              ← 日志
│   __init_otel()                 ← 可观测性
│   __init_system()               ← DB / gRPC / Gunicorn
│   __init_extension_modules()    ← 扩展/钩子
│   __init_monitor()              ← 监控
│
└── 步骤 10: ★ __scan_and_register() ────── 组件扫描注册
    │
    ├── 为 22 种组件类型确定路径列表
    │   AGENT  → [用户路径...] + [agentuniverse.agent.default...]
    │   LLM    → [用户路径...] + [agentuniverse.llm.default...]
    │   ...
    │
    ├── scan() → 遍历所有路径 → rglob('*.yaml')
    │   → 每个 YAML → Configer → ComponentConfiger → 类型过滤
    │
    └── __register() → 对每个 ComponentConfiger
        → 动态导入 Python 类
        → 实例化
        → 生成 instance_code
        → register() 到对应 Manager 的 _instance_obj_map

结果:
  AgentManager._instance_obj_map = {
      'simple_qa_agent_app.agent.simple_qa_agent': <Agent 实例>,
  }
  LLMManager._instance_obj_map = {
      'simple_qa_agent_app.llm.qwen_llm': <LLM 实例>,
  }
  PromptManager._instance_obj_map = {
      'simple_qa_agent_app.prompt.simple_qa_prompt': <Prompt 实例>,
  }
  ...

现在你可以通过任何 Manager 获取任何组件:
  AgentManager().get_instance_obj('simple_qa_agent').run(input='你好')
```

---

## 9. 动手练习

### 练习 1：跟踪 startup 源码

打开 `agentuniverse/base/agentuniverse.py`，找到 `start()` 方法。在代码旁边注释每个步骤的作用。

### 练习 2：验证 instance_code

```bash
cd examples/sample_apps/simple_qa_agent_app
# 找出 appname
grep appname config/config.toml
# 看 Agent YAML 的 metadata
cat intelligence/agentic/agent/agent_instance/*.yaml | grep -A5 metadata
# 手动推算出 instance_code，格式: {appname}.agent.{info.name}
```

### 练习 3：阅读 Manager 源码

打开 `agentuniverse/base/component/component_manager_base.py`，重点看 `register()` 和 `get_instance_obj()` 方法。思考：为什么返回 `create_copy()` 而不是原始实例？

### 练习 4：新增一个 YAML 验证扫描

1. 复制 `simple_qa_agent.yaml` 为 `test_agent.yaml`
2. 只修改 `info.name: 'test_agent'`
3. 不修改 `config.toml`，启动框架
4. 尝试 `AgentManager().get_instance_obj('test_agent')` 看能否获取到
5. 思考：为什么能或为什么不能？

### 练习 5：画出组件注册流程图

从 `start()` → `__scan_and_register()` → `scan()` → `__register()` → `Manager.register()` 画出一个完整的流程图。

---

## 10. 概念速查表

| 概念 | 含义 | 关键文件 |
|------|------|---------|
| **AgentUniverse** | 框架启动入口 | `base/agentuniverse.py` |
| **start()** | 10 步启动引导 | `agentuniverse.py:64` |
| **Configer** | TOML/YAML 文件加载器 | `base/config/configer.py` |
| **AppConfiger** | 应用级配置容器 | `base/config/application_configer/` |
| **CustomKeyConfiger** | API Key 解析器（支持 ${VAR}） | `base/config/custom_configer/` |
| **ComponentEnum** | 22 种组件类型枚举 | `base/component/component_enum.py` |
| **ComponentConfiger** | 组件 YAML 的通用配置容器 | `base/config/component_configer/` |
| **ComponentManagerBase** | 所有 Manager 的基类 | `base/component/component_manager_base.py` |
| **ComponentConfigerUtil** | 根据 ComponentEnum 查找各类资源 | `base/component/component_configer_util.py` |
| **@singleton** | 简单但有效的单例装饰器 | `base/annotation/singleton.py` |
| **instance_code** | `{appname}.{type}.{name}` 唯一标识 | `component_base.py:29` |
| **_instance_obj_map** | Manager 内部的实例池（dict） | `component_manager_base.py:28` |
| **CORE_PACKAGE** | config.toml 中控制扫描路径的配置段 | — |

---

## 下一步

你已经理解了框架如何从 `start()` 一路走到所有组件就绪。下一步进入 **[04-prompt-engineering.md](04-prompt-engineering.md)**——Prompt 如何定义 Agent 的人设、如何组装、如何版本化管理。
