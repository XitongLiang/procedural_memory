# Mem0 程序记忆（Procedural Memory）全流程分析

> 基于 https://github.com/mem0ai/mem0 源码（2026-04 版本）及 arXiv 论文 2504.19413
> Stars: ~48K | License: Apache 2.0

---

## 一、Mem0 是什么

Mem0 是一个**通用记忆中间件**（Memory as a Service），为 LLM/Agent 提供跨会话的持久化记忆存储和检索能力。定位是记忆**基础设施层**，不规定上层 Agent 的结构，只提供 `add / search / update / delete` API。

核心特点：**双阶段 LLM 提取 + ADD/UPDATE/DELETE 决策 + 三层混合存储（向量DB + 图DB + SQLite）**。

与其他系统的本质区别：

| 系统 | 定位 | 程序记忆产物 |
|------|------|------------|
| **Mem0** | 通用记忆中间件 | Agent 执行历史的结构化快照（"做了什么"） |
| **AutoSkill** | Skill 自动生成 | SKILL.md（可复用方法论，"怎么做更好"） |
| **MemOS** | 任务知识库 | SKILL.md + 脚本 + 评估集 |
| **Hermes** | 自进化 Agent | SKILL.md + 离线 Dreaming 精化版本 |

---

## 二、记忆类型体系

### 2.1 代码层定义

```python
class MemoryType(Enum):
    SEMANTIC   = "semantic_memory"
    EPISODIC   = "episodic_memory"
    PROCEDURAL = "procedural_memory"
```

实际只有 `procedural_memory` 有**独立的处理路径**。其他两种是文档概念分类，底层都走同一个向量存储管线。

### 2.2 按作用域的四层记忆

| 层 | 名称 | 生命周期 | 绑定字段 |
|----|------|---------|---------|
| L1 | Conversation memory | 单轮内 | 无持久化 |
| L2 | Session memory | 分钟到小时 | `run_id` |
| L3 | User memory | 永久 | `user_id` |
| L4 | Organizational memory | 全局共享 | `agent_id` 共享 |

**程序记忆绑定 `agent_id`（L4）**，其他记忆类型绑定 `user_id`。

---

## 三、程序记忆提取流程

### 3.1 触发条件

```python
# main.py: add() 方法路由逻辑
if agent_id is not None and memory_type == MemoryType.PROCEDURAL.value:
    results = self._create_procedural_memory(messages, metadata=..., prompt=prompt)
    return results
# 否则走普通语义记忆路径（双阶段提取 + ADD/UPDATE/DELETE 决策）
```

**必要条件**：同时传 `agent_id` + `memory_type="procedural_memory"`。

### 3.2 完整走线

```
调用: memory.add(messages, agent_id="agent-001", memory_type="procedural_memory")
    │
    ├─ 构造 Prompt
    │   ├─ system: PROCEDURAL_MEMORY_SYSTEM_PROMPT（或自定义 prompt）
    │   ├─ 原始对话消息（verbatim）
    │   └─ user: "Create procedural memory of the above conversation."
    │
    ├─ LLM 调用 → 生成结构化执行历史摘要
    │   （逐步骤记录，Action + Result + 错误 + 导航历史）
    │
    ├─ embedding_model.embed(summary)
    │
    ├─ vector_store.insert(
    │    vectors=[embedding],
    │    payloads=[{
    │        "data": summary,
    │        "memory_type": "procedural_memory",
    │        "agent_id": "agent-001",
    │        "hash": md5(summary),
    │        "created_at": ISO_UTC,
    │    }]
    │  )
    │
    └─ SQLite.add_history(memory_id, None, summary, "ADD")

返回: {"results": [{"id": "uuid", "memory": "...", "event": "ADD"}]}
```

**关键特征**：程序记忆**跳过 ADD/UPDATE/DELETE 决策逻辑**，每次都是直接 ADD 新记录，不做向量搜索去重。

---

## 四、核心 Prompt

### 4.1 PROCEDURAL_MEMORY_SYSTEM_PROMPT（程序记忆专用）

**定位**：生成 Agent 执行历史的逐步骤结构化快照，设计目标是**中断后任务恢复**，不是方法论抽象。

```
You are a memory summarization system that records and preserves the complete interaction 
history between a human and an AI agent. Your task is to produce a comprehensive summary 
of the agent's output history that contains every detail necessary for the agent to 
continue the task without ambiguity.
**Every output produced by the agent must be recorded verbatim as part of the summary.**

### Overall Structure:
- Overview (Global Metadata):
  - Task Objective: The overall goal the agent is working to accomplish.
  - Progress Status: Current completion percentage and milestones completed.

- Sequential Agent Actions (Numbered Steps):
  Each step must include:
  1. Agent Action: Precisely describe what the agent did (parameters, target elements).
  2. Action Result (Mandatory, Unmodified): Record all returned data exactly as received.
  3. Embedded Metadata:
     - Key Findings
     - Navigation History
     - Errors & Challenges
     - Current Context

Guidelines:
1. Preserve Every Output — do not paraphrase or summarize the output.
2. Chronological Order — number actions sequentially.
3. Detail and Precision — exact URLs, element indexes, error messages, JSON responses.
4. Output Only the Summary — no additional commentary.
```

### 4.2 USER_MEMORY_EXTRACTION_PROMPT（普通用户记忆，双阶段第一阶段）

**定位**：从对话中提取用户相关事实，仅基于 user 消息，不含 assistant 消息。

```
You are a Personal Information Organizer, specialized in accurately storing facts,
user memories, and preferences.

[IMPORTANT]: GENERATE FACTS SOLELY BASED ON THE USER'S MESSAGES.
[IMPORTANT]: YOU WILL BE PENALIZED IF YOU INCLUDE INFORMATION FROM ASSISTANT OR SYSTEM MESSAGES.

Types to extract:
1. Personal Preferences (food, products, activities, entertainment)
2. Important Personal Details (names, relationships, dates)
3. Plans and Intentions
4. Activity and Service Preferences
5. Health and Wellness Preferences
6. Professional Details
7. Miscellaneous Information

Return format: {"facts": ["fact1", "fact2", ...]}
Detect the language of the user input and record facts in the same language.
```

### 4.3 AGENT_MEMORY_EXTRACTION_PROMPT（Agent 自身特征记忆）

**触发条件**：`agent_id` 存在 AND 消息中含 `role=="assistant"`。仅基于 assistant 消息。

提取内容：Agent 的偏好、能力、个性特征、任务处理方式——用于跨会话保持 Agent 行为一致性。

### 4.4 DEFAULT_UPDATE_MEMORY_PROMPT（记忆更新决策，双阶段第二阶段）

**定位**：将新提取的事实与向量搜索召回的旧记忆对比，决定 ADD/UPDATE/DELETE/NONE。

```
You are a smart memory manager which controls the memory of a system.
You can perform four operations: (1) add, (2) update, (3) delete, (4) no change.

Compare newly retrieved facts with existing memory. For each new fact:
- ADD:    new info not present → generate new ID
- UPDATE: info present but different → keep same ID, update content
- DELETE: new info contradicts existing → remove
- NONE:   info already present, no change

Examples:
- "Loves cricket" + "Loves cricket with friends" → UPDATE（更丰富）
- "Name is John" + "Name is John" → NONE（完全相同）
- "Loves cheese pizza" + "Dislikes cheese pizza" → DELETE（矛盾）

Output JSON:
{"memory": [{"id": "0", "text": "...", "event": "ADD/UPDATE/DELETE/NONE", "old_memory": "..."}]}
```

**UUID 幻觉防护**：发给 LLM 时将真实 UUID 替换为整数索引（"0", "1", "2"...），解析响应时再还原，防止 LLM 编造 UUID。

### 4.5 EXTRACT_RELATIONS_PROMPT（图谱关系提取，可选）

从文本中提取实体-关系三元组，构建知识图谱。自引用（"I", "me"）统一映射为 `USER_ID` 节点。关系类型要求通用、一致（"professor" 而非 "became_professor"）。

---

## 五、普通记忆的双阶段处理流程

（程序记忆绕过此流程，直接 ADD）

```
阶段 1 — 事实提取
  输入：原始对话（格式化为 parse_messages）
  Prompt：USER_MEMORY_EXTRACTION_PROMPT 或 AGENT_MEMORY_EXTRACTION_PROMPT
  输出：{"facts": ["fact1", "fact2", ...]}

阶段 2 — 记忆合并决策
  输入：new_facts + 向量搜索 top-5 旧记忆
  Prompt：DEFAULT_UPDATE_MEMORY_PROMPT
  输出：{"memory": [{id, text, event, old_memory?}, ...]}

四种事件的存储行为：
  ADD    → vector_store.insert()   + SQLite(event="ADD",  old=None)
  UPDATE → vector_store.update()   + SQLite(event="UPDATE", old=prev)
  DELETE → vector_store.delete()   + SQLite(event="DELETE", is_deleted=1)
  NONE   → 仅更新 session_id（不改内容）
```

**去重窗口**：每条 new_fact 搜索 top-5 相似旧记忆，多条 new_fact 命中同一旧记忆时去重，确保每条旧记忆只被决策一次。

---

## 六、存储结构

### 6.1 三层存储分工

```
向量DB  →  主存储（内容 + 语义检索 + 元数据过滤）
图DB    →  关系存储（实体-关系三元组，可选启用）
SQLite  →  变更历史（审计、回溯，本地文件）
```

### 6.2 向量DB payload 结构

```json
{
  "data": "User likes to play cricket with friends",
  "hash": "md5_hex_string",
  "memory_type": "procedural_memory",
  "user_id": "alice",
  "agent_id": "my-agent",
  "run_id": "session-xyz",
  "created_at": "2026-04-13T09:00:00+00:00",
  "updated_at": "2026-04-13T09:00:00+00:00"
}
```

支持 20+ 向量 DB 后端：Qdrant（默认）、Pinecone、Chroma、FAISS、PGVector、Milvus 等。

### 6.3 图DB 结构（可选，Neo4j/Kuzu 等）

```
节点: (Entity {name, user_id, embedding, created, mentions})
关系: (source)-[RELATIONSHIP {created_at, updated_at, mentions, valid}]->(destination)

例：(alice)-[LIVES_IN]->(new_york)
```

关系用 BM25 检索（三元组格式：`source -- RELATIONSHIP -- destination`）。

### 6.4 SQLite 历史表

```sql
CREATE TABLE history (
    id          TEXT PRIMARY KEY,
    memory_id   TEXT,
    old_memory  TEXT,
    new_memory  TEXT,
    event       TEXT,      -- ADD/UPDATE/DELETE
    created_at  DATETIME,
    updated_at  DATETIME,
    is_deleted  INTEGER
);
```

---

## 七、检索注入机制

### 7.1 搜索流程

```
memory.search(query, user_id, top_k=10, threshold=0.3, rerank=True)
    │
    ├─ 并行执行：
    │   ├─ 向量检索：embed(query) → vector_store.search(top_k, filters)
    │   └─ 图检索（可选）：LLM提取实体 → Cypher查询 → BM25重排 top-5 三元组
    │
    ├─ [可选] reranker.rerank(query, memories, top_k)
    │
    └─ 返回 {"results": [...], "relations": [...]}
```

### 7.2 注入方式

**Mem0 不负责注入**，只提供搜索 API。调用方自行将结果拼入系统提示：

```python
memories = memory.search(query=user_message, user_id="alice")
context = "\n".join([m["memory"] for m in memories["results"]])
messages = [
    {"role": "system", "content": f"Relevant memories:\n{context}\n\n{system_prompt}"},
    {"role": "user",   "content": user_message},
]
```

注入格式、位置、截断策略全由上层应用决定。

---

## 八、与其他系统的核心差异

### 8.1 程序记忆本质差异

```
Mem0 Procedural Memory（执行历史快照）：
  "Step 1: Opened URL 'https://openai.com'
   Result: HTML content loaded.
   Step 2: Clicked on 'Blog' link.
   Result: Navigated to /blog/..."
  → "视频录像"：记录做了什么，支持中断后恢复

AutoSkill / MemOS / Hermes 的 Skill（方法论抽象）：
  "## Web Scraping Strategy
   When scraping paginated content:
   1. Identify pagination patterns (rel=next, page=N)
   2. Use BFS to traverse..."
  → "技巧总结"：记录怎么做更好，跨任务复用
```

| 维度 | Mem0 程序记忆 | AutoSkill/MemOS Skill |
|------|-------------|----------------------|
| 目标 | 中断后任务恢复 | 方法论跨任务复用 |
| 粒度 | 单次执行历史（逐步骤） | 通用化方法论 |
| 通用化 | 无（保留具体参数/URL/ID） | 有（去标识化，提取模式） |
| 更新策略 | 总是 ADD（不合并去重） | decide → add/merge/upgrade |
| 质量门控 | 无 | 有（LLM 打分 0-10） |
| 版本管理 | 无 | 有（版本号递增） |

### 8.2 架构差异对比

| 维度 | Mem0 | AutoSkill | MemOS TS | Hermes |
|------|------|-----------|----------|--------|
| 提取触发 | 显式调用 `add()` | agent_end hook 自动 | agent_end hook 自动 | LLM 自主调用工具 |
| 提取 Prompt | 通用（2 个） | 专用 4000 字 8 章节 | 专用多段（含任务摘要） | 系统提示引导 |
| 去重机制 | 向量 top-5 + LLM | BM25 + LLM 4 维评估 | hash + 语义 LLM | 无独立机制 |
| 离线演进 | 无 | SkillEvo（遗传算法） | 无 | GEPA + Dreaming |
| 存储媒介 | 向量DB行记录 | SKILL.md 磁盘文件 | SQLite + SKILL.md | SKILL.md 磁盘文件 |
| 检索注入 | 纯 API（手动注入） | before_prompt_build hook | before_prompt_build hook | 系统提示动态加载 |

### 8.3 设计哲学

```
Mem0：      Memory as Facts（记忆即事实）
            最小化颗粒，最大化灵活性，不规定上层结构

AutoSkill：  Memory as Skills（记忆即技能）
            强制通用化，严格质量门控，宁缺毋滥

MemOS：      Memory as Knowledge Assets（记忆即知识资产）
            企业级管理，版本/质量/复用全覆盖

Hermes：     Memory as Evolution Capital（记忆即进化资本）
            离线 Dreaming 从错误中学习，Skill 随时间自动精化
```

---

## 九、关键设计洞察

### 9.1 Mem0 程序记忆不是"技能"

Mem0 的程序记忆设计目标是**Agent 状态保存**（类似操作系统的进程 checkpoint），而不是可复用工作流的抽象。两者面向不同问题：
- Mem0：如何让 Agent 从中断点继续任务？
- AutoSkill/MemOS：如何让 Agent 复用成功经验？

### 9.2 Mem0 适合作为其他系统的基础设施层

Hermes 的 L4 记忆 Provider 支持将 Mem0 作为外部存储插件。用 Mem0 存事实记忆，用 SKILL.md 存程序记忆，两者互补。

### 9.3 无去重是有意为之

程序记忆每次直接 ADD 不做合并，是因为执行历史快照天然不重复——每次任务执行都是独特的。合并会破坏历史完整性，与"中断恢复"的设计目标矛盾。

### 9.4 论文关键参数

来自 arXiv 2504.19413：

- 提取窗口：近 10 条消息（`m=10`）
- 向量检索 k：top-10（`s=10`）
- 默认 LLM：GPT-4o-mini（提取 + 决策均用）
- LOCOMO benchmark：66.9% 准确率，P95 延迟 1.44s
- 比 OpenAI Memory 提升 26%，比 Full-context 延迟低 91%，token 成本节省 90%+

---

## 参考资料

- [mem0ai/mem0 GitHub](https://github.com/mem0ai/mem0)
- [Mem0 arXiv 论文 2504.19413](https://arxiv.org/abs/2504.19413)
- [Mem0 Memory Types 文档](https://docs.mem0.ai/core-concepts/memory-types)
- [GitHub Issue #3644 — Procedural Memory 官方解释](https://github.com/mem0ai/mem0/issues/3644)
