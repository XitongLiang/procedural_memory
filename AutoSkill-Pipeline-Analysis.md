# AutoSkill 消息处理与程序记忆提取全流程分析

> 基于 https://github.com/ECNU-ICALK/AutoSkill 源码（2026-04 版本）

---

## 一、Embedded vs Sidecar 模式

AutoSkill 在 OpenClaw 场景下有两种运行模式。**Embedded 是精简版，Sidecar 是完整版**。

### 1.1 架构差异

```
Embedded（默认）：100% JS，跑在 OpenClaw 进程内，无 Python 参与
  adapter/index.js → embedded_runtime.js → 直接读写 SKILL.md

Sidecar：JS 薄客户端 + Python HTTP 服务
  adapter/index.js → HTTP POST → service_runtime.py (port 9100) → AutoSkill SDK
```

### 1.2 功能差异

**Embedded = Sidecar 减去 4 个 LLM 模块**

| 模块 | Sidecar（完整版） | Embedded（精简版） |
|---|---|---|
| **查询改写** | LLMQueryRewriter（LLM 调用） | 没有，原始 query 直接搜 |
| **检索** | 向量 + BM25 混合（0.9:0.1） | 纯 BM25（JS 实现，含 CJK bigram） |
| **Skill 选择** | LLMSkillSelector 二次筛选 | 没有，BM25 top-k 直接用 |
| **提取 Prompt** | 完整 4000 字（8 章节） | 精简版（共享 Prompt 块） |
| **维护(decide/merge)** | 向量搜索 + LLM 4 维判断 | BM25 搜索 + LLM 判断 |
| **使用审计** | retrieved→relevant→used 闭环 | 没有 |
| **提取时机** | 每轮滑动窗口（`messages[-6:]`，延迟 1 轮） | 会话结束 + 每 5 轮 live checkpoint |
| **提取执行** | Python 后台线程异步 | hook 回调内同步 |

### 1.3 LLM 调用次数

```
Sidecar 每轮：5-6 次 LLM 调用
  查询改写(1) + embed(1) + 选择(1) + 提取(1) + 维护decide(1) + 审计(1)

Embedded 每次提取：2-3 次 LLM 调用
  提取(1) + 维护decide(1) + 可能的merge(1)
```

### 1.4 Embedded 默认配置

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `runtimeMode` | `"embedded"` | 默认 Embedded 模式 |
| `extractOnAgentEnd` | `true` | agent_end 时检查是否提取 |
| `successOnly` | `true` | 仅成功轮次 |
| `liveExtractEveryTurns` | `5` | 每 5 轮 live checkpoint（0=关闭） |
| `sessionMaxTurns` | `20` | 20 轮强制关闭 session |
| `bm25TopK` | `8` | BM25 返回 top 8 候选 |
| `skillRetrieval.enabled` | `false`（Embedded 下自动关闭） | 检索注入默认关闭 |
| `maxInjectedChars` | `1500` | 注入 Skill 上下文最大字符数 |

### 1.5 核心共同点

两者的**提取 → decide → merge/add** 管线是一样的：
- 共享同一套 Prompt 模板（`openclaw_prompt_pack.txt`）
- 同样的 add / merge / discard 三种决策
- 同样的 SKILL.md 输出格式
- 同样的 Mirror 到 `~/.openclaw/workspace/skills/`

**以下内容以 Sidecar（完整版）为主线展开。**

---

## 二、Sidecar 完整走线

### 2.1 整体流程

```
用户发送消息
    │
    ▼
[Hook: before_prompt_build]  ──── TS 层 ────────────────────────
    │
    ├── 消息归一化（从 OpenClaw 事件中提取统一格式）
    ├── 提取 session_id, user_id, query
    │
    ├── POST /v1/autoskill/openclaw/ ──────► [Python: before_agent_start]
    │   hooks/before_agent_start             │
    │                                        ├── [1] 查询改写
    │                                        ├── [2] 向量+BM25 混合检索
    │                                        ├── [3] LLM Skill 选择
    │                                        └── [4] 渲染 Skill 上下文
    │   ◄──────────────────────────────────── 返回 { injection_block }
    │
    ├── 缓存检索结果（给 agent_end 用）
    └── 返回 { appendSystemContext: injection_block }
    │
    ▼
OpenClaw 处理轮次（LLM 调用、工具使用）
    │
    ▼
[Hook: agent_end]  ──── TS 层 ──────────────────────────────────
    │
    ├── 过滤（successOnly）
    ├── 消费检索缓存
    ├── 收集 used_skill_ids
    │
    ├── POST /v1/autoskill/openclaw/ ──────► [Python: agent_end]（后台线程）
    │   hooks/agent_end                      │
    │   (fire-and-forget)                    ├── [5] 弹出上一轮 pending → 提取
    │                                        ├── [6] 维护（add/merge/discard）
    │                                        ├── [7] 使用审计
    │                                        └── [8] Skill 镜像
    └── 继续
```

### 2.2 TS 层：Hook 注册

```javascript
// adapter/index.js
export default {
  id: "autoskill-openclaw-adapter",
  name: "AutoSkill OpenClaw Adapter",
  kind: "lifecycle",
  register(api) {
    api.on("message_received",    handler);  // 纯诊断，仅记日志
    api.on("before_prompt_build", handler);  // 检索 + 注入
    api.on("agent_end",           handler);  // 提取 + 维护
  }
}
```

TS 层在 Sidecar 模式下是**薄客户端**，只做：
- 消息归一化（从各种 OpenClaw 事件格式提取统一结构）
- HTTP POST 转发到 Python
- 注入块格式化

---

## 三、Step 1 — 查询改写

文件：`autoskill/interactive/rewriting.py`

```
用户消息 + 最近 6 轮历史（max 2000 chars）
  → LLMQueryRewriter.rewrite()
  → 一行检索 query
```

**作用**：把用户当前消息改写为检索友好的独立查询，解决代词引用（"it"/"that"/"上面的"）、继承话题锚点。

**Prompt**：

```
You are AutoSkill's retrieval query rewriter.
Task: rewrite the current user query into ONE concise search query.

### DECIDE CONVERSATION STATE FIRST
- State A (continuation): current turn updates the same task.
- State B (topic switch): current turn starts a different task.
- If unsure, default to State A.

### PRIMARY OBJECTIVE: topic-anchored, constraint-aware rewrite
- Build standalone query preserving task identity.
- State A: keep existing anchor + add new constraints.
- State B: replace anchor with new task/topic.
- Resolve references ('it/that/this/above') using history.

### TOPIC ANCHOR SPEC (CRITICAL)
- Must include explicit topic anchor noun phrase.
- Anchor = <domain/object> + <deliverable form> + <operation intent>.
- Constraint-only turn → prepend inherited anchor.
- Do not collapse anchor into generic labels like 'document', 'content'.

### WHAT TO KEEP
- Core task/topic + operation + durable constraints.
- Constraints that change skill choice: output format, tone, audience, depth.

### WHAT TO AVOID
- Do NOT replace topic with generic process words.
- Do NOT include long pasted content.
- Do NOT include one-off entities.

### QUALITY BAR
- One line, concise but specific.
- Pattern: <topic anchor> + <operation> + <key constraints>.

Output ONLY the rewritten query text. No quotes, no preamble.
```

**参数**：`temperature=0.0`，max 256 chars。

**示例**：
```
历史: "Write a government report on LLM self-evolution" + 后续约束
当前: "Do not use tables, use Word format."
输出: "政府报告 大模型自进化 不使用表格 Word格式"
```

---

## 四、Step 2 — Skill 检索

文件：`autoskill/interactive/retrieval.py`

```
改写后的 query
  → embed(query) → 向量搜索 (cosine similarity)
  → BM25 全文检索
  → 混合得分 = 0.9 * 向量得分 + 0.1 * BM25 得分
  → 分 scope 搜索（user scope + library scope）
  → 按 min_score 过滤（默认 0.4）
  → 合并排序 → 返回 top-k（默认 3）
```

---

## 五、Step 3 — LLM Skill 选择

文件：`autoskill/interactive/selection.py`

从检索结果中让 LLM 选出真正要注入的 Skill。

**Prompt**：

```
You are AutoSkill's skill selector for retrieval augmentation.
Task: decide which retrieved Skills should be injected into context.
Be conservative: select only clearly relevant Skills.
It is OK to select none.

Rules:
- Only select from provided skill IDs.
- Prefer user skills over library skills.
- Do not select generic or unrelated skills.
- Consider resource_paths as utility signals only when directly helpful.
- Select at most max_selected.

Output ONLY strict JSON:
{"use_skills": true|false, "selected_skill_ids": ["..."], "reason": "..."}
```

**参数**：`temperature=0.0`。

**降级**：LLM 失败时用关键词重叠率 heuristic 选择。

---

## 六、Step 4 — 渲染注入

文件：`autoskill/render.py`

```python
def render_skills_context(skills, query, max_chars=6000):
    header = (
        "## AutoSkill Skills\n"
        "Instructions: Choose the most relevant skill and follow its prompt; "
        "ignore if none applies.\n"
        f"Query: {query}\n"
    )
    for i, skill in enumerate(skills):
        block = f"""
### Skill {i+1}: {skill.name} (v{skill.version})
- Id: {skill.id}
- Description: {skill.description}
- Tags: {', '.join(skill.tags)}
- Triggers:
{chr(10).join('- ' + t for t in skill.triggers)}
- Resources: {', '.join(resource_paths)}
- Prompt:
{skill.instructions}
"""
        if len(header + block) > max_chars:
            break
        header += block
    return header
```

注入到助手的 system prompt：

```python
system = (
    "You are a helpful assistant.\n"
    "You have access to a list of retrieved Skills below.\n"
    "**CRITICAL:** These Skills are retrieved automatically and may be irrelevant.\n"
    "1. **Evaluate:** Use a Skill ONLY if it directly matches the user's current intent.\n"
    "2. **Ignore:** If unrelated, IGNORE THEM COMPLETELY and answer normally.\n"
    "3. **Silence:** Do not mention the existence of these Skills.\n\n"
    f"Skills Context:\n{context}"
)
```

---

## 七、Step 5 — Skill 提取

文件：`autoskill/management/extraction.py`

### 7.1 提取时机：滑动窗口 + 延迟一轮

```
轮N:  user_N → assistant_N → 暂存为 pending（包含 messages 快照）
轮N+1: user_{N+1} 进来 → 弹出 pending → 取 messages[-6:] → 后台提取
```

每次提取的输入是最近 6 条消息（≈3 轮对话），窗口随每轮右移 2 条（1 user + 1 assistant），有 4 条重叠。

### 7.2 构建 payload

```python
payload = {
    "user_id": user_id,
    "messages": messages,
    "primary_user_questions": [仅 user 角色的消息文本],   # 主要证据
    "full_conversation": [完整对话格式化文本],              # 仅参考
    "max_candidates": 1,                                  # 默认每次最多提取 1 个
    "hint": None,                                         # 用户显式请求时的提示
    "retrieved_reference": top1_hit                        # 检索到的最相关已有 Skill
}
```

### 7.3 提取 Prompt（完整，约 4000 字）

```
You are AutoSkill's Skill Extractor.
Task: Extract reusable, executable Skills from messages/events.
Quality-first: extract when reusable signal is reasonably clear,
and skip only clearly weak or one-off candidates.
User-reuse-first: prefer extraction only when the skill is likely
to be reused by this same user in future similar tasks.
Input contract: DATA.primary_user_questions is the main extraction evidence;
DATA.full_conversation is context reference only.

### 1) EVIDENCE AND PROVENANCE (CRITICAL)
- Prioritize DATA.primary_user_questions (USER inputs) as primary evidence.
- ASSISTANT/model replies are reference-only, not extraction evidence.
- USER turns are source of truth; ASSISTANT turns are supporting context only.
- Do not extract requirements that appear only in assistant output
  unless user requested/confirmed/corrected/reinforced them.
- Weak acknowledgements ('ok', 'continue', 'sounds good')
  do not validate all assistant details.
- Every major requirement must be traceable to user evidence.
- Reject assistant-invented specifics unless user explicitly requested/approved.
- If evidence is ambiguous or mostly assistant-authored → {"skills": []}.

### 2) RECENCY, TOPIC COHERENCE, AND RETRIEVED REFERENCE (CRITICAL)
- Prioritize recent user rounds; older context mainly for continuity judgment.
- Use latest 3-6 user messages to detect intent/topic/task shifts.
- Infer current work item = objective + deliverable/channel + operation class.
- Boundary rule: detect the first recent user turn that introduces
  a new objective/deliverable/channel/task as the boundary turn.
- If boundary exists → post-boundary intent is authoritative.
- If no boundary → continue using broader context.
- If multiple work items → keep only the latest active one.
- Never mix constraints from different work items.
- DATA.retrieved_reference is only identity context, never extraction evidence.
- SAME capability requires alignment on objective + deliverable/channel
  + operation class + acceptance criteria.

### 3) WHEN TO EXTRACT
- DO NOT EXTRACT for:
  one-shot/generic tasks, trivial logic, assistant-only requirements,
  stale requirements, low reuse value.
- EXTRACT when user provides reusable constraints/policies:
  - One explicit reusable constraint can be sufficient
  - Corrections/interventions or must-do requirements
  - Implementation/output policy (language/tooling, format, safety bounds, SOP steps)
  - Single-turn explicit policy change
  - Revision feedback that changes generation policy
    (detail, tone, audience fit, structure strictness)
  - Explicit multi-step AI operation sequence (workflow/SOP)
  - Schema/template requirements
- Decision boundary: WHAT (content) → don't extract; HOW (process) → extract.

### 4) CONSTRAINTS OVER CONTENT + DE-IDENTIFICATION (CRITICAL)
- Focus on HOW, not this-instance WHAT.
- Capture both must-do and must-not-do constraints.
- Abstract specific feedback into reusable rules.
- Keep capability invariants (workflow/constraints/output contract/quality checks).
- Drop payload (topic facts, domain claims, project details).
- Use placeholders: <SOURCE_CONTENT>, <TARGET_TOPIC>, <KEY_POINTS>.
- Remove case-specific entities (org/person/URL/date/budget/contract).
- If de-identification removes core value → {"skills": []}.
- Reuse test: can it transfer to a similar different topic?

### 5) NO INVENTION + WORKFLOW RULE (CRITICAL)
- Extract only logic supported by conversation evidence.
- # Workflow only when user explicitly specifies multi-stage operations.
- Do not invent unrequested standards, thresholds, or specs.

### 6) OUTPUT REQUIREMENTS
- Strict JSON: {"skills": [...]} with at most {max_candidates} items.
- Language consistency: match user instruction language.
- De-identify aggressively.

### 7) FIELD DEFINITIONS
- name: kebab-case, encode intent + action + domain, self-explanatory.
- description: 1-2 sentences, third person, WHAT + WHEN.
- prompt: executable Markdown:
  1) # Goal (required)
  2) # Constraints & Style (required, include must-do and must-not-do)
  3) # Workflow (optional, only for explicit multi-stage operations)
- triggers: 3-5 short intent phrases.
- tags: 1-6 keywords.
- confidence: 0.0-1.0.

### 8) MINDSET QUICK CHECKS
- Is this reusable user-specific logic, not generic baseline ability?
- Is there reusable method/policy/process?
- Likely to be reused by this same user?
- A single constraint can be enough.
- Did user feedback revise generation policy?
- If not → {"skills": []}.
```

### 7.4 提取输出格式

```json
{
  "skills": [
    {
      "name": "government-report-writing-policy",
      "description": "Generates government-style reports with specific formatting constraints. Use when writing formal reports for government audiences.",
      "prompt": "# Goal\nGenerate a structured government report on <TARGET_TOPIC>...\n\n# Constraints & Style\n- Must use formal tone\n- Do not use tables\n- Word format only...",
      "triggers": ["write government report", "draft formal policy document"],
      "tags": ["government", "report", "formal-writing"],
      "confidence": 0.85
    }
  ]
}
```

### 7.5 JSON 解析容错

```
LLM 返回文本
  ├── 直接 JSON 解析 → 成功 → 继续
  │
  └── 失败 → 三级降级：
        ├── [1] 从半结构化文本恢复 name/description/prompt/triggers/tags
        ├── [2] repair prompt（第二次 LLM 调用修复 JSON）
        └── [3] 仍失败 → 放弃本次提取
```

---

## 八、Step 6 — 维护（add / merge / discard）

文件：`autoskill/management/maintenance.py`

### 8.1 决策流程

```
候选 Skill
  │
  ├──[0] identity hash 快速匹配
  │       normalize(candidate.description) == normalize(existing.description)
  │       → 匹配 → 直接跳到 [3B] merge
  │
  ├──[1] 向量相似度搜索（user scope + library scope）
  │
  ├──[2] LLM 决策（Skill Set Manager）
  │       发送 candidate + similar_skills → LLM
  │       │
  │       ├── Guardrail：LLM 说 add 但向量相似度 > 阈值 → 强制走 merge
  │       │
  │       └── 返回 add / merge / discard
  │
  ├──[3A] ADD → 创建新 Skill → 生成 SKILL.md → 写入存储
  │
  ├──[3B] MERGE
  │       │
  │       ├── [3B-1] 合并判断（Capability Identity Judge）← 保险：二次确认
  │       │          "候选和目标 Skill 真的是同一个能力吗？"
  │       │          → same_capability=true  → 继续合并
  │       │          → same_capability=false → 改为 [3A] ADD 新建
  │       │
  │       └── [3B-2] 合并执行（Skill Merger）
  │                  发送 existing + candidate → LLM 语义合并
  │                  → 版本号 +1 → 覆写 SKILL.md
  │
  └──[3C] DISCARD → 丢弃
```

### 8.2 维护决策 Prompt

```
You are AutoSkill's Skill Set Manager.
Task: decide how to handle a newly extracted candidate skill.

### Decision Procedure (follow in order)
0) Capability-overlap hard gate
   SAME capability → MUST NOT add → only merge or discard.
0.1) Name-collision hard gate
   Same normalized name → MUST NOT add.
1) Topic continuity and family check
   Different capability families → add.
2) Discard gate
   Generic / low-signal / library covers it → discard.
3) Capability identity test (4 axes)
   a. core job-to-be-done
   b. output contract / deliverable type
   c. hard constraints and success criteria
   d. required context / tools / workflow
   Hard non-merge: deliverable class/channel or audience differ.
4) Merge vs add
   SAME capability → merge; topic broken → add.
5) Tie-breakers
   Unclear → add; same-capability overlap → merge/discard.

Return: {"action": "add"|"merge"|"discard", "target_skill_id": ..., "reason": ...}
```

### 8.3 合并判断 Prompt

当决策为 merge 前，先做能力身份判断：

```
You are AutoSkill's capability identity judge.
Task: decide whether candidate should UPDATE/MERGE existing or be NEW.

### Decision Principles
- same_capability=true: same job-to-be-done + mainly iteration.
- same_capability=false: different deliverable/audience/criteria.
- Hard split: deliverable class changes, or audience changes.
- Wording/naming changes → NOT a new capability.
- Policy upgrades → same-capability refinement.

### Recency and Continuity
- Candidate = recent intent; existing = historical memory.
- Work item shifts → same_capability=false.
- Unclear continuity → default false.

Return: {"same_capability": true|false, "confidence": 0.0-1.0, "reason": "..."}
```

### 8.4 合并执行 Prompt

新旧 Skill 的原始字段**不做预处理，直接喂给 LLM**：

```
发给 LLM：

system: （合并 Prompt，见下）

user: {
  "existing_skill": {
    "name": "报告格式规范",
    "description": "...",
    "prompt": "# Goal\n...\n# Constraints & Style\n- 正式语气\n- 不用表格",
    "triggers": ["write report", "draft document"],
    "tags": ["report", "formal"]
  },
  "candidate_skill": {
    "name": "报告撰写规范",
    "description": "...",
    "prompt": "# Goal\n...\n# Constraints & Style\n- 正式语气\n- Word格式\n- 先写结论",
    "triggers": ["write report", "create formal report"],
    "tags": ["report", "writing"]
  }
}
```

**合并 System Prompt**：

```
You are AutoSkill's Skill Merger.
Task: Merge existing + candidate into ONE improved Skill.

### Merge Principles
- Shared intent: keep the same capability.
- Diff-aware: semantic union, NOT concatenation.
- Avoid regressions: don't drop existing constraints.

### Anti-Duplication (CRITICAL)
- Never duplicate section headers.
- Near-duplicate phrases → keep one canonical phrasing.

### De-identification (CRITICAL)
- Remove case-specific entities.
- Keep portable capability rules.

### No-Invention
- Use ONLY information from existing + candidate.

### Field Definitions
- name: keep existing unless new clearly improves retrieval.
- prompt: # Goal → # Constraints & Style → # Workflow (optional).
- triggers: 3-5, semantic union then dedupe.
- tags: 1-6, semantic union, canonical only.

Return: {name, description, prompt, triggers, tags}
```

**LLM 返回（合并结果）**：

```json
{
  "name": "报告格式规范",
  "description": "...",
  "prompt": "# Goal\n...\n# Constraints & Style\n- 正式语气\n- 不用表格\n- Word格式\n- 先写结论",
  "triggers": ["write report", "draft document", "create formal report"],
  "tags": ["report", "formal", "writing"]
}
```

**降级路径**：LLM 合并失败时走 heuristic merge——不融合，直接比较两边指令质量分，二选一取更好的那份，triggers/tags 拼接后 lowercase 去重，版本号 +1。

### 8.5 滑动窗口与去重

由于滑动窗口有 4 条消息重叠，相邻轮次可能提取出相似的候选 Skill。维护管线负责去重：

```
轮5: 提取 → 新建 "报告格式规范" v0.1.0
轮6: 提取 → 候选相似 → merge → v0.1.1（可能补充新约束）
轮7: 提取 → 候选相似 → discard（没有新信息）
```

重复提取不会产生重复 Skill，反而是一个**渐进进化机制**——每次窗口多 2 条新消息，可能捕获新约束，通过 merge 逐步完善。

---

## 九、Step 7 — 使用审计

文件：`autoskill/interactive/usage_tracking.py`

在助手回复生成后，后台审计检索到的 Skill 是否真的被用到。

**Prompt**：

```
You are AutoSkill's skill-usage auditor.
Task: for each skill id, judge CURRENT query + reply pair:
- relevant: skill matches current user intent?
- used: reply actually depends on this skill's constraints?
Rules:
- used=true only when relevant=true.
- If reply can be produced without this skill → used=false.
- Uncertain → false.

Return: {"skills":[{"id":"...","relevant":bool,"used":bool,"reason":"..."}]}
```

审计结果更新三个计数器：

```
每个 Skill 维护：
  retrieved_count   ← 被检索到的次数
  relevant_count    ← 被判为相关的次数
  used_count        ← 被判为实际使用的次数

自动清理规则：
  retrieved_count >= 40 && used_count == 0 → 该 Skill 被自动删除
```

---

## 十、Step 8 — Skill 存储与镜像

### 10.1 SKILL.md 格式

```markdown
---
id: abc-123-def
name: government-report-writing-policy
description: >
  Generates government-style reports with formatting and tone constraints.
  Use when writing formal reports for government audiences.
version: "0.2.0"
tags: [government, report, formal-writing]
triggers:
  - write government report
  - draft formal policy document
  - create official report
---

# government-report-writing-policy

Generates government-style reports...

## Prompt

# Goal
Generate a structured government report on <TARGET_TOPIC>...

# Constraints & Style
- Must use formal tone
- Do not use tables
- Word format only
- Lead with executive summary

## Triggers
- write government report
- draft formal policy document

## Examples
### Example 1
**Input:** Write a report on AI policy
**Output:** A structured government report...
```

### 10.2 文件系统结构

```
~/.openclaw/autoskill/
  └── SkillBank/
      ├── Users/
      │   └── {user_id}/
      │       ├── government-report-writing-policy/
      │       │   ├── SKILL.md
      │       │   ├── scripts/validate.py       (可选)
      │       │   └── references/template.md    (可选)
      │       └── deploy-docker-app/
      │           └── SKILL.md
      ├── Common/
      │   └── skill-creator/
      │       └── SKILL.md
      └── vectors/                 ← 持久化向量索引

~/.openclaw/workspace/skills/      ← Mirror 目标（OpenClaw 原生识别）
  ├── government-report-writing-policy/
  │   └── SKILL.md
  └── deploy-docker-app/
      └── SKILL.md
```

Mirror 机制：每次 add/merge 后，自动把 Skill 目录复制到 `~/.openclaw/workspace/skills/`，让 OpenClaw 原生识别。

---

## 十一、离线自进化（SkillEvo）

文件：`SkillEvo/runner.py`、`SkillEvo/mutators.py`、`SkillEvo/evals.py`

SkillEvo 是一个**独立的离线工具**，不在对话流程中触发，需要手动或定时批处理运行。它是唯一能**主动改进 Skill 质量**的机制——前面的在线管线只做增删合并，不会让一个 Skill 本身变得更好。

### 11.1 三个 LLM 角色

SkillEvo 用三个独立的 LLM 实例（可分别配置不同模型）：

| 角色 | 配置项 | 做什么 |
|---|---|---|
| **Responder** | `config.llm` | replay 时用指令生成回复 |
| **Mutator** | `config.mutator_llm` | 根据失败样本改进指令 |
| **Judge** | `config.judge_llm` | 评估回复是否满足规则 |

### 11.2 总体流程

```
[Step 1] 收集样本 + 分集                                        
    │
    ▼
[Step 2] 编译评估规则                                           
    │
    ▼
[Step 3] 第一轮 replay：用 champion 跑 dev 集        🤖 Responder LLM
    │
    ▼
[Step 4] 评估打分，找出失败样本
    ├── 程序化规则（代码检查）                        
    └── LLM 判断规则                                 🤖 Judge LLM
    │
    ▼
[Step 5] 生成变体
    ├── Heuristic 变异（机械拼接）                    
    └── LLM 变异                                     🤖 Mutator LLM
    │
    ▼
[Step 6] 第二轮 replay：用每个变体跑 dev 集          🤖 Responder LLM
    │
    ▼
[Step 7] 评估打分，选出最优变体
    ├── 程序化规则（代码检查）                        
    └── LLM 判断规则                                 🤖 Judge LLM
    │
    ▼
[Step 8] 第三轮 replay：最优变体 vs champion 跑 test  🤖 Responder LLM
    │
    ▼
[Step 9] 晋升判断（纯数值比较）                       
    │
    ├── 通过 → promote 为新 champion → 写入 SKILL.md
    └── 不通过 → 保留原 champion
```

---

### 11.3 Step 1 — 收集样本 + 分集

从在线管线的**使用审计记录**中找到关联这个 Skill 的历史对话：

```
Skill "报告格式规范" 的使用审计记录：
  对话1: query="写AI政策报告"       → used=true  ✅ 收集
  对话2: query="帮我翻译一段话"      → used=false ❌ 无关
  对话3: query="写教育改革报告"      → used=true  ✅ 收集
  对话4: query="写预算分析报告"      → used=true  ✅ 收集
  对话5: query="写人才规划报告"      → used=true  ✅ 收集
  对话6: query="debug代码"          → used=false ❌ 无关

收集到: [对话1, 对话3, 对话4, 对话5]
```

随机分成两组（类似机器学习的 train/test split）：

```
mutate_dev 集:     [对话1, 对话4]     ← 用于评估变体 + 找失败样本
promotion_test 集: [对话3, 对话5]     ← 用于最终晋升对比（变体没见过）
```

分两个集是为了**防止过拟合**——变体在 dev 上针对性优化过，必须在没见过的 test 上也表现好才能晋升。

如果总样本 < `min_replay_samples`，或 dev/test 任一为空 → 跳过 SkillEvo，状态标记为 `"incubating"`。

---

### 11.4 Step 2 — 编译评估规则

从 Skill 内容自动生成评估规则（**不用 LLM**，纯代码逻辑）：

**程序化规则**（代码直接检查）：

| 规则 | 检查方式 | hard/soft |
|---|---|---|
| `response_nonempty` | 回复非空？ | hard |
| `json_parseable` | JSON 可解析？ | soft |
| `markdown_table` | 含表格？ | soft |
| `paragraph_limit` | 段落数 <= N？ | soft |
| `must_cite_sources` | 包含引用/链接？ | soft |
| `lead_with_conclusion` | 第一段是结论？ | soft |

**LLM 判断规则**（需要 Judge LLM 评估）：

| 规则 | 判断方式 | hard/soft |
|---|---|---|
| `no_unfounded_claims` | LLM 判断有无无据断言 | soft |
| `requirement_satisfaction` | LLM 判断是否满足需求 | soft |

**评分权重**：通过 hard 规则 +2.0 分，通过 soft 规则 +1.0 分，未通过 +0.0 分。

---

### 11.5 Step 3 — 第一轮 replay：champion 跑 dev 集

用 champion 的指令当 system prompt，对 dev 集中的每个历史 query 重新生成回复：

**Replay Prompt**（Responder LLM）：

```
system: {champion 的 instructions 原文，如：
  # Goal
  生成政府报告
  # Constraints & Style
  - 正式语气
  - 不用表格
}

user:
  Replay the following conversation with the skill instructions already injected.

  Conversation:
  [user] 帮我写一篇AI政策报告

  [assistant] 好的...

  [user] 要有数据支撑

  Respond to the latest user message.
```

**参数**：`temperature=0.0`

注意：这里不是回放原始回复，是让 LLM **用 champion 指令重新生成一遍回复**。

---

### 11.6 Step 4 — 评估打分，找出失败样本

对每个 replay 回复，逐条跑评估规则：

**程序化规则**直接代码检查：

```
样本1 replay 回复: "根据教育部数据(来源:xxx)，..."
  → response_nonempty: ✅ pass (+2.0)
  → must_cite_sources: ✅ pass (+1.0)  有引用
  → lead_with_conclusion: ✅ pass (+1.0)  先写了结论
  → 总分 4.0
```

**LLM 判断规则**用 Judge LLM 评估：

```
system:
  You are a strict binary evaluator for skill replay results.
  Output ONLY strict JSON parseable by json.loads.
  Schema: {"pass": true|false, "reason": "short reason"}
  Judge only against the requirement provided.
  Prefer false if the requirement is not clearly satisfied.

user: {
  "requirement": "Avoid unfounded claims and state uncertainty when needed.",
  "skill_name": "报告格式规范",
  "latest_user_message": "帮我写教育改革报告",
  "response": "（Step 3 replay 生成的完整回复文本）"
}
```

**参数**：`temperature=0.0`

**汇总结果**：

```
样本1: query="写AI政策报告"   → 总分 8.5 ✅
样本2: query="写教育改革报告"  → 总分 4.2 ❌ 失败（没引用来源）
样本3: query="写预算分析报告"  → 总分 5.1 ❌ 失败（结论没写在前面）

champion 在 dev 集平均分: 5.9
失败样本: [样本2, 样本3]
```

---

### 11.7 Step 5 — 生成变体

根据评估规则和失败样本，生成多个变体指令。

#### 方式一：Heuristic 变异（不用 LLM）

根据评估规则机械地在指令后面附加约束文本：

```
原始指令:
  # Constraints & Style
  - 正式语气
  - 不用表格

Heuristic 变异后:
  # Constraints & Style
  - 正式语气
  - 不用表格

  ## SkillEvo Mutation Guards
  - Always cite concrete sources or links for factual claims.
  - Open with a direct conclusion or answer before elaboration.
```

每条规则对应的固定文本：

| 规则 | 附加约束 |
|---|---|
| `must_cite_sources` | `Always cite concrete sources or links for factual claims.` |
| `paragraph_limit` | `Keep the final answer within {n} short paragraphs.` |
| `lead_with_conclusion` | `Open with a direct conclusion or answer before elaboration.` |
| `json_parseable` | `Output valid JSON only, with no markdown fences or extra commentary.` |
| `markdown_table` | `Include a markdown table when presenting the final answer structure.` |
| `no_unfounded_claims` | `Do not invent facts; if evidence is missing, state uncertainty explicitly.` |

#### 方式二：LLM 变异（用 Mutator LLM）

把 champion 指令 + 评估规则 + 失败样本喂给 Mutator，让它针对性改进：

```
system:
  You improve a skill prompt for replay evaluation.
  Output ONLY strict JSON parseable by json.loads.
  Schema: {"name": "...", "description": "...", "instructions": "...",
           "tags": ["..."], "triggers": ["..."], "notes": "..."}
  Rules:
  - Keep the same skill identity and user scope.
  - Only make small durable improvements.
  - Strengthen requirements that failed evaluation.
  - Do not invent new capabilities.

user: {
  "base_skill": {
    "name": "报告格式规范",
    "description": "...",
    "instructions": "# Goal\n...\n# Constraints & Style\n- 正式语气\n- 不用表格"
  },
  "eval_rules": [                    ← 最多 4 条
    {
      "rule_id": "must_cite_sources",
      "kind": "programmatic",
      "hard": false,
      "description": "Response must cite sources for factual claims"
    }
  ],
  "failing_samples": [               ← 最多 4 个
    {
      "sample_id": "...",
      "latest_user_message": "帮我写教育改革报告"
    },
    {
      "sample_id": "...",
      "latest_user_message": "帮我写预算分析报告"
    }
  ]
}
```

**参数**：`temperature=0.2`（允许一点创造性）

**注意**：失败样本只给了 `latest_user_message`，没给 replay 的回复内容。Mutator 需要根据评估规则**猜测**回复在哪里失败，然后改进指令。

**两种变体都参与后续评估**：

```
champion（原版）
  ├── heuristic 变体 1（加了 cite_sources）
  ├── heuristic 变体 2（加了 lead_with_conclusion）
  └── LLM 变体（Mutator 针对性改进的）
```

---

### 11.8 Step 6 — 第二轮 replay：变体跑 dev 集

用每个变体的指令替换 champion，在 dev 集上重新 replay（Prompt 格式同 Step 3，只是 system 换成变体指令）。

---

### 11.9 Step 7 — 评估打分，选出最优变体

对每个变体的 replay 结果跑评估规则（同 Step 4），选出得分最高的：

```
champion:       dev 平均分 5.9
heuristic 变体1: dev 平均分 6.8
heuristic 变体2: dev 平均分 6.2
LLM 变体:       dev 平均分 7.6 ← 最优

选出 LLM 变体进入晋升
```

选择逻辑（纯数值，不用 LLM）：最高平均分优先，平分时取硬性失败更少的。

---

### 11.10 Step 8 — 第三轮 replay：最优变体 vs champion 跑 test 集

在**变体从未见过的** test 集上再跑一轮，防止过拟合：

```
champion 在 test 集: 平均分 6.1
LLM 变体在 test 集: 平均分 7.8
```

---

### 11.11 Step 9 — 晋升判断

纯数值比较，不用 LLM：

```python
def _should_promote(champion, candidate):
    if candidate.sample_count <= 0:          # 至少评估了 1 个样本
        return False
    if candidate.average_score < champion.average_score + min_score_delta:  # 明显更好
        return False
    if candidate.hard_failures > champion.hard_failures:  # 硬性失败不能增加
        return False
    return True
```

```
champion: test 平均分 6.1, hard_failures 0
LLM 变体: test 平均分 7.8, hard_failures 0

7.8 >= 6.1 + 0.5 (delta) ✅
0 <= 0 (hard_failures) ✅

→ promote! 写入新版 SKILL.md v0.2.0 + champion.json
```

---

### 11.12 完整示例

```
Skill "报告格式规范" v0.1.0 champion:
  # Constraints & Style
  - 正式语气
  - 不使用表格

[Step 1] 收集 6 个关联对话，分成 dev(3个) + test(3个)

[Step 2] 编译评估规则: response_nonempty(hard), must_cite_sources(soft),
         lead_with_conclusion(soft), no_unfounded_claims(soft/llm)

[Step 3] champion 在 dev 上 replay（3次 Responder LLM 调用）

[Step 4] 评估打分（3次程序化检查 + 3次 Judge LLM 调用）
         平均分 5.9，失败样本: 样本2(没引用), 样本3(结论在最后)

[Step 5] 生成变体:
         heuristic: 加 "Always cite sources" + "Open with conclusion"
         LLM: Mutator 针对样本2/3改进（1次 Mutator LLM 调用）

[Step 6] 2个变体在 dev 上 replay（6次 Responder LLM 调用）

[Step 7] 评估打分（6次检查 + 6次 Judge LLM 调用）
         heuristic 变体: 6.8
         LLM 变体: 7.6 ← 最优

[Step 8] champion + LLM 变体在 test 上 replay（6次 Responder LLM 调用）
         + 评估（6次检查 + 6次 Judge LLM 调用）
         champion: 6.1, LLM 变体: 7.8

[Step 9] 7.8 >= 6.1 + 0.5 → promote!

总计 LLM 调用: 约 15次 Responder + 15次 Judge + 1次 Mutator ≈ 31次
```

---

### 11.13 三种 Skill 维护时机总结

| 时机 | 机制 | 做什么 |
|---|---|---|
| **对话中（在线）** | SkillMaintainer | 提取后立即 add/merge/discard |
| **对话后（在线）** | 使用审计 | 低质量 Skill 自动清理 |
| **离线批处理** | SkillEvo | replay + 变异 + 评估 + 晋升，主动改进 Skill 质量 |

---

## 十二、关键设计洞察

### 12.1 证据溯源原则

贯穿所有 Prompt 的最重要原则：

```
USER turns = 证据来源（source of truth）
ASSISTANT turns = 仅参考上下文（reference only）
弱确认 ("ok", "thanks") ≠ 验证助手的所有细节
```

防止把 LLM 幻觉固化为 Skill。

### 12.2 HOW vs WHAT 决策边界

```
"把报告改成中文"           → WHAT（一次性内容指令）→ 不提取
"以后报告都要先写结论再展开" → HOW（可复用的生成策略）→ 提取
"不要用表格，用 Word 格式"  → HOW（持久的输出约束）  → 提取
"帮我总结这篇论文"         → WHAT（一次性任务）     → 不提取
```

### 12.3 去标识化

```
原始: "把 John 的 Q1 年度报告发到 Slack #reports"
提取: "将 <TARGET_DOCUMENT> 按 <TIME_SEGMENTS> 分段，发布到 <OUTPUT_CHANNEL>"
```

去标识化测试：去掉具体实体后是否还有价值？变成通用建议 → 不提取。

### 12.4 检索-提取闭环

检索到的最相关 Skill 作为 `retrieved_reference` 传给提取器：
- 帮助判断"这是同一个 Skill 的更新还是全新的"
- 避免创建重复 Skill
- 但**不作为提取证据**——只有用户说的话才是证据
