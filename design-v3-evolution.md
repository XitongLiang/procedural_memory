# Spec 004 — 程序记忆离线自演进模块（迭代一）

> 最后更新：2026-04-13

---

## 0. 概述

### 目标

在 agent 不工作的空闲时段，对已积累的程序记忆库（workflow / experience / tool_preference）做自主演进：
- 发现并修复低质量或失效的 skill
- 优化 description 触发率
- 合并冗余 skill
- 归档旧版本（merge loser / evolve 前的历史快照）

### 与学习模块的关系

| 学习模块（Spec 003） | 演进模块（Spec 004） |
|---------------------|---------------------|
| session_end 触发，实时 | Cron 离线触发，异步 |
| 从对话中提取/升级记忆 | 对已有记忆库做审查和优化 |
| 生产原料 | 精炼原料 |

### 迭代一范围

- Cron 调度（每日定时）
- Skill 健康审计
- Description 触发率优化（借鉴 Skill-Creator）
- 内容质量演进（借鉴 Hermes GEPA 思路，LLM-as-judge）
- 冗余合并
- 旧版本归档（inactive）

---

## 1. 整体架构

```
触发：Cron 定时 / 手动 /dream 指令
输入：skills 库 + session_archive（历史会话记录）

──────────────────────────────────────────────────────────
Step 1  健康审计（零 LLM）
          → 分类：active / stale
          → 生成工作队列

Step 2  Description 触发优化（active skill，按需）
          → 自动生成触发评估集
          → 优化循环（最多 3 轮）
          ← Skill-Creator 阶段 5

Step 3  内容质量演进（active skill）
          → Session DB 挖掘：找该 skill 的使用记录
          → LLM-as-judge 质量评估
          → 失败模式分析 → 改写建议
          ← Hermes GEPA 思路

Step 4  冗余合并（active / stale skill）
          → 相似度聚类 → 合并候选对
          → 能力身份判断 → Skill Merger
          → merge loser → inactive（旧版本归档）
──────────────────────────────────────────────────────────
```

---

## 2. 参数

| 参数 | 值 | 说明 |
|------|----|------|
| `cron_schedule` | `0 3 * * *` | 每日凌晨 3 点 |
| `stale_sessions` | 30 | 超过 N 个 session 未使用 → stale（移出 agent 索引） |
| `redundancy_threshold` | 0.82 | 向量相似度超过此值进入合并候选 |
| `description_max_iterations` | 3 | Description 优化最多轮次 |
| `trigger_threshold` | 0.5 | 触发率 ≥ 50% 视为触发 |
| `eval_queries_per_skill` | 12 | 触发评估集大小（6 should + 6 not） |
| `runs_per_query` | 3 | 每个 query 测试 3 次取可靠触发率 |
| `holdout_ratio` | 0.4 | 40% 数据留做测试集（防过拟合） |
| `quality_pass_threshold` | 0.70 | LLM-as-judge 质量分 ≥ 70% 跳过 GEPA |
| `max_gepa_generations` | 3 | GEPA 最多优化代数 |
| `mutations_per_generation` | 3 | 每代并行生成的变异候选数 |
| `max_gepa_per_run` | 3 | 每次 Dreaming 最多对几个 skill 跑完整 GEPA（成本控制）|
| `gepa_min_usage` | 5 | usage_count 低于此值的 skill 跳过 GEPA |

---

## 3. Step 1 — 健康审计（零 LLM）

扫描 skills 库，为每个 skill 计算健康指标，输出工作队列。

### 3.1 指标采集

```
for each skill in skills_library:
  usage_count    ← 总使用次数（从 session_archive 统计）
  last_used_at   ← 最近一次使用的 session 序号
  sessions_since ← 当前 session 数 - last_used_at
  status         ← skill.status（active / draft）
  quality_score  ← skill.quality_score（§8.3.3 Evals 验证写入的，可能为空）
  version        ← skill.version
  size_chars     ← len(SKILL.md)
  has_evals      ← evals/evals.json 是否存在
```

### 3.2 分类规则

健康审计扫描 `status ∈ {draft, active}` 的 skill（inactive / stale 跳过）。

```
draft   ← workflow Evals 验证未达标（verified_ratio < 0.5），已入库但不在检索索引
            → 演进模块 Step 2（description 优化）→ 通过后升为 active
active  ← sessions_since ≤ stale_sessions
stale   ← sessions_since > stale_sessions
            → 移出 agent 检索索引，保留在 learning 索引等待复活
            → 演进模块不主动处理
```

### 3.3 工作队列优先级

演进模块只处理 `status = active` 的 skill。

| 层级 | 条件 | 路由 |
|------|------|------|
| P1（新增） | `added_at = today` | Step 2（description 优化）→ 通过后 Step 3 |
| P2（近期更新） | `last_evolved_at ≥ yesterday`（merge / evolve / enrich 触发） | Step 3 |
| P3（近期调用） | `last_used_at ≥ 7 天内` AND `usage_count > 0` | Step 3 |
| P4（历史循环） | `usage_count > 0` AND `status = active`，按 `last_evolved_at` 升序 | Step 3 |
| stale | `sessions_since > stale_sessions` | 演进模块跳过，等学习模块复活 |
| 任意 active | — | Step 4（冗余合并，与其他步骤并行） |

### 3.4 日选队列构建

每次 Dreaming 最多处理 **10 个 skill**：8 个新增/更新 slot（P1-P3）+ 2 个历史循环 slot（P4）。

#### 入队规则

```
stale skill → 演进模块跳过（不计入 10 个限额，等学习模块复活）

P1 slot（上限 8，优先占用）：
  新增当天的 skill，按 added_at DESC
  → status = draft → Step 2（描述优化）
  → status = active（当天由 draft 升级）→ Step 3（内容演进）

P2 slot（与 P1 共享 8 个上限，排除已在 P1 的）：
  last_evolved_at ≥ yesterday（merge / evolve / enrich 操作写入）
  → Step 3

P3 slot（填满剩余 8 个 slot，排除已在 P1/P2 的）：
  last_used_at ≥ 7 天内 AND usage_count > 0
  → Step 3

P4 slot（独立 2 个，排除已在 P1-P3 的）：
  usage_count > 0 AND status = active
  从未使用（usage_count = 0）的 skill 不进入 P4
  按 last_evolved_at ASC（最久未演进的优先）
  → Step 3
```

跨层去重：每个 skill 只能以最高优先级进入一次，不重复处理。

#### 溢出处理

当天 P1 新增 skill 数超过 8 时：

- 8 个 slot 内按 added_at DESC 处理
- 超出部分持久化写入 `pending_queue`
- 次日 Dreaming 启动时，`pending_queue` 中的 skill 以 P1 优先级插入队首

```sql
CREATE TABLE IF NOT EXISTS pending_queue (
  skill_id   TEXT PRIMARY KEY,
  priority   TEXT    NOT NULL,   -- P1 / P2 / P3
  queued_at  INTEGER NOT NULL,
  reason     TEXT               -- 溢出原因
);
```

---

## 4. Step 2 — Description 触发优化（借鉴 Skill-Creator 阶段 5）

针对 draft skill（Evals 验证未达标），自动优化 description 使其能被正确召回。

**核心洞察**（来自 Skill-Creator）：
- description 太窄 → 该触发时不触发（undertrigger，更常见）
- description 太宽 → false trigger
- description 包含流程摘要 → agent 按摘要走捷径，跳过完整 skill

### 4.1 生成触发评估集（小模型）

自动生成 eval_queries，不需要人工审核。

```
[System]
为以下 skill 生成触发率评估查询集。

生成 {eval_queries_per_skill} 个查询，均分两类：

should_trigger（应触发，{n/2} 个）：
- 用户真正需要这个 skill 能力的场景
- 多种表述：直接询问 / 间接描述 / 不说技能名
- 包含具体细节：路径、工具名、随机上下文
- 混合正式/口语风格，可含拼写简写

should_not_trigger（不应触发，{n/2} 个）：
- 关键：近似但不相关，不是明显无关
- 同领域但不需要这个具体 skill 的任务
- 能区分 skill 边界的用例

❌ 坏查询（太抽象）："处理文件" / "执行任务"
✅ 好查询（具体细节）："我boss让我把这个xlxs的C列和D列做差然后加到E列"

只输出 JSON：
[{"query": "...", "should_trigger": true|false}]

[User]
Skill name: {name}
Skill description: {description}
Skill context: {context}
```

### 4.2 触发率测试（零 LLM，并行）

直接通过 openclaw 的 `memory_search` MCP 接口测试召回，不启动 Claude CLI：

```
run_single_query(query, target_skill_id, top_k=5):
  → MCP 调用：memory_search(query=query, topK=top_k)
  → 解析 results[]，查找 target_skill_id
  → triggered = (skill 在 results 中 且 score ≥ recallMinScore)
  → 返回 {triggered, score, rank}

  注：召回是确定性的（同 embedding + 同 query → 同结果），
  无需重复多次，每个 query 跑一次即可。

run_eval(skill_id, evals):
  for e in evals:
    r = run_single_query(e.prompt, skill_id)
    passed = r.triggered     if e.should_trigger
             NOT r.triggered if NOT e.should_trigger
  score = passed_count / total
  返回 {score, details[]}    ← details 含每条 query 的 score/rank，供优化循环使用
```

> **为什么不用 `claude -p` 测？**
> openclaw 架构中 skill 触发分两层：L1 召回层（memory_search 向量+关键词打分，确定性）
> 和 L2 使用层（agent 收到注入后是否遵循，非确定性）。
> description 优化（§4.3）改善的是 L1 召回，直接测 memory_search 更准确且快得多。

数据集拆分（防过拟合，借鉴 Skill-Creator）：
```
queries 按 should_trigger/should_not_trigger 分层
→ 60% train / 40% holdout（test）
→ 每组至少 1 个进入 test
```

### 4.3 优化循环（小模型，最多 3 轮）

```python
for iteration in 1..description_max_iterations:

  run_eval(train + test)  # 并行测试
  train_score, test_score = aggregate_results()

  history.append({
    "iteration": iteration,
    "description": current_description,
    "train_score": train_score,
    "test_score": test_score
  })

  if train 全部通过: break
  if 最后一轮: break

  current_description = improve_description(
    train_results,
    blinded_history  ← 不含 test 分数，防过拟合
  )
```

**improve_description Prompt**（小模型）：

```
[System]
你是一个 skill description 优化专家。
根据触发测试失败情况改写 description，使 skill 在正确场景触发、在不相关场景不触发。

要求：
- 祈使句开头："当...时使用" / "用于..."
- 列举触发场景和关键词，不描述实现细节
- 稍微 "主动" 一点（agent 倾向于 undertrigger）
- 不超过 150 字

改写规则：
- should_trigger 失败 → description 太窄，补充更多触发场景和同义表达
- should_not_trigger 失败 → description 太宽，收窄范围或加排除条件

只输出新 description 字符串。

[User]
当前 description：{current_description}

失败用例：
{failed_queries_with_reasons}

优化历史（不含 test 分数）：
{blinded_history}
```

### 4.4 选择最佳描述并写入

```
best = max(history, key=lambda h: h["test_score"])
                              ↑ 按 TEST 分数选，不按 TRAIN 分数

if best.test_score > original.test_score:
  备份原 SKILL.md → 更新 frontmatter description → 记录 DESCRIPTION_EVOLVED
else:
  记录 DESCRIPTION_NO_IMPROVEMENT
```

---

## 5. Step 3 — 内容质量演进（Hermes GEPA）

针对 active skill，构建评估数据集，通过多代变异+实际执行测试选出最优版本。

### 5.1 构建评估数据集

两个来源合并，数据不足（< 5 条）→ 跳过，记录 `CONTENT_SKIP(no_evidence)`：

```
来源 a：skill 的 evals/evals.json（LearnSkill §8.3.3 时已生成）
  → 取其中 task_input 字段作为测试任务

来源 b：session_archive 挖掘（零 LLM）
  session_archive WHERE skill_id = {skill.id} AND skill_used = true
  ORDER BY session_id DESC LIMIT 20
  → 提取 user 消息作为 task_input
  → 同时保存 tool_uses / 报错记录 / 用户纠正（作为执行轨迹参考）

合并去重后按 6:4 分层拆分：
  train_tasks（60%）← 优化过程中使用
  holdout_tasks（40%）← 仅用于最终选版，防过拟合
```

### 5.2 基线质量评估（小模型）

对当前 SKILL.md 在 train_tasks 上跑 batch_runner，得到基线分：

```
overall ≥ quality_pass_threshold（0.70）→ 记录 CONTENT_OK，跳过 GEPA
overall < 0.70 → 进入 §5.3 GEPA 优化循环
```

LLM-as-judge 评分维度（0-1）：
```
1. 遵循流程：agent 是否按 skill 步骤执行？
2. 输出正确性：任务结果是否正确/有用？
3. 简洁度：是否在合理步骤数内完成？
overall = 均值
```

### 5.3 GEPA 优化循环（最多 `max_gepa_generations` 代）

```python
current_skill = original_skill
history = []

for generation in 1..max_gepa_generations:

  # Step 1：生成语义变异候选（大模型，并行）
  candidates = generate_mutations(
    skill=current_skill,
    failure_patterns=failure_patterns,  # 从 batch_runner 轨迹提取
    n=mutations_per_generation          # 默认 3 个
  )

  # Step 2：batch_runner 并行测试每个候选（Claude CLI）
  for candidate in candidates:
    traces = batch_run(skill=candidate, tasks=train_tasks)
    candidate.train_score = llm_judge(traces)
    candidate.failure_patterns = extract_failures(traces)

  # Step 3：选本代最佳，用 holdout 验证
  best = max(candidates, key=lambda c: c.train_score)
  holdout_traces = batch_run(skill=best, tasks=holdout_tasks)
  best.test_score = llm_judge(holdout_traces)

  history.append(best)
  if best.train_score >= 0.90: break   # 提前收敛
  current_skill = best                 # 下一代从最佳出发

# 按 holdout test_score 选最终版本（不按 train，防过拟合）
final = max(history, key=lambda h: h.test_score)
```

**变异生成 Prompt**（大模型）：

```
[System]
你是一个 workflow skill 演进专家。
基于执行轨迹中的失败模式，生成 {n} 个语义变异版本的 SKILL.md。

变异原则：
- 每个候选从不同角度修复失败模式（不是随机改写）
- 从失败模式泛化，不过拟合到特定用例
- 保持 skill 精简，删除不起作用的部分
- 解释 WHY，不堆 MUST/ALWAYS/NEVER
- 重复出现的辅助步骤 → 提取为 scripts/

修复方向示例（每个候选选其中一到两个方向）：
- missing_steps → 补充步骤（判断是否足够通用）
- wrong_steps → 精确修正（不改动无关部分）
- missing_notes → 添加注意事项（❌ 错误做法 → 原因 → ✅ 正确做法）
- 结构优化 → 合并冗余步骤，提升清晰度

只输出 JSON 数组：
[
  {
    "candidate_id": 1,
    "approach": "修复方向说明",
    "content": "完整新 SKILL.md",
    "changes": ["修改了什么及原因"]
  },
  ...
]

[User]
当前 SKILL.md（v{VERSION}）：
{skill_content}

执行轨迹失败模式：
{failure_patterns}
```

### 5.4 约束验证 + 写入

```
final.test_score > baseline.test_score？
  否 → 记录 CONTENT_NO_IMPROVEMENT

  是 → 约束验证：
        ① Skill 大小 ≤ 原来 +20%
        ② 规则验证（frontmatter / steps / 去标识化）
        ③ holdout 全量任务通过率 ≥ 基线

        通过 → 备份原版本（→ inactive）→ 写入 final
               触发增量 scripts / references 更新（同 §8.3.2）
               版本注释：<!-- v{N}: gepa(content) gen:{G} session:{RANGE} -->
               记录 CONTENT_EVOLVED
        失败 → 恢复备份，记录 CONTENT_VALIDATION_FAILED
```

---

## 6. Step 4 — 冗余合并（与其他步骤并行）

### 6.1 相似度聚类（零 LLM）

```
取所有 skill 的 embedding（来自入库时存储的向量）

两两计算余弦相似度：
  combined_score = 0.70 × 向量相似度
                 + 0.18 × 关键词重叠
                 + 0.12 × name token Jaccard

combined_score ≥ redundancy_threshold（0.82）
  → 加入合并候选对列表
  → 按 score 降序排列
```

### 6.2 能力身份判断（小模型）

同 Spec 003 §8.5 阶段二，确认是否同一能力：

```
same_capability=true  → 进入 §6.3 合并执行
same_capability=false → 跳过（相似但不同能力）
```

### 6.3 合并执行（大模型）

同 Spec 003 §8.5 阶段三，语义合并两个 SKILL.md：

```
合并策略：
- 取 usage_count 更高的作为"主 skill"（保留其 name/id）
- 另一个作为"候选 skill"（内容并入后归档为旧版本）
- 触发条件/步骤/注意事项：语义并集去重
- description：合并语义后重写为主动触发式

版本注释：<!-- v{N}: merge({merged_skill_name}) session:{DATE} -->
```

备份主 skill（旧版本 → inactive）→ 写入合并结果 → 候选 skill → inactive → 记录 `MERGED`

**inactive 语义**：旧版本归档记录，不可 promote 回 active（已被更新版本取代）。保留在 DB 供版本追溯，不出现在任何检索索引中。

---

## 7. Skill 状态机

演进模块涉及的状态及转换关系：

```
  draft ──────────────────────────────────────► inactive（优化失败，test_score 仍 < 0.30）
    │   Step 2 description 优化通过
    ▼
  active ──────────────────────────────────────► stale
    │   sessions_since > stale_sessions              │
    │   （移出 agent 索引，保留 learning 索引）         │
    │                                                 │
    │  Step 3 evolve（内容演进）                       │ 学习模块 merge/evolve/dedup 命中
    │    → 旧版本快照 → inactive                       │   → last_evolved_at 更新
    │                                                 │   → sessions_since 重置
    │  Step 4 merge（合并）                            └──► active（复活）
    │    → 主 skill 旧版本 → inactive
    └──► 候选 skill → inactive
```

#### inactive 语义

inactive = **旧版本归档**，不是状态流转的一个节点，而是每次写操作的副产物：

| 触发时机 | 产生的 inactive 记录 |
|---------|-------------------|
| Step 3 内容演进写入新版本 | 演进前的旧版本快照 |
| Step 4 合并，主 skill 更新 | 主 skill 合并前的旧版本 |
| Step 4 合并，候选 skill 被吸收 | 候选 skill 整体 |

inactive 记录：
- 文件保留原路径（不删除、不移动）
- 不出现在任何检索索引（agent 索引 / learning 索引均不可见）
- 保留在 DB 供版本追溯
- **不可 promote 回 active**（已被更新版本取代）

---

## 8. 审计日志

每次演进操作写入 `evolution_log`：

```sql
CREATE TABLE evolution_log (
  id            TEXT PRIMARY KEY,
  run_id        TEXT NOT NULL,        -- 本次 Dreaming 运行 ID
  skill_id      TEXT NOT NULL,
  skill_name    TEXT NOT NULL,
  step          TEXT NOT NULL,        -- description_optimize / content_evolve / merge
  action        TEXT NOT NULL,        -- EVOLVED / NO_IMPROVEMENT / SKIP / MERGED / ARCHIVED
  version_from  TEXT,
  version_to    TEXT,
  detail        JSON,                 -- 改了什么 / 分数变化 / 合并对象
  created_at    INTEGER NOT NULL
);
```

每次 Dreaming 结束输出摘要：

```
Dreaming run {run_id} — {date}
  总耗时：{duration}
  审计 skill 数：{total}

  Description 优化：
    改善 {n}，无改善 {n}，跳过 {n}
    平均触发率：{before}% → {after}%

  内容演进：
    改写 {n}，质量达标跳过 {n}，证据不足跳过 {n}

  冗余合并：
    合并对数 {n}，合并执行 {n}，身份不同跳过 {n}
    产生 inactive 旧版本记录 {n} 条
```

---

## 9. 调度配置

```yaml
dreaming:
  enabled: true
  schedule: "0 3 * * *"            # 每日凌晨 3 点

  slots:
    total: 10
    new_slots: 8                   # P1-P3 共享
    historical_slots: 2            # P4 独立（usage_count > 0 的 active skill）

  priority:
    P1_new_today: true             # added_at = today
    P2_recently_evolved: true      # last_evolved_at >= yesterday
    P3_recently_called:
      window_days: 7               # last_used_at 在 7 天内
    P4_historical:
      order_by: last_evolved_at    # ASC，最久未演进的优先
      require_usage: true          # usage_count > 0，status = active

  overflow:
    persist_queue: true            # 超出 slot 写入 pending_queue
    next_day_priority: P1          # 次日以 P1 优先级插入队首

  steps:
    description_optimize: true
    content_evolve: true
    redundancy_merge: true
  dry_run: false                   # true = 只记录不写入（调试用）
```

手动触发（不等 cron）：`/dream` 或 `/dream --skill {name}` 仅演进指定 skill。

---

## 10. 模型分配

| 调用 | 模型 | 理由 |
|------|------|------|
| 触发评估集生成 | 小模型 | 模板化生成，模式固定 |
| improve_description | 小模型 | 局部改写，范围明确 |
| LLM-as-judge 质量评估 | 小模型 | 结构化评分，输入已精简 |
| 内容演进（RefineSkill） | 大模型 | 改写质量要求高 |
| 能力身份判断 | 小模型 | 二分类 |
| Skill Merger（冗余合并） | 大模型 | 语义合并，质量要求高 |

---

## 11. 参考来源

| 设计点 | 来源 |
|--------|------|
| Cron 调度基础设施 | Hermes Cron Engine §5.3 |
| Gateway Flush（空闲/定时触发审查） | Hermes §5.1 |
| Session DB 挖掘构建 eval 数据集 | Hermes GEPA §5.6 Step 2b |
| 执行轨迹驱动的失败模式分析 | Hermes GEPA 核心创新 |
| LLM-as-judge 打分（3 维度） | Hermes GEPA §5.6 Step 5 |
| 约束验证（不超大小 +20%，不回退） | Hermes GEPA §5.6 Step 6 |
| Description 触发率优化整体框架 | Skill-Creator 阶段 5 §5.2 |
| 评估集分层拆分（train/test holdout） | Skill-Creator run_loop.py §5.4 |
| 触发率测试实现（runs_per_query=3） | Skill-Creator run_eval.py §5.5 |
| 按 test 分数选最佳（防过拟合） | Skill-Creator §5.4 |
| 4 条内容改进原则（泛化/精简/WHY/提取） | Skill-Creator §4.5 |
| 内容优化 with_skill vs without_skill | Skill-Creator 阶段 4 |
| 能力身份判断 + Skill Merger | AutoSkill 维护管线 §8.3-§8.4 |
| 三路相似度冗余检测 | AutoSkill + 自研 |

---

## 12. 迭代二预留

- **跨 skill 归纳**：发现多个 experience/skill 中的共同模式 → 生成更高层的 skill
- **Hindsight Reflect 接入**：跨记忆综合推理（如 Hermes Hindsight §5.4）
- **Darwinian Code Evolver**：对 scripts/ 中的脚本做代码级演进
- **用户确认模式**：dry_run 后向用户展示摘要，需确认才写入
