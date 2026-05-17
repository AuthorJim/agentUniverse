# pyproject.toml 完全指南：写给 Node.js 工程师

> **比喻先行：** 如果把 Python 项目比作一个人，`pyproject.toml` 就是他的 **身份证 + 体检报告 + 工具箱清单** 三位一体。它告诉全世界：我是谁、我需要什么才能活、我有哪些工具、我有什么毛病不能容忍。

在 Node.js 世界里，这些信息散落在 `package.json`、`.eslintrc`、`.prettierrc`、`tsconfig.json`、`.npmrc` 等多个文件中。Python 社区从 2016 年（PEP 518）开始推动 "大一统"——把所有配置收归到 `pyproject.toml` 一个文件里。这就是它的底层哲学：**一个项目，一个配置文件**。

> 本文以 agentUniverse 项目的真实 `pyproject.toml` 为解剖样本，所有代码片段均来自项目根目录下的原文件。

---

## 一、全景对照：pyproject.toml vs package.json

先把两张"身份证"并排放在一起，感受一下整体气质差异：

```
┌─────────────────────────────────────────────────────────────────┐
│                    package.json (Node.js)                        │
│  ┌─────────┐ ┌──────────┐ ┌───────────┐ ┌──────────────────┐   │
│  │  name   │ │ version  │ │  scripts  │ │  dependencies    │   │
│  │  main   │ │ license  │ │  engines  │ │  devDependencies │   │
│  └─────────┘ └──────────┘ └───────────┘ └──────────────────┘   │
│                                                                  │
│  其他配置散落各地：                                               │
│  .eslintrc.js  .prettierrc  tsconfig.json  jest.config.js ...   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                  pyproject.toml (Python/Poetry)                  │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────┐   │
│  │ [tool.poetry]│ │[tool.poetry.deps]│ │ [tool.ruff]       │   │
│  │  name,ver... │ │  = dependencies  │ │  = .eslintrc      │   │
│  └──────────────┘ └──────────────────┘ └───────────────────┘   │
│  ┌──────────────┐ ┌──────────────────┐ ┌───────────────────┐   │
│  │ [tool.black] │ │  [tool.mypy]     │ │ [build-system]    │   │
│  │  = .prettierc│ │  = tsconfig.json │ │  = build script   │   │
│  └──────────────┘ └──────────────────┘ └───────────────────┘   │
│                                                                  │
│  一个文件，统治所有。                                              │
└─────────────────────────────────────────────────────────────────┘
```

**核心字段映射表：**

| 你要做的事 | package.json 里放哪 | pyproject.toml 里放哪 |
|-----------|-------------------|---------------------|
| 项目名 + 版本 | `name`, `version` | `[tool.poetry]` → `name`, `version` |
| 生产依赖 | `dependencies` | `[tool.poetry.dependencies]` |
| 开发依赖 | `devDependencies` | `[tool.poetry.group.dev.dependencies]` |
| 可选依赖 | `optionalDependencies` | `[tool.poetry.extras]` + `optional = true` |
| 运行脚本 | `scripts` | `[tool.poetry.scripts]`（入口点），或用 Makefile |
| Node/Python 版本 | `engines` | `[tool.poetry.dependencies]` → `python = "^3.10"` |
| registry 源 | `.npmrc` → `registry=...` | `[[tool.poetry.source]]` |
| 构建工具 | `build` script / bundler | `[build-system]` |
| ESLint 配置 | `.eslintrc` | `[tool.ruff]` 或 `[tool.ruff.lint]` |
| Prettier 配置 | `.prettierrc` | `[tool.black]` |
| TypeScript 配置 | `tsconfig.json` | `[tool.mypy]` |
| Jest 配置 | `jest.config.js` 或 `jest` 字段 | `[tool.pytest.ini_options]` 或 `pytest.ini` |
| Coverage 配置 | `jest` → `coverageThreshold` | `[tool.coverage.run]` |
| 项目文件范围 | `files` | `packages` + `include` |

---

## 二、逐段解剖 agentUniverse 的 pyproject.toml

下面用项目的真实配置逐段讲解。你可以打开根目录的 `pyproject.toml` 对照着看。

### 2.1 `[tool.poetry]` — 项目的"身份证"

```toml
[tool.poetry]
name = "agentUniverse"
version = "0.0.19"
description = "agentUniverse is a framework for ..."
authors = ["AntGroup <jerry.zzw@antgroup.com>"]
repository = "https://github.com/agentuniverse-ai/agentUniverse"
readme = "README_PYPI.md"
```

**对照 package.json：**

```json
{
  "name": "agentUniverse",
  "version": "0.0.19",
  "description": "agentUniverse is a framework for ...",
  "author": "AntGroup <jerry.zzw@antgroup.com>",
  "repository": "https://github.com/agentuniverse-ai/agentUniverse",
  "readme": "README_PYPI.md"
}
```

几乎一一对应。但有两个 Python 独有概念值得展开：

#### `packages` — 哪些目录属于这个包？

```toml
packages = [
    { include = "agentuniverse" },
    { include = "agentuniverse_connector" },
    { include = "agentuniverse_extension" },
    { include = "agentuniverse_product" },
]
include = ["*.yaml"]
```

**比喻：** `packages` 就像你搬家时告诉搬家公司 "这几箱是我的"。Poetry 构建时会把这些目录打包进最终产物。

- `packages` ≈ `package.json` 的 `"files"` 字段——指定发布时包含哪些文件。
- `include = ["*.yaml"]` 是一个 Python 特有的需求：**默认情况下，Poetry 只打包 `.py` 文件**。但 agentUniverse 的配置全部写在 `.yaml` 文件里，如果不加这一行，发布出去的包就是个空壳——所有 Agent 配置都丢了。这就像你把 `.vue` 或 `.jsx` 文件 exclude 了，组件全没了。

#### `classifiers` — 给 PyPI 看的标签

```toml
classifiers = [
    "Programming Language :: Python :: 3.10",
    "License :: OSI Approved :: Apache Software License",
]
```

这在 npm 生态中没有直接对应物（npm 用 `keywords` 数组），PyPI 的分类器是一个**受控词表**——你只能从官方列表中选择，不能自创。类似于 npm 的 `"license": "Apache-2.0"` 但更结构化。

---

### 2.2 `[tool.poetry.dependencies]` — "我饿了，要吃饭"

```toml
[tool.poetry.dependencies]
python = "^3.10"
requests = "^2.32.0"
flask = "^2.3.2"
langchain = "0.1.20"
openai = '1.55.3'
loguru = '0.7.2'
pydantic = "^2.6.4"
```

**对照 package.json：**

```json
{
  "dependencies": {
    "express": "^4.18.0",
    "axios": "^1.6.0"
  },
  "engines": {
    "node": ">=18"
  }
}
```

#### 关键差异点

**差异 1：`python = "^3.10"` 写在 dependencies 里**

在 Node.js 中，`engines` 是一个独立字段。在 Python/Poetry 中，Python 版本限制**本身就是一个依赖项**。这很直观——"我要用 Python 解释器，版本至少 3.10"。`^3.10` 含义：`>=3.10, <4.0`。

**差异 2：版本约束语法**

| 写法 | 含义 | npm 等价 |
|------|------|---------|
| `"^2.32.0"` | `>=2.32.0, <3.0.0` | 同 npm 的 `^` |
| `"~1.15.1"` | `>=1.15.1, <1.16.0` | 同 npm 的 `~` |
| `"0.1.20"`（无前缀） | **精确锁定** `==0.1.20` | npm 的 `"0.1.20"` 其实是 `^` |
| `">=0.27.2"` | 大于等于 | 同 npm |
| `"<1.22.0"` | 小于（不包含） | 同 npm |
| `">=2.4.0,<3.0.0"` | 范围约束（AND） | 同 npm 的 `">=2.4.0 <3.0.0"` |

**踩坑预警：** `langchain = "0.1.20"` 在 npm 里等于 `^0.1.20`（允许小版本浮动），但在 Poetry 里等于 `==0.1.20`（**精确锁定，纹丝不动**）。这是两个生态最容易被忽视的差异——npm 默认宽松，Poetry 无前缀默认严格。

**差异 3：optional dependencies 的声明方式不同**

```toml
[tool.poetry.dependencies]
aliyun-log-python-sdk = { version = "0.8.8", optional = true }
pymilvus = { version = "^2.4.3", optional = true }

[tool.poetry.extras]
log_ext = ["aliyun-log-python-sdk"]
store_ext = ["pymilvus"]
```

在 npm 中，可选依赖直接放 `optionalDependencies` 里就完事了。Poetry 则需要**两步**：
1. 先在 `dependencies` 中标记 `optional = true`
2. 再在 `[tool.poetry.extras]` 中分组命名

**类比：** 想象你在餐厅点套餐。`optional = true` 意思是 "这道菜不强制点"。`extras` 是套餐名——"海鲜套餐 = 龙虾 + 鲍鱼"，"素食套餐 = 沙拉 + 豆腐"。用户安装时通过 `poetry install --extras store_ext` 选择套餐。这比 npm 的 `optionalDependencies` 更灵活，可以组合命名。

---

### 2.3 `[tool.poetry.group.dev.dependencies]` — "我的工具箱"

```toml
[tool.poetry.group.dev.dependencies]
pytest = "^7.2.0"
pytest-cov = "^4.0.0"
pre-commit = "^2.20.0"
```

**对照 package.json：**

```json
{
  "devDependencies": {
    "jest": "^29.0.0",
    "prettier": "^3.0.0"
  }
}
```

`group.dev` 是 Poetry 的依赖分组机制。你可以自创更多 group：

```toml
[tool.poetry.group.docs.dependencies]    # 构建文档用的
sphinx = "^7.0"

[tool.poetry.group.test.dependencies]    # 测试用的（如果不想全放 dev）
pytest = "^7.2.0"
```

**类比：** npm 里你可能会用 `--save-dev` vs `--save` 来区分，Poetry 用 `-G dev` vs 无参数 `add`。group 相当于 npm workspaces 中不同 package 的依赖分离，只是这里在一个 pyproject.toml 内完成。

安装时：
```bash
poetry install                    # ≈ npm install（包含 dev）
poetry install --without dev      # ≈ npm install --production
poetry install --with docs        # 安装 main + docs group
```

---

### 2.4 `[[tool.poetry.source]]` — "我去哪买菜"

```toml
[[tool.poetry.source]]
name = "china"
url = "https://mirrors.aliyun.com/pypi/simple/"
priority = "primary"
```

**对照：** `.npmrc` 中的 `registry=https://registry.npmmirror.com`

注意：`[[double-brackets]]` 是 TOML 语法的**数组嵌套表**标记——表示这是一个数组，可以有多项。用 bracket 数区分层级：

| TOML 写法 | 含义 | JSON 等价 |
|-----------|------|----------|
| `[section]` | 单层表 | `{ "section": {} }` |
| `[[array]]` | 数组中的元素 | `{ "array": [{...}] }` |
| `[[[nested]]]` | 数组中嵌套数组 | `{ "nested": [[{...}]] }` |

agentUniverse 配了阿里云镜像源，并把 `priority` 设为 `"primary"`（优先使用）。对于国内开发者这相当于 npm 的 `registry=https://registry.npmmirror.com`，解决网络问题。

---

### 2.5 `[build-system]` — "谁负责打包"

```toml
[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

**对照：** `package.json` 的 `"build"` 脚本 + bundler 配置（如 webpack/rollup/tsup）。

`build-backend` 指定了**谁来把源码变成可发布的包**。`poetry.core.masonry.api` 是 Poetry 的构建引擎。

**类比：** 如果你用 Vite 构建前端项目，`build-backend` 就相当于 `"build": "vite build"`；用 tsc 的话相当于 `"build": "tsc"`。Python 社区常见的构建后端有：
- `poetry.core` — Poetry 的，最流行
- `setuptools` — 老牌，相当于 webpack（功能全但配置重）
- `hatchling` — Hatch 的，相当于 tsup（轻量快速）
- `flit_core` — Flit 的，相当于 esbuild（极简极快）

---

### 2.6 `[tool.xxx]` — 工具配置，全收进来

这是 `pyproject.toml` 真正展现 "大一统" 哲学的部分。所有开发工具的配置都放在 `[tool.工具名]` 段落下。

#### `[tool.black]` ≈ `.prettierrc`

```toml
[tool.black]
line-length = 120
target-version = ['py310']
```

Prettier 对应的配置：
```json
{ "printWidth": 120 }
```

#### `[tool.mypy]` ≈ `tsconfig.json`

```toml
[tool.mypy]
files = ["agentuniverse"]          # ≈ "include": ["src"]
disallow_untyped_defs = "True"     # ≈ "noImplicitAny": true
no_implicit_optional = "True"      # ≈ "strictOptionalTypes": true
show_error_codes = "True"          # ≈ 显示 TS 错误码
```

**核心类比：** mypy 之于 Python ≈ TypeScript 编译器之于 JavaScript。都提供可选的静态类型检查。区别在于：
- TypeScript 是**独立语言**，需要编译成 JS。
- Python type hints 是**纯标注**，不编译、不影响运行时。mypy 只是一个"检查器"（类似 `tsc --noEmit`）。

#### `[tool.ruff]` ≈ `.eslintrc`

```toml
[tool.ruff]
target-version = "py310"
line-length = 120

[tool.ruff.lint]
select = ["YTT", "S", "B", "A", "SIM", "I", "E", "W", "F", "RUF", "TRY"]
ignore = ["E501", "E731"]

[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]
```

ruff 相当于 ESLint，但用 Rust 重写所以**快 10-100 倍**。每个字母代码是一组规则：
- `E`, `W` = pycodestyle 规则（风格检查）
- `F` = pyflakes 规则（逻辑错误检测）
- `B` = flake8-bugbear 规则（常见 bug 模式）
- `SIM` = 简化代码建议（≈ ESLint 的 `no-unnecessary-else` 之类）
- `S` = 安全检查（≈ ESLint 的 security 插件）

`per-file-ignores` 类似于 ESLint 的 `overrides`，对测试文件放宽某些规则。

#### `[tool.coverage.run]` ≈ Jest coverage 配置

```toml
[tool.coverage.run]
branch = true
source = ["agentuniverse"]
```

等价 Jest 配置：
```json
{ "collectCoverageFrom": ["agentuniverse/**"] }
```

---

## 三、poetry.lock — 不只是 package-lock.json

`poetry.lock` 记录了**所有依赖的精确版本 + 文件哈希**。首次调用 `poetry install` 时自动生成，之后每次 `poetry add`/`poetry update` 会更新。

打开它，你会发现它记录的信息比 `package-lock.json` 更细致：

```
[[package]]
name = "aiohttp"
version = "3.13.5"
description = "Async http client/server framework (asyncio)"
optional = false
python-versions = ">=3.9"
groups = ["main"]
files = [
    {file = "aiohttp-3.13.5-cp310-cp310-macosx_10_9_universal2.whl", hash = "sha256:..."},
    {file = "aiohttp-3.13.5.tar.gz", hash = "sha256:..."},
]
```

**关键差异：**

| 维度 | package-lock.json | poetry.lock |
|------|------------------|-------------|
| 文件哈希 | SHA-1（integrity） | SHA-256 |
| 平台兼容标记 | 无（靠 optionalDependencies） | `python-versions` 字段标记 Python 版本兼容性 |
| 源码/二进制区分 | 不区分 | `.whl`（预编译二进制）和 `.tar.gz`（源码）分开记录 |
| 依赖分组 | 无 | `groups = ["main"]` 或 `["dev"]` 标记 |

**`提交锁文件` vs `不提交锁文件`：** npm 世界有争议（库不提交，应用提交）。Python 社区共识更明确：**应用提交，库也建议提交**。Poetry 文档明确推荐提交 `poetry.lock`，因为它的哈希校验能防止供应链攻击。

---

## 四、常用命令对照速查

| 操作 | npm 命令 | Poetry 命令 |
|------|---------|------------|
| 初始化项目 | `npm init` | `poetry init`（交互式） |
| 安装所有依赖 | `npm install` | `poetry install` |
| 添加生产依赖 | `npm install pkg` | `poetry add pkg` |
| 添加开发依赖 | `npm install -D pkg` | `poetry add -G dev pkg` |
| 移除依赖 | `npm uninstall pkg` | `poetry remove pkg` |
| 更新依赖 | `npm update` | `poetry update` |
| 更新单个包 | `npm update pkg` | `poetry update pkg` |
| 生产环境安装 | `npm install --production` | `poetry install --without dev` |
| 安装可选依赖 | 自动安装（optionalDependencies） | `poetry install --extras store_ext` |
| 查看已安装 | `npm ls` | `poetry show --tree` |
| 查看过时依赖 | `npm outdated` | `poetry show --outdated` |
| 运行脚本 | `npm run build` | `poetry run python script.py` |
| 进入环境 | `npx ...` | `poetry shell`（进入 venv 子 shell） |
| 发布 | `npm publish` | `poetry publish` |
| 构建产物 | `npm pack` | `poetry build` |

---

## 五、POETRY 和 PIP 的关系

Node.js 开发者最常问的一个问题：**Poetry 和 pip 到底什么关系？**

**一句话回答：** pip 之于 Poetry，就像 `npm install --no-save` 之于 yarn/pnpm——一个只管下载，一个管理项目生命周期。

```
Node.js 世界：
  npm (包管理器)  →  下载包到 node_modules/
  yarn/pnpm        →  下载包 + 管理锁文件 + workspace 等高级功能

Python 世界：
  pip (包安装器)   →  下载包到 site-packages/
  Poetry            →  下载包 + 管理锁文件 + 虚拟环境 + 依赖解析 + 构建 + 发布
```

Poetry 内部**仍然调用 pip** 来下载包，但它多做了一层：
1. **依赖解析**（pip 也能做，但 Poetry 的解析器更快更准确）
2. **锁文件生成**（pip 的 `pip freeze > requirements.txt` 非常粗糙）
3. **虚拟环境管理**（pip 需要搭配 `venv`/`virtualenv` 手动管理）
4. **构建和发布**（pip 需要搭配 `setuptools`/`twine` 等额外工具）

**如果你只想快速试一个包**，用 `pip install xxx` 就行（就像偶尔用 `npx xxx`）。**如果你在做一个正式项目**，用 Poetry（就像用 `npm init` + `npm install` 正式搭建项目）。

---

## 六、总结：一张图说清 pyproject.toml 的"大一统"哲学

```
  Node.js 方式（配置分散）               Python/Poetry 方式（配置集中）

  ┌──────────┐  ┌───────────┐         ┌──────────────────────────┐
  │package.json│  │ .eslintrc │         │     pyproject.toml       │
  │  name     │  │  rules    │         │                          │
  │  deps    │  └───────────┘         │ [tool.poetry]  ← 项目信息 │
  │  scripts │                        │ [tool.poetry.dependencies]│
  └──────────┘  ┌───────────┐         │ [tool.ruff]    ← ESLint   │
                │.prettierrc│         │ [tool.black]   ← Prettier │
  ┌──────────┐  │  options  │         │ [tool.mypy]    ← tsconfig │
  │tsconfig  │  └───────────┘         │ [tool.coverage]← Jest cov │
  │  options │                        │ [build-system] ← bundler  │
  └──────────┘  ┌───────────┐         └──────────────────────────┘
                │jest.config│
  ┌──────────┐  │  options  │             一个文件包含所有
  │ .npmrc   │  └───────────┘
  │ registry │
  └──────────┘
```

这种 "大一统" 的好处：
- **新成员入职**：看一个文件就知道项目全貌
- **工具版本升级**：改一处，相关工具都能感知
- **CI 配置简洁**：不用维护十几个 config 文件路径

代价是：文件会变得比较长（agentUniverse 的这个有 160+ 行）。但 Python 社区认为，**一个长文件比十几个短文件更好管理**。

---

> **下一篇建议：** 如果你理解了 pyproject.toml，下一步可以看 `config/config.toml`——agentUniverse 的主配置文件。两兄弟一个管项目（pyproject.toml），一个管框架（config.toml），职责分明。
>
> 回到学习路径：[../README.md](../README.md)
