# GSPD 记忆引擎/上下文引擎分层设计深度评估报告

> 基于 XSkill、Mem0、ReMe、GEPA、Hermes、MemOS、AutoSkill、Skill-Creator 等 8 个业界记忆系统的源码级调研分析

---

## 前置说明

本报告旨在回答 gspd 记忆引擎设计中的 30+ 个核心问题，并通过跨系统对比给出可落地的设计建议。以下分析主要基于当前目录下已有的深度源码调研文档（这些文档直接引用了各系统的核心源码、Prompt 原文和数据结构），部分关键结论已核对至代码实现层面。

---

## 第一部分：GSPD 4 层架构总评

### 你的设计骨架：L3(原始会话) → L2(原子事实/六类) → L1(渐进摘要) → L0(固定加载)

**优点：**
1. **方向正确**：从"细粒度原始数据"向上"蒸馏"到"粗粒度可直接用"的知识，符合信息论中的压缩-重构原理。
2. **渐进加载思想先进**：L1 用于渐进加载、L0 固定加载，这与 Anthropic Skill-Creator 的三级加载（L0 description / L1 SKILL.md / L2 scripts）、MemOS 的渐进式披露（progressive disclosure）、Hermes 的三级 Skill 加载（L0/L1/L2）在理念上高度一致。
3. **异步提取与分层检索解耦**：生成顺序（L3→L2→L1→L0）和检索顺序（L0→L1→L2→L3）相反，这是经典的知识金字塔模型（DIKW：Data→Information→Knowledge→Wisdom 的变体）。

**但存在 4 个根本性结构性风险，需要在详细设计前澄清：**

| 风险 | 说明 |
|------|------|
| **R1: L2 六分类的边界极其模糊** | "原子事实"被强制分配到情景/语义/程序/前瞻/心智/通用六类，但对话中的大量信息是跨类的（例如："用户喜欢 Python"既是语义记忆也是心智记忆；"用 Docker 部署"既是程序记忆也包含工具偏好）。分类冲突会在提取和检索时造成系统性噪声。 |
| **R2: L1 的摘要生成时机和输入范围不明确** | 如果 L1 是"按记忆类别或任务主题抽取的摘要"，那么当同一个对话同时产出 3 类记忆时，L1 需要合并这 3 类的摘要吗？如果合并，谁来决定主题边界？ |
| **R3: L3→L2 的提取窗口没有定义** | 是用整条对话提取？滑动窗口？还是按任务切分？不同的切分策略会彻底改变 L2 的质量。 |
| **R4: 缺乏质量门控和版本管理** | 当前设计中没看到提取失败时的回退策略、记忆矛盾时的仲裁机制、以及旧版本的归档逻辑。 |

---

## 第二部分：逐问题跨系统映射分析

### 1. L3 原始会话层

#### 1.1 L3 是否需要存储？如何存储？Block 粒度？包含哪些信息？

**各系统的设计思路对比：**

| 系统 | L3 等价物 | 是否持久存储 | Block 粒度 | 包含内容 |
|------|----------|-------------|-----------|---------|
| **Mem0** | `messages` 参数传入的原始对话 | **不独立存储原始对话**，只存储提取后的记忆事实/程序记忆摘要 | 近 10 条消息（`m=10`）为一个提取窗口 | user/assistant/tool 全有，但程序记忆路径会跳过 ADD/UPDATE/DELETE 决策，直接 ADD |
| **MemOS TS** | `dialog/YYYY-MM-DD.jsonl` | **存原始对话**，但用游标机制增量处理 | 以 `turn` 为单位（user+assistant 为 1 turn），用 `turnId` 关联 | `role`, `content`, `timestamp`, `turnId`, `toolName`, `owner` |
| **ReMe** | `dialog/YYYY-MM-DD.jsonl` | **存原始对话** | 原始对话轮次 | 完整消息历史 |
| **Hermes** | `Session Archive` (SQLite + FTS5) | **存原始对话**，但 Agent 通过 `session_search` 工具按需检索，不是自动全量加载 | 单条消息或会话级别 | 完整对话文本 + tool_calls + tool_results；检索时先做 FTS5 关键词搜索，再对 top-3 会话做智能窗口截断（~100k chars  centered on matches），最后由辅助模型（Gemini Flash）生成摘要返回 |
| **XSkill** | `rollout_X/trajectory.jsonl` | **存完整轨迹**（这是训练框架，非在线服务） | 一次 rollout（最多 20 轮工具调用） | 完整 trajectory + 工具生成的中间图像 (`tool_image_*.jpg`) + 原始图像 |
| **AutoSkill** | `messages`（通过 hook 传入） | **不存原始对话**，只从 `messages[-6:]` 滑动窗口中提取 Skill | 滑动窗口：最近 6 条消息 | user/assistant，tool 结果在轨迹提取中有，但在对话提取中弱化 |

**针对性分析与建议：**

**L3 原始会话强烈建议存储。** 原因：
- **Mem0 的问题**：不存原始对话导致一旦提取失败或提取偏了，无法回溯修正。Mem0 的论文自己也承认这是 trade-off——为省 90% token 成本牺牲了完整可审计性。
- **MemOS/ReMe/Hermes 的做法更稳妥**：原始对话作为"录像带"保留，记忆提取只是"笔记"。当上层记忆（L2/L1）出现矛盾、过时时，可以用 L3 做 ground truth 校验。

**建议的存储方案（借鉴 MemOS TS + ReMe）：**

```
L3 存储结构（每条消息一个记录）:
{
  "block_id": "session-turn level identifier",
  "session_id": "...",
  "turn_id": "uuid",
  "seq": 0,
  "role": "user" | "assistant" | "tool",
  "content": "清洗后的纯文本",
  "raw_content": "原始未清洗文本（用于审计）",
  "tool_calls": [{"name": "...", "arguments": "...", "id": "..."}],  // assistant 消息时
  "tool_results": [{"name": "...", "content": "...", "exitCode": 0}], // tool 消息时
  "timestamp": 1234567890,
  "metadata": {
    "model": "gpt-4o",
    "token_count": 150,
    "compressed_from": "block_id_older"  // 如果本条来自上下文压缩
  }
}
```

**Block 粒度建议：**
- **最小单元**：单条消息（与 MemOS Chunk 对齐）。
- **聚合单元**：一个 `turn`（user + assistant + 中间 tool_calls/results）。
- **任务单元**：多个 turn 组成一个 `task`（由主题分割算法决定，见下文）。

**是否包含 tool_call/result？**
- **必须包含**。XSkill、MemOS、AutoSkill 的轨迹提取都证明：没有 tool 调用记录，无法提取程序记忆/工具偏好/纠错经验。
- **但要压缩**：MemOS 的"60/30 截断规则"非常值得借鉴——成功的 tool_result 截断到 1500 字符（保留头部 60% + 尾部 30%），失败的 tool_result 完整保留（因为失败记录是经验来源）。

**还需要包含的信息：**
- **上下文压缩链路**（如果某条消息是压缩后的 summary，需要标注 `compressed_from`）
- **注入的记忆/Skill ID 列表**（用于后续做使用审计和反馈闭环）
- **环境指纹**（Agent 版本、可用工具列表、系统提示哈希）——这对处理"环境和工具变化"至关重要

#### 1.2 Block 之间是否需要关联？

**业界系统的关联设计：**

| 系统 | 关联方式 | 想解决的问题 | 引入的问题 |
|------|---------|-------------|-----------|
| **MemOS TS** | `taskId` + `turnId` + `seq` | 同一任务内的消息关联；支持按任务提取 Skill | 主题切换判断（Topic Split）需要额外的 LLM 调用 |
| **MemOS Py** | `message_indices`（支持非连续归组） | 用户可能跳跃式对话（先聊旅行，再聊代码，再聊旅行） | LLM 聚类的准确性依赖模型能力，可能误分 |
| **ReMe** | 时间戳 + 会话 ID | 简单的时间序列关联 | 无任务语义关联，跨任务提取质量低 |
| **Hermes** | SQLite + FTS5 全文索引 | Agent 通过 `session_search` 按关键词主动查 | 完全依赖 Agent 自觉，无自动关联 |
| **XSkill** | 按 `sample` → `rollout` 层级 | 训练框架内的样本- rollout 关联 | 不适合在线服务 |

**建议：**
Block 之间**必须建立三层关联**：
1. **时序关联**：`prev_block_id` / `next_block_id`（链表结构，支持快速遍历）
2. **任务关联**：`task_id`（同一任务主题的 Block 聚合，借鉴 MemOS）
3. **因果关联**：`triggered_by_tool_call_id`（tool_result 指向产生它的 assistant tool_call）

此外，建议增加一个**压缩关联**：如果某段 Block 被压缩成 Summary Block，建立 `compressed_from: [block_ids]` 和 `expanded_to: summary_block_id` 的双向指针。这在上下文压缩后追溯原始信息时非常关键。

#### 1.3 Block 如何与 Agent 的 Context 对应？

这是 gspd 设计中**最棘手的工程问题**之一。Context 不是静态的，它会经历：压缩、环境变化（工具增删）、系统提示更新。

**各系统的对应策略：**

| 系统 | Context 映射策略 | 压缩策略 | 环境变化处理 |
|------|-----------------|---------|-------------|
| **Hermes** | 会话中 MEMORY.md 是**冻结快照**——磁盘文件改了，当前 Context 里的系统提示不改。这是源码级设计：`MemoryStore._system_prompt_snapshot` 在 `load_from_disk()` 时捕获，之后 mid-session 的 `memory` 工具调用只更新磁盘文件（atomic temp-file + rename），不更新 in-flight system prompt | 压缩前先触发 memory flush，然后 `ContextCompressor` 执行 5 阶段算法：① 旧 tool result 剪枝（生成 1 行摘要）② 保护 head ③ token-budget 保护 tail ④ LLM 结构化 summarization（12 字段模板）⑤ tool-call 完整性修复。摘要注入时使用严格的 handoff framing 前缀 | 工具/schema 变化靠 Skill 的 patch 机制在线修复 |
| **MemOS** | `before_prompt_build` hook 动态注入检索结果，每次 turn 都可能变 | 无内置上下文压缩（依赖上游 OpenClaw） | 通过 `context_assemble` 工具感知当前可用 Skill |
| **XSkill** | 训练时把 Experience 和 Skill 注入系统提示；测试时先做 Skill Adaptation 再注入 | 不适用（离线训练框架，Context 由研究者控制） | 不适用 |
| **ReMe** | 检索后通过 `RewriteMemoryOp` 将多条经验重写为统一建议再注入 | 文件系统版依赖外部 Agent 的压缩策略；向量版无压缩 | 无专门处理 |
| **AutoSkill** | `before_prompt_build` 注入检索到的 Skill 上下文 | 无 | 无 |

**核心洞察：**

Hermes 的**冻结快照**设计非常关键。如果 L3 Block 被提取后，每次提取都会改变当前 Context 的系统提示，那么 LLM 的 prefix cache 会持续失效，导致推理延迟飙升。更重要的是，在线修改系统提示会让 Agent 行为在会话中发生不可预测的偏移。

**建议的映射策略：**

```
会话启动时：
  → 加载 L0 固定内容（固定加载层）
  → 组装初始 Context（System Prompt + L0 + 最近的 N 条 L3 Block）

会话进行中：
  → L3 的原始 Block 持续追加写入存储
  → 但 L2/L1/L0 的更新**只写磁盘/向量库，不改当前 Context**
  → 只有当 Agent 显式调用 memory_recall / workflow_recall 时，才把最新检索结果以 tool_result 形式注入

**Hermes 的源码级实现（`tools/memory_tool.py`）：**
- `MemoryStore` 维护两个状态：`_system_prompt_snapshot`（冻结）和 `memory_entries`/`user_entries`（ live ）。
- `load_from_disk()` 时从 `~/.hermes/memories/MEMORY.md` 和 `USER.md` 读取，按 `\n§\n`（section sign）分割成条目，去重后捕获快照。
- `memory_tool(action='add')` 时：
  1. 获取文件锁（跨平台，`fcntl` 或 `msvcrt`）
  2. 重新从磁盘 reload 目标文件（防多 session 覆盖）
  3. 检查字符限制（`memory_char_limit=2200`，`user_char_limit=1375`）
  4. 原子写入（`tempfile.mkstemp` + `os.replace`）
  5. **但绝不更新 `_system_prompt_snapshot`**
- 这意味着：Agent 在本会话中写入的记忆，只有通过 `memory_tool` 的 JSON 返回结果才能看到；系统提示中的 `MEMORY.md` 快照保持会话开始时的状态。
- **优点**：prefix cache 完全稳定，推理成本低。
- **代价**：如果 Agent 忘记主动阅读 tool 返回值，它会"意识不到"自己刚写入了新记忆。

Context 接近满时：
  → 触发压缩：将早期的 L3 Block 替换为压缩 Summary Block
  → 压缩 Summary 本身也作为新的 Block 写入 L3
  → 标注 `compressed_from` 关联，保证 L2 提取时知道哪些 Block 已被压缩

环境变化时（如新增/删除工具）：
  → 不修改历史 L3 Block
  → 在 L3 中新增一条 "environment_change" 类型的 Block，记录变化内容
  → L2 提取时，对于跨越环境变化边界的 Block，降低其通用化权重（因为旧工具的经验可能已失效）
```

#### 1.4 Block 的数量是否有限制？

**各系统的限制策略：**

| 系统 | 限制策略 | 理由 |
|------|---------|------|
| **Hermes** | `MEMORY.md` 2200 字符硬限制，到 80% 时报错逼 Agent 合并 | 容量压力驱动整合，防止记忆膨胀 |
| **XSkill** | Experience 库 120 条硬限制，超限时强制合并最相似对 | 保证检索和注入效率 |
| **Mem0** | 无明确限制（依赖向量 DB 容量） | 作为中间件，把限制交给上层 |
| **MemOS** | SKILL.md 400 行（~6000 词）限制，超限时触发精炼 | 保证单次注入不爆上下文 |
| **AutoSkill** | `sessionMaxTurns=20` 强制关闭 session | 防止单次会话过长导致提取质量下降 |

**建议：**
L3 Block 的**数量不应有硬限制**，但应该有**分层管理策略**：

1. **热区（Hot Zone）**：最近 1-2 个会话的 Block，常驻快速索引
2. **温区（Warm Zone）**：30 天内的 Block，存储在标准向量/SQLite 中
3. **冷区（Cold Zone）**：超过 30 天的 Block，归档到压缩存储（只保留任务级摘要，原始消息可抽样保留）

对于单个**会话（Session）**内的 Block 数量，建议设置**软限制**：
- 超过 50 个 turn 时，提示 Agent 或系统自动触发会话总结
- 超过 100 个 turn 时，强制建议开启新会话（不是删除旧 Block，而是将旧会话标记为只读归档）

---

### 2. L2 原子事实层

#### 2.1 当前 L2 的六大分类是否能覆盖所有记忆数据？是否有理论支撑？是否有更好的分类理论？

**你的六类分类：情景记忆、语义记忆、程序记忆、前瞻记忆、心智记忆、通用记忆**

**这六类的来源分析：**
这基本上是将认知心理学中的记忆分类（Tulving 的情景/语义记忆、Cohen 的程序记忆、Atkinson-Shiffrin 的工作记忆模型）直接映射到了 LLM Agent 架构中。这种映射有两个问题：

**问题 1：分类标准不统一**
- 情景/语义/程序记忆是按**内容类型**分的
- 前瞻记忆是按**时间取向**分的（面向未来）
- 心智记忆/通用记忆是按**归属对象**分的（用户心智 vs 通用知识）

当分类维度交叉时，一条信息可能同时属于多个类别。例如：
- "用户计划下周去日本旅行" → 情景记忆（具体事件）+ 前瞻记忆（未来计划）+ 语义记忆（用户的旅行偏好）
- "Agent 在处理 JSON 时应该先 validate 再 parse" → 程序记忆（工作流程）+ 通用记忆（编程常识）+ 情景记忆（某次具体任务中总结出来的）

**问题 2：在工程实现中，分类歧义会导致提取和检索的系统性错误。**

**各系统的分类策略对比：**

| 系统 | 分类方式 | 理论来源 | 工程效果 |
|------|---------|---------|---------|
| **Mem0** | `semantic_memory` / `episodic_memory` / `procedural_memory` | Tulving + Cohen | 实际上只有 `procedural_memory` 有独立处理路径，其他两类底层走同一存储管线 |
| **ReMe** | `Personal Memory` / `Task Memory` / `Tool Memory` | 按**使用场景**分 | 非常清晰，工程实现简单，检索时按 kind 过滤即可 |
| **MemOS** | `workflow` / `experience` / `tool_preference` / `preference` | 按**功能角色**分 | 与你的"程序记忆"概念一致，但粒度更粗，避免了心理学分类的歧义 |
| **XSkill** | `Experience`（战术提示）/ `Skill`（任务级 SOP） | 按**粒度 + 使用方式**分 | 没有心理学分类，纯粹从工程价值出发：Experience 用于情境微调，Skill 用于结构防错 |
| **Hermes** | `MEMORY.md`（事实）/ `Skill`（程序） | 二元分法 | 最简单，最不容易出错 |
| **AutoSkill** | 单一 `SKILL.md` | 无分类，只按任务类型组织 | 依靠 Skill 的 name/tags 做检索，不内建记忆类型分类 |

**更好的分类理论建议：**

对于 LLM Agent 的记忆系统，**心理学分类是过度工程化的**。更工程化的分类应该基于两个维度：
1. **谁在什么时候用它？**（检索时态）
2. **以什么形式注入上下文？**（知识载体）

**推荐方案 A（借鉴 ReMe + MemOS）：按功能角色分类**

```
L2 分类（3+1 类）：
├── Facts（事实记忆）      ← 对应你的语义记忆+心智记忆
│     用户偏好、画像、静态关系
├── Experiences（经验记忆）  ← 对应你的情景记忆+程序记忆的战术部分
│     工具技巧、踩坑记录、纠错反馈、最佳实践
├── Workflows（流程记忆）    ← 对应你的程序记忆的战略部分
│     完整 SOP、多工具编排、错误恢复机制
└── Plans（计划记忆）        ← 对应你的前瞻记忆
      用户待办、长期目标、时间约束事件
```

**推荐方案 B（借鉴 XSkill）：按粒度 + 注入方式分类**

```
L2 分类（2 类）：
├── Experience（≤64 词的条件-动作对）
│     检索方式：按子任务查询做向量相似度搜索
│     注入方式：非强制性建议（"Consider..."）
└── Skill（~600 词的结构化 SOP）
│     检索方式：任务描述匹配 frontmatter description
│     注入方式：触发后加载完整内容
```

XSkill 的消融实验证明：**仅 Skill 错误率 15.3%，仅 Experience 错误率 29.9%，两者结合最佳**。这证明不同粒度的知识需要不同的表示和检索方式，而不是按心理学内容类型分。

**推荐方案 C（混合）：三维标签系统**

如果一定要保留心理学分类的直觉价值，建议不要用"大类"做硬切分，而是给每条 L2 原子事实打**三维标签**：

```
- content_type: [fact | experience | workflow | plan]
- time_orientation: [past | present | future]
- scope: [user_specific | agent_specific | universal]
```

这样"用户计划去日本"可以同时被打上 `content_type=fact`, `time_orientation=future`, `scope=user_specific`，避免了被迫选一个大类的问题。

#### 2.2 L2 的原子事实是什么？如何定义范围/粒度/边界？

**这是 gspd 设计中最关键、也是最模糊的问题。**

**各系统对"原子事实"的定义：**

| 系统 | 原子单元定义 | 粒度控制 | 边界原则 |
|------|------------|---------|---------|
| **Mem0** | 一句话级别的事实（"User likes cricket"） | LLM 提取 Prompt 要求生成 `facts` 列表 | 基于 user 消息，不含 assistant 消息 |
| **XSkill Experience** | `e = (c, a, v_e)`，条件+动作 ≤ 64 词 | **硬约束：≤64 词** | 以触发条件开头（"When..."），聚焦可操作指导 |
| **XSkill Skill** | 一个 Markdown SOP 文档 | **硬约束：≤1000 词** | 任务级，包含 Workflow/Tool Templates/Pitfalls |
| **ReMe Task Memory** | `when_to_use + experience + tags + confidence` | 自然语言段落，无硬长度限制 | step-level，每次提取 1-3 条 |
| **MemOS TS Chunk** | 一个名词短语标题（15-30 中文字） | LLM 摘要 Prompt 控制 | 消息级，保留完整原始内容 |
| **MemOS Py** | `SkillMemory`（name, description, procedure, experience, preference, examples） | 两阶段生成：骨架 + 详细 | 任务级，但 procedure/experience 分开存储 |
| **AutoSkill** | `Skill` JSON（name, description, prompt, triggers, tags, confidence） | Prompt 要求 `prompt` 包含 #Goal / #Constraints / #Workflow | 滑动窗口级（最近 6 条消息） |

**核心洞察：**

XSkill 的 **64 词硬约束** 是一个极其成功的设计。它强制 LLM 提炼核心洞察，而不是复制粘贴对话原文。AutoSkill 和 MemOS 没有这种硬约束，导致提取的知识越来越冗长、具体、过拟合。

**对 gspd L2 的建议：**

```
L2 原子事实 = 一条可被独立检索、独立验证、独立注入的知识单元

定义要素：
- identity: 唯一 ID
- content: 核心陈述（一句话到一段话）
- scope: 适用边界（何时/何地/对谁有效）
- source: 来源 L3 Block IDs（用于溯源和审计）
- embedding: 语义向量（用于检索）
- kind: [fact | experience | workflow | plan]（软标签，可多选）
- confidence: 0.0-1.0（提取时的置信度）
- is_hard: bool（是否来自用户明确纠正，优先召回）
```

**粒度边界建议：**

| 类型 | 建议粒度 | 长度约束 | 示例 |
|------|---------|---------|------|
| **Fact** | 单一事实陈述 | ≤ 50 词 | "User prefers Python over JavaScript for backend development." |
| **Experience** | 条件-动作对 | ≤ 64 词（强制） | "When handling large CSV files, use streaming read instead of loading entire file into memory." |
| **Workflow** | 一个可复用任务的最小完整流程 | 200-1000 词 | "How to deploy a Node.js app to Docker"（含步骤、 pitfalls、代码模板） |
| **Plan** | 一个具体待办事项 | ≤ 100 词 | "User plans to migrate database from PostgreSQL to MySQL before June 30." |

#### 2.3 L2 原子事实如何从 L3 提取？

##### 2.3.1 是否要包含前后上下文？

**各系统的做法：**

| 系统 | 上下文范围 | 理由 |
|------|-----------|------|
| **Mem0** | 近 10 条消息（`m=10`） | 保证提取时有足够的对话背景 |
| **AutoSkill** | `messages[-6:]`（最近 6 条） | 滑动窗口，延迟 1 轮提取 |
| **MemOS TS** | 完整任务对话（主题分割后的全部 chunks） | 任务驱动，保证上下文完整 |
| **MemOS Py** | 最近 20 条消息 + `chat_history` 辅助 | 预切分 10 条消息一组，重叠 2 条 |
| **XSkill** | N=4 条 rollout 的完整轨迹 + 图像 | 跨 Rollout 对比需要全局上下文 |
| **ReMe** | 成功/失败/对比 trajectory | 按 trajectory 级别提取 |

**建议：**
L2 提取**必须包含上下文**，但上下文的长度应该**因类型而异**：

```
Fact 提取：包含当前 turn 及前后各 2 个 turn（共 5 个 turn）
Experience 提取：包含最近 10 个 turn 或当前任务的全部 chunks
Workflow 提取：包含整个任务的完整对话（可能 20-50 个 turn）
Plan 提取：包含当前 turn 及前后各 1 个 turn
```

**关键原则（来自 AutoSkill 的 Evidence & Provenance）：**
- **USER 消息 = 主要证据来源**
- **ASSISTANT 消息 = 仅参考上下文**
- **Tool 结果 = 程序记忆的辅助证据**

这意味着在提取 Fact 时，应该优先从 user 消息中提取；在提取 Experience/Workflow 时，需要结合 tool 调用结果来判断成功/失败模式。

##### 2.3.2 原子事实按照什么原则提取？粒度/范围/边界如何确定？

**各系统的提取原则：**

| 系统 | 提取触发原则 | 边界确定方式 |
|------|------------|-------------|
| **MemOS TS** | 任务完成时（`finalizeTask`）触发 | 主题分割算法决定任务边界（时间间隔>2h 或 LLM 判断主题变化） |
| **AutoSkill** | 每轮滑动窗口右移 2 条消息时触发 | 固定窗口大小：6 条消息 |
| **XSkill** | 每个训练样本的 N 次 rollout 完成后触发 | 一次完整任务（question + ground_truth + 轨迹） |
| **ReMe** | 按 trajectory（成功/失败/对比）提取 | 按 score 阈值分类 trajectory |
| **Hermes** | 无独立提取管线，靠 LLM 自主判断 | LLM 自己决定何时用 `memory` / `skill_manage` 工具 |

**边界确定是最大难点。**

MemOS TS 的**主题分割**设计非常精巧：
1. 未分配的 chunks 按 user turn 分组
2. 逐 turn 处理，检查时间间隔（>2h → 强制切分）
3. LLM 判断当前消息是否与当前任务主题一致（`SAME` / `NEW`）
4. 低置信度时进行二次仲裁

**建议 gspd 采用混合边界策略：**

```
Step 1: 硬边界（零 LLM）
  - 会话切换 → 强制切分
  - 时间间隔 > 2h → 强制切分
  - 系统提示/工具列表发生显著变化 → 强制切分

Step 2: 软边界（小模型 LLM）
  - 构建当前任务状态（第一条 user 消息摘要 + 最近 3 轮对话）
  - LLM 判断：新消息是"子问题/跟进"还是"全新任务"
  - 输出：{"same_topic": true/false, "confidence": 0.0-1.0}

Step 3: 仲裁（仅当 confidence 低时触发）
  - 人类规则 override：工具变化、领域跳跃、目标变化 → 切分
```

##### 2.3.3 同一个原子事实是否会归到多个分类？

**答案是：会，而且很常见。**

例如：
- "用户纠正 Agent：处理图片时应该先用 histogram equalization" → 这既是 Experience（具体技巧），也是 Fact（用户的偏好/要求），也可以视为 Workflow 的一部分（如果用户在教一个完整流程）。

**各系统的处理方式：**

| 系统 | 处理方式 |
|------|---------|
| **Mem0** | 程序记忆单独路径，其他记忆底层统一存储。没有跨类去重机制。 |
| **MemOS Py** | `SkillMemory` 同时包含 `procedure`（流程）、`experience`（经验）、`preference`（偏好）三个字段，**不拆成独立记录**，而是存为一个复合文档。 |
| **XSkill** | Experience 和 Skill **完全分离存储**，Experience 是独立三元组列表，Skill 是独立 Markdown 文档。不会重叠。 |
| **AutoSkill** | 单一 SKILL.md，所有内容都写在一个文档里。 |
| **ReMe** | Task Memory 和 Tool Memory 分离，Personal Memory 单独存储。 |

**对 gspd 的建议：**

如果 gspd 坚持 L2 六分类，**必须建立交叉引用机制**，否则同一洞察会被重复提取 2-3 次，导致检索时噪声和上下文膨胀。

更推荐的做法是：**降低分类的"排他性"，采用主标签 + 副标签机制**：

```json
{
  "content": "When handling dark images, use histogram equalization first.",
  "primary_kind": "experience",
  "secondary_kinds": ["fact", "workflow"],
  "linked_ids": ["fact_xxx", "workflow_yyy"]
}
```

或者更进一步，像 MemOS Py 的 `SkillMemory` 一样，**不把 L2 当作独立原子，而是作为复合知识包**：一个 L2 条目可以同时包含多个 section（facts / experiences / preferences），每个 section 独立嵌入、统一检索。

##### 2.3.4 抽取 L2 的时机如何确定？

**各系统的提取时机：**

| 系统 | 提取时机 | 优缺点 |
|------|---------|--------|
| **MemOS TS** | `agent_end` hook，游标增量处理 + 任务 finalize 时触发 Skill 提取 | 实时性好，但 task finalize 有延迟 |
| **AutoSkill** | `agent_end` hook，滑动窗口延迟 1 轮提取 | 每轮都提取，渐进进化，但相邻窗口重叠导致重复提取 |
| **Mem0** | 显式调用 `memory.add()` | 最灵活，但用户/Agent 可能忘记调用 |
| **XSkill** | 离线批处理（训练框架） | 不适用于在线服务 |
| **Hermes** | 在线：LLM 自主判断；离线：Gateway Flush / Cron | 最轻量，但质量完全依赖模型能力 |
| **ReMe** | 批量 trajectory 处理（非每轮实时） | 适合后处理，不适合实时反馈 |

**对 gspd 的建议：**

采用**多时机触发**的混合策略：

```
时机 1: 会话进行中（实时轻量）
  - 每个 user turn 结束后，小模型快速提取 Fact/Experience 候选
  - 不立即写入 L2 主库，先放入候选池（candidate pool）

时机 2: 任务结束时（深度提取）
  - 主题分割判定任务 finalize 后
  - 大模型从整个任务对话中提取 Workflow / 复杂 Experience
  - 同时聚合候选池中的事实，做去重和升级

时机 3: 离线批处理（演进优化）
  - 每日 Cron（如凌晨 3 点）
  - 对候选池中 support_score 达标的候选升格为 active
  - 对已有 L2 做冗余合并、description 优化、质量演进
```

这与你的"异步提取"设计完全兼容，只是需要明确**三层时间粒度**。

##### 2.3.5 L2 抽取开销：Block 限制大小？上下文取多少？需要多大的模型？

**各系统的开销数据：**

| 系统 | Block/窗口大小 | 上下文长度 | 提取模型 | 单次提取 LLM 调用数 |
|------|---------------|-----------|---------|-------------------|
| **Mem0** | 近 10 条消息 | 对话原文 | GPT-4o-mini | 2 次（提取事实 + 更新决策）|
| **MemOS TS** | 完整任务（通常 10-50 条消息） | 任务摘要 Prompt 输入截断到 12000 字符 | 默认模型（可配） | Skill 路径：≈ 3N + 8 次（N=消息条数）|
| **AutoSkill** | 6 条消息 | messages[-6:] | 可配 | 2-3 次（提取 + decide + merge）|
| **XSkill** | N=4 rollouts × 20 turns | 完整轨迹 + 图像 | MLLMkb（知识模型，通常 GPT-4o 级别） | 每个样本 ≈ 8-10 次 |
| **ReMe** | trajectory 级别 | 完整 trajectory | 可配 | 3 条并行路径（成功/失败/对比）+ 验证 + 去重 |
| **Hermes** | 无独立管线 | 当前会话 | 执行模型自身 | 1 次（LLM 自主调用工具）|

**对 gspd 的关键建议：**

**Block 限制大小：**
- 传给 L2 提取模型的输入应该有**硬截断**，否则长对话会导致成本和延迟失控。
- **MemOS 的截断策略值得直接借鉴**：
  - 普通消息：完整保留
  - Tool 结果（成功）：截断到 1500 字符（60% 头 + 30% 尾）
  - Tool 结果（失败/报错）：**完整保留**
  - 代码块：保留前 10 行 + `[截断]`

**上下文取多少？**

```
实时轻量提取（Fact/Experience 候选）：
  → 最多 10 个 turn（约 4000-8000 tokens）
  → 使用小模型（如 GPT-4o-mini / Qwen2.5-7B）

depth 提取（Workflow/复杂 Experience）：
  → 最多 50 个 turn 或 15000 tokens
  → 使用大模型（如 GPT-4o / Claude Sonnet）
  → 超长任务先做压缩摘要，再喂给提取模型
```

**模型分配建议（借鉴各系统的分层模型策略）：**

| 任务 | 推荐模型 | 理由 |
|------|---------|------|
| 主题分割判断 | 小模型 | 二分类任务，结构简单 |
| Fact 提取 | 小模型 | 事实抽取是成熟能力 |
| Experience 提取 | 小模型 | 条件-动作对生成 |
| Workflow 提取 | 大模型 | 需要理解长程依赖和工具编排 |
| 去重/合并决策 | 小模型 | 结构化输出，有明确规则 |
| 语义合并（merge） | 大模型 | 需要高质量文本融合 |
| 离线演进（GEPA） | 大模型 | 需要分析失败模式并提出泛化修复 |

---

### 3. L1 渐进加载摘要层

#### 3.1 L1 如何确认任务/主题等分类？分类原则是什么？

**你的设计中 L1 是"按记忆类别或任务主题抽取的摘要"，用于渐进加载。**

**各系统的相关设计：**

| 系统 | 摘要/分类策略 | 分类原则 | 工程效果 |
|------|-------------|---------|---------|
| **MemOS TS** | `task-processor.ts` 中的主题分割 → 任务摘要 | LLM 判断：当前消息是否继续当前任务 | 线性处理，不支持非连续回溯 |
| **MemOS Py** | `TASK_CHUNKING_PROMPT`：一次性 LLM 聚类 | 任务独立性 + 主任务子任务不分离 + 非连续回溯 | 最灵活，但最依赖模型能力 |
| **AutoSkill** | 无 L1 层，直接 messages[-6:] → Skill | 滑动窗口即"分类" | 简单但缺少任务级全局视角 |
| **XSkill** | `Task Decomposition`：将测试任务分解为 2-3 个抽象子任务 | 每个子任务生成检索查询 | 用于检索 Experience，不是摘要 |
| **ReMe** | 按 `tags` + `when_to_use` 分类 | 提取时由 LLM 自动生成 tags | 检索时按向量匹配 `when_to_use` |
| **Hermes** | 无显式分类，MEMORY.md 是平文本 | 靠 LLM 自主管理 | 无系统分类 |

**Hermes `ContextCompressor` 的源码级压缩算法（`agent/context_compressor.py`）：**

Hermes 的 L1 等价物就是上下文压缩产生的 "context checkpoint"。其算法非常工程化，值得 gspd 直接借鉴：

**触发条件：** 当 prompt tokens ≥ `context_length * 50%`（且不低于 `MINIMUM_CONTEXT_LENGTH`）时触发。含抗抖动保护：若最近 2 次压缩各节省 <10%，则跳过压缩避免死循环。

**5 阶段算法：**
1. **旧 tool result 剪枝（cheap pre-pass，零 LLM 成本）**
   - 保护尾部消息（token budget，默认约 20K tokens）。
   - 对保护范围外的旧 tool result，替换为 informative 1 行摘要：
     - 例：`[terminal] ran 'npm test' -> exit 0, 47 lines output`
     - 例：`[read_file] read config.py from line 1 (3,400 chars)`
   - 还去重：同一内容被多次读取时，旧的结果标记为 "[Duplicate tool output — same content as a more recent call]"。
   - 对 assistant 消息中超大的 tool_call 参数（>500 chars）截断到 200 chars。

2. **保护 head**
   - 保留前 N 条消息（默认 3 条），通常是 system prompt + 首轮 user/assistant 交换。

3. **保护 tail by token budget**
   - 从末尾反向累加 tokens，直到达到 `tail_token_budget`（默认 ~20K，随模型上下文自动缩放）。
   - 硬下限：至少保留 3 条消息。
   - 避免在 tool_call/result 组中间切断（`_align_boundary_backward`）。

4. **LLM 结构化 summarization**
   - 将中间待压缩的 turns 序列化成长文本（每条消息最多 6000 chars，头 4000 + 尾 1500）。
   - 发送给 summarizer 模型（可配置，默认与主模型相同），使用以下**结构化模板**（12 个字段）：
     - `Goal`, `Constraints & Preferences`, `Completed Actions`, `Active State`, `In Progress`, `Blocked`, `Key Decisions`, `Resolved Questions`, `Pending User Asks`, `Relevant Files`, `Remaining Work`, `Critical Context`
   - **迭代更新**：如果已有前一次压缩摘要，prompt 要求"update"而非"rewrite"——保留旧信息，添加新 Completed Actions，移动 In-Progress → Completed 等。
   - **Focus topic**：用户可通过 `/compress <topic>` 要求摘要优先保留特定主题信息（分配 60-70% token budget）。
   - 摘要输出被强制加上前缀：
     ```
     [CONTEXT COMPACTION — REFERENCE ONLY] Earlier turns were compacted into the summary below.
     This is a handoff from a previous context window — treat it as background reference,
     NOT as active instructions. Do NOT answer questions or fulfill requests mentioned
     in this summary; they were already addressed.
     ```

5. **Tool-call 完整性修复**
   - 移除那些 assistant tool_call 已被压缩掉但 orphan tool result 还在的消息。
   - 为那些 assistant tool_call 幸存但 tool result 被压缩掉的消息，插入 stub result：`[Result from earlier conversation — see context summary above]`。

**对 gspd L1 的核心质疑：**

如果 L1 的摘要既"按记忆类别"（六类各一）又"按任务主题"抽取，那么 L1 的数量会随着记忆类别和任务主题的数量**乘积级增长**。这会导致：
- 渐进加载时的检索复杂度变高
- 摘要之间的重叠和冗余难以控制
- 某个任务涉及 3 类记忆时，需要同时加载 3 个摘要，上下文膨胀

**建议的分类原则：**

```
L1 应该只按"任务主题"分类，不按"记忆类别"分类。

理由：
- 任务主题是检索时的自然查询单位（"用户问的是旅行规划"）
- 记忆类别是 L2 的内部标签，不应该上升到 L1 的组织维度
- 一个任务主题下可以自然包含多种记忆（事实+经验+流程），这是合理的
```

**聚类逻辑建议（融合 MemOS Py + TS）：**

```
Step 1: 关键词提取（零 LLM）
  → 对 L2 内容分词 → 去停用词 → 取 top-N TF 词

Step 2: Jaccard 快速门控
  → 两个 L2 的关键词重叠率 > 0.6 → 高置信同主题，直接聚类
  → 重叠率 ≤ 0.3 → 高置信不同主题，直接分开
  → 0.3-0.6 → 进入 Step 3

Step 3: LLM 语义判断
  → 小模型判断："这两个记忆是否属于同一任务主题？"
  → 输出：same_topic + reason
```

#### 3.2 聚类有什么逻辑？聚类 L1 内容的数量、大小、范围、维度等？

**各系统的聚类/摘要控制策略：**

| 系统 | 聚类结果控制 | 大小限制 | 范围 |
|------|------------|---------|------|
| **MemOS TS** | 任务 finalize 时生成一个任务摘要 | 30-50% 原始对话长度 | 单任务 |
| **XSkill** | 跨 Rollout 对比批判后产出 0-3 条 Experience | ≤64 词/条 | 跨 4 次 rollout |
| **ReMe** | 向量余弦相似度 > 0.5 视为同一类 | 自然语言段落 | 全局库 |
| **Hermes** | 无自动聚类 | MEMORY.md 2200 字符 | 全局 |
| **Mem0** | 向量相似度 top-5 + LLM 更新决策 | 单条事实 | 全局 |

**对 gspd L1 的建议：**

L1 作为"渐进加载摘要层"，其核心指标是：**被加载时，能在多少 tokens 内覆盖该主题下最有价值的知识**。

```
L1 聚类控制参数：
- max_clusters: 50（最多 50 个主题摘要，超出时合并低频主题）
- max_l1_tokens: 500（单个 L1 摘要不超过 500 tokens）
- min_l2_items: 3（一个主题下至少 3 条 L2 才生成 L1）
- max_l2_items: 20（一个 L1 最多聚合 20 条 L2，超出时拆分或升级 L1）
- l1_refresh_interval: 每次新增/更新 5 条 L2 后，重新生成该主题的 L1
```

#### 3.3 记忆的 dreaming、合并、消歧在哪层维护？如何处理？

**这是所有记忆系统最复杂的问题。**

**各系统的演进层设计：**

| 系统 | dreaming/合并/消歧位置 | 处理方式 | 触发时机 |
|------|----------------------|---------|---------|
| **Hermes** | **L2/L1 边界模糊的多层离线机制** | Gateway Flush（会话结束审查）→ Cron Reflect（知识图谱跨记忆综合）→ GEPA（Skill 进化） | 空闲/凌晨/Cron |
| **MemOS** | **L2 层**（SkillEvolver 在线 merge + 离线 dreaming） | 在线：add/merge/discard；离线：description 优化 + GEPA 内容演进 + 冗余合并 | agent_end / Cron |
| **AutoSkill** | **L2 层**（SkillEvo） | replay + 评估 + 变异 + 晋升 | 后台批处理 |
| **XSkill** | **Experience 层和 Skill 层分别合并** | Experience：嵌入相似度 ≥0.70 → LLM 合并，超 120 条强制合并；Skill：LLM 合并 + 精炼 | 每批 8 个样本后 |
| **Mem0** | **L2 层**（普通记忆有 ADD/UPDATE/DELETE 决策） | 向量 top-5 + LLM 决策 | 每次 add() 调用时 |
| **ReMe** | **L2 层**（MemoryDeduplicationOp + DeleteMemoryOp） | 向量相似度去重 + 效用老化删除 | 提取后 / 定期维护 |
| **GEPA** | **L1/L2 层**（Prompt 作为可优化参数） | 分析执行轨迹 → 自然语言反思 → 语义变异 → Pareto 选择 | 离线批处理 |

**对 gspd 的建议：**

Dreaming、合并、消歧**主要应该在 L2 层维护**，但 L1 层需要配合做**刷新**。

```
维护层级分工：

L2 层（核心维护层）：
├── 合并（Consolidation）
│   触发：新 L2 入库时、每日 Cron 时
│   机制：向量相似度门控 → LLM 语义合并
│   产物：合并后的高质量 L2 + inactive 旧版本归档
├── 消歧（Disambiguation）
│   触发：检测到矛盾记忆时（如"用户喜欢 A" vs "用户讨厌 A"）
│   机制：时间戳优先 + 用户纠正信号优先 + LLM 仲裁
│   产物：标记 deprecated 的旧记忆 + 更新后的新记忆
└── Dreaming（跨记忆归纳）
    触发：每日 Cron
    机制：遍历知识库 → 发现跨记忆的共同模式 → 生成更高层抽象
    产物：新的 Workflow / 更新的 L1 摘要

L1 层（配合刷新层）：
├── 当 L2 发生合并/消歧后，重新生成受影响主题的 L1 摘要
└── 当 Dreaming 产生新的高层抽象时，更新对应的 L1

L3 层（审计基底层）：
└── 不参与合并/消歧，只做原始记录保留。当 L2 出现争议时，回溯 L3 做 ground truth。
```

**Hermes 的 Dreaming 与离线演进（源码级解析）：**

Hermes 没有单一的 "dreaming" 模块，而是构建了一个**从简单到复杂的离线演进谱系**：

**1. Gateway Memory Flush（`gateway/run.py`）——最轻量的自动 dreaming**
- **触发时机**：会话不活动超时、每日凌晨 4 AM 重置、会话显式退出。
- **实现**：启动一个临时 `AIAgent(max_iterations=8, quiet_mode=True, skip_memory=True, enabled_toolsets=["memory", "skills"])`。
- **关键防覆盖机制**：Flush agent 会读取 MEMORY.md 和 USER.md 的**实时磁盘状态**并注入 prompt，确保不会因为多 session/cron 的并发而覆盖更新的记忆。
- **Flush Prompt 原文**（从源码提取）：
  > "[System: This session is about to be automatically reset due to inactivity or a scheduled daily reset. The conversation context will be cleared after this turn.\n\nReview the conversation above and:\n1. Save any important facts, preferences, or decisions to memory (user profile or your notes) that would be useful in future sessions.\n2. If you discovered a reusable workflow or solved a non-trivial problem, consider saving it as a skill.\n3. If nothing is worth saving, that's fine — just skip.\n\nIMPORTANT — here is the current live state of memory. Other sessions, cron jobs, or the user may have updated it since this conversation ended. Do NOT overwrite or remove entries unless the conversation above reveals something that genuinely supersedes them. Only add new information that is not already captured below.\n\nDo NOT respond to the user. Just use the memory and skill_manage tools if needed, then stop.]"
- **本质**：这只是一个"用 LLM 看完整对话并决定要不要保存"的简单批处理。没有结构化的合并算法、没有去重消歧流水线。

**2. Cron Reflect / Hindsight Reflect（外部 Provider）**
- 当配置了 Hindsight 插件时，可以调用 `hindsight_reflect(query)` 进行**跨记忆综合推理**（knowledge graph traversal + LLM synthesis）。
- 这是 Hermes 最接近真正 dreaming 的primitive，但它依赖外部服务。

**3. GEPA 离线 Skill 进化（`evolution/skills/evolve_skill.py`）——重 artillery**
- 位于独立的 `hermes-agent-self-evolution` 仓库，**不会自动触发**，需要手动或 cron 调用。
- **完整 Pipeline：**
  1. **Dataset build**：`synthetic`（从 skill 文本生成）、`golden`（人工数据集）、`sessiondb`（从会话历史挖掘）。
  2. **Baseline evaluation**：baseline skill module 在数据集上跑分。
  3. **GEPA optimization**：`dspy.GEPA(metric=skill_fitness_metric, max_steps=iterations)` 编译优化 skill module。
  4. **Constraint validation**：
     - Size limit（`max_skill_size`）
     - Growth limit（默认 ≤ baseline + 20%）
     - 非空 + 结构完整性（YAML frontmatter 含 `name` + `description`）
     - 可选：pytest 100% pass
  5. **Holdout evaluation**：baseline vs evolved 在 holdout 上对比。
  6. **部署**：**永不自动 commit**。输出保存到 `output/<skill>/<timestamp>/`，需要人类 review diff 后手动合并。
- **Fitness 评分**（`evolution/core/fitness.py`）：
  - LLM-as-judge 打三个维度：`correctness`（0.5 权重）、`procedure_following`（0.3 权重）、`conciseness`（0.2 权重）。
  - 加上 `length_penalty`（若长度超过 max size 的 90%，从 0 线性增长到 0.3）。
  - 快速启发式 proxy：keyword overlap。

**对 gspd 的启示：**
- Gateway Flush 式的"LLM 审查对话"非常容易实现，应该作为第一阶段 offline pipeline。
- GEPA 式的执行轨迹驱动进化非常强大，但成本极高（$2-10/次优化），且需要评估数据集和约束门。应该作为第三阶段。
- 真正的**结构化 dreaming**（自动矛盾检测、冗余合并、能力身份判断）在 Hermes 中是不存在的，这正是 gspd 可以做出差异化的地方。

**触发时机建议：**

```
在线（实时）：
  → 新 L2 入库时做轻量去重（相似度 > 0.85 直接 merge）
  → 候选池内部互相比对，防止批内重复

离线（每日 Cron）：
  → Step 1: 健康审计（扫描 stale、draft、active）
  → Step 2: 冗余合并（相似度 > 0.82 的候选对做能力身份判断 + LLM 合并）
  → Step 3: 内容质量演进（GEPA 式：对低质量 Skill 做 batch_runner + LLM-as-judge + 变异）
  → Step 4: Description 触发优化（借鉴 Skill-Creator：生成评估集 → 测试触发率 → 改写 description）
  → Step 5: L1 摘要刷新（所有受影响的主题重新生成摘要）
```

#### 3.4 渐进加载如何关联数据？怎么判断是否需要深入到下一层？

**各系统的渐进/召回策略：**

| 系统 | 渐进策略 | 深入下一层的判断 | 想解决的问题 |
|------|---------|-----------------|-------------|
| **Anthropic Skill-Creator** | L0 description 常驻 → L1 SKILL.md 触发后加载 → L2 scripts 按需 | 由 LLM 根据 description 判断是否触发 | 防止上下文被大 Skill 文档塞满 |
| **MemOS** | description 常驻（~100 词）→ SKILL.md body 触发后加载（<400 行）→ scripts/refs 按需 | Agent 调用 `skill_get` 获取完整内容 | 渐进式披露 |
| **Hermes** | L0 description 常驻 → L1 SKILL.md 按需 → L2 scripts 按需 | Agent 自动触发或用户 `/skill-name` 调用 | 三级加载 |
| **XSkill** | 测试时先 Task Decomposition → Experience 按子任务检索 top-3 → Skill Adaptation 后注入 | 子任务查询与 Experience 的语义相似度 | 防止无关经验干扰 |
| **ReMe** | `when_to_use` 向量匹配 → 检索后 LLM 重排 → RewriteMemory 定制注入 | 当前 query 与 `when_to_use` 的向量相似度 | 经验建议的动态适配 |
| **gspd design-v3-recall** | E+TP 分类目录常驻 + agent 按需 `memory_recall`；Workflow 三种模式（主动召回/常驻热门/自动匹配） | agent 自主判断或系统自动匹配 | 渐进加载 |

**对 gspd 渐进加载的建议：**

你的 design-v3-recall 已经有一个相当完善的渐进加载设计了。关键是**把"是否需要深入"的决策权交给谁**：

**方案 A：Agent 自主决策（模式 A）**
- L0 只注入目录/提示
- Agent 遇到复杂任务时主动调用 `workflow_recall` / `memory_recall`
- **优点**：零噪声，最灵活
- **缺点**：Agent 可能"意识不到"应该搜索（under-recall）

**方案 B：系统自动匹配 + Agent 过滤（模式 C）**
- 每次用户消息时，系统后台搜索匹配的 L1/L2，自动注入 top-k
- Agent 收到后自行判断是否使用
- **优点**：召回率高
- **缺点**：可能注入噪声，增加上下文压力

**方案 C：混合（推荐）**
```
L0：固定加载（零 LLM）
  → 注入：高频 Experience 分类目录 + 热门 Workflow 的 description
  
用户新消息到来时：
  → 小模型改写 query（如果需要）
  → 混合搜索 L1（主题摘要）和 L2（原子事实）
  → 注入 top-3 最相关的 L1 摘要到 Context
  
Agent 执行中：
  → 如果 Agent 主动调用 memory_recall / workflow_recall → 深入返回具体 L2/L1 完整内容
  → 如果当前 tool_use 的上下文与某个 L2 高度匹配 → 系统可主动推送（预留，迭代二）
```

**判断是否深入下一层的规则：**

```
搜索 L1 摘要的相似度 score：
- score < 0.35 → 不加载任何 L1
- 0.35 ≤ score < 0.60 → 加载 L1 摘要（简短提示）
- score ≥ 0.60 → 除了加载 L1 摘要，还加载该主题下最相关的 top-3 L2 原子事实
- score ≥ 0.80 → 如果是 Workflow 类型，加载完整 SKILL.md
```

---

### 4. L0 固定加载层

#### 4.1 固定加载的内容如何确定？需要哪些？总的固定加载的 token 量定多少合适？

**各系统的 L0 等价物：**

| 系统 | 固定加载内容 | Token 量 | 设计意图 |
|------|------------|---------|---------|
| **Hermes** | `MEMORY.md` + `USER.md` 的冻结快照 | ~800 + ~500 = ~1300 tokens | 跨会话持久事实 + 用户画像 |
| **Anthropic Skill-Creator** | Skill frontmatter 的 `description` | ~100 词/Skill | 决定 Agent 是否触发该 Skill |
| **MemOS** | `description`（~100 词）+ 安装状态的 Skill 索引 | 取决于 Skill 数量，通常 500-1500 tokens | 渐进式披露的第一层 |
| **ReMe** | `MEMORY.md`（长期记忆日志）+ experience 分类索引 | 文件系统版有人类可读的上限，向量版无固定加载 | 跨会话经验 |
| **gspd design-v3-recall** | E+TP 分类目录 + 热门 Workflow description | 30 行目录 + top-10 描述 | Agent 按需召回 |

**对 gspd L0 的建议：**

L0 固定加载的内容应该**极度精简**，因为它的目标是：在不占用太多上下文的前提下，让 Agent"知道有什么可用"。

```
L0 内容清单：
1. Agent 核心身份和约束（系统提示本身）
2. 用户画像摘要（5-10 条最关键的事实）
3. Experience / Tool Preference 分类目录（表格形式，≤30 行）
4. 热门 Workflow 的 description 列表（top-10，每项 1 行名称+1 行描述）
5. 渐进加载的调用指引（"如果你需要某个主题的详细经验，调用 memory_recall"）
```

**Token 预算建议：**

```
总 L0 token 量应控制在 1500-2500 tokens 之间。

细分：
- 系统提示核心：500-800 tokens
- 用户画像摘要：200-400 tokens
- Experience 分类目录：300-600 tokens
- Workflow 热门列表：300-600 tokens
- 调用指引：100-200 tokens
```

**为什么不能超过 2500 tokens？**
- 如果 L0 太大，留给当前对话和检索结果的上下文空间就小了
- LLM 的 prefix cache 在 L0 变化时会失效（如果 L0 动态更新的话）
- Agent 容易"淹没"在固定信息中，忽略当前用户消息

**固定加载内容的筛选策略：**

```
用户画像摘要：
  → 按 recency + importance + usage_count 排序
  → 只保留 top-5 最关键的个人事实
  → 每月未使用的事实自动降级

Experience 分类目录：
  → 只显示有 ≥1 条 active experience 的分类
  → 按 usage_count 降序排列
  → 超过 30 行时截断低频分类

Workflow 热门列表：
  → 按 usage_count 降序取 top-10
  → 如果总描述超过 600 tokens，只保留前 5-7 个
```

**Hermes L0 的源码级实现：**

Hermes 的 L0 固定加载由 `agent/prompt_builder.py` 中的 `_build_system_prompt()` 组装，包含两个核心部分：

**A. Prompt Memory（`tools/memory_tool.py`）**
- `MEMORY.md`：agent 的个人笔记，上限 **2,200 字符**。
- `USER.md`：用户画像，上限 **1,375 字符**。
- 分隔符：`\n§\n`（section sign）。支持多行条目。
- 在系统提示中渲染为：
  ```
  ══════════════════════════════════════════════
  MEMORY (your personal notes) [45% — 990/2,200 chars]
  ══════════════════════════════════════════════
  <entries separated by §>
  ```
- **Frozen Snapshot**：`MemoryStore.load_from_disk()` 捕获 `_system_prompt_snapshot`，之后 mid-session 的写入**不更新**系统提示中的这部分内容。

**B. Skills Index（`agent/prompt_builder.py` `build_skills_system_prompt()`）**
- 扫描 `~/.hermes/skills/` 下的所有 `SKILL.md`，读取 YAML frontmatter 中的 `name` 和 `description`。
- 构建紧凑索引（按 category 分组），格式：
  ```
  ## Skills (mandatory)
  <available_skills>
    devops:
      - docker-deploy: How to containerize Node.js apps
    coding:
      - refactor-loop: Guidelines for simplifying nested loops
  </available_skills>
  ```
- **性能优化**：双层缓存
  - Layer 1：in-process LRU cache（最大 8 个 entry，按 skills_dir + tools + toolsets + platform 组合键区分）。
  - Layer 2：磁盘快照 `.skills_prompt_snapshot.json`，用 mtime/size manifest 验证有效性。冷启动时无需全量扫描文件系统。
- **条件激活**：Skill frontmatter 可声明 `requires_tools`、`requires_toolsets`、`fallback_for_tools`、`fallback_for_toolsets`。不兼容当前环境的 Skill 自动从索引中隐藏。
- **外部目录**：支持 `skills.external_dirs` 配置，外部 Skill 只读、本地 Skill 优先。

**C. 工具行为指导（`agent/prompt_builder.py`）**
- 若工具集包含 `memory`，注入 `MEMORY_GUIDANCE`：
  > "Save durable facts... keep it compact... User preferences and recurring corrections matter more than procedural task details... Do NOT save task progress... use session_search to recall those."
- 若包含 `session_search`，注入 `SESSION_SEARCH_GUIDANCE`：
  > "When the user references something from a past conversation... use session_search to recall it before asking them to repeat themselves."
- 若包含 `skill_manage`，注入 `SKILLS_GUIDANCE`：
  > "After completing a complex task (5+ tool calls)... save the approach as a skill... patch it immediately when outdated."

---

## 第三部分：替代设计思路与理论支撑

### 5. 是否存在其他更好的分类、分层、或其他设计逻辑？（至少列出 3 条）

#### 思路 1：XSkill 式"双轨制"——不按内容类型分，按粒度/使用方式分

**设计：** 放弃 L2 的六分类，改为 **Experience（≤64 词条件-动作对）+ Skill（~600 词结构化 SOP）** 两层。

**理论支撑：**
- XSkill 论文消融实验：仅 Skill 错误率 15.3%，仅 Experience 错误率 29.9%，两者结合最佳。
- 这证明了**不同粒度的知识需要不同的表示和检索方式**，而不是按心理学内容类型分。

**解决的问题：**
- 消除六分类的边界模糊问题
- Experience 适合快速检索和情境微调，Skill 适合防止结构性错误
- 工程实现简单：两种存储格式、两种注入模板、两种合并策略

**引入的新问题：**
- 某些对话内容可能同时适合提取为 Experience 和 Skill（例如："处理图片时先用 histogram equalization"既可以当 Experience 也可以写入 Skill 的 Pitfalls 段），需要额外的去重机制
- 对于用户画像类事实，双轨制没有专门的存储位置，需要额外补充 Fact 层

#### 思路 2：Mem0 式"通用记忆中间件"——不做分类，只做存储和检索 API

**设计：** 放弃所有分类，L2 只存"记忆文本"，由上层 Agent 自行决定如何使用。

**理论支撑：**
- Mem0 的定位是 Memory-as-a-Service，核心理念是"最小化颗粒，最大化灵活性"。
- 论文 arXiv 2504.19413：在 LOCOMO benchmark 上达到 66.9% 准确率，延迟比 Full-context 低 91%。

**解决的问题：**
- 彻底消除分类冲突和边界争议
- 提取和检索的工程实现最简单
- 上层 Agent 可以按自己需求任意组织记忆

**引入的新问题：**
- 不分类意味着无法针对不同类型的记忆优化提取 Prompt 和检索策略
- 程序记忆（SOP）和事实记忆（用户偏好）混在一起，检索时噪声更大
- 缺乏离线演进机制（ dreaming/合并），记忆库会随时间膨胀退化

#### 思路 3：Hermes 式"知识图谱 + 冻结快照"——用图结构替代分层金字塔

**设计：** 放弃 L3→L2→L1→L0 的线性分层，改为四层互补架构：
- Prompt Memory（冻结快照，2200 字符）
- Skills（文件系统，三级加载）
- Session Archive（SQLite 全文检索）
- External Providers（Hindsight 知识图谱：事实+实体+关系）

**理论支撑：**
- Hindsight 的 LongMemEval 基准最高分 91.4%。
- 知识图谱支持跨记忆综合推理（`hindsight_reflect`），这是简单向量检索无法做到的深度整理。

**解决的问题：**
- 线性分层在"跨主题关联"时非常脆弱（例如："用户喜欢 Docker"和"用户讨厌 Kubernetes"如果分在不同 L1 主题，很难被同时召回）
- 图结构的 `reflect` 可以发现矛盾、冗余、缺失，这是 dreaming 的核心能力
- 冻结快照解决了在线修改导致的 prefix cache 失效问题

**引入的新问题：**
- 知识图谱的构建和维护成本极高（需要实体抽取、关系抽取、消歧）
- 不适合作为简单 Agent 的默认记忆方案
- Hindsight 是外部服务，有额外的延迟和依赖

---

## 第四部分：分层科学性评估

### 6. 当前 4 层分层是否科学合理？N 应该是 4 还是多少层？

**评估结论：4 层的方向是对的，但具体分层定义需要调整。**

从信息论和认知科学的角度，记忆系统的确需要**分层压缩**。但你的 4 层设计存在两个问题：
1. **L2 的六分类过于复杂**，实际上把一层拆成了六层的心理学映射
2. **L1 和 L0 的功能边界不够清晰**，两者都是"摘要"，区别只在"是否固定加载"

**参考业界系统的层数：**

| 系统 | 层数 | 分层逻辑 |
|------|------|---------|
| **Anthropic Skill-Creator** | 3 层 | L0 description / L1 SKILL.md / L2 resources |
| **Hermes** | 4 层 | Prompt Memory / Skills / Session Archive / External Providers |
| **MemOS** | 3 层（Skill 视角） | description / SKILL.md body / scripts+refs |
| **XSkill** | 2 层 | Experience / Skill |
| **ReMe** | 3 层 | Personal / Task / Tool |
| **你的 gspd** | 4 层 | L3 raw / L2 atomic / L1 progressive / L0 fixed |

**推荐的调整方案：**

如果保持**4 层架构**，建议重新定义各层的职责，使其更工程化：

```
L3: 原始记录层（Raw Trace）
  → 只负责" faithfully 记录发生了什么"
  → 包含：对话消息、tool_calls、tool_results、环境变化、压缩链路

L2: 知识原子层（Knowledge Atoms）
  → 负责"从原始记录中提取可复用的最小知识单元"
  → 分类简化为：[fact | experience | workflow | plan]
  → 每个原子有独立向量、独立置信度、独立溯源

L1: 主题索引层（Topic Index）
  → 负责"按主题聚合 L2，生成检索友好的摘要"
  → 不是按记忆类别分，而是按任务主题分
  → 是渐进加载的主要载体

L0: 常驻上下文层（Resident Context）
  → 负责"每次会话必须加载的极简上下文"
  → 只包含最高频、最重要的信息索引
```

**如果必须简化层数，3 层也是可行的：**

```
L2: 原始记录（合并你的 L3）
L1: 知识原子（合并你的 L2）
L0: 常驻上下文（合并你的 L1 渐进摘要 + L0 固定加载）
```

但**4 层比 3 层更优**，因为把"原始记录"和"知识原子"分开，可以在知识原子出错时回溯到原始记录做审计和修正。这是 MemOS/Hermes 等系统都选择保留原始对话的原因。

**最终结论：N=4 是合理的，但需要重新定义 L2 的分类和 L1/L0 的边界。**

---

## 第五部分：重大设计缺陷与质疑

### 7. gspd 分层设计至少 3 个比较大的设计缺陷

#### 缺陷 1：L2 六分类的边界模糊，会导致提取冲突和检索噪声

**质疑问题：**
- 当 LLM 提取到"用户计划下周去日本旅行"时，它应该被归类为情景记忆、前瞻记忆还是语义记忆？如果三类各存一份，如何保证一致性？
- 程序记忆和通用记忆的边界在哪里？"用 Docker 部署 Node.js"是程序记忆，但"Docker 是一种容器技术"是通用记忆——Agent 如何自动判断一条信息属于哪一边？
- 六分类是否需要六套不同的提取 Prompt 和六套不同的检索策略？如果是，提取和检索的 LLM 调用成本会乘以 6。

**影响：**
- 分类歧义会导致同一条信息被重复提取到多个类别中，造成记忆库膨胀
- 检索时如果按类别过滤，可能因分类错误而漏召
- 如果不用类别过滤，六分类的设计就失去了工程价值

#### 缺陷 2：L3→L2 的提取窗口和时机没有定义，长对话会导致提取质量断崖式下降

**质疑问题：**
- 如果一次任务包含 100 个 turn，L2 提取模型是一次性读取全部 100 个 turn，还是按滑动窗口分批提取？
- 如果是全量读取，超长上下文会超出模型窗口限制，且成本极高；如果是滑动窗口，窗口之间的重叠部分会导致重复提取，窗口边界处的信息会被截断。
- 提取是在每个 turn 后实时进行，还是等任务结束后再批量进行？如果是实时，如何在线处理去重？如果是批量，Agent 在会话中如何获取刚刚产生的经验？

**影响：**
- 窗口策略不明确会导致 L2 要么遗漏关键信息，要么充斥着重复和碎片化记录
- 时机策略不明确会导致"提取滞后"或"提取过早"（在任务还没完成时就提取了不完整的经验）

#### 缺陷 3：缺乏在线/离线的质量门控、矛盾仲裁和版本回退机制

**质疑问题：**
- 当新提取的 L2 与已有 L2 矛盾时（例如旧记录说"用户喜欢辣"，新记录说"用户不吃辣"），系统如何仲裁？是按时间优先，还是按用户纠正信号优先？
- 当 L2 的合并操作导致内容缩水超过 30% 时（例如一个详细的 Workflow 被合并成了过于简略的摘要），是否有自动检测和回退机制？
- 当 Dreaming 或 GEPA 演进后的 L1/L2 质量反而下降时，系统如何恢复到旧版本？

**影响：**
- 没有矛盾仲裁会导致 Agent 的行为反复无常（这次按旧记忆回答，下次按新记忆回答）
- 没有质量门控会导致记忆库随时间退化（AutoSkill 的早期版本就有这个问题，后来被 SkillEvo 和 usage auditing 弥补）
- 没有版本回退会让离线演进成为高风险操作，不敢大规模启用

#### 缺陷 4：Hermes 式 frozen snapshot 导致"写入即遗忘"的延迟问题

**质疑问题：**
- 如果 gspd 采用 frozen L0（如 Hermes 的 `MEMORY.md` 快照），那么 Agent 在本会话中通过 `memory` 工具写入的新事实，要到**下一次会话启动**才会出现在系统提示中。
- 如果 Agent 没有养成"写完后立刻读 tool_result"的习惯，它会在同一会话的后续对话中"忘记"自己刚学到的偏好。
- 例如：用户在第 10 turn 说"记住我讨厌 YAML"，第 15 turn 说"给我写个配置文件"——如果中间没有 session 重启，Agent 可能仍按默认习惯生成 YAML。

**影响：**
- 用户体验上的"健忘感"，尤其是在长会话中
- 需要额外设计一个"live read"机制（如 tool 调用后立即把返回值注入上下文）来弥补

#### 缺陷 5：辅助模型单点故障导致压缩/检索完全失效

**质疑问题：**
- Hermes 的 `ContextCompressor` 完全依赖辅助模型（summarizer）生成摘要。如果该模型不可用（网络故障、额度耗尽、配置错误），压缩进入 10 分钟冷却期，期间只能**硬丢弃**中间消息，上下文窗口会迅速爆满。
- 同样，`session_search` 也依赖辅助模型（Gemini Flash）对检索到的会话做摘要。如果辅助模型不可用，只能回退到原始 500 字符 preview。
- gspd 的 L1 摘要层如果也采用"纯 LLM 摘要"设计，会面临同样的单点故障风险。

**影响：**
- 系统在没有备用方案时的降级路径不清晰
- 长对话在辅助模型故障时会迅速陷入上下文耗尽、行为异常的境地

---

## 第六部分：XSkill 的针对性设计深度解析

XSkill 是你特别强调关注的系统，下面针对 gspd 的每一个核心问题，分析 XSkill 是否有针对性的设计。

### XSkill 针对 L3 原始会话的设计

**1.1 L3 是否需要存储？如何存储？**
- **XSkill 的做法**：**完整存储**。每个 rollout 保存 `trajectory.jsonl` + 工具生成的中间图像 `tool_image_*.jpg` + 原始图像 `original_image*.jpg`。
- **想解决的问题**：为后续的跨 Rollout 对比批判提供完整的证据链；视觉锚定分析需要原始图像。
- **引入的问题**：存储开销大（多模态数据），仅适用于离线训练框架，不适合高频在线服务。

**1.2 Block 之间是否需要关联？**
- **XSkill 的做法**：按 `sample` → `rollout` → `turn` 三级层级关联。一次任务可能有 N=4 条 rollout，rollout 之间是平行关系。
- **想解决的问题**：支持跨 rollout 的对比分析（成功 vs 失败）。
- **引入的问题**：关联结构复杂，需要额外的对齐和汇总逻辑。

**1.3 Block 如何与 Agent Context 对应？**
- **XSkill 的做法**：训练和推理时通过 `INJECTION_HEADER` 和 `SKILL_INJECTION_HEADER` 注入系统提示。注入是**非强制性的**（"You can use it as reference... You can also have your own ideas"）。
- **想解决的问题**：防止 Agent 过度依赖先验经验，保留自由裁量权。
- **引入的问题**：Agent 可能忽略有用的经验，导致 under-utilization。

**1.4 Block 数量是否有限制？**
- **XSkill 的做法**：Experience 库硬限制 120 条，Skill 文档限制 1000 词。
- **想解决的问题**：防止知识膨胀，保证检索和注入效率。
- **引入的问题**：超限后需要强制合并，可能丢失低频但有价值的经验。

### XSkill 针对 L2 原子事实层的设计

**2.1 L2 分类是否覆盖所有记忆？**
- **XSkill 的做法**：没有心理学六分类，只有 **Experience + Skill** 两类。
- **想解决的问题**：简化分类，专注工程价值——Experience 做战术微调，Skill 做结构防错。
- **引入的问题**：用户画像类事实没有专门存储位置（XSkill 是任务级训练框架，不关注跨会话用户画像）。

**2.2 原子事实的定义、范围、粒度**
- **Experience**：`e = (c, a, v_e)`，条件+动作 ≤ 64 词，以 "When..." 开头。
- **Skill**：Markdown SOP，约 600 词，包含 Workflow/Tool Templates/Pitfalls。
- **硬约束的价值**：消融实验证明，64 词约束强制泛化，是系统性地对抗过拟合的关键设计。

**2.3 如何从 L3 提取 L2？**

*2.3.1 是否包含前后上下文？*
- **XSkill**：训练时基于完整 rollout 提取；推理时先做 Task Decomposition（2-3 个子任务），再按子任务检索。
- **上下文策略**：跨 Rollout Critique 会对比所有 N 条 rollout 的总结，上下文范围是全局的。

*2.3.2 按什么原则提取？*
- **核心原则**：多路 Rollout 对比批判（Cross-Rollout Critique）。成功 rollout 告诉你什么管用，失败 rollout 告诉你什么不管用。
- **粒度**：Experience 是行动级（action-level），Skill 是任务级（task-level）。

*2.3.3 同一个事实是否会归到多个分类？*
- **XSkill**：Experience 和 Skill 是**完全分离的存储**。同一条洞察如果适合两者，会分别写入 Experience 库和 Skill 文档，但不会在一个库内重复。

*2.3.4 提取时机？*
- **离线批处理**：每个训练样本的 N 次 rollout 完成后统一提取。不适合在线实时服务。

*2.3.5 抽取开销？*
- **模型**：MLLMkb（知识模型），通常比执行模型更强。
- **调用数**：每个样本约 8-10 次 MLLMkb 调用（图像分析 + rollout summary + critique + 合并 + skill 生成）。

### XSkill 针对 L1/L0 的设计

**L1 渐进加载摘要**
- **XSkill 的做法**：推理时没有"渐进加载摘要层"，但有类似的 `Task Decomposition` → `Experience Retrieval` → `Experience Rewrite` → `Skill Adaptation` 流程。
- **想解决的问题**：测试时根据当前任务动态检索和改写经验，而不是预先生成固定摘要。
- **与 gspd 的映射**：XSkill 的 `Experience Rewrite` 和 `Skill Adaptation` 相当于**按需生成的 L1**，而不是提前存储的 L1。这意味着 XSkill 没有离线维护 L1 的负担，但每次推理都需要额外的 LLM 调用。

**L0 固定加载**
- **XSkill 没有明确的 L0 层**。全局 Skill 文档在测试时会被 Adaptation 裁剪到 ~400 词后注入，这有点像 L1 而非 L0。
- **启示**：XSkill 证明了"动态裁剪注入"可以替代"固定加载"，但代价是每次测试任务需要 1 次额外的多模态 LLM 调用（Skill Adaptation）。

### XSkill 对 gspd 的核心启示

1. **硬约束（64 词/1000 词）是控制知识质量的最有效手段**。gspd 的 L2 应该引入类似的长度硬约束。
2. **双轨制（Experience + Skill）比六分类更工程化**。如果 gspd 坚持用六分类，必须有非常明确的分类决策规则，否则不如简化为 2-4 类。
3. **跨 Rollout 对比是提取高质量泛化知识的关键**。gspd 的 L3→L2 提取应该引入"成功 vs 失败"的对比分析（借鉴 XSkill 的 Cross-Rollout Critique 和 ReMe 的 ComparativeExtractionOp）。
4. **非强制性注入值得借鉴**。gspd 的 L0/L1 注入可以使用 "Consider applying..." 的语气，而不是强制命令。

---

## 第六部分（续）：Hermes 源码级深度解析

> 以下分析基于 `hermes-agent-main` 和 `hermes-agent-self-evolution-main` 仓库的实际源码阅读。

### Hermes 的 Dual-Track 架构哲学

Hermes 没有采用 L3→L2→L1→L0 的线性分层，而是使用**双轨**架构：

| 轨道 | 组件 | 设计哲学 |
|------|------|----------|
| **Online / In-Context** | `MEMORY.md` + `USER.md`（frozen snapshot）、Skills index + `skill_view`、External provider prefetch | 在线路径**极简、快速、缓存友好**。完全信任 LLM 自主判断何时保存记忆/Skill。 |
| **Offline / Out-of-Band** | SQLite session archive + `session_search`、Gateway memory flush、GEPA skill evolution、Hindsight sync | 把所有复杂优化**推离线**。重提取、合成、变异都在 critical path 之外。 |

这与 MemOS/AutoSkill 的"重在线 pipeline"思路完全相反。Hermes 用"即时不完美 + 离线深度优化"换取了在线的低延迟和稳定性。

### 关键源码发现 1：Session Archive 的检索-摘要流程

`tools/session_search_tool.py` 实现了 L3 的按需召回：

1. **FTS5 搜索**：在 SQLite 消息表上做全文搜索，取 top 50 条匹配。
2. **Parent 解析**：把子 session（如 delegation、压缩产生的子会话）追溯到根 parent session，避免同一活跃对话被重复召回。
3. **智能窗口截断**：对每个命中的会话，加载完整消息后做 `_truncate_around_matches()`：
   - 优先尝试完整短语匹配
   - 若无，尝试所有查询词在 200 字符内的共现窗口
   - 最后回退到单个词位置
   - 选取覆盖最多匹配位置的窗口（bias：25% before，75% after），截断到 ~100k chars
4. **并行摘要**：每个会话同时调用辅助模型，使用 Prompt：
   > "Summarize the conversation with a focus on the search topic... Include what the user asked, actions taken, key decisions, specific commands/files/URLs, and anything unresolved. Write in past tense as a factual recap."
5. **Fallback**：若 summarizer 不可用，返回原始 conversation 的 500 字符 preview，而不是静默丢弃。

### 关键源码发现 2：`memory` 工具的行为注入设计

`tools/memory_tool.py` 中的 `MEMORY_SCHEMA` 不仅是工具声明，更是一份**行为控制文档**：

- 明确优先级："User preferences and corrections > environment facts > procedural knowledge"
- 明确禁止："Do NOT save task progress, session outcomes, completed-work logs, or temporary TODO state"
- 明确分流："If you've discovered a new way to do something... save it as a skill with the skill tool"

Hermes 没有独立的提取 pipeline，而是把这些规则直接写进 tool schema description，靠 LLM 的指令遵循能力来驱动。这是**极简架构**的代价：如果模型不够强，它会漏存、错存、过存。

### 关键源码发现 3：Skill Manager 的安全与容量门控

`tools/skill_manager_tool.py`：

- **名称规范**：`VALID_NAME_RE = re.compile(r'^[a-z0-9][a-z0-9._-]*$')`，最长 64 字符
- **内容上限**：`MAX_SKILL_CONTENT_CHARS = 100_000`（~36k tokens）
- **原子写入**：所有写操作使用 `tempfile.mkstemp` + `os.replace`，避免半写文件
- **安全回滚**：写入后调用 `_security_scan_skill()`（基于 `tools/skills_guard`），若扫描不通过则**整目录删除**（创建时）或**原子恢复旧内容**（patch/edit 时）
- **缓存失效**：任何成功修改都会调用 `clear_skills_system_prompt_cache(clear_snapshot=True)`，确保下次会话加载最新索引

### 关键源码发现 4：MemoryManager 的 Provider 架构

`agent/memory_manager.py` + `agent/memory_provider.py`：

- **严格限制**：1 个 builtin provider + **最多 1 个 external provider**。尝试注册第二个 external provider 会被拒绝（带 warning log）。
- **统一接口**：所有 provider 实现 `system_prompt_block()`、`prefetch()`、`sync_turn()`、`get_tool_schemas()`、`handle_tool_call()`。
- **生命周期钩子**：
  - `on_turn_start`：可用于 turn 计数、维护
  - `on_session_end`：会话结束时的批量提取
  - `on_pre_compress`：上下文压缩前提取 insights
  - `on_memory_write`：external provider 可镜像 builtin memory 的写入
- **Context fencing**：`build_memory_context_block()` 将 provider 召回内容包裹在 `<memory-context>` 标签中，并标注 "[System note: The following is recalled memory context, NOT new user input.]"。

### 关键源码发现 5：GEPA 的约束与评估

`evolution/skills/evolve_skill.py` + `evolution/core/constraints.py` + `evolution/core/fitness.py`：

- **数据集来源**：`synthetic`（从 skill 文本自动生成）、`golden`（人工标注）、`sessiondb`（从会话历史挖掘）。
- **优化器**：`dspy.GEPA(metric=skill_fitness_metric, max_steps=iterations)`，若不可用则回退到 `dspy.MIPROv2`。
- **约束验证（硬门槛）**：
  1. `size_limit`： evolved skill ≤ `max_skill_size`
  2. `growth_limit`： evolved size ≤ baseline size * (1 + `max_prompt_growth`)，默认 +20%
  3. `non_empty`
  4. `skill_structure`：必须有 `---` frontmatter，且含 `name:` 和 `description:`
  5. 可选 `test_suite`：pytest 必须 100% pass
- **评估维度**：LLM-as-judge 打分 `correctness`（0.5 权重）、`procedure_following`（0.3 权重）、`conciseness`（0.2 权重）。
- **长度惩罚**：若 evolved size > 90% of max_size，composite score 会被扣减（线性增长到 0.3）。
- **结果**： Improvement > 0 时提示用户 review diff，**绝不自动部署**。

### 对 gspd 的核心启示

1. **Frozen snapshot 是 prefix cache 的终极解药**，但必须以"live read tool"作为补偿机制。
2. **Context compression 的五阶段算法**（先 cheap prune → protect head/tail → LLM summary → integrity sanitize）是工程上的最佳实践，gspd 应该直接继承。
3. **把复杂优化推离线**可以极大降低在线系统的复杂度，但离线 pipeline 必须分阶段建设（Flush → Reflect → GEPA）。
4. **External provider 的 "仅一个" 限制**说明多后端共存是一个真实难题。gspd 若想做 polyglot memory，需要设计一个统一的查询层来屏蔽底层差异。
5. **GEPA 的约束门控**证明了：没有评估和约束的自动演进是危险的。任何 dreaming/merge/evolve 都必须有"不过就回退"的机制。

---

## 第七部分：其他关键系统的针对性设计速查

### Mem0

| gspd 问题 | Mem0 设计 | 想解决的问题 | 引入的问题 |
|----------|----------|-------------|-----------|
| L3 存储 | 不独立存原始对话，只存提取后的记忆 | 降低存储成本和延迟 | 无法回溯修正 |
| L2 分类 | semantic/episodic/procedural，但只有 procedural 有独立路径 | 概念上覆盖全面 | 实际上后两类走同一管线，分类价值有限 |
| L2 提取 | 双阶段：提取事实 → 向量 top-5 + LLM 更新决策 | 自动化记忆管理 | 更新决策的 LLM 调用增加成本 |
| 去重/合并 | 普通记忆有 ADD/UPDATE/DELETE；程序记忆直接 ADD | 程序记忆不合并，保证历史完整性 | 程序记忆库会膨胀 |
| 检索 | 向量+图+SQLite 三层存储，search API 纯手动注入 | 灵活，不绑定上层 Agent | 上层需要自行决定注入策略 |

### ReMe

| gspd 问题 | ReMe 设计 | 想解决的问题 | 引入的问题 |
|----------|----------|-------------|-----------|
| L3 存储 | `dialog/YYYY-MM-DD.jsonl` 存原始对话 | 审计和回溯 | 文件管理成本 |
| L2 分类 | Personal / Task / Tool，非常清晰 | 工程落地简单 | 缺少 Workflow/Plan 层 |
| L2 提取 | 成功/失败/对比 三路并行 + 5 维验证 | 从失败中学习 | LLM 调用成本高 |
| 去重 | 向量余弦相似度 > 0.5 跳过 | 快速去重 | 阈值固定，可能漏掉语义相似但表述不同的记忆 |
| 检索 | `when_to_use` 向量化 + LLM 重排 + RewriteMemory | 检索语义对齐 | 每次检索需要额外的重写 LLM 调用 |
| 老化 | `utility/freq` 双维度追踪删除 | 防止记忆退化 | 需要较长的使用历史才能准确计算 |

### AutoSkill

| gspd 问题 | AutoSkill 设计 | 想解决的问题 | 引入的问题 |
|----------|---------------|-------------|-----------|
| L3 存储 | 不存原始对话，messages[-6:] 滑动窗口提取 | 低延迟实时提取 | 缺少长程上下文 |
| L2 提取 | 4000 字超长 Prompt，8 章节详细规则 | 提取质量 | Prompt 工程复杂，对模型遵循度要求高 |
| 去重 | BM25/向量 + LLM 4 维能力身份判断 + merge | 精细去重 | 每次提取后需要多次 LLM 调用 |
| 离线演进 | SkillEvo：replay + 评估 + 变异 + 晋升 | 主动改进 Skill 质量 | 需要构建评估数据集，成本高 |
| 使用审计 | retrieved→relevant→used 闭环 | 发现僵尸 Skill | 需要额外的审计 LLM 调用 |

### Hermes

| gspd 问题 | Hermes 设计 | 想解决的问题 | 引入的问题 |
|----------|----------|-------------|-----------|
| L3 存储 | SQLite + FTS5 Session Archive；`session_search` 做 FTS5 查询 → 智能窗口截断 → LLM 摘要 | 按需召回历史对话，不自动全量加载 | 依赖辅助模型；纯 FTS5 缺少语义检索能力 |
| L3-L1 压缩 | `ContextCompressor`：5 阶段算法（tool prune → protect head → token-budget tail → LLM 结构化 summary → integrity sanitize） | 在上下文满前做 lossy compression | 摘要质量依赖 summarizer 模型；模型不可用时只能硬丢弃 |
| L2 提取 | 无独立提取管线；靠 `MEMORY_SCHEMA` / `SKILL_MANAGE_SCHEMA` 的行为注入，由 LLM 自主决定何时调用工具 | 极简架构，零提取 pipeline 维护成本 | 质量完全依赖模型自觉性；可能漏存、错存、过存 |
| L0 固定加载 | `MEMORY.md`（2200 字符）+ `USER.md`（1375 字符）+ Skills index；`_system_prompt_snapshot` 冻结快照，mid-session 写入不改 in-flight prompt | 保 prefix cache 性能，降低推理成本 | 实时性差，当前会话的 memory 写入下次会话才生效 |
| Skill 管理 | `skill_manage` 工具：create/patch/edit/delete/write_file/remove_file；100K 字符上限；原子写入 + 安全扫描 + 缓存失效 | Agent 自主维护 procedural memory | 没有自动触发器，靠 Agent 自觉 |
| 离线演进 | Gateway Flush（会话结束 LLM 审查）→ Hindsight Reflect（跨记忆综合）→ GEPA（DSPy 进化算法 + 约束门控） | 从轻到重的完整离线优化谱系 | Gateway Flush 无结构化合并；GEPA 成本高且需人工 review |
| Provider 架构 | `MemoryManager`：1 builtin + 最多 1 external provider；统一 `prefetch`/`sync_turn`/`on_pre_compress` 接口 | 避免工具 schema 膨胀和 backend 冲突 | 无法同时组合 Hindsight + Mem0 等多个外部后端 |
| 知识图谱 | Hindsight 插件：云/本地嵌入模式；支持 `recall`/`reflect`/`retain`；auto-recall 每 turn 后台线程执行 | 跨记忆综合推理、实体解析 | 外部依赖，本地嵌入模式需额外 daemon 和 LLM key |

### GEPA

| gspd 问题 | GEPA 设计 | 想解决的问题 | 引入的问题 |
|----------|----------|-------------|-----------|
| L1/L2 演进 | 把 Prompt（=Skill）当作可优化参数，用进化算法 + 自然语言反思迭代 | 不动模型权重，持续改进 Agent 表现 | 依赖强反思模型（GPT-5/Claude Opus） |
| 评估驱动 | 用 batch_runner 真实执行 + LLM-as-judge 评分 | 客观验证改进效果 | 评估成本高（每次优化 ~$2-10） |
| 防过拟合 | Minibatch 快速验证 + D_pareto 完整评估 + Pareto 采样 | 保留多样性，防止陷入局部最优 | 算法复杂，理解门槛高 |

### MemOS

| gspd 问题 | MemOS 设计 | 想解决的问题 | 引入的问题 |
|----------|----------|-------------|-----------|
| L3 存储 | TS 端 `dialog/YYYY-MM-DD.jsonl` + 游标增量处理 | 原始对话保留 + 实时增量提取 | 文件管理成本 |
| L2 提取 | TS 端：任务 finalize 后提取；Py 端：全量消息一次性 LLM 聚类 | TS 实时性好，Py 端支持非连续回溯 | Py 端延迟高 |
| 主题分割 | TS：逐 turn LLM 判断 + 仲裁；Py：LLM 一次性分块 | 任务边界识别 | 都需要额外的 LLM 调用 |
| 质量门控 | TS 端：Rule filter + LLM 评估(confidence) + LLM 验证(score 0-10) | 多层次质量控制 | LLM 调用成本高 |
| L0/L1 | `description`（~100 词）常驻 + `SKILL.md` 触发后加载 + `scripts` 按需 | 渐进式披露 | description 写法对触发率影响极大 |

### Skill-Creator

| gspd 问题 | Skill-Creator 设计 | 想解决的问题 | 引入的问题 |
|----------|------------------|-------------|-----------|
| L2 提取 | 不自动提取，靠人机协作迭代 | 质量至上 | 无法规模化 |
| L2 优化 | 4 子代理并行测试 + eval-viewer 人工评审 | 用真实执行验证 Skill 效果 | 需要人工投入时间 |
| L0/L1 触发 | Description Optimization：train/test 拆分 + `claude -p` 黑盒测试触发率 | 自动化优化 frontmatter | 需要多次启动 Claude CLI，成本高 |
| 防过拟合 | 60/40 train/test + blinded history + 按 test 分数选最佳 | 经典 ML 范式用于 Prompt Engineering | 仅适用于触发率这种二值指标 |

---

## 第八部分：对 gspd 的 10 条 actionable 设计建议

1. **简化 L2 分类**：从六类简化为 `[fact | experience | workflow | plan]`，或采用 XSkill 的 `[experience | skill]` 双轨制。如果一定要保留心理学分类直觉，使用**三维标签系统**（content_type + time_orientation + scope）替代硬分类。

2. **引入硬约束**：Experience ≤ 64 词，Workflow/Skill 摘要 ≤ 500 词，完整 Workflow ≤ 1500 词。这是防止知识膨胀和过拟合的最有效手段。

3. **L3 必须存储原始对话**：使用 JSONL 格式，包含消息、tool_calls、tool_results、环境指纹和压缩关联。参考 MemOS TS 的 Chunk 结构和清洗规则。

4. **采用混合主题分割策略**：硬边界（会话切换/时间>2h/环境变化）+ 软边界（LLM 判断同主题）+ 仲裁（低置信度时二次确认）。参考 MemOS TS 的 `TaskProcessor`。

5. **L2 提取分两层时机**：实时轻量提取（每 turn 后，小模型，候选池）+ 深度提取（任务 finalize 后，大模型，Workflow/复杂 Experience）。参考 MemOS TS 的 finalize 机制和 gspd 自己的 design-v3-learning。

6. **去重和合并主要放在 L2 层**：新 L2 入库时做轻量去重（相似度 > 0.85 直接 merge），每日 Cron 做深度冗余合并和质量演进。参考 AutoSkill 的维护管线和 MemOS 的 SkillEvolver。

7. **L1 只按任务主题分类**：不按记忆类别分。一个主题下自然包含多种记忆。使用关键词 Jaccard + LLM 语义判断做聚类。

8. **L0 控制在 1500-2500 tokens**：包含系统提示核心 + 用户画像 top-5 + Experience 分类目录（≤30 行）+ 热门 Workflow 列表（top-10）。超出时截断低频项。

9. **建立版本管理和回退机制**：每次 merge/evolve 前备份旧版本到 `inactive` 状态。如果新版本内容缩水 >30% 或质量评分下降，自动回退。参考 MemOS TS 的升级回退机制和 Hermes GEPA 的约束验证。

10. **离线演进必须分阶段实施**：第一阶段先做 Gateway Flush 式的会话审查和简单合并；第二阶段加入 Description 触发优化（借鉴 Skill-Creator）；第三阶段再引入 GEPA 式的执行轨迹驱动演进。不要试图一次性做完整套 Dreaming。

---

## 第九部分：MemOS 源码级深度解析

> 以下分析基于 `MemTensor/MemOS` 源码，重点覆盖 TS 端（`apps/memos-local-openclaw/`）和 Python 端（`src/memos/`）的核心实现细节。

### MemOS 的 Dual-Implementation 特殊性

MemOS 同时维护两套**完全独立**的代码库：

| 维度 | TS 端（本地插件） | Python 端（云端服务） |
|---|---|---|
| **运行方式** | OpenClaw 进程内插件 | FastAPI 独立服务（port 8001） |
| **存储** | SQLite 本地文件 | 多后端：OSS/Local + MySQL + 图谱 DB |
| **消息处理** | 实时增量（游标机制） | 批量/异步（粗存 + 调度器精读） |
| **主题分割** | 逐 turn 线性 LLM 判断 | 一次性 LLM 聚类（支持非连续回溯） |
| **Skill 提取** | 任务 finalize 后 6+ 子模块流水线 | 两阶段：骨架提取 → 详细生成 |
| **质量门控** | Rule filter + LLM 评估(confidence) + LLM 评分(0-10) | 无独立门控 |
| **文件输出** | SKILL.md + scripts/ + references/ + evals/ | zip 包（含 SKILL.md + scripts/ + reference/） |

TS 端是**最完整、工程细节最丰富**的实现，以下以 TS 端为主线，Python 端作为补充。

---

### 1. 消息捕获与增量处理（TS 端）

#### 1.1 游标机制

文件：`apps/memos-local-openclaw/index.ts`

OpenClaw 每次 `agent_end` 给出全量消息，MemOS 用 `sessionMsgCursor` 避免重复处理：

```typescript
const sessionMsgCursor = new Map<string, number>();
// 首次：定位到最后一条 user 消息
// 之后：只取游标之后的新消息
const newMessages = allMessages.slice(cursor);
sessionMsgCursor.set(cursorKey, allMessages.length);
```

#### 1.2 消息清洗（`captureMessages`）

文件：`src/capture/index.ts`

**直接丢弃**：system 角色、哨兵回复（`NO_REPLY`）、自身工具结果（`memory_search`, `skill_search` 等，防循环）、启动样板。

**清洗规则**：
- 用户消息：去掉 OpenClaw 元数据 JSON 块、`<memory_context>` 注入块、时间戳前缀、`[message_id]` 标签
- Assistant 消息：去掉 `<think>...</think>`（DeepSeek 推理标签）、证据引用包装

**输出结构**：`{ role, content, timestamp, turnId, sessionKey, toolName?, owner }`

---

### 2. 消息存储与去重（IngestWorker）

文件：`src/ingest/worker.ts`

#### 2.1 处理流程

```
enqueue(messages)
  → 过滤临时 session
  → 逐条 ingestMessage()
    → [短路] 内容 ≤ 10 词 → 直接用原文当摘要，不调 LLM
    → LLM 生成摘要（名词短语标题，10-50 中文字 / 5-15 英文词）
    → embedder.embed([summary])
    → 两级去重
    → 写入 SQLite + 向量索引
  → taskProcessor.onChunksIngested()  // 触发主题分割
```

#### 2.2 摘要 Prompt

不是生成完整摘要，而是生成**检索友好的名词短语标题**：

> "Return exactly one noun phrase that names the topic AND its key details. Same language as input. Keep proper nouns, API/function names, specific parameters, versions, error codes. Prefer concrete topic words over generic words. No verbs unless unavoidable."

参数：`temperature: 0`，`max_tokens: 100`，`timeout: 30s`

#### 2.3 两级去重

**Level 1 — 精确去重**：`content_hash` 完全匹配 → 淘汰旧 chunk，保留新的。

**Level 2 — 语义去重**：向量相似度 > 0.80 的 top-5 候选 → LLM 判断：
- `DUPLICATE`：核心信息相同，即使措辞不同
- `UPDATE`：新信息补充旧信息 → 返回 `mergedSummary`
- `NEW`：完全不同的话题

参数：`temperature: 0`，`max_tokens: 300`，`timeout: 15s`

#### 2.4 Chunk 数据结构

```typescript
interface Chunk {
  id: string;
  sessionKey: string;
  turnId: string;
  seq: number;
  role: "user" | "assistant" | "tool";
  content: string;         // 原始内容
  summary: string;         // LLM 生成的标题
  embedding: number[];
  taskId: string | null;
  skillId: string | null;
  dedupStatus: "active" | "duplicate" | "merged";
  mergeCount: number;
  mergeHistory: string;    // JSON 合并历史
  createdAt: number;
  updatedAt: number;
}
```

---

### 3. 主题分割与任务 Finalize（TaskProcessor）

文件：`src/ingest/task-processor.ts`

#### 3.1 触发新任务的三种条件

| 条件 | 判断方式 | 阈值 |
|------|----------|------|
| Session 变了 | 硬切分 | — |
| 时间间隔超限 | 硬切分 | > 2h |
| 主题变了 | LLM 判断 | confidence ≥ 0.65 |

#### 3.2 主题分类 Prompt

> "Classify if NEW MESSAGE continues current task or starts an unrelated one. Output ONLY JSON: {\"d\":\"S\"|\"N\",\"c\":0.0-1.0}. d=S(same) or N(new). c=confidence. Default S. Only N if completely unrelated domain. Sub-questions, tools, methods, details of current topic = S."

参数：`temperature: 0`，`max_tokens: 60`，`timeout: 15s`

**仲裁 Prompt**（当 NEW 但 confidence < 0.65 时）：
> "A classifier flagged this message as possibly new topic (low confidence). Is it truly UNRELATED, or a sub-question/follow-up? Tools/methods/details of current task = SAME. Shared entity/theme = SAME. Entirely different domain = NEW. Reply one word: NEW or SAME"

#### 3.3 任务 Finalize 与摘要

主题切换时触发 `finalizeTask(task)`：

**跳过条件**（任一满足则不提取 Skill）：
1. chunks < 4
2. 真实对话轮数 < 2
3. 无 user 消息
4. 总内容 < 200 字符
5. 闲聊/测试（hello、test、ok 等）
6. 全是 tool 结果
7. 高重复率（调试循环）

**任务摘要 Prompt**：
> "You create a DETAILED task summary from a multi-turn conversation. This summary will be the ONLY record of this conversation, so it must preserve ALL important information."
>
> 输出结构：`📌 Title`、`🎯 Goal`、`📋 Key Steps`（保留实际代码/命令，最多 ~30 行）、`✅ Result`、`💡 Key Details`
>
> 规则：PRESERVE verbatim: code, commands, URLs, file paths, error messages. DISCARD only: greetings, filler. Target length: 30-50% of original conversation.

参数：`temperature: 0.1`，`max_tokens: 4096`，`timeout: 60s`

---

### 4. Skill 评估与进化（SkillEvolver + Evaluator）

文件：`src/skill/evolver.ts`、`src/skill/evaluator.ts`

#### 4.1 入口

```
Task finalize → onTaskCompletedCallback → SkillEvolver.onTaskCompleted(task)
```

#### 4.2 Rule Filter（零 LLM）

- chunks < 6 → skip
- task.status === "skipped" → skip
- task.summary.length < 100 → skip
- 无 user/assistant chunks → skip

#### 4.3 搜索相关 Skill（两阶段）

**Stage 1**：FTS + 向量搜索（cosine 下限 0.35），融合得分 = 0.7 × 向量分 + 0.3 × FTS 分，取 top 10。

**Stage 2**：LLM Judge 从 top 10 中选出真正匹配的 1 个（或 0 个）。

> "You are a strict judge: decide whether a completed TASK should be merged into an EXISTING SKILL... Output exactly ONE skill index (1 to N) ONLY if the task's experience clearly belongs to that skill's domain. If no skill is clearly relevant, output 0."

输出：`{ "selectedIndex": 0, "reason": "..." }`

#### 4.4 创建评估 Prompt

决定是否将任务提取为新 Skill：

**必须满足 ALL**：
1. Repeatable（任务类型可能重复出现）
2. Transferable（方法可帮助他人解决同类问题）
3. Technical depth（包含非平凡的步骤、命令、代码、诊断推理）

**不值得提取的情况**：纯事实 Q&A、单轮简单回答、碎片化对话、一次性个人任务、闲聊、简单信息查找、通用概述无具体操作步骤。

输出：`{ "shouldGenerate": bool, "reason": "...", "suggestedName": "kebab-case", "suggestedTags": [...], "confidence": 0.0-1.0 }`

门槛：`shouldGenerate && confidence >= 0.7`

#### 4.5 升级评估 Prompt

判断新任务是否值得升级已有 Skill：

**值得升级**（任一）：更快、更优雅、更方便、更少 token、更准确、更健壮、新场景、修复过时信息。

输出：`{ "shouldUpgrade": bool, "upgradeType": "refine"|"extend"|"fix", "dimensions": [...], "reason": "...", "mergeStrategy": "...", "confidence": 0.0-1.0 }`

门槛：`shouldUpgrade && confidence >= 0.7`

---

### 5. Skill 生成（Generator）

文件：`src/skill/generator.ts`

#### 5.1 预处理

- **敏感数据脱敏**：`sk-***REDACTED***`、`Bearer ***REDACTED***`、`AKIA***REDACTED***`、`api_key = "***REDACTED***"`
- **Tool 输出截断**：每个 tool chunk 最多 1500 字符（60% 头部 + 30% 尾部 + 截断提示）
- **语言检测**：CJK 字符比例 > 15% → 中文

#### 5.2 SKILL.md 生成 Prompt

核心原则：
- **Progressive disclosure**：frontmatter description（~100 词）常驻上下文；SKILL.md body（<400 行）触发后加载；大配置/脚本放入 references/。
- **Description as trigger mechanism**：description 必须" proactive "——不仅说做什么，还要列出应该触发它的场景、关键词、措辞。
  - Bad: "How to deploy Node.js to Docker"
  - Good: "How to containerize and deploy a Node.js application using Docker. Use when the user mentions Docker deployment, Dockerfile writing, container builds, multi-stage builds, port mapping..."
- **通用化**：从具体任务抽象出可跨场景应用的方法论。

输出格式：YAML frontmatter（`name`, `description`, `metadata`）+ Markdown body（`# Title`, `## When to use`, `## Steps`, `## Pitfalls and solutions`, `## Key code and configuration`, `## Environment and prerequisites`, `## Companion files`, `## Task record`）

参数：`temperature: 0.2`，`max_tokens: 6000`，`timeout: 120s`

#### 5.3 并行生成附属文件

- **Scripts Prompt**：提取可重用的 shell/Python/TypeScript 自动化脚本，返回 JSON 数组 `[{filename, content}]`
- **References Prompt**：提取重要的 API 文档、配置参考、技术笔记，返回 JSON 数组 `[{filename, content}]`
- **Evals Prompt**：生成 3-4 条应该触发该 skill 的测试 prompt，返回 JSON 数组（含 `expectations` 和 `trigger_confidence`）

参数：`temperature: 0.1~0.3`，`max_tokens: 2000~3000`，`timeout: 120s`

#### 5.4 Eval 验证（零 LLM）

用 RecallEngine 对每个 eval prompt 做搜索，检查该 Skill 是否能被检索到（`minScore: 0.3`，`topScore >= 0.4`）。未通过意味着 description 写得不够好，触发率低。

#### 5.5 持久化与状态

```
{stateDir}/skills-store/{skill-name}/
  ├── SKILL.md
  ├── scripts/
  ├── references/
  └── evals/evals.json
```

`qualityScore < 6` → status = "draft"（不会自动召回注入）
`qualityScore >= 6` → status = "active"

---

### 6. Skill 升级与回退（Upgrader）

文件：`src/skill/upgrader.ts`

#### 6.1 升级 Prompt

> "You are a Skill upgrade expert. You're merging new real-world execution experience into an existing Skill to make it better... Preserve all valid core content from existing skill. Merge new experience strategically: Better approach → replace old, keep old as 'Alternative' if still valid. New scenario → add new section. Bug corrected → replace directly, add to Pitfalls."

约束：总长度 ≤ 400 行。添加版本注释：`<!-- v{NEW_VERSION}: {change note} (from task: {TASK_ID}) -->`

输出格式：完整 upgraded SKILL.md，后接 `---CHANGELOG---` 和 `---CHANGE_SUMMARY---`。

参数：`temperature: 0.2`，`max_tokens: 6000`，`timeout: 90s`

#### 6.2 回退机制

```
升级前 → 备份 Skill 目录
  → LLM 生成新内容
  → 写入新 SKILL.md + 重建附属文件
  → Validator 验证（含回归检查：新内容比旧内容缩水 >30% → warning）
  → 验证失败 → 从备份恢复整个目录
  → 验证通过 → version +1，持久化新版本
```

---

### 7. 质量验证（Validator）

文件：`src/skill/validator.ts`

#### 7.1 五阶段验证

1. **格式验证**：SKILL.md 存在且非空、有 YAML frontmatter、有 `name`（≤64 字符，kebab-case）、有 `description`（≤1024 字符）、行数 ≤ 400、内容 ≥ 200 字符
2. **回归检查**（升级时）：新内容比旧内容缩水 > 30%（且旧内容 > 20 行）→ warning；检测缺失的 `##` 章节
3. **附属文件一致性**：SKILL.md 引用的文件必须在磁盘上存在；孤立文件 → warning
4. **密钥扫描**：`sk-...`、`Bearer ...`、`AKIA...`、`api_key/secret/token/password = "..."`、长 Base64
5. **LLM 质量评分**（0-10 分）：
   - Clarity（0-2）
   - Completeness（0-2）
   - Reusability（0-2）
   - Accuracy（0-2）
   - Structure（0-2）

`score < 6` → 标记为 draft。

---

### 8. Skill 召回与注入（TS 端）

文件：`index.ts`（`before_prompt_build` hook）

#### 8.1 召回流程

```
用户消息
  → normalizeAutoRecallQuery（确定性清洗）
  → 并行搜索：本地引擎（maxResults: 10, minScore: 0.45）+ Hub 远程搜索
  → LLM 相关性过滤（FILTER_RELEVANT_PROMPT）
  → 去重（>70% 词汇重叠 → 去掉）
  → 组装注入块 → prependContext
```

#### 8.2 相关性过滤 Prompt

> "Given a QUERY and CANDIDATE memories, decide: does each candidate's content contain information that would HELP ANSWER the query? CORE QUESTION: 'If I include this memory, will it help produce a better answer?' ... Merely shares same broad topic but NO useful information → NOT relevant. When multiple candidates convey same information, keep ONLY the most complete/detailed one."

输出：`{ "relevant": [1,3], "sufficient": true }`

#### 8.3 注入格式

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
- A hit has task_id → call task_summary(taskId="...")
- A task may have a reusable guide → call skill_get(taskId="...")
- Need surrounding dialogue → call memory_timeline(chunkId="...")
```

**Skill 注入块**：
```
## Relevant skills from past experience

The following skills were distilled from similar previous tasks.
You SHOULD call skill_get to retrieve the full guide before attempting the task.

1. **{name}** [installed] — {description}
   → call skill_get(skillId="{id}")
```

---

### 9. Python 端核心差异

#### 9.1 异步调度器架构

`MOSCore.add()` 有两种模式：

- **同步模式**：在 `add()` 内一步完成切块 + LLM 精读 + 写入。延迟高但简单。
- **异步模式**（默认）：`add()` 只做粗存储（Fast 切块 + 写入），把精读任务提交给调度器后台处理。
  - 调度器消费 `MEM_READ_TASK` → `fine_transfer_simple_mem()` → **4 线程并行**：
    - 线程 A：`process_string_fine`（结构化记忆提取）
    - 线程 B：`process_skill_memory_fine`（Skill 提取）
    - 线程 C：`process_preference_fine`（偏好提取）
    - 线程 D：`process_tool_trajectory_fine`（工具轨迹提取）

#### 9.2 一次性任务分块（支持非连续回溯）

`TASK_CHUNKING_PROMPT`：
- 输入：完整对话（最近 20 条）
- 输出：JSON 数组，每个任务包含 `task_id`, `task_name`, `message_indices: [[start,end], ...]`
- 支持"跳跃式对话"：messages 8-11 = 旅行, 12-22 = 编程, 23-24 = 旅行 → "旅行规划" 同一个任务包含 `[8,11]` 和 `[23,24]`

这是 TS 端线性主题分割做不到的。

#### 9.3 切块策略矩阵

Python 端有 6 种切块器，针对不同场景：

| 切块器 | 粒度 | 重叠方式 | 适用场景 |
|--------|------|---------|---------|
| `_iter_chat_windows` | token 级 (默认 1024) | 弹出头部到 200 tokens | 主力记忆提取 |
| `get_scene_data_info` | 每 10 条消息 | 尾部 2 条 | 预切分（粗粒度） |
| `content_length` | 字符数 | 尾部 1 条 | 长消息场景 |
| `session` | N 条消息 | 步长滑动 | 固定窗口 |
| `lookback` | QA 对 | 回看 N 轮 | 偏好提取 |
| `overlap` | 每 10 条消息 | 尾部 2 条 | 偏好提取 |
| `SentenceChunker` | 句子边界 | token overlap | 文档类消息 |

**三层保底机制**（`_concat_multi_modal_memories`）：
1. 单条超大 → 先用 `SentenceChunker` 拆小
2. 窗口不空 → buf 为空时即使超 token 也先放进去
3. 尾部不丢 → 循环结束后 buf 剩余内容全部输出

#### 9.4 偏好提取（Preference Extraction）

文件：`src/memos/mem_reader/read_pref_memory/process_preference_memory.py`

**两类偏好并行提取**：
- **显式偏好**：用户明确表达的态度、选择、拒绝。输出：`{explicit_preference, context_summary, reasoning, topic}`
- **隐式偏好**：从行为模式、决策逻辑推断。限制：只有一轮问答不提取；Assistant 建议未被用户认可不提取。

**去重流程**：
1. 向量召回已有记忆 → LLM 判断语义相似
2. 若相似 → 再判断是同一核心问题（更新）还是不同问题（新增）

#### 9.5 工具轨迹提取（Tool Trajectory）

文件：`templates/tool_mem_prompts.py`

Prompt 要求：
1. 判断任务完成度 → `success` / `failed`
2. **Success**：提炼 `when...then...` 格式的经验规则（触发场景特征 + 有效参数模式/最佳实践）
3. **Failed**：
   - 任务是否需要工具？（需要/直接回答/误调用）
   - 工具调用检查：存在性、选择正确性、参数正确性、幻觉检测
   - 错误根因定位
   - 经验提炼（`when...then...` 给出避免错误的通用策略）

输出包含 `correctness`、`trajectory`、`experience`、`tool_used_status`（含 `success_rate`、`error_type`、`tool_experience`）。

#### 9.6 反馈修正闭环

文件：`src/memos/mem_feedback/`

三步串行处理用户纠正：
1. **关键词替换检测**：识别是否是批量替换（如把"张三"都改成"李四"）
2. **反馈有效性判断**：识别用户态度（dissatisfied/satisfied/irrelevant），提取纠正后的核心事实
3. **记忆更新操作**：
   - `NONE`：新事实未提供额外信息
   - `UPDATE`：新事实更准确/完整/需修正，仅修改相关错误片段
   - `ADD`：无匹配记忆，全新写入

#### 9.7 检索增强管线

```
用户查询 → 意图识别（是否需要检索？）→ 关键词提取 → 多路召回
  ├─ 向量检索 (Qdrant)
  ├─ 图检索 (Neo4j subgraph)
  ├─ BM25 全文检索
  └─ 互联网检索（可选）
       → 记忆过滤 + 去重 + 重排序
       → 记忆增强（消歧：代词→全名、相对时间→具体日期、合并互补细节）
       → 注入 System Prompt
```

涉及 4 个 Prompt：
- **INTENT_RECOGNIZING_PROMPT**：`trigger_retrieval=true/false`
- **MEMORY_RECREATE_ENHANCEMENT_PROMPT**：消歧增强
- **MEMORY_COMBINED_FILTERING_PROMPT**：两步过滤 + 冗余去重
- **ENLARGE_RECALL_PROMPT_ONE_SENTENCE**：生成改写查询用于二次检索

---

### 10. MemOS 对 gspd 的核心启示

1. **任务驱动提取优于滑动窗口**：MemOS TS 的 `TaskProcessor` 在主题切换时 finalize 任务，保证了完整上下文和清晰边界。AutoSkill 的 `messages[-6:]` 虽然延迟低，但跨任务混合和碎片化严重。
2. **实时增量 vs 异步批量的权衡**：TS 端适合本地、低延迟、实时反馈；Python 端的异步调度器适合云端、高吞吐、不在乎即时性。gspd 应该同时支持两种模式。
3. **Description 是触发机制，不是装饰**：MemOS 反复强调 description 要"proactive"——列出所有可能触发的场景、关键词、变体表述。这直接决定了 Skill 的利用率。
4. **Eval 即质量门控**：MemOS TS 在生成 Skill 后自动生成 eval prompts，并用 RecallEngine 测试触发率。这是把"可检索性"作为质量指标的先驱做法，gspd 应该借鉴。
5. **多层质量门控是必要的**：Rule filter → LLM confidence → LLM 0-10 score → 回归检查（缩水 >30%）。没有这些门控，Skill 库会迅速退化。
6. **非连续回溯是高级能力**：Python 端的一次性 LLM 聚类可以处理跳跃式对话，但这需要额外的 LLM 调用。gspd 的 v1 可以只做线性分割，v2 再引入非连续回溯。
7. **工具轨迹提取的 `when...then...` 格式**：这是把程序记忆固化为条件-动作对的最佳实践，与 XSkill 的 Experience 设计不谋而合。
8. **反馈修正闭环是记忆系统的最后一块拼图**：没有它，用户的纠正只能影响当次回答，无法持久化。MemOS 的三步闭环（关键词检测 → 有效性判断 → UPDATE/ADD 决策）可以直接借鉴。

---

*报告完*
