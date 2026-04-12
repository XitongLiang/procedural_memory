# Hermes Agent 在线技能生成与离线 Dreaming 演进全流程分析

> 基于 https://github.com/NousResearch/hermes-agent 源码（2026-04 版本）
> 离线演进部分基于 https://github.com/NousResearch/hermes-agent-self-evolution

---

## 一、Hermes 是什么

Nous Research 开发的开源自进化 AI Agent，2026 年 2 月发布。口号是 "The agent that grows with you"。

核心特点：**四层记忆架构 + 在线技能生成 + 离线 Dreaming 演进**。

跟 AutoSkill / MemOS 最大的区别：**Hermes 有真正的离线演进机制**。AutoSkill 靠 SkillEvo 后台主动改进，MemOS 只有被动升级（用户再做类似任务时才触发），而 Hermes 有完整的 "睡眠整理" 流水线。

---

## 二、四层记忆架构

```
Layer 1: Prompt Memory（冻结快照）
  MEMORY.md (~800 tokens) + USER.md (~500 tokens)
  会话启动时注入系统提示，会话中修改写磁盘但不改当前 prompt
  → 保 LLM prefix cache 性能，下次会话才生效

Layer 2: Skills（三级加载）
  ~/.hermes/skills/*.md
  L0 description 始终在上下文 / L1 SKILL.md 按需 / L2 scripts 按需

Layer 3: Session Archive（主动搜索）
  SQLite + FTS5 全文检索
  Agent 通过 session_search 工具主动查历史对话

Layer 4: External Providers（插件）
  8 种：Hindsight、Honcho、Mem0 等
  Agent 主动调用，各有不同数据结构
```

**关键设计**：Prompt Memory 是**冻结快照**。会话期间写 `MEMORY.md` 改的是磁盘文件，不改当前系统提示。这跟 MemOS 的 Chunk 实时入库不同——Hermes 牺牲实时性换 prefix cache 命中率。

**容量限制**：
- MEMORY.md: 2200 字符（~800 tokens）
- USER.md: 1375 字符（~500 tokens）
- 到 80% 容量时 memory 工具报错，逼 Agent 合并或删除后重试

---

## 三、在线技能生成（会话中）

用户说 "简单写写" 就行的部分。Hermes 的在线技能生成全靠系统提示里的指令 + `memory` 和 `skill_manage` 两个工具，没有独立的提取管线。

### 3.1 memory 工具

**工具描述**（LLM 看到的）：

```
"Save durable information to persistent memory that survives across sessions.
 Memory is injected into future turns, so keep it compact and focused on
 facts that will still matter later."
```

**三个动作**：

| 动作 | 参数 | 说明 |
|------|------|------|
| `add` | target, content | 新增条目 |
| `replace` | target, old_text, content | 子字符串匹配更新（"dark mode" 能匹配到 "User prefers dark mode in all editors"） |
| `remove` | target, old_text | 子字符串匹配删除 |

target 可选 `"memory"`（自己的笔记）或 `"user"`（用户画像）。

### 3.2 skill_manage 工具

**工具描述**（LLM 看到的）：

```
"Manage skills (create, update, delete). Skills are your procedural memory —
 reusable approaches for recurring task types."

"Actions: create (full SKILL.md + optional category), patch (old_string/new_string
 — preferred for fixes), edit (full SKILL.md rewrite — major overhauls only),
 delete, write_file, remove_file."
```

**六个动作**：

| 动作 | 用途 | 关键参数 |
|------|------|---------|
| `create` | 创建新 Skill | name, content（完整 SKILL.md） |
| `patch` | 局部修改（推荐） | name, old_string, new_string |
| `edit` | 完整重写（大改） | name, content |
| `delete` | 删除 | name |
| `write_file` | 写支撑文件 | name, file_path, content |
| `remove_file` | 删支撑文件 | name, file_path |

**Skill 目录结构**（跟 Anthropic 的 agentskills.io 标准一致）：

```
skill-name/
├── SKILL.md           # YAML frontmatter + Markdown 指令
├── scripts/           # 可执行代码
├── references/        # 参考文档
├── templates/         # 模板
└── assets/            # 图标等
```

### 3.3 系统提示中的行为指令

**记忆指导**（prompt_builder.py）：

```
"You have persistent memory across sessions. Save durable facts using the
memory tool: user preferences, environment details, tool quirks, and stable
conventions. Memory is injected into every turn, so keep it compact and
focused on facts that will still matter later.

Prioritize what reduces future user steering — the most valuable memory is
one that prevents the user from having to correct or remind you again.
User preferences and recurring corrections matter more than procedural task
details.

Do NOT save task progress, session outcomes, completed-work logs, or temporary
TODO state to memory; use session_search to recall those from past transcripts.
If you've discovered a new way to do something, solved a problem that could be
necessary later, save it as a skill with the skill tool."
```

**技能指导**（prompt_builder.py）：

```
"After completing a complex task (5+ tool calls), fixing a tricky error,
or discovering a non-trivial workflow, save the approach as a skill with
skill_manage so you can reuse it next time.

When using a skill and finding it outdated, incomplete, or wrong, patch it
immediately with skill_manage(action='patch') — don't wait to be asked.
Skills that aren't maintained become liabilities."
```

**创建标准**（skill_manager_tool.py）：

```
"Create when: complex task succeeded (5+ calls), errors overcome,
 user-corrected approach worked, non-trivial workflow discovered,
 or user asks you to remember a procedure."

"Update when: instructions stale/wrong, OS-specific failures,
 missing steps or pitfalls found during use."

"Good skills: trigger conditions, numbered steps with exact commands,
 pitfalls section, verification steps."
```

### 3.4 在线生成 vs AutoSkill / MemOS

| 维度 | Hermes | AutoSkill | MemOS |
|------|--------|-----------|-------|
| **谁决定提取** | LLM 自己判断 | 代码自动（滑动窗口/会话结束） | 代码自动（话题切换时） |
| **提取管线** | 无独立管线，靠系统提示指导 | 完整 Python 管线（提取→decide→merge） | IngestWorker→TaskProcessor→SkillEvolver |
| **LLM 调用** | 1 次（生成 SKILL.md） | 2-3 次（提取+decide+merge） | 5-8 次（摘要+去重+分类+仲裁+提取+验证） |
| **质量门控** | 无自动门控，靠 LLM 自觉 | 4 维能力身份判断 | qualityScore < 6 → draft |
| **维护** | patch/edit 即时修复 | merge/discard/add 三选一 | upgrade path（被动） |

**Hermes 的哲学**：不做复杂管线，信任 LLM 的判断力，用系统提示引导而非代码控制。好处是简单直接，坏处是质量完全取决于模型能力。

---

## 四、Nudge 机制（周期性提醒）

### 4.1 配置

```yaml
# cli-config.yaml
memory:
  nudge_interval: 10        # 每 10 轮用户消息提醒一次存记忆（0=关闭）
  flush_min_turns: 6        # 退出/重置时最少 6 轮用户消息才触发 flush（0=关闭）

skills:
  creation_nudge_interval: 15  # 每 15 轮提醒一次创建技能
```

### 4.2 演进历史

v0.4.0 的 Release Notes 里写：**"Background memory/skill review replaces inline nudges"**（#2235）。

原来的 nudge 是**内联注入**——每 N 轮往对话里插一条系统消息提醒 Agent "该存记忆了" / "该建技能了"。v0.4.0 改成了**后台审查**，不再打断对话流。

具体机制：配置里的 `nudge_interval` 和 `creation_nudge_interval` 仍然控制触发频率，但不再以可见的系统消息形式注入，而是在后台触发一次轻量 LLM 调用审查最近活动。

---

## 五、离线 Dreaming——这是重点

Hermes 的 "Dreaming" 不是一个单一功能，而是**多层机制的组合**。从内置的简单 flush 到用户自建的复杂整理流程，层层叠加。

### 5.1 机制 1：Gateway Memory Flush（内置）

**触发条件**：
- 空闲超时（用户不活跃）
- 每日凌晨 4 点定时重置
- 会话退出时（需满足 `flush_min_turns ≥ 6`）

**工作方式**：

```
触发
  → 创建临时 AIAgent（max_iterations=8, quiet_mode=True）
  → 只启用 memory + skills 两个工具集
  → 注入旧对话历史
  → 注入当前磁盘上的 MEMORY.md 实时状态（防覆盖）
  → 临时 Agent 执行审查
  → 销毁临时 Agent
```

**代码**（gateway/run.py `_flush_memories_for_session`）：

```python
tmp_agent = AIAgent(
    **runtime_kwargs,
    model=model,
    max_iterations=8,
    quiet_mode=True,
    skip_memory=True,
    enabled_toolsets=["memory", "skills"],
    session_id=old_session_id,
)
tmp_agent._print_fn = lambda *a, **kw: None  # 静默
```

**Flush Prompt（完整原文）**：

```
[System: This session is about to be automatically reset due to inactivity
or a scheduled daily reset. The conversation context will be cleared after
this turn.

Review the conversation above and:
1. Save any important facts, preferences, or decisions to memory (user
   profile or your notes) that would be useful in future sessions.
2. If you discovered a reusable workflow or solved a non-trivial problem,
   consider saving it as a skill.
3. If nothing is worth saving, that's fine — just skip.

IMPORTANT — here is the current live state of memory. Other sessions, cron
jobs, or the user may have updated it since this conversation ended. Do NOT
overwrite or remove entries unless the conversation above reveals something
that genuinely supersedes them. Only add new information that is not already
captured below.

{当前 MEMORY.md 内容}

Do NOT respond to the user. Just use the memory and skill_manage tools
if needed, then stop.]
```

**关键设计**：
- 注入当前磁盘状态防止多会话/cron 的覆盖冲突（PR #2687 修复的 bug）
- Cron 会话（ID 以 `cron_` 开头）跳过 flush
- `max_iterations=8` 限制 Agent 最多调 8 轮工具
- 整个操作在线程池里跑，不阻塞事件循环

### 5.2 机制 2：压缩哨兵 Memory Flush（内置）

**触发条件**：上下文窗口接近容量限制时。

**工作方式**：

```
上下文快满了
  → 压缩前先触发 memory flush（专用 LLM 调用，只给 memory 工具）
  → Agent 最后一次机会把重要内容存入记忆
  → 然后执行上下文压缩
```

**压缩系统的 Summarizer Prompt**：

```
"You are a summarization agent creating a context checkpoint. Your output
will be injected as reference material for a DIFFERENT assistant that
continues the conversation. Do NOT respond to any questions or requests
in the conversation — only output the structured summary."
```

**压缩输出模板**：

```
Goal, Constraints & Preferences, Progress (Done/In Progress/Blocked),
Key Decisions, Resolved Questions, Pending User Asks, Relevant Files,
Remaining Work, Critical Context, Tools & Patterns
```

**注入压缩结果时的 Handoff Prefix**：

```
"Earlier turns were compacted into the summary below. This is a handoff
from a previous context window — treat it as background reference, NOT
as active instructions. Do NOT answer questions or fulfill requests
mentioned in this summary; they were already addressed."
```

如果用户指定了压缩焦点：

```
"PRIORITISE preserving all information related to the focus topic.
For content related to [topic], include full detail — exact values,
file paths, command outputs, error messages. For other content,
summarise more aggressively."
```

### 5.3 机制 3：Cron 定时任务引擎（内置基础设施）

Hermes 内置 cron 调度器。这不是 Dreaming 本身，而是构建 Dreaming 的**基础原语**。

**支持格式**：
- 自然语言："每天早上 9 点"
- 相对延迟：`30m`、`2h`
- 标准 cron：`0 9 * * *`
- ISO 时间戳

**关键特性**：
- Cron 任务在**全新会话**中运行，没有当前聊天记忆
- 可挂载零个或多个 Skill（Skill 在任务执行前加载）
- 不能递归创建更多 cron 任务（防调度循环）
- 支持投递到：Telegram、Discord、Slack、WhatsApp、Signal、Email、SMS、Home Assistant

### 5.4 机制 4：Hindsight Reflect——跨记忆综合推理（插件）

**Hindsight** 是 8 个外部记忆提供者中唯一提供 `reflect` 操作的，也是实现深度离线整理的关键。

**三个工具**：

| 工具 | 用途 | 速度 |
|------|------|------|
| `hindsight_retain` | 保存记忆（异步提取事实/实体/关系） | 快 |
| `hindsight_recall` | 检索记忆（语义+BM25+实体图+时间） | 快 |
| `hindsight_reflect` | 跨记忆综合推理（遍历知识图谱） | 慢 |

**Hindsight 的数据结构**（PostgreSQL + pgvector）：

不存纯文本，存**结构化知识**：
- **事实**：离散陈述（"生产 DB 跑在端口 5433"）
- **实体**：命名元素 + 实体消解（"Alice" = "工程部的同事 Alice"）
- **关系**：跨会话关系追踪（"auth 服务依赖 Redis 做 session 管理"）

**reflect 的工作流**：

```
hindsight_reflect(query)  # query 最多 800 字符
  → 不是简单检索，而是遍历整个知识图谱
  → 跨所有存储记忆进行推理
  → 产出综合性答案（不是检索结果列表）
  → 新洞察写回知识图谱
```

**Recall 的四路并行检索**：
1. 语义向量搜索
2. BM25 关键词匹配
3. 实体图遍历
4. 时间过滤

然后用 cross-encoder 重排序。

**日常运作**：每个对话轮次结束后，对话内容异步发送给 Hindsight 做后台提取（不消耗推理 tokens，post-response）。

LongMemEval 基准最高分：**91.4%**。

### 5.5 机制 5：用户自建 "梦境" 技能（实践案例）

Alexander Kucera 在 ["One Week on Hermes"](https://alexanderkucera.com/2026/04/10/one-week-on-hermes.html) 中实现的：

```
每 4 小时（cron 触发）：
  运行 hindsight-reflect 技能
  → 对所有记忆进行跨记忆综合
  → 发现矛盾、冗余、缺失
  → 新洞察写回知识图谱

每晚（过夜梦境，cron 触发）：
  综合当天所有会话
  → 整合所学内容
  → 早上向用户呈现发现结果

效果：
  Agent 在用户睡觉时整理知识
  多个 Agent 互相调试代码、维护 wiki
  早上起来收到整理报告
```

**这不是 Hermes 的内置功能**，而是用 Hermes 的原语（cron + hindsight_reflect + skills）组合出来的**涌现能力**。框架提供积木，用户搭出梦境。

### 5.6 机制 6：GEPA 自动进化（伴随仓库，人工把关）

**仓库**：[NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution)

这是最复杂的离线演进机制。用 **DSPy + GEPA**（Gradient-free Evolutionary Prompt Adaptation，ICLR 2026 Oral）自动演进 Skill、工具描述、系统提示甚至代码。

**GEPA 的核心创新**：不像 RL 把执行轨迹压成一个标量奖励，GEPA 读**完整执行轨迹**——错误信息、推理日志、工具调用、中间步骤——来诊断*为什么*失败，然后提出针对性修复。最少只需 3 个样例。成本 ~$2-10/次优化。无需 GPU。

**三个优化引擎**：

| 引擎 | 目标 | 许可证 |
|------|------|--------|
| DSPy + GEPA | Skill、提示、工具描述 | MIT |
| Darwinian Evolver | 代码文件、算法 | AGPL v3 |
| DSPy MIPROv2 | Few-shot 样例、指令文本 | MIT |

**完整流水线**：

```
Step 1: 选择目标（Skill / 工具描述 / 提示 / 代码）

Step 2: 构建评估数据集
  方式 a: 合成——强模型生成 15-30 个 (task_input, expected_behavior) 对
  方式 b: SessionDB 挖掘——查询真实会话中 Skill 被加载的记录，LLM-as-judge 打分
  方式 c: 手工策划的黄金集（JSONL，可选）
  方式 d: Skill 自测（如：埋 bug → 运行 Skill → 检查是否修复）

Step 3: 把目标包装成 DSPy 模块
  SKILL.md 内容 → 系统提示

Step 4: 运行 GEPA 优化器
  batch_runner 在测试任务上并行执行 Agent
  → 收集执行轨迹（输入 + 推理步骤 + 工具调用 + 输出）
  → GEPA 分析轨迹，识别失败模式
  → 提出语义驱动的变异（不是随机变异）
  → 评估候选变体

Step 5: LLM-as-judge 打分
  - Agent 是否遵循流程？(0-1)
  - 输出是否正确/有用？(0-1)
  - 是否在 token 预算内简洁？(0-1)

Step 6: 约束验证
  - 完整测试套件必须 100% 通过
  - Skill 不超过当前大小 +20%（15KB 上限）
  - 工具描述 ≤ 500 字符
  - 不能破坏 prompt caching 兼容性
  - 基准不回退（TBLite 容差 2%）

Step 7: 通过 PR 部署（人工审查）
  永远不直接 commit，所有改动走 Pull Request
```

**CLI**：

```bash
# 演进一个 Skill，跑 10 轮迭代
python -m evolution.skills.evolve_skill \
    --skill github-code-review \
    --iterations 10 \
    --eval-source synthetic

# 用真实会话数据演进
python -m evolution.skills.evolve_skill \
    --skill arxiv \
    --dataset eval_tasks.jsonl \
    --iterations 5

# 对比版本
hermes evolve compare github-code-review --version latest

# 部署特定版本
hermes evolve deploy github-code-review --version 3
```

**五个阶段**（13-17 周）：

| 阶段 | 目标 | 周期 | 风险 |
|------|------|------|------|
| 1 | Skill 文件 | 3-4 周 | 最低 |
| 2 | 工具描述 | 2-3 周 | 低 |
| 3 | 系统提示 | 2-3 周 | 中 |
| 4 | 代码 | 3-4 周 | 最高 |
| 5 | 持续循环 | 2 周 | — |

---

## 六、提案中的认知记忆操作（Issue #509，未合并）

受 CrewAI 启发的完整认知记忆系统提案，5 大操作：

### 6.1 Encode（编码）— 5 步流水线

```
批量嵌入 → 批内去重(≥0.98) → 并行相似度搜索 → 并行 LLM 分析 → 批量执行
```

LLM 分析产出：
- `suggested_scope`：层级路径（如 `/infrastructure/database`）
- `categories`：标签
- `importance`：0.0-1.0
- `extracted_metadata`：实体、日期、主题

### 6.2 Consolidate（合并整理）

- 触发：相似度 ≥ 0.85
- 生成 `ConsolidationPlan`：keep / update / delete
- 例：已有 "用 PostgreSQL" + 新 "迁移到 MySQL" → 合并为 "从 PostgreSQL 迁移到 MySQL"

### 6.3 Recall（召回）— 复合评分

```
score = 0.5 × similarity + 0.3 × decay + 0.2 × importance
decay = 0.5^(age_days / 30)

置信度路由：
  ≥ 0.8 → 直接返回
  < 0.5 → 有预算就深挖
  < 0.7 且复杂查询 → 深挖
```

### 6.4 Extract（提取）

原始文本 → 分解为原子事实 → 每个事实独立进入编码流水线

### 6.5 Forget（遗忘）

按范围/类别/年龄/记录 ID 定向清除 + 自动 importance 衰减

---

## 七、全部 LLM Prompt 清单

### 7.1 在线——系统提示指令

| # | 位置 | 内容摘要 |
|---|------|---------|
| P1 | prompt_builder.py / 身份 | "You are Hermes Agent, an intelligent AI assistant created by Nous Research..." |
| P2 | prompt_builder.py / 记忆指导 | 存持久事实、减少未来用户纠正、不存任务进度 |
| P3 | prompt_builder.py / 技能指导 | 5+ 调用后建技能、发现过时立即 patch |
| P4 | prompt_builder.py / 工具执行 | "You MUST use your tools to take action — do not describe what you would do" |
| P5 | prompt_builder.py / Skill 索引 | "Skills (mandatory)" — 扫描并加载相关 Skill |
| P6 | prompt_builder.py / 平台提示 | WhatsApp/Telegram/Discord/Slack 等格式约束 |

### 7.2 在线——工具描述

| # | 工具 | 描述原文 |
|---|------|---------|
| T1 | memory | "Save durable information to persistent memory that survives across sessions..." |
| T2 | skill_manage | "Manage skills (create, update, delete). Skills are your procedural memory..." |
| T3 | skill_view | 查看 Skill 内容和支撑文件 |
| T4 | session_search | 搜索历史会话 |

### 7.3 在线——Skill 创建标准

| # | 位置 | 内容 |
|---|------|------|
| S1 | skill_manager_tool.py / 创建标准 | "Create when: complex task succeeded (5+ calls), errors overcome..." |
| S2 | skill_manager_tool.py / 更新标准 | "Update when: instructions stale/wrong, OS-specific failures..." |
| S3 | skill_manager_tool.py / 质量标准 | "Good skills: trigger conditions, numbered steps with exact commands..." |
| S4 | skill_manager_tool.py / 确认要求 | "Confirm with user before creating/deleting" |

### 7.4 离线——Gateway Flush

| # | 位置 | 内容 |
|---|------|------|
| F1 | gateway/run.py | "[System: This session is about to be automatically reset...Review the conversation above and: 1. Save any important facts... 2. If you discovered a reusable workflow... 3. If nothing is worth saving, just skip...]" |
| F2 | gateway/run.py / 防覆盖 | "IMPORTANT — here is the current live state of memory. Other sessions, cron jobs, or the user may have updated it..." |
| F3 | gateway/run.py / 结束 | "Do NOT respond to the user. Just use the memory and skill_manage tools if needed, then stop." |

### 7.5 离线——上下文压缩

| # | 位置 | 内容 |
|---|------|------|
| C1 | context_compressor.py / Summarizer | "You are a summarization agent creating a context checkpoint..." |
| C2 | context_compressor.py / 模板 | Goal, Constraints & Preferences, Progress, Key Decisions... |
| C3 | context_compressor.py / Handoff | "Earlier turns were compacted into the summary below. This is a handoff from a previous context window..." |
| C4 | context_compressor.py / 焦点 | "PRIORITISE preserving all information related to the focus topic..." |

### 7.6 离线——GEPA 评估

| # | 位置 | 内容 |
|---|------|------|
| G1 | hermes-agent-self-evolution / LLM-as-judge | 三维评分：遵循流程(0-1)、输出正确性(0-1)、简洁度(0-1) |
| G2 | hermes-agent-self-evolution / 约束 | Skill ≤15KB、工具描述 ≤500 字符、测试 100% 通过 |

---

## 八、离线 Dreaming 完整分层

```
Layer 1: Gateway Flush（内置，被动）
  空闲/4AM → 临时 Agent 审查旧对话 → 写 MEMORY.md / 建 Skill
  ↓ 最基础的记忆持久化

Layer 2: 压缩哨兵 Flush（内置，被动）
  上下文快满 → memory flush → 压缩
  ↓ 防止上下文丢失

Layer 3: Cron 定时反思（用户配置）
  每 N 小时 → hindsight_reflect → 跨记忆综合 → 知识图谱更新
  ↓ 定期整理

Layer 4: 过夜梦境（用户自建 Skill）
  每晚 cron → 综合当天所有会话 → 早上报告
  ↓ 深度整理

Layer 5: GEPA 自动进化（伴随仓库，人工把关）
  按需/定期 → 挖掘 SessionDB 找失败 → GEPA 分析执行轨迹
  → 演进 Skill/提示文本 → 验证 → PR 人工审查
  ↓ 系统性优化

Layer 6（提案中）: 认知记忆操作（Issue #509）
  编码+分类 → 合并矛盾 → 遗忘过时 → 提取原子事实 → 自适应深度召回
```

**从下到上，自动化程度递增，人工参与递减。**

---

## 九、三系统离线演进对比

| 维度 | AutoSkill (SkillEvo) | MemOS | Hermes Dreaming |
|------|---------------------|-------|-----------------|
| **有没有离线演进** | 有（SkillEvo 后台） | 没有 | 有（多层） |
| **触发方式** | 后台定时扫描 | 用户再做类似任务时被动触发 | Gateway flush + Cron + GEPA |
| **演进粒度** | 单个 Skill 改写 | 单个 Skill 升级（merge） | 跨所有记忆综合推理 |
| **数据结构** | 平文本 SKILL.md | Chunk + Skill（SQLite） | 知识图谱（事实+实体+关系） |
| **质量验证** | SkillEvo 自带评估 | qualityScore LLM 打分 | GEPA + 测试套件 + 基准门控 |
| **人工参与** | 无 | 无 | GEPA 阶段需 PR 审查 |
| **人类类比** | 后台自动复习 | 下次遇到才想起来改 | 睡觉时大脑整理记忆 |

---

## 十、设计洞察

### 10.1 信任 LLM vs 信任管线

Hermes 的在线技能生成几乎没有独立管线——全靠系统提示里的指令引导 LLM 自主决定何时存记忆、何时建技能。这跟 AutoSkill（滑动窗口自动提取）和 MemOS（话题切换自动触发）形成鲜明对比。

**取舍**：管线更可靠但更僵化，LLM 自主更灵活但不可预测。Hermes 赌的是模型能力会持续提升，所以管线越简单越好。

### 10.2 冻结快照是聪明的

会话中修改 MEMORY.md 只写磁盘不改当前 prompt，这意味着 LLM 的 prefix cache 在整个会话中保持有效。MemOS 每次入库都可能改变系统提示，cache 失效。

### 10.3 容量压力是特性不是缺陷

MEMORY.md 只有 2200 字符。到 80% 时工具报错逼 Agent 合并。这跟人脑的遗忘曲线类似——不是所有记忆都值得保留，压力促进筛选。

### 10.4 Dreaming 是组合不是功能

Hermes 没有一个叫 "Dreaming" 的按钮。它提供原语（cron + memory + skills + hindsight_reflect），用户组合出 Dreaming 流程。这是平台思维 vs 产品思维。

### 10.5 GEPA 是真正的闭环

AutoSkill 的 SkillEvo 用 LLM 改写 Skill 但没有客观验证。Hermes 的 GEPA 用 batch_runner 执行、LLM-as-judge 评分、测试套件把关、基准回归检查——这是目前最完整的 Skill 演进质量保证。

---

## 参考资料

- [NousResearch/hermes-agent](https://github.com/nousresearch/hermes-agent)
- [NousResearch/hermes-agent-self-evolution](https://github.com/NousResearch/hermes-agent-self-evolution)
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs/)
- [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/)
- [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)
- [Skills System](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills)
- [Cron Scheduling](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron)
- [Issue #509: Cognitive Memory Operations](https://github.com/NousResearch/hermes-agent/issues/509)
- [GEPA Paper (ICLR 2026 Oral)](https://arxiv.org/abs/2507.19457)
- [One Week on Hermes — Alexander Kucera](https://alexanderkucera.com/2026/04/10/one-week-on-hermes.html)
- [Hindsight Blog](https://hindsight.vectorize.io/blog/2026/03/17/hermes-agent-memory-memory)
