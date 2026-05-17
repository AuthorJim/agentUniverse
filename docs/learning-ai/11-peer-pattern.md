# 11 — PEER 模式：Plan → Execute → Express → Review

> **前置要求：** 完成 [10-is-pattern.md](10-is-pattern.md)
> **学习目标：** 深入理解 PEER 四角色协作模式，这是 agentUniverse 最具特色的多 Agent 协作范式
> **预计时间：** 2-3 天

---

## 1. PEER 是什么？

**PEER = Plan（规划） → Execute（执行） → Express（表达） → Review（评审）**

四个 Agent 各司其职，类似于一支微型工程团队：

```
PlanningAgent     → 产品经理：拆需求、定方案
ExecutingAgent    → 开发工程师：查资料、执行
ExpressingAgent   → 技术写作：整理输出、结构化
ReviewingAgent    → QA 测试：评分、反馈、兜底
```

**这是 agentUniverse 框架的标志性协作模式**，对应的学术论文已被 arXiv 收录。

### 1.1 为什么 PEER 比其他模式更强大？

| 对比维度 | 单 Agent（ReAct） | GRR | IS | PEER |
|---------|-------------------|-----|-----|------|
| 角色分工 | 无 | 3 人 | 2 人 | 4 人 |
| 推理深度 | 单层 | 2 层（评审+改写） | 多层 checkpoint | 4 层串行 |
| 适用场景 | 简单问答 | 内容生成 | 目标对齐 | 复杂推理分析 |
| 质量保障 | 无 | 评分+重写 | 逐步检查 | 评审+迭代 |

---

## 2. PEER 完整流程

```
用户输入: "分析2026年新能源汽车行业趋势"

┌──────────────────────────────────────────────────────┐
│ ① PlanningAgent（规划）                               │
│    目标：拆解用户问题为2-4个子问题                      │
│    Thought: 需要从政策、市场、技术三个维度分析...        │
│    Framework: [                                       │
│      "2026年新能源汽车的政策环境如何？",                │
│      "2026年新能源汽车市场规模和增长趋势？",            │
│      "2026年新能源汽车关键技术进展？"                  │
│    ]                                                  │
└────────────────────┬─────────────────────────────────┘
                     ↓
┌──────────────────────────────────────────────────────┐
│ ② ExecutingAgent（执行）                              │
│    目标：对每个子问题进行信息检索和整合                  │
│    使用 google_search 工具逐个搜索子问题...            │
│    整合搜索结果，生成初步的 Q&A 信息                    │
└────────────────────┬─────────────────────────────────┘
                     ↓
┌──────────────────────────────────────────────────────┐
│ ③ ExpressingAgent（表达）                             │
│    目标：将执行结果组织为结构化报告                     │
│    生成两大段：总结陈述 + 详细阐述                      │
│    使用数据支撑论点，遵循总-分结构                      │
└────────────────────┬─────────────────────────────────┘
                     ↓
┌──────────────────────────────────────────────────────┐
│ ④ ReviewingAgent（评审）                              │
│    目标：评分并决定是否通过                             │
│    评分维度：完整性、准确性、逻辑性、结构性...           │
│    判断：score >= eval_threshold?                      │
│    ├── YES → 输出最终结果                              │
│    └── NO  → 回到 Planning（或指定的 jump_step）       │
│              → 重新规划和执行改进                       │
└──────────────────────────────────────────────────────┘
```

---

## 3. 源码详解

### 3.1 PeerWorkPattern 类

```python
# agentuniverse/agent/work_pattern/peer_work_pattern.py:17-21
class PeerWorkPattern(WorkPattern):
    planning: PlanningAgentTemplate = None       # 规划 Agent
    executing: ExecutingAgentTemplate = None     # 执行 Agent
    expressing: ExpressingAgentTemplate = None   # 表达 Agent
    reviewing: ReviewingAgentTemplate = None     # 评审 Agent
```

### 3.2 invoke() 核心循环

```python
# peer_work_pattern.py:23-57
def invoke(self, input_object, work_pattern_input, **kwargs):
    retry_count = work_pattern_input.get('retry_count')       # 最多迭代次数
    jump_step = work_pattern_input.get('jump_step')           # 从哪步开始重来
    eval_threshold = work_pattern_input.get('eval_threshold')  # 评分阈值

    planning_result = {}    # 缓存规划结果
    executing_result = {}   # 缓存执行结果
    expressing_result = {}  # 缓存表达结果
    reviewing_result = {}   # 缓存评审结果

    for _ in range(retry_count):
        peer_round_results = {}

        # ★ 如果之前没有规划结果，或者 jump_step 指定了从 planning 开始
        if not planning_result or jump_step == "planning":
            planning_result = self._invoke_planning(...)

        # ★ 同理，跳过已有结果的步骤
        if not executing_result or jump_step in ["planning", "executing"]:
            executing_result = self._invoke_executing(...)

        if not expressing_result or jump_step in ["planning", "executing", "expressing"]:
            expressing_result = self._invoke_expressing(...)

        if not reviewing_result or jump_step in ["planning", "executing", "expressing", "reviewing"]:
            reviewing_result = self._invoke_reviewing(...)

        peer_results.append(peer_round_results)

        # ★ 评分达标就退出
        if (reviewing_result.get('score')
                and reviewing_result.get('score') >= eval_threshold):
            break

    return {'result': peer_results}
```

### 3.3 jump_step 机制：精准重试

`jump_step` 让你指定从哪一步开始重试：

```
jump_step = "planning"    → 全部重来（最彻底）
jump_step = "executing"   → 跳过规划，重新执行和表达
jump_step = "expressing"  → 只重做表达（换个说法）
jump_step = "reviewing"   → 重新评审（通常没必要）
```

**最佳实践：** 设置 `jump_step = "planning"` 让每次迭代都全面重新推理，质量最高。设置 `jump_step = "executing"` 可以节省 token 但可能复用有问题的规划。

### 3.4 四个 Agent 的数据传递

PEER 通过 `input_object.add_data()` 在 Agent 间传递数据：

```python
# Planning 输出 → 后续可用
input_object.add_data('planning_result', planning_output)
#   包含: framework (子问题列表)

# Executing 输出 → Expressing 可用
input_object.add_data('executing_result', executing_output)
#   包含: 各子问题的搜索结果和初步结论

# Expressing 输出 → Reviewing 可用
input_object.add_data('expressing_result', expressing_output)
#   包含: 结构化的最终回答

# Reviewing 输出 → 下一轮 Planning 可用
input_object.add_data('reviewing_result', reviewing_output)
#   包含: score, 评审意见, 改进建议
```

---

## 4. 四个角色的 Prompt 设计要点

### 4.1 Planning Agent

```yaml
introduction: 你是一位精通信息分析的 AI 助手
target: 拆解用户问题，生成 2-4 个子问题
instruction: |
  1. 子问题必须逻辑递进
  2. 每个子问题完整可独立搜索
  3. 输出 JSON: {"thought": "...", "framework": ["...", "..."]}
```

### 4.2 Executing Agent

```yaml
introduction: 你是一位精通信息分析的 AI 助手
target: 对查找到的知识进行整合、修正，回答用户问题
instruction: |
  1. 去除重复信息
  2. 去除错误信息
  3. 去除无用信息
  4. 尽量使用数值信息
  5. 注意时效性
```

### 4.3 Expressing Agent

```yaml
introduction: 你是一个文本编辑专家
target: 生成完整的结构化问题答案
instruction: |
  1. 总-分结构：先总结，再详细展开
  2. 使用数据和数值作为论据
  3. 语义连贯
  4. 没有重复信息
```

### 4.4 Reviewing Agent

```yaml
introduction: 你是一位专业的内容评审专家
target: 评估内容质量并提供改进建议
instruction: |
  评分维度：完整性、准确性、逻辑性、结构性...
  评分: 0-100
  输出 JSON: {"score": ..., "output": "...", "suggestion": "..."}
```

---

## 5. PEER 参数速查

| 参数 | 含义 | 典型值 |
|------|------|--------|
| `retry_count` | 最大迭代轮数 | 3-5 |
| `jump_step` | 重试起点（planning/executing/expressing/reviewing） | planning |
| `eval_threshold` | 评审通过分数线 | 60-80 |

---

## 6. 动手练习

### 练习 1：运行 peer_agent_app

```bash
cd examples/sample_apps/peer_agent_app
# 配置 API Key
python bootstrap/intelligence/server_application.py
```

用一个复杂的分析问题测试（如"分析2026年AI行业趋势"），观察四个 Agent 的各自输出。

### 练习 2：修改 jump_step 观察变化

将 `jump_step` 从 `planning` 改为 `executing`，再次运行。观察在重试时 Planning 的输出是否被复用。

### 练习 3：调低 eval_threshold

将 `eval_threshold` 从 60 调低到 30，观察 Agent 是否更早退出迭代。

### 练习 4：阅读 PeerWorkPattern 源码

打开 `agentuniverse/agent/work_pattern/peer_work_pattern.py`，逐行阅读 `invoke()` 方法。画出四个 Agent 之间的数据流图。

### 练习 5：设计 PEER 应用

设计一个使用 PEER 模式的"股票分析报告"Agent 系统。写出：
1. 四个 Agent 各自的 Prompt 设计
2. 给 Executing Agent 绑定什么 Tool
3. 设置什么 `eval_threshold` 合适

---

## 7. 概念速查表

| 概念 | 含义 |
|------|------|
| **PEER** | Plan→Execute→Express→Review 四角色模式 |
| **PlanningAgentTemplate** | 任务拆解（生成子问题列表） |
| **ExecutingAgentTemplate** | 信息检索与整合 |
| **ExpressingAgentTemplate** | 结构化输出 |
| **ReviewingAgentTemplate** | 质量评审与评分 |
| **jump_step** | 迭代重试起点 |
| **retry_count** | 最大迭代次数 |
| **eval_threshold** | 评审通过分数线 |

---

## 下一步

你已经理解了 PEER 模式——agentUniverse 最强大的多 Agent 协作范式。

下一步进入最后一章 **12-building-your-own.md**，综合运用所学知识，从零构建自己的 Agent 应用。
