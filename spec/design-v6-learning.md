# Spec 004 — 程序记忆学习模块（v6）

> 最后更新：2026-04-22
>
> 程序记忆三元结构：`task_specific_experience` / `feedback` / `tool_preference`，辅以 `task_trace` 过程层。

---

## 0. 概述

### 目标

从 agent 真实工作历史中**自动提取并演进程序记忆**。整体架构分为**三层**：

- **事件层**（`procedural_task_trace`，§8.2）：每次执行的完整结构化轨迹（task / sub-task / step 三级），允许同任务做 N 次产 N 条
- **事实层**（`procedural_concept`，§8.1）：task / sub-task 的 canonical 名字字典，去重 + 计数；不重复
- **知识层**（`task_specific_experience` / `feedback` / `tool_preference`）：从事件层提炼的可复用知识，通过 canonical name 字面值（TEXT 列，不走 FK）与事实层对应
  - **TSE 通过 `task` 列锚 task**（整任务级经验，同任务复用；存 canonical name 字面值）
  - **Fb/Tp 通过 `sub_task` 列锚 sub-task**（子事务级纠正/偏好，**跨任务复用**——同一 sub-task name 出现在不同 task 时，对应 fb/tp 都能召回；存 canonical name 字面值）

| 类型 | 表 | 存储内容 | 核心价值 |
|------|---|---------|---------|
| `task_trace` | `procedural_task_trace` | 事件层：单次执行的结构化 Markdown 轨迹 | Fb/Tp/TSE 提取的统一证据源；离线 dreaming 的原材料 |
| `task` / `sub_task` | `procedural_concept` | 事实层：canonical 名字 + occurrence 计数 + (task 的) description | 跨 chunk 名字去重与统计；description 供 §7 查询拼 search_key。不被业务表 FK 引用，fb/tp/tse 表直接存同名 TEXT 列索引 |
| `task_specific_experience` | `mem_record` + `procedural_tse_meta`（含 `task` TEXT 列） | 任务执行经验：情境化的决策参考、技巧组合、风险规避、失败恢复路径、用户主动提出的任务要求与上下文依赖 | 提升同类任务的执行质量，避免重蹈覆辙 |
| `feedback` | `mem_record` + `procedural_fb_meta`（含 `sub_task` TEXT 列） | 错误更正记忆：用户纠正或 agent 自身纠错后形成的**硬约束**行为规则 | 避免重复犯错，优先级最高；**跨任务复用** |
| `tool_preference` | `mem_record` + `procedural_tp_meta`（含 `sub_task` TEXT 列） | 工具选择偏好：做某类任务优先用哪个工具 | 减少决策开销，提升执行效率；**跨任务复用** |

> **应用原则**：`task_specific_experience` 在召回后作为**情境化建议**供 agent 参考，agent 自行判断是否采纳；`feedback` 则是必须遵循的硬约束。

## 1. 整体架构

```
触发：定时器每 120s 扫一次（TimerProcessProceduralCache）
输入：mem_conversation 里水位之后的对话行

─────────────────────────────────────────────────────────────────────
Step 1  预过滤（零 LLM，§3）
          §3.1 TS 侧 autoCapture（主力）：整轮跳过 heartbeat/startup、
              role 分流（system 丢 / assistant 留 text+thinking+toolCall）、
              user 消息 9 条正则 sanitize、
              成功 toolResult 整条丢 / 错误截 300 字
          §3.2 C 端补齐：JSON blob 反序列化 + 敏感数据脱敏 + 语言标注

Step 2  生成滑动窗口（n=8, m=2, step=6）
          冷启动：攒够 8 条新 turn → 1 个满窗口
          热启动：6 条新 turn + 2 条 overlap → 1 个满窗口

Step 3  窗口主题抽取（小模型 + DB 缓存）
          先查 procedural_topic_cache(session_id, first_turn_id,
          last_turn_id)；命中直接复用，未命中调 LLM 并回填缓存。
          提取字段（v6 精简至两个）：
            - topic_keywords（3-8 个关键词，Jaccard 边界判断用，
              零 LLM 可算）
            - topic_summary（15-40 字自然语言一句话，
              §5.4 Jaccard 兜底时 LLM 的语义判断输入；
              v6 不传给 Task Trace / TSE（§6.0.2 已明确理由）；
              无明确任务时空串）

Step 4  主题拼接状态机
          每窗口进来比较 topic_keywords 与 chunk 缓存：
            - Jaccard > 0.6 → 同主题，追加进 chunk，更新 topic
            - Jaccard ≤ 0.6 → 小模型 JudgeTopicBoundary 兜底
            - chunk 达 3 窗口 / 2h gap → 强制 flush
          chunk 仅存在 InstanceWorker 内存，不落 DB；重启缓存丢失
          不丢数据（水位未推 → 下次重拉重组）。

Step 5  Chunk flush（Task Trace + Fb/TP + TSE）
          flush 时对 chunk 一次性跑：

          (A) Task Trace 提取（单次 LLM 调用）
              ExtractTaskTrace(chunk_messages)    ← 仅 chunk_messages
              → Markdown 含：
                  **User Query**
                  **Task**: <name>：<description>   （skill 风格行内）
                  **Time Range**: <代码后处理填>
                  ### sub-task 1 — <10-25 字 name>
                    Step 1.1 / 1.2 / ...
                  ### sub-task 2 — ...
              → 解析 **Task** 行 split 冒号 → (task_name, task_desc)
              → 解析每个 ### sub-task N — 头 → sub_task_name 列表
              → UPSERT procedural_concept（task + 每个 sub_task，仅字典统计）
              → INSERT procedural_task_trace（§8.2，独立表）
              → 下游 Fb/TP/TSE 均从 trace.content 消费

          (B) Fb/Tp 提取（每 task 一次 LLM 调用，sub-task 作锚点）
              LearnFeedbackAndPreference(Task Trace)
              一次 LLM 调用看完整 Trace，产出所有 fb/tp 条目，
              每条带 sub_task name 字面值（Trace 段头原文）。
              单层查重（procedural_candidates，(kind, sub_task) 分区）：
                  hybrid score = 0.80·cos + 0.20·kw
                  ≥ 0.85 → 命中现有 candidate，mention++/hard++
                  < 0.50 → 新建 pending candidate
                  0.50–0.85 → 小模型决策 merge/add
              在线仅 is_hard=true 时立即升格（§6.2 短路），
              其余非 hard 条目走 §13 离线 worker 升格

          (C) TSE 提取（在线只写候选池，不升格）
              §7.1 召回 top-5 candidates（按 task 名 + hybrid score）
              §7.2 Judge 小模型选 selectedIndex
              命中 → §7.3 Enrich 大模型，把新 bullets 合并进 candidate
              未命中 → §7.4 Learn 大模型新建 candidate
              升格统一由 §13 离线 worker 处理

Step 6  水位推进
          只有 flush 真实发生时才推进到 chunk 覆盖的最后 turn id；
          未 flush 的 chunk 缓存留在内存，水位原地不动，下次 tick
          从水位重新拉 turn 重组。flush 成功后
          StoreProceduralTopicCacheDeleteByRange 清理被覆盖区间
          的 topic 缓存，7 天 GC 兜底。
─────────────────────────────────────────────────────────────────────
```

**Task Trace 两级 topic 层次示意：**

```
Task Trace（一条 procedural_task_trace 行）
├── **Task**: <canonical name 4-10 字>：<description ≤ 30 字>
│     └── TSE 检索锚点（任务级，search_key = "{task}: {description}"）
├── ### sub-task 1 — <10-25 字描述短语>
│   ├── Step 1.1 — UserRequest / AgentAction / ToolCall / UserCorrection
│   ├── Step 1.2 — ...
│   └── Fb/Tp 条目 sub_task = "sub-task 1" 段头部 name 字面值
├── ### sub-task 2 — ...
│   ├── Step 2.1 — ...
│   └── Fb/Tp 条目 sub_task = "sub-task 2" name 字面值
└── ...
```

**架构全景图**（从下到上：原始对话 → 知识层）：

```
 ┌─────────────────── Agent（读） ───────────────────┐
 │                    memory_search                  │
 └────────────────────────▲──────────────────────────┘
                          │ reads (仅 active 可见)
                          │
 ┌──────── 知识层（active 快照，§8.3/8.4/8.5）────────┐
 │  mem_record                                         │
 │   ├─ category=EXPERIENCE        + procedural_tse_meta  (bullets_json)
 │   ├─ category=FEEDBACK          + procedural_fb_meta   (description)
 │   └─ category=TOOL_PREFERENCE   + procedural_tp_meta   (description)
 │  + mem_record_raw_vec_{10,11,12}                    │
 └────────────────────────▲──────────────────────────┘
                          │ upsert (INSERT 首次 / UPDATE 重升)
                          │
 ╔════════════════════════╧═══════════════════════════╗
 ║        §13 离线升格 worker（每天 1 次）             ║
 ║                                                     ║
 ║  处理：fb/tp 非 hard + 全部 tse                     ║
 ║    （fb/tp is_hard 已在 §6.2 在线立即升格）          ║
 ║                                                     ║
 ║  Step A 按次数规则挑候选（零 LLM）：                 ║
 ║    delta = mention - last_promoted ≥ 3              ║
 ║                                                     ║
 ║  Step B 合理性检查（小模型）+ 通过则 upsert active ║
 ║    读 candidate 的最近 5 条 source_trace 做证据    ║
 ║    失败 → 不升格，下次 delta 够时再检查              ║
 ║                                                     ║
 ║  candidate 保留，可重复升格（last_promoted_*）      ║
 ╚════════════════════════▲═══════════════════════════╝
                          │ scan
                          │
 ┌──── 候选池（§8.0.3，三类合表）─────┐          ┌──── 事件层（§8.2）────┐
 │ procedural_candidates              │          │ procedural_task_trace │
 │  kind ∈ {fb, tp, tse}              │          │  + task_trace_vec     │
 │  anchor / payload / search_key     │          │  每次 chunk 一条      │
 │  mention_count / is_hard /          │          │                       │
 │  last_promoted_at_mention           │          │                       │
 │  + procedural_candidates_vec       │          └───────────▲───────────┘
 └──────▲──────────────▲──────────────┘                      │ INSERT trace
        │ 写 fb/tp      │ 写 tse                              │
        │               │                                    │
 ┌──────┴─────────┐ ┌───┴──────────────┐                     │
 │ LearnFb+Tp     │ │ TSE 四步           │                     │
 │   ← 大模型      │ │ §7.1 Search 零LLM │                     │
 │   每 Trace 一次 │ │ §7.2 Judge 小     │                     │
 │   per sub_task  │ │ §7.3 Enrich 大    │                     │
 │     锚点        │ │ §7.4 Learn 大     │                     │
 └────────▲───────┘ └────▲──────────────┘                     │
          │ 读 Trace       │ 读 Trace                          │
          └───────┬────────┘                                   │
                  │ Task Trace                                 │
                  │                                            │
 ╔════════════════╧════════════════════════════════════════════╧══╗
 ║   ExtractTaskTrace  ← 大模型                                    ║
 ║   chunk_messages → Trace Markdown                              ║
 ║                                                                 ║
 ║   同时 UPSERT procedural_concept（§8.1 事实层）                  ║
 ║     task / sub-task name + description 字典（去重、计数）       ║
 ╚══════════════════════════▲══════════════════════════════════════╝
                            │ Step 5 Chunk Flush（§6, §7）
                            │ flush when chunk ready
                            │
 ┌──── 窗口主题 + 拼接状态机（§5）────┐
 │  滑动窗口 n=8 / m=2 / step=6        │
 │  小模型抽 topic_keywords + topic_summary │
 │    （查 procedural_topic_cache 复用） │
 │  Jaccard > 0.6 合 / ≤ 0.6 小模型兜底 │
 │  3 窗口 / 2h gap 强制 flush          │
 └─────────────────────────▲─────────────────────────┘
                           │ 120s 定时器拉水位之后的行
                           │
 ┌──── mem_conversation（append-only + 水位）────┐
 └─────────────────────────▲─────────────────────────┘
                           │ memory_add
                           │
 ┌──── TS 端 autoCapture（§3.1，预过滤主力）────┐
 │  · 整轮跳过：heartbeat / startup / 无 user    │
 │  · role 分流：system 丢 / assistant 留        │
 │       text+thinking+toolCall                   │
 │  · user 消息 sanitize（9 条正则）             │
 │  · 成功 toolResult 整条丢，错误截到 300 字     │
 │  · 打包 JSON blob → memory_add                │
 └─────────────────────────▲─────────────────────────┘
                           │ agent_end event
                           │
 ┌─────────────────── Agent（写） ───────────────────┐
 │                  产生对话 / 工具调用               │
 └────────────────────────────────────────────────────┘
```
**读写边界一句话总结**：

| 操作 | 读 | 写 |
|---|---|---|
| 在线 chunk flush | `procedural_candidates` + `procedural_concept` | `procedural_task_trace`、`procedural_concept`、`procedural_candidates` |
| 离线升格 worker | `procedural_candidates` | `mem_record` + `procedural_{fb,tp,tse}_meta`、回写 `procedural_candidates.last_promoted_*` |
| Agent 召回 (memory_search) | `mem_record` + 三个 meta 扩展表 | — |

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
| `jaccard_threshold` | 0.6 | §5.4 主题边界判断 Jaccard 快速门控阈值 |
| `promote_delta` | 3 | §8.7 升格所需的 mention_count 累积增量 |

> v5 的 `candidate_match_threshold = 0.70` 已删除——v6 候选池查重用 §2.3 的 hybrid score 三档阈值（0.50 / 0.85）。

窗口分布（N=24 轮示例）：

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

### 2.3 Hybrid Score 与阈值（Fb/TP/TSE 共用，公式统一）

**Hybrid Score 公式**：

```
combined_score = 0.80 × cos(embedding_a, embedding_b) + 0.20 × keyword_overlap(search_key_a, search_key_b)
```

- `embedding_*` 来自 `procedural_candidates_vec`，由 `search_key` 算出（§8.11）
- `keyword_overlap` 在两条 search_key 的分词结果上算 Jaccard
- TSE 无 `name` 字段，v6 已取消 v5 的三路 name_Jaccard 分量

**查重阈值（§8.6 候选池写入 / §6.2 chunk flush dedup）**：

| 区间 | 动作 |
|------|------|
| ≥ 0.85 | 直接 merge（RECENCY BIAS 覆盖） |
| 0.50 – 0.85 | 小模型灰区决策（§8.9） |
| < 0.50 | add 新 pending candidate |

**召回过滤阈值（§7.1 TSE SearchCandidate）**：

| 区间 | 动作 |
|------|------|
| < 0.60 | 过滤掉（不进 top-5 候选）|
| ≥ 0.60 | 进 top-5，交 §7.2 Judge 判断 |

> 召回阈值 0.60 —— 要求候选与当前 task 有**明显语义相关度**才进 Judge；跨 task 弱相关（cos 0.3~0.5）基本不会误匹配。比查重阈值 0.85 低一档，允许字面漂移但语义接近的 task（如 "修登录 bug" vs "修复登录 module 错误"）进 Judge 做最终判定。

---

## 3. Step 1 — 预过滤（零 LLM）

预过滤由 **TS 侧 autoCapture 钩子**和 **C 端消费 pipeline** 两处协作完成。绝大多数的元数据剥离、角色过滤、工具结果缩减在 **TS 侧**做掉；C 端只负责 TS 没覆盖的补丁项（敏感脱敏、机械回复、语言标注、JSON blob 反序列化）。

### 3.1 TS 侧 autoCapture（进 `mem_conversation` 前）

实现位置：`openclaw-plugin/memory-plugin/src/memory/hooks.ts` 的 `registerConversationCapture()` + `src/memory/text-utils.ts` 的 `sanitizeUserTextForCapture()`。触发事件 `agent_end`，按轮打包写入 `mem_conversation`。

#### 3.1.1 轮入口拦截（整轮跳过）

以下情况直接 return，本轮不入 `mem_conversation`：

| 条件 | 判定 | 理由 |
|----|----|----|
| 心跳 run | `ctx.trigger === "heartbeat"` | 心跳 run 无用户真实对话 |
| Server 未就绪 | `client.isConnected()` 且 `waitReady()` 失败 | 避免写入失败产生脏状态 |
| Agent 失败 | `event.success === false` 或无 messages | 无完整对话上下文 |
| 无 user 消息 | 反向扫 messages 找不到 `role==="user"` | 不是一轮完整交互 |
| Session 启动序列 | 最后一条 user 消息前 200 字符匹配 `/session.*(started\|reset).*\/(new\|reset)/i` 或 `/Run your Session Startup sequence/i` | `/new` / `/reset` 触发的框架元操作，不含用户意图 |

#### 3.1.2 按 role 分流清洗

从最后一条 user 消息 (`lastUserIdx`) 开始向后扫整轮。角色白名单只保留 **user / assistant / toolResult**，其他一律丢（包括 system）。

**user 消息处理**（仅保留 text block，过 `sanitizeUserTextForCapture`）：

以下正则按顺序作用在 text 上，整块剥离：

| 序 | 正则 / 操作 | 说明 |
|----|----|----|
| 1 | `\0` → 删 | null byte |
| 2 | `<relevant-memories>[\s\S]*?</relevant-memories>` → 删 | recall 注入块 |
| 3 | `##\s*\S[^\n]*\n以下核心用户信息已固定加载在上下文中[\s\S]*` → 删 | OpenViking profile context |
| 4 | `(^\|\n)\s*(Conversation info\|Conversation metadata\|会话信息\|对话信息)\s*(\([^)]+\))?\s*:\s*` ``` `[\s\S]*?` ``` → 删 | 会话元数据块 |
| 5 | `Sender\s*\([^)]*\)\s*:\s*` ``` `[\s\S]*?` ``` → 删 | Sender 元数据块 |
| 6 | ` ```json\s*([\s\S]*?)``` ` + JSON 体匹配 `(session\|sessionid\|sessionkey\|conversationid\|channel\|sender\|userid\|agentid\|timestamp\|timezone\|message_id\|sender_id)` **≥3 个键** → 删 | 平台注入的结构化元数据 |
| 7 | `\[message_id:\s*[^\]]+\]\s*` → 删 | 消息 ID 前缀 |
| 8 | `^[a-z0-9_]{10,}:\s*` （多行，逐行） → 删 | 飞书等平台的 `ou_xxxxx: ` 发言人前缀 |
| 9 | `\n{3,}` → `\n\n` ；末端 `.trim()` | 空白归一化 |

清洗后为空串 → 本 user 消息跳过。

**assistant 消息处理**（保留结构化，不再是平面字符串）：

content 必须是 Array（OpenClaw 原生格式）。按 block 过滤：

| block.type | 处理 |
|----|----|
| `text` | 保留 |
| `thinking` | 保留，但删 `thinkingSignature` 字段（Anthropic base64 加密签名，几 KB，对学习无用） |
| `toolCall` | 保留，删 `id` 字段（仅用于与 toolResult 配对，而 toolResult 已整条丢）；若 `arguments == null` 或 `Object.keys(arguments).length === 0` → 整个 block 跳过（传参失败的废调用） |
| 其他 | 丢 |

assistant 顶层元数据（api / provider / model / usage / cost / stopReason / responseId / timestamp）单条可占几十 KB → 都不保留（因为只透传 `content` 字段，顶层字段天然被忽略）。

过滤后 block 数组为空 → 本 assistant 消息跳过。

#### 3.1.3 toolResult 过滤：为什么只留错误、为什么截到 300 字符

这是 §3 里最容易误解的一段，展开说明：

**isError 判定**：取自消息对象的 `isError: boolean` 字段，由发起 tool_use 的 provider 层（Anthropic / OpenAI SDK）在工具执行后直接写入——工具抛异常 / 返回非零退出码 / provider 判定失败都会置 `true`。TS 侧**不做任何语义判断**，直接读这个字段。

**过滤规则**（顺序）：

```
① 若 !isError → continue（成功一律丢弃）
② 读 content，展开为纯文本 errText：
     - string：直接用
     - Array：遍历 type==="text" 的 block，text 字段用 "\n" 拼
③ errText.trim()；为空 → continue
④ errText.length > 300 → 截到 300 + "…(truncated)"
⑤ push { role: "toolResult", toolName, isError: true, content: errText }
```

**memory 工具（memory_recall / memory_store / memory_flush 等）与普通工具一视同仁**：成功丢（召回的记忆内容本来就存在别处，进 procedural 学习无增量价值，只加噪声）、错误保留（memory 调用的失败模式——查询参数错误、flush 超时、recall 零命中等——同样是 feedback 可学习的信号）。assistant 的 `toolCall` block 仍然保留，agent "何时决定调 memory" 的时机信号从 toolCall 直接可见，不依赖 toolResult。

**为什么不保留成功的 tool_result**：
- 成功工具输出通常是**KB 级日志**（build 日志、文件读出、命令 stdout），进 `mem_conversation` 会极快膨胀 token
- 语义上它是对话的副产物——agent 已经在后续 text / thinking 里消化并复述了结论
- 记忆提取关心的是"agent 做了什么、学到了什么"，而不是工具中间产物
- 若后续需要工具成功路径，Task Trace 阶段可从 assistant 的 `toolCall` + 后续回复重建 How/Result 摘要，不需要原始大日志

**为什么错误 tool_result 要保留**：
- 失败尝试**必须**进记忆——它是 feedback 和 task_specific_experience 的核心证据（"出现该报错时，改用此做法"）
- 如果丢了，下游永远学不到 error-pattern

**为什么截到 300 字符**：
- 错误输出头部通常已含关键信号（异常类型、报错消息、定位线索）；stack trace 末尾的帧对跨场景复用价值低
- 300 是经验值：既能放下一条典型 Python/Go/JS 错误的 "ErrorType: message + 1-2 frame"，又能把 `mem_conversation` 单条控在 KB 级
- 超长时追加 `…(truncated)` 明确标示，避免下游误以为是完整串

#### 3.1.4 打包写入

`cleaned: Array<{role, content, ...}>` 长度为 0 → 整轮跳过。否则：

```
roundText = JSON.stringify(cleaned)
memory_add({
  content: roundText,
  userId,
  sessionId: toolSessionId,
  conversationId: ctx.sessionId ?? "",
  ingestMode: (cfg.autoCapture || hasStoreSignal) ? "immediate" : "deferred"
})
```

> `scope` 字段不传——MCP 层 `ParseScope(NULL)` 默认返回 `GSPD_SCOPE_USER`，配合 `userId` 通过 `ValidateScopeConsistency`，与显式传 `scope: "user"` 等价。特殊场景（如 after_compaction 的系统级记录）才需要显式传 `scope: "system"`。

**一轮对话 = `mem_conversation` 一行**。`content` 是 JSON 数组字符串，内含 user / assistant / toolResult 三类条目。这是 C 端 Step 1 必须处理的前置结构。

---

### 3.2 C 端补齐（消费 `mem_conversation` 时）

TS 已经做掉大部分清洗，C 端只做 TS 覆盖不到的三件事：

| # | 工作 | 理由 |
|----|----|----|
| 1 | **JSON blob 反序列化** | TS 写入的是一整轮的 JSON 数组；窗口切分、质量过滤、Task Trace 提取等下游步骤均以"逻辑消息（单条 user / assistant / toolResult）"为单元，C 端必须先反序列化 |
| 2 | **敏感数据脱敏** | TS 侧**完全不做**密钥脱敏（依赖用户自觉）；进 LLM 前必须过 |
| 3 | **语言标注** | TS 仅用 CJK 字符做 minLen 阈值，不输出语言标记；C 端要算 session 级 lang 写入元数据供后续 LLM prompt 选中/英 |

> **不做机械 user 回复过滤**：TS 已通过 `NON_CONTENT_TEXT_RE`（纯标点）和 `COMMAND_TEXT_RE`（`/command`）兜底明显噪声；对"好的/继续/ok"这类短确认不再做 whole-turn 剔除——它们虽然主题密度低，但在 Task Trace 切分 sub-task 时是有效的**事务边界信号**（用户确认完这一步才进下一步），剔除反而会让 LLM 判断段界变困难。保留原文进窗口，让下游 LLM 自行消化。

#### 3.2.1 JSON blob 反序列化

```
读 mem_conversation.content (TEXT)
  → JSON.parse
  → 失败 → WARN log + 整条跳过（不进窗口，但推水位，避免卡住）
  → 成功 → Array<{role, content, ...}>

for 每个元素 el：
  el.role == "user"       → 作为一条 user 逻辑消息
  el.role == "assistant"  → 把 Array content 扁平化为文本：
                              text block → 直接拼
                              thinking block → `<thinking>{text}</thinking>`
                              toolCall block → `{toolName}({arguments_json})`
                            多 block 间 "\n\n" 分隔
  el.role == "toolResult" → 文本 `[tool:{toolName} ERROR] {content}`
```

一行 `mem_conversation` 拆出的多条逻辑消息共用同一 `id`；窗口切分按逻辑消息计数 (n=8 轮)。

#### 3.2.2 敏感数据脱敏（正则，对 user + assistant + toolResult 文本通吃）

```
sk-[A-Za-z0-9]{20,}                              → sk-***REDACTED***
Bearer\s+[A-Za-z0-9+/=]{20,}                     → Bearer ***REDACTED***
AKIA[A-Z0-9]{16}                                 → AKIA***REDACTED***
(api_key|secret|token|password)\s*=\s*["'][^"']+["']  → {key}="***REDACTED***"
/Users/[^/]+/                                    → /Users/****/
/home/[^/]+/                                     → /home/****/
```

顺序执行，原地替换，不可逆。

#### 3.2.3 语言标注（session 级，一次算出复用）

```
取 session 内近 N=20 条 user 逻辑消息文本拼接
CJK_chars / total_chars > 0.15 → session.lang = "zh"
否则                            → session.lang = "en"
```

写入 session 元数据（内存态即可，不必落 DB）；Task Trace / Fb/Tp / TSE 提取 prompt 按此选 `.zh.md` 或 `.en.md` 模板。

---

> **v5 曾提到的"窗口级 tool 输出 60/30 截断"已废弃**：TS 侧直接丢成功 tool_result、错误截到 300 字符，窗口内不会再出现大体量的 tool_result，60/30 截断失去作用对象。同样废弃："探索性工具调用整轮移除"（白名单匹配无意义，因为成功的 tool_result 已经不在窗口里）、"文件读取结果前 5 行保留"、"assistant 代码块前 10 行保留"。

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

## 5. Step 3/4 — 主题抽取 + 主题拼接

Step 3 抽取的主题（即 task）同时作为 Task Trace 的输入锚点和 TSE 的检索锚点。Fb/Tp 的分段锚点 sub-task 不在此阶段生成，由 Task Trace 提取阶段（§6.0）随 Trace 一起产出。

### 5.1 窗口主题提取（小模型 + DB 缓存）

调用入口 `ExtractTopicMetadataCached`（pipeline.c）——先查 DB 缓存，miss 才调 LLM。

**提取字段**（v6 精简至两个）：

| 字段 | 说明 |
|------|------|
| `topic_keywords` | 3-8 个 top-N 关键词，用于 Jaccard 主题边界判断；**零 LLM 可算**（TF 关键词） |
| `topic_summary` | 15-40 字自然语言一句话，TSE 检索锚点 + Task Trace task 头；**需要 LLM** |

已删：`topic_tag`（短标签职能被 summary 涵盖；sub-task 已承担 Step 组级的短语角色）、`task_category`（离散枚举当前无真实消费点，未来需要时再加）。

Prompt 文件：`src/prompts/files/procedural/topic_extract.{zh,en}.md`（Prompt ID `GSPD_PROMPT_PROCEDURAL_TOPIC_EXTRACT`）。

降级：LLM 不可用时走零 LLM TF 关键词提取路径，`topic_summary` 置空串；仅 `topic_keywords` 可用，Jaccard 边界判断仍能工作，TSE 检索锚点退化到 `user_query`。

#### 5.1.1 v6 目标 Prompt（中文版）

```markdown
# 窗口主题提取

你是一个对话主题分析专家。从滑动窗口的对话中提取主题元数据，用于后续的主题边界判断
和 Chunk 积累。

## 背景说明

当前窗口由 n=8 轮对话组成，首尾各有 m=2 轮与相邻窗口重叠（step=6）。
**锚点轮次**（anchor_turns）是窗口中间的非重叠部分，代表本窗口最核心的独有内容；
首尾重叠轮次仅作上下文补充，不作为主要提取依据。

## 字段说明

**topic_keywords**（3-8 个）
- 优先从锚点轮次的用户消息中提取
- 聚焦任务对象和操作动词（如"Docker 部署"、"SQL 优化"、"重构认证模块"）
- 过滤停用词（好的/继续/帮我/可以/这个）和短词（< 2 字符）
- 不同表述指同一对象时只保留一个
- 用途：下游主题边界判断（Jaccard 快速门控），以及 LLM 不可用时
  TSE 检索的关键词兜底

**topic_summary**（一句话，15-40 字）
- 自然语言描述窗口中"谁在做什么"
- 用途：作为 Task Trace 的 task 头，并作为 TSE 向量检索的语义锚点
- 包含具体对象和动作，例："用户在 production 环境调试 Docker
  部署，卡在 registry 推送权限"
- 纯闲聊 / 无明确任务 → 输出空串 ""

## 提取原则

1. 以锚点轮次为主要依据；完整窗口仅作上下文参考
2. 关键词来自用户消息，不从 assistant 回复中直接摘取
3. 没有明确任务时（如纯闲聊）→ topic_keywords 返回空列表，
   topic_summary 返回空串

## 输出格式

只输出 JSON：

{
  "topic_keywords": ["关键词1", "关键词2", "关键词3"],
  "topic_summary": "一句话 15-40 字，用户在做什么"
}

[User]
锚点轮次（主要依据，窗口非重叠核心内容）：
{{anchor_turns}}

完整窗口（上下文参考）：
{{window_messages}}
```

英文版 `topic_extract.en.md` 字段名与结构同步。

#### 5.1.2 待落地的代码改动（spec 已更新，代码尚未动）

prompt 文件本身（`src/prompts/files/procedural/topic_extract.{zh,en}.md`）当前仍是 v5 四字段版本，且窗口参数注释也是过期的 n=10/m=2。落地 v6 时按下表一次清掉，**不做向后兼容**（清空 DB 重建）：

| 文件 | 改动 |
|---|---|
| `topic_extract.zh.md` / `topic_extract.en.md` | ① 删 `topic_tag` / `task_category` 字段说明、JSON schema 对应键、"无明确任务"兜底分支里这两个字段；② "n=10 轮 / m=2 轮" → "n=8 轮 / m=2 轮 / step=6" |
| `src/ltm/procedural/learn/ltm_procedural_learn_chunk.c` | 删 L703-704 的 `out->topicTag = CopyJsonStr(...)` 与 `out->taskCategory = CopyJsonStr(...)`；同步清掉 `WindowTopic` 结构体里的 `topicTag` / `taskCategory` 字段及所有写入/打印路径（含 L499 的 LLM 数组兜底注释） |
| `src/ltm/procedural/learn/ltm_procedural_learn_pipeline.c` | 删 L401-402 日志格式串里的 `topic_tag='%s' task_category='%s'` 与对应实参 |
| `src/sql/sqlite/ltm_engine.sql` | `procedural_topic_cache` DDL 删 `topic_tag` / `task_category` 列；`SELECT` (L126) / `UPSERT` (L134-140) 同步去掉这两列绑定与 `excluded.*` 引用；ALTER 兼容路径（L122 附近）一并清理 |
| `src/ltm/procedural/learn/ltm_procedural_learn_chunk.c` + `include/ltm_procedural_learn.h` | ① 删除 `IsWindowHighQuality` 函数 + 头文件声明 + 调用点（v6 用空主题门控替代，见 §5.5）；同步清掉 `ChunkFlushV4` 里的 `(void)forced;` 短路标记。② 把 `PROCEDURAL_LEARN_JACCARD_THRESH` 从 `0.3f` 改成 `0.6f`（v6 收紧主题边界判断，避免主题误合并；L401 旧注释 "Jaccard ≤ 0.6" 反而是对的，常量改完后注释和实现就一致了） |
| `topic_boundary.zh.md` / `topic_boundary.en.md` | 重写为 v6 简化版（见 §5.4.1）：保留 `topic_summary × 2` + `keywords × 2`；删除 `chunk_anchor_turns` / `next_anchor_turns` / `chunk_last_user_msg` / `next_first_user_msg` 四类输入及其 prompt 占位符 |
| `ltm_procedural_learn_chunk.c` 调用 `JudgeTopicBoundary` 处 | 删除上述 4 个变量的绑定逻辑（anchor_turns 拼接、last_user_msg 抓取等）；`Render` 调用只保留 `topic_summary × 2` + `keywords × 2` |
| `src/sql/sqlite/ltm_engine.sql` | **新增**：① `procedural_concept` + `procedural_task_trace` + `procedural_task_trace_vec`（schema §8.1 / §8.2）；② `procedural_candidates`（fb/tp/tse 三类共用候选池，含 `anchor` / `payload` / `search_key` 三个泛化列 + 计数列，schema §8.0.3）+ 配套 `procedural_candidates_vec`；③ active 表加 canonical name 列：`procedural_tse_meta.task`、`procedural_fb_meta.sub_task`、`procedural_tp_meta.sub_task` + 索引（不走 FK，直存字面值） |
| `src/ltm/store/ltm_schema.c` | DDL 注册新表 + ALTER 加 TEXT 列（首次启动建库逻辑，不做老 DB 数据迁移） |
| `src/ltm/procedural/learn/ltm_procedural_learn_trace_store.{h,c}` | **删除** 旧的 `TaskTraceRecordInsert`（`mem_record(category=TASK_TRACE)` 路径），**新增** `ProceduralTaskTraceInsert(user_id, task, content, ...)` 直写新表；同时新增 `ProceduralConceptUpsert(user_id, kind, name, desc)` facade（仅维护字典计数） |
| `ltm_procedural_learn_chunk.c` Task Trace flush 路径 | ① 解析 Trace Markdown：`**Task**:` 行按全角冒号 split → (task_name, task_desc)，每个 `### sub-task N — ` 头 → sub_task_name；② 后处理 Time Range（从 chunk 边界 turn 的 created_at_ms 算 → 在 **Task** 行后插一行）；③ 对 task + 每个 sub_task UPSERT `procedural_concept`（仅统计）；④ INSERT `procedural_task_trace(user_id, task=task_name, content, ...)` |
| `ltm_procedural_learn_fb_store.c` / `_tp_store.c` | `FbBuildMemRecordContent` / `TpBuildMemRecordContent` 签名都改为 `(sub_task, description)`——**去掉 `context` 与 `name` 入参**（v6 已删两列；TP 的工具名由 LLM 写进 description）；写 `procedural_fb_meta` / `procedural_tp_meta` 时直填 `sub_task` TEXT 列；TSE 写入路径同样直填 `task` TEXT 列 |
| `mem_record` category 枚举 | 删除 `TASK_TRACE(13)` 这个 category；删除 `mem_record_raw_vec_13` 向量表（如有）；LEAF 层按 category 分表的清单减一类 |

C/SQL 改完后 §5.2 `procedural_topic_cache` schema 已是目标形态，无需再变。

### 5.2 主题缓存表 `procedural_topic_cache`

**背景**：水位只在 flush 时推进，未 flush 的 chunk 在下次 tick 会被重新组窗——如果不缓存，同一窗口会反复调 LLM 提主题。

```sql
CREATE TABLE procedural_topic_cache (
    session_id     TEXT    NOT NULL,
    first_turn_id  INTEGER NOT NULL,
    last_turn_id   INTEGER NOT NULL,
    keywords_json  TEXT,      -- JSON array，3-8 个关键词
    topic_summary  TEXT,      -- 15-40 字自然语言；LLM 不可用时为空串
    created_at_ms  INTEGER NOT NULL,
    PRIMARY KEY (session_id, first_turn_id, last_turn_id)
);
```

**生命周期**：
- Get：命中直接复用（跳过 LLM）
- Upsert：miss 后回填
- DeleteByRange：chunk flush 后清理 `last_turn_id <= chunk_last_id` 的所有缓存
- GC：7 天未命中的老条目兜底清理

### 5.3 主题关键词提取（零 LLM 降级）

`topic_keywords` 的零 LLM 兜底路径（LLM 不可用时走）：

```
取窗口内所有 user 消息
  → 分词（中文分词 or 空格分割）
  → 过滤停用词（好的/继续/ok/是的/不对/帮我/可以...）
  → 过滤短词（< 2 字符）
  → top-N TF 词 → topic_keywords
```

### 5.4 主题边界判断（Jaccard 快速门控 + 小模型兜底）

```
overlap = |keywords_A ∩ keywords_B| / |keywords_A ∪ keywords_B|

overlap > 0.6  → 同主题，直接 MERGE（跳过 LLM）
overlap ≤ 0.6  → LLM_DECIDE：调小模型判断（关键词不重叠不代表
                  主题不同，可能只是表述偏移，需要语义理解）
```

阈值常量：`PROCEDURAL_LEARN_JACCARD_THRESH`（`include/ltm_procedural_learn.h:43`），v6 目标 **0.6f**；代码当前仍是 `0.3f`，需更新（见 §5.1.2）。

#### 5.4.1 v6 目标 Prompt（中文版）

只用两端的 `topic_summary` 做核心判据，`topic_keywords` 作为辅助参考。删除原 prompt 里的 `chunk_anchor_turns` / `next_anchor_turns` / `chunk_last_user_msg` / `next_first_user_msg`——这四类输入是 LLM 重做一遍 summary 提取，冗余且占 token。

```markdown
# 主题边界判断

你是一个对话主题连续性分析专家。判断两段对话是否属于同一个任务主题，决定是否将它们合并为同一个 Chunk。

## 同主题 / 不同主题 的判断标准

**同主题**：
- 任务对象相同（同一文件、同一功能、同一问题）
- 后续内容是前续的延伸、深化、修正或补充
- 主题描述不完全重叠但语义属于同一工作流（如"写测试"→"修复测试失败"仍是同主题）

**不同主题**：
- 任务类型切换（如从"写代码"转向"写文档"）
- 工作对象切换（如从"前端组件"转向"数据库 schema"）
- 两段主题描述毫无关联

## 判断策略

1. **主题描述对比**（核心）：比较两段 topic_summary，看任务对象和操作是否连续。
2. **关键词参考**（辅助）：summary 描述偏差时（如双方都笼统说"调试问题"），用关键词核验是否同一领域；词汇重叠低不等于主题不同，不单独决定结果。

## 注意

片段 A 的主题描述是 chunk 内**最近一个窗口**的 topic_summary（chunk 主题渐变时，
chunk 整体身份按最近主题对齐——chunk 内部细节由后续 Task Trace 阶段重新整理，
不影响本判断）。

## 输出格式

只输出 JSON：`{"same_topic": true|false, "reason": "..."}`

[User]
片段 A 主题：{{chunk_topic_summary}}
片段 B 主题：{{next_window_topic_summary}}

关键词（参考）：
A: {{chunk_keywords}}
B: {{next_window_keywords}}
```

`same_topic=true → MERGE`，`false → FLUSH + 新 chunk`。

#### 5.4.2 已知的代码偏差

| 项 | 现状 | v6 目标 | 落地动作 |
|---|---|---|---|
| Prompt 输入字段 | 6 个（含 anchor_turns × 2、last_user_msg × 2） | 4 个（topic_summary × 2 + keywords × 2） | `topic_boundary.{zh,en}.md` 重写为上方版本；调用点删除多余 4 个变量绑定 |
| Jaccard 阈值常量 | `PROCEDURAL_LEARN_JACCARD_THRESH = 0.3f` | `0.6f`（v6 收紧） | `include/ltm_procedural_learn.h:43` 改值；`ltm_procedural_learn_chunk.c:401` 注释 "Jaccard ≤ 0.6" 已与目标一致，无需改 |

均纳入 §5.1.2 清理清单统一落地。


### 5.5 Chunk 积累与 Flush

```
chunk = ∅  (内存状态，不落 DB)

for 每个窗口 W:

  ── 空主题门控（v6 新增，替代旧的 IsWindowHighQuality）──
  if W.topic_keywords == [] and W.topic_summary == "":
      # 主题提取认定本窗口无 task（纯闲聊 / 无意义内容）
      跳过：不进 chunk，不参与边界判断
      水位推进到 W.last_turn_id（视作已处理）
      continue

  if chunk == ∅:
      chunk = [W]
      continue

  act = DecideChunkAction(chunk, W.firstMs, W.keywords)

  MERGE        → chunk.merge(W)            （Jaccard > 0.6）
  LLM_DECIDE   → JudgeTopicBoundary（LLM）
                   sameTopic → chunk.merge(W)
                   else      → flush(chunk) + chunk = [W]
  FLUSH        → flush(chunk) + chunk = [W]   （非同主题）
  FORCED       → flush(chunk, forced=1) + chunk = [W]   （达 3 窗口 / 2h gap）

兜底规则：
  正常 tick 末尾 → 不强 flush 残余 chunk，水位不动，下次重组
  forceSingleChunk=1（memory_flush 同步屏障）→ 强制 flush 残余
```

**关键行为**：
- 水位推进有两条路径：① **空主题窗口**直接推进（不需要 flush）；② **非空主题窗口**只在所属 chunk 真实 flush 时推进（`lastFlushedIdx` 跟踪）。
- 未 flush 的 chunk 留在内存，重启缓存丢失不丢数据——下次 tick 从水位重拉重组。
- memory_flush 同步屏障路径会叠加 `LTM_PROCEDURAL_PIPELINE_FLAG_FORCE_SINGLE_CHUNK`，把所有非空窗口合成一 chunk 一次 flush，跳过主题边界判断。

**为什么用空主题门控替代旧的 `IsWindowHighQuality`**：
- 旧门控规则：窗口含 tool_use 或 assistant 回复总长 ≥200 字符 → 通过；否则跳过
- 旧规则的问题：① 跟 autoCapture 的 JSON blob 单条 USER 包装冲突，长期短路；② 启发式不准——纯文本对话也可能含有效任务（讨论方案、决策），代码窗口也可能是无意义试错
- v6 新规则：让**主题提取的 LLM 自己判断有无 task**——主题提不出（keywords=[] 且 summary=""）= 没 task；用 LLM 的语义判断替代字符长度的启发式，更准确，且复用已经在跑的 LLM 调用，无额外成本

**Merge 时 chunk 字段更新规则**（零 LLM）：

| 字段 | 规则 | 理由 |
|---|---|---|
| `chunk.keywords` | **直接替换为 `W.keywords`**（最新 window 覆盖） | Jaccard 比较只看相邻两窗口，历史 keywords 累积只会膨胀 \|union\| 拉低后续命中率；忘掉旧 kw 反而让边界判断更敏锐 |
| `chunk.summary` | **直接替换为 `W.topic_summary`**（最新 window 覆盖） | TSE 检索锚点跟随主题最新进展即可；chunk 内主题渐变没关系——flush 时 Task Trace 提取会按整 chunk 原文重整结构化轨迹，summary 漂移不影响最终产出 |
| `chunk.first_turn_id` | 不变（首个 window 创立时锁定） | 用于水位推进与 topic_cache key |
| `chunk.last_turn_id` | 推进到 `W.last_turn_id` | 用于水位推进 |
| `chunk.window_count` | `++`（用于强制 flush 计数） | |

> 由于空主题窗口已在入口被门控，能进入 merge 的 W 必然 `topic_summary != ""`，无需再做"空主题保护"分支。

---

## 6. Step 5 — Chunk Flush（Task Trace + Fb/Tp + TSE）

Chunk flush 时依次跑 Task Trace → Fb/Tp → TSE。Task Trace 是 Fb/Tp/TSE 的统一证据源——原始对话不再直接喂给后三者。

---

### 6.0 Task Trace 提取

在主题拼接完成、chunk 即将 flush 时，先将 chunk 的原始对话整理为结构化的 Task Trace，作为下游 Fb/TP/TSE 提取的统一输入。

#### 6.0.1 设计意图

原始对话是 flat 的消息流，包含大量噪声（问候、 filler、重复确认）。直接用它作为 Fb/TP/TSE 的提取输入会导致：
- LLM 注意力被分散到无关消息
- 多次提取时重复解析同一对话结构
- 用户纠正信号和执行步骤混杂，不易定位

Task Trace 作为**原子事实层**：
- 把原始对话整理为"逐步做了什么"的结构化记录
- 保留工具调用、参数、结果、用户纠正等关键信息
- 去噪但不丢失证据（filler 移除，约束保留）
- 按 task / sub-task 两级组织，供下游按粒度选锚点：
  - **task**（chunk 级整体任务）→ TSE 检索/归类锚点
  - **sub-task**（Step 组级子主题，10-25 字描述短语）→ Fb/Tp 分段扫描的锚点
- 一次整理，多次消费（Fb/TP/TSE 共享）

#### 6.0.2 输入

```
chunk_messages: 已过滤已脱敏的对话序列（§3.2.1 反序列化产出）
                ── 这是 Task Trace LLM 的唯一输入

不传 topic_summary：
  - topic_summary 是窗口级产出（§5.1），chunk merge 时按
    recency wins（§5.5）只保留最后一个窗口的，对整 chunk 任务
    不一定准
  - LLM 应直接从 chunk_messages 归纳 canonical task name + description

不传 timestamp：
  - Time Range 由代码从 chunk_first_turn / chunk_last_turn 的
    created_at_ms 直接算，规则可达
  - LLM 在输出里不出 Time Range 行，代码后处理插入
```

#### 6.0.3 提取 Prompt（v6 精细化版）

```markdown
# Task Trace 提取

你是一个对话整理专家。把一段 agent 工作对话整理为 **Task Trace**——按 task / sub-task 两级层次组织的、按时间顺序排列的结构化执行记录。

## 1. 输入

**chunk_messages**：一段 agent 工作对话的清洗后消息序列（已过 §3 全部清洗，无平台元数据）。每条消息形如：

- `[user] {text}` —— 用户原话
- `[assistant] {text}` —— assistant 输出文本（可能含 `<thinking>...</thinking>` 段落，那是 agent 的内部推理）
- `[assistant] {toolName}({arguments_json})` —— assistant 发起的工具调用
- `[tool:{toolName} ERROR] {content}` —— 工具调用的错误返回（注：成功返回已被上游过滤丢弃，所以你只会看到失败 case；错误内容已截到 300 字符）

## 2. 术语严格定义

**Step** = 对话流里一个**有 procedural learning 价值**的逻辑节点，必须落入下表 4 类之一：

| Step 类型 | 触发条件 | 必填字段 |
|---|---|---|
| `UserRequest` | user 提出新请求、新约束、新上下文 | Action |
| `AgentAction` | assistant 仅含 text/thinking，无 tool 调用——做了规划、解释或决策表述 | Action, How（仅当含 thinking 时填） |
| `ToolCall` | assistant 含 toolCall 块——调了一次或多次工具 | Action, How, Parameters, Result |
| `UserCorrection` | user 对前一个 Step 表达不满 / 否定 / 偏好补充 | Action, Type, Trigger |

**task** = 你从 chunk_messages 归纳出的 canonical 任务概念，含两部分：
- **name**（4-10 字、动宾结构、抽象不带具体上下文，如 "修登录 bug" / "调 Docker 部署"）
- **description**（≤ 30 字一句话、抽象定义、不混入本次具体上下文，如 "修复登录系统中的代码 bug 或配置问题"）

输出时按 skill 风格 `name：description` 行内写在 `**Task**:` 行（中文全角冒号分隔）。

**sub-task** = task 内部一个**子事务**对应的 Step 序列。**只需 name**（10-25 字描述短语，自描述清楚做了什么），不附 description。子事务切换的硬触发信号：
1. 用户明确换方向（"好，这个搞定，现在做 X"）
2. 前一段交付完成（任务被验证通过 / 用户认可后转下一题）
3. 前一段失败放弃（多次尝试无果，agent 或 user 明示放弃此路径）
4. 工具链彻底切换（从"读 + 写文件"组合切到"运行测试 + 改 BUG"组合）

不切的情况：单纯多轮对同一对象的迭代（如反复修同一个文件）= 同一 sub-task。所有 sub-task 都从属于本 chunk 唯一的 task。

## 3. 字段语义（每个 Step 都按下表填，不留空，无内容写 "(none)"）

| 字段 | 适用 Step | 写什么 | 长度上限 |
|---|---|---|---|
| `Action` | 全部 | 一句话陈述"做了什么"——动宾结构，主语隐含；用户类 Step 主语隐含为 user，agent 类 Step 主语隐含为 agent | ≤ 30 字 |
| `Context` | UserRequest（可选） | 用户给出的相关背景（如"在排查上次失败的 build"） | ≤ 30 字 |
| `How` | AgentAction（必填当含 thinking）/ ToolCall | 实施方式：含工具名 + 关键决策理由（thinking 段落里"为什么选这个方案"提炼一句）。**不复述 Action**。 | ≤ 50 字 |
| `Parameters` | ToolCall | 关键参数键值对，按下表"关键参数白名单"取，不在表里的工具取 arguments 前 2 个键 | 每参数 value ≤ 80 字，超长截断加 `…` |
| `Result` | ToolCall | 成功 → "OK: {一句话产出概括}"；失败 → "ERROR: {错误首句}"（错误本身已截到 300，再选首行/首句即可） | ≤ 100 字 |
| `Type` | UserCorrection | enum，按下表"纠正类型判定"选 | — |
| `Trigger` | UserCorrection | 引用所纠正 Step 的编号（如 "Step 1.2"）+ 问题简述 | ≤ 30 字 |

### 3.1 关键参数白名单

按工具名前缀匹配：

| 工具类别 | 匹配规则 | 取的键 |
|---|---|---|
| 文件读写 | name ∈ {Read, Write, Edit, NotebookEdit, cat, head, tail} | `file_path`（必）；Edit 类多取 `old_string`/`new_string` 各 30 字 |
| Shell / 命令执行 | name ∈ {Bash, Execute, exec, sh} | `command`（首 100 字） |
| 搜索 | name ∈ {Grep, grep, Glob, Search, find} | `pattern` 或 `query` |
| 网页 | name ∈ {WebFetch, WebSearch, fetch, http} | `url` 或 `query` |
| 内存工具 | name 以 `memory_` 开头 | `query`（recall）/ `content` 首 50 字（store） |
| Agent / Task | name ∈ {Agent, Task} | `description` 或 `subagent_type` |
| 其他 / 未知 | — | arguments 的前 2 个键，按出现顺序 |

参数值含路径或 ID → 去标识化保留结构（如 `/Users/****/proj/main.c`）。

### 3.2 纠正类型判定

| Type | 信号词（user 消息内任一即命中） | 例子 |
|---|---|---|
| `correction` | 不对 / 错了 / 别这样 / 改用 / 不要用 / 应该 X 不是 Y / 反了 | "不对，应该用 await，不是 then" |
| `refinement` | 还要 / 也加上 / 顺便 / 记得 / 别忘 / 此外 | "还要考虑超时" |
| `feedback` | 以后 / 下次 / 我习惯 / 我喜欢 / 偏好 / 一律 | "以后这种 case 都用 X" |

同时命中多个 → 按 `correction > feedback > refinement` 优先级取一个。

## 4. 整理规则（按顺序执行）

1. **遍历 chunk_messages**，按消息时间顺序识别 Step；非上述 4 类的消息**整体丢**（filler、纯标点、纯标点 ack）。
2. **Filler 严格定义**：
   - user 消息 trim 后 ≤ 5 字符且匹配正则 `^(好的?|继续|ok|嗯+|收到|了解|yes|y|sure)[。！.!]*$` → 丢
   - assistant 仅 `<thinking>` 块、无 text 也无 toolCall → 把 thinking 内容并入下一条非 filler assistant 的 `How` 字段，本条不出 Step
3. **Thinking 处理**：thinking 段落不出独立 Step；其内容用于：
   - 紧邻其后的 `ToolCall` Step 的 `How` 字段（"决策理由"那一句从 thinking 里提炼）
   - 紧邻其后的 `AgentAction` Step（如果 assistant 没调工具）的 `How` 字段
   - thinking 后是 user 消息且没有 agent action → thinking 整体丢
4. **并发 ToolCall 处理**：单条 assistant 消息内多个 toolCall → 每个工具一个独立 ToolCall Step，按 arguments 出现顺序编号（Step 1.2、Step 1.3、...）；对应的错误 toolResult 按顺序+toolName 配对（不保证 100% 准，下游接受此风险）。
5. **Step 编号**：`Step {smallTopicIdx}.{stepIdxInSegment}`，从 1 开始；编号只用于 `Trigger` 字段引用，不做跨 chunk 持久化。
6. **sub-task 切分**：沿时间顺序扫 Step，按 §2 的硬触发信号切段；每段 **10-25 字描述短语**命名，动宾结构 + 必要修饰（"定位 NPE 崩溃点并修栈跟踪"、"为新增字段补单元测试"），名字本身要能说清这一段在做什么——不需要再附 description。整 chunk 只有一个子事务 → 只产出一个 sub-task 段。
7. **User Query 汇总**：扫所有 UserRequest Step 的 Action，归纳一句 1-2 句话的 "User Query"，写在 Trace 头部。多个不相关请求时取最初的那个 + 简短一句"以及后续 X、Y"。
8. **Step 不打时间戳**：Trace 头部已有 Time Range 字段；Step 内不出时间。
9. **不发明内容**：每个字段必须有 chunk_messages 里的可追溯证据；编不出来就写 `(none)`。

## 5. 输出格式

只输出 Markdown，严格按下方模板（不要外包代码围栏）。

- **task** 用 skill 风格行内格式：`name：description`（中文全角冒号分隔）。
- **sub-task** 只写 name，不附 description——name 本身写到 10-25 字描述短语级别，足以自描述（不像 task 那样需要额外 description 补语义）。

## Task Trace

**User Query**: <一两句话汇总，从所有 UserRequest Step 的 Action 归纳>
**Task**: <name 4-10 字 动宾结构>：<description ≤ 30 字 抽象不带具体上下文>

### sub-task 1 — <name 10-25 字 描述短语，动宾结构 + 必要修饰>

#### Step 1.1 — UserRequest
- Action: <…>
- Context: <… or (none)>

#### Step 1.2 — ToolCall
- Action: <…>
- How: <…>
- Parameters: <key1=value1 / key2=value2>
- Result: <OK: … or ERROR: …>

#### Step 1.3 — UserCorrection
- Action: <…>
- Type: correction
- Trigger: Step 1.2 / <问题简述>

### sub-task 2 — <name>

#### Step 2.1 — …
…

约束：
- **不输出 `**Time Range**` 行**——代码后处理时会在 `**Task**:` 行后面插入
- task name 与 sub-task name 都要**稳定**（同一类任务/子事务跨 chunk 应给同一 name）
- task description 描述**通用任务类型本身**而非本次具体上下文（不能写"修复 login.py 的 bug"，要写"修复登录 bug" / 描述"修复登录系统中的代码 bug 或配置问题"）
- 同一 trace 多个 sub-task 段重名时（不该出现，但兜底）→ 仍按段独立列出，下游 UPSERT 时 occurrence 自然合并

[User]
chunk_messages:
{{chunk_messages}}
```
#### 6.0.4 设计注解（为什么这么做）

**为什么严格定义 4 类 Step**：v5 prompt 里 Step 类型是开放的（"agent thinking / tool call / result / user correction"），LLM 经常把同一个工具调用拆成两个 Step（一个 thinking、一个 tool call），下游 Fb/Tp 提取又得重组。固定 4 类可枚举且互斥，LLM 输出形态稳定。

**为什么 thinking 不出独立 Step，而是并入 How**：
- thinking 自身没有外部副作用（不改文件、不调工具），单独成 Step 会让 Trace 充满"agent 想了想"这种水分行
- 但 thinking 里的"为什么选 X 不选 Y"是 feedback / TSE 提取的核心证据——不能丢
- 解法：把 thinking 折叠进它**直接驱动**的下一个有副作用 Step 的 `How` 字段，这样既保留决策理由，又不让 Trace 膨胀
- 边界 case：thinking 后是 user 消息（agent 想完没动手），说明这次思考没产出动作 → 整体丢

**为什么 Parameters 用白名单 + 长度上限，而不是 dump 整个 arguments**：
- 工具 arguments 经常含大字段（Write 的 content 几十 KB、Edit 的 old/new_string 数 KB），整 dump 会让 Trace 膨胀到 LLM context 爆
- 但 file_path / command / query 这类**定位性参数**对复盘价值最高（"agent 在改哪个文件"比"改成什么内容"更重要——内容是结果，文件是动作）
- 白名单按工具类别给 sensible default；未知工具用前 2 键 fallback，不会丢调用形态信息

**为什么 Result 只摘要不全留**：
- 成功 toolResult 上游已整条丢（§3.1.3），Trace 阶段拿不到原文，只能让 LLM 从后续 assistant 文本里反推"这次调用结果用在了哪"
- 失败 toolResult 上游已截到 300 字符，Trace 阶段再选首行/首句进 Result 字段，避免 stack trace 把单 Step 撑爆
- "OK / ERROR" 前缀让下游 Fb/Tp 提取一眼定位失败 Step

**为什么 UserCorrection 要分 correction / refinement / feedback**：
- Fb/Tp 提取阶段会把 correction 直接转成 hard feedback（is_hard=true，source_user_correction=true）
- refinement 是软补充，多半进 task_specific_experience 而非 feedback
- feedback 类（"以后..."）通常是跨任务的偏好，TSE 锚点应升到 chunk 级而非 step 级
- 不分类的话 Fb/Tp 提取得在它的 prompt 里再做一次分类，重复劳动且 LLM 容易判断不一致

**为什么 task 由 LLM 直接从 chunk_messages 归纳，连 topic_summary 都不传**：
- topic_summary 是窗口级产出（§5.1），chunk merge 时按 recency wins（§5.5）只保留**最后一个**窗口的——长 chunk 主题渐变时，最后一个窗口的 summary 不能代表整 chunk 的 task
- 不让 LLM "复用"一个不准的输入，反而避免 LLM 被误导；LLM 看 chunk_messages 全量自行归纳更准
- topic_summary 仍在 §5 留作 Jaccard 边界判断和窗口级语义信号，但不进 Task Trace prompt 输入
- canonical 名字漂移风险（"修登录 bug" vs "修复登录 bug"）由 §8.1 的 exact match dedup 容忍，未来可加 embedding 合并

**为什么 Time Range 完全不进 LLM 输出，由代码后处理插入**：
- chunk 边界 turn 的 `created_at_ms` 在写入路径上 100% 可知，规则可达
- LLM 看不到时间戳（`chunk_messages` 已剥离），让它输出占位符或自己编都是浪费 token
- 写入 `procedural_task_trace.content` 前，代码在 `**Task**:` 行后面直接 prepend `**Time Range**:` 行，零成本
- 这一行只是给人读和审计用，不参与解析流程

**为什么 task 用 skill 风格 `name：description` 行内格式，而 sub-task 只写 name**：
- task 是更高层的概念锚点（chunk 整体在做什么），跨 chunk 复用时光看 4-10 字短语可能不够（"调 Docker 部署"——是部署阶段还是调试阶段？），description 给一句话抽象定义补足语义
- sub-task name 写成 10-25 字描述短语（"为代码逻辑补 mock 单元测试"、"跑 build verify 检验改动"、"补错误处理与边界 case"），name 本身就承担了 description 的语义负载；额外加 description 重复劳动、占 token，收益不显著
- description 只在 task 首次出现时入库，后续 UPSERT 不更新——避免同一 task 的描述在多个 trace 里抖动
- 行内格式让 name 和 description 在 Markdown 结构里**就近耦合**，下游解析按中文全角冒号 split 即可拿到 (name, description)；如果走单独的"Concept Definitions"末段会要求 LLM 在两处都写一致的 name，漂移风险高

**为什么 sub-task 切分用"硬触发信号"而不是 LLM 自由判断**：
- 子事务边界判断对 LLM 来说是非常主观的任务，给 4 个明确信号能让不同 LLM、不同 chunk 的切分结果可重复
- 兜底"整 chunk 只有一个子事务时只出一段"明确禁止过度切分，避免每个 Step 都被切成单独 sub-task 段

**为什么 Step 不打 timestamp**：
- 头部 Time Range 已经界定时间范围，每 Step 标 timestamp 在 token 上是浪费
- 真要复盘时间线时通过 Step 编号顺序就够（chunk_messages 本来就是时间序）
- 但凡 LLM 看到模板里有 `[timestamp]` 占位符就会去编一个，干脆从模板里删掉

**为什么"不发明内容"用 `(none)` 而不是省略字段**：
- 字段稳定可让下游 Fb/Tp prompt 用固定正则定位 "Action:"、"Type:" 等行
- 省略字段会让结构不可预测，下游 prompt 得加各种"如果有 X 字段..."分支
- `(none)` 显式占位 + 一致字段顺序 = 下游零成本解析

**未在本版处理、留待迭代**：
- 多 chunk 拼接时同一对象的 sub-task 跨 chunk 归一（目前 chunk 内独立）
- ToolCall 失败-恢复链路的自动标注（目前需要 Fb/Tp 阶段自己识别）
- Parameters 白名单的项目特定扩展（如自家工具 `gspd_xxx` 的关键键）


#### 6.0.5 存储

Task Trace 写入 §8.2 `procedural_task_trace` 独立表（**v6 关键改动**：从早期 `mem_record(category=TASK_TRACE)` 路径完全搬出，事件记忆与知识记忆 schema 边界彻底分离）。

写入流程（chunk flush 阶段一次性完成）：

```
0. 解析 Trace Markdown：
   - **Task** 行 → split 中文全角冒号 → (task_name, task_description)
   - 每个 ### sub-task N — 头 → 取 "—" 后的整段作为 sub_task_name（无 description）
1. 后处理 Time Range：
   - 从 chunk_first_turn / chunk_last_turn 的 created_at_ms 算出范围
   - 在 Trace 文本里 **Task** 行后面插一行：
     **Time Range**: <first_iso> → <last_iso>
   - 这一行不参与解析，仅供 trace.content 内人读 / 审计
2. UPSERT procedural_concept(user_id, kind='task',
       name=task_name, description=task_description)
   - 仅维护字典统计；返回值不被业务表使用
3. for each sub_task_name in trace:
       UPSERT procedural_concept(user_id, kind='sub_task',
           name=sub_task_name, description=NULL)
   - 同样仅维护统计；不需要缓存任何 id
4. INSERT procedural_task_trace(
       user_id, task=task_name,
       content=<已插入 Time Range 的完整 Markdown>,
       session_id, chunk_first_turn_id, chunk_last_turn_id,
       created_at_ms=now)
   → 拿到 trace_id
5. 生成 embedding(input="{task_name} {user_query}")
   INSERT procedural_task_trace_vec(trace_id, embedding)
```

下游 Fb/Tp 提取的每条 item 直接用 LLM 输出的 sub_task name 字面值作为锚点，无需查表 resolve。

**向量入库目的**：trace 本身**不参与在线检索**（`memory_search` 默认过滤掉这张表）。embedding 留给离线 dreaming 阶段做"同类任务 trace 聚类"——按 `task` name group 后向量近邻挖跨任务共享模式。

#### 6.0.6 生命周期

- **写入**：chunk flush 时同步提取 + UPSERT concept + INSERT trace；事务包裹保证 trace 与 concept 一致性
- **在线消费**：仅作为 Fb/Tp 提取（§6.1）和 TSE 提取（§7）的统一证据源；不开放 agent 直接召回
- **离线消费**：积累足够多同类任务的 trace 后，可用于跨任务模式挖掘（workflow 抽象等方向，待后续迭代）
- **清理**：30 天 TTL，定时扫 `created_at_ms < now - 30day` 的行删除（CASCADE 自动清 trace_vec）；但 `procedural_concept` 字典**不跟随清理**——concept 是事实层，长期保留 occurrence/last_seen 计数

---

### 6.1 提取 Feedback + Tool Preference（每 task 一次 LLM 调用）

**调用结构**：每个 chunk 的 task 触发**一次** `LearnFeedbackAndPreference` LLM 调用——LLM 看完整 Task Trace，产出本 task 内所有的 fb/tp 条目，每条带 `sub_task` 字段标注归属。**不**按 sub-task 分别调用（那样 N 个 sub-task 就要 N 次 LLM，浪费且无收益）。

**sub-task 在这里的角色**：仅作输出 item 的**锚点字段**，给下游 dedup 做分区键用。LLM 在分析时按 sub-task 段定位证据范围（同 sub-task 段内的 Step 才算同一上下文），但全部输出走一个 JSON。

**输入结构**：
```
Task Trace ← 完整 Markdown 文本（§6.0 产出，唯一证据源）
  含 User Query、**Task**: name：description、若干 ### sub-task 段
  每段内 Step-by-step 记录 + correction 标注
```

Task Trace 自身已含 task name + description（在 `**Task**:` 行）和所有用户纠正信号（已结构化为 Type=correction Step），不需要额外传 task name/description 或 raw user messages。

**LearnFeedbackAndPreference Prompt**：

```
你是一个程序记忆提取专家。从以下 Task Trace 中提取 feedback 和 tool_preference，每条结果标注归属的 sub-task。

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

## 证据范围与 sub_task 归属（CRITICAL）

- 你看到的 Task Trace 含若干 `### sub-task N — <短语>` 段
- **证据范围限定在 sub-task 段内**：判定一条 fb/tp 时只看它所在 sub-task 段的 Step；不要把段 A 的纠正信号和段 B 的工具调用拼一起当依据
- 产出每条 item 时填 `sub_task` 字段 = 该条证据所在段头部的 `<短语>` 原文（原样照抄、含空格标点）
- 同一条规则若在多个段都独立成立 → 每段各出一条（下游 dedup 阶段会按 sub_task 分区合并）
- 整个 Trace 只有一个 sub-task → 所有 item 的 sub_task 都填该段短语

## 分析框架

**1. 用户纠正分析**
- Trace 里标注 "Type: correction" 的 Step 是主要分析对象
- 提取结果 → feedback（is_hard=true，source_user_correction=true）
- 若纠正涉及工具选择 → 同时产出 tool_preference（is_hard=true）

**2. 失败→恢复分析**
- 扫描 Result 为 ERROR 的 Step，看后续是否有同 sub-task 内的恢复 Step
- 关键：该错误是否可能在下次同类 sub-task 中重现？
- 可泛化的恢复路径 → feedback（is_hard=true，source_user_correction=false）
- 一次性错误 / 环境偶发错误 → 不提取

**3. 工具选择模式分析**
- 同 sub-task 段内反复选择同一工具（≥ 3 次）？→ tool_preference（is_hard=false）
- 是否有明确的"优先用 X 不用 Y"模式？→ tool_preference

## 提取规则

- Task Trace 是唯一证据源（已结构化，定位高效）
- 去标识化：用占位符替代具体文件名 / 项目名 / 用户名
- 一次性、不可泛化的操作 → 不提取
- 纯闲聊 / 无操作内容 → 不提取
- 没有任何可提取内容时输出 `{"items": []}`，不要硬凑

## 输出格式

只输出 JSON：

{
  "items": [
    {
      "kind": "feedback" | "tool_preference",
      "sub_task": "<该条证据所在 sub-task 段头部短语原文>",
      "description": "<规则描述，≤ 50 字>",
      "is_hard": true | false,
      "source_user_correction": true | false
    }
  ]
}

[User]
{{task_trace}}
```

**回填保护**：若 LLM 输出的 `sub_task` 与 Task Trace 里任何 `### sub-task` 头部短语都对不上（拼写漂移、编造等），该 item 记 WARN log 并降级为"取该条目证据所在最邻近的 sub-task 段"的兜底归属，不丢数据但标记 `sub_task_fallback=true`。

**直接用 name 作锚点**：每条 item 的 `sub_task` 字段就是 canonical name 字面值，与 Trace 段头一致。下游 §8 候选池 dedup 按 `(kind, sub_task)` 这一键直接分区，不需要任何 id 解析步骤。


### 6.2 候选池更新（单层，只查候选池）

提取出的候选条目进入 §8 候选池处理。**v6 取消"先查 active 表再查 candidate 池"的两层结构**——在线只对候选池做查重（按 `(kind, sub_task)` 严格分区，hybrid score 门控），命中就 mention/hard ++ + bullets/description 合并；未命中就新建 pending。

active 表（`procedural_fb_meta` / `procedural_tp_meta` / `procedural_tse_meta`）由 §13 离线 worker 在升格时刻从 candidate 快照写入；在线流程**不读不写** active 表。

---

## 7. Step 5 — 提取 task_specific_experience（TSE）

### 7.0 总体流程

**Candidate 是 TSE 内容的"主文件"**，所有 LLM 提取/合并都在 candidate 上进行；active `mem_record` 只是 candidate 在升格时刻的**快照副本**。

```
chunk flush 时：
  ① 召回候选（按当前 task 找相似 candidates）
  ② Judge：LLM 选 selectedIndex（0=新建 / >0=命中某 candidate）
  ③ 命中 → Enrich：LLM 把本 chunk 的 bullets 合并进该 candidate
     未命中 → Learn：LLM 从 Task Trace 提新 bullets，INSERT 新 candidate
  ④ candidate.mention_count++（升格不在线触发，见 §8.7）
  ⑤ 不在线升格——升格统一在 §13 离线 worker 触发

升格时（§13 离线，可重复）：
  ⑥ candidate.mention_count 跨过新阈值 → 把 candidate 当前 bullets_json
     快照同步到 active（INSERT 新 mem_record 或 UPDATE 已有 mem_record.content）
  ⑦ candidate **保留**，继续累积；下一阈值跨过时再同步一次
```

为什么 candidate 要保留：① 后续 chunk 仍要往这个 task 的"主文件"里加 bullets；② Judge/Enrich 都基于 candidate 而不是 active，避免 active 过时不准；③ 升格本质是把 candidate 的当前态拷贝给召回路径用，candidate 自身仍是 live。

### 7.1 召回候选（按 task 名相似度）

输入：当前 chunk 的 `task_name` + `task_description`（来自 §8.1 procedural_concept）。

```
embed_input = "{task_name}: {task_description}"
TseCandidateSearchByEmbed(user_id, embed_input)
  → top-5 candidates 按 hybrid score 排序
    （0.80×cos + 0.20×kw，仅 status='pending' 或 'promoted' 都算——
     candidate 表里 promoted 行也保留参与查重）
  → score < 0.60 过滤（参见 §2.3 召回过滤阈值）
```

如果当前 task 是首次出现（exact name 在 candidate 表里没有），向量召回也可能命中"语义相近的旧 task"——比如 "修登录 bug" vs "修复登录 module" — 让 Judge 决定是否归并。

### 7.2 Judge（小模型）

判断当前 chunk 的 task 是否归入 top-5 中某条 candidate。

**Prompt**：

```
[System]
你是一个严格的匹配判断器。判断当前 chunk 的 task 是否应归入下面某条 candidate task。

必须是同一类工作任务才算匹配——宽泛 / 切线相关不算。

当前 task：{{task_name}}
当前 description：{{task_description}}

候选 candidates（序号、task name、description、bullets 数）：
{{candidate_list}}

规则：
- 仅当当前 task 明确属于某 candidate 的领域时输出该序号（1~N）
- 不确定时倾向于匹配（输出最接近的序号），宁可走 Enrich 也不要重复创建

只输出 JSON：{"selectedIndex": 0, "reason": "..."}
```

- `selectedIndex = 0` → 走 §7.4 Learn（新建 candidate）
- `selectedIndex > 0` → 走 §7.3 Enrich（合并进选中 candidate）

### 7.3 Enrich（大模型，当 Judge 命中时）

把本 chunk 总结的新 bullets 合并进选中 candidate 的 `bullets_json`。

**输入**：
- 选中 candidate（含 task_name、task_description、当前 bullets_json）
- 当前 Task Trace 完整 Markdown

**Prompt**：

```
[System]
你是一个任务经验合并专家。把本次 chunk 的 Task Trace 中可复用的经验合并进已有 candidate 的 bullets 列表。

## 合并原则

1. 语义并集：去重融合，意思相近的多条留表述更清晰的一份
2. 防退化：不能丢已有 bullets 的有效约束
3. 去标识化：移除案例特定实体（项目名/文件名/具体值），用占位符
4. RECENCY BIAS：冲突时新 chunk 的表述优先（近期意图）
5. 不发明内容：只用 Task Trace 里有证据的步骤/技巧
6. 单条 bullet ≤ 50 字

## 提取来源

- 失败-恢复路径中提炼的"以后遇到 X 时应该 Y"
- 用户主动追加的隐性要求（"做 X 时记得 Y"）
- agent 多次重复且看似有效的工具组合 / 步骤模式
- 用户明确认可的有效做法

## 不提取（避免与 fb / tp 重复）

- Trace 里某 sub-task 段已标 `Type: correction` 的 → 那是 feedback
- 工具偏好类（"做 X 优先用 Y 不用 Z"）→ 那是 tool_preference
- 一次性、不可泛化的操作

## 输出 JSON

{
  "action": "enrich" | "no_change",
  "bullets": [
    {"text": "<≤ 50 字>"}
  ],
  "added": ["新增内容简述"],
  "removed": ["合并去掉的内容简述"],
  "reason": "总体判断"
}

action=no_change 时 bullets 保留原 candidate 的不动；下游只 mention++ 不改 bullets_json。bullet 不区分 hard/soft——TSE 整体按次数规则升格（§8.7），与 fb/tp 的 is_hard 短路路径无关。

[User]
candidate 当前 bullets:
{{candidate.bullets_json}}

新 Task Trace:
{{task_trace}}
```

**写入**：UPDATE candidate
- `bullets_json` ← 输出的 bullets（action=enrich）/ 不动（action=no_change）
- `mention_count++`
- `total_updates++`
- `last_seen_at = now`
- `source_trace_ids` append 本次 chunk 的 `procedural_task_trace.id`（超 20 条 FIFO 淘汰最老）
- `hard_count` 不动（TSE 不用，恒为 0）

### 7.4 Learn（大模型，当 Judge 未命中时）

从 Task Trace 提取新 candidate 的初始 bullets。

**输入**：当前 Task Trace 完整 Markdown（task_name + task_description 已编进 Trace 头部 `**Task**:` 行，LLM 自然能看到）。

**Prompt**：

```
[System]
你是一个任务经验提取专家。从 Task Trace 中提取本次执行总结出的可复用经验 bullets，作为该 task 的新 candidate 的初始内容。

## bullet 定义

- 每条 ≤ 50 字，描述"做这类任务时应该 / 可以 / 注意 ..."的执行智慧
- 写"做法"，不写"具体做了什么"
- 去标识化：用占位符替代具体文件名 / 项目名 / 数值

## 提取来源 / 不提取

（同 §7.3 Enrich 中的列表）

## 输出 JSON

{
  "bullets": [
    {"text": "<≤ 50 字>"}
  ]
}

无可提取内容 → `{"bullets": []}`，本 chunk 不创建 candidate。bullet 不区分 hard/soft——TSE 整体按次数规则升格。

[User]
{{task_trace}}
```

**写入**：`TseCandidateInsert(user_id, task=task_name, task_description, bullets=输出, trace_id)` → INSERT procedural_candidates(
- user_id, kind='tse',
- anchor=task_name,
- payload=JSON(输出的 bullets),
- search_key="{task_name}: {task_description}"（写入时拼好固化）,
- source_trace_ids=JSON([trace_id])   -- 本次 chunk 的 trace_id
- mention_count=1, hard_count=0, total_updates=1, is_hard=NULL,
- last_promoted_at_mention=0,
- status='pending', first_seen_at=now, last_seen_at=now)

### 7.5 升格

不在线触发。详见 §13 离线 worker：mention_count 跨过下一阈值时，把 candidate.bullets_json 当前态快照到 active（mem_record + procedural_tse_meta），candidate 本身保留继续累积。

## 8. 数据结构

三种程序记忆各用一张独立表存储，schema 按类型特点设计，辅以两张管理表（§8.0）支撑水位与 topic 缓存。

### 8.0 管理表

#### 8.0.1 `procedural_session_progress`

```sql
CREATE TABLE procedural_session_progress (
    session_key          TEXT    PRIMARY KEY,
    last_processed_id    INTEGER NOT NULL DEFAULT 0,
    last_processed_at_ms INTEGER NOT NULL DEFAULT 0
);
CREATE INDEX idx_procedural_sess_last_id
    ON procedural_session_progress(last_processed_id);
```

per-session 一条水位；Fb/TP 和 TSE 合并进 chunk flush 统一处理，不按 kind 分轨道。

#### 8.0.2 `procedural_topic_cache`

见 §5.2。用途：跨 tick 复用同窗口的 topic 提取 LLM 结果，flush 后按覆盖范围清理，7 天 GC 兜底。

#### 8.0.3 `procedural_candidates`（fb / tp / tse 三类共用候选池）

v6 把 fb / tp / tse 的候选池**合并到一张表**——所有在线提取/更新都落到这里，§13 离线 worker 从这里扫升格。

```sql
CREATE TABLE procedural_candidates (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id         TEXT    NOT NULL,
  kind            TEXT    NOT NULL
                  CHECK(kind IN ('feedback', 'tool_preference', 'tse')),

  anchor          TEXT    NOT NULL,
                  -- fb / tp: sub_task canonical name
                  -- tse:    task canonical name
  payload         TEXT    NOT NULL,
                  -- fb / tp: description（单条规则文本）
                  -- tse:    bullets_json（JSON 数组，多条 ≤ 50 字 bullet）
  search_key      TEXT    NOT NULL,
                  -- 用作 hybrid 召回的 embedding 输入，格式统一 "{anchor}: {content}":
                  -- fb / tp: "{sub_task}: {description}" 字面值（INSERT 时拼好，固化）
                  -- tse:    "{task}: {task_description}" 字面值
                  --         （task_description 来自 §8.1 procedural_concept join）
                  -- 设计意图：
                  --   fb/tp 同 sub_task 分区内 description 不同 → cos 能区分
                  --   跨分区时共享前缀 → 有相关性但不干扰
                  --   tse 每 task 一行，不重复
                  -- search_key 一次写入**不更新**（即使 payload 改 / task_description 改也不动）

  -- 计数（升格完全按次数触发，见 §8.8）
  mention_count   INTEGER NOT NULL DEFAULT 1,    -- 学到的次数（每次 chunk 提取命中/新建 +1）
  hard_count      INTEGER NOT NULL DEFAULT 0,    -- fb/tp 专用：is_hard=true 命中次数；tse 恒 0（不使用）

  -- fb / tp 用；tse 留 NULL（tse 的 hard 信号在 bullet 粒度，不作整 candidate 级短路）
  is_hard                INTEGER,                  -- 1=每次都升格（硬约束）
  source_user_correction INTEGER,

  -- 溯源（升格时 LLM 合理性检查要读）
  source_trace_ids TEXT NOT NULL DEFAULT '[]',            -- JSON array of procedural_task_trace.id
                                                           -- 每次 chunk flush 命中/新建 candidate 时 append
                                                           -- 超过上限（如 20 条）按 FIFO 淘汰最老的

  -- 升格追踪（§13 离线 worker 维护）
  last_promoted_at_mention INTEGER NOT NULL DEFAULT 0,   -- 上次成功升格时的 mention_count
  last_promoted_at_ms      INTEGER,                       -- 上次升格的时间戳（audit / debug）
  last_promoted_record_id  INTEGER,                       -- 指向已生成的 mem_record.id；NULL = 还未升过格

  status          TEXT    NOT NULL DEFAULT 'pending'
                  CHECK(status IN ('pending', 'promoted')),
                  -- 'promoted' 表示至少升过一次；不阻止后续再升格。
                  -- v6 暂不引入 'discarded' 降级路径
  first_seen_at   INTEGER NOT NULL,
  last_seen_at    INTEGER NOT NULL
);

-- 候选池主查询索引：按 (user_id, kind, anchor) 分区做 hybrid 查重
CREATE INDEX idx_proc_cand_lookup
    ON procedural_candidates(user_id, kind, anchor);

-- TSE 每个 task 仅 1 行（fb / tp 同 sub_task 下允许多条不同 description 的候选）
CREATE UNIQUE INDEX idx_proc_cand_tse_unique
    ON procedural_candidates(user_id, anchor)
    WHERE kind = 'tse';

-- 离线 worker 升格扫描：待升格（mention_count > last_promoted_at_mention）的行
CREATE INDEX idx_proc_cand_promote
    ON procedural_candidates(status, mention_count, last_promoted_at_mention);
```

**关键约束**：
- `anchor` / `payload` / `search_key` 三列在 schema 层用泛化命名以容纳 3 类——业务代码按 kind 分支知道每列具体含义
- 同 `(user_id, kind)` 分区内才参与 hybrid 查重；跨分区永远不混淆
- `search_key` 一次写入后**不重算**——避免 candidate 频繁更新时 embedding 反复刷新；除非 task_description 在 §8.1 被刷写（罕见），不动它
- 配套向量表 `procedural_candidates_vec(candidate_id, embedding)`：embedding 用 search_key 作输入，§7.1 召回时按这个向量做 hybrid

### 8.1 procedural_concept（事实层：task / sub-task 名字字典）

事实记忆——每个 user 一份独立字典，task 和 sub-task 的 canonical 名字去重 + 计数 + （task 的）description。

**纯统计 / 描述用途，不被业务表通过 FK 引用**——FB/TP/TSE/trace 表直接存 sub_task / task name 作 TEXT 列，自建索引。本表的 `id` 列保留但未被 FK 引用，主键功能由 `(user_id, kind, name)` 唯一索引承担。

```sql
CREATE TABLE procedural_concept (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id         TEXT    NOT NULL,
  kind            TEXT    NOT NULL,             -- 'task' | 'sub_task'
  name            TEXT    NOT NULL,             -- canonical 名字（exact match 去重）
  description     TEXT,                          -- task 的一句话定义（LLM 给）；sub_task 通常 NULL
  occurrence      INTEGER NOT NULL DEFAULT 1,    -- 跨所有 chunk 累计出现次数
  first_seen_at   INTEGER NOT NULL,
  last_seen_at    INTEGER NOT NULL,
  UNIQUE(user_id, kind, name),
  CHECK(kind IN ('task', 'sub_task'))
);
CREATE INDEX idx_procedural_concept_lookup
    ON procedural_concept(user_id, kind, name);
```

**Canonicalization**：用 `(user_id, kind, name)` exact match，名字字面相同才视为同一条。名字漂移（"写单测" vs "写单元测试"）会产生两条独立条目，v6 容忍此风险——后续可加 embedding 相似度合并。

**索引使用模型**（v6：直存 name，不走 FK）：
- FB/TP/TSE/trace 直接存 sub_task / task 的 canonical name TEXT 列，各自建 `(user_id, name)` 索引
- 业务查询不 join 本表（如 "查 sub-task 'X' 的所有 fb" 直接 `WHERE sub_task = 'X'`）
- 本表只有两个用途：① 给 trace 写入路径累计 occurrence；② 存 task 的 description 供回填 prompt 上下文用
- 删除本表条目对业务表无影响（业务数据按 name 自洽）

**写入入口**：`ProceduralConceptUpsert(user_id, kind, name, description)` —— 命中 (user_id, kind, name) 则 occurrence++ 并刷新 last_seen_at；未命中 insert，occurrence=1。

### 8.2 procedural_task_trace（事件层：每次执行的结构化轨迹）

事件记忆——每次 chunk flush 产出一条 trace，记录"这一次"的完整执行轨迹。同一个 task 做 N 次 → N 条 trace（与事实层去重计数对照）。

```sql
CREATE TABLE procedural_task_trace (
  id              INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id         TEXT    NOT NULL,
  task            TEXT    NOT NULL,              -- canonical task name (字面值)

  content         TEXT    NOT NULL,              -- Task Trace Markdown 全文
                                                  -- 含 task / sub-task / step 三层结构

  session_id           TEXT,                      -- 来源 chunk 溯源
  chunk_first_turn_id  INTEGER,
  chunk_last_turn_id   INTEGER,

  created_at_ms   INTEGER NOT NULL
);
CREATE INDEX idx_proc_trace_user_task
    ON procedural_task_trace(user_id, task);
CREATE INDEX idx_proc_trace_created
    ON procedural_task_trace(created_at_ms);

-- 配套向量表：trace 嵌入（离线"同类任务 trace 聚类"用）
CREATE TABLE procedural_task_trace_vec (
  trace_id        INTEGER PRIMARY KEY
                  REFERENCES procedural_task_trace(id) ON DELETE CASCADE,
  embedding       BLOB    NOT NULL
  -- embedding 输入: "{task_name} {user_query}"
);
```

**task 列存 canonical name 字面值**——不走 FK，删除 procedural_concept 字典条目对 trace 无影响（trace 自带 task name，能独立查询）。

**TTL**：30 天自动清理，定时扫 `created_at_ms < now - 30day` 的行删掉。不加 `expires_at` 列。

**故意不放进 schema 的字段**：
- `sub_task` 列表 / JSON——能从 content Markdown 反扫得到（每个 `### sub-task N — <name>` 头部正则匹配即可拿 name 字面值）。先不冗余存，未来发现反扫太慢再补
- `state` / `status` 等过程态——trace 写入即终态

**写入入口**：`ProceduralTaskTraceInsert(user_id, task, content, session_id, chunk_first_turn_id, chunk_last_turn_id)`——`task` 字段直接传 canonical name 字符串。

**v6 新行为**：trace **不参与在线检索**——`memory_search` 默认过滤掉这张表的内容；它只服务两个用途：① Fb/Tp/TSE 提取的统一证据源（chunk flush 同步消费）；② 离线 dreaming（跨任务模式挖掘，未来）。

> **从旧设计搬出**：v6 早期版本曾把 Task Trace 放在 `mem_record(category=TASK_TRACE=13)` + `mem_record_raw_vec_13`。该路径已废弃；事件记忆与知识记忆 schema 边界彻底分离，`mem_record` 只承载 EXPERIENCE / FEEDBACK / TOOL_PREFERENCE 三个 category。

### 8.3 TSE（候选池 = 主文件 + active 快照，与 fb/tp 对称）

v6 起 TSE 与 fb/tp 一样走两层结构——同 task 多次出现累积进候选池，整体达阈值后才升 active。**升格不在线触发**，由 §13 离线 worker 统一处理。

#### 8.3.1 active 层：`mem_record` + `procedural_tse_meta`

- **主表 `mem_record`**：`memory_type = PROCEDURAL(2)`、`category = GSPD_CAT_EXPERIENCE(10)`、`tier = LEAF(2)`。`content` 字段由 `TseBuildMemRecordContent(task, bullets_json)` 拼成多行 Markdown（见 §7.3）。**embedding 输入**取 `"{task}: {task_description}"`（与 bullets 解耦——bullets 变化时只重写 content，不重算 embedding），写入 `mem_record_raw_vec_10`。
- **扩展表 `procedural_tse_meta`**：

```sql
CREATE TABLE procedural_tse_meta (
  record_id       INTEGER PRIMARY KEY
                  REFERENCES mem_record(id) ON DELETE CASCADE,
  task            TEXT NOT NULL,                 -- canonical task name 字面值（索引在此）
  bullets_json    TEXT NOT NULL,                  -- JSON array，每条 ≤ 50 字
  mention_count   INTEGER NOT NULL DEFAULT 1,     -- active 升格后仍累计
  hard_count      INTEGER NOT NULL DEFAULT 0,
  last_seen_at    INTEGER NOT NULL,               -- 最后一次被 chunk 命中时刷新
  source_sessions TEXT                            -- JSON array，来源 session_id 列表
);
CREATE INDEX idx_procedural_tse_meta_task ON procedural_tse_meta(task);
```

TSE 锚点为 task（chunk 级整体任务），直接存 task name 字面值（不走 FK）。**不携带 sub_task 字段**——同一 chunk 的所有 sub-task 段共享同一个整体任务视图。`mention_count` / `hard_count` / `last_seen_at` 在 active 期继续累积，供未来"长期不活跃 → 降级"等离线策略使用。

#### 8.3.2 pending 层：`procedural_candidates(kind='tse')`

TSE 候选**复用** §8.0.3 共享候选池，`kind='tse'` 区分。每行字段映射：

| `procedural_candidates` 列 | TSE 语义 |
|---|---|
| `kind` | `'tse'` |
| `anchor` | task canonical name |
| `payload` | `bullets_json`（JSON 数组，跨 chunk 累积的 bullets） |
| `search_key` | `"{task}: {task_description}"`（创建时从 §8.1 join 出 task_description 固化） |
| `mention_count / total_updates / last_seen_at / last_promoted_at_mention / ...` | 同 fb/tp 共用 |
| `hard_count` | TSE 不使用，恒 0（v6：bullet 不区分 hard/soft） |
| `is_hard` | TSE 留 NULL（v6：TSE 不走 is_hard 短路升格路径） |

`UNIQUE(user_id, anchor) WHERE kind='tse'` 唯一索引保证：每个 user 的同一 task 在候选池里**最多 1 行**（区别于 fb/tp 一个 sub_task 下可有多条不同 description 的候选）。新 chunk 命中已有 task 候选时，把新 bullets merge 进现有 `payload`，mention_count++。

#### 8.3.3 写入入口

procedural/learn 层 facade（`ltm_procedural_learn_tse_store.h`）——内部均落到 `procedural_candidates`：

- `TseCandidateSearchByEmbed(user_id, embed_input)` —— §7.1 召回 top-5（`WHERE kind='tse' AND user_id=U`，按 `procedural_candidates_vec` 向量近邻）
- `TseCandidateLoad(candidate_id)` —— 拿单条 candidate（payload 反序列化为 bullets list）
- `TseCandidateUpsertBullets(candidate_id, new_bullets)` —— §7.3 Enrich 写回 payload + mention++（TSE 不动 hard_count）
- `TseCandidateInsert(user_id, task, task_description, bullets)` —— §7.4 Learn 新建（写入时拼 `search_key="{task}: {task_description}"`）
- `TsePromoteCandidate(candidate_id)` —— §13 离线 worker 调用，把 candidate 当前 payload 反序列化为 bullets，写入 `mem_record + procedural_tse_meta`

检索（在线召回 agent）通过 `StoreSemanticSearch(tier=LEAF, filter.category=EXPERIENCE)` 命中 `mem_record_raw_vec_10`，仅 active 行可见——pending 候选不参与 agent 召回（candidate 池只服务下次 chunk flush 的查重）。

### 8.4 Feedback（候选池 = 主文件 + active 快照）

Fb 采用 **候选池 = 主文件 / active = 周期快照** 结构（v6 三类统一）：

- **候选池**：`procedural_candidates`（`kind='feedback'`）——Fb 内容的"主文件"，所有在线提取/合并都在此处累积；保留 mention_count / hard_count / total_updates / last_seen_at / last_promoted_at_mention 等计数与升格追踪字段，`anchor` 存 sub_task canonical name 字面值；升格后仍**保留**继续累积，下次 delta 再满 3 时同步 active（见 §8.7）。
- **active 层**：`mem_record`（`memory_type=PROCEDURAL, category=FEEDBACK(12)`）+ `procedural_fb_meta` 扩展表——由 §13 离线 worker 在升格时刻从候选池快照写入；在线流程**不读不写** active 表。

主表 `content` 由 `FbBuildMemRecordContent(sub_task, description)`
拼为 `"[{sub_task}] {description}"`（`sub_task` 缺失时退化为
`description`），同时作为 `mem_record_raw_vec_12` 的向量嵌入输入。
`sub_task` 作为 Step 组级锚点进入 embedding 有助于在向量空间中
天然把不同子事务的 Fb 分开。

```sql
CREATE TABLE procedural_fb_meta (
  record_id              INTEGER PRIMARY KEY
                         REFERENCES mem_record(id) ON DELETE CASCADE,
  sub_task               TEXT,                   -- canonical sub-task name 字面值（索引在此）
  description            TEXT NOT NULL,
  is_hard                INTEGER NOT NULL DEFAULT 1,
  source_user_correction INTEGER NOT NULL DEFAULT 0,
  source_sessions        TEXT
);
CREATE INDEX idx_procedural_fb_meta_sub_task
    ON procedural_fb_meta(sub_task);
```

`sub_task` 直接存 canonical name 字面值——这是 FB 跨任务复用的关键：在新 task 里识别到同名 sub-task 时，`WHERE sub_task = 'X'` 一查即拿到所有相关 FB，零 join。

### 8.5 Tool Preference（候选池 = 主文件 + active 快照）

TP 结构与 Fb 对称（候选池 = 主文件 / active = 快照）：

- **候选池**：`procedural_candidates`（`kind='tool_preference'`），同样记 `sub_task` TEXT；升格后保留继续累积。
- **active 层**：`mem_record`（`category=TOOL_PREFERENCE(11)`）+ `procedural_tp_meta` 扩展表，由 §13 离线 worker 同步。
- **facade**：`TpCandidateUpsert/SearchByEmbed/LoadByTaskAndName`、`TpPromoteCandidate`（`ltm_procedural_learn_tp_store.h`）。

主表 `content` 由 `TpBuildMemRecordContent(sub_task, description)`
拼为 `"[{sub_task}] {description}"`（`sub_task` 缺失时退化为
`description`），同时作为 `mem_record_raw_vec_11` 的嵌入输入。
工具名（如 "Bash" / "Read"）由 LLM 写进 description（如 "做日志
检索时优先用 grep 而不是 cat"），不再单独建列。

```sql
CREATE TABLE procedural_tp_meta (
  record_id       INTEGER PRIMARY KEY
                  REFERENCES mem_record(id) ON DELETE CASCADE,
  sub_task        TEXT,                          -- canonical sub-task name 字面值（索引在此）
  description     TEXT NOT NULL,
  is_hard         INTEGER NOT NULL DEFAULT 1,
  source_sessions TEXT
);
CREATE INDEX idx_procedural_tp_meta_sub_task
    ON procedural_tp_meta(sub_task);
```

同 §8.4 Feedback：`sub_task` 直接存 canonical name 字面值，是 TP 跨任务复用的关键。

### 8.6 候选池查重流程（v6 单层）

**核心原则**（v6 单层化）：在线只对候选池做查重，按 `(kind, sub_task)` 严格分区。**不查 active 表**——active 是 §13 离线 worker 周期性从 candidate 快照写入的衍生层，candidate 是主文件。

```
提取出候选条目 c（kind = feedback | tool_preference，带 sub_task name）
  │  ※ sub_task 字段直接是 LLM 输出的 canonical name 字面值，
  │    用 name 本身做分区键
  ▼
查候选池（按 (kind, sub_task) 过滤后跑 hybrid score）
  │
  │  combined = 0.80×向量余弦 + 0.20×关键词重叠（仅同 sub_task 分区内）
  │
  │  score ≥ 0.85 → merge
  │  score < 0.50 → add
  │  0.50–0.85   → 小模型决策 merge/add（§8.9 prompt）
  │
  ├─ merge（找到匹配 m）
  │   mention_count++
  │   if c.is_hard: hard_count++
  │   last_seen_at = now
  │   total_updates++
  │   source_trace_ids append 本次 chunk 的 trace_id（超 20 条 FIFO）
  │
  │   ── payload 合并（§8.10）──
  │   if score ≥ 0.85: 直接按 RECENCY BIAS 用 c.description 覆盖，is_hard 取 OR
  │   else (0.50–0.85 小模型裁为 merge): 调 §8.10 合并 prompt
  │     → 更新 m.description / m.is_hard（search_key 固化不动，embedding 不重算）
  │
  │   ── is_hard 短路（在线立即升格，见 §8.7 规则 A）──
  │   if m.is_hard:   # 取合并后的新值（原 false + c.is_hard=true 也要触发）
  │     promote(m)   # 同步 upsert active，不调证据检查
  │
  └─ add（未找到）
      新建 pending candidate（
        kind='feedback' / 'tool_preference'，
        anchor=c.sub_task, payload=c.description,
        search_key=f"{c.sub_task}: {c.description}"（INSERT 时拼好，固化），
        mention_count=1, hard_count=0 or 1,
        is_hard / source_user_correction 按 LLM 输出填,
        status='pending', last_promoted_at_mention=0,
        source_trace_ids=[本次 chunk trace_id]，
        first_seen_at=now, last_seen_at=now,
        total_updates=1）
      同步算 embedding(search_key) → INSERT procedural_candidates_vec

      ── is_hard 短路（在线立即升格）──
      if c.is_hard:
        promote(new_cand)   # 同步 upsert active，不调证据检查

非 is_hard 升格不在线触发 —— 由 §13 离线 worker 每天扫候选池，
按 §8.7 规则 B（delta ≥ 3）+ §13.2 证据检查 → upsert active。
```

### 8.7 升格规则（v6：次数驱动，零公式）

v6 放弃 support_score 公式，改用**纯次数阈值 + is_hard 短路**两条规则。**规则 A 在线触发**（§6.2 chunk flush 里立即升格），**规则 B 离线触发**（§13 worker 每天扫一次）。

```python
PROMOTE_DELTA = 3  # 每累积 3 次 mention 升一次

def should_promote_online(cand):
    """§6.2 chunk flush 里即时判断；仅 fb/tp 的 is_hard=true 走这条"""
    if cand.kind in ('feedback', 'tool_preference') and cand.is_hard:
        return True
    return False

def should_promote_offline(cand):
    """§13 离线 worker 里判断（fb/tp 的非 hard 条目 + tse）"""
    delta = cand.mention_count - cand.last_promoted_at_mention
    if delta <= 0:
        return False
    # 规则 A 的 fb/tp+is_hard 已在线处理，离线这里作兜底：
    # 如果在线 promote 失败（服务挂了/事务回滚），离线看到仍满足即补升
    if cand.kind in ('feedback', 'tool_preference') and cand.is_hard:
        return True
    # 规则 B: 自上次升格以来累积 ≥ 3 次 mention → 升格
    #   首次（last=0, mention=3）→ delta=3，升
    #   mention=4（last=3，delta=1）→ 不升
    #   mention=6（last=3，delta=3）→ 升，last=6
    #   mention 首次扫到时已跨到 4（last=0, mention=4，delta=4）→ 升
    #   （之前"mention % 3 == 0"会漏掉这个场景，v6 改为 delta 规则避免）
    if delta >= PROMOTE_DELTA:
        return True
    return False
```

**为什么 is_hard 在线不调证据检查**：
- is_hard=true 的判定来源就是刚才这次 chunk 的 Task Trace（LLM 在 §6.1 提取时读了 Trace 并标了 is_hard）
- 证据检查（§13.2）让同一小模型读**相同的 Trace** 再验证一次 → 循环，无意义
- is_hard 是"用户明确纠正"或"失败-恢复"路径，信号强，可直接信任 LLM 标注
- 离线 worker 处理非 hard 时才需要证据检查（那些是 soft 经验，需要累积证据）

**升格动作**（是 upsert）：

```python
def promote(cand):
    if cand.last_promoted_record_id is None:
        # 首次升 → INSERT mem_record + 对应 active 扩展表
        record_id = INSERT_MEM_RECORD(build_content(cand))
        INSERT_ACTIVE_META(record_id, cand)
        cand.last_promoted_record_id = record_id
    else:
        # 已有快照 → UPDATE 替换（不重算 embedding，除非 anchor 变，罕见）
        UPDATE_MEM_RECORD(cand.last_promoted_record_id, build_content(cand))
        UPDATE_ACTIVE_META(cand.last_promoted_record_id, cand)

    cand.last_promoted_at_mention = cand.mention_count
    cand.last_promoted_at_ms = now()
    cand.status = 'promoted'
```

**设计要点**：
- candidate 本身**永远不消亡**——升格后继续接受在线 chunk flush 的更新（mention++、bullets 合并等），下次自上次 promote 以来累积 ≥ 3 次 mention 时再同步 active
- active 快照**按 last_promoted_record_id 严格 1-to-1**，同一个 candidate 永远只对应同一个 mem_record.id，多次升格是对该 id 做 UPDATE 不是新增
- `last_promoted_at_mention` 防重：同一天 worker 跑两次不会多次升格

### 8.8 升格节奏示意

规则：
- **is_hard=true（仅 fb/tp）**：每次 chunk flush 命中/新建都**在线**立即升格（§6.2），delta ≥ 1 即触发
- **非 is_hard（fb/tp 软规则 + 所有 tse）**：离线 worker（§13）按 `delta ≥ 3` 触发

**平稳递增场景**（每 worker 扫描时 mention 只 +1）：

| mention_count | last_promoted | delta | 非 is_hard（离线） | is_hard（在线即时） |
|---|---|---|---|---|
| 1 | 0 | 1 | 不升 | **升** → last=1 |
| 2 | 0 | 2 | 不升 | **升** → last=2 |
| 3 | 0 | 3 | **首次升** → last=3（离线次日） | 升 → last=3 |
| 4 | 3 | 1 | 不升 | 升 → last=4 |
| 5 | 3 | 2 | 不升 | 升 → last=5 |
| 6 | 3 | 3 | **第二次升** → last=6 | 升 → last=6 |
| 9 | 6 | 3 | **第三次升** → last=9 | 升 |
| ... | ... | 累积到 3 即升 | 每次都升 | |

**跳跃场景**（worker 扫描间 mention 跨多步，`% 3 == 0` 旧规则会漏）：

| mention_count | last_promoted | delta | 新规则 | 旧规则（% 3）对比 |
|---|---|---|---|---|
| 初次看到 mention=4 | 0 | 4 | **升** → last=4 | skip（4%3≠0），要等 mention=6 |
| mention 从 3 跳到 7 | 3 | 4 | **升** → last=7 | skip（7%3≠0），要等 mention=9 |
| mention 从 6 跳到 10 | 6 | 4 | **升** → last=10 | skip（10%3≠0），要等 mention=12 |

新规则保证：只要自上次 promote 以来累积 ≥ 3 次 mention，worker 下次扫到必 promote，不受 mention 是不是恰好落在 3 的倍数影响。

这里的"升"= upsert active（首次 INSERT / 已有 UPDATE），见 §8.7。

### 8.9 小模型决策 Prompt（0.50–0.85 灰区，候选池查重用）

> **前置**：调用该 prompt 前已经按 `(kind, sub_task)` 做过分区过滤，传进来的 existing 和 candidate 的 sub_task name 字面相同。因此决策流程里不再讨论"场景不同就 add"的维度——sub-task 层面的分离已在上游完成。Prompt 模板里 `{{sub_task}}` 直接是 canonical name 字面值。

```
[System]
你是程序记忆库的条目管理器。
两条条目已经属于同一个 sub-task 子事务，
判断新提取的条目应该与已有条目合并（merge）还是作为独立条目新增（add）。

### 决策流程（按顺序执行）

0) 能力重叠硬门控
   如果两条描述的是同一核心能力 → 不得 add，只能 merge。

1) 3 轴身份判断（同 sub-task 分区内）
   a. 核心任务目标：要完成的事是否相同？
   b. 硬约束：不能做什么？关键限制是否一致？
   c. 工具或操作方法：涉及的工具/命令是否相同？
   → 3 轴高度重叠 → merge
   → 有实质差异（工具不同 / 约束冲突 / 方法相异）→ add

2) 兜底
   不确定 → add（宁可多一条，不误合并）

只输出 JSON：
{"action": "merge" | "add", "reason": "..."}

[User]
sub_task: {{sub_task}}

已有条目：
  kind: {{existing.kind}}
  description: {{existing.description}}

新提取条目：
  kind: {{candidate.kind}}
  description: {{candidate.description}}
```

### 8.10 合并执行 Prompt（score ≥ 0.85 自动触发 / 决策=merge 后触发）

```
[System]
你是程序记忆合并器。将已有条目与新条目合并为一个更完整的条目。
两条条目属于同一 sub-task 子事务（sub_task 相同，合并后保持该值）。

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
- kind / sub_task 保持不变

只输出 JSON：
{
  "kind": "...",
  "sub_task": "...",
  "description": "...",
  "is_hard": true | false
}

[User]
sub_task: {{sub_task}}

已有条目（历史）：
  kind: {{existing.kind}}
  description: {{existing.description}}
  is_hard: {{existing.is_hard}}

新提取条目（近期）：
  kind: {{candidate.kind}}
  description: {{candidate.description}}
  is_hard: {{candidate.is_hard}}
```

降级路径：LLM 失败时取两边 description 中较长的一份（近期优先 — RECENCY BIAS），`is_hard` 取 OR（任一 true 则 true），`sub_task` 沿用（同分区必然一致）。

---

### 8.11 Embedding 生成与降级

v6 涉及 3 张向量表，规则统一："一次写入、不重算"。

#### 8.11.1 三张向量表概览

| 向量表 | 源表 | embedding 输入 | 生成时机 | 重算？ |
|---|---|---|---|---|
| `procedural_candidates_vec` | `procedural_candidates` | `search_key`（§8.0.3） | INSERT candidate 时同步 | ❌（search_key 固化不更新） |
| `procedural_task_trace_vec` | `procedural_task_trace` | `"{task_name} {user_query}"`（trace 头部摘要） | INSERT trace 时同步 | ❌（trace 写入即终态） |
| `mem_record_raw_vec_{10,11,12}` | `mem_record`（active 层） | **与对应 candidate.search_key 字面一致**（见 §8.11.2） | §13 升格首次 INSERT 时 | ❌（后续 UPDATE 只改 content） |

#### 8.11.2 active 层 embedding 输入与 candidate 对齐

| Category | vec 表 | embedding 输入 | 对应 candidate search_key |
|---|---|---|---|
| EXPERIENCE(10) = TSE | `mem_record_raw_vec_10` | `"{task}: {task_description}"` | **完全一致** |
| TOOL_PREFERENCE(11) = Tp | `mem_record_raw_vec_11` | `"{sub_task}: {description}"` | **完全一致** |
| FEEDBACK(12) = Fb | `mem_record_raw_vec_12` | `"{sub_task}: {description}"` | **完全一致** |

**关键优化**：升格时 active 层的 embedding **不重算**——直接从 `procedural_candidates_vec` 拷贝过来。两张 vec 表的 embedding 值相同（只是 row key 不同），保证 agent 召回（查 active）和在线查重（查 candidate）使用**同一向量空间**，不会出现"candidate 层相似但 active 层不相似"的不一致。

#### 8.11.3 写入路径（同步生成）

默认**同步**内联在写入事务：

```
INSERT procedural_candidates (...)                 -- 行写入
  ↓
embedding = embedding_service.embed(search_key)    -- 同步调用
  ↓
INSERT procedural_candidates_vec(candidate_id, embedding)   -- 向量写入
```

Task Trace 和 active 层 upsert 的 embedding 写入流程同此。

#### 8.11.4 降级（embedding 服务不可用）

1. **内联重试 2 次**（指数退避 500ms → 1s）
2. 仍失败 → 源表行**照常写入**，向量表**跳过 INSERT**
3. 记 WARN log（table + row_id + 失败原因）
4. **后台 retry worker**（每 5 分钟扫一次）补齐缺失的 embedding：
   ```sql
   -- candidates
   SELECT c.id FROM procedural_candidates c
     LEFT JOIN procedural_candidates_vec v ON v.candidate_id = c.id
     WHERE v.candidate_id IS NULL
     LIMIT 100
   -- trace 同理
   ```
5. 补齐成功 → INSERT 对应 vec 行；失败 → 下次 retry

**降级期间影响**：
- candidate 缺 embedding → §7.1 SearchTseCandidate / §8.6 候选池 hybrid 查重漏掉该 candidate（跨 anchor 查重失效，同 anchor 内精确 eq 仍命中）；retry 后恢复
- trace 缺 embedding → 不影响在线（§13 合理性检查读 content 不读 embedding）；仅影响未来离线 dreaming 聚类
- mem_record 缺 embedding → agent 召回漏掉该记录；retry 后恢复

#### 8.11.5 Embedding 服务与模型

- 由 `adapters/llm` 的 embedding adapter 提供（配置项 `GSPD_ADAPTER_EMBED`）
- v6 spec **不锚定具体模型**，由部署配置决定
- 要求：
  - 向量维度与现有 `mem_record_raw_vec_*` schema 的 BLOB 列对齐
  - 三张 vec 表**用同一模型**——否则相似度不可比
  - 切换模型需**全量重算**所有 vec 表（工程动作，v6 不做在线迁移）

#### 8.11.6 候选池 vec 的 1:1 关系

```
procedural_candidates        procedural_candidates_vec
       id (PK)                       candidate_id (PK, FK)
       ...                           embedding BLOB
```

- `candidate_id` 是 vec 表的**主键 + 外键**
- `ON DELETE CASCADE`：candidate 删除 → vec 行跟随删除
- vec 表**从不 UPDATE**（search_key 不变 → embedding 不变）

同构规则用于 `procedural_task_trace_vec`（trace_id 为 PK+FK）和 `mem_record_raw_vec_*`（record_id 为 PK+FK）。

---

## 9. 完整数据流

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
  第一层清洗（autoCapture 侧已做；C 端补齐见 §3.2）
  │
  生成滑动窗口 [W0, W1, W2, ...]（n=8, m=2, step=6）
  │
  for Wi in windows:
    │
    ├─ [Step 3] ExtractTopicMetadataCached(Wi)
    │    先查 procedural_topic_cache（session × first_turn_id
    │    × last_turn_id）；miss → 调 LLM + 回填。
    │    输出 topic_keywords（Jaccard 用） + topic_summary
    │    （15-40 字一句话，TSE 检索锚点 + task 头）。
    │
    └─ [Step 4] 主题拼接状态机 DecideChunkAction
         Jaccard > 0.6 同主题 / ≤ 0.6 小模型 JudgeTopicBoundary
         达 3 窗口 / 2h gap 强制 flush
         │
         flush → ChunkFlush(chunk):
           │
           (A) Task Trace 提取（单次 LLM 调用）
               ExtractTaskTrace(chunk_messages)
               → Markdown trace 含：
                   User Query + **Task**: name：description
                   + 若干 ### sub-task N — <name> 段（每段含 Step 序列）
               → 后处理：插入 **Time Range** 行
               → UPSERT procedural_concept(kind='task', task_name, task_desc)
                 （字典统计；返回值不被业务表使用）
               → for each sub_task_name:
                   UPSERT procedural_concept(kind='sub_task', sub_task_name, NULL)
               → INSERT procedural_task_trace(user_id, task=task_name,
                                              content, session_id, ...)
                 + INSERT procedural_task_trace_vec(trace_id, embedding)
               → 下游 Fb/Tp/TSE 直接用 task_name / sub_task_name 字面值
           │
           (B) Fb/Tp 提取（每 task 一次 LLM 调用，sub-task 作锚点）
               LearnFeedbackAndPreference(Task Trace)
               LLM 按 sub-task 段产出 fb/tp 条目。
               单层查重（候选池，按 (kind, sub_task) 严格分区）：
                 hybrid score = 0.80·cos + 0.20·kw
                 ≥ 0.85 命中 → mention/hard/total ++
                 < 0.50 未命中 → 新建 pending candidate
                 0.50–0.85 灰区 → 小模型决策 merge/add
               在线仅 is_hard=true 时立即升格（§6.2 短路），
               其余非 hard 条目走 §13 离线 worker 升格
           │
           (C) TSE 提取（在线只写候选池，不升格）
               §7.1 SearchTseCandidate 按 task hybrid 召回 top-5
               §7.2 Judge 小模型选 selectedIndex
               命中 → §7.3 Enrich 合并新 bullets 进 candidate
               未命中 → §7.4 Learn 新建 candidate
               candidate.mention_count++ / hard_count++ / score 重算
               升格统一走 §13 离线 worker
           │
           (D) 推水位到 chunk 末尾 turn id
               StoreProceduralTopicCacheDeleteByRange 清缓存
               （升格不在此路径——由 §13 离线 worker 处理）
  │
  末尾：未 flush 的 chunk 留在内存，水位原地不动
        下次 tick 从水位重新拉 turn 重组（跨 tick 持久化）
        forceSingleChunk=1（memory_flush）→ 强制 flush 残余
```

---

## 10. 触发时机

| 事件 | 输入 | 备注 |
|------|------|------|
| `TimerProcessProceduralCache` | 每 **120 秒** 扫一次 `mem_conversation` 水位之后的新行 | 单一主触发路径 |
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

**数据来源 `mem_conversation`**：agent 每轮消息 append-only 入库，`status=0/pending`。procedural worker 只消费 `skip_extraction=0` 的对话行，不修改 `status`（该状态由 fact 管道拥有）；水位记录在 `procedural_session_progress`，推进由 `pipeline flush` 驱动（§5.5）。

### 10.1 跨 tick 持久化策略

未 flush 的 chunk 缓存只在 `InstanceWorker` 内存，水位不推；下一 tick 从水位重新拉 turn 重组。好处：
- 进程重启缓存丢失**不丢数据**——水位没推，再拉一遍即可。
- 不需要 DB 记录 chunk 缓存态，简化 schema。

### 10.2 串行处理

学习管线采用串行处理：一批消息跑完再捞下一批，不做并行。后台批处理不需要并发，串行天然避免多批同时修改同一经验的问题。

---

## 11. 模型分配

| 调用 | 模型 | 理由 |
|------|------|------|
| **§5 主题抽取 + 拼接** | | |
| §5.1 ExtractTopicMetadata | 小模型 | 结构化提取 topic_keywords + topic_summary；procedural_topic_cache 复用 |
| §5.4 JudgeTopicBoundary | 小模型 | 二分类（same_topic），结构化输出 |
| **§6.0 Task Trace 提取** | | |
| §6.0.3 ExtractTaskTrace | 大模型 | 长 chunk_messages 输入 + 两级层次（task + sub-task）Markdown 输出 + 按 4 类识别 Step + sub-task 硬切分，指令多，小模型稳定性不够 |
| **§6.1 Fb/Tp 提取（每 task 一次 LLM 调用）** | | |
| §6.1 LearnFeedbackAndPreference | 大模型 | 一次看完整 Trace，识别用户纠正意图 + 按 sub-task 段分区产出条目；每条带 sub_task 锚点 |
| **§7 TSE 提取（四步）** | | |
| §7.1 SearchTseCandidate | — | 零 LLM，`procedural_candidates_vec` 向量近邻 + hybrid score |
| §7.2 JudgeTaskExperienceMatch | 小模型 | 在 top-5 candidates 中选 selectedIndex（0=未命中） |
| §7.3 EnrichTseBullets | 大模型 | 命中时把新 bullets 合并进 candidate.bullets_json（语义去重融合） |
| §7.4 LearnTseBullets | 大模型 | 未命中时从 Trace 提初始 bullets 新建 candidate |
| **§8 候选池在线查重（灰区兜底）** | | |
| §8.9 小模型决策 Prompt | 小模型 | 0.50–0.85 灰区 merge/add 结构化决策 |
| §8.10 合并执行 Prompt | 小模型 | ≥ 0.85 或灰区决策=merge 时融合两条 description |
| **§13 离线升格 worker** | | |
| §13.2 CheckEvidence 合理性检查 | 小模型 | 读 candidate + 最近 5 条 source traces，判 passed: true/false |

---

## 12. 参考来源

| 设计点 | 来源 |
|--------|------|
| 滑动窗口思路（v6 参数 n=8/m=2/step=6） | MemOS Python 端（原参数 n=10，v6 收紧） |
| 敏感数据脱敏思路 | MemOS Generator（v6 在 §3.2 C 端落地） |
| 元数据剥离（Sender 块 / message_id / speaker 前缀） | MemOS captureMessages（v6 在 §3.1 TS 端做） |
| 语言检测（CJK 比例） | MemOS（§3.2 补齐项） |
| 哨兵消息过滤（NO_REPLY / HEARTBEAT） | AutoSkill captureMessages（v6 在 §3.1 TS 端做） |
| Task Trace 为主要证据 | AutoSkill（原为 USER-only；v6 升级为 Trace 统一证据源） |
| WHAT vs HOW 判断边界 | AutoSkill（§7 Learn prompt 仍强调） |
| RECENCY BIAS（近期意图优先） | AutoSkill 合并执行（§8.10 prompt 保留） |
| Anti-Duplication（近似短语去重） | AutoSkill Skill Merger（§8.10 合并原则保留） |

> **v6 已偏离原始设计**（来源 → v6 替代方案）：
> - Tool 输出 60/30 截断（MemOS）→ v6 不需要：TS 直接丢成功 tool_result、错误截到 300 字（§3.1.3）
> - 推理标签 `<think>` 剥离（MemOS）→ v6 不需要：assistant 的 thinking 保留为结构化 block（§3.1.2）
> - 分层输入 Primary + Full（AutoSkill）→ v6 取消：只传 Task Trace（§6.1、§7）
> - 支持度打分公式 + 动态阈值（AutoSkill Requirement Memory）→ v6 换成次数 delta 规则（§8.7）
> - 三路相似度 dedup（AutoSkill）→ v6 只剩两路 `0.80·cos + 0.20·kw`（§2.3），删除 name_jaccard
> - 维护决策 4 轴 add/merge/discard（AutoSkill）→ v6 只剩 3 轴（§8.9）：删除"适用场景 / context"轴（因为 v6 已按 sub_task 分区，context 字段整个废弃）；discarded 路径暂不做
> - Requirement 候选池架构（AutoSkill）→ v6 fb/tp/tse 三类共用单张 `procedural_candidates`（§8.0.3），原 AutoSkill 是独立表

---

## 13. 离线升格 worker

**触发**：每天 1 次（cron 时间 TBD），扫 `procedural_candidates` 全表——fb / tp / tse 三类共用。

**两步流程**（主要处理 fb/tp 非 hard + 全部 tse；fb/tp 的 is_hard 已在线升格）：

### 13.1 Step A — 按次数规则挑出待升格 candidates（零 LLM）

按 §8.7 `should_promote_offline()` 筛选，挑出候选集 `to_promote`：
- fb/tp 非 hard 且 `delta ≥ 3` → 入选
- tse 且 `delta ≥ 3` → 入选
- fb/tp 的 is_hard=true 在线通常已升过（§6.2），离线见到 `delta > 0` 也会入选作兜底

### 13.2 Step B — 合理性检查（小模型）+ 通过后升格

对 `to_promote` 里每条 candidate：

```python
def check_and_promote(cand):
    # 1. 拉 source traces 作证据
    trace_ids = json.loads(cand.source_trace_ids)[-5:]   # 最近 5 条
    traces = SELECT content FROM procedural_task_trace
             WHERE id IN trace_ids
             ORDER BY created_at_ms DESC

    # 2. is_hard 跳过证据检查（见 §8.7 理由）；否则调小模型检查
    if cand.kind in ('feedback', 'tool_preference') and cand.is_hard:
        pass   # 直接通过
    else:
        verdict = CheckEvidence(cand, traces)    # 见下方 prompt
        if not verdict.passed:
            log_warn(f"candidate {cand.id} fails evidence check: {verdict.reason}")
            return   # candidate 保留不升格；下次 delta 再满 3 时重试检查

    # 3. 通过 → upsert active（§8.7 promote 伪代码）
    promote(cand)
    cand.last_promoted_at_mention = cand.mention_count
    cand.last_promoted_at_ms = now()
    cand.status = 'promoted'
```

**CheckEvidence Prompt**（小模型）：

```
[System]
判断一条程序记忆条目是否在给出的证据 traces 里真的成立。

如果条目描述的规则 / 经验能在 traces 里找到具体对应的发生过的事件或模式
  → passed=true
如果 traces 里找不到足够支撑这条规则的实际痕迹（可能是 LLM 瞎编 / 过拟合单次偶发 / 证据薄弱）
  → passed=false

只输出 JSON：{"passed": true|false, "reason": "<一句话说明>"}

[User]
条目类型：{{candidate.kind}}
条目锚点（sub_task / task）：{{candidate.anchor}}
条目内容：{{candidate.payload}}

证据 traces（最近 {{N}} 条）：
{{traces_concatenated}}
```

### 13.3 核心语义

- **fb/tp 的 is_hard=true 走在线升格路径**（§6.2 chunk flush 即时 promote，不经 §13 离线），硬约束零延迟生效
- **其他条目**（fb/tp 非 hard + 所有 tse）等离线 worker 按 delta ≥ 3 触发 + 证据检查
- **candidate 永远保留**——status='promoted' 不阻止后续 chunk flush 更新它；mention_count 继续累积
- **active 快照 1-to-1**：`last_promoted_record_id` 指向同一个 mem_record.id，多次升格是 UPDATE
- **candidate 是主文件，active 是派生快照**：agent 召回读 active，离线 worker 让 active 周期性追上 candidate
- **合理性检查失败**：不报错、不降级，candidate 原样保留；下次 delta 再满 3 时重试

### 13.4 idempotent / 失败处理

- `last_promoted_at_mention` 充当幂等键——同一 mention_count 下重复触发 `check_and_promote()` 结果一致（要么再次成功升格 upsert 同一 record_id；要么再次失败，candidate 原样）
- 合理性检查 LLM 调用失败 → 本次跳过，记 log，下次重试；不影响其他 candidate
- Step A 扫描与 Step B 检查用不同事务：Step A 只读，Step B 每条 candidate 一个短事务

### 13.5 升格 upsert 动作

沿用 §8.7 `promote()` 伪代码——检查 `last_promoted_record_id`：
- NULL → INSERT 新 mem_record + 扩展表；回填 `last_promoted_record_id`
- 非 NULL → UPDATE 现有 mem_record.content + 扩展表内容

**不调 LLM**（除 §13.2 合理性检查外）——bullets 列表做纯规则整理：按 candidate 原顺序编号、trim、去重相同文本；fb/tp 的 description 直接复制。

### 13.6 当前 v6 实现状态

在线 §6.1（Fb/Tp）和 §7（TSE）只写候选池。**本节离线 worker 尚未实现，升格路径暂未开通**——agent 通过 `memory_search` 召回 active 表会返回空集；等 worker 实装后三类一起接入。

**降级路径**（长期 pending → discarded）**v6 暂不做**，候选池会无限增长；后续迭代再引入清理策略。

---

## 14. 后续迭代预留

- **Agent 召回侧**：v6 只写到 `mem_record`，召回由 agent 侧实现；未来若统一到本模块，需补召回 query 构造 / 排序 / 硬约束注入策略
- **候选池降级路径**：v6 暂不做 discarded；长期 pending 无新证据 → 清理的细则待后续定（§13 只说不做）
- **Task Trace 离线挖掘**：积累 N 条同类任务的 Trace 后，跨任务模式挖掘（workflow / skill 模板），§13 worker 可扩展成处理这类任务
- **Dreaming 跨 session 归纳**：候选池 mention_count 天然积累跨 session 信号；可叠加 sub_task 语义聚类（向量近邻归一"写单测" vs "写单元测试"）
- **TSE 主动应用策略**：从召回结果外推到新任务（如根据当前 task name 预加载相关 TSE bullets 到 agent context）
- **sub-task 命名稳定化**：跨 session 同子事务字面漂移时依赖向量召回兜底；误分区高发时可加"sub_task 近邻归一"（embedding 聚类 or 小模型短语规范化）
- **并发 / 失败恢复**：在线 chunk flush 的 A/B/C 三步事务边界、水位推进时机、worker 与在线 flush 并发（当前 v6 未规定，属 Gap）
