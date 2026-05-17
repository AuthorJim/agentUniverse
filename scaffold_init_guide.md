# agentUniverse 最小脚手架初始化指南（Node.js 开发者视角）

本文档面向有 Node.js 背景的全栈工程师，解释如何基于 agentUniverse 的工程化设计创建最小脚手架项目。文中会持续对照 Node.js 生态中的等价概念，帮助你在已有知识体系上建立映射。

**该脚手架不包含任何具体 Agent 实现，仅包含框架启动、配置和组件注册所需的基础结构。**

---

## 零、先建立概念映射

在深入细节之前，先建立 Python 生态与 Node.js 生态的核心概念映射。阅读后续章节时随时回这里查阅。

| 概念 | Python / agentUniverse | Node.js 等价物 |
|------|----------------------|---------------|
| 包管理器 | Poetry (`poetry install`, `pyproject.toml`) | npm/yarn/pnpm (`package.json`) |
| 依赖锁文件 | `poetry.lock` | `package-lock.json` / `yarn.lock` |
| 项目隔离环境 | virtualenv（Poetry 自动管理） | `node_modules/`（项目级隔离） |
| 模块声明 | `__init__.py`（标记目录为包） | `index.js` 或 `package.json` 的 `exports` |
| 模块查找路径 | `sys.path`（Python 运行时搜索路径） | `NODE_PATH` + `node_modules` 解析算法 |
| 入口脚本 | `if __name__ == "__main__"` | `"main"` 字段 / `node server.js` |
| 配置文件格式 | TOML（类似 INI + JSON 的融合） | JSON / YAML / dotenv |
| 环境变量/密钥 | `custom_key.toml`（不提交 Git） | `.env` 文件 + `dotenv` |
| 变量插值 | `${VAR_NAME}`（框架级占位符解析） | `process.env.VAR` / `dotenv-expand` |
| Lint 工具 | ruff（≈ ESLint） | ESLint |
| 格式化工具 | black（≈ Prettier） | Prettier |
| 类型检查 | mypy（基于类型标注） | TypeScript 编译器 |
| 测试框架 | pytest | Jest / Vitest |
| 日志库 | loguru | winston / pino |
| HTTP 框架 | Flask | Express |
| 进程管理 | Gunicorn（WSGI 服务器，多 worker） | PM2 / Node `cluster` 模块 |
| ORM | SQLAlchemy | Prisma / Sequelize / Knex |
| gRPC | 内置 `grpc` 模块 | `@grpc/grpc-js` |
| 可观测性 | OpenTelemetry（OTEL） | OpenTelemetry JS SDK |
| 装饰器 | `@singleton`、`@lru_cache` | TypeScript 装饰器 / HOF 包装 |
| 组件注册 | YAML 声明 + 自动扫描 → 单例 Manager | DI 容器（Awilix / InversifyJS / NestJS DI） |
| 生命周期钩子 | `ConfigExtension.__init__(configer)` | `onModuleInit` / Express middleware / webpack plugins |

### 几个"思维拐弯"点

这些是 Node.js 开发者切换到 Python 时最容易困惑的地方，提前说明：

1. **`__init__.py` 是用来干什么的？**  
   在 Node.js 中，`require('./foo')` 会自动找 `foo/index.js`。Python 不同——必须有一个 `__init__.py` 文件（可以为空），目录才会被视为"包"（package），否则 `import foo` 会失败。可以把它理解为 Python 的"这个目录是一个模块"的显式声明。

2. **`sys.path` 而不是 `node_modules` 查找。**  
   Node.js 有复杂的 `node_modules` 递归查找算法。Python 更简单（也更粗暴）：`sys.path` 是一个目录列表，`import` 时逐个查找。agentUniverse 在启动时将项目父目录和 `intelligence/` 加入 `sys.path`，相当于 Node.js 中往 `NODE_PATH` 里追加目录。

3. **TOML 是什么？**  
   TOML ≈ INI + JSON 的融合体。有 `[section]` 分层结构，支持字符串、数字、布尔、数组等类型。比 JSON 多了注释支持和更人性化的多行字符串，比 YAML 少了缩进敏感。Node.js 生态中也常见 TOML（如 `pnpm-workspace.yaml` 其实用的是 YAML，但 rust 工具链大量使用 TOML）。

4. **Python 的类型系统是"渐进式"的。**  
   和 TypeScript 不同，Python 的类型标注是**可选的、运行时不检查的**。mypy 是一个独立的静态检查工具，类似把 `tsc --noEmit` 单独跑。类型标注不会影响运行时行为。

---

## 一、设计思路概述

agentUniverse 的工程化核心是 **声明式配置 + 自动扫描注册** 的组件体系。如果你用过 NestJS，这个思路非常熟悉——定义 module/controller/provider 然后框架自动装配。如果你用过 Next.js 的 `pages/` 目录自动路由，扫描注册的概念也很类似。

1. **配置驱动**：所有组件（Agent、LLM、Tool、Memory、Prompt 等）通过 YAML/TOML 配置文件声明。类比：像 NestJS 的 `@Module()` 装饰器声明 providers/controllers，但这里用 YAML 文件代替装饰器。
2. **包路径扫描**：`config.toml` 中指定各组件的 Python 包路径（类似在 `tsconfig.json` 中指定 `include` 路径），框架递归扫描路径下所有 `.yaml`/`.toml` 文件。
3. **组件化管理**：每种组件类型对应一个单例 Manager（≈ 单例 DI 容器）。通过 `instance_code`（格式：`{appname}.{component_type}.{name}`，类似 DI 容器的 key）进行注册和检索。
4. **扩展钩子**：`ConfigExtension` ≈ Express 中间件/lifecycle hook；`YamlFuncExtension` ≈ 模板引擎中的自定义 helper 函数。

**启动流程概览（对照 Node.js 思维）：**

```
AgentUniverse().start(config_path)          ≈  await app.init() / bootstrap()
  ├── 1. 将项目根目录及 intelligence/、
  │      app/ 加入 sys.path                 ≈  追加目录到 NODE_PATH
  ├── 2. 加载 config.toml                   ≈  require('./config/default.json')
  │     （不解析 ${VAR} 占位符）               （第一遍：原始文本加载）
  ├── 3. 加载 custom_key.toml               ≈  dotenv.config() 加载 .env
  │     （API Key 等敏感配置）
  ├── 4. 重新加载 config.toml               ≈  用 env 值做字符串替换
  │     （此时解析 ${VAR} 占位符）              （类似 dotenv-expand）
  ├── 5. 初始化日志系统                      ≈  winston.configure({...})
  │     （基于 log_config.toml）
  ├── 6. 初始化 OpenTelemetry（可选）
  ├── 7. 初始化数据库连接、gRPC、Gunicorn
  ├── 8. 加载 EXTENSION_MODULES             ≈  注册插件/中间件
  │     （ConfigExtension / YamlFuncExtension）
  ├── 9. 初始化监控模块
  └── 10. 扫描并注册所有组件                 ≈  NestJS 的模块扫描 + DI 注册
       ├── 全局扫描各包路径下的 YAML/TOML     ≈  Glob + auto-discover
       ├── 解析为配置对象
       ├── 实例化组件                        ≈  DI 容器 resolve
       └── 注册到对应 Manager                ≈  container.register(key, instance)
```

**关键理解**：这个启动流程相当于一次性完成 Node.js 项目中 `dotenv` → `winston` → `express/grpc` → `DI container` 的初始化串联。agentUniverse 把它封装在一个 `start()` 调用里。

---

## 二、最小项目目录结构

以下是脚手架完整目录结构。`your_app` 替换为实际应用名（Python 包名，用 snake_case，类似 Node.js 包的 kebab-case 命名习惯）：

```
your_app/
├── pyproject.toml                          # ≈ package.json（依赖+工具链配置）
├── poetry.toml                             # ≈ .npmrc（包管理器行为）
├── poetry.lock                             # ≈ package-lock.json（自动生成）
├── .gitignore
├── __init__.py                             # 标记为 Python 包（≈ 有 index.js 才叫模块）
│
├── config/                                 # 配置目录（≈ config/ 或 .env 目录）
│   ├── config.toml                         # 主配置（≈ 结构化 config/default.json）
│   ├── custom_key.toml                     # API Key（≈ .env，不提交 Git）
│   ├── custom_key.toml.sample              # Key 模板（≈ .env.example，提交 Git）
│   ├── log_config.toml                     # 日志配置（≈ winston config 对象）
│   ├── config_extension.py                 # 自定义初始化钩子（≈ app.on('init')）
│   └── yaml_func_extension.py             # YAML 模板函数（≈ Handlebars helpers）
│
├── intelligence/                           # 业务逻辑根目录（≈ src/）
│   ├── __init__.py
│   ├── agentic/                            # Agent 相关组件（≈ services/ 或 domain/）
│   │   ├── __init__.py
│   │   ├── llm/                            # LLM 配置（≈ AI provider 配置）
│   │   │   ├── __init__.py
│   │   │   └── default_llm.toml           # 默认模型声明
│   │   ├── agent/                          # Agent 配置
│   │   │   └── __init__.py
│   │   ├── tool/                           # Tool 配置
│   │   │   └── __init__.py
│   │   ├── memory/                         # Memory 配置
│   │   │   └── __init__.py
│   │   ├── prompt/                         # Prompt 模板
│   │   │   └── __init__.py
│   │   ├── knowledge/                      # RAG 知识库
│   │   │   ├── __init__.py
│   │   │   ├── store/                      # 向量存储
│   │   │   │   └── __init__.py
│   │   │   ├── rag_router/                 # RAG 路由
│   │   │   │   └── __init__.py
│   │   │   ├── doc_processor/              # 文档处理
│   │   │   │   └── __init__.py
│   │   │   └── query_paraphraser/          # 查询改写
│   │   │       └── __init__.py
│   │   ├── toolkit/                        # Toolkit 配置
│   │   │   └── __init__.py
│   │   └── work_pattern/                   # 多 Agent 协作模式
│   │       └── __init__.py
│   ├── service/                            # HTTP/RPC 服务配置（≈ routes/）
│   │   ├── __init__.py
│   │   └── agent_service/
│   │       └── __init__.py
│   ├── utils/                              # 工具类（≈ lib/ 或 helpers/）
│   │   ├── __init__.py
│   │   └── common/
│   │       └── __init__.py
│   └── test/                               # 测试（≈ __tests__/）
│       └── __init__.py
│
└── bootstrap/                              # 应用入口（≈ bin/ 或 server.js）
    ├── __init__.py
    └── intelligence/
        ├── __init__.py
        └── server_application.py           # 启动文件（≈ npm start 入口）
```

### 目录设计原则（Node.js 类比）

| 目录 | 用途 | Node.js 类似物 |
|------|------|---------------|
| `config/` | 框架约定配置根目录。`config.toml` 中的相对路径基于此目录解析 | `config/` 目录 + `.env` |
| `intelligence/agentic/` | Agent 相关组件。框架将 `intelligence` 加入 `sys.path`，其下包可直接 `import` | `src/` 目录 + `NODE_PATH=src` |
| `bootstrap/` | 应用启动入口，与业务逻辑解耦 | `server.js` / `bin/www` |
| `intelligence/utils/` | 自定义工具 | `lib/` / `helpers/` |
| `intelligence/test/` | 本地测试脚本 | `__tests__/` |

---

## 三、各文件说明与作用

### 3.1 `pyproject.toml` —— ≈ `package.json`

**作用**：Python 项目的"身份证"——包名、版本、依赖、工具链配置全部在此。

**对照理解**：

| pyproject.toml 内容 | package.json 等价 |
|---------------------|-------------------|
| `[tool.poetry]` → `name`, `version`, `packages` | `name`, `version`, `files` |
| `[tool.poetry.dependencies]` → `python = "^3.10"`, `agentUniverse = "~0.0.15"` | `"engines": { "node": ">=18" }` + `dependencies` |
| `[tool.poetry.group.dev.dependencies]` → pytest, ruff, mypy | `devDependencies` → jest, eslint, typescript |
| `[build-system]` → `poetry-core` | `"build"` script / bundler 配置 |
| `[tool.ruff]` → lint 规则选择 | `.eslintrc` / `eslintConfig` |
| `[tool.black]` → `line-length = 120` | `.prettierrc` → `"printWidth": 120` |
| `[tool.mypy]` → 类型检查严格度 | `tsconfig.json` → `"strict": true` |

**踩坑提示**：`[tool.poetry.packages]` 中的 `include` 配置告诉 Poetry 哪些文件属于这个包。如果漏配，`poetry install` 后你的代码可能 `import` 不到。

### 3.2 `poetry.toml` —— ≈ `.npmrc`

**作用**：控制 Poetry 自身行为。推荐：

```toml
[virtualenvs]
in-project = true
```

这会让 Poetry 在项目目录下创建 `.venv/`，而不是全局目录。等价于 `npm config set prefix ./` 的效果——让环境隔离更直观。

### 3.3 `.gitignore` —— 和 Node.js 完全一样的概念

额外注意：**务必加上 `custom_key.toml`**。类比 `.env` 绝对不能提交到 Git。

### 3.4 `config/config.toml` —— 主配置文件（核心，≈ `config/default.json` + 路由注册表）

**作用**：定义框架全局参数 + 各组件类型的扫描路径。这是启动时最重要的文件，相当于一个结构化的配置中心。

**必需配置项及 Node.js 对照**：

| 配置节 | 关键字段 | 说明 | Node.js 类比 |
|--------|---------|------|-------------|
| `[BASE_INFO]` | `appname` | 应用名，组件 `instance_code` 的前缀。**必须与 Python 包名不同** | 微服务中的 `serviceName` |
| `[CORE_PACKAGE]` | `default` | 默认扫描路径，兜底注册 | glob pattern 的 fallback |
| `[CORE_PACKAGE]` | `agent`、`llm`、`tool` 等 | 各组件专属扫描路径，优先于 `default` | 各模块的独立 glob pattern |
| `[SUB_CONFIG_PATH]` | `custom_key_path` | 指向 `custom_key.toml` | `dotenv.config({ path: './.env' })` |
| `[SUB_CONFIG_PATH]` | `log_config_path` | 指向 `log_config.toml` | `winston.configure({...})` 配置文件路径 |
| `[DB]` | `system_db_uri` | 数据库连接串。留空→自动用 SQLite | `DATABASE_URL` 环境变量 |
| `[GUNICORN]` | `activate` | 是否用 Gunicorn，脚手架期 `'false'` | 是否用 PM2（开发期不需要） |
| `[GRPC]` | `activate` | 是否启用 gRPC，脚手架期 `'false'` | `@grpc/grpc-js` server.start |
| `[MONITOR]` | `activate` | 监控开关 | OTEL / Prometheus exporter 开关 |
| `[EXTENSION_MODULES]` | `class_list` | 扩展类全路径列表 | 插件注册表 / middleware 列表 |

**设计要点**：
- CORE_PACKAGE 下的路径必须是可以被 Python `import` 的**完整包路径**（如 `your_app.intelligence.agentic.agent`），类似于 Node.js 中 `require('@scope/pkg/lib')` 的路径格式。
- 即使某些组件类型暂时没有实例，也建议保留路径配置（指向已存在的 `__init__.py` 目录），方便后续扩展。
- `default` 路径是兜底：当某组件类型的专属扫描路径为空时，使用 `default` 路径。

### 3.5 `config/custom_key.toml` —— ≈ `.env` 文件

**作用**：存储 API Key，通过 `${VAR_NAME}` 语法在 TOML 和 YAML 配置中引用。

**与 Node.js `.env` 的关键区别**：
- `.env` 加载后通过 `process.env.VAR` 访问，是**运行时**访问。
- `custom_key.toml` 加载后，框架在配置解析时做**字符串替换**（类似 Webpack 的 `DefinePlugin` 做编译时替换，但这里是启动时替换）。
- 框架加载顺序：先加载 `custom_key.toml` → 再**重新解析**所有配置文件中的 `${...}` 占位符。这是两步加载机制。

Key 变量名约定：`OPENAI_API_KEY`、`DASHSCOPE_API_KEY`（Qwen）、`DEEPSEEK_API_KEY`、`ANTHROPIC_API_KEY`（Claude）等。对应 `_API_BASE`、`_PROXY` 等可选字段也一并配置。

### 3.6 `config/log_config.toml` —— ≈ winston/pino 配置对象

**作用**：配置 loguru 日志系统（Python 生态的主流日志库，功能 ≈ winston）。

关键字段：`log_level`（≈ `"info"`）、`log_path`（输出路径）、`log_rotation`（轮转策略 ≈ winston 的 `maxsize`）、`log_retention`（保留时间 ≈ winston 的 `maxFiles`）。

### 3.7 `config/config_extension.py` —— ≈ 生命周期钩子

**作用**：`ConfigExtension` 类在框架初始化阶段自动执行，等价于：
- Express: `app.use()` 注册的中间件
- NestJS: `OnModuleInit` / `OnApplicationBootstrap`
- Webpack: plugin 的 `apply()` 方法

约定：`__init__(self, configer: Configer)` 接受框架注入的配置对象（≈ 依赖注入）。即使不需要自定义逻辑，也保留空实现——类似保留一个空的 middleware 函数体等待填充。

### 3.8 `config/yaml_func_extension.py` —— ≈ Handlebars / EJS helpers

**作用**：让 YAML 配置中可以使用 `@FUNC(function_name(args))` 调用 Python 函数。

典型场景：`@FUNC(load_api_key('qwen'))` 从环境变量动态获取 API Key，而不是在 YAML 中硬编码。

**与 Node.js 模板引擎类比**：
- Handlebars: `{{helper_name arg}}` → `@FUNC(helper_name(arg))`
- EJS: `<%= helper(arg) %>` → `@FUNC(helper(arg))`

注意：由于每次读取 YAML 配置都会解析 `@FUNC()`，函数应加缓存（`@lru_cache` ≈ `memoizee` / `lru-cache` npm 包）。

### 3.9 `intelligence/agentic/llm/default_llm.toml` —— ≈ 默认 AI provider 声明

**作用**：声明默认 LLM 实例名。框架在扫描 LLM 前先读此文件，只有被 Agent 引用的 LLM 才会被实例化（≈ **按需加载 / tree-shaking**）。未被引用的 LLM 配置只缓存不实例化，避免浪费资源。

### 3.10 `bootstrap/intelligence/server_application.py` —— ≈ `server.js` / `bin/www`

**作用**：`ServerApplication` 类封装框架启动逻辑。三种运行模式对应 Node.js 中不同类型的入口：

| 模式 | agentUniverse | Node.js 类比 |
|------|-------------|-------------|
| 脚本/测试 | `AgentUniverse().start(core_mode=True)` | `node script.js`（跑完就退出） |
| Web 服务 | `start()` + `start_web_server()` | `node server.js`（Express 持续监听） |
| MCP 服务 | `start(core_mode=True)` + `MCPServerManager().start_server()` | gRPC/WebSocket 服务入口 |

---

## 四、初始化步骤清单

### 步骤 1：创建项目根目录和基础文件

- 创建 `your_app/` 目录（≈ `mkdir my-app && cd my-app && npm init`）
- 编写 `pyproject.toml`（包名、依赖 `agentUniverse`、Python `^3.10`、lint/type 工具链配置）
- 编写 `poetry.toml`（设置 `virtualenvs.in-project = true`）
- 编写 `.gitignore`（Python 标准忽略 + `custom_key.toml`）
- **在所有子目录创建空 `__init__.py`**（≈ 确保每个目录都可被 `require`）

### 步骤 2：创建 `config/` 目录

- 编写 `config.toml`：`appname` 不能与包名同名；配置各组件的包扫描路径；关闭 Gunicorn/gRPC/Monitor
- 编写 `custom_key.toml.sample`（提交 Git）并复制为 `custom_key.toml`（填 Key，不提交）
- 编写 `log_config.toml`
- 编写 `config_extension.py`（空 `__init__` 即可）
- 编写 `yaml_func_extension.py`（空类或含 `load_api_key` 函数）

### 步骤 3：创建 `intelligence/agentic/` 目录结构

- 按第二部分目录结构创建所有子目录和 `__init__.py`
- 创建 `intelligence/agentic/llm/default_llm.toml`，声明默认 LLM 名称

### 步骤 4：创建 `bootstrap/` 入口

- 编写 `bootstrap/intelligence/server_application.py`：`ServerApplication.start()` → `AgentUniverse().start(config_path='...config/config.toml')`

### 步骤 5：安装依赖 ≈ `npm install`

```bash
cd your_app
poetry install          # ≈ npm install（含 devDependencies）
```

### 步骤 6：验证启动 ≈ `npm start`

```bash
python bootstrap/intelligence/server_application.py
```

期望：控制台输出 framework banner + 各组件扫描路径日志，无报错。

---

## 五、关键设计约定（容易踩坑的地方）

1. **`appname` 命名空间**：`config.toml` 中 `appname` 是所有组件 `instance_code` 的前缀（`{appname}.{component_type}.{name}`）。**必须与 Python 包名不同**，否则产生命名冲突（类似 Node.js 中 `package.json` name 与内部模块路径同名导致 require 歧义）。

2. **包路径 import 规则**：框架将项目**父目录**加入 `sys.path`，因此 `config.toml` 中的 CORE_PACKAGE 路径必须以包名开始（`your_app.intelligence.agentic.llm`），不能是相对路径。类比：在 monorepo 中，需要确保 `require('@scope/pkg')` 能从 `node_modules` 或 workspaces 解析到。

3. **YAML 配置的 `metadata` 块**：每个组件 YAML 必须含 `metadata.type`（组件类型）和 `metadata.module`/`metadata.class`（Python 类路径）。这相当于 DI 容器中的 token + factory 声明。

4. **LLM 按需加载**：只有被 `default_llm.toml` 声明或被 Agent YAML 的 `profile.llm_model.name` 引用的 LLM 才会实例化。其余只缓存配置。这是一个内置的"懒加载"机制，避免启动时初始化无关 LLM。

5. **配置解析两阶段**：先加载 `custom_key.toml` → 再重新解析所有配置中的 `${VAR}`。这确保了 Key 的引用链路正确，类似 `dotenv` 必须先于配置加载执行。

6. **扫描优先级**：专属路径（如 `agent = [...]`）优先级高于 `default = [...]`。如果同名组件在两个路径都存在，专属路径的覆盖 `default` 的。

---

## 六、从脚手架到实际项目（添加第一个功能）

完成脚手架后，添加具体功能的路径和 Node.js 类比：

| 步骤 | 操作 | Node.js 类比 |
|------|------|-------------|
| 1. 添加 LLM | 在 `agentic/llm/` 下创建 YAML，引用框架内置 LLM 类 | 配置 AI SDK provider |
| 2. 添加 Agent | 在 `agentic/agent/` 下创建 YAML，关联 LLM、Tool 等 | 注册一个 DI service，注入依赖 |
| 3. 添加 Tool | 在 `agentic/tool/` 下创建 YAML 或编写自定义 Python 类 | 编写和注册一个工具函数（≈ MCP tool） |
| 4. 添加 Memory | 在 `agentic/memory/` 下创建 YAML | 配置会话/缓存服务 |
| 5. 添加 Prompt | 在 `agentic/prompt/` 下创建 YAML（支持版本化） | 模板管理（≈ 短信/邮件模板服务） |
| 6. 编写测试 | 在 `test/` 下通过 `AgentManager().get_instance_obj('name')` 获取 Agent 并调用 `run()` | `const agent = container.resolve('agent'); await agent.run()` |

---

## 七、常用命令速查表（Node.js → Python）

| 操作 | Node.js 命令 | Python/Poetry 命令 |
|------|-------------|-------------------|
| 安装依赖 | `npm install` | `poetry install` |
| 添加依赖 | `npm install pkg` | `poetry add pkg` |
| 添加开发依赖 | `npm install -D pkg` | `poetry add -G dev pkg` |
| 运行脚本 | `npm run build` | `poetry run python script.py` |
| 激活环境 | `nvm use` | `poetry shell`（进入 venv 子 shell） |
| Lint | `npx eslint .` | `poetry run ruff check .` |
| Lint 自动修复 | `npx eslint --fix .` | `poetry run ruff check --fix .` |
| 格式化 | `npx prettier --write .` | `poetry run ruff format .` 或 `poetry run black .` |
| 类型检查 | `npx tsc --noEmit` | `poetry run mypy your_app` |
| 运行测试 | `npx jest` | `poetry run pytest` |
| 过滤测试 | `npx jest -t "pattern"` | `poetry run pytest -k "pattern"` |
| 构建产物 | `npm run build` | `poetry build` |

---

## 八、参考资源

- 完整脚手架示例：`examples/sample_standard_app/`（对标 Node.js 的 full-stack boilerplate）
- 渐进式示例：`examples/startup_app/` 下 5 个递进示例（单 Agent → 多 Agent → +Memory → +Action → +Template）
- 框架启动源码：`agentuniverse/base/agentuniverse.py`（`start()` 方法的完整实现）
- 组件注册机制：`agentuniverse/base/component/`
