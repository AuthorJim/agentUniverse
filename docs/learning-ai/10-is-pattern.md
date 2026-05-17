# 10 — IS 模式：执行 + 监督的双 Agent 协作

> **前置要求：** 完成 [09-grr-pattern.md](09-grr-pattern.md)
> **学习目标：** 理解 IS（Implementation + Supervision）双 Agent 协作模式，掌握分步执行和检查点反馈机制
> **预计时间：** 1-2 天

---

## 1. IS 是什么？

**IS = Implementation（执行）+ Supervision（监督）**

两个 Agent，一个干活，一个盯着。像极了"开发写代码 + Reviewer 盯着检查"的日常：

```
ImplementationAgent   ←→   SupervisionAgent
    (干活的)                  (盯着的)
```

### 1.1 和 GRR 的关键区别

| 维度 | GRR | IS |
|------|-----|-----|
| Agent 数量 | 3 (生成→评审→改写) | 2 (执行⇄监督) |
| 反馈粒度 | 全文完成后一起评审 | **每步都检查**（checkpoint 机制） |
| 修正方式 | 整篇重写 | **局部修正** |
| 退出条件 | 分数达标 | 完成所有 checkpoint |
| 最佳场景 | 内容创作（文章、博客） | 目标对齐型任务（代码生成、任务执行） |

---

## 2. IS 执行流程

```
用户目标: "开发一个 Flask API 项目，包含用户注册和登录"

┌──────────────────────────────────────────────────┐
│ IS 流程 (checkpoint_count=3)                     │
│                                                  │
│  Checkpoint 1: "设计项目结构"                     │
│    Implementation: 生成目录结构和依赖文件          │
│    Supervision: 检查 ─ 结构合理，通过 ✓            │
│                                                  │
│  Checkpoint 2: "实现注册功能"                     │
│    Implementation: 编写注册 API 代码              │
│    Supervision: 检查 ─ 缺少密码加密，不通过 ✗     │
│    → needs_correction = True                     │
│    Implementation: 修正（加上 bcrypt 加密）       │
│    Supervision: 检查 ─ 通过 ✓                    │
│                                                  │
│  Checkpoint 3: "实现登录功能"                     │
│    Implementation: 编写登录 API 代码              │
│    Supervision: 检查 ─ JWT token 正确，通过 ✓     │
│                                                  │
│  返回: 所有 checkpoint 的结果 + execution_context │
└──────────────────────────────────────────────────┘
```

---

## 3. 源码解读

### 3.1 ISWorkPattern 类

```python
# agentuniverse/agent/work_pattern/is_work_pattern.py:28-100 (简化)
class ISWorkPattern(WorkPattern):
    implementation: ImplementationAgentTemplate = None
    supervision: SupervisionAgentTemplate = None

    def invoke(self, input_object, work_pattern_input, **kwargs):
        checkpoint_count = work_pattern_input.get('checkpoint_count', 3)
        max_corrections = work_pattern_input.get('max_corrections', 2)

        execution_context = {
            'user_goal': work_pattern_input.get('input'),
            'corrections_made': 0,
            'checkpoint_history': []
        }

        for checkpoint_idx in range(checkpoint_count):
            # ① Implementation 执行当前步骤
            impl_result = self._invoke_implementation(
                checkpoint_index=checkpoint_idx,
                total_checkpoints=checkpoint_count,
                execution_context=execution_context
            )

            # ② Supervision 检查
            supervision_result = self._invoke_supervision(
                checkpoint_index=checkpoint_idx,
                implementation_output=impl_result,
                execution_context=execution_context
            )

            # ③ 需要修正且还有修正次数
            if (supervision_result.get('needs_correction')
                    and execution_context['corrections_made'] < max_corrections):
                self._invoke_correction(
                    supervision_feedback=supervision_result.get('feedback'),
                    execution_context=execution_context
                )
                execution_context['corrections_made'] += 1

            # ④ 记录 checkpoint 历史
            execution_context['checkpoint_history'].append({
                'checkpoint': checkpoint_idx,
                'result': impl_result,
                'needs_correction': supervision_result.get('needs_correction')
            })

        return {'result': execution_context['checkpoint_history']}
```

### 3.2 三个关键参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `checkpoint_count` | 3 | 任务分几个检查点 |
| `max_corrections` | 2 | 整个流程最多修正几次 |
| `execution_context` | — | 跨 checkpoint 共享的状态 |

### 3.3 两个 Agent 的交互模式

ImplementationAgent 有两种工作模式：
- **正常模式**：接收 `checkpoint_index`、`total_checkpoints`、`execution_context`，执行当前步骤
- **修正模式**：接收 `correction_mode=True`、`supervision_feedback`，基于反馈修正

SupervisionAgent 总是：
- 接收当前 checkpoint 的 Implementation 输出
- 输出 `needs_correction`（bool）和 `feedback`（str）

---

## 4. IS 适用场景

| 场景 | Implementation 做什么 | Supervision 检查什么 |
|------|---------------------|---------------------|
| 代码生成 | 逐步实现功能模块 | 代码正确性、风格、安全性 |
| 项目规划 | 逐步制定计划 | 是否偏离目标、资源是否合理 |
| 数据分析 | 逐步分析数据 | 分析方法正确性、结论合理性 |
| 文档写作 | 逐章撰写 | 逻辑一致性、格式规范性 |

---

## 5. IS vs GRR vs PEER — 如何选择？

| 你的任务 | 推荐模式 | 理由 |
|---------|---------|------|
| 写一篇博客 | GRR | 整篇完成后再评审更自然 |
| 开发一个项目（分模块） | IS | 分步执行、每步检查 |
| 复杂推理分析报告 | PEER | 需要深度推理和专业输出 |
| 简单问答 | ReAct 单 Agent | 不需要多 Agent 开销 |

---

## 6. 动手练习

### 练习 1：运行 is_agent_app

```bash
cd examples/sample_apps/is_agent_app
python simple_example.py
```

观察每个 checkpoint 的 Implementation 输出和 Supervision 反馈。

### 练习 2：阅读源码

打开 `agentuniverse/agent/work_pattern/is_work_pattern.py`，阅读 `invoke()` 方法。特别关注 `execution_context` 如何在 checkpoint 间传递。

### 练习 3：设计 IS 应用

设计一个使用 IS 模式的"API 开发 Agent"：
- Implementation Agent：逐步编写 Flask REST API
- Supervision Agent：检查路由设计、错误处理、安全性

写出两个 Agent 的 Prompt 设计。

### 练习 4：修改 checkpoint_count

将 `checkpoint_count` 从 3 改为 5，观察任务被拆分成更多步骤后的效果。

---

## 下一步

掌握了 IS（双 Agent 检查点式协作）后，进入 **[11-peer-pattern.md](11-peer-pattern.md)**——PEER 模式，agentUniverse 最强大的四角色流水线。
