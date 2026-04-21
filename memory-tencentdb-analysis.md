# memory-tencentdb 代码分析报告

生成时间：2026-04-21  
仓库路径：`/home/lsh/.openclaw/extensions/memory-tencentdb`

## 1. 分析范围

本报告基于当前工作区代码进行静态分析，重点覆盖：

- 插件入口：`index.ts`
- 插件配置：`src/config.ts`、`openclaw.plugin.json`
- L0 对话捕获：`src/hooks/auto-capture.ts`、`src/conversation/l0-recorder.ts`
- L1 结构化记忆：`src/record/*`、`src/prompts/l1-*`
- L2 场景块：`src/scene/*`、`src/prompts/scene-extraction.ts`
- L3 用户画像：`src/persona/*`、`src/prompts/persona-generation.ts`
- 召回与工具：`src/hooks/auto-recall.ts`、`src/tools/*`
- 存储抽象与后端：`src/store/*`
- 管线调度：`src/utils/pipeline-manager.ts`、`src/utils/pipeline-factory.ts`
- CLI/Seed/迁移辅助：`src/cli/*`、`src/seed/*`、`bin/*`
- 清理、备份、checkpoint、manifest、profile sync、reporter 等支撑模块

未将 `node_modules/` 作为业务代码分析对象；`scripts/*/dist` 属于构建产物，仅在判断 CLI 包装和发布形态时参考。

## 2. 项目定位

该仓库是 OpenClaw 插件 `memory-tencentdb`，实现一个四层长期记忆系统：

- L0：原始对话消息记录
- L1：结构化记忆提取
- L2：场景块归纳
- L3：用户画像生成

插件支持两类存储后端：

- `sqlite`：本地 SQLite + `sqlite-vec` + FTS5
- `tcvdb`：腾讯云向量数据库，配合服务端 embedding、BM25 稀疏向量和 hybrid search

整体设计是“本地 JSONL 归档 + 检索数据库双写 + LLM 后台管线”的混合架构。

## 3. 插件入口与生命周期

`index.ts` 是插件主入口，默认导出 `register(api)`。

主要职责：

- 解析插件配置。
- 初始化数据目录：`conversations/`、`records/`、`scene_blocks/`、`.metadata/`、`.backup/`。
- 初始化 reporter 和 instance id。
- 注册两个工具：
  - `tdai_memory_search`
  - `tdai_conversation_search`
- 注册 `before_prompt_build` hook，用于自动召回并注入系统上下文。
- 注册 `agent_end` hook，用于自动捕获本轮对话并通知后台管线。
- 懒启动 L1/L2/L3 调度器，避免短生命周期 CLI 命令触发重任务。
- 在 `gateway_stop` 时按顺序释放 cleaner、scheduler、vector store、embedding service。
- 注册 `openclaw memory-tdai` CLI 命令空间。

关键生命周期：

```text
before_prompt_build
  -> 清理用户输入
  -> 召回 L1 相关记忆
  -> 读取 L2 scene navigation
  -> 读取 L3 persona
  -> appendSystemContext 注入上下文

agent_end
  -> 捕获本轮 L0 消息
  -> 写 JSONL
  -> 写向量/FTS 索引
  -> 通知 MemoryPipelineManager
  -> 后台触发 L1 -> L2 -> L3
```

## 4. 配置模型

配置由 `src/config.ts` 解析。所有字段都有默认值，支持零配置运行。

主要配置组：

- `capture`：控制 L0 捕获、排除 agent、L0/L1 TTL 清理。
- `extraction`：控制 L1 提取、去重、每 session 最大记忆数、模型。
- `persona`：控制 L2/L3 场景和画像生成阈值、备份数、模型。
- `pipeline`：控制 L1 idle、L2 延迟、L2 最小/最大间隔、session 活跃窗口。
- `recall`：控制自动召回开关、召回数量、阈值、策略、超时。
- `embedding`：控制远端 embedding provider、baseUrl、apiKey、model、dimensions。
- `storeBackend`：选择 `sqlite` 或 `tcvdb`。
- `tcvdb`：腾讯云向量数据库连接信息。
- `bm25`：BM25 稀疏向量配置。
- `report`：结构化指标日志开关。

重要细节：

- 默认 `storeBackend = sqlite`。
- 默认 embedding provider 为 `none`，因此纯零配置下向量搜索不会真正启用，主要依赖 FTS 能力。
- `provider = local` 已不对用户暴露，会被当作禁用并记录配置错误。
- `qclaw` provider 需要 `proxyUrl/baseUrl/apiKey/model/dimensions` 全部配置。
- L0/L1 清理默认禁用；若配置 1-2 天保留期，必须显式开启 `allowAggressiveCleanup`。

## 5. 数据目录结构

运行时数据目录为：

```text
<openclaw state dir>/memory-tdai/
```

典型结构：

```text
memory-tdai/
├── conversations/       # L0 原始对话 JSONL，每日分片
├── records/             # L1 结构化记忆 JSONL，每日分片
├── scene_blocks/        # L2 场景块 Markdown
├── persona.md           # L3 用户画像
├── vectors.db           # sqlite 后端数据库
├── .metadata/
│   ├── manifest.json
│   ├── checkpoint.json
│   └── scene_index.json
└── .backup/
```

JSONL 与数据库的关系：

- L0/L1 JSONL 是追加式归档和恢复来源。
- SQLite/TCVDB 是主要检索引擎。
- update/merge 会实时删除 VectorStore 中旧记录，但 JSONL 保持 append-only，由 cleaner 周期清理。

## 6. L0：对话捕获

相关文件：

- `src/hooks/auto-capture.ts`
- `src/conversation/l0-recorder.ts`
- `src/utils/checkpoint.ts`
- `src/utils/sanitize.ts`

L0 捕获触发于 `agent_end`。

核心流程：

1. 通过 checkpoint 的 `captureAtomically()` 对同一 session 做原子游标更新。
2. 从 hook 传入的完整消息历史中定位本轮新增消息。
3. 优先使用 `before_prompt_build` 缓存的 message count 做位置切片。
4. fallback 使用 timestamp cursor 过滤旧消息。
5. 用缓存的 `originalUserText` 替换可能被记忆注入污染过的用户消息。
6. 对内容做 sanitize。
7. assistant 消息会剥离 fenced code block，减少后续 embedding 和记忆提取噪声。
8. 写入 `conversations/YYYY-MM-DD.jsonl`。
9. 同步写入 VectorStore 的 L0 表/FTS/向量索引。

L0 的过滤策略是“宽松捕获”：

- 保留大多数真实用户输入。
- 只过滤空文本、框架噪声、slash 命令等结构性无效内容。
- 质量判断留给 L1。

## 7. L1：结构化记忆提取

相关文件：

- `src/record/l1-extractor.ts`
- `src/prompts/l1-extraction.ts`
- `src/record/l1-dedup.ts`
- `src/prompts/l1-dedup.ts`
- `src/record/l1-writer.ts`
- `src/record/l1-reader.ts`

L1 从 L0 中读取消息，调用 LLM 做“情境切分 + 记忆提取”。

L1 支持三类记忆：

- `persona`：稳定属性、偏好、技能、价值观、习惯。
- `episodic`：客观事件、计划、决定、达成结果。
- `instruction`：用户对 AI 的长期行为规则、格式偏好、语气控制。

其中 `instruction` 是当前代码中最接近“程序性记忆”的类型，后文单独分析。

L1 提取流程：

1. 从 L0 读取新消息，按 sessionId 分组。
2. 使用 `shouldExtractL1()` 做更严格的质量过滤。
3. 拆分为 background messages 和 new messages。
4. 调用 `CleanContextRunner`，在空上下文中运行 LLM。
5. LLM 输出 JSON 数组，每个元素包含：
   - `scene_name`
   - `message_ids`
   - `memories`
6. 解析、归一化类型。
7. 限制单 session 最大提取数量。
8. 执行批量去重/冲突检测。
9. 写入 L1 JSONL 和 VectorStore。

去重/冲突检测：

- 候选召回优先走向量搜索。
- 向量不可用时走 FTS5。
- 二者都不可用时跳过去重，直接存储。
- LLM 支持四类动作：
  - `store`
  - `skip`
  - `update`
  - `merge`
- 支持跨 type 合并、多目标合并、timestamp 并集。

## 8. L2：场景块归纳

相关文件：

- `src/scene/scene-extractor.ts`
- `src/prompts/scene-extraction.ts`
- `src/scene/scene-index.ts`
- `src/scene/scene-format.ts`
- `src/scene/scene-navigation.ts`

L2 将 L1 的碎片化记忆整合为 `scene_blocks/*.md` 场景叙事文件。

设计特点：

- LLM 以“Memory Consolidation Architect”角色运行。
- LLM 工具能力开启，但 workspace 被限制在 `scene_blocks/`。
- LLM 只能操作场景 Markdown 文件，看不到 `.metadata`、`persona.md` 等系统文件。
- 场景索引由工程系统维护，不交给 LLM 直接改。
- 删除文件采用软删除：LLM 写入 `[DELETED]`，工程侧清理。

场景文件模板包括：

- META：created、updated、summary、heat
- 用户基础信息
- 用户核心特征
- 用户偏好
- 隐性信号
- 核心叙事
- 演变轨迹
- 待确认/矛盾点

L2 的关键约束：

- 默认优先 UPDATE，最后才 CREATE。
- 达到或接近 `maxScenes` 时强制/建议 MERGE。
- CREATE 前要求读取相似已有场景验证无法融入。
- 禁止创建 BATCH、REPORT、SUMMARY、ARCHIVE 等汇总类文件。
- 每次批处理最多新增 1 个场景。

L2 还可通过文本信号触发 L3：

```text
[PERSONA_UPDATE_REQUEST]
reason: ...
[/PERSONA_UPDATE_REQUEST]
```

该信号会被 `parsePersonaUpdateSignal()` 解析并写入 checkpoint。

## 9. L3：用户画像

相关文件：

- `src/persona/persona-generator.ts`
- `src/persona/persona-trigger.ts`
- `src/prompts/persona-generation.ts`

L3 基于场景块生成或增量更新 `persona.md`。

触发条件：

- L2 明确请求 persona 更新。
- 冷启动后首次已有 scene blocks。
- persona 曾生成但正文丢失，需要恢复。
- 第一次 scene block 提取完成。
- `memories_since_last_persona >= triggerEveryN`。

Persona prompt 使用“四层深度扫描”：

- Layer 1：基础锚点，事实和当前状态。
- Layer 2：兴趣图谱，兴趣、消费、生活习惯。
- Layer 3：交互协议，沟通习惯、雷区、工作流偏好。
- Layer 4：认知内核，决策逻辑、矛盾点、驱动力。

对主 Agent 最有执行价值的是 Chapter 3：

- `Interaction & Cognitive Protocol`
- `How to Speak`
- `How to Think`

这部分会把 L1 的 `instruction` 和 L2 中提炼出的交互偏好整合为可执行的画像。

## 10. 自动召回与上下文注入

相关文件：

- `src/hooks/auto-recall.ts`
- `src/tools/memory-search.ts`
- `src/tools/conversation-search.ts`

召回发生在 `before_prompt_build`。

注入内容顺序：

1. `<user-persona>`：L3 persona 正文。
2. `<scene-navigation>`：L2 场景导航和 read_file 路径提示。
3. `<relevant-memories>`：L1 当前相关记忆。
4. `<memory-tools-guide>`：工具调用指南。

召回策略：

- `keyword`：FTS5 BM25。
- `embedding`：向量相似度。
- `hybrid`：关键词 + 向量，RRF 融合。

召回有整体超时，默认 5000ms；超时后跳过注入，避免阻塞用户对话。

工具能力：

- `tdai_memory_search`
  - 搜索 L1 结构化长期记忆。
  - 支持 `type` 和 `scene` 过滤。
- `tdai_conversation_search`
  - 搜索 L0 原始对话消息。
  - 支持 `session_key` 过滤。

工具限制目前主要写在 tool description 和注入指南中：

- 两个工具每轮合计最多 3 次。
- 代码中有 TODO，尚未通过 hook 做硬性限制。

## 11. 存储抽象与后端

相关文件：

- `src/store/types.ts`
- `src/store/factory.ts`
- `src/store/sqlite.ts`
- `src/store/tcvdb.ts`
- `src/store/tcvdb-client.ts`
- `src/store/embedding.ts`
- `src/store/bm25-local.ts`
- `src/store/search-utils.ts`

`IMemoryStore` 是核心抽象，统一定义：

- L1 写入、删除、查询、向量搜索、FTS 搜索、hybrid 搜索。
- L0 写入、删除、查询、向量搜索、FTS 搜索。
- L2/L3 profile pull/sync/delete。
- reindex。
- capability flags。

SQLite 后端：

- 本地 `vectors.db`。
- 支持 FTS5。
- 支持 `sqlite-vec`。
- 支持 L0 deferred embedding：先写 metadata + FTS，后台补向量。

TCVDB 后端：

- 要求 `url`、`apiKey`、`database`。
- 使用 `NoopEmbeddingService` 表示服务端 embedding。
- 可配合 BM25 sparse vector。
- 支持 L2/L3 profile sync。

Embedding：

- 用户配置主要面向 OpenAI-compatible 远端服务。
- 本地 embedding 实现仍在代码中，但配置层已经不暴露给普通用户。
- provider 为 `none` 时 dimensions 为 0，避免提前创建错误维度的 vec0 表。

## 12. 管线调度

相关文件：

- `src/utils/pipeline-manager.ts`
- `src/utils/pipeline-factory.ts`
- `src/utils/serial-queue.ts`
- `src/utils/managed-timer.ts`
- `src/utils/checkpoint.ts`

`MemoryPipelineManager` 管理 L1/L2/L3 后台调度。

关键机制：

- 每 session 独立状态。
- L1 有 warm-up 阈值：1 -> 2 -> 4 -> ... -> everyN。
- L1 有 idle timeout：用户停止说话后触发。
- L1 失败后保留 buffer 并最多重试 5 次。
- L2 在 L1 成功后延迟触发，且有最小间隔和最大轮询间隔。
- L3 是全局队列，避免并发重复生成。
- checkpoint 记录 pipeline state、L1 cursor、L2 cursor、persona trigger 状态等。
- gateway_stop 时尝试在超时内 flush pending work。

## 13. 程序性记忆分析

这里的“程序性记忆”可理解为：用户长期希望 AI 遵守的行为规则、回答方式、工作流偏好、格式规范、交互协议。

当前代码没有单独命名为 `procedural memory` 的层或类型。它主要由以下路径承载：

### 13.1 L1 的 `instruction` 类型

`src/prompts/l1-extraction.ts` 明确定义：

- `instruction` 是“用户对 AI 提出的长期行为规则、格式偏好、语气控制”。
- 触发词包括“以后都”“从现在开始”“记住”“必须”等。
- 优先级可以为 `-1`，表示极其严格的全局死命令。

这就是最直接的程序性记忆入口。

### 13.2 L1 记忆可被召回并注入

`instruction` 类型记录进入 L1 后：

- 写入 JSONL。
- 写入 VectorStore。
- 可被 auto-recall 搜索到。
- 可通过 `tdai_memory_search` 按 `type=instruction` 检索。
- 召回后进入 `<relevant-memories>`。

因此它能影响后续对话行为。

### 13.3 L3 Persona 的交互协议

L3 prompt 明确要求生成：

- `Chapter 3: Interaction & Cognitive Protocol`
- `3.1 沟通策略`
- `3.2 决策逻辑`

这会把多条 L1 instruction、偏好和场景信号综合为更高层的“如何和用户交互”的规则。

### 13.4 L2 场景也会保留偏好和演变

L2 场景模板包含：

- 用户偏好
- 隐性信号
- 演变轨迹
- 待确认/矛盾点

这可以承载“用户工作流长期如何变化”“偏好从 A 变成 B”等程序性信息，但它不是直接的执行规则，更多是中间叙事证据。

### 13.5 当前程序性记忆的不足

当前实现能保存和召回程序性记忆，但存在几个边界：

- 没有独立 `procedural` 类型，长期规则被折叠进 `instruction`。
- `instruction` 的生效强度主要依赖召回排序和 persona 汇总，没有单独强制层。
- `priority = -1` 被 prompt 定义，但召回和注入阶段没有看到专门把 `-1` 指令置顶或强制注入的策略。
- 工具调用限制目前是软约束，不是程序级硬约束。
- 冲突解决支持 update/merge，但没有专门区分“旧规则废止”和“新规则覆盖”的生命周期模型。

建议：

- 若产品目标强调程序性记忆，可考虑新增 `procedural` 或把 `instruction` 明确重命名为 `procedural_instruction`。
- 对 `priority = -1` 或高优先级 instruction 建立常驻注入通道，而不是完全依赖语义召回。
- 在 L3 中单独维护 “Hard Rules / Soft Preferences / Deprecated Rules”。
- 为程序性记忆增加 source、scope、expiresAt、supersedes 等字段。
- 对“以后都”“不要再”“从现在开始”等规则建立覆盖关系。

## 14. Skill 相关内容分析

当前代码没有实现独立的 Skill 管理系统，也没有把 skill 作为单独的记忆类型。skill 相关内容主要体现在“过滤 skill wrapper 噪声”和“避免污染记忆”。

### 14.1 明确的 skill wrapper 过滤

`src/utils/sanitize.ts` 中有规则：

```text
Remove injected skill-selection wrappers, e.g. ¥¥[... ]¥¥
```

对应逻辑会移除：

```text
¥¥[ ... ]¥¥
```

这说明系统已观察到 OpenClaw 或上游框架会向消息中注入 skill-selection wrapper。如果不清理，这些 wrapper 可能被：

- L0 记录进原始对话。
- L1 提取为错误记忆。
- 后续召回再注入，形成污染循环。

CHANGELOG 中也记录过：

- “过滤 skill wrapper 噪声标记（`¥¥[...]¥¥`）”

### 14.2 Skill 不作为 L1 记忆类型

L1 只允许：

- `persona`
- `episodic`
- `instruction`

没有 `skill` 类型。

如果用户说“以后使用某个 skill 处理某类任务”，这类内容理论上会进入 `instruction`，但不会被标记为 skill-specific memory。

### 14.3 Skill 与工具指南的区别

auto-recall 注入的 `<memory-tools-guide>` 是对主 Agent 的工具调用指南，不是 OpenClaw skill 系统。

它告诉 Agent：

- 注入记忆不够时调用 `tdai_memory_search`。
- 需要原文时调用 `tdai_conversation_search`。
- 已定位场景时可用 `read_file` 读取 scene block。

这属于记忆系统自带工具使用协议，不是 skill 注册、skill 选择或 skill 执行。

### 14.4 当前 Skill 相关风险

- 只过滤了 `¥¥[...]¥¥` 形式，若上游 skill wrapper 格式变化，可能再次污染记忆。
- 没有针对 skill 调用结果、skill 选择理由、skill 元数据做结构化过滤。
- 如果用户明确表达“某任务应该使用某 skill”，系统会保存为普通 instruction，召回时未必稳定命中。

建议：

- 把 skill wrapper 格式作为可配置或集中维护的 sanitizer pattern。
- 如果业务需要“技能偏好记忆”，可新增字段：
  - `skill_name`
  - `task_pattern`
  - `selection_rule`
  - `confidence`
  - `scope`
- 对 skill 偏好建立专门召回策略：当用户任务与 `task_pattern` 匹配时优先注入。
- 避免记录框架自动选择 skill 的内部痕迹，只记录用户明确表达的 skill 偏好或长期任务路由规则。

## 15. CLI 与 Seed

CLI 命名空间：

```bash
openclaw memory-tdai
```

当前文档重点是 `seed` 子命令，用于导入历史对话并运行 L0 -> L1 -> L2 -> L3 管线。

Seed 设计：

- 支持对象包装格式和顶层数组格式。
- 支持 `--config` 对插件配置做两级深度合并。
- 支持输出到独立目录。
- 使用共享 pipeline factory，尽量复用线上逻辑。
- 当前 `seed-runtime.ts` 明确有 FIXME：
  - 只等待 L1 idle。
  - L2/L3 可能还在运行时 pipeline destroy。
  - seed 输出可能不包含最新 L2/L3 artifacts。

这是一处需要注意的行为边界。

## 16. 清理、备份、同步和指标

清理：

- `LocalMemoryCleaner` 负责 L0/L1 本地文件保留期清理。
- 清理前提是配置了有效 retentionDays。
- cleaner 是进程级 singleton，避免并发清理竞态。

备份：

- L2 scene blocks 和 L3 persona 写入前会滚动备份到 `.backup/`。

Manifest：

- `.metadata/manifest.json` 记录 store 绑定信息和 seed 信息。
- 初始化 store 时会检测当前配置与 manifest 的 store binding 是否漂移。

Profile Sync：

- 对支持 profile sync 的 store，L2/L3 可在本地文件和远端向量库之间双向同步。
- 使用 baseline version 做乐观锁语义。

Reporter：

- 支持结构化指标日志：
  - `agent_turn`
  - `tool_call`
  - `l1_extraction`
  - `l2_extraction`
  - `l3_persona_generation`
  - `error_degradation`

## 17. 安全与污染防护

已有防护：

- `sanitizeText()` 删除记忆注入标签，避免 `<relevant-memories>`、`<user-persona>` 等反馈进入新记忆。
- 删除 inbound metadata JSON blocks。
- 删除 reply directive。
- 删除 skill wrapper。
- 删除媒体附件 marker、图片发送说明、base64 image data。
- L0 使用原始 prompt cache 替换被 prependContext 污染的用户消息。
- L1 使用 clean context runner，避免主对话上下文污染。
- L2 workspace 限定到 `scene_blocks/`。
- L3 限制 prompt 要求只操作 `persona.md`。

需要注意：

- `looksLikePromptInjection()` 存在，但 `shouldExtractL1()` 中相关调用被注释掉了。因此 L1 目前不会实际拒绝疑似 prompt injection 文本。
- `escapeXmlTags()` 用于 persona 后处理，可降低 XML-like 标签注入风险。
- L2/L3 仍依赖 LLM 遵守工具约束，工程侧通过 workspace 限制和后处理兜底。

## 18. 关键风险与改进建议

### 高优先级

1. 程序性记忆没有硬性生效层  
   高优先级 `instruction` 可能因召回策略、阈值、query 不匹配而不注入。建议对 `priority = -1` 或高优先级 instruction 做常驻注入或独立 hard rules 文件。

2. 工具调用次数限制是软约束  
   `tdai_memory_search` 和 `tdai_conversation_search` 的每轮 3 次限制目前写在 description 和 prompt 中，TODO 显示尚未实现硬限制。建议在 tool execution 或 before_tool_call 层实现计数。

3. Prompt injection 检测未启用  
   `looksLikePromptInjection()` 已实现，但 L1 过滤中调用被注释。建议明确产品取舍：若担心误杀，可改为打标、降权或仅禁止进入 `instruction` 类型。

4. Seed 不等待 L2/L3 完整 idle  
   如果用户期望 seed 后立即拿到完整 scene/persona，需要补充 L2/L3 idle 等待能力或在 CLI 输出中明确提示。

### 中优先级

5. Skill 规则没有结构化建模  
   如果 OpenClaw 后续更重度使用 skill，建议把“用户希望某类任务使用某 skill”的规则从普通 instruction 中拆出来。

6. `instruction` 缺少 scope 和生命周期  
   长期规则可能有全局、项目、session、任务类型等不同范围。建议增加 scope、expiresAt、supersedes。

7. L2 场景数量约束依赖 prompt 和清理  
   当前强约束主要靠 LLM 遵守，工程侧可增加最终数量检查和自动回滚/报警。

8. JSONL append-only 与 VectorStore 实时状态可能短暂不一致  
   update/merge 删除只实时作用于 VectorStore，JSONL 旧记录依赖 cleaner。对导出、迁移、恢复路径要明确“以 VectorStore 为准”还是“以 JSONL 为准”。

### 低优先级

9. 本地 embedding 代码仍存在但配置层禁用  
   可考虑清理文档和注释，避免 README、代码注释和实际配置行为不一致。

10. `searchMemoriesWithDetails()` 看起来未被主流程使用  
   可以检查是否是遗留函数，减少维护面。

## 19. 总结

当前代码实现的是一个比较完整的 OpenClaw 长期记忆插件，核心优势是：

- L0/L1/L2/L3 分层清晰。
- 召回、记录、提取、场景归纳、画像生成形成闭环。
- SQLite 与 TCVDB 通过统一 store interface 解耦。
- 有较多工程兜底：checkpoint、manifest、backup、cleaner、timeout、degraded fallback、profile sync。
- 已注意到记忆污染问题，并对注入标签、metadata、skill wrapper 等做了 sanitizer。

关于用户特别关注的两个点：

- 程序性记忆：当前由 L1 的 `instruction` 类型和 L3 的 Interaction & Cognitive Protocol 承载，但没有单独的程序性记忆层，也没有硬性常驻注入机制。
- Skill 相关内容：当前不是功能主体，主要是过滤 skill-selection wrapper 噪声；没有 skill 记忆类型、skill 偏好结构化存储或 skill 路由规则。

如果下一步要强化这两个方向，建议优先实现：

1. `instruction` 高优先级常驻注入。
2. 程序性规则的 scope/supersedes/expiresAt。
3. skill 偏好结构化字段和任务匹配召回。
4. 工具调用次数硬限制。
5. L1 prompt injection 过滤重新启用或降权处理。
