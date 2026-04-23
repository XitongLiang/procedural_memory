# V6 Learning — 整体架构图

> 基于 `design-v6-learning.md` §1 整体架构生成的可视化架构图

---

## 一、数据流总览（从下到上）

```mermaid
flowchart TB
    subgraph AgentWrite["🤖 Agent（写端）"]
        A1["产生对话 / 工具调用 / 执行结果"]
    end

    subgraph Step1["Step 1 — 预过滤（零 LLM）"]
        S1_1["TS 侧 autoCapture<br/>· 跳过 heartbeat/startup<br/>· role 分流<br/>· user 消息 9 条正则 sanitize<br/>· 成功 toolResult 丢弃 / 错误截 300 字"]
        S1_2["C 端补齐<br/>· JSON blob 反序列化<br/>· 敏感数据脱敏<br/>· 语言标注"]
    end

    subgraph StorageRaw["原始对话存储"]
        MC[("mem_conversation<br/>append-only + 水位")]
    end

    subgraph Step2_4["Step 2/3/4 — 窗口 + 主题 + 拼接"]
        S2["滑动窗口<br/>n=8, m=2, step=6"]
        S3["窗口主题抽取<br/>· topic_keywords（3-8 词）<br/>· topic_summary（15-40 字）<br/>· 查 procedural_topic_cache 复用"]
        S4["主题拼接状态机<br/>· Jaccard > 0.6 → MERGE<br/>· Jaccard ≤ 0.6 → 小模型 Judge<br/>· 3 窗口 / 2h gap → 强制 flush"]
    end

    subgraph Step5["Step 5 — Chunk Flush"]
        direction TB
        S5_A["(A) ExtractTaskTrace<br/>大模型 → Markdown Trace<br/>· User Query<br/>· Task: name：description<br/>· sub-task 1/2/3...<br/>· Step 1.1 / 1.2 / ..."]
        S5_B["(B) LearnFeedback + Preference<br/>大模型 per Trace<br/>→ 查重 → procedural_candidates<br/>· hybrid = 0.80·cos + 0.20·kw<br/>· ≥0.85 merge / <0.50 add / 灰区 Judge"]
        S5_C["(C) TSE 提取<br/>· §7.1 Search top-5<br/>· §7.2 Judge 选 candidate<br/>· §7.3 Enrich / §7.4 Learn<br/>→ 在线只写候选池"]
    end

    subgraph FactLayer["事实层"]
        PC[("procedural_concept<br/>· task / sub-task canonical name<br/>· occurrence 计数<br/>· description 字典")]
    end

    subgraph EventLayer["事件层"]
        PTT[("procedural_task_trace<br/>· 每次 chunk 一条 Markdown<br/>· + task_trace_vec")]
    end

    subgraph CandidatePool["候选池"]
        PCan[("procedural_candidates<br/>kind ∈ {fb, tp, tse}<br/>· anchor / payload / search_key<br/>· mention_count / is_hard<br/>· + procedural_candidates_vec")]
    end

    subgraph OfflineWorker["§13 离线升格 Worker（每天 1 次）"]
        OW1["Step A: 挑候选<br/>delta = mention - last_promoted ≥ 3"]
        OW2["Step B: 合理性检查<br/>小模型 + 最近 5 条 source_trace 证据"]
        OW3["通过 → upsert active"]
    end

    subgraph KnowledgeLayer["知识层（active 快照）"]
        MR[("mem_record<br/>· category=EXPERIENCE<br/>· category=FEEDBACK<br/>· category=TOOL_PREFERENCE")]
        META["扩展表<br/>procedural_tse_meta (task TEXT)<br/>procedural_fb_meta (sub_task TEXT)<br/>procedural_tp_meta (sub_task TEXT)"]
        VEC["mem_record_raw_vec_{10,11,12}"]
    end

    subgraph AgentRead["🤖 Agent（读端）"]
        R1["memory_search<br/>召回 active 记忆"]
    end

    AgentWrite --> S1_1
    S1_1 --> S1_2
    S1_2 --> MC
    MC -->|"120s 定时器<br/>拉水位之后行"| S2
    S2 --> S3
    S3 --> S4
    S4 -->|"flush"| S5_A
    S5_A -->|"UPSERT"| PC
    S5_A -->|"INSERT"| PTT
    S5_A -->|"消费 Trace"| S5_B
    S5_A -->|"消费 Trace"| S5_C
    PTT -->|"source_trace 证据"| S5_B
    PTT -->|"source_trace 证据"| S5_C
    S5_B -->|"写 fb/tp"| PCan
    S5_C -->|"写 tse"| PCan
    PCan -->|"scan"| OW1
    OW1 --> OW2
    OW2 --> OW3
    OW3 -->|"upsert active"| MR
    MR --> META
    MR --> VEC
    MR -->|"reads (仅 active 可见)"| AgentRead

    style AgentWrite fill:#e1f5e1
    style AgentRead fill:#e1f5e1
    style Step5 fill:#fff3e0
    style OfflineWorker fill:#fce4ec
    style KnowledgeLayer fill:#e3f2fd
    style CandidatePool fill:#f3e5f5
    style EventLayer fill:#fff8e1
    style FactLayer fill:#e8f5e9
```

---

## 二、Chunk Flush 内部细节（Step 5 展开）

```mermaid
flowchart LR
    Chunk["Chunk Messages<br/>（已过滤已脱敏）"] -->|"大模型"| ETT["ExtractTaskTrace"]
    
    ETT -->|"解析 Task 行"| PC[("procedural_concept<br/>UPSERT task + sub_tasks")]
    ETT -->|"INSERT"| PTT[("procedural_task_trace<br/>Markdown + vec")]
    
    PTT -->|"读取 Trace.content"| LFP["LearnFeedback+Preference<br/>大模型 per Trace"]
    PTT -->|"读取 Trace.content"| TSE["TSE 四步<br/>Search → Judge → Enrich/Learn"]
    
    LFP -->|"查重 hybrid score"| PCan[("procedural_candidates<br/>fb/tp 分区")]
    TSE -->|"查重 hybrid score"| PCan
    
    PCan -->|"is_hard=true 立即升格<br/>其余离线处理"| Active[("mem_record<br/>+ meta 扩展表")]

    style Chunk fill:#e1f5e1
    style ETT fill:#fff3e0
    style LFP fill:#fff3e0
    style TSE fill:#fff3e0
    style Active fill:#e3f2fd
```

---

## 三、三层数据模型

```mermaid
flowchart TB
    subgraph Layer1["事件层（每次执行一条，允许重复）"]
        L1["procedural_task_trace<br/><br/>· 完整结构化 Markdown 轨迹<br/>· task / sub-task / step 三级<br/>· Fb/Tp/TSE 提取的统一证据源<br/>· 离线 dreaming 原材料"]
    end

    subgraph Layer2["事实层（去重 + 计数）"]
        L2["procedural_concept<br/><br/>· task / sub-task canonical name<br/>· occurrence 计数<br/>· description 供查询拼 search_key<br/>· 不被业务表 FK 引用"]
    end

    subgraph Layer3["知识层（可复用知识）"]
        direction TB
        L3_TSE["task_specific_experience<br/>（任务级经验，锚 task TEXT）<br/>→ 情境化决策参考、技巧组合、风险规避"]
        L3_FB["feedback<br/>（子事务级硬约束，锚 sub_task TEXT）<br/>→ 跨任务复用，优先级最高"]
        L3_TP["tool_preference<br/>（子事务级偏好，锚 sub_task TEXT）<br/>→ 跨任务复用，减少决策开销"]
    end

    Layer1 -->|"提炼"| Layer3
    Layer2 -.->|"canonical name<br/>字面值索引"| Layer3

    style Layer1 fill:#fff8e1
    style Layer2 fill:#e8f5e9
    style Layer3 fill:#e3f2fd
```

---

## 四、核心参数速查

| 参数 | 值 | 说明 |
|------|----|------|
| 窗口大小 `n` | 8 轮 | 1 轮 = user + assistant 一对 |
| 重叠 `m` | 2 轮 | 相邻窗口重叠 |
| 步长 `step` | 6 轮 | n - m |
| 定时器周期 | 120s | InstanceWorker 扫描间隔 |
| Jaccard 阈值 | 0.6 | 主题边界快速门控 |
| chunk 上限 | 3 窗口 / 2h | 强制 flush 条件 |
| hybrid 查重 | 0.80·cos + 0.20·kw | 三档阈值：≥0.85 / 0.50-0.85 / <0.50 |
| 升格 delta | 3 | mention - last_promoted ≥ 3 |

---

## 五、读写边界总结

| 操作 | 读 | 写 |
|---|---|---|
| 在线 chunk flush | `procedural_candidates` + `procedural_concept` | `procedural_task_trace`、`procedural_concept`、`procedural_candidates` |
| 离线升格 worker | `procedural_candidates` | `mem_record` + 三个 meta 扩展表、回写 `procedural_candidates.last_promoted_*` |
| Agent 召回 | `mem_record` + 三个 meta 扩展表 | — |
