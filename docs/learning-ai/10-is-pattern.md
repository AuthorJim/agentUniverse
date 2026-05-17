# 10 — IS 模式：执行与监督

> **前置要求：** 完成 [09-grr-pattern.md](09-grr-pattern.md)
> **学习目标：** 理解 IS 双 Agent 协作模式，掌握分步执行+监督反馈的工作方式
> **预计时间：** 1-2 天

---

## 1. 什么是 IS？

**IS = Implementation（执行） + Supervision（监督）**

两个 Agent 组成一个"开发+Code Review"式的协作对：

```
ImplementationAgent   ←→   SupervisionAgent
    (干活的)                  (盯着的)
```

类比全栈经验：Implementation = 开发写代码，Supervision = Reviewer 盯着检查。

```
┌────────────────────────────────────────────────┐
│ IS 执行流程（checkpoint_count=3）                │
│                                                │
│  Checkpoint 1:                                 │
│    Implementation 执行第1步 → Supervision 检查  │
│    → 没问题，继续                                │
│                                                │
│  Checkpoint 2:                                 │
│    Implementation 执行第2步 → Supervision 检查  │
│    → needs_correction=true !                   │
│    → Implementation 修正                       │
│                                                │
│  Checkpoint 3:                                 │
│    Implementation 执行第3步 → Supervision 检查  │
│    → 通过！                                     │
│                                                │
│  返回: 所有 checkpoint 的结果 + execution_context│
└────────────────────────────────────────────────┘
```

**和 GRR 的区别：**
- GRR：生成→评审→改写，整个输出一起评审
- IS：分步执行，每步都检查，发现问题立即修正

---

## 2. 源码详解

### 2.1 ISWorkPattern.invoke()

```python
# agentuniverse/agent/work_pattern/is_work_pattern.py:28-100（简化）
class ISWorkPattern(WorkPattern):
    implementation: ImplementationAgentTemplate = None
    supervision: SupervisionAgentTemplate = None

    def invoke(self, input_object, work_pattern_input, **kwargs):
        checkpoint_count = work_pattern_input.get('checkpoint_count', 3)
        max_corrections = work_pattern_input.get('max_corrections', 2)

        execution_context = {
            'user_goal': work_pattern_input.get('input'),
            'corrections_made': 0,           # ★ 已修正次数
            'checkpoint_history': []         # 历史记录
        }

        for checkpoint_idx in range(checkpoint_count):
            # ① Implementation 执行当前步骤
            implementation_result = self._invoke_implementation(...)

            # ② Supervision 监督检查
            supervision_result = self._invoke_supervision(...)

            # ③ 如果需要修正且还有修正机会
            if (supervision_result.get('needs_correction')
                    and execution_context['corrections_made'] < max_corrections):
                # Implementation 进行修正
                self._invoke_correction(...)
                execution_context['corrections_made'] += 1

            # ④ 记录本 checkpoint
            execution_context['checkpoint_history'].append({...})

        return {'result': is_results, 'execution_context': execution_context}
```

### 2.2 三个关键参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `checkpoint_count` | 3 | 执行分为几个检查点 |
| `max_corrections` | 2 | 整个流程最多修正几次 |
| `execution_context` | — | 跟踪执行状态（目标、已修正次数、检查点历史） |

### 2.3 Implementation 和 Supervision 的交互

```
ImplementationAgent
  ├── 正常模式：接收 checkpoint_index, total_checkpoints, execution_context
  │      → 执行当前步骤的任务
  │
  └── 修正模式：接收 correction_mode=True, supervision_feedback
         → 基于监督反馈修正之前的输出

SupervisionAgent
  接收：checkpoint_index, execution_context
  输出：needs_correction (bool), feedback (str)
```

### 2.4 IS 的适用场景

| 场景 | Implementation 做什么 | Supervision 检查什么 |
|------|---------------------|---------------------|
| 代码生成 | 逐步实现功能 | 检查代码正确性、风格 |
| 文档写作 | 逐步撰写章节 | 检查逻辑一致性、格式 |
| 数据分析 | 逐步分析数据 | 检查分析方法的正确性 |
| 任务规划 | 逐步拆解执行 | 检查是否偏离目标 |

---

## 3. 和 GRR 的对比

| 维度 | GRR | IS |
|------|-----|-----|
| Agent 数量 | 3 | 2 |
| 控制流 | 整篇贯穿式评审 | 分步 checkpoint 式 |
| 反馈粒度 | 粗粒度（全文） | 细粒度（每步） |
| 最佳场景 | 内容创作 | 目标对齐型任务 |
| 修正方式 | 全文重写 | 局部修正 |
| 退出条件 | 分数达标 | 达到 checkpoint 数量 |

---

## 4. 动手练习

### 练习 1：运行 is_agent_app

```bash
cd examples/sample_apps/is_agent_app
python simple_example.py
```

观察每个 checkpoint 的 Implementation 输出和 Supervision 反馈。

### 练习 2：阅读源码

打开 `agentuniverse/agent/work_pattern/is_work_pattern.py`，逐行阅读 `invoke()` 方法。特别关注：
1. `execution_context` 如何在各 checkpoint 之间传递
2. `_invoke_correction` 如何复用 Implementation Agent
3. `needs_correction` 的判断逻辑

### 练习 3：设计 IS 应用

设计一个使用 IS 模式的"Python 项目开发" Agent 系统。Implementation 负责逐步编写代码，Supervision 负责检查代码质量。描述两个 Agent 的 Prompt 设计。

---

## 下一步

你已经理解了 IS 模式——执行和监督的双 Agent 协作。

下一步进入 **11-peer-pattern.md**，学习最复杂的 PEER 模式——四种角色的完整推理流水线。
