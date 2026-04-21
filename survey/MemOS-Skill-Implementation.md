# MemOS 消息处理与程序记忆提取全流程分析

> 基于 https://github.com/MemTensor/MemOS 源码（2026-04 版本）

---

## 一、TS 端 vs Python 端

MemOS 包含两套独立实现，代码完全独立。

### 1.1 架构差异

```
TS 端：本地嵌入式插件，跑在 OpenClaw 进程内
  apps/memos-local-openclaw/index.ts → 各子模块 → SQLite 本地文件

Python 端：云端/自托管 API 服务
  FastAPI (port 8001) → 多后端存储 → OSS/Local + MySQL + 图谱DB
```

### 1.2 功能差异

| 维度 | TS 端（本地插件） | Python 端（云端服务） |
|---|---|---|
| **运行方式** | 进程内，无网络依赖 | 独立部署，FastAPI |
| **存储** | SQLite（本地文件） | 多后端（OSS/Local + MySQL） |
| **消息处理** | 实时增量（游标机制） | 批量处理（全量消息） |
| **主题分割** | 逐 turn 线性判断（LLM） | 一次性 LLM 聚类（支持非连续回溯） |
| **Skill 提取** | 评估 → 搜索 → 创建/升级（6 个子模块） | 两阶段提取（简单 + 详细） |
| **Skill 进化** | 升级已有 Skill（版本 +1） | 更新模式（old_memory_id 回写） |
| **检索注入** | before_prompt_build hook + LLM 过滤 | API 调用 |
| **去重** | 两级（精确 hash + 语义 LLM） | 由 LLM 在提取时判断 update vs 新建 |
| **质量门控** | Rule filter + LLM 评估 + LLM 质量打分(0-10) | 无独立质量门控 |
| **文件输出** | SKILL.md + scripts/ + references/ + evals/ | SKILL.md + scripts/ + reference/ → zip 包 |
| **语言检测** | CJK 字符比例 > 15% → 中文 | `detect_lang()` 函数 |

### 1.3 LLM 调用次数

```
TS 端——一个任务从消息到 Skill 的完整链路：
  消息摘要(N次) + 去重(N次) + 主题分类(N次)     ← IngestWorker + TaskProcessor
  + 任务摘要(1) + 任务标题(1)                    ← Task Finalize
  + 相关性搜索 Judge(1)                          ← SkillEvolver
  + 创建/升级评估(1)                             ← Evaluator
  + SKILL.md 生成(1) + scripts(1) + refs(1) + evals(1)  ← Generator（新建路径）
  + 质量评估(1)                                  ← Validator
  ≈ 3N + 8 次（N = 消息条数）

  或：升级路径
  + 升级 Prompt(1) + scripts rebuild(1) + evals rebuild(1) + refs rebuild(1)
  + 质量评估(1)                                  ← Validator
  ≈ 3N + 7 次

Python 端——一次批处理：
  任务分块(1) + 查询改写(M次) + Skill 提取(M次)   ← M = 任务数
  + scripts(K次) + tools(K次) + others(K次)       ← K = 需要详细生成的 Skill 数
  ≈ 1 + 2M + 3K 次
```

### 1.4 核心共同点

- 都输出 SKILL.md（YAML frontmatter + Markdown 内容）
- 都做语言一致性（Prompt 和输出跟用户对话语言匹配）
- 都支持 update/merge 逻辑（避免重复 Skill）
- 都强调通用化——去掉具体细节，提取可复用方法论
- 都有脚本和参考文档的附属文件

**以下以 TS 端（完整版）为主线展开，Python 端在第十一章单独介绍。**

---

## 二、TS 端完整走线

### 2.1 整体流程

```
用户发送消息
    │
    ▼
[Hook: before_prompt_build]  ──── 召回路径 ────────────────
    │
    ├── normalizeAutoRecallQuery（清洗 query）
    ├── 并行搜索：本地记忆 + Hub 记忆
    ├── LLM 相关性过滤（FILTER_RELEVANT_PROMPT）
    ├── 搜索相关 Skill
    └── 返回 { prependContext: 注入块 }
    │
    ▼
OpenClaw 处理轮次（LLM 调用、工具使用）
    │
    ▼
[Hook: agent_end]  ──── 提取路径 ────────────────────────
    │
    ├── 游标增量提取新消息（sessionMsgCursor）
    ├── captureMessages（清洗过滤）
    │
    ├── IngestWorker
    │   ├── [Step 1] 摘要生成（LLM）
    │   ├── [Step 2] 两级去重（hash + LLM）
    │   └── 写入 SQLite
    │
    ├── TaskProcessor
    │   ├── [Step 3] 主题分类（LLM）
    │   ├── [Step 3b] 低置信度仲裁（LLM）
    │   └── 主题切换时 → finalizeTask
    │       ├── [Step 4] 任务摘要（LLM）
    │       └── 跳过条件检查
    │
    └── SkillEvolver（Task finalize 回调）
        ├── [Step 5] Rule Filter（纯逻辑）
        ├── [Step 6] 搜索相关 Skill + LLM Judge
        │
        ├── 无匹配 → 创建路径
        │   ├── [Step 7a] 创建评估（LLM）
        │   ├── [Step 8a] SKILL.md 生成（LLM）
        │   ├── [Step 9a] Scripts + Refs（并行 LLM）
        │   ├── [Step 10a] Evals 生成（LLM）
        │   └── [Step 11] 质量验证（LLM）
        │
        └── 有匹配 → 升级路径
            ├── [Step 7b] 升级评估（LLM）
            ├── [Step 8b] 升级合并（LLM）
            ├── [Step 9b] 附属文件重建（3× LLM）
            └── [Step 11] 质量验证（LLM）
```

### 2.2 Hook 注册

```typescript
// apps/memos-local-openclaw/index.ts
const memosLocalPlugin = {
  id: "memos-local-openclaw-plugin",
  kind: "memory",
  register(api) {
    // 记忆能力声明（注入 system prompt 的记忆指引）
    api.registerMemoryCapability({ promptBuilder: buildMemoryPromptSection });

    // ~20 个工具（memory_search, skill_get, skill_install 等）
    api.registerTool("memory_search", ...);
    api.registerTool("skill_get", ...);
    // ...

    // 两个核心 Hook
    api.on("before_prompt_build", handler);  // 召回 + 注入
    api.on("agent_end", handler);            // 捕获 + 提取
  }
};
```

---

## 三、Step 1 — 消息捕获

### 3.1 游标机制（增量提取）

文件：`index.ts`

OpenClaw 每次 `agent_end` 给的是全量消息，用游标避免重复处理：

```typescript
const sessionMsgCursor = new Map<string, number>();

// 首次遇到该 session：定位到最后一条 user 消息
if (!sessionMsgCursor.has(cursorKey)) {
  let lastUserIdx = findLastUserMessage(allMessages);
  sessionMsgCursor.set(cursorKey, lastUserIdx);
}

// 只取游标之后的新消息
const newMessages = allMessages.slice(cursor);
sessionMsgCursor.set(cursorKey, allMessages.length);
```

```
第1轮: messages = [u1, a1]             → cursor 0→2, 处理 [u1, a1]
第2轮: messages = [u1, a1, u2, a2]     → cursor 2→4, 处理 [u2, a2]
第3轮: messages = [u1, a1, u2, a2, u3, a3] → cursor 4→6, 处理 [u3, a3]
```

### 3.2 captureMessages — 消息清洗

文件：`src/capture/index.ts`

**直接丢弃**：

| 类型 | 示例 |
|------|------|
| system 角色 | 系统提示 |
| 哨兵回复 | `NO_REPLY`, `HEARTBEAT_OK`, `HEARTBEAT_CHECK` |
| 启动检查 | `"You are running a boot check..."` |
| 自身工具结果 | `memory_search`, `memory_get`, `skill_search` 等（防循环） |
| 系统样板 | `/new` 或 `/reset` 提示 |

**清洗规则**（剥离元数据，保留内容）：

- 用户消息 → `stripInboundMetadata()`：
  - 去掉 OpenClaw 元数据（`Sender (untrusted metadata):` + JSON 块）
  - 去掉注入的记忆上下文（`<memory_context>...</memory_context>`）
  - 去掉时间戳前缀 `[Tue 2026-03-03 21:58 GMT+8]`
  - 去掉 `[message_id: ...]`、`[[reply_to_current]]` 等标签
- 助手消息：
  - `stripThinkingTags()` — 去掉 `<think>...</think>`（DeepSeek 推理标签）
  - `stripEvidenceWrappers()` — 去掉证据引用包装

**输出结构**：

```typescript
{
  role: "user" | "assistant" | "tool",
  content: string,      // 清洗后的纯文本
  timestamp: number,
  turnId: string,
  sessionKey: string,
  toolName?: string,
  owner: string,        // "agent:main" 等
}
```

---

## 四、Step 2 — 消息存储（IngestWorker）

文件：`src/ingest/worker.ts`

### 4.1 处理流程

```
enqueue(messages)
  → 过滤临时 session（temp:、internal:、system:）
  → 逐条 ingestMessage()
    → [短路] 内容 ≤ 10 词 → 直接用原文当摘要，不调 LLM
    → summarizer.summarize(content)   // LLM 生成摘要
    → embedder.embed([summary])       // 生成向量
    → 去重检查（两级）
    → store.insertChunk(chunk)        // 写入 SQLite
    → store.upsertEmbedding(...)      // 写入向量索引
  → taskProcessor.onChunksIngested()  // 触发主题分割
```

### 4.2 摘要 Prompt

**作用**：为每条消息生成一个检索友好的名词短语标题（不是完整摘要）。

```
You generate a retrieval-friendly title.

Return exactly one noun phrase that names the topic AND its key details.

Requirements:
- Same language as input
- Keep proper nouns, API/function names, specific parameters, versions, error codes
- Include WHO/WHAT/WHERE details when present
  (e.g. person name + event, tool name + what it does)
- Prefer concrete topic words over generic words
- No verbs unless unavoidable
- No generic endings like:
  功能说明、使用说明、简介、介绍、用途、summary、overview、basics
- Chinese: 10-50 characters (aim for 15-30)
- Non-Chinese: 5-15 words (aim for 8-12)
- Output title only
```

**参数**：`temperature: 0`，`max_tokens: 100`，`timeout: 30s`

**输入**：`[TEXT TO SUMMARIZE]\n{content}\n[/TEXT TO SUMMARIZE]`

**短路**：内容 ≤ 10 词 → 直接返回清洗后的原文，不调 LLM。

### 4.3 两级去重

**Level 1 — 精确去重**：`content_hash` 完全匹配 → 淘汰旧 chunk，保留新的。

**Level 2 — 语义去重**：向量相似度 > 0.80 的 top-5 候选 → LLM 判断。

**去重 Prompt**：

```
You are a memory deduplication system.

LANGUAGE RULE (MUST FOLLOW): You MUST reply in the SAME language as the input
memories. 如果输入是中文，reason 和 mergedSummary 必须用中文。If input is English,
reply in English.

Given a NEW memory summary and several EXISTING memory summaries, determine
the relationship.

For each EXISTING memory, the NEW memory is either:
- "DUPLICATE": NEW conveys the same intent/meaning as an EXISTING memory,
  even if worded differently. Examples: "请告诉我你的名字" vs "你希望我怎么称呼你";
  greetings with minor variations. If the core information/intent is the same,
  it IS a duplicate.
- "UPDATE": NEW contains meaningful additional information that supplements
  an EXISTING memory (new data, status change, concrete detail not present before)
- "NEW": NEW covers a genuinely different topic/event with no semantic overlap

IMPORTANT: Lean toward DUPLICATE when memories share the same intent, topic,
or factual content. Only choose NEW when the topics are truly unrelated.

Pick the BEST match among all candidates. If none match well, choose "NEW".

Output a single JSON object:
- If DUPLICATE: {"action":"DUPLICATE","targetIndex":2,"reason":"..."}
- If UPDATE: {"action":"UPDATE","targetIndex":3,"reason":"...","mergedSummary":"..."}
- If NEW: {"action":"NEW","reason":"..."}
```

**参数**：`temperature: 0`，`max_tokens: 300`，`timeout: 15s`

**输入**：`NEW MEMORY:\n{newSummary}\n\nEXISTING MEMORIES:\n1. {summary1}\n2. {summary2}\n...`

**执行逻辑**：

```
DUPLICATE → 淘汰旧 chunk（标记 dedupStatus="duplicate"），新 chunk 标记 active
UPDATE   → 合并摘要（用 LLM 返回的 mergedSummary），淘汰旧 chunk
NEW      → 直接存储为 active
```

### 4.4 Chunk 数据结构

```typescript
interface Chunk {
  id: string;
  sessionKey: string;
  turnId: string;
  seq: number;
  role: "user" | "assistant" | "tool";
  content: string;         // 原始内容
  kind: "paragraph";
  summary: string;         // LLM 生成的标题
  embedding: number[];     // 向量
  taskId: string | null;   // 所属任务
  skillId: string | null;  // 关联 Skill
  owner: string;
  dedupStatus: "active" | "duplicate" | "merged";
  mergeCount: number;
  mergeHistory: string;    // JSON 合并历史
  createdAt: number;
  updatedAt: number;
}
```

---

## 五、Step 3 — 主题分割（TaskProcessor）

文件：`src/ingest/task-processor.ts`

### 5.1 触发时机

每次 IngestWorker 存完一批 chunks 后调用 `onChunksIngested()`。

### 5.2 判断逻辑

三种条件触发"新任务"：

| 条件 | 判断方式 | 阈值 |
|------|----------|------|
| Session 变了 | 直接切 | — |
| 时间间隔超限 | 直接切 | > 2h |
| 主题变了 | LLM 判断 | confidence ≥ 0.65 |

### 5.3 增量主题检测流程

```
未分配的 chunks → 按 user turn 分组
  ↓
逐 turn 处理：
  ↓
当前任务有 ≥1 个 user turn？
  → 没有 → 直接归入当前任务
  → 有 → 继续判断
  ↓
时间间隔检查（> 2h？）
  → 超时 → finalize 旧任务，创建新任务
  ↓
buildTopicJudgeState() 构建上下文：
  - topic: 第一条 user 消息摘要（或内容前 80 字符）
  - 最近 3 轮 user/assistant 摘要
  - 短消息（<30字符）或代词开头 → 附带上一条 assistant 回复（前 200 字符）
  ↓
classifyTopic(taskState, newMsg)                    🤖 LLM
  → 返回 { decision: "SAME"|"NEW", confidence }
  ↓
decision == "NEW" && confidence < 0.65？
  → 二次仲裁 arbitrateTopicSplit()                   🤖 LLM
  ↓
确认 NEW → finalizeTask(旧任务) → 创建新任务
```

### 5.4 主题分类 Prompt

```
Classify if NEW MESSAGE continues current task or starts an unrelated one.
Output ONLY JSON: {"d":"S"|"N","c":0.0-1.0}
d=S(same) or N(new). c=confidence. Default S.
Only N if completely unrelated domain.
Sub-questions, tools, methods, details of current topic = S.
```

**参数**：`temperature: 0`，`max_tokens: 60`，`timeout: 15s`

**输入**：`TASK:\n{taskState}\n\nMSG:\n{newMessage}`

**输出**：`{"d":"S","c":0.85}` → 映射为 `{decision: "SAME", confidence: 0.85}`

### 5.5 仲裁 Prompt（二次确认）

当分类返回 NEW 但 confidence < 0.65 时触发：

```
A classifier flagged this message as possibly new topic (low confidence).
Is it truly UNRELATED, or a sub-question/follow-up?
Tools/methods/details of current task = SAME.
Shared entity/theme = SAME. Entirely different domain = NEW.
Reply one word: NEW or SAME
```

**参数**：`temperature: 0`，`max_tokens: 10`，`timeout: 15s`

### 5.6 任务 Finalize

```typescript
async finalizeTask(task) {
  const chunks = this.store.getChunksByTask(task.id);

  // 跳过条件（任一满足则不生成摘要）
  const skipReason = this.shouldSkipSummary(chunks);
  if (skipReason) { /* 标记 skipped */ return; }

  // 生成任务摘要
  const summary = await this.summarizer.summarizeTask(conversationText);

  // 更新任务状态
  this.store.updateTask(task.id, { title, summary, status: "completed" });

  // 触发 Skill 提取回调
  this.onTaskCompletedCallback(finalized);
}
```

**跳过条件**（任一满足则不触发 Skill 提取）：
1. chunks < 4
2. 真实对话轮数 < 2
3. 无 user 消息
4. 总内容 < 200 字符
5. 用户内容是闲聊/测试（hello、test、ok 等）
6. 全是 tool 结果
7. 高重复率（调试循环）

### 5.7 任务摘要 Prompt

```
You create a DETAILED task summary from a multi-turn conversation.
This summary will be the ONLY record of this conversation, so it must
preserve ALL important information.

## LANGUAGE RULE (HIGHEST PRIORITY)
Detect the PRIMARY language of the user's messages. If most user messages
are Chinese, ALL output MUST be in Chinese. If English, output in English.

Output EXACTLY this structure:

📌 Title / 标题
A short, descriptive title (10-30 characters).

🎯 Goal / 目标
One sentence: what the user wanted to accomplish.

📋 Key Steps / 关键步骤
- Describe each meaningful step in detail
- Include the ACTUAL content produced: code snippets, commands, config blocks
- For code: include function signature and core logic (up to ~30 lines)
- Do NOT over-summarize: "provided a function" is BAD; show the actual function

✅ Result / 结果
What was the final outcome?

💡 Key Details / 关键细节
- Decisions made, trade-offs discussed, caveats noted
- Specific values: numbers, versions, thresholds, URLs, file paths

RULES:
- This summary is a KNOWLEDGE BASE ENTRY, not a brief note. Be thorough.
- PRESERVE verbatim: code, commands, URLs, file paths, error messages
- DISCARD only: greetings, filler
- Replace secrets (API keys, tokens) with [REDACTED]
- Target length: 30-50% of original conversation.
- Output summary only, no preamble.
```

**参数**：`temperature: 0.1`，`max_tokens: 4096`，`timeout: 60s`

**输入**：完整对话文本，格式为 `[User]: {content}\n\n[Assistant]: {content}\n\n...`

**标题解析**：正则 `/📌\s*(?:Title|标题)\s*\n(.+)/` 从输出中提取。

---

## 六、Step 4-6 — Skill 评估（SkillEvolver + Evaluator）

文件：`src/skill/evolver.ts`、`src/skill/evaluator.ts`

### 6.1 入口

```
Task finalize → onTaskCompletedCallback → SkillEvolver.onTaskCompleted(task)
```

如果 evolver 正在处理其他任务，新任务进入队列，按顺序处理。

### 6.2 Step 4 — Rule Filter（纯逻辑，不用 LLM）

```
chunks.length < 6          → skip（默认 minChunksForEval = 6）
task.status === "skipped"   → skip
task.summary.length < 100   → skip
无 user chunks              → skip
无 assistant chunks         → skip
```

### 6.3 Step 5 — 搜索相关 Skill

**两阶段搜索**：

```
Stage 1: FTS + 向量搜索（cosine 下限 0.35）
  → 融合得分 = 0.7 × 向量分 + 0.3 × FTS 分
  → 取 top 10

Stage 2: LLM Judge（从 top 10 中选出真正匹配的 1 个）
```

**Skill Relevance Judge Prompt**：

```
You are a strict judge: decide whether a completed TASK should be merged
into an EXISTING SKILL. The task and the skill must be in the SAME
domain/topic — e.g. same type of problem, same tool, same workflow.
Loose or tangential relevance is NOT enough.

TASK TITLE: ${taskTitle}

TASK SUMMARY:
${taskSummary}

CANDIDATE SKILLS (index, name, description):
${skillList}

RULES:
- Output exactly ONE skill index (1 to N) ONLY if the task's experience
  clearly belongs to that skill's domain.
- If no skill is clearly relevant, output 0. When in doubt, output 0.
- Do not force a match.

LANGUAGE RULE: "reason" MUST use the SAME language as the task title/summary.

Reply with JSON only:
{"selectedIndex": 0, "reason": "brief explanation (same language as input)"}
```

**参数**：`temperature: 0`，`max_tokens: 256`

**分支逻辑**：
- `selectedIndex > 0` → 有匹配 → 走升级路径（Step 7b）
- `selectedIndex == 0` 且 `preferUpgradeExisting=true` → 尝试 name similarity 二次搜索
- 无匹配 → 走创建路径（Step 7a）

### 6.4 Step 6a — 创建评估 Prompt（CREATE_EVAL_PROMPT）

```
You are a strict experience evaluation expert. Based on the completed task
record below, decide whether this task contains **reusable, transferable**
experience worth distilling into a "skill".

A skill is a reusable guide that helps an AI agent handle **the same type
of task** better in the future.

STRICT criteria — must meet ALL of:
1. **Repeatable**: The task type is likely to recur
2. **Transferable**: The approach would help others facing the same problem
3. **Technical depth**: Contains non-trivial steps, commands, code, configs,
   or diagnostic reasoning

Worth distilling (at least ONE):
- Solves a recurring technical problem with a specific approach/workflow
- Went through trial-and-error — the learning is valuable
- Involves non-obvious usage of specific tools, APIs, or frameworks
- Contains debugging/troubleshooting with diagnostic reasoning
- Shows how to combine multiple tools/services
- Contains deployment, configuration, or infrastructure setup steps
- Demonstrates a reusable data processing or automation pipeline

NOT worth distilling (if ANY matches → shouldGenerate=false):
- Pure factual Q&A with no process
- Single-turn simple answers with no workflow
- Conversation too fragmented or incoherent
- One-off personal tasks: identity confirmation, preference setting
- Casual chat, opinion discussion, brainstorming without actionable output
- Simple information lookup or summarization
- Organizing/listing personal information
- Generic product/system overviews without specific operational steps
- Tasks where the "steps" are just the AI answering questions

Task title: {TITLE}
Task summary:
{SUMMARY}

LANGUAGE RULE: reason in same language as input.
Only "suggestedName" stays in English kebab-case.

Reply in JSON only:
{
  "shouldGenerate": boolean,
  "reason": "brief explanation",
  "suggestedName": "kebab-case-name",
  "suggestedTags": ["tag1", "tag2"],
  "confidence": 0.0-1.0
}
```

**参数**：`temperature: 0.1`，`max_tokens: 1024`，`timeout: 30s`

**输入**：task.title + task.summary（截断到 3000 字符）

**门槛**：`shouldGenerate && confidence >= 0.7`（默认 minConfidence = 0.7）

### 6.5 Step 6b — 升级评估 Prompt（UPGRADE_EVAL_PROMPT）

```
You are a skill upgrade evaluation expert.

Existing skill (v{VERSION}):
Name: {SKILL_NAME}
Content:
{SKILL_CONTENT}

Newly completed task:
Title: {TITLE}
Summary:
{SUMMARY}

Does the new task bring substantive improvements to the existing skill?

Worth upgrading (any one qualifies):
1. Faster — shorter path discovered
2. More elegant — cleaner, follows best practices better
3. More convenient — fewer dependencies or complexity
4. Fewer tokens — less exploration/trial-and-error needed
5. More accurate — corrects wrong parameters/steps
6. More robust — adds edge cases, error handling
7. New scenario — covers a variant the old skill didn't
8. Fixes outdated info — old skill has stale information

NOT worth upgrading:
- New task is identical to existing skill
- New task's approach is worse
- Differences are trivial

LANGUAGE RULE: "reason" and "mergeStrategy" same language as input.

Reply in JSON only:
{
  "shouldUpgrade": boolean,
  "upgradeType": "refine" | "extend" | "fix",
  "dimensions": ["faster", "more_elegant", ...],
  "reason": "what new value the task brings",
  "mergeStrategy": "which specific parts need updating",
  "confidence": 0.0-1.0
}
```

**参数**：`temperature: 0.1`，`max_tokens: 1024`，`timeout: 30s`

**输入**：Skill 内容截断到 4000 字符，task summary 截断到 3000 字符

**分支**：
- `shouldUpgrade && confidence >= 0.7` → 走升级路径
- `confidence < 0.3` → 低相关性逃逸，改走创建路径
- 其余 → 关联任务但不升级

---

## 七、Step 7a — Skill 生成（Generator）

文件：`src/skill/generator.ts`

### 7.1 预处理

**敏感数据脱敏**（`redactSensitive`）：
- `sk-[20+字符]` → `sk-***REDACTED***`
- `Bearer [20+字符]` → `Bearer ***REDACTED***`
- `AKIA[16字符]` → `AKIA***REDACTED***`
- `api_key/secret/token/password = "..."` → `="***REDACTED***"`
- `/Users/<username>/` → `/Users/****/`

**Tool 输出截断**：每个 tool chunk 最多 1500 字符（60% 头部 + 30% 尾部 + 截断提示）。

**语言检测**：CJK 字符比例 > 15% → 中文，否则英文。

### 7.2 Step 1: 生成 SKILL.md（STEP1_SKILL_MD_PROMPT）

```
You are a Skill creation expert. Your job is to distill a completed task's
execution record into a reusable SKILL.md file.

This Skill is special: it comes from real execution experience — every step
was actually run, every pitfall was actually encountered and resolved.

## Core principles

### Progressive disclosure
- frontmatter description (~100 words): ALWAYS in agent's context,
  must be self-sufficient for deciding whether to use this skill
- SKILL.md body: loaded when triggered, keep under 400 lines
- Large configs/scripts → reference, don't inline

### Description as trigger mechanism
The description field decides whether the agent activates this skill.
Write it "proactively":
- Don't just say what it does — list situations, keywords, phrasings
  that should trigger it
- Bad: "How to deploy Node.js to Docker"
- Good: "How to containerize and deploy a Node.js application using Docker.
  Use when the user mentions Docker deployment, Dockerfile writing,
  container builds, multi-stage builds, port mapping, .dockerignore,
  image optimization, CI/CD container pipelines, or any task involving
  packaging a Node/JS backend into a container — even if they don't say
  'Docker' explicitly but describe wanting to 'package the app for
  production' or 'run it anywhere'."

### Writing style
- Imperative form
- Explain WHY for each step, not just HOW
- ALWAYS or NEVER in caps is a yellow flag — rephrase with reasoning
- Generalize from the specific task
- Keep real commands/code/config — these are verified to work

### Language matching (CRITICAL)
Write the ENTIRE skill in the SAME language as the user's messages.
"name" field stays English kebab-case (machine identifier).

## Output format

---
name: "{NAME}"
description: "{60-120 words, proactive trigger description}"
metadata: { "openclaw": { "emoji": "{emoji}" } }
---

# {Title}
{What this skill helps you do and why}

## When to use this skill
{2-4 bullet points}

## Steps
{Numbered steps from the task record}

## Pitfalls and solutions
{❌ Wrong → Why → ✅ Correct}

## Key code and configuration
{Verified code blocks}

## Environment and prerequisites
{Versions, dependencies, permissions}

## Companion files
{List of scripts/ and references/}

## Task record
Task title: {TITLE}
Task summary: {SUMMARY}
Conversation highlights: {CONVERSATION}
```

**参数**：`temperature: 0.2`，`max_tokens: 6000`，`timeout: 120s`

**输入**：task.summary（5000 字符）+ conversation text（12000 字符）

**后处理**：去掉第一个 `---` 之前的任何文本。

### 7.3 Step 2: Scripts + References（并行）

两个 LLM 调用通过 `Promise.all` 并行执行：

**Scripts Prompt**：

```
Based on the following SKILL.md and task record, extract reusable
automation scripts.

Rules:
- Only extract if the task record contains concrete shell commands,
  Python scripts, or TypeScript code that form a complete, reusable automation
- Each script must be self-contained and runnable
- If no automatable scripts, return empty array
- Don't fabricate scripts — only extract what was actually used
- Script should COMPLEMENT the SKILL.md, not duplicate it

Reply with a JSON array only:
[
  { "filename": "deploy.sh", "content": "#!/bin/bash\n..." }
]

If no scripts: reply with []
```

**参数**：`temperature: 0.1`，`max_tokens: 3000`，`timeout: 120s`

**References Prompt**：

```
Based on the following SKILL.md and task record, extract reference
documentation worth preserving.

Rules:
- Only extract if the task involved important API docs, configuration
  references, or technical notes
- Each reference should be a standalone markdown document
- Don't duplicate what's in SKILL.md
- LANGUAGE RULE: Same language as SKILL.md and task record

Reply with a JSON array only:
[
  { "filename": "api-notes.md", "content": "# API Reference\n..." }
]

If no references: reply with []
```

**参数**：`temperature: 0.1`，`max_tokens: 3000`，`timeout: 120s`

### 7.4 Step 3: Evals 生成（STEP3_EVALS_PROMPT）

```
Based on the following skill, generate realistic test prompts that
should trigger this skill.

Requirements:
- Write 3-4 test prompts that a real user would type
- Mix of direct and indirect phrasings
- Include realistic details: file paths, project names, error messages
- Mix formal and casual tones, include some with typos or shorthand
- Each prompt should be complex enough that the agent would need the skill
- Write expectations that are specific and verifiable
- LANGUAGE RULE: Same language as skill content

Reply with a JSON array only:
[
  {
    "id": 1,
    "prompt": "A realistic user message",
    "expectations": ["Expected behavior 1", "Expected behavior 2"],
    "trigger_confidence": "high|medium"
  }
]
```

**参数**：`temperature: 0.3`，`max_tokens: 2000`，`timeout: 120s`

### 7.5 Step 4: Eval 验证

**不用 LLM** — 用 RecallEngine 对每个 eval prompt 做搜索，检查该 Skill 是否能被检索到（`minScore: 0.3`，`topScore >= 0.4`）。

### 7.6 持久化

```
{stateDir}/skills-store/{skill-name}/
  ├── SKILL.md
  ├── scripts/
  ├── references/
  └── evals/evals.json

qualityScore < 6 → status = "draft"
qualityScore >= 6 → status = "active"
```

---

## 八、Step 7b — Skill 升级（Upgrader）

文件：`src/skill/upgrader.ts`

### 8.1 升级 Prompt（UPGRADE_PROMPT）

```
You are a Skill upgrade expert. You're merging new real-world execution
experience into an existing Skill to make it better.

Remember: this is based on ACTUAL execution — the new task was really run,
errors were really encountered and fixed.

## Core principles
[与 Generator 相同的 progressive disclosure、trigger description、writing style 原则]

## Existing skill (v{VERSION}):
{SKILL_CONTENT}

## Upgrade context
- Type: {UPGRADE_TYPE}
- Dimensions improved: {DIMENSIONS}
- Reason: {REASON}
- Merge strategy: {MERGE_STRATEGY}

## New task record
Title: {TITLE}
Summary: {SUMMARY}

## Merge rules
1. Preserve all valid core content from existing skill
2. Merge new experience strategically:
   - Better approach → replace old, keep old as "Alternative" if still valid
   - New scenario → add new section (don't replace unrelated content)
   - Bug/error corrected → replace directly, add to "Pitfalls"
   - Performance improvement → update steps, note improvement
3. Update description if new scenarios/keywords/triggers needed
4. Update "When to use this skill" if new use cases revealed
5. Append new pitfalls to existing section
6. Total length ≤ 400 lines
7. Add version comment:
   <!-- v{NEW_VERSION}: {one-line change note} (from task: {TASK_ID}) -->

## Output format
Output the complete upgraded SKILL.md, then:
---CHANGELOG---
{one-line changelog title}
---CHANGE_SUMMARY---
{3-5 sentence summary}
```

**参数**：`temperature: 0.2`，`max_tokens: 6000`，`timeout: 90s`

**输入**：Skill 内容（6000 字符）+ task summary（4000 字符）

### 8.2 附属文件重建

升级后执行 3 个并行 LLM 调用重建附属文件（Prompt 结构与 Generator 的 Step 2-3 相同）：

| 文件 | temperature | max_tokens | timeout |
|---|---|---|---|
| Scripts rebuild | 0.1 | 3000 | 60s |
| Evals rebuild | 0.3 | 2000 | 60s |
| Refs rebuild | 0.1 | 3000 | 60s |

### 8.3 回退机制

```
升级前 → 备份 Skill 目录
  → LLM 生成新内容
  → 写入新 SKILL.md + 重建附属文件
  → Validator 验证（含回归检查：新内容比旧内容缩水 >30% → warning）
  → 验证失败 → 从备份恢复整个目录
  → 验证通过 → 持久化新版本（version +1）
```

---

## 九、Step 8 — 质量验证（Validator）

文件：`src/skill/validator.ts`

### 9.1 验证阶段

**阶段 1 — 格式验证**（不用 LLM）：
- SKILL.md 必须存在且非空
- 必须有 YAML frontmatter（`---...---`）
- 必须有 `name` 字段（max 64 字符，kebab-case 警告）
- 必须有 `description` 字段（max 1024 字符警告）
- 行数 vs `skillMaxLines`（默认 400）
- 内容 < 200 字符 → 警告

**阶段 2 — 回归检查**（升级时，不用 LLM）：
- 新内容比旧内容缩水 > 30%（且旧内容 > 20 行）→ 警告
- 检测缺失的 `##` 章节（按名称对比）

**阶段 3 — 附属文件一致性**（不用 LLM）：
- SKILL.md 引用的 `scripts/X` 和 `references/X` 必须在磁盘上存在
- 磁盘上未被引用的孤立文件 → 警告
- `evals/evals.json` 结构验证

**阶段 4 — 密钥扫描**（不用 LLM）：
- `sk-[20+字符]`、`Bearer [20+字符]`、`AKIA[16字符]`
- `api_key/secret/token/password = "..."`
- Base64 字符串（40+ 字符）

**阶段 5 — LLM 质量评分**：

```
You are a skill quality reviewer. Evaluate the following SKILL.md
and give a score from 0 to 10.

Criteria:
1. Clarity: Are the steps clear and actionable? (0-2 pts)
2. Completeness: Does it cover scenarios, pitfalls, and key code? (0-2 pts)
3. Reusability: Can this skill be applied to similar future tasks? (0-2 pts)
4. Accuracy: Are commands, code, and configurations correct? (0-2 pts)
5. Structure: Is the format well-organized with proper sections? (0-2 pts)

LANGUAGE RULE: strengths/weaknesses/suggestions same language as SKILL.md.

Reply in JSON only:
{
  "score": 0-10,
  "strengths": ["..."],
  "weaknesses": ["..."],
  "suggestions": ["..."]
}
```

**参数**：`temperature: 0.1`，`max_tokens: 1024`

**质量阈值**：`score < 6` → Skill 标记为 "draft"

---

## 十、Skill 召回与注入

文件：`index.ts`（before_prompt_build hook）

### 10.1 完整召回流程

```
用户消息
  │
  ▼
[1] normalizeAutoRecallQuery（确定性清洗，不用 LLM）
    - 去掉 Sender metadata 块
    - 去掉 HTML 标签
    - 去掉 session startup 样板
    - 去掉 "Current time:..." 行
    - 去掉 internal context
    - 去掉 continuation prompt
    结果 < 2 字符 → 跳过召回
  │
  ▼
[2] 并行搜索
    ├── 本地引擎：engine.search({ query, maxResults: 10, minScore: 0.45 })
    └── Hub 远程搜索（如果启用 sharing）
  │
  ▼
[3] 无结果？
    ├── 尝试 Skill 搜索 → 有 Skill → 注入 Skill 提示
    └── query > 50 字符 → 注入"必须调用 memory_search"的强制提示
  │
  ▼
[4] LLM 相关性过滤（FILTER_RELEVANT_PROMPT）                🤖 LLM
  │
  ▼
[5] 去重（>70% 词汇重叠 → 去掉）
  │
  ▼
[6] 组装注入块 → 返回 { prependContext: context }
```

### 10.2 相关性过滤 Prompt

```
You are a memory relevance judge.

Given a QUERY and CANDIDATE memories, decide: does each candidate's content
contain information that would HELP ANSWER the query?

CORE QUESTION: "If I include this memory, will it help produce a better answer?"
- YES -> include
- NO -> exclude

RULES:
1. Relevant if content provides facts, context, or data that directly
   supports answering the query.
2. Merely shares same broad topic but NO useful information → NOT relevant.
3. If NO candidate can help, return {"relevant":[],"sufficient":false}
   — do NOT force-pick the "least irrelevant" one.
4. DEDUPLICATION: When multiple candidates convey same information,
   keep ONLY the most complete/detailed one.

OUTPUT — JSON only:
{"relevant":[1,3],"sufficient":true}
```

**参数**：`temperature: 0`，`max_tokens: 200`

### 10.3 注入格式

**记忆注入块**：

```
## User's conversation history (from memory system)

IMPORTANT: The following are facts from previous conversations with this user.
You MUST treat these as established knowledge and use them directly when answering.

1. [user]
   {excerpt}
   chunkId="{id}"
   task_id="{taskId}"

Available follow-up tools:
- A hit has `task_id` → call `task_summary(taskId="...")` for full context
- A task may have a reusable guide → call `skill_get(taskId="...")` for skill
- Need surrounding dialogue → call `memory_timeline(chunkId="...")`
```

**Skill 注入块**（追加在记忆之后）：

```
## Relevant skills from past experience

The following skills were distilled from similar previous tasks.
You SHOULD call `skill_get` to retrieve the full guide before attempting the task.

1. **{name}** [installed] — {description}
   → call `skill_get(skillId="{id}")`
```

**不足提示**（如果 sufficient=false）：

```
If these memories don't fully answer the question, call `memory_search`
with a shorter or rephrased query to find more.
```

---

## 十一、Skill 存储与文件系统

### 11.1 SKILL.md 格式

```markdown
---
name: deploy-docker-app
description: >
  How to containerize and deploy a Node.js application using Docker.
  Use when the user mentions Docker deployment, Dockerfile writing,
  container builds, multi-stage builds, port mapping...
metadata:
  openclaw:
    emoji: "🐳"
---

# Deploy Docker App
One sentence: what this skill helps you do.

## When to use this skill
- ...

## Steps
1. ...

## Pitfalls and solutions
❌ Wrong → ✅ Correct

## Key code and configuration
```

### 11.2 文件系统结构

```
{stateDir}/skills-store/{skill-name}/
  ├── SKILL.md           # 主文档（YAML frontmatter + 内容）
  ├── scripts/           # 可执行脚本
  ├── references/        # 参考文档
  └── evals/evals.json   # 测试用例
```

### 11.3 数据模型

```typescript
interface Skill {
  id: string;
  name: string;                    // kebab-case
  description: string;
  version: number;                 // 每次升级 +1
  status: "active" | "archived" | "draft";
  tags: string;                    // JSON 数组
  sourceType: "task" | "manual";
  dirPath: string;
  installed: number;
  owner: string;
  qualityScore: number | null;     // 0-10
  createdAt: number;
  updatedAt: number;
}

interface SkillVersion {
  id: string;
  skillId: string;
  version: number;
  content: string;                 // 完整 SKILL.md
  changelog: string;
  changeSummary: string;
  upgradeType: "create" | "refine" | "extend" | "fix";
  sourceTaskId: string | null;
  metrics: string;                 // JSON: dimensions, confidence, validation
  qualityScore: number | null;
  createdAt: number;
}
```

### 11.4 安装机制

`SkillInstaller` 没有 LLM 调用，纯文件操作：

- `install(skillId)` → 复制到 `{workspaceDir}/skills/{skillName}/`，标记 installed
- `uninstall(skillId)` → 删除 workspace 副本，标记 uninstalled
- `syncIfInstalled(skillName)` → 升级后重新复制（如果已安装）

**自动安装判断**（heuristic）：
- 3+ 可执行脚本，或
- 2+ 大文件（>5KB），或
- 总附属文件 > 20KB

### 11.5 默认配置

| 配置项 | 默认值 | 含义 |
|---|---|---|
| `skillEvolutionEnabled` | `true` | 开启 Skill 进化 |
| `skillAutoEvaluate` | `true` | 自动评估任务 |
| `skillMinChunksForEval` | `6` | 最少 chunk 数 |
| `skillMinConfidence` | `0.7` | 最低置信度 |
| `skillMaxLines` | `400` | SKILL.md 最大行数 |
| `skillAutoInstall` | `false` | 不自动安装 |
| `skillAutoRecall` | `true` | 自动召回 |
| `skillAutoRecallLimit` | `2` | 最多注入 2 个 Skill |
| `skillPreferUpgrade` | `true` | 优先升级而非新建 |
| `skillRedactSensitive` | `true` | 脱敏敏感数据 |

---

## 十二、Python 端处理管线

文件：`src/memos/templates/skill_mem_prompt.py`、`src/memos/mem_reader/read_skill_memory/process_skill_memory.py`

### 12.1 整体流程

```
对话消息（全量）
    │
    ▼
[Step 1] 消息重建 + 预处理
    - TextualMemoryItem → 平铺 {role, content} 列表
    - 截断 chat_history 到最近 20 条
    - 给每条消息加 idx 编号
    │
    ▼
[Step 2] 任务分块                                           🤖 LLM
    - TASK_CHUNKING_PROMPT / _ZH
    - 支持非连续消息归组（跳跃式对话）
    - 过滤闲聊
    │
    ▼
[Step 3] 召回相关 Skill（并行，max 5 workers）
    - 先查询改写                                             🤖 LLM
      TASK_QUERY_REWRITE_PROMPT / _ZH
    - 再搜索 top-5 相关已有 SkillMemory
    │
    ▼
[Step 4] Skill 提取（两种模式）
    │
    ├── 简单模式                                             🤖 LLM
    │   SKILL_MEMORY_EXTRACTION_PROMPT / _ZH
    │   → 输出完整 Skill JSON（含 scripts 代码）
    │
    └── 完整模式（默认）
        ├── Phase 1: 提取 Skill 骨架                         🤖 LLM
        │   SKILL_MEMORY_EXTRACTION_PROMPT_MD / _ZH
        │   → 输出 Skill JSON（scripts 为 TODO 列表）
        │
        └── Phase 2: 详细生成（并行）
            ├── scripts → SCRIPT_GENERATION_PROMPT            🤖 LLM
            ├── tools → TOOL_GENERATION_PROMPT                🤖 LLM
            └── others → OTHERS_GENERATION_PROMPT / _ZH       🤖 LLM
    │
    ▼
[Step 5] 文件写入
    - SKILL.md（YAML frontmatter + sections）
    - scripts/ 目录
    - reference/ 目录
    - 打包 {skill-name}.zip
    │
    ▼
[Step 6] 上传 + 注册
    - 上传到 OSS 或 Local 存储
    - 如果是 update → 删除旧 zip + 旧图谱记录
    - 创建 TextualMemoryItem（嵌入 description 向量）
    - 注册到 MySQL
```

### 12.2 任务分块 Prompt（TASK_CHUNKING_PROMPT）

```
# Context (Conversation Records)
{{messages}}

# Role
You are an expert in NLP and dialogue logic analysis. You excel at
organizing logical threads from complex long conversations and accurately
extracting users' core intentions to segment the dialogue into distinct tasks.

# Task
Analyze the provided conversation records, identify all independent "tasks",
and assign the corresponding dialogue message indices to each task.

**Note**: Tasks should be high-level and general. Group similar activities
under broad themes such as "Travel Planning", "Code Review", "Data Analysis".

# Rules & Constraints
1. **Task Independence**: Completely unrelated topics → different tasks.
2. **Main Task and Subtasks**: If a subtask serves a primary objective
   (e.g., "checking weather" within "Travel Planning"), do NOT separate it.
   **Only split when truly independent and unrelated.**
3. **Non-continuous Processing**: Identify "jumping" conversations.
   e.g., messages 8-11 = Travel, 12-22 = other, 23-24 = Travel again
   → both [8,11] and [23,24] go to "Travel Planning".
4. **Filter Chit-chat**: Only extract tasks with clear goals.
5. **Output Format**: Strict JSON.
6. **Language Consistency**: task_name matches conversation language.
7. **Generic Task Names**: "Travel Planning" not "Planning a 5-day trip to Chengdu".

```json
[
  {
    "task_id": 1,
    "task_name": "Generic task name",
    "message_indices": [[0, 5], [16, 17]],
    "reasoning": "Brief logic explanation"
  }
]
```
```

**重试**：最多 3 次，含 JSON 清洗（去掉 ````json` 围栏）。

### 12.3 查询改写 Prompt（TASK_QUERY_REWRITE_PROMPT）

```
# Role
You are an expert in understanding user intentions.

# Task
Based on the task type and conversation messages, rewrite into a clear,
concise task query string.

# Requirements
1. Analyze conversation to understand core intention
2. Consider task type as context
3. Extract and summarize key task objective
4. Output one sentence
5. Same language as conversation
6. Focus on WHAT, not HOW
7. No explanations, just output the string

# Output
Output only the rewritten task query string.
```

### 12.4 Skill 提取 Prompt — 简单版（SKILL_MEMORY_EXTRACTION_PROMPT）

输出字段：`name, description, procedure, experience, preference, examples, tags, scripts, others, update, old_memory_id`

**核心原则**：

```
1. 通用化：提取可跨场景应用的抽象方法论，避免具体细节
2. 普适性：除 examples 外，所有字段保持通用
3. 相似性检查：存在相似 skill → update=true + old_memory_id
4. 语言一致性
5. 历史使用约束：chat_history 仅作辅助上下文
   - 只在能提供 messages 中缺失的增量信息时才使用
   - 不得作为 skill 的主要来源
```

### 12.5 Skill 提取 Prompt — 完整版（SKILL_MEMORY_EXTRACTION_PROMPT_MD）

与简单版的主要区别：

| 差异 | 简单版 | 完整版（MD） |
|---|---|---|
| 分类字段 | `tags` | `trigger`（触发关键词列表） |
| scripts | 实际代码 dict | TODO 列表（Phase 2 再生成） |
| 新增字段 | 无 | `tool`（所需外部工具列表） |
| 设计目的 | 一步到位 | 两阶段管线 |

**核心额外原则**：

```
技能提取原则：
1. 通用化：提取的技能应该持久有效，而非与特定时间绑定
2. 相似性检查：存在相同主题的技能 → 必须用更新操作
3. 语言一致性
4. 历史使用约束

关键指导：
- 无法提取时返回 null
- 同样场景尽量合并（update: true）
  如饮食规划合并为一条，不要已有"饮食规划"又新增"生酮饮食规划"
  因为技能是通用模版，可以添加 preference 和 trigger 来更新
```

### 12.6 详细生成 Prompt

**Script Generation**：

```
# Role
You are a Senior Python Developer and Architect.

# Task
Generate production-ready, executable Python scripts.

# Instructions
1. Completeness: Fully functional, no placeholders
2. Robustness: Error handling + input validation
3. Style: PEP 8, type hints
4. Dependencies: Standard libraries first
5. Main Guard: Include if __name__ == "__main__" with examples

# Output Format
JSON: { "filename.py": "code..." }
```

**Tool Generation**：

```
Analyze Requirements and Context to identify relevant tools from
Available Tools. Return a list of matching tool names.

Constraints:
1. Include only if tool schema directly addresses requirements
2. Empty tools or no match → return []
3. Return ONLY JSON array of strings

Output: ["tool_name_1", "tool_name_2"]
```

**Others Generation**：

```
Create detailed documentation for '{filename}'.

Structure:
1. Introduction: Brief overview
2. Detailed Content: Organized with headers
3. Key Concepts/Reference: Definitions or tables
4. Conclusion/Next Steps

Formatting: Markdown. Language: Same as context.
Output: Markdown directly.
```

### 12.7 Python 端 Skill 数据模型

```python
class SkillMemory(TextualMemoryMetadata):
    memory_type = "SkillMemory"
    name: str
    description: str
    procedure: str              # 步骤流程
    experience: list[str]       # 经验教训
    preference: list[str]       # 用户偏好
    examples: list[str]         # 输出模板
    scripts: dict | None        # {filename: code}
    others: dict | None         # 补充文档
    url: str                    # Skill zip URL
    # 继承：sources, embedding, tags, status, created_at, updated_at
```

### 12.8 与 TS 端的主要差异总结

| 维度 | TS 端 | Python 端 |
|---|---|---|
| **上游处理** | 实时游标 + 逐条去重 + 逐 turn 主题分割 | 全量消息 → LLM 一次性分块 |
| **主题分割** | 线性，不支持非连续回溯 | 支持非连续（跳跃式对话） |
| **质量门控** | Rule filter + LLM 评估 + LLM 打分(0-10) | 无独立门控，由 LLM 提取时自行判断 |
| **Skill 搜索** | FTS + 向量 + LLM Judge 三阶段 | 查询改写 + top-5 搜索 |
| **生成流程** | 评估 → 搜索 → 创建/升级（多步） | 一步提取（或两步：骨架 + 详细） |
| **update 决策** | LLM Judge 判断是否同一 Skill | LLM 提取时直接返回 update=true |
| **附属文件** | scripts + references + evals | scripts + reference + others (zip) |
| **版本管理** | 完整版本历史（SkillVersion 表） | 覆盖式更新（old_memory_id） |

---

### 12.9 MOSCore.add() 同步/异步两条路径

**干什么**：Python 端消息入口 `MOSCore.add()` 有两种处理模式。同步模式在 `add()` 内一步完成切块+LLM精读+写入，延迟高但简单；异步模式只做粗存储，把精读任务提交给调度器后台处理，用户不用等 LLM，技能/偏好/工具轨迹提取全部在后台 4 线程并行执行。

```
                    同步模式 (sync)              异步模式 (async)
                    ─────────────              ──────────────────
add()中做什么?       切块 + LLM精读 + 写入       切块(仅Fast包装) + 写入粗记忆
调度器做什么?         仅日志                      驱动精读: 记忆/技能/偏好/工具 4并行
技能提取何时触发?     不触发(SimpleStruct)         在 fine_transfer_simple_mem() 中
                    或在add()内(MultiModal)       由调度器的 MEM_READ_TASK 触发
偏好提取何时触发?     add()中直接同步完成           由调度器的 PREF_ADD_TASK 异步触发
延迟                 高(用户等待LLM)              低(用户只等Fast切块，精读在后台)
```

**同步模式走线**：

```
MOSCore.add(messages)
  │  线程池并发 (max_workers=2)
  ├─ 线程1: process_textual_memory()
  │   └─ mem_reader.get_memory(mode="fine")
  │       ├─ coerce_scene_data()              ← 格式标准化 + 注入时间戳
  │       ├─ get_scene_data_info()            ← 预切分 (每10条消息一组，重叠2条)
  │       ├─ _iter_chat_windows()             ← token级滑动窗口切块
  │       └─ LLM提取结构化记忆 → 直接写入图+向量库
  └─ 线程2: process_preference_memory()
      └─ pref_mem.get_memory() → pref_mem.add()   ← 直接同步提取+写入偏好
```

**异步模式走线**：

```
MOSCore.add(messages)
  │  线程池并发 (max_workers=2)
  ├─ 线程1: process_textual_memory()
  │   └─ mem_reader.get_memory(mode="fast")   ← 不调LLM，直接包装为MemoryItem
  │       └─ scheduler.submit(MEM_READ_TASK)  ← 提交调度器精读 ★
  └─ 线程2: process_preference_memory()
      └─ scheduler.submit(PREF_ADD_TASK)      ← 提交调度器异步处理

调度器消费 MEM_READ_TASK → fine_transfer_simple_mem()
  4线程并行:
  ├─ 线程A: _process_string_fine()            ← LLM提取结构化记忆
  ├─ 线程B: process_skill_memory_fine()       ← 技能提取 ★
  ├─ 线程C: process_preference_fine()         ← 偏好提取
  └─ 线程D: _process_tool_trajectory_fine()   ← 工具轨迹提取
```

**关键纠正**：
1. 技能提取**不是调度器的独立任务类型**，而是在 `fine_transfer_simple_mem()` 内部 4 线程并行执行
2. 切块**发生在 `add()` 阶段**，调度器拿到的已经是切好的 memory IDs
3. 同步模式下**调度器不参与核心流程**

---

### 12.10 切块实现详解

**干什么**：Python 端有 6 种切块策略，针对不同场景和粒度需求，保证不同大小/类型的内容都能被正确切块而不丢失数据。

| 切块器 | 粒度 | 重叠方式 | 适用场景 |
|--------|------|---------|---------|
| `_iter_chat_windows` | token 级 (默认1024) | 弹出头部到 200 tokens | 主力记忆提取 |
| `get_scene_data_info` | 每10条消息 | 尾部 2 条 | 预切分（粗粒度，在token窗口之前执行） |
| Strategy `content_length` | 字符数 | 尾部 1 条消息 | 长消息场景 |
| Strategy `session` | N 条消息 | 步长滑动 | 固定窗口场景 |
| Splitter `lookback` | QA 对 | 回看 N 轮 | 偏好提取（高重叠，捕捉偏好演变） |
| Splitter `overlap` | 每10条消息 | 尾部 2 条 | 偏好提取（低重叠，快速切块） |
| SentenceChunker | 句子边界 | token overlap | 文档类消息 |

**MultiModal 主力切块的三层保底机制**（`_concat_multi_modal_memories`）：

```
保底1 — 单条超大: 超过 max_tokens 的 item 先用 SentenceChunker 拆小，拆失败原样保留
保底2 — 窗口不空: buf 为空时即使超了 max_tokens 也先放进去，保证每窗口至少一条
保底3 — 尾部不丢: 循环结束后 buf 里剩余内容全部输出，不截断
```

**切块前的预处理（保真度优先，轻量标准化）**：

| 步骤 | 做了什么 |
|------|---------|
| 格式标准化 | 补时间戳、拉平多模态 content |
| 消息格式化 | 拼成 `role: [time]: content\n` |
| URL 保护 | 替换为占位符防切断，完成后恢复 |
| 智能断点 | 优先在 `\n\n` / `。` / `. ` 处切分 |

---

### 12.11 偏好提取 (Preference Extraction)

**干什么**：从对话中并行提取两类偏好，分别存储，并通过去重和更新判断防止重复写入。与技能提取一起在异步模式的 4 线程中并行运行。

**核心文件**: `src/memos/mem_reader/read_pref_memory/process_preference_memory.py`

两类偏好**并行提取**：

**显式偏好（NAIVE_EXPLICIT_PREFERENCE_EXTRACT_PROMPT_ZH）**：用户明确表达的态度、选择、拒绝。要求提取偏好的演变过程（原始偏好 + 更新后偏好），同主题多个偏好合并为一条。

```
输出：[{explicit_preference, context_summary, reasoning, topic}]
```

**隐式偏好（NAIVE_IMPLICIT_PREFERENCE_EXTRACT_PROMPT_ZH）**：用户未直接表达，通过行为模式、决策逻辑、情境信号推断。限制：只有一轮问答不提取，Assistant 建议未被用户认可不提取。

```
输出：[{implicit_preference, context_summary, reasoning, topic}]
```

**去重流程**（两步）：

```
Step 1 — NAIVE_JUDGE_DUP_WITH_TEXT_MEM_PROMPT_ZH
  新偏好 vs 已有记忆（向量召回）→ 语义相似即判 exists=true

Step 2 — NAIVE_JUDGE_UPDATE_OR_ADD_PROMPT_ZH（exists=true 时触发）
  判断新旧偏好是否表达相同核心问题 → is_same=true → 更新，false → 新增
```

---

### 12.12 工具轨迹提取 (Tool Trajectory)

**干什么**：从工具调用历史中提取 `when...then...` 格式的经验规则，成功路径提炼最佳实践，失败路径分析根因并提炼避坑规则。每条工具调用记录 success_rate，供未来相似场景复用。类比 design-v2 中的 experience + tool_preference，但 MemOS 合并为一个 Prompt 处理。

**核心 Prompt**: `TOOL_TRAJECTORY_PROMPT_ZH`（`templates/tool_mem_prompts.py`）

```
步骤1：判断任务完成度 → success / failed（用户反馈优先于执行结果）

步骤2（success）：经验提炼
  采用 when...then... 结构：
  - when: 触发场景特征（任务类型、工具环境、参数特征）
  - then: 有效的参数模式、调用策略、最佳实践
  注意：经验是整个轨迹级别的，不仅针对单个工具

步骤3（failed）：错误分析
  3.1 任务是否需要工具？（需要/直接回答/误调用）
  3.2 工具调用检查：存在性、选择正确性、参数正确性、幻觉检测
  3.3 错误根因定位
  3.4 经验提炼（when...then... 给出避免错误的通用策略）
```

输出格式：

```json
[{
  "correctness": "success | failed",
  "trajectory": "任务 -> 执行动作 -> 执行结果 -> 最终回答",
  "experience": "when...then...格式",
  "tool_used_status": [{
    "used_tool": "工具名",
    "success_rate": "0.0-1.0",
    "error_type": "失败时的错误类型，成功为空",
    "tool_experience": "调用该工具的经验，含前置条件和后置效果"
  }]
}]
```

---

### 12.13 反馈修正闭环

**干什么**：处理用户对 Agent 回答的纠正反馈，将用户的纠正自动同步回记忆库。三步串行：先识别是否为关键词替换，再判断反馈有效性，最后决定对已有记忆执行 ADD/UPDATE/NONE 操作。

**核心文件**: `src/memos/mem_feedback/`（`templates/mem_feedback_prompts.py`）

**Step 1 — 关键词替换检测（KEYWORDS_REPLACE_ZH）**

识别用户是否在要求批量替换某个词（如"把文档里的'张三'都改成'李四'"），输出替换范围、原词、目标词。不是关键词替换则走后续判断流程。

```json
{"if_keyword_replace": "true/false", "doc_scope": "...|NONE", "original": "...", "target": "..."}
```

**Step 2 — 反馈有效性判断（FEEDBACK_JUDGEMENT_PROMPT_ZH）**

判断反馈是否与对话历史相关，识别用户态度（dissatisfied/satisfied/irrelevant），提取纠正后的核心事实信息（保留时间信息）。

```json
[{"validity": "true/false", "user_attitude": "dissatisfied|satisfied|irrelevant",
  "corrected_info": "事实信息", "key": "记忆标题", "tags": "关键词"}]
```

**Step 3 — 记忆更新操作（UPDATE_FORMER_MEMORIES_ZH）**

将新事实与已有记忆逐一对比，决定操作类型：

| 操作 | 条件 |
|------|------|
| `NONE` | 新事实未提供额外信息 |
| `UPDATE` | 新事实更准确/完整/需修正，仅修改相关错误片段 |
| `ADD` | 无匹配的已有记忆，作为全新信息写入 |

```json
{"operations": [{"id": "记忆ID", "text": "最终采用的记忆", "operation": "ADD|UPDATE|NONE", "old_memory": "..."}]}
```

---

### 12.14 记忆检索增强管线

**干什么**：查询时的记忆召回增强流程。先判断是否需要检索，多路召回后过滤冗余，对保留的记忆做消歧增强（代词→全名、相对时间→具体日期），不足时扩大召回，最终注入 System Prompt。

**检索流程**：

```
用户查询 → 意图识别 → 关键词提取 → 多路召回
                                      ├─ 向量检索 (Qdrant)
                                      ├─ 图检索 (Neo4j subgraph)
                                      ├─ BM25 全文检索
                                      └─ 互联网检索 (可选)
                                           │
                                           ▼
                                    记忆过滤 + 去重 + 重排序
                                           │
                                           ▼
                                    记忆增强 (消歧/合并/补全)
                                           │
                                           ▼
                                    注入 System Prompt → LLM 生成回答
```

**4 个 Prompt**：

**意图识别（INTENT_RECOGNIZING_PROMPT）**：判断当前工作记忆是否足够回答用户问题，输出 `trigger_retrieval=true/false` + 已有证据 + 缺失证据类型。

**记忆增强（MEMORY_RECREATE_ENHANCEMENT_PROMPT）**：将召回的原始记忆转换为完全消歧的陈述。代词→全名，相对时间→具体日期，合并互补细节，过滤与查询无关内容。

**联合过滤（MEMORY_COMBINED_FILTERING_PROMPT）**：两步过滤：① 移除与查询完全无关的记忆；② 冗余去重，保留最简洁且与查询最相关的版本。输出保留记忆的索引列表 + 移除统计。

**扩大召回（ENLARGE_RECALL_PROMPT_ONE_SENTENCE）**：分析现有记忆与用户查询的差距，识别缺失事实，生成一句改写查询用于二次检索。`trigger_recall=false` 时不触发。

---

## 十三、LLM Prompt 完整清单

### TS 端

| # | Prompt 名称 | 文件 | temp | max_tokens | timeout | 用途 |
|---|---|---|---|---|---|---|
| 1 | Summarize | providers/*.ts | 0 | 100 | 30s | 消息摘要 |
| 2 | DEDUP_JUDGE_PROMPT | providers/*.ts | 0 | 300 | 15s | 语义去重 |
| 3 | TOPIC_CLASSIFIER_PROMPT | providers/*.ts | 0 | 60 | 15s | 主题分类 |
| 4 | TOPIC_ARBITRATION_PROMPT | providers/*.ts | 0 | 10 | 15s | 主题仲裁 |
| 5 | TASK_SUMMARY_PROMPT | providers/*.ts | 0.1 | 4096 | 60s | 任务摘要 |
| 6 | TASK_TITLE_PROMPT | providers/*.ts | 0 | 100 | 30s | 任务标题 |
| 7 | Skill Relevance Judge | evolver.ts | 0 | 256 | 30s | 搜索相关 Skill |
| 8 | CREATE_EVAL_PROMPT | evaluator.ts | 0.1 | 1024 | 30s | 创建评估 |
| 9 | UPGRADE_EVAL_PROMPT | evaluator.ts | 0.1 | 1024 | 30s | 升级评估 |
| 10 | STEP1_SKILL_MD_PROMPT | generator.ts | 0.2 | 6000 | 120s | 生成 SKILL.md |
| 11 | STEP2_SCRIPTS_PROMPT | generator.ts | 0.1 | 3000 | 120s | 提取脚本 |
| 12 | STEP2B_REFS_PROMPT | generator.ts | 0.1 | 3000 | 120s | 提取参考文档 |
| 13 | STEP3_EVALS_PROMPT | generator.ts | 0.3 | 2000 | 120s | 生成测试用例 |
| 14 | UPGRADE_PROMPT | upgrader.ts | 0.2 | 6000 | 90s | 升级合并 |
| 15 | Upgrader scripts rebuild | upgrader.ts | 0.1 | 3000 | 60s | 重建脚本 |
| 16 | Upgrader evals rebuild | upgrader.ts | 0.3 | 2000 | 60s | 重建测试 |
| 17 | Upgrader refs rebuild | upgrader.ts | 0.1 | 3000 | 60s | 重建参考 |
| 18 | QUALITY_PROMPT | validator.ts | 0.1 | 1024 | 30s | 质量评分 |
| 19 | FILTER_RELEVANT_PROMPT | providers/*.ts | 0 | 200 | 15s | 召回相关性过滤 |

### Python 端

| # | Prompt 名称 | temp | 用途 |
|---|---|---|---|
| 1 | TASK_CHUNKING_PROMPT / _ZH | — | 任务分块 |
| 2 | TASK_QUERY_REWRITE_PROMPT / _ZH | — | 查询改写 |
| 3 | SKILL_MEMORY_EXTRACTION_PROMPT / _ZH | — | 简单提取 |
| 4 | SKILL_MEMORY_EXTRACTION_PROMPT_MD / _ZH | — | 完整提取 |
| 5 | SCRIPT_GENERATION_PROMPT | — | 生成脚本 |
| 6 | TOOL_GENERATION_PROMPT | — | 识别工具 |
| 7 | OTHERS_GENERATION_PROMPT / _ZH | — | 生成参考文档 |

---

## 十四、关键设计洞察

### 14.1 TS 端的"任务驱动"提取

与 AutoSkill 的"滑动窗口"不同，MemOS TS 端是**任务驱动**的：

```
AutoSkill: 每轮取 messages[-6:] → 提取 → 下一轮    （窗口驱动）
MemOS:     累积消息 → 主题切换时 finalize → 提取     （任务驱动）
```

- AutoSkill 的优势：延迟低，每轮都可能提取，渐进进化
- MemOS 的优势：有完整任务上下文，提取质量更高，不会跨任务混合

### 14.2 Python 端的"非连续回溯"

```
AutoSkill: 线性窗口，不回溯
MemOS TS:  线性切割，不回溯
MemOS Py:  LLM 聚类，支持非连续回溯

例：messages 8-11 = 旅行, 12-22 = 编程, 23-24 = 旅行
MemOS Py: [8-11, 23-24] → "旅行规划" 同一个任务
MemOS TS: [8-11] → 任务1, [12-22] → 任务2, [23-24] → 可能新任务3
```

### 14.3 "通用化"原则

MemOS 最强调的设计原则。所有 Prompt 都反复要求：

```
- "Travel Planning" 而非 "Beijing Travel Planning"
- 提取可跨场景应用的抽象方法论
- 除 examples 外所有字段保持通用
- 同场景合并：不要已有"饮食规划"又新增"生酮饮食规划"
```

对比 AutoSkill 的"去标识化"（placeholder 替换），MemOS 的通用化更激进——不只是去掉实体名，而是要求把整个方法论抽象到可复用的层次。

### 14.4 两级触发机制（TS 端）

```
L0: description（始终在上下文中，~100 词）
    → 决定是否触发 Skill
L1: SKILL.md 完整内容（触发后加载）
    → 提供完整指导
L2: scripts/ references/（按需加载）
    → 可执行资源
```

这与 Anthropic 官方 Skill-Creator 和 Superpowers 的三级加载架构一致。description 的写法是"proactive trigger"——不只描述做什么，还要列出所有应该触发的场景和关键词。

### 14.5 质量门控差异

```
AutoSkill:  提取 → decide(add/merge/discard) → 能力身份判断 → 合并
            无独立质量评分，靠 decide 阶段的 LLM 判断

MemOS TS:   Rule filter → LLM 评估(0-1 confidence) → 生成 → 验证(0-10 score)
            score < 6 → draft（不会被自动注入）
            升级时有回归检查（缩水 >30% → 警告 → 可回退）

MemOS Py:   无独立质量门控
            由 LLM 提取时自行判断 null（无法提取）
```

### 14.6 证据使用对比

```
AutoSkill:
  USER turns = 证据来源（source of truth）
  ASSISTANT turns = 仅参考上下文
  弱确认 ≠ 验证

MemOS TS:
  完整对话（user + assistant + tool）都是输入
  无显式的证据溯源原则
  但有 tool 输出截断（1500 字符）和敏感数据脱敏

MemOS Py:
  messages = 主要来源
  chat_history = 辅助上下文（有严格使用约束）
  不得作为 skill 的主要来源，仅补充 preference/experience
```
