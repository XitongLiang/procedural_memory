# GSPD 程序记忆设计

> 基于 [design-v5-learning.md](design-v5-learning.md) 的设计概要与对外说明
> 最后更新：2026-04-21

---

## 一、三类知识的分类设计

GSPD 程序记忆的**知识层**分为三类：

- **Feedback (Fb)** — 错误更正记忆（动作级硬规则）
- **Tool Preference (TP)** — 工具选择偏好（工具场景级）
- **Task-specific Experience (TSE)** — 任务经验（任务级软建议）

下文回答两个问题：**为什么这么分？** 以及 **这么分带来什么优势？**

---

### 1.1 三类速览表

| 维度 | **Feedback (Fb)** | **Tool Preference (TP)** | **Task-specific Experience (TSE)** |
|------|-------------------|--------------------------|-------------------------------------|
| **一句话本质** | 错误纠正后的硬规则 | 工具选择的稳定倾向 | 任务情境化的执行经验 |
| **强度** | 硬约束（必须遵循） | 硬/软（视来源） | 软建议（agent 自行判断） |
| **匹配粒度** | **动作级** | **工具场景级** | **任务级** |
| **来源** | 用户纠正 / 单次失败→恢复 / **反复失败累积** | 用户指定 / 反复选择 (≥3 次) | 用户主动补充 / 成功失败归纳 |
| **内容回答的问题** | 禁止/必须做什么 | 选哪个工具 | 怎么做、注意什么 |
| **适用范围（新任务）** | **跨任务**：任何包含该动作的任务都触发 | **跨任务**：任何该工具场景都触发 | **任务绑定**：仅相似任务触发 |
| **检索查询输入**（问题） | 即将执行的动作/工具 | 当前工具选择点 | 任务摘要 + 标签 |
| **注入时机**（问题） | 动作决策前 | 工具选择时 | 任务开始时 |
| **长度限制** | content ≤ 50 字 | content ≤ 50 字 | 每条 bullet ≤ 50 字 |
| **合并/丰富方式** | hybrid 查重 + LLM 灰区决策 + 候选池升格 | hybrid 查重 + LLM 灰区决策 + 候选池升格 | hybrid 查重 + JudgeMatch → Enrich（bullet 并集）/ Learn（新建） |

---

### 1.2 为什么这么分类

#### 1.2.1 强度差异（硬约束 vs 软建议）

| 类型 | 强度语义 | 违反后果 |
|------|---------|---------|
| Fb | 禁止 / 必须 | 会直接出错 |
| TP | 倾向 | 效率损失 |
| TSE | 建议 | 可能次优 |

**合成一类的代价**：
- 要么**过强**（建议变硬约束 → agent 死板不灵活）
- 要么**过弱**（硬约束降为建议 → 重犯已犯过的错）

#### 1.2.2 匹配粒度差异（跨任务 vs 任务绑定）

这是 GSPD 分类的**关键洞察**——Fb/TP 的核心价值正在于它们是横向规则，能在**未执行过的新任务**里触发。

**案例**：首次做"给 Rust 项目加 CI"（过去从未做过）：

- **TSE**（过往经验）：无匹配，因为未做过 Rust CI
- **Fb**：任务涉及修改 CI 配置 → `"改 config 后要重启 service"` 命中
- **TP**：任务需要搜 build 配置 → `"搜代码用 rg"` 命中；需要查进程 → `"用 ps/pgrep"` 命中

**结论**：从 0 到 1 的新任务仍能受益于过往教训——**这只有在 Fb/TP 跨任务触发时才成立**。

**合成一类的代价**：所有知识按任务相似度检索 → Fb/TP 的横向价值被任务相似度过滤掉 → 退化成"任务绑定"的规则。

#### 1.2.3 `description` 字段语义差异

| 类型 | description 应是什么 | 好例子 | 坏例子 |
|------|-----------------|--------|--------|
| **Fb** | **动作触发条件**（agent 即将执行某动作/遇到某情境时召回） | • "修改配置后"<br>• "删除文件前"<br>• "遇到 ECONNREFUSED 时"<br>• "推送代码前"<br>• "运行迁移脚本前"<br>• "调 API 返回非 200 时"<br>• "回滚 commit 后"<br>• "清理临时文件时" | • "部署场景"（是任务领域，非动作）<br>• "xxx 服务"（是领域标签）<br>• "后端"（过于宽泛） |
| **TP** | **操作场景**（agent 在某类操作点选工具时召回） | • "搜代码时"<br>• "管 Python 依赖时"<br>• "读大文件时"<br>• "查进程状态时"<br>• "生成代码 diff 时"<br>• "运行单元测试时"<br>• "处理 JSON/YAML 时"<br>• "查看 git 历史时" | • "后端开发"（任务领域）<br>• "调试任务"（过宽）<br>• "自动化"（无具体操作） |
| **TSE** | **完整描述句**（类似 skill description，描述"在什么任务情境下参考本经验"） | • "调试 FastAPI 后端项目，涉及 schema 生成、client 对接、接口测试等场景"<br>• "编写每周工作总结，需要汇总完成事项、进行中任务与遇到的问题"<br>• "推送代码到生产环境部署前，验证变更符合预期避免事故"<br>• "为新项目首次配置 CI/CD pipeline（GitHub Actions / GitLab CI 等）"<br>• "对已有代码模块进行重构或架构调整，需要保持行为兼容"<br>• "接入外部第三方 API 服务（支付、认证、消息推送等）"<br>• "进行数据库 schema 变更、数据迁移或索引优化"<br>• "分析和优化系统性能瓶颈（CPU / 内存 / IO / 查询慢）" | • "当用 curl 时"（是工具场景 → 应为 TP）<br>• "修改配置后"（是动作触发 → 应为 Fb）<br>• "FastAPI"（只是领域标签，非描述句） |

三类字段含义不同 → 检索查询的抽象层级必须分开 → **存储必须分开**（不能共享一张 description 向量表）。

**识别规则**：

- 若 description 可直接作为"**agent 执行某动作前/后**"的钩子 → 属于 **Fb**
- 若 description 是"**执行某类操作时需选工具**" → 属于 **TP**
- 若 description 是"**完成某类任务目标**" → 属于 **TSE**

#### 1.2.4 生命周期差异

| 类型 | 演进特征 | 老化策略 |
|------|---------|---------|
| Fb | 一次纠正长期有效（硬约束稳定） | 低老化权重 |
| TP | 工具会更新换代（uv 替代 pip、rg 替代 grep） | 中等老化 |
| TSE | 经验会迭代优化（新场景加入 bullet） | 不老化，靠 enrich 持续累积 |

**合成一类的代价**：单一老化策略无法兼顾——Fb 过快老化会丢失硬约束；TSE 过慢老化会积累过时经验。

#### 1.2.5 证据要求差异

| 类型 | 升格门槛 | 证据类型 |
|------|---------|---------|
| Fb | 强证据 | 用户亲口纠正 / 可泛化的失败恢复 |
| TP | 重复出现 | 同类任务 ≥ 3 次选择同一工具 |
| TSE | 较宽松 | 用户主动补充 / 成功路径归纳 |

**合成一类的代价**：弱证据条目（TSE）会稀释强证据（Fb）的 `support_score`，动态升格阈值无法针对性设定。

---

### 1.3 分类带来的优势

分开存储 + 分开检索 + 分开注入，换来五项核心能力：

#### 优势 1：强度分级 → 决策边界清晰

- **Fb** 注入 system prompt 作硬约束（"不允许做 X / 必须做 Y"）
- **TP** 在工具选择点作偏好（"优先用 X"）
- **TSE** 在任务开始时作参考（"可以考虑..."）

agent 知道**哪些必须遵守、哪些可以参考**——不会把建议误当规则，也不会把规则降为建议。

#### 优势 2：跨任务泛化 → Fb/TP 在新任务中依然起效

即使是从未做过的新任务，只要**动作/工具层面**匹配，Fb/TP 就能发挥作用（1.2.2 示例）。

**单类设计（Mem0/ReMe）做不到**：它们按任务相似度检索，新任务因 task_summary 无匹配而召回空集。

#### 优势 3：检索精准 → 各粒度不互相稀释

- Fb/TP 用动作/工具作查询主体
- TSE 用任务摘要作查询主体
- `description` 字段语义对齐，相似度计算不漂移

#### 优势 4：生命周期独立 → 老化与 enrich 各自适配

每类用适合自己特征的策略：
- Fb：低老化（稳定规则）
- TP：中老化（跟随工具演进）
- TSE：不老化 + enrich（bullet_points 语义并集，累积而非替换）

#### 优势 5：证据不稀释 → 升格阈值可针对性设定

`support_score = 0.50·mention_ratio + 0.25·recency + 0.25·hard_ratio` 对三类的 `hard_count` 含义各自清晰：
- Fb 的 hard_ratio 只在 Fb 候选池内计算，不被 TSE 的大量软条目拉低
- 动态阈值（0.18 → 0.38）可针对每类的特征独立调参

---

### 1.4 与"单类经验记忆"的对比

以 Mem0/ReMe 的单类设计为对照：

| 维度 | **单类设计（Mem0/ReMe）** | **三类设计（GSPD）** |
|------|-------------------------|-------------------|
| 强度 | 全部同一强度（参考性） | 硬约束 / 偏好 / 建议三档分级 |
| 新任务泛化 | 按任务相似度检索 → 新任务无命中 | 动作级 Fb + 工具级 TP 跨任务触发 |
| 检索精准度 | description 字段模糊（任务/动作/工具混杂） | description 按粒度分层，语义对齐 |
| 注入时机 | 统一一次注入（任务开始） | 按时机分三次（任务开始 / 工具选择 / 动作决策） |
| 升格阈值 | 单一阈值 | 各类独立阈值 |
| 老化策略 | 全部统一（utility/freq） | 各类独立 |

---

### 1.5 统一结构：`description + content`

三类在结构层面**完全统一**：

```
每条知识 = description + content

  description = 使用场景 / 召回触发条件（类比 skill 的 description / when-to-use）
  content     = 具体做什么 / 经验内容
```

**三类的 `content` 形态差异**：

| 类型 | content 形态 | 例子 |
|------|-------------|------|
| **Fb** | 单句规则 | "必须重启 service 才生效" |
| **TP** | 单句偏好 | "优先用 `uv`" |
| **TSE** | bullet points 列表（`1. 2. 3.` 编号） | 1. 先跑 schema<br>2. 再写 client<br>3. 避免手写 stub 过时 |

**意义**：
- **Schema 一致**：三类共用 `description + content` 数据模型，读写接口对称
- **提取模板一致**：LLM prompt 可以抽出共同骨架（"生成 `description: ...` + `content: ...`"），仅三类的 content 约束不同
- **展示一致**：UI 侧统一按 `[{description}] {content}` 呈现，不需要按类型写三套组件
- **TSE 的 bullet points 是 content 的一种形态**（同一情境下的多条相关经验），不是额外的第四字段
- **命名对齐 skill 惯例**：`description` 对应 skill `description` 的"何时召回"；`content` 对应 skill body 的"做什么"

---

### 1.6 Feedback 详解

**字段含义**：

| 字段 | 含义 | 约束 / 取值 |
|------|------|------------|
| `name` | 条目短标题（用于候选池 kind+name 精确匹配） | 4-8 字 |
| `description` | **When to Use**：动作触发条件（类比 skill 的 description，告诉 agent 何时召回本条）| 以动作条件短语开头（示例："修改配置后" / "删除文件前" / "遇到 ECONNREFUSED 时" / "推送代码前" / "运行迁移前" / "调 API 返回非 200 时" / "回滚 commit 后" / "清理临时文件时"）|
| `content` | **做什么**：禁止或必须的行为指令 | ≤ 50 字，可操作动作（非抽象原则） |
| `is_hard` | 硬约束标志 | `true` = 必须遵循；`false` = 警示（累积后可升格） |
| `source_user_correction` | 来源区分 | `true` = 用户亲口纠正；`false` = agent 失败恢复 / 反复累积 |
| `source_sessions` | 来源 session ID 列表 | JSON array |

**示例**：

**例 1：用户纠正型**
```yaml
name: "删文件工具"
description: "删除文件时"
content: "用 trash-cli，不要直接 rm -rf"
is_hard: true
source_user_correction: true
```

**例 2：单次失败恢复型**
```yaml
name: "连接重试"
description: "遇到 ECONNREFUSED 时"
content: "改用本地回环端口 127.0.0.1"
is_hard: true
source_user_correction: false
```

**例 3：反复失败累积型**
```yaml
name: "配置生效"
description: "修改 config 后"
content: "必须重启 service 才生效"
is_hard: false    # 初始（累积前） → 升格后为 true
source_user_correction: false
```

---

### 1.7 Tool Preference 详解

**字段含义**：

| 字段 | 含义 | 约束 / 取值 |
|------|------|------------|
| `name` | 工具名 / 条目标题 | 4-8 字，通常是具体工具名 |
| `description` | **When to Use**：操作类型（类比 skill 的 description，告诉 agent 何时召回本条）| 操作类别短语（示例："搜代码时" / "管 Python 依赖时" / "读大文件时" / "查进程状态时" / "生成代码 diff 时" / "运行单元测试时" / "处理 JSON/YAML 时" / "查看 git 历史时"）|
| `content` | **选什么**：偏好工具及简要理由 | ≤ 50 字，可操作选择指令 |
| `is_hard` | 硬约束标志 | `true` = 用户指定必用；`false` = agent 自发倾向 |
| `source_sessions` | 来源 session ID 列表 | JSON array |

**示例**：

**例 1：用户指定型**
```yaml
name: "搜代码工具"
description: "搜代码时"
content: "优先用 rg，不用 grep"
is_hard: true
```

**例 2：自发规律型**（同类任务 ≥ 3 次反复选择）
```yaml
name: "Python 依赖"
description: "管 Python 依赖时"
content: "优先用 uv（比 pip / poetry 快）"
is_hard: false
```

**例 3：读大文件**
```yaml
name: "读大文件"
description: "读大文件时"
content: "用 head / tail，避免 cat 整个加载"
is_hard: false
```

---

### 1.8 Task-specific Experience 详解

**字段含义**：

| 字段 | 含义 | 约束 / 取值 |
|------|------|------------|
| `name` | 经验主题 | 4-8 字，描述适用任务类型 |
| `description` | **When to Use**：完整描述句，直接对应 skill 的 `description` 字段（告诉 agent"在什么任务情境下参考本条经验"）| **完整描述句**（skill description 风格）。示例：<br>• "调试 FastAPI 后端项目，涉及 schema 生成、client 对接、接口测试等场景"<br>• "编写每周工作总结，需要汇总完成事项、进行中任务与遇到的问题"<br>• "推送代码到生产环境部署前，验证变更符合预期避免事故"<br>• "为新项目首次配置 CI/CD pipeline（GitHub Actions / GitLab CI 等）"<br>• "对已有代码模块进行重构或架构调整，需要保持行为兼容"<br>• "接入外部第三方 API 服务（支付、认证、消息推送等）"<br>• "进行数据库 schema 变更、数据迁移或索引优化"<br>• "分析和优化系统性能瓶颈（CPU / 内存 / IO / 查询慢）" |
| `content` | **怎么做**：bullet points 列表 | 每条 ≤ 50 字，多条相关经验组合 |
| `source_sessions` | 来源 session ID 列表 | JSON array |

**注意**：TSE **不使用 `is_hard` 字段**（统一为软建议），也不区分 `source_user_correction`（用户主动补充与 agent 归纳都归入此类）。

**示例**：

**例 1：FastAPI 开发**
```yaml
name: "FastAPI 开发"
description: "调试 FastAPI 后端项目，涉及 schema 生成、client 对接、接口测试等场景"
content:
  - "1. 先跑 schema 生成，再写 client，避免手写 stub 过时"
  - "2. 用 pydantic v2 的严格模式校验请求体"
  - "3. 测试时用 httpx.AsyncClient 而非 requests"
```

**例 2：周报生成**
```yaml
name: "周报生成"
description: "编写每周工作总结，需要汇总完成事项、进行中任务与遇到的问题"
content:
  - "1. 先查上周 TODO 列表和完成状态"
  - "2. 再扫本周会议纪要和关键讨论"
  - "3. 汇总格式：完成项 + 进行中 + 遇到的 blocker"
```

**例 3：部署前检查**
```yaml
name: "部署前检查"
description: "推送代码到生产环境部署前，验证变更符合预期避免事故"
content:
  - "1. 先跑 dry-run 看预期变更"
  - "2. 检查 config diff 是否符合预期"
  - "3. 确认依赖服务健康状态"
```

---

## 二、提取流水线设计（Step 1 → Step 6）

GSPD 程序记忆的提取管线分 6 个主 Step，核心提取发生在 Step 5（Chunk Flush）内的 4 个子步骤。本节逐步讲**设计原理 / 优势 / 对任务完成度的贡献**。

基础流水线：

```
Step 1 预过滤 → Step 2 滑动窗口 → Step 3 主题抽取 → Step 4 主题拼接
                                                              │
                                     flush 触发 ────────────────┘
                                     │
                                     ▼
                     ┌──────── Step 5 Chunk Flush ────────┐
                     │  (A) Task Trace 提取                │
                     │  (B) Fb/TP 提取 + 两层查重           │
                     │  (C) TSE 提取 + Enrich/Learn        │
                     │  (D) MaintainPendingPool            │
                     └─────────────────┬───────────────────┘
                                       │
                                       ▼
                                Step 6 水位推进
```

---

### 2.1 Step 1 — 预过滤（零 LLM）

**设计原理**：
两层清洗，全部正则/规则，不调 LLM：
- 第一层（全量）：整轮移除（system/心跳/机械回复）+ 内容改写（剥元数据/时间戳/`<think>` 标签）+ **敏感脱敏**（sk-xxx / Bearer / AKIA / 路径）+ 语言检测
- 第二层（窗口级）：探索性工具调用移除 + **tool_result 60/30 截断**（头 60% + 尾 30%，失败完整保留）

**优势**：
- **降噪**：不让 filler / 心跳 / 机械回复进 LLM 提取阶段
- **零成本**：规则层先过滤，省大模型 token
- **保留关键信号**：失败 tool_result 完整保留（经验提取的原材料）
- **隐私兜底**：敏感数据在进 LLM 前就被脱敏

**提升任务完成度**：
提取器看到的证据**只有真实有效的执行痕迹**。记忆库不会被噪声污染；失败信号完整 → 下次同类失败可被召回预警。

---

### 2.2 Step 2 — 生成滑动窗口（n=8, m=2, step=6）

**设计原理**：
- 窗口大小 8 轮（1 轮 = user+assistant 一对）= 一个典型"思考-行动-观察"周期
- overlap 2 轮保证跨窗口话题续接
- step=6 = 8-2，滑动向前

**优势**：
- **粒度平衡**：太小会切断话题，太大会吃光 LLM context
- **跨窗续接**：overlap 让边界话题不被割裂
- **冷热启动分档**：冷启 8 turn 保证首批质量；热启 6 turn 降延迟

**提升任务完成度**：
窗口边界 ≈ 任务单元边界 → 后续提取的 description 天然贴合任务语义，不会把"部署"和"写文档"的经验混在一起。

---

### 2.3 Step 3 — 窗口主题抽取（小模型 + DB 缓存）

**设计原理**：
- 每个窗口提取 4 个字段：`topic_keywords` / `topic_tag` / `task_category` / `topic_summary`
- **`topic_summary` 是 v5 核心创新**：15-40 字自然语言句子（不是词标签）
- `procedural_topic_cache` 跨 tick 复用同窗口 LLM 结果
- 零 LLM 降级：LLM 不可用时走 TF 关键词兜底

**优势**：
- **跨 session 语义对齐**：词标签粒度粗（"API 调试"/"接口测试"/"HTTP 排错"在不同 session 会漂移），**一句话让 LLM 稳定对齐到主题本质**
- **调用成本可控**：缓存命中直接跳过 LLM；v5 水位未推时同窗口可能复查多次，缓存显著降本
- **永不阻塞**：LLM 故障时走 TF 降级路径

**提升任务完成度**：
`topic_summary` 作为下游 Fb/TP/TSE 的 description 锚点 → **同类经验跨 session 归集到同一条**，dedup 命中率提升 → 记忆库更简洁、检索更精准。

---

#### 2.3.1 为什么用"主题"作为锚点

提取管线里有多个可选锚点：tool 序列、task_tag、session ID、file path、时间窗、user 原句⋯⋯v5 选择 **topic（一句话主题摘要）** 作为贯穿 Step 3 → Step 4 → Step 5 的统一锚点。好处如下：

**1. 粒度天然对齐任务单元**

| 候选锚点 | 粒度问题 |
|---------|---------|
| tool 序列 | 太细 —— 同任务可能用多种 tool 组合 |
| task_tag / category | 太粗 —— 一个 category 下可能混不同任务 |
| session ID | 无跨 session 能力 |
| file path / repo | 只对编码任务有效，通用性差 |
| user 原句 | 太具体 —— "帮我写 FastAPI schema" 与 "FastAPI 接口怎么定义" 表达不同但是同主题 |
| **topic_summary（一句话）** | **刚好 = 一个"思考-行动-观察"周期，粒度 ≈ 任务单元** |

**2. 跨 session 语义对齐**

同一主题在不同 session 的用词会漂移（"API 调试"/"接口测试"/"HTTP 排错"），但让 LLM 抽一句 15-40 字 summary 时，**模型倾向于收敛到主题本质**（例如都被归为"调试 FastAPI 接口，排查请求/响应问题"）。

➜ 下游 embedding 在同主题点附近聚集 → **同类经验跨 session 归集到同一条 TSE** 而不是各自开新条目。

**3. Chunk 边界 = 任务边界（Step 4 受益）**

Step 4 拼接状态机用 **topic_keywords Jaccard** 做快判，主题相同则粘在同一 chunk：

- 零 LLM 主路径：`overlap > 0.3` 直接合并，`≤ 0.3` 直接切断
- 只有 0.1–0.3 灰区才调小模型兜底

➜ 主题锚点让 **85%+ 的边界判断不调 LLM**，极大降本。

**4. 下游三类提取共用同一 description 骨架**

Fb / TP / TSE 虽然 content 形态不同，但 description 字段都以 topic_summary 作参考背景：

- Fb 的"修改配置后" ← 主题"部署上线前置检查"给出情境
- TP 的"搜代码时" ← 主题"在 Rust 项目里找 symbol"给出情境
- TSE 的 description 直接派生自主题 summary

➜ 三类共享主题锚点 → **embedding 空间对齐**，检索时无粒度漂移。

**5. 查重命中率提升（Step 5-B / 5-C 受益）**

hybrid 查重（`0.70·cos + 0.18·kw + 0.12·name`）里 cos 部分强依赖 description 稳定：

- 若没有主题锚点，description 字段会因 session 措辞不同而漂移，cos 分值抖动
- 有主题锚点 → 同主题的 description 都靠近同一向量簇 → dedup 阈值 0.85 的命中稳定

➜ 记忆库**不长"同义双胞胎"**（同一规则被多次以不同措辞存入）。

**6. Topic Cache 可跨 tick 复用**

`procedural_topic_cache` 以 window 范围为 key 缓存主题抽取结果。v5 水位未推时同一窗口会被多 tick 复查，主题缓存让**每窗口只调一次 LLM**，长 idle 场景下降本显著。

若锚点是 tool / task_tag，缓存命中率会低得多（tool 序列对同窗口不同 tick 可能不稳定）。

**7. 可解释 + 可调试**

主题是一句人类可读的自然语言，**审计记忆库**时直接看 description 就知道"这条经验属于什么任务情境"。相比 tag id / tool 序列 hash，调试成本低一个量级。

**8. 零外部依赖，通用性强**

主题抽取**只依赖对话内容本身**，不依赖 file path / repo / IDE / 特定 tool 生态。同一套管线可以跨编程、写作、规划、对话多种 agent 场景复用。

**小结**：

| 锚点收益 | 作用环节 | 效果 |
|---------|---------|------|
| 粒度对齐任务 | Step 2–4 分 chunk | chunk ≈ 任务，不粘不裂 |
| 跨 session 对齐 | Step 3 抽主题 | 同类经验归一条 |
| 边界快判 | Step 4 拼接 | 85%+ 零 LLM |
| description 统一骨架 | Step 5-B/C 提取 | Fb/TP/TSE embedding 对齐 |
| 查重命中稳定 | Step 5 dedup | 不长同义双胞胎 |
| 缓存高命中 | Step 3 topic cache | 长 idle 降本 |
| 可审计 | 运维 / 调试 | 自然语言直观 |
| 通用性 | 跨 agent 场景 | 不绑定编码生态 |

**一句话**：主题是**粒度最贴合任务、漂移最小、解释性最强**的锚点，让整条管线在**提取、切分、查重、检索**四个环节共享同一坐标系。

---

### 2.4 Step 4 — 主题拼接状态机（Jaccard + LLM 兜底）

**设计原理**：
- **Jaccard 门控快判**：`overlap > 0.3` → 同主题合并；`≤ 0.3` → 小模型 `JudgeTopicBoundary` 兜底
- chunk 上限：3 窗口 / 2h gap 强制 flush
- chunk 仅存内存，不落 DB

**优势**：
- **零 LLM 快判为主**：高置信区直接合并/切断，灰区才调 LLM
- **任务边界 = chunk 边界**：一个 chunk 对应一个完整任务单元
- **2h gap 兜底**：防长 idle 后跨天数据被粘在一起
- **内存态简化 schema**：不需要 DB 跟踪 chunk 缓存

**提升任务完成度**：
Fb/TP/TSE 的**提取粒度 = 真实任务粒度**。避免"跨任务污染"（如把 A 任务的 tool choice 错归到 B 任务的 TP）。

---

### 2.5 Step 5 — Chunk Flush 核心提取层

这是真正做"程序记忆提取"的环节，4 个子步骤。

#### 2.5.1 Step 5-(A) Task Trace 提取 — v5 最大创新

**设计原理**：
- 把 chunk 原始对话 → 结构化 Markdown 执行轨迹（带主题锚点）
- 明确标注 `User Request` / `Agent Action / Tool Call` / `User Correction` / `Result` 每步
- 作为下游 Fb/TP/TSE **共用**的证据源（一次整理多次消费）

**优势**：
- **降噪一次、复用三次**：下游三个提取器不用各自再解析原始对话
- **显式标注 correction Step**：失败恢复、用户纠正有锚点，不再淹没在消息流里
- **为 v6 workflow 挖掘铺路**：积累同类任务 trace 后可做跨任务模式挖掘

**提升任务完成度**：
下游三类提取的**输入质量都 +1 档**，尤其失败学习信号被显式标注 → 累积型 Fb 有原料。

---

##### 2.5.1.1 为什么要多一层 Task Trace？（而不是直接从对话抽 Fb/TP/TSE）

> 关键问题：多一次 LLM 生成结构化 trace 是额外成本，为什么不让 Fb/TP/TSE 三个 extractor 直接读原始 chunk？

**对照两种设计**：

```
方案 A（直接抽取）                方案 B（v5 方案，经 Task Trace 中间层）
─────────────────                 ─────────────────────────────────
chunk ──┬──► Fb extractor         chunk ──► Task Trace（主题锚点 + 结构化）
        ├──► TP extractor                       │
        └──► TSE extractor                      ├──► Fb/TP extractor
                                                └──► TSE extractor
```

**1. 降噪一次 vs 降噪三次（token 成本下降约 50%）**

直接抽取方案里，Fb/TP/TSE 三个 extractor **各自** prompt 要处理：
- tool_result 冗长输出（即便预过滤做了 60/30 截断仍很长）
- 大量 tool_use JSON 结构噪声
- 交错的 assistant reasoning / user 补充

每个 extractor 要自行识别"哪些是证据、哪些是噪声"。

**粗略 token 估算**（单 chunk ≈ 8000 tokens）：

| 方案 | 输入 token 数 | 输出 token 数 |
|------|--------------|--------------|
| A：直接抽取 | 3 × 8000 = **24000** | 3 × 300 = 900 |
| B：先生成 Task Trace | 1 × 8000 + 2 × 1500 = **11000** | 1 × 1500 + 2 × 300 = 2100 |

➜ **输入 token 砍半**（LLM 成本主要在输入），输出稍增但总成本下降。Task Trace 是**压缩表示**，下游看 1500 token 的结构化轨迹代替 8000 token 的原始对话。

**2. 关键信号显式化：correction / failure 不再被淹没**

原始对话里，用户纠正可能是随手一句 "不对，应该用 rg"，失败信号可能埋在 200 行 tool_result stacktrace 末尾。三个 extractor 各自要**从头搜索这些信号**，搜漏一次就丢一条经验。

Task Trace 阶段**强制标注**：

```markdown
### Step 4: Agent Action
tool: grep
args: { pattern: "...", path: "..." }

### Step 5: User Correction  ← 显式锚点
user_feedback: "不对，应该用 rg，grep 在大 repo 会卡"
corrected_intent: "搜代码用 rg"
```

➜ Fb extractor 只要扫 `User Correction` 节点即可确定证据位置，**零启发式搜索**。信号召回率 +1 档。

**3. 主题锚点贯穿三个 extractor，保证 description 对齐**

Task Trace 的 header 带主题（`topic_summary`），三个 extractor 读到的是**同一份带主题上下文的证据**，不会各自从对话里"感受"出不同的主题：

```
Task Trace Header:
  Topic: 调试 FastAPI 后端项目，涉及 schema 生成与接口测试
  Task Tag: debug-backend
  ...
```

若三个 extractor 各读对话，可能：
- Fb extractor 归纳出 description = "部署场景"
- TP extractor 归纳出 description = "后端开发"
- TSE extractor 归纳出 description = "接口调试"

同一任务的三条记忆 description **互不对齐**，检索时无法联动召回。Task Trace 把主题"钉死"，**三类共享同一情境坐标**。

**4. Extractor prompt 简单化 → 可靠性 +1 档**

直接抽取时，每个 extractor prompt 必须处理三件事：
1. 从对话里判断任务边界
2. 识别关键信号（correction / failure / tool choice）
3. 按类型归纳（Fb / TP / TSE）

Task Trace 方案里 (1)(2) 已由上游完成，extractor 只做 (3)。**单一职责 → prompt 短 → 幻觉少 → 输出稳定**。

➜ 生产环境观察：同任务重复跑时 extractor 输出方差显著小于直接抽取（同一 trace 多次产出相同 Fb/TP/TSE）。

**5. 可缓存、可重跑、可回放**

Task Trace **作为中间产物落盘**（或持久缓存）：

- **prompt 调参**：改 Fb extractor prompt 调优时，无需重读原始对话，直接喂 Task Trace 重跑
- **新类型扩展**：未来加第四类记忆（如 v6 的 workflow 模板）时，已有 trace 可回放抽取
- **跨任务挖掘**：v6 workflow 挖掘需要**结构化可比较**的任务历史，原始对话不可比，Task Trace 可以

➜ **一次整理，多次消费**，且消费路径不限于当前三类 extractor。

**6. 人类可审计（debug 面板）**

Task Trace 是 Markdown，运维/调试时能直接看：

- 某条 Fb 是否归纳正确 → 回看对应 Task Trace 的 User Correction
- 某类任务为何召回率低 → 看 Task Trace 主题抽取是否偏了
- 某次 TSE enrich 为何不合并 → 对比两份 Trace 的 topic / tags

直接抽取方案只能看原始 8000 token 对话，**调试成本高一个量级**。

**7. 防止"同一事件被三份上下文各自解读"产生的幻觉**

LLM 读长对话时会对模糊地带自己"补全"。三个 extractor 各读一次原始对话 = **三次独立幻觉**，可能把同一事件解读成彼此矛盾的结论（Fb 说"用户要求 rg"，TP 说"agent 自发选 rg"）。

Task Trace 是**权威中间表示**：事实在 trace 里已被定性（`User Correction` vs `Agent Action`），下游 extractor 不再二次解释。

➜ 三类记忆之间**不冲突**。

**8. 失败路径的信号密度保真**

v5 的预过滤在 tool_result 失败时**完整保留**（不做 60/30 截断）。直接抽取方案里，这段长失败 log 会重复出现在三个 extractor 的 prompt 里，**token 浪费 ×3** 且每次都要求 extractor 忽略无关栈信息。

Task Trace 阶段一次性把长失败 log 浓缩成：

```markdown
### Step 7: Result
status: failed
error_class: ECONNREFUSED
recovery_action: 改用 127.0.0.1:9000
outcome: success
```

➜ 失败信号**结构化保真 + token 压缩**两不误。

---

**成本 / 收益对账**：

| 维度 | 直接抽取（方案 A）| 经 Task Trace（方案 B）|
|------|-----------------|---------------------|
| LLM 调用次数 | 3 | 3（1 trace + 2 extractor）|
| 输入 token 总量 | 24000 | 11000（**-54%**）|
| correction 信号召回 | 启发式搜索，可能漏 | 显式锚点，不漏 |
| 三类 description 对齐 | 各自归纳，可能漂移 | 共享主题锚点 |
| Extractor 可靠性 | 三职责混合，易幻觉 | 单一职责，稳定 |
| 可缓存/回放 | ✗（原始对话） | ✓（结构化中间产物）|
| 可审计 | 读 8000 token 对话 | 读 1500 token Markdown |
| 扩展新记忆类型 | 需重读对话 | 回放 Trace 即可 |
| 总 LLM 成本 | 基准 | ≈ 基准 × 0.55 |
| 输出质量 | 基准 | 基准 +1 档 |

**一句话**：Task Trace 是把"**理解对话**"这件事从 3 次下放成 1 次，把"**做归纳**"这件事保留给 3 个单一职责 extractor。结果是**更便宜 + 更准 + 更可调试 + 更可扩展**——这是典型的"提取侧多一层结构化，下游全链受益"的 ETL 设计。

---

#### 2.5.2 Step 5-(B) Fb/TP 提取 + 两层查重 + 候选池

**设计原理**：
- **一次大模型调用同时产 Fb + TP**（共享 Task Trace 上下文）
- 两层查重：
  - **Layer 1（active 主表）**：hybrid score `0.70·cos + 0.18·kw + 0.12·name`
    - ≥ 0.85 → 直接 merge
    - < 0.50 → 进 Layer 2
    - 0.50-0.85 → 小模型灰区决策
  - **Layer 2（候选池）**：`kind+name` 精确匹配 + `support_score` 累积 + 动态阈值升格
- **零 LLM is_hard 验证**：LLM 标 is_hard=true 时用正则扫否定词+操作词做二次审核

**优势**：
- **强证据快速入库**：用户明确纠正 hybrid 高分直接 merge，不用等 N 次累积
- **弱证据观察期**：agent 自发观察的规律先进候选池
- **三维证据打分**：`0.5·mention + 0.25·recency + 0.25·hard_ratio` 综合**频次 + 新鲜度 + 硬度**
- **阶梯阈值**：早期 0.18 易入、成熟 0.38 难撼 → 敏感度和稳定性平衡
- **零 LLM 二次审核**：防 LLM 把建议误标为硬约束（减少假阳性）
- **一次调用双类**：Fb/TP 合并提取，省一次 LLM

**提升任务完成度**：
- **Fb 硬约束注入 system prompt** → agent 下次直接被阻止重犯
- **TP 在工具选择点注入** → 每次任务省工具决策成本
- **跨 session 累积**能把"反复失败"升格成硬规则，长期越跑越强

---

#### 2.5.3 Step 5-(C) TSE 提取 + Enrich/Learn 决策

**设计原理**：
- `ExtractChunkOverview` 先提任务概览（task_summary / tools / steps / tags）
- `SearchTseForChunk` hybrid 召回 top-5 候选经验
- `JudgeTaskExperienceMatch` 小模型判断是否归入已有经验：
  - **命中** → `EnrichTaskExperience` 合并 bullet points（语义并集，防退化）
  - **未命中** → `LearnTaskExperience` 新建 → `TseDedupAndSave`
- Embedding：写入用 `name: description`、查询用 `task_summary + context_tags`（两端短语级对齐）

**优势**：
- **Judge 先于 Learn**：避免每次新建，同类经验能累积 enrich
- **bullet_points 累积结构**：经验不是覆盖而是并集，越做越丰富
- **Enrich 有防退化规则**：不丢失历史有效约束（prompt 里明写"两边冲突新条目优先，但已有独有约束保留"）
- **短语级 embedding**：写入/查询量级对齐，向量空间偏差小

**提升任务完成度**：
同类任务**每做一次 TSE 就丰富一次**（bullet 累积）→ 新任务召回时直接获得历经多次打磨的经验清单，不用从零重新摸索。

---

#### 2.5.4 Step 5-(D) MaintainPendingPool

**设计原理**：
- 定时扫描候选池（非触发式）
- `mention_count / recency / hard_ratio` 三维打分
- 动态阈值升格 / 长期低分淘汰

**优势**：
- **跨 session 累积**：mention_count 天然跨任务，不依赖单 session
- **recency 防老化**：老条目自然衰减
- **hard_ratio 奖励高证据**：多次用户纠正 → 更快升格
- **非触发式**：升格不依赖新任务触发，长 idle 后也会工作

**提升任务完成度**：
低置信信号有"**成长路径**"（pending → active），避免一次观察就入库（噪声）或永远淘汰（丢失弱信号）。**长期自学习**不需要人工干预。

---

### 2.6 Step 6 — 水位推进

**设计原理**：
- 水位**只在真实 flush 时推进**
- 未 flush 的 chunk 留内存，下次 tick 从水位重拉重组
- 配合 topic_cache `DeleteByRange` 清理被覆盖区间

**优势**：
- **崩溃安全**：进程重启不丢数据（水位没推，再拉一遍即可）
- **Schema 最简**：不需要 DB 跟踪 chunk 缓存态
- **自愈**：无人工干预

**提升任务完成度**：
生产环境**零数据丢失** + **零运维**。长期稳定运行是记忆系统逐步积累到"真正有用"的前提——任何一次丢数据都可能丢失关键 Fb。

---

### 2.7 全流程串起来看

| Step | 零 LLM？ | 关键创新点 | 核心价值 |
|------|---------|-----------|---------|
| 1 预过滤 | ✅ | 60/30 截断保留失败 | 证据纯、失败信号保真 |
| 2 滑动窗口 | ✅ | overlap=2 跨窗续接 | 提取粒度贴合任务周期 |
| 3 主题抽取 | 部分（含缓存降级） | **topic_summary（一句话）** | 跨 session 语义对齐 |
| 4 主题拼接 | 以零 LLM 为主 | Jaccard 门控 + LLM 兜底 | chunk 边界 = 任务边界 |
| 5-A Task Trace | ❌ | 过程层原子事实，下游共享 | 一次降噪、三次复用 |
| 5-B Fb/TP 双层查重 | 部分 | 候选池 + support_score + 动态阈值 | 强证据快入、弱证据观察 |
| 5-C TSE Enrich/Learn | ❌ | bullet 语义并集防退化 | 经验累积越用越丰富 |
| 5-D 候选池维护 | ❌（小模型） | 非触发式定时扫描 | 跨 session 长期自学习 |
| 6 水位推进 | ✅ | 晚推进保崩溃安全 | 零数据丢失 |

---

### 2.8 六步的整体设计逻辑

v5 的提取管线遵循**"降噪 → 分割 → 锚点 → 提取 → 观察 → 稳定"**六个环节，每步处理一个具体问题，且每步都有**"零 LLM 主路径 + LLM 兜底"**或**"LLM 主路径 + 零 LLM 校验"**的双保险设计——成本可控、质量可靠、崩溃安全。

**对任务完成度的三大根本贡献**：

1. **避免重犯错**（Fb 硬约束）—— 下次直接被阻止，不再踩同样的坑
2. **节省决策**（TP 工具偏好）—— 每次任务少想一步，工具选择有默认答案
3. **累积智慧**（TSE 越用越丰）—— 同类任务从笨拙到熟练，经验条目随次数 enrich

---

## 三、GSPD 针对性解决的问题（基于横向对比）

对照 [Procedural-Memory-System-Comparison.md](./Procedural-Memory-System-Comparison.md) 里 10 个系统的劣势，GSPD 的设计取舍是**针对每个具体痛点选择对应的工程解**。下面按痛点归类。

---

### 3.1 痛点 A：单类记忆无法兼顾"硬约束 / 泛化 / 累积"

**其他系统的问题**：

| 系统 | 观察到的缺陷 |
|------|------------|
| Mem0 | 只有 facts，**无通用化**，"跨任务无法迁移" |
| ReMe | 单一 experience 类，建议性弱注入，agent 可能忽略；**强弱混在一起** |
| MemOS / AutoSkill / Hermes | 只有 skill 一类，**无动作级硬约束**，无法覆盖"修改配置后必须重启"这种横向规则 |
| Skill-Creator | 精品 skill 但**无轻量教训层**，所有知识都需经人工打磨 |
| pskoett SI | 虽有三级漏斗但**Learning 层无结构化强度**，强/弱规则混在同一 Markdown |

**GSPD 解法 → Fb/TP/TSE 三类分层**（§1.2）：

- **Fb（动作级硬约束）**：覆盖 Mem0/MemOS/Hermes 缺失的"跨任务横向规则"
- **TP（工具场景级偏好）**：覆盖纯技能类系统没有的"工具决策点"层
- **TSE（任务级软建议）**：承担 ReMe/XSkill 的经验建议角色

三类**强度 / 粒度 / 生命周期 / 证据门槛独立**（§1.2.1–1.2.5），合成一类的代价是"要么过强要么过弱"。

---

### 3.2 痛点 B：新任务（从未做过）完全无记忆可用

**其他系统的问题**：

- Mem0/ReMe/MemOS TS/AutoSkill **按任务相似度检索** → 新任务 task_summary 无匹配 → 召回空集
- XSkill 同样基于任务对齐，跨任务类型迁移实验显示 2-3 分提升已是上限
- Hermes/Skill-Creator 的 skill 粒度也是任务级，**没有动作级横向锚点**

**GSPD 解法 → Fb/TP 跨任务触发**（§1.2.2）：

- Fb 按**动作触发条件**（"修改配置后"）召回，不管是哪个任务只要包含该动作就命中
- TP 按**操作场景**（"搜代码时"）召回，同样跨任务

➜ 首次做"Rust 项目加 CI"这种**从未见过的任务**，Fb/TP 仍能命中——其他系统的单类设计做不到。

---

### 3.3 痛点 C：Description 粒度漂移导致检索/去重失败

**其他系统的问题**：

| 系统 | 问题 |
|------|------|
| Mem0 | verbatim 保留 URL/ID/参数 → 换个任务措辞即无命中 |
| ReMe | description 向量按 `when_to_use` 建索引但**粒度混合**（有的句子有的短语）|
| OpenViking | 8 类单标签**吃掉跨类语义**；score 传播 α=0.5 过激，深层 fact 几乎必降分 |
| pskoett | Pattern-Key 需人工约定，**两 agent 分别用不同 key 仍会重复** |

**GSPD 解法**：

1. **主题锚点贯穿全管线**（§2.3.1）—— 粒度统一、跨 session 语义对齐、embedding 不漂移
2. **TSE 用完整 description 句**（§1.8）—— LLM 归纳收敛到主题本质，新任务不因措辞失配
3. **三类 description 风格分档**（§1.2.3）—— Fb 动作短语 / TP 工具场景 / TSE 描述句，空间不互相挤占

---

### 3.4 痛点 D：Extractor 重复消费原始对话，成本 × 3

**其他系统的问题**：

- MemOS TS：5-9 次 LLM 调用，每次都要从原始对话里重新解析
- AutoSkill：5-8 次/轮，开销大
- Skill-Creator：2-3M tokens/技能（极端案例）
- 多数系统的多个 extractor 都是**各自读原始 chunk**，任务边界识别 + 信号定位 + 归纳三件事混在一个 prompt

**GSPD 解法 → Task Trace 中间层**（§2.5.1.1）：

- 一次 LLM 把原始 chunk 压成**结构化 Markdown 轨迹**（带主题锚点 + 显式标注 correction）
- 三个 extractor 读轨迹而非原始对话，**输入 token 从 24000 降到 11000（-54%）**
- correction / failure 被显式标为锚点，不再靠 extractor 启发式搜索
- Task Trace 可缓存可回放，加新记忆类型无需重读对话

---

### 3.5 痛点 E：强证据与弱证据同等对待

**其他系统的问题**：

- Mem0：ADD only，无证据分级 → 一次性入库
- ReMe：**单一阈值**（score < 0.3 剔除），要么一次入库要么直接丢
- MemOS：0-10 评分但无"观察期"概念
- Hermes：LLM 自觉 + 无自动门控 → 弱模型可能漏记
- pskoett：Recurrence ≥ 3 是**硬阈值**，未满 3 次的信号完全无处可去

**GSPD 解法 → 两层查重 + 候选池**（§2.5.2）：

- **Layer 1（active）**：用户明确纠正 hybrid 高分直接 merge，不用等 N 次
- **Layer 2（pending pool）**：agent 自发观察的弱规律先进候选池，跨 session 累积 mention_count 后升格
- **三维 support_score**（mention / recency / hard_ratio）+ **阶梯阈值**（0.18 → 0.38）
- **零 LLM is_hard 二次审核**：正则扫否定词 + 操作词，防 LLM 误标硬约束

➜ 强证据快入、弱证据有成长路径、噪声自然衰减——三个都做到。

---

### 3.6 痛点 F：失败信号被截断或丢弃

**其他系统的问题**：

- AutoSkill：显式 **`successOnly=true`**，不学失败
- Mem0/MemOS：无失败轨迹机制
- OpenViking：ToolPart 输出**截断 500 字符**，长日志关键信息会丢
- Skill-Creator：基线对比但需要构造 without_skill 路径

**GSPD 解法 → 60/30 截断 + 显式标注**（§2.1, §2.5.1）：

- **失败 tool_result 完整保留**（不做 60/30 截断）→ 经验提取原材料不丢
- Task Trace 阶段**显式标注 `User Correction` / 失败恢复节点**
- 累积型 Fb 用"反复失败"作硬证据升格条件
- error recovery 动作沉淀成 TP/Fb 而非被忽略

---

### 3.7 痛点 G：演进成本随规模爆炸

**其他系统的问题**：

| 系统 | 演进成本问题 |
|------|------------|
| Hermes GEPA | 每代 N 次 LLM 变异 + judge + benchmark，**代数叠加爆炸** |
| MemOS/AutoSkill SkillEvo | replay 成本随 skill 数 × 代数，遗传算法不可规模化 |
| Skill-Creator | 60 × `claude -p` × 5 轮描述优化，单技能 2-3M tokens |
| OpenSpace | 3 触发器全面但架构复杂度 ~7800 行 |
| pskoett | 演进摊入 agent 自身推理，**每轮 +500-2k tokens 隐性累积** |

**GSPD 解法 → 提取侧做重，演进靠规则**（§2.5.4）：

- 生成时就把 description 写好（§1.8）→ 减少未来的 re-rewrite
- **MaintainPendingPool 定时扫描**（非 LLM）→ mention_count / recency / hard_ratio 规则升格
- Enrich 是**语义并集**（bullet 累积），不是重新生成 → 无昂贵 SkillEvo/GEPA
- 老化靠三维分数**自然衰减**，不需遗传算法

➜ **学习成本**中等、**演进成本**近零（规则主导）、**长期总成本**可控。

---

### 3.8 痛点 H：注入粒度粗、时机单一

**其他系统的问题**：

- Mem0/MemOS：**全文注入** → token 爆炸
- ReMe：统一 RewriteMemory 建议，弱注入（Agent 可能忽略）
- Hermes：三级加载 L0/L1/L2 但仍是**skill 粒度**，无动作级钩子
- Skill-Creator：读完整 SKILL.md，最高注入 token
- OpenViking：Agent 主动 read，无主动注入 → 容易漏

**GSPD 解法 → 三类三时机分别注入**（§1.3 优势 1）：

- **Fb 注入 system prompt** 作硬约束（动作决策前触发）
- **TP 在工具选择点注入**（即将调工具时触发）
- **TSE 在任务开始时注入**（参考性，非强制）

➜ **粒度对齐触发时机**：硬约束不会被建议稀释、建议不会被强约束绑架。

---

### 3.9 痛点 I：解耦 vs 耦合二元对立

**其他系统的问题**：

- **解耦派**（Mem0/ReMe/MemOS/AutoSkill）：稳定可靠但**智能上限受规则限制**
- **耦合派**（Hermes/Skill-Creator/pskoett）：智能度高但**成本贵、强依赖 agent 能力**
- **委派派**（OpenSpace）：宿主 0 负担但**完全黑盒失去可见性**

**GSPD 解法 → 解耦为主 + 按需 pull 钩子**：

- 提取管线**完全解耦**（后台异步 pipeline，agent 不感知）→ 稳定 + 可移植
- 注入阶段三类按不同时机触发（Fb system prompt 推送 / TP 工具决策点 / TSE 任务开始）→ **混合 Push + Pull**
- 不依赖 agent 强能力（弱模型也能跑）+ 不依赖 agent 理解记忆系统
- 关键点在**提取侧把结构打好**，注入侧可以按场景选 Push/Pull

---

### 3.10 痛点 J：崩溃即丢数据

**其他系统的问题**：

- OpenViking：Phase 2 抽取 **fire-and-forget，无重试无 SLA，崩溃即永久丢失**
- MemOS/AutoSkill：后台异步但无水位机制
- pskoett：纯 Markdown 靠 git 兜底

**GSPD 解法 → 水位晚推 + chunk 重组**（§2.6）：

- **水位只在真实 flush 时推进**，未 flush chunk 留内存
- 进程重启从水位重拉重组 → **零数据丢失**
- `procedural_topic_cache.DeleteByRange` 清理被覆盖区间 → 自愈

---

### 3.11 痛点 K：无升格路径，知识停留在原始层

**其他系统的问题**：

- Mem0：ADD only，**无升格**
- ReMe/MemOS：只有合并，无"弱 → 强"提升路径
- Hermes：LLM 自主 patch/edit，无级别概念
- pskoett：有三级漏斗但**需 agent 自评"是否广泛适用"，主观性强**

**GSPD 解法 → 候选池 → active 的客观升格**（§2.5.2, §2.5.4）：

- 弱信号进候选池，满足**客观三维阈值**（mention + recency + hard_ratio）自动升格
- 动态阈值（0.18 → 0.38）**早期易入、成熟难撼** → 敏感度和稳定性平衡
- 升格不依赖 agent 主观判断，完全规则驱动

---

### 3.12 总结：GSPD 的定位

| 对比维度 | GSPD 的选择 | 对应痛点 |
|---------|------------|---------|
| 知识分层 | **三类独立**（Fb/TP/TSE）| A 单类无法兼顾 |
| 检索锚点 | **动作 / 工具 / 任务三级** | B 新任务无召回 |
| Context 形态 | **主题锚点 + description 句** | C 粒度漂移 |
| Extraction 架构 | **Task Trace 中间层** | D 重复消费对话 |
| 证据门控 | **两层查重 + 候选池** | E 强弱一刀切 |
| 失败学习 | **60/30 截断 + 显式锚点** | F 失败信号丢失 |
| 演进机制 | **规则主导 + Enrich** | G 演进成本爆炸 |
| 注入时机 | **三类三时机** | H 粒度粗 |
| 耦合度 | **解耦 pipeline + 按需 pull** | I 二元对立 |
| 崩溃恢复 | **水位晚推 + chunk 重组** | J 数据丢失 |
| 升格路径 | **三维客观阈值** | K 无升格 |

**一句话**：GSPD 不追求单项极致，而是**把各系统各自解决的局部痛点在一套分层架构里聚合**——用**三类分层 + 主题锚点 + 中间层 Task Trace + 两层查重 + 规则演进**的组合拳，把"跨任务泛化、强度分级、成本可控、崩溃安全、解耦可移植"这几个通常互相冲突的目标**同时达成到可生产的水平**。代价是架构复杂度介于 Mem0（最简）和 OpenSpace（最全）之间，学习曲线中等——但每一项复杂度都对应一个在横向对比中被其他系统各自踩过的坑。

---

## 四、LLM Prompt 清单（基于实际实现）

> 源目录：`/home/x/桌面/gspd_memory/gspd_memo/src/prompts/files/procedural/`
> 每个 prompt 提供中英双版本（`.zh.md` / `.en.md`）。本章**完整抄录** `.zh.md`。
> **v5 更新**：新增 `generate_task_trace`（Step 5-A），并把 Fb/TP/TSE 提取链
> （`extract_exp_tp` / `learn` / `evolve`）改为**以 Task Trace 为主要证据源**，
> 兑现 §2.5.1 设计文档的中间层架构。

---

### 4.1 Prompt 总览表（10 个）

| 编号 | 文件名 | Step | 调用时机 | 证据源 |
|------|-------|------|---------|-------|
| **P1** | `topic_extract` | Step 3 | 每窗口 1 次 | 锚点轮次 + 窗口对话 |
| **P2** | `topic_boundary` | Step 4 | Jaccard 灰区 | 两片段锚点轮次 |
| **P3** | `generate_task_trace` | Step 5-A | 每 chunk 1 次 | Chunk 原始对话 |
| **P4** | `overview` | Step 5-C 开头 | 每 chunk 1 次 | 对话片段 |
| **P5** | `extract_exp_tp` | Step 5-B | 每 chunk 1 次 | **Task Trace** + 原文(可选) |
| **P6** | `dedup_decide` | Fb/TP 查重判决 | 候选对判决 | 两条条目 |
| **P7** | `merge` | Fb/TP 合并执行 | decide=merge 时 | 两条条目 |
| **P8** | `judge` | Step 5-C 匹配 | TSE 召回后 | task_summary + 候选列表 |
| **P9** | `learn` | Step 5-C 新建 | judge=0 时 | **Task Trace** + 原文(可选) |
| **P10** | `evolve` | Step 5-C 丰富 | judge≥1 时 | **Task Trace** + 原文(可选) |

**调用拓扑**：

```
Step 3:  topic_extract（窗口→主题 4 字段）
Step 4:  topic_boundary（灰区判同/异主题）

Step 5-A（新增的中间层）:
  generate_task_trace（chunk 原始对话 → 结构化 Markdown 执行轨迹）
      │
      └──► task_trace 作为下游三条提取链的统一证据源

Step 5-B（Fb/TP 提取链）:
  extract_exp_tp（Task Trace → {feedback[], tool_preference[]}）
    └─ dedup_decide（候选 vs 已有 → merge/add）
         └─ merge（若 merge：合并两条 Fb 或 TP）

Step 5-C（TSE 提取链）:
  overview（对话→{task_summary, tools_used, key_steps, context_tags}）
    └─ judge（selectedIndex=0 无匹配 / 1..N 匹配第 N 条）
         ├─ =0 → learn（Task Trace → 新 TSE，含多路径自检子分支）
         └─ ≥1 → evolve（Task Trace + 已有 TSE → enrich）
```

---

### 4.2 P1 — `topic_extract.zh.md`（窗口主题提取）

```markdown
# 窗口主题提取

你是一个对话主题分析专家。从滑动窗口的对话中提取主题元数据，用于后续的主题边界判断
和 Chunk 积累。

## 背景说明

当前窗口由 n=10 轮对话组成，首尾各有 m=2 轮与相邻窗口重叠。
**锚点轮次**（anchor_turns）是窗口中间的非重叠部分，代表本窗口最核心的独有内容；
首尾重叠轮次仅作上下文补充，不作为主要提取依据。

## 字段说明

**topic_keywords**（3-8 个）
- 优先从锚点轮次的用户消息中提取
- 聚焦任务对象和操作动词（如"Docker 部署"、"SQL 优化"、"重构认证模块"）
- 过滤停用词（好的/继续/帮我/可以/这个）和短词（< 2 字符）
- 不同表述指同一对象时只保留一个

**topic_tag**（1-3 个）
- 概括本窗口的任务域，粒度比 keywords 更粗
- 示例："前端调试"、"数据库迁移"、"API 集成"、"文档撰写"

**task_category**（单选）
- `coding`：写代码、调试、重构、代码审查
- `planning`：方案设计、架构讨论、需求分析、任务拆解
- `debugging`：排查错误、日志分析、环境问题、修复 bug
- `writing`：文档撰写、内容生成、翻译、总结
- `other`：以上均不符合

**topic_summary**（一句话，15-40 字）
- 自然语言描述窗口中"谁在做什么"，供下游 feedback / tool_preference
  提取时作为 context 锚点，稳定归类
- 包含具体对象和动作，例："用户在 production 环境调试 Docker
  部署，卡在 registry 推送权限"
- 纯闲聊 / 无明确任务 → 输出空串 ""

## 提取原则

1. 以锚点轮次为主要依据；完整窗口仅作上下文参考
2. 关键词来自用户消息，不从 assistant 回复中直接摘取
3. 没有明确任务时（如纯闲聊）→ topic_keywords 返回空列表，
   topic_tag 返回 ["general"]，task_category 返回 "other"，
   topic_summary 返回空串

## 输出格式

只输出 JSON：

{
  "topic_keywords": ["关键词1", "关键词2", "关键词3"],
  "topic_tag": ["主题标签1", "主题标签2"],
  "task_category": "coding | planning | debugging | writing | other",
  "topic_summary": "一句话 15-40 字，用户在做什么"
}

[User]
锚点轮次（主要依据，窗口非重叠核心内容）：
{{anchor_turns}}

完整窗口（上下文参考）：
{{window_messages}}
```

---

### 4.3 P2 — `topic_boundary.zh.md`（主题边界判断）

```markdown
# 主题边界判断

你是一个对话主题连续性分析专家。判断两段对话是否属于同一个任务主题，决定是否将它们
合并为同一个 Chunk。

## 背景说明

相邻窗口之间有 m=2 轮重叠。**锚点轮次**是每个窗口去掉首尾重叠后的非重叠核心内容，
代表该窗口最真实的主题，是判断的首要依据。

## 同主题 / 不同主题 的判断标准

**同主题**：
- 任务对象相同（同一文件、同一功能、同一问题）
- 后续内容是前续的延伸、深化、修正或补充
- 关键词不完全重叠但语义属于同一工作流（如"写测试"→"修复测试失败"仍是同主题）

**不同主题**：
- 任务类型切换（如从"写代码"转向"写文档"）
- 工作对象切换（如从"前端组件"转向"数据库 schema"）
- 用户明确结束一个话题并开启新话题（"好，这个搞定了，现在来做..."）
- 片段 B 的锚点轮次与片段 A 毫无关联

## 判断策略（按顺序执行）

1. **锚点内容对比**（最可靠）：比较两段锚点轮次，看任务对象和操作是否连续。
2. **边界消息对比**：片段 A 最后一条用户消息 vs 片段 B 第一条用户消息
   （去重叠后），是否有明显的话题切换信号。
3. **关键词参考**（辅助）：词汇重叠低不等于主题不同（可能只是表述偏移），
   不单独决定结果。

## 输出格式

只输出 JSON：`{"same_topic": true|false, "reason": "..."}`

[User]
片段 A 关键词：{{chunk_keywords}}
片段 B 关键词：{{next_window_keywords}}

片段 A 锚点轮次（核心内容）：
{{chunk_anchor_turns}}

片段 B 锚点轮次（核心内容）：
{{next_anchor_turns}}

片段 A 最后一条用户消息：{{chunk_last_user_msg}}
片段 B 第一条用户消息（去重叠后）：{{next_first_user_msg}}
```

---

### 4.4 P3 — `generate_task_trace.zh.md`（任务轨迹提取，v5 新增）⭐

```markdown
# 任务轨迹提取

你是一个任务执行轨迹结构化专家。将一段原始对话（含 user 消息 / assistant 回复 /
tool_use / tool_result）浓缩成一份结构化 Markdown 执行轨迹（Task Trace），为下游
feedback / tool_preference / task_specific_experience 的提取提供**统一证据源**。

## 背景说明

本 prompt 是程序记忆提取管线的**中间层**：下游三个 extractor（extract_exp_tp /
learn / evolve）不再各自解析原始对话，而是共用本 prompt 产出的 Task Trace。目标：

1. 消除对话噪声（闲聊/填充词/tool_result 冗长输出），只保留任务执行骨架
2. **显式标注关键信号**（User Correction / Failure / Recovery），不再埋在消息流里
3. 保留时序与因果关系，便于下游归纳
4. 合并同一逻辑步骤的多次重试，避免下游把重试错认为多步骤

## 结构 Schema

输出**必须是结构化 Markdown**，按以下格式组织：

# Task Trace
topic: <对话主题，来自 topic_summary>
task_tag: <任务域标签>

## User Intent
<一句话描述本段对话用户的核心诉求，≤ 40 字>

### Step 1: <步骤短名，4-8 字>
actor: user | assistant
action_type: request | tool_call | correction | summary | observation
content: <关键内容，≤ 60 字>
tool: <若 tool_call，记录工具名，否则省略此字段>
args: <若 tool_call，记录关键参数摘要，不要 dump 全文>
result: <若 tool_call，记录关键结果，≤ 60 字>
status: success | failed | partial
attempts: <若失败→重试的同一步骤，记录尝试次数；单次成功省略此字段>
recovery: <若失败后成功恢复，记录恢复手段，≤ 40 字>

### Step 2: User Correction                ← 显式锚点
actor: user
action_type: correction
user_feedback: <用户纠正原文，≤ 40 字>
corrected_intent: <纠正后的正确做法，≤ 40 字>
target_step: <被纠正的 Step 编号，如 Step 1>

### Step 3: ...
(继续)

## Outcome
status: success | partial | failed
summary: <本次任务执行结果，≤ 60 字>
key_findings: <关键产出或结论，可选>

## 提取规则

### 1) 重试合并（关键）

- 同一逻辑步骤的多次尝试**合并为一个 Step**
- 合并条件（需同时满足）：
  - 相同 tool
  - 相同核心意图（操作同一对象）
  - 连续发生（中间无其他实质步骤插入）
- 合并后 `attempts: N`，并在 `recovery:` 记录最终成功的手段
- **不要**把每次 tool_call 列为独立 Step，否则下游会错把重试当多步

### 2) User Correction 显式锚点

- 用户用否定/切换方案词（不要/别/错了/换成/应该/不对/改用）时**单独成 Step**
- `action_type: correction`
- 必须记录：
  - `user_feedback`：用户原话
  - `corrected_intent`：纠正后应做的事
  - `target_step`：指向被纠正的 Step 编号
- 这是下游 feedback extractor 的关键证据锚点

### 3) Failure + Recovery 标注

- `status: failed` 且随后有成功恢复 → 在同一 Step 记 `recovery:`
- `status: failed` 且未恢复（放弃/任务失败）→ 只记 failed，不记 recovery
- 失败信息不要丢：错误类型 + 关键字段 **必须保留**（下游累积型 Fb 靠此升格）

### 4) tool_result 压缩

- **保留**：状态码、错误类型、关键字段、关键数值、文件路径
- **丢弃**：长 stacktrace 中间段、冗长日志、重复结构、大 JSON 数组 dump
- 成功：≤ 60 字摘要
- 失败：**保留完整错误信息**（对下游学习至关重要，不做 60/30 截断）

### 5) 噪声过滤

- **丢弃**：纯闲聊、"好的"/"继续"/"ok" 等弱应答、assistant 的空洞承接词
- **保留**：任何带意图或判断的 user 消息、assistant 有实质动作的段落
- 判断标准：去掉后是否丢失任务理解？

### 6) 不发明内容

- 只抽取对话中真实存在的事件
- 不补充"通常还需要"的步骤
- 不推测用户未明说的意图
- 保持时序真实

### 7) 去标识化（轻度）

- Task Trace 本身**不做强去标识化**（下游 extractor 负责）
- 但敏感信息（密钥/token/密码）必须打码
- 文件路径、项目名可原样保留，便于下游做因果推理

## 空结果处理

纯闲聊无任务 → 输出最小结构：
  # Task Trace
  topic: <主题>
  ## User Intent
  无明确任务（闲聊/填充）
  ## Outcome
  status: na

下游 extractor 见此格式会自动跳过。

## 输出格式

**只输出 Markdown Task Trace 正文**（如上 Schema 所示），不要 JSON 包裹，不要前后
解释文字。下游管线会直接把输出作为字符串存入 `{{task_trace}}` 占位符。

[User]
Topic Summary（来自 Step 3 主题抽取）：
{{topic_summary}}

Task Tag：
{{task_tag}}

Chunk 原始对话：
{{chunk_messages}}
```

---

### 4.5 P4 — `overview.zh.md`（工作概览提取）

```markdown
# 工作概览提取

你是一个对话分析专家。请从提供的对话片段中提取结构化工作概览，
用于后续检索已有任务经验和判断是否需要学习新经验。

## 任务目标

1. 理解用户在该对话片段中试图完成的**核心任务**是什么。
2. 识别用户或 agent 使用的**关键工具/命令**。
3. 提炼出**关键操作步骤**（3-5 条）。
4. 提取 2-4 个**场景标签**（context_tags），用于描述任务所属领域/场景。

## 字段说明

- **task_summary**：一句话概括用户要完成的核心任务。聚焦意图，不要罗列细节。
  - 示例："将 Node.js 应用容器化并部署到生产环境"
  - 反例："用户和助手讨论了 Docker 和端口映射的问题"（太泛）

- **tools_used**：对话中实际涉及的工具、框架或命令列表。
  - 示例：["Docker", "npm", "Node.js"]

- **key_steps**：用 3-5 条简短语句描述关键操作节点，不是每句话都记。
  - 示例：["编写 Dockerfile", "构建镜像并测试", "配置端口映射", "部署到服务器"]

- **context_tags**：2-4 个简短标签，描述任务的领域/场景/技术栈。
  - 示例：["容器化", "部署", "Node.js", "生产环境"]
  - 反例：["帮我看看"]（太口语化）、["任务"] （太宽泛）

## 输出格式

只输出 JSON，不要任何解释：

{
  "task_summary": "<一句话描述用户要完成的任务>",
  "tools_used": ["<使用的工具/命令列表>"],
  "key_steps": ["<关键操作步骤，3-5 条>"],
  "context_tags": ["<场景标签，2-4 个>"]
}

## 注意事项

- 只提取 HOW（执行过程、工具选择、步骤约束），不提取 WHAT（具体内容、一次性数据）。
- 如果对话中没有明确任务或无可复用流程，返回空对象 `{}`。
- context_tags 要通用可复用，避免案例特定实体（如具体项目名、文件名）。
```

---

### 4.6 P5 — `extract_exp_tp.zh.md`（Feedback + Tool Preference 联合抽取，读 Task Trace）⭐

```markdown
# 提取 Feedback 和工具偏好

你是一个程序记忆提取专家。从以下 **Task Trace**（结构化任务轨迹）中提取
feedback 和 tool_preference。Task Trace 是上游 `generate_task_trace` prompt 产出
的执行骨架，已经标注好了 User Correction / Failure / Recovery 等关键锚点，请**以
Task Trace 为主要证据源**，Full Conversation 仅在需要核对原文时参考。

## 知识类型定义

**feedback**（错误更正记忆）— 错误发生后形成的硬约束行为规则，agent 必须遵循：
- 来源一（用户纠正）：用户亲口说"不要用 X"、"错了，应该 Y"、"顺序不对"等
  → 在 Task Trace 中体现为 `action_type: correction` 的 Step
- 来源二（agent 纠错）：agent 执行失败后自行换方案并成功恢复的路径
  → 在 Task Trace 中体现为 `status: failed` 且 `recovery:` 非空的 Step
- 内容：工具选择纠正、步骤/顺序纠正、失败恢复路径、风格/格式纠正
- 优先级最高
- 格式："[被纠正的行为] → [正确做法]" 或 "[错误情境] 时，改用 [恢复做法]"
- 长度：description ≤ 50 字
- is_hard = true
- source_user_correction = true（用户纠正）/ false（agent 自身纠错）

**tool_preference**（工具偏好）— 同类任务上的稳定工具选择倾向：
- 目标：减少决策开销，提升执行效率
- 格式："做 [任务类型] 时，优先用 [工具]"
- 注意：互补工具组合（"用 A 后配合 B"）→ 不归此类，应忽略
- 长度：description ≤ 50 字
- is_hard = true（如果来自用户纠正）或 false（如果来自 agent 自发规律）

## 分析框架（基于 Task Trace 锚点）

**1. 用户纠正分析 → feedback**

- 扫描 Task Trace 里所有 `action_type: correction` 的 Step
- 对每个 correction Step：
  - 读取 `user_feedback`（纠正原话）+ `corrected_intent`（正确做法）
  - 读取 `target_step`，理解被纠正的是什么行为
  - 生成 feedback：is_hard=true，source_user_correction=true
- 若纠正涉及**工具选择**（`corrected_intent` 含"用 X"/"改用 Y"等工具替换）：
  - 同时产出 tool_preference（is_hard=true）

**2. 失败→恢复分析 → feedback**

- 扫描 Task Trace 里 `status: failed` 且 `recovery:` 非空的 Step
- 对每个这样的 Step：
  - 判断该错误是否**可能在下次同类任务中重现**（不是一次性环境偶发）
  - 可泛化 → feedback：is_hard=true，source_user_correction=false
    - context：基于 Step 的 tool + failure pattern 派生
    - description："出现 [错误模式] 时，改用 [recovery]"
  - 一次性/环境偶发 → 不提取

**3. 工具选择模式分析 → tool_preference**

- 统计 Task Trace 里 `action_type: tool_call` 的 Step：
  - 同一 `tool` 在同类 Step（相似 content）是否反复出现（≥ 3 次）？
    → tool_preference（is_hard=false）
  - 是否有 correction Step 明确要求"优先 X 不用 Y"？
    → tool_preference（is_hard=true）

## 提取规则

- **只提取 Task Trace 中有明确证据支持的内容**（不发明，不从 Full Conversation
  "脑补"未在 trace 中结构化的信号）
- **每条 feedback/tool_preference 必须指向 Task Trace 的某个 Step 作为证据**
  （内部校验，不必输出）
- 去标识化：用占位符替代具体文件名/项目名/用户名
- 一次性、不可泛化的操作 → 不提取
- 纯闲聊/无操作内容（Outcome.status=na）→ 直接返回 `{"items": []}`

## 证据来源优先级

1. **Task Trace**（主要，必须引用）
2. Full Conversation（仅用于：Task Trace 语义歧义时回看原文）

不要把 Full Conversation 当作独立证据源 —— 任何未被 Task Trace 结构化的信号都应
视为噪声或已被上游判定为无关。

## 输出格式

只输出 JSON：

{
  "items": [
    {
      "kind": "feedback" | "tool_preference",
      "name": "<简短标题，5字以内>",
      "context": "<适用场景标签，1-3个>",
      "description": "<描述，≤50字>",
      "is_hard": true | false,
      "source_user_correction": true | false
    }
  ]
}

is_hard=true：用户明确纠正（correction Step）或用户指定必用
is_hard=false：agent 自身推断（failed+recovery / 统计性工具偏好）
source_user_correction=true：来自 Task Trace 的 correction Step
source_user_correction=false：来自 Task Trace 的 failed+recovery Step 或 tool 统计

无可提取内容 → `{"items": []}`

[User]
Task Trace（主要证据，来自 generate_task_trace）：
{{task_trace}}

Full Conversation（仅用于歧义核对，不作独立证据源）：
{{full_conversation}}
```

---

### 4.7 P6 — `dedup_decide.zh.md`（Fb/TP 去重决策）

```markdown
# 程序记忆去重决策

判断新提取的条目应该与已有条目合并（merge）还是作为独立条目新增（add）。

## 决策流程

0) 能力重叠硬门控：如果两条描述的是同一核心能力，不得 add，只能 merge。
1) 能力族检查：完全不同的任务领域 → add。
2) 4 轴身份判断（核心任务目标、适用场景/context、硬约束、工具或操作方法）→ 高度重叠则 merge，有实质差异则 add。
3) 不确定 → add（宁可多一条，不误合并）。

只输出 JSON：{"action": "merge" | "add", "reason": "..."}

[User]
已有条目：
  kind: {{existing.kind}}
  name: {{existing.name}}
  context: {{existing.context}}
  description: {{existing.description}}

新提取条目：
  kind: {{candidate.kind}}
  name: {{candidate.name}}
  context: {{candidate.context}}
  description: {{candidate.description}}
```

---

### 4.8 P7 — `merge.zh.md`（Fb/TP 合并执行）

```markdown
# 程序记忆合并专家

将已有条目与新条目合并为一个更完整的程序记忆条目。

## 合并原则

- 语义并集：保留双方所有重要信息，去重融合
- 防退化：不能丢失已有条目中的约束
- 去标识化：移除案例特定细节，保留可移植规则
- RECENCY BIAS：两边冲突时，新条目内容优先
- 不发明内容：只用两边原有信息

只输出 JSON：

{
  "kind": "feedback" | "tool_preference",
  "name": "<保留或优化后的标题>",
  "context": "<场景标签，取两边并集>",
  "description": "<合并后描述，≤50字>",
  "is_hard": true | false
}

[User]
已有条目（历史）：
  kind: {{existing.kind}}
  name: {{existing.name}}
  context: {{existing.context}}
  description: {{existing.description}}
  is_hard: {{existing.is_hard}}

新提取条目（近期）：
  kind: {{candidate.kind}}
  name: {{candidate.name}}
  context: {{candidate.context}}
  description: {{candidate.description}}
  is_hard: {{candidate.is_hard}}
```

---

### 4.9 P8 — `judge.zh.md`(TSE 匹配裁决)

```markdown
# 任务经验匹配裁决

你是一个严格的匹配判断器。判断当前工作窗口是否应该归入已有的某个
task-specific experience。

必须是同一类型的工作任务才算匹配，宽泛或切线相关不算。

规则：
- 仅当任务明确属于某个经验的领域时，输出该序号（1~N）
- 不确定时倾向于匹配（输出最接近的序号），宁可走升级路线也不要重复创建

只输出 JSON：

{
  "selectedIndex": <int>,
  "reason": "<选择原因>"
}

- `selectedIndex`: 0 表示无匹配（应创建新经验），1~N 表示选择第 N 个候选
- 如果候选列表为空，必须返回 0

[User]
当前任务概览：{{task_summary}}
当前标签：{{context_tags}}

候选经验（序号、name、description）：
{{experience_list}}
```

---

### 4.10 P9 — `learn.zh.md`（TSE 新建，读 Task Trace，含多路径自检）⭐

```markdown
# 从 Task Trace 学习任务经验

你是一个任务经验提取专家。将一段**Task Trace**（结构化任务轨迹）提炼为可复用的
task_specific_experience。Task Trace 是上游 `generate_task_trace` prompt 产出的
执行骨架，已经标注好了 Step 序列、User Correction、Failure/Recovery 等关键节点。
请**以 Task Trace 为主要证据源**，Full Conversation 仅在需要核对原文时参考。

## 知识类型定义

**task_specific_experience**（任务经验）— 有助于下次更好完成同类任务的情境化知识：
- 不是严格执行的 SOP，是"遇到 [情境] 时，[推荐做法]"的执行智慧
- **建议性**：agent 召回后自行判断是否采纳
- 内容可以包括：
  1. 工具参数/组合技巧（"用 A 后接 B 效果更好"）
  2. 已安装 skill 的调用场景与组合方式（"做此类任务时优先调用 skill:X" /
     "先用 skill:X 生成草案，再手动调整"）
  3. 常见陷阱与规避方式（"出现该报错时，改用此做法"）
  4. 隐性的执行顺序建议（"先做 X 再做 Y 更稳"）
  5. 从成功/失败路径中 inferred 的有效做法
  6. 用户主动提出的任务上下文依赖/隐性要求（"编排周日行程时，需要先看备忘录和
     待办"）
- description：一句话摘要，≤ 50 字
- bullet_points：以 Markdown 列表形式组织的具体经验点，每条 ≤ 50 字

## 与 feedback 的边界

- 用户**纠正** agent 错误（Task Trace 中的 `action_type: correction` Step）
  → feedback，**本路不提取**
- 用户主动提出要求/补充上下文（在 Task Trace 中通常是 `action_type: request`
  Step 里的附加说明）→ task_specific_experience
- agent 自己从执行结果中总结的（从整条 Step 序列 inferred）
  → task_specific_experience
- 两者不可混淆

## 提取规则

### 1) 证据来源

- **以 Task Trace 为主要提取证据**：读 Step 序列、User Intent、Outcome 段落
- **Bullet 直接映射 Step 的因果关系**：
  - `Step N tool_call + status:success` 可派生执行技巧
  - `Step N status:failed + recovery` 可派生陷阱规避
  - `Step N action_type:summary` 可派生总结性建议
- Full Conversation 仅作歧义核对，不作为独立证据
- Task Trace 中 `User Intent` 字段对应本次任务的核心诉求，用于判断是否属于
  同一类任务

### 2) 何时提取

- 提取：HOW（执行过程、工具选择、步骤约束、失败恢复）
- 不提取：WHAT（内容本身、结果数据、一次性任务）
- 不提取：泛化价值低、仅此一次的操作
- 判断标准：该经验能否被同一用户在未来类似任务中复用？
- Task Trace 的 `Outcome.status=na`（闲聊）→ 直接返回空

### 3) 去标识化（CRITICAL）

- 移除案例特定实体（项目名/文件名/URL/具体数值）
- 用占位符替代：<项目名>、<目标路径>、<配置项>
- 保留真实命令和配置结构（已验证可用）
- 去标识后核心价值消失 → 不提取

### 4) 不发明内容

- 只提取 Task Trace 中有证据支持的步骤/技巧
- 不补充"通常还需要"的额外步骤
- 不发明用户未提及的约束或规范
- 不把 Task Trace 未结构化但 Full Conversation 里的内容当成证据

### 5) 多路径自检（对比型经验）

**先判断**：Task Trace 内是否存在"同一任务的多条替代执行路径"？

触发特征（需同时满足）：
- Task Trace 中存在 `action_type: correction` 的 Step，且 `user_feedback`
  明确表达"换一种"/"重新做"/"改用 X"/"scratch that"/"try a different way" 等
  **切换方案**（不是小幅修正）
- 切换前（被纠正 Step）和切换后（后续 Step）**都有实质性执行动作**（tool_call
  或带 tool_use 的 Step）
- 两条路径尝试的是**同一任务**（User Intent 未变），不是更换任务

**如果存在多路径**（按对比型经验格式输出）：
- `description` 以触发条件开头，如"选择 X 还是 Y 时" /
  "遇到 <情境> 且需权衡 <维度> 时"
- `bullet_points` 用对比格式：
  "1. 路径 A（<方案>）：适合 <场景>，优势 <...>，代价 <...>"
  "2. 路径 B（<方案>）：适合 <场景>，优势 <...>，代价 <...>"
  "3. 选择建议：<何时选 A / 何时选 B>"
- 不要刻意制造对比：用户只是微调参数 / 表述纠正 / 风格改写，
  **不是多路径**，按单路径输出
- 单纯"第一次做错、用户纠正、第二次做对"属于 feedback 范畴，
  **不是多路径**，本路不输出

**如果不存在多路径**：按下方单路径 schema 正常输出。

## 输出格式

只输出 JSON：

{
  "experiences": [
    {
      "kind": "task_specific_experience",
      "name": "<简短标题，4-8字>",
      "context": "<适用场景标签，1-3个>",
      "description": "<一句话摘要，≤50字>",
      "bullet_points": [
        "1. <具体经验点1>",
        "2. <具体经验点2>",
        "3. <具体经验点3>"
      ]
    }
  ],
  "skip_reason": "<若无可提取内容，说明原因>"
}

无可提取内容 → `{"experiences": [], "skip_reason": "..."}`

[User]
Task Trace（主要证据，来自 generate_task_trace）：
{{task_trace}}

Full Conversation（仅用于歧义核对，不作独立证据源）：
{{full_conversation}}

Task Summary:
{{task_summary}}

Context Tags:
{{context_tags}}
```

---

### 4.11 P10 — `evolve.zh.md`（TSE 丰富，读 Task Trace）⭐

```markdown
# 用 Task Trace 丰富已有任务经验

你是一个任务经验合并专家。对比新执行记录的 **Task Trace**（结构化任务轨迹）与已
有 task_specific_experience，判断是否有值得合并的增量信息。Task Trace 是上游
`generate_task_trace` prompt 产出的执行骨架，**以 Task Trace 为主要证据源**，
Full Conversation 仅在需要核对原文时参考。

## 合并原则

1. 语义并集：合并 bullet points，去重融合，不是简单拼接
2. 防退化：不能丢失已有经验中的有效约束或技巧
3. 去标识化：移除案例特定实体，保留可移植规则
4. RECENCY BIAS：两边冲突时，新 Task Trace 派生的内容优先
5. 不发明内容：只使用两边原有信息
6. Anti-Duplication：意思相近的点保留表述更清晰的一个

## 证据来源

- **Task Trace**（主要，必须引用）：从 Step 序列、User Intent、Outcome 派生新的
  bullet 候选
- **已有经验**：待合并的目标条目（context / description / bullet_points）
- Full Conversation（仅作歧义核对）：不作独立证据

## 判断维度（基于 Task Trace 的证据）

1. Task Trace 中的 Step 是否体现了已有经验**未覆盖**的技巧/陷阱/上下文依赖？
   - 新的 `status: failed + recovery` → 可能是新陷阱规避
   - 新的 `action_type: tool_call` 成功组合 → 可能是新工具技巧
   - `User Intent` 里的新上下文依赖 → 可能是新的隐性要求
2. Task Trace 是否提供了同一技巧的**不同适用场景**？
   - 已有 bullet 说的做法在本 trace 里对应新的情境标签
3. Task Trace 是否**纠正或优化**了已有经验的某个表述？
   - Task Trace 中的 `action_type: correction` 若指向已有 bullet 描述的做法
4. 以上均无 → `no_change`

## 合并决策规则

- 新增（added）的 bullet **必须溯源到 Task Trace 某个 Step**
- 移除（removed）的 bullet **必须说明理由**（过时/冗余/被新条目覆盖）
- bullets 总数上限 8 条，超限时保留信息密度最高的
- 已有经验中的独有约束若无冲突证据，**保留不动**

## 输出格式

只输出 JSON：

{
  "action": "enrich" | "no_change",
  "name": "<保留或优化后的标题>",
  "context": "<适用场景标签>",
  "description": "<一句话摘要，≤50字>",
  "bullet_points": [
    "1. <经验点1>",
    "2. <经验点2>"
  ],
  "added": ["新增了什么及 Task Trace 证据来源"],
  "removed": ["合并时去掉了什么及原因"],
  "reason": "总体判断说明"
}

action=no_change 时，bullet_points 可为空。

[User]
已有经验：
{{existing_experience}}

新 Chunk 的 Task Trace（主要证据，来自 generate_task_trace）：
{{task_trace}}

Full Conversation（仅用于歧义核对，不作独立证据源）：
{{full_conversation}}

Task Summary:
{{task_summary}}
```

---

### 4.12 设计文档 vs 实际实现的对齐状态（v5 更新后）

| 设计文档 | 实际实现 | 状态 |
|---------|---------|------|
| **§2.5.1 Task Trace 中间层**（generate_task_trace prompt）| **已新增**，下游 3 个 extractor 改为读 task_trace | ✅ 对齐 |
| **§2.5.1.1 降本论证（-54% token）** | Task Trace ~1500 tokens 替代 8000 tokens 原始对话，传给 3 个 extractor | ✅ 兑现 |
| **§2.5.2 两层查重 + hybrid 分段打分** | `dedup_decide` 对每对候选都调 LLM | 🟡 未分段，实现简化 |
| **§3.6 多路径对比**（已从 §三 删除）| `learn` prompt §5) 内嵌子分支 | ✅ 保留为 learn 的条件分支 |
| **§2.5.2 零 LLM is_hard 二次审核** | 各 prompt 自行输出 is_hard | 🟡 代码侧后置校验待确认 |
| **§2.5.4 候选池 MaintainPendingPool** | 未见对应 prompt | ✅ 纯规则（非 LLM），无需 prompt |
| **overview 字段** | 输出 task_summary / tools_used / key_steps / context_tags，无 description | 🟡 TSE context 由 learn/evolve 自行生成 |

---

### 4.13 可能需要补充的 Prompt

| 候选 | 对应功能 | 是否需要 |
|------|---------|---------|
| **refine_bullets** | TSE bullets 超上限时精炼 | 🟡 evolve prompt 虽约束上限 8 条，但未定义超限时的 refine 策略 |
| **comparative_learn**（独立）| 多路径对比独立 prompt | ❌ 不需要，已内嵌 learn §5) |
| **injection_select** | 三类注入时的 top-k LLM 精排 | 🟡 可选，当前可能走向量+关键词硬选 |
| **promote_from_pending_pool** | 候选池升格 | ❌ 不需要（纯规则 support_score 阈值）|
| **is_hard_verify** | is_hard 二次审核 | ❌ 不需要（正则扫否定词+操作词，代码侧执行）|
| **overview_from_task_trace** | 把 overview 也改为读 task_trace | 🟡 可选一致性优化：当前 overview 仍读原始对话 |

---

### 4.14 观察到的实现取舍（v5 更新后）

1. **Task Trace 中间层已建立** —— `generate_task_trace` 产出结构化 Markdown 轨迹，
   下游 `extract_exp_tp` / `learn` / `evolve` 共用，兑现 §2.5.1.1 的"降噪一次复用
   三次"论证
2. **双证据源设计** —— Task Trace 作主证据，Full Conversation 作歧义核对
   - 优点：Task Trace 覆盖率高，Full Conversation 兜底极端歧义
   - 注意：prompt 明确"不要把 Full Conversation 当独立证据源"
3. **decide + merge 拆分成两个 prompt** —— 判断合并和执行合并分开，成本增加但
   职责清晰
4. **TSE 分叉用单一 judge 入口** —— `selectedIndex=0` 走 learn，`≥1` 走 evolve，
   逻辑简洁
5. **多路径自检内嵌 learn** —— 不是独立 workflow，而是 learn prompt 的条件分支
6. **overview 暂保留读原始对话** —— 与 TSE 的 learn/evolve 分用不同输入是有意的
   冗余（overview 用于检索匹配，learn/evolve 用于经验提取，目的不同）

---

### 4.15 补齐建议

**短期（立即可做）**：
- [ ] 补齐 `generate_task_trace.en.md` / `extract_exp_tp.en.md` /
      `learn.en.md` / `evolve.en.md` 的英文版（目前只更新了中文版）
- [ ] 代码调用侧接入 `generate_task_trace`：Step 5-A 作为 Step 5-B/5-C 的前置
- [ ] 代码层补 `is_hard` 正则审核（设计文档 §2.5.2 所述的零 LLM 校验）

**中期（对齐与打磨）**：
- [ ] 评估 overview 是否也该改读 Task Trace（统一 TSE 链路的证据源）
- [ ] 更新 §2.5.2 的"两层查重"描述，与 `dedup_decide` 统一走 LLM 的实际行为对齐
- [ ] 补 `refine_bullets` 在 evolve 超过 8 条上限时的精炼逻辑

---