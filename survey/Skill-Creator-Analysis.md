# Claude Skill-Creator 技能创建与优化全流程分析

> 基于 https://github.com/anthropics/skills/tree/main/skills/skill-creator 源码（2026-04 版本）
> 本地路径：`~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/`

---

## 一、Skill-Creator 是什么

Anthropic 官方为 Claude Code 开发的**元技能**（meta-skill）——一个专门用来创建、测试和优化其他技能的技能。

核心特点：**人机协作迭代 + 4 子代理并行测试 + 描述自动优化**。

跟 AutoSkill / Hermes 最大的区别：**Skill-Creator 不做自动提取**。它不监听对话、不后台抽取——而是用户主动触发，通过严格的「创建 → 测试 → 评审 → 改进 → 再测试」循环来打磨技能质量。优化不是附加功能，**优化就是核心流程**。

### 1.1 文件结构

```
skill-creator/
├── SKILL.md                    # 主指令文件（~900 行）
├── agents/                     # 3 个子代理定义
│   ├── grader.md               # 评分代理
│   ├── comparator.md           # 盲测对比代理
│   └── analyzer.md             # 模式分析代理
├── scripts/                    # 7 个 Python 脚本
│   ├── run_eval.py             # 触发率测试
│   ├── improve_description.py  # 描述改进（LLM 调用）
│   ├── run_loop.py             # 评估+改进循环
│   ├── aggregate_benchmark.py  # 聚合基准统计
│   ├── generate_report.py      # HTML 报告生成
│   ├── quick_validate.py       # 基础校验
│   ├── package_skill.py        # 打包 .skill 文件
│   └── utils.py                # 解析 SKILL.md frontmatter
├── eval-viewer/                # 人工评审界面
│   ├── generate_review.py      # 生成评审页面
│   └── viewer.html             # 评审 SPA（含 Outputs + Benchmark 两个标签）
├── assets/
│   └── eval_review.html        # 触发评估集编辑模板
└── references/
    └── schemas.md              # 所有 JSON 结构定义
```

---

## 二、5 阶段管线总览

```
[1] 意图捕获        用户说 "做个 X 技能" → 四问确认意图
        │
        ▼
[2] 访谈研究        追问边缘情况、依赖项、成功标准
        │
        ▼
[3] 编写 SKILL.md   渐进式披露三级加载 + CSO 优化
        │
        ▼
[4] 测试迭代        ← 核心循环：4 子代理并行测试 → eval-viewer 人工评审 → 改进 → 重测
        │                                    ↑_______________|
        ▼
[5] 描述优化        自动化触发率优化（独立子系统）
        │
        ▼
    打包交付         package_skill.py → .skill 文件
```

**关键设计理念**：Skill-Creator 把"优化"拆成两个独立维度——

| 维度 | 优化什么 | 方法 | 谁评判 |
|------|---------|------|--------|
| **Skill 内容优化**（阶段 4） | SKILL.md 的指令质量 | 子代理执行 + 人工评审循环 | 人类 + Grader |
| **Description 触发优化**（阶段 5） | frontmatter 的 description 字段 | 自动化 eval + LLM 改写循环 | 脚本自动判定 |

---

## 三、阶段 1-3：创建技能

### 3.1 意图捕获（4 个核心问题）

```
Q1: 这个技能应该让 Claude 做什么？
Q2: 什么时候应该触发？（用户的哪些话/场景）
Q3: 期望的输出格式是什么？
Q4: 是否需要测试用例？
    → 可客观验证的输出（文件转换、数据提取、代码生成）→ 建议测试
    → 主观输出（写作风格、艺术）→ 通常不需要
```

如果当前对话已经包含了一个工作流（用户说 "把这个变成技能"），则从对话历史中提取：用过的工具、步骤序列、用户纠正、输入输出格式。

### 3.2 访谈研究

主动追问：
- 边缘情况、输入输出格式、示例文件
- 成功标准、依赖项
- 如果有可用的 MCP，并行研究相关文档

**关键规则**：在访谈完成之前不写任何技能内容。

### 3.3 编写 SKILL.md

**渐进式披露三级加载**：

| 层级 | 加载时机 | 大小目标 | 内容 |
|------|---------|---------|------|
| L0 元数据 | 始终在上下文中 | ~100 词 | name + description |
| L1 指令 | 技能被触发时 | <500 行 | SKILL.md 完整内容 |
| L2 资源 | 按需加载 | 无限制 | scripts/ references/ assets/ |

**Description 字段写法要求**（直接影响后续触发优化）：

```yaml
# 要求：
# - 祈使句开头："Use this skill for..."
# - 聚焦用户意图，不描述实现细节
# - 稍微 "pushy" 一点（Claude 倾向于 undertrigger）
# - 100-200 词，硬限制 1024 字符

# 示例：
description: >
  Use when building dashboards or data visualizations for internal metrics.
  Make sure to use this skill whenever the user mentions dashboards, data
  visualization, internal metrics, or wants to display any kind of company
  data, even if they don't explicitly ask for a 'dashboard.'
```

**写作风格要求**：

```
- 解释 WHY 而不是堆 MUST（"今天的 LLM 足够聪明"）
- 用 theory of mind 让技能通用化，不绑定具体示例
- 写完初稿后 "换双眼睛" 重审改进
- 如果发现子代理重复写相同的辅助脚本 → 提取为 scripts/ 中的共享工具
```

---

## 四、阶段 4：测试迭代（Skill 内容优化）

这是 Skill-Creator 的**核心循环**，也是回答"技能生成后如何优化"的主要答案。

### 4.1 整体流程

```
                    ┌──────────────────────────────────────────┐
                    │           一次迭代（iteration-N）          │
                    │                                          │
用户确认测试用例 ──→ │  Step 1: 并行 spawn 所有 runs             │
                    │    每个 eval 同时 spawn 2 个子代理：       │
                    │    ├─ with_skill（带技能执行任务）          │
                    │    └─ without_skill（无技能基线）           │
                    │                                          │
                    │  Step 2: 等待期间草拟 assertions           │
                    │    可客观验证的断言 + 描述性命名             │
                    │                                          │
                    │  Step 3: runs 完成 → 抓取 timing 数据      │
                    │    从子代理通知中提取 total_tokens + duration│
                    │                                          │
                    │  Step 4: 评分 + 聚合 + 启动 viewer         │
                    │    ├─ Grader 评估每个 assertion             │
                    │    ├─ aggregate_benchmark.py 生成统计       │
                    │    ├─ Analyzer 发现隐藏模式                 │
                    │    └─ generate_review.py 启动评审界面       │
                    │                                          │
                    │  Step 5: 人工评审 → 读取 feedback.json      │
                    └──────────┬───────────────────────────────┘
                               │
                    ┌──────────▼───────────────────────────────┐
                    │           改进技能                         │
                    │  1. 从 feedback 泛化（不过拟合到特定用例）   │
                    │  2. 保持 prompt 精简（删除不起作用的部分）   │
                    │  3. 解释 WHY（不堆 ALWAYS/NEVER）           │
                    │  4. 提取重复的辅助脚本到 scripts/           │
                    └──────────┬───────────────────────────────┘
                               │
                               ▼
                    下一次迭代（iteration-N+1）
                    直到：用户满意 / feedback 全空 / 不再有意义进步
```

### 4.2 工作区目录结构

```
skill-name-workspace/
├── iteration-1/
│   ├── eval-descriptive-name/
│   │   ├── eval_metadata.json      # 测试用例元数据 + assertions
│   │   ├── with_skill/
│   │   │   ├── outputs/            # 技能执行产物
│   │   │   ├── timing.json         # {total_tokens, duration_ms}
│   │   │   └── grading.json        # Grader 评分结果
│   │   └── without_skill/
│   │       ├── outputs/            # 基线执行产物
│   │       ├── timing.json
│   │       └── grading.json
│   ├── benchmark.json              # 聚合统计（pass_rate, time, tokens）
│   ├── benchmark.md                # 可读版统计
│   └── feedback.json               # 人工评审反馈
├── iteration-2/
│   └── ...（同上结构）
└── skill-snapshot/                  # 改进已有技能时保存原版快照
```

### 4.3 三个子代理

#### Grader（评分代理）

**角色**：评估 assertions 是否通过，**同时评判 assertions 本身的质量**。

```
输入：expectations 列表 + transcript 路径 + outputs 目录
       │
  Step 1: 完整阅读执行转录
  Step 2: 检查 output 文件（不只看转录声称产出了什么）
  Step 3: 逐条评估 assertion
           PASS = 有明确证据 + 不是表面合规
           FAIL = 无证据 / 证据矛盾 / 表面合规（文件名对但内容空）
  Step 4: 提取隐式声明并验证
           事实声明："表单有 12 个字段" → 对照输出验证
           过程声明："用了 pypdf 填表" → 对照转录验证
           质量声明："所有字段都正确填写" → 评估是否成立
  Step 5: 读取 user_notes.md（executor 标记的不确定项）
  Step 6: 评判 eval 本身
           "这个 assertion 太弱——即使输出完全错误也会通过"
           "这个重要结果没有 assertion 覆盖"
  Step 7: 写出 grading.json
  Step 8: 读取执行指标和计时数据
```

**输出结构**：

```json
{
  "expectations": [
    {"text": "...", "passed": true, "evidence": "转录 Step 3 中发现..."}
  ],
  "summary": {"passed": 2, "failed": 1, "total": 3, "pass_rate": 0.67},
  "execution_metrics": {"tool_calls": {"Read": 5, "Bash": 8}, "total_tool_calls": 15},
  "timing": {"executor_duration_seconds": 165.0, "total_duration_seconds": 191.0},
  "claims": [
    {"claim": "表单有 12 个可填字段", "type": "factual", "verified": true, "evidence": "..."}
  ],
  "eval_feedback": {
    "suggestions": [
      {"assertion": "输出包含名字", "reason": "幻觉文档也能通过——应检查是否为主联系人+匹配电话邮箱"}
    ],
    "overall": "断言只检查存在不检查正确性。建议增加内容验证。"
  }
}
```

#### Comparator（盲测对比代理）

**角色**：不知道哪个输出来自哪个技能版本，纯粹基于质量判定胜负。

```
输入：output_a 路径 + output_b 路径 + eval 提示 + expectations（可选）
       │
  Step 1: 读两份输出
  Step 2: 理解任务要求
  Step 3: 生成评估 rubric（内容 + 结构两个维度，各 1-5 分）
           内容：correctness / completeness / accuracy
           结构：organization / formatting / usability
           → 根据具体任务调整（PDF 表单 → 字段对齐/文字可读性/数据放置）
  Step 4: 双维度打分，汇总为 1-10 总分
  Step 5: 检查 assertions（作为次要证据）
  Step 6: 判定胜者
           主要：rubric 总分
           次要：assertion 通过率
           平局极少，通常能分出胜负
  Step 7: 写出 comparison.json
```

#### Analyzer（模式分析代理）

**两种模式**：

**模式 A — Post-hoc 分析**（盲测后）：

```
输入：winner/loser 的技能路径 + 转录路径 + comparison 结果
       │
  → 读两个技能的 SKILL.md，识别结构差异
  → 读两份转录，对比执行模式
  → 评估指令遵循度（1-10 分 + 具体问题列表）
  → 识别 winner 优势和 loser 弱点
  → 生成按优先级排序的改进建议
       high：可能改变对比结果的
       medium：提升质量但不改变胜负
       low：锦上添花
```

**模式 B — Benchmark 分析**（多轮统计后）：

```
输入：benchmark.json + 技能路径
       │
  → 逐 assertion 跨所有 run 分析模式：
     始终通过（两种配置都通过）→ 不能区分技能价值
     始终失败（两种配置都失败）→ 可能 eval 有问题
     有技能通过 / 无技能失败 → 技能明确有价值
     有技能失败 / 无技能通过 → 技能可能有害
     高方差 → flaky assertion
  → 跨 eval 分析模式
  → 分析资源消耗模式（时间、tokens、tool calls）
  → 输出：JSON 数组的观察笔记
```

### 4.4 Eval-Viewer 人工评审界面

`generate_review.py` 生成一个 SPA，包含两个标签：

**Outputs 标签**（定性评审）：
- 逐个展示测试用例
- 显示：Prompt → Output（内联渲染文件）→ 上一轮输出（折叠）→ 评分结果（折叠）
- 每个用例有文本框留反馈，自动保存
- 支持键盘导航（←→ 箭头）
- "Submit All Reviews" 保存 `feedback.json`

**Benchmark 标签**（定量统计）：
- 各配置的 pass_rate / timing / token 消耗
- 逐 eval 细分 + Analyzer 观察笔记
- mean ± stddev + delta 对比

**iteration 2+ 特性**：`--previous-workspace` 参数可加载上一轮结果做对比。

### 4.5 改进策略（Skill-Creator 的 4 条改进原则）

这是 SKILL.md 中最核心的指导思想，直接影响技能优化质量：

```
原则 1：从 feedback 泛化，不过拟合
  技能要用一百万次，而你只在几个例子上迭代。
  不要做 "fiddly overfitty changes" 或 "oppressively constrictive MUSTs"。
  如果某个问题很顽固，试试换隐喻、换模式。

原则 2：保持 prompt 精简
  删除不起作用的部分。
  读转录，不只读最终输出——如果技能让模型浪费时间做无用功，删掉那些指令。

原则 3：解释 WHY
  "今天的 LLM 足够聪明。它们有良好的 theory of mind。"
  如果你发现自己在写全大写的 ALWAYS 或 NEVER → 黄旗。
  重构为解释原因，让模型理解重要性。

原则 4：提取重复工作
  如果 3 个测试用例的子代理都独立写了类似的 helper 脚本
  → 提取为 scripts/ 中的共享工具，技能里指向它。
```

---

## 五、阶段 5：描述触发优化（Description Optimization）

这是一个**独立于技能内容优化的自动化子系统**，专门优化 SKILL.md frontmatter 中的 `description` 字段——决定 Claude 何时触发技能的关键。

### 5.1 为什么需要单独优化 Description

```
用户发送消息
    │
    ▼
Claude 看到 available_skills 列表
（每个技能只有 name + description）
    │
    ▼
Claude 决定：是否需要调用某个技能？
    ├─ 是 → 读取完整 SKILL.md → 按指令执行
    └─ 否 → 用自己的能力直接回答
```

**关键洞察**：
- 描述太窄 → 该触发时不触发（undertrigger，更常见）
- 描述太宽 → 不该触发时触发（false trigger）
- 描述包含工作流摘要 → Claude 按描述走捷径，跳过完整技能内容
- 简单查询（如 "读这个 PDF"）→ Claude 认为自己能搞定，不触发任何技能

### 5.2 四步流程

```
Step 1: 生成 20 个触发评估查询
          ├─ 8-10 个 should_trigger（应触发）
          │    多种表述、不同正式度、包含不明确提及技能名的情况
          └─ 8-10 个 should_not_trigger（不应触发）
               关键：近似但不相关的查询，不是明显无关的
                                │
                                ▼
Step 2: 人工审核评估集
          → eval_review.html 模板生成可交互页面
          → 用户编辑查询、切换 should_trigger、增删条目
          → 导出 eval_set.json
                                │
                                ▼
Step 3: 运行优化循环 (run_loop.py)
          → 自动化：评估 → 改进 → 再评估 → 再改进...
          → 最多 5 轮
          → 实时 HTML 报告（浏览器自动刷新）
                                │
                                ▼
Step 4: 应用最佳描述
          → 取 test 分数最高的（非 train 分数）
          → 更新 SKILL.md frontmatter
          → 向用户展示 before/after + 分数
```

### 5.3 评估查询质量要求

SKILL.md 中对评估查询有**严格的质量要求**，这是整个优化系统的数据质量保障：

```
❌ 坏查询（太抽象）：
   "Format this data"
   "Extract text from PDF"
   "Create a chart"

✅ 好查询（具体、有细节、像真人说话）：
   "ok so my boss just sent me this xlsx file (its in my downloads,
    called something like 'Q4 sales final FINAL v2.xlsx') and she
    wants me to add a column that shows the profit margin as a
    percentage. The revenue is in column C and costs are in column D
    i think"

要求：
  - 文件路径、个人上下文、列名、公司名、URL
  - 混合长短、混合正式/口语
  - 可能有缩写、拼写错误、随意语气
  - should_not_trigger 的查询必须是"近似但不相关"，
    不能是明显无关的（"写斐波那契函数"对 PDF 技能毫无测试价值）
```

### 5.4 run_loop.py：评估+改进循环（核心实现）

```python
# 源码：scripts/run_loop.py

┌─────────────────────────────────────────┐
│  初始化                                  │
│  eval_set (20 queries)                   │
│  ├─ 分层抽样拆分                          │
│  │   trigger 组随机打散 → 60% train / 40% test │
│  │   no_trigger 组同样拆分                │
│  │   每组至少保留 1 个到 test 集           │
│  └─ current_description = 原始描述        │
└─────────────┬───────────────────────────┘
              │
    ┌─────────▼──────────────────────────┐
    │  for iteration in 1..max_iterations │◄─────────────┐
    │                                     │              │
    │  run_eval(train + test, 一批并行)    │              │
    │  每个 query 跑 runs_per_query=3 次   │              │
    │  trigger_rate = 触发次数 / 总次数    │              │
    │  should_trigger: pass = rate >= 0.5  │              │
    │  should_not:     pass = rate < 0.5   │              │
    │                                     │              │
    │  拆回 train_results / test_results   │              │
    │                                     │              │
    │  if train 全部通过 → exit            │              │
    │  if 最后一轮 → exit                  │              │
    │                                     │              │
    │  improve_description(                │              │
    │    train_results,                    │              │
    │    blinded_history  ← 不含 test 分数  │              │
    │  )                                  │              │
    │  → new_description                   │──────────────┘
    │                                     │
    └─────────────────────────────────────┘
              │
    ┌─────────▼──────────────────────────┐
    │  选择最佳描述                        │
    │  best = max(history, key=test_score) │
    │  ← 按 TEST 分数选，不按 TRAIN 分数   │
    │  ← 防止过拟合                        │
    └─────────────────────────────────────┘
```

**关键设计细节**：

| 参数 | 默认值 | 说明 |
|------|-------|------|
| `max_iterations` | 5 | 最多改进轮次 |
| `runs_per_query` | 3 | 每个查询测试 3 次取可靠触发率 |
| `trigger_threshold` | 0.5 | 触发率 ≥50% 视为触发 |
| `holdout` | 0.4 | 40% 数据留做测试集 |
| `num_workers` | 10 | 并行 worker 数 |
| `timeout` | 30 | 每次查询超时（秒） |
| 描述上限 | 1024 字符 | 超出则要求 LLM 缩短 |

### 5.5 run_eval.py：单次触发测试（底层实现）

```
run_single_query(query, skill_name, skill_description):
  │
  │  1. 创建临时 command 文件
  │     路径：.claude/commands/{skill_name}-skill-{uuid8}.md
  │     内容：YAML frontmatter (description) + "This skill handles: {description}"
  │
  │  2. 执行 claude -p "{query}" --output-format stream-json --verbose --include-partial-messages
  │     移除 CLAUDECODE 环境变量（允许在 Claude Code 会话中嵌套调用）
  │
  │  3. 解析 stream events 判断是否触发
  │     ├─ content_block_start + tool_use:
  │     │   ├─ tool_name == "Skill" 或 "Read" → pending
  │     │   └─ 其他工具 → return False（Claude 选择了别的工具）
  │     ├─ content_block_delta + input_json_delta:
  │     │   └─ 累积 JSON 中包含 clean_name → return True（触发了）
  │     ├─ content_block_stop / message_stop:
  │     │   └─ 检查累积 JSON 是否包含 clean_name
  │     └─ 最终 fallback: 完整 assistant message 中检查 tool_use
  │
  │  4. 清理临时 command 文件
  │
  └→ return bool（是否触发）
```

**核心思路**：不是"让 LLM 评判描述好不好"，而是**真实执行 `claude -p`**，看 Claude 在面对这个查询时是否真的去读了这个技能。这是一种 **end-to-end 黑盒测试**。

### 5.6 improve_description.py：描述改进（LLM 调用）

这是描述优化中唯一的 LLM 调用点。使用 Anthropic API 直接调用，启用 extended thinking。

**输入给 LLM 的信息**：

```
1. 当前描述
2. 当前分数（Train: X/Y, Test: X/Y）
3. 失败案例：
   FAILED TO TRIGGER: "查询文本" (triggered 0/3 times)
   FALSE TRIGGERS:    "查询文本" (triggered 3/3 times)
4. 历史尝试（标注 "do NOT repeat these — try something structurally different"）：
   每次尝试的描述 + 分数 + 逐条 train 结果
5. 技能完整内容（作为上下文）
```

**给 LLM 的改进指导**：

```
- 不要过拟合：不要列出越来越长的具体查询清单
- 从失败中泛化：提炼更广泛的用户意图和使用场景
- 100-200 词上限，即使牺牲准确度
- 祈使句："Use this skill for..." 而非 "this skill does..."
- 聚焦用户意图，不描述实现细节
- 如果多轮失败后仍卡住，换个风格/句式
- 鼓励创造性：多轮迭代中尝试不同风格，最后取最高分的

API 参数：
  model = 当前会话的模型
  max_tokens = 16000
  thinking.budget_tokens = 10000
  → 如果描述 > 1024 字符，追加一轮要求缩短
```

---

## 六、全部 LLM 提示词清单

### 6.1 系统指令类（SKILL.md 中对使用者的指导）

| ID | 位置 | 内容摘要 |
|----|------|---------|
| **S1** | SKILL.md "Capture Intent" | 4 个核心问题：做什么、何时触发、输出格式、是否测试 |
| **S2** | SKILL.md "Interview" | "追问边缘情况、输入输出格式、示例文件、成功标准、依赖项" |
| **S3** | SKILL.md "Writing Style" | "解释 WHY 而非堆 MUST...用 theory of mind...写初稿后换双眼睛重审" |
| **S4** | SKILL.md "Improving the skill" | 4 条改进原则：泛化/精简/解释WHY/提取重复 |
| **S5** | SKILL.md "Description" | "稍微 pushy 一点...即使用户没明确提到 dashboard 也要触发" |
| **S6** | SKILL.md "Communicating" | "plumbers are opening terminals, parents googling npm install...注意用户技术水平" |

### 6.2 子代理 Prompt 类

| ID | 文件 | 角色 | 核心指令 |
|----|------|------|---------|
| **A1** | agents/grader.md | 评分代理 | "两个工作：评分输出 + 评判 eval 本身。弱 assertion 上的 PASS 比 useless 更糟——它制造假信心" |
| **A2** | agents/comparator.md | 盲测代理 | "你不知道哪个 skill 产出了哪个输出。纯粹基于输出质量判断。Stay blind." |
| **A3** | agents/analyzer.md (Post-hoc) | 事后分析 | "unblind 结果，分析 WHY winner won。聚焦可操作的改进建议" |
| **A4** | agents/analyzer.md (Benchmark) | 统计分析 | "surface patterns aggregate stats 会隐藏的模式。不建议改进——只报告观察" |

### 6.3 子代理 Spawn 模板

| ID | 位置 | 模板 |
|----|------|------|
| **T1** | SKILL.md Step 1 (with-skill) | `"Execute this task: Skill path: <path>, Task: <prompt>, Input files: <files>, Save outputs to: iteration-N/eval-ID/with_skill/outputs/"` |
| **T2** | SKILL.md Step 1 (baseline) | 同上但无 skill path，保存到 `without_skill/outputs/` |

### 6.4 描述优化 Prompt（improve_description.py）

| ID | 触发条件 | 完整 Prompt 结构 |
|----|---------|-----------------|
| **D1** | 每轮改进时 | 见下文 |
| **D2** | 描述 > 1024 字符时 | "Your description is {N} characters, which exceeds the hard 1024 character limit. Please rewrite it to be under 1024 characters while preserving the most important trigger words and intent coverage." |

**D1 完整结构**（improve_description.py 第 49-112 行）：

```
You are optimizing a skill description for a Claude Code skill called "{skill_name}".

[解释技能/描述/触发机制的上下文]

<current_description>"{current_description}"</current_description>

Current scores ({train_score}, {test_score}):
<scores_summary>
  FAILED TO TRIGGER:
    - "{query}" (triggered X/Y times)
  FALSE TRIGGERS:
    - "{query}" (triggered X/Y times)
  PREVIOUS ATTEMPTS (do NOT repeat these — try something structurally different):
    <attempt train=X/Y, test=X/Y>
      Description: "..."
      Train results:
        [PASS/FAIL] "query" (triggered X/Y)
    </attempt>
</scores_summary>

<skill_content>{full_skill_content}</skill_content>

[改进指导：不过拟合、不列长清单、100-200词、祈使句、聚焦意图、鼓励创造性]

Please respond with only the new description text in <new_description> tags.
```

### 6.5 评估执行类（run_eval.py）

| ID | 机制 | 说明 |
|----|------|------|
| **E1** | `claude -p "{query}"` | 不是 LLM prompt——是对 Claude Code CLI 的黑盒调用，通过 stream-json 解析是否触发 |

---

## 七、两层优化的协作关系

```
                 Skill 内容优化（阶段 4）
                 ┌─────────────────────────────────────┐
                 │  人工驱动：人看输出 → 给反馈 → 改内容  │
                 │  评判标准：输出质量（定性 + 定量）      │
                 │  循环次数：直到人满意                   │
                 │  LLM 角色：执行任务的子代理              │
                 └──────────────┬──────────────────────┘
                                │
                   技能内容稳定后
                                │
                 ┌──────────────▼──────────────────────┐
                 │  Description 触发优化（阶段 5）        │
                 │  自动驱动：脚本跑 eval → LLM 改描述     │
                 │  评判标准：触发准确率（二值：触发/不触发）│
                 │  循环次数：最多 5 轮                    │
                 │  LLM 角色：改写描述文本                 │
                 └─────────────────────────────────────┘
```

**为什么分开**：

1. **内容质量** 必须人工评判（"这个图表好不好看" 无法自动化）
2. **触发准确率** 可以完全自动化（二值判定：触发了没有）
3. 两者的反馈信号不同：内容优化靠自然语言 feedback，触发优化靠 pass/fail 统计
4. 两者的改进对象不同：一个改 SKILL.md body，一个改 frontmatter description

---

## 八、与 AutoSkill / Hermes 的技能优化对比

### 8.1 优化触发方式

| 系统 | 谁触发优化 | 何时优化 | 自动化程度 |
|------|-----------|---------|-----------|
| **Skill-Creator** | 用户主动触发 | 创建/编辑技能时 | 内容优化=人机循环，描述优化=全自动 |
| **AutoSkill SkillEvo** | 系统后台自动 | 用户做类似任务时 | 全自动（LLM 读旧 Skill + 新对话 → 改写） |
| **Hermes 在线** | Nudge 定时提醒 | 会话中周期性 | 半自动（Agent 自行决定是否 patch） |
| **Hermes GEPA** | 用户运行演进脚本 | 离线批量 | 全自动（执行→诊断→变异→再评估） |

### 8.2 优化评判机制

| 系统 | 内容质量评判 | 触发/检索评判 | 是否真实执行 |
|------|------------|-------------|-------------|
| **Skill-Creator** | 4 子代理 + 人工评审 | `claude -p` 黑盒测试触发率 | **是**（子代理真实执行任务 + CLI 真实测试触发） |
| **AutoSkill SkillEvo** | LLM 读对话后改写 | BM25/向量检索 + LLMSkillSelector | 否（纯 LLM 判断） |
| **Hermes 在线** | 无评判，靠系统提示指导 | Agent 自行决定 | 否 |
| **Hermes GEPA** | batch_runner 执行 → LLM-as-judge | 不涉及（优化内容不优化触发） | **是**（真实执行 + trace 分析） |

### 8.3 防过拟合机制

| 系统 | 防过拟合策略 |
|------|-------------|
| **Skill-Creator** | 60/40 train/test 拆分 + 按 test 分数选最佳 + 盲 history（改进时看不到 test 分数） |
| **AutoSkill SkillEvo** | 无（LLM 直接改写，没有量化评估） |
| **Hermes GEPA** | 约束门：新版本必须在所有约束门上 ≥ 旧版本 |

### 8.4 优化粒度

| 系统 | 优化什么 | 优化粒度 |
|------|---------|---------|
| **Skill-Creator** | SKILL.md 内容 + description 触发描述 | 两个独立维度，分别优化 |
| **AutoSkill SkillEvo** | 整个 SKILL.md | 整体改写 |
| **Hermes 在线 patch** | SKILL.md 的局部 | 子字符串替换（old_string → new_string） |
| **Hermes GEPA** | 技能/提示/工具描述 | DSPy 语义变异 |

---

## 九、设计洞察

### 9.1 Skill-Creator 独特的设计选择

**1. 优化 = 核心流程，不是附加功能**

AutoSkill 的 SkillEvo 是后台自动运行的"升级补丁"。Hermes 的 Dreaming 是独立的离线系统。而 Skill-Creator 的创建和优化是**同一个流程**——你不能"创建一个技能然后决定要不要优化它"，创建过程本身就包含多轮测试和改进。

**2. 人工评审 > LLM 评审（内容层面）**

Skill-Creator 在内容质量评判上明确选择了人工评审而非 LLM 自动评分。这不是技术限制，而是设计哲学：

```
"写初稿 → 用全新眼光审视 → 改进" 是指导思想
"the user reviews the results in a browser" 是核心环节
eval-viewer 是精心设计的 SPA，不是简单的日志输出
```

Grader/Comparator/Analyzer 是辅助工具，但最终决策权在人。

**3. 描述优化的黑盒测试思路**

run_eval.py 不让 LLM 评判"这个描述好不好"，而是**真的启动 `claude -p`**，看 Claude 在真实环境中是否触发。这是 end-to-end 黑盒测试——测试的是整个触发链路，不是描述文本本身。

**4. Train/Test 分离 + Blinded History**

```
优化 LLM 只看到 train 集的失败案例 → 提出改进
评估时同时跑 train + test → 用 test 分数选最佳
历史记录中剥离 test 分数 → 改进模型无法 "教考试"
```

这是经典的 ML 防过拟合范式，但应用在了 prompt engineering 上。

### 9.2 跟其他系统的互补关系

```
Skill-Creator：精雕细琢单个技能（深度优化）
  → 适合：重要的、长期使用的核心技能
  → 代价：每个技能需要人工投入时间

AutoSkill：自动从对话中提取技能（广度覆盖）
  → 适合：快速积累大量技能
  → 代价：质量参差不齐，SkillEvo 改进有限

Hermes GEPA：离线批量优化已有技能（深度优化）
  → 适合：已有技能库的系统性提升
  → 代价：需要构建评估数据集 + 计算资源
```

**理想组合**：AutoSkill 自动提取 → Skill-Creator 精炼关键技能 → GEPA 批量优化长尾技能。

---

## 参考资料

**源码**
- [anthropics/skills - skill-creator](https://github.com/anthropics/skills/tree/main/skills/skill-creator)
- 本地路径：`~/.claude/plugins/marketplaces/claude-plugins-official/plugins/skill-creator/`

**官方文档**
- [Agent Skills Specification](https://agentskills.io/specification)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Equipping agents for the real world](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Improving skill-creator: Test, measure, and refine](https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills)
