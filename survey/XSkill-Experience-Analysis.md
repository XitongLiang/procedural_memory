# XSkill Experience 提取与技能演进全流程分析

> 基于 https://github.com/XSkill-Agent/XSkill 源码
> 论文：XSkill: Continual Learning from Experience and Skills in Multimodal Agents (arXiv 2603.12056, 2026-03)

---

## 一、XSkill 是什么

双流持续学习框架，让多模态 Agent 从执行轨迹中自动提取两种互补知识——**Experience**（行动级战术提示）和 **Skill**（任务级结构化 SOP）——不更新任何模型参数。

核心特点：**视觉锚定提取 + 跨 Rollout 对比批判 + 层次化合并**。

跟其他系统的关键区别：

| 维度 | XSkill | AutoSkill | Hermes | Skill-Creator |
|------|--------|-----------|--------|--------------|
| **提取粒度** | 双流：Experience（≤64词）+ Skill（~1000词） | 单一 SKILL.md | 单一 SKILL.md + MEMORY.md | 单一 SKILL.md |
| **视觉锚定** | 图像与轨迹联合分析 | 纯文本 | 纯文本 | 纯文本 |
| **多路对比** | N=4 Rollouts → 跨轨迹对比批判 | 单次对话 | 单次会话 | 子代理 A/B 对比 |
| **知识合并** | 嵌入相似度 + LLM 合并 + 全局精炼 | BM25/向量 + LLM decide | 子字符串匹配 + 容量压缩 | 人工评审迭代 |

### 1.1 两种知识类型

**Experience（经验）** — 行动级战术提示：

```
定义：e = (c, a, v_e)
  c = 触发条件（condition）
  a = 推荐动作（action）
  v_e = 语义嵌入向量（embedding，用于检索）

约束：c + a 合计 ≤ 64 词
格式："When [trigger condition], [recommended action]"

示例：
  [E12] When handling dark or low-contrast images, use Code Interpreter
        to apply histogram equalization before extracting features.
  [E35] For geometric comparison tasks, decompose into individual
        measurements first rather than attempting holistic visual matching.
```

**Skill（技能）** — 任务级结构化 SOP：

```markdown
---
name: VisualLogicArchitect
description: |
  Structured approach for visual reasoning tasks requiring
  multi-step tool orchestration and image analysis.
version: 1.0.0
---

# Visual Logic Architect

## When to Use
- **Task patterns**: [what kinds of questions]
- **Visual signals**: [what the image might contain]

## Strategy Overview
[1-2 sentences on core approach]

## Workflow
1. **[Phase]**: [Action and rationale]
2. **[Phase]**: [Action and rationale]

## Tool Templates
- **Code Interpreter - [Purpose]**:
  ```python
  # [Brief comment]
  [code with placeholders like [TARGET], [QUERY]]
  ```

## Watch Out For
- [Common mistake or trap from trajectory]
```

```
约束：全局 Skill 文档 ≤ 1000 词（超出触发精炼）
目标词数：~600 词（生成时）/ ~1000 词（合并后上限）
```

### 1.2 两种知识的互补关系

论文消融实验清楚展示了互补性：

| 配置 | 效果 | 说明 |
|------|------|------|
| 仅 Experience | 错误率 29.9% | 工具选择更灵活，但结构性错误多 |
| 仅 Skill | 错误率 15.3% | 语法错误从 20.3% 降到 11.4%，但工具选择僵化 |
| Experience + Skill | 最佳 | Skill 防结构性错误，Experience 做情境微调 |

```
Skill 像食谱：告诉你做红烧肉的标准流程
Experience 像厨师笔记：提醒你这口锅火力偏大要提前关火
```

### 1.3 双 LLM 分离架构

```
MLLMexec（执行模型）：执行推理任务，使用工具，产出轨迹
  → 可以是任意骨干模型（GPT-4o / Gemini / Qwen）
  → 不需要很强，跟着 Skill 指令走就行

MLLMkb（知识模型）：提取、合并、适配知识
  → 负责所有知识操作：summarize / critique / consolidate / adapt
  → 需要较强的分析能力
  → 跟 MLLMexec 可以是不同模型 → 支持跨模型知识迁移
```

---

## 二、Phase I：知识积累（Accumulation）

### 2.1 整体流程

```
对于每个训练任务 T_i：
    │
    ├─ Step 1: 多路 Rollout
    │    执行 N=4 次独立推理
    │    每次可能成功也可能失败
    │    产出 N 条轨迹 {τ_1, τ_2, ..., τ_N}
    │
    ├─ Step 2: Rollout Summary（逐条总结）
    │    MLLMkb 对每条轨迹做视觉锚定分析
    │    输入：轨迹文本 + 交错的图像观察
    │    输出：结构化总结 S_i
    │
    ├─ Step 3: Cross-Rollout Critique（跨轨迹批判）
    │    MLLMkb 对比所有 N 条总结
    │    对比成功 vs 失败的轨迹
    │    输出：Experience 更新操作 [{add, e}, {modify, E_id, e'}]
    │
    ├─ Step 4: Experience Consolidation（经验合并）
    │    新经验嵌入 → 检索相似 → 超阈值则 LLM 合并
    │    超容量(120)则强制合并最相似对
    │    定期全局精炼
    │
    ├─ Step 5: Skill Generation（技能生成）
    │    MLLMkb 从轨迹提取可复用的 SOP
    │    每个样本产出一个局部 skill_raw.md
    │
    ├─ Step 6: Skill Consolidation（技能合并）
    │    将新技能片段整合进全局 SKILL.md
    │
    └─ Step 7: Skill Refinement（技能精炼）
         超过 1000 词时触发
         去冗余、泛化、重组

每批处理 batch_size=8 个样本后执行一次合并+精炼
```

### 2.2 Step 1：多路 Rollout

```
配置：
  rollouts_per_sample = 2~4（默认 4，脚本示例 2）
  每次 rollout 独立执行，使用 MLLMexec
  最多 max_turns=20 轮工具调用
  可用工具：web_search, image_search, visit, code_interpreter, zoom

执行时注入已有知识：
  Experience → 通过 INJECTION_HEADER 注入系统提示
  Skill → 通过 SKILL_INJECTION_HEADER 注入系统提示

产出：
  每个 rollout 保存：
    rollout_X/trajectory.jsonl    # 完整轨迹（JSONL 格式）
    rollout_X/tool_image_*.jpg    # 工具生成的中间图像
    original_image*.jpg           # 任务原始图像
```

### 2.3 Step 2：Rollout Summary（视觉锚定总结）

**这是 XSkill 区别于纯文本系统的核心——图像与轨迹联合分析。**

```
输入准备：
  1. 扫描样本目录中所有图像（原始 + 工具生成的中间图像）
  2. MLLMkb 为每张图像生成视觉分析（generate_image_captions）
     → 对工具生成的图像：同时传入原始图像做对比
     → 保留"视觉状态 ↔ 推理决策"的关联
  3. 用视觉分析替换轨迹中的图像引用
     → ![tool_image_1.jpg] → [Image: tool_image_1.jpg]\nCaption: {analysis}
  4. 替换问题中的 <image> 标签为分析文本

最终传给 MLLMkb 的内容：
  question + system_prompt + ground_truth + 处理后的轨迹文本 + 原始图像（PIL）
```

**Prompt（SINGLE_ROLLOUT_SUMMARY）核心指令**：

```
"You are a World-Class reasoning analysis expert."

要求：
1. 逐步描述：哪个工具 + 什么参数 + 推理原因 + 应用了哪个 Experience/Skill
   → 识别元推理类型：问题分解/顺序反思/自我纠正/自我验证
2. 对照正确答案，识别绕路/错误/回退
   → 解释为什么发生 + 影响
3. 保留每步的核心结果（即使过程有缺陷）
4. 图像操作分析：
   → 记录预处理操作（裁剪/滤波/增强/搜索）及其影响
   → 识别提取了哪些视觉特征
   → 建议可改进的视觉操作
```

### 2.4 Step 3：Cross-Rollout Critique（跨轨迹批判）

**这是 Experience 提取的核心环节。**

```
输入：
  question（问题） + ground_truth（正确答案）
  summaries（N 条轨迹总结的合集）
  experiences_used（本次任务检索到的已有经验）
  question_images（原始图像，传给多模态 LLM）
```

**Prompt（INTRA_SAMPLE_CRITIQUE）核心指令**：

```
"Review these problem-solving attempts and extract practical lessons."

分析框架：
1. Trajectory Review：
   - 什么有效？哪些关键决策/洞察导致了正确答案？
   - 什么无效？推理在哪里出错，为什么？
   - 已有 experience 是否被遗漏或不够有用？
   - 不同工具如何组合？哪些序列有效？
   - 视觉操作：什么预处理有帮助？缺了什么？

2. Experience Extraction：
   两个类别：
   - Execution Tips：工具使用实操建议
   - Decision Rules：推理选择简单准则

   两种操作：
   - add：真正新的经验，现有经验未覆盖
   - modify：改进现有经验（引用其 ID）

   最多 max_ops=3 条更新

3. Experience Quality：
   - 每条 ≤ 64 词
   - 以情境/条件开头
   - 足够通用以适用于类似问题
   - 聚焦可操作指导，不要抽象原则
   - 避免具体例子

输出格式：
[
  {"option": "add", "experience": "the new generalizable experience"},
  {"option": "modify", "experience": "the modified experience", "modified_from": "E17"}
]
```

### 2.5 Step 4：Experience Consolidation（经验合并）

这是一个**多层合并管线**，从单条 add/modify 到全局精炼：

```
                    新经验操作 [{add, e1}, {modify, E17, e2}]
                            │
                 ┌──────────▼──────────┐
                 │  Layer 1: 逐条处理    │
                 │                      │
                 │  add 操作：            │
                 │    embed(e1)          │
                 │    → search_similar(  │
                 │        threshold=0.70,│
                 │        top_k=10)      │
                 │    → 找到相似？        │
                 │      ├ 是 → LLM 合并   │
                 │      │   (MERGE_PROMPT)│
                 │      │   删除旧的，     │
                 │      │   加入合并结果   │
                 │      └ 否 → 直接添加   │
                 │                      │
                 │  modify 操作：         │
                 │    → 直接替换原文本     │
                 │    → ID 不存在？当 add  │
                 └──────────┬───────────┘
                            │
                 ┌──────────▼──────────┐
                 │  Layer 2: 容量限制    │
                 │                      │
                 │  if len > N_max(120): │
                 │    → 计算全体嵌入      │
                 │    → 余弦相似度矩阵    │
                 │    → 从最相似对开始     │
                 │      强制 LLM 合并     │
                 │    → 重复直到 ≤ 限制    │
                 │    → 最多 3 轮         │
                 │    → 降低阈值到 0.60   │
                 └──────────┬───────────┘
                            │
                 ┌──────────▼──────────┐
                 │  Layer 3: 全局精炼    │
                 │  (每批 8 个样本后)     │
                 │                      │
                 │  if len >= 10:        │
                 │    EXPERIENCE_REFINE  │
                 │    → merge 真正冗余的  │
                 │    → generalize 过窄的 │
                 │    → delete 低价值的   │
                 │    → 目标：80-100 条   │
                 └──────────┬───────────┘
                            │
                         重编号
                    E0, E1, E2, ...
```

**MERGE_PROMPT**（合并两条相似经验）：

```
"Merge the following experiences into a single, comprehensive experience."

要求：
1. 包含所有重要信息点
2. 清晰、可泛化、不超过 64 词
3. 保留核心经验和决策点
4. 避免冗余但保留独特洞察
```

**EXPERIENCE_REFINE_PROMPT**（全局精炼）：

```
"You are an experience library curator."

当前规模：{exp_count} 条
目标：减少到 80-100 条高质量、多样化经验

精炼目标：
1. Merge Truly Redundant：合并表达相同核心洞察的
2. Generalize Over-Specific：将绑定特定场景的抽象为通用模式
3. Delete Low-Value：删除太显而易见或极少适用的

操作：merge（提供 merged_from 列表）/ delete（提供 deleted_id）

质量标准：
- ≤ 64 词，清晰可操作
- 以触发条件开头（"When...", "For...", "If..."）
- 可泛化到一类问题
- 无具体例子或对象名称（除非必要）
```

### 2.6 Step 5-7：Skill 提取 → 合并 → 精炼

**GENERATE_SKILL_PROMPT**（从轨迹提取技能）：

```
"Analyze the trajectory and extract a reusable Standard Operating Procedure."

原则：
1. 从成功中学习有效工作流，从失败中学教训
2. 用占位符 [TARGET], [QUERY] 代替具体值
3. 有效代码提取为可复用模板
4. 目标 ~600 词

输出结构：
  --- (frontmatter: name, description, version)
  # Title
  ## When to Use
  ## Strategy Overview
  ## Workflow (numbered phases)
  ## Tool Templates (code with placeholders)
  ## Watch Out For
```

**MERGE_SKILL_PROMPT**（合并进全局技能文档）：

```
"Think of the global skill as a living document."

整合策略——对新技能的每个部分问：
- 比现有的更好？→ 重写现有版本
- 冗余或太具体？→ 删除
- 互补？→ 合并为更通用形式
- 真的不同？→ 添加为变体工作流（但尽量合并）

长度预算：~1000 词
最多 4 个变体工作流——更多说明可以合并

"You are a knowledge architect."
```

**SKILL_REFINE_PROMPT**（超 1000 词时精炼）：

```
精炼目标：
1. Remove Redundancy：合并重复内容
2. Avoid Too Specific Cases：替换为泛化模式
3. Logical Consolidation：合并重叠的工作流
4. Format Optimization：确保一致的结构
5. Content Quality：保留可操作和可复用的内容

"Prioritize generalizability over specificity"
```

---

## 三、Phase II：推理时检索与适配

### 3.1 整体流程

```
测试任务 T = (q, I)
    │
    ├─ Step 1: Task Decomposition（任务分解）
    │    q + I → 2-3 个抽象子任务
    │    每个子任务生成检索查询
    │
    ├─ Step 2: Experience Retrieval（经验检索）
    │    每个子任务查询 → embed → top-k=3 余弦相似度检索
    │    合并去重
    │
    ├─ Step 3: Experience Rewrite（经验改写）
    │    MLLMkb 根据当前任务和图像改写检索到的经验
    │    不相关的删掉，相关的适配到当前语境
    │
    ├─ Step 4: Skill Adaptation（技能适配）
    │    MLLMkb 根据任务剪裁全局 Skill
    │    集成改写后的经验到工作流步骤
    │    调整代码模板
    │    ≤ 400 词
    │
    └─ Step 5: Non-prescriptive Injection（非强制注入）
         Experience → INJECTION_HEADER 模板注入系统提示
         Skill → SKILL_INJECTION_HEADER 模板注入系统提示
         "You can use it as reference... You can also have your own ideas"
```

### 3.2 Task Decomposition（任务分解）

**TASK_DECOMPOSITION_PROMPT**：

```
"Deconstruct a complex visual task into 2-3 distinct, actionable subtasks
 to retrieve the most relevant methodological guidance."

每个子任务生成查询，目标三个方面之一：
1. Tool Utilization：工具使用最佳实践
2. Reasoning Strategy：推理框架或解决路径
3. Challenge Mitigation：处理预期挑战的技术

关键要求：
  查询必须抽象掉具体图像细节（不说"红苹果"）
  聚焦底层技术挑战（说"基于颜色的物体过滤"）

输出：
[
  {"type": "visual_extraction", "query": "Techniques for robustly analyzing dark images using Code Interpreter"},
  {"type": "logic_synthesis", "query": "Frameworks for mapping visual features to knowledge using Web Search"}
]
```

### 3.3 Experience Retrieval（经验检索）

```python
# 检索配置
embedding_model = "text-embedding-3-small"
top_k = 3          # 每个子任务取 top-3
min_similarity = 0.0  # 最低相似度阈值（实际上不过滤）

# 流程：
for subtask in decomposed_subtasks:
    query_embedding = embed(subtask.query)
    candidates = cosine_similarity(query_embedding, all_experience_embeddings)
    top_k_experiences = sorted(candidates, reverse=True)[:3]
    retrieved.extend(top_k_experiences)

# 合并去重后传给 Rewrite
```

### 3.4 Experience Rewrite（经验改写）

**EXPERIENCE_REWRITE_PROMPT**：

```
"Transform abstract descriptions into specific, actionable execution tips."

改写准则：
1. Operational Focus：转换为具体可操作的执行提示
2. Pitfalls & Best Practices：明确集成常见陷阱和最佳实践
3. Contextual Adaptation：保持核心方法论洞察，适配当前任务语境
   → 不要太具体（过拟合），但确保实用
4. Tone：建议性语气（"Consider checking...", "It is effective to..."）
   而非命令式

不相关或冗余的经验 → 不输出（等于删掉）

输出：{"E12": "rewritten experience 1", "E35": "rewritten experience 2"}
```

### 3.5 Skill Adaptation（技能适配）

**ADAPT_SKILL_PROMPT**：

```
"Tailor the general skill to this specific task."

目标：
1. Select What's Relevant：看任务和图像，哪些部分适用？删除其余
2. Integrate Experiences：将经验洞察编织进相关工作流步骤
3. Keep Templates Ready：保留可能有用的代码模板
4. Stay Lean：~400 词最大

关键：输出可复用方法指南，不是预填答案
使用占位符（"the observed value"）而非图像中的实际数据
```

### 3.6 Non-prescriptive Injection（非强制注入）

```
Experience 注入头：
"Here are practical tips for tool-based visual reasoning,
 gathered from similar problems:
 {bullets}
 These experiences highlight common patterns and pitfalls.
 When you encounter matching situations, consider applying
 the suggested approaches."

Skill 注入头：
"Here are practical experiences and skills for tool-based
 visual reasoning:
 <skill>{skill_content}</skill>
 You can use it as reference if it is relevant.
 You can also have your own ideas or other approaches."
```

**关键设计**：注入是**建议性的**（"consider applying"、"You can also have your own ideas"），不是强制性的。Agent 保留自由裁量权——当实际情况偏离先验经验时可以即兴发挥。

---

## 四、全部 LLM 提示词清单

### 4.1 Phase I：知识积累

| ID | 文件 | Prompt 名 | 用途 | 输入 | 输出 |
|----|------|----------|------|------|------|
| **P1** | experience_prompts.py | `SINGLE_ROLLOUT_SUMMARY` | 逐条轨迹视觉锚定总结 | 轨迹文本+图像 | 结构化总结 |
| **P2** | experience_prompts.py | `INTRA_SAMPLE_CRITIQUE` | 跨轨迹批判提取经验 | 问题+正确答案+N条总结+已用经验+图像 | [{add,e}/{modify,id,e}] |
| **P3** | experience_prompts.py | `MERGE_PROMPT` | 合并相似经验 | 2+条经验（含相似度分数） | 合并后的单条经验 |
| **P4** | experience_prompts.py | `EXPERIENCE_REFINE_PROMPT` | 全局经验库精炼 | 完整经验库 | [{merge,...}/{delete,...}] |
| **P5** | skill_prompts.py | `GENERATE_SKILL_PROMPT` | 从轨迹提取技能 SOP | 轨迹+正确答案 | SKILL.md 内容 |
| **P6** | skill_prompts.py | `MERGE_SKILL_PROMPT` | 合并新技能到全局文档 | 现有全局 Skill + 新技能列表 | 合并后的 SKILL.md |
| **P7** | skill_prompts.py | `SKILL_REFINE_PROMPT` | 精炼超长技能文档 | 当前 SKILL.md + 词数 | 精炼后的 SKILL.md |

### 4.2 Phase II：推理适配

| ID | 文件 | Prompt 名 | 用途 | 输入 | 输出 |
|----|------|----------|------|------|------|
| **P8** | experience_prompts_test_time.py | `TASK_DECOMPOSITION_PROMPT` | 任务分解为检索子任务 | 任务描述 | [{type, query}] |
| **P9** | experience_prompts_test_time.py | `EXPERIENCE_REWRITE_PROMPT` | 改写检索经验适配当前任务 | 任务描述+检索到的经验 | {id: rewritten_text} |
| **P10** | skill_prompts_test_time.py | `ADAPT_SKILL_PROMPT` | 适配全局技能到当前任务 | 基础 Skill + 经验 + 任务 + 图像 | 适配后的 Skill (~400词) |

### 4.3 注入模板

| ID | 文件 | 模板名 | 用途 |
|----|------|--------|------|
| **I1** | experience_prompts.py | `INJECTION_HEADER` | 训练时注入经验到系统提示 |
| **I2** | experience_prompts_test_time.py | `INJECTION_HEADER` | 推理时注入改写后经验 |
| **I3** | skill_prompts_test_time.py | `SKILL_INJECTION_HEADER` | 推理时注入适配后技能 |

### 4.4 LLM 调用统计

```
Phase I 每个训练样本：
  图像分析：每张图像 1 次 MLLMkb 调用
  Rollout Summary：N 次（每条轨迹 1 次，支持多模态）
  Cross-Rollout Critique：1 次（多模态，含所有总结）
  Experience 合并：0~K 次（取决于相似经验数量）
  Skill 生成：1 次
  ─────────────────
  ≈ N+3+K 次 MLLMkb 调用/样本（N=4 时约 8-10 次）

每批 8 个样本后：
  Skill 合并：1 次
  Skill 精炼：0~1 次（超 1000 词才触发）
  Experience 精炼：0~1 次（≥10 条才触发）

Phase II 每个测试任务：
  Task Decomposition：1 次
  Experience Rewrite：1 次（多模态）
  Skill Adaptation：1 次（多模态）
  Embedding 调用：2-3 次（子任务查询嵌入）
  ─────────────────
  ≈ 3 次 MLLMkb 调用 + 2-3 次 embedding
```

---

## 五、关键超参数

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `rollouts_per_sample` | 4（论文）/ 2（脚本示例） | 每个训练任务的 rollout 数 |
| `experience_max_ops` | 3 | 每次 critique 最多产出的操作数 |
| `experience_max_items` | 120 | 经验库容量上限 |
| `similarity_threshold` | 0.70 | 触发合并的余弦相似度阈值 |
| `similarity_threshold_reduction` | 0.60 | 强制缩减时的降低阈值 |
| `experience_refine_target` | 80-100 | 全局精炼的目标数量 |
| `max_experience_words` | 64 | 单条经验最大词数 |
| `skill_max_length` | 1000 | 技能文档最大词数（触发精炼） |
| `skill_generate_target` | 600 | 技能生成目标词数 |
| `retrieval_top_k` | 3 | 每个子任务检索的 top-k 经验 |
| `embedding_model` | text-embedding-3-small | 经验嵌入模型 |
| `experience_large_batch` | 8 | 批处理大小（多少样本后合并精炼） |

---

## 六、与其他系统的对比

### 6.1 知识提取方式

| 系统 | 提取输入 | 提取方法 | 知识结构 |
|------|---------|---------|---------|
| **XSkill** | N 条多模态轨迹 + 图像 + 正确答案 | 跨轨迹对比批判 | Experience（≤64词三元组）+ Skill（Markdown SOP） |
| **AutoSkill** | 单次对话窗口（6 条消息） | LLM 单次提取 | 单一 SKILL.md |
| **Hermes** | 当前会话上下文 | Agent 自行决定 patch | SKILL.md + MEMORY.md |
| **Skill-Creator** | 人工定义 + 子代理执行结果 | 人机迭代 | 单一 SKILL.md |

### 6.2 知识合并机制

| 系统 | 去重方法 | 合并触发 | 容量管理 |
|------|---------|---------|---------|
| **XSkill** | 嵌入余弦相似度 ≥0.70 → LLM 合并 | 每次 add + 每批精炼 | 硬限制 120 条 + 强制合并 |
| **AutoSkill** | BM25/向量搜索 + LLM 4 维判断 | 每次提取后 decide | 无明确限制 |
| **Hermes** | 子字符串匹配 replace | 容量到 80% 时 | 2200 字符硬限制 |
| **Skill-Creator** | 人工判断 | 人工决定 | 无（靠人控制） |

### 6.3 知识优化/演进

| 系统 | 优化方式 | 评判标准 | 是否真实执行 |
|------|---------|---------|-------------|
| **XSkill** | 跨 Rollout 对比 + 层次化合并精炼 | 对照正确答案 + 成功/失败轨迹对比 | **是**（N 次 rollout 真实执行） |
| **AutoSkill SkillEvo** | LLM 读旧 Skill + 新对话 → 改写 | LLM 自行判断 | 否 |
| **Hermes GEPA** | batch_runner 执行 → LLM-as-judge → 语义变异 | 执行 trace + LLM 评分 | **是** |
| **Skill-Creator** | 子代理 A/B 对比 + 人工评审 → 改进 | 人工 + Grader/Comparator | **是** |

### 6.4 独特设计对比

| 设计点 | XSkill | 最接近的对标 |
|--------|--------|-------------|
| 双流知识（Experience + Skill） | 行动级 + 任务级分离 | Hermes 的 MEMORY.md + SKILL.md（但不同粒度） |
| 视觉锚定 | 图像与轨迹联合分析 | 无（其他系统均为纯文本） |
| 多路 Rollout 对比 | N=4 独立执行 → 成功/失败对比 | Skill-Creator 的 with/without 对比（但只有 2 路） |
| 嵌入驱动合并 | cosine similarity + LLM merge | AutoSkill 的向量搜索 + LLM decide（但 XSkill 更自动化） |
| 非强制注入 | "consider applying" / "your own ideas" | Hermes 的非强制系统提示指导 |

---

## 七、设计洞察

### 7.1 XSkill 的核心创新

**1. Experience ≠ Skill，粒度分离是关键**

```
Skill 防结构性错误：提供工作流骨架，避免 Agent 用错工具组合
Experience 做战术微调：在具体情境下给出"这里要小心"的提示

消融实验证据：
  仅 Skill：错误率 15.3%（结构好但不灵活）
  仅 Experience：错误率 29.9%（灵活但结构乱）
  两者结合：最佳

这是 XSkill 区别于所有其他系统的核心设计。
其他系统只有一种知识载体（SKILL.md），
XSkill 证明了行动级和任务级知识需要不同的表示和检索方式。
```

**2. 多路 Rollout + 对比批判 > 单次观察**

```
AutoSkill 从单次对话提取 → 无法区分"碰巧成功"和"真正有效"
Hermes 从当前会话提取 → 只有当前上下文

XSkill 执行 4 次 → 对比哪些做法在多次中稳定有效
  成功 Rollout 告诉你什么管用
  失败 Rollout 告诉你什么不管用（"these lessons are often more valuable"）
  → 提取的知识天然具有泛化性
```

**3. 嵌入驱动的自动合并替代了手动去重**

```
Hermes：子字符串匹配（"dark mode" 匹配 "User prefers dark mode"）→ 脆弱
AutoSkill：BM25 + LLM 判断 → 每次都要 LLM 调用
XSkill：先用嵌入快速找相似 → 只对高相似度的调 LLM 合并
  → 嵌入做粗筛（快+便宜），LLM 做精合并（慢+贵）
  → 自然实现了成本和质量的平衡
```

**4. 非强制注入 = 信任 Agent 的判断力**

```
Skill-Creator 的 SKILL.md 是强指令："你必须按这个流程执行"
Hermes 的系统提示也是指导性的

XSkill 的注入是建议性的：
  "You can use it as reference... You can also have your own ideas"
  → Agent 可以偏离先验知识
  → 适合开放式任务（视觉推理无法完全预测）
  → 代价：Agent 可能忽略有用的经验
```

**5. 64 词硬约束 = 强制泛化**

```
一般系统没有经验长度限制 → 提取的知识越来越具体和冗长
XSkill 强制 ≤ 64 词 → 迫使 LLM 提炼出核心洞察
  + REFINE_PROMPT 要求 "以触发条件开头"
  + "避免具体例子或对象名称"
  → 系统性地对抗知识膨胀和过拟合
```

---

## 参考资料

**论文**
- [XSkill: Continual Learning from Experience and Skills in Multimodal Agents](https://arxiv.org/abs/2603.12056) (arXiv 2603.12056, 2026-03)

**源码**
- [XSkill-Agent/XSkill](https://github.com/XSkill-Agent/XSkill)

**关键源文件**
- `eval/prompts/experience_prompts.py` — Experience 提取/合并/精炼 Prompt
- `eval/prompts/skill_prompts.py` — Skill 提取/合并/精炼 Prompt
- `eval/prompts/experience_prompts_test_time.py` — 推理时任务分解 + 经验改写 Prompt
- `eval/prompts/skill_prompts_test_time.py` — 推理时技能适配 Prompt
- `eval/exskill/experience_critique.py` — 跨 Rollout 批判实现
- `eval/exskill/experience_manager.py` — 经验合并/缩减/精炼实现
- `eval/exskill/trajectory_summary.py` — 视觉锚定轨迹总结实现
- `eval/exskill/skill_builder.py` — 技能生成/合并/适配实现
- `eval/exskill/experience_retriever.py` — 嵌入检索实现
