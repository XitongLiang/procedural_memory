# OpenViking 记忆/上下文引擎深度分析（对标 GSPD 分层设计）

> **分析目标**：把 OpenViking 的设计文档（`docs/en/concepts/*.md`）与真实代码（`openviking/`）对齐到 GSPD 四层模型（L3/L2/L1/L0）提出的每一个问题，并标注文档与代码的实际分歧。所有结论均带 `file_path:line_number` 引用，便于复核。
>
> **核心结论（TL;DR）**：
> - OpenViking 实为 **"3 层 + 隐式中间层"** 的分层结构，不存在 GSPD 意义上的 **L0 恒常加载层**。它的 L0/L1/L2 是"摘要粒度层级"（abstract / overview / full content），而非"加载优先级层级"。
> - L3（原始会话）以 `messages.jsonl` 逐条事件持久化，Block 粒度 = 单条 Message；Block 间无显式关联；归档靠手动 `commit()`。
> - L2（原子事实）采用 **8 类单标签分类**：User 侧 PROFILE / PREFERENCES / ENTITIES / EVENTS，Agent 侧 CASES / PATTERNS / TOOLS / SKILLS；与 GSPD 六分类**不一一对应**，尤其**缺失 Prospective**。
> - L1 为**目录摘要**（`.overview.md` ~1–2k tokens），算法上靠目录结构聚合，**不是语义聚类**。
> - 无专门的 L0 固定加载层；"恒常"内容只能通过 IntentAnalyzer 拼接最近 5 条消息 + 会话摘要。
> - 抽取是 **Session commit 两阶段**：Phase 1 同步写归档，Phase 2 异步抽取 → dedup → 向量化。**完全依赖手动 commit**，无自动触发、无 SLA、无重试。
> - **最大槽点**：Dedup 的 SKIP 是硬丢弃 + 不审计；immutable 类别（EVENTS/CASES）无生命周期上限；全量 re-extraction 成本 O(session size)。

---

## 一、L3 原始会话层

### Q1. 是否持久化原始会话？格式？每条记录包含什么？

**OpenViking 的做法**
- 持久化。会话以 **`messages.jsonl`（按行 JSON）** 写到：
  `viking://session/{user_space}/{session_id}/messages.jsonl`（`openviking/session/session.py:192-198`）。
- 每条记录：`id`（`msg_{UUID}`）、`role`（user/assistant）、`parts`（消息片段列表）、`created_at`。
- Part 类型细分为 `TextPart` / `ContextPart` / `ToolPart`（`session.py:77-94`）；`ToolPart` 含 `tool_name` / `tool_input` / `tool_output`（**截断到 500 字符**）/ `tool_status` / `duration_ms` / `skill_uri`。
- 没有把 system message 设为顶层字段；系统消息内嵌在 parts 里。

**解决了什么**
- 支持逐轮回放、审计、不依赖 LLM 的二次抽取。
- 工具调用语义（不仅文本）被保留，可以做"工具使用"学习。

**引入了什么新问题**
- **存储无上限**：历史 session 没有自动驱逐；只靠 archival 压缩。
- **工具输出被截断（500 字符）**：大输出关键信息会丢。
- **系统消息无独立追踪**：混在 parts 里。
- **无结构化 turn 元数据**（延迟、token 计数），metrics 走另一条管道。

**与 GSPD naive 假设的差异**
- GSPD 默认"L3 = 消息"；OpenViking 的 L3 还包含工具 I/O 语义。
- 指标与消息分离，存储时没有延迟 / token 数等 turn-level 元数据。

---

### Q2. Block 粒度？是定长 / 事件驱动 / 按 turn？

**OpenViking 的做法**
- **Block = 单条 Message**（不是 turn，也不是一次 LLM 交换）。一个 turn = user + assistant = 2 个 block。
- 粒度是**事件驱动**：每次 `session.add_message(role, parts)` 就是一个 block。
- **无按 token 自动切块**，也不做自动 batch。
- 压缩触发：手动调 `session.commit()`；每次 commit 把所有累积消息归档成一个 `archive_NNN/messages.jsonl`，同时自增 `compression_index`（`session.py:204-212`）。

**归档结构**
```
viking://session/{user_space}/{session_id}/
├── messages.jsonl             # 当前活跃消息
├── .abstract.md               # 当前会话摘要（L0）
├── .overview.md               # 当前会话概览（L1）
└── history/
    └── archive_001/
        ├── messages.jsonl     # 归档消息（Phase 1 同步）
        ├── .abstract.md       # 异步生成（Phase 2）
        ├── .overview.md       # 异步生成（Phase 2）
        └── .done              # 完成标记
```
（`session.py:174-186`，`docs/en/concepts/08-session.md:175-186`）

**解决了什么**
- 事件驱动粒度贴合 agent 自然轮次。
- 归档命名有序、不会冲突。

**引入了什么新问题**
- **缺少"一轮对话"的显式边界**：用 message 做单元，拼 turn 要外部逻辑。
- **必须手动 commit**：崩溃期间新消息可能丢。
- **归档不可变**：抽取错了必须整批重跑。
- **难"取最近 N 轮"**：必须扫归档找 boundary。

---

### Q3. Block 间是否关联 / 有向 / 跨会话图？

**OpenViking 的做法**
- **无显式 cross-block 链接**。
- **Session 内**：靠 `created_at` 时间排序。
- **归档间**：靠 `compression_index` 序号。
- **跨会话**：**无图**，会话彼此独立。
- Memory 与 session 的关联：抽取时写入 `source_session` 字段（`openviking/session/memory_extractor.py:66`）。
- 无 session 间继承 / 延续 / 话题关联。

**问题 / 启示**
- 简化了隔离，但失去了跨会话话题连续性；Agent 要自己 chain session。
- Dedup 时没法用"是否同一 session"作为权重。

---

### Q4. Block 如何映射到 Agent 的 in-memory Context？

**OpenViking 的做法**
- `Session._messages` 是内存列表；`session.load()` 反序列化 jsonl；修改不自动持久化，必须显式 `commit()`（`session.py:185-199`）。
- `commit()` 后，内存列表被清空，旧消息只能从 `archive_NNN/messages.jsonl` 冷读。
- 工具 / 环境变更**不做版本化**：如果工具签名变了，旧 archive 里的 ToolPart 与新语义不兼容。

**隐含风险**
- 提交后历史就是冷存储；要看历史必须 re-load archive。
- 无抽取模型版本记录：用新模型对旧 archive 重抽取，结果不可复现。

---

### Q5. Block 数量上限 / 轮转 / 驱逐？

**OpenViking 的做法**
- 原始 session **没有硬上限**，也无自动轮转。
- `MemoryArchiver`（`openviking/session/memory_archiver.py:50-80`）操作的是**抽取出的 memory**，不是原始 session：
  - `hotness_score = sigmoid(log1p(active_count)) * exp(-decay_rate * age_days)`（`openviking/retrieve/memory_lifecycle.py:19-64`）
  - 默认阈值 0.1，最小 age 7 天，批大小 100。
  - 低热度 memory 被移到 `_archive/` 子目录，**默认检索会排除**它们，但不删除。
- 原始 session 历史没有 GC。

**与 GSPD 假设的差异**
- GSPD 期望 L3 有明确 token 预算；OpenViking 无此约束。
- 驱逐在 memory 层而非 session 层。

---

## 二、L2 原子事实层

### Q6. 是否分类？六大分类如何映射？

**OpenViking 的做法**（`memory_extractor.py:40-56`）
- **8 类单标签**，分 User-side 与 Agent-side 两组：

| 侧 | 类别 | 语义 | 合并策略 |
|---|---|---|---|
| User | `PROFILE` | 稳定身份 / 属性 | Append-only merge 到单个 profile.md |
| User | `PREFERENCES` | 话题偏好 | 可合并，按话题一个文件 |
| User | `ENTITIES` | 人 / 项目 / 概念 | 可合并 |
| User | `EVENTS` | 日期事件 | **不可变**（add-only） |
| Agent | `CASES` | 具体问题 → 解法 | **不可变** |
| Agent | `PATTERNS` | 可复用 workflow | 可合并 |
| Agent | `TOOLS` | 工具使用统计 | 可合并 |
| Agent | `SKILLS` | 技能执行知识 | 可合并 |

**存储路径**（`memory_extractor.py:104-115`）：
```
PROFILE     → viking://user/{space}/memories/profile.md
PREFERENCES → viking://user/{space}/memories/preferences/{topic}.md
ENTITIES    → viking://user/{space}/memories/entities/{entity}.md
EVENTS      → viking://user/{space}/memories/events/{year}/{month}/{day}/{event}.md
CASES       → viking://agent/{space}/memories/cases/{case_name}.md
PATTERNS    → viking://agent/{space}/memories/patterns/{pattern_name}.md
TOOLS       → viking://agent/{space}/memories/tools/{tool_name}.md
SKILLS      → viking://agent/{space}/memories/skills/{skill_name}.md
```

**映射到 GSPD 六分类**

| GSPD | OpenViking 对应 | 覆盖度 |
|---|---|---|
| 情景（Episodic） | EVENTS | ✅ 完整 |
| 语义（Semantic） | PROFILE / PREFERENCES / ENTITIES | ✅ 完整 |
| 程序（Procedural） | PATTERNS / TOOLS / SKILLS / CASES | ✅ 完整 |
| 前瞻（Prospective） | ❌ **缺失**（只在 PATTERNS 中隐式出现） |
| 心智—Self | 分散在 PATTERNS + CASES（无聚合） |
| 心智—User | 分散在 PROFILE + PREFERENCES + ENTITIES（无聚合） |
| 通用（General） | ❌ **无此桶** |

**与 GSPD 的关键差异**
- **缺 Prospective**：没有"用户 / Agent 准备做的事"的独立分类。
- **无多标签**：每条 fact 只能属于一个 category。
- **Mental model 被稀释**：PROFILE 混合了身份和心智模型。

---

### Q7. "原子事实"如何定义？粒度？

**OpenViking 的做法**（`memory_extractor.py:58-68`）
```python
@dataclass
class CandidateMemory:
    category: MemoryCategory
    abstract: str   # L0: 一句话
    overview: str   # L1: 中等详细
    content: str    # L2: 完整叙述
```

- **原子性由抽取 prompt 决定**，每类 prompt 给出不同粒度：
  - `prompts/templates/memory/profile.yaml:20` → "每条 <30 词的独立陈述"——**句子级**。
  - `prompts/templates/memory/events.yaml:13` → "单个事件内尽量写完整"——**事件级**。
  - `prompts/templates/memory/tools.yaml:40` → 一条 fact = 一个工具——**工具级**。
- 不强制限定 token，但 profile 条目最短、event 最长。

**问题**
- "原子"是 prompt 的隐式契约，无机器可检验的原子性断言。
- 不同类别粒度混用 → dedup / 检索难用统一的相似度阈值。

---

### Q8. 抽取机制：上下文？窗口？切块方式？

**OpenViking 的做法**
- **用整个 session 作为上下文**（`memory_extractor.py:225-232`）。
- 每条消息完整 + 其下所有 tool call/result（工具输出截断 500 字符）（`memory_extractor.py:197-223`）。
- **并发上限 = 10** LLM 调用（`storage/queuefs/semantic_processor.py:85`）。
- 切块粒度 = 按消息边界，不做话题切块。

**引入的问题**
- Session 越长 → extraction 输入越大 → LLM 成本线性增长，无增量抽取。
- 超上下文窗口会**静默失败或截断**，无容错策略。
- 工具输出截断 500 字符 → 失去长日志 / 长结果中的信号。

---

### Q9. 是否多标签？

**OpenViking**：**硬单标签**。`CandidateMemory.category` 是单值枚举。

**后果**
- 跨类 fact 要么重复抽（增加 dedup 负担），要么只选一类（损失语义）。
- Dedup 永远按 category 分桶，无法跨桶发现重复。

**GSPD 借鉴**
- 建议改为 **主类别 + 附加标签**（如 `primary: Procedural, tags: [Semantic, Mental-User]`），避免信息损失。

---

### Q10. 抽取时机：同步 / 异步？触发？

**OpenViking 的做法**（`session.py:100-113`）
- **两阶段 commit**：
  - **Phase 1（同步）**：写 `archive_NNN/messages.jsonl`，清空内存，立即返回 `task_id`。
  - **Phase 2（异步后台）**：memory 抽取 → dedup → 向量化。
- 触发源：**只由 `session.commit()` 触发**。
- `auto_commit_threshold=8000`（tokens）可触发自动 commit（`session.py:164`），但**默认关闭**。

**流程**（`docs/en/concepts/08-session.md:113-120`）
```
Messages → Compress (Phase 1 同步) → Archive (同步)
                                    + Memory extraction (Phase 2 异步)
                                      → Dedup → Storage
```

**问题**
- **Phase 2 无重试 / 监控**：崩溃就永久丢失抽取结果。
- 异步抽取意味着"刚说的话"短时间内搜不到。
- 手动 commit 容易忘 → 消息丢。

---

### Q11. 模型与成本？

**OpenViking 的做法**
- 模型：`get_openviking_config().vlm`（`memory_extractor.py:239`），全局配置。
- 并发：10。
- 指标（Prometheus）：`openviking_vlm_calls_total`、`openviking_vlm_tokens_input/output_total`、`openviking_vlm_call_duration_seconds`（`docs/en/concepts/12-metrics.md:175-186`）。
- **无 budget 限制、无 circuit breaker、无回退模型**。

**估算**
- 每次 commit 约 2–5k 输入 tokens × N 个类别，成本随 session 大小线性放大。
- 无 incremental extraction，无缓存。

---

## 三、L1 渐进加载层

### Q12. 是否有介于原子事实和顶层摘要之间的层？

**OpenViking 的做法**
- 有**隐式的中间层**：
  - **L0 `.abstract.md`**：~100 tokens，ultra-concise。
  - **L1 `.overview.md`**：~1–2k tokens，导航 + 关键点。
  - **L2**：原文（资源），或抽取后的 memory 内容。
- `SemanticProcessor`（`storage/queuefs/semantic_processor.py:66-75`）**自底向上**：先处理叶子文件的摘要，再合并到父目录生成父级 L0/L1。

**例子**（`docs/en/concepts/03-context-layers.md:44-62`）：
```markdown
# Authentication Guide Overview

## Sections
- **OAuth 2.0** (L2: oauth.md): Complete OAuth flow
- **JWT Tokens** (L2: jwt.md): Token generation
- **API Keys** (L2: api-keys.md): Simple key-based auth

## Key Points
- OAuth 2.0 recommended for user-facing apps
- JWT for service-to-service
```

**解决 / 局限**
- 优点：进入目录前可以先看 overview，不必读全部叶子。
- 缺点：L1 = 目录的 Markdown 摘要，**不是按任务 / 主题的聚类摘要**，也**不随检索场景动态生成**。

---

### Q13. 聚类逻辑？算法或 LLM？

**OpenViking 的做法**
- **按目录结构 + 文件名分组**，不做向量聚类。
- 每个 category 一个目录；文件名自带语义（如 `events/2026/04/15/*.md`）。
- Session 摘要是**整段 LLM 生成**（`openviking/session/compressor.py`），不做子话题切分。

**问题**
- 无动态主题检测：若抽取 prompt 把同一实体叫了两个名字，就会分到两个文件。
- 无相似度聚类 → 同语义不同类别的 fact 不会合并。
- 长 session 的摘要是单点 LLM 调用，摘要质量随长度下降。

---

### Q14. Dreaming / 合并 / dedup 在哪层？时机？

**OpenViking 的做法**
- **Dedup**（`openviking/session/memory_deduplicator.py:67-90`）
  - 时机：**Session commit Phase 2 异步**。
  - 步骤：向量预过滤（最多 5 个相似 memory） → LLM 决策。
  - 决策等级：
    - Candidate 级：`SKIP` / `CREATE` / `NONE`。
    - 与已有 memory 的关系：`MERGE` / `DELETE`。
  - 向量相似度阈值 0.0（无阈值）。
  - `PROFILE` 永远 MERGE（跳过 dedup）；不可变类别（CASES / EVENTS）不允许 MERGE。
- **MemoryArchiver**（热度低 + age>7d → 移到 `_archive/`）——Q5 已述。
- **没有显式的 dreaming / 一致性维护阶段**——只有 dedup。

**问题**
- `SKIP` 会**直接丢弃候选**，**无审计日志**。
- Dedup cost 随候选数增长，且每个候选要做 N 次 LLM 决策（N = vector top-K）。
- 没有"定期整理所有 memory"机制 → 累积的相似 memory 不会自动合并。

---

### Q15. 渐进加载如何决定往下钻？

**OpenViking 的做法**（`openviking/retrieve/hierarchical_retriever.py:88-223`）
- 两阶段：
  1. **IntentAnalyzer**：LLM 重写 query → 0–5 个 TypedQuery。
  2. **HierarchicalRetriever**：优先队列递归目录树。
- 核心循环：
  ```python
  while dir_queue:
      current_uri, parent_score = heapq.heappop(dir_queue)
      results = await search(parent_uri=current_uri)
      for r in results:
          final_score = 0.5 * embedding_score + 0.5 * parent_score
          if final_score > threshold:
              collected.append(r)
              if not r.is_leaf:
                  heapq.heappush(dir_queue, (r.uri, final_score))
      if topk_unchanged_for_3_rounds: break
  ```
- 关键常量：`SCORE_PROPAGATION_ALPHA=0.5`、`MAX_CONVERGENCE_ROUNDS=3`、`GLOBAL_SEARCH_TOPK=10`、`HOTNESS_ALPHA=0.2`。
- 深入条件：`child_score > parent_score × DIRECTORY_DOMINANCE_RATIO(1.2)` 就递归。

**问题**
- 50/50 分数传播过激 → 深层 fact 几乎必降分，可能漏掉深嵌关键信息。
- 收敛判据朴素（"top-K 3 轮不变"）→ 可能过早剪枝。
- 没有"query 长度 / 复杂度 → 最大深度"的自适应。

---

## 四、L0 恒常加载层

### Q16. 是否有 L0 固定加载层？

**OpenViking 的做法**
- **没有显式 L0 恒常加载层**。
- 最接近的是 IntentAnalyzer 把 **session compression 摘要 + 最近 5 条消息** 拼到检索 prompt 里（`openviking/retrieve/intent_analyzer.py:116`）。
- User profile / agent identity 是 memory，按需检索，不会固定注入到 prompt。
- 所谓 L0 `.abstract.md` 只是"摘要文件"，**不会自动加到对话上下文**。

**与 GSPD 的差异**
- GSPD 的 L0 是 "top-of-prompt 恒常常量"（身份、规则、关键承诺）。
- OpenViking 没有等价物；Agent 要自己拼 system prompt。

---

## 五、替代方案与批评

### Q17. 有效层数是几层？

OpenViking 的实际层次可分两种视角：

**视角 A（粒度层）**：3 层
- L0 = abstract（~100 tokens）
- L1 = overview（~1–2k tokens）
- L2 = 原文 / memory 内容（不限）

**视角 B（生命周期层）**：4 层
- L3 = 原始会话
- L2 = 抽取出的 candidate memory（8 类）
- L1 = category / session 摘要
- L0 = abstract（但不恒常加载）

`docs/design/openclaw-context-engine-refactor.md:59-85` 指出系统缺少：**统一的 context assembly 层**、**显式 context budget**。建议在 L3 和 L2 之间补一个 **"当前 session 工作上下文"**（最近 N 轮 + 活跃 fact）。

### Q18. 理论基础？

- 粒度分层（L0/L1/L2）有**信息金字塔**的直觉，但**无文献引用**。
- 8 类分类类似 Tulving 的"declarative / procedural / episodic"三分，但没有正式对齐。
- Hotness score（`sigmoid(log1p(count)) * exp(-decay * age)`）来自工程直觉（Ebbinghaus-like），未声明理论出处。
- 结论：**偏实用主义，缺理论锚点**。

### Q19. 对 GSPD 的启示性陷阱（至少 3 条）

1. **抽取延迟 → 记忆"刚说就搜不到"**：OpenViking 的 Phase 2 是 fire-and-forget，无 SLA、无重试。GSPD 若也 fire-and-forget，Agent 自己无法判断"我刚说的 fact 存进去了吗"。**建议**：抽取队列要有 completion webhook 或 poll API；critical fact（goal/commitment）走同步路径。
2. **单标签分类吃掉跨类语义**：如"用户偏好 Python 因为 pattern X"，OpenViking 只能落一类。**建议**：GSPD 用 primary + tags 或 fact-graph。
3. **不可变类别 + 手动归档 → 无限膨胀**：EVENTS / CASES 只加不删，Archiver 靠手动。**建议**：强制生命周期（比如 events 月级 roll-up）。
4. **全量 re-extraction 成本随 session 线性**：GSPD 必须做 incremental extraction。
5. **Dedup SKIP 无审计**：GSPD 必须记录每一次 SKIP / MERGE / DELETE 的决策、原因、被丢弃 fact 的指纹。
6. **分数传播 0.5/0.5 → 深层 recall 下降**：GSPD 若做层间路由要用 query-aware 的深度决策，而非固定 alpha。

---

## 六、文档 vs 代码的分歧（精选）

| 文档声称 | 代码实际 | 影响 |
|---|---|---|
| "L0/L1/L2 支持渐进加载"（`docs/en/concepts/03-context-layers.md:1-3`） | 检索从 IntentAnalyzer 开始，**不会自动先 L0 再 L1 再 L2**；每次要取哪层需要 Agent 显式调用（`hierarchical_retriever.py:88-150`） | "progressive loading"是文档口号，非运行时行为 |
| "8 类 memory 统一 merge/delete dedup"（`docs/en/concepts/08-session.md:135-169`） | PROFILE 永远 merge、CASES/EVENTS 不可 merge、PREFERENCES/ENTITIES/PATTERNS 可 merge（`compressor.py:37-41`，`memory_deduplicator.py:31-44`） | 实际规则比文档复杂 |
| "Session compression = compress + archive + extract 三阶段" | 只有两阶段（同步归档 + 异步抽取），compress 是 extraction 的副作用（`session.py:100-113`） | 文档误导 |
| "Intent Analysis 生成 0–5 TypedQuery" | 正确，输入含 compression summary + 最近 5 条 + 当前消息（`intent_analyzer.py:38-100`） | 一致 |
| "Rerank 通过 rerank_config 启用" | 只在 THINKING 模式用，QUICK 模式禁用；rerank 失败会降级到向量分数（`hierarchical_retriever.py:39-86`） | 文档没讲模式门 |
| "L0 从 L1 overview 抽取" | 代码实际先生成 L1（汇总子 L0 + 文件摘要）再从 L1 派生 L0（`docs/en/concepts/06-extraction.md:117-124`） | 生成方向与文档描述相反 |
| "Archiver 把冷 memory 移到 `_archive/`" | 正确；但**默认检索不排除** `_archive/`，需要显式过滤（`memory_archiver.py:50-80`） | 文档遗漏 |

---

## 七、给 GSPD 的核心启示

1. **显式生命周期必须强制**，不能是"可选操作"。Session age cap、定期 consolidation、cost budget 都要默认开启。
2. **支持多标签 / fact graph**，拒绝单标签桶。
3. **层间保留双向链接**：L1 摘要必须索引它的 L2 来源；否则丢失溯源能力。
4. **增量抽取 + 抽取产物缓存**：按消息 digest 做键，避免重复 LLM 调用。
5. **抽取必须有可观测性**：队列深度、失败率、每个 fact 的 model version / prompt hash。
6. **显式 L0 + token 预算**：L0 占多少 token、哪些一定进、哪些被裁剪，必须有策略。
7. **抗膨胀机制**：时间窗口 + hotness + decay 混合打分；不可变类别也要做月度 / 季度 roll-up。
8. **多 agent / 多 session 模型**：OpenViking 完全缺失；GSPD 若面向 agent swarm 要补。
9. **读缓存**：相同 query 在无新 commit 时直接返回缓存结果。
10. **标记抽取模型版本**：未来换模型必须能按 version 重跑。

---

## 附：关键代码 / 文档路径索引

| 模块 | 路径 |
|---|---|
| Session 生命周期 | `openviking/session/session.py`、`openviking/session/compressor.py`、`compressor_v2.py` |
| 抽取 | `openviking/session/memory_extractor.py`、`openviking/prompts/templates/memory/*.yaml` |
| Dedup / 归档 | `openviking/session/memory_deduplicator.py`、`openviking/session/memory_archiver.py`、`openviking/retrieve/memory_lifecycle.py` |
| 检索 | `openviking/retrieve/hierarchical_retriever.py`、`openviking/retrieve/intent_analyzer.py` |
| 存储 | `openviking/storage/viking_fs.py`、`openviking/storage/collection_schemas.py`、`openviking/storage/queuefs/semantic_processor.py` |
| 设计文档 | `docs/en/concepts/01`–`12.md`，`docs/design/memory-extractor-optimization.md`、`openclaw-context-engine-refactor.md`、`parser-two-layer-refactor-plan.md` |
