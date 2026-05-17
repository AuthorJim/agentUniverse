# 08 — RAG：领域知识注入

> **前置要求：** 完成 [07-memory-system.md](07-memory-system.md)
> **学习目标：** 理解 RAG 的完整流水线，掌握 Reader / DocProcessor / Embedding / Store 各组件的协同工作
> **预计时间：** 2-3 天

---

## 1. 为什么需要 RAG？

LLM 的知识有两个致命局限：

**局限 1：时效性——"你的训练数据截止到..."**

```
用户: "2026年新颁布的XX法规有什么影响？"
LLM: "抱歉，我无法提供2026年的信息。"  ← 训练数据是旧的
```

**局限 2：私有性——"你怎么不知道我们的产品？"**

```
用户: "根据我们公司内部文档，这个 API 的限流策略是什么？"
LLM: "我没有关于你公司内部文档的信息。"  ← 不知道你的私有数据
```

**RAG（Retrieval-Augmented Generation）解决的就是这两个问题：** 在 LLM 推理前，先从知识库中检索相关文档，把检索结果"塞进" Prompt，让 LLM 基于这些外部知识来回答。

```
无 RAG:  用户问题 → LLM → 输出
有 RAG:  用户问题 → [检索知识库] → LLM + 检索结果 → 输出
```

**类比全栈架构：** RAG 管线的离线阶段类似于 ETL（Extract-Transform-Load），在线阶段类似于搜索引擎的 query → retrieve → rank 流程。

---

## 2. agentUniverse 的 RAG 流水线架构

```
┌────────────────────────────────────────────────┐
│              ① 离线阶段：构建知识库                 │
│                                                  │
│  PDF/TXT/JSON → Reader(读取)                    │
│       → DocProcessor(切分/清洗)                  │
│       → Embedding(向量化)                        │
│       → Store(存储，如 ChromaDB, FAISS)          │
│                                                  │
└────────────────────────────────────────────────┘
                      ↓
┌────────────────────────────────────────────────┐
│              ② 在线阶段：查询时                    │
│                                                  │
│  用户问题 → QueryParaphraser(改写)              │
│       → RagRouter(路由到哪个Store)              │
│       → Store.query(向量检索)                   │
│       → DocProcessor(后处理，如重排序)           │
│       → 注入 LLM Prompt (background 变量)        │
│                                                  │
└────────────────────────────────────────────────┘
```

### 2.1 各组件的职责

| 组件 | 阶段 | 职责 | 本项目位置 |
|------|------|------|-----------|
| **Reader** | 离线 | 读取文件（PDF/TXT/JSON/URL）→ Document | `knowledge/reader/` |
| **DocProcessor** | 离线 | 文档切分、清洗、关键词提取 | `knowledge/doc_processor/` |
| **Embedding** | 离线 | Document → 向量（浮点数数组） | `knowledge/embedding/` |
| **Store** | 离线+在线 | 向量存储 + 相似度检索 | `knowledge/store/` |
| **QueryParaphraser** | 在线 | 改写用户查询（补全、拆分、关键词提取） | `knowledge/query_paraphraser/` |
| **RagRouter** | 在线 | 决定去哪个 Store 检索 | `knowledge/rag_router/` |
| **Knowledge** | 编排 | 串联上述所有组件 | `knowledge/knowledge.py` |

### 2.2 Knowledge 基类：RAG 的指挥中心

```python
# agentuniverse/agent/action/knowledge/knowledge.py:36-84
class Knowledge(ComponentBase):
    name: str = ""
    stores: List[str] = []               # 要写入/查询哪些 Store
    query_paraphrasers: List[str] = []    # 查询改写器
    insert_processors: List[str] = []     # 写入时的 DocProcessor
    post_processors: List[str] = []       # 检索后的 DocProcessor
    readers: Dict[str, str] = {}          # 文件类型 → Reader 映射
    rag_router: str = "base_router"       # 路由策略

    def insert_knowledge(self, **kwargs):
        """插入知识：Reader 读取 → DocProcessor 处理 → Store 存储"""
        document_list = self._load_data(**kwargs)        # Reader 读取
        document_list = self._insert_process(document_list)  # DocProcessor 处理
        for store_code in self.stores:                   # 存入所有 Store
            StoreManager().get_instance_obj(store_code).insert_document(document_list)

    def query_knowledge(self, query_str, **kwargs):
        """查询知识：QueryParaphraser 改写 → RagRouter 路由 → Store 检索"""
        query = self._paraphrase_query(Query(origin_query=query_str))
        store_names = self._route_stores(query)          # RagRouter 决定去哪个 Store
        docs = []
        for store_name in store_names:
            docs.extend(StoreManager().get_instance_obj(store_name).query(query))
        docs = self._rag_post_process(docs, query)       # 后处理（重排序等）
        return docs
```

---

## 3. 从 rag_app 看完整流程

以 `examples/sample_apps/rag_app` 为例，它加载了《刑法》和《民法典》PDF，构建了一个法律知识库 Agent。

### 3.1 离线阶段配置

```yaml
# intelligence/agentic/knowledge/law_knowledge.yaml
name: 'law_knowledge'
description: '中国法律知识库'
stores: ['criminal_law_chroma_store', 'civil_law_chroma_store']  # 两个 Store
insert_processors: ['query_keyword_extractor']                    # 提取关键词
readers:
  pdf: 'default_pdf_reader'        # ★ PDF → Reader 映射
post_processors: []
rag_router: 'nlu_rag_router'       # NLU 路由：根据问题分类选择 Store
query_paraphrasers: ['custom_query_keyword_extractor']
```

### 3.2 Store 配置示例

```yaml
# intelligence/agentic/knowledge/store/criminal_law_chroma_store.yaml
name: 'criminal_law_chroma_store'
description: '刑法知识库 - ChromaDB 存储'
collection_name: 'criminal_law'
embedding_model: 'default_openai_embedding'
persist_directory: './db/criminal_law.db'
metadata:
  type: 'STORE'
  module: 'agentuniverse.agent.action.knowledge.store.chroma_store'
  class: 'ChromaStore'
```

### 3.3 Agent 中使用 Knowledge

```yaml
# Agent YAML
action:
  knowledge: ['law_knowledge']    # ★ 绑定知识库
```

当 Agent 的 `action.knowledge` 包含 `law_knowledge` 时，ReActPlanner 会将知识库转换为一个可调用的 Tool：

```python
# react_planner.py:98-99
for knowledge_name in knowledge:
    knowledge_tool = KnowledgeManager().get_instance_obj(knowledge_name)
    lc_tools.append(knowledge_tool.as_langchain_tool())
```

这意味着 **Knowledge 在 ReAct 视角下就是一个 Tool**——LLM 通过 ReAct 循环自主决定什么时候查询知识库。

---

## 4. 向量检索原理（简明版）

### 4.1 什么是 Embedding？

Embedding 把一个文本转换为一个固定长度的浮点数数组（向量）：

```
"北京是中国的首都"
  → Embedding 模型
  → [0.023, -0.451, 0.782, ..., 0.134]    # 1536 维向量
```

**语义相近的文本，向量距离近：**

```
"北京是中国的首都" → 向量 A
"中国的首都是北京" → 向量 A'（和 A 距离很近）
"今天天气不错"     → 向量 B（和 A 距离很远）
```

### 4.2 检索过程

```
用户问题: "什么是故意杀人罪？"
  → QueryParaphraser 改写: "故意杀人罪 定义 构成要件"
  → Embedding: [0.1, -0.3, 0.8, ...]
  → Store.query(向量, top_k=5):
      在向量空间中找最相似的 5 个文档块
  → 返回: [Document("刑法第232条..."), Document("故意杀人..."), ...]
  → 注入 Prompt: {background}
```

---

## 5. 动手练习

### 练习 1：运行 rag_app

```bash
cd examples/sample_apps/rag_app
# 配置 API Key 后启动
python bootstrap/intelligence/server_application.py
```

向 Agent 提问法律相关问题（如"什么是正当防卫？"），观察它如何引用民法典和刑法。

### 练习 2：用自己的文档替换

1. 准备一份你自己的 PDF/TXT 文档
2. 修改 `law_knowledge.yaml`，指向你的文档
3. 修改 Store 的 `collection_name` 和 `persist_directory`
4. 运行 `insert_knowledge()` 导入文档
5. 测试查询

### 练习 3：对比不同 Store

将 `chroma_store` 替换为 `faiss_flat_store` 或 `faiss_ivf_store`，对比：
- 插入速度
- 查询速度
- 检索准确率

### 练习 4：追踪 query_knowledge 源码

打开 `agentuniverse/agent/action/knowledge/knowledge.py`，逐行阅读 `query_knowledge()` 方法。画出从用户问题到检索结果的完整调用链。

---

## 6. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **RAG** | Retrieval-Augmented Generation | 本章主题 |
| **Knowledge** | RAG 流水线的编排器 | `knowledge/knowledge.py` |
| **Reader** | 文件读取器（PDF/TXT/JSON/URL） | `knowledge/reader/` |
| **DocProcessor** | 文档处理（切分/清洗/提取） | `knowledge/doc_processor/` |
| **Embedding** | 文本 → 向量 | `knowledge/embedding/` |
| **Store** | 向量存储 + 检索 | `knowledge/store/` |
| **QueryParaphraser** | 查询改写 | `knowledge/query_paraphraser/` |
| **RagRouter** | 查询路由 | `knowledge/rag_router/` |
| **Document** | 文档对象（content + metadata） | `knowledge/store/document.py` |
| **Query** | 查询对象 | `knowledge/store/query.py` |

---

## 下一步

你已经理解了 RAG 的完整流水线——从文档读取到向量检索到注入 Agent。

下一步进入 **09-grr-pattern.md**，开始多 Agent 协作模式的学习，从最简单的 GRR（Generate-Review-Rewrite）开始。
