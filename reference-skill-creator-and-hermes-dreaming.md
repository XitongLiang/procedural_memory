# 两种程序记忆演进范式：Claude Skill-Creator vs Hermes Dreaming

---

## 一、Claude Skill-Creator：在线工作中提炼技能

### 1.1 三套实现

实际上存在三套独立但理念相通的 skill-creator：

| 系统 | 开发者 | 哲学 | 触发方式 |
|------|--------|------|---------|
| **Anthropic 官方** skill-creator | Anthropic | 测试驱动 + 描述优化 | 用户主动触发 |
| **Superpowers** writing-skills | Jesse Vincent (obra) | TDD 红绿重构 + 说服心理学 | 用户主动触发 |
| **OpenClaw** learn_skill | OpenClaw | 自动提取 + 去标识化 | 会话结束后自动 |

### 1.2 Anthropic 官方管道（5 阶段）

```
[1] 意图捕获 → [2] 访谈研究 → [3] 编写 SKILL.md → [4] 测试 → [5] 迭代优化
```

**阶段 1-2：意图捕获与访谈**
- 技能应该让 Claude 做什么？何时触发？期望输出？
- 询问边缘情况、示例、成功标准、依赖项
- 在此阶段完成前不写任何技能内容

**阶段 3：编写 SKILL.md — 渐进式披露**

三级加载架构：

| 层级 | 加载时机 | 大小 | 内容 |
|------|---------|------|------|
| L0 元数据 | 始终在上下文中 | ~100 词 | name + description |
| L1 指令 | 技能被触发时 | <500 行 | SKILL.md 完整内容 |
| L2 资源 | 按需加载 | 无限制 | scripts/ references/ assets/ |

目录结构：
```
skill-name/
├── SKILL.md           # 必需：YAML frontmatter + Markdown 指令
├── scripts/           # 可执行代码
├── references/        # 参考文档
├── assets/            # 模板、图标等
└── evals/evals.json   # 测试用例
```

**阶段 4：测试 — 四个并行子代理**

| 子代理 | 职责 |
|--------|------|
| Executor | 对 eval 提示运行技能 |
| Grader | 根据期望评估输出 |
| Comparator | 有技能 vs 无技能 盲测 A/B 对比 |
| Analyzer | 发现聚合统计隐藏的模式 |

工作区结构：`iteration-N/eval-ID/{with_skill,without_skill}/outputs/`

**阶段 5：描述优化（核心子系统）**

```
生成 20 个评估查询（混合"应触发"和"不应触发"）
  → 60% 训练 / 40% 测试 拆分
  → 每个查询运行 3 次获取可靠触发率
  → Claude 基于失败案例提出改进建议
  → 最多 5 轮迭代
  → 按测试分数（非训练分数）选最佳描述 ← 防过拟合
```

### 1.3 Superpowers writing-skills：TDD 哲学

**铁律：没有先失败的测试，就不能创建技能。**

RED-GREEN-REFACTOR 循环：

```
RED（基线测试）：
  无技能情况下运行压力场景
  → 记录 agent 的确切行为和合理化借口（逐字记录）
  → 识别哪些压力触发了违规

GREEN（最小化技能）：
  编写技能专门针对基线中发现的合理化借口
  → 不为假设情况添加内容
  → 验证 agent 现在遵守规则

REFACTOR（封堵漏洞）：
  发现新合理化借口 → 添加明确反驳
  → 构建"合理化借口表" + "红旗清单"
  → 更新描述包含"即将违规"症状
  → 重新测试直到无懈可击
```

**说服心理学应用**（Cialdini 2021, Meincke et al. 2025 N=28,000）：

合规率从 33% 提升到 72%，最有效组合：
- **权威**："YOU MUST"、"Never"、"No exceptions"
- **承诺**：要求宣布技能使用、强制显式选择
- **稀缺性**："IMMEDIATELY"、防止拖延
- **社会证明**：建立"每次都这样"的规范

**CSO（Claude Search Optimization）陷阱**：

```yaml
# 错误：描述总结了工作流 → Claude 直接按描述执行，不读完整技能
description: Use when executing plans - dispatches subagent per task with code review

# 正确：只描述触发条件
description: Use when executing implementation plans with independent tasks
```

### 1.4 OpenClaw 自动技能提取

与前两者不同，OpenClaw 完全自动化，无需用户主动触发：

```
会话结束
  → 证据溯源（用户发言为主，agent 输出仅参考）
  → 话题边界检测（从末尾往前找最新完整任务）
  → 去标识化（项目名/路径/人名 → 占位符）
  → 复用性判断（<3 步 → too_simple → 跳过）
  → 踩坑经验提取（失败后成功的模式 → 注意事项/禁止事项）
  → 生成 SKILL.md
  → 与已有技能去重/合并
```

**提取质量门控**：
- 任务未成功 → 跳过（`"reason":"not_success"`）
- 去标识化后无复用价值 → 跳过（`"reason":"not_reusable"`）
- 工作流 < 3 步 → 跳过（`"reason":"too_simple"`）

**证据信任层级**：
1. 用户发言 = 主要证据
2. Agent 成功执行的工具调用序列 = 有效证据
3. Agent 结构化输出 = 有效证据
4. Agent 解释性文字 = 仅参考
5. skill_search / memory_search 返回 = 完全忽略

---

## 二、Hermes Dreaming：离线记忆整理

### 2.1 Hermes 是什么

- **开发者**：Nous Research
- **发布**：2026 年 2 月
- **定位**：开源自进化 AI Agent，"The agent that grows with you"
- **GitHub**: [NousResearch/hermes-agent](https://github.com/nousresearch/hermes-agent)

### 2.2 四层记忆架构

| 层 | 名称 | 存储 | 大小限制 | 加载时机 |
|----|------|------|---------|---------|
| 1 | **Prompt Memory** | MEMORY.md + USER.md | ~800 + ~500 tokens | 会话启动时冻结注入 |
| 2 | **Skills** | ~/.hermes/skills/*.md | 三级加载 | L0 始终/L1 按需/L2 按需 |
| 3 | **Session Archive** | SQLite + FTS5 | 无限制 | Agent 主动搜索 |
| 4 | **External Providers** | 8 种插件 | 取决于提供者 | Agent 主动调用 |

**关键设计**：Prompt Memory 在会话中是**冻结快照**——会话期间的修改写入磁盘但不影响当前系统提示（保持 LLM prefix cache 性能），下一个会话才生效。

### 2.3 在线记忆处理（会话中）

**Nudge Interval（周期性提醒）**：
- 可配置间隔，Agent 定期收到内部提示
- 要求回顾最近活动，评估是否有内容值得持久化
- 无需用户输入，自动触发
- 超过"对未来有用"阈值 → 写入记忆文件

**记忆操作**（通过 `memory` 工具）：
- `add`：插入新条目
- `replace`：子字符串匹配更新（"dark mode" 能匹配到 "User prefers dark mode in all editors"）
- `remove`：子字符串匹配删除

**容量触发合并**：
- 达到 80% 容量限制时工具返回错误
- Agent 必须合并或删除条目后才能重试
- 例：三个独立项目记录 → 一个综合描述

### 2.4 离线记忆整理（"Dreaming"）

这是 Hermes 最独特的机制，由多个层次组成：

#### 机制 1：Gateway 模式主动刷新

```
空闲超时 / 每日凌晨 4 点
  → 生成临时 AIAgent
  → 回顾旧对话历史
  → 写入 MEMORY.md
  → 销毁临时 Agent
```

这是消息场景（Telegram/Discord）中持久化记忆的主要机制。

#### 机制 2：Cron 定时任务

Hermes 内置 cron 引擎，支持自然语言配置：
- 例："每天早上 9 点检查 HN 的 AI 新闻，给我发 Telegram 摘要"
- 在**全新会话**中运行，没有当前聊天记忆
- 提示必须完全自包含

#### 机制 3：Hindsight Reflect — 跨记忆综合（核心）

**Hindsight** 是 8 个外部记忆提供者中唯一提供 `reflect` 操作的，也是实现深度离线整理的关键：

```
hindsight_reflect(query)
  → 遍历整个知识图谱
  → 跨所有存储记忆推理
  → 产生综合性答案（比 recall 慢但更深度）
  → 将新洞察写回知识图谱
```

**Hindsight 独特的数据结构**：
- 不存纯文本，而是存**结构化知识**
- 提取离散事实："生产 DB 运行在端口 5433"
- 提取命名实体：人、服务、项目
- 提取实体间关系："auth 服务依赖 Redis 做会话管理"
- LongMemEval 基准最高分：91.4%

三个工具：
| 工具 | 用途 | 速度 |
|------|------|------|
| `hindsight_retain` | 保存记忆（异步提取事实/实体/关系） | 快 |
| `hindsight_recall` | 检索记忆 | 快 |
| `hindsight_reflect` | 跨记忆综合 | 慢（需要遍历图谱推理） |

#### 机制 4：用户自建"梦境"技能（实践案例）

Alexander Kucera 在 "One Week on Hermes" 中实现的：

```
每 4 小时：
  运行 hindsight-reflect 技能
  → 对所有记忆进行跨记忆综合

每晚（过夜梦境）：
  综合当天所有会话
  → 整合所学内容
  → 早上呈现发现结果

效果：
  Agent 在用户睡觉时整理知识
  多个 Agent 互相调试代码、维护 wiki
```

### 2.5 提案中的认知记忆操作（Issue #509）

受 CrewAI 启发的完整认知记忆系统提案，5 大操作：

#### Encoding（编码）— 5 步流水线
```
批量嵌入 → 批内去重(>=0.98) → 并行相似度搜索 → 并行 LLM 分析 → 批量执行
```

LLM 产出结构：
- `suggested_scope`：层级路径（如 `/infrastructure/database`）
- `categories`：标签
- `importance`：0.0-1.0
- `extracted_metadata`：实体、日期、主题

#### Consolidation（合并整理）
- 触发：相似度 >= 0.85
- 生成 `ConsolidationPlan`：keep / update / delete
- 例：已有 "用 PostgreSQL" + 新 "迁移到 MySQL" → 合并为 "从 PostgreSQL 迁移到 MySQL"

#### Recall（召回）— 复合评分
```
score = 0.5 × similarity + 0.3 × decay + 0.2 × importance
decay = 0.5^(age_days / 30)

置信度路由：
  >= 0.8 → 直接返回
  < 0.5 → 有预算就深挖
  < 0.7 且复杂查询 → 深挖
```

#### Extraction（提取）
原始文本 → 分解为原子事实 → 每个事实独立进入编码流水线

#### Forgetting（遗忘）
按范围/类别/年龄/记录 ID 定向清除

---

## 三、在线 vs 离线对比

| 维度 | 在线（工作过程中） | 离线（Dreaming） |
|------|------------------|-----------------|
| **触发** | 用户交互 / 会话结束 / nudge 定时器 | Cron / 空闲超时 / 凌晨重置 |
| **操作粒度** | 单条记忆增删改 / 单个技能提取 | 跨所有记忆综合推理 |
| **处理深度** | 浅：这条对话值不值得记 | 深：所有记忆之间的矛盾/冗余/缺失 |
| **数据结构** | 平文本 Markdown / 单条 Skill | 知识图谱 / 层级作用域 / 实体关系 |
| **LLM 调用** | 1-2 次（提取+决策） | N 次（遍历图谱推理） |
| **对会话影响** | 写磁盘但不改当前 prompt | 更新知识图谱，下次会话生效 |
| **人类类比** | 白天边做边记笔记 | 睡觉时大脑整理记忆 |

---

## 四、对你的程序记忆模块的启示

### 来自 Skill-Creator 的启示（在线路径）

1. **渐进式披露是共识**：三套系统都用了三级加载，区别只在自动化程度
2. **证据溯源很关键**：OpenClaw 和 AutoSkill 都强调"用户输入为主要证据，agent 输出仅参考"
3. **测试驱动创建**：Anthropic 用 4 个并行子代理做 A/B 对比，Superpowers 用 TDD 红绿重构
4. **描述优化防过拟合**：训练/测试拆分，按测试分数选最佳描述

### 来自 Hermes Dreaming 的启示（离线路径）

1. **冻结快照 + 异步写入**：在线修改不影响当前会话的系统提示，下次生效
2. **Reflect > Recall**：简单检索不够，需要跨记忆综合推理
3. **知识图谱是深度整理的基础**：Hindsight 的事实+实体+关系结构是 reflect 能力的前提
4. **容量压力驱动整合**：容量限制不是缺陷，而是触发记忆合并的机制
5. **用户可自建梦境**：框架提供原语，用户组合出 dreaming 流程

### 两条路径的结合点

```
在线路径（工作过程中）：
  对话 → 提取候选 Skill → 质量门控 → 存储
  ↓ 产生原料

离线路径（Dreaming）：
  定时触发 → 遍历所有 Skill/记忆
  → 合并重复 / 发现矛盾 / 补全缺失 / 淘汰过时
  → 测试验证（模拟任务）
  → 更新 Skill 库
  ↓ 精炼产出

下次会话：加载精炼后的 Skill
```

---

## 参考资料

**Claude Skill-Creator**
- [Anthropic Skills 仓库 - skill-creator](https://github.com/anthropics/skills/blob/main/skills/skill-creator/SKILL.md)
- [Improving skill-creator: Test, measure, and refine](https://claude.com/blog/improving-skill-creator-test-measure-and-refine-agent-skills)
- [Agent Skills Specification](https://agentskills.io/specification)
- [Skill authoring best practices](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices)
- [Equipping agents for the real world](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)

**Superpowers**
- [obra/superpowers](https://github.com/obra/superpowers/)
- 本地路径：`~/.claude/plugins/cache/claude-plugins-official/superpowers/5.0.7/skills/writing-skills/`

**Hermes Agent**
- [GitHub - NousResearch/hermes-agent](https://github.com/nousresearch/hermes-agent)
- [Hermes Agent 文档](https://hermes-agent.nousresearch.com/docs/)
- [Persistent Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory/)
- [Memory Providers](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory-providers)
- [Issue #509: Cognitive Memory Operations](https://github.com/NousResearch/hermes-agent/issues/509)
- [One Week on Hermes - Alexander Kucera](https://alexanderkucera.com/2026/04/10/one-week-on-hermes.html)
- [Hindsight Blog](https://hindsight.vectorize.io/blog/2026/03/17/hermes-agent-memory-memory)