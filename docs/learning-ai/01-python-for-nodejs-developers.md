# 01 — Python 速成：从 Node.js 到 Python 的思维跃迁

> **目标读者：** 有 Node.js/JavaScript 丰富经验，但 Python 还处于"看得懂、写不出"阶段的工程师。
> **学习目标：** 能流畅阅读、修改 agentUniverse 源码；能独立编写自定义 Tool、Memory、Prompt 的 Python 代码。
> **预计时间：** 2-3 天（不要跳，动手练）

---

## 0. 开始之前：换个大脑操作系统

作为一名资深 Node.js 工程师，你脑子里已经有一套完整的后端架构模型——事件循环、中间件模式、依赖注入、模块解析算法。好消息是：**Python 没有推翻这些概念，只是换了种表达方式。** 坏消息是：**有些你习以为常的东西在 Python 里根本不存在（比如 `{}` 代码块），有些 Python 习以为常的东西你在 JS 里从没见过（比如装饰器、列表推导式）。**

本章的目标不是教你"从零学 Python"，而是帮你建立 **Node.js → Python 的快速映射表**。每学一个新概念，问自己一句："这在我的 Node.js 世界里等价于什么？"

### 0.1 最核心的心理差异（3 分钟速览）

| 维度 | JavaScript / Node.js | Python |
|------|---------------------|--------|
| **代码块** | `{}` 大括号 | **缩进**（4 空格，写错了直接报错） |
| **语句结束** | `;` 分号（可选） | 换行（不需要分号） |
| **变量声明** | `let/const/var` | 无关键字，直接赋值 |
| **this/self** | `this` 隐式绑定 | `self` 显式传参 |
| **异步模型** | 天生异步，Promise/async-await | 默认同步，`async/await` 是后来加的 |
| **包管理** | npm/yarn/pnpm | pip + Poetry |
| **布尔值** | `true/false` | `True/False`（大写！） |
| **空值** | `null` / `undefined` | `None`（只有一个） |
| **类型系统** | 动态 + TypeScript 可选 | 动态 + Type Hints 可选 |
| **模块入口** | `index.js` 自动解析 | 需要 `__init__.py` 文件 |
| **运行环境** | V8 引擎，事件驱动 | CPython 解释器，同步优先 |

### 0.2 一个小实验：用两种语言写同一个函数

```python
# Python 版本
def create_agent_config(name: str, temperature: float = 0.7) -> dict:
    """创建一个 Agent 的配置字典"""
    if not name:
        raise ValueError("Agent 名字不能为空")  # 注意：raise 不是 throw
    config = {
        "name": name,
        "temperature": temperature,
        "created_at": "2026-05-17"
    }
    return config

# 使用
config = create_agent_config("my_agent", 0.5)
print(config["name"])  # 用方括号取字典值
```

```javascript
// JavaScript 版本（对照）
function createAgentConfig(name, temperature = 0.7) {
    if (!name) {
        throw new Error("Agent 名字不能为空");  // throw 不是 raise
    }
    const config = {
        name,
        temperature,
        createdAt: "2026-05-17"  // 驼峰命名
    };
    return config;
}

// 使用
const config = createAgentConfig("my_agent", 0.5);
console.log(config.name);  // 用点号取对象值
```

---

## 1. 语法第一课：缩进不是审美，是语法

这是 JavaScript 工程师转 Python **最容易踩的坑**，所以放在最前面讲。

### 1.1 缩进 = 代码块

在 JavaScript 中，`{}` 决定代码块边界，缩进只是装饰。在 Python 中，**缩进就是代码块**：

```python
# ✅ 正确：一致的 4 空格缩进
def handle_agent(agent_name: str):
    print(f'正在处理 Agent: {agent_name}')
    if agent_name.startswith('peer'):
        print('这是 PEER 模式的 Agent')
        if 'planning' in agent_name:
            print('具体角色: 规划者')
    else:
        print('这是其他模式的 Agent')

# ❌ 错误 1：混用 Tab 和空格 → IndentationError
def handle_agent(agent_name: str):
    print('hello')       # 4 个空格
\tprint('world')         # 1 个 Tab —— 和上面不一致，报错！

# ❌ 错误 2：缩进层级不对 → IndentationError
if True:
print('这行没缩进')     # 期望有缩进，但没写

# ❌ 错误 3：冒号忘记写 → SyntaxError
if True                 # 缺少冒号
    print('hello')
```

**记住一条铁律：** Python 用 `:` 冒号表示"下面是一个新代码块"，用缩进表示"哪些代码属于这个块"。agentUniverse 项目用 **4 空格** 缩进。

### 1.2 没有分号，没有 let/const/var

```python
# Python：直接赋值，清爽但需要适应
name = 'qwen_llm'                    # 字符串
temperature = 0.7                    # 浮点数
max_tokens = 1000                    # 整数
tools = ['tool_a', 'tool_b']        # 列表（JS Array）
config_map = {'key': 'value'}       # 字典（JS Object/Map）
is_active = True                     # 注意大写！
result = None                        # 注意不是 null
```

对应的 JavaScript：
```javascript
let name = 'qwen_llm';
const temperature = 0.7;
var maxTokens = 1000;            // 老派写法
const tools = ['tool_a', 'tool_b'];
const configMap = { key: 'value' };
const isActive = true;           // 小写
const result = null;
```

**注意：** Python 用 `snake_case`（下划线），JavaScript 用 `camelCase`（驼峰）。agentUniverse 源码中你会频繁看到 `agent_model`、`input_object`、`max_tokens` 这种命名。

### 1.3 注释用 `#`

```python
# 这是单行注释
# Python 没有 /* */ 多行注释语法
# 多行注释一般用连续的 # 或三引号字符串（但三引号本质是字符串不是注释）

"""
这是一个多行字符串（docstring），
通常放在函数/类定义的下一行作为文档。
它本质上不是注释——Python 运行时会保留它。
"""

def my_function():
    """这是 docstring，放在函数定义下一行。类似 JSDoc 的 /** ... */"""
    pass
```

---

## 2. 数据类型：从 JS 到 Python 的映射

### 2.1 基础类型速查

```python
# ── 数字 ──
count = 42              # int（整数）
price = 99.99           # float（浮点数）
complex_num = 3 + 4j    # complex（复数——JS 没有）

# ── 字符串 ──
name = 'qwen_llm'
description = "支持换行符 \n 和制表符 \t"
multiline = '''
这是多行字符串。
不需要 + 拼接。
缩进会被保留。
'''
# 常用方法
name.upper()            # 'QWEN_LLM'
name.startswith('qwen') # True —— 相当于 JS 的 startsWith
'agent' in name         # False —— 相当于 JS 的 includes
len(name)               # 8 —— 相当于 JS 的 .length

# ── 布尔 ──
is_active = True        # 大写 T！
is_empty = False        # 大写 F！
none_value = None       # 等价于 JS 的 null/undefined 合二为一

# ── 列表（JS Array） ──
tools = ['google_search', 'python_runner', 'calculator']
tools.append('new_tool')        # push
tools.pop()                     # pop（弹出末尾）
tools[0]                        # 按索引访问
tools[-1]                       # 最后一个元素（JS 没有负索引）
tools[1:3]                      # 切片：第 2 到第 3 个 → ['python_runner', 'calculator']
len(tools)                      # 长度
'google_search' in tools        # True —— 检查是否存在

# ── 字典（JS Object / Map） ──
config = {
    'name': 'qwen_llm',
    'temperature': 0.7,
    'max_tokens': 1000
}
config['model_name'] = 'qwen2.5-72b-instruct'  # 添加/修改
config.get('missing_key', 'default_value')      # 安全访问（给默认值）
config.keys()                                    # 所有键
config.values()                                  # 所有值
config.items()                                   # 键值对元组列表

# ── 元组（不可变列表） ──
point = (10, 20)         # 创建后不能修改，类似于 Object.freeze(['a','b'])
name, temp = ('qwen', 0.7)  # 解构赋值（JS: const [name, temp] = ['qwen', 0.7]）

# ── 集合（去重） ──
unique = {'a', 'b', 'a'}   # 结果: {'a', 'b'}
```

### 2.2 f-string：Python 的模板字符串

f-string 是 Python 3.6+ 最常用的字符串格式化方式：

```python
model_name = 'qwen2.5-72b-instruct'
temperature = 0.7

# f-string：字符串前加 f，变量写在大括号里
config_line = f'model: {model_name}, temperature: {temperature}'

# 支持表达式
summary = f'两数之和: {100 + 200}'

# 支持格式化
price = 123.456
display = f'价格: {price:.2f}'  # '价格: 123.46'

# 对比 JavaScript 模板字符串:
# const configLine = `model: ${modelName}, temperature: ${temperature}`;
```

### 2.3 条件判断和循环

```python
# ── 条件判断 ──
if temperature > 0.8:
    print('温度较高，输出更随机')
elif temperature > 0.3:         # elif 不是 else if！
    print('温度适中')
else:
    print('温度较低，输出更确定')

# 链式比较（JS 不支持这种写法）
if 0 < temperature <= 1.0:      # 等价于 temperature > 0 and temperature <= 1.0
    print('温度在有效范围内')

# ── for 循环 ──
for tool in tools:
    print(tool)

# 带索引（enumerate 非常 Pythonic）
for i, tool in enumerate(tools):
    print(f'{i}: {tool}')

# 遍历字典
for key, value in config.items():
    print(f'{key} = {value}')

# ── while 循环 ──
count = 0
while count < 3:
    print(count)
    count += 1         # Python 没有 count++！

# ── 列表推导式（JS 没有直接对应，但非常强大） ──
names = [t.name for t in tools]                         # 提取所有 tool 的名称
active = [t for t in tools if t.startswith('google')]   # 过滤
result = [(t.name, t.description) for t in tools]       # 生成元组列表
# JavaScript 等价：tools.map(t => t.name).filter(...)
```

### 2.4 函数定义

```python
# 基本函数（没有 function 关键字，没有 {}）
def greet(name: str) -> str:
    """返回问候语（这是 docstring）"""
    return f'Hello, {name}!'

# 带默认参数
def create_llm_config(
    model_name: str,
    temperature: float = 0.7,
    max_tokens: int = 1000
) -> dict:
    return {
        'model_name': model_name,
        'temperature': temperature,
        'max_tokens': max_tokens
    }

# 可变参数 *args（元组）和 **kwargs（字典）
def log_all(*args, **kwargs):
    """打印所有传入的参数"""
    for arg in args:
        print(f'位置参数: {arg}')
    for key, value in kwargs.items():
        print(f'关键字参数: {key} = {value}')

log_all('hello', 'world', name='qwen', temp=0.7)
# 输出:
# 位置参数: hello
# 位置参数: world
# 关键字参数: name = qwen
# 关键字参数: temp = 0.7
```

---

## 3. 面向对象：class、self、继承

Python 是纯基于类的 OOP 语言。agentUniverse 中几乎所有组件都是类。

### 3.1 基本类定义

```python
class MyAgent:
    """Agent 的文档字符串（类似 JSDoc）"""

    # 类属性（所有实例共享，类似 JS static）
    component_type = 'AGENT'

    # 构造函数（__init__ 前后各两个下划线！）
    def __init__(self, name: str, temperature: float = 0.7):
        # self.xxx 是实例属性
        self.name = name
        self.temperature = temperature

    # 实例方法 — self 必须显式写出！
    def run(self, input_text: str) -> str:
        return f'{self.name} says: {input_text}'

# 使用
agent = MyAgent(name='qwen_agent', temperature=0.5)
result = agent.run('你好')
print(result)  # "qwen_agent says: 你好"
```

**与 JavaScript 的关键差异：**

| JavaScript | Python |
|-----------|--------|
| `constructor(name, temp) { this.name = name; }` | `def __init__(self, name, temp): self.name = name` |
| `this` 隐式绑定 | `self` 必须显式声明为第一个参数 |
| `class MyAgent extends Base` | `class MyAgent(Base):` |
| `super(name)` | `super().__init__(name)` |
| `static foo = 'bar'` | 类属性直接写在 class 下 |
| `method() { return this.x; }` | `def method(self): return self.x` |

### 3.2 继承与多继承

agentUniverse 的核心继承链：`ComponentBase → Agent → AgentTemplate → ReActAgentTemplate`

```python
# 单继承
class Agent(ComponentBase):
    """所有 Agent 的基类"""
    agent_model: Optional[AgentModel] = None

    def run(self, **kwargs) -> OutputObject:
        ...

# 多继承（Python 支持！JS 不支持原生多继承）
from abc import ABC, abstractmethod

class Agent(ComponentBase, ABC):  # 同时继承 ComponentBase 和 ABC
    """ABC = Abstract Base Class，用于定义抽象方法"""
    
    @abstractmethod
    def input_keys(self) -> list[str]:
        """子类必须实现此方法"""
        pass

# 实例化一个未实现抽象方法的子类 → TypeError
class BadAgent(Agent):
    pass
# agent = BadAgent()  # ❌ TypeError: Can't instantiate abstract class
```

### 3.3 特殊方法（Magic / Dunder Methods）

以双下划线开头和结尾的方法是 Python 的特殊方法：

```python
class Tool:
    def __init__(self, name: str):
        """构造函数: Tool('search')"""
        self.name = name

    def __str__(self) -> str:
        """print(obj) 或 str(obj) 时调用，类似 JS toString()"""
        return f'Tool({self.name})'

    def __repr__(self) -> str:
        """开发者友好的字符串，用于调试（在 REPL 中输入变量名时显示）"""
        return f'Tool(name={self.name!r})'

    def __len__(self) -> int:
        """len(obj) 时调用"""
        return len(self.name)

    def __call__(self, *args):
        """让实例可以像函数一样调用: tool_instance()"""
        print(f'执行工具: {self.name}')

    def __eq__(self, other):
        """== 比较时调用"""
        return self.name == other.name
```

---

## 4. 类型提示（Type Hints）：Python 的"可选 TypeScript"

### 4.1 这是什么？

Python 的类型提示是**纯粹的标注，不影响运行时行为**。类比：你在 JS 代码里写 JSDoc 注释来标注类型，VSCode 能看懂帮你补全，但 Node.js 运行时完全不管。

```python
# 标注了类型，但 Python 不会强制检查
def greet(name: str) -> str:
    return f'Hello, {name}'

# 这行代码运行不会报错！因为运行时没有类型检查
greet(123)  # 传了 int 但标注了 str → 能运行，但 mypy 会警告
```

运行 `mypy agentuniverse/` 相当于 `tsc --noEmit`——只检查类型，不生成任何输出。

### 4.2 agentUniverse 中的实际用法

```python
from typing import Optional, List, Dict, Any, Union

# 来自 agentuniverse/agent/agent_model.py
class AgentModel(BaseModel):
    info: Optional[dict] = dict()       # Optional[X] = X | None
    profile: Optional[dict] = dict()    #     ≈ TypeScript: profile?: Record<string, any>
    plan: Optional[dict] = dict()
    memory: Optional[dict] = dict()
    action: Optional[dict] = dict()

# 来自 agentuniverse/agent/agent.py
def input_keys(self) -> list[str]:      # 返回字符串列表
    return ['input']

def run(self, **kwargs) -> OutputObject:  # 返回 OutputObject 实例
    ...

# 来自 react_agent_template.py
max_iterations: Optional[int] = None    # 可选整数
stop_sequence: Optional[list[str]] = None  # 可选的字符串列表
```

### 4.3 常用类型对照速查

| TypeScript | Python (3.10+) | 旧写法 |
|-----------|---------------|--------|
| `string` | `str` | — |
| `number` | `int` / `float` | — |
| `boolean` | `bool` | — |
| `null \| undefined` | `None` | — |
| `Array<T>` | `list[T]` | `List[T]` from typing |
| `Record<K,V>` | `dict[K, V]` | `Dict[K, V]` |
| `T \| undefined` | `T \| None` | `Optional[T]` |
| `void` | `None`（作为返回） | — |
| `any` | `Any` | `from typing import Any` |
| `never` | `Never` (3.11+) | `NoReturn` |
| `interface {...}` | `class X(BaseModel)` | Pydantic |

---

## 5. Pydantic：Python 的"zod + interface"

### 5.1 一句话理解

**Pydantic = TypeScript 的 `interface` + `zod` 的运行时验证**。你定义一个字段及其类型，Pydantic 自动帮你验证、序列化、反序列化。

```python
from pydantic import BaseModel
from typing import Optional

# 定义一个数据模型
class AgentConfig(BaseModel):
    name: str                          # 必填，str 类型
    temperature: float = 0.7           # 可选，默认 0.7
    max_tokens: int = 1000             # 可选，默认 1000
    tools: list[str] = []              # 可选，默认空列表
    metadata: Optional[dict] = None    # 可选，可以为 None

# ✅ 正常使用
config = AgentConfig(name='qwen_agent', temperature=0.5)
print(config.name)           # 'qwen_agent'
print(config.model_dump())   # {'name': 'qwen_agent', 'temperature': 0.5, ...}

# ❌ 类型不匹配 → 运行时抛 ValidationError
config = AgentConfig(name=123)  # name 期望 str，传了 int → 报错！

# ❌ 缺少必填字段 → 运行时抛 ValidationError
config = AgentConfig()  # name 是必填的 → 报错！
```

**在 agentUniverse 中，`ComponentBase` 继承自 `BaseModel`，所以所有组件（Agent、LLM、Tool、Memory...）自动获得 Pydantic 的数据校验能力。**

### 5.2 和 TypeScript + zod 的对照

```typescript
// TypeScript + zod 等价写法
import { z } from 'zod';

const AgentConfigSchema = z.object({
    name: z.string(),
    temperature: z.number().default(0.7),
    maxTokens: z.number().default(1000),
    tools: z.array(z.string()).default([]),
    metadata: z.record(z.unknown()).nullable().default(null),
});

type AgentConfig = z.infer<typeof AgentConfigSchema>;
```

---

## 6. 装饰器：Python 的"高阶函数语法糖"

装饰器是 agentUniverse 中使用最频繁的 Python 特性之一。

### 6.1 装饰器是什么？

一个函数，接受一个函数或类作为参数，在不修改原代码的前提下增强它的能力。**等价于 JavaScript 的高阶函数包装。**

```python
# 一个最简单的装饰器
def log_call(func):
    """每次调用被装饰的函数时打印日志"""
    def wrapper(*args, **kwargs):
        print(f'调用 {func.__name__}...')
        result = func(*args, **kwargs)
        print(f'{func.__name__} 执行完毕')
        return result
    return wrapper

@log_call
def run_agent(agent_name: str):
    print(f'Agent {agent_name} 正在运行...')

run_agent('demo_agent')
# 输出:
# 调用 run_agent...
# Agent demo_agent 正在运行...
# run_agent 执行完毕
```

### 6.2 `@singleton`：全局唯一实例

agentUniverse 中几乎所有 Manager 类都用 `@singleton` 装饰：

```python
# agentuniverse/base/annotation/singleton.py（简化）
def singleton(cls):
    """让一个类变成单例模式"""
    instances = {}

    def get_instance(*args, **kwargs):
        if cls not in instances:
            instances[cls] = cls(*args, **kwargs)
        return instances[cls]

    return get_instance

# 使用
@singleton
class AgentManager:
    def __init__(self):
        self.pool = {}  # 组件实例池

# 多次调用，返回同一个实例
mgr1 = AgentManager()
mgr2 = AgentManager()
assert mgr1 is mgr2  # True, 是同一个对象！
```

**Node.js 对照：** 相当于模块级单例 `module.exports = new AgentManager()`——所有 import 该模块的地方拿到同一个实例。但 `@singleton` 更灵活：它改变的是类本身，调用者无需知道是否是单例。

### 6.3 `@abstractmethod`：子类必须实现

```python
from abc import ABC, abstractmethod

class Tool(ABC):
    @abstractmethod
    def execute(self, **kwargs):
        """每个 Tool 子类必须实现此方法"""
        pass

# 如果你的 Tool 子类忘记实现 execute()，实例化时会立即报错，不会等到运行时才发现
```

### 6.4 `@trace_agent` / `@trace_tool`：自动可观测性

```python
@trace_agent
def run(self, **kwargs) -> OutputObject:
    """这个装饰器自动记录: 开始时间、输入参数、输出结果、耗时、异常"""
    ...
# 不需要手写任何埋点代码，装饰器帮你搞定了
```

---

## 7. 异步编程：Python 的 async/await

### 7.1 核心差异

JavaScript 是"异步优先"，Python 是"同步优先"：

| 特性 | JavaScript | Python |
|------|-----------|--------|
| 异步声明 | `async function` | `async def` |
| 等待 | `await` | `await` |
| 并发 | `Promise.all([a, b])` | `asyncio.gather(a, b)` |
| 启动入口 | 自动（事件循环） | `asyncio.run(coro)` |
| HTTP 请求 | `fetch()` 原生异步 | `requests` 是同步的！用 `aiohttp` 代替 |

### 7.2 agentUniverse 中的实际使用

```python
import asyncio

class Agent(ComponentBase, ABC):
    # 同步版本
    def run(self, **kwargs) -> OutputObject:
        return self._execute(input_object, agent_input)

    # 异步版本 — 签名和同步版本几乎一样
    async def async_run(self, **kwargs) -> OutputObject:
        result = await self._async_execute(input_object, agent_input)
        return OutputObject(result)
```

### 7.3 三条关键规则

```python
# 规则 1：await 只能在 async def 函数里使用
async def fetch_data():
    result = await some_async_call()    # ✅ 正确

def normal_func():
    result = await some_async_call()    # ❌ SyntaxError

# 规则 2：调用 async 函数不执行，返回的是 coroutine 对象
coro = fetch_data()            # 没执行！返回 coroutine 对象
result = await coro             # 这时才执行
# 或者在同步代码中启动
result = asyncio.run(coro)      # 在同步世界启动异步

# 规则 3：requests 库是同步的，async 函数里需要 HTTP 请用 aiohttp
import requests
resp = requests.get('https://api.example.com')  # ✅ 同步中 OK

import aiohttp
async def fetch_url(url):
    async with aiohttp.ClientSession() as session:
        async with session.get(url) as resp:
            return await resp.json()
```

---

## 8. 模块导入：`import` 的 Python 方式

### 8.1 和 Node.js 的映射

| 概念 | Node.js | Python |
|------|---------|--------|
| 导入整个模块 | `const fs = require('fs')` | `import os` |
| 导入特定符号 | `const { readFile } = require('fs')` | `from os import getenv` |
| 重命名导入 | `const { readFile: rf } = ...` | `from os import getenv as gv` |
| 相对导入 | `require('./utils')` | `from . import utils` |
| 目录 = 模块 | `index.js` 自动解析 | 必须有 `__init__.py` |
| 导入路径 | `node_modules/` + 递归查找 | `sys.path` 列表顺序查找 |

### 8.2 agentUniverse 源码中的导入

```python
# 来自 agentuniverse/agent/agent.py (简化)

# ── 标准库 ──
import json
from abc import abstractmethod, ABC
from typing import Optional, List

# ── 第三方库 ──
from langchain_core.runnables import RunnableSerializable
from pydantic import BaseModel

# ── 本项目模块（绝对导入） ──
from agentuniverse.agent.agent_model import AgentModel
from agentuniverse.agent.action.tool.tool import Tool
from agentuniverse.base.component.component_base import ComponentBase

# ── 本项目模块（相对导入） ──
from .agent_model import AgentModel           # 同目录
from ..base.component import ComponentBase    # 上一级
```

### 8.3 `__init__.py` 是怎么回事？

```python
# agentuniverse/agent/__init__.py
# 这个文件可以为空，但必须有它，agentuniverse/agent/ 才能被 import。
# 如果某个目录没有 __init__.py，Python 不会把它视为包。
# 等价于：Node.js 中 require('./foo') 找不到 foo/index.js
```

---

## 9. 其它 Python 工程师每天都会用的特性

### 9.1 `with` 语句：自动资源管理

```python
# 文件操作：自动关闭
with open('config.yaml', 'r') as f:
    content = f.read()
# 缩进块结束后，文件自动关闭。不需要 f.close()！

# 等价 JavaScript（但 JS 也有 using 了）：
# try {
#     const content = fs.readFileSync('config.yaml', 'utf8');
# } finally { /* close */ }
```

### 9.2 异常处理

```python
try:
    result = agent.run(input='test')
except ValueError as e:             # 捕获特定异常
    print(f'参数错误: {e}')
except Exception as e:              # 捕获所有异常
    print(f'未知错误: {e}')
else:                               # try 块成功执行后运行（JS 没有）
    print('一切正常')
finally:                            # 无论如何都执行
    print('清理工作')
```

### 9.3 生成器（Generator）

```python
def count_up_to(max_num: int):
    """生成 0 到 max_num-1 的数字序列"""
    n = 0
    while n < max_num:
        yield n      # 不是 return！yield 会暂停函数
        n += 1

for num in count_up_to(5):
    print(num)  # 0, 1, 2, 3, 4
# 等价 JavaScript 生成器 function* countUpTo(max) { for(let i=0; i<max; i++) yield i; }
```

### 9.4 `pass`：什么都不做

```python
# 当语法上需要一条语句但你不想写任何逻辑时：
@abstractmethod
def input_keys(self) -> list[str]:
    pass    # "我知道这个方法需要存在，但现在不实现"

# 空类或空配置文件：
class MyConfigExtension:
    pass    # 暂时没有扩展逻辑
```

---

## 10. 动手练习

### 练习 1：对照阅读 agent.py

打开 `agentuniverse/agent/agent.py`，找出以下知识点各一个实例：
- `class` 继承
- `@decorator` 装饰器
- `Optional[X]` 类型提示
- `self.xxx` 实例属性
- `**kwargs` 可变参数
- `async def` 异步方法

### 练习 2：写一个简单的 Tool

用 Python 写一个 `EchoTool` 类：
1. 继承 `Tool` 基类
2. 实现 `execute(self, message: str) -> str` 方法
3. 添加 `@trace_tool` 装饰器
4. 在 `__init__` 中设置 tool name

### 练习 3：追踪装饰器

打开 `agentuniverse/base/annotation/singleton.py`，理解它如何实现单例。然后用自己写的简单例子测试 `@singleton` 的效果。

---

## 11. 常见错误与排查

| 错误类型 | 原因 | 解决方法 |
|---------|------|---------|
| `IndentationError` | 混用 tab 和空格，或缩进层级错误 | 统一用 4 空格，让 ruff/black 自动检查 |
| `NameError: name 'xxx' is not defined` | 变量/函数名拼写错误 | 检查拼写，或用 IDE 自动补全 |
| `AttributeError: 'X' object has no attribute 'Y'` | 访问了不存在的属性 | 检查是否在 `__init__` 中赋值了 `self.y = ...` |
| `TypeError: Can't instantiate abstract class` | 尝试实例化有 @abstractmethod 的类 | 确保实现了所有抽象方法 |
| `ImportError / ModuleNotFoundError` | 模块路径不对或 `__init__.py` 缺失 | 检查路径、`sys.path` 和 `__init__.py` |
| `SyntaxError: 'await' outside async function` | 在普通函数里用了 await | 把函数改成 `async def` |
| `ValidationError` (Pydantic) | 传给 BaseModel 的数据类型不对 | 检查字段类型是否匹配 |
| 忘记 `self` | 方法定义忘记写 self | 实例方法第一个参数必须是 self |

---

## 下一步

现在你应该能：
1. 打开 agentUniverse 任意源码文件，看懂 80% 以上的语法
2. 区分哪些是 Python 语法（`def`、`self`、`@`），哪些是 Pydantic 特性（`BaseModel`）
3. 用 `import` 正确导入本项目的模块
4. 理解装饰器如何增强函数/类

下一章 **[02-agent-fundamentals.md](02-agent-fundamentals.md)** 进入核心——Agent 到底是什么、agentUniverse 的六维模型如何工作。
