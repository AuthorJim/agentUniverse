# Poetry 实战指南：从 npm 思维到 Python 包管理

> **比喻先行：** Poetry 之于 Python，就像 npm/yarn 之于 Node.js。它管依赖、管虚拟环境、管构建、管发布——一站式解决 Python 项目"依赖地狱"的问题。
> **目标读者：** 熟悉 npm/yarn，但第一次认真使用 Poetry 的 Node.js 工程师。
> **预计时间：** 30 分钟阅读 + 动手操作。

---

## 0. 为什么 Python 需要 Poetry？

在 Node.js 世界，`npm install` 之后依赖进 `node_modules/`，`package.json` + `package-lock.json` 锁死版本——这一套你闭着眼都能操作。但 Python 的包管理历史是一部"战国史"：

```
Python 包管理演化（极简版）：

2008  pip + requirements.txt     ← 只有直接依赖，没有 lock 文件
2011  pip + virtualenv            ← 隔离环境，但 venv 和依赖分开管
2016  pipenv                       ← 学 npm，但慢得要命
2018  Poetry                       ← 学 yarn，终于好用了
2024  uv                           ← Rust 写的极速版（新兴力量）
```

**Poetry 是 Python 社区事实上的标准包管理器**（就像 npm 是 Node.js 的标配，虽然 yarn/pnpm 存在）。agentUniverse 就是用 Poetry 管理的。

### 0.1 一张图看懂：npm 和 Poetry 的对应关系

```
npm / yarn                          Poetry
──────────────────────────────      ──────────────────────────────
npm init -y                     →   poetry init / poetry new
package.json                    →   pyproject.toml
package-lock.json               →   poetry.lock
node_modules/                   →   {venv}/lib/python3.x/site-packages/
npm install                     →   poetry install
npm install <pkg>               →   poetry add <pkg>
npm install -D <pkg>            →   poetry add --group dev <pkg>
npm uninstall <pkg>             →   poetry remove <pkg>
npm update                      →   poetry update
npm run <script>                →   poetry run <command>
npx <command>                   →   poetry run <command>
npm exec                        →   poetry shell (进入 venv)
nvm                             →   pyenv (管理 Python 版本)
.env                            →   .venv/ (虚拟环境目录)
```

> 关键心理锚点：**把 `poetry add` 想象成 `npm install`，把 `pyproject.toml` 想象成 `package.json`。** 剩下的就是把 Python 世界的一些小差异搞清楚。

---

## 1. 安装 Poetry（一次性）

### 1.1 macOS / Linux

```bash
curl -sSL https://install.python-poetry.org | python3 -
```

安装完确认：

```bash
poetry --version   # 应该输出类似 Poetry (version 1.8.x)
```

### 1.2 配置（建议做的两件事）

```bash
# 1. 让虚拟环境创建在项目根目录下（而不是系统某个角落）
#    类比：把 node_modules 放在项目根目录
poetry config virtualenvs.in-project true

# 2. 查看当前所有配置
poetry config --list
```

> 设置 `virtualenvs.in-project true` 后，每次 `poetry install` 会在项目根目录生成 `.venv/` 文件夹。这样做的好处是：IDE（VSCode/PyCharm）能自动识别；删项目时一起删掉，不留系统垃圾。

---

## 2. 核心命令速查（npm → Poetry 对照表）

以下命令按使用频率排序，每个都配有 npm 对照和实际输出示例。

### 2.1 `poetry install` —— 安装所有依赖

```bash
# 等同于 npm install
poetry install

# 带可选依赖（如 pymilvus）
poetry install --extras store_ext
```

实际效果：
1. 读 `pyproject.toml` 算出依赖树
2. 读 `poetry.lock` 锁死版本（如果不存在则生成）
3. 创建/激活虚拟环境
4. 把包安装到虚拟环境中

### 2.2 `poetry add` —— 添加依赖

```bash
# 生产依赖   ≈ npm install <pkg>
poetry add requests

# 开发依赖   ≈ npm install -D <pkg>
poetry add --group dev pytest

# 可选依赖   ≈ npm install --save-optional <pkg>
poetry add --optional pymilvus

# 指定版本
poetry add requests@^2.32.0      # ^2.32.0（兼容 2.x）
poetry add requests@~2.32.0      # ~2.32.0（只升级补丁版本）
poetry add requests@2.32.0       # 精确版本
poetry add requests@latest       # 最新版
```

> **类比：** `^2.32.0` = npm 的 `^2.32.0`（主版本锁死，允许升级次版本和补丁版本）。
> `~2.32.0` = npm 的 `~2.32.0`（主次版本都锁死，只允许升级补丁）。

### 2.3 `poetry remove` —— 移除依赖

```bash
# 等同于 npm uninstall <pkg>
poetry remove requests
poetry remove --group dev pytest
```

### 2.4 `poetry update` —— 更新依赖

```bash
# 更新所有依赖到允许的最新版本   ≈ npm update
poetry update

# 只更新某一个包
poetry update requests

# 只更新开发依赖
poetry update --group dev

# 预览会更新什么，但不实际改（干跑）
poetry update --dry-run
```

### 2.5 `poetry show` —— 查看依赖

```bash
# 列出所有已安装的依赖    ≈ npm list --depth=0
poetry show

# 只看最顶层依赖（不看传递依赖）
poetry show --top-level

# 查看某个包的详细信息
poetry show requests

# 树状图显示依赖关系     ≈ npm ls
poetry show --tree

# 检查哪些依赖有更新
poetry show --outdated
```

### 2.6 `poetry run` —— 在虚拟环境中执行命令

```bash
# 这条命令让你不用手动激活虚拟环境，直接在里面执行任意命令
# 类比 npx：在项目环境中运行

# 运行测试
poetry run pytest

# 运行 lint
poetry run ruff check .

# 格式化代码
poetry run black .

# 类型检查
poetry run mypy agentuniverse

# 启动 Python 交互环境（带着你项目的所有依赖）
poetry run python

# 运行一个脚本
poetry run python examples/my_script.py
```

### 2.7 `poetry shell` —— 进入虚拟环境

```bash
# 激活虚拟环境（给当前 shell 打上"本项目的补丁"）
poetry shell

# 之后你可以直接用 python、pytest 等命令，不用加 poetry run
python --version      # 用的是 .venv 里的 Python
pytest                # 用的是 .venv 里的 pytest
```

> **和 Node.js 的差异：** Node.js 的 `node_modules/.bin/` 让 `npx` 自动找到本地工具。Python 没有自动找机制，必须要么 `poetry run`，要么先 `poetry shell` 激活环境。

---

## 3. 虚拟环境：Python 的 `node_modules` 替代品

这是 Node.js 工程师最容易困惑的地方，所以需要专门讲清楚。

### 3.1 Node.js 的方式 vs Python 的方式

```
Node.js：
  你全局装了一个 Node.js
  每个项目有自己的 node_modules/
  npx 自动找到 node_modules/.bin/ 里的工具
  
Python：
  你全局装了一个 Python（或 pyenv 装了多个版本）
  每个项目有自己的虚拟环境（.venv/）
  虚拟环境里有完整的 Python 解释器 + 所有依赖
  必须先激活虚拟环境，才能用到项目依赖
```

### 3.2 虚拟环境到底是什么？

一个虚拟环境就是一个独立的小 Python 世界。当你执行 `poetry install`，Poetry 在项目根目录创建 `.venv/`，里面长这样：

```
.venv/
├── bin/
│   ├── python          ← 指向系统 Python 的链接
│   ├── pip             ← 这个环境专属的 pip
│   ├── pytest          ← 所有可执行工具入口
│   └── ...
└── lib/
    └── python3.10/
        └── site-packages/
            ├── requests/       ← npm install 的包最终到这
            ├── flask/
            ├── pydantic/
            └── ...
```

> **直觉理解：** `.venv/` = `node_modules/` + `node` 二进制。它不仅存依赖，还存一个隔离的 Python 运行时。

### 3.3 检查你当前用的是哪个 Python

```bash
# 查看当前 Python 位置
which python
# 如果在 .venv 内，输出 /path/to/project/.venv/bin/python
# 如果在全局，输出 /usr/bin/python 或 /opt/homebrew/bin/python

# 查看当前虚拟环境变量
echo $VIRTUAL_ENV
# 激活后会有值，否则为空
```

### 3.4 IDE 如何识别虚拟环境

**VSCode：** 打开项目后，按 `Cmd+Shift+P` → "Python: Select Interpreter" → 选择 `.venv/bin/python`。之后终端会自动激活该环境。

**PyCharm：** 自动检测 `.venv/`，无需手动配置。

> 如果项目有 `.vscode/settings.json`，可以预设 interpreter 路径，团队共享。

---

## 4. 版本号语法：和 npm 的细微差异

`pyproject.toml` 里的版本约束用的是 [Semantic Versioning](https://semver.org/)，和 npm 一样，但写法略有不同：

```
npm                     Poetry             含义
─────────────────────   ───────────────    ──────────────────────
"^2.32.0"               "^2.32.0"          兼容 2.x（>=2.32.0, <3.0.0）
"~2.32.0"               "~2.32.0"          兼容 2.32.x（>=2.32.0, <2.33.0）
"2.32.0"                "2.32.0"           精确版本（只有 2.32.0）
">=2.0.0 <3.0.0"        ">=2.0.0,<3.0.0"  手动指定范围（注意 Python 用逗号）
"*"                     "*"                任意版本
"latest"                "latest"           最新版（仅 poetry add 时用）
```

> npm 的范围写法："^1.0.0 || ^2.0.0" → Poetry："^1.0.0 || ^2.0.0"（一样）

agentUniverse 实际例子：

```toml
[tool.poetry.dependencies]
python = "^3.10"              # 只要 >=3.10，小版本无所谓
requests = "^2.32.0"          # 只要 2.x，>=2.32.0
pydantic = "^2.6.4"           # 只要 2.x，>=2.6.4
grpcio = "1.63.0"             # 锁死在 1.63.0
```

---

## 5. poetry.lock：比 package-lock.json 更重要的文件

### 5.1 一个根本区别

```
npm:  package-lock.json 只有在你真正 npm install 时才有用
      你可以删掉它再 npm install，自动重新生成

Poetry: poetry.lock 是唯一的真理来源
        poetry install 不看 pyproject.toml 的版本范围，看 poetry.lock 的精确版本
```

**poetry.lock 必须提交到 Git。** 它锁死了每个依赖的精确版本和哈希值，保证团队所有人安装的依赖完全一致。

### 5.2 两个关键操作的区别

```bash
poetry install    # 按 poetry.lock 装（锁死）
poetry update     # 先按 pyproject.toml 版本范围更新 poetry.lock，再按新 lock 装
```

> **记忆口诀：**
> `install` = "给我和上次一模一样的"（看书架上的清单，照单抓药）
> `update` = "按我的范围要求，看看有没有更新版可用"（重新调研市场，更新清单，然后抓药）

---

## 6. 日常工作流（做这些事的频率最高）

### 6.1 克隆项目后的第一次

```bash
git clone <repo-url>
cd agentUniverse

# 安装所有依赖（生产 + 开发 + store_ext 可选）
poetry install --extras store_ext

# 验证
poetry run pytest                    # 跑测试确认环境正常
poetry run python -c "import agentuniverse; print('OK')"
```

### 6.2 日常开发

```bash
# 加一个新依赖
poetry add httpx

# 加一个开发工具
poetry add --group dev pytest-watch

# 跑测试
poetry run pytest

# 跑单个测试文件
poetry run pytest tests/test_something.py

# 格式化代码
poetry run black .

# Lint
poetry run ruff check .

# 类型检查
poetry run mypy agentuniverse
```

### 6.3 Git 提交前

```bash
# 运行所有检查
poetry run pre-commit run --all-files

# 或者直接用 agentUniverse 配置好的 pre-commit 钩子
pre-commit run   # 如果在 poetry shell 里
```

### 6.4 更新依赖

```bash
# 看看哪些依赖有更新
poetry show --outdated

# 干跑，看会更新什么
poetry update --dry-run

# 实际更新
poetry update

# 跑测试确认没坏
poetry run pytest
```

---

## 7. 常见坑与解决方案

### 7.1 "command not found: poetry"

```bash
# 检查 Poetry 装在哪
which poetry

# 大概率没装，或路径没加 PATH
# Poetry 默认安装位置：~/.local/bin/poetry
export PATH="$HOME/.local/bin:$PATH"

# 永久生效：把上面那行加到 ~/.zshrc
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### 7.2 "Python version mismatch" / 依赖解析失败

```bash
# 确认系统 Python 版本
python3 --version

# agentUniverse 要求 Python 3.10+，推荐用 pyenv 管理
pyenv install 3.10.14
pyenv local 3.10.14

# 确认 Poetry 用的是 pyenv 版本
poetry env info
```

### 7.3 `poetry install` 非常慢

```bash
# 这个项目配置了国内镜像源，确认已生效：
poetry config --list | grep source

# 如果没配，手动加阿里云镜像：
poetry source add --priority=primary china https://mirrors.aliyun.com/pypi/simple/
```

### 7.4 "venv 在哪？"——找不到虚拟环境

```bash
# 查看虚拟环境信息
poetry env info

# 输出示例：
# Virtualenv
# Python:         3.10.14
# Path:           /path/to/project/.venv
# Executable:     /path/to/project/.venv/bin/python

# 如果虚拟环境不在项目根目录（没设 virtualenvs.in-project）：
# 通常在 ~/Library/Caches/pypoetry/virtualenvs/（macOS）或 ~/.cache/...（Linux）

# 查看所有虚拟环境
poetry env list
```

### 7.5 依赖冲突 / lock 文件问题

```bash
# 如果 poetry.lock 有问题，重新生成：
rm poetry.lock
poetry install

# 如果某个包安装有问题：
poetry remove <pkg>
poetry add <pkg>
```

---

## 8. agentUniverse 项目的 Poetry 配置解读

打开项目根目录的 `pyproject.toml`，从 Node.js 视角逐段解读：

```toml
# 第一部分：项目元数据（≈ package.json 前几行）
[tool.poetry]
name = "agentUniverse"
version = "0.0.19"
description = "agentUniverse is a framework for developing applications..."

# 第二部分：哪些目录属于这个包（≈ package.json 的 files 字段）
packages = [
    { include = "agentuniverse" },
    { include = "agentuniverse_connector" },
    { include = "agentuniverse_extension" },
    { include = "agentuniverse_product" },
]
include = ["*.yaml"]    # 额外把 YAML 文件也打进包

# 第三部分：生产依赖（≈ dependencies）
[tool.poetry.dependencies]
python = "^3.10"
requests = "^2.32.0"
flask = "^2.3.2"
# ... 等等

# 第四部分：可选依赖（≈ optionalDependencies）
[tool.poetry.extras]
store_ext = ["pymilvus"]     # poetry install --extras store_ext 时才装

# 第五部分：开发依赖（≈ devDependencies）
# 注意语法差异：npm 是不同字段名，Poetry 是用 group 组织
[tool.poetry.group.dev.dependencies]
pytest = "^7.2.0"
pre-commit = "^2.20.0"
# ...

# 第六部分：镜像源（≈ .npmrc 里 registry=https://...）
[[tool.poetry.source]]
name = "china"
url = "https://mirrors.aliyun.com/pypi/simple/"
priority = "primary"

# 第七部分：构建系统（≈ npm 的 build script 配置）
[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

> 更详细的 `pyproject.toml` 逐行解读，参见 [pyproject.toml 完全指南](pyproject-toml-guide.md)。

---

## 9. 和 npm 的一些"玄学"差异（日常操作注意事项）

### 9.1 `poetry add` 比 `npm install` 慢得多

npm 从 registry 下载 tarball 就行。Poetry 还要**解析整个依赖树、检查版本兼容、生成/更新 lock 文件**。因为 Python 没有 npm 那样的"扁平化 node_modules"算法，依赖树解析复杂得多。

> **缓解：** 配置国内镜像源（本项目已配好），或使用 `uv` 替代（新兴的 Rust 包管理器，速度是 Poetry 的 10-100 倍）。

### 9.2 没有 `npm scripts` 的直接等价物

`pyproject.toml` 里有 `[tool.poetry.scripts]`，但那是定义**命令行入口点**的（类似于 npm 的 `bin` 字段），不是 `scripts`（快捷别名）。

```bash
# Node.js：npm run test  → 执行 package.json 里的 scripts.test
# Python：没有这个机制。你需要：
poetry run pytest      # 直接写完整命令

# 或者用 Makefile 充当 scripts 角色
# 项目里可以创建 Makefile：
#   test:
#       poetry run pytest
# 然后 make test
```

### 9.3 全局安装 vs 本地安装

```bash
# Node.js: npm install -g typescript  →  全局可用 tsc 命令
# Python: pip install --user <pkg>    →  全局可用（不推荐）
#         poetry add <pkg>            →  只在当前项目可用

# Poetry 的原则：所有依赖都是项目级的，不鼓励全局安装
# 需要全局工具（如 poetry 自身），用 pipx 安装：
pipx install poetry
```

---

## 10. 速查表：打印贴在显示器旁边

```
┌─────────────────────────────────────────────────────────────────┐
│                    Poetry 日常命令速查                           │
├───────────────────────────────────┬─────────────────────────────┤
│  要做的事                         │  命令                       │
├───────────────────────────────────┼─────────────────────────────┤
│  安装所有依赖                     │  poetry install             │
│  加生产依赖                       │  poetry add <pkg>           │
│  加开发依赖                       │  poetry add --group dev <p> │
│  加可选依赖                       │  poetry add --optional <p>  │
│  移除依赖                         │  poetry remove <pkg>        │
│  更新依赖                         │  poetry update              │
│  看依赖列表                       │  poetry show --tree         │
│  看哪些过时了                     │  poetry show --outdated     │
│  在 venv 里执行命令               │  poetry run <cmd>           │
│  进入 venv shell                 │  poetry shell               │
│  查看 venv 信息                   │  poetry env info            │
│  构建分发包                       │  poetry build               │
│  发布到 PyPI                      │  poetry publish             │
└───────────────────────────────────┴─────────────────────────────┘
```

---

## 下一步

- 读完本文后，建议打开 [pyproject.toml 完全指南](pyproject-toml-guide.md) 深入理解项目的配置文件结构。
- 如果你还没安装 Python 环境，推荐用 `pyenv` 管理版本（2 分钟搞定）：
  ```bash
  brew install pyenv              # 装 pyenv
  pyenv install 3.10.14          # 装 Python
  pyenv local 3.10.14            # 当前项目使用这个版本
  ```
- 返回 [学习路线图](../README.md) 继续学习计划。
