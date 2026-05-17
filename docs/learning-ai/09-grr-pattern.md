# 09 — GRR 模式：Generate → Review → Rewrite

> **前置要求：** 完成 [08-rag-knowledge.md](08-rag-knowledge.md)
> **学习目标：** 理解 GRR 多 Agent 协作模式，掌握带反馈回路的内容生成流程
> **预计时间：** 1-2 天

---

## 1. 什么是 GRR？

**GRR = Generate（生成） → Review（评审） → Rewrite（改写）**

三个 Agent 组成一个带质量反馈的流水线，适用于内容生成场景：文章、报告、代码文档等。

```
用户: "写一篇关于AI Agent的技术博客"

┌──────────────────────────────────────────┐
│ ① GeneratingAgent（生成）                │
│    生成初稿...                            │
│    输出: "AI Agent是一种..."              │
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│ ② ReviewingAgent（评审）                 │
│    评分: 65分（60分为及格线）             │
│    建议: "缺少实例，逻辑可以更清晰"        │
│    判断: 65 < eval_threshold=80 → 需要重写│
└──────────────────┬───────────────────────┘
                   ↓
┌──────────────────────────────────────────┐
│ ③ RewritingAgent（改写）                 │
│    基于评审意见修改...                     │
│    输出: "AI Agent是人工智能领域的..."     │
└──────────────────┬───────────────────────┘
                   ↓
             回到 ② 再次评审
                   ↓
          评分: 85 ≥ 80 → 通过！输出最终结果
```

**类比全栈经验：** GRR 类似于 CI/CD pipeline 中的 build → test → fix 循环。ReviewingAgent 是"自动化测试"，RewritingAgent 是"自动修复"。

---

## 2. GRR 源码详解

### 2.1 GRRWorkPattern 类

```python
# agentuniverse/agent/work_pattern/grr_work_pattern.py:15-26
class GRRWorkPattern(WorkPattern):
    generating: GeneratingAgentTemplate = None    # 生成 Agent
    reviewing: ReviewingAgentTemplate = None      # 评审 Agent
    rewriting: RewritingAgentTemplate = None      # 改写 Agent
```

### 2.2 invoke() 核心循环

```python
# grr_work_pattern.py:30-80（简化）
def invoke(self, input_object, work_pattern_input, **kwargs):
    retry_count = work_pattern_input.get('retry_count', 2)    # 最多迭代2轮
    eval_threshold = work_pattern_input.get('eval_threshold', 60)  # 及格线60分

    for iteration in range(retry_count):
        # ① 生成（第一轮用 GeneratingAgent，后续用 RewritingAgent 的输出）
        if iteration == 0:
            generating_result = self._invoke_generating(input_object, ...)
        else:
            generating_result = rewriting_result  # 上一轮的改写结果

        # ② 评审
        reviewing_result = self._invoke_reviewing(input_object, ...)

        # ③ 判断：分数达标就提前结束
        if reviewing_result.get('score', 0) >= eval_threshold:
            break

        # ④ 改写（最后一轮不改写，直接返回）
        if iteration < retry_count - 1:
            rewriting_result = self._invoke_rewriting(input_object, ...)

    return {'result': grr_results}
```

### 2.3 三个 Agent 的特定 Template

| Agent | 模板类 | input_keys | output_keys |
|-------|--------|-----------|-------------|
| Generating | `GeneratingAgentTemplate` | `['input']` | `['output']` |
| Reviewing | `ReviewingAgentTemplate` | `['input', 'expressing_result']` | `['output', 'score', 'suggestion']` |
| Rewriting | `RewritingAgentTemplate` | `['input', 'generating_result', 'reviewing_result']` | `['output']` |

**关键设计：** ReviewingAgentTemplate 的 `output_keys` 包含 `score`——这个分数是 Reviewing Agent 通过 LLM 推理打出的，用于控制循环是否继续。工作模式通过 `input_object.add_data()` 在 Agent 之间传递数据。

### 2.4 参数说明

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `retry_count` | 2 | 最多迭代轮数（包括首次生成） |
| `eval_threshold` | 60 | 评审分数阈值（0-100），达标即退出 |

---

## 3. 动手练习

### 练习 1：运行 grr_agent_app

```bash
cd examples/sample_apps/grr_agent_app
python quick_start.py
```

观察每一轮的输出：生成 → 评审 → 改写 → 评审...

### 练习 2：调整参数

1. 将 `eval_threshold` 调到 90，观察需要多少轮迭代
2. 将 `retry_count` 调到 1，观察单轮就结束的效果
3. 修改 Reviewing Agent 的 Prompt，改变评分标准

### 练习 3：阅读源码

打开 `agentuniverse/agent/work_pattern/grr_work_pattern.py`，逐行阅读 `invoke()` 方法。理解 `_invoke_generating`、`_invoke_reviewing`、`_invoke_rewriting` 之间的关系。

### 练习 4：设计你自己的 GRR 应用

设计一个使用 GRR 模式的技术博客写作 Agent。写出 Generating / Reviewing / Rewriting 三个 Agent 的 Prompt 设计要点。

---

## 下一步

你已经理解了 GRR 模式——最简单的多 Agent 协作（3 Agent，带评分反馈）。

下一步进入 **10-is-pattern.md**，学习 IS（Implementation-Supervision）模式——两种角色的分工协作。
