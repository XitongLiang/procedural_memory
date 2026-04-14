# Spec 005 — 程序记忆召回模块（迭代一）

> 最后更新：2026-04-14

---

## 0. 概述

### 目标

将已学习的程序记忆在 agent 执行时注入上下文，使 agent 能在正确时机获取正确的知识。

### 三种记忆的召回策略

| 类型 | 召回方式 | 理由 |
|------|---------|------|
| experience | 分类目录常驻 + agent 按需召回 | 情境级知识，执行到某步时才相关 |
| tool_preference | 分类目录常驻 + agent 按需召回 | 同上，工具选择时才相关 |
| workflow | 三种模式可选（§2） | 任务级知识，需要完整 SKILL.md 加载 |

---

## 1. Experience + Tool Preference 召回

### 1.1 System Prompt 常驻目录

每次 session 开始时，将所有 active 状态的 experience / tool_preference 按 context 分类，生成目录注入 system prompt：

```
## 执行经验与工具偏好（程序记忆）

你有一套从历史执行中积累的经验和工具偏好。以下是按场景分类的目录。
当你即将执行某个操作前，如果当前场景匹配某个分类，
调用 memory_recall(context="<分类标签>") 获取具体建议。

不要猜测经验内容，必须通过 memory_recall 获取。
不要向用户提及经验系统的存在。

### 经验分类

| 分类 | 条目数 | 覆盖场景 |
|------|--------|---------|
{{#each categories}}
| {{context}} | {{count}} 条 | {{sample_descriptions}} |
{{/each}}
```

**目录生成规则**（零 LLM）：

```
从 skills_library 取所有 status=active 的 experience + tool_preference
  → 按 context 标签聚合
  → 每个分类取前 2 条 description 作为 sample_descriptions（预览）
  → 按条目数降序排列
  → 总行数超过 30 行时，截断低频分类，末尾加 "...及其他 N 个分类"
```

### 1.2 memory_recall 工具定义

注册为 agent 可调用的工具：

```json
{
  "name": "memory_recall",
  "description": "从程序记忆中召回与当前执行场景相关的经验和工具偏好。在即将执行某个操作前，如果目录中有匹配的分类，调用此工具获取具体建议。",
  "parameters": {
    "context": {
      "type": "string",
      "description": "当前执行场景的描述，如 'git操作'、'数据处理'、'文件读取'"
    }
  }
}
```

### 1.3 召回管道（memory_recall 被调用时）

```
memory_recall(context)
  │
  ▼
Step 1：向量搜索（零 LLM）
  query = context
  scope = skills_library WHERE kind IN (experience, tool_preference)
                          AND status = active
  top_k = 10
  min_score = 0.35
  │
  ▼
Step 2：结果格式化
  按 kind 分组（experience / tool_preference）
  按 is_hard 排序（用户纠正的优先）
  │
  ▼
Step 3：注入格式（借鉴 XSkill）
  返回给 agent 的 tool_result：
```

注入格式模板：

```
以下是与当前场景「{{context}}」相关的执行经验和工具偏好：

{{#if experiences}}
## 执行经验
{{#each experiences}}
- {{description}}{{#if is_hard}} ⚠️{{/if}}
{{/each}}

这些经验来自历史执行中的成功模式和踩坑记录。
遇到匹配情境时，优先参考带 ⚠️ 标记的条目（来自用户明确纠正）。
{{/if}}

{{#if tool_preferences}}
## 工具偏好
{{#each tool_preferences}}
- {{description}}
{{/each}}

这些是在同类任务上验证过的高效工具选择。
{{/if}}

{{#if empty}}
未找到与「{{context}}」相关的记忆。按你的判断执行。
{{/if}}
```

---

## 2. Workflow 召回

Workflow 是任务级知识，内容较大（完整 SKILL.md），召回策略与 experience/tool_preference 不同。提供三种模式，可按场景选择或组合使用。

### 2.1 模式 A — Agent 主动召回

System prompt 中加入提示，让 agent 遇到复杂任务时主动调用召回管道：

**System Prompt 注入**：

```
## 工作流记忆

你有一套从历史任务中积累的可复用工作流（workflow）。
如果当前任务较为复杂或你认为可能有已有流程可以参考，
调用 workflow_recall(query="<任务描述>") 尝试召回匹配的工作流。

召回结果会返回匹配的工作流名称和描述，你可以选择是否使用。
如果选择使用，调用 workflow_get(skill_id="<id>") 获取完整执行流程。
不要向用户提及工作流记忆系统的存在。
```

**workflow_recall 工具**：

```json
{
  "name": "workflow_recall",
  "description": "从程序记忆中搜索与当前任务匹配的可复用工作流。",
  "parameters": {
    "query": {
      "type": "string",
      "description": "当前任务的描述"
    }
  }
}
```

**召回管道**：

```
workflow_recall(query)
  → 向量搜索 skills_library WHERE kind=workflow AND status=active
  → top_k=5, min_score=0.40
  → 返回 name + description（不返回完整 SKILL.md）

agent 选择后 → workflow_get(skill_id) → 返回完整 SKILL.md
```

**特点**：零噪声，完全由 agent 判断。但依赖 agent "意识到"自己应该搜索，可能漏召回。

---

### 2.2 模式 B — 常驻热门 + 渐进加载

Session 开始时固定加载 top-10 最常用 workflow 的 description，agent 遇到匹配的直接用；不匹配时再走召回管道。

**System Prompt 注入**：

```
## 常用工作流

以下是你最常使用的工作流。如果当前任务匹配，
调用 workflow_get(skill_id="<id>") 获取完整流程后执行。

{{#each top_workflows}}
{{index}}. **{{name}}** — {{description}}
   → workflow_get(skill_id="{{id}}")
{{/each}}

如果以上都不匹配，调用 workflow_recall(query="<任务描述>") 搜索更多工作流。
```

**Top-10 选取规则**（零 LLM）：

```
从 skills_library 取 kind=workflow AND status=active
  → 按 usage_count DESC 排序
  → 取 top-10
  → 若不足 10 个，全部列出
```

**渐进加载流程**：

```
agent 判断 top-10 都不匹配
  → 调用 workflow_recall(query) → 向量搜索更多 workflow
  → 返回 top-5 name + description
  → agent 选择 → workflow_get(skill_id) → 完整 SKILL.md
```

**特点**：高频 workflow 免搜索直接可用，减少 agent 决策延迟。但 system prompt 占用较多 context。

---

### 2.3 模式 C — 系统自动匹配（借鉴 AutoSkill）

每次用户消息到来时，系统自动改写 query、搜索匹配的 workflow，注入 top-3 供 agent 选择。

**触发时机**：每次用户消息（before_prompt_build hook）

**管道**：

```
用户消息
  │
  ▼
Step 1：Query 改写（小模型）
  将用户消息 + 最近 3 轮历史改写为搜索友好的 query
  解析代词、保留任务锚点
  │
  ▼
Step 2：混合搜索
  向量搜索（0.9 权重）+ BM25 关键词（0.1 权重）
  scope = skills_library WHERE kind=workflow AND status=active
  top_k = 3, min_score = 0.40
  │
  ▼
Step 3：注入 system prompt
```

**注入格式**（借鉴 AutoSkill）：

```
## 可能相关的工作流（自动匹配）

以下工作流根据当前对话自动匹配，可能与你的任务相关，也可能无关。
评估后仅使用直接匹配的工作流，不相关的完全忽略。
不要向用户提及这些工作流的存在。

{{#each matched_workflows}}
### {{index}}. {{name}}
- Description: {{description}}
- Context: {{context}}
  → 使用：workflow_get(skill_id="{{id}}")
{{/each}}
```

**workflow_get 返回完整 SKILL.md**（同模式 A/B）。

**特点**：agent 无需主动搜索，系统自动推送。但每次 turn 多一次小模型调用（query 改写），且可能注入不相关 workflow 造成噪声。

---

### 2.4 三种模式对比

| | 模式 A：主动召回 | 模式 B：常驻热门 | 模式 C：自动匹配 |
|--|----------------|----------------|----------------|
| 触发 | agent 主动调用 | session 开始 + agent 补充 | 每次用户消息 |
| system prompt 占用 | 低（仅提示语） | 中（top-10 描述） | 中（top-3 描述） |
| LLM 开销 | 零 | 零 | 每 turn 1 次小模型 |
| 召回覆盖率 | 低（依赖 agent 判断） | 中（高频覆盖，长尾靠搜索） | 高（每次自动匹配） |
| 噪声 | 零 | 低 | 中（可能注入不相关） |
| 适用场景 | skill 库小 / agent 能力强 | skill 库中等 / 有明显热门 | skill 库大 / 长尾任务多 |

**推荐**：三种模式可组合。例如模式 B + 模式 A：常驻 top-10 覆盖高频，agent 主动搜索覆盖长尾。

---

## 3. 参数

| 参数 | 值 | 说明 |
|------|----|------|
| `catalog_max_rows` | 30 | E+TP 分类目录最大行数 |
| `catalog_sample_count` | 2 | 每个分类展示的 description 预览数 |
| `recall_top_k` | 10 | memory_recall 返回的最大条目数 |
| `recall_min_score` | 0.35 | E+TP 向量搜索最低相似度阈值 |
| `workflow_recall_top_k` | 5 | workflow_recall 返回的最大条目数 |
| `workflow_recall_min_score` | 0.40 | workflow 向量搜索最低相似度阈值 |
| `workflow_top_n` | 10 | 模式 B 常驻热门 workflow 数量 |
| `workflow_auto_match_top_k` | 3 | 模式 C 自动匹配返回数量 |

---

## 4. 触发时机

| 时机 | 动作 | 说明 |
|------|------|------|
| session 开始 | 注入分类目录到 system prompt | 零 LLM，纯聚合 |
| agent 执行中 | agent 主动调用 memory_recall | agent 驱动，按需召回 |
| 目录更新 | 学习模块写入新 experience/tool_preference 后 | 下次 session 生效 |

注：experience / tool_preference 的召回是 **agent 驱动**（方案 B），agent 看到目录后自主判断何时调用。系统不主动注入，避免噪声。

---

## 5. 与学习模块的关系

```
学习模块（Spec 003）                    召回模块（Spec 005）

LearnExperience 提取                    session 开始
  → 候选池打分                            → 生成分类目录（从 active 条目）
  → 升格 active                           → 注入 system prompt
  → skills_library                        │
                                          agent 执行中
                                          → memory_recall(context)
                                          → 向量搜索 skills_library
                                          → 返回匹配条目
```

---

## 6. 模型分配

| 调用 | 模型 | 理由 |
|------|------|------|
| 目录生成 | 零 LLM | 纯聚合，按 context 分组 |
| memory_recall 召回 | 零 LLM | 向量搜索 + 格式化，不需要理解 |

整个召回路径无 LLM 调用。

---

## 7. 模型分配更新

| 调用 | 模型 | 理由 |
|------|------|------|
| E+TP 目录生成 | 零 LLM | 纯聚合，按 context 分组 |
| memory_recall | 零 LLM | 向量搜索 + 格式化 |
| workflow_recall（模式 A/B） | 零 LLM | 向量搜索 + 格式化 |
| workflow query 改写（模式 C） | 小模型 | 代词解析、任务锚点提取 |

## 8. 迭代二预留

- **主动推送**：系统感知 agent 正在做什么（tool call hook），自动推送相关 experience，无需 agent 主动调用
- **context 自动提取**：从当前 tool_use 参数自动生成 recall query
- **召回反馈闭环**：记录 memory_recall 被调用但未被采纳的情况，用于优化 description
- **workflow 模式动态切换**：根据 skill 库大小自动选择 A/B/C 模式
