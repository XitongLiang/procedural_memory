# ReMe 程序记忆（Task Memory）全流程分析

> 基于 https://github.com/agentscope-ai/ReMe 源码（2026-04 版本）
> 阿里巴巴 AgentScope 团队 | arXiv:2512.10696
> LoCoMo SOTA 86.23 | HaluMem SOTA 88.78

---

## 一、ReMe 是什么

ReMe（Remember Me, Refine Me）是阿里巴巴 AgentScope 团队开发的 **Agent 记忆管理工具包**，解决两个问题：

1. **上下文窗口受限**：长对话中早期信息被截断/丢失
2. **无状态会话**：新会话无法继承历史，每次从零开始

**两套并行实现**：

| 系统 | 存储 | 特点 |
|------|------|------|
| **ReMeLight（文件系统版）** | Markdown 文件 + JSONL | 人类可读可编辑，与 AgentScope ReActAgent 深度集成 |
| **reme_ai（向量版）** | 向量DB（Memory/ES/Chroma） | HTTP/MCP 服务接口，SOTA 基准分数 |

**三种记忆类型**：

```
Agent Memory
├── Personal Memory   — 用户偏好、事实、洞察（user-centric）
├── Task Memory       — 任务执行经验，成功/失败/对比分析（程序记忆）
└── Tool Memory       — 工具调用经验，使用指南（工具偏好）
```

与其他系统的本质区别：ReMe 的程序记忆产物是**自然语言经验建议**（when_to_use + experience），通过 Prompt 注入使用；AutoSkill/MemOS/Hermes 的产物是**可直接执行的 SKILL.md**。

---

## 二、Task Memory 提取流程（程序记忆）

### 2.1 完整走线

```
输入: trajectories（含 messages + score）
    │
    ▼
[TrajectoryPreprocessOp]
  按 score >= success_threshold(默认1.0) 分类：
  success_trajectories / failure_trajectories
    │
    ├─────────────────────┬──────────────────────────┐
    ▼                     ▼                          ▼
[SuccessExtractionOp]  [FailureExtractionOp]  [ComparativeExtractionOp]
  成功路径经验             失败防范建议             成功/失败步骤对比
  when_to_use +           prevention strategy +    差异性洞察
  experience              confidence               soft/hard 对比
    └─────────────────────┴──────────────────────────┘
                    合并为 memory_list
                          │
                          ▼
               [MemoryValidationOp]
               LLM 验证每条记忆的 5 个维度：
               Actionability / Accuracy / Relevance / Clarity / Uniqueness
               score < 0.3 → 剔除
                          │
                          ▼
               [MemoryDeduplicationOp]
               向量余弦相似度去重（阈值默认 0.5）
               ① 与现有库中记忆对比
               ② 当前批次内部互相对比
                          │
                          ▼
               [UpdateVectorStoreOp]
               写入向量DB（content = when_to_use，不是 experience）
```

**关键设计**：向量化的是 `when_to_use`（使用时机描述），而不是经验内容本身。检索时用当前任务去匹配"使用场景"，语义对齐更好。

### 2.2 Tool Memory 提取流程

```
输入: tool_call_results（tool_name/input/output/success/time_cost）
    │
    ▼
[ParseToolCallResultOp]
  每次工具调用存为 ToolCallResult
  MD5(input + output) 精确去重
  积累到 max_history_tool_call_cnt(100) 条后触发汇总
    │
    ▼
[EvaluateToolCallOp] → LLM 评估每次调用质量（score 0.0 或 1.0）
    │
    ▼
[SummarizeToolUsageOp] → LLM 生成工具使用指南（core function/patterns/issues/best practices）
    │
    ▼
[UpdateVectorStoreOp]
  工具名作为 workspace_id key 存储 ToolMemory
```

---

## 三、核心 Prompt 完整原文

### 3.1 成功轨迹提取（`success_extraction_prompt.yaml`）

```
You are an expert AI analyst reviewing successful step sequences from an AI agent execution.

Your task is to extract reusable, actionable step-level task memories that can guide future agent executions.
Focus on identifying specific patterns, techniques, and decision points that contributed to success.

ANALYSIS FRAMEWORK:
● STEP PATTERN ANALYSIS: Identify the specific sequence of actions that led to success
● DECISION POINTS: Highlight critical decisions made during these steps
● TECHNIQUE EFFECTIVENESS: Analyze why specific approaches worked well
● REUSABILITY: Extract patterns that can be applied to similar scenarios

EXTRACTION PRINCIPLES:
● Focus on TRANSFERABLE TECHNIQUES and decision frameworks
● Frame insights as actionable guidelines and best practices

# Original Query
{query}

# Step Sequence Analysis
{step_sequence}

# Context Information
{context}

# Outcome
This step sequence was part of a {outcome} trajectory.

OUTPUT FORMAT:
Generate 1-3 step-level success insights as JSON objects:
[
  {
    "when_to_use": "Specific conditions when this step pattern should be applied",
    "experience": "Detailed description of the successful step pattern and why it works",
    "tags": ["relevant", "keywords", "for", "categorization"],
    "confidence": 0.8,
    "step_type": "reasoning|action|observation|decision",
    "tools_used": ["list", "of", "tools"]
  }
]
```

### 3.2 失败轨迹提取（`failure_extraction_prompt.yaml`）

```
You are an expert AI analyst reviewing failed step sequences from an AI agent execution.

Your task is to extract learning task memories from failures to prevent similar mistakes in future executions.
Focus on identifying error patterns, missed opportunities, and alternative approaches.

ANALYSIS FRAMEWORK:
● FAILURE POINT IDENTIFICATION: Pinpoint where and why the steps went wrong
● ERROR PATTERN ANALYSIS: Identify recurring mistakes or problematic approaches
● ALTERNATIVE APPROACHES: Suggest what could have been done differently
● PREVENTION STRATEGIES: Extract actionable insights to avoid similar failures

EXTRACTION PRINCIPLES:
● Extract GENERAL PRINCIPLES as well as SPECIFIC INSTRUCTIONS
● Focus on PATTERNS and RULES as well as particular instances

# Original Query
{query}

# Step Sequence Analysis
{step_sequence}

# Context Information
{context}

# Outcome
This step sequence was part of a {outcome} trajectory.

OUTPUT FORMAT:
Generate 1-3 step-level failure prevention insights as JSON objects:
[
  {
    "when_to_use": "Specific situations where this lesson should be remembered",
    "experience": "Universal principle or rule extracted from the failure pattern",
    "tags": ["error_prevention", "failure_analysis", "relevant_keywords"],
    "confidence": 0.7,
    "step_type": "reasoning|action|observation|decision",
    "tools_used": ["list", "of", "tools"]
  }
]
```

### 3.3 对比提取——Hard（成功 vs 失败）（`comparative_extraction_prompt.yaml`）

```
You are an expert AI analyst comparing successful and failed step sequences to extract differential insights.

Your task is to identify the key differences between success and failure patterns at the step level.
Focus on critical decision points, technique variations, and approach differences.

COMPARATIVE ANALYSIS FRAMEWORK:
● DECISION CONTRAST: Compare critical decisions made in success vs failure cases
● TECHNIQUE VARIATIONS: Identify different approaches and their outcomes
● TIMING DIFFERENCES: Analyze when certain actions were taken and their impact
● SUCCESS FACTORS: Extract what specifically made the difference

EXTRACTION PRINCIPLES:
● Frame comparisons as PRINCIPLES as well as case-specific SOLUTIONS
● Identify PATTERNS that differentiate effective vs ineffective approaches
● Extract RULES that can guide future similar situations
● Focus on UNDERLYING MECHANISMS rather than surface-level differences

# Successful Step Sequence
{success_steps}

# Failed Step Sequence
{failure_steps}

# Similarity Score: {similarity_score}

OUTPUT FORMAT:
Generate 1-2 comparative insights as JSON objects:
[
  {
    "when_to_use": "Specific scenarios where this comparative insight applies",
    "experience": "Detailed comparison highlighting why success approach works better",
    "tags": ["comparative_analysis", "success_factors", "relevant_keywords"],
    "confidence": 0.8,
    "step_type": "reasoning|action|observation|decision"
  }
]
```

### 3.4 对比提取——Soft（高分 vs 低分）（`comparative_extraction_prompt.yaml`）

```
You are an expert AI analyst comparing higher-scoring and lower-scoring step sequences to extract performance insights.

SOFT COMPARATIVE ANALYSIS FRAMEWORK:
● PERFORMANCE FACTORS: Identify what specifically contributed to the higher score
● APPROACH DIFFERENCES: Compare methodologies and execution strategies
● EFFICIENCY ANALYSIS: Analyze why one approach was more efficient or effective
● OPTIMIZATION INSIGHTS: Extract lessons for improving performance

EXTRACTION PRINCIPLES:
● Focus on INCREMENTAL IMPROVEMENTS and performance optimization
● Extract QUALITY INDICATORS that differentiate better vs good approaches
● Identify REFINEMENT STRATEGIES that lead to higher scores
● Frame insights as PERFORMANCE ENHANCEMENT guidelines

# Higher-Scoring Step Sequence (Score: {higher_score})
{higher_steps}

# Lower-Scoring Step Sequence (Score: {lower_score})
{lower_steps}

OUTPUT FORMAT:
Generate 1-2 performance improvement insights as JSON objects:
[
  {
    "when_to_use": "Specific scenarios where this performance insight applies",
    "experience": "Detailed analysis of what made the higher-scoring approach more effective",
    "tags": ["performance_optimization", "score_improvement", "relevant_keywords"],
    "confidence": 0.7,
    "step_type": "reasoning|action|observation|decision",
    "tools_used": ["list", "of", "tools"]
  }
]
```

### 3.5 记忆验证（`memory_validation_prompt.yaml`）

```
You are an expert AI analyst tasked with validating the quality and usefulness of extracted step-level task memories.

VALIDATION CRITERIA:
● ACTIONABILITY: Is the task memory specific enough to guide future actions?
● ACCURACY: Does the task memory correctly reflect the patterns observed?
● RELEVANCE: Is the task memory applicable to similar future scenarios?
● CLARITY: Is the task memory clearly articulated and understandable?
● UNIQUENESS: Does the task memory provide novel insights or common knowledge?

# Task Memory to Validate
Condition: {condition}
Task Memory Content: {task_memory_content}

OUTPUT FORMAT:
{
  "is_valid": true/false,
  "score": 0.8,
  "feedback": "Detailed explanation of validation decision",
  "recommendations": "Suggestions for improvement if applicable"
}

Score should be between 0.0 (poor quality) and 1.0 (excellent quality).
Mark as invalid if score is below 0.3 or if there are fundamental issues with the task memory.
```

### 3.6 检索重排（`rerank_memory_prompt.yaml`）

```
You are an expert AI analyst tasked with reranking retrieved experiences based on their relevance to a specific query.

Consider:
● DIRECT RELEVANCE: How directly applicable the experience is to the current query
● SITUATION SIMILARITY: How similar the experience context is to the current situation
● ACTIONABILITY: How actionable and specific the experience is
● QUALITY: The overall quality and clarity of the experience

# Current Query
{query}

# Candidate Experiences (Total: {num_candidates})
{candidates}

OUTPUT FORMAT:
{
  "ranked_indices": [2, 0, 4, 1, 3],
  "reasoning": "Brief explanation of ranking rationale"
}

Note: Include ALL candidate indices in the ranking, even if some are less relevant.
```

### 3.7 检索后重写（`rewrite_memory_prompt.yaml`）——关键

```
You are an expert AI assistant tasked with rewriting and reorganizing context content
to make it more relevant and actionable for the current task.

Your task is to take the original context (containing multiple experiences) and rewrite it
as a cohesive, task-specific guidance that directly addresses the current situation.

REWRITING GUIDELINES:
● RELEVANCE FOCUS: Emphasize the most relevant aspects of each experience.
  Prioritize the most relevant experiences. Use clear, direct language.
● ACTIONABLE INSIGHTS: Extract specific, actionable guidance.
  Make the context immediately actionable.
● COHERENT NARRATIVE: Create a flowing narrative rather than disconnected tips.
● SITUATIONAL AWARENESS: Adapt the guidance to the current situation.

# Current Task/Query
{current_query}

# Current Trajectory
{current_context}

# Original Context Content (Multiple Experiences)
{original_context}

OUTPUT FORMAT:
{
  "rewritten_context": "A cohesive, task-specific context message that reorganizes
    and adapts the original experiences for the current task. This should be written
    as a unified guidance rather than separate experience items."
}

Guidelines:
- Rewrite as a unified, flowing guidance
- Adapt terminology and examples to match the current task domain
- Consolidate overlapping insights into coherent recommendations
- Prioritize experiences most relevant to the current situation
- Make the guidance feel custom-written for this specific task
```

### 3.8 工具调用评估（`parse_tool_call_result_prompt.yaml`）

```
You are an expert in evaluating tool invocation results.

## Tool Call Information:
- Tool Name: {tool_name}
- Success Flag: {success_flag}
- Time Cost: {time_cost}s / Token Cost: {token_cost} tokens
- Input Parameters: {input_params}
- Output: {output}

### Important: Score independently from the success flag
The success_flag indicates whether the tool executed without technical errors.
The score should evaluate the QUALITY and RELEVANCE of the result.

A tool can execute successfully (success=True) but still produce low-quality or
irrelevant results (score=0.0).

## Scoring Guidelines (Focus on Result Quality):
- 1.0: High Quality - output is relevant, useful, and appropriate for the input parameters
- 0.0: Low Quality - output is irrelevant, incorrect, unhelpful, or inappropriate

Examples:
- success=True, but output is "No results found" for a reasonable query → score=0.0
- success=True, but output contains generic/irrelevant information → score=0.0
- success=True, and output provides relevant, useful information → score=1.0
- success=False, with error messages → score=0.0

OUTPUT FORMAT:
{
  "summary": "A brief one-sentence summary of the tool call result",
  "evaluation": "A brief evaluation (2-3 sentences) explaining result quality and relevance",
  "score": 1.0
}
```

### 3.9 工具使用指南生成（`summary_tool_memory_prompt.yaml`）

```
You are an expert in analyzing tool usage patterns and generating practical usage guidelines.

## Tool Information:
- Tool Name: {tool_name}

## Recent Tool Call History:
{call_summaries}

## Your Task:
Based on the tool call history, generate a concise and logical tool usage description:

1. **Core Function**: What this tool does and when to use it
2. **Success Patterns**: Parameter patterns and usage scenarios that work well
3. **Common Issues**: Main pitfalls to avoid and why they fail
4. **Best Practices**: 2-3 actionable recommendations

## Response Format:
Provide a structured, concise description (max 200 words).
Focus on actionable insights derived from actual usage data. Avoid generic advice.
```

---

## 四、Personal Memory 核心 Prompt

### 4.1 信息过滤（`info_filter_prompt.yaml`）

**作用**：对每条用户消息打 0/1/2/3 分，过滤后再进入提取步骤。

```
Task: Score the information about {user_name} contained in the given batch of
{batch_size} sentences, with scores of 0, 1, 2, or 3.

0 = no user information included
1 = only hypothetical/fictional content created by the user
2 = general information, timely information, or information requiring inference
3 = clear and important information, or information explicitly requested for recording

Think step-by-step. Output format (wrap final result in <>):
Thought: Basis and reasoning process, within 30 characters.
Result: <Sentence Index> <Score: 0, 1, 2, or 3>
```

### 4.2 信息提取（`get_observation_prompt.yaml`）

**作用**：从得分 ≥ 2 的消息中提取关于用户的重要信息 + 关键词。

```
Task: Sequentially extract important information about {user_name} from the following
{num_obs} sentences along with corresponding keywords. If no important information,
answer "None".

Important information can include: basic info, user profile, interests/preferences,
personality, values, relationships, major life events.

If sentence only contains hypothetical info or fictional content (novels/scripts), respond "None".

Output format (ending with '<>'):
Thought: Basis and process of thinking, within 50 words.
Information: <Sentence number> <> <Clear important information or "None"> <Keywords>
```

### 4.3 矛盾/去重判断（`contra_repeat_prompt.yaml`）

**作用**：顺序扫描新旧记忆，判断矛盾（删旧）或被包含（删新）。

```
Task: For the following {num_obs} sentences, determine whether each sentence
contradicts any of the previous numbered sentences or if the main information
in the sentence is contained within the information from any of the previous
numbered sentences.

Note: Only determine the relationship with the "previous numbered" sentences.
The forms of contradiction can be varied: logical contradiction, attribute changes
(not being in two places at once, not doing two things simultaneously, etc.)

Output format:
Thought: Basis and process of thinking, within 30 words.
Judgment: <Sentence Number> <Contradiction, Contained, None>
```

**Few-shot 示例（节选）**：

```
Example:
1 {user_name} suffers from insomnia and is interested in sleeping pills.
2 {user_name} suffers from insomnia and seeks remedies.
3 Charles is {user_name}'s supervisor.
4 Charles is {user_name}'s supervisor.
5 Charles is {user_name}'s supervisor and the branch manager of a bank.

Judgment: <1> <None>
Judgment: <2> <Contained>   ← 信息被第1句覆盖
Judgment: <3> <None>
Judgment: <4> <Contained>   ← 与第3句完全重复
Judgment: <5> <None>        ← 包含新增信息（分行行长），不算被包含
```

### 4.4 洞察进化（`update_insight_prompt.yaml`）

**作用**：积累 N 条 observations 后，对已有的用户属性 insight 做更新/合并。

```
Extract the given category of {user_name}'s profile information from the following
sentences and determine if it contradicts the existing information.
If there is a contradiction, integrate existing and new information, prioritizing
the new information, and output the result.
If no changes are needed, respond with 'None'.

Output format:
Thoughts: The basis and process of your thinking, within 150 words.
{user_name}'s profile: <Information>
```

**Few-shot 示例（节选）**：

```
句子: {user_name}最近养好了肠胃。{user_name}关注中医养生。
类别: {user_name}健康状况
已有信息: 肠胃不好，高血压

思考: 第一句与已有信息矛盾，以新信息为准。整合后：肠胃健康，高血压。
{user_name}的资料: <肠胃健康，高血压>
```

---

## 五、去重机制

### 5.1 Task Memory — 向量余弦相似度（无 LLM）

`MemoryDeduplicationOp`：
- embedding = `when_to_use + content` 拼接向量化
- 与库中已有记忆余弦相似度 > 0.5 → 跳过
- 同批次内部互相比较，防止批内重复

### 5.2 Personal Memory — LLM 语义判断（矛盾/包含）

`ContraRepeatOp`：顺序扫描，先到先得原则。

| 判断结果 | 处理 |
|---------|------|
| **被包含（Contained）** | 丢弃新条目（旧信息已覆盖） |
| **矛盾（Contradiction）** | 旧条目加入 `deleted_memory_ids`，保留新条目 |
| **无（None）** | 两者都保留 |

### 5.3 Tool Memory — MD5 精确去重

`ToolCallResult.generate_hash()`：对 `input + output` 计算 MD5，防止完全相同的工具调用重复记录。

---

## 六、记忆效用追踪与老化删除

Task Memory 有完整的生命周期管理：

| Op | 操作 | 触发时机 |
|----|------|---------|
| `UpdateMemoryFreqOp` | 每次被检索 → `freq += 1` | 记忆被使用后 |
| `UpdateMemoryUtilityOp` | 每次对任务有帮助 → `utility += 1` | 任务完成后 |
| `DeleteMemoryOp` | `freq >= threshold AND utility/freq < threshold` → 删除 | 定期维护调用 |

**双维度老化**：高频检索但低效用（被检索但没帮上忙）的记忆会被清理，防止记忆库退化。

---

## 七、存储结构

### 7.1 VectorNode 结构（三种记忆类型）

| 记忆类型 | content（向量化字段） | metadata（存储） |
|---------|---------------------|----------------|
| `PersonalMemory` | `when_to_use`（关键词/触发条件） | content/target/time/memory_type |
| `TaskMemory` | `when_to_use`（应用场景描述） | experience/confidence/tags/tools_used |
| `ToolMemory` | `when_to_use`（工具名/使用场景） | tool_call_results 历史记录序列 |

### 7.2 时间信息双轨存储（Personal Memory）

```
msg_year/month/day/week   ← 对话发生时间
event_year/month/day/week ← 事件发生时间（LLM 从"上个月"等相对时间推断）
```

FuseRerank 时分别匹配两类时间，解决相对时间表达（"我上个月做了..."）。

### 7.3 文件系统版目录结构（ReMeLight）

```
working_dir/
├── MEMORY.md                    # 长期记忆（人类可读/编辑）
├── memory/
│   └── YYYY-MM-DD.md            # 每日经验日志
├── dialog/
│   └── YYYY-MM-DD.jsonl         # 原始对话记录
└── tool_result/
    └── <uuid>.txt               # 大型工具输出（N天 TTL 自动清理）
```

---

## 八、检索注入机制

### 8.1 Task Memory 检索流程

```
BuildQueryOp（格式化查询）
    │
    ▼
RecallVectorStoreOp（向量召回 top-k，匹配 when_to_use）
    │
    ▼
RerankMemoryOp（LLM 重排）
  评估三个维度：Direct Relevance / Situation Similarity / Actionability
    │
    ▼
RewriteMemoryOp（关键：LLM 将多条历史经验重写为当前任务专用统一建议）
    │
    ▼
注入 System Prompt
```

### 8.2 Personal Memory 检索流程（FuseRerank）

```
并行：
├── ExtractTimeOp（LLM 解析查询中的时间点 → 绝对日期）
└── RetrieveMemoryOp >> SemanticRankOp（向量检索 + 重排）

FuseRerankOp（纯算法融合，无 LLM）：
  final_score = base_score × type_ratio × time_ratio

  type_ratio（记忆类型权重）：
    insight(2.0) > obs_customized(1.2) > observation(1.0) > conversation(0.5)

  time_ratio（时间匹配奖励）：
    时间匹配（对话时间或事件时间命中）→ × 2.0
    不匹配 → × 1.0
```

---

## 九、与其他系统的核心差异

| 维度 | ReMe Task Memory | AutoSkill/MemOS Skill | Mem0 程序记忆 |
|------|-----------------|----------------------|--------------|
| **产物形式** | 自然语言经验（when_to_use + experience） | SKILL.md（结构化步骤 + 代码） | 执行历史逐步骤快照 |
| **使用方式** | Prompt 注入（RewriteMemory 定制建议） | 直接执行（工具调用） | Prompt 注入（原始记录） |
| **通用化** | 有（去掉具体参数，提取模式） | 有（去标识化） | 无（保留所有具体细节） |
| **失败轨迹** | 有（专用提取 + 对比分析） | 无 | 无 |
| **质量门控** | 5 维验证（score < 0.3 淘汰） | LLM 打分 0-10 | 无 |
| **离线演进** | 仅效用老化删除（无主动合并） | SkillEvo 遗传算法 | 无 |
| **LoCoMo 分数** | 86.23（SOTA） | — | 61.00 |

### 设计哲学

```
ReMe：      Memory as Experience（记忆即经验建议）
            → 自然语言，通过 RewriteMemory 为每次任务定制注入
            → 三类记忆（Personal/Task/Tool）全覆盖

AutoSkill：  Memory as Skills（记忆即技能）
            → 结构化 SKILL.md，直接执行，严格质量门控

Mem0：      Memory as Facts（记忆即事实）
            → 最小颗粒，通用存储基础设施
```

---

## 十、关键设计洞察

### 10.1 向量化 when_to_use 而非 content

Task Memory 存储时 VectorNode.content 是 `when_to_use`，不是经验正文。检索时用当前任务描述去匹配"使用时机"，比匹配经验内容本身准确得多——任务描述和使用时机在语义空间上更对齐。

### 10.2 RewriteMemory 比直接拼接效果好

多条历史经验检索后不直接拼接进 Prompt，而是让 LLM 重写为针对当前任务的统一建议。减少 Prompt 冗余，提高经验的任务适配度。

### 10.3 失败轨迹是独特优势

同时提取成功/失败/对比三类经验，特别是 `ComparativeExtractionOp` 的对比分析（hard: 成功 vs 失败；soft: 高分 vs 低分），是 AutoSkill/MemOS 没有的能力。

### 10.4 效用追踪防止记忆退化

`utility/freq` 双维度追踪：一条被频繁检索但从不带来实际帮助的记忆，是噪声而非资产，定期清理。

### 10.5 信息过滤是提取质量的关键

Personal Memory 提取前先用 0/1/2/3 分级过滤，只处理 score ≥ 2 的消息，避免将虚构内容（用户创作的小说/剧本）或无关内容（通用知识查询）误存为个人记忆。

---

## 参考资料

- [agentscope-ai/ReMe GitHub](https://github.com/agentscope-ai/ReMe)
- [ReMe arXiv 论文 2512.10696](https://arxiv.org/abs/2512.10696)
