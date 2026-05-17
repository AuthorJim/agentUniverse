# 06 — Tool 系统：给 Agent 装上手脚

> **前置要求：** 完成 [05-react-pattern.md](05-react-pattern.md)
> **学习目标：** 理解 Tool 的完整生命周期（注册→绑定→调用），能够独立编写自定义 Tool 并绑定到 Agent
> **预计时间：** 2-3 天

---

## 1. Tool 的本质：给 LLM 一个可调用的函数

### 1.1 一个 Tool 就是三个东西

| 要素 | 含义 | 谁是"读者" |
|------|------|-----------|
| `name` | 工具名称 | LLM — 用来决定"调用哪个工具" |
| `description` | 工具描述 | LLM — 用来决定"什么时候用这个工具" |
| `execute()` | 执行逻辑 | 框架 — 真正干活的代码 |

**和 REST API 的类比（你应该很熟了）：**

```
Tool ≈ REST API Endpoint
  name        = URL 路径 (GET /api/search)
  description = OpenAPI Spec 的 description 字段
  execute()   = 路由 handler 函数
  input_keys  = 请求参数定义 (query params / body schema)
```

### 1.2 最关键的差异：description 是给 LLM 看的

REST API 的文档是写给人类开发者看的，Tool 的 description 是写给 LLM 看的。LLM 通过阅读 description 来判断"我现在该不该用这个工具"。

```python
# ✗ 糟糕的 description（LLM 看不懂什么时候该用）
description: '执行计算'

# ✓ 好的 description（LLM 能准确匹配使用场景）
description: |
  当你需要进行数学计算时使用此工具。
  支持加、减、乘、除、幂运算和基本数学函数。
  输入必须是合法的 Python 数学表达式。
  适用场景: 用户问"计算xxx"、"xxx等于多少"等需要精确计算的问题。
```

---

## 2. Tool 的源码结构

### 2.1 Tool 基类

```python
# agentuniverse/agent/action/tool/tool.py:48-88 (简化)
class Tool(ComponentBase):
    """所有 Tool 的基类"""

    name: str = ""                           # 工具名
    description: Optional[str] = None        # ★ LLM 看到的文档
    tool_type: ToolTypeEnum = ToolTypeEnum.FUNC
    input_keys: Optional[List] = None        # 参数名列表

    def run(self, **kwargs):
        """★ 调用入口：先校验输入，再调用 execute()"""
        self.input_check(kwargs)
        return self.execute(**kwargs)

    @abstractmethod
    def execute(self, **kwargs):
        """★ 子类必须实现的方法 — 工具的真正逻辑"""
        raise NotImplementedError

    def as_langchain(self) -> LangchainTool:
        """转为 LangChain 兼容的 Tool 格式"""
        return LangchainTool(
            name=self.name,
            func=self.langchain_run,
            description=self.description    # ← LLM 看到的
        )

    def langchain_run(self, *args, **kwargs):
        """LangChain 调用入口 — 解析 JSON 参数并调用 run()"""
        parse_result = parse_and_check_json_markdown(args[0], self.input_keys)
        return self.run(**parse_result)
```

### 2.2 一个真实的 Tool：PythonRunner

```python
# examples/sample_apps/react_agent_app/intelligence/agentic/tool/python_runner.py
import re
from langchain_community.utilities import PythonREPL
from pydantic import Field
from agentuniverse.agent.action.tool.tool import Tool

class PythonRunner(Tool):
    """执行 Python 代码的工具"""

    # Pydantic Field: PythonREPL 是代码执行沙箱
    client: PythonREPL = Field(default_factory=lambda: PythonREPL())

    def execute(self, input: str) -> str:
        """从 input 中提取 Python 代码并执行"""
        # ① 尝试提取 ```python ... ``` 包裹的代码块
        pattern = re.compile(r"```python(.*?)```", re.DOTALL)
        matches = pattern.findall(input)
        if len(matches) == 0:
            # ② 如果没有代码块标记，直接执行整个 input
            return self.client.run(input)

        # ③ 执行提取到的代码
        res = self.client.run(matches[0])
        if res == "" or res is None:
            return "ERROR: 代码中没有使用 print() 输出内容"
        return res
```

**关键观察：**
- 你只需要继承 `Tool` 和实现 `execute()` 方法
- 输入校验、日志追踪、LangChain 适配都由父类自动处理
- `@trace_tool` 装饰器在父类 `run()` 方法上，自动记录调用

### 2.3 Tool 的 YAML 配置

```yaml
# intelligence/agentic/tool/python_runner.yaml
name: 'python_runner'
description: |
  使用该工具可以执行 Python 代码。
  输入必须是 JSON 格式: {"input": "你的 Python 代码"}
  注意：如果想看到执行结果，必须在代码中使用 print()。
  适用场景：用户需要进行数学计算、数据处理、字符串操作等。
  不要使用此工具执行危险操作（如文件删除、系统命令）。
tool_type: 'api'
input_keys: ['input']
metadata:
  type: 'TOOL'
  module: 'react_agent_app.intelligence.agentic.tool.python_runner'
  class: 'PythonRunner'
```

---

## 3. Tool 的完整生命周期

### 3.1 注册阶段（启动时）

```
启动时 scan() 扫描 YAML
  → 找到 python_runner.yaml
  → 解析 metadata.type = 'TOOL'
  → 解析 metadata.module + metadata.class = PythonRunner 类
  → 动态导入 PythonRunner
  → 实例化 PythonRunner()
  → 调用 initialize_by_component_configer() 注入 name、description 等
  → 注册到 ToolManager._instance_obj_map
  → key: 'react_agent_app.tool.python_runner'
```

### 3.2 绑定阶段（Agent 初始化时）

```
Agent YAML:
  action:
    tool: ['python_runner', 'google_search_tool']

Agent 初始化:
  → 遍历 tool 列表
  → 按名称从 ToolManager 获取实例
  → 调用 tool.as_langchain() 转为 LangChain 格式
  → 添加到 Planner 的工具池
```

### 3.3 调用阶段（ReAct 循环中）

```
LLM 决定使用工具:
  Action: python_runner
  Action Input: {"input": "print(123 * 456)"}

框架处理:
  → 解析出 Action='python_runner', Input='{"input": "print(123 * 456)"}'
  → ToolManager().get_instance_obj('python_runner')
  → tool.langchain_run('{"input": "print(123 * 456)"}')
  → parse_and_check_json_markdown() → {'input': 'print(123 * 456)'}
  → tool.run(input='print(123 * 456)')
  → tool.execute(input='print(123 * 456)')
  → 返回: '56088'

  → 格式化为: Observation: 56088
  → 追加到 agent_scratchpad
  → 进入下一轮 ReAct 循环
```

---

## 4. 编写自定义 Tool 的完整步骤

### 步骤 1：创建 Python 类

```python
# intelligence/agentic/tool/weather_tool.py
import requests
from agentuniverse.agent.action.tool.tool import Tool

class WeatherTool(Tool):
    """查询城市天气的工具"""

    def execute(self, city: str) -> str:
        """根据城市名查询天气信息"""
        # 实际项目中这里调用天气 API
        # response = requests.get(f'https://api.weather.com/v1/current?city={city}')
        # return response.json()['summary']

        # 开发阶段用 mock 数据
        weather_map = {
            '北京': '晴，15°C~25°C，北风3级',
            '上海': '多云，18°C~24°C，东南风2级',
            '深圳': '阵雨，22°C~28°C，西南风3级',
        }
        return weather_map.get(city, f'未找到"{city}"的天气数据，请检查城市名称')
```

### 步骤 2：创建 YAML 配置

```yaml
# intelligence/agentic/tool/weather_tool.yaml
name: 'weather_tool'
description: |
  查询指定城市或地区的实时天气信息。
  当用户询问以下内容时使用此工具：
  - 某城市的天气、气温
  - 是否需要带伞（降水概率）
  - 穿衣建议（温度范围）
  Args:
    city: 城市名称（中文，如"北京"、"上海浦东"）
  Returns:
    包含温度、天气状况、湿度和风速的文字描述
tool_type: 'api'
input_keys: ['city']
metadata:
  type: 'TOOL'
  module: 'your_app.intelligence.agentic.tool.weather_tool'
  class: 'WeatherTool'
```

### 步骤 3：在 Agent YAML 中绑定

```yaml
action:
  tool: ['weather_tool', 'python_runner']    # 加上你的工具
```

### 步骤 4：确保 config.toml 扫描路径包含 Tool 目录

```toml
[CORE_PACKAGE]
tool = ['your_app.intelligence.agentic.tool']
```

### 步骤 5：启动验证

```python
from agentuniverse.base.agentuniverse import AgentUniverse
from agentuniverse.agent.agent_manager import AgentManager

AgentUniverse().start(config_path='config/config.toml', core_mode=True)
agent = AgentManager().get_instance_obj('your_agent')
result = agent.run(input='北京今天天气怎么样？')
print(result.get_data('output'))
```

---

## 5. Tool 的设计原则

### 5.1 Description 是你最重要的代码

Description 直接决定了 LLM 会不会在正确的时候调用你的 Tool：

```yaml
# ✅ 好的 description — 告诉 LLM 什么时候用、怎么用、返回什么
description: |
  查询指定城市的实时天气信息。
  当用户询问天气、气温、降水、风力等气象问题时使用此工具。
  Args: city (城市名，中文)
  Returns: 温度、天气状况、湿度和风速的文字描述

# ❌ 不好的 description — LLM 不知道该不该用
description: '查询天气'
```

### 5.2 单一职责

每个 Tool 只做一件事，LLM 更容易准确选择：

```
✅ python_runner  → 只执行 Python 代码
✅ google_search  → 只搜索网页
✅ weather_tool   → 只查天气

❌ mega_tool      → 搜索 + 计算 + 翻译 + 天气（LLM 会困惑）
```

### 5.3 输入输出明确

```python
# ✅ 参数名清晰，返回类型明确
def execute(self, city: str) -> str:
    ...

# ❌ 参数名模糊
def execute(self, arg1: str) -> Any:
    ...
```

### 5.4 错误友好

```python
def execute(self, city: str) -> str:
    try:
        return weather_api.get(city)
    except CityNotFoundError:
        return f'未找到"{city}"的天气数据。可查询的城市有：北京、上海、深圳...'
    except NetworkError:
        return '天气服务暂时不可用，请稍后重试。'
    # 不要返回空字符串或 None — LLM 不知道发生了什么
```

---

## 6. Tool 高级特性

### 6.1 异步 Tool

如果 Tool 需要异步 I/O（如 HTTP 请求外部 API）：

```python
import aiohttp

class AsyncWeatherTool(Tool):
    async def async_execute(self, city: str) -> str:
        """异步查询天气"""
        async with aiohttp.ClientSession() as session:
            url = f'https://api.weather.com/v1/current?city={city}'
            async with session.get(url) as resp:
                data = await resp.json()
                return data['summary']
```

父类的 `async_run()` 方法会自动调用 `async_execute`（如果子类实现了的话）。

### 6.2 参数校验

父类的 `input_check()` 只验证参数是否存在。值的合法性校验需要在 `execute()` 中自己实现：

```python
class CalculatorTool(Tool):
    input_keys = ['expression']

    def execute(self, expression: str) -> str:
        # 安全校验：只允许数字和基本运算符
        import re
        if not re.match(r'^[\d+\-*/().%\s]+$', expression):
            return 'ERROR: 表达式包含不允许的字符。只允许数字和 + - * / ( ) %'
        try:
            return str(eval(expression))
        except Exception as e:
            return f'ERROR: 表达式计算失败 — {e}'
```

### 6.3 环境变量 / API Key 注入

```python
from pydantic import Field
from agentuniverse.base.util.env_util import get_from_env

class GoogleSearchTool(Tool):
    # ★ 自动从 custom_key.toml 或环境变量读取
    serper_api_key: Optional[str] = Field(
        default_factory=lambda: get_from_env("SERPER_API_KEY")
    )

    def execute(self, query: str) -> str:
        # 使用 self.serper_api_key 调用 Serper API
        response = requests.post(
            'https://google.serper.dev/search',
            json={'q': query},
            headers={'X-API-KEY': self.serper_api_key}
        )
        return response.json()['organic'][0]['snippet']
```

---

## 7. 动手练习

### 练习 1：阅读 GoogleSearchTool 源码

打开 `examples/sample_apps/react_agent_app/intelligence/agentic/tool/google_search_tool.py`，追踪：
1. API Key 的获取方式（`get_from_env`）
2. `execute()` 的参数 → 返回值
3. 从 `langchain_run → run → execute` 的完整链路

### 练习 2：编写一个 Calculator Tool

要求：
- 继承 `Tool`，实现 `execute(expression: str) -> str`
- 支持 + - * / ( )，过滤非法输入
- 写好 description（让 LLM 知道什么时候用它）
- 完整的 YAML 配置

### 练习 3：编写一个 Joke Tool

要求：
- `execute(topic: str = 'random') -> str`
- 根据主题返回一个笑话（可以 mock 数据）
- 让 Agent 在用户问"讲个笑话"时自动调用

### 练习 4：编写一个数据库查询 Tool

要求：
- `execute(sql: str) -> str`
- 安全过滤（只允许 SELECT，禁止 INSERT/UPDATE/DELETE/DROP）
- `input_keys: ['sql']`

### 练习 5：画出 Tool 完整生命周期

画一张图，展示 Tool 从 YAML 配置到 ReAct 循环中被调用执行的完整链路。

---

## 8. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **Tool** | Agent 可调用的外部功能 | `agent/action/tool/tool.py` |
| **name** | 工具名（LLM 用来选择） | — |
| **description** | 工具文档（LLM 用来判断何时调用） | — |
| **execute()** | 工具的真正逻辑 | 子类实现 |
| **run()** | 调用入口（封装校验 + 追踪） | `tool.py` |
| **as_langchain()** | 转为 LangChain 兼容格式 | `tool.py` |
| **langchain_run()** | LangChain → agentUniverse 桥接 | `tool.py` |
| **input_keys** | 参数名列表（用于校验） | YAML + Python |
| **@trace_tool** | 自动记录工具调用（OTEL） | `tool.py` 的 `run()` 上 |
| **ToolManager** | 全局 Tool 注册表 | `agent/action/tool/tool_manager.py` |

---

## 下一步

你已经掌握了给 Agent 编写自定义 Tool 的完整技能。下一步进入 **[07-memory-system.md](07-memory-system.md)**——记忆系统，让 Agent 不再"金鱼记忆"。
