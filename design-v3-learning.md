# Spec 003 — 程序记忆学习模块（迭代一）

> 最后更新：2026-04-13（§6.2 对齐 MemOS 原始 prompt；Step A-3 安全验证层）

---

## 0. 概述

### 目标

从 agent 真实工作历史中**自动提取并演进三种程序记忆**：

| 类型 | 存储内容 | 核心价值 |
|------|---------|---------|
| `workflow` | 可复用的任务执行流程模板（步骤、工具、参数配置）| 标准化重复性任务，下次直接套用 |
| `experience` | agent 执行经验（工具使用技巧、自动纠错记忆、用户纠错反馈）| 避免重蹈覆辙，提升执行质量 |
| `tool_preference` | 工具选择偏好（做某类任务优先用哪个工具）| 提升执行效率 |

### 迭代一范围

- 学习与演进合并在同一个模块，在 session_end / compact 时批处理
- 不依赖实时 Cursor / 主题漂移检测
- Dreaming 跨 session 归纳 → 迭代二

---

## 1. 整体架构

```
触发：session_end / compact
输入：N 轮完整对话消息

─────────────────────────────────────────────────────────────────────
Step 1  预过滤（零 LLM）
          第一层：全量内容清洗（哨兵/注入/敏感脱敏/推理标签）
          第二层：窗口级 tool 输出截断（60/30 规则）
          ← MemOS captureMessages + Generator 预处理
          ← AutoSkill captureMessages（哨兵消息识别）

Step 2  生成滑动窗口（n=10, m=2, step=8）
          ← MemOS Python 端滑动窗口参数

Step 3  逐窗口处理
          扫描 context_assemble → used_skill_ids
          ├─ 非空 → §6 升级路线（反馈纠正驱动）
          │          A轨：MemOS 反馈修正闭环（FEEDBACK_JUDGEMENT_PROMPT）
          │          B轨：规则层偏差分析（零 LLM）
          │          EvolveSkill 改写  ← MemOS Upgrader
          │
          └─ 空   → 进入主题拼接积累

Step 4  主题拼接 → Chunk flush
          Jaccard 关键词相似度判断主题边界
          上限：3 窗口 / 2h 强制 flush
          ← 自研（替代 MemOS 实时 Cursor 机制）

Step 5  Chunk 处理
          混合搜索（向量 + 关键词 + 名称）
          LLM Judge 语义匹配
          ← MemOS SkillEvolver search → judge 路径
          │
          ├─ 未命中 → §8.3 创建新 skill
          │            LearnSkill Prompt（提取规则 ← AutoSkill §1-§5）
          │                            （写作原则 ← MemOS STEP1_SKILL_MD_PROMPT）
          │            并行：Scripts / References / Evals  ← MemOS Generator
          │            Evals 向量召回验证（零 LLM）        ← MemOS Generator
          │            Dedup（三阶 add/merge/discard）     ← AutoSkill 维护管线
          │
          └─ 命中   → §8.4 内容合并
                       规则层差异分析（零 LLM）
                       EnrichSkill 合并               ← AutoSkill Skill Merger

Step 6  E+TP 候选池
          LearnExperience Prompt
            分析框架（成功/失败/纠正）  ← XSkill INTRA_SAMPLE_CRITIQUE
            格式："When X, do Y"        ← XSkill Experience 定义
          候选池打分 + 动态阈值          ← AutoSkill Requirement Memory
          Dedup（三阶 add/merge/discard）← AutoSkill 维护管线
─────────────────────────────────────────────────────────────────────
```

---

## 2. 参数

| 参数 | 值 | 说明 |
|------|----|------|
| `n` | 10 轮 | 窗口大小（1 轮 = user + assistant 一对） |
| `m` | 2 轮 | 相邻窗口重叠轮次 |
| `step` | 8 轮 | 每次滑动步长（n - m） |
| `max_windows_per_chunk` | 3 | 主题拼接上限（强制 flush） |
| `max_gap_hours` | 2 | 时间间隔上限（强制 flush） |
| `jaccard_threshold` | 0.3 | 主题相似度判断阈值 |
| `candidate_match_threshold` | 0.70 | 候选池向量匹配阈值 |

窗口分布（N=30 轮示例）：
```
W0: [0,  10)
W1: [8,  18)
W2: [16, 26)
W3: [24, 30)  ← 最后一个对齐到 N
```

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

```
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

## 5. Step 3 — 逐窗口处理

### 5.1 扫描是否用过 Workflow（零 LLM）

```
遍历窗口消息，找 tool_result where name="context_assemble"
  → 解析 content 中的 skill_id 列表
  → 去重 → used_skill_ids[]
```

### 5.2 分流

```
用了 Workflow (used_skill_ids 非空)
  → 升级路线（§6）
  → 窗口消耗完毕，不进入主题拼接

没用 Workflow
  → 主题拼接路线（§7）
```

---

## 6. 升级路线（Workflow Evolve）

窗口内用过已有 workflow → 评估是否需要改写。

### 6.1 串行处理总览

```
for skill_id in used_skill_ids:

  ① EvaluateSkill（两轨并行，小模型）
     轨 A：反馈修正检测（MemOS 反馈修正闭环，§6.2）
     轨 B：规则层分析（零 LLM，§6.3）
     合并两轨结果 → signal

  ② signal 为空 → 记录 EVALUATED(no_signal)，跳过

  ③ signal 非空 → EvolveSkill（大模型，§6.4）
     输入：SKILL.md + signal（不传原始对话）
     输出：{ action: rewrite|no_change, upgrade_type, content, applied, ignored }

  ④ 规则验证（§6.5）
     通过 → 升级前备份 → SkillUpdateInternal → 记录 EVOLVED
     不通过 → 恢复备份，记录 EVOLVE_FAILED
```

---

### 6.2 轨 A — 反馈修正检测（MemOS 反馈修正闭环）

逐条扫描窗口内 user 消息，识别用户对 agent 行为的纠正态度，提取纠正内容后与 skill 步骤对应。

**来源**：`memos/templates/mem_feedback_prompts.py` — `FEEDBACK_JUDGEMENT_PROMPT` + `UPDATE_FORMER_MEMORIES` + `OPERATION_UPDATE_JUDGEMENT`

---

#### Step A-1：反馈有效性判断（小模型）

对窗口内**每条 user 消息**单独调用（以该消息为 `user_feedback`，其前 3 轮作为 `chat_history`）。

基于 MemOS `FEEDBACK_JUDGEMENT_PROMPT_ZH`，在开头加 scope 限定，只关注 workflow 过程纠正：

```
【范围限定】本分析仅关注用户对 agent 操作流程的纠正，包括：
工具选择（"应该用X不用Y"）、执行步骤（"顺序不对"）、操作方式（"不要这样做"）。
与 agent 操作流程无关的纠正（个人喜好、世界事实、内容本身）视为 irrelevant。

您是一个回答质量分析专家。请严格按照以下步骤和标准分析提供的"用户与助手聊天历史"和"用户反馈"，并将最终评估结果填入指定的JSON格式中。

分析步骤和标准：
1. *有效性判断*：(validity字段)
   - 有效（true）：用户反馈的内容与聊天历史中的主题、任务或助手的最后回复相关。例如：提出后续问题、进行纠正、提供补充或评估最后回复。
   - 无效（false）：用户反馈与对话历史完全无关，与之前内容没有任何语义、主题或词汇联系。

2. *用户态度判断*：(user_attitude字段)
   - 不满意：反馈显示负面情绪，如直接指出错误、表达困惑、抱怨、批评，或明确表示问题未解决。
   - 满意：反馈显示正面情绪，如表达感谢或给予赞扬。
   - 无关：反馈内容与评估助手回答无关。

3. *摘要信息生成*（corrected_info字段）：
   - 从用户反馈中总结核心信息，生成简洁的事实陈述列表。
   - 当反馈提供纠正时，仅关注纠正后的信息。
   - 当反馈提供补充时，整合所有有效信息（包括旧信息和新信息）。
   - 非常重要：保留相关时间信息，并以具体、明确的日期或时间段表达。
   - 对于"满意"态度，此列表可能包含确认性陈述，如果没有提供新事实则为空。
   - 专注于客观事实陈述。

输出格式：
[
    {
        "validity": "<字符串，'true' 或 'false'>",
        "user_attitude": "<字符串，'dissatisfied' 或 'satisfied' 或 'irrelevant'>",
        "corrected_info": "<字符串，用中文书写的事实信息记录>",
        "key": "<字符串，简洁的中文记忆标题，用于快速识别该条目的核心内容（2-5个汉字）>",
        "tags": "<列表，中文关键词列表（每个标签1-3个汉字），用于分类和检索>"
    }
]

示例1：
对话历史：
用户：这些天我不能吃辣。能给我推荐一些合适的餐厅吗？
助手：好的，我推荐您附近的鱼类餐厅。他们的招牌菜包括各种蒸海鲜和海鱼生鱼片。
反馈时间：2023-1-18T14:25:00.856481

用户反馈：
哦，不！我对海鲜过敏！而且我不喜欢吃生鱼。

输出：
[
    {
        "validity": "true",
        "user_attitude": "dissatisfied",
        "corrected_info": "用户对海鲜过敏且不喜欢吃生鱼",
        "key": "饮食限制",
        "tags": ["过敏", "海鲜", "生鱼", "饮食偏好"]
    }
]

示例2：
对话历史：
用户：我2025年11月25日买了什么？
助手：一件红色外套
反馈时间：2025-11-28T20:45:00.875249

用户反馈：
不对，我还买了一件蓝色衬衫。

输出：
[
    {
        "validity": "true",
        "user_attitude": "dissatisfied",
        "corrected_info": "用户于2025年11月25日购买了一件红色外套和一件蓝色衬衫",
        "key": "购物记录",
        "tags": ["红色外套", "蓝色衬衫", "服装购物"]
    }
]

示例3：
对话历史：
用户：我最喜欢的食物是什么？
助手：披萨和寿司
反馈时间：2024-07-15T10:30:00.000000

用户反馈：
错了！我讨厌寿司。我喜欢汉堡。

输出：
[
    {
        "validity": "true",
        "user_attitude": "dissatisfied",
        "corrected_info": "用户喜欢披萨和汉堡，但讨厌寿司",
        "key": "食物偏好",
        "tags": ["偏好", "披萨和汉堡"]
    }
]

对话历史：
{chat_history}

反馈时间：{feedback_time}

用户反馈：
{user_feedback}

输出：
```

过滤：`validity=false` 或 `user_attitude != dissatisfied` 的结果直接丢弃。

---

#### Step A-2：纠正内容与 Skill 步骤对应（小模型）

将 Step A-1 提取的 `corrected_info` 与 `skill_steps` 逐一比对（借鉴 `UPDATE_FORMER_MEMORIES` 的操作结构，适配为 skill 步骤场景）：

```
[System]
你是一个 skill 步骤分析器。判断用户的纠正信息与已有 skill 步骤的关系。

对每条纠正信息，决定操作类型：
- UPDATE：纠正命中了某个具体步骤（该步骤有错误/过时/应该换工具）
          → step_index 指向具体步骤（从 1 开始）
- ADD：纠正涉及 skill 没有覆盖的场景（加入 Notes 或新步骤）
- NONE：纠正与该 skill 领域无关

重要：仅修改被纠正的步骤，其余步骤完全保持不变。

只输出 JSON：
{
  "operations": [
    {
      "corrected_info": "...",
      "operation": "UPDATE|ADD|NONE",
      "step_index": N,
      "reason": "..."
    }
  ]
}

[User]
已有 skill 步骤：
{skill_steps}

用户纠正列表：
{corrections}
```

---

#### Step A-3：安全验证（零 LLM — 借鉴 OPERATION_UPDATE_JUDGEMENT）

对 Step A-2 返回的每条 UPDATE 操作，做规则层安全检查，防止错误覆盖：

```
for each op in operations where op.operation == "UPDATE":

  ① 实体一致性检查
     corrected_info 与 skill_steps[step_index] 描述同一操作领域？
     → 否 → judgement = INVALID（跳过该 op）

  ② 语义相关性检查
     corrected_info 是否直接纠正该步骤的某个具体做法？
     → 否（完全不相关） → judgement = NONE（跳过该 op）

  ③ 通过 → judgement = UPDATE_APPROVED
```

规则实现：用 corrected_info 的关键词与 skill_steps[step_index] 文本做 token 重叠计算，重叠 < 0.15 视为实体不一致（INVALID）；重叠 0.15–0.30 视为弱相关，由 LLM 在 Step A-2 的 reason 字段辅助判断。

只有 `UPDATE_APPROVED` 的 op 才进入最终 signal。

---

### 6.3 轨 B — 规则层分析（零 LLM）

从执行轨迹中提取结构化偏差，不做任何语义判断。

```
输入：
  skill_steps   ← 解析 SKILL.md 的 ## 执行步骤，提取关键词
  tool_uses     ← 窗口内所有工具调用 [{index, name, args_summary}]
  errors        ← exitCode != 0 的 tool_result [{index, tool, error_first_line}]

提取：
  deviations    ← agent 使用了与 skill 步骤关键词不匹配的工具/命令
                  [{step, expected_keyword, actual_tool, outcome: success|fail}]

  errors_on_steps ← 报错的 tool_use 与 skill 步骤关键词匹配
                    [{step, error_summary}]
```

---

### 6.4 EvolveSkill Prompt（大模型）

合并两轨信号后，仅当有有效 signal 时调用：

```
[System]
你是一个 workflow 改写编辑器。根据以下使用信号改写 SKILL.md。

改写规则：
- corrections(UPDATE)：用户明确纠正了某个步骤 → 必须采纳，替换该步骤
- corrections(ADD)：用户纠正涉及 skill 未覆盖场景 → 加入 Notes 或新步骤
  （判断是否足够通用，一次性操作不加）
- deviations(outcome=success)：agent 用了不同方法且成功
  → 判断是否比 skill 更好；更好则替换，旧方法可保留为 Alternative
- deviations(outcome=fail)：agent 偏离且失败 → 忽略
- errors_on_steps：步骤有 bug → 修复步骤，或在 Notes 加警告

upgrade_type 判断（写入输出）：
  corrections(UPDATE) 非空    → "fix"
  deviations(success) 非空    → "refine"
  corrections(ADD) 非空       → "extend"
  仅 errors_on_steps          → "fix"

保持 SKILL.md 格式不变（frontmatter + 触发条件 + 执行步骤 + 注意事项）。
版本注释加在 frontmatter 后第一行：
  <!-- v{{NEW_VERSION}}: {{一行变更说明}} (session: {{SESSION_ID}}) -->

只输出 JSON：
{
  "action": "rewrite|no_change",
  "upgrade_type": "fix|refine|extend",
  "content": "完整新 SKILL.md",
  "applied": ["采纳了什么及原因"],
  "ignored": ["忽略了什么及原因"]
}

[User]
已有 SKILL.md（v{{VERSION}}）：
{{skill_content}}

使用信号：
  用户纠正（corrections）：{{corrections}}
  步骤偏差（deviations）：{{deviations}}
  报错步骤（errors_on_steps）：{{errors_on_steps}}
```

---

### 6.5 规则验证（持久化前）

| 检查项 | 规则 | 不通过 |
|--------|------|--------|
| frontmatter 完整 | 有 name / description / context，非空 | 恢复备份 |
| sections 存在 | 含 ## 触发条件 和 ## 执行步骤 | 恢复备份 |
| steps 非空 | 执行步骤 ≥ 1 条 | 恢复备份 |
| 内容长度 | 100 ~ 5000 字符 | 恢复备份 |
| name 变化幅度 | 编辑距离 ≤ 50%（防改面目全非） | 恢复备份 |
| 内容缩水 | 新内容 < 旧内容 70% | warning，仍写入 |

验证前先备份原 skill 目录（借鉴 MemOS），验证失败则从备份恢复。

---

### 6.6 证据来源原则（USER-only，借鉴 AutoSkill）

- **USER 消息** = 唯一有效纠正证据
- **ASSISTANT 回复** = 仅用于确认执行边界，不作为 signal 来源
- `satisfied` / `irrelevant` 消息不产生任何 signal
- 弱应答（"好的"/"继续"/"ok"）= `irrelevant`，直接过滤

---

## 7. 主题拼接路线（Workflow Learn）

没用已有 workflow 的窗口，进入主题拼接积累，等待提取新 workflow。

### 7.1 主题关键词提取（零 LLM）

```
取窗口内所有 user 消息
  → 分词（中文分词 or 空格分割）
  → 过滤停用词（好的/继续/ok/是的/不对/帮我/可以...）
  → 过滤短词（< 2 字符）
  → top-N TF 词 → topic_keywords
```

### 7.2 相邻窗口 Jaccard 比较

```
overlap = |keywords_A ∩ keywords_B| / |keywords_A ∪ keywords_B|
overlap > 0.3 → 同主题，合并
overlap ≤ 0.3 → 不同主题，不合并
```

### 7.3 Chunk 积累与 Flush

```
chunk = [W_current]

for 下一个窗口 W_next:

  gap = W_next.first_turn_time - W_current.last_turn_time

  if gap > 2h:
    flush(chunk, forced=True)   ← 时间上限
    chunk = [W_next]

  elif len(chunk) >= 3:
    flush(chunk, forced=True)   ← 窗口数上限
    chunk = [W_next]

  elif Jaccard(chunk.keywords, W_next.keywords) > 0.3:
    chunk.merge(W_next)         ← 同主题，继续积累
    （合并时 overlap 的 m 轮去重，只保留一次）

  else:
    flush(chunk, forced=False)  ← 主题切断
    chunk = [W_next]

session 结束 → flush(chunk, forced=False)  ← 保底
```

**强制 flush（forced=True）跳过质量过滤，必须提取。**

### 7.4 窗口质量过滤（普通 flush）

满足以下任一条件才值得提取，否则 skip：
- 有工具调用（任意 tool_use）
- assistant 回复总长 ≥ 200 字符

---

## 8. Step 5 — Chunk 提取新 Workflow（MemOS 路径）

对每个 flush 出的 Chunk，走 search → judge → create/upgrade。

### 8.1 混合搜索已有 Workflow

```
得分 = 0.70 × 向量余弦相似度
     + 0.18 × 关键词重叠（signal_overlap）
     + 0.12 × 名称 token Jaccard（name_similarity）

取 top-5，score < 0.35 的过滤掉
```

### 8.2 LLM Judge（小模型）

```
输入：
  chunk_summary ← Chunk 内 user 消息拼接（前 500 字符）
  top-5 skill 的 name + description + context

输出：
  { "selectedIndex": 0-5, "reason": "..." }
  0 = 无匹配 → 创建路线
  >0 = 有匹配 → 升级路线
```

**Judge Prompt**：

```
你是一个严格的匹配判断器。判断当前工作窗口是否应该归入已有的某个 workflow。

必须是同一类型的工作流程才算匹配，宽泛或切线相关不算。

当前窗口摘要：{{chunkSummary}}

候选 workflow（序号、name、description）：
{{skillList}}

规则：
- 仅当任务明确属于某个 workflow 的领域时，输出该序号（1~N）
- 不确定时**倾向于匹配**（输出最接近的序号），宁可走升级路线也不要重复创建

只输出 JSON：{"selectedIndex": 0, "reason": "..."}
```

> **设计说明**：Judge 故意偏宽松。误判为升级的代价（合并进已有 Skill）远低于误判为新建的代价（白跑 LearnSkill 大模型调用）。8.5 Dedup 作为保底，负责处理漏网情况。

### 8.3 创建路线（selectedIndex = 0）

#### 8.3.1 LearnSkill（大模型）

输入：
```
Primary User Questions ← Chunk 内所有 user 消息（主要证据）
Full Conversation      ← 完整过滤后的 Chunk 消息（上下文参考）
context_pool           ← 已有 context 标签列表
```

**LearnSkill Prompt**（融合 AutoSkill 提取规则 + MemOS 写作原则）：

```
你是一个 workflow 提取专家。将一段真实工作历史提炼为可复用的 SKILL.md。

## 提取规则（借鉴 AutoSkill）

### 1) 证据来源
- 以 Primary User Questions（用户消息）为主要提取证据
- Full Conversation 仅作上下文参考，不作为提取依据
- assistant 回复中未经用户确认/纠正的内容不提取
- "好的"/"继续"/"ok" 等弱应答不视为用户认可

### 2) 何时提取
- 提取：HOW（执行过程、工具选择、步骤约束）
- 不提取：WHAT（内容本身、结果数据、一次性任务）
- 不提取：泛化价值低、仅此一次的操作
- 判断标准：该流程能否被同一用户在未来类似任务中复用？

### 3) 去标识化（CRITICAL）
- 移除案例特定实体（项目名/文件名/URL/具体数值）
- 用占位符替代：<项目名>、<目标路径>、<配置项>
- 保留真实命令和配置结构（已验证可用）
- 去标识后核心价值消失 → 不提取

### 4) 不发明内容
- 只提取对话中有证据支持的步骤
- 不补充"通常还需要"的额外步骤
- 不发明用户未提及的约束或规范

## 写作原则（借鉴 MemOS）

### description 作为触发机制
description 决定 agent 是否激活该 skill，要主动描述：
- 列举适用场景、触发关键词、用户可能的表达方式
- 不只说"做什么"，要说"什么情况下用"
- 示例：
  - 差："如何用 Docker 部署 Node.js"
  - 好："将 Node.js 应用容器化并部署。当用户提到 Dockerfile、容器构建、
    端口映射、多阶段构建、镜像优化，或描述'把应用打包到生产环境'时使用"

### 写作风格
- 祈使句（"运行 X"，不是"需要运行 X"）
- 每步说明 WHY，不只说 HOW
- 从具体任务中泛化，不照抄案例细节
- 保留真实命令/配置——来自真实执行，已验证可用

## 输出格式

---
name: <动词+宾语，4-10字>
description: <60-100字，主动描述触发场景和适用情况>
kind: workflow
context: [<标签1>, <标签2>, ...]  # 从 context_pool 选择，2-5个
---

## 触发条件
（2-4 条，何时应该用此 workflow）

## 执行步骤
（编号步骤，每步说明 WHY）

## 注意事项
（❌ 错误做法 → 原因 → ✅ 正确做法）

[User]
Primary User Questions:
{primary_user_questions}

Full Conversation:
{full_conversation}

已有 context 标签池：
{context_pool}
```

无法提取（全 WHAT / 泛化价值为零）→ 输出 `{"skip": true, "reason": "..."}`

---

#### 8.3.2 并行依赖文件生成（借鉴 MemOS Generator）

LearnSkill 完成后，`Promise.all` 并行生成三类附属文件：

**Scripts**（从对话中提取可复用脚本）：
```
基于以下 SKILL.md 和对话记录，提取可复用的自动化脚本。

规则：
- 只提取对话中实际出现的完整可运行命令/脚本
- 每个脚本必须自包含、可直接运行
- 不捏造脚本——只提取实际用过的
- 脚本应补充 SKILL.md，不重复其内容
- 无可提取内容 → 返回 []

输出 JSON 数组：
[{"filename": "xxx.sh", "content": "..."}]
```

**References**（保留值得存档的技术参考）：
```
基于以下 SKILL.md 和对话记录，提取值得保留的参考文档。

规则：
- 只提取对话中涉及的重要 API 文档、配置参考、技术说明
- 每份参考应为独立的 Markdown 文档
- 不重复 SKILL.md 中已有内容
- 语言与 SKILL.md 一致
- 无可提取内容 → 返回 []

输出 JSON 数组：
[{"filename": "xxx.md", "content": "..."}]
```

**Evals**（生成测试触发 prompt）：
```
基于以下 skill，生成能触发该 skill 的真实测试 prompt。

要求：
- 生成 3-4 个真实用户会输入的 prompt
- 混合直接和间接表达方式
- 包含真实细节（路径、项目名、报错信息等）
- 混合正式/口语风格，可含拼写简写
- 每个 prompt 复杂度要足够需要该 skill
- 语言与 skill 内容一致

输出 JSON 数组：
[{
  "id": 1,
  "prompt": "...",
  "expectations": ["期望行为1", "期望行为2"],
  "trigger_confidence": "high|medium"
}]
```

#### 8.3.3 Evals 验证（零 LLM）

用向量搜索对每条 eval prompt 检索，验证该 skill 能否被召回：

```
for each eval in evals:
  hits = vector_search(eval.prompt, top_k=5)
  if skill.id in hits AND hits[0].score >= 0.4:
    eval.verified = true
  else:
    eval.verified = false

verified_ratio = verified / total
verified_ratio < 0.5 → skill.status = "draft"，入库但不加入检索索引
                        （description 召回率不足，待演进模块 Step 2 优化后升为 active）
verified_ratio >= 0.5 → 进入规则验证
```

---

#### 8.3.4 规则验证后入库

frontmatter / sections / steps / 长度 / 去标识化检查（同 §6.5）。

持久化结构：
```
skills/{skill-name}/
  ├── SKILL.md
  ├── scripts/
  ├── references/
  └── evals/evals.json
```

### 8.4 内容合并路线（selectedIndex > 0 — agent 未使用已有 skill）

Judge 命中了已有 skill，但 agent 在本窗口内没有调用它（未被召回或未激活）。
核心问题：**Chunk 的执行记录能否丰富已有 skill？**

与 §6 升级路线的区别：§6 是用户纠正驱动，§8.4 是新执行经验驱动。

#### 8.4.1 规则层差异分析（零 LLM）

```
输入：
  existing_skill  ← 已有 SKILL.md（name + steps + 触发条件）
  tool_uses       ← Chunk 内所有工具调用
  user_messages   ← Chunk 内所有 user 消息

分析：
  ① 场景覆盖对比
     user_messages 的主题关键词 vs skill 触发条件关键词
     → 有 skill 未覆盖的新触发场景？

  ② 步骤偏差分析
     tool_uses 与 skill_steps 关键词比对
     → agent 用了 skill 没提到的工具/做法，且执行成功？

  ③ 召回缺口判断
     Chunk 主题与 skill description 的 token 重叠
     → 重叠 < 0.2 → 很可能是 description 不够好导致没被召回
```

**无增量信号**（新场景=0、成功偏差=0）：
- 若召回缺口存在 → 仅标记 `needs_description_improve=true`，进入 §8.4.3
- 否则 → 记录 `ENRICHED(no_signal)`，跳过

**有增量信号** → 进入 §8.4.2

#### 8.4.2 EnrichSkill（大模型）

```
[System]
你是一个 workflow 内容合并专家。
将新执行记录中的增量信息合并到已有 SKILL.md，使其更完整。

合并原则（借鉴 AutoSkill Skill Merger）：
- 语义并集：不是拼接，是去重融合
- 不能丢失已有约束（防退化）
- 去标识化：移除案例特定实体，保留可移植规则
- Anti-Duplication：近似表达保留一个规范版本
- 不发明内容：只用两边原有信息

新场景处理：
- 新触发场景 → 补充到 ## 触发条件
- agent 用了不同做法且成功 → 补充到 ## 执行步骤（作为 Alternative）
  或补充到 ## 注意事项（作为替代方案说明）
- 有新 pitfall → 补充到 ## 注意事项

name 处理：保留已有 name，除非新内容明显扩大了 skill 适用范围

只输出 JSON：
{
  "action": "enrich|no_change",
  "content": "完整新 SKILL.md",
  "added": ["新增了什么及原因"],
  "skipped": ["忽略了什么及原因"]
}

[User]
已有 SKILL.md：
{existing_skill}

Chunk 执行摘要：
  新触发场景：{new_scenarios}
  成功替代做法：{successful_deviations}
```

**写入前先备份原 skill 目录** → 规则验证（同 §6.5）→ 通过则写入，失败则从备份恢复。

#### 8.4.3 Description 改善（小模型，可选）

仅当 `needs_description_improve=true` 时触发：

```
[System]
当前 skill 的 description 可能导致相关任务未能召回该 skill。
请改写 description，使其更主动地覆盖触发场景。

要求：
- 列举适用场景、关键词、用户可能的表达方式
- 60-100字，不改变 skill 的核心能力定义
- 只输出新 description 字符串

[User]
已有 description：{existing_description}
未能匹配的 Chunk 摘要：{chunk_summary}
```

备份原 skill 目录 → 仅替换 frontmatter 中的 description 字段 → 若写入失败则从备份恢复。

---

### 8.5 创建后 Dedup（借鉴 AutoSkill 维护管线）

LearnSkill 生成新 SKILL.md 后，写入前做去重，防止库中出现重复 skill。

#### 阶段一：快速相似度门控（零 LLM）

```
combined_score =
  0.70 × 向量余弦相似度
+ 0.18 × 关键词重叠（signal_overlap）
+ 0.12 × 名称 token Jaccard（name_similarity）

score < 0.50  → 直接写入（无重复）
score ≥ 0.90  → 进入合并流程（高置信重复）
0.50–0.90     → 进入 LLM 决策
```

identity 快速检查：`normalize(new.description) == normalize(existing.description)` → 直接合并，跳过 LLM。

#### 阶段二：能力身份判断（小模型）

```
[System]
你是 skill 能力身份判断器。
判断候选 skill 与已有 skill 是否是同一项能力。

- same_capability=true：核心任务相同，主要是细节迭代或场景扩展
- same_capability=false：适用场景、输出类型或核心约束有本质差异
- 不确定时默认 false

只输出 JSON：
{"same_capability": true|false, "confidence": 0.0-1.0, "reason": "..."}

[User]
已有 skill：{existing_skill_summary}
候选 skill：{new_skill_summary}
```

`same_capability=false` → 直接写入为新 skill。

#### 阶段三：合并执行（大模型）

`same_capability=true` 时，将两个 SKILL.md 语义合并：

```
[System]
你是 workflow skill 合并专家。
将已有 skill 与候选 skill 合并为一个更完整的 SKILL.md。

合并原则：
- 语义并集：不是拼接，是去重融合
- 不能丢失已有 skill 的约束（防退化）
- 去标识化：移除案例特定实体
- Anti-Duplication：近似步骤保留一个规范表达
- name：保留已有，除非候选 name 明显更好
- description：取两者并集语义，重写为主动触发式描述
- 触发条件：语义并集后去重
- 执行步骤：去重融合，保留两边均有的核心步骤
- 注意事项：语义并集后去重

版本注释：<!-- v{N}: merge from session {SESSION_ID} -->

只输出完整新 SKILL.md。

[User]
已有 skill（历史）：
{existing_skill}

候选 skill（本次生成）：
{new_skill}
```

降级路径：合并 LLM 失败 → 取两边 SKILL.md 中步骤数更多的一份，description 取较长者，context 标签取并集去重，版本 +1。

---

## 9. Step 6 — Experience + Tool Preference 候选池

E+TP 不直接入库，先放候选池打分，积累足够证据后才升格。

### 9.1 提取（每个 Chunk flush 时同时触发）

一次大模型调用，同时提取 experience 和 tool_preference。

**分层输入结构**（同 AutoSkill）：
```
Primary User Questions ← Chunk 内所有 user 消息（主要证据）
Full Conversation      ← 完整过滤后消息（上下文参考）
```

---

**LearnExperience Prompt**（借鉴 XSkill INTRA_SAMPLE_CRITIQUE 的分析框架）：

```
你是一个执行经验提取专家。从以下对话记录中提取可复用的 experience 和 tool_preference。

## 知识类型定义

**experience**（执行经验）— agent 在执行中积累的情境行动建议：
格式："遇到 [触发情境] 时，[推荐做法]"
三类：
- 工具使用技巧：工具参数选择、操作顺序、配合方式的实操建议
  （含"用 A 之后配合 B 效果更好"等互补组合模式）
- 纠错记忆：失败后找到的正确路径（出现该报错时，改用此做法）
- 用户反馈：用户明确纠正 agent 做法形成的行为规则（is_hard=true）
长度：description ≤ 50 字，简洁可操作

**tool_preference**（工具偏好）— 工具选择偏好，优化执行效率：
格式："做 [任务类型] 时，优先用 [工具]"
目标：记录 agent 在同类任务上的稳定工具选择，减少决策开销
示例："处理数据时优先用 Python"、"代码搜索优先用 ripgrep"
注意：互补工具组合（A 配合 B 使用）→ 归为 experience，不是 tool_preference

## 分析框架（适配单 session，借鉴 XSkill Critique）

**1. 成功路径分析**
- agent 哪些做法有效？关键决策是什么？
- 是否用了特定工具/参数/顺序且效果好？→ 工具使用技巧 or tool_preference

**2. 失败→恢复分析**
- agent 报错后如何换方案？成功的恢复方法是什么？
- → 纠错记忆（出现该错误时，改用此做法）

**3. 用户纠正分析**（以 Primary User Questions 为主）
- 用户明确纠正了 agent 的哪些做法？
- "不要用 X"/"应该用 Y"/"顺序不对" → 用户反馈（is_hard=true）
- 工具选择被纠正 → 同时考虑产出 tool_preference（is_hard=true）

**4. 工具选择模式**
- 同类任务是否反复选择同一工具（≥ 3次）？→ tool_preference
- 用户是否纠正了工具选择？→ tool_preference（is_hard=true）

## 提取规则

- 只提取有对话证据支持的内容（不发明）
- 去标识化：用占位符替代具体文件名/项目名
- 一次性的、不可泛化的操作 → 不提取
- 纯闲聊/无操作内容 → 不提取

## 输出格式

只输出 JSON，结构如下：
  { "items": [
      { "kind": "experience" | "tool_preference",
        "name": "<简短标题，5字以内>",
        "context": "<适用场景标签，1-3个>",
        "description": "<experience: '遇到X时，做Y' | tool_preference: '做X时优先用Y'，≤50字>",
        "is_hard": true | false }
  ] }

is_hard 判定：
- true：用户明确纠正（"别用X"/"不对，应该Y"）
- false：agent 自身推断（报错修复 / 行为模式 / 工具选择规律）

无可提取内容 → {"items": []}

[User]
Primary User Questions（主要证据）：
{primary_user_questions}

Full Conversation（上下文参考）：
{full_conversation}
```

---

**is_hard 规则辅助验证**（零 LLM，提取后执行）：

LLM 输出 `is_hard=true` 时，验证 Primary User Questions 中是否存在否定词+操作词的模式：
```
否定词：不要/别/不能/不该/不应该/错了/换成
操作词：工具名/命令名/动词短语
匹配 → 确认 is_hard=true
不匹配 → 降级为 is_hard=false
```

### 9.2 候选池数据结构

```sql
CREATE TABLE memory_candidates (
  id              TEXT PRIMARY KEY,
  kind            TEXT NOT NULL,      -- experience | tool_preference
  name            TEXT NOT NULL,
  context         TEXT,
  description     TEXT NOT NULL,
  embed_text      TEXT NOT NULL,

  -- 打分字段
  mention_count   INTEGER DEFAULT 0,  -- 累计出现次数（跨窗口/session）
  recent_count    INTEGER DEFAULT 0,  -- 近 3 个 session 出现次数
  hard_count      INTEGER DEFAULT 0,  -- 作为 hard 约束出现次数
  total_updates   INTEGER DEFAULT 0,  -- 总更新次数（决定成熟度）
  first_seen_at   INTEGER NOT NULL,
  last_seen_at    INTEGER NOT NULL,

  support_score   REAL DEFAULT 0.0,
  status          TEXT DEFAULT 'pending'  -- pending | promoted | discarded
);
```

### 9.3 打分公式

```
mention_ratio = mention_count / total_updates
recent_ratio  = recent_count / 3
recency_score = exp(-(now - last_seen_at) / 7days)
hard_ratio    = hard_count / mention_count

support_score =
  0.45 × mention_ratio   （跨次出现率，权重最高）
+ 0.25 × recent_ratio    （近期活跃度）
+ 0.10 × recency_score   （时间衰减）
+ 0.20 × hard_ratio      （显式纠正比例）
```

**核心效果**：
- 用户显式纠正（hard=true）→ hard_ratio=1.0 → score ≥ 0.20 → 第一次即可通过早期阈值
- Agent 推断的 soft 规律 → 需要跨多次 session 积累 mention_ratio → 反复验证后才升格

### 9.4 动态升格阈值

| 成熟度 | total_updates | 阈值 |
|--------|--------------|------|
| 早期   | ≤ 2          | 0.18 |
| 成长期 | 3–6          | 0.30 |
| 成熟期 | 7–16         | 0.34 |
| 稳定期 | > 16         | 0.38 |

### 9.5 候选池更新流程

```
提取出候选条目 c
  │
  向量搜索候选池（similarity > 0.70，同 kind）
  │
  ├─ 找到匹配 m
  │   mention_count++
  │   if c.is_hard: hard_count++
  │   recent_count 滚动更新（最近 3 session 窗口）
  │   total_updates++
  │   重新计算 support_score
  │
  └─ 未找到
      新建候选条目（mention_count=1，hard_count=0 or 1）
      total_updates = 1
  │
  support_score ≥ dynamic_threshold(total_updates)?
  │
  ├─ 是 → 升格
  │        走 dedup（三路相似度）→ add / merge / discard → 写入 skills 库
  │        写入时 entry.status = 'active'
  │        candidate.status = 'promoted'
  │
  └─ 否 → 留池等待
```

### 9.6 Dedup 阶段（借鉴 AutoSkill 维护管线）

候选条目升格后，走完整的 **add / merge / discard** 三阶决策，确保库中不产生重复条目。

#### 阶段一：快速门控（零 LLM）

```
三路相似度：
  combined_score =
    0.70 × 向量余弦相似度
  + 0.18 × signal_overlap（关键词重叠）
  + 0.12 × name_similarity（名称 token Jaccard）

combined_score < 0.50  → 直接 ADD（无重复）
combined_score ≥ 0.90  → 进入合并流程（高置信重复）
0.50 ≤ score < 0.90   → 进入 LLM 决策
```

还有一个零成本 identity 快速检查：
```
normalize(candidate.description) == normalize(existing.description)
→ 完全一致 → 直接 MERGE，跳过 LLM 决策
```

#### 阶段二：LLM 决策（小模型 — 维护决策）

发送候选条目 + 相似条目列表，输出 add / merge / discard：

```
[System]
你是程序记忆库的条目管理器。
任务：决定如何处理新升格的候选条目。

决策流程（按顺序）：
0) 能力重叠硬门控
   相同能力 → 不得 ADD，只能 merge 或 discard。
1) 能力族检查
   完全不同的能力族 → add。
2) 丢弃门控
   过于宽泛 / 信号弱 / 已有条目完全覆盖 → discard。
3) 能力身份判断（4轴）
   a. 核心任务目标
   b. 适用场景 / context 标签
   c. 硬约束（不能做什么）
   d. 工具或操作方法
   → 4轴高度重叠 → merge；有实质差异 → add
4) 兜底
   不确定 → add；明确重叠 → merge 或 discard

只输出 JSON：
{"action": "add"|"merge"|"discard", "target_id": "...", "reason": "..."}

[User]
候选条目：{{candidate}}
相似条目列表：{{similar_items}}
```

Guardrail：LLM 返回 add 但 combined_score > 0.85 → 强制走 merge。

#### 阶段三：合并执行（小模型 — Skill Merger）

仅 action=merge 时触发，先做一次身份二次确认：

> **为什么需要二次确认**：阶段二的向量相似度是语义层面的粗筛，combined_score 高只能说明两个条目"看起来相似"，并不能保证它们是同一项能力。合并是破坏性操作——一旦内容被覆盖写入，原有步骤和注意事项可能永久丢失。二次确认用更严格的能力身份判断（含 RECENCY BIAS）做最后一道安全闸，将"相似但不同能力"的误合并拦在执行之前。

```
[System]
你是能力身份判断器。
任务：确认候选条目与目标条目是否真的是同一项能力。

原则：
- same_capability=true：同一核心任务 + 主要是细节迭代
- same_capability=false：适用场景或核心约束发生本质变化
- 最近性原则（RECENCY BIAS）：候选=近期意图；目标=历史记录
  若意图已切换 → same_capability=false

不确定时默认 false。

只输出 JSON：
{"same_capability": true|false, "confidence": 0.0-1.0, "reason": "..."}

[User]
目标条目：{{existing_item}}
候选条目：{{candidate_item}}
```

`same_capability=false` → 改为 ADD 新建。

`same_capability=true` → 执行语义合并：

```
[System]
你是程序记忆合并器。
任务：将目标条目与候选条目合并为一个更完整的条目。

合并原则：
- 语义并集：不是拼接，是去重融合
- 不能丢失目标条目中的约束（防退化）
- 去标识化：移除案例特定的细节，保留可移植规则
- Anti-Duplication：近似短语保留一个规范表达
- 不发明内容：只用两边原有信息

输出字段：name, context, description

[User]
目标条目（历史）：{{existing_item}}
候选条目（近期）：{{candidate_item}}
```

降级路径：合并 LLM 失败时，直接取两边 description 中较长的一份（近期优先 — RECENCY BIAS），context 标签取并集去重，version +1。

---

## 10. 完整数据流

```
session_end / compact
  │
  第一层清洗（全量）
    → 移除 system/哨兵/注入/机械回复整轮
    → 剥离元数据/推理标签
    → 敏感脱敏
    → 语言检测 → session.lang
  │
  生成滑动窗口 [W0, W1, W2, ...]
  │
  （每窗口进 LLM 前执行第二层：tool_result 60/30 截断）
  │
  for Wi in windows:
    │
    scan(Wi) → used_skill_ids
    │
    ├─ 非空 → 升级路线（§6）
    │          A轨（反馈检测）+ B轨（规则层）→ signal
    │          signal 非空 → EvolveSkill → 规则验证 → 写入
    │          窗口消耗，不进拼接
    │
    └─ 空 → 主题拼接（§7）
             keyword 提取 → Jaccard 比较
             积累 chunk（上限：3窗口 / 2h → 强制flush）
             │
             flush → Chunk
               │
               ├─ Workflow 处理（§8）
               │   混合搜索 → Judge
               │     ├─ 未命中 → §8.3 创建
               │     │           LearnSkill → 并行(Scripts/Refs/Evals)
               │     │           → Evals验证 → 规则验证 → §8.5 Dedup → 写入
               │     └─ 命中   → §8.4 内容合并
               │                 规则层差异分析 → EnrichSkill
               │                 → 规则验证 → 写入
               │
               └─ E+TP 候选提取（§9）
                   → 候选池打分 → 达阈值升格
                   → Dedup（三阶 add/merge/discard）→ skills 库
```

---

## 11. 触发时机

| 事件 | 输入 | 备注 |
|------|------|------|
| `session_end` | 完整 session 消息 | 主路径 |
| `compact` | sessionFile（compact 前全量） | 防止 compact 丢失细节 |

agent_end 不触发（单个 agent 任务不足以跑完整 pipeline）。

---

## 12. 模型分配

| 调用 | 模型 | 理由 |
|------|------|------|
| **§6 升级路线** | | |
| Step A-1 反馈有效性判断 | 小模型 | 结构化分类，每条消息独立 |
| Step A-2 纠正→步骤映射 | 小模型 | 结构化输出，输入已精简 |
| EvolveSkill | 大模型 | 改写质量要求高 |
| **§8 Workflow 学习** | | |
| §8.2 Workflow Judge | 小模型 | 选择题，结构化输出 |
| §8.3 LearnSkill（创建） | 大模型 | 核心提取，质量优先 |
| §8.3.2 Scripts / References | 小模型 | 从已有 SKILL.md 提取，模式明确 |
| §8.3.2 Evals 生成 | 小模型 | 生成测试 prompt，模式明确 |
| §8.4.2 EnrichSkill | 大模型 | 合并需要语义理解 |
| §8.4.3 Description 改善 | 小模型 | 局部改写，范围明确 |
| §8.5 能力身份判断 | 小模型 | 二分类判断 |
| §8.5 Workflow 合并执行 | 大模型 | SKILL.md 格式合并，质量要求高 |
| **§9 E+TP** | | |
| §9.1 LearnExperience/ToolPref | 大模型 | 需要理解对话语义，含成功/失败路径分析 |
| §9.6 维护决策（add/merge/discard） | 小模型 | 结构化决策 |
| §9.6 E+TP 合并执行 | 小模型 | 条目简单，轻量合并 |

---

## 13. 参考来源

| 设计点 | 来源 |
|--------|------|
| 滑动窗口参数（n=10, m=2） | MemOS Python 端 |
| Search → Judge → Create/Upgrade | MemOS TS 端 SkillEvolver |
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
| LearnSkill 提取规则（USER-only、WHAT vs HOW、去标识化） | AutoSkill 提取 Prompt §1-§5 |
| LearnSkill 写作原则（description 触发机制、说明 WHY） | MemOS Generator STEP1_SKILL_MD_PROMPT |
| 并行依赖文件生成（Scripts / References / Evals） | MemOS Generator §7.2-§7.4 |
| Evals 向量召回验证（零 LLM） | MemOS Generator §7.5 |
| Step A-1 FEEDBACK_JUDGEMENT_PROMPT（反馈有效性判断） | MemOS `mem_feedback_prompts.py` |
| Step A-2 UPDATE_FORMER_MEMORIES（操作结构适配） | MemOS `mem_feedback_prompts.py` |
| Step A-3 OPERATION_UPDATE_JUDGEMENT（安全验证层） | MemOS `mem_feedback_prompts.py` |
| Experience "When X, do Y" 格式 + 两类分类 | XSkill Experience 定义 |
| Critique 分析框架（成功/失败/纠正路径） | XSkill INTRA_SAMPLE_CRITIQUE |
| 三路相似度 dedup | AutoSkill |
| 维护决策 add/merge/discard 4轴判断 | AutoSkill 维护管线 §8.2 |
| 合并身份二次确认（same_capability） | AutoSkill 合并判断 §8.3 |
| RECENCY BIAS（近期意图优先） | AutoSkill 合并执行 §8.3/8.4 |
| Anti-Duplication（近似短语去重） | AutoSkill Skill Merger §8.4 |
| identity hash 快速匹配（normalize description） | AutoSkill 维护管线 §8.1 |

---

## 14. 迭代二预留

- Dreaming 跨 session 归纳（候选池的 mention_count 天然积累跨 session 信号）
- Skill 质量审查（merge 候选 / 休眠检测）
- 候选池定期清理（长期 pending 且无新证据 → discard）
