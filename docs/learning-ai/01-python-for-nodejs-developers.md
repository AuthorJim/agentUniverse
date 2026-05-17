# 01 — Python 入门：写给 Node.js 全栈工程师

> **目标读者：** 有 Node.js/JavaScript 经验，但 Python 不熟悉的工程师。
> **学习目标：** 能顺利阅读、修改 agentUniverse 源码和示例代码。
> **预计时间：** 2-3 天（动手练习为主）

本教程与 agentUniverse 项目代码强绑定——每个知识点都从项目真实源码中抽出例子。请确保已经 clone 项目：

```bash
git clone https://github.com/agentuniverse-ai/agentUniverse.git
cd agentUniverse
```

---

## 1. 心理模型转换：Node.js → Python

离开 JavaScript 世界前，先把几件最大的差异摆在明处。后续各章节会逐个展开。

| 维度 | JavaScript / Node.js | Python |
|------|---------------------|--------|
| **运行环境** | V8 引擎，事件驱动 | CPython 解释器，同步优先 |
| **异步模型** | 天生异步，Promise/async-await | 默认同步，`async/await` 是后来加的 |
| **包管理** | npm / yarn / pnpm | pip + Poetry（本项目用） |
| **入口文件** | `package.json` | `pyproject.toml` |
| **代码块** | `{}` 大括号 | **缩进**（Indentation） |
| **面向对象** | 基于原型链，ES6 `class` 是语法糖 | 纯类继承体系 |
| **类型系统** | 动态 + TypeScript 可附加 | 动态 + 可选的 Type Hints |
| **this/self** | `this` 隐式绑定 | `self` 必须显式写在第一个参数 |
| **模块导入** | `require()` / `import` | `import` / `from ... import` |
| **包结构** | `node_modules/` | `site-packages/`（Poetry 虚拟环境） |

### 1.1 缩进不是风格问题，是语法

这是 JavaScript 转 Python **最容易踩的坑**。Python 用缩进表示代码块，而不是 `{}`。

```python
# ✅ 正确：一致的 4 空格缩进
if name == 'agent':
    print('这是一个 agent')
    if name.startswith('peer'):
        print('这是 PEER 模式的 agent')

# ❌ 错误：混用 tab 和空格 → IndentationError
if name == 'agent':
    print('hello')
\tprint('world')  # 用了 tab，前面一行是空格
```

agentUniverse 项目使用 **4 空格** 缩进（Python 社区标准）。项目的 ruff/black 配置会自动检查并修复。

### 1.2 没有 `;`，没有 `const/let/var`

```python
# Python：直接赋值，无需声明关键字
name = 'qwen_llm'           # 字符串
temperature = 0.7           # 浮点数
max_tokens = 1000           # 整数（没有 int/float 之分，都是对象）
tools = ['tool_a', 'tool_b']  # 列表（类似 JS Array）
config = {'key': 'value'}   # 字典（类似 JS Object）
```

对比 JavaScript：
```javascript
// JavaScript 需要声明关键字
let name = 'qwen_llm';
const temperature = 0.7;
var maxTokens = 1000;
```

### 1.3 注释用 `#` 不是 `//`

```python
# 这是 Python 单行注释
# Python 没有 /* */ 多行注释语法，多行注释一般用连续 # 或三引号字符串
```

---

## 2. 基础语法速览

### 2.1 变量与基本类型

```python
# 字符串
name = 'qwen_llm'
description = "一个 Qwen 大模型实例"
multiline = '''
这是多行字符串，
类似 JavaScript 的模板字符串。
'''

# 数字
count = 42          # 整数
price = 99.9        # 浮点数

# 布尔值（注意大写）
is_active = True    # 不是 true
is_empty = False    # 不是 false
is_none = None      # 不是 null / undefined

# 列表（类似 JS Array）
tools = ['google_search', 'python_runner', 'param_converter']
tools.append('new_tool')        # push
tools[0]                         # 索引访问

# 字典（类似 JS Object，但 key 可以是任何不可变类型）
config = {
    'name': 'qwen_llm',
    'temperature': 0.7,
    'max_tokens': 1000
}
config['model_name'] = 'qwen2.5-72b-instruct'   # 添加/修改
config.get('missing_key', 'default_value')       # 安全访问（有默认值）

# 元组（不可变列表）
coordinates = (10, 20)    # 创建后不能修改

# 集合（去重）
unique_names = {'agent_a', 'agent_b', 'agent_a'}  # 结果是 {'agent_a', 'agent_b'}
```

### 2.2 字符串格式化：f-string

Python 最常用的字符串拼接方式是 f-string（类似 JS 模板字符串）。

```python
model_name = 'qwen2.5-72b-instruct'
temperature = 0.7

# f-string：在字符串前加 f，变量写在大括号里
config_line = f'model: {model_name}, temperature: {temperature}'

# 也支持表达式
summary = f'参数数量: {1000 + 500}'

# 对比 JavaScript：
# const configLine = `model: ${modelName}, temperature: ${temperature}`;
```

### 2.3 条件判断

```python
# Python 用 if / elif / else，没有 switch-case（Python 3.10+ 有 match-case）
if temperature > 0.8:
    print('温度较高，输出更随机')
elif temperature > 0.3:
    print('温度适中')
else:
    print('温度较低，输出更确定')

# 支持链式比较（JS 不支持）
if 0 < temperature <= 1.0:   # 等价于 0 < temperature and temperature <= 1.0
    print('温度在有效范围内')
```

### 2.4 循环

```python
# for...in 遍历列表（最常用）
for tool in tools:
    print(tool)

# 带索引的遍历
for i, tool in enumerate(tools):
    print(f'{i}: {tool}')

# 遍历字典
for key, value in config.items():
    print(f'{key} = {value}')

# while 循环
count = 0
while count < 3:
    print(count)
    count += 1   # 没有 count++ 语法

# 列表推导式（List Comprehension）—— Python 特色，非常常用
names = [tool.upper() for tool in tools]  # 转大写
active_tools = [t for t in tools if t.startswith('google')]  # 带过滤
```

### 2.5 函数

```python
# 基本函数（没有 function 关键字）
def greet(name):
    return f'Hello, {name}!'

# 带类型提示和默认参数
def create_config(model_name: str, temperature: float = 0.7) -> dict:
    return {
        'model_name': model_name,
        'temperature': temperature
    }

# 对比 JavaScript：
# function createConfig(modelName, temperature = 0.7) {
#     return { modelName, temperature };
# }
```

---

## 3. 面向对象：class、继承、self

Python 是纯基于类的面向对象语言，这一点和 JavaScript 的原型链很不同。agentUniverse 中几乎每个组件都是类。

### 3.1 基本类定义

> 📌 **关键规则：** 实例方法的第一个参数**必须**是 `self`，相当于 JS 的 `this`，但 Python 要求**显式声明**。

```python
class MyAgent:
    """类的文档字符串（类似 JSDoc）"""

    # 类属性（所有实例共享，类似 JS static 属性）
    component_type = 'AGENT'

    # 构造函数（类似 JS constructor）
    # __init__ 不是 __init ！（注意前后都是双下划线）
    def __init__(self, name: str, temperature: float = 0.7):
        # self.xxx 是实例属性
        self.name = name
        self.temperature = temperature

    # 实例方法
    def run(self, input_text: str) -> str:
        return f'{self.name} says: {input_text}'

# 使用
agent = MyAgent(name='qwen_agent', temperature=0.5)
result = agent.run('你好')
```

对比 JavaScript：
```javascript
// JavaScript 等价写法
class MyAgent {
    static componentType = 'AGENT';

    constructor(name, temperature = 0.7) {
        this.name = name;
        this.temperature = temperature;
    }

    run(inputText) {
        return `${this.name} says: ${inputText}`;
    }
}
```

### 3.2 继承

这是 agentUniverse 中最核心的面向对象模式。框架中所有组件都继承自 `ComponentBase`。

```python
# 来自 agentuniverse/base/component/component_base.py
from pydantic import BaseModel

class ComponentBase(BaseModel):
    """所有组件的基类"""
    component_type: ComponentEnum

    def get_instance_code(self) -> str:
        """返回组件的完整名称"""
        appname = ApplicationConfigManager().app_configer.base_info_appname
        return f'{appname}.{self.component_type.value.lower()}.{self.name}'


# agentUniverse 源码中实际的继承链：
# agentuniverse/agent/agent.py:58
from abc import ABC
class Agent(ComponentBase, ABC):      # 多继承：ComponentBase + ABC
    """所有 Agent 的父类"""
    agent_model: Optional[AgentModel] = None

    @abstractmethod
    def input_keys(self) -> list[str]:
        pass    # 抽象方法，子类必须实现

# agentuniverse/agent/template/react_agent_template.py:37
class ReActAgentTemplate(AgentTemplate):   # AgentTemplate 又继承自 Agent
    """ReAct 模式的 Agent 模板"""
    max_iterations: Optional[int] = None
```

继承链：`ComponentBase → Agent → AgentTemplate → ReActAgentTemplate`

```python
# 关键点：
# 1. Python 支持多继承：class Child(Parent1, Parent2)
# 2. super().__init__() 调用父类构造函数
# 3. @abstractmethod 装饰器标记抽象方法（子类必须实现，否则报错）
# 4. ABC（Abstract Base Class）是 Python 的抽象基类
```

### 3.3 特殊方法（Magic Methods / Dunder Methods）

以双下划线开头和结尾的方法是 Python 的特殊方法，类似 JavaScript 的 `Symbol.*`。

```python
class Tool:
    def __init__(self, name: str):
        """构造函数"""
        self.name = name

    def __str__(self) -> str:
        """类似 JS 的 toString()"""
        return f'Tool({self.name})'

    def __repr__(self) -> str:
        """开发者友好的字符串表示，用于调试"""
        return f'Tool(name={self.name!r})'

    def __len__(self) -> int:
        """类似 JS 的 length 属性，但 Python 是方法"""
        return len(self.name)
```

---

## 4. 类型提示（Type Hints）

agentUniverse 项目**大量使用类型提示**。这是 Python 3.5+ 引入的可选语法，类似 TypeScript 但**不做运行时校验**。

### 4.1 基本类型注解

```python
from typing import Optional, List, Dict, Any, Union

# 来自 agentuniverse/agent/agent_model.py:12-20
class AgentModel(BaseModel):
    info: Optional[dict] = dict()          # Optional[X] = X | None
    profile: Optional[dict] = dict()
    memory: Optional[dict] = dict()
    action: Optional[dict] = dict()

# 来自 agentuniverse/agent/agent.py:61
agent_model: Optional[AgentModel] = None

# 来自 agentuniverse/agent/agent.py:68
def input_keys(self) -> list[str]:        # 返回类型是字符串列表
    return ['input']

# 来自 agentuniverse/agent/agent.py:110
def run(self, **kwargs) -> OutputObject:  # **kwargs 是可变关键字参数
    ...
```

### 4.2 常用类型对照表

| TypeScript | Python | 说明 |
|-----------|--------|------|
| `string` | `str` | |
| `number` | `int`, `float` | Python 区分整数和浮点 |
| `boolean` | `bool` | |
| `null \| undefined` | `None` | Python 只有 `None` |
| `Array<T>` | `list[T]` | Python 3.9+ |
| `Record<K,V>` | `dict[K, V]` | |
| `T \| undefined` | `Optional[T]` | 等价于 `T \| None` |
| `void` | `None`（作为返回值） | |
| `any` | `Any` | 从 `typing` 导入 |
| `never` | `Never` | Python 3.11+ |
| `interface` | `class(BaseModel)` | 用 Pydantic 代替 |
| `enum` | `Enum` | |

### 4.3 你会在项目中看到这些写法

```python
# 多个类型用 Union（Python 3.10+ 可以用 |）
from typing import Union, Sequence, Optional, List
from langchain_core.runnables import Runnable

# 来自 react_agent_template.py
def build_agent_executor(self) -> Runnable:
    ...

# Optional 其实是 Union[X, None] 的简写
name: Optional[str] = None    # 等价于 name: str | None = None
```

---

## 5. Pydantic BaseModel：Python 的"数据类 Schema"

agentUniverse 中**几乎所有核心类**都继承自 `pydantic.BaseModel`，包括 `ComponentBase`、`AgentModel`、`ToolInput` 等。

**Pydantic 是什么？** 它是一个数据校验和序列化库。和 TypeScript 的 `interface` + `zod` 很像：定义结构 → 自动校验 → 自动序列化。

### 5.1 基本用法

```python
from pydantic import BaseModel
from typing import Optional

# 来自 agentuniverse/agent/agent_model.py:12
class AgentModel(BaseModel):
    info: Optional[dict] = dict()
    profile: Optional[dict] = dict()
    plan: Optional[dict] = dict()
    memory: Optional[dict] = dict()
    action: Optional[dict] = dict()

# 使用
model = AgentModel(
    info={'name': 'demo_agent'},
    profile={'llm_model': {'name': 'qwen_llm', 'temperature': 0.1}}
)
print(model.info['name'])       # 正常访问
print(model.model_dump())       # 类似 JSON.stringify()
```

对比 TypeScript + zod：
```typescript
import { z } from 'zod';

const AgentModelSchema = z.object({
    info: z.record(z.unknown()).optional().default({}),
    profile: z.record(z.unknown()).optional().default({}),
});
```

### 5.2 Pydantic 的核心能力

```python
# 1. 属性访问
agent.profile          # dict 访问

# 2. 序列化
agent.model_dump()     # 转 dict（Python 3.12+ 不用 .dict()）
agent.model_dump_json()# 转 JSON 字符串

# 3. 自动校验（类型不匹配会报 ValidationError）
agent = AgentModel(info="wrong_type")   # 期望 dict，传了 str → 报错

# 4. 默认值
agent = AgentModel()   # 所有字段都有默认值，可以不传
```

**关键：** 本项目中 `ComponentBase` 继承 `BaseModel`，所以所有组件（Agent、LLM、Tool、Knowledge...）**自动获得 Pydantic 的所有能力**。

---

## 6. 装饰器（Decorator）：Python 的"注解"

装饰器是 agentUniverse 项目中最常用的 Python 特性之一。

**装饰器是什么？** 一个函数，接受另一个函数（或类）作为参数，在不修改原代码的前提下增加额外逻辑。类似 JavaScript 的 ES7 decorator（但更简单直接）。

### 6.1 `@singleton`：保证全局只有一个实例

本项目几乎所有 Manager 类都用了 `@singleton`。

```python
# 来自 agentuniverse/base/annotation/singleton.py
def singleton(cls):
    """装饰器：让一个类变成单例模式"""
    instances = {}

    @wraps(cls)
    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

# 使用实例（来自 agentuniverse/agent/agent_manager.py）
@singleton
class AgentManager(BaseManager):
    def get_instance_obj(self, component_instance_name: str):
        # 获取注册的 agent 实例
        ...
```

**对 Node.js 工程师的理解：**

```javascript
// 这类似于 JavaScript 的模块级单例模式：
// 但不完全一样—— @singleton 改变了类的行为
class AgentManager {
    // ...
}
// 手动实现单例：
const agentManagerInstance = new AgentManager();
export default agentManagerInstance;
```

### 6.2 `@trace_agent` 和 `@trace_tool`：自动追踪

```python
# 来自 agentuniverse/agent/agent.py:109-110
@trace_agent
def run(self, **kwargs) -> OutputObject:
    """Agent 实例运行入口"""
    ...

@trace_agent
async def async_run(self, **kwargs) -> OutputObject:
    """Agent 异步运行入口"""
    ...

# 当调用 agent.run() 时，装饰器会自动记录：
# - 开始时间
# - 输入参数
# - 输出结果
# - 耗时
# - 异常信息
# 所有这些会通过 OpenTelemetry 上报
```

### 6.3 `@abstractmethod`：标记抽象方法

```python
# 来自 agentuniverse/agent/agent.py:67-69
from abc import abstractmethod

class Agent(ComponentBase, ABC):
    @abstractmethod
    def input_keys(self) -> list[str]:
        """子类必须实现此方法"""
        pass

# 如果子类不实现 input_keys，实例化时会报 TypeError
```

---

## 7. 异步编程：Python 的 async/await

### 7.1 核心差异

JavaScript 的异步是"原生实现"，Python 的异步是"后来追加的"。

| 概念 | JavaScript | Python |
|------|-----------|--------|
| 基本单位 | Promise | Coroutine（协程） |
| 声明 | `async function` | `async def` |
| 等待 | `await` | `await` |
| 运行入口 | 自动（事件循环） | 需要 `asyncio.run()` |
| 并发 | `Promise.all()` | `asyncio.gather()` |
| 请求库差异 | `fetch` 原生支持 | 需要 `aiohttp` 等（`requests` 不支持） |

### 7.2 在 agentUniverse 中的实际例子

```python
import asyncio

# 来自 agentuniverse/agent/agent.py:131-138
class Agent(ComponentBase, ABC):

    # 同步版本
    def run(self, **kwargs) -> OutputObject:
        """同步运行 Agent"""
        agent_result = self.execute(input_object, agent_input)
        return OutputObject(agent_result)

    # 异步版本
    async def async_run(self, **kwargs) -> OutputObject:
        """异步运行 Agent"""
        agent_result = await self.async_execute(input_object, agent_input)
        return OutputObject(agent_result)

    async def async_execute(self, input_object: InputObject, agent_input: dict) -> dict:
        """异步执行"""
        ...
```

### 7.3 关键规则

```python
# 规则 1：await 只能在 async def 函数里使用
async def fetch_data():
    result = await some_async_call()    # ✅ 正确
    return result

def normal_function():
    result = await some_async_call()    # ❌ SyntaxError

# 规则 2：调用 async 函数不执行，返回的是一个 coroutine 对象
coro = fetch_data()         # 返回 coroutine，没开始执行！
result = await coro          # 这时才真正执行
result = asyncio.run(coro)  # 在同步代码中启动异步函数

# 规则 3：Python 的 requests 库是同步的，不能 await
import requests
response = requests.get('https://api.example.com')  # ✅ 同步调用

# 如果在 async 函数里需要 HTTP 请求，用 aiohttp
import aiohttp
async def fetch_url(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as response:
            return await response.json()
```

---

## 8. 包管理：Poetry（类似 npm）

agentUniverse 使用 **Poetry** 管理依赖，而非 pip + requirements.txt。Poetry 之于 Python 就像 npm 之于 Node.js。

### 8.1 对比表

| 功能 | Node.js (npm) | Python (Poetry) |
|------|--------------|-----------------|
| 包描述文件 | `package.json` | `pyproject.toml` |
| 锁文件 | `package-lock.json` | `poetry.lock` |
| 安装依赖 | `npm install` | `poetry install` |
| 添加依赖 | `npm install <pkg>` | `poetry add <pkg>` |
| 添加开发依赖 | `npm i -D <pkg>` | `poetry add -G dev <pkg>` |
| 运行脚本 | `npm run <script>` | `poetry run <cmd>` |
| 全局安装 | `npm install -g` | `pip install`（不归 Poetry 管） |
| 发布 | `npm publish` | `poetry publish` |

### 8.2 实际工作流

```bash
# 安装项目（类似 npm install）
poetry install

# 带可选依赖安装
poetry install --extras store_ext

# 开发依赖已经包含 pytest, pre-commit, deptry
poetry install --with dev

# 在虚拟环境中运行命令
poetry run pytest                    # 运行测试
poetry run python some_script.py     # 执行 Python 脚本

# 进入虚拟环境的 shell
poetry shell

# 添加新依赖
poetry add anthropic          # 生产依赖
poetry add -G dev pytest-cov  # 开发依赖
```

### 8.3 pyproject.toml 结构

```toml
# 来自 agentUniverse/pyproject.toml（简化）

[tool.poetry]
name = "agentUniverse"
version = "0.0.19"
description = "..."
authors = ["AntGroup <jerry.zzw@antgroup.com>"]

[tool.poetry.dependencies]             # 类似 dependencies
python = "^3.10"                       # 类似 "node": ">=18"
requests = "^2.32.0"                   # 类似 "requests": "^2.32.0"
pydantic = "^2.6.4"

[tool.poetry.group.dev.dependencies]   # 类似 devDependencies
pytest = "^7.2.0"
pre-commit = "^2.20.0"

[tool.ruff]                            # 类似 .eslintrc
line-length = 120
```

---

## 9. 模块导入系统

### 9.1 对比表

| 概念 | JavaScript | Python |
|------|-----------|--------|
| 导入模块 | `import x from 'y'` | `import x` 或 `from y import x` |
| 导出 | `export default` / `export` | 所有顶层符号自动可被导入 |
| 命名导出 | `export const foo` | 没有这个概念；只有 `__all__` 可选控制 |
| 相对导入 | `import './utils'` | `from . import utils` |
| 目录即模块 | `index.js` 自动解析 | 需要 `__init__.py` 文件 |

### 9.2 agentUniverse 中的实际导入示例

```python
# 来自 agentuniverse/agent/agent.py:8-55（简化版）

# --- 导入标准库 ---
import json
import uuid
from abc import abstractmethod, ABC
from typing import Optional, Any, List

# --- 导入第三方库 ---
from langchain_core.runnables import RunnableSerializable
from pydantic import BaseModel

# --- 导入本项目模块 ---
# 绝对导入（从项目根开始）
from agentuniverse.agent.agent_model import AgentModel
from agentuniverse.agent.action.tool.tool import Tool
from agentuniverse.base.component.component_base import ComponentBase
from agentuniverse.base.annotation.trace import trace_agent
from agentuniverse.prompt.prompt import Prompt
```

### 9.3 关键规则

```python
# 1. import X：导入整个模块
import json
json.loads('{"key": "value"}')

# 2. from X import Y：从模块中导入特定名称
from typing import Optional, List
name: Optional[str] = None

# 3. from X import Y as Z：导入并重命名
import numpy as np   # 社区惯例
arr = np.array([1, 2, 3])

# 4. 相对导入（模块内部使用）
from . import utils               # 同目录下的 utils 模块
from ..base import ComponentBase  # 上一级目录的 base 模块

# 5. __init__.py：把一个目录变成 Python 包
# 每个 agentuniverse/ 的子目录都有一个 __init__.py
# 这就是为什么可以 from agentuniverse.agent import agent
```

---

## 10. agentUniverse 项目中你会频繁见到的 Python 模式

### 10.1 `**kwargs`：可变关键字参数

类似 JavaScript 的 `...rest` 参数，但 Python 更常用 `**kwargs`。

```python
# 来自 agentuniverse/agent/agent.py:110
def run(self, **kwargs) -> OutputObject:
    """kwargs 是一个 dict，收集所有关键字参数"""
    # 调用者可以传任意关键字参数
    input_object = InputObject(kwargs)
    ...

# 调用示例
agent.run(input='你的问题', session_id='123', temperature=0.5)
# kwargs = {'input': '你的问题', 'session_id': '123', 'temperature': 0.5}

# 对应的 *args：可变位置参数（收集为 tuple）
def log(*args):
    for arg in args:
        print(arg)
```

### 10.2 `Optional[X]` = `X | None`

```python
# 以下两种写法等价（Python 3.10+ 推荐后者）
from typing import Optional
name: Optional[str] = None     # 旧写法
name: str | None = None        # 新写法
```

### 10.3 `pass`：占位符

当语法要求有内容但你不想写实现时：

```python
@abstractmethod
def input_keys(self) -> list[str]:
    pass    # 抽象方法，仅声明，不实现

# 对比 JavaScript：
# inputKeys() {}  // 空函数体
```

### 10.4 `with` 语句：上下文管理器

类似 JavaScript 的 `using`（但更常用）。

```python
# 文件操作：自动关闭
with open('config.yaml', 'r') as f:
    content = f.read()
# 出了 with 块，文件自动关闭（不需要 f.close()）

# 对比 JavaScript：
# const content = fs.readFileSync('config.yaml', 'utf8');  // 同步

# Python 也有异步版
async with aiohttp.ClientSession() as session:
    async with session.get(url) as response:
        data = await response.json()
```

### 10.5 异常处理

```python
try:
    result = agent.run(input='test')
except ValueError as e:
    print(f'参数错误: {e}')
except Exception as e:
    print(f'未知错误: {e}')
finally:
    print('清理资源')

# 对比 JavaScript：
# try { ... } catch (e) { ... } finally { ... }
```

### 10.6 生成器（Generator）和 `yield`

```python
# 生成器：产生一系列值，用 yield 暂停
def generate_messages():
    yield '你好'
    yield '这是第二条消息'
    yield '这是第三条消息'

for msg in generate_messages():
    print(msg)

# agentUniverse 中的实际用途：
# base/util/common_util.py - stream_output 使用 yield 流式输出
def stream_output(content: str):
    for char in content:
        yield char
```

---

## 11. 动手练习：阅读并运行 agentUniverse 代码

现在用你的 Node.js 经验来阅读这段 agentUniverse 源码。试着理解每一行的含义。

### 练习 1：阅读 Agent 基类

```python
# agentuniverse/agent/agent.py:58-130（简化）

class Agent(ComponentBase, ABC):                    # ① 继承 ComponentBase 和 ABC
    agent_model: Optional[AgentModel] = None        # ② 类型注解

    @abstractmethod                                 # ③ 装饰器：抽象方法
    def input_keys(self) -> list[str]:              # ④ 方法签名 + 返回类型
        pass

    @trace_agent                                    # ⑤ 装饰器：追踪
    def run(self, **kwargs) -> OutputObject:        # ⑥ **kwargs
        self.input_check(kwargs)                    # ⑦ self 调用自身方法
        input_object = InputObject(kwargs)
        agent_input = self.pre_parse_input(input_object)
        result = self.execute(input_object, agent_input)
        agent_result = self.parse_result(result)
        return OutputObject(agent_result)

    async def async_run(self, **kwargs) -> OutputObject:  # ⑧ 异步版本
        ...
        agent_result = await self.async_execute(...)       # ⑨ await
        ...
```

对照检查你能识别出哪些知识点：
- [ ] ① 多继承
- [ ] ② Type hints（`Optional[AgentModel]`）
- [ ] ③ `@abstractmethod` 装饰器
- [ ] ④ 返回类型注解 `-> list[str]`
- [ ] ⑤ `@trace_agent` 装饰器
- [ ] ⑥ `**kwargs` 可变关键字参数
- [ ] ⑦ `self.method()` 调用自身方法
- [ ] ⑧ `async def` 异步函数
- [ ] ⑨ `await` 等待异步结果

### 练习 2：跑通一个简单 Agent

```bash
# 进入 simple_qa_agent_app 例子
cd examples/sample_apps/simple_qa_agent_app

# 阅读以下文件：
# 1. intelligence/agentic/agent/agent_instance/simple_qa_agent.yaml  # Agent 配置
# 2. intelligence/agentic/prompt/simple_qa_prompt.yaml    # Prompt 模板
# 3. intelligence/agentic/llm/qwen_llm.yaml              # LLM 配置
# 4. bootstrap/intelligence/server_application.py        # 启动入口

# 配置 API Key（以 Qwen 为例）
# 编辑 config/custom_key.toml，设置 DASHSCOPE_API_KEY

# 启动服务
python bootstrap/intelligence/server_application.py
```

### 练习 3：修改代码验证理解

1. 修改 `simple_qa_prompt.yaml` 中的 `instruction`，添加一条新规则
2. 修改 `simple_qa_agent.yaml` 中的 `temperature`，观察回复变化
3. 尝试给 agent 添加内存（uncomment `memory` 配置）

---

## 12. 常见错误排查

### 12.1 IndentationError: unexpected indent / unindent does not match...
这是最常见的错误。检查某处混用了 tab 和空格，或者缩进层级不对。agentUniverse 项目使用 **4 空格** 缩进。

### 12.2 NameError: name 'xxx' is not defined
变量未定义。检查是否有拼写错误，或是否忘记 import。

### 12.3 AttributeError: 'xxx' object has no attribute 'yyy'
对象没有这个属性。通常是因为忘记在 `__init__` 中赋值 `self.yyy = ...`。

### 12.4 ImportError / ModuleNotFoundError
- 检查是否 run 了 `poetry install`
- 检查 `PYTHONPATH` 是否包含项目根目录
- 检查对应的 `__init__.py` 是否存在

### 12.5 SyntaxError: invalid syntax
- 检查是否忘记函数定义后的 `:` 冒号
- 检查是否忘记 `if` / `for` / `while` / `def` / `class` 后的冒号
- 检查是否在非 async 函数里使用了 `await`

---

## 下一步

完成本文档后，你应该能够：
1. 阅读 agentUniverse 源码不出手汗
2. 理解项目中 `class`, `def`, `@decorator`, `-> ReturnType`, `await`, Pydantic 等关键语法
3. 修改 YAML 配置和示例代码中的 Python 部分

下一步进入 **02-Agent 基础概念**，学习 Agent 是什么、ReAct 模式的工作原理、以及 agentUniverse 的核心架构。
