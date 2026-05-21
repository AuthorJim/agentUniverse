# start() 函数完整执行追踪

> **定位：** 这是 [03-framework-startup.md](03-framework-startup.md) 的补充深度阅读。第三章用简化伪代码讲解 10 步流程，本文档直接追踪**真实源码**的每一行，适合看完第三章后深入细节。
> **源码文件：** `agentuniverse/base/agentuniverse.py:64-141`

---

## 0. 先看全貌：`start()` 到底是一个什么样的函数？

把 `start()` 想象成一个**工厂的启动按钮**。按下按钮后，工厂不是马上开始生产，而是先做一系列准备工作：通电、通水、检查机器、分配工位、给每个工人发工具。所有这些做完后，工厂才进入"可生产"状态。

`start()` 就是这个按钮。它不生产任何东西（不运行 Agent、不调用 LLM），它只做**初始化**。调用完 `start()` 之后，所有组件就绪，你才能通过 `AgentManager().get_instance_obj('xxx').run(input='...')` 来真正干活。

在 Node.js 世界里，它等于一次性执行：

```javascript
// 这一行 Python 等于下面所有这些 JS
await Promise.all([
    showBanner(),                // 打印 Logo
    loadDotEnv(),                // 加载环境变量
    loadAppConfig(),             // 加载并解析主配置
    initWinston(),               // 初始化日志
    initOpenTelemetry(),         // 初始化可观测性
    initDatabase(),              // 初始化数据库
    configureGrpc(),             // 配置 gRPC
    configureGunicorn(),         // 配置 HTTP 服务器
    loadPlugins(),               // 加载扩展/插件
    startMonitor(),              // 启动监控
    scanAndRegisterComponents(), // ★ 扫描并注册所有组件
]);
```

---

## 1. 前置：`__init__` — 构造函数里的"预设"

在调用 `start()` 之前，先要创建 `AgentUniverse` 实例：

```python
AgentUniverse().start()
#            ^^ —— 这里先执行 __init__()
```

`__init__` 做了两件事（`agentuniverse.py:44-62`）：

### 1.1 创建两个容器对象

```python
self.__application_container = ApplicationComponentManager()
self.__config_container: ApplicationConfigManager = ApplicationConfigManager()
```

| 容器 | 类型 | 用途 | 类比 |
|------|------|------|------|
| `__application_container` | `ApplicationComponentManager` | 管理**运行时组件实例**（Agent、LLM 等） | DI 容器的实例池 |
| `__config_container` | `ApplicationConfigManager` | 管理**应用级配置**（AppConfiger） | 应用配置的 holder |

> **使用场景：** `__config_container` 在 `start()` 中被反复使用 —— 存储 AppConfiger、传递 `yaml_func_instance`。`__application_container` 目前代码中使用较少，主要作为预留的运行时组件管理入口。

### 1.2 声明 22 种系统内建组件的扫描路径

```python
self.__system_default_agent_package = ['agentuniverse.agent.default']
self.__system_default_llm_package = ['agentuniverse.llm.default']
self.__system_default_tool_package = ['agentuniverse.agent.action.tool']
# ... 共 22 种
```

这些是框架**内置**的默认组件所在路径。后面在 `__scan_and_register` 中，它们会被追加到用户配置的路径后面，形成最终的扫描列表。

---

## 2. `start()` 逐行拆解

### 第 1 步：打印 Banner（第 68 行）

```python
character_util.show_au_start_banner()
```

在终端打印 agentUniverse 的 ASCII Art 字样的 Logo。纯粹是为了"仪式感"，不涉及任何逻辑。≈ Node.js 里 `console.log(figlet('agentUniverse'))`。

---

### 第 2 步：设置 Python 搜索路径（第 70-72 行）

```python
project_root_path = get_project_root_path()
sys.path.append(str(project_root_path.parent))
self._add_to_sys_path(project_root_path, ['intelligence', 'app'])
```

这三行做的事情：**让 Python 能 import 到你的项目代码**。

**为什么要手动加？** Python 的 `import` 搜索路径（`sys.path`）默认不包含你的项目目录。如果不加这两行，`config.toml` 里配置的 `${ROOT_PACKAGE}.config.config_extension.ConfigExtension` 这种动态导入就会失败——Python 根本找不到 `sample_standard_app` 这个包。

具体操作：
1. `get_project_root_path()` 找到项目根目录
2. 把根目录的**父目录**加入 `sys.path`（这样 `from your_project.xxx import yyy` 才能工作）
3. 把 `intelligence/` 和 `app/` 子目录也加入（如果存在）—— 这是约定俗成的用户代码目录

> **Node.js 对照：** ≈ 动态修改 `NODE_PATH` 环境变量，或者在运行时往 `require.resolve.paths` 里追加路径。

---

### 第 3 步：第 1 次加载 config.toml（第 74-79 行）

```python
if not config_path:
    config_path = project_root_path / 'config' / 'config.toml'
    config_path = str(config_path)

configer = Configer(path=config_path).load()
```

如果调用 `start()` 时没传 `config_path`，默认使用 `<项目根>/config/config.toml`。

`Configer(path).load()` 内部流程：

```
Configer.__init__(path)          # 存路径
    → .load()                    # 入口
    → load_by_path(path)         # 分发
    → __choice_load_method(path) # 根据后缀选 loader
    → __load_toml_file(path)     # TOML 解析 + PlaceholderResolver
```

> **使用场景：** `Configer` 是框架中**所有**配置文件的加载器，不仅用于 TOML，也用于 YAML。在组件扫描阶段，每个 `.yaml` 文件也是通过 `Configer(path=xxx).load()` 加载的。

**此时 `${VAR}` 占位符还无法解析，因为 API 密钥还没加载到环境变量里。**

---

### 第 4 步：加载 custom_key.toml（第 82-85 行）

```python
custom_key_configer_path = self.__parse_sub_config_path(
    configer.value.get('SUB_CONFIG_PATH', {}).get('custom_key_path'),
    config_path)
CustomKeyConfiger(custom_key_configer_path)
```

**完整调用链：**

```
config.toml:
  [SUB_CONFIG_PATH]
  custom_key_path = './custom_key.toml'    ← 从这里读取路径

__parse_sub_config_path('./custom_key.toml', 'config/config.toml')
    → 相对路径 → 拼接为绝对路径: config/custom_key.toml

CustomKeyConfiger('config/custom_key.toml')
    → 继承自 Configer，构造函数自动调用 self.load()
    → load_by_path → __choice_load_method → __load_toml_file
    → 解析 TOML 得到 {"KEY_LIST": {"DASHSCOPE_API_KEY": "sk-xxx", ...}}
    → 遍历 KEY_LIST，逐个写入 os.environ:
        os.environ['DASHSCOPE_API_KEY'] = 'sk-xxx'
        os.environ['OPENAI_API_KEY'] = 'sk-yyy'
```

> **使用场景：** `CustomKeyConfiger` 是 `@singleton`，全局只有一个实例。它继承 `Configer`，所以复用了 TOML 加载逻辑。写入 `os.environ` 后，后续所有 `PlaceholderResolver().resolve()` 都能通过 `${VAR_NAME}` 读取这些密钥。

`__parse_sub_config_path` 方法（第 355-375 行）负责把相对路径转为绝对路径：

```python
def __parse_sub_config_path(self, input_path, reference_file_path):
    if not input_path:
        return None
    input_path_obj = Path(input_path)
    if input_path_obj.is_absolute():
        combined_path = input_path_obj
    else:
        # 相对路径 → 以主配置文件所在目录为基准拼接
        combined_path = Path(reference_file_path).parent / input_path_obj
    return str(combined_path)
```

---

### 第 5 步：第 2 次加载 config.toml + 创建 AppConfiger（第 88-90 行）

```python
configer = Configer(path=config_path).load()
app_configer = AppConfiger().load_by_configer(configer)
self.__config_container.app_configer = app_configer
```

**为什么加载两次？** 现在 `os.environ` 里已经有 API 密钥了，`PlaceholderResolver` 能把 `${DASHSCOPE_API_KEY}` 替换为实际值。第二次加载得到的配置才是**完整的、可用的**。

**`AppConfiger().load_by_configer(configer)`** 做了什么：

把 `Configer` 中的**原始字典**拆解为结构化的属性，方便后续代码按类型访问：

```python
# AppConfiger 内部（app_configer.py）：
self.base_info_appname = configer.value.get('BASE_INFO', {}).get('appname')
self.core_agent_package_list = configer.value.get('CORE_PACKAGE', {}).get('agent')
self.core_llm_package_list = configer.value.get('CORE_PACKAGE', {}).get('llm')
# ... 所有 CORE_PACKAGE 下的路径配置
```

> **使用场景：** `AppConfiger` 是整个启动流程中的**配置中枢**。后续所有步骤 —— 日志初始化、gRPC 配置、扩展加载、组件扫描 —— 都从它读取配置。它也是 `yaml_func_instance` 的载体，在后续组件注册时传播给每个 `ComponentConfiger`。

最后一行把 `app_configer` 存入 `__config_container`，使得后续方法都能通过 `self.__config_container.app_configer` 访问。

---

### 第 6 步：初始化日志系统（第 93-96 行）

```python
log_config_path = self.__parse_sub_config_path(
    configer.value.get('SUB_CONFIG_PATH', {}).get('log_config_path'),
    config_path)
init_loggers(log_config_path)
```

`init_loggers` 在 `agentuniverse/base/util/logging/logging_util.py` 中，基于 **loguru**（Python 的日志库，≈ Node.js 的 winston）初始化：

1. 加载 `log_config.toml`，解析日志级别、输出路径、轮转策略
2. 配置 loguru 的 handler（文件输出 + 标准错误输出）
3. 全局 `LOGGER` 对象就绪

**日志在这个时间点初始化，意味着此前如果有错误，只能通过 `print()` 输出。**

---

### 第 7 步：初始化 OpenTelemetry（第 99 行）

```python
TelemetryManager().init_from_config(configer.value.get('OTEL', {}))
```

从 `config.toml` 读 `[OTEL]` 配置段，初始化分布式追踪（traces）和指标（metrics）。如果没配置或 `activate = 'false'`，跳过。

> **使用场景：** OpenTelemetry 用于生产环境的可观测性 —— 追踪一次 Agent 调用穿过了哪些组件、每个阶段耗时多少。开发阶段通常关闭。

---

### 第 8 步：初始化请求任务数据库（第 102 行）

```python
RequestLibrary(configer=configer)
```

`RequestLibrary` 管理**异步请求任务**的持久化。它从 `config.toml` 读 `[DB].system_db_uri` 创建数据库连接（默认 SQLite）。用于 Web 服务模式下，客户端提交异步 Agent 任务时存储任务状态。

---

### 第 9 步：配置 gRPC（第 105-108 行）

```python
grpc_activate = configer.value.get('GRPC', {}).get('activate')
if grpc_activate and grpc_activate.lower() == 'true':
    ACTIVATE_OPTIONS["grpc"] = True
    set_grpc_config(configer)
```

如果 `config.toml` 中 `[GRPC].activate = 'true'`，就初始化 gRPC 服务器配置。gRPC 是 agentUniverse 对外暴露 API 的可选方式之一（另一种是 HTTP/Gunicorn）。

---

### 第 10 步：配置 Gunicorn（第 111-123 行）

```python
sync_service_timeout = configer.value.get('HTTP_SERVER', {}).get('sync_service_timeout')
if sync_service_timeout:
    FlaskServerManager().sync_service_timeout = sync_service_timeout

gunicorn_activate = configer.value.get('GUNICORN', {}).get('activate')
if gunicorn_activate and gunicorn_activate.lower() == 'true':
    ACTIVATE_OPTIONS["gunicorn"] = True
    gunicorn_config_path = self.__parse_sub_config_path(
        configer.value.get('GUNICORN', {}).get('gunicorn_config_path'), config_path
    )
    from ..agent_serve.web.gunicorn_server import GunicornApplication
    GunicornApplication(config_path=gunicorn_config_path)
```

Gunicorn 是 Python 生态的**生产级 WSGI HTTP 服务器**（≈ Node.js 的 PM2 + cluster 模式）。如果激活，会加载 `gunicorn_config.toml` 配置 bind 地址、worker 数量、超时等。

**延迟导入** `from ..agent_serve.web.gunicorn_server import GunicornApplication` 写在 `if` 里面，意味着只有启用 Gunicorn 时才导入，避免不必要的依赖加载。

> **使用场景：** 开发阶段通常在 `config.toml` 里设置 `activate = 'false'`，用 Flask 内置开发服务器。部署到生产时启用 Gunicorn 获取多 worker 并发能力。

---

### 第 11 步：加载扩展模块（第 126-132 行）

```python
ext_classes = configer.value.get('EXTENSION_MODULES', {}).get('class_list')
if isinstance(ext_classes, list):
    for ext_class in ext_classes:
        if "YamlFuncExtension" in ext_class:
            self.__config_container.app_configer.yaml_func_instance = \
                self.__dynamic_import_and_init(ext_class)
        else:
            self.__dynamic_import_and_init(ext_class, configer)
```

这段逻辑我们在之前详细分析过。快速回顾：

| 扩展类 | 构造参数 | 实例保存 | 使用时机 |
|--------|---------|---------|---------|
| `YamlFuncExtension` | 无 | 是 → `app_configer.yaml_func_instance` | 运行时持续使用（解析 YAML 中的 `@FUNC(...)`） |
| `ConfigExtension` | `configer`（应用配置） | 否（fire-and-forget） | 仅在启动时执行一次 |

`__dynamic_import_and_init`（第 377-388 行）做的事：

```
类路径字符串 'sample_standard_app.config.yaml_func_extension.YamlFuncExtension'
    → rpartition('.') → module='sample_standard_app.config.yaml_func_extension'
                       class='YamlFuncExtension'
    → importlib.import_module(module)  ← 动态导入
    → getattr(module, class)           ← 获取类
    → cls() 或 cls(configer)           ← 按需传参实例化
```

> **使用场景：** `YamlFuncExtension` 的实例被传播到每个 `ComponentConfiger`，然后在解析 LLM YAML 配置时，通过 `process_yaml_func()` 调用其方法（如 `load_api_key('qwen')`），实现"在 YAML 配置中调用 Python 函数"的能力。

---

### 第 12 步：初始化监控（第 135 行）

```python
Monitor(configer=configer)
```

从 `config.toml` 读 `[MONITOR]` 配置段。如果激活，启动一个后台监控线程，收集框架运行指标。开发期间通常关闭。

---

### 第 13 步：扫描并注册所有组件（第 138 行）★ 最核心

```python
self.__scan_and_register(self.__config_container.app_configer)
```

这是整个启动流程中最复杂的一步。详见下一节。

### 第 14 步：执行 Post-Fork 队列（第 139-141 行）

```python
if core_mode:
    for _func, args, kwargs in POST_FORK_QUEUE:
        _func(*args, **kwargs)
```

`core_mode` 模式下，在组件注册完成后执行 `POST_FORK_QUEUE` 中的延迟任务。

> **使用场景：** `POST_FORK_QUEUE` 是 Gunicorn 多进程架构的配套机制。Gunicorn 使用 pre-fork 模型（主进程 fork 出多个 worker），某些初始化操作（如 MCP 服务器启动）必须在 fork **之后**的 worker 进程中执行，而不能在 fork 之前的主进程中执行。这些操作被推入 `POST_FORK_QUEUE`，在 `core_mode=True` 时由 worker 逐个执行。

---

## 3. `__scan_and_register()` 深度拆解

`agentuniverse.py:143-223`

### 3.1 组装扫描路径

```python
core_agent_package_list = (
    (app_configer.core_agent_package_list or app_configer.core_default_package_list)
    + self.__system_default_agent_package
)
```

每种组件类型的最终扫描路径 = **用户配置路径（或 default 兜底）** + **系统内建路径**。

**覆盖规则：** `core_agent_package_list or core_default_package_list`
- 如果用户在 `config.toml` 的 `[CORE_PACKAGE].agent` 配了路径 → 用用户的
- 如果没配 → 用 `[CORE_PACKAGE].default`
- 两者都有 → 用户的优先（`or` 短路）

22 种组件类型都遵循同样的模式，组装成 `component_package_map`：

```python
component_package_map = {
    ComponentEnum.AGENT: core_agent_package_list,
    ComponentEnum.LLM: core_llm_package_list,
    ComponentEnum.TOOL: core_tool_package_list,
    # ... 共 22 对映射
}
```

### 3.2 两阶段循环

```python
# 阶段 1：扫描 —— 找出所有 YAML 文件并解析为 ComponentConfiger
component_configer_list_map = {}
for component_enum, package_list in component_package_map.items():
    if not package_list:
        continue
    component_configer_list = self.scan(package_list, ConfigTypeEnum.YAML, component_enum)
    component_configer_list_map[component_enum] = component_configer_list

# 阶段 2：注册 —— 将 ComponentConfiger 实例化并注册到 Manager
for component_enum, component_configer_list in component_configer_list_map.items():
    self.__register(component_enum, component_configer_list)
```

**为什么分两阶段而不是边扫描边注册？** 因为注册 Agent 时需要知道哪些 LLM、Tool 是 Agent 声明的依赖（用于后续的"按需注册"优化）。先扫描所有类型，再统一注册，确保依赖信息完整。

### 3.3 `scan()` 方法

`agentuniverse.py:225-270`

```python
def scan(self, package_list, config_type_enum, component_enum):
    component_configer_list = []

    # LLM 特殊处理：找默认 LLM 配置
    if component_enum.value == ComponentEnum.LLM.value:
        default_llm_config_path = find_default_llm_config(package_list)
        if default_llm_config_path:
            default_llm_configer = DefaultLLMConfiger(default_llm_config_path)
            self.__config_container.app_configer.default_llm_configer = default_llm_configer

    # 遍历每个包路径
    for package_name in package_list:
        package_path = self.__package_name_to_path(package_name)
        # 递归查找所有 .yaml 文件
        config_files = Path(package_path).rglob(f'*.{config_type_enum.value}')

        for config_file in config_files:
            configer = Configer(path=str(config_file)).load()
            component_configer = ComponentConfiger().load_by_configer(configer)

            # 类型过滤：metadata.type 必须匹配
            if component_configer.get_component_config_type() == component_enum.value:
                component_configer_list.append(component_configer)

    return component_configer_list
```

**`__package_name_to_path`（第 338-353 行）** 将 Python 包名转为文件系统路径：

```python
# 'agentuniverse.agent.default' → '/path/to/agentuniverse/agent/default/'
spec = importlib.util.find_spec(package_name)
package_path = spec.submodule_search_locations[0]
```

> **使用场景：** 这是连接 Python 模块系统和文件系统的桥梁。`config.toml` 里配置的是 Python 包名（方便 IDE 跳转和重构），但实际扫描需要文件路径。

**LLM 的特殊处理：** `find_default_llm_config` 在 LLM 的扫描包中寻找 `default_llm.toml` 文件，它定义了系统默认使用的 LLM（例如 `qwen2.5-72b-instruct`）。这个默认 LLM 的名称被加入 `agent_llm_set`，确保即使 Agent 没有显式声明 LLM，也有一个可用的默认值。

### 3.4 `__register()` 方法

`agentuniverse.py:272-336`

对每个 `ComponentConfiger`，做以下 5 步：

```python
# 1. 获取 Manager 类
manager_clz = ComponentConfigerUtil.get_component_manager_clz_by_type(component_enum)

for component_configer in component_configer_list:
    # 2. 获取类型专属的 Configer 类并二次加载
    configer_clz = ComponentConfigerUtil.get_component_config_clz_by_type(component_enum)
    configer_instance = configer_clz().load_by_configer(component_configer.configer)

    # 3. 注入全局依赖
    configer_instance.yaml_func_instance = yaml_func_instance
    configer_instance.default_llm_configer = default_llm_configer

    # 4. AGENT/LLM/TOOL/TOOLKIT 特殊处理
    #    （收集 agent 声明的 llm、tool 依赖名到对应的 set）

    # 5. 动态导入 + 实例化 + 注册
    component_clz = ComponentConfigerUtil.get_component_object_clz_by_component_configer(configer_instance)
    component_instance = component_clz().initialize_by_component_configer(configer_instance)
    component_instance.component_config_path = component_configer.configer.path
    manager_clz().register(component_instance.get_instance_code(), component_instance)
```

**`ComponentConfigerUtil`** 是整个注册流程的**路由器**——根据 `ComponentEnum` 查找对应的 Manager 类、Configer 类和 Python 组件类。

**AGENT/LLM/TOOL 的特殊处理逻辑：**

- **Agent** → 提取 `profile.llm_model.name` 加入 `agent_llm_set`，提取 `action.tool` 加入 `agent_tool_set`。这是"依赖收集"阶段。
- **LLM** → 只有在 `agent_llm_set` 中的 LLM 才直接注册给 Manager；不在的放入 `llm_configer_map`（按需加载）。
- **TOOL/TOOLKIT** → 类似逻辑，支持 MCP 工具注册。

本质上是**按需注册优化**：如果一个 LLM 没有被任何 Agent 引用，就不需要完整注册。

**Instance Code 公式：**
```
{appname}.{component_type}.{component_name}
```
例如：`simple_qa_agent_app.agent.simple_qa_agent`

---

## 4. 完整执行流程图

```
AgentUniverse.__init__()
├── 创建 ApplicationComponentManager
├── 创建 ApplicationConfigManager
└── 声明 22 种系统内建组件路径

AgentUniverse().start(config_path)
│
├── [68]  show_au_start_banner()          打印 Logo
├── [70-72] sys.path 设置                 让 Python 能找到项目代码
│
├── [74-79] 第 1 次加载 config.toml       ${VAR} 还是原始占位符
│   └── Configer.load() → __load_toml_file → PlaceholderResolver
│
├── [82-85] CustomKeyConfiger()            加载 API 密钥 → 注入 os.environ
│   └── 遍历 KEY_LIST → os.environ[key] = value
│
├── [88-90] 第 2 次加载 config.toml       这次 ${VAR} 能正确解析
│   ├── Configer.load() → __load_toml_file
│   └── AppConfiger().load_by_configer()  结构化配置
│
├── [93-96] init_loggers()                基于 log_config.toml 初始化 loguru
├── [99]    TelemetryManager.init()       OpenTelemetry 追踪/指标
├── [102]   RequestLibrary()              异步任务数据库
│
├── [105-108] gRPC 配置                   条件：activate = 'true'
├── [111-123] Gunicorn 配置               条件：activate = 'true'
│
├── [126-132] 扩展模块加载
│   ├── YamlFuncExtension → 存为 yaml_func_instance（运行时持续使用）
│   └── ConfigExtension → fire-and-forget（启动钩子）
│
├── [135]   Monitor()                     条件：activate = 'true'
│
├── [138]   __scan_and_register() ★★★
│   │
│   ├── 组装 22 种组件类型的扫描路径
│   │   每种 = 用户路径(or default) + 系统内建路径
│   │
│   ├── 阶段 1: scan()
│   │   ├── 对每种类型:
│   │   │   ├── 包名 → 文件路径 (__package_name_to_path)
│   │   │   ├── rglob('*.yaml') 找所有 YAML
│   │   │   ├── 每个 YAML → Configer → ComponentConfiger
│   │   │   └── metadata.type 过滤
│   │   └── LLM 特殊: find_default_llm_config()
│   │
│   └── 阶段 2: __register()
│       ├── 获取 Manager 类 + Configer 类
│       ├── 对每个 ComponentConfiger:
│       │   ├── 注入 yaml_func_instance + default_llm_configer
│       │   ├── 依赖收集 (AGENT → agent_llm_set / agent_tool_set)
│       │   ├── 按需注册 (LLM/TOOL 不在依赖集的延后)
│       │   ├── 动态导入 Python 类 + 实例化
│       │   └── manager.register(instance_code, instance)
│       └── 结果: 22 个 Manager 的 _instance_obj_map 填充完毕
│
└── [139-141] POST_FORK_QUEUE              仅 core_mode（fork 后延迟任务）

结果:
  所有 Manager 就绪，可以通过 get_instance_obj() 获取任何组件
```

---

## 5. 关键设计决策

### 5.1 为什么 config.toml 加载两次？

第一次拿 `custom_key_path`，加载 API 密钥到 `os.environ`，第二次解析所有 `${VAR}`。这是因为 `PlaceholderResolver` 依赖 `os.environ`，而密钥必须先注入。

### 5.2 为什么扫描和注册要分两阶段？

Agent 配置里声明了它依赖哪些 LLM、Tool。先扫描所有 YAML 收集依赖信息，注册时就能做"按需注册"——不被任何 Agent 引用的 LLM 放到 `llm_configer_map` 延后加载。

### 5.3 为什么 YAML 用 rglob 而不是 glob？

`rglob('*.yaml')` 递归搜索所有子目录，`glob('*.yaml')` 只搜索当前目录。组件 YAML 可以嵌套在多层子目录中（如 `agent/agent_instance/demo_agent.yaml`），必须用 rglob。

### 5.4 为什么 get_instance_obj 返回深拷贝？

`ComponentManagerBase.get_instance_obj()` 默认返回 `create_copy()`。因为框架支持并发请求，如果多个请求共享同一个 Agent 实例，各自的 memory 会互相污染。深拷贝保证了实例隔离。

---

## 6. 与现有文档的关系

| 文档 | 定位 | 适合 |
|------|------|------|
| [03-framework-startup.md](03-framework-startup.md) | 概念讲解 + 简化伪代码 | 初次学习 |
| **本文档** | 真实源码逐行追踪 | 深入理解 |

建议先读第三章建立概念框架，再用本文档对照源码验证每个细节。
