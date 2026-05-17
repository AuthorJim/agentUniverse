# 09 — GRR 模式：三个 Agent 的创作流水线

> **前置要求：** 完成 [08-rag-knowledge.md](08-rag-knowledge.md)
> **学习目标：** 理解 GRR（Generate → Review → Rewrite）多 Agent 协作模式，掌握带质量反馈的迭代内容生成
> **预计时间：** 1-2 天

---

## 1. GRR 是什么？

**GRR = Generate（生成）→ Review（评审）→ Rewrite（改写）**

三个 Agent 组成一个带质量反馈的创作流水线，最适合**内容生成**场景：文章写作、报告生成、技术文档、翻译等。

### 1.1 为什么需要"质量反馈"？

单 Agent 写文章的问题：**写完了没人检查。** 就像你写代码不跑测试直接上线——质量全靠运气。

GRR 解决的就是这个问题：**有人写、有人审、有人改。**

### 1.2 用 CI/CD 类比

```
CI/CD Pipeline:  Build → Test → Fix → Test → Deploy
GRR Pipeline:    Generate → Review → Rewrite → Review → Output
```

ReviewingAgent 是"自动化测试套件"，RewritingAgent 是"自动修复脚本"。

---

## 2. GRR 的完整执行流程

```
用户: "写一篇关于 AI Agent 的技术博客"

┌──────────────────────────────────────────────────┐
│ ① GeneratingAgent（生成）                        │
│    Prompt: 你是一位技术博客作者                   │
│    输出: "AI Agent 是人工智能领域的前沿技术..."    │
│    (初稿，质量参差不齐)                           │
└─────────────────────┬────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ ② ReviewingAgent（评审）                         │
│    Prompt: 你是内容评审专家，从 5 个维度打分       │
│    评分: 65/100                                  │
│    建议: "缺少具体案例，第3段逻辑跳跃，建议增加..." │
│    判断: 65 < eval_threshold(80) → 不合格！      │
└─────────────────────┬────────────────────────────┘
                      ↓
┌──────────────────────────────────────────────────┐
│ ③ RewritingAgent（改写）                         │
│    Prompt: 基于评审意见改进这篇文章               │
│    接收: 初稿 + 评审意见                          │
│    输出: 改进版 "AI Agent 是... 以 AlphaGo 为例..."│
└─────────────────────┬────────────────────────────┘
                      ↓
              回到 ② 再次评审
                      ↓
          评分: 85 ≥ 80 → 通过！输出最终版本
```

---

## 3. 源码解读

### 3.1 GRRWorkPattern 类

```python
# agentuniverse/agent/work_pattern/grr_work_pattern.py:15-26
class GRRWorkPattern(WorkPattern):
    generating: GeneratingAgentTemplate = None    # 生成 Agent
    reviewing: ReviewingAgentTemplate = None      # 评审 Agent
    rewriting: RewritingAgentTemplate = None      # 改写 Agent
```

### 3.2 invoke() 核心循环

```python
# grr_work_pattern.py:30-80 (简化)
def invoke(self, input_object, work_pattern_input, **kwargs):
    retry_count = work_pattern_input.get('retry_count', 2)      # 最多迭代 2 轮
    eval_threshold = work_pattern_input.get('eval_threshold', 60) # 及格线 60 分

    for iteration in range(retry_count):
        # ① 生成 / 使用上一轮的改写结果
        if iteration == 0:
            generating_result = self._invoke_generating(...)   # 首次：用 GeneratingAgent
        else:
            generating_result = rewriting_result                # 后续：用改写结果

        # ② 评审
        reviewing_result = self._invoke_reviewing(input_object, ...)

        # ③ 判断：分数达标 → 提前退出
        if reviewing_result.get('score', 0) >= eval_threshold:
            break

        # ④ 改写（最后一轮跳过，直接返回评审结果）
        if iteration < retry_count - 1:
            rewriting_result = self._invoke_rewriting(input_object, ...)

    return {'result': grr_results}
```

### 3.3 关键参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `retry_count` | 2 | 最多迭代轮数（含首次生成） |
| `eval_threshold` | 60 | 评审通过的最低分数（0-100） |

### 3.4 三个 Agent 的职责

| Agent | Template 类 | 输入 | 输出 |
|-------|-----------|------|------|
| Generating | `GeneratingAgentTemplate` | 用户原始需求 | 初稿内容 |
| Reviewing | `ReviewingAgentTemplate` | 初稿 + 编辑标准 | score + output + suggestion |
| Rewriting | `RewritingAgentTemplate` | 初稿 + 评审意见 | 改进版内容 |

**ReviewingAgent 的 `output_keys` 是关键：** 它必须输出 `score`（数字分数），这个分数就是循环是否继续的判断依据。

---

## 4. 各 Agent 的 Prompt 设计要点

### GeneratingAgent — 内容生产者

```yaml
introduction: '你是一位资深的技术博客作者，拥有10年写作经验。'
target: '根据用户需求，生成一篇结构清晰、内容翔实的文章。'
instruction: |
  1. 使用总-分结构：开篇概述 + 分段详述 + 总结
  2. 包含 2-3 个具体案例
  3. 语言专业但不生涩，面向中级读者
  4. 字数 800-1500
```

### ReviewingAgent — 质量守门员

```yaml
introduction: '你是一位严格但公正的内容评审专家。'
target: '从多个维度评估文章质量并打分。'
instruction: |
  评分维度 (每项 20 分，满分 100):
  1. 内容准确性 — 事实正确，概念清晰
  2. 结构完整性 — 开头-主体-结尾完整
  3. 逻辑连贯性 — 段落过渡自然
  4. 语言表达 — 流畅，无语病
  5. 实用性 — 对读者有实际价值

  输出 JSON: {"score": int, "output": "评审总结", "suggestion": "改进建议"}
```

### RewritingAgent — 修改执行者

```yaml
introduction: '你是一位善于根据反馈改进文章的编辑。'
target: '基于评审意见，修改文章以提升质量。'
instruction: |
  1. 逐条处理评审意见
  2. 保留原文优点
  3. 修改后标注主要改动
  4. 输出完整改进版，不需要 JSON
```

---

## 5. GRR 的适用场景与局限

| 适用场景 | 不适用场景 |
|---------|-----------|
| 技术博客、文章写作 | 实时信息查询（用 ReAct） |
| 报告生成（行业分析、周报） | 多步推理计算（用 PEER） |
| 翻译（原文→评审→润色） | 简单问答 |
| 代码文档生成 | 需要复杂外部工具的交互 |

---

## 6. 动手练习

### 练习 1：运行 grr_agent_app

```bash
cd examples/sample_apps/grr_agent_app
python quick_start.py     # 或者直接运行示例脚本
```

观察每轮迭代的生成→评审→改写→评审...直到通过。

### 练习 2：调参实验

1. 将 `eval_threshold` 调到 90 — 观察需要多少轮才能达标
2. 将 `retry_count` 调到 1 — 观察单轮就结束的效果
3. 修改 Reviewing Agent 的评分标准 — 看输出变化

### 练习 3：阅读源码

打开 `agentuniverse/agent/work_pattern/grr_work_pattern.py`，逐行阅读 `invoke()` 方法。理解三个 Agent 之间的数据传递方式（通过 `input_object.add_data()`）。

### 练习 4：设计 GRR 翻译系统

设计一个使用 GRR 的中→英翻译系统：
1. GeneratingAgent：初译
2. ReviewingAgent：从准确性、流畅度、地道性评分
3. RewritingAgent：根据评审意见润色

写出三个 Agent 的 Prompt 设计。

---

## 下一步

掌握了 GRR（3 Agent 创作流水线）后，进入 **[10-is-pattern.md](10-is-pattern.md)**——IS 模式，双 Agent 的"开发+Review"协作方式。
