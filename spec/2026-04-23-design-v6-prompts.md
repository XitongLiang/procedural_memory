# Procedural Memory v6 — 中文提示词汇编

本文件聚合 v6 程序记忆管线中所有中文 LLM 提示词，仅作阅读与审查用途。
运行时仍由 `src/prompts/files/procedural/*.zh.md` 载入，修改提示词时请直接
改源文件，修改后同步更新本汇编。

提示词与管线阶段对应关系：

| 阶段 | 提示词（文件名） | 归档章节 |
|---|---|---|
| §5.1 窗口主题提取 | `topic_extract.zh.md` | [1. 窗口主题提取](#1-窗口主题提取) |
| §5.2 主题边界判定 | `topic_boundary.zh.md` | [2. 主题边界判断](#2-主题边界判断) |
| §6.0 Task Trace | `extract_task_trace.zh.md` | [3. Task Trace 提取](#3-task-trace-提取) |
| §6.1 Fb/Tp 提取 | `extract_exp_tp.zh.md` | [4. Feedback + Tool Preference 提取](#4-feedback--tool-preference-提取) |
| §7.2 TSE Judge | `judge.zh.md` | [5. TSE 匹配裁决](#5-tse-匹配裁决) |
| §7.3 TSE Learn | `learn.zh.md` | [6. TSE Learn（新 candidate 初始 bullets）](#6-tse-learn新-candidate-初始-bullets) |
| §7.4 TSE Enrich | `evolve.zh.md` | [7. TSE Enrich（合并 bullets）](#7-tse-enrich合并-bullets) |
| §8.6 灰区裁决 | `dedup_decide.zh.md` | [8. 候选池查重决策（灰区）](#8-候选池查重决策灰区) |
| §8.x Fb/Tp 合并 | `merge.zh.md` | [9. 程序记忆合并](#9-程序记忆合并) |
| §13 离线升格 | `check_evidence.zh.md` | [10. 候选池升格证据检查](#10-候选池升格证据检查) |

---

## 1. 窗口主题提取

> 源文件：`src/prompts/files/procedural/topic_extract.zh.md`

你是一个对话主题分析专家。从滑动窗口的对话中提取主题元数据，用于后续的主题边界
判断和 Chunk 积累。

### 背景说明

当前窗口由 n=10 轮对话组成，首尾各有 m=2 轮与相邻窗口重叠（step=8）。
**锚点轮次**（anchor_turns）是窗口中间的非重叠部分，代表本窗口最核心的独有
内容；首尾重叠轮次仅作上下文补充，不作为主要提取依据。

### 字段说明

**topic_keywords**（3-8 个）
- 优先从锚点轮次的用户消息中提取
- 聚焦任务对象和操作动词（如"Docker 部署"、"SQL 优化"、"重构认证模块"）
- 过滤停用词（好的/继续/帮我/可以/这个）和短词（< 2 字符）
- 不同表述指同一对象时只保留一个

**topic_summary**（一句话，15-40 字）
- 自然语言描述窗口中"谁在做什么"，供下游 Task Trace / TSE 提取时作为 context
  锚点
- 包含具体对象和动作，例："用户在 production 环境调试 Docker 部署，
  卡在 registry 推送权限"
- 纯闲聊 / 无明确任务 → 输出空串 ""

### 提取原则

1. 以锚点轮次为主要依据；完整窗口仅作上下文参考
2. 关键词来自用户消息，不从 assistant 回复中直接摘取
3. 没有明确任务时（如纯闲聊）→ topic_keywords 返回空列表，
   topic_summary 返回空串

### 输出格式

只输出 JSON：

```json
{
  "topic_keywords": ["关键词1", "关键词2", "关键词3"],
  "topic_summary": "一句话 15-40 字，用户在做什么"
}
```

### [User]

```
锚点轮次（主要依据，窗口非重叠核心内容）：
{{anchor_turns}}

完整窗口（上下文参考）：
{{window_messages}}
```

---

## 2. 主题边界判断

> 源文件：`src/prompts/files/procedural/topic_boundary.zh.md`

你是一个对话主题连续性分析专家。判断两段对话是否属于同一个任务主题，决定是否
将它们合并为同一个 Chunk。

### 同主题 / 不同主题 的判断标准

**同主题**：
- 任务对象相同（同一文件、同一功能、同一问题）
- 后续内容是前续的延伸、深化、修正或补充
- 主题描述不完全重叠但语义属于同一工作流（如"写测试"→"修复测试失败"仍是同
  主题）

**不同主题**：
- 任务类型切换（如从"写代码"转向"写文档"）
- 工作对象切换（如从"前端组件"转向"数据库 schema"）
- 两段主题描述毫无关联

### 判断策略

1. **主题描述对比**（核心）：比较两段 topic_summary，看任务对象和操作是否连续。
2. **关键词参考**（辅助）：summary 描述偏差时（如双方都笼统说"调试问题"），用
   关键词核验是否同一领域；词汇重叠低不等于主题不同，不单独决定结果。

### 注意

片段 A 的主题描述是 chunk 内**最近一个窗口**的 topic_summary（chunk 主题渐变时，
chunk 整体身份按最近主题对齐——chunk 内部细节由后续 Task Trace 阶段重新整理，
不影响本判断）。

### 输出格式

只输出 JSON：`{"same_topic": true|false, "reason": "..."}`

### [User]

```
片段 A 主题描述（chunk 最近窗口的 topic_summary）：{{chunk_summary}}
片段 A 关键词：{{chunk_keywords}}

片段 B 主题描述：{{next_summary}}
片段 B 关键词：{{next_keywords}}
```

---

## 3. Task Trace 提取

> 源文件：`src/prompts/files/procedural/extract_task_trace.zh.md`

你是一个对话整理专家。把一段 agent 工作对话整理为 **Task Trace**——按 task /
sub-task 两级层次组织的、按时间顺序排列的结构化执行记录。

### 1. 输入

**chunk_messages**：一段 agent 工作对话的清洗后消息序列（已过预过滤，无平台
元数据）。每条消息形如：

- `[user] {text}` —— 用户原话
- `[assistant] {text}` —— assistant 输出文本（可能含
  `<thinking>...</thinking>` 段落，那是 agent 的内部推理）
- `[assistant] {toolName}({arguments_json})` —— assistant 发起的工具调用
- `[tool:{toolName} ERROR] {content}` —— 工具调用的错误返回（注：成功返回
  已被上游过滤丢弃，所以你只会看到失败 case；错误内容已截到 300 字符）

### 2. 术语严格定义

**Step** = 对话流里一个**有 procedural learning 价值**的逻辑节点，必须落入
下表 4 类之一：

| Step 类型 | 触发条件 | 必填字段 |
|---|---|---|
| `UserRequest` | user 提出新请求、新约束、新上下文 | Action |
| `AgentAction` | assistant 仅含 text/thinking，无 tool 调用——做了规划、解释或决策表述 | Action, How（仅当含 thinking 时填） |
| `ToolCall` | assistant 含 toolCall 块——调了一次或多次工具 | Action, How, Parameters, Result |
| `UserCorrection` | user 对前一个 Step 表达不满 / 否定 / 偏好补充 | Action, Type, Trigger |

**task** = 你从 chunk_messages 归纳出的 canonical 任务概念，含两部分：
- **name**（4-10 字、动宾结构、抽象不带具体上下文，如 "修登录 bug" /
  "调 Docker 部署"）
- **description**（≤ 30 字一句话、抽象定义、不混入本次具体上下文，如
  "修复登录系统中的代码 bug 或配置问题"）

输出时按 skill 风格 `name：description` 行内写在 `**Task**:` 行（中文全角
冒号分隔）。

**sub-task** = task 内部一个**子事务**对应的 Step 序列。**只需 name**
（10-25 字描述短语，自描述清楚做了什么），不附 description。子事务切换的硬
触发信号：

1. 用户明确换方向（"好，这个搞定，现在做 X"）
2. 前一段交付完成（任务被验证通过 / 用户认可后转下一题）
3. 前一段失败放弃（多次尝试无果，agent 或 user 明示放弃此路径）
4. 工具链彻底切换（从"读 + 写文件"组合切到"运行测试 + 改 BUG"组合）

不切的情况：单纯多轮对同一对象的迭代（如反复修同一个文件）= 同一 sub-task。
所有 sub-task 都从属于本 chunk 唯一的 task。

### 3. 字段语义（每个 Step 都按下表填，不留空，无内容写 "(none)"）

| 字段 | 适用 Step | 写什么 | 长度上限 |
|---|---|---|---|
| `Action` | 全部 | 一句话陈述"做了什么"——动宾结构，主语隐含；用户类 Step 主语隐含为 user，agent 类 Step 主语隐含为 agent | ≤ 30 字 |
| `Context` | UserRequest（可选） | 用户给出的相关背景（如"在排查上次失败的 build"） | ≤ 30 字 |
| `How` | AgentAction（必填当含 thinking）/ ToolCall | 实施方式：含工具名 + 关键决策理由（thinking 段落里"为什么选这个方案"提炼一句）。**不复述 Action**。 | ≤ 50 字 |
| `Parameters` | ToolCall | 关键参数键值对，按下表"关键参数白名单"取，不在表里的工具取 arguments 前 2 个键 | 每参数 value ≤ 80 字，超长截断加 `…` |
| `Result` | ToolCall | 成功 → "OK: {一句话产出概括}"；失败 → "ERROR: {错误首句}"（错误本身已截到 300，再选首行/首句即可） | ≤ 100 字 |
| `Type` | UserCorrection | enum，按下表"纠正类型判定"选 | — |
| `Trigger` | UserCorrection | 引用所纠正 Step 的编号（如 "Step 1.2"）+ 问题简述 | ≤ 30 字 |

#### 3.1 关键参数白名单

按工具名前缀匹配：

| 工具类别 | 匹配规则 | 取的键 |
|---|---|---|
| 文件读写 | name ∈ {Read, Write, Edit, NotebookEdit, cat, head, tail} | `file_path`（必）；Edit 类多取 `old_string`/`new_string` 各 30 字 |
| Shell / 命令执行 | name ∈ {Bash, Execute, exec, sh} | `command`（首 100 字） |
| 搜索 | name ∈ {Grep, grep, Glob, Search, find} | `pattern` 或 `query` |
| 网页 | name ∈ {WebFetch, WebSearch, fetch, http} | `url` 或 `query` |
| 内存工具 | name 以 `memory_` 开头 | `query`（recall）/ `content` 首 50 字（store） |
| Agent / Task | name ∈ {Agent, Task} | `description` 或 `subagent_type` |
| 其他 / 未知 | — | arguments 的前 2 个键，按出现顺序 |

参数值含路径或 ID → 去标识化保留结构（如 `/Users/****/proj/main.c`）。

#### 3.2 纠正类型判定

| Type | 信号词（user 消息内任一即命中） | 例子 |
|---|---|---|
| `correction` | 不对 / 错了 / 别这样 / 改用 / 不要用 / 应该 X 不是 Y / 反了 | "不对，应该用 await，不是 then" |
| `refinement` | 还要 / 也加上 / 顺便 / 记得 / 别忘 / 此外 | "还要考虑超时" |
| `feedback` | 以后 / 下次 / 我习惯 / 我喜欢 / 偏好 / 一律 | "以后这种 case 都用 X" |

同时命中多个 → 按 `correction > feedback > refinement` 优先级取一个。

### 4. 整理规则（按顺序执行）

1. **遍历 chunk_messages**，按消息时间顺序识别 Step；非上述 4 类的消息
   **整体丢**（filler、纯标点、纯标点 ack）。
2. **Filler 严格定义**：
   - user 消息 trim 后 ≤ 5 字符且匹配正则
     `^(好的?|继续|ok|嗯+|收到|了解|yes|y|sure)[。！.!]*$` → 丢
   - assistant 仅 `<thinking>` 块、无 text 也无 toolCall → 把 thinking
     内容并入下一条非 filler assistant 的 `How` 字段，本条不出 Step
3. **Thinking 处理**：thinking 段落不出独立 Step；其内容用于：
   - 紧邻其后的 `ToolCall` Step 的 `How` 字段（"决策理由"那一句从
     thinking 里提炼）
   - 紧邻其后的 `AgentAction` Step（如果 assistant 没调工具）的 `How` 字段
   - thinking 后是 user 消息且没有 agent action → thinking 整体丢
4. **并发 ToolCall 处理**：单条 assistant 消息内多个 toolCall → 每个工具
   一个独立 ToolCall Step，按 arguments 出现顺序编号（Step 1.2、Step 1.3、
   ...）；对应的错误 toolResult 按顺序+toolName 配对（不保证 100% 准，
   下游接受此风险）。
5. **Step 编号**：`Step {subTaskIdx}.{stepIdxInSegment}`，从 1 开始；编号
   只用于 `Trigger` 字段引用，不做跨 chunk 持久化。
6. **sub-task 切分**：沿时间顺序扫 Step，按 §2 的硬触发信号切段；每段
   **10-25 字描述短语**命名，动宾结构 + 必要修饰（"定位 NPE 崩溃点并修栈
   跟踪"、"为新增字段补单元测试"），名字本身要能说清这一段在做什么——不
   需要再附 description。整 chunk 只有一个子事务 → 只产出一个 sub-task
   段。
7. **User Query 汇总**：扫所有 UserRequest Step 的 Action，归纳一句 1-2
   句话的 "User Query"，写在 Trace 头部。多个不相关请求时取最初的那个 +
   简短一句"以及后续 X、Y"。
8. **Step 不打时间戳**：Trace 头部已有 Time Range 字段；Step 内不出时间。
9. **不发明内容**：每个字段必须有 chunk_messages 里的可追溯证据；编不出
   来就写 `(none)`。

### 5. 输出格式

只输出 Markdown，严格按下方模板（不要外包代码围栏）。

- **task** 用 skill 风格行内格式：`name：description`（中文全角冒号分隔）。
- **sub-task** 只写 name，不附 description——name 本身写到 10-25 字描述
  短语级别，足以自描述（不像 task 那样需要额外 description 补语义）。

```markdown
## Task Trace

**User Query**: <一两句话汇总，从所有 UserRequest Step 的 Action 归纳>
**Task**: <name 4-10 字 动宾结构>：<description ≤ 30 字 抽象不带具体上下文>

### sub-task 1 — <name 10-25 字 描述短语，动宾结构 + 必要修饰>

#### Step 1.1 — UserRequest
- Action: <…>
- Context: <… or (none)>

#### Step 1.2 — ToolCall
- Action: <…>
- How: <…>
- Parameters: <key1=value1 / key2=value2>
- Result: <OK: … or ERROR: …>

#### Step 1.3 — UserCorrection
- Action: <…>
- Type: correction
- Trigger: Step 1.2 / <问题简述>

### sub-task 2 — <name>

#### Step 2.1 — …
…
```

约束：
- **不输出 `**Time Range**` 行**——代码后处理时会在 `**Task**:` 行后面插入
- task name 与 sub-task name 都要**稳定**（同一类任务 / 子事务跨 chunk 应给
  同一 name）
- task description 描述**通用任务类型本身**而非本次具体上下文（不能写
  "修复 login.py 的 bug"，要写"修复登录 bug" / 描述"修复登录系统中的代码
  bug 或配置问题"）
- 同一 trace 多个 sub-task 段重名时（不该出现，但兜底）→ 仍按段独立列出，
  下游 UPSERT 时 occurrence 自然合并

### [User]

```
chunk_messages:
{{chunk_messages}}
```

---

## 4. Feedback + Tool Preference 提取

> 源文件：`src/prompts/files/procedural/extract_exp_tp.zh.md`

你是一个程序记忆提取专家。从以下 Task Trace 中提取 feedback 和
tool_preference，每条结果标注归属的 sub-task。

### 知识类型定义

**feedback**（错误更正记忆）— 错误发生后形成的硬约束行为规则，agent 必须
遵循：
- 来源一（用户纠正）：用户亲口说"不要用 X"、"错了，应该 Y"、"顺序不对"等
- 来源二（agent 纠错）：agent 执行失败后自行换方案并成功恢复的路径（"出现
  该报错时，改用此做法"）
- 内容：工具选择纠正、步骤 / 顺序纠正、失败恢复路径、风格 / 格式纠正
- 优先级最高
- 格式："[被纠正的行为] → [正确做法]" 或 "[错误情境] 时，改用 [恢复做法]"
- 长度：content ≤ 50 字
- is_hard = true
- source_user_correction = true（用户纠正） / false（agent 自身纠错）

**tool_preference**（工具偏好）— 同类任务上的稳定工具选择倾向：
- 目标：减少决策开销，提升执行效率
- 格式："做 [任务类型] 时，优先用 [工具]"
- 注意：互补工具组合（"用 A 后配合 B"）→ 不归此类，应忽略
- 长度：content ≤ 50 字
- is_hard = true（如果来自用户纠正）或 false（如果来自 agent 自发规律）

### 证据范围与 sub_task 归属（CRITICAL）

- 你看到的 Task Trace 含若干 `### sub-task N — <短语>` 段
- **证据范围限定在 sub-task 段内**：判定一条 fb/tp 时只看它所在 sub-task 段
  的 Step；不要把段 A 的纠正信号和段 B 的工具调用拼一起当依据
- 产出每条 item 时填 `sub_task` 字段 = 该条证据所在段头部的 `<短语>` 原文
  （原样照抄、含空格标点）
- 同一条规则若在多个段都独立成立 → 每段各出一条（下游 dedup 阶段会按
  sub_task 分区合并）
- 整个 Trace 只有一个 sub-task → 所有 item 的 sub_task 都填该段短语

### 分析框架

**1. 用户纠正分析**
- Trace 里标注 "Type: correction" 的 Step 是主要分析对象
- 提取结果 → feedback（is_hard=true，source_user_correction=true）
- 若纠正涉及工具选择 → 同时产出 tool_preference（is_hard=true）

**2. 失败→恢复分析**
- 扫描 Result 为 ERROR 的 Step，看后续是否有同 sub-task 内的恢复 Step
- 关键：该错误是否可能在下次同类 sub-task 中重现？
- 可泛化的恢复路径 → feedback（is_hard=true，source_user_correction=false）
- 一次性错误 / 环境偶发错误 → 不提取

**3. 工具选择模式分析**
- 同 sub-task 段内反复选择同一工具（≥ 3 次）？→ tool_preference
  （is_hard=false）
- 是否有明确的"优先用 X 不用 Y"模式？→ tool_preference

### 提取规则

- Task Trace 是唯一证据源（已结构化，定位高效）
- 去标识化：用占位符替代具体文件名 / 项目名 / 用户名
- 一次性、不可泛化的操作 → 不提取
- 纯闲聊 / 无操作内容 → 不提取
- 没有任何可提取内容时输出 `{"items": []}`，不要硬凑

### 输出格式

只输出 JSON：

```json
{
  "items": [
    {
      "kind": "feedback" | "tool_preference",
      "sub_task": "<该条证据所在 sub-task 段头部短语原文>",
      "content": "<规则描述，≤ 50 字>",
      "is_hard": true | false,
      "source_user_correction": true | false
    }
  ]
}
```

### [User]

```
{{task_trace}}
```

---

## 5. TSE 匹配裁决

> 源文件：`src/prompts/files/procedural/judge.zh.md`

你是一个严格的匹配判断器。判断当前 chunk 的 task 是否应归入下面某条 candidate
task。

必须是同一类工作任务才算匹配——宽泛 / 切线相关不算。

### 规则

- 仅当当前 task 明确属于某 candidate 的领域时输出该序号（1~N）
- 不确定时倾向于匹配（输出最接近的序号），宁可走 Enrich 也不要重复创建
- 候选列表为空时返回 0

### 输出格式

只输出 JSON：

```json
{
  "selectedIndex": <int>,
  "reason": "<选择原因>"
}
```

- `selectedIndex`: 0 表示无匹配（应创建新 candidate），1~N 表示选择第 N 个
  候选

### [User]

```
当前 task：{{task_name}}
当前 description：{{task_description}}

候选 candidates（序号、task name、description、bullets 数）:
{{candidate_list}}
```

---

## 6. TSE Learn（新 candidate 初始 playbook）

> 源文件：`src/prompts/files/procedural/learn.zh.md`

你是一个任务经验提取专家。从 Task Trace 中提取本次执行总结出的可复用
经验，输出**编号 playbook 纯文本**，直接作为该 task 的新 candidate
初始内容。

> **契约升级（v6.1，2026-04-24）**：输出从 `{"bullets":[{"text":"..."}]}`
> JSON 改为纯文本编号 playbook，下游零解析直接写入 candidate.content +
> 升格时原样进 mem_record.content。

### bullet 定义

- 每条 ≤ 50 字（`1.` 编号前缀不占字数额度），描述"做这类任务时
  应该 / 可以 / 注意 ..."的执行智慧
- 写"做法"，不写"具体做了什么"
- 去标识化：用占位符替代具体文件名 / 项目名 / 数值

### 提取来源

- Step 内 **Reasoning 字段的场景洞察**（"在场景 C 下..." 的泛化部分）
- ToolCall 聚合 Step 的 `(共 N 次)` 高频模式（N ≥ 3）
- ERROR → 恢复 Step 的可泛化规律
- AgentAction 的 How 字段（规划 / 框架切入）

### 不提取（避免与 fb / tp 重复）

- Trace 里某 sub-task 段已标 `Type: correction` 的 → 那是 feedback
- sub-task 内 `**Constraints**` / `**Preferences**` 原文（除非能 task 化改写）
- 头部 `**User Preferences**` 块（冗余聚合）
- Step 的 `Params` / `Result` 原始内容、`Action` 原文
- 一次性、不可泛化的操作

### 输出格式

**编号 + 点 + 空格 + 正文；条与条之间空一行**：

```
1. 先 tail -n 200 看日志再决定是否全读

2. 改关键模块前先梳理上下游调用点

3. 异常路径补 log 输出参数原值
```

- 第一个字符必须是 `1`；不用 `-` / `•` / Markdown 加粗 / JSON 包装
- 无可提取内容 → **完全空输出**（零字节），下游据此跳过创建 candidate
- bullet 不区分 hard/soft——TSE 整体按次数规则升格

### [User]

```
{{task_trace}}
```

---

## 7. TSE Enrich（合并 playbook）

> 源文件：`src/prompts/files/procedural/evolve.zh.md`

你是一个任务经验合并专家。把本次 chunk 的 Task Trace 中可复用的经验合并进
已有 candidate 的 playbook，输出**合并后的完整编号 playbook 文本**；
若无新增则输出空字符串。

> **契约升级（v6.1，2026-04-24）**：输出从
> `{"action":"enrich|no_change","bullets":[...]}` JSON 改为纯文本；
> `action=no_change` 由**空输出**隐式表达（零字节），`action=enrich`
> 由**非空输出**隐式表达。下游按字节判断路径。

### 合并原则

1. **语义并集**：去重融合，意思相近的多条留表述最清晰的一份
2. **防退化**：不能丢已有 bullets 的有效约束
3. **RECENCY BIAS**：冲突时新 chunk 的表述优先（近期意图）
4. **去标识化**：移除项目名 / 文件名 / 具体值，用占位符
5. **不发明内容**：只用 Task Trace 里有证据的步骤 / 技巧
6. **单条 bullet ≤ 50 字**

### 提取来源 / 不提取

同 §6。

### 输出形态判定（由输出字节数隐式表达）

- **有新增 / 质量提升** → 输出**完整合并后的编号 playbook**
  （覆盖原 candidate.content，整段替换）
- **无新增 / 现有已覆盖所有候选条目** → 输出**空字符串**
  （下游仅 `mention_count++`，不改 content）

### 输出格式

同 §6：`1. …\n\n2. …\n\n3. …`。**严格编号文本**，不用 `-` / JSON 包装。
输出的是**合并后完整 playbook**，不是 diff。

### [User]

```
candidate 当前 bullets：
{{candidate_bullets}}

新 Task Trace：
{{task_trace}}
```

---

## 8. 候选池查重决策（灰区）

> 源文件：`src/prompts/files/procedural/dedup_decide.zh.md`

已有条目与新提取条目已通过 `(user_id, kind)` 分区筛选（v6.1 起**取消
anchor 分区**，统一全量加载 + 向量 hybrid 打分——LLM 生成的 task /
sub-task 名字不稳定，按字符串等值分区会漏掉同语义不同措辞的条目），
且 hybrid score（§2.3：`0.80·cos + 0.20·kw`）落在灰区 0.50–0.85。
判断它们是否是**同一条规则**：

- **merge**：描述同一个可迁移的规则 / 偏好，只是表述有漂移、侧重点略不同
  → 合并为一条（保留最新表述，is_hard 取 OR）
- **add**：虽然 sub_task 相同，但描述的是该子事务里**另一条独立的规则**
  → 作为新条目并存

### 决策原则

1. **优先 add**（宁可多存一条，不误合并不同规则）
2. merge 需要同时满足：
   - 指向同一个操作对象或同一种偏好
   - 两条描述的条件 / 结论不冲突（冲突时按 RECENCY BIAS 取新的，仍可 merge）
3. 条件或结论出现**实质性差异**（不同约束、不同工具、不同场景）→ 必须 add

### 输出格式

只输出 JSON：`{"action": "merge" | "add", "reason": "..."}`

### [User]

```
所属 sub_task：{{sub_task}}

已有条目（历史）：
  kind: {{existing.kind}}
  content: {{existing.content}}
  is_hard: {{existing.is_hard}}

新提取条目（近期）：
  kind: {{candidate.kind}}
  content: {{candidate.content}}
  is_hard: {{candidate.is_hard}}
```

---

## 9. 程序记忆合并

> 源文件：`src/prompts/files/procedural/merge.zh.md`

上游已确认两条候选条目描述的是同一条可复用规则（由 ≥0.85 hybrid 直通 merge，
或 0.50–0.85 灰区 dedup_decide 返回 merge 落地）。现在需要把"已有条目"与
"新条目"的 **content 文本**融合为一条规范化描述。

### 合并原则

1. **语义并集**：保留双方所有关键信息（约束 / 对象 / 动作），是融合不是拼接
2. **防退化**：已有条目中的独有约束不得丢失
3. **去标识化**：移除案例特定实体（人名 / 项目名 / 具体文件路径），保留可迁移规则
4. **Anti-Duplication**：近似表达只保留一个规范版本
5. **不发明内容**：只用两边原有信息，不添加新推测
6. **最近性原则（RECENCY BIAS）**：
   - 条件 / 结论冲突时，取新条目的表述（近期意图优先）
   - 已有条目的独有约束若与新表述不冲突则保留

### 输出要求

- `content` ≤ 50 字，清晰可操作
- 以触发条件开头（如"遇到 X 时..."、"做 X 时优先用 Y"）
- 中文输出
- 其它字段（kind / sub_task / is_hard）由调用方按元数据规则合并，**不要**在
  本次输出里返回

只输出 JSON：

```json
{"content": "<合并后内容>", "reason": "<为什么这样合，≤30字>"}
```

LLM 不可用、JSON 非法、或返回 `content=""` 时，上游回退到 RECENCY BIAS
（新条目内容直接覆盖旧条目）。仅在 `kind ∈ {feedback, tool_preference}` 时
调用本 prompt；TSE 结构化内容的合并走 `evolve.zh.md`。

### [User]

```
sub_task: {{sub_task}}

已有条目（历史）:
  {{existing.content}}

新条目（近期）:
  {{candidate.content}}
```

---

## 10. 候选池升格证据检查

> 源文件：`src/prompts/files/procedural/check_evidence.zh.md`

判断一条程序记忆条目是否在给出的证据 traces 里真的成立。

- 如果条目描述的规则 / 经验能在 traces 里找到具体对应的发生过的事件或模式
  → passed=true
- 如果 traces 里找不到足够支撑这条规则的实际痕迹（可能是 LLM 瞎编 / 过拟合
  单次偶发 / 证据薄弱）
  → passed=false

### 输出格式

只输出 JSON：`{"passed": true | false, "reason": "<一句话说明>"}`

### [User]

```
条目类型：{{candidate.kind}}
条目锚点（sub_task / task）：{{candidate.anchor}}
条目内容：{{candidate.payload}}

证据 traces（最近 {{N}} 条）：
{{traces_concatenated}}
```
