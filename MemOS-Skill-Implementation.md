# MemOS Skill 实现分析

## 项目概览

MemOS 是一个 AI 记忆操作系统，包含两套独立实现：

| | Python 端 | TypeScript 端 |
|---|-----------|---------------|
| 定位 | 云端/自托管 API 服务 | 本地嵌入式插件 |
| 框架 | FastAPI (port 8001) | OpenClaw Plugin SDK |
| 存储 | 多后端 | SQLite (本地文件) |
| Skill 处理 | 批量聚类 | 实时增量判断 |
| 依赖 | 独立部署 | 进程内运行，无网络依赖 |

两端实现相同的概念（memory cube、skill evolution、hybrid search），但代码完全独立。

---

## 一、TS 端整体架构

### 1.1 插件注册

入口文件：`apps/memos-local-openclaw/index.ts`

通过 OpenClaw Plugin SDK 注册，声明文件 `openclaw.plugin.json`：

```typescript
const memosLocalPlugin = {
  id: "memos-local-openclaw-plugin",
  kind: "memory",
  register(api: OpenClawPluginApi) {
    // 1. 注册记忆能力（构建 system prompt 中的记忆指引）
    api.registerMemoryCapability({ promptBuilder: buildMemoryPromptSection });

    // 2. 注册工具（供 Agent 主动调用）
    api.registerTool("memory_search", ...);
    api.registerTool("memory_get", ...);
    api.registerTool("skill_get", ...);
    // ... 共 ~20 个工具

    // 3. 注册生命周期 Hook
    api.on("before_prompt_build", ...);  // prompt 构建前：自动召回记忆/Skill
    api.on("agent_end", ...);            // 对话结束后：捕获消息、存储、触发 Skill 进化
  }
};
```

### 1.2 两个核心 Hook

#### `before_prompt_build` — 自动召回

用户发消息前触发，注入相关记忆和 Skill 到 prompt：

```
用户消息 → normalizeAutoRecallQuery（清洗 query）
  → 并行搜索：本地记忆 + Hub 记忆
  → LLM 过滤相关性
  → 搜索相关 Skill
  → 返回 { prependContext: "..." } 注入 system prompt
```

#### `agent_end` — 消息捕获与处理

对话结束后触发，捕获新消息并处理：

```
event.messages（全量消息）
  → 游标增量提取（sessionMsgCursor）
  → captureMessages（清洗）
  → worker.enqueue（存储）
  → TaskProcessor（主题分割）
  → SkillEvolver（Skill 提取）
```

---

## 二、消息捕获流程

### 2.1 游标机制（增量提取）

文件：`index.ts` L2155-2298

OpenClaw 每次 `agent_end` 给的是全量消息，用游标避免重复处理：

```typescript
const sessionMsgCursor = new Map<string, number>();

// 首次遇到该 session：定位到最后一条 user 消息
if (!sessionMsgCursor.has(cursorKey)) {
  let lastUserIdx = findLastUserMessage(allMessages);
  sessionMsgCursor.set(cursorKey, lastUserIdx);
}

// 只取游标之后的新消息
const newMessages = allMessages.slice(cursor);
sessionMsgCursor.set(cursorKey, allMessages.length);
```

示意：

```
第1轮: messages = [u1, a1]             → cursor 0→2, 处理 [u1, a1]
第2轮: messages = [u1, a1, u2, a2]     → cursor 2→4, 处理 [u2, a2]
第3轮: messages = [u1, a1, u2, a2, u3, a3] → cursor 4→6, 处理 [u3, a3]
```

### 2.2 captureMessages — 消息清洗

文件：`src/capture/index.ts`

输入原始消息，输出干净的 `ConversationMessage[]`：

**过滤规则**（直接丢弃）：

| 类型 | 示例 |
|------|------|
| system 角色 | 系统提示 |
| 哨兵回复 | `NO_REPLY`, `HEARTBEAT_OK`, `HEARTBEAT_CHECK` |
| 启动检查 | `"You are running a boot check..."` |
| 自身工具结果 | `memory_search`, `memory_get`, `skill_search` 等（防循环） |
| 系统样板 | `/new` 或 `/reset` 提示 |

**清洗规则**（剥离元数据，保留内容）：

- 用户消息 → `stripInboundMetadata()`：
  - 去掉 OpenClaw 元数据（`Sender (untrusted metadata):` + JSON 块）
  - 去掉注入的记忆上下文（`<memory_context>...</memory_context>`）
  - 去掉时间戳前缀 `[Tue 2026-03-03 21:58 GMT+8]`
  - 去掉 `[message_id: ...]`、`[[reply_to_current]]` 等标签
- 助手消息：
  - `stripThinkingTags()` — 去掉 `<think>...</think>`（DeepSeek 推理标签）
  - `stripEvidenceWrappers()` — 去掉证据引用包装

**输出结构**：

```typescript
{
  role: "user" | "assistant" | "tool",
  content: string,      // 清洗后的纯文本
  timestamp: number,    // 毫秒时间戳
  turnId: string,       // 本轮唯一 ID
  sessionKey: string,   // 会话标识
  toolName?: string,    // tool 消息的工具名
  owner: string,        // "agent:main" 等
}
```

---

## 三、消息存储（IngestWorker）

文件：`src/ingest/worker.ts`

### 3.1 处理流程

```
enqueue(messages)
  → 过滤临时 session（temp:、internal:、system:）
  → 逐条 ingestMessage()
    → summarizer.summarize(content)  // LLM 生成摘要
    → embedder.embed([summary])      // 生成向量
    → 去重检查（两级）
    → store.insertChunk(chunk)       // 写入 SQLite
    → store.upsertEmbedding(...)     // 写入向量索引
  → taskProcessor.onChunksIngested() // 触发主题分割
```

### 3.2 两级去重

**Level 1 — 精确去重**：`content_hash` 完全匹配 → 淘汰旧 chunk，保留新的

**Level 2 — 语义去重**：
- 向量相似度 > 0.80 的 top-5 候选
- LLM 判断：
  - `DUPLICATE` → 淘汰旧的，新的保持 active
  - `UPDATE` → 合并摘要，淘汰旧的
  - `NEW` → 直接存储

### 3.3 Chunk 数据结构

```typescript
interface Chunk {
  id: string;
  sessionKey: string;
  turnId: string;
  seq: number;
  role: "user" | "assistant" | "tool";
  content: string;         // 原始内容
  kind: "paragraph";
  summary: string;         // LLM 生成的摘要
  embedding: number[];     // 向量
  taskId: string | null;   // 所属任务
  skillId: string | null;  // 关联 Skill
  owner: string;
  dedupStatus: "active" | "duplicate" | "merged";
  mergeCount: number;
  mergeHistory: string;    // JSON 合并历史
  createdAt: number;
  updatedAt: number;
}
```

---

## 四、主题分割（TaskProcessor）

文件：`src/ingest/task-processor.ts`

### 4.1 触发时机

每次 IngestWorker 存完一批 chunks 后调用 `onChunksIngested()`。

### 4.2 判断逻辑

三种条件触发"新任务"：

| 条件 | 判断方式 | 阈值 |
|------|----------|------|
| Session 变了 | 直接切 | — |
| 时间间隔超限 | 直接切 | > 2h |
| 主题变了 | LLM 判断 | confidence ≥ 0.65 |

### 4.3 增量主题检测流程

```
未分配的 chunks → 按 user turn 分组
  ↓
逐 turn 处理：
  ↓
当前任务有 ≥1 个 user turn？
  → 没有 → 直接归入当前任务
  → 有 → 继续判断
  ↓
时间间隔检查（> 2h？）
  → 超时 → finalize 旧任务，创建新任务
  ↓
buildTopicJudgeState() 构建上下文：
  - topic: 第一条 user 消息摘要
  - 最近 3 轮 user/assistant 摘要
  - 短消息/代词开头 → 附带上一条 assistant 回复
  ↓
summarizer.classifyTopic(taskState, newMsg)
  → LLM 返回：{ decision: "SAME"|"NEW", confidence, boundaryType, reason }
  ↓
decision == "NEW" && confidence < 0.65？
  → 二次仲裁 arbitrateTopicSplit()
  ↓
确认 NEW → finalizeTask(旧任务) → 创建新任务
```

### 4.4 任务 Finalize

```typescript
async finalizeTask(task: Task) {
  const chunks = this.store.getChunksByTask(task.id);

  // 过滤：太少(<4 chunks)、太短(<200 chars)、纯闲聊等 → 标记 skipped
  const skipReason = this.shouldSkipSummary(chunks);
  if (skipReason) { /* 标记 skipped */ return; }

  // 生成任务摘要
  const summary = await this.summarizer.summarizeTask(conversationText);

  // 更新任务状态
  this.store.updateTask(task.id, { title, summary, status: "completed" });

  // 触发 Skill 提取回调
  this.onTaskCompletedCallback(finalized);
}
```

**跳过条件**（任一满足则不生成摘要）：
1. chunks < 4
2. 真实对话轮数 < 2
3. 无 user 消息
4. 总内容 < 200 字符
5. 用户内容是闲聊/测试（hello、test、ok 等）
6. 全是 tool 结果
7. 高重复率（调试循环）

---

## 五、Skill 提取与进化（SkillEvolver）

文件：`src/skill/evolver.ts`

### 5.1 触发条件

**输入不是单条消息，而是一个完整的已结束任务（Task）及其全部 chunks。**

```
Task finalize → onTaskCompletedCallback → SkillEvolver.onTaskCompleted(task)
```

时间线示意：

```
轮1-3: 用户讨论主题A  →  归入 Task-1
轮4:   用户切换到主题B →  触发 Task-1 finalize
                            ↓
                        Task-1 全部 chunks → SkillEvolver
                            ↓
                        评估整体是否值得提取 Skill
```

### 5.2 完整流程

```
SkillEvolver.process(task)
  ↓
[1] Rule Filter（evaluator.passesRuleFilter）
    - chunks ≥ 5
    - status != "skipped"
    - summary ≥ 100 字符
    - 必须有 user + assistant 消息
  ↓
[2] LLM 评估是否值得（evaluator.evaluateCreate）
    - CREATE_EVAL_PROMPT
    - 标准：可重复？可迁移？有技术深度？
    - 排除：纯 Q&A、一次性任务、闲聊
    - 输出：{ shouldGenerate, reason, suggestedName, suggestedTags, confidence }
  ↓
[3] 搜索相关已有 Skill（findRelatedSkill）
    - FTS + 向量搜索（阈值 0.35）
    - Top-10 候选 → LLM 严格判断哪个高度相关
  ↓
[4A] 无匹配 → 创建新 Skill（handleNewSkill → SkillGenerator.generate）
    - Step 1: STEP1_SKILL_MD_PROMPT → 生成 SKILL.md
    - Step 2: STEP2_SCRIPTS_PROMPT → 提取脚本
              STEP2B_REFS_PROMPT → 提取引用文档
    - Step 3: STEP3_EVALS_PROMPT → 生成测试用例
    - 验证（SkillValidator）
    - 写入 DB + 文件系统
  ↓
[4B] 有匹配 → 升级已有 Skill（handleExistingSkill → SkillUpgrader.upgrade）
    - UPGRADE_EVAL_PROMPT → 判断升级价值
    - 升级类型：refine | extend | fix
    - UPGRADE_PROMPT → 合并生成新版本 SKILL.md
    - 重建附属文件
    - 验证 → 保存新版本，版本号 +1
  ↓
[5] 安装（SkillInstaller）
    - 决定安装模式：inline / on_demand / install_recommended
    - 构建附属文件清单
```

### 5.3 相关文件

| 文件 | 职责 |
|------|------|
| `src/skill/evolver.ts` | 总调度器 |
| `src/skill/evaluator.ts` | 评估是否值得创建/升级 |
| `src/skill/generator.ts` | 从任务生成新 Skill |
| `src/skill/upgrader.ts` | 升级已有 Skill |
| `src/skill/validator.ts` | 质量验证（0-10 分） |
| `src/skill/installer.ts` | 安装管理 |

### 5.4 Skill 数据模型

**数据库（SQLite）**：

```typescript
interface Skill {
  id: string;
  name: string;                    // kebab-case，如 "deploy-docker-app"
  description: string;             // 60-120 词，用于触发匹配
  version: number;                 // 每次升级 +1
  status: "active" | "archived" | "draft";
  tags: string;                    // JSON 数组
  sourceType: "task" | "manual";
  dirPath: string;                 // 文件系统路径
  installed: number;               // 是否已安装
  owner: string;
  visibility: "private" | "public";
  qualityScore: number | null;     // 0-10
  createdAt: number;
  updatedAt: number;
}

interface SkillVersion {
  id: string;
  skillId: string;
  version: number;
  content: string;                 // 完整 SKILL.md
  changelog: string;
  changeSummary: string;
  upgradeType: "create" | "refine" | "extend" | "fix";
  sourceTaskId: string | null;
  metrics: string;                 // JSON: dimensions, confidence, validation
  qualityScore: number | null;
  createdAt: number;
}
```

**文件系统**：

```
{stateDir}/skills-store/{skill-name}/
  ├── SKILL.md          # 主文档（YAML frontmatter + 内容）
  ├── scripts/          # 可执行脚本
  ├── references/       # 补充文档
  └── evals/evals.json  # 测试用例
```

### 5.5 Skill 进化机制

每个 Skill 保留完整版本历史：

| 升级类型 | 含义 |
|----------|------|
| `create` | 新建 |
| `refine` | 改进清晰度、修正错误、更优方案 |
| `extend` | 扩展场景、增加适用范围 |
| `fix` | 修正过时/错误信息 |

进化内容包括：
- description 更新（新增触发关键词/场景）
- procedure 丰富（补充边界情况）
- pitfalls 扩展（新增遇到的失败模式）
- examples 更新（真实输出样例）

---

## 六、Python 端 Skill 处理

核心文件：
- `src/memos/templates/skill_mem_prompt.py` — 所有 prompt 模板
- `src/memos/mem_reader/read_skill_memory/process_skill_memory.py` — 主处理管线

### 6.1 与 TS 端的区别

| | Python 端 | TS 端 |
|---|-----------|-------|
| 时机 | 对话结束后批量处理 | 实时增量，任务 finalize 时触发 |
| 主题分割 | 一次性把全部消息按主题聚类 | 逐 turn 线性判断 |
| 非连续任务 | 支持回溯（A→B→A 归为同一主题） | 线性切割，不回溯 |
| Skill 提取 | 两阶段（简单 + 详细） | 评估 → 生成/升级 |
| Prompt | `TASK_CHUNKING_PROMPT` 等 | `CREATE_EVAL_PROMPT` 等 |

### 6.2 处理流程

```
对话消息（全量）
  ↓
Phase 1: 任务分块（_split_task_chunk_by_llm）
  - TASK_CHUNKING_PROMPT / TASK_CHUNKING_PROMPT_ZH
  - 将消息按独立任务分组（支持非连续）
  - 过滤闲聊
  ↓
Phase 2: 召回相关 Skill（_recall_related_skill_memories）
  - TASK_QUERY_REWRITE_PROMPT → 重写查询
  - 搜索 top-5 相关已有 SkillMemory
  ↓
Phase 3A: 简单提取（_extract_skill_memory_by_llm）
  - SKILL_MEMORY_EXTRACTION_PROMPT
  - 输出：name, description, procedure, experience, preference, examples, tags
  ↓
Phase 3B: 详细提取（_extract_skill_memory_by_llm_md）
  - SKILL_MEMORY_EXTRACTION_PROMPT_MD（增加 trigger 关键词）
  - 并行生成附加内容：
    - SCRIPT_GENERATION_PROMPT → 可执行脚本
    - TOOL_GENERATION_PROMPT → 推荐工具
    - OTHERS_GENERATION_PROMPT → 补充文档
```

### 6.3 Prompt 清单

| Prompt | 文件行号 | 用途 | 输出 |
|--------|----------|------|------|
| `TASK_CHUNKING_PROMPT` | L1-33 | 消息按任务分组 | `{task_id, task_name, message_indices[]}` |
| `TASK_CHUNKING_PROMPT_ZH` | L36-67 | 中文版 | 同上 |
| `SKILL_MEMORY_EXTRACTION_PROMPT` | L69-143 | 提取 Skill | JSON: name, description, procedure 等 |
| `SKILL_MEMORY_EXTRACTION_PROMPT_ZH` | L146-227 | 中文版 | 同上 |
| `SKILL_MEMORY_EXTRACTION_PROMPT_MD` | L230-300 | 增强提取（含 trigger） | JSON + trigger 字段 |
| `SKILL_MEMORY_EXTRACTION_PROMPT_MD_ZH` | L303-373 | 中文版 | 同上 |
| `TASK_QUERY_REWRITE_PROMPT` | L376-400 | 重写搜索查询 | 单字符串 |
| `TASK_QUERY_REWRITE_PROMPT_ZH` | L403-427 | 中文版 | 同上 |
| `SCRIPT_GENERATION_PROMPT` | L433-461 | 生成 Python 脚本 | `{filename: code}` |
| `TOOL_GENERATION_PROMPT` | L463-488 | 识别所需工具 | JSON 工具名数组 |
| `OTHERS_GENERATION_PROMPT` | L490-534 | 生成补充文档 | Markdown |

### 6.4 Python 端 Skill 数据模型

```python
class SkillMemory(TextualMemoryMetadata):
    memory_type = "SkillMemory"
    name: str                     # 技能名
    description: str              # 描述
    procedure: str                # 步骤流程
    experience: list[str]         # 经验教训
    preference: list[str]         # 用户偏好
    examples: list[str]           # 输出模板
    scripts: dict | None          # 代码片段 {filename: code}
    others: dict | None           # 补充文档
    url: str                      # Skill 仓库 URL
    # 继承字段：
    sources: list[SourceMessage]  # 来源
    embedding: list[float]        # 向量
    tags: list[str]               # 标签
    status: "activated" | "archived"
    created_at: str               # ISO 8601
    updated_at: str               # ISO 8601
```

---

## 七、Skill 检索与使用

### 7.1 检索引擎

文件：`src/recall/engine.ts`

混合搜索策略：
- **FTS 搜索**：对 skill name + description 做全文检索
- **向量搜索**：embed query → cosine similarity
- **融合排序**：RRF（Reciprocal Rank Fusion）+ MMR（Maximal Marginal Relevance）
- **Scope**：`"self"`（自己的）| `"public"`（公开的）| `"mix"`（两者）

### 7.2 自动召回（before_prompt_build）

```
用户消息 → 搜索相关 Skill
  → 格式化为提示注入：
    "## Relevant skills from past experience
     1. **skill-name** — description
        → call `skill_get(skillId="...")` for the full guide"
  → 返回 { prependContext: skillContext }
```

### 7.3 主动调用（registerTool）

Agent 可通过注册工具主动使用：

| 工具 | 作用 |
|------|------|
| `skill_get` | 获取完整 SKILL.md 内容 |
| `skill_search` | 搜索相关 Skill |
| `skill_install` | 安装 Skill 脚本到工作目录 |
| `network_skill_pull` | 从 Hub 拉取远程 Skill |

### 7.4 Hub 同步与发布

文件：`src/client/skill-sync.ts`

```typescript
// 发布到 Hub
publishSkillBundleToHub()  → 打包 SkillBundle → 上传

// 从 Hub 拉取
fetchHubSkillBundle()      → 下载
restoreSkillBundleFromHub() → 还原到本地

// SkillBundle 结构
{
  metadata: { id, name, description, version, qualityScore },
  bundle: { skill_md, scripts[], references[], evals[] }
}
```

---

## 八、完整数据流总览

```
                         ┌─────────────────────────────┐
                         │     OpenClaw Agent Runtime    │
                         └──────────┬──────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │                     │                     │
     before_prompt_build       Agent 运行中            agent_end
              │                     │                     │
    ┌─────────▼──────────┐  ┌──────▼───────┐    ┌───────▼────────┐
    │ 自动召回记忆/Skill  │  │ 主动调用工具  │    │  游标增量提取   │
    │ → 注入 prompt      │  │ memory_search │    │  sessionMsgCursor│
    └────────────────────┘  │ skill_get     │    └───────┬────────┘
                            │ skill_install │            │
                            └──────────────┘    ┌───────▼────────┐
                                                │ captureMessages │
                                                │   消息清洗       │
                                                └───────┬────────┘
                                                        │
                                                ┌───────▼────────┐
                                                │  IngestWorker   │
                                                │  摘要+向量+去重  │
                                                │  → SQLite 存储   │
                                                └───────┬────────┘
                                                        │
                                                ┌───────▼────────┐
                                                │ TaskProcessor   │
                                                │ 增量主题分割     │
                                                │ 时间/Session/LLM │
                                                └───────┬────────┘
                                                        │
                                                  主题切换时
                                                        │
                                                ┌───────▼────────┐
                                                │  finalizeTask   │
                                                │  生成任务摘要    │
                                                └───────┬────────┘
                                                        │
                                                ┌───────▼────────┐
                                                │  SkillEvolver   │
                                                │  Rule Filter    │
                                                │  LLM 评估       │
                                                │  搜索已有 Skill  │
                                                │  创建 / 升级     │
                                                │  验证 + 安装     │
                                                └────────────────┘
```
