# Ruff 完全指南：从零集成到项目

> **比喻先行：** Ruff 就像你请了一位"代码管家"——他既是挑剔的 ESLint（揪出你的坏习惯），又是固执的 Prettier（把你的代码格式化成标准模样）。但这位管家速度极快，因为他脑子里装的是 Rust 引擎，而不是 JavaScript。

在 Node.js 世界里，代码质量工具是分裂的：ESLint 管逻辑、Prettier 管格式、还要配一堆插件和 parser。Ruff 的野心是**一个工具，统治 lint + format**。它用 Rust 重写了 Python 社区几十个经典检查工具，速度比传统方案快 10-100 倍。

> 本文以 agentUniverse 项目的真实配置为样本，带你从零开始把 Ruff 集成到 Python 项目中。

---

## 一、Ruff 是什么？一张图说清

```
  Node.js 方式（工具分裂）                  Python Ruff 方式（工具统一）

  ┌──────────────┐                           ┌─────────────────────────┐
  │   ESLint      │  ← 逻辑检查              │       Ruff              │
  │   + 20 plugins│                          │                         │
  └──────────────┘                           │  ruff check  ← lint     │
  ┌──────────────┐                           │  ruff format ← format   │
  │   Prettier    │  ← 代码格式化             │                         │
  └──────────────┘                           │  一个二进制，全搞定。     │
  ┌──────────────┐                           └─────────────────────────┘
  │  import sort  │  ← import 排序
  └──────────────┘
```

**Ruff 内部"吞并"了哪些 Python 工具？**

Ruff 的每个字母规则代码对应一个上游工具的同类规则：

| 规则前缀 | 上游工具 | 做什么 | Node.js 类比 |
|---------|---------|--------|-------------|
| `E`, `W` | pycodestyle | PEP 8 风格检查 | ESLint 的 `stylistic` 规则 |
| `F` | pyflakes | 逻辑错误检测（未定义变量等） | ESLint 的 `no-undef` 等 |
| `B` | flake8-bugbear | 常见 bug 模式 | ESLint 的 `@typescript-eslint/no-unused-vars` |
| `SIM` | flake8-simplify | 简化代码建议 | ESLint 规则不建议写冗余代码 |
| `I` | isort | import 语句排序 | `eslint-plugin-import` 的 `order` 规则 |
| `S` | flake8-bandit | 安全检查 | ESLint 的 `eslint-plugin-security` |
| `UP` | pyupgrade | 自动升级到新语法 | 类似于 `tsc` 的 `target` 升级 |
| `C4` | flake8-comprehensions | 推导式优化建议 | — |
| `T10` | flake8-debugger | 禁止提交 debug 语句 | ESLint 的 `no-debugger` |
| `TRY` | tryceratops | try/except 块规范 | — |
| `RUF` | Ruff 自己的规则 | Ruff 专属检查 | ESLint 的 `eslint:recommended` |

---

## 二、第一步：安装 Ruff

### 2.1 全局安装（推荐先试试）

```bash
pip install ruff
```

装完验证：

```bash
ruff --version
# ruff 0.8.4
```

**对比：** 相当于 `npm install -g eslint prettier`。

### 2.2 项目级安装（正式项目应该这样做）

```bash
# 用 Poetry（推荐）
poetry add -G dev ruff

# 或用 pip
pip install ruff
```

在 `pyproject.toml` 中会看到：

```toml
[tool.poetry.group.dev.dependencies]
ruff = "^0.8.4"
```

**比喻：** 全局安装等于在你电脑上装了个瑞士军刀（随时可用，但跟项目没关系），项目安装等于把瑞士军刀写进了项目的"必备工具清单"——其他克隆这个项目的人执行 `poetry install` 也会自动装上这把刀。

---

## 三、第二步：在 pyproject.toml 中配置 Ruff

Ruff 配置写在 `pyproject.toml` 的 `[tool.ruff]` 段落下（也可以单独写 `ruff.toml`，但集中管理更好）。

### 3.1 最小配置：开箱即用

```toml
[tool.ruff]
target-version = "py310"   # 目标 Python 版本
line-length = 120           # 每行最大字符数
```

就这两行，`ruff check` 已经可以跑了。**比喻：** 这就像 ESLint 的 `extends: ["eslint:recommended"]`——不配置也能用，只是用默认规则集。

### 3.2 实战配置：agentUniverse 的真实配置

agentUniverse 的 ruff 配置分三层，下面是逐层拆解：

#### 第一层：`[tool.ruff]` — 基础设定

```toml
[tool.ruff]
target-version = "py310"   # 生成的代码兼容 Python 3.10+
line-length = 120           # 每行 120 字符（比 PEP 8 默认的 79 宽松）
fix = true                  # ruff check 时自动修复可修问题
```

`fix = true` 的含义：运行 `ruff check` 时，能自动修的自动修，不能修的才报出来。相当于 `eslint --fix` 默认开启。

#### 第二层：`[tool.ruff.lint]` — 规则开关

```toml
[tool.ruff.lint]
select = [
    "YTT",   # flake8-2020: 禁止已弃用的 Python 版本检查
    "S",     # flake8-bandit: 安全检查
    "B",     # flake8-bugbear: 常见 bug 模式
    "A",     # flake8-builtins: 禁止覆盖内置函数名
    "C4",    # flake8-comprehensions: 推导式优化
    "T10",   # flake8-debugger: 禁止提交 breakpoint()
    "SIM",   # flake8-simplify: 简化代码建议
    "I",     # isort: import 排序
    "C90",   # mccabe: 圈复杂度检查
    "E",     # pycodestyle: 风格错误
    "W",     # pycodestyle: 风格警告
    "F",     # pyflakes: 逻辑错误
    "PGH",   # pygrep-hooks: 框架约定检查
    "UP",    # pyupgrade: 语法升级建议
    "RUF",   # Ruff 专属规则
    "TRY",   # tryceratops: try/except 规范
]
ignore = [
    "E501",  # LineTooLong（被 black 格式化的行可能超过 120）
    "E731",  # DoNotAssignLambda（有时有合理用途）
]
```

**关键概念：**
- `select` = **显式开启**的规则集。相当于 ESLint 的 `rules` 里设成 `"error"`。
- `ignore` = **显式关闭**的某些规则。相当于 ESLint 的 `rules` 里设成 `"off"`。

**类比：** 想象你在学校，`select` 是你选修的课程，`ignore` 是你申请免修的那些课。Ruff 有上百条规则，你不可能全选——选择跟项目风格匹配的就好。

#### 第三层：`[tool.ruff.lint.per-file-ignores]` — 文件级豁免

```toml
[tool.ruff.lint.per-file-ignores]
"tests/*" = ["S101"]
```

`S101` 规则禁止使用 `assert`（因为生产代码中 assert 可能被 Python 优化掉）。但**测试代码恰恰需要 assert**——这条配置就是告诉 Ruff："测试目录下的文件，别管 assert 的事"。

**对比：** ESLint 的 `overrides` 数组：

```json
{
  "overrides": [
    {
      "files": ["tests/**"],
      "rules": { "no-assert": "off" }
    }
  ]
}
```

### 3.3 格式配置：`[tool.ruff.format]`

Ruff 0.2+ 内置了 formatter（对标 Black/Prettier），配置很简单：

```toml
[tool.ruff.format]
quote-style = "double"     # 用双引号还是单引号
indent-style = "space"      # 空格还是 Tab
line-ending = "auto"        # 换行符：auto / lf / crlf
```

agentUniverse 项目没有显式写 `[tool.ruff.format]` 段，因为 Ruff formatter 默认就是 Black 风格——和 Black 兼容，不需要额外配置。

**如果你已经用了 Black：** Ruff formatter 的设计目标之一是 **100% 兼容 Black 的输出**。迁移时把 `black` 命令换成 `ruff format` 就行，产出的 diff 应该是空的。

---

## 四、第三步：在命令行中使用

### 4.1 日常命令速查

```bash
# Lint 检查（类似 eslint）
ruff check .

# Lint + 自动修复（类似 eslint --fix）
ruff check --fix .

# 格式化（类似 prettier --write）
ruff format .

# 格式化 + 检查（只检查不修改，CI 用）
ruff format --check .

# 只检查特定文件
ruff check src/app.py

# 只检查特定目录
ruff check agentuniverse/
```

**对照 Node.js：**

| 操作 | npm 命令 | Ruff 命令 |
|------|---------|----------|
| Lint 检查 | `npx eslint .` | `ruff check .` |
| Lint + 自动修复 | `npx eslint --fix .` | `ruff check --fix .` |
| 格式化 | `npx prettier --write .` | `ruff format .` |
| 格式化检查 | `npx prettier --check .` | `ruff format --check .` |
| 查看规则说明 | 查 ESLint 文档 | `ruff rule E501` |
| 列出所有规则 | — | `ruff linter` |

### 4.2 查看规则详情

当你看到一条报错但不知道什么意思：

```bash
ruff rule E501
# 输出：
# line-too-long (E501)
# Derived from the pycodestyle linter.
#
# What it does: Checks for lines that exceed the specified maximum length.
# ...

ruff rule S101
# assert (S101)
# Derived from the flake8-bandit linter.
#
# What it does: Checks for the use of assert statements.
# ...
```

**类比：** 这就像 eslint.org 上搜索规则名但不需要打开浏览器。

---

## 五、第四步：集成到 pre-commit（提交门禁）

lint 最大的悲剧是：没人手动跑，直到 CI 报错才知道。最好的解决方案是 **pre-commit hook**——提交前自动检查。

### 5.1 安装 pre-commit

```bash
pip install pre-commit        # 或 poetry add -G dev pre-commit
```

`pre-commit` 是 Python 世界管理 Git hooks 的标准工具（≈ `husky` + `lint-staged`）。

### 5.2 配置 `.pre-commit-config.yaml`

在项目根目录创建 `.pre-commit-config.yaml`：

```yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.8.4                            # Ruff 版本，跟你安装的一致
    hooks:
      - id: ruff                           # lint 检查
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format                    # 格式化
```

**两个 hook 解释：**

| Hook ID | 做什么 | 如果 fail 会怎样 |
|---------|--------|-----------------|
| `ruff` | 跑 `ruff check --fix` | `--exit-non-zero-on-fix` 表示如果修了东西就阻止提交，让你重新 `git add` |
| `ruff-format` | 跑 `ruff format --check` | 格式不对就阻止提交，需要手动 `ruff format .` |

**比喻：** 这就像在 Node.js 项目中配置 `husky` + `lint-staged`：

```json
// package.json（对比）
{
  "lint-staged": {
    "*.py": ["ruff check --fix", "ruff format"]
  }
}
```

### 5.3 安装 hooks

```bash
pre-commit install
```

这条命令读取 `.pre-commit-config.yaml`，在 `.git/hooks/` 下生成 pre-commit 脚本。之后每次 `git commit`，hooks 自动执行。

**对比：** 相当于 `npx husky install` 或 `npm run prepare`（husky v4 方式）。

### 5.4 手动触发（不提交也跑）

```bash
pre-commit run --all-files    # 对所有文件跑一遍
pre-commit run ruff           # 只跑 ruff hook
```

---

## 六、第五步：集成到 CI（持续集成门禁）

pre-commit 只是"本地保险"，CI 是"远程保险"——有人跳过 hook（`git commit --no-verify`）时，CI 是最后一道防线。

### 6.1 GitHub Actions 示例

```yaml
name: Lint

on: [push, pull_request]

jobs:
  ruff:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/ruff-action@v1
        with:
          args: "check ."
      - uses: astral-sh/ruff-action@v1
        with:
          args: "format --check ."
```

### 6.2 通用 CI 脚本

如果你用的是其他 CI 系统（Jenkins、GitLab CI 等），直接调命令行：

```bash
# lint 检查（有错误返回非 0）
ruff check .

# 格式检查（格式不对返回非 0）
ruff format --check .
```

只要这两个命令都返回 0，就说明代码质量过关。

---

## 七、进阶技巧

### 7.1 只看被某规则命中的代码

```bash
ruff check . --select S       # 只看安全检查相关的问题
ruff check . --select E501    # 只看行太长的问题
```

### 7.2 临时忽略某一行

跟 ESLint 的 `// eslint-disable-next-line` 一样，Ruff 支持行级忽略：

```python
# noqa: E501
very_long_variable_name = "this line is intentionally long because the string is a URL template https://example.com/api/v2/..."

# noqa: E731
add = lambda x, y: x + y  # 虽然不推荐 lambda 赋值，但偶尔有合理场景
```

**语法：** `# noqa: RULE_CODE`。`noqa` 是 flake8 时代留下的约定（"no quality assurance"），Ruff 兼容这个格式。

### 7.3 自动添加 `# noqa` 注释

不确定该不该加 noqa？先让 Ruff 帮你加，你再逐个审核：

```bash
ruff check . --add-noqa
```

这会在所有报错的地方自动插入 `# noqa` 注释。之后你可以逐个审查，觉得合理的保留，不合理的删掉注释并修复代码。

### 7.4 VS Code 集成

安装 Ruff 官方扩展后，保存文件时自动格式化和 lint。`.vscode/settings.json`：

```json
{
  "[python]": {
    "editor.formatOnSave": true,
    "editor.defaultFormatter": "charliermarsh.ruff",
    "editor.codeActionsOnSave": {
      "source.fixAll.ruff": "explicit",
      "source.organizeImports.ruff": "explicit"
    }
  }
}
```

这就像 VS Code 的 ESLint + Prettier 扩展，但配置更少。

---

## 八、为什么 Ruff 这么快？一个通俗版解释

传统 Python lint 工具的架构：

```
源码(.py) → Python 解析器(AST) → 检查插件1 → 检查插件2 → ... → 输出结果
              ↑ 这一步是性能瓶颈，每个插件都要重新解析 AST
```

Ruff 的架构：

```
源码(.py) → Rust 解析器(AST) → 所有规则遍历一次 AST → 输出结果
              ↑ Rust 写的，快       ↑ 一次遍历，全部完成
```

**类比：** 传统方案像你去超市买菜，买了鸡蛋回家，发现忘了酱油又跑一趟，发现忘了葱又跑一趟。Ruff 是一次性把所有东西买齐——进超市一次，全搞完。

具体到数字：flake8 检查 agentUniverse 可能需要 30 秒，Ruff 只需要 0.3 秒。对于大型项目，这种差距会从 "快一点" 变成 "能用 vs 不能用" 的区别。

---

## 九、迁移指南：从 flake8/Black/isort 到 Ruff

如果你接手一个还在用 flake8 + Black + isort 的老项目，迁移只需要两步：

### Step 1: 替换配置

把 `setup.cfg` 或 `tox.ini` 里的 flake8 配置翻译到 `pyproject.toml`：

```bash
# Ruff 自带迁移脚本（实验性）
ruff config migrate
```

### Step 2: 替换命令

```bash
# 之前
flake8 .
black .
isort .

# 之后
ruff check .
ruff format .   # 包含 import 排序，不需要 isort 了
```

### Step 3: 卸载旧依赖

```bash
poetry remove flake8 black isort flake8-bugbear flake8-bandit
poetry add -G dev ruff
```

**验证迁移成功：** 运行 `ruff check .` 和旧 flake8 的结果应该基本一致——Ruff 的规则设计就是兼容的。

---

## 十、常见踩坑

### 坑 1: `E501` 和 Black/Ruff formatter 冲突

**现象：** Ruff formatter 把一行代码格式化成 125 字符（因为那个写法确实折不断），但 `E501` 规则说"最多 120"。

**解决：** agentUniverse 的做法——直接 ignore `E501`，信任 formatter 的判断。formatter 比你更懂怎么断行。

### 坑 2: `select` 和 `ignore` 的优先级

**现象：** 你在 `select` 里选了 `E`（包含 `E501`），又在 `ignore` 里写了 `E501`——哪个生效？

**答案：** `ignore` 永远优先生效。Ruff 的处理顺序：`select` 确定规则池 → `ignore` 从中剔除 → 执行检查。

### 坑 3: `ruff format` 和 `ruff check --fix` 各管各的

**现象：** 跑完 `ruff format .`，`ruff check .` 仍然报错。

**原因：** 这两个命令**职责不同**：
- `ruff format` = 纯格式（空格、换行、引号风格）→ 对标 Prettier
- `ruff check --fix` = 逻辑修复（删除未用 import、简化表达式）→ 对标 ESLint --fix

**两者都需要跑，不存在一个替代另一个。**

### 坑 4: pre-commit 版本和项目版本不一致

**现象：** pre-commit 配置写 `rev: v0.5.0`，但 pyproject.toml 里装的是 `ruff ^0.8.4`。pre-commit 会用 v0.5.0 的 Ruff 去检查。

**解决：** `.pre-commit-config.yaml` 里的 `rev` 和 `pyproject.toml` 里的 ruff 版本保持同步。它们实际上是**两个独立的 Ruff 安装**——pre-commit 装在自己的隔离环境里。

---

## 十一、总结：给你的 Checklist

从零集成 Ruff 到项目的完整步骤：

```
□ 1. 安装依赖：
     poetry add -G dev ruff pre-commit

□ 2. 配置 pyproject.toml：
     [tool.ruff] + [tool.ruff.lint] + [tool.ruff.lint.per-file-ignores]

□ 3. 配置 .pre-commit-config.yaml：
     两个 hook：ruff + ruff-format

□ 4. 安装 pre-commit hooks：
     pre-commit install

□ 5. 跑一遍全量检查（看现有代码有多少问题）：
     ruff check .
     ruff format --check .

□ 6. 修复（或不理）现有问题：
     ruff check --fix .
     ruff format .

□ 7. 配 CI（可选）：
     加一步 ruff check + ruff format --check

□ 8. 配 VS Code（可选）：
     安装 Ruff 扩展 + 保存时自动修复
```

完成这些后，你的 Python 项目就有了和 Node.js 项目同等级别的代码质量保障。

---

> **相关文档：**
> - [pyproject.toml 完全指南](./pyproject-toml-guide.md) — Ruff 配置就写在 pyproject.toml 里
> - 回到学习路径：[../README.md](../README.md)
