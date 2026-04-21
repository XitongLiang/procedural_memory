# Spec 004 — 程序记忆学习模块（迭代二，v5）

> 最后更新：2026-04-20
> 核心变更：程序记忆分类重构，移除 workflow，引入 task-specific experience / feedback / tool preference 三元结构。
> **v5 新增**：Task Trace（结构化执行轨迹）作为原子事实层，Fb/TP/TSE 均从 Task Trace 提取。
> 命名调整（2026-04-20）：模块前缀 `skill`/`pmem` 统一改为 `procedural`，相关 DB 表加 `procedural_` 前缀（如 `procedural_tse_meta` / `procedural_fb_meta` / `procedural_tp_meta`）。不做老 DB 兼容，需清空重建。
>
> **v5 主要变更（2026-04-19）**：
> - **单轨道单水位**：Fb/TP 和 TSE 合并进 chunk flush 统一提取，不再双轨道。`procedural_session_progress` 移除 `kind` 列，per-session 一条水位。
> - **Fb/TP 提取时机后移**：从 per-window 移到 chunk flush，以 chunk 的主题为锚点，LLM 上下文更完整。
> - **滑动窗口参数收紧**：n=10/step=8 → **n=8/step=6**。触发门槛冷启动 8 / 热启动 6 / idle 20 min。
> - **Topic 缓存表 `procedural_topic_cache`**：同一窗口复用 LLM 结果，flush 覆盖后 `DeleteByRange` 清理，7 天 GC。
> - **Topic 多出一字段 `topic_summary`**（15-40 字自然语言句子）：供 Fb/TP 提取 prompt 作为 context 锚点，稳定跨 session 的相似反馈归类。

---

## 0. 概述

### 目标

从 agent 真实工作历史中**自动提取并演进程序记忆**。v5 架构分为**两层**：

- **过程层**（`task_trace`）：原子事实，记录"做了什么"，为下游提取提供去噪后的统一证据
- **知识层**（`task_specific_experience` / `feedback` / `tool_preference`）：从过程层提炼的可复用知识

| 类型 | 存储内容 | 核心价值 |
|------|---------|---------|
| `task_trace` | **原子事实层**：结构化执行轨迹（User Query + Step-by-step 操作记录），从原始对话整理而来 | 去噪后的统一证据源，供 Fb/TP/TSE 提取消费；离线 dreaming 的原材料 |
| `task_specific_experience` | 任务执行经验：情境化的决策参考、技巧组合、风险规避、失败恢复路径、用户主动提出的任务要求与上下文依赖 | 提升同类任务的执行质量，避免重蹈覆辙 |
| `feedback` | 错误更正记忆：用户纠正或 agent 自身纠错后形成的**硬约束**行为规则 | 避免重复犯错，优先级最高 |
| `tool_preference` | 工具选择偏好：做某类任务优先用哪个工具 | 减少决策开销，提升执行效率 |

> **应用原则**：`task_specific_experience` 在召回后作为**情境化建议**供 agent 参考，agent 自行判断是否采纳；`feedback` 则是必须遵循的硬约束。

## 1. 整体架构

```
触发：定时器每 120s 扫一次（TimerProcessProceduralCache）
输入：mem_conversation 里水位之后的对话行

─────────────────────────────────────────────────────────────────────
Step 1  预过滤（零 LLM）
          第一层：全量内容清洗（哨兵/注入/敏感脱敏/推理标签）
          第二层：窗口级 tool 输出截断（60/30 规则）
          （v5 当前实现暂时短路，由 autoCapture 侧做基本清洗）

Step 2  生成滑动窗口（n=8, m=2, step=6）
          冷启动：攒够 8 条新 turn → 1 个满窗口
          热启动：6 条新 turn + 2 条 overlap → 1 个满窗口

Step 3  窗口主题抽取（小模型 + DB 缓存）
          先查 procedural_topic_cache(session_id, first_turn_id,
          last_turn_id)；命中直接复用，未命中调 LLM 并回填缓存。
          提取字段：
            - topic_keywords（关键词列表）
            - topic_tag（主题标签，1-3 个）
            - task_category（coding / planning / debugging / writing / other）
            - topic_summary（15-40 字自然语言一句话，v5 新增，
              供 Fb/TP 作为 context 锚点；无明确任务时空串）

Step 4  主题拼接状态机
          每窗口进来比较 topic_keywords 与 chunk 缓存：
            - Jaccard > 0.3 → 同主题，追加进 chunk，更新 topic
            - Jaccard ≤ 0.3 → 小模型 JudgeTopicBoundary 兜底
            - chunk 达 3 窗口 / 2h gap → 强制 flush
          chunk 仅存在 InstanceWorker 内存，不落 DB；重启缓存丢失
          不丢数据（水位未推 → 下次重拉重组）。

Step 5  Chunk flush（Task Trace + Fb/TP + TSE，v5 核心变更）
          flush 时对 chunk 一次性跑：

          (A) Task Trace 提取（v5 新增）
              ExtractTaskTrace(chunk + topic 锚点)
              → 结构化执行记录（User Query + Step-by-step）
              → 写入 mem_record(category=TASK_TRACE)
              → 下游 Fb/TP/TSE 均从 Task Trace 消费

          (B) Fb/TP 提取（从 Task Trace 提取）
              LearnFeedbackAndPreference
              以 chunk 的 topic_tag + task_category + topic_summary
              为 context 锚点；Task Trace 为主要证据。
              两层去重：
                Layer 1：active 表（procedural_fb_meta / procedural_tp_meta）
                         向量召回 + hybrid score
                         （0.70·cos + 0.18·kw + 0.12·nameJaccard）
                  ≥ 0.85 → 直接 merge 覆盖 active
                  < 0.50 → 进 Layer 2
                  0.50–0.85 → 小模型决策 merge/add
                Layer 2：pending 候选池（procedural_candidates）
                         kind+name 精确匹配
                  命中 → mention++/hard++ → 重算 support_score
                  未命中 → 新建 pending
                  support_score ≥ 0.60 → 升格为 active

          (C) TSE 提取（从 Task Trace 提取）
              ExtractChunkOverview(Task Trace) → task_summary / tools_used /
              key_steps / context_tags
              → SearchTseForChunk (hybrid 召回 top-5)
                   → JudgeTaskExperienceMatch（小模型选最佳）
                      命中 → EnrichTaskExperience
                      未命中 → LearnTaskExperience → TseDedupAndSave

          (D) MaintainPendingPool：扫描 pending 候选做升格/清理

Step 6  水位推进
          只有 flush 真实发生时才推进到 chunk 覆盖的最后 turn id；
          未 flush 的 chunk 缓存留在内存，水位原地不动，下次 tick
          从水位重新拉 turn 重组。flush 成功后
          StoreProceduralTopicCacheDeleteByRange 清理被覆盖区间
          的 topic 缓存，7 天 GC 兜底。
─────────────────────────────────────────────────────────────────────
```

**v5 关键变化对比 v4：**

| 项 | v4 | v5 |
|----|----|----|
| **Task Trace（新增）** | 无 | chunk flush 时先提取结构化执行轨迹，Fb/TP/TSE 均从 Task Trace 消费 |
| Fb/TP 提取时机 | per-window | chunk flush 时 |
| 主题锚点字段 | topic_tag + task_category | + topic_summary（一句话）|
| 滑动窗口 | n=10, step=8 | **n=8, step=6** |
| 水位粒度 | `(session_key, kind)` 双轨道 | `session_key` 单水位 |
| Topic LLM 缓存 | 无 | `procedural_topic_cache` 复用 |
| 触发门槛 | newCount ≥ 8 | 冷 ≥ 8 / 热 ≥ 6 / idle 20min |
| 水位推进时机 | 每次调 pipeline 即推 | 仅 flush 成功才推 |

---

## 2. 参数

### 2.1 窗口 & chunk

| 参数 | 值 | 说明 |
|------|----|------|
| `PROCEDURAL_LEARN_WINDOW_N` | **8** 轮 | 窗口大小（1 轮 = user + assistant 一对） |
| `PROCEDURAL_LEARN_WINDOW_M` | 2 轮 | 相邻窗口重叠轮次 |
| `PROCEDURAL_LEARN_WINDOW_STEP` | **6** 轮 | 滑动步长（n - m） |
| `max_windows_per_chunk` | 3 | 主题拼接上限（强制 flush） |
| `max_gap_hours` | 2 | 时间间隔上限（强制 flush） |
| `jaccard_threshold` | 0.3 | 主题相似度快速门控阈值 |
| `candidate_match_threshold` | 0.70 | 候选池向量匹配阈值 |

v5 窗口分布（N=24 轮示例）：

```
W0: [0,  8)
W1: [6, 14)
W2: [12, 20)
W3: [18, 24)  ← 最后一个对齐到 N
```

### 2.2 Worker 触发

| 参数 | 值 | 常量 |
|------|----|------|
| 定时器周期 | 120 s | `INSTANCE_WORKER_PROCEDURAL_MS` |
| 冷启动门槛 | 8 新 turn | `..._MIN_TURNS_COLD` |
| 热启动门槛 | 6 新 turn | `..._MIN_TURNS_STEP` |
| Idle 兜底 | 20 min | `..._IDLE_MS` |

### 2.3 Hybrid 去重阈值（Fb/TP/TSE 共用）

| 区间 | 动作 |
|------|------|
| ≥ 0.85 | 直接 merge（RECENCY BIAS 覆盖） |
| 0.50 – 0.85 | 小模型灰区决策 |
| < 0.50 | add（Fb/TP 进第二层候选池；TSE 直接新建） |

Hybrid Score = 0.70·cosine + 0.18·keyword_overlap + 0.12·name_Jaccard。

---

## 3. Step 1 — 预过滤（零 LLM）

两层执行：**第一层**在生成窗口前对全量消息做内容清洗；**第二层**在每个窗口传给 LLM 前做噪声压缩。

### 3.1 第一层：内容清洗（全量，进窗口前）

顺序执行，不可逆地改写消息内容。

**整轮移除**（满足任一条件则移除该轮 user + assistant）：

| 过滤对象 | 识别方式 |
|----------|----------|
| system role 消息 | role = "system" |
| 哨兵/心跳消息 | content = "NO_REPLY" \| "HEARTBEAT_OK" \| "HEARTBEAT_CHECK" |
| 系统样板 | content 以 "/new" 或 "/reset" 开头，或含 "You are running a boot check" |
| 记忆系统注入 | tool_result name = context_assemble / skill_search / memory_search |
| 机械回复 | user ≤ 10 词 + 匹配关键词（好的/继续/ok/yes/下一步/确认） |

**消息内容改写**（保留轮次，仅清洗内容）：

| 过滤对象 | 识别方式 | 处理 |
|----------|----------|------|
| 推理标签 | `<think>...</think>` | 整段移除 |
| OpenClaw 元数据块 | `Sender (untrusted metadata): {...}` | 移除 |
| 注入的记忆上下文 | `<memory_context>...</memory_context>` | 移除 |
| 时间戳前缀 | `[Mon 2026-04-13 10:00 GMT+8]` 开头 | 移除前缀 |
| 消息 ID 标签 | `[message_id: ...]` | 移除 |
| 敏感数据 | 见下方脱敏规则 | 正则替换 |

**敏感数据脱敏**（正则，进 LLM 前必做）：

```
sk-[A-Za-z0-9]{20,}             → sk-***REDACTED***
Bearer [A-Za-z0-9+/=]{20,}      → Bearer ***REDACTED***
AKIA[A-Z0-9]{16}                → AKIA***REDACTED***
(api_key|secret|token|password)\s*=\s*["'][^"']+["']  → ="***REDACTED***"
/Users/[^/]+/                   → /Users/****/
```

**语言检测**（写入 session 元数据，供 LLM prompt 使用）：

```
CJK 字符数 / 总字符数 > 15% → lang = "zh"
否则                         → lang = "en"
```

---

### 3.2 第二层：窗口级噪声压缩（每窗口传 LLM 前）

**整轮移除**：

| 过滤对象 | 识别方式 | 处理 |
|----------|----------|------|
| 探索性工具调用 | tool name 在白名单（ls/pwd/git_status/cat/head/echo）且无报错 | 移除 tool_use + tool_result 对 |

**内容截断**：

| 过滤对象 | 识别方式 | 处理 |
|----------|----------|------|
| tool_result（无报错） | exitCode = 0 或无 exitCode | 60% 头部 + 30% 尾部，总上限 1500 字符，中间插 `[... 截断 N 字符 ...]` |
| 报错 tool_result | exitCode != 0 或含 stderr/error | **完整保留**（经验来源） |
| 文件读取结果 | tool name = Read/cat/head，无报错 | 保留前 5 行 + `[截断]` |
| assistant 代码块 | ` ``` ` 包裹，无报错上下文 | 保留前 10 行 + `[截断]` |

**60/30 截断规则说明**（来自 MemOS）：

```
原始内容 = 5000 字符
  头部 = 前 900 字符（60% × 1500）← 含命令/参数
  尾部 = 后 450 字符（30% × 1500）← 含执行结果
  中间 = [... 截断 3650 字符 ...]
```

尾部保留是关键——tool_result 的最终状态（成功/失败/输出）通常在末尾。

---

## 4. Step 2 — 生成滑动窗口

输入是 Step 1 第一层已清洗完毕的 `clean_turns`（元数据已剥离、敏感数据已脱敏、噪声轮次已移除）。

```python
windows = []
i = 0
while i < len(clean_turns):
    end = min(i + n, len(clean_turns))
    windows.append(clean_turns[i:end])
    if end == len(clean_turns): break
    i += step
```

第二层压缩（tool_result 截断等）在每个窗口**传给 LLM 前**单独执行，不修改 `clean_turns` 本身。

---

## 5. Step 5 — Chunk Flush（Fb/TP + TSE + Task Trace，v5 核心变更）

> **v5 变更**：Fb/TP 提取不再 per-window 执行，改为 chunk flush 时一次处理。
> **v5 新增**：chunk flush 前先提取 **Task Trace**（结构化执行轨迹），Fb/TP/TSE 均从 Task Trace 提取，不再直接消费原始 chunk。

---

### 5.0 Task Trace 提取（Step 4.5，v5 新增）

在主题拼接完成、chunk 即将 flush 时，先将 chunk 的原始对话整理为结构化的 Task Trace，作为下游 Fb/TP/TSE 提取的统一输入。

#### 5.0.1 设计意图

原始对话是 flat 的消息流，包含大量噪声（问候、 filler、重复确认）。直接用它作为 Fb/TP/TSE 的提取输入会导致：
- LLM 注意力被分散到无关消息
- 多次提取时重复解析同一对话结构
- 用户纠正信号和执行步骤混杂，不易定位

Task Trace 作为**原子事实层**：
- 把原始对话整理为"逐步做了什么"的结构化记录
- 保留工具调用、参数、结果、用户纠正等关键信息
- 去噪但不丢失证据（filler 移除，约束保留）
- 一次整理，多次消费（Fb/TP/TSE 共享）

类比 Mem0 的 Procedural Memory 快照，但粒度更粗（以 turn 为单位而非每步操作），且保留 topic 锚点用于检索。

#### 5.0.2 输入

```
Chunk 原始消息（已过滤、已脱敏）
Topic 锚点：
  topic_tag: <chunk 主题标签>
  task_category: <任务分类>
  topic_summary: <15-40 字自然语言>
```

#### 5.0.3 提取 Prompt

```
[System]
你是一个对话整理专家。将以下 agent 工作对话整理为结构化的 Task Trace。

## 整理目标

把原始对话转化为"逐步执行记录"，保留所有对后续复盘有价值的操作细节，移除 filler 和噪声。

## 输出结构

```markdown
## Task Trace

**User Query**: {从 Primary User Questions 中提取的汇总请求，1-2 句话}
**Topic**: {topic_tag} | {task_category}
**Topic Summary**: {topic_summary}
**Time Range**: {first_turn_time} → {last_turn_time}

### Step 1 — [timestamp] User Request
- Action: {用户请求或补充要求}
- Context: {相关上下文}

### Step 2 — [timestamp] Agent Action / Tool Call
- Action: {agent 的思考或工具调用}
- How: {具体做法，含工具名、参数}
- Parameters: {关键参数（完整版保留）}
- Result: {工具返回或执行结果（成功/失败/输出摘要）}

### Step 3 — [timestamp] User Correction (if any)
- Action: {用户纠正、否定、换方案}
- Type: correction | feedback | refinement
- Trigger: {触发纠正的前置问题}

...
```

## 整理规则

1. **User Query 提取**：从所有 user 消息中提取一个汇总的 "User Query"，放在最前面。这是用户最初的目标请求，后续所有操作都围绕它展开。
2. **按时间顺序**：每个有意义的 turn 对应一个 Step（user request / agent thinking / tool call / result / user correction）
3. **合并 filler**："好的"/"继续"/"ok" 等无信息量的回合合并到相邻 Step，不单独列出
4. **保留参数**：工具调用的关键参数保留（去敏感后），方便后续复盘
5. **标注纠正**：用户说"不对"/"改用"/"不要"等 → 明确标注为 correction Step，记录触发原因
6. **结果摘要**：tool_result 保留核心结论（成功/失败/关键输出），过长内容可截断
7. **不发明内容**：只整理对话中实际发生的内容

[User]
Topic 锚点：
topic_tag: {topic_tag}
task_category: {task_category}
topic_summary: {topic_summary}

原始对话：
{chunk_messages}
```

**输出**：Markdown 格式的 Task Trace 文本，直接作为字符串写入 `mem_record.content`。

#### 5.0.4 存储

Task Trace 存入 `mem_record` 主表：

- `memory_type = PROCEDURAL(2)`
- `category = TASK_TRACE(13)`（v5 新增 category）
- `tier = LEAF(2)`
- `content = Task Trace Markdown 全文`

**不建扩展表**，全部信息塞进 `content`（Markdown 自描述）。

**向量写入**：Task Trace 本身**不参与向量检索**（它是对话的记录而非可复用知识），但写入时生成 embedding 供后续"同类任务 trace 聚类"使用（离线 dreaming 阶段）。embedding 输入取 `"{topic_summary} {user_query}"`。

**写入向量表**：`mem_record_raw_vec_13`（LEAF 层按 category 分表，v5 新增）。

#### 5.0.5 生命周期

- **在线**：chunk flush 时同步提取并写入
- **离线**：积累足够多同类任务的 trace 后，可用于跨任务模式挖掘（v6 预留：workflow 抽象）
- **清理**：30 天 TTL，过期自动删除（Trace 是过程记录，非持久资产）

---

### 5.1 提取 Feedback + Tool Preference（从 Task Trace）

一次大模型调用，同时从 Task Trace 中提取 feedback 和 tool_preference。

**分层输入结构**：
```
Topic（主题锚点）
  topic_tag: <chunk 主题标签>
  task_category: <任务分类>
  topic_summary: <15-40 字自然语言>

Task Trace ← 结构化执行记录（§5.0 产出，主要证据）
  含 User Query、Step-by-step 记录、correction 标注

Raw User Messages ← Chunk 内用户原始消息（辅助证据，用于验证纠正信号）
```

> 主证据从 Task Trace 读取（已结构化，LLM 定位纠正/工具模式更高效）；Raw User Messages 作为辅助，用于验证 Task Trace 中 correction 标注的准确性。

`topic_summary` 是 v5 新增的 context 锚点：词标签归类粒度粗，一句话自然语言能让 LLM 把 Fb 的 `context` 字段（"在 XX 场景下..."）稳定对齐到主题语义，减少跨 session 同类反馈的 context 字符串漂移，进而提升 dedup 命中率。

**LearnFeedbackAndPreference Prompt**：

```
你是一个程序记忆提取专家。从以下 Task Trace 中提取 feedback 和 tool_preference。

## 知识类型定义

**feedback**（错误更正记忆）— 错误发生后形成的硬约束行为规则，agent 必须遵循：
- 来源一（用户纠正）：用户亲口说"不要用 X"、"错了，应该 Y"、"顺序不对"等
- 来源二（agent 纠错）：agent 执行失败后自行换方案并成功恢复的路径（"出现该报错时，改用此做法"）
- 内容：工具选择纠正、步骤/顺序纠正、失败恢复路径、风格/格式纠正
- 优先级最高
- 格式："[被纠正的行为] → [正确做法]" 或 "[错误情境] 时，改用 [恢复做法]"
- 长度：description ≤ 50 字
- is_hard = true
- source_user_correction = true（用户纠正） / false（agent 自身纠错）

**tool_preference**（工具偏好）— 同类任务上的稳定工具选择倾向：
- 目标：减少决策开销，提升执行效率
- 格式："做 [任务类型] 时，优先用 [工具]"
- 注意：互补工具组合（"用 A 后配合 B"）→ 不归此类，应忽略
- 长度：description ≤ 50 字
- is_hard = true（如果来自用户纠正）或 false（如果来自 agent 自发规律）

## 分析框架

**1. 用户纠正分析**（从 Task Trace 中定位 correction Step）
- Task Trace 中已标注 "Type: correction" 的 Step 是主要分析对象
- 核对 Raw User Messages 验证纠正信号的真实性
- 提取结果 → feedback（is_hard=true，source_user_correction=true）
- 若纠正涉及工具选择 → 同时产出 tool_preference（is_hard=true）

**2. 失败→恢复分析**
- 扫描 Task Trace 中 Result 为失败的 Step，看后续是否有成功恢复的 Step
- 关键：该错误是否可能在下次同类任务中重现？
- 可泛化的恢复路径 → feedback（is_hard=true，source_user_correction=false）
- 一次性错误/环境偶发错误 → 不提取

**3. 工具选择模式分析**
- 同类任务是否反复选择同一工具（≥ 3次）？→ tool_preference（is_hard=false）
- 是否有明确的"优先用 X 不用 Y"模式？→ tool_preference

## 提取规则

- 以 Task Trace 为主要提取证据（已结构化，定位高效）
- Raw User Messages 仅作验证辅助（核对纠正信号）
- 去标识化：用占位符替代具体文件名/项目名/用户名
- 一次性、不可泛化的操作 → 不提取
- 纯闲聊/无操作内容 → 不提取

## 输出格式

只输出 JSON：
{
  "items": [
    {
      "kind": "feedback" | "tool_preference",
      "name": "<简短标题，5字以内>",
      "context": "<适用场景标签，1-3个>",
      "description": "<规则描述，≤50字>",
      "is_hard": true | false,
      "source_user_correction": true | false
    }
  ]
}

无可提取内容 → {"items": []}

[User]
Task Trace（主要证据）：
{task_trace}

Raw User Messages（验证辅助）：
{raw_user_messages}
```

### 5.2 is_hard 规则辅助验证（零 LLM）

LLM 输出 `is_hard=true` 或 `source_user_correction=true` 时，验证 Primary User Questions 中是否存在否定词+操作词模式：

```
否定词：不要/别/不能/不该/不应该/错了/换成/不对/先用/后做
操作词：工具名/命令名/动词短语
匹配 → 确认
不匹配 → 降级为 is_hard=false，source_user_correction=false
```

### 5.3 两层查重 + 候选池更新

提取出的候选条目统一进入 §8 的候选池流程处理。

---

## 6. Step 3/4 — 主题抽取 + 主题拼接

v5 起主题抽取为所有 chunk flush（Fb/TP + TSE）共享的锚点来源。

### 6.1 窗口主题提取（小模型 + DB 缓存）

调用入口 `ExtractTopicMetadataCached`（pipeline.c）——先查 DB 缓存，miss 才调 LLM。

**提取字段**：

| 字段 | 说明 |
|------|------|
| `topic_keywords` | 3-8 个 top-N 关键词，用于 Jaccard 主题边界判断 |
| `topic_tag` | 1-3 个主题标签（"部署"、"API 调试"）|
| `task_category` | 单选 `coding / planning / debugging / writing / other` |
| `topic_summary` | 15-40 字自然语言一句话（**v5 新增**），Fb/TP 锚点 |

Prompt 文件：`src/prompts/files/procedural/topic_extract.{zh,en}.md`（Prompt ID `GSPD_PROMPT_PROCEDURAL_TOPIC_EXTRACT`）。

降级：LLM 不可用时走零 LLM TF 关键词提取路径，`topic_tag` 空、`task_category="other"`、`topic_summary` 空串。

### 6.2 主题缓存表 `procedural_topic_cache`（v5 新增）

**背景**：v5 水位只在 flush 时推进，未 flush 的 chunk 在下次 tick 会被重新组窗——如果不缓存，同一窗口会反复调 LLM 提主题。

```sql
CREATE TABLE procedural_topic_cache (
    session_id     TEXT    NOT NULL,
    first_turn_id  INTEGER NOT NULL,
    last_turn_id   INTEGER NOT NULL,
    topic_tag      TEXT,
    task_category  TEXT,
    keywords_json  TEXT,      -- JSON array
    topic_summary  TEXT,      -- v5 一句话
    created_at_ms  INTEGER NOT NULL,
    PRIMARY KEY (session_id, first_turn_id, last_turn_id)
);
```

**生命周期**：
- Get：命中直接复用（跳过 LLM）
- Upsert：miss 后回填
- DeleteByRange：chunk flush 后清理 `last_turn_id <= chunk_last_id` 的所有缓存
- GC：7 天未命中的老条目兜底清理

### 6.3 主题关键词提取（零 LLM 降级）

`topic_keywords` 的零 LLM 兜底路径（LLM 不可用时走）：

```
取窗口内所有 user 消息
  → 分词（中文分词 or 空格分割）
  → 过滤停用词（好的/继续/ok/是的/不对/帮我/可以...）
  → 过滤短词（< 2 字符）
  → top-N TF 词 → topic_keywords
```

### 6.4 主题边界判断（Jaccard 快速门控 + 小模型兜底）

```
overlap = |keywords_A ∩ keywords_B| / |keywords_A ∪ keywords_B|

overlap > 0.6  → 同主题，直接合并（高置信，跳过 LLM）
overlap ≤ 0.6  → 调用小模型判断（关键词不重叠不代表主题不同，
                  可能只是表述偏移，需要语义理解）
```

**小模型主题判断 Prompt**：

```
[System]
判断以下两段对话是否属于同一个任务主题。
只输出 JSON：{"same_topic": true|false, "reason": "..."}

[User]
片段 A 关键词：{chunk_keywords}
片段 B 关键词：{next_window_keywords}
片段 A 最后一条用户消息：{chunk_last_user_msg}
片段 B 第一条用户消息：{next_first_user_msg}
```

`same_topic=true → 合并`，`false → 切断`

### 6.5 Chunk 积累与 Flush

```
chunk = [W_current]  (内存状态，不落 DB)

for 下一个窗口 W_next:

  act = DecideChunkAction(chunk, W_next.firstMs,
                          W_next.keywords)

  MERGE        → chunk.merge(W_next)       （Jaccard > 0.3）
  LLM_DECIDE   → JudgeTopicBoundary（LLM）
                   sameTopic → merge
                   else      → flush + merge
  FLUSH        → flush + merge             （非同主题）
  FORCED       → flush(forced=1) + merge   （达 3 窗口 / 2h gap）

v5 兜底规则：
  正常 tick 末尾 → 不强 flush 残余 chunk，水位不动，下次重组
  forceSingleChunk=1（memory_flush 同步屏障）→ 强制 flush 残余
```

**v5 变更**：
- 水位只在真实 flush 时推进（`lastFlushedIdx` 跟踪）。未 flush 的 chunk 留在内存，重启缓存丢失不丢数据——下次 tick 从水位重拉重组。
- memory_flush 同步屏障路径会叠加 `LTM_PROCEDURAL_PIPELINE_FLAG_FORCE_SINGLE_CHUNK`，把所有窗口合成一 chunk 一次 flush，跳过主题边界判断。

### 6.6 窗口质量过滤（普通 flush）

设计意图：满足以下任一条件才值得提取，否则 skip：
- 有工具调用（任意 tool_use）
- assistant 回复总长 ≥ 200 字符

> **v5 实现状态**：autoCapture 把整批消息打成单条 `role=USER` 的 JSON blob 存进 `mem_conversation`，`IsWindowHighQuality` 永远命中 0 → 当前代码**临时短路质量过滤**（`ChunkFlushV4` 里 `(void)forced;`），由上游触发条件（newTurns / idle / flush）控场。待 ingest 拆行或 worker 层解析 JSON 后恢复。

---

## 7. Step 5 — Chunk 提取 Task-specific Experience

### 7.1 工作概览提取（小模型）

从 Task Trace 中提取结构化工作概览：

```
[System]
从以下 Task Trace 中提取工作概览，用于后续检索和提取任务经验。

输出 JSON：
{
  "task_summary": "<一句话描述用户要完成的任务，基于 User Query>",
  "tools_used": ["<使用的工具/命令列表>"],
  "key_steps": ["<关键操作步骤，3-5 条>"],
  "context_tags": ["<适用场景标签，2-4 个>"]
}

[User]
{{task_trace}}
```

### 7.2 匹配已有经验

用 `task_summary` + `context_tags` 作为检索输入，经 procedural/learn 层
`TseSearchByEmbed`（内部走 `StoreSemanticSearch(filter.category=EXPERIENCE)`，
精确命中 `mem_record_raw_vec_10`）：

```
得分 = 0.70 × 向量余弦相似度
     + 0.18 × 关键词重叠（signal_overlap）
     + 0.12 × 名称 token Jaccard（name_similarity）

取 top-5，score < 0.35 的过滤掉
```

### 7.3 Judge（小模型）

判断当前 Chunk 的任务是否属于某个已有经验的任务领域。

```
[System]
你是一个严格的匹配判断器。判断当前工作窗口是否应该归入已有的某个 task-specific experience。

必须是同一类型的工作任务才算匹配，宽泛或切线相关不算。

当前任务概览：{{task_summary}}
当前标签：{{context_tags}}

候选经验（序号、name、description）：
{{experience_list}}

规则：
- 仅当任务明确属于某个经验的领域时，输出该序号（1~N）
- 不确定时倾向于匹配（输出最接近的序号），宁可走升级路线也不要重复创建

只输出 JSON：{"selectedIndex": 0, "reason": "..."}
```

- `selectedIndex = 0` → 未命中 → **§7.5 LearnTaskExperience（创建新经验）**
- `selectedIndex > 0` → 命中 → **§7.4 EnrichTaskExperience（丰富已有经验）**

### 7.4 EnrichTaskExperience（大模型）

已有经验命中时，将 Chunk 的新执行记录与已有经验对比，判断是否有增量信息，有则合并。

**输入**：
```
已有经验（name + description + bullet_points）
Task Trace（结构化执行记录，含 User Query + Step-by-step）
Task Summary（§7.1 提取）
```

**EnrichTaskExperience Prompt**：

```
你是一个任务经验合并专家。对比新执行记录与已有 task-specific experience，判断是否有值得合并的增量信息。

## 合并原则

1. 语义并集：合并两边的 bullet points，不是简单拼接，要去重融合
2. 防退化：不能丢失已有经验中的有效约束或技巧
3. 去标识化：移除案例特定实体，保留可移植规则
4. RECENCY BIAS：两边冲突时，新 Chunk 的内容优先（近期意图）
5. 不发明内容：只使用两边原有的信息
6. Anti-Duplication：意思相近的点保留表述更清晰的一个

## 判断维度

1. 新 Chunk 是否有已有经验未覆盖的技巧 / 陷阱 / 上下文依赖？
2. 新 Chunk 是否提供了同一技巧的不同适用场景？
3. 新 Chunk 是否纠正或优化了已有经验的某个表述？
4. 以上均无 → no_change

## 输出格式

只输出 JSON：
{
  "action": "enrich | no_change",
  "name": "<保留或优化后的标题>",
  "context": "<适用场景标签>",
  "description": "<一句话摘要，≤50字>",
  "bullet_points": [
    "1. <经验点1>",
    "2. <经验点2>"
  ],
  "added": ["新增了什么及原因"],
  "removed": ["合并时去掉了什么及原因"],
  "reason": "总体判断说明"
}

action=no_change 时，bullet_points 可为空。

[User]
已有经验：
{existing_experience}

新 Task Trace：
{task_trace}

Task Summary:
{task_summary}
```

**写入规则**：
- `action=enrich` → 备份原条目 → 替换 name/context/description/bullet_points → 刷新 `updated_at`
- `action=no_change` → 仅刷新 `updated_at`，不做内容变更

### 7.5 LearnTaskExperience（大模型）

未命中已有经验时，从 Chunk 中生成新的 task-specific experience。

**输入**：
```
Task Trace             ← 结构化执行记录（§5.0 产出，主要证据）
  含 User Query（用户原始请求汇总，作为 TSE 锚点）
  含 Step-by-step 执行记录
Raw User Messages      ← Chunk 内原始 user 消息（辅助证据，验证约束来源）
Task Summary           ← §7.1 提取的任务概览
Context Tags           ← §7.1 提取的标签
```

**LearnTaskExperience Prompt**：

```
你是一个任务经验提取专家。将一段真实工作历史提炼为可复用的 task_specific_experience。

## 知识类型定义

**task_specific_experience**（任务经验）— 有助于下次更好完成同类任务的情境化知识：
- 不是严格执行的 SOP，不规定固定步骤顺序
- 是"遇到 [情境] 时，[推荐做法]"的执行智慧
- **建议性**：agent 召回后自行判断是否采纳，不强制执行
- 内容可以包括：
  1. 工具参数/组合技巧（"用 A 后接 B 效果更好"）
  2. 已安装 skill 的调用场景与组合方式（"做此类任务时优先调用 skill:X" / "先用 skill:X 生成草案，再手动调整"）
  3. 常见陷阱与规避方式（"出现该报错时，改用此做法"）
  4. 隐性的执行顺序建议（"先做 X 再做 Y 更稳"）
  5. 从成功/失败路径中 inferred 的有效做法
  6. 用户主动提出的任务上下文依赖 / 隐性要求（"编排周日行程时，需要先看备忘录和待办"）
- description：一句话摘要，≤50 字
- bullet_points：以 Markdown 列表形式组织的具体经验点

## 与 feedback 的边界

- 用户纠正 agent 错误（"不对，应该 Y" / "别用 X"）→ feedback
- 用户主动提出要求 / 补充上下文（"做 X 时要考虑 Y" / "记得看 Z"）→ task_specific_experience
- agent 自己从执行结果中总结的 → task_specific_experience
- 两者不可混淆

## 提取规则（借鉴 AutoSkill）

### 1) 证据来源
- 以 Task Trace 为主要提取证据（已结构化，Step-by-step 清晰）
- Task Trace 中的 User Query 作为任务锚点，用于判断经验归属
- Raw User Messages 仅作验证辅助（核对 Task Trace 中标注的约束来源）
- assistant 回复中未经用户确认/纠正的内容不提取
- "好的"/"继续"/"ok" 等弱应答不视为用户认可

### 2) 何时提取
- 提取：HOW（执行过程、工具选择、步骤约束、失败恢复）
- 不提取：WHAT（内容本身、结果数据、一次性任务）
- 不提取：泛化价值低、仅此一次的操作
- 判断标准：该经验能否被同一用户在未来类似任务中复用？

### 3) 去标识化（CRITICAL）
- 移除案例特定实体（项目名/文件名/URL/具体数值）
- 用占位符替代：<项目名>、<目标路径>、<配置项>
- 保留真实命令和配置结构（已验证可用）
- 去标识后核心价值消失 → 不提取

### 4) 不发明内容
- 只提取对话中有证据支持的步骤/技巧
- 不补充"通常还需要"的额外步骤
- 不发明用户未提及的约束或规范

## 输出格式

只输出 JSON：
{
  "experiences": [
    {
      "kind": "task_specific_experience",
      "name": "<简短标题，4-8字>",
      "context": "<适用场景标签，1-3个>",
      "description": "<一句话摘要，≤50字>",
      "bullet_points": [
        "1. <具体经验点1>",
        "2. <具体经验点2>",
        "3. <具体经验点3>"
      ]
    }
  ],
  "skip_reason": "<若无可提取内容，说明原因>"
}

无可提取内容 → {"experiences": [], "skip_reason": "..."}

[User]
Task Trace（主要证据）：
{task_trace}

Raw User Messages（验证辅助）：
{raw_user_messages}

Task Summary:
{task_summary}

Context Tags:
{context_tags}
```

### 7.6 直接入库

task_specific_experience 提取/丰富后直接以 `active` 状态入库。

#### 7.6.1 Embedding 输入约定

写入和查询**都只用 `"name: description"` 作为 embedding 输入**，不含
bullet_points：

- **写入侧**（LearnTaskExperience / EnrichTaskExperience）：
  `embed_text = "{name}: {description}"`
- **查询侧**（SearchTseForChunk）：
  `embed_input = "{task_summary} {context_tag_1} {context_tag_2} ..."`

两端都保持短语级语义摘要，量级对齐，向量空间偏差最小化。
bullet_points 只存进 `content` 列，由 Hybrid Score 中的 keyword_overlap
分量（0.18×）负责关键词命中匹配。

#### 7.6.2 查重流程（Learn 产出新经验时执行）

```
提取出 task_specific_experience 条目 e
  │
  ▼
生成 embedding(name: description)
  │
  ├─ Embed 可用  ──┐
  │                ▼
  │           TseSearchByEmbed（mem_record_raw_vec_10，三路相似度）
  │                │
  │                │  combined_score = 0.70×余弦
  │                │                 + 0.18×关键词重叠
  │                │                 + 0.12×名称 Jaccard
  │                │
  │                │  score ≥ 0.85 → 直接 merge（RECENCY BIAS 覆盖）
  │                │  0.50–0.85  → LLM 决策 merge/add（§8.5 Prompt）
  │                │  score < 0.50 → add 新条目
  │                └─ 结束
  │
  └─ Embed 不可用 ──┐
                    ▼
              字符串降级：按 name 精确匹配最新一条
                    │
                    │  命中 → merge（等同 ≥0.85 RECENCY BIAS，
                    │           UPDATE 不重新写入向量）
                    │  未命中 → add 新条目（向量延后补入，
                    │           下次 Embed 可用时 StoreSave 补）
                    └─ 结束
```

**降级合理性**：
- TSE 的 `name` 字段是 LLM 生成的"简短标题（4-8 字）"，同一任务多次
  提取时 LLM 应给出稳定命名 → name 精确匹配可覆盖主要重复场景。
- name 未命中时选择 add 而非丢弃，保证 Embed 恢复后仍有数据可合并。
- 写入 WARN log 提示运维 Embed 服务异常。

**Enrich 产出的经验**：由于已绑定到具体条目并做合并，跳过查重，直接
覆盖原条目。

---

## 8. 数据结构

三种程序记忆各用一张独立表存储，schema 按类型特点设计。v5 另引入两张辅助表（§8.0）支撑单水位 + topic 缓存。

### 8.0 v5 辅助表

#### 8.0.1 `procedural_session_progress`（v5：去除 kind）

```sql
CREATE TABLE procedural_session_progress (
    session_key          TEXT    PRIMARY KEY,
    last_processed_id    INTEGER NOT NULL DEFAULT 0,
    last_processed_at_ms INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_procedural_sess_last_id
    ON procedural_session_progress(last_processed_id);
```

- v4 的 `(session_key, kind)` 复合 PK 在 v5 删除（Fb/TP + TSE 合并进 chunk flush，不再按 kind 分轨道）。
- 迁移：`SchemaMigrateProceduralProgress` 探测 `kind` 列存在 → 无条件 DROP 重建（plan C）。水位丢失可接受，cold-start 由 dedup 兜底。

#### 8.0.2 `procedural_topic_cache`（v5 新增）

见 §6.2。用途：跨 tick 复用同窗口的 topic 提取 LLM 结果，flush 后按覆盖范围清理，7 天 GC 兜底。

### 8.1 TSE 主表 + 扩展表（v4 Phase 1）

task_specific_experience 直接 active，无候选池。v4 Phase 1 将 TSE 物理
合并进 `mem_record` 主表：

- **主表 `mem_record`**：`memory_type = PROCEDURAL(2)`、`category =
  GSPD_CAT_EXPERIENCE(10)`、`tier = LEAF(2)`。`content` 字段由
  `TseBuildMemRecordContent(name, description)` 拼装为
  `"{name}：{description}"`（中文全角冒号）；同时作为向量嵌入的输入，
  写入 `mem_record_raw_vec_10`（LEAF 层按 category 分表）。
- **扩展表 `procedural_tse_meta`**（仅承载结构化字段）：

```sql
CREATE TABLE procedural_tse_meta (
  record_id       INTEGER PRIMARY KEY
                  REFERENCES mem_record(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  context         TEXT,
  description     TEXT NOT NULL,
  bullets         TEXT,          -- JSON array，仅用于召回展示，不进向量
  source_sessions TEXT           -- JSON array，来源 session_id 列表
);
CREATE INDEX idx_procedural_tse_meta_name ON procedural_tse_meta(name);
```

读写入口：procedural/learn 层的 facade `TseRecordInsert` / `TseRecordUpdate`
/ `TseSearchByEmbed` / `TseLoadByName`（`ltm_procedural_learn_tse_store.h`）。
检索通过 `StoreSemanticSearch(tier=LEAF, filter.category=EXPERIENCE)`
精确命中 `mem_record_raw_vec_10`，无需 over-fetch 后置过滤。

**历史变更**：
- v4 Phase 1 之前的 `task_specific_experience_memory` 独立表和
  `task_specific_experience_memory_vec` 向量表已废弃。
- 2026-04-20：模块前缀从 `pmem` 改为 `procedural`，扩展表
  `task_specific_experience_meta` 更名为 `procedural_tse_meta`。不做
  数据迁移，需清空老库。

### 8.2 Feedback（v4 Phase 2 双层架构）

Fb 采用 **候选池 + 主表** 两层结构：

- **pending 层**：`procedural_candidates`（`kind='feedback'`）——保留
  mention/hard/total_updates/support_score/first_seen_at/last_seen_at
  等打分字段；由 `ProceduralLearnUpdateFeedbackAndPreferences` 维护。
- **active 层**：`mem_record`（`memory_type=PROCEDURAL, category=FEEDBACK(12)`）
  + `procedural_fb_meta` 扩展表——仅 active 后才写入。升格入口：
  `PromoteOneCandidateToActive → FbRecordInsert`（
  `ltm_procedural_learn_fb_store.h`）。

主表 `content` 由 `FbBuildMemRecordContent(context, description)` 拼为
`"{context}场景中，{description}"`（context 空时退化为 description），
同时作为 `mem_record_raw_vec_12` 的向量嵌入输入。

```sql
CREATE TABLE procedural_fb_meta (
  record_id              INTEGER PRIMARY KEY
                         REFERENCES mem_record(id) ON DELETE CASCADE,
  name                   TEXT NOT NULL,
  context                TEXT,
  description            TEXT NOT NULL,
  is_hard                INTEGER NOT NULL DEFAULT 1,
  source_user_correction INTEGER NOT NULL DEFAULT 0,
  source_sessions        TEXT
);
```

**历史变更**：
- 老独立表 `feedback_memory`（包含 pending/active 同表 +
  status/support_score/counts）Phase 2 起不再由热路径使用。
- 2026-04-20：`feedback_meta` 扩展表更名为 `procedural_fb_meta`；老
  独立表 `feedback_memory` / `feedback_memory_vec` 彻底废弃（不保留
  兜底 DDL）。不做数据迁移。

### 8.3 Tool Preference（v4 Phase 2 双层架构）

TP 结构与 Fb 对称：

- **pending 层**：`procedural_candidates`（`kind='tool_preference'`）。
- **active 层**：`mem_record`（`category=TOOL_PREFERENCE(11)`）+
  `procedural_tp_meta` 扩展表。
- **facade**：`TpRecordInsert/Update/SearchByEmbed/LoadByName`
  （`ltm_procedural_learn_tp_store.h`）。

主表 `content` 由 `TpBuildMemRecordContent(context, name, description)`
拼为 `"{context}时优先用 {name}：{description}"`（context 空时退化为
`"优先用 {name}：{description}"`），同时作为 `mem_record_raw_vec_11`
的嵌入输入。

```sql
CREATE TABLE procedural_tp_meta (
  record_id       INTEGER PRIMARY KEY
                  REFERENCES mem_record(id) ON DELETE CASCADE,
  name            TEXT NOT NULL,
  context         TEXT,
  description     TEXT NOT NULL,
  is_hard         INTEGER NOT NULL DEFAULT 1,
  source_sessions TEXT
);
```

**历史变更**：2026-04-20 起，扩展表更名为 `procedural_tp_meta`；老
独立表 `tool_preference_memory` / `tool_preference_memory_vec` 彻底
废弃（不保留兜底 DDL）。不做数据迁移。

### 8.4 Task Trace（v5 新增）

Task Trace 是 **agentic episodic memory**（程序记忆的过程层），记录 chunk 内完整的结构化执行轨迹。

- **主表 `mem_record`**：`memory_type = PROCEDURAL(2)`、`category = TASK_TRACE(13)`、`tier = LEAF(2)`。
- **不建扩展表**：全部信息（User Query + Step-by-step Markdown）塞进 `content`。
- **向量表**：`mem_record_raw_vec_13`（LEAF 层按 category 分表，v5 新增）。
  embedding 输入：`"{topic_summary} {user_query}"`（用于离线 dreaming 阶段的同类任务聚类）。
- **TTL**：30 天自动清理（过程记录，非持久资产）。

**写入入口**：`TaskTraceRecordInsert`（`ltm_procedural_learn_trace_store.h`）。

**内容格式**（Markdown，见 §5.0.3 Prompt 中的输出结构）：

```markdown
## Task Trace

**User Query**: {汇总请求}
**Topic**: {topic_tag} | {task_category}
**Topic Summary**: {topic_summary}
**Time Range**: {first_turn_time} → {last_turn_time}

### Step 1 — [timestamp] User Request
...

### Step 2 — [timestamp] Agent Action / Tool Call
...
```

**用途**：
1. **在线**：Fb/TP/TSE 提取的统一输入（去噪后的结构化证据）
2. **离线**：积累同类任务 trace → 跨任务模式挖掘 → workflow 抽象（v6 预留）

---

### 8.2 两层查重 + 候选池更新流程

```
提取出候选条目 c（kind = feedback | tool_preference）
  │
  ▼
第一层：查 skills_library（同 kind）
  │
  │  三路相似度：
  │    combined_score = 0.70×向量余弦 + 0.18×关键词重叠 + 0.12×名称Jaccard
  │
  │  score ≥ 0.85 → 直接 merge（高置信）
  │  score < 0.50 → 直接 add（无重复）
  │  0.50–0.85   → 小模型决策 merge/add
  │
  ├─ merge → 直接更新 active 条目
  │   刷新 last_seen_at（防 stale）
  │   is_hard=true 且内容冲突 → RECENCY BIAS 覆盖内容
  │   一致 → 仅刷新时间戳
  │   不进入候选池
  │
  └─ add → 进入第二层
       │
       ▼
第二层：查候选池（同 kind）
  │
  │  combined_score = 0.70×向量余弦 + 0.18×关键词重叠 + 0.12×名称Jaccard
  │
  │  score ≥ 0.85 → 直接 merge
  │  score < 0.50 → 直接 add
  │  0.50–0.85   → 小模型决策 merge/add（同上 prompt）
  │
  ├─ merge（找到匹配 m）
  │   mention_count++
  │   if c.is_hard: hard_count++
  │   last_seen_at = now
  │   total_updates++
  │   重新计算 support_score
  │
  └─ add（未找到）
      新建候选条目（mention_count=1，hard_count=0 or 1）
      total_updates = 1
  │
  support_score ≥ dynamic_threshold(total_updates)?
  │
  ├─ 是 → 升格，直接写入 skills_library
  │        entry.status = 'active'
  │        candidate.status = 'promoted'
  │        （无需再 dedup，第一层已排除库中重复）
  │
  └─ 否 → 留池等待
```

### 8.3 打分公式

```
mention_ratio = mention_count / total_updates
recency_score = exp(-(now - last_seen_at) / 7days)
hard_ratio    = hard_count / mention_count

support_score =
  0.50 × mention_ratio   （跨次出现率，权重最高）
+ 0.25 × recency_score   （时间衰减，越近越高）
+ 0.25 × hard_ratio      （显式纠正比例）
```

**核心效果**：
- 用户显式纠正（hard=true）→ hard_ratio=1.0 + recency_score≈1.0 → score ≈ 0.50 → 第一次即过早期阈值
- Agent 推断的 soft 规律 → 需要跨多次积累 mention_ratio → 反复验证后才升格

### 8.4 动态升格阈值

| 成熟度 | total_updates | 阈值 |
|--------|--------------|------|
| 早期   | ≤ 2          | 0.18 |
| 成长期 | 3–6          | 0.30 |
| 成熟期 | 7–16         | 0.34 |
| 稳定期 | > 16         | 0.38 |

### 8.5 小模型决策 Prompt（0.50–0.85 区间，两层共用）

```
[System]
你是程序记忆库的条目管理器。
判断新提取的条目应该与已有条目合并（merge）还是作为独立条目新增（add）。

### 决策流程（按顺序执行）

0) 能力重叠硬门控
   如果两条描述的是同一核心能力 → 不得 add，只能 merge。

1) 能力族检查
   完全不同的任务领域 → add。

2) 4 轴身份判断
   a. 核心任务目标：要完成的事是否相同？
   b. 适用场景 / context 标签：在什么情境下触发？
   c. 硬约束：不能做什么？关键限制是否一致？
   d. 工具或操作方法：涉及的工具/命令是否相同？
   → 4 轴高度重叠 → merge
   → 有实质差异（场景不同 / 工具不同 / 约束冲突）→ add

3) 兜底
   不确定 → add（宁可多一条，不误合并）

只输出 JSON：
{"action": "merge" | "add", "reason": "..."}

[User]
已有条目：
  kind: {{existing.kind}}
  name: {{existing.name}}
  context: {{existing.context}}
  description: {{existing.description}}

新提取条目：
  kind: {{candidate.kind}}
  name: {{candidate.name}}
  context: {{candidate.context}}
  description: {{candidate.description}}
```

### 8.6 合并执行 Prompt（score ≥ 0.85 自动触发 / 决策=merge 后触发）

```
[System]
你是程序记忆合并器。将已有条目与新条目合并为一个更完整的条目。

### 合并原则

1. 语义并集：保留双方所有重要信息，不是拼接是融合
2. 防退化：不能丢失已有条目中的约束
3. 去标识化：移除案例特定细节，保留可移植规则
4. Anti-Duplication：近似表达保留一个规范版本
5. 不发明内容：只用两边原有信息，不添加推测
6. 最近性原则（RECENCY BIAS）：
   - 两边冲突时，新条目 = 近期意图，优先采纳
   - 已有条目的独有约束仍保留（不算冲突）

### 输出要求

- description ≤ 50 字，清晰可操作
- 以触发条件开头："遇到...时" / "做...时优先用"
- context 标签取并集去重
- kind 保持不变

只输出 JSON：
{
  "kind": "...",
  "name": "...",
  "context": "...",
  "description": "...",
  "is_hard": true | false
}

[User]
已有条目（历史）：
  kind: {{existing.kind}}
  name: {{existing.name}}
  context: {{existing.context}}
  description: {{existing.description}}
  is_hard: {{existing.is_hard}}

新提取条目（近期）：
  kind: {{candidate.kind}}
  name: {{candidate.name}}
  context: {{candidate.context}}
  description: {{candidate.description}}
  is_hard: {{candidate.is_hard}}
```

降级路径：LLM 失败时取两边 description 中较长的一份（近期优先 — RECENCY BIAS），context 取并集去重，`is_hard` 取 OR（任一 true 则 true）。

---

## 9. 完整数据流（v5）

```
120s 定时器 TimerProcessProceduralCache
  │
  per-session 读水位（procedural_session_progress）
  拉 id > watermark 的新 turn（可选 overlap 2 行做跨 batch 续接）
  │
  触发条件（满足任一即触发）：
    newCount ≥ 8（冷启动）或 ≥ 6（热启动）
    OR sinceProcessedMs ≥ 20min
    OR wk->flushForce（memory_flush 同步屏障）
  │
  第一层清洗（autoCapture 侧已做；C 端 v5 暂时短路 IsWindowHighQuality）
  │
  生成滑动窗口 [W0, W1, W2, ...]（n=8, m=2, step=6）
  │
  for Wi in windows:
    │
    ├─ [Step 3] ExtractTopicMetadataCached(Wi)
    │    先查 procedural_topic_cache（session × first_turn_id
    │    × last_turn_id）；miss → 调 LLM + 回填。
    │    输出 topic_keywords / topic_tag / task_category /
    │    topic_summary（15-40 字自然语言，v5 新增）。
    │
    └─ [Step 4] 主题拼接状态机 DecideChunkAction
         Jaccard > 0.3 同主题 / ≤ 0.3 小模型 JudgeTopicBoundary
         达 3 窗口 / 2h gap 强制 flush
         │
         flush → ChunkFlushV4(chunk):
           │
           (A) Task Trace 提取（v5 新增，Step 4.5）
               ExtractTaskTrace(chunk + topic 锚点)
               → 结构化执行记录（User Query + Step-by-step）
               → 写入 mem_record(category=TASK_TRACE)
               → 下游 Fb/TP/TSE 均从 Task Trace 消费
           │
           (B) Fb/TP 提取（v5: per-chunk，从 Task Trace 提取）
               LearnFeedbackAndPreference（
               Task Trace 为主要证据 + Raw User 为验证辅助；
               topic_tag + task_category + topic_summary 为锚点）
               → Layer 1 active 表 hybrid 查重
                   ≥ 0.85 merge | < 0.50 下 Layer 2 | 灰区 LLM
               → Layer 2 候选池（procedural_candidates）
                   kind+name 精确匹配 → mention/hard/total ++
                   support_score ≥ 0.60 → 升格 active
           │
           (C) TSE 提取（从 Task Trace 提取）
               ExtractChunkOverview(Task Trace)
               → SearchTseForChunk hybrid 召回
                  → JudgeTaskExperienceMatch
                     命中 → EnrichTaskExperience(Task Trace)
                     未命中 → LearnTaskExperience(Task Trace)
                           → TseDedupAndSave（hybrid）
           │
           (D) MaintainPendingPool 升格/清理
           │
           (E) 推水位到 chunk 末尾 turn id
               StoreProceduralTopicCacheDeleteByRange 清缓存
  │
  末尾：未 flush 的 chunk 留在内存，水位原地不动
        下次 tick 从水位重新拉 turn 重组（跨 tick 持久化）
        forceSingleChunk=1（memory_flush）→ 强制 flush 残余
```

---

## 10. 触发时机

| 事件 | 输入 | 备注 |
|------|------|------|
| `TimerProcessProceduralCache` | 每 **120 秒** 扫一次 `mem_conversation` 水位之后的新行 | v5 单一主触发路径 |
| `memory_flush` 同步屏障 | 调用方显式请求 | 置 `wk->flushForce=1`，绕过数量/时间门槛一次性处理，并叠加 `FORCE_SINGLE_CHUNK` flag 把所有窗口并一 chunk |

**触发条件（per session，满足任一即触发）**：

```
读 procedural_session_progress (session_key) 得 watermark、
  lastProcessedAtMs

fetch id > watermark 的新 turn → newCount = N

dataEnough  = newCount >= (watermark > 0 ? 6 : 8)   // 热 6 / 冷 8
timeReached = (now - lastProcessedAtMs) >= 20min
forceRun    = wk->flushForce

触发 = dataEnough || timeReached || forceRun
```

**数据来源 `mem_conversation`**：agent 每轮消息 append-only 入库，`status=0/pending`。procedural worker 只消费 `skip_extraction=0` 的对话行，不修改 `status`（该状态由 fact 管道拥有）；水位记录在 `procedural_session_progress`，推进由 `pipeline flush` 驱动（§6.5）。

### 10.1 跨 tick 持久化策略

v5：未 flush 的 chunk 缓存只在 `InstanceWorker` 内存，水位不推；下一 tick 从水位重新拉 turn 重组。好处：
- 进程重启缓存丢失**不丢数据**——水位没推，再拉一遍即可。
- 不需要 DB 记录 chunk 缓存态，简化 schema。

### 10.2 串行处理

学习管线采用串行处理：一批消息跑完再捞下一批，不做并行。后台批处理不需要并发，串行天然避免多批同时修改同一经验的问题。

---

## 11. 模型分配

| 调用 | 模型 | 理由 |
|------|------|------|
| **§6 主题抽取** | | |
| §6.1 ExtractTopicMetadata（含 topic_summary）| 小模型 | 结构化提取，模式明确；有 DB 缓存复用 |
| §6.4 主题边界判断 JudgeTopicBoundary | 小模型 | 二分类判断，结构化输出 |
| **§5 Chunk flush — Feedback + TP**（v5：per-chunk）| | |
| LearnFeedbackAndPreference | 大模型 | 需要理解对话语义，识别用户纠正意图 |
| DedupDecideFeedbackTp（灰区） | 小模型（lightLlm 可用时） | 结构化决策，成本低 |
| MergeFeedbackTp | 小模型 | 条目简单，轻量合并 |
| **§5.0 Task Trace 提取** | | |
| ExtractTaskTrace | 小模型 | 结构化整理，模式明确；去噪但不丢失关键证据 |
| **§7 Chunk flush — Task-specific Experience** | | |
| §7.1 ExtractChunkOverview | 小模型 | 结构化提取，模式明确 |
| §7.3 JudgeTaskExperienceMatch | 小模型 | 序号选择，结构化输出 |
| §7.4 EnrichTaskExperience | 大模型 | 合并 bullet points，需要语义融合 |
| §7.5 LearnTaskExperience | 大模型 | 核心提取，需要泛化与去标识化 |
| TseDedupDecide（灰区） | 小模型（lightLlm 可用时） | 结构化决策 |
| **§8 候选池** | | |
| §8.5 维护决策（add/merge） | 小模型 | 结构化决策 |
| §8.6 合并执行 | 小模型 | 条目简单，轻量合并 |

---

## 12. 参考来源

| 设计点 | 来源 |
|--------|------|
| 滑动窗口参数（n=10, m=2） | MemOS Python 端 |
| Tool 输出 60/30 截断策略 | MemOS Generator |
| 敏感数据脱敏（API key/secret/路径） | MemOS Generator |
| 推理标签剥离（`<think>`） | MemOS captureMessages |
| 元数据剥离（Sender块/时间戳/message_id） | MemOS captureMessages |
| 语言检测（CJK 比例） | MemOS |
| 哨兵消息过滤（NO_REPLY/HEARTBEAT） | AutoSkill captureMessages |
| USER-only 证据原则 | AutoSkill |
| WHAT vs HOW 判断边界 | AutoSkill |
| 分层输入结构（Primary + Full） | AutoSkill |
| 支持度打分公式 + 动态阈值 | AutoSkill Requirement Memory |
| 三路相似度 dedup | AutoSkill |
| 维护决策 add/merge/discard 4轴判断 | AutoSkill 维护管线 §8.2 |
| RECENCY BIAS（近期意图优先） | AutoSkill 合并执行 §8.3/8.4 |
| Anti-Duplication（近似短语去重） | AutoSkill Skill Merger §8.4 |

---

## 13. 迭代三预留

- **Task Trace 离线挖掘**：积累 N 条同类任务的 Task Trace 后，定期运行离线管线从中抽象 workflow / skill 模板 / 最佳实践（v6 workflow 提取的前置条件）
- Task Trace 30 天 TTL 自动清理机制实现
- Dreaming 跨 session 归纳（候选池的 mention_count 天然积累跨 session 信号）
- 候选池定期清理（长期 pending 且无新证据 → discard）
- Task-specific experience 的主动应用策略（如何从候选池外推到新任务）
- 恢复 v4 设计文档里的 §3 两层预过滤（v5 当前由 autoCapture 侧做，C 端 IsWindowHighQuality 临时短路）
- v4 质量过滤（§6.6）恢复：先让 ingest 层把 autoCapture 写入的 JSON blob 拆成单独 turns，否则 `IsWindowHighQuality` 永远命中 0 → TSE 被静默跳过
