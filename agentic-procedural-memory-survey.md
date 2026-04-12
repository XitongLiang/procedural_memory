# AI Agent 程序性记忆（Procedural Memory）全景调研

> 调研时间：2026-04-12 | 覆盖范围：学术论文 (2023-2026) + 开源框架 + 商业产品

---

## 一、概念与分类体系

### 1.1 什么是程序性记忆

程序性记忆源自认知科学（Tulving 1972, Squire 1987），指关于 **"如何做事"** 的知识——工作流、操作技能、执行步骤等，区别于：

| 记忆类型 | 回答的问题 | Agent 中的例子 |
|---------|-----------|--------------|
| **语义记忆** | "是什么" | 用户偏好 DD/MM/YYYY 格式 |
| **情景记忆** | "发生了什么" | 昨天部署时遇到了 OOM |
| **程序性记忆** | "怎么做" | 部署 Docker 应用的 5 步流程 |
| **工作记忆** | "正在处理什么" | 当前 context window 内容 |

### 1.2 主流分类框架

**CoALA (2023)** — 最具影响力的 Agent 认知架构框架（Sumers, Yao, Narasimhan, Griffiths）：
- 将记忆分为 工作/情景/语义/程序性 四类
- 关键洞察：程序性记忆有两种载体——(1) LLM 参数权重中的隐式知识 (2) Agent 代码/指令中的显式程序
- 不同于情景和语义记忆可以从空开始，程序性记忆**必须由设计者初始化**

**"Memory in the Age of AI Agents" (2025.12)** — 47 位作者，目前最全面的综述：
- 三维分类：形式（token/参数/隐空间）× 功能（事实/经验/工作）× 动态（形成→演化→检索）
- 将程序性知识归入"经验记忆"的子类

**"Anatomy of Agentic Memory" (2026.02)** — 基于架构视角的分析，指出当前系统的核心痛点：benchmark 饱和、指标有效性、骨干模型依赖、记忆维护开销

### 1.3 记忆生命周期

所有主流框架都认同的生命周期模型：

```
Formation (形成/提取)  →  Evolution (演化/巩固/遗忘)  →  Retrieval (检索/应用)
```

---

## 二、程序性记忆的六种表示形式

### 2.1 可执行代码

| 代表 | 格式 | 特点 |
|------|------|------|
| **Voyager** (2023, NVIDIA) | JavaScript 函数 | 技能可组合，能力指数增长，3.3x 物品获取提升 |
| **CodeMem** (2025) | 通过 MCP 管理的代码 | 将代码确立为程序性记忆的最优表示 |
| **CREATOR** (2023) | Python 工具代码 | LLM 自主创建工具而非仅使用已有工具 |

**优点**：精确可执行、可复现、支持条件/循环/组合
**缺点**：需要编程环境、对非代码任务不适用、调试困难

### 2.2 自然语言规则/洞察

| 代表 | 格式 | 特点 |
|------|------|------|
| **Reflexion** (2023, NeurIPS) | "上次失败因为...下次应该..." | 言语强化学习，HumanEval 91% |
| **ExpeL** (2023, AAAI) | 跨任务高层规则 | 无需参数更新，兼容闭源模型 |
| **Generative Agents** (2023, Stanford) | 记忆流 + 反思 | 三重评分：近因+重要性+相关性 |

**优点**：灵活通用、易于人类理解和编辑
**缺点**：执行精度低、依赖 LLM 解释能力

### 2.3 结构化工作流/轨迹

| 代表 | 格式 | 特点 |
|------|------|------|
| **Mem^p** (2025, 浙大) | 轨迹 + 脚本双层编码 | 支持 Add/Remove/Update，可跨模型迁移 |
| **AWM** (2024, ICML 2025) | 从轨迹归纳的工作流模板 | WebArena 提升 51.1% |
| **LEGOMem** (2025, MSR) | 模块化任务/子任务记忆 | 全任务+子任务两种粒度 |

**优点**：保留完整上下文、结构清晰、可复用
**缺点**：冗长、构建成本高

### 2.4 层次化技能库

| 代表 | 格式 | 特点 |
|------|------|------|
| **SkillX** (2026.04, 浙大) | 三级：规划/功能/原子技能 | 统一结构 Name+Doc+Content |
| **GITM** (2023) | 三层 LLM 分解体系 | 效率比 OpenAI VPT 提升 10,000x |

**优点**：兼顾战略和操作层面、可迁移
**缺点**：构建成本高

### 2.5 可训练令牌

| 代表 | 格式 | 特点 |
|------|------|------|
| **TokMem** (2025.10) | Memory Token | 常数级上下文开销、模块化复用 |

**优点**：极低开销、模块化
**缺点**：需要训练、缺乏可解释性

### 2.6 Prompt 模板 / 指令更新

| 代表 | 格式 | 特点 |
|------|------|------|
| **LangMem** (LangChain) | Agent Prompt 更新 | 程序性记忆 = Prompt 指令 |
| **ReMe** (2025.12) | 结构化经验卡片 | 多面蒸馏 + 效用评分驱动精炼 |

**优点**：即插即用、自动化程度高
**缺点**：受 context window 限制

---

## 三、学术界核心论文精读

### 3.1 Voyager — 代码式技能库的开创者

> Wang et al., 2023, NVIDIA | [arXiv:2305.16291](https://arxiv.org/abs/2305.16291)

```
新任务 → GPT-3.5 生成任务建议
       → 以建议+环境反馈为 query
       → 向量检索 Top-5 相关技能（JS 代码）
       → 组合已有技能 + 生成新代码
       → 执行 → 环境反馈/错误 → 迭代修正
       → 自验证通过 → 存入技能库
```

- **存储**：向量数据库，Key = 程序描述 embedding，Value = JS 代码
- **性能**：3.3x 物品、2.3x 距离、15.3x 更快解锁科技树
- **局限**：只增不减，无法废弃过时技能

### 3.2 Mem^p — 程序性记忆的系统性探索

> 浙江大学 zjunlp, 2025 | [arXiv:2508.06433](https://arxiv.org/abs/2508.06433)

首个系统探索程序性记忆 Build/Retrieval/Update 三维度的工作：

- **双层编码**：Trajectory（逐步指令）+ Script（LLM 总结的抽象程序）
- **更新策略**：Addition（新增）+ Validation Filtering（验证过滤）+ Reflection（反思修正）+ Dynamic Discarding（动态丢弃）
- **关键发现**：强模型构建的程序性记忆可迁移到弱模型，仍带来显著提升

### 3.3 ReMe — 动态程序性记忆框架

> 2025.12 | [arXiv:2512.10696](https://arxiv.org/abs/2512.10696)

解决"被动累积"范式问题：

- **多面蒸馏**：成功模式 + 失败分析 + 成功 vs 失败对比 → 结构化经验 E = <场景, 内容, 关键词, 置信度, 工具列表>
- **上下文自适应复用**：Embedding → Top-K → LLM 重排 → 智能重写为任务特定指导
- **效用驱动精炼**：追踪检索频率和效用，自动裁剪低效记忆
- **关键结果**：Qwen3-8B + ReMe 超越 Qwen3-14B，记忆质量可替代模型规模

### 3.4 SkillX — 层次化技能知识库自动构建

> 浙江大学 zjunlp, 2026.04 | [arXiv:2604.04804](https://arxiv.org/abs/2604.04804)

三级技能层次：

| 级别 | 名称 | 内容 | 粒度 |
|-----|------|------|------|
| 高层 | Planning Skills | 压缩成功轨迹为有序步骤 | 战略级 |
| 中层 | Functional Skills | 可复用多步工具子程序 | 功能级 |
| 底层 | Atomic Skills | 单一工具使用约束和模式 | 操作级 |

三阶段管线：多级提取 → 迭代精炼（聚类合并+验证过滤）→ 探索性扩展（识别薄弱工具，合成新任务）

### 3.5 MACLA — 层次化程序性记忆学习

> AAMAS 2026 | [arXiv:2512.18950](https://arxiv.org/abs/2512.18950)

- 保持 LLM 冻结，所有适配在外部层次化程序性记忆中完成
- 贝叶斯后验跟踪可靠性 + 期望效用评分选择动作 + 对比成功/失败精化
- 56 秒构建记忆，比 SOTA 参数训练快 2800x，2851 条轨迹压缩为 187 个程序

### 3.6 其他重要工作

| 论文 | 年份 | 核心贡献 |
|------|------|---------|
| **AWM** (Agent Workflow Memory) | 2024/ICML 2025 | 从轨迹归纳可复用工作流，WebArena +51.1% |
| **TokMem** | 2025.10 | 程序压缩为可训练 Memory Token，常数级开销 |
| **PRAXIS** | 2025.11 | 部署后实时习得程序性知识 |
| **A-MEM** | NeurIPS 2025 | Zettelkasten 式自组织记忆，复杂推理性能翻倍 |
| **JARVIS-1** | 2023 | 多模态记忆增强，200+ 种任务 |
| **Reflexion** | NeurIPS 2023 | 言语自我反思，HumanEval 91% |
| **MemoryOS** | EMNLP 2025 | OS 式三层存储（短期/中期/长期），F1 +48.36% |
| **MemOS (MemTensor)** | 2025.05 | 记忆作为一等操作资源的统一 OS |
| **程序性记忆检索 Benchmark** | 2025.11 | 首个专门 benchmark，发现 LLM 抽象比纯 Embedding 泛化更好 |
| **"Procedural Memory Is Not All You Need"** | ACM UMAP'25 | 仅程序性记忆不够，需要语义记忆和关联学习协同 |

---

## 四、工业界实现对比

### 4.1 开源框架

| 系统 | Stars | 存储后端 | 程序性记忆方式 | 检索机制 |
|------|-------|---------|--------------|---------|
| **Mem0** | ~48K | 向量DB + 图DB + KV | 行为模式编码，工作流捕获 | 语义搜索 + 图遍历 |
| **Letta/MemGPT** | ~37K | 向量DB + SQLite/PG | Agent 自主管理三层记忆 | Agent 主动函数调用 |
| **Zep/Graphiti** | — | 时序知识图谱(Neo4j) | 时序关系建模，工作流演变追踪 | 语义+BM25+图遍历(无LLM) |
| **LangMem** | — | LangGraph 存储 | 学到的程序保存为 Prompt 更新 | 统一 API |
| **CrewAI** | — | ChromaDB + SQLite3 | 长期记忆中的任务结果 | LLM 分析 + 语义搜索 |
| **Agent Zero** | — | FAISS | SKILL.md 文件 + 向量索引 | 语义搜索 + 两阶段加载 |

#### Mem0 — 混合存储架构

```
用户交互 → LLM 提取行为模式
         → 向量DB (语义检索) + 图DB (关系推理) + KV (快速查找)
         → 查询时三层合并，按优先级排序
```

- LOCOMO 基准 66.9% 准确率，P95 延迟 1.44s
- 支持四种记忆类型：语义/情景/程序性/联想

#### Letta/MemGPT — OS 式记忆管理

```
Core Memory (RAM, 始终在 context 中)
  ↕ Agent 主动调用函数管理
Recall Memory (磁盘缓存, 完整对话历史)
  ↕
Archival Memory (冷存储, 向量数据库)
```

- 独特之处：Agent 主动管理自己的记忆（而非被动注入）
- 通过 `core_memory_append`、`archival_memory_insert` 等函数调用

#### Zep/Graphiti — 时序知识图谱

```
情节子图 (原始输入)
  ↓ 提取
语义实体子图 (实体 + 关系)
  ↓ 聚合
社区子图 (高层模式)
```

- **双时态模型**：每条边包含 t_created/t_expired/t_valid/t_invalid 四个时间戳
- 特别适合追踪工作流随时间的演变
- P95 延迟 300ms，检索时不涉及 LLM 调用

#### LangMem — Prompt 即程序性记忆

核心理念：程序性记忆 = 模型权重 + Agent 代码 + **Agent Prompt**

三种更新算法：
1. **Metaprompt**：反思 + 元提示提出更新
2. **Gradient**：批评 + 提示提案两步
3. **Simple**：单步完成

### 4.2 商业产品

| 产品 | 程序性记忆方式 | 特点 |
|------|--------------|------|
| **Claude Code** | SKILL.md 文件 + MEMORY.md + CLAUDE.md | 渐进式披露（元数据→指令→资源），Git 可控 |
| **ChatGPT** | Custom Instructions + 自动记忆 | 高层偏好为主，非精确程序 |
| **Gemini** | Gems / Super Gems + user_context | 参考文档注入，非真正程序性学习 |
| **Cursor** | .cursor/rules + SKILL.md | Rules(声明式,始终开启) + Skills(程序式,按需发现) |
| **Windsurf** | Memory + Rules + M-Query | 随时间学习，避免重复解释 |
| **AWS AgentCore** | 显式程序性记忆类型 | 四种记忆类型的托管服务 |

#### Claude Skills — 渐进式披露架构

```
Layer 1: 元数据加载 (~100 tokens) — 扫描可用技能
Layer 2: 完整指令 (<5K tokens) — 确定适用时加载
Layer 3: 捆绑资源 — 文件和代码按需加载
```

- 纯文本文件，天然支持 Git 版本控制
- 技能目录：SKILL.md + scripts/ + references/ + evals/

---

## 五、关键架构决策对比

### 5.1 两大流派

**流派一：显式程序性记忆（代码/文件）**

代表：Voyager（代码）、Claude Skills / Agent Zero（SKILL.md）、SkillX（层次化技能库）

```
特点：
+ 可审计、可版本控制、可组合、可迁移
+ 精确可执行
- 需要专门的构建/维护机制
- 对非代码任务需要额外适配
```

**流派二：隐式程序性记忆（嵌入/Prompt/参数）**

代表：LangMem（Prompt 更新）、Mem0（行为模式）、TokMem（可训练令牌）

```
特点：
+ 自动化程度高，无需人工干预
+ 与 LLM 推理天然融合
- 可解释性差，难以精确控制
- 受 context window 限制
```

### 5.2 检索机制演进

```
第一代：纯 Embedding 检索（Voyager）
  → 熟悉场景 84% MAP，新场景降至 59%

第二代：混合检索（Zep: 语义+BM25+图遍历）
  → 无 LLM 调用，P95 300ms

第三代：LLM 增强检索（ReMe: Embedding→Top-K→LLM 重排→智能重写）
  → 泛化能力最强，新场景仅降 11 个百分点

关键发现（Benchmark 2025.11）：
LLM 生成的程序抽象 >> 纯 Embedding
原因：均值池化将轨迹视为"无序词袋"，丢失时序结构
```

### 5.3 更新/进化机制演进

```
第一代：只增不减（Voyager — 验证通过即添加）
  ↓
第二代：支持增删改（Mem^p — Add/Remove/Update）
  ↓
第三代：效用驱动精炼（ReMe — 追踪效用，自动裁剪）
  ↓
第四代：主动探索扩展（SkillX — 识别薄弱点，合成新任务扩展技能库）
```

### 5.4 存储后端选择

| 方案 | 适用场景 | 代表 |
|------|---------|------|
| **向量数据库** | 语义相似度检索 | Voyager, Mem0, CrewAI |
| **知识图谱** | 关系推理、时序演变 | Zep/Graphiti |
| **文件系统(Markdown)** | 人类可读、Git 管理 | Claude Skills, Agent Zero |
| **SQLite/关系DB** | 结构化查询、版本管理 | MemOS(你的项目), Letta |
| **混合架构** | 综合优势 | Mem0(向量+图+KV) |

---

## 六、核心挑战

### 6.1 记忆巩固 (Consolidation)
- 如何将短期情景经验转化为持久程序性知识
- MemoryBank 采用 Ebbinghaus 遗忘曲线的指数衰减模型
- A-MEM 新记忆触发旧记忆的上下文表示更新

### 6.2 选择性遗忘 (Forgetting)
- 被严重忽视但至关重要的能力
- 仅 MemoryAgentBench 明确测试选择性遗忘
- 无法丢弃过时信息会逐渐毒化检索精度

### 6.3 跨场景泛化
- Embedding 方法在新场景退化 30-42%（Benchmark 2025.11）
- LLM 生成的抽象表示泛化更好（仅降 11 个百分点）
- 强模型构建的记忆可迁移到弱模型（Mem^p）

### 6.4 规模与效率
- 程序性记忆池增长后的检索效率
- 渐进式披露（Claude Skills）和分层加载（Agent Zero）是当前最佳实践

### 6.5 评估体系
- 遗忘、过时、矛盾等维度的评估几乎空白
- 需要联合评估记忆质量和决策质量

---

## 七、与 MemOS 的关系

你的 MemOS 项目在架构上与学术界和工业界的多个方向有对应关系：

| MemOS 特性 | 对应学术工作 | 对应工业实现 |
|-----------|-------------|-------------|
| 消息捕获 + 主题分割 | AWM 的轨迹归纳 | CrewAI 的任务结果存储 |
| 两级去重（精确+语义） | ReMe 的相似性去重 | Mem0 的记忆去重 |
| Skill 评估 → 生成/升级 | SkillX 的提取→精炼 | Agent Zero 的 SKILL.md |
| 混合搜索 (FTS + 向量 + RRF) | 第二代混合检索 | Zep 的语义+BM25+图 |
| Skill 版本管理 | Mem^p 的 Update 策略 | Claude Skills 的 Git 管理 |
| before_prompt_build 自动召回 | 上下文注入范式 | Windsurf 的上下文组装 |

**MemOS 的差异化优势**：
1. **完整的 Skill 生命周期管理**：从评估→生成→验证→安装→升级，比大多数系统更完整
2. **双端实现**（Python 批量 + TS 实时）覆盖不同场景
3. **Hub 同步机制**支持技能的发布和共享

**潜在改进方向**（基于调研发现）：
1. 引入 **效用评分** 驱动的精炼机制（参考 ReMe），追踪技能被检索和使用的效果
2. 考虑 **层次化技能组织**（参考 SkillX 的三级体系），当前 MemOS 的 Skill 是扁平结构
3. 增加 **选择性遗忘** 能力，自动废弃长期低效用的 Skill
4. 检索可引入 **LLM 重排序** 环节（参考 ReMe 的三阶段检索），提升新场景泛化

---

## 八、研究趋势总结

1. **从被动存储到主动精炼**：只增不减 → 效用驱动增删改 → 主动探索扩展
2. **从单一格式到层次化表示**：纯代码/纯文本 → 三级技能体系（战略/功能/原子）
3. **从 Embedding 到 LLM 增强检索**：纯向量相似度 → 混合检索 → LLM 重排+智能重写
4. **文件即记忆 (Markdown Memory) 成为趋势**：SKILL.md / CLAUDE.md 等纯文本方式支持 Git 版本控制
5. **记忆质量可替代模型规模**：ReMe 证明 8B+好记忆 > 14B 无记忆
6. **非参数化方案成为主流**：冻结 LLM + 外部记忆 > 微调方案
7. **多记忆类型必须协同**："Procedural Memory Is Not All You Need"

---

## 九、参考文献索引

### 综述论文
- [A Survey on the Memory Mechanism of LLM-based Agents](https://arxiv.org/abs/2404.13501) (TOIS 2025)
- [Memory in the Age of AI Agents](https://arxiv.org/abs/2512.13564) (2025.12)
- [Memory for Autonomous LLM Agents](https://arxiv.org/html/2603.07670v1) (2026.03)
- [Anatomy of Agentic Memory](https://arxiv.org/abs/2602.19320) (2026.02)
- [Agent Skills from the Perspective of Procedural Memory](https://www.techrxiv.org/users/1016212/articles/1376445) (2025)

### 核心论文
- [CoALA: Cognitive Architectures for Language Agents](https://arxiv.org/abs/2309.02427) (2023)
- [Voyager: Open-Ended Embodied Agent](https://arxiv.org/abs/2305.16291) (2023)
- [Mem^p: Exploring Agent Procedural Memory](https://arxiv.org/abs/2508.06433) (2025)
- [ReMe: Remember Me, Refine Me](https://arxiv.org/abs/2512.10696) (2025.12)
- [SkillX: Automatically Constructing Skill Knowledge Bases](https://arxiv.org/abs/2604.04804) (2026.04)
- [MACLA: Learning Hierarchical Procedural Memory](https://arxiv.org/abs/2512.18950) (AAMAS 2026)
- [TokMem: Tokenized Procedural Memory](https://arxiv.org/abs/2510.00444) (2025.10)
- [AWM: Agent Workflow Memory](https://arxiv.org/abs/2409.07429) (ICML 2025)
- [A-MEM: Agentic Memory](https://arxiv.org/abs/2502.12110) (NeurIPS 2025)
- [PRAXIS: Real-Time Procedural Learning](https://arxiv.org/abs/2511.22074) (2025.11)
- [Procedural Memory Retrieval Benchmark](https://arxiv.org/abs/2511.21730) (2025.11)
- [CodeMem: Procedural Memory via MCP](https://arxiv.org/abs/2512.15813) (2025)
- [LEGOMem: Modular Procedural Memory](https://arxiv.org/abs/2510.04851) (AAMAS 2026)
- [Synthesizing Procedural Memory](https://arxiv.org/abs/2512.20278) (2025)
- [Procedural Memory Is Not All You Need](https://arxiv.org/abs/2505.03434) (ACM UMAP'25)

### 经典 Agent 记忆论文
- [MemGPT: Towards LLMs as Operating Systems](https://arxiv.org/abs/2310.08560) (2023)
- [Reflexion: Language Agents with Verbal Reinforcement Learning](https://arxiv.org/abs/2303.11366) (NeurIPS 2023)
- [ExpeL: LLM Agents Are Experiential Learners](https://arxiv.org/abs/2308.10144) (AAAI 2024)
- [Generative Agents: Interactive Simulacra of Human Behavior](https://arxiv.org/abs/2304.03442) (2023)
- [GITM: Ghost in the Minecraft](https://arxiv.org/abs/2305.17144) (2023)
- [Memory OS of AI Agent](https://aclanthology.org/2025.emnlp-main.1318/) (EMNLP 2025)
- [MemOS (MemTensor)](https://arxiv.org/abs/2505.22101) (2025.05)
- [MIRIX: Multi-Agent Memory System](https://arxiv.org/abs/2507.07957) (2025)

### GitHub 资源
- [Agent Memory Paper List](https://github.com/Shichun-Liu/Agent-Memory-Paper-List)
- [LLM Agent Memory Survey](https://github.com/nuster1128/LLM_Agent_Memory_Survey)
- [MemTensor/MemOS](https://github.com/MemTensor/MemOS)
- [BAI-LAB/MemoryOS](https://github.com/BAI-LAB/MemoryOS)
- [zjunlp/MemP](https://github.com/zjunlp/MemP)
- [zjunlp/SkillX](https://github.com/zjunlp/SkillX)