# 11 — PEER 模式：四 Agent 的梦幻团队

> **前置要求：** 完成 [10-is-pattern.md](10-is-pattern.md)
> **学习目标：** 深入理解 PEER（Plan → Execute → Express → Review）四角色协作模式——agentUniverse 最强大的多 Agent 范式
> **预计时间：** 2-3 天

---

## 1. PEER 是什么？—— 一支微型工程团队

**PEER = Plan（规划）→ Execute（执行）→ Express（表达）→ Review（评审）**

如果你把四个 Agent 比作一个工程团队，这就是：

```
PlanningAgent    → 产品经理：拆需求、定方案、分任务
ExecutingAgent   → 开发工程师：查资料、调工具、执行
ExpressingAgent  → 技术写作：整理输出、结构化、润色
ReviewingAgent   → QA 测试：评分、挑错、决定 pass/fail
```

**这是 agentUniverse 框架最具标志性的协作模式**，对应的学术论文已被 arXiv 收录：[PEER: Expertizing Domain-Specific Tasks with a Multi-Agent Framework](https://arxiv.org/abs/2407.06985)。

---

## 2. PEER 为什么更强大？

| 对比维度 | 单 Agent (ReAct) | GRR (3 Agent) | IS (2 Agent) | PEER (4 Agent) |
|---------|-------------------|---------------|-------------|----------------|
| 角色分工 | 无 | 生成/评审/改写 | 执行/监督 | 规划/执行/表达/评审 |
| 推理深度 | 单层 | 2 层（评审+改写） | 多层 checkpoint | **4 层串行专业推理** |
| 适用范围 | 简单问答 | 内容创作 | 目标对齐 | **复杂分析决策** |
| 质量保障 | 无 | 评分重写 | 逐步检查 | 评审+迭代+jump_step |

---

## 3. PEER 完整执行流程

```
用户: "分析 2026 年新能源汽车行业趋势"

┌──────────────────────────────────────────────────────────┐
│ ① PlanningAgent（规划 — 产品经理）                        │
│    Prompt: 你是信息分析专家，拆解用户问题                  │
│    输出:                                                  │
│    {                                                      │
│      "thought": "从政策、市场、技术、竞争四维度分析...",    │
│      "framework": [                                       │
│        "2026年新能源汽车政策环境如何？",                   │
│        "2026年新能源汽车市场规模和增长趋势？",             │
│        "2026年新能源汽车关键技术进展？",                   │
│        "前5大厂商的竞争格局如何？"                         │
│      ]                                                    │
│    }                                                      │
└──────────────────────┬───────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ ② ExecutingAgent（执行 — 开发工程师）                     │
│    Prompt: 对每个子问题检索信息并整合                      │
│    Action: 逐个搜索 4 个子问题                             │
│    输出: 每个子问题的搜索结果 + 初步分析                   │
│    (可用 google_search、rag_knowledge 等工具)              │
└──────────────────────┬───────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ ③ ExpressingAgent（表达 — 技术写作）                      │
│    Prompt: 将执行结果整理为结构化报告                      │
│    输出:                                                  │
│    - 总结陈述（200 字概括）                                │
│    - 详细分析（四个维度逐一展开，数据支撑）                 │
│    - 结论与展望                                           │
└──────────────────────┬───────────────────────────────────┘
                       ↓
┌──────────────────────────────────────────────────────────┐
│ ④ ReviewingAgent（评审 — QA 测试）                        │
│    从 5 个维度评分:                                       │
│    - 完整性、准确性、逻辑性、结构性、实用性                │
│    评分: 78/100                                            │
│    判断: 78 > eval_threshold(70) → 通过！                 │
│    (如果 < 70，根据 jump_step 回到指定步骤重来)            │
└──────────────────────────────────────────────────────────┘

最终输出：一份结构化的行业分析报告
```

---

## 4. 源码深度解读

### 4.1 PeerWorkPattern 类

```python
# agentuniverse/agent/work_pattern/peer_work_pattern.py:17-21
class PeerWorkPattern(WorkPattern):
    planning: PlanningAgentTemplate = None        # 规划
    executing: ExecutingAgentTemplate = None      # 执行
    expressing: ExpressingAgentTemplate = None    # 表达
    reviewing: ReviewingAgentTemplate = None      # 评审
```

### 4.2 invoke() 核心循环

```python
# peer_work_pattern.py:23-57 (简化)
def invoke(self, input_object, work_pattern_input, **kwargs):
    retry_count = work_pattern_input.get('retry_count', 3)
    jump_step = work_pattern_input.get('jump_step', 'planning')
    eval_threshold = work_pattern_input.get('eval_threshold', 60)

    planning_result = {}
    executing_result = {}
    expressing_result = {}
    reviewing_result = {}

    for _ in range(retry_count):
        # ★ 核心逻辑：只在需要时才执行（缓存结果 + jump_step 控制）
        if not planning_result or jump_step == "planning":
            planning_result = self._invoke_planning(...)

        if not executing_result or jump_step in ["planning", "executing"]:
            executing_result = self._invoke_executing(...)

        if not expressing_result or jump_step in ["planning", "executing", "expressing"]:
            expressing_result = self._invoke_expressing(...)

        if not reviewing_result or jump_step in ["planning", "executing", "expressing", "reviewing"]:
            reviewing_result = self._invoke_reviewing(...)

        # 评分达标 → 退出
        if (reviewing_result.get('score')
                and reviewing_result.get('score') >= eval_threshold):
            break

    return {'result': {
        'planning': planning_result,
        'executing': executing_result,
        'expressing': expressing_result,
        'reviewing': reviewing_result,
    }}
```

### 4.3 jump_step 机制：精准重试

这是 PEER 最聪明的设计之一——不是每次都全量重来，而是从指定的步骤开始：

```
jump_step = "planning"    → 全部重来（最彻底，质量最高）
jump_step = "executing"   → Planning 结果复用，重新执行+表达（省 token）
jump_step = "expressing"  → 只重做表达（Planning 和 Executing 结果复用）
jump_step = "reviewing"   → 重新评审（一般没必要）
```

**最佳实践：**
- 追求质量：`jump_step = "planning"`（每次迭代全面重新推理）
- 省成本：`jump_step = "executing"`（复用规划，只重新执行）
- 微调输出：`jump_step = "expressing"`（只换说法不改逻辑）

### 4.4 Agent 间的数据传递

PEER 通过 `input_object.add_data()` 在 Agent 间传递信息：

```python
# Planning → Executing
input_object.add_data('planning_result', {'framework': ['子问题1', '子问题2', ...]})

# Executing → Expressing
input_object.add_data('executing_result', {'answers': {'子问题1': '...', ...}})

# Expressing → Reviewing
input_object.add_data('expressing_result', {'output': '结构化报告...'})

# Reviewing → 下一轮 Planning
input_object.add_data('reviewing_result', {'score': 78, 'suggestion': '...'})
```

**这些就是多 Agent 之间的"消息队列"——** 通过 `input_object` 这个共享数据容器传递各阶段的结果。

---

## 5. PEER 的参数速查

| 参数 | 含义 | 典型值 |
|------|------|--------|
| `retry_count` | 最大迭代轮数 | 3-5 |
| `jump_step` | 重试起点 | planning（最彻底）/ executing（折中） |
| `eval_threshold` | 评审通过分数线 | 60-80（视任务难度） |

---

## 6. 四角色 Prompt 设计速览

| Agent | 核心 Prompt 要点 |
|-------|----------------|
| **Planning** | 拆解为 2-4 个子问题；子问题必须完整且可独立回答；输出 JSON：`{"thought": "...", "framework": [...]}` |
| **Executing** | 对每个子问题检索信息；去除重复/错误/无用信息；尽量使用数值数据；注意时效性 |
| **Expressing** | 总-分结构：先总结再展开；用数据作论据；语义连贯无重复；结构化输出 |
| **Reviewing** | 多维度评分（完整性/准确性/逻辑性/结构性/实用性）；0-100 分制；输出 JSON：`{"score": int, "suggestion": "..."}` |

---

## 7. 动手练习

### 练习 1：运行 peer_agent_app

```bash
cd examples/sample_apps/peer_agent_app
# 配置 API Key
python bootstrap/intelligence/server_application.py
```

用一个复杂的分析问题测试（如"分析 2026 年 AI 行业趋势"），观察四个 Agent 各自的输出。

### 练习 2：修改 jump_step 观察变化

```bash
# 在 peer_agent YAML 或启动代码中
jump_step = "executing"

# 对比 jump_step="planning" 时，retry 的起点是否不同
```

### 练习 3：调低 eval_threshold

将 `eval_threshold` 从 60 调低到 30，观察 Agent 是否更早退出迭代。

### 练习 4：阅读 PeerWorkPattern 源码

打开 `agentuniverse/agent/work_pattern/peer_work_pattern.py`，画出四个 Agent 之间的数据流图。

### 练习 5：设计 PEER 应用

设计一个使用 PEER 模式的代码审查系统：
- Planning Agent：分析代码结构，确定审查重点
- Executing Agent：逐模块检查代码问题
- Expressing Agent：整理为审查报告
- Reviewing Agent：评估审查质量

---

## 8. 多 Agent 模式总结对比

| 模式 | Agent 数 | 流程 | 反馈机制 | 最佳场景 |
|------|---------|------|---------|---------|
| **ReAct** | 1 | Thought→Action→Observation 循环 | 无 | 工具调用、简单问答 |
| **GRR** | 3 | Generate→Review→Rewrite | 评分 + 全文重写 | 内容创作、翻译 |
| **IS** | 2 | Implementation⇄Supervision | Checkpoint + 局部修正 | 目标对齐任务 |
| **PEER** | 4 | Plan→Execute→Express→Review | 评分 + jump_step 重试 | 复杂分析报告 |

---

## 下一步

你已经掌握了 agentUniverse 最强大的 PEER 模式。最后一章 **[12-building-your-own.md](12-building-your-own.md)** 综合运用所有知识，从零构建属于你自己的 Agent 应用。
