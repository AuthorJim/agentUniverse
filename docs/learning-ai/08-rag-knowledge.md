# 08 — RAG：让 Agent 读懂你的私有数据

> **前置要求：** 完成 [07-memory-system.md](07-memory-system.md)
> **学习目标：** 理解 RAG 的完整流水线（Reader → DocProcessor → Embedding → Store → 检索），掌握配置和调试技巧
> **预计时间：** 2-3 天

---

## 1. 为什么 LLM 需要"翻书"？

LLM 的知识有两个致命局限：

**局限 1：时效性——模型是过去训练的**

```
用户: "2026 年新颁布的《人工智能法》有什么主要条款？"
LLM:  "抱歉，我的训练数据截止于 2024 年..."    ← 不知道新法
```

**局限 2：私有性——模型不知道你公司的秘密**

```
用户: "根据我们公司的内部 Wiki，这个 API 的限流策略是什么？"
LLM:  "我没有关于你公司内部文档的信息。"        ← 不知道你的私有数据
```

**RAG（Retrieval-Augmented Generation，检索增强生成）解决的就是这两个问题：**

```
无 RAG:  用户问题 → LLM → 输出（只能靠记忆）
有 RAG:  用户问题 → [检索知识库] → LLM + 检出的相关文档 → 输出（有凭有据）
```

**类比你的后端经验：** RAG 就像在你调用微服务之前先查一下 Elasticsearch——找到相关文档后，把文档内容一起传给下游服务做决策。

---

## 2. agentUniverse 的 RAG 流水线架构

### 2.1 两阶段流水线

```
┌─────────────────────────────────────────────────────┐
│            ① 离线阶段：构建知识库（ETL）              │
│                                                     │
│  PDF/TXT/JSON → Reader(读取文件)                    │
│       → DocProcessor(切分/清洗/提取关键词)           │
│       → Embedding(文本 → 向量，即浮点数数组)         │
│       → Store(向量存储: ChromaDB / FAISS / Milvus)  │
│                                                     │
└─────────────────────────────────────────────────────┘
                        ↓
┌─────────────────────────────────────────────────────┐
│            ② 在线阶段：查询时（Query → Retrieve）     │
│                                                     │
│  用户问题 → QueryParaphraser(改写补全查询)          │
│       → RagRouter(路由到哪个 Store)                 │
│       → Store.query(向量相似度检索 top_k 结果)      │
│       → DocProcessor(后处理: 重排序/去重)           │
│       → 注入 LLM Prompt → 生成回答                 │
│                                                     │
└─────────────────────────────────────────────────────┘
```

### 2.2 七大组件的职责

| 组件 | 阶段 | 职责 | 代码位置 |
|------|------|------|---------|
| **Reader** | 离线 | 读取文件（PDF/TXT/JSON/URL）→ 统一 Document 对象 | `knowledge/reader/` |
| **DocProcessor** | 离线 | 文档切分/清洗/关键词提取 | `knowledge/doc_processor/` |
| **Embedding** | 离线 | 文本 → 向量（浮点数数组） | `knowledge/embedding/` |
| **Store** | 离线+在线 | 向量存储 + 相似度检索 | `knowledge/store/` |
| **QueryParaphraser** | 在线 | 改写用户查询（补全/拆分/关键词提取） | `knowledge/query_paraphraser/` |
| **RagRouter** | 在线 | 决定去哪个 Store 检索 | `knowledge/rag_router/` |
| **Knowledge** | 编排 | 串联上述所有组件 | `knowledge/knowledge.py` |

### 2.3 Knowledge 基类：RAG 总指挥

```python
# agentuniverse/agent/action/knowledge/knowledge.py:36-84
class Knowledge(ComponentBase):
    name: str = ""
    stores: List[str] = []                # 写入/查询哪些 Store
    query_paraphrasers: List[str] = []    # 查询改写器列表
    insert_processors: List[str] = []     # 写入时的 DocProcessor
    post_processors: List[str] = []       # 检索后的 DocProcessor
    readers: Dict[str, str] = {}          # 文件扩展名 → Reader 映射
    rag_router: str = "base_router"       # 路由策略

    def insert_knowledge(self, **kwargs):
        """★ 插入知识：Reader → DocProcessor → Store"""
        # ① Reader 读取原始文件
        document_list = self._load_data(**kwargs)

        # ② DocProcessor 处理（切分、清洗、关键词提取）
        document_list = self._insert_process(document_list)

        # ③ 存入所有配置的 Store
        for store_code in self.stores:
            StoreManager().get_instance_obj(store_code).insert_document(document_list)

    def query_knowledge(self, query_str, **kwargs):
        """★ 查询知识：Paraphraser → Router → Store → PostProcess"""
        # ① QueryParaphraser 改写查询
        query = self._paraphrase_query(Query(origin_query=query_str))

        # ② RagRouter 决定去哪些 Store
        store_names = self._route_stores(query)

        # ③ 并行查询所有目标 Store
        docs = []
        for store_name in store_names:
            docs.extend(StoreManager().get_instance_obj(store_name).query(query))

        # ④ 后处理（去重、重排序）
        docs = self._rag_post_process(docs, query)
        return docs
```

---

## 3. 实战：rag_app 的完整拆解

以 `examples/sample_apps/rag_app` 为例——它加载了《刑法》和《民法典》PDF，构建了一个法律知识库。

### 3.1 Knowledge YAML 配置

```yaml
# intelligence/agentic/knowledge/law_knowledge.yaml
name: 'law_knowledge'
description: '中国法律知识库（刑法 + 民法典）'
stores: ['criminal_law_chroma_store', 'civil_law_chroma_store']  # 两个 Store
insert_processors: ['query_keyword_extractor']                    # 写入时提取关键词
readers:
  pdf: 'default_pdf_reader'        # PDF 文件使用这个 Reader
post_processors: []
rag_router: 'nlu_rag_router'       # NLU 路由: 根据问题意图选择 Store
query_paraphrasers: ['custom_query_keyword_extractor']
metadata:
  type: 'KNOWLEDGE'
  module: 'agentuniverse.agent.action.knowledge.knowledge'
  class: 'Knowledge'
```

### 3.2 Store YAML 配置

```yaml
# intelligence/agentic/knowledge/store/criminal_law_chroma_store.yaml
name: 'criminal_law_chroma_store'
description: '刑法知识库 - ChromaDB'
collection_name: 'criminal_law'             # ChromaDB 的 collection 名
embedding_model: 'default_openai_embedding' # 用哪个 Embedding 模型
persist_directory: './db/criminal_law.db'   # 持久化目录
metadata:
  type: 'STORE'
  module: 'agentuniverse.agent.action.knowledge.store.chroma_store'
  class: 'ChromaStore'
```

### 3.3 Agent 中绑定 Knowledge

```yaml
# Agent YAML
action:
  knowledge: ['law_knowledge']    # ★ 一行绑定
```

绑定后，ReActPlanner 会把 Knowledge 转换成一个可调用的 Tool。这意味着 **LLM 在 ReAct 循环中可以自主决定什么时候去查知识库。**

---

## 4. 向量检索到底是怎么工作的？

### 4.1 Embedding：把文字变成数字

```
"北京是中国的首都"
  → Embedding 模型(如 text-embedding-3-small)
  → [0.023, -0.451, 0.782, ..., 0.134]    # 1536 维浮点数数组

"中国的首都是北京"
  → Embedding 模型
  → [0.025, -0.449, 0.780, ..., 0.132]    # 和上面非常接近！

"今天天气不错"
  → Embedding 模型
  → [-0.812, 0.334, -0.201, ..., 0.567]   # 和上面完全不同！
```

**语义相近的文本 → 向量在空间中距离近。** 这不是关键词匹配，而是语义匹配。

### 4.2 检索过程

```
用户问题: "什么是故意杀人罪？"
  ↓
① QueryParaphraser 改写:
  "故意杀人罪 定义 构成要件 刑法"
  ↓
② Embedding 向量化:
  [0.12, -0.34, 0.87, ...]
  ↓
③ Store.query(向量, top_k=5):
  在向量空间中找余弦距离最近的 5 个文档块
  ↓
④ 返回结果:
  [
    Document("刑法第232条: 故意杀人的，处死刑..."),
    Document("故意杀人罪是指故意非法剥夺他人生命..."),
    ...
  ]
  ↓
⑤ 注入 Prompt → LLM 基于这些内容生成回答
```

**类比 Elasticsearch：** 向量检索 ≈ 语义搜索；BM25 全文检索 ≈ 关键词搜索。向量检索能找到"意思相近但用词不同"的内容。

---

## 5. RagRouter：智能路由

当你有多个 Store（如刑法 Store 和民法典 Store），RagRouter 决定去哪个查：

```yaml
rag_router: 'nlu_rag_router'    # 基于 NLU 意图分类的路由
```

```
用户问: "盗窃罪的量刑标准是什么？"
  → NLU Router 判断: 意图 = 刑事法律
  → 路由到: criminal_law_chroma_store
  → 不查 civil_law_chroma_store（节省检索时间）

用户问: "合同违约的赔偿标准？"
  → NLU Router 判断: 意图 = 民事法律
  → 路由到: civil_law_chroma_store
```

---

## 6. 动手练习

### 练习 1：运行 rag_app

```bash
cd examples/sample_apps/rag_app
# 配置 API Key（需要 Embedding 模型和 LLM 的 Key）
python bootstrap/intelligence/server_application.py
```

向 Agent 提问法律相关问题，观察它如何引用法典原文。

### 练习 2：用自己的文档替换

1. 准备一份你自己的 PDF 或 TXT 文档
2. 新建一个 Knowledge YAML，配置你的文档路径
3. 新建一个 Store YAML，设置 `collection_name` 和 `persist_directory`
4. 调用 `knowledge.insert_knowledge()` 导入文档
5. 通过 Agent 提问测试检索效果

### 练习 3：对比不同的 DocProcessor

对比以下 DocProcessor 的效果差异：
- `doc_splitter`（按长度切分）
- `query_keyword_extractor`（提取关键词）
- 两个同时使用

### 练习 4：调整 top_k 和 chunk_size

修改 Store 配置中的 `top_k`（返回文档数）和 DocProcessor 的 `chunk_size`（块大小）：
1. top_k=3 vs top_k=10 对回答质量的影响
2. chunk_size=500 vs chunk_size=2000 对检索精度的影响

### 练习 5：追踪 query_knowledge 源码

打开 `agentuniverse/agent/action/knowledge/knowledge.py`，逐行阅读 `query_knowledge()` 方法，画出完整的调用链。

---

## 7. 概念速查表

| 概念 | 含义 | 关键位置 |
|------|------|---------|
| **RAG** | Retrieval-Augmented Generation | 本章主题 |
| **Knowledge** | RAG 流水线编排器 | `knowledge/knowledge.py` |
| **Reader** | 文件读取器（PDF/TXT/JSON/URL/Excel） | `knowledge/reader/` |
| **DocProcessor** | 文档处理（切分/清洗/提取关键词） | `knowledge/doc_processor/` |
| **Embedding** | 文本 → 向量 | `knowledge/embedding/` |
| **Store** | 向量存储 + 检索（ChromaDB/FAISS/Milvus） | `knowledge/store/` |
| **QueryParaphraser** | 查询改写（补全/拆分） | `knowledge/query_paraphraser/` |
| **RagRouter** | 查询路由（决定去哪个 Store） | `knowledge/rag_router/` |
| **Document** | 文档对象（content + metadata） | `knowledge/store/document.py` |
| **Query** | 查询对象 | `knowledge/store/query.py` |
| **ChromaStore** | ChromaDB 实现（轻量，适合入门） | `knowledge/store/chroma_store.py` |

---

## 下一步

你已经理解了 RAG 的完整流水线。下一步进入 **[09-grr-pattern.md](09-grr-pattern.md)**——GRR 多 Agent 协作模式，3 个 Agent 组成一个带质量反馈的创作流水线。
