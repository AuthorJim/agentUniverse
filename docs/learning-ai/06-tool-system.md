# 06 — Tool 系统：给 Agent 装上手脚

> **前置要求：** 完成 [05-react-pattern.md](05-react-pattern.md)
> **学习目标：** 理解 Tool 的注册执行流程，能够为 Agent 编写自定义工具
> **预计时间：** 2-3 天

---

## 1. Tool 的本质

**一个 Tool 就是一个被 LLM 可编程调用的函数。** 它有三个核心要素：

| 要素 | 含义 | 谁看 |
|------|------|------|
| `name` | 工具名称 | LLM（决定调用哪个工具） |
| `description` | 工具描述 | LLM（决定何时调用此工具） |
| `execute()` | 执行逻辑 | 框架（真正干活） |

对比全栈经验——Tool 类似于：

```
Tool ≈ REST API endpoint
  - name     = 端点路径  (如 /api/search)
  - description = OpenAPI Spec 中的 description
  - execute()  = 端点处理函数
  - input_keys = 请求参数定义
```

**最大的不同：** Tool 的 "文档"（description）是给 LLM 看的而不是给人看的。LLM 通过阅读 description 来决定调用哪个 Tool。

---

## 2. Tool 的源码结构

### 2.1 Tool 基类

```python
# agentuniverse/agent/action/tool/tool.py:48-88（简化）
class Tool(ComponentBase):
    """工具基类"""

    name: str = ""
    description: Optional[str] = None
    tool_type: ToolTypeEnum = ToolTypeEnum.FUNC
    input_keys: Optional[List] = None

    def run(self, **kwargs):
        """调用入口：校验输入 → 调用 execute()"""
        self.input_check(kwargs)
        return self.execute(**kwargs)

    @abstractmethod
    def execute(self, **kwargs):
        """★ 子类必须实现的方法——工具的真正逻辑"""
        raise NotImplementedError

    def as_langchain(self) -> LangchainTool:
        """转为 LangChain 可识别的 Tool 格式"""
        return LangchainTool(
            name=self.name,
            func=self.langchain_run,
            description=self.description    # ← LLM 看到的描述
        )
```

### 2.2 一个真实的 Tool 示例：PythonRunner

```python
# examples/sample_apps/react_agent_app/intelligence/agentic/tool/python_runner.py
import re
from langchain_community.utilities import PythonREPL
from pydantic import Field
from agentuniverse.agent.action.tool.tool import Tool

class PythonRunner(Tool):
    """可以执行 Python 代码的工具"""

    client: PythonREPL = Field(default_factory=lambda: PythonREPL())

    def execute(self, input: str):
        """从 input 中提取 Python 代码并执行"""
        # 1. 尝试提取 ```python ... ``` 包裹的代码块
        pattern = re.compile(r"```python(.*?)```", re.DOTALL)
        matches = pattern.findall(input)
        if len(matches) == 0:
            # 2. 没有代码块标记，直接执行
            return self.client.run(input)

        # 3. 执行提取到的代码
        res = self.client.run(matches[0])
        if res == "" or res is None:
            return "ERROR: 你的 Python 代码中没有使用 print 输出任何内容"
        return res
```

**关键点：**
- 只需要继承 `Tool` 和实现 `execute()` 方法
- 其他一切（输入校验、日志、追踪）都由父类处理
- `@trace_tool` 装饰器在父类的 `run()` 方法上，自动记录调用

### 2.3 Tool 的 YAML 配置

```yaml
# examples/sample_apps/react_agent_app/intelligence/agentic/tool/python_runner.yaml
name: 'python_runner'
description: |
  使用该工具可以执行 Python 代码。
  你的输入必须是 JSON 格式。
  如果你想查看工具的执行结果，必须在 Python 代码中使用 print() 打印内容。
  Args: input: 完整代码
tool_type: 'api'
input_keys: ['input']
metadata:
  type: 'TOOL'
  module: 'react_agent_app.intelligence.agentic.tool.python_runner'
  class: 'PythonRunner'
```

**description 是写给 LLM 的文档。** LLM 看到它后决定"什么时候该调用 python_runner"。

---

## 3. Tool 的完整生命周期

### 3.1 注册阶段

```
启动时 scan() 扫描 YAML
  → 找到 python_runner.yaml
  → 解析 metadata.type = 'TOOL'
  → 解析 metadata.module + metadata.class = PythonRunner 类
  → 实例化 PythonRunner()
  → 调用 initialize_by_component_configer()（注入 name、description、input_keys）
  → 注册到 ToolManager._instance_obj_map
  → Instance Code: 'react_agent_app.tool.python_runner'
```

### 3.2 绑定阶段

```
Agent YAML 的 action.tool 列表:
  tool: ['python_runner', 'google_search_tool']

Agent 初始化时:
  → 按名称从 ToolManager 获取 Tool 实例
  → 转为 LangChain Tool 格式
  → 注入到 ReAct Planner 的工具池
```

### 3.3 调用阶段（ReAct 循环中）

```
LLM 输出: Action: python_runner
          Action Input: {"input": "print(2 + 3)"}

框架解析:
  → extract Action=pyth_runner
  → extract Action Input={"input": "print(2 + 3)"}

Tool 执行:
  → Tool.langchain_run(args='{"input": "print(2 + 3)"}')
  → parse_and_check_json_markdown() → {'input': 'print(2 + 3)'}
  → Tool.run(input='print(2 + 3)')
  → Tool.execute(input='print(2 + 3)')
  → 返回: '5'

框架处理:
  → 将结果格式化为: Observation: 5
  → 追加回 Prompt
  → 进入下一轮 ReAct 循环
```

---

## 4. 编写自定义 Tool 的步骤

### 步骤 1：创建 Python 类

```python
# intelligence/agentic/tool/weather_tool.py
from agentuniverse.agent.action.tool.tool import Tool

class WeatherTool(Tool):
    """查询天气的工具"""

    def execute(self, city: str):
        """根据城市名查询天气（这里用 mock 数据）"""
        # 实际项目中这里调用天气 API
        weather_data = {
            '北京': '晴，15°C~25°C',
            '上海': '多云，18°C~24°C',
            '深圳': '阵雨，22°C~28°C',
        }
        return weather_data.get(city, f'未找到{city}的天气数据')
```

### 步骤 2：创建 YAML 配置

```yaml
# intelligence/agentic/tool/weather_tool.yaml
name: 'weather_tool'
description: |
  查询指定城市的今日天气。当用户询问任何城市的天气时使用此工具。
  Args:
      city: 城市名称（中文）
tool_type: 'api'
input_keys: ['city']
metadata:
  type: 'TOOL'
  module: 'your_app.intelligence.agentic.tool.weather_tool'
  class: 'WeatherTool'
```

### 步骤 3：注册到 Agent

```yaml
# Agent YAML 中
action:
  tool: ['weather_tool']    # 加上你的工具
```

### 步骤 4：确保扫描路径包含 Tool 目录

```toml
# config.toml
[CORE_PACKAGE]
tool = ['your_app.intelligence.agentic.tool']
```

---

## 5. Tool 的设计原则

### 5.1 Description 是最重要的代码

LLM 通过 description 决定是否使用你的 Tool。写好 description：

```yaml
# ✅ 好的 description
description: |
  查询指定城市或地区的实时天气信息。
  当用户询问天气、气温、降水、风力等气象相关问题时使用此工具。
  Args:
    location: 城市或地区名称，如"北京"或"上海浦东"
  Returns:
    包含温度、天气状况、湿度、风速的文本描述

# ❌ 不好的 description
description: '查询天气'
```

### 5.2 单一职责

每个 Tool 只做一件事：

```
✅ python_runner: 只执行 Python 代码
✅ google_search: 只搜索网页
✅ weather_tool: 只查天气

❌ mega_tool: 搜索 + 计算 + 翻译 + 天气
```

LLM 更容易选择专一工具，复合工具让 LLM 困惑。

### 5.3 明确的输入和输出

```python
def execute(self, city: str) -> str:
    """输入城市名，返回天气字符串"""
```

- 参数名清晰（`city` 而不是 `arg1`）
- 返回类型明确（`str` 而不是 `Any`）
- 错误时有友好提示（不是空字符串）

### 5.4 幂等性（如果可能）

同样参数调用多次应返回相同结果。搜索工具天然不一定幂等（搜索结果会变），但计算工具应该是。

---

## 6. Tool 的高级特性

### 6.1 异步 Tool

```python
class AsyncWeatherTool(Tool):
    async def async_execute(self, city: str):
        """异步查询天气"""
        async with aiohttp.ClientSession() as session:
            async with session.get(f'https://api.weather.com/{city}') as resp:
                return await resp.text()
```

父类的 `async_run()` 方法自动支持——如果子类实现了 `async_execute`。

### 6.2 Tool 参数校验

```python
class CalculatorTool(Tool):
    input_keys: List[str] = ['expression']

    def execute(self, expression: str):
        # 安全校验：只允许数字和基本运算符
        import re
        if not re.match(r'^[\d\+\-\*\/\(\)\.\s]+$', expression):
            return 'ERROR: 表达式包含不允许的字符'
        return str(eval(expression))
```

父类的 `input_check()` 会验证 `input_keys` 中的参数是否存在。但值的合法性校验需要在 `execute()` 中自己做。

### 6.3 Pydantic Field 默认值

```python
from pydantic import Field
from agentuniverse.base.util.env_util import get_from_env

class GoogleSearchTool(Tool):
    serper_api_key: Optional[str] = Field(
        default_factory=lambda: get_from_env("SERPER_API_KEY")
    )
    # ★ 自动从环境变量或 custom_key.toml 读取 API Key
```

---

## 7. 动手练习

### 练习 1：阅读 GoogleSearchTool 源码

打开 `examples/sample_apps/react_agent_app/intelligence/agentic/tool/google_search_tool.py`：
1. 理解它如何从环境变量读取 API Key
2. 理解 `execute()` 的参数签名
3. 追踪它的 langchain_run → run → execute 调用链

### 练习 2：编写一个 Calculator Tool

创建一个能执行基本四则运算的 Tool。要求：
- `name`: calculator_tool
- `description`: 描述清楚什么情况下使用
- `execute(expression: str)`: 解析并计算表达式
- YAML 配置完整

### 练习 3：编写一个翻译 Tool

创建一个翻译 Tool。要求：
- `execute(text: str, target_language: str)`: 翻译文本到目标语言
- YAML 中 `input_keys: ['text', 'target_language']`
- 添加到 Agent 中测试

### 练习 4：理解工具注册链路

画出一个 Tool 从 YAML 到内存实例的完整链路图，包括：
- scan() 发现 YAML
- ComponentConfiger 解析配置
- 动态导入 Python 类
- initialize_by_component_configer() 注入配置
- ToolManager.register() 登记
- Agent 通过 tool: ['tool_name'] 绑定

---

## 下一步

你已经理解了 Tool 系统——如何为 Agent 编写工具、工具如何注册和绑定、工具在 ReAct 循环中如何被调用。

下一步进入 **07-memory-system.md**，学习 Memory 系统——让 Agent 记住上下文，不再"金鱼记忆"。
