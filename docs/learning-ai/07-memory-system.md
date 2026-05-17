# 07 — Memory 系统：给 Agent 装上记忆

> **前置要求：** 完成 [06-tool-system.md](06-tool-system.md)
> **学习目标：** 理解 Memory 的三层架构（Memory → Storage → Compressor），掌握对话记忆的配置、裁剪和压缩策略
> **预计时间：** 1-2 天

---

## 1. 为什么需要 Memory？—— 金鱼 vs 大象

### 1.1 没有 Memory 的 Agent

```
用户: "我叫小明"
Agent: "你好小明！"
用户: "我喜欢喝咖啡"
Agent: "咖啡确实不错！"
用户: "我叫什么名字？"
Agent: "抱歉，我不知道你的名字。"    ← 忘了！
```

每次请求都是独立的，Agent 不记得上一轮说的任何东西。这就像一个**失忆的客服**——每通电话都是"您好，第一次为您服务"。

### 1.2 有 Memory 的 Agent

```
用户: "我叫小明"
Agent: "你好小明！"          [Memory 记录: 用户叫小明]
用户: "我喜欢喝咖啡"
Agent: "小明你喜欢喝咖啡啊！"  [Memory 记录: 小明喜欢咖啡]
用户: "我叫什么名字？"
Agent: "你叫小明。"          ← 记住了！
```

### 1.3 Memory 在技术上做了什么？

非常简单：**每次请求时，把之前的对话历史注入到 Prompt 的 `{chat_history}` 占位符中。**

```yaml
# Prompt 模板中
instruction: |
  历史对话:
  {chat_history}            ← 这里注入之前的对话记录

  用户当前问题: {input}
```

就这么简单。但简单背后有很多工程细节——存哪里？太长了怎么办？用什么格式？

---

## 2. agentUniverse Memory 的三层架构

```
Memory（记忆管理器）
  │  职责: 决定"存什么、怎么取、什么时候裁剪"
  │
  ├── MemoryStorage（存储后端）── "对话存哪里？"
  │   ├── RAM Memory Storage      → 内存（最快，但重启即丢）
  │   ├── Chroma Memory Storage   → ChromaDB（持久 + 向量检索）
  │   └── SQLite Memory Storage   → SQLite 文件（本地持久化）
  │
  └── MemoryCompressor（压缩器）── "对话太长了怎么办？"
      └── 用 LLM 把长对话摘要为一段概述
```

### 2.1 Memory 基类详解

```python
# agentuniverse/agent/memory/memory.py:27-56
class Memory(ComponentBase):
    name: str = ""
    description: Optional[str] = None
    type: MemoryTypeEnum = None              # short-term / long-term
    memory_key: str = 'chat_history'         # ★ 注入 Prompt 的变量名
    max_tokens: int = 2000                   # ★ Token 上限（超出即裁剪）
    memory_compressor: Optional[str] = None  # 压缩器名称
    memory_storages: List[str] = ['ram_memory_storage']  # 存储后端列表
    memory_retrieval_storage: Optional[str] = None       # 检索用的后端

    def add(self, message_list, session_id=None, agent_id=None):
        """写入：将新消息存入所有配置的 Storage"""
        for storage_name in self.memory_storages:
            storage = MemoryStorageManager().get_instance_obj(storage_name)
            if storage:
                storage.add(message_list, session_id, agent_id)

    def get(self, session_id=None, agent_id=None, prune=True):
        """读取：从检索 Storage 获取历史，超出则裁剪"""
        storage = MemoryStorageManager().get_instance_obj(self.memory_retrieval_storage)
        if storage:
            memories = storage.get(session_id, agent_id)
            if prune:
                memories = self.prune(memories)  # ★ 超出 max_tokens 就裁剪
            return memories
        return []
```

### 2.2 Message 对象

```python
# agentuniverse/agent/memory/message.py
class Message(BaseModel):
    content: str                    # 消息内容
    type: str = 'human'             # system / human / ai
    # 其他元数据字段...
```

每条对话记录是一个 `Message` 对象——类似于在数据库里存的一条"聊天记录"行。

---

## 3. Memory 的核心工作流程

### 3.1 写入：Agent 回复后自动存储

```
Agent.run() 的完整流程:
  ├── ① 从 Memory 加载历史 → 注入 {chat_history}
  ├── ② LLM 推理 → 生成回复
  └── ③ 存入 Memory:
        memory.add([
            Message(type='human', content=user_input),
            Message(type='ai', content=agent_output)
        ])
```

### 3.2 读取：每次请求时自动注入

```python
# agentuniverse/agent/agent.py:164-181
def pre_parse_input(self, input_object) -> dict:
    agent_input = dict()
    # ★ 从 Memory 加载历史对话
    agent_input['chat_history'] = input_object.get_data('chat_history') or ''
    agent_input['date'] = datetime.now().strftime('%Y-%m-%d')
    return agent_input
```

### 3.3 裁剪（Pruning）：防止超出 Token 上限

这是 Memory 系统最精妙的部分：

```python
# agentuniverse/agent/memory/memory.py:97-120 (简化)
def prune(self, memories: List[Message]) -> List[Message]:
    """当对话历史 tokens > max_tokens 时，从最早的消息开始裁剪"""

    # ① 计算当前历史的 token 数
    current_tokens = get_memory_tokens(memories, agent_llm_name)

    if current_tokens <= self.max_tokens:
        return memories  # 没超出，直接返回

    # ② 从最早的消息开始 pop，直到不超出
    pruned = []
    while get_memory_tokens(memories, agent_llm_name) > self.max_tokens:
        pruned.append(memories.pop(0))  # 移除最早的

    # ③ 如果有 Compressor，被裁掉的消息不会丢弃，而是先压缩
    if pruned and self.memory_compressor:
        compressor = MemoryCompressorManager().get_instance_obj(self.memory_compressor)
        if compressor:
            compressed_msg = compressor.compress_memory(pruned, ...)
            if compressed_msg:
                # ★ 将压缩后的摘要插入到剩余消息的最前面
                memories.insert(0, Message(
                    type='system',
                    content=f'[对话历史摘要] {compressed_msg}'
                ))

    return memories
```

**为什么不全丢？** 因为早期对话可能包含重要信息（如"用户叫小明"）。压缩比丢弃好——保留摘要至少能让 Agent 知道"之前大致聊了什么"。

---

## 4. 存储后端对比

| 存储后端 | 持久化 | 速度 | 适用场景 |
|---------|--------|------|---------|
| **RAM** | ❌ 进程重启即丢失 | 最快 | 开发测试、短会话 |
| **ChromaDB** | ✅ 持久化到磁盘 | 快 | 生产环境、长会话、需要向量检索 |
| **SQLite** | ✅ 持久化到文件 | 快 | 小规模生产、需要 SQL 查询 |

在 Agent YAML 中配置：

```yaml
memory:
  name: 'demo_memory'
  memory_storages: ['chroma_memory_storage']        # 写入这个 Storage
  memory_retrieval_storage: 'chroma_memory_storage'  # 从这里读取
  max_tokens: 2000
```

**Node.js 对照：**
- RAM Storage = `new Map()` 或 `{}` 存 session 数据
- ChromaDB Storage = Redis（持久化 + 向量搜索）
- SQLite Storage = SQLite 文件数据库

---

## 5. MemoryCompressor：长对话的摘要压缩

### 5.1 工作原理

```
原始对话（30 轮，超出 max_tokens）:
  轮1: "我叫小明" / "你好小明"
  轮2: "Python是什么" / "Python是一门..."
  ...
  轮28: "装饰器的语法" / "装饰器是@..."
  轮29: "帮我写个代码" / "好的..."
  轮30: "这段代码有bug" / "让我看看..."

MemoryCompressor 压缩被裁掉的消息（轮1-20）:
  [摘要] 用户小明在前半段对话中询问了Python基础知识，
  包括变量、函数、类、装饰器等概念。Agent对每个主题
  提供了代码示例和解释。

新 Prompt 注入:
  {chat_history} =
    [摘要] + 轮21~轮30 的完整对话记录
```

### 5.2 配置方法

```yaml
# Memory YAML
name: 'demo_memory'
max_tokens: 2000
memory_compressor: 'demo_memory_compressor'   # 指向 Compressor 实例

# Memory Compressor YAML
name: 'demo_memory_compressor'
compressor_prompt_version: 'memory_compressor_prompt.v1'
compressor_llm_name: 'qwen_llm'   # 用哪个 LLM 做摘要
metadata:
  type: 'MEMORY_COMPRESSOR'
  module: 'agentuniverse.agent.memory.memory_compressor'
  class: 'MemoryCompressor'
```

---

## 6. 动手练习

### 练习 1：配置 Memory 观察效果

1. 修改 `simple_qa_agent`，取消 memory 配置的注释
2. 多轮对话：`agent.run(input='我叫小明')` → `agent.run(input='我喜欢喝咖啡')` → `agent.run(input='我叫什么名字？')`
3. 观察 Agent 是否记住了第一轮的输入

### 练习 2：调整 max_tokens 观察裁剪

将 `max_tokens` 设为很小的值（如 50），进行 10 轮以上的长对话。观察：
- 什么时候开始"遗忘"早期对话
- 如果有 Compressor，摘要是否保留了关键信息

### 练习 3：切换存储后端

将 `ram_memory_storage` 改为 `chroma_memory_storage`：
1. 启动 Agent 进行多轮对话
2. 重启 Agent 服务
3. 再次对话 — 观察对话历史是否保留

### 练习 4：阅读 prune() 源码

打开 `agentuniverse/agent/memory/memory.py`，阅读 `prune()` 方法。理解：
1. Token 数量如何计算
2. 裁剪顺序（从最早的消息开始）
3. 压缩摘要的触发条件

### 练习 5：对比有无 Memory

用同一个 Agent 进行以下对比实验：
1. 无 Memory：连续问 3 个关联问题，观察 Agent 的"失忆"表现
2. 有 Memory：同样的问题序列，观察 Agent 是否能对答如流

---

## 7. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **Memory** | 对话记忆管理器 | `agent/memory/memory.py` |
| **memory_key** | Prompt 中的变量名（默认 `chat_history`） | — |
| **max_tokens** | Token 上限（超出触发裁剪） | — |
| **MemoryStorage** | 存储后端（RAM / Chroma / SQLite） | `agent/memory/memory_storage/` |
| **MemoryCompressor** | 长对话摘要压缩器 | `agent/memory/memory_compressor.py` |
| **prune()** | 裁剪方法（pop 最早消息 + 压缩摘要） | `memory.py:97` |
| **Message** | 单条消息（type + content） | `agent/memory/message.py` |
| **session_id** | 会话标识（区分不同用户/对话） | — |

---

## 下一步

你已经掌握了 Memory 系统。下一步进入 **[08-rag-knowledge.md](08-rag-knowledge.md)**——RAG 知识注入，让 Agent 能阅读你的私有文档。
