# OpenSpace 自进化技能引擎全流程分析

> 基于 https://github.com/HKUDS/OpenSpace 源码（2026-04 main 分支）
> 项目定位：Make Your Agents Smarter, Low-Cost, Self-Evolving
> 主页：[open-space.cloud](https://open-space.cloud)

---

## 一、OpenSpace 是什么

一个以 **MCP Server** 形式对接任何支持 SKILL.md 的宿主 agent（Claude Code / Codex / OpenClaw / nanobot / Cursor 等），让宿主自动获得"可自演进 procedural memory"的通用框架。自身并不是独立 agent，而是以 **4 个 MCP 工具 + 共享云端技能库**的方式被宿主调用。

核心特点：**三触发 × 三操作 + 两层质量监控 + 级联演进 + 云端共享**。

跟其他系统的关键区别：

| 维度 | OpenSpace | AutoSkill | Skill-Creator | MemOS | Hermes |
|------|-----------|-----------|---------------|-------|--------|
| **宿主集成方式** | **MCP 统一接入**（跨 agent 复用） | OpenClaw 插件 | Claude Code 插件 | TS-end 内置 | Hermes 专属 |
| **演进类型** | **FIX / DERIVED / CAPTURED 3 种** | add / merge / discard | 人工迭代 | add/merge/upgrade | LLM create/patch |
| **触发来源** | **3 个独立触发器**（分析/工具降级/周期指标） | agent_end | 用户主动 | agent_end | LLM 自主 |
| **质量监控** | **Skill 层 4 比率 + Tool 层 penalty** | 使用审计闭环 | eval pass_rate | 0-10 评分 | 无 |
| **级联演进** | **有**（Tool 降级 → 依赖 skill 批量修复） | 无 | 无 | 无 | 无 |
| **版本模型** | **版本 DAG**（支持多父 merge） | 线性版本 | iteration-N/ | 版本号递增 | patch 覆盖 |
| **云端协作** | **有**（public/private + diff 凭证） | 无 | 无 | 无 | 无 |

### 1.1 三种演进模式（Evolution Types）

来自 [openspace/skill_engine/types.py:26-33](../tmp/openspace-repo/openspace/skill_engine/types.py#L26-L33)：

```python
class EvolutionType(str, Enum):
    FIX      = "fix"       # 修复过时/错误的 skill 指令（原地覆盖，同 name 同 path）
    DERIVED  = "derived"   # 派生增强/专用版本（新 name，新目录）
    CAPTURED = "captured"  # 捕获全新可复用模式（新目录，无父节点）
```

对应到版本 DAG 的节点类型：

```
IMPORTED / CAPTURED → 根节点，parent_skill_ids = []，generation = 0
DERIVED             → 1+ 父，多父=merge，generation = max(parents) + 1
FIXED               → 严格 1 父（前一个版本），同 name 同 path，
                      仅最新版本 is_active=True
```

### 1.2 三个独立触发器（Evolution Triggers）

[evolver.py:113-118](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L113-L118)：

```python
class EvolutionTrigger(str, Enum):
    ANALYSIS         = "analysis"           # 每次任务后的 LLM 回放分析
    TOOL_DEGRADATION = "tool_degradation"   # 工具成功率下降
    METRIC_MONITOR   = "metric_monitor"     # 周期扫描 skill 健康度
```

三个触发器共享同一个入口 `SkillEvolver.evolve(ctx)`，由 `EvolutionContext` 统一描述差异，架构上非常简洁。

### 1.3 两种知识层面的互补

```
Skill 层面（执行蓝图）：
  可执行的 SKILL.md + scripts/ + references/ 多文件结构
  LLM 在 Context 中直接消费 → 指导 agent 的下一步动作

Tool Quality 层面（工具健康度）：
  每个工具的成功率 / 最近成功率 / 连续失败 / penalty 系数
  驱动 Search 层重排 + 驱动 skill 级联演进
```

OpenSpace 独特之处在于两层会"互相发信号"——工具质量下滑会触发依赖它的 skill 修复；skill 执行分析中 LLM 标注的 tool_issues 会注入 ToolQualityManager，形成完整闭环。

---

## 二、产物形态（Skill Artifacts）

### 2.1 SKILL.md 目录结构

每个 skill 是一个目录，最少包含 `SKILL.md`，可选 `scripts/`、`references/` 等辅助文件。Frontmatter 遵循 Anthropic 官方规范，只保留 **name + description**：

```yaml
---
name: document-gen-resilient-workflow
description: End-to-end document generation with multi-engine PDF fallback and post-write verification
---

# Resilient Document Generation

## When to Use
...

## Workflow
1. **Parse**: ...
2. **Generate**: ...
3. **Verify**: ...
```

### 2.2 skill_id sidecar

`skill_id` **不写在 frontmatter**，而是写在目录内的 `.skill_id` 边文件（[registry.py:44-70](../tmp/openspace-repo/openspace/skill_engine/registry.py#L44-L70)）：

- 首次发现 → `{name}__imp_{uuid8}`
- FIX 后 → `{name}__v{gen}_{uuid8}`（同目录覆盖，新 ID）
- DERIVED / CAPTURED → 新目录，新 ID

**好处**：目录搬移、跨机器同步时 skill_id 保持稳定；frontmatter 对外保持干净，供 Anthropic 官方工具链直接消费。

### 2.3 三个核心数据类型

来自 [types.py](../tmp/openspace-repo/openspace/skill_engine/types.py)：

**`SkillLineage`**（第 69-159 行）——版本 DAG 节点：

| 字段 | 说明 |
|------|------|
| `origin` | IMPORTED / CAPTURED / DERIVED / FIXED |
| `parent_skill_ids` | DAG 上的父节点列表（多父=merge） |
| `generation` | 到根的距离 |
| `content_diff` | 统一 unified diff（给人读） |
| `content_snapshot` | `{relative_path: content}` 全目录快照（机器恢复） |
| `change_summary` | LLM 生成的变化描述（如 "Fixed curl parameter format in step 3"） |

`content_diff` 与 `content_snapshot` 互补：diff 便于人类追溯变化原因，snapshot 保证任何时间点都能恢复到历史版本。

**`SkillRecord`**（第 333-464 行）——主记录：

```python
@dataclass
class SkillRecord:
    skill_id: str
    name: str
    description: str
    path: str

    is_active: bool = True           # 仅最新版本激活
    category: SkillCategory          # TOOL_GUIDE / WORKFLOW / REFERENCE
    visibility: SkillVisibility      # PRIVATE / PUBLIC
    lineage: SkillLineage

    tool_dependencies: List[str]     # 所有涉及的 tool key
    critical_tools: List[str]        # 必需 tool key

    # 四个原始计数器（SQL 层原子递增）
    total_selections: int = 0
    total_applied: int = 0
    total_completions: int = 0
    total_fallbacks: int = 0
```

派生四项核心比率（`@property`）：

```python
applied_rate    = total_applied    / total_selections   # 选了后真用了吗
completion_rate = total_completions / total_applied      # 用了后完成了吗
effective_rate  = total_completions / total_selections   # 端到端有效性
fallback_rate   = total_fallbacks   / total_selections   # 选了没用（撤销信号）
```

**`EvolutionSuggestion`**（第 197-248 行）——LLM 分析结果：

| 字段 | 约束 |
|------|------|
| `evolution_type` | FIX / DERIVED / CAPTURED |
| `target_skill_ids` | FIX=1 个 / DERIVED=1+ 个 / CAPTURED=0 个 |
| `category` | 新 skill 的分类 |
| `direction` | 演进方向描述（供下游 prompt） |

---

## 三、触发 1：Post-Execution Analysis（任务后分析）

### 3.1 整体流程

每次任务 `finally` 块里自动触发。由 `ExecutionAnalyzer` 发起 LLM 回放分析，输出结构化 JSON 建议，交给 `SkillEvolver` 并行执行。

```
任务完成
    │
    ├─ Step 1: 加载 recording artifacts
    │    metadata.json + conversations.jsonl + traj.jsonl
    │
    ├─ Step 2: 构造 analysis prompt
    │    向 LLM 展示任务上下文 + 选中的 skills + 工具调用时间线
    │
    ├─ Step 3: Analysis agent loop
    │    LLM 可调 read_file / list_dir / run_shell 收集证据
    │    最后一轮关闭工具强制输出结构化 JSON
    │
    ├─ Step 4: 解析为 ExecutionAnalysis
    │    task_completed / skill_judgments / tool_issues / evolution_suggestions
    │
    ├─ Step 5: 持久化到 SQLite
    │    原子更新 SkillRecord 四个计数器
    │
    ├─ Step 6: 反哺 ToolQualityManager
    │    LLM 标注的 tool_issues → ToolQualityRecord 失败计数
    │
    └─ Step 7: SkillEvolver.process_analysis()
         每个 evolution_suggestion → 一个 EvolutionContext
         并行跑 evolution agent loop（semaphore 节流）
```

### 3.2 分析 Prompt（`_EXECUTION_ANALYSIS_TEMPLATE`）

来自 [openspace/prompts/skill_engine_prompts.py:157-348](../tmp/openspace-repo/openspace/prompts/skill_engine_prompts.py#L157-L348)。输出结构化 JSON，关键字段：

```json
{
  "task_completed": true | false,
  "skill_judgments": [
    {"skill_id": "...", "status": "applied" | "fallback" | "ineffective"}
  ],
  "tool_issues": [
    {"tool_key": "...", "issue": "timeout / wrong_output / ..."}
  ],
  "evolution_suggestions": [
    {
      "evolution_type": "fix" | "derived" | "captured",
      "target_skill_ids": [...],
      "direction": "..."
    }
  ]
}
```

### 3.3 Skill ID 纠错

LLM 经常把 skill_id 的 hex 后缀搞错（把 `cb` 写成 `bc`），分析器用 **Levenshtein 距离做前缀匹配 + 编辑距离纠错**（[analyzer.py:59-106](../tmp/openspace-repo/openspace/skill_engine/analyzer.py#L59-L106)）。这是让 LLM 输出可用的关键工程细节——纯靠 prompt 约束无法避免拼写波动，必须在后处理层兜底。

### 3.4 Tool Quality 反哺

[analyzer.py:275-327](../tmp/openspace-repo/openspace/skill_engine/analyzer.py#L275-L327) 中的 `_record_tool_quality_feedback` 把 LLM 的 `tool_issues` 注入 `ToolQualityManager`，配合基于规则的成功/失败计数，共同维护 tool 质量曲线：

```
规则计数（客观，rule-based）──┐
                             ├─► ToolQualityRecord.recent_success_rate
LLM 标注（语义，dedup 避免重复）┘
```

这是触发 2 能工作的基础——没有 LLM 标注，很多"语义失败"（比如返回了结果但结果错）在规则层看起来是成功的。

---

## 四、触发 2：Tool Degradation（工具降级级联修复）

### 4.1 流程

[evolver.py:310-438](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L310-L438) 的 `process_tool_degradation()`：

```
ToolQualityManager.get_problematic_tools(threshold=0.5, min_calls=5)
    │
    ▼ 返回成功率 < 50% 且调用 ≥ 5 次的工具
    │
    for each degraded_tool:
        │
        ▼ 反 loop：已修过的 skill 跳过
        │  _addressed_degradations[tool_key] 记录已处理集合
        │  tool 若从列表中消失（恢复）→ 清除对应条目
        │
        ▼ store.find_skills_by_tool(tool_key)
        │  查询依赖该 tool 的所有 active skill
        │
        ▼ 规则筛选（宽松阈值）
        │
        ▼ LLM 二次确认（_EVOLUTION_CONFIRM_TEMPLATE）
        │  从 4 个维度判断：信号真实性 / skill 坏了吗 /
        │  修的代价值不值 / 方向对不对
        │
        ▼ 通过的 skill 进入 FIX 流程
```

### 4.2 Tool Quality Penalty 曲线

[grounding/core/quality/types.py:107-140](../tmp/openspace-repo/openspace/grounding/core/quality/types.py#L107-L140) 的 `ToolQualityRecord.penalty` 把最近成功率映射到 `[0.2, 1.0]`：

```
success_rate >= 0.4 → penalty = 1.0（不惩罚）
否则                → base = 0.3 + (rate / 0.4) * 0.7
连续失败 ≥ 3 次      → 再额外 -0.1/次，封顶 -0.3
```

penalty 参与 tool search 的重排，让失败的 tool 在后续 retrieval 中降权。这是"tool 质量→search→skill 选择"的前端联动；而"tool 质量→skill 演进"是后端联动，两条线共同维护系统健康。

### 4.3 反 Loop 设计

两类反 loop（文档见 [evolver.py:167-183](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L167-L183)）：

- **状态驱动**（Trigger 2）：`_addressed_degradations: Dict[tool_key, Set[skill_id]]`，tool 恢复自动清除。
- **数据驱动**（Trigger 3）：新 skill `total_selections = 0`，自然需要 `min_selections=5` 个新样本才能被再次评估。无需基于时间的冷却期。

---

## 五、触发 3：Metric Monitor（周期健康扫描）

### 5.1 健康诊断规则

[evolver.py:1563-1598](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L1563-L1598) 的 `_diagnose_skill_health()`：

```python
# 规则 1：选了没用 → 指令不清 → FIX
if fallback_rate > 0.4:
    return FIX, "High fallback rate ..."

# 规则 2：用了常失败 → 指令错 → FIX
if applied_rate > 0.4 and completion_rate < 0.35:
    return FIX, "Low completion rate despite high applied rate ..."

# 规则 3：还行但可以更好 → DERIVED
if effective_rate < 0.55 and applied_rate > 0.25:
    return DERIVED, "Moderate effectiveness ..."

return None, ""  # healthy
```

阈值**刻意放宽**，因为后面有 LLM 二次确认挡误报。

### 5.2 流程

[evolver.py:441-517](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L441-L517)：

```
for each active skill:
    if total_selections < min_selections(5): skip
    evo_type, direction = _diagnose_skill_health(record)
    if evo_type is None: skip
    if not _llm_confirm_evolution(...): skip
    build EvolutionContext → append to confirmed_contexts

并行执行 confirmed_contexts（semaphore 节流）
```

---

## 六、Evolution Agent Loop

### 6.1 统一入口

[evolver.py:234-263](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L234-L263) 的 `evolve()` 根据 evolution_type 分派到 `_evolve_fix` / `_evolve_derived` / `_evolve_captured`，但三者共享同一个 agent loop 框架：

```
_run_evolution_loop()
    │
    ├─ Iteration 1 ~ N-1：
    │    LLM 可调 read_file / list_dir / run_shell / web_search
    │    探索代码库、查询文档、理解 skill 当前状态
    │
    ├─ Iteration N（最后一轮）：
    │    工具关闭，强制 LLM 输出 patch / full / diff
    │    以 <EVOLUTION_COMPLETE> 或 <EVOLUTION_FAILED> 结尾
    │
    └─ Apply with retry（最多 3 次）：
         失败 → 错误 + 磁盘当前内容 回喂 LLM
```

关键常量（[evolver.py:102-103](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L102-L103)）：

```python
_MAX_EVOLUTION_ITERATIONS = 5   # Agent 循环最多 5 轮
_MAX_EVOLUTION_ATTEMPTS   = 3   # Apply 最多 retry 3 次
```

### 6.2 五个 Prompt 模板

集中在 [openspace/prompts/skill_engine_prompts.py](../tmp/openspace-repo/openspace/prompts/skill_engine_prompts.py)：

| Prompt | 作用 | 关键指令 |
|--------|------|----------|
| `_EXECUTION_ANALYSIS_TEMPLATE` (157-348) | Trigger 1 源头 | 输出结构化 JSON，含 tool_issues + evolution_suggestions |
| `_EVOLUTION_FIX_TEMPLATE` (351-484) | FIX 执行 | **优先用 Patch 格式**，覆盖 > 60% 才用 Full |
| `_EVOLUTION_DERIVED_TEMPLATE` (487-631) | DERIVED 执行 | 新 name ≤ 50 字符，小写加连字符，**不要 `-enhanced-enhanced` 链** |
| `_EVOLUTION_CAPTURED_TEMPLATE` (634-740) | CAPTURED 执行 | LLM 自评泛化价值，没有就返回 `<EVOLUTION_FAILED>` |
| `_EVOLUTION_CONFIRM_TEMPLATE` (743-803) | Trigger 2/3 确认 | 4 维度决策，歧义默认 skip |

### 6.3 Apply-Retry 的关键工程

[evolver.py:1271-1424](../tmp/openspace-repo/openspace/skill_engine/evolver.py#L1271-L1424) 的 `_apply_with_retry()`——当首次 apply 失败（parse error / anchor 找不到 / 路径越界），**把磁盘当前内容回喂 LLM**：

```python
retry_prompt = f"The previous edit was not successful...\n{error_msg}\n"
if current_on_disk:
    retry_prompt += (
        f"Here is the CURRENT content of the skill files on disk "
        f"(use this as the ground truth for any SEARCH/REPLACE or "
        f"context anchors):\n\n{current_on_disk}\n\n"
    )
```

**为什么重要**：LLM 常常基于记忆中过时的内容生成 patch，导致 anchor 对不上。把真实磁盘内容塞回去等于强制它"重新读题"，成功率显著提升。

失败的 DERIVED / CAPTURED 目录在下次尝试前 `shutil.rmtree` 清理，避免脏状态。

---

## 七、Patch / Diff 机制

### 7.1 三种 Patch 格式

[openspace/skill_engine/patch.py:334-365](../tmp/openspace-repo/openspace/skill_engine/patch.py#L334-L365) 的 `detect_patch_type()` 按特征识别：

| 格式 | 触发标记 | 适用场景 |
|------|---------|---------|
| **PATCH** | `*** Begin Patch` | 多文件 diff，最省 token |
| **FULL** | `*** Begin Files` / `*** File:` | 多文件整块覆盖 |
| **DIFF** | `<<<<<<< SEARCH` | 单文件 SEARCH/REPLACE |
| **FULL**（默认） | 无标记 | 单文件整块覆盖 |

### 7.2 PATCH 格式的 @@ 锚点语法

与 unified diff 不同，PATCH 格式使用"原文行作为锚点 + `-/+/ ` 前缀"：

```
*** Begin Patch
*** File: SKILL.md
@@
 ## Workflow
-1. Use curl with --data flag
+1. Use curl with --data-raw flag (--data may URL-encode)
 2. Parse JSON response
*** End Patch
```

### 7.3 4 级降级匹配

定位算法 `seek_sequence()`（[patch.py:498-538](../tmp/openspace-repo/openspace/skill_engine/patch.py#L498-L538)）按以下顺序尝试：

```
1. 精确匹配
2. rstrip 后匹配
3. strip 后匹配
4. Unicode 规范化（智能引号、破折号等）+ strip 后匹配
```

这让 patch 对 LLM 输出的空白/引号波动极为鲁棒——实测中是 retry 成功率最高的单点优化之一。

### 7.4 Token 效率

FIX 场景下 LLM 只需输出 patch（通常 10-50 行），而非整个 SKILL.md（常 200-500 行），**节省 4-10 倍 token**。这是 OpenSpace 宣称 "46% 更少 tokens" 的一个重要组成。

### 7.5 Validation

apply 成功后立刻跑 `validate_skill_dir()`（[skill_utils.py:254-301](../tmp/openspace-repo/openspace/skill_engine/skill_utils.py#L254-L301)）：

```
目录存在 → SKILL.md 存在且非空 → frontmatter 语法正确 → name 字段非空
```

任一失败进入 retry 循环。只有通过 validation 的 skill 才会写 store 并替换 Registry 中的前置版本。

---

## 八、检索与注入

### 8.1 Skill 发现与选择的 5 步管线

[registry.py:340-503](../tmp/openspace-repo/openspace/skill_engine/registry.py#L340-L503) 的 `select_skills_with_llm()`：

```
① 质量过滤
  ├ 剔除 selections ≥ 2 且 completions = 0
  └ 剔除 applied ≥ 2 且 fallback_rate > 0.5

② BM25 + 向量预筛（本地 skill > 10 时启用）
  SkillRanker.hybrid_rank() → 候选收到 max(15, max_skills*5)

③ 渐进式暴露
  LLM 只看 skill_id + description + quality stats
  不读 SKILL.md 全文（节省 token）

④ Plan-then-Select
  prompt 让 LLM 先写 1-2 句 brief_plan
  再挑 skill_id 列表
  强调"宁缺毋滥，选错比不选更糟"

⑤ ID 校验 → build_context_injection() 注入全文
  最多 max_skills = 2 个 skill
```

### 8.2 Hybrid Ranking

[skill_ranker.py:99-129](../tmp/openspace-repo/openspace/skill_engine/skill_ranker.py#L99-L129)：

```python
# Stage 1: BM25 粗排 → top_k * 3 候选
bm25_top = self._bm25_rank(query, candidates, top_k * 3)

# Stage 2: Embedding 精排（仅在 BM25 候选上）
emb_results = self._embedding_rank(query, bm25_top, top_k)
```

- BM25：`rank_bm25.BM25Okapi`（回退到 token 交集）
- Embedding：`openai/text-embedding-3-small`（与云平台同型号，保证跨端向量兼容）
- 文本构成：`name + description + SKILL.md body[:2000]`
- Pickle 缓存：`.openspace/skill_embedding_cache/`

### 8.3 Context 注入

[registry.py:554-630](../tmp/openspace-repo/openspace/skill_engine/registry.py#L554-L630) 的 `build_context_injection()` 构造系统消息：

- 以 `### Skill: {skill_id}` 分块，analyzer 可反向解析
- 自动替换 `{baseDir}` 占位符为真实目录
- 根据 `backends`（shell / mcp / gui / web）动态生成 tool hint，避免误导 LLM 使用不存在的后端

### 8.4 中途 Retrieve 工具

[retrieve_tool.py](../tmp/openspace-repo/openspace/skill_engine/retrieve_tool.py) 里的 `RetrieveSkillTool` 作为 LocalTool 注册，让 agent 在执行中途遇到困难时主动调 `retrieve_skill(query="...")`，复用同一 pipeline 拉 1 个最相关 skill 补救。这是"预注入"之外的"按需召回"。

---

## 九、存储（SQLite）

### 9.1 6 张表

[store.py:79-166](../tmp/openspace-repo/openspace/skill_engine/store.py#L79-L166)：

| 表 | 用途 |
|---|------|
| `skill_records` | SkillRecord 主表（含 lineage + quality 计数器） |
| `skill_lineage_parents` | DAG 多对多父子关系（级联删除） |
| `execution_analyses` | 一任务一分析，task_id UNIQUE |
| `skill_judgments` | 每个 analysis 内 per-skill 判断 |
| `skill_tool_deps` | 工具依赖（供 `find_skills_by_tool` 查询） |
| `skill_tags` | LLM 生成的辅助标签 |

关键索引：`idx_sr_active` / `idx_lp_parent` / `idx_ea_task` / `idx_td_tool`。

### 9.2 读写分离

- **写路径**：持久化 write connection + `threading.Lock()`，async 写通过 `asyncio.to_thread()` 转入后台线程，不阻塞 event loop。
- **读路径**：每次开独立短连接，带 `PRAGMA query_only=ON`，WAL 模式支持"多读一写"并发。
- **崩溃恢复**：启动时若检测空 DB 但残留 WAL/SHM，自动删除，防脏数据锁死。
- **关机 checkpoint**：`close()` 时 `PRAGMA wal_checkpoint(TRUNCATE)`，把 WAL 刷回主 DB。

### 9.3 原子演进事务

[store.py:602-637](../tmp/openspace-repo/openspace/skill_engine/store.py#L602-L637) 的 `_evolve_skill_sync()` 在单一事务中：

```
1. FIXED：所有父节点 is_active = 0（实现"仅最新激活"）
2. new_record.is_active = True
3. _upsert 新记录 + 写 skill_lineage_parents 多对多
```

### 9.4 原子计数器更新

四个 quality 计数器通过 SQL 原子递增（[store.py:582-595](../tmp/openspace-repo/openspace/skill_engine/store.py#L582-L595)）：

```sql
UPDATE skill_records SET
    total_selections = total_selections + 1,
    total_applied    = total_applied + ?,
    total_completions = total_completions + ?,
    total_fallbacks  = total_fallbacks + ?
WHERE skill_id = ?
```

避免了 Python 层的 read-modify-write 竞态。

### 9.5 Evolution 候选生命周期（2026-04-16 新增）

新增 `evolution_processed_at` 字段（[store.py:119-131](../tmp/openspace-repo/openspace/skill_engine/store.py#L119-L131)），幂等迁移（`ALTER TABLE ... ADD COLUMN`）。

- `load_evolution_candidates(include_processed=False)` 默认只返回未处理候选
- `process_analysis` 结束时调用 `mark_evolution_processed(task_id)` 打标
- 防止重启后重复演进同一 task

---

## 十、质量监控（两层）

### 10.1 Skill 层：4 比率

原始 4 计数器 + 派生 4 比率（见 §2.3），每次 `ExecutionAnalysis` 持久化时原子更新。

### 10.2 Tool 层：penalty 曲线

除了成功率基本计数，还跟踪：

| 字段 | 说明 |
|------|------|
| `total_calls` / `success_calls` | 累计计数 |
| `recent_window` | 滚动窗口（默认 20 次调用） |
| `recent_success_rate` | 窗口内成功率 |
| `consecutive_failures` | 连续失败计数 |
| `penalty` | 映射到 [0.2, 1.0] 的惩罚系数 |

### 10.3 级联演进链（核心创新）

```
任务执行
    │
    ├─ Rule-based tracking：tool 调用结果 → recent_success_rate
    │
    ├─ LLM analysis：tool_issues 标注（含语义失败）
    │    → _record_tool_quality_feedback 注入 ToolQualityManager
    │
    ▼
ToolQualityRecord 更新
    │
    ▼
get_problematic_tools(threshold=0.5, min_calls=5)
    │
    ▼
SkillEvolver.process_tool_degradation()
    │
    ▼ (per degraded tool)
store.find_skills_by_tool(tool_key) → active skills
    │
    ▼ LLM confirm
    │
    ▼ FIX 演进 → new SkillRecord
```

这是 OpenSpace 相对其他系统（Mem0 / ReMe / AutoSkill / Hermes / XSkill / Skill-Creator / MemOS）**最独特的闭环**——其他系统或者只关注 skill 质量、或者只关注 tool 质量，不做跨层级联。

---

## 十一、云端协作

### 11.1 OpenSpaceClient

[openspace/cloud/client.py:286-378](../tmp/openspace-repo/openspace/cloud/client.py#L286-L378) 的 `upload_skill()`：

```
stage zip → compute diff vs 父节点 → create record
```

Visibility：
- `public`：社区可见，搜索可达
- `private`（2026-04-10 正式支持）：仅 owner 可见
- `group`：云平台管理（组内共享）

对 public + 有父节点的 skill，额外计算 vs 祖先的 diff 作为"改进凭证"，云端展示时可看到完整演化链。

### 11.2 Import 路径穿越防护（2026-03-31）

`import_skill()` 在解压下载的 zip 时硬化防 `../../../etc/passwd` 条目（同样的保护应用于 `patch.py` 的 `_apply_multi_file_patch` / `_apply_multi_file_full`——所有写路径都经 `abs_path.startswith(resolved_dir)` 校验）。

### 11.3 四个对外 MCP 工具

[host_skills/delegate-task/SKILL.md](../tmp/openspace-repo/openspace/host_skills/delegate-task/SKILL.md)：

| 工具 | 作用 |
|------|------|
| `execute_task` | 委派整个任务，含 skill 搜索 + 执行 + 自动演进，返回 `evolved_skills` 列表 |
| `search_skills` | 纯检索，本地+云端合并，默认 `auto_import=true` |
| `fix_skill` | 手动触发对某个本地 skill 的 FIX |
| `upload_skill` | 三模式：演进产物只需 `skill_dir` + `visibility`，全新 skill 需补齐 tags + change_summary |

### 11.4 通讯网关（2026-04-09）

[openspace/communication/](../tmp/openspace-repo/openspace/communication/) 支持：
- **WhatsApp**（Baileys bridge + QR 认证）
- **Feishu**（HTTP webhook）

让 OpenSpace 不只能被 MCP 宿主调用，还能被外部 IM 直接触达——完整的 session 管理、附件缓存、allowlist 访问控制。

---

## 十二、安全防护

### 12.1 反失控演进

三道闸：
1. **LLM 确认门**（`_EVOLUTION_CONFIRM_TEMPLATE`）：4 维决策，歧义默认 skip
2. **状态驱动反 loop**（Trigger 2）：`_addressed_degradations` 记录已处理集合，tool 恢复自动清除
3. **数据驱动反 loop**（Trigger 3）：`min_selections=5` 门槛，无需时间 cooldown

### 12.2 Skill 内容安全扫描

[skill_utils.py:23-50](../tmp/openspace-repo/openspace/skill_engine/skill_utils.py#L23-L50) 的 `check_skill_safety()` 用正则扫描 SKILL.md：

```python
_SAFETY_RULES = [
    ("blocked.malware",          r"(ClawdAuthenticatorTool)"),
    ("suspicious.keyword",       r"(malware|stealer|phish|keylogger)"),
    ("suspicious.secrets",       r"(api[-_ ]?key|token|password|private key)"),
    ("suspicious.crypto",        r"(wallet|seed phrase|mnemonic|crypto)"),
    ("suspicious.webhook",       r"(discord\.gg|webhook|hooks\.slack)"),
    ("suspicious.script",        r"(curl[^\n]+\|\s*(sh|bash))"),
    ("suspicious.url_shortener", r"(bit\.ly|tinyurl\.com|t\.co|goo\.gl)"),
]
_BLOCKING_FLAGS = frozenset({"blocked.malware"})
```

- `blocked.*` → 拒绝注册
- `suspicious.*` → 记录警示，在 UI / 搜索结果中打标

这是对"社区贡献 skill 做 prompt injection / 凭据外泄"的第一道防线。

### 12.3 沙箱与命令确认

[grounding/core/security/](../tmp/openspace-repo/openspace/grounding/core/security/)：

- `sandbox.py` / `e2b_sandbox.py`：E2B 云沙箱 + `BaseSandbox` 抽象
- `policies.py`：`SecurityPolicyManager` 对危险命令弹交互式 y/n 确认，CLI 端 ANSI 高亮 + 行号标注，最多显示 10 行

### 12.4 LLM 评分防误报

Patch 应用失败后的 retry 机制（把磁盘内容回喂 LLM）本身也是一种"防凭空想象"的安全保护——LLM 不靠记忆猜内容，而是基于真实 ground truth 重新生成。

---

## 十三、真实演进产物（GDPVal 165 Skills）

### 13.1 实验数据

据 [README.md](../tmp/openspace-repo/README.md) 与 `gdpval_bench/`：

| 指标 | 数值 |
|------|------|
| 经济收益 | **4.2×** vs ClawWork（同骨干 LLM Qwen 3.5-Plus） |
| Value Capture | $11,484 / $15,764 = **72.8%** |
| 平均质量 | 70.8%（vs 最好 baseline 40.8%，+30pp） |
| Phase 2 Token | Phase 1 的 **45.9%**（-54% 节约） |

### 13.2 7 大族群（从 `gdpval_bench/skills/` 203 个子目录归纳）

| 类别 | 数量 | 代表 skill |
|------|------|-----------|
| **File Format I/O** | 44 | `docx-shell-extract` / `openpyxl-merged-cell-safe-write` / `reliable-pdf-extraction` |
| **Execution Recovery** | 29 | `code-exec-fallback` / `sandbox-fallback-execution` / `python-heredoc-fallback` |
| **Document Generation** | 26 | `document-gen-fallback` 家族（13 代演化链） |
| **Quality Assurance** | 23 | `task-verification-checkpoints` / `spreadsheet-direct-verification` |
| **Task Orchestration** | 17 | `multi-deliverable-tracking` / `zip-deliverable-finalization` |
| **Domain Workflow** | 13 | `create-soap-note` / `audio-track-production-enhanced-enhanced` |
| **Web & Research** | 11 | `regulatory-research-fallback` / `ssl-proxy-debug-workflow` |

**关键观察**：大部分演进产物聚焦于"工具可靠性 + 错误恢复"而非"领域知识"——这是 agent 在真实世界执行时最大的痛点。

### 13.3 DERIVED 链实证

`document-gen-fallback` 家族的多代演化清楚展示了 DERIVED 机制：

```
document-gen-fallback/                        ← IMPORTED（原始）
    name: document-gen-fallback
        │
        ▼
document-gen-fallback-enhanced/              ← DERIVED（v1）
    name: document-gen-unicode-safe          ← frontmatter 已改名
    新增 Unicode 章节
        │
        ▼
document-gen-fallback-enhanced-enhanced/     ← DERIVED（v2）
    拆出 sanitize_for_pdf.sh 辅助脚本
        │
        ▼
document-gen-fallback-enhanced-enhanced-2794b4/  ← DERIVED（v3，uuid 避冲突）
    name: document-gen-resilient-workflow
    多引擎级联 PDF fallback
```

目录名保留了历史后缀，但 frontmatter 的 name 已升级为真正语义化的名字。这正对应 `_EVOLUTION_DERIVED_TEMPLATE` 主动教导 LLM 不要无限追加 `-enhanced`。

---

## 十四、独特优势与劣势

### 优势

| 优势 | 说明 |
|------|------|
| **三触发 × 三操作的 9 元组演进空间** | 唯一提供这种结构化组合能力的系统 |
| **Tool 质量 → Skill 演进级联** | 跨层级联，其他系统都是单层监控 |
| **LLM 确认门 + 双反 loop** | 规避了"误报→反复演进"的陷阱 |
| **Apply-retry 回喂磁盘内容** | 小但关键的工程细节，显著提升 patch 成功率 |
| **PATCH + 4 级降级匹配 + Unicode 规范化** | token 效率最优，对 LLM 拼写波动鲁棒 |
| **skill_id sidecar + 版本 DAG + snapshot** | 可移植 + 可回滚 + 多父合并全支持 |
| **MCP 统一接入多种 host agent** | Claude Code / Codex / OpenClaw / nanobot / Cursor 通吃 |
| **云端协作 + diff 改进凭证** | 社区贡献质量可追溯、可审计 |
| **实测 4.2× 收入 + -46% tokens** | GDPVal 实战验证（165 skills 自主演化） |
| **IM 网关（WhatsApp / Feishu）** | 突破 MCP 边界，面向真实用户触达 |

### 劣势

| 劣势 | 说明 |
|------|------|
| **架构复杂度较高** | 7777 行核心代码（仅 skill_engine 目录），学习曲线陡 |
| **依赖 OpenAI embedding API** | text-embedding-3-small 跨端兼容，但自建部署需兼容此接口 |
| **LLM 确认门增加延迟** | Trigger 2/3 每次演进多一次 LLM 调用 |
| **纯文本 skill** | 不支持多模态（vs XSkill 的视觉锚定） |
| **规则阈值需人工调优** | `_FALLBACK_THRESHOLD=0.4` 等常量在不同场景可能需要调整 |
| **评估依赖任务完成信号** | 当任务本身难以自动判定成败时，analyzer 的 task_completed 可能不准 |

---

## 十五、与现有系统的定位

### 15.1 在 §1.2 光谱中的定位

```
原始保真 ←────────────────────────────────────────────────→ 通用抽象

Mem0   ReMe   MemOS   AutoSkill   Hermes   Skill-Creator   OpenSpace   XSkill
(快照) (经验) (技能)   (方法论)    (进化)    (精品课程)      (自演进)      (双流)
 │      │      │        │          │          │              │           │
 ▼      ▼      ▼        ▼          ▼          ▼              ▼           ▼
verbatim 建议 可执行   可执行+    可执行+   测试验证的    自演进+        Exp≤64词
记录   bullet  SOP   审计闭环   离线进化  可执行SOP     级联闭环        +Skill SOP
```

### 15.2 与 Hermes（最相似的系统）的差异

| 维度 | Hermes | OpenSpace |
|------|--------|-----------|
| **演进由谁决定** | LLM 自主判断（skill_manage 工具） | 三触发器（规则 + 周期 + 事件） |
| **质量监控** | 无自动门控 | 4 比率 + tool penalty |
| **级联机制** | 无 | Tool → Skill 跨层 |
| **多宿主** | Hermes 专属 | MCP 跨 agent |
| **云协作** | 无 | 完整（public/private/group + diff 凭证） |
| **演进模式** | create/patch/edit | FIX/DERIVED/CAPTURED |
| **版本管理** | patch 覆盖 | 版本 DAG（多父 merge） |

可以说 OpenSpace 把 Hermes 的"LLM 自主进化"思路做了工程化：把"LLM 自主"变成"LLM + 规则 + 数据"三方共治，用 confirm gate 降低误报、用 DAG 保留历史、用 MCP 打开生态。

### 15.3 选型建议增补

| 需求 | 推荐 |
|------|------|
| **想让任意 agent（Claude/Codex/Cursor...）都获得自进化** | **OpenSpace** — MCP 统一接入 |
| **需要 tool 质量 → skill 级联修复** | **OpenSpace** — 独有 |
| **需要完整演化链可追溯** | **OpenSpace** — 版本 DAG + diff + snapshot |
| **需要社区贡献 + 私有隔离** | **OpenSpace** — 云端 public/private/group |
| **需要 IM 直接触达** | **OpenSpace** — WhatsApp/Feishu 网关 |

---

## 关键文件速查表

| 主题 | 文件 | 关键行 |
|------|------|-------|
| 三种 Origin / Lineage 定义 | `openspace/skill_engine/types.py` | 26-66 / 69-159 / 197-248 / 333-464 |
| 三种 Evolution Type 实现 | `openspace/skill_engine/evolver.py` | 702-794（FIX）/ 796-926（DERIVED）/ 928-1043（CAPTURED） |
| 三个 Trigger 入口 | `openspace/skill_engine/evolver.py` | 266-307 / 310-438 / 441-517 |
| Evolution agent loop | `openspace/skill_engine/evolver.py` | 1084-1239 |
| Apply-retry（磁盘回喂） | `openspace/skill_engine/evolver.py` | 1271-1424 |
| 健康诊断规则 | `openspace/skill_engine/evolver.py` | 1563-1598 |
| Analyzer 主循环 | `openspace/skill_engine/analyzer.py` | 154-241 |
| Skill ID 纠错 | `openspace/skill_engine/analyzer.py` | 59-106 |
| Tool Quality 反哺 | `openspace/skill_engine/analyzer.py` | 275-327 |
| 5 个 Prompt 模板 | `openspace/prompts/skill_engine_prompts.py` | 157-803 |
| Skill 发现 & 选择 | `openspace/skill_engine/registry.py` | 120-177 / 340-503 |
| Context 注入 | `openspace/skill_engine/registry.py` | 554-630 |
| BM25 + 向量混排 | `openspace/skill_engine/skill_ranker.py` | 99-129 / 191-239 / 260-301 |
| Patch 识别 & 应用 | `openspace/skill_engine/patch.py` | 334-365 / 626-680 / 773-826 |
| 6 张 SQLite 表 | `openspace/skill_engine/store.py` | 79-166 |
| 原子演进 SQL | `openspace/skill_engine/store.py` | 602-637 |
| Evolution 候选生命周期 | `openspace/skill_engine/store.py` | 119-131 / 829-864 |
| Tool quality penalty | `openspace/grounding/core/quality/types.py` | 107-140 |
| Skill 安全扫描 | `openspace/skill_engine/skill_utils.py` | 23-50 / 254-301 |
| 命令确认沙箱 | `openspace/grounding/core/security/policies.py` | 21-136 |
| Cloud client | `openspace/cloud/client.py` | 286-378 / 379-420 |
| 通讯网关 | `openspace/communication/gateway.py` | 63-80 |
| 演化链实例 | `gdpval_bench/skills/document-gen-fallback*/SKILL.md` | — |
| Host-facing MCP | `openspace/host_skills/delegate-task/SKILL.md` | 1-133 |

---

## 参考资料

- [HKUDS/OpenSpace](https://github.com/HKUDS/OpenSpace) — 源码仓库
- [open-space.cloud](https://open-space.cloud) — 云端协作平台
- [GDPVal Benchmark](https://huggingface.co/datasets/openai/gdpval) — 220 个真实专业任务评估集
- [ClawWork](https://github.com/HKUDS/ClawWork) — 基线对比系统（同骨干 LLM）
- [HKUDS/nanobot](https://github.com/HKUDS/nanobot) — 支持 OpenSpace 的宿主 agent 之一
