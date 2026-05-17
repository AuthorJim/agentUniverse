# 07 — Memory 系统：让 Agent 记住上下文

> **前置要求：** 完成 [06-tool-system.md](06-tool-system.md)
> **学习目标：** 理解 Memory 的三层架构（Storage / Compressor / Manager），掌握对话记忆的配置与使用
> **预计时间：** 1-2 天

---

## 1. 为什么需要 Memory？

没有 Memory 的 Agent 是"金鱼记忆"——每次对话都是全新的，记不住上一轮说过什么。

```
无 Memory:
  用户: "我叫小明"
  Agent: "你好小明！"
  用户: "我叫什么名字？"
  Agent: "我不知道你的名字。"         ← 忘了

有 Memory:
  用户: "我叫小明"
  Agent: "你好小明！"  [存入 Memory]
  用户: "我叫什么名字？"
  Agent: "你叫小明。"                ← 记住了
```

**Memory 的本质：** 在每轮对话的 Prompt 中注入之前的对话历史 `{chat_history}`。

---

## 2. agentUniverse Memory 的三层架构

```
Memory（记忆管理器）
  │  决定：对话存哪里？太长了怎么压缩？什么时候清理？
  │
  ├── MemoryStorage（存储后端）── "存哪里？"
  │   ├── RAM Memory Storage     → 内存（进程重启即丢失）
  │   ├── Chroma Memory Storage  → ChromaDB（持久化 + 向量检索）
  │   └── SQLite Memory Storage  → SQLite 本地文件
  │
  └── MemoryCompressor（压缩器）── "太长了怎么办？"
      └── 用 LLM 把长对话摘要为简短概述
```

### 2.1 Memory 基类

```python
# agentuniverse/agent/memory/memory.py:27-56
class Memory(ComponentBase):
    name: str = ""
    description: Optional[str] = None
    type: MemoryTypeEnum = None          # short-term or long-term
    memory_key: str = 'chat_history'     # ★ 在 Prompt 中注入的变量名
    max_tokens: int = 2000               # ★ Token 上限（超出则裁剪）
    memory_compressor: Optional[str] = None     # 压缩器名称
    memory_storages: List[str] = ['ram_memory_storage']  # 存储后端列表
    memory_retrieval_storage: Optional[str] = None       # 检索用的后端

    def add(self, message_list, session_id=None, agent_id=None):
        """添加消息到所有存储后端"""
        for storage in self.memory_storages:
            memory_storage = MemoryStorageManager().get_instance_obj(storage)
            if memory_storage:
                memory_storage.add(message_list, session_id, agent_id)

    def get(self, session_id=None, agent_id=None, prune=True):
        """从检索后端获取消息（超出 max_tokens 会自动裁剪）"""
        memory_storage = MemoryStorageManager().get_instance_obj(self.memory_retrieval_storage)
        if memory_storage:
            memories = memory_storage.get(session_id, agent_id)
            if prune:
                memories = self.prune(memories)  # ★ 超出 token 限制就裁剪
            return memories
        return []
```

### 2.2 Message 对象

```python
# agentuniverse/agent/memory/message.py
class Message(BaseModel):
    content: str                           # 消息内容
    type: str = 'human'                    # system / human / ai
    # 其他元数据...
```

每条对话记录是一个 `Message` 对象，带有角色类型（system / human / ai）。

---

## 3. Memory 的工作流程

### 3.1 写入：Agent 回复后自动存储

```
Agent.run()
  ├── ① 从 Memory 加载历史 → get_memory_string() → {chat_history}
  ├── ② LLM 推理 → 生成回复
  └── ③ 存入 Memory → memory.add([Message(human=input), Message(ai=output)])
```

### 3.2 读取与注入：每次请求时自动注入 Prompt

```python
# agent.py:164-181
def pre_parse_input(self, input_object) -> dict:
    agent_input = dict()
    # ★ 从 Memory 加载历史，注入到 chat_history 变量
    agent_input['chat_history'] = input_object.get_data('chat_history') or ''
    agent_input['date'] = datetime.now().strftime('%Y-%m-%d')
    ...
```

然后 Prompt 中通过 `{chat_history}` 引用：

```yaml
instruction: |
  之前的对话:
  {chat_history}

  今天的日期是: {date}
  用户问题: {input}
```

### 3.3 裁剪（Pruning）：防止超出 Context Window

```python
# memory.py:97-120
def prune(self, memories: List[Message]) -> List[Message]:
    """当对话历史 token 数超出 max_tokens 时，从最早的消息开始移除"""
    new_memories = memories[:]
    tokens = get_memory_tokens(new_memories, agent_llm_name)

    if tokens <= self.max_tokens:
        return new_memories    # 没超出，直接返回

    # ★ 从最早的消息开始弹出，直到不超过 max_tokens
    pruned_memories = []
    while tokens > self.max_tokens:
        pruned_memory = new_memories.pop(0)
        pruned_memories.append(pruned_memory)
        tokens = get_memory_tokens(new_memories, agent_llm_name)

    # ★ 如果有 MemoryCompressor，对被裁掉的消息做摘要
    if pruned_memories:
        memory_compressor = MemoryCompressorManager().get_instance_obj(self.memory_compressor)
        if memory_compressor:
            compressed_memory = memory_compressor.compress_memory(pruned_memories, ...)
            if compressed_memory:
                new_memories.insert(0, Message(content=compressed_memory))
    return new_memories
```

**关键理解：** 当对话太长，最早的消息会被"压缩"为一条摘要，而不是直接丢弃。这保证了 Agent 始终有"之前的上下文"。

---

## 4. 存储后端对比

| 后端 | 持久化 | 适用场景 | 特点 |
|------|--------|---------|------|
| **RAM** | ❌ 进程重启丢失 | 开发测试、短会话 | 最快 |
| **ChromaDB** | ✅ | 生产环境、长会话 | 向量检索，可按语义搜索历史 |
| **SQLite** | ✅ | 小规模生产 | SQL 查询，兼容性好 |

在 Agent YAML 中配置：

```yaml
memory:
  name: 'demo_memory'
  memory_storages: ['chroma_memory_storage']    # 存储到 ChromaDB
  memory_retrieval_storage: 'chroma_memory_storage'
  max_tokens: 2000
```

---

## 5. MemoryCompressor：对话摘要

当对话超出 `max_tokens`，MemoryCompressor 用 LLM 对被裁掉的消息做摘要：

```
原始对话（超长）:
  用户: "我想了解 Python..."
  Agent: "Python 是一门..."
  用户: "那 Python 的装饰器..."
  Agent: "装饰器是..."
  ...(30 轮对话)...

MemoryCompressor 压缩后:
  [摘要] 用户询问了 Python 基础知识，包括语法、装饰器、类型提示。
  Agent 对每个主题做了解释并提供代码示例。

新对话上下文:
  {chat_history} = [摘要] + 最近几轮完整对话
```

**配置：**

```yaml
memory:
  name: 'demo_memory'
  max_tokens: 2000
  memory_compressor: 'demo_memory_compressor'   # ★ 启用压缩

# 对应的 memory_compressor YAML:
name: 'demo_memory_compressor'
compressor_prompt_version: 'memory_compressor_prompt'
compressor_llm_name: 'qwen_llm'
metadata:
  type: 'MEMORY_COMPRESSOR'
  module: '...'
  class: 'MemoryCompressor'
```

---

## 6. 动手练习

### 练习 1：配置 Memory 并观察效果

1. 修改 simple_qa_agent，取消 memory 配置的注释
2. 运行 Agent，问两个相关问题（如"我喜欢喝咖啡"→"我喜欢喝什么？"）
3. 观察 Agent 是否记住了第一轮的对话

### 练习 2：调整 max_tokens 观察裁剪效果

将 `max_tokens` 设为很小的值（如 50），进行多轮对话。观察什么时候开始"遗忘"早期对话。

### 练习 3：切换存储后端

将 `ram_memory_storage` 改为 `chroma_memory_storage`，重启 Agent。观察不同之处（ChromaDB 在本地创建持久化文件）。

### 练习 4：阅读 prune() 源码

打开 `agentuniverse/agent/memory/memory.py`，逐行阅读 `prune()` 方法。理解：
1. 如何计算 token 数
2. 裁剪策略（从最早的消息开始 pop）
3. 压缩摘要的触发条件

---

## 下一步

你已经理解了 Memory 系统——对话历史的存储、裁剪和压缩。

下一步进入 **08-rag-knowledge.md**，学习 RAG（检索增强生成）——如何将私有文档的知识注入 Agent。
