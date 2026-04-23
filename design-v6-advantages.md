# V6 程序记忆设计的中心思想与定位

> 最后更新：2026-04-22
>
> 本文不重复 `Procedural-Memory-System-Comparison.md` 的 21 维横向表。目标是把 V6 的设计压缩成 **3 个中心思想**，说明每一条解决了什么问题、对比别家方案修了哪一块，让人一眼看懂 V6 的差异化定位。

---

## 0. 中心思想速览

V6 的架构由四块基石组成，逐块展开在 §1–§4：

| #   | 中心思想                        | 核心产出                         | 修的痛点                          | 对应系统                                                                     | 设计优势                                                                                                      |
| --- | --------------------------- | ---------------------------- | ----------------------------- | ------------------------------------------------------------------------ | --------------------------------------------------------------------------------------------------------- |
| 1   | **Task Trace 提取**           | 每 chunk 1 次结构化 Markdown，下游共享 | 提取器各扫原始对话、过程/知识态混存、Step 识别不稳定 | MemOS / AutoSkill / Mem0 / Hermes / Self-Improvement Skill               | **一份 Trace 喂多路提取**（原始对话解析 N 次 → 1 次）+ **事件层 / 事实层 / 知识层三层分离存储**（trace / concept 字典 / mem_record 生命周期各自独立） |
| 2   | **滑动窗口 + 主题拼接**             | 长任务被自动拼成一个 chunk             | 单窗口碎片化、长任务跨窗口丢失、token 线性涨     | MemOS / AutoSkill / Mem0 / XSkill                                        | **小窗口省 token + 长任务吃得下**，85% 拼接零 LLM                                                                       |
| 3   | **分类 + 候选池 + 快慢车道**         | 三元 × 双锚 × 单表候选池 × 双车道        | 粒度单一、硬约束生效慢、软经验无验证、向量不一致      | memory-tencentdb / AutoSkill / Mem0 / ReMe / MemOS / OpenViking / Hermes | **粒度 + 强度（强度决定节奏）**，硬约束零延迟 + 软经验有证据门                                                                      |
| 4   | **保证性提取（scheduler-driven）** | 定时器扫水位 + 6 层硬流水线             | agent 自觉触发有风险，用户反馈**可能被漏记**   | Hermes / Self-Improvement Skill / Skill-Creator                          | **agent 可能漏记，定时器不会**                                                                                      |

前三块是**做什么 / 怎么做**（数据结构 + 算法），第 4 块是**什么时候做 / 谁触发做**（执行平面）。组合起来：Task Trace 做中间层、窗口 + 主题拼接负责喂干净 chunk、分类 + 候选池 + 快慢车道组织下游知识、**而整条流水线由外部定时器驱动——不依赖 agent 自觉**。

剩下的小优化（水位重传、向量 1:1 拷贝、embedding 一致性）都可以归到这四块的延伸里。

---

## 1. Task Trace 提取：一次整理、多次消费

### 1.1 核心主张

Chunk flush 时**先**跑一次 `ExtractTaskTrace`（大模型，§6.0.3），把原始对话整理为结构化的 Markdown Task Trace，然后才交给 Fb/Tp 和 TSE 提取器消费。

产物结构（§6.0.3 输出模板）：

```markdown
## Task Trace

**User Query**: <一两句汇总>
**Task**: <name 4-10 字>：<description ≤ 30 字>
**Time Range**: <代码后处理插入>

### sub-task 1 — <name 10-25 字描述短语>

#### Step 1.1 — UserRequest
- Action: <...>
- Context: <... or (none)>

#### Step 1.2 — ToolCall
- Action: <...>
- How: <...>
- Parameters: <key1=value1 / key2=value2>
- Result: <OK: ... or ERROR: ...>

#### Step 1.3 — UserCorrection
- Action: <...>
- Type: correction | refinement | feedback
- Trigger: Step 1.2 / <问题简述>

### sub-task 2 — ...
```

四类互斥 Step：`UserRequest` / `AgentAction` / `ToolCall` / `UserCorrection`。每个字段都有明确语义和长度约束（§6.0.3 Section 3 字段表）。

### 1.2 为什么这么设计

**(a) 一次整理，多次消费**（§6.0.1）——原始对话是 flat 消息流，含大量噪声（问候、filler、重复确认）。直接喂给 Fb/Tp/TSE 三个提取器会：
- LLM 注意力被分散到无关消息
- 多次提取时重复解析同一对话结构
- 用户纠正信号和执行步骤混杂难定位

Task Trace 作为"原子事实层"一次整理后，下游按已清洗、已结构化的 Markdown 消费，每条记忆提取的 LLM 调用都变轻。

**(b) 严格 4 类 Step 让输出稳定**（§6.0.4 "为什么严格定义 4 类 Step"）——v5 prompt 里 Step 类型开放，LLM 经常把同一个工具调用拆成两个 Step（一个 thinking + 一个 tool call），下游还得重组。V6 固定 4 类可枚举且互斥，不同 LLM、不同 chunk 的输出形态都稳定。

**(c) thinking 折叠进 How 不独立成 Step**（§6.0.4）——thinking 本身没有外部副作用，单独成 Step 会让 Trace 充满"agent 想了想"的水分行；但 thinking 里"为什么选 X 不选 Y"是 Fb/TSE 的核心证据不能丢。V6 把 thinking 折叠进它直接驱动的下一个有副作用 Step 的 `How` 字段，既保留决策理由又不膨胀 Trace。

**(d) 两级锚点供下游按粒度选**（§6.0.1）：
- `task`（chunk 级整体任务）→ TSE 检索/归类锚点
- `sub-task`（Step 组级子主题，10-25 字描述短语）→ Fb/Tp 分段扫描的锚点

Fb/Tp 提取 prompt（§6.1）按 sub-task 段定位证据范围，严格限制"段 A 的纠正信号不能和段 B 的工具调用拼一起当依据"。

**(e) 事件层与知识层 schema 完全分离**（§8.2）——`procedural_task_trace` 是**独立表**，不进 `mem_record`。对比 v4/v5 的旧设计（把 Trace 塞 `mem_record(category=TASK_TRACE=13)`），V6 让 `mem_record` 只承载 EXPERIENCE/FEEDBACK/TOOL_PREFERENCE 三类知识。事件层 30 天 TTL 清理，知识层升格后 UPDATE，生命周期彻底独立。

**(f) Trace 不参与在线检索**（§8.2）——`memory_search` 默认过滤掉 trace 表；trace 只服务两个用途：① Fb/Tp/TSE 的统一证据源；② 离线 dreaming（跨任务模式挖掘）。agent 不会误召回到"上次执行的 raw Step 列表"。

### 1.3 对比别家

| 系统 | 证据源组织 | 痛点 |
|---|---|---|
| MemOS | 每提取器各自走 pipeline，无统一中间层 | 3N+8 次 LLM（`survey/MemOS-Skill-Implementation.md:47-52`） |
| AutoSkill | Primary Input (USER only) + Full Context (reference) 双层输入 | 每个提取器重复解析；且只看 successOnly 轨迹（`survey/AutoSkill-Pipeline-Analysis.md:52, 312`） |
| Hermes | LLM 自主决定什么时候保存哪段 | 无结构化证据层，信号主观（`survey/hermes-deep-dive-source-analysis.md:299`） |
| Mem0 | verbatim 保留完整对话 | "每步 Action + Result 原样记录"（`survey/Mem0-Procedural-Memory-Analysis.md:111`），无结构化 |
| Self-Improvement Skill | agent 按 spec 手写 Markdown | 线性膨胀无容量管理（`survey/Self-Improvement-Skill-Analysis.md:322`） |

V6 的 Task Trace 是 **procedural memory 领域少见的"中间事件层"设计**——既不是原始对话（Mem0）、也不是最终知识（AutoSkill SKILL.md），而是**可被下游多次消费的结构化事实基底**。这是 V6 能同时做到"提取便宜 + 下游精准 + 事件可审计 + 支持未来 dreaming"的根基。

---

## 2. 滑动窗口 + 主题拼接：省 token、吃得下长任务

### 2.1 核心主张

把原始对话**两段式组织**（§4, §5, §6）：

```
第一段：滑动窗口（局部结构）
   n=8 轮 / m=2 轮 overlap / step=6 轮
   每窗口抽 topic_summary（15-40 字）+ topic_keywords（3-8 个）

第二段：主题驱动拼接成 chunk（全局结构）
   Jaccard(keywords) > 0.6  → 同主题，合并
   Jaccard ≤ 0.6            → 小模型 JudgeTopicBoundary 兜底
   达 3 窗口 / 2h gap        → 强制 flush
   空主题窗口                → 门控跳过，不进 chunk
```

Chunk 才是下游提取的真正单位（§6 Chunk Flush 跑 Task Trace + Fb/Tp + TSE）。

### 2.2 为什么这么设计

**(a) 省 token**——一次 LLM 只看 8 轮窗口而非全 session。窗口小 → 主题抽取快而便宜（小模型即可）；只有真正需要提取知识时才调大模型处理整个 chunk。

**(b) 吃得下长任务**——真实"调 Docker 部署"这种任务可能横跨 20+ 轮对话，单窗口根本装不下。V6 让主题相同的窗口自动拼到一个 chunk，Task Trace 看到的是整段任务的完整上下文，而不是被切碎的 8 轮片段。

**(c) 捕捉大颗粒度**——chunk 级 task 才有"整任务级经验"的归纳价值。单窗口层面永远只能看到子事务片段，TSE 的 "bullets 描述做这类任务时应该怎么搞"需要 chunk 级视野。

**(d) 跨 tick 持久化零成本**（§10.1）——未 flush 的 chunk 只在内存，水位不推进；进程重启 chunk 缓存丢，但水位没动，下次 tick 从水位重拉 turn 自然重组成同样 chunk，**不丢数据**。不需要 DB 存 chunk 态。

**(e) Topic 缓存复用 LLM**（§5.2 `procedural_topic_cache`）——同一窗口反复重组时主题抽取结果缓存，LLM 只调一次。

**(f) 空主题门控替代启发式**（§5.5）——v5 的 `IsWindowHighQuality` 用"tool_use 存在 / 字符数 ≥ 200"启发式，不准还跟 autoCapture 的 JSON blob 冲突。V6 让主题提取的 LLM 自己判断：提不出 task（keywords=[] 且 summary=""）= 没有任务内容，直接跳过。**复用已经在跑的 LLM 调用，零额外成本**。

### 2.3 对比别家

| 系统 | 窗口策略 | 痛点 |
|---|---|---|
| MemOS TS | 固定 chunk 边界 + 话题切换触发 | 每任务 `≈ 3N + 8` 次 LLM（`survey/MemOS-Skill-Implementation.md:47-52`），N 越大越线性涨 |
| AutoSkill | 滑动窗口 `messages[-6:]` 每轮扫一遍 | 无 chunk 拼接，短任务 OK 长任务就碎；且 `successOnly=true` 成功才提（`survey/AutoSkill-Pipeline-Analysis.md:52`） |
| Mem0 | 无窗口，每次 API 调用全量处理 | 长对话 token 爆炸 |
| XSkill | 按 N=4 rollout 粒度（需真实执行） | 离线训练时 OK，但在线无等价机制 |

V6 窗口 + 主题拼接的组合，是少数能同时做到"LLM 调用便宜"和"长任务不切碎"的设计。

---

## 3. 分类 + 候选池 + 快慢车道：粒度、存储、升格节奏一体化

### 3.1 核心主张

Task Trace 产出结构化证据后，下游要做三件事：**分类**（按粒度存成什么）、**入池**（存哪里、怎么去重）、**升格**（什么时候让 agent 看到）。V6 把这三件事设计成一套耦合的方案。

**A. 三种知识类型，双级锚点**（§0 总览表）

| 类型 | 锚点 | 粒度 | 复用语义 |
|---|---|---|---|
| `task_specific_experience` (TSE) | `task`（整 chunk 的 canonical 任务） | **整任务级** | "做这类任务时怎么搞"，同 task 复用 |
| `feedback` (Fb) | `sub_task`（子事务） | **子事务级** | "做这一步时不要做 X"，**跨任务复用**——同名 sub-task 出现在新 task 里都能召回 |
| `tool_preference` (Tp) | `sub_task` | **子事务级** | "做这一步时优先用 X 工具"，跨任务复用 |

关键是 Fb/Tp 用 sub-task 而非 task 作锚：一个"写单元测试"子事务可能出现在"调 API"、"修登录 bug"、"重构模块"任何 task 里，只要同名 sub-task 再出现，`WHERE sub_task = '写单元测试'` 一查就拿到所有历史硬约束。这才是**真正的跨任务复用**。

**B. 单表候选池：主文件 + 快照两层**（§8.0.3）

V6 把三类程序记忆都放进同一张候选池表 `procedural_candidates`，确立一个关键语义：

> **Candidate 是内容的"主文件"，active `mem_record` 是周期快照。**

具体说：

- 在线 chunk flush 时**只读写 candidate**，不碰 active 层（§6.2）
- 所有 LLM 提取/合并/计数都积累在 candidate 里
- 升格时把 candidate 当前态**快照同步**给 active，让 agent 召回读到最新版本
- Agent 召回**只读 active**（candidate 可能有未验证的噪声）

单表 schema：

```sql
CREATE TABLE procedural_candidates (
  kind            TEXT CHECK(kind IN ('feedback','tool_preference','tse')),
  anchor          TEXT,  -- fb/tp: sub_task name；tse: task name
  payload         TEXT,  -- fb/tp: content；tse: bullets_json
  search_key      TEXT,  -- hybrid 召回的 embedding 输入，一次写入固化
  mention_count   INT,
  is_hard         INT,
  source_trace_ids TEXT, -- JSON array，最多 20 条 FIFO
  last_promoted_at_mention INT,
  last_promoted_record_id  INT,  -- 对应 mem_record.id，严格 1-to-1
  status          TEXT   -- 'pending' | 'promoted'
);
```

**C. 快慢两条升格通路，按信号强度分车道**（§6.2, §8.7, §13）

```
提取出 candidate
   │
   ├─ 快车道（在线立即升格）
   │    触发：fb/tp + is_hard=true
   │    来源：① 用户亲口纠正  ② agent 失败→恢复路径
   │    延迟：零（chunk flush 时同步升 active）
   │    证据检查：跳过（LLM 刚从 Trace 里标了 is_hard，
   │              让同一 LLM 再验一次是循环论证）
   │
   └─ 慢车道（离线 worker 周期升格）
        触发：fb/tp 非 hard + 所有 tse
        条件：delta = mention_count - last_promoted_at_mention ≥ 3
        延迟：1 天（每日 cron）
        证据检查：小模型读最近 5 条 source_trace 验证规则是否真成立
```

### 3.2 为什么这三件事要一起设计

**这三块互相耦合，拆开讲会失去逻辑**：分类决定了候选池的分区键（按 `kind + anchor`），候选池的两层结构决定了快慢车道的执行平面（在线写 candidate、离线 worker 快照到 active），快慢车道又回头决定了候选池的字段设计（`is_hard` / `last_promoted_at_mention`）。

下面分别说明每块的关键取舍。

**分类维度的取舍：**

- **为什么是三元而不是二元或五元**——Fb 和 Tp 语义上都是"子事务级硬约束"但一个约束"不要做什么"、一个约束"优先用什么"，合并会让召回 prompt 和升格规则糊成一团；TSE 是"整任务级归纳套路"，和 Fb/Tp 根本不在一个粒度，必须分开。再往下细分（如把 Fb 拆成"风格纠正"/"步骤纠正"）边际收益递减，LLM 也难稳定分类。
- **为什么 TSE 锚 task 而 Fb/Tp 锚 sub-task**——用户反馈往往针对具体一步（"这一步的工具不对"），而不是整个任务；但"做这类任务的经验套路"需要整 chunk 视野。锚点粒度错配会让召回要么过细（task 级 Fb 跨任务不通用）要么过粗（sub-task 级 TSE 没法归纳整任务套路）。

**候选池的取舍：**

- **为什么两层而不是一层**——candidate 里可能有刚提取但证据不足、或 0.50-0.85 灰区的条目；直接给 agent 会污染召回。只把经过升格条件的内容快照到 active，agent 读到的就是"高置信度版本"。
- **为什么 candidate 永不消亡**（§7.0 末尾 + §13.3）——升格后 `status='promoted'` 但行不删，mention_count 继续累积，下次 `delta ≥ 3` 再同步一次（UPDATE 已有 `mem_record.id`，不 INSERT 新的）。这让"同一类任务反复执行"能持续优化 bullets，而不是升过一次就冻结。
- **为什么三类合一单表**——fb/tp/tse 在 `kind` 列区分，`anchor/payload/search_key` 泛化列容纳所有类型。比起 AutoSkill / v5 的独立候选表，schema 一张、索引一套、升格 worker 一次扫全部，工程成本降一个量级。
- **Hybrid score 三档阈值**（§2.3）——查重用 `0.80·cos + 0.20·kw`：`≥ 0.85` 直接 merge（RECENCY BIAS 覆盖）/ `< 0.50` 直接 add 新 candidate / `0.50-0.85` 小模型兜底决策。灰区才调 LLM，大部分命中/未命中零 LLM 决策。
- **向量空间一致**（§8.11.2）——`procedural_candidates_vec` 和 `mem_record_raw_vec_{10,11,12}` 用**完全相同**的 search_key 作 embedding 输入。升格时 active 层 embedding 直接从 candidate 拷贝，不重算。agent 召回和在线查重共享同一向量空间，消除"candidate 层相似但 active 层不相似"的隐患。
- **次数规则替代打分公式**（§8.7）——去掉 v5 的 `support_score = 0.50·mention_ratio + 0.25·recency + 0.25·hard_ratio` 公式，改成 `delta ≥ 3` 纯规则。没有可调系数，升格时机完全可预测。

**快慢车道的取舍：**

- **强度决定节奏，不是独立维度**——"节奏"（快/慢车道）不是跟"强度"（hard/soft）正交的第二个维度，而是**强度的直接衍生**。V6 主动把两条对角线绑死：`hard → 快车道零延迟` / `soft → 慢车道累积`。剩下两种组合没意义、不支持：
  - `hard + 慢` 无意义——最强信号何必延迟；
  - `soft + 快` 无意义——弱信号进 active 会污染召回。
- **用户反馈信号最强、频次最低**——用户明确说"不要这样"就是一次判决，必须零延迟生效；放到离线 worker 等 3 次累积就失去意义。这是 Fb/Tp 的 `is_hard=true` 走快车道的根本理由。
- **TSE 是归纳出的套路**——单次 chunk 里 LLM 可能过拟合偶发现象，必须累积 ≥ 3 次 mention 才信任，且离线 worker 要读 source_trace 验证。所有 TSE 都走慢车道（TSE 没有 `is_hard` 字段，天然只有一种节奏）。
- **为什么 is_hard 在线不调证据检查**（§8.7）——is_hard 的判定来源就是刚才这次 chunk 的 Task Trace，让同一小模型读同一份 Trace 再验证一次是循环论证，无意义。
- **`delta ≥ 3` 而不是 `mention % 3 == 0`**——worker 扫描间隔 mention 可能跨多步跳（比如直接从 3 跳到 7），取模规则会漏；delta 规则保证"自上次升格以来只要累积 3 次 mention，下次扫到必升"（§8.8 跳跃场景表）。

### 3.3 对比别家

**分类粒度**

| 系统 | 粒度 | 出处 |
|---|---|---|
| memory-tencentdb | `instruction` 单一类型 | `survey/memory-tencentdb-analysis.md:446` "没有独立 procedural 类型" |
| AutoSkill | skill（task 级） | `survey/AutoSkill-Pipeline-Analysis.md` 全篇只讲 skill |
| Mem0 | 逐步骤 verbatim 快照 | `survey/Mem0-Procedural-Memory-Analysis.md:96` |

**候选池结构**

| 系统 | 结构 | 痛点 |
|---|---|---|
| AutoSkill | 独立 Skill Bank + Mirror | 同步 add/merge/discard + 4 维身份 + Capability Identity Judge 二次确认，2 次 LLM（`survey/AutoSkill-Pipeline-Analysis.md:497, 515`） |
| MemOS | Chunk 层 DUPLICATE/UPDATE/NEW + Skill 层 add/upgrade 两套 | 每任务 5-9 次 LLM |
| Mem0 | 无候选池，直接 ADD | 无去重、无演进 |
| ReMe | 向量 DB 单层 | 只做老化删除（`survey/ReMe-Procedural-Memory-Analysis.md:653`），无主动合并升格 |
| OpenViking | 类别独立桶 | "SKIP 会直接丢弃候选，无审计日志"（`survey/OpenViking_分析.md:320-321`） |

**升格节奏**

| 系统 | 策略 | 痛点 |
|---|---|---|
| Hermes | LLM 自主触发（`skill_manage`） | "no classifier, no hard trigger, no online evaluator"（`survey/hermes-deep-dive-source-analysis.md:299`）|
| AutoSkill | 每轮同步走 add/merge/discard | 无快慢分流，硬约束和软经验走同一路径 |
| ReMe | 只老化删除 | 无主动升格 |

V6 是唯一同时做了"粒度分层 × 主文件-快照两层候选池 × 快慢双车道"的设计。**粒度（task / sub_task）** 决定锚点和复用语义；**强度（hard / soft）** 决定升格节奏（hard → 快车道零延迟 / soft → 慢车道累积 ≥ 3 次证据）——强度和节奏 1:1 绑死，剩下 `hard+慢` 和 `soft+快` 两种组合没意义，不支持。复杂度都压在设计期，运行期规则极简。

---

## 4. 保证性提取：用户反馈不依赖 agent 自觉

### 4.1 核心主张

**V6 的所有提取由外部定时器驱动（每 120s 扫 `mem_conversation` 水位），agent 不参与决策、不触发、不感知。**

这是区别于 **skill-triggered 自演进程序**（Hermes / Self-Improvement Skill / Skill-Creator 等）最根本的定位差异。只要用户在对话里说了"不对，改用 X"，**下一个水位扫描周期就会把它沉淀成 Fb/Tp**——agent 不需要意识到自己被纠正了，也不需要主动调工具。

### 4.2 Skill-triggered 的根本问题：agent 可能漏记

| 系统 | 触发方式 | 漏判风险 |
|---|---|---|
| **Hermes** | agent 自调 `skill_manage` 工具 | **高**——依赖 agent 判断"值得记"；弱模型 / 长会话 / 模型注意力飘了都可能漏（`survey/hermes-deep-dive-source-analysis.md:299` "no classifier, no hard trigger"）|
| **Self-Improvement Skill** | agent 自觉 append Markdown + `activator.sh` 每轮提醒 | **中**——依赖 agent 读懂 573 行 SKILL.md 元 spec；每轮摊入 500-2k tokens |
| **Skill-Creator** | 用户说"做个技能" | **高**——用户必须主动触发，默认不记 |
| **Hermes GEPA** | 离线人工 PR 评审 | **高**——需人工介入 |
| **V6** | 外部定时器扫水位 → 6 层硬流水线 | **零**——水位不丢、扫描不漏、agent 不参与 |

**共同问题**：skill-triggered 系统把"要不要记"的决策点**塞进 agent 思考环**，等于把记忆系统的召回率绑在 agent 的自省能力上。用户给的最强信号（纠正）最容易被 agent "读过就忘"。

### 4.3 V6 的 6 层保证链

用户反馈从"说出口"到"agent 下次读到"，经过 6 层硬流水线，每层都是规则驱动、可验证：

```
用户说 "不要用 X，改用 Y"
    │
    ▼（1）mem_conversation 水位前推（§10.1）
    └─ turn 落库就前推，agent 无感；重启水位不回退
    │
    ▼（2）滑动窗口覆盖（§4）
    └─ n=8 / step=6 / overlap=2——纠正一定落在某窗口内；
       overlap 让边界纠正被两个窗口同时看到，不漏
    │
    ▼（3）topic 抽取 + chunk 拼接（§5）
    └─ 抽不到 topic 的纯闲聊窗口直接跳过（空主题门控 §5.5）；
       有纠正的窗口一定进 chunk
    │
    ▼（4）强制 flush（§5）
    └─ 达 3 窗口 / 2h gap 一定 flush；agent 事后沉默也照样触发
    │
    ▼（5）Task Trace 抽 UserCorrection（§6.0.3 / §6.0.4）
    └─ 4 类 Step 中 UserCorrection 是一等公民；
       prompt 明确列出"Type: correction" 必须抽
    │
    ▼（6）Fb/Tp 提取 + is_hard=true → 快车道（§3.1 C / §6.1）
    └─ 来自 UserCorrection 的 fb/tp 永远 is_hard=true
       → 在线立即升格到 active，下次召回就读到
```

**链路上没有任何一步问 agent "你要记这条吗"**。每层由代码或 prompt 规则强制执行，不依赖 agent 自觉。

### 4.4 这个性质为什么重要

- **用户信号是最强证据**——用户亲口纠正是程序记忆里唯一"不需要多次 mention 就可信"的信号（对应 Fb `is_hard=true` 走快车道）。漏掉一次等于浪费最高质量的学习机会
- **Agent 不能靠自己选择记什么**——agent 在长会话里注意力漂移是常态；让它自己判"值得记"本质是把一个业务决策塞进 prompt，不稳定
- **定时器 + 水位**把"该不该触发"从"**模型自省问题**"变成"**工程问题**"——工程问题可通过测试验证（水位前推了吗？窗口覆盖了 turn 吗？），模型自省问题只能靠 prompt tuning 碰运气

### 4.5 代价

这个保证性不是免费的，需要清楚：

- **周期开销**：每 120s 扫一次水位（可配置，典型 2-5% 额外 LLM 成本）
- **不能会话内主动学**：agent 当前无 API 可说"这条特别重要立刻记"；但用户纠正信号本身已最强，已覆盖核心需求
- **噪声对话也被扫**：纯闲聊 / 无操作的窗口由 §5.5 空主题门控 + §6.1 `"items": []` 兜底挡住，不污染候选池
- **延迟 ≤ 一个扫描周期 + 一个 flush 周期**：从用户说出纠正到 active 可见，最坏约 1-2 个 120s 周期

### 4.6 一句话

> **skill-triggered 把"要不要记"交给 agent 判断；V6 把"要不要记"交给定时器的硬流水线。agent 可能漏记，定时器不会。**

---

## 5. 四块如何协同

用一次典型 chunk flush 走一遍（§9 完整数据流）：

```
【中心思想 4：保证性提取】
120s 定时器扫水位
  │
  ├─ 读 mem_conversation 水位之后的新 turn（水位驱动，agent 无感）
  │
  ├─【中心思想 2：滑动窗口 + 主题拼接】
  │   生成 W0 / W1 / W2...（n=8, step=6）
  │   每窗口抽 topic_keywords + summary（小模型 + 缓存）
  │   Jaccard 拼 chunk，空主题跳过
  │   → chunk 就绪
  │
  └─ chunk flush:
       │
       ├─【中心思想 1：Task Trace 提取】
       │   ExtractTaskTrace（大模型 1 次）
       │   → 结构化 Markdown：task + sub-task + Step 四类
       │     （UserCorrection 作为一等 Step 类型，保证用户反馈信号不丢）
       │   → INSERT procedural_task_trace（独立表）
       │   → UPSERT procedural_concept（字典层）
       │
       └─【中心思想 3：分类 + 候选池 + 快慢车道】
           │
           ├─ 分类提取
           │   ① Fb/Tp：LearnFeedbackAndPreference 一次 LLM 读 Trace
           │       每条标 kind=feedback|tool_preference + sub_task 锚点 + is_hard
           │       （来自 UserCorrection 的 item 永远 is_hard=true）
           │   ② TSE：Search → Judge → Enrich/Learn 四步
           │       每条标 kind=tse + task 锚点 + bullets
           │
           ├─ 候选池入池
           │   hybrid score 查重（按 kind + anchor 严格分区）
           │   ≥ 0.85 merge / < 0.50 add / 灰区小模型兜底
           │   命中 → mention++、合并 payload
           │   未命中 → 新建 pending
           │
           └─ 升格分车道
               is_hard=true → 快车道：在线立即升格到 active（用户反馈零延迟）
               非 hard → 慢车道：留给离线 worker 按 delta ≥ 3 升格
```

四个中心思想各管一段，接口清晰：

- 思想 4 提供**触发平面**（定时器 + 水位，agent 不参与）
- 思想 2 把原始对话聚合成 **chunk**
- 思想 1 把 chunk 转成 **Task Trace + concept 字典**（下游统一证据源）
- 思想 3 从 Trace 提取、入池、升格，完成从"事件"到"知识"的跃迁

---

## 6. V6 主动不做的事

这三块是 V6 的设计取舍，不是能力缺陷，但选型时需要清楚：

- **多路径对比**——XSkill 的 N=4 rollout（`survey/XSkill-Experience-Analysis.md:158-172`）和 Skill-Creator 的 with/without skill A/B（`survey/Skill-Creator-Analysis.md:57`）能做"同任务多解法对比"，V6 当前只消费 agent 自然执行轨迹，没有多路径机制。§14 "后续迭代预留"占了坑位。
- **可执行产物**——Skill-Creator 产出 `SKILL.md + scripts + evals` 完整可执行技能包；V6 的 TSE bullets 是建议性文本（"做这类任务时应该 / 可以 / 注意 ..."），需要 agent 自己消化落地。
- **跨 host 共享**——OpenSpace 的 MCP 委派模式能让 Claude Code / Codex / Cursor 共享同一套 skill；V6 目前绑定单 user 命名空间，无跨 instance 同步路径。

这些都是**单 agent / 文本建议 / 本地部署**场景的主动简化。

---

## 7. 如果只记一句

> **V6 的核心是 Task Trace 作中间事件层把所有下游提取的成本摊到一次 LLM 调用；上游用滑动窗口 + 主题拼接喂它完整长任务；下游按粒度分类、按信号强度分车道更新候选池；而整条流水线由定时器驱动——agent 不参与、不感知、漏不了用户反馈。**

四块基石都不独立创新，但**组合起来**第一次同时解决了以下 8 个传统痛点：

1. 每提取器各扫一遍原始对话（LLM 成本线性涨）
2. 过程态和知识态混存（召回误命中 raw trace）
3. 长任务被窗口切碎（无 chunk 拼接）
4. 粒度单一（所有 procedural 都塞进 skill / instruction 单类型）
5. 硬约束生效慢（等累积 N 次才升格）
6. 软经验无证据验证（LLM 单次标注直接入库）
7. 候选和活跃层向量空间不一致（查重和召回判断矛盾）
8. **skill-triggered 系统 agent 漏记用户反馈**（Hermes / Self-Improvement Skill / Skill-Creator 都靠 agent 自觉；V6 定时器硬流水线不漏）

这是 V6 对外讲定位时最硬的差异化。
