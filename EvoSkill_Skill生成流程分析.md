# EvoSkill: Skill 生成流程代码级分析

> 仓库: [sentient-agi/EvoSkill](https://github.com/sentient-agi/EvoSkill)
> 论文: https://arxiv.org/abs/2603.02766

---

## 一、整体架构总览

EvoSkill 通过一个 **进化自改进循环 (Self-Improving Loop)** 来发现和合成可复用的 Agent Skill。每次迭代包含 5 个阶段:

```
┌─────────────────────────────────────────────────────────────────────┐
│                     SelfImprovingLoop.run()                        │
│                     src/loop/runner.py                              │
│                                                                     │
│  1. Base Agent 尝试回答问题 ──→ 收集失败 (score < 0.8)              │
│         │                                                           │
│  2. 消息预处理: build_proposer_query()                              │
│     (截断trace, 整理feedback history, 列举现有skills)                │
│         │                                                           │
│  3. Skill Proposer 分析失败 ──→ 提出 create/edit 建议               │
│         │                                                           │
│  4. Skill Generator 实际创建/编辑 skill 文件                        │
│         │                                                           │
│  5. 评估 → Frontier 管理 → 反馈记录                                │
│     (git branch 版本管理, 淘汰/保留)                                │
└─────────────────────────────────────────────────────────────────────┘
```

两种演化模式:
- **`skill_only`** (默认): 生成/编辑 `.claude/skills/` 下的 skill 文件
- **`prompt_only`**: 重写 base agent 的 system prompt

本文聚焦 `skill_only` 模式的完整流程。

---

## 二、关键数据结构

### 2.1 AgentTrace — Agent 执行记录

**文件**: `src/agent_profiles/base.py`

```python
class AgentTrace(BaseModel, Generic[T]):
    uuid: str
    session_id: str
    model: str
    tools: list[str]
    duration_ms: int
    total_cost_usd: float
    num_turns: int
    usage: dict[str, Any]
    result: str              # Agent 的原始输出文本
    is_error: bool
    output: Optional[T]      # 经 Pydantic 验证的结构化输出
    parse_error: Optional[str]
    raw_structured_output: Optional[Any]
    messages: list[Any]      # 完整消息列表
```

`summarize()` 方法用于生成 trace 摘要（传给 proposer）:
```python
def summarize(self, head_chars=60_000, tail_chars=60_000) -> str:
    # 成功时: 返回完整 trace
    # 失败时 (parse_error): 截断为 head + tail，避免 context 爆炸
    if self.parse_error and len(result_str) > (head_chars + tail_chars):
        # 返回: 前 head_chars + "[...truncated...]" + 后 tail_chars
    else:
        # 返回完整 result
```

### 2.2 SkillProposerResponse — Proposer 输出

**文件**: `src/schemas/skill_proposer.py`

```python
class SkillProposerResponse(BaseModel):
    action: Literal["create", "edit"] = "create"  # 创建新 skill 还是编辑现有
    target_skill: Optional[str] = None             # edit 时的目标 skill 名
    proposed_skill: str                             # skill 高层描述
    justification: str                              # 为什么需要这个 skill
    related_iterations: List[str] = []              # 关联的历史迭代
```

### 2.3 ToolGeneratorResponse — Generator 输出

**文件**: `src/schemas/tool_generator.py`

```python
class ToolGeneratorResponse(BaseModel):
    generated_skill: str    # 生成的 skill 内容
    reasoning: str          # 推理过程
```

### 2.4 LoopConfig — 循环配置

**文件**: `src/loop/config.py`

```python
@dataclass
class LoopConfig:
    max_iterations: int = 5           # 最大迭代次数
    frontier_size: int = 3            # Frontier 保留的最优 program 数
    no_improvement_limit: int = 5     # 连续无改进后停止
    concurrency: int = 4              # 并发评估数
    evolution_mode: "skill_only" | "prompt_only"
    selection_strategy: "best" | "random" | "round_robin"
    categories_per_batch: int = 3     # 每轮采样的类别数
    samples_per_category: int = 2     # 每类别采样数
    proposer_max_truncation_level: int = 2  # 最大截断级别
    proposer_single_failure_fallback: bool = True  # 单失败回退
```

### 2.5 ProgramConfig — 程序版本配置

**文件**: `src/registry/models.py`

每个 "program" 是一个 git branch，包含:
- `.claude/program.yaml` — 配置 (system_prompt, tools, metadata)
- `.claude/skills/` — 该版本的 skill 集合

---

## 三、完整流程详解

### 阶段 0: 初始化

**入口**: `SelfImprovingLoop.run()` @ `src/loop/runner.py:218`

```
1. 如果非 continue_mode:
   - 删除 feedback_history.md
   - 删除 checkpoint
2. 创建 base program (git branch: program/base)
3. 在验证集上评估 base → 加入 Frontier
```

---

### 阶段 1: Base Agent 尝试回答 → 收集失败

**位置**: `runner.py:260-321`

```python
# 轮询采样: 从 M 个类别中各取 N 个问题
for j in range(n_cats_this_iter):
    cat_idx = (self._category_offset + j) % n_cats
    cat = categories[cat_idx]
    # 取 samples_per_category 个样本

# 并发运行 base agent
traces = await asyncio.gather(*[
    self.agents.base.run(question) for question, _, _ in test_samples
])

# 收集失败 (score < 0.8)
for trace, (question, answer, category) in zip(traces, test_samples):
    agent_answer = trace.output.final_answer if trace.output else "[PARSE FAILED]"
    avg_score = self.scorer(question, agent_answer, answer)
    if avg_score < 0.8:
        failures.append((trace, agent_answer, answer, category))
```

评分函数 `_score_multi_tolerance()`:
```python
TOLERANCE_LEVELS = [0.05, 0.01, 0.1, 0.0, 0.025]

def _score_multi_tolerance(question, predicted, ground_truth):
    # 加权平均, 严格容忍度权重更高
    # weight = 1.0 / (1.0 + 20.0 * tol)
    # 0.0% 容忍度 → 权重 1.00 (精确匹配最重要)
    # 10.0% 容忍度 → 权重 0.33
```

---

### 阶段 2: 消息预处理 — `build_proposer_query()`

**文件**: `src/loop/helpers.py:11-106`

这是整个流程中最关键的预处理步骤，将原始失败 trace 转化为 proposer 可消化的结构化 query。

#### 步骤 2.1: Trace 截断级别选择

```python
TRUNCATION_SETTINGS = [
    (60_000, 60_000, None, None),    # Level 0: 完整 (head=60k, tail=60k)
    (20_000, 10_000, 20, 3),         # Level 1: 中等 (head=20k, tail=10k, feedback=20行, 最多3个failure)
    (5_000, 2_000, 5, 2),            # Level 2: 激进 (head=5k, tail=2k, feedback=5行, 最多2个failure)
]
```

#### 步骤 2.2: 限制失败数量

```python
if max_failures is not None and len(traces_with_answers) > max_failures:
    traces_with_answers = traces_with_answers[:max_failures]
```

#### 步骤 2.3: 截断 Feedback History

```python
if feedback_lines is not None:
    feedback_lines_list = feedback_history.split("\n")
    if len(feedback_lines_list) > feedback_lines:
        feedback_history = "\n".join(feedback_lines_list[-feedback_lines:])
```

#### 步骤 2.4: 盘点现有 Skills

```python
skills_dir = Path(".claude/skills")
existing_skills = []
if skills_dir.exists():
    for skill_dir in skills_dir.iterdir():
        if skill_dir.is_dir() and (skill_dir / "SKILL.md").exists():
            existing_skills.append(skill_dir.name)
skills_list = "\n".join([f"- {s}" for s in existing_skills]) or "None"
```

#### 步骤 2.5: 类别汇总

```python
categories = [cat for _, _, _, cat in traces_with_answers]
category_summary = ", ".join(sorted(set(categories)))
```

#### 步骤 2.6: 逐个失败生成 Trace 摘要

```python
for i, (trace, agent_answer, ground_truth, category) in enumerate(traces_with_answers, 1):
    if evolution_mode == "prompt_only":
        effective_head = min(head_chars, 20_000)  # prompt 模式更紧凑
        effective_tail = min(tail_chars, 10_000)
    else:
        effective_head = head_chars
        effective_tail = tail_chars

    trace_summary = trace.summarize(head_chars=effective_head, tail_chars=effective_tail)
```

#### 步骤 2.7: 组装最终 Query

完整的 query 模板:

```
## Existing Skills (check before proposing new ones)
{skills_list}

## Task Constraints                    ← 可选, 来自 .evoskill/task.md
{task_constraints}

## Previous Attempts Feedback
{feedback_history}

## Current Failures ({count} samples across categories: {category_summary})

Analyze the patterns across these failures to identify a GENERAL improvement,
not a fix for any single case.

### Failure 1 [Category: {category}]
Model: {model}
Turns: {num_turns}
Duration: {duration_ms}ms
Is Error: {is_error}
Parse Error: {parse_error}       ← 如有
Output: {output}                 ← 如有

## Full Result / Result (truncated)
{trace_result 或 head+tail截断}

Agent Answer: {agent_answer}
Ground Truth: {ground_truth}

### Failure 2 [Category: ...]
...

## Your Task
1. Check if any EXISTING skill should have handled these failures
2. If yes → propose EDITING that skill (action="edit", target_skill="skill-name")
3. If no → propose a NEW skill (action="create")
4. Reference any related DISCARDED iterations and explain how your proposal differs
5. Identify what COMMON pattern or capability gap caused these failures across categories
```

---

### 阶段 3: Skill Proposer — 分析失败并提出建议

**Prompt 文件**: `src/agent_profiles/skill_proposer/prompt.py`
**Agent 配置**: `src/agent_profiles/skill_proposer/skill_proposer.py`

#### 3.1 Agent 配置

```python
# System prompt = Claude Code 预设 + 追加的 SKILL_PROPOSER_SYSTEM_PROMPT
skill_proposer_system_prompt = {
    "type": "preset",
    "preset": "claude_code",           # 基于 Claude Code 预设
    "append": SKILL_PROPOSER_SYSTEM_PROMPT  # 追加 proposer 专用 prompt
}

# 输出格式: JSON Schema (来自 SkillProposerResponse)
skill_proposer_output_format = {
    "type": "json_schema",
    "schema": SkillProposerResponse.model_json_schema()
}

# 可用工具 (只读, 不能写文件)
SKILL_PROPOSER_TOOLS = [
    "Read", "Bash", "Glob", "Grep",
    "WebFetch", "WebSearch", "TodoWrite", "BashOutput"
]
```

#### 3.2 完整 System Prompt

```
You are an expert agent performance analyst specializing in identifying
opportunities to enhance agent capabilities through skill additions or
modifications. Your role is to carefully analyze agent execution traces
and propose targeted skill improvements.

## Your Task

Given an agent's execution trace, its answer, and the ground truth answer,
propose either:
- A **new skill** (action="create") if no existing skill covers the capability gap
- An **edit to an existing skill** (action="edit") if an existing skill
  SHOULD have prevented the failure but didn't

Your proposal will be passed to a downstream Skill Builder agent for implementation.

## Required Pre-Analysis Steps

BEFORE proposing any skill, you MUST use the **Brainstorming skill**
(read `.claude/skills/brainstorming/SKILL.md`):

1. **Use Brainstorming skill** (MANDATORY):
   - Read and follow the process in `.claude/skills/brainstorming/SKILL.md`
   - Propose 2-3 different approaches to address the failures
   - For each approach: describe the core idea, trade-offs, and complexity
   - Explore alternatives before settling on your final proposal
   - Apply YAGNI - choose the simplest solution that addresses the root cause

2. **Inventory existing skills**: Review the list of existing skills provided
   - Understand what capabilities are already available
   - Check if any existing skill covers similar ground

3. **Analyze feedback history** for:
   - DISCARDED proposals similar to what you're considering
   - Patterns in what works vs what regresses scores
   - Skills that were active when failures occurred

4. **Determine action type**:
   - If an existing skill SHOULD have prevented this failure → propose EDIT
   - If no existing skill covers this capability → propose CREATE
   - If a DISCARDED proposal was on the right track → explain how yours differs

## Analysis Process

Before proposing a solution, work through these steps:

<analysis>
1. **Trace Review**: Examine the agent's execution trace step-by-step
   - What actions did the agent take?
   - Where did it succeed or struggle?
   - What information was available vs. missing?

2. **Gap Analysis**: Compare the agent's answer to the ground truth
   - What specific information is incorrect or missing?
   - What reasoning errors occurred?
   - What capabilities would have prevented these issues?

3. **Existing Skill Check**: Review the listed existing skills
   - Does any existing skill cover this capability?
   - If yes, why did it fail to prevent the error?
   - Should that skill be EDITED instead of creating a new one?

4. **Skill Identification**: Determine what skill would address the failure
   - What new capability, tool, or workflow would help?
   - What inputs should it accept?
   - What outputs should it produce?
   - How would it integrate with existing capabilities?
</analysis>

## Anti-Patterns to Avoid

- DON'T propose a new skill if an existing one covers similar ground → propose EDIT
- DON'T ignore previous DISCARDED proposals → explain how yours differs
- DON'T create narrow skills that only fix one specific failure → ensure broad applicability
- DON'T propose capabilities that overlap with existing skills → consolidate

## When to Propose Skills

Propose a skill when ANY of these apply:
- Agent lacks access to information, APIs, or computational capabilities
- The fix requires a multi-step procedure (>3 sequential steps)
- The fix involves output structuring, formatting, or templates
- The improvement would be reusable across different tasks
- The issue is about WHAT steps to take, not HOW to think

## Output Requirements

1. **action**: "create" or "edit"
2. **target_skill**: (edit 时必填) 目标 skill 名
3. **proposed_skill**: 详细描述
4. **justification**: 推理过程, 引用 trace 中具体时刻
5. **related_iterations**: 关联的历史迭代列表
```

#### 3.3 调用位置

**`runner.py:508-524`**:

```python
if evolution_mode == "skill_only":
    proposer_trace = await self.agents.skill_proposer.run(proposer_query)

    if proposer_trace.output is None:
        return None  # proposer 失败

    proposer_output = proposer_trace.output
    proposed = proposer_output.proposed_skill      # 提议的 skill 描述
    justification = proposer_output.justification  # 理由
    action_type = proposer_output.action           # "create" 或 "edit"
    target_skill = proposer_output.target_skill    # edit 时的目标
```

---

### 阶段 4: Skill Generator — 实际创建 Skill

**Prompt 文件**: `src/agent_profiles/skill_generator/prompt.py`
**Agent 配置**: `src/agent_profiles/skill_generator/skill_generator.py`

#### 4.1 Agent 配置

```python
# System prompt = Claude Code 预设 + Generator 专用 prompt
skill_generator_system_prompt = {
    "type": "preset",
    "preset": "claude_code",
    "append": SKILL_GENERATOR_SYSTEM_PROMPT
}

# 输出格式
skill_generator_output_format = {
    "type": "json_schema",
    "schema": ToolGeneratorResponse.model_json_schema()
}

# 可用工具 (有写权限!)
SKILL_GENERATOR_TOOLS = [
    "Read", "Write", "Bash", "Glob", "Grep", "Edit",
    "WebFetch", "WebSearch", "TodoWrite", "BashOutput", "Skill"
]

skill_generator_options = ClaudeAgentOptions(
    ...,
    setting_sources=["user", "project"],  # 加载项目 skills
    permission_mode='acceptEdits',         # 自动批准文件修改
)
```

#### 4.2 完整 System Prompt

```
You are an expert skill developer specializing in creating tools and capabilities
for Claude Code agents. Your role is to implement well-structured, production-ready
skills based on high-level descriptions provided by a Proposer agent.

## Primary Directive

**Before implementing any skill, always read and follow the
`.claude/skills/skill-creator/SKILL.md` skill.** This skill contains essential
patterns, validation requirements, scripts, and best practices that ensure your
implementations work correctly within the Claude Code ecosystem.

## Your Task

Given a proposed skill description, implement a complete, functional skill that:
1. Follows the skill-creator's structure and conventions
2. Integrates properly with the Claude Code SDK
3. Is well-documented and maintainable
4. Handles edge cases gracefully

## Implementation Process

<implementation_steps>
1. **Read the Skill-Creator Skill**: Load and follow `.claude/skills/skill-creator/SKILL.md`
2. **Implement and Validate**: Build, test, and package the skill
</implementation_steps>

## Quality Reminder

The context window is a shared resource. Every token in your skill competes with
conversation history, other skills, and user requests. Challenge each piece of
content: "Does Claude really need this?" Keep skills concise and let Claude's
intelligence fill in the gaps.
```

#### 4.3 传给 Generator 的 Query

根据 action 类型不同，构建不同的 query:

**创建新 Skill** (`helpers.py:224-237`):

```python
def build_skill_query_from_skill_proposer(proposer_trace):
    return f"""Proposed tool or skill (high level description): {proposer_trace.output.proposed_skill}

Justification: {proposer_trace.output.justification}"""
```

**编辑现有 Skill** (`runner.py:535-543`):

```python
skill_query = f"""EDIT existing skill: {target_skill}

Modifications needed:
{proposed}

Justification: {justification}

Read the existing skill at .claude/skills/{target_skill}/SKILL.md
and modify it to add these capabilities. Preserve all existing content that is still relevant."""
```

#### 4.4 Generator 内部工作流 (由 skill-creator 元技能驱动)

Generator 被要求首先读取 `.claude/skills/skill-creator/SKILL.md`，该元技能定义了标准的 skill 创建流程:

```
1. 理解 skill 的具体用例
2. 规划可复用内容 (scripts, references, assets)
3. 初始化 skill (运行 scripts/init_skill.py)
4. 编辑 skill (实现资源, 编写 SKILL.md)
5. 打包 skill (运行 scripts/package_skill.py)
6. 基于实际使用迭代
```

**Skill 目录结构**:

```
skill-name/
├── SKILL.md (必需)
│   ├── YAML frontmatter (name + description)
│   └── Markdown 指令体
└── 可选资源/
    ├── scripts/      — 可执行代码
    ├── references/   — 参考文档
    └── assets/       — 输出资产 (模板等)
```

**SKILL.md frontmatter 示例**:

```yaml
---
name: financial-methodology-guide
description: Guide for financial calculations including ES/CVaR, z-scores,
  and bond pricing. Use when tasks involve quantitative finance methodology.
---
```

#### 4.5 调用和后续

```python
# runner.py:548-556
skills_before = set(self._get_active_skills())
skill_trace = await self.agents.skill_generator.run(skill_query)

if skill_trace.output:
    skills_after = set(self._get_active_skills())
    new_skills = skills_after - skills_before  # diff 检测新 skill
    created_skill = next(iter(new_skills)) if new_skills else None
```

---

### 阶段 5: 后处理 — 评估、Frontier 管理、反馈记录

#### 5.1 Git Commit

**`runner.py:590`**:

```python
self.manager.commit(f"{child_name}: {proposed[:50]}")
```

`ProgramManager.commit()` 会 `git add .` + `git commit`。

#### 5.2 评估新版本

```python
# runner.py:345-347
child_score = await self._evaluate(self.val_data)
```

在验证集上运行 base agent，计算加权多容忍度分数。

#### 5.3 Frontier 更新

**`src/registry/manager.py:333-385`**:

```python
def update_frontier(self, name, score, max_size=5):
    # 1. 更新 program.yaml 中的 score
    # 2. 如果 frontier 未满 → 直接加入
    # 3. 如果 score > 最差成员 → 替换最差, 加入
    # 4. 否则 → 不加入 (返回 False)
```

每个 frontier 成员通过 git tag `frontier/{name}` 标记。

#### 5.4 淘汰机制

```python
# runner.py:358-361
if not added:
    outcome = "discarded"
    self.manager.discard(child_name)  # 删除 git branch + frontier tag
```

#### 5.5 反馈记录

**`helpers.py:145-195`**:

```python
def append_feedback(path, iteration, proposal, justification,
                    outcome=None, score=None, parent_score=None,
                    active_skills=None, ...):
```

写入 `.claude/feedback_history.md`:

```markdown
## iter-skill-3
**Proposal**: Create a financial-methodology-guide skill...
**Justification**: The trace shows...
**Outcome**: IMPROVED (score: 0.5200, +0.0900)
**Active Skills**: brainstorming, skill-creator, financial-methodology-guide
```

这些反馈在下一轮迭代中会被传给 Proposer，形成 **闭环学习**。

---

### Fallback 机制: `_mutate_with_fallback()`

**`runner.py:595-630`**

当 proposer 因 context 过长失败时，逐级加大截断:

```python
async def _mutate_with_fallback(self, parent, failures, iteration):
    max_level = self.config.proposer_max_truncation_level  # 默认 2

    # 尝试逐级截断: level 0 → 1 → 2
    for truncation_level in range(max_level + 1):
        result = await self._mutate(parent, failures, iteration, truncation_level)
        if result is not None:
            return result

    # 终极回退: 只用最短的单个失败
    if self.config.proposer_single_failure_fallback and len(failures) > 1:
        single_failure = self._pick_shortest_failure(failures)
        result = await self._mutate(parent, [single_failure], iteration, max_level)
        if result is not None:
            return result

    return None  # 所有尝试均失败
```

`_pick_shortest_failure()` 选择 trace 最短的失败案例:

```python
def _pick_shortest_failure(self, failures):
    shortest = failures[0]
    shortest_len = len(shortest[0].summarize())
    for failure in failures[1:]:
        length = len(failure[0].summarize())
        if length < shortest_len:
            shortest = failure
            shortest_len = length
    return shortest
```

---

## 四、Git 版本管理系统

**文件**: `src/registry/manager.py`

每个 "program" 对应一个 git branch:

```
program/base         ← 初始版本
program/iter-skill-1 ← 第1轮 skill 变异
program/iter-skill-2 ← 第2轮 skill 变异
...
```

关键操作:

| 操作 | Git 命令 | 位置 |
|------|---------|------|
| 创建 program | `git checkout -b program/{name}` | `create_program()` |
| 切换 program | `git checkout program/{name}` (自动 stash) | `switch_to()` |
| 标记 frontier | `git tag frontier/{name}` | `mark_frontier()` |
| 淘汰 program | `git branch -D program/{name}` + `git tag -d frontier/{name}` | `discard()` |
| 提交变更 | `git add . && git commit -m ...` | `commit()` |

Frontier 选择策略:

```python
def select_from_frontier(self, strategy, iteration=0):
    scored = self.get_frontier_with_scores()  # 按分数降序
    if strategy == "random":
        return random.choice(scored)[0]
    if strategy == "round_robin":
        return scored[iteration % len(scored)][0]
    # "best" (默认): 返回最高分
    return scored[0][0]
```

---

## 五、Skill 复用机制

生成的 skill 以文件形式存储在 `.claude/skills/{skill-name}/SKILL.md`。

Base agent 通过 `setting_sources=["user", "project"]` 配置:

```python
# src/agent_profiles/skill_generator/skill_generator.py
skill_generator_options = ClaudeAgentOptions(
    setting_sources=["user", "project"],  # 自动加载项目 skills
    ...
)
```

Claude Code SDK 会自动扫描 `.claude/skills/` 目录:

1. **Metadata 层**: 所有 skill 的 `name` + `description` 始终在 context 中 (~100 词/skill)
2. **Body 层**: 当 agent 判断某 skill 与当前任务相关时，加载其 markdown body (<5k 词)
3. **Resource 层**: 按需读取 scripts/references/assets (无限制，scripts 可不加载直接执行)

---

## 六、Prompt Only 模式 (对比参考)

### Prompt Proposer

**文件**: `src/agent_profiles/prompt_proposer/prompt.py`

与 Skill Proposer 类似，但输出的是 prompt 修改建议:

```python
class PromptProposerResponse(BaseModel):
    proposed_prompt_change: str  # 行为变更描述
    justification: str
```

### Prompt Generator

**文件**: `src/agent_profiles/prompt_generator/prompt.py`

核心特色 — **反过拟合规则 (Abstraction Ladder)**:

```
BAD:  "Use np.std(data, ddof=1) for standard deviation"
BETTER: "Use sample standard deviation (n-1) for inferential statistics"
BEST: "Choose statistical methods appropriate for your sample type and inference goals"
```

原则: **Prompts guide HOW to think, not WHAT to calculate.**

传给 generator 的 query:

```python
def build_prompt_query_from_prompt_proposer(proposer_trace, original_prompt):
    return f"""## Original Prompt
{original_prompt}

## Proposed Change
{proposer_trace.output.proposed_prompt_change}

## Justification
{proposer_trace.output.justification}"""
```

生成的新 prompt 写入 `src/agent_profiles/base_agent/prompt.txt`，base agent 每次 `run()` 时从磁盘重新读取。

---

## 七、端到端数据流总结

```
┌─────────────────┐
│  训练集问题池     │ train_pools: dict[category → list[(question, answer)]]
│  (按类别分组)     │
└────────┬────────┘
         │ 轮询采样 (round-robin across categories)
         ▼
┌─────────────────┐
│  Base Agent      │ 对每个问题调用 agents.base.run(question)
│  尝试回答        │ → AgentTrace[AgentResponse]
└────────┬────────┘
         │ 评分 (score < 0.8 为失败)
         ▼
┌─────────────────┐
│  build_proposer_ │ 截断 trace (3级)
│  query()         │ 整理 feedback history
│  消息预处理      │ 列举现有 skills
│                  │ 组装结构化 query
└────────┬────────┘
         ▼
┌─────────────────┐
│ Skill Proposer   │ 分析多个失败 trace
│ (只读 agent)     │ 强制使用 brainstorming skill
│                  │ 输出: action, proposed_skill, justification
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Skill Generator  │ 读取 skill-creator 元技能
│ (读写 agent)     │ 创建/编辑 skill 文件
│                  │ 输出: generated_skill, reasoning
│                  │ 副作用: 在磁盘写入 .claude/skills/{name}/SKILL.md
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ git commit       │ 提交所有变更到子分支
│ + 评估           │ 在验证集上评估新版本
│ + Frontier 管理  │ 加入/淘汰
│ + 反馈记录       │ 写入 feedback_history.md → 下轮 proposer 可见
└─────────────────┘
```

---

## 八、关键设计要点

1. **多失败模式分析**: Proposer 一次接收多个失败案例(跨类别)，避免过拟合到单个案例
2. **渐进截断回退**: context 爆炸时自动降级 (60k → 20k → 5k → 单个最短失败)
3. **闭环反馈**: feedback_history.md 记录每轮结果 (IMPROVED/DISCARDED)，下轮 proposer 可学习
4. **Edit vs Create**: Proposer 必须先检查现有 skills，优先 edit 而非重复创建
5. **Git 版本树**: 每个变异是独立 branch，Frontier 保留 top-N，淘汰直接删 branch
6. **元技能驱动**: Generator 不是硬编码流程，而是通过读取 `skill-creator` 元技能来学习如何创建 skill
7. **Brainstorming 强制**: Proposer 必须先执行 brainstorming skill，探索 2-3 种方案后再决定
