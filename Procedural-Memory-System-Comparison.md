# 程序记忆系统横向对比

> 对比范围：Mem0 / ReMe / MemOS / AutoSkill / Hermes / OpenSpace / Skill-Creator / XSkill / OpenViking / pskoett SI / memory-tencentdb
> 分析日期：2026-04-21（v8：新增 memory-tencentdb）

---

## 总览大表（一页看懂）

> 21 个关键维度 × 11 个系统的横向摘要。详细分析见后续各章。

| #   | 维度             | **Mem0**                           | **ReMe**              | **MemOS TS**                                              | **AutoSkill**                                    | **Hermes**                            | **OpenSpace**                                                              | **Skill-Creator**            | **OpenViking**                                                              | **pskoett SI**                        | **XSkill**                                         | **memory-tencentdb**                                                                                                |
| --- | -------------- | ---------------------------------- | --------------------- | --------------------------------------------------------- | ------------------------------------------------ | ------------------------------------- | -------------------------------------------------------------------------- | ---------------------------- | --------------------------------------------------------------------------- | ------------------------------------- | -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| 1   | **设计哲学**       | Facts                              | Experience            | Knowledge Assets                                          | Skills                                           | Evolution Capital                     | Self-Evolving Skill Engine                                                 | Quality-Tested Skills        | Context as Filesystem                                                       | Learning as Manual Log                | Dual-Stream                                        | Layered LTM (L0-L3)                                                                                                 |
| 2   | **产物形态**       | 文本快照                               | `when_to_use`+经验      | SKILL+脚本+评估                                               | SKILL.md(YAML)                                   | SKILL+脚本                              | SKILL.md+scripts/+references/（版本 DAG + content_snapshot）                   | SKILL+子代理+评估                 | 目录+URI 文件(L0/L1/L2)                                                         | LEARNINGS.md/ERRORS.md/FEATURE.md     | Experience(≤64词)+Skill                             | L0 JSONL + L1 JSONL(persona/episodic/instruction) + L2 scene MD + L3 persona.md                                     |
| 3   | **架构耦合度**      | 解耦-开放协议（HTTP）                      | 解耦-框架 hook            | 解耦-框架 hook（TS-end）                                        | 解耦-框架 hook（OpenClaw）                             | **耦合-agent 自觉**（`skill_manage` 工具）    | **解耦-委派模式**（MCP 宿主 0 负担，OpenSpace 自执行）                                     | **耦合-agent 执行**（读元指令自跑）      | 解耦-专有接口（`viking://`）                                                        | **耦合-agent 自觉**（自觉 append Markdown）   | 解耦-独立框架                                            | 解耦-框架 hook（OpenClaw 插件）                                                                                             |
| 4   | **提取触发**       | 显式API                              | `agent_end`           | `agent_end`+话题切换                                          | 滑窗+`agent_end`                                   | **LLM自主**                             | **三独立触发器**（ANALYSIS 每任务后 / TOOL_DEGRADATION 工具降级 / METRIC_MONITOR 周期扫描）    | **用户主动**                     | session.commit() 异步                                                         | Hook提醒+Agent自觉                        | batch后                                             | `agent_end` hook + 后台 L1→L2→L3 管线                                                                                   |
| 5   | **触发决策者**      | 调用方                                | 代码硬编码                 | 代码硬编码                                                     | 代码硬编码                                            | LLM 自己                                | 规则+LLM 二次确认（`_llm_confirm_evolution`）                                      | 用户                           | 代码硬编码+手动                                                                    | Agent自己                               | 代码硬编码                                              | 代码硬编码（checkpoint + serial queue）                                                                                    |
| 6   | **失败轨迹学习**     | ✗                                  | **✓**(独有)             | ✗                                                         | ✗                                                | 隐式                                    | **✓**（analyzer 从轨迹提取 tool_issues + evolution_suggestions）                  | **✓**(基线对比)                  | ✗(隐式)                                                                       | ✓(ERRORS.md)                          | **✓**(跨rollout)                                    | ✗                                                                                                                   |
| 7   | **去重机制**       | 无                                  | 向量余弦                  | hash+语义LLM                                                | 向量+4维LLM                                         | 无                                     | skill_id 稳定锚 + BM25+向量预筛 + LLM 最终选择                                        | 人工                           | 向量预过滤+LLM决策                                                                 | Pattern-Key人工分配                       | 嵌入≥0.7+LLM                                         | LLM 四动作（store/skip/update/merge）+ 向量优先 FTS fallback                                                                 |
| 8   | **质量门控**       | ✗                                  | 5维验证                  | 0-10评分                                                    | 4维身份                                             | ✗                                     | **4 比率**（applied/completion/effective/fallback）+ tool penalty 曲线 + LLM 确认门 | 4子代理+人工                      | ✗                                                                           | Recurrence≥3+跨2任务+30天                 | 跨轨迹+64词                                            | ✗（仅 shouldExtractL1 结构性过滤）                                                                                          |
| 9   | **使用审计闭环**     | ✗                                  | utility/freq          | ✗                                                         | **✓**(retrieved→used)                            | ✗                                     | **✓**（4 计数器原子递增：selections/applied/completions/fallbacks）                  | eval pass_rate               | ✗                                                                           | ✗                                     | ✗                                                  | ✗（仅 reporter 指标）                                                                                                    |
| 10  | **通用化程度**      | 无                                  | 中                     | 高                                                         | 高                                                | 高                                     | 高（版本 DAG + 跨 host MCP 复用）                                                  | **最高**                       | 中                                                                           | 中(三级升格)                               | 高                                                  | 中（instruction 支持；L3 Chapter 3 汇总）                                                                                   |
| 11  | **复用测试**       | ✗                                  | ✗                     | evals/                                                    | SkillEvo replay                                  | GEPA测试套件                              | **GDPVal 50 任务**（4.2× 收入 / -46% tokens，Qwen 3.5-Plus 同骨干）                  | **`claude -p`黑盒**            | 无公开 benchmark（分析文档未记录）                                                      | ✗                                     | 5bench×4backbone                                   | ✗                                                                                                                   |
| 12  | **更新默认操作**     | ADD                                | Rewrite合并             | **add / upgrade**（Skill 层；chunk 层另有 DUPLICATE/UPDATE/NEW） | add/merge/discard                                | LLM create/patch                      | **FIX / DERIVED / CAPTURED**                                               | 人工迭代                         | MERGE/CREATE/SKIP                                                           | ADD(追加Markdown)                       | add/modify+合并                                      | store / skip / update / merge（LLM 四动作）                                                                              |
| 13  | **主存储**        | 向量DB(20+)                          | 向量DB                  | SQLite+文件                                                 | 文件+向量                                            | 文件系统                                  | **SQLite**（6 张表 + 版本 DAG 多对多）+ 文件系统 skill 目录                               | 文件系统                         | 文件系统(viking://)                                                             | 文件系统(.learnings/)                     | 文件+numpy                                           | **双后端**：SQLite(`vectors.db`)+`sqlite-vec`+FTS5 / TCVDB+BM25 sparse                                                  |
| 14  | **检索方式**       | 语义向量                               | 向量+LLM重排              | FTS+向量混合                                                  | 向量+BM25(0.9:0.1)                                 | 系统提示动态加载                              | BM25 粗排(×3) + 向量精排 + LLM 最终选择（Plan-then-Select）                            | `claude -p`端到端               | 目录递归（IntentAnalyzer→优先队列，score 传播 α=0.5）；L0/L1/L2 非 runtime 自动，需 Agent 显式调用 | 无(pre-flight-check)                   | 任务分解+top-k=3                                       | keyword(FTS5) / embedding / **hybrid(RRF 融合)**，5s 超时兜底                                                              |
| 15  | **注入方式**       | 调用方决定                              | System Prompt         | `before_prompt_build` hook 追加                             | `before_prompt_build` hook 追加（top-k 拼接 + LLM 二选） | **三级动态加载**（L0 索引 / L1 概述 / L2 完整）     | **宿主 0 注入**（委派模式，skill 由 OpenSpace 内部执行 agent 消费）                          | Agent 读 SKILL.md 进 context   | Agent 主动 `read`/`ls`                                                        | **bootstrap 虚拟文件 + 每轮 activator 提醒**  | System Prompt（**LLM Rewrite + Adapt**，非强制）         | **4 段 System Prompt**（`<user-persona>` + `<scene-navigation>` + `<relevant-memories>` + `<memory-tools-guide>`）     |
| 16  | **离线演进**       | ✗                                  | 仅老化删除                 | upgrade 路径（7 维评估+合并）                                      | SkillEvo(遗传 + replay)                            | **GEPA**(最完整)                         | **Tool 降级 → Skill 级联修复**（独有：跨层闭环）                                          | 描述优化(train/test)             | ✗(仅热度归档)                                                                    | 三级升格漏斗                                | 层次化精炼                                              | ✗（instruction 不演进；L2 场景可 UPDATE/MERGE，L3 persona 可重生成）                                                              |
| 17  | **生成成本**（单次学习） | 1 次                                | 3-4 次                 | 5-9 次                                                     | 5-8 次/轮                                          | 1 次（`skill_manage` 工具调用）              | agent loop ≤5 轮 + apply-retry ≤3 次                                         | 2-3M tokens/技能               | 2-5k tokens × N 类/commit                                                    | **无独立调用**（摊入 agent：每轮 +500-2k tokens） | N+3+K/样本                                           | L1 每次 1-2 次（warm-up 1→2→4→...）                                                                                      |
| 18  | **演进成本**（长期迭代） | **无**                              | 仅老化（0 LLM）            | upgrade 路径（每任务 ~4 次 LLM）                                  | SkillEvo 遗传 replay（M 个候选 × 3 轮）                  | **GEPA 最重**（每代 N 次变异+judge+benchmark） | 3 触发器（ANALYSIS 每任务 + TOOL_DEGRADATION 级联 + METRIC 周期）                      | 描述优化 60 × `claude -p` × 5 轮  | MemoryArchiver（0 LLM）                                                       | 摊入 agent（每次升格判断）                      | 每批 8 样本后 REFINE(1) + 容量>120 强制合并(1-3)              | ✗ instruction 不演进（L2 场景可 UPDATE，L3 全量重生成 1 次/触发）                                                                    |
| 19  | **最大优势**       | 实现最简 1 次 LLM + 历史保真                | **失败轨迹提取**（独有）        | 产物最完整（SKILL + 脚本 + 评估集）                                   | **使用审计闭环**（retrieved→relevant→used）              | **GEPA 最完整进化闭环** + 知识图谱               | **委派模式** + **Tool→Skill 级联**（跨层闭环，独有）                                      | 优化即核心流程（测试驱动 + 人工 A/B）       | 文件系统范式 + 三级粒度（L0/L1/L2）                                                     | 零基础设施 + skill-as-system 范式 + 跨编辑器通用   | **双流分离**（Experience+Skill 互补）+ 视觉锚定 + 跨 rollout 对比 | **四阶分层最清晰**（对话→画像）+ 双后端（SQLite/TCVDB）+ RRF hybrid                                                                   |
| 20  | **最大劣势**       | **无复用 + 无演进**（verbatim 快照，跨任务无法迁移） | 产物松散不可执行 + 无主动演进      | 被动升级 + LLM 调用多（5-9 次）                                     | 仅成功轨迹（`successOnly=true`）                        | **GEPA 成本最高** + 需 PR 审查               | 架构复杂度最高（skill_engine ~7800 行）                                              | **token 极高**（2-3M/技能）+ 需人工全程 | 抽取 fire-and-forget 无重试 + SKIP 硬丢弃无审计                                        | 成本摊入 agent + Markdown 线性膨胀            | **需 ground_truth** + N=4 rollout 真实执行成本高           | **粒度单一**：所有程序记忆都塞进 `instruction` 一个类型，不区分**动作级**（修改 config 时）/ **工具级**（搜代码选 rg）/ **任务级**（调 FastAPI 的经验）—— 跨任务触发能力缺失 |
| 21  | **未覆盖的能力**     | 跨任务复用 / 离线演进 / 质量门控 **全无**         | 主动演进 / 可执行 SOP / 强度分级 | 使用审计闭环 / 主动触发演进 / 分支派生                                    | 失败轨迹学习 / 多路径对比 / 跨层（仅 skill 级）                   | 使用审计 / 分支派生 / 自动质量门控                  | **非委派场景无学习**（宿主未调 `execute_task` 时 OpenSpace 不感知、不提取、不演进）/ 多模态             | 自动提取 / 在线演进 / 规模化批量          | Prospective 类别 / 真正演进（仅归档不合并优化）/ 失败轨迹                                       | 自动检索 / 跨项目共享 / 结构化审计                  | 候选池升格 / 无 ground_truth 场景 / 单路径场景                  | **多粒度分层**（动作级/工具级/任务级三合一塞进 `instruction`，跨任务触发失效）                                                                   |

**阅读这张表的三条捷径**：

- **按产物看光谱**：Mem0 快照 → **memory-tencentdb 分层记忆** → ReMe 建议 → MemOS/AutoSkill/Hermes/**OpenSpace** 可执行 SOP → Skill-Creator 测试验证 SOP → XSkill 双流分离 → OpenViking 目录文件 → pskoett 日志漏斗。产物越往右越"通用+抽象"，越往左越"保真+具体"。
- **按注入方式看**（第 15 行）：Push（Mem0/ReMe/MemOS/AutoSkill/memory-tencentdb，系统推）vs Pull（Hermes/Skill-Creator/OpenViking/pskoett，agent 拉）vs **委派**（OpenSpace 独有，宿主 0 注入）三种范式。
- **按演进闭环看代差**：**OpenSpace 是唯一一个做 "Tool 质量 → Skill 级联修复" 的跨层闭环**（第 16 行）——其他系统要么只监控 skill（AutoSkill 使用审计），要么只监控 tool，不跨层联动。
- **按成本/质量看定位**（第 17-18 行分开看）：
  - **生成成本**：pskoett（摊入 agent）和 Hermes（1 次）最低，Skill-Creator（2-3M tokens）最高；中间按 Mem0 < ReMe < AutoSkill ≈ MemOS < OpenSpace < OpenViking < XSkill 递增。
  - **演进成本**：Mem0/OpenViking（无/极低）← Hermes GEPA / Skill-Creator 描述优化（极高）是两个极端；OpenSpace 和 pskoett 通过精准触发/agent 摊入控制在中等可控范围。
  - **Hermes 是典型"生成极低但演进极重"**；**Skill-Creator 两段都贵**；**pskoett 看似 0 LLM 其实都摊进 agent 自身**。
- **最大优劣一眼看**（第 19-20 行）：两端给出每个系统的"最突出优点"和"最突出痛点"，适合做决策时快速筛选。

---

## 一、核心定位对比

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **设计哲学** | Memory as Facts | Memory as Experience | Memory as Knowledge Assets | Memory as Skills | Memory as Evolution Capital | Memory as Self-Evolving Skill Graph | Memory as Quality-Tested Skills | Context as Filesystem | Learning as Manual Log | Memory as Dual-Stream Knowledge | Memory as Layered Long-term Store |
| **核心目标** | 任务中断后精确恢复 | 提供情境化经验建议 | 生成可执行技能资产 | 提取可复用方法论 | Agent 随时间自动进化 | 跨 agent 共享自演进技能 + 云协作 | 精雕细琢高质量技能 | 统一上下文管理(memory+resource+skill) | 记录教训→升格为规则→结晶为skill | 双流互补持续积累 | 长程 persona + 事件 + 交互规则统一管理 |
| **产物本质** | 执行历史"录像" | 经验"建议书" | 可执行"技能包" | 可复用"方法论" | 可进化"知识资本" | 版本化 DAG 的可执行 skill（FIX/DERIVED/CAPTURED） | 经测试验证的"精品课程" | 层级目录中的技能/经验文件 | Markdown日志+三级升格漏斗 | 行动级"笔记" + 任务级"SOP" | L0 对话 + L1 结构化记忆（persona/episodic/instruction）+ L2 场景叙事 + L3 用户画像 |
| **与 Agent 系统耦合度** | **解耦-开放协议**（HTTP API，任意 agent 可调） | **解耦-框架 hook**（`agent_start` hook）| **解耦-框架 hook**（TS-end timer + `before_prompt_build`）| **解耦-框架 hook**（OpenClaw hook 系统）| **耦合-agent 自觉**（`skill_manage` 工具内嵌思考环）| **解耦-委派模式**（MCP 宿主 0 记忆负担，OpenSpace 自己执行）| **耦合-agent 执行**（读 SKILL.md 元指令自跑评估）| **解耦-专有接口**（`viking://` URI + HTTP）| **耦合-agent 自觉**（自觉 append Markdown）| **解耦-独立框架**（自带骨干，跨模型迁移）| **解耦-框架 hook**（OpenClaw 插件，`before_prompt_build` + `agent_end` hook）|
| **适用场景** | 通用记忆基础设施 | 长对话 Agent | 企业级任务管理 | OpenClaw 插件 | 自进化个人助手 | 多 host 共享技能生态（Claude Code/Codex/OpenClaw/nanobot/Cursor） | Claude Code 技能开发 | 长程 Agent 上下文引擎 | 个人Claude Code/Copilot开发 | 多模态视觉推理 | OpenClaw 企业级长程记忆（TCVDB 云后端 + 本地 SQLite 双模式）|

---

## 二、记忆产物形态

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **产物结构** | 逐步骤文本快照 | `when_to_use` + `experience` + tags | SKILL.md + scripts/ + references/ + evals/ | SKILL.md (YAML frontmatter + Markdown) | SKILL.md + scripts/ + references/ | SKILL.md + scripts/（版本 DAG + content_snapshot/diff） | SKILL.md + scripts/ + agents/ + evals/ + eval-viewer/ | viking://agent/skills/ + memories/ | .learnings/LEARNINGS.md ERRORS.md FEATURE_REQUESTS.md | **双流**：Experience(≤64词 condition-action-embedding) + Skill(Markdown SOP) | — |
| **内容示例** | Step 1: Opened URL... Result: HTML loaded... | {"when_to_use": "当需要分页抓取时", "experience": "使用 BFS 而非 DFS..."} | # Goal / # Steps / # Pitfalls / evals/test_*.json | # Goal / # Constraints & Style / # Workflow | 同标准 SKILL.md 格式 | 同标准 SKILL.md + version DAG（FIX/DERIVED/CAPTURED 分支） | 同标准 SKILL.md + 4子代理评审产物 | ToolPart(截断500字符)+L0/L1摘要 | ## [LRN-YYYYMMDD-XXX] category + Pattern-Key | Experience: "When handling dark images, use histogram equalization" / Skill: # Workflow + ## Tool Templates | — |
| **去标识化** | **无**（保留 URL/ID/参数） | 中等（提取模式） | **高**（placeholder 替换） | **高**（placeholder 替换） | 高（LLM 自主判断） | 中（LLM 演进时隐式清洗） | 高（用户主动控制） | 中（URI路径天然抽象） | 高（人工提炼通用模式） | **高**（占位符 `[TARGET]` `[QUERY]` + 64词强制泛化） | — |
| **可执行性** | 不可执行（纯记录） | 不可执行（建议性） | **可执行**（含脚本和评估集） | 可执行（Prompt 指令） | 可执行（Skill 三级加载） | **可执行**（SKILL.md + scripts/，MCP 工具触发执行） | **可执行**（含脚本+评估集+子代理） | **不可执行**（描述性 memory：abstract/overview/content 三级文本摘要，非执行性 SKILL.md；`memories/skills/` 仅是 8 类分桶之一） | 不可执行（建议性+pre-flight-check） | Experience 建议性 / Skill 可执行（含代码模板） | — |

---

## 三、提取触发机制

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **触发方式** | 显式 API 调用 | 自动（`agent_end` hook） | 自动（`agent_end` + 话题切换） | 自动（滑动窗口 + `agent_end`） | **LLM 自主判断**（系统提示引导） | **三独立触发器**（ANALYSIS / TOOL_DEGRADATION / METRIC_MONITOR） | **用户主动触发** | session.commit() 后异步 | Hook触发+Agent自觉 | 自动（batch 训练后） | — |
| **触发条件** | `add(agent_id=..., memory_type="procedural")` | 任务完成，score >= threshold | 任务完成，chunk 数/轮数达标 | 每轮 `messages[-6:]` 延迟提取 | 5+ tool calls / 修复错误 / 发现工作流 | 任务完成 / 工具降级 / 4 比率周期扫描越线 | 用户说"做个技能"/"优化这个技能" | 手动commit 或 auto_commit_threshold=8000 | activator.sh每轮提醒/error-detector.sh命中 | N 次 rollout 全部完成后 | — |
| **谁决定提取** | 调用方（外部代码） | 代码硬编码 | 代码硬编码 | 代码硬编码 | **LLM 自己**（`skill_manage` 工具） | 规则触发 + LLM 二次确认（`_llm_confirm_evolution`） | **用户自己** | 代码硬编码（两阶段commit） | Agent自己 | 代码硬编码（batch pipeline） | — |
| **人类类比** | 手动按录像键 | 自动剪辑精彩集锦 | 自动归档项目文档 | 每轮自动扫描笔记 | 自觉记笔记的学习者 | 教练+监察员：任务后复盘 + 指标异常自查 | 手工打造精品课程 |  |  | 考完试后批量总结错题 | — |

---

## 四、提取 Prompt 设计

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **Prompt 数量** | 1 个专用 | 4 个（成功/失败/对比/验证） | 6-8 个（分段管线） | 3-4 个（提取/决策/合并） | **无独立提取 Prompt**（仅靠系统提示指令） | **5 个**（analysis / fix / derived / captured / confirm） | **无提取 Prompt**（人机迭代创建） | **8 个**（每类别一份 yaml，统一用 VLM） | 0 | **10 个**（P1-P10 覆盖积累+推理两阶段） | — |
| **Prompt 长度** | ~800 字 | ~400 字 × 4 | ~500-2000 字 × N | **~4000 字**（8 章节） | ~500 字（系统提示中的行为指令） | ~400-800 字 × 5（每个演进类型专用） | ~900 行 SKILL.md（元技能指令） | 按类别模板（profile/events/tools等） | 0（无LLM抽取） | ~200-600 字 × 10 | — |
| **核心指令** | "记录每步 Action + Result，保留 verbatim" | "提取可复用模式、决策点、失败防范" | "生成任务摘要 → 评估可复用性 → 生成 SKILL.md" | "提取 HOW 非 WHAT，去标识化，用户输入为证据" | "5+ calls 后保存为 skill，发现过时立即 patch" | "分析轨迹 → 选 FIX/DERIVED/CAPTURED → patch-first 改写 → apply-retry 3 次" | "Draft → Test → Review → Improve → Repeat" | 8类单标签分类抽取 | 按spec手写Markdown | "Review attempts and extract practical lessons (≤64 words)" | — |
| **证据来源** | 完整对话（verbatim） | 轨迹（messages + score） | 任务摘要（LLM 生成） | **仅 user 消息**（assistant 仅参考） | LLM 自主判断 | 任务转录 + tool_issues + 4 比率指标 + 磁盘真实内容（apply-retry 回喂） | 子代理执行转录 + 人工评审 | 完整session（含ToolPart截断500字符） | Agent自身执行观察 | **N 条轨迹 + 图像**（视觉锚定） | — |
| **失败轨迹** | 无 | **有**（专用失败提取 + 对比分析） | 无 | 无（`successOnly=true`） | 无（隐式包含） | **有**（analyzer 从轨迹提取 tool_issues + evolution_suggestions） | **有**（without_skill 基线对比） |  |  | **有**（N 路 rollout 含成功+失败，跨轨迹对比） | — |

---

## 五、多路径对比能力

这是 v5 新增的核心对比维度。不同系统处理"同一任务多种做法"的能力差异巨大。

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **多路径来源** | 无 | 无 | 无 | 无 | 无 | — | **刻意构造**（with/without skill 并行 spawn） | — | — | **自然产生**（N=4 独立 rollout） | — |
| **对比方式** | — | — | — | — | — | — | Grader 评分 + Comparator 盲测 A/B + Analyzer 事后分析 | — | — | **Cross-Rollout Critique**（成功/失败轨迹对比提取经验） | — |
| **对比粒度** | — | — | — | — | — | — | 整体输出质量（rubric 1-10） | — | — | **逐步骤**（哪步有效、哪步出错、工具组合效果） | — |
| **谁做对比** | — | — | — | — | — | — | 3 个子代理（Grader/Comparator/Analyzer） | — | — | MLLMkb（知识模型，可多模态） | — |
| **对比产物** | — | — | — | — | — | — | feedback.json → 人工改进技能 | — | — | Experience 更新操作 [{add, e}, {modify, id, e'}] | — |
| **真实执行** | — | — | — | — | — | — | **是**（子代理真实执行任务） | — | — | **是**（N 次独立 rollout 真实执行） | — |
| **视觉对比** | — | — | — | — | — | — | 否（纯文本） | — | — | **是**（图像与轨迹联合分析） | — |

**与 v5 设计（§7.1a）的对应关系**：

| 维度 | **v5 设计** | **XSkill** | **Skill-Creator** |
|------|------------|------------|-------------------|
| **多路径来源** | **会话内自然产生**（用户纠正/agent 换方案） | N=4 独立 rollout | with/without skill 并行 spawn |
| **检测方式** | **零 LLM 正则扫**（correction Step / 失败→成功 / 切换意图词） | 不需检测（天然 N 条路径） | 不需检测（刻意构造） |
| **路径切分** | 以多路径信号 Step 为切分点 → Path A / Path B | 天然独立（rollout_0 ~ rollout_3） | 天然独立（with_skill / without_skill） |
| **优劣推断** | **零 LLM 关键词匹配** → preferred_path_hint (A/B/unclear) | 对照 ground_truth（有标准答案） | 人工评审 + rubric 打分 |
| **对比提取** | ComparativeLearnTaskExperience（大模型） | INTRA_SAMPLE_CRITIQUE（MLLMkb） | Analyzer agent（事后分析） |
| **产物** | 方案对比型 TSE（含 comparative_analysis） | Experience [{add,e}/{modify,id,e'}] | improvement_suggestions（技能改进建议） |

---

## 六、去重与合并机制

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **去重策略** | **无**（有意为之） | 向量余弦相似度（阈值 0.5） | 两级：hash + 语义 LLM | 向量 + LLM 4 维判断 | 无独立机制 | 两层：skill_id sidecar + BM25+向量预筛 + LLM Plan-then-Select | 无自动去重（人工控制） | 向量预过滤(top-5)+LLM决策 | Pattern-Key人工分配+Recurrence计数 | **嵌入余弦 ≥0.70 → LLM 合并** | — |
| **合并策略** | 无 | RewriteMemory 重写合并 | DUPLICATE/UPDATE/NEW 决策 | add / merge / discard 决策 | LLM 自主 patch/edit | FIX（覆写）/ DERIVED（分支）/ CAPTURED（新建），version DAG 多对多 | 人工评审后改进 | MERGE/CREATE/SKIP（类别特定） | 人工See Also链接+状态机更新 | **三层合并**（逐条/容量缩减/全局精炼） | — |
| **判断维度** | — | 向量相似度（无 LLM） | 语义 LLM 评估 | **4 维能力身份** | — | skill_id 稳定锚 + BM25 top-3 粗排 + 向量精排 + LLM 最终选择 | 人工判断 | SKIP直接丢弃; PROFILE永远MERGE; CASES/EVENTS不可MERGE | Recurrence-Count+跨任务数+时间窗口 | 嵌入相似度（粗筛） + LLM（精合并） | — |
| **批内去重** | 无 | 有（同批次互相比） | 有（IngestWorker 批处理） | 有（滑动窗口重叠 4 条） | 无 | 无（单次演进，但 DERIVED 前 LLM 选择阻止同名分支） | 无（迭代式改进非批量） | 按session commit批次 | 无 | **有**（每批 8 样本后全局精炼） | — |
| **容量管理** | 无限制 | utility/freq 老化删除 | 无明确限制 | retrieved≥40 & used==0 删除 | 2200 字符硬限制 | version DAG 保留历史，fallback_rate 驱动淘汰 | 无（靠人控制） | MemoryArchiver热度归档(age>7d+score<0.1) | ✗（Markdown线性增长） | **硬限制 120 条 + 强制合并最相似对** | — |
| **设计意图** | 历史保真 | 减少噪声积累 | 避免重复 Skill | 渐进完善 | 信任 LLM 判断力 | 稳定锚防漂移 + 混合检索抗知识膨胀 | 手工打磨不需去重 | 层级文件系统+渐进加载降token | 极简零成本+人工把关 | 嵌入粗筛+LLM精合并，平衡成本和质量 | — |

---

## 七、质量门控

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **质量评估** | **无** | **5 维验证** | **0-10 评分** | **4 维能力身份** | **无自动门控** | **4 比率 + tool penalty + LLM 确认门** | **4 子代理 + 人工评审** | ✗ | Recurrence≥3+跨2任务+30天 | **跨 Rollout 对比 + 64 词硬约束** | — |
| **评估维度** | — | ACTIONABILITY / ACCURACY / RELEVANCE / CLARITY / UNIQUENESS | Repeatable / Transferable / Technical depth / 完整度 | core job / output contract / constraints / context | — | applied_rate / completion_rate / effective_rate / fallback_rate | Grader 断言 + Comparator rubric + Analyzer 模式 + 人工 feedback | — | 出现频率+跨任务泛化+时间稳定性 | 成功/失败轨迹对比 + REFINE 去低质量 | — |
| **评分方式** | — | 独立 LLM 打分（0.0-1.0） | 独立 LLM 打分（0-10） | LLM 决策（add/merge/discard） | LLM 自觉 + 安全扫描 | 原子计数器聚合 4 比率 + LLM 二次确认 | 子代理打分 + **人工最终决策** | — | 硬阈值（非连续分数） | LLM 对比批判 + 嵌入相似度筛选 | — |
| **淘汰阈值** | — | **score < 0.3 → 剔除** | qualityScore < 6 → draft | discard 直接丢弃 | 50+ 正则威胁检测 | fallback_rate 越线 + min_selections≥5 → 触发 FIX/DERIVED | 人工说"不满意" → 重做 | — | wont_fix状态 | 全局精炼 delete 低价值条目 | — |
| **使用审计** | 无 | 无（有 utility/freq 追踪） | 无 | **有**（retrieved→relevant→used 闭环） | 无 | **有**（4 计数器原子递增：selections / applied / completions / fallbacks） | 有（eval pass_rate 追踪） |  |  | 无（但有跨任务迁移实验验证泛化性） | — |
| **自动清理** | 无 | **utility/freq 双维度老化** | 无 | retrieved>=40 & used==0 → 删除 | 容量压力逼合并 | METRIC_MONITOR 周期扫描 → FIX 覆写低分版本 | 无（人控制生命周期） | 热度低记忆移到_archive/ | ✗ | **全局精炼** merge/delete（目标 80-100 条） | — |

---

## 八、通用化与复用

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **通用化程度** | **无** | 中等 | **高** | **高** | 高 | 高（DERIVED 分支 + GDPVal 实证） | **最高**（测试驱动验证） | 中 | 中(三级升格) | **高**（64 词硬约束 + 占位符） | — |
| **去标识化方法** | 保留所有具体值 | 抽象为模式描述 | placeholder 替换 | placeholder 替换 | LLM 自主判断 | LLM 演进时自然抽象（无显式占位符机制） | 用户主动控制 | URI路径抽象+目录层级 | 人工提炼Pattern-Key | `[TARGET]` `[QUERY]` 占位符 + 64 词限制强制泛化 | — |
| **跨任务复用** | 不适合（太具体） | 适合（经验建议） | **适合（可执行 SOP）** | 适合（方法论抽象） | 适合（Skill 格式标准） | **适合**（跨 host MCP 复用 + GDPVal 50 任务验证） | **适合**（经 A/B 测试验证） | 适合（User/Agent 两侧共 8 桶存储复用） | 中（靠升格到CLAUDE.md/SKILL.md） | **适合**（零样本迁移实验验证，跨 benchmark 2-3 分提升） | — |
| **复用测试** | 无 | 无 | **有（evals/ 评估集）** | **有（SkillEvo replay 评估）** | 有（GEPA 测试套件） | **有**（GDPVal 50 任务 benchmark：4.2× 收入 / -46% tokens） | **有**（run_eval.py 触发率 + 子代理 A/B + 人工评审） | 无公开 benchmark（分析文档未记录） | ✗ | **有**（5 benchmark × 4 backbone + 零样本迁移实验） | — |
| **版本管理** | 无 | 无 | 有（版本号递增） | 有（版本号递增） | 有（patch/edit 历史） | **有**（version DAG 多对多 + content_snapshot + content_diff） | 有（iteration-N/ 目录结构） | archive_NNN顺序归档 | 无 | 有（batch 迭代，全局文档版本） | — |

---

## 九、更新策略

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **默认操作** | **ADD**（总是新增） | RewriteMemory 合并更新 | **Skill 层**：`add / upgrade`（upgrade 内部含"升级评估 + 合并 + 附属重建"子步骤）<br>**Chunk 层**：`DUPLICATE / UPDATE / NEW` | **decide** → add/merge/discard | **LLM 自主选择** create/patch/edit | **FIX / DERIVED / CAPTURED**（LLM 选择演进类型） | **人工迭代改进** | MERGE/CREATE/SKIP（类别决定） | ADD（追加Markdown） | **add/modify** → 嵌入去重 → LLM 合并 | — |
| **UPDATE 条件** | 不支持 | 向量相似命中 → 重写合并 | 语义相似 → 合并内容 | 4 维身份匹配 → merge | LLM 判断 outdated → patch | fallback_rate 越线 / 工具降级 → FIX 覆写 | 人工评审不满意 → 改进 | 向量相似命中+LLM决策MERGE | Recurrence-Count++ / Status更新 | 嵌入余弦 ≥0.70 → LLM 合并 / modify 引用 ID 直接替换 | — |
| **DELETE 条件** | 不支持 | utility/freq 老化删除 | 无自动删除 | 使用审计未使用 → 删除 | LLM 判断 → delete | 无硬删除（旧版本保留在 DAG，靠检索排序淘汰） | 人工决定 | SKIP直接丢弃（无审计） | wont_fix | 全局精炼 delete 低价值 / 容量超限强制合并 | — |
| **历史保留** | SQLite 审计表全保留 | 删除标记 + 频率追踪 | Chunk 标记 dedupStatus | 版本号递增覆盖 | patch 覆盖 | version DAG 全保留 + content_snapshot + content_diff | iteration-N/ 目录保留所有迭代 | archive_NNN/messages.jsonl归档 | 完整保留所有entry | 重编号覆盖（E0, E1, E2...） | — |
| **合并质量** | — | LLM 重写为统一建议 | LLM 语义合并 | LLM 语义合并（防回归） | LLM 自主 edit | patch-first 生成 + apply-retry 3 次（回喂磁盘真实内容） | 人工把关 | LLM决策（无人工门控） | 人工判断 | LLM 合并（MERGE_PROMPT：包含所有信息点，≤64词） | — |

---

## 十、存储与检索

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **主存储** | 向量 DB（20+ 后端） | 向量 DB（Memory/ES/Chroma） | SQLite + 文件系统 | 文件系统 + 向量索引 | 文件系统（~/.hermes/skills/） | **SQLite 6 张表 + 文件系统** skill 目录 | 文件系统（skill-name/） | 文件系统(viking://) | 文件系统(.learnings/) | **文件系统**（experiences.json + SKILL.md） | — |
| **辅助存储** | SQLite（审计历史） | 文件系统（ReMeLight） | SKILL.md 磁盘文件 | SkillBank/ + Mirror | SQLite（Session Archive） | version_nodes/edges + BM25 索引 + 向量嵌入 | workspace/ 迭代目录 | archive_NNN/ 顺序归档目录 | 无 | numpy 嵌入缓存 | — |
| **向量字段** | 完整摘要 | **`when_to_use`（非 content）** | Chunk summary | Skill description + prompt | 无（纯文件系统） | SKILL.md 描述 + 用途（BM25 + 向量双索引） | 无（`claude -p` 黑盒测试触发） | memory overview内容 | 无 | **Experience 的 `name: description`**（非 bullet_points） | — |
| **检索方式** | 语义向量搜索 | 语义向量 + LLM 重排 | FTS + 向量混合 | **向量 + BM25 混合（0.9:0.1）** | 系统提示动态加载 Skill 索引 | **BM25 粗排 ×3 + 向量精排 + LLM Plan-then-Select** | `claude -p` 端到端触发测试 | IntentAnalyzer(LLM重写 query)→HierarchicalRetriever(优先队列递归目录, α=0.5 score 传播) | 无(pre-flight-check) | **任务分解 → 嵌入 top-k=3 余弦相似度** | — |
| **检索后处理** | 无（上层手动拼接） | **RewriteMemory**（重写为统一建议） | 直接注入 SKILL.md | LLM Skill 选择（二次筛选） | 三级加载（L0/L1/L2） | LLM Plan-then-Select（从候选列表生成 skill_plan） | 无（触发后读完整 SKILL.md） | L0/L1/L2 粒度层 by Agent 显式调用（非 runtime 自动 progressive loading） | pre-flight-check提醒（无主动检索） | **Experience Rewrite + Skill Adaptation**（MLLMkb 根据当前任务和图像适配） | — |
| **注入位置** | 由调用方决定 | System Prompt | before_prompt_build hook | before_prompt_build hook | System Prompt（动态构建） | MCP 工具响应（execute_task 返回 skill_plan + SKILL.md 内容） | 触发时读取 SKILL.md | Agent通过read/ls主动读取（非强制注入） | bootstrap虚拟文件+每轮activator提醒 | **System Prompt**（非强制注入："consider applying... your own ideas"） | — |

---

## 十一、注入机制

> 记忆提取完后，**如何把知识送进 agent 的上下文**是决定实际效果的关键一环。这一节对比各系统的注入路径。

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **注入触发时机** | 调用方主动（API） | `agent_start` hook | `before_prompt_build` hook | `before_prompt_build` hook | Agent 主动调 `skill_search` 工具 | **宿主 0 注入**（委派模式：宿主调 `execute_task` 甩手，OpenSpace 内部自注入到自己的 execution agent） | skill 触发后 agent 读 SKILL.md | Agent 主动 `read` / `ls` | bootstrap 虚拟文件 + 每轮 activator 提醒 | 测试阶段任务开始时 | `before_prompt_build` hook（auto-recall）|
| **注入位置** | 由调用方决定 | System Prompt | System Prompt（hook 追加） | System Prompt（hook 追加） | System Prompt（三级动态加载） | **宿主侧：无**（只收任务结果 + `evolved_skills` 列表）<br>**OpenSpace 内部：自己执行 agent 的 context** | Agent context（读文件进 context） | Agent context（主动 read） | System Prompt 前置 + 每轮 activator | System Prompt | System Prompt（`appendSystemContext`，4 段注入顺序）|
| **注入形式** | 原始摘要文本 | **RewriteMemory** 统一适配建议 | SKILL.md 原文 | SKILL.md 原文 top-k 拼接 | L0 摘要 / L1 概述 / L2 完整 三级 | **宿主 0 注入**——skill 全部在 OpenSpace 内部执行 agent 消费（Plan-then-Select 剪裁版） | 完整 SKILL.md 原文 | 按 URI 读取文件内容 | 虚拟文件预加载 + 每轮提醒 | Experience bullets + Skill 片段（适配后） | **4 段**：`<user-persona>`（L3 正文）+ `<scene-navigation>`（L2 导航）+ `<relevant-memories>`（L1 召回）+ `<memory-tools-guide>` |
| **注入预处理** | 无（原样） | **LLM 改写**（统一建议） | 无（hook 直接注入原文） | LLM Skill 选择（二次筛选） | **三级按需加载** | **LLM Plan-then-Select**（OpenSpace 内部选择注入什么给自己的执行 agent） | 无（全文原样） | 无（按需 read） | bootstrap 虚拟化 | **LLM Rewrite + Adaptation**（Experience 改写 + Skill 剪裁到 ≤ 400 词） | 无预处理（原文召回后原样注入）|
| **注入强度** | 由上层决定 | 建议性 | 直接插入（强） | 直接插入（强） | 中等（system 级指导） | **N/A**（宿主不接收 skill，内部强约束） | **强**（SKILL.md 作工作流约束） | 弱（agent 自行判断） | 中（提醒 + agent 自觉采纳） | **弱**（非强制："consider... your own ideas"） | 中（System Prompt 直接注入，但 `instruction` 优先级未硬置顶）|
| **Agent 感知记忆？** | ❌ 不感知 | ❌ 不感知 | ❌ 不感知 | ❌ 不感知 | ✅ **调工具** | ❌ **宿主不感知**（只知有个能干活的 MCP 工具） | ✅ **读 SKILL.md** | ✅ **主动 read/ls** | ✅ **自觉 append/read** | ❌ 被动接收 | 部分（auto-recall 推 + 工具 `tdai_memory_search`/`tdai_conversation_search` 拉，双通道）|
| **注入 token 开销** | 摘要固定长度 | 重写后长度受控 | SKILL.md 原文 | top-k 拼接（可控） | L0 轻 → L2 重（按需） | **宿主侧零**（只传任务描述 + 拿结果） | 最高（全文） | 中（按需整文件） | 低（摘要 + 每轮少量提醒） | **低**（适配后 ≤ 400 词） | 中高（persona + scene nav + top-k memories，5s 超时兜底）|

### 注入机制的两条主轴

**主轴 1：Push / Pull / 委派 —— 谁在消费 skill？**

```
Push（外部推进宿主 prompt）        Pull（宿主自己拉取）        委派（宿主 0 注入）
────────────────────────────────────────────────────────────────────────────
Mem0 / ReMe / MemOS TS /          Hermes / OpenViking /       OpenSpace
AutoSkill / XSkill                Skill-Creator / pskoett     （宿主只收结果，
（hook 把记忆推进 context，        （agent 调工具/读文件          skill 由 OpenSpace
 宿主 agent 不感知）                宿主 agent 必须懂记忆）       内部 agent 消费）
```

- **Push 派**：外部系统决定注入什么、何时注入；**宿主 agent 不感知** 记忆存在
- **Pull 派**：**宿主 agent 主动拉取**（调工具/读文件）；必须懂记忆系统
- **委派派（OpenSpace 独有）**：宿主把任务**整个扔给 OpenSpace**，OpenSpace 内部是一个带记忆的独立执行 agent，宿主既不感知也不消费 skill —— 只收结果

这条主轴和 §十五 的耦合度分类高度吻合 —— **Push = 解耦**，**Pull = 耦合**（agent 在思考环内参与）。

**主轴 2：原样注入 vs LLM 适配注入**

```
原样注入                                           LLM 适配后注入
────────────────────────────────────────────────────────────────────
Mem0 / MemOS TS / AutoSkill /                     ReMe（Rewrite）
Skill-Creator / pskoett                           Hermes（三级加载）
                                                  OpenSpace（Plan-then-Select）
                                                  XSkill（Rewrite + Adapt）
```

- **原样注入**：直接插入原始 SKILL.md / 文件内容
  - 优势：零预处理成本
  - 劣势：可能过泛（注入无关内容）或过详（占 context window）
- **LLM 适配**：召回后再用 LLM 改写/剪裁/适配到当前任务
  - 优势：token 效率高，贴合当前情境
  - 劣势：多一次 LLM 调用

### 四种独特的注入设计

**Hermes 三级加载**：
```
L0（~200 字）：skill 索引摘要，常驻 system prompt
L1（~500 字）：skill 概述，agent 判断相关才请求
L2（完整 SOP）：显式调 skill_read 才加载
```
分层加载解决"想注入完整 SKILL.md 但太长"的问题 —— 用最少 token 让 agent 知道"存在哪些 skill"，真用的时候再按需展开。

**OpenSpace Plan-then-Select**：
召回 top-k 候选 skill → LLM 根据任务需求生成 `skill_plan`（剪裁版方案）→ 只注入 plan 指定的片段。这是**让 LLM 参与"选择注入什么"**的最精细设计，结果从 MCP 工具响应返回给 agent，token 效率 4-10×。

**XSkill 双重适配**：
```
Experience Rewrite：LLM 按当前任务改写召回的 Experience 条目
Skill Adaptation：LLM 剪裁全局 Skill 到 ≤ 400 词任务特定版
```
召回后做两次 LLM 适配，确保注入内容**贴合当前任务情境**而非原始通用版本。

**pskoett 虚拟文件 + activator 提醒**：
bootstrap 阶段把 LEARNINGS.md / ERRORS.md 作为**虚拟文件**加载进 agent context；每轮 activator 提醒 agent "遇到错误要查 ERRORS.md"。这是**最轻量的 Pull 式**注入 —— 不强塞进 system prompt，agent 需要时主动查。

### 注入机制对任务效果的影响

| 关注点 | 关键维度 | 最佳实践 |
|--------|---------|---------|
| **不被忽略** | 注入强度 + 位置 | System Prompt（强）> Tool response（中）> 文件读取（弱）|
| **精准触发** | 注入预处理 | LLM 适配 > Top-k 拼接 > 原样注入 |
| **成本可控** | 注入形式 + 长度 | 剪裁版 > 摘要 > 三级加载 > 原文 |
| **智能性** | Push vs Pull | Pull（agent 按需）更智能，Push 更稳定 |

**没有银弹**：
- **解耦-Push** 系统（Mem0 / AutoSkill / MemOS）稳定可靠但智能上限受规则限制
- **耦合-Pull** 系统（Hermes / Skill-Creator / pskoett）智能度高但成本贵、可移植性差
- **混合模式**（OpenSpace 外部触发 + LLM 适配 + MCP 工具响应）可能是当前最优平衡点

---

## 十二、离线演进

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **有无离线演进** | **无** | **无**（仅老化删除） | **有**（task-finalize upgrade 路径） | **有**（SkillEvo 遗传） | **有**（多层 Dreaming） | **有**（METRIC_MONITOR 周期扫描 + Tool→Skill 级联修复） | **有**（描述自动优化） | **✗**（无 dreaming / 一致性维护；仅有 Phase 2 异步 extraction + hotness archiver） | ✓（三级升格漏斗） | **有**（层次化合并精炼） | — |
| **触发方式** | — | — | 后台定时扫描 | 后台定时扫描 | Gateway flush + Cron + GEPA | METRIC_MONITOR 周期扫描 + TOOL_DEGRADATION 工具降级 | 用户主动 + run_loop.py 自动 | MemoryArchiver 定期扫描（age>7d + score<0.1 → `_archive/`） | Hook触发+Agent自觉 | 每批 8 样本后自动 | — |
| **演进机制** | — | — | **upgrade 路径**：task 完成回调 → LLM 7 维评估（Faster/Elegant/...）→ 合并新旧 → 附属文件重建 | **遗传算法 + replay**：变异（Heuristic+LLM Mutator）→ 3 轮 replay（dev+test 分集）→ 晋升 | **GEPA** | **Tool 降级 → Skill 级联修复**（独有跨层闭环）+ 4 比率驱动 FIX/DERIVED | **描述优化**（eval→improve→loop, train/test 拆分） | 仅冷记忆下沉到 `_archive/`，默认检索排除但不删除 | Learning→Project Memory→Skill | **三层合并精炼**（逐条/容量缩减/全局 REFINE） | — |
| **评估方式** | — | — | LLM 7 维评估 + `evals/` 测试集 + 质量验证 | replay 评分 + 规则检查（dev/test 分集）| LLM-as-judge + 测试套件 + 基准门控 | GDPVal 50 任务 benchmark + 4 比率聚合 | `claude -p` 黑盒触发率测试 + test score 选最佳 | hotness_score=sigmoid(log1p(count))*exp(-decay*age)，阈值 0.1，最小 age 7 天 | 硬阈值门控 | 对照 ground_truth + 成功/失败轨迹对比 | — |
| **防过拟合** | — | — | 无 | 无 | 约束门：新版 ≥ 旧版 | 双反 loop（`_addressed_degradations` 状态驱动 + min_selections=5 数据驱动） | **60/40 train/test 拆分 + blinded history** | — | 跨2任务+30天窗口 | **64 词硬约束 + "避免具体例子" + 全局精炼去过窄条目** | — |
| **人工参与** | — | — | 无 | 无 | GEPA 需 PR 审查 | 无（全自动）；仅 upload_skill 可选人工发布到公共池 | **内容优化全程人工参与**，描述优化全自动 | 无 | **全程人工**（agent按spec写） | 无 | — |

---

## 十三、演进机制（知识如何成长）

> §十二 讲"**什么时候**演进"（触发），本节讲"**怎么**演进"——每个系统支持哪些演进操作、版本如何管理、质量反馈从哪来。

### 13.1 演进操作矩阵

不同系统支持的演进操作类型差异很大。这决定了记忆库从"新建"走向"精炼"的路径：

| 系统 | ADD<br>新增 | MERGE<br>合并 | UPDATE<br>覆盖 | FORK<br>派生变体 | DELETE<br>删除 | PROMOTE<br>升级 | 版本化 | 独特演进模式 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|------|
| **Mem0** | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ | — 仅追加，历史保真 |
| **ReMe** | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | RewriteMemory 统一重写合并 |
| **MemOS TS** | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ✅（线性）| **upgrade 路径**（7 维评估 + 合并，非遗传）|
| **AutoSkill** | ✅ | ✅ | ❌ | ❌ | ✅ | ❌ | ✅（线性）| 4 维身份判断 + SkillEvo + 使用审计驱动 delete |
| **Hermes** | ✅ | ❌ | ✅ | ❌ | ✅ | ❌ | ✅（patch 链）| **GEPA 遗传进化** + 知识图谱综合推理 |
| **OpenSpace** | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ | ✅（**DAG**）| **FIX / DERIVED / CAPTURED 三态**（独有）+ Tool→Skill 级联 |
| **Skill-Creator** | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ | ✅（iteration-N/）| 人工迭代 + A/B 测试 |
| **OpenViking** | ✅ | ✅ | ❌ | ❌ | ✅（冷归档）| ❌ | ❌ | MERGE / CREATE / SKIP 三分 + hotness 归档 |
| **pskoett SI** | ✅ | ❌ | ❌ | ❌ | ❌ | ✅ | ❌ | **三级升格漏斗**（Learning → Project Memory → Skill，独有）|
| **XSkill** | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌（重编号） | **跨 Rollout Critique + 三层合并精炼** |

**操作定义**：
- **ADD**：新建条目
- **MERGE**：多条相似合成一条（语义并集）
- **UPDATE**：单条内容覆盖（新版替换旧版）
- **FORK**：从现有条目派生变体（保留原条目）
- **DELETE**：物理或逻辑删除
- **PROMOTE**：低级别升到高级别（如 pending → active）
- **版本化**：保留历史版本可追溯

### 13.2 十种系统的演进路径详解

#### Mem0：**只增不改**
```
Session 1 → Session 2 → Session 3 → ...
  [M1]       [M1, M2]    [M1, M2, M3]
```
历史保真优先，不做任何合并或演进 —— 记忆库线性膨胀，适合需要完整审计的场景。

#### ReMe：**向量合并覆盖**
```
新记忆 M' → 向量相似度命中旧记忆 M
         → RewriteMemory LLM 重写为统一版本 M''
         → M'' 覆盖 M（单一演进方向：合并成更好的）
         → utility/freq 双维度老化删除
```
**演进只有一个方向**：趋于统一。没有派生/分支。

#### MemOS TS：**Task-finalize upgrade 路径**
```
Task 完成 → SkillEvolver.onTaskCompleted()
         ├─ 搜索相似 skill
         ├─ 无匹配 → 走创建路径（add）
         └─ 命中   → 走升级路径：
             ├─ Step 7b 升级评估（7 维度：Faster/Elegant/Convenient/
             │                     Fewer tokens/Accurate/Robust/New scenario/
             │                     Fixes outdated）
             ├─ Step 8b 升级合并（LLM 融合新旧内容）
             ├─ Step 9b 附属文件重建（scripts/refs/evals 3× LLM）
             └─ Step 11 质量验证
```
**不是遗传算法**——是 task 完成后的 LLM 判断型演进，决定"这次任务对已有 skill 是否有实质改进"。没有变异/种群/选择概念。

#### AutoSkill：**SkillEvo 遗传 + replay**（独立离线工具）
```
                  当前 champion skill
                        │
                        ▼
         ┌─ Heuristic 变异（机械拼接）
         ├─ LLM Mutator 变异（基于失败样本改写）
         │   ↓
         │  2-N 个变体
         │   ↓
  第 1 轮 replay（dev 集）: 用 champion 跑 → 找失败样本
  第 2 轮 replay（dev 集）: 用每个变体跑 → 打分选最优
  第 3 轮 replay（test 集）: 最优变体 vs champion 跑
                        │
                        ▼
              胜出 → promote 为新 champion
              未胜出 → 保留原 champion
```
**遗传 = 变异 + 选择 + 晋升**；**replay = 用历史 user 消息作"考题"，不在真实新对话试错**。
dev/test 分集防过拟合。**独有**：使用审计闭环（retrieved→relevant→used），未使用的 skill 自动 delete。

#### Hermes：**GEPA 图谱遗传**
```
Session 结束
  └─ Gateway flush → Dreaming Pipeline
      ├─ 提取 new skills
      ├─ patch 已有 skills（LLM 自主 edit）
      ├─ 合并到知识图谱节点
      └─ GEPA 遗传优化（约束门：新版 ≥ 旧版）
         需 PR 审查
```
**特色**：知识图谱节点 + 遗传进化，跨 skill 综合推理。

#### OpenSpace：**三态 + DAG**（独有）
```
ANALYSIS trigger:
  evolution_suggestions → {FIX / DERIVED / CAPTURED}
    │
    ├─ FIX: 单父覆盖（同 name 同 path）→ is_active 切到新版本
    ├─ DERIVED: 多父 merge / 单父派生（新 name）→ DAG 新枝
    └─ CAPTURED: 无父节点（全新捕获）→ DAG 新根

TOOL_DEGRADATION trigger:
  工具质量下降 → 依赖 skill 批量 FIX（独有跨层闭环）

METRIC_MONITOR trigger:
  周期扫 4 比率（applied/completion/effective/fallback）
  → 失效 skill 触发 FIX
```
**唯一**支持 DAG 版本管理 + 多父 merge + 派生变体的系统。

#### Skill-Creator：**人工 A/B 迭代**
```
SKILL.md v1 → 跑 eval
            → 子代理 Grader/Comparator/Analyzer 出报告
            → 人工评审 feedback.json
            → 改进 SKILL.md v2 → 跑 eval → 人工 ...
            → 直到满意
```
演进靠**人机迭代**而非自动，适合少量关键 skill 精雕细琢。

#### OpenViking：**三态 + 冷归档**
```
新 memory → VLM 向量预筛
         ├─ 高相似 → MERGE（LLM 决策合并）
         ├─ 中相似 → CREATE（新条目）
         └─ 低价值 → SKIP（直接丢）

MemoryArchiver 定期扫描
  └─ age > 7d + hotness_score < 0.1
     → 下沉到 _archive/（检索默认排除，但不物理删除）
```
**冷归档**思路：低价值记忆不删，只"移出视野"，可恢复。

#### pskoett SI：**三级升格漏斗**（独有）
```
Level 1: LEARNINGS.md（Learning）
   ↓  Recurrence ≥ 3 + 跨 2 任务 + 30 天
Level 2: Project Memory
   ↓  广泛适用 + 稳定
Level 3: Skill（独立 SKILL.md）
```
**多层级演进**——知识逐级从具体（单次 learning）抽象到通用（可复用 skill）。每升一级要过硬阈值门。

#### XSkill：**跨轨迹批判 + 三层合并**
```
N 条 Rollout 轨迹
  └─ Cross-Rollout Critique
     → [{add, e}, {modify, E17, e'}]

Experience 合并三层：
  Layer 1 逐条处理：嵌入 >0.70 → LLM 合并
  Layer 2 容量限制：>120 条 → 强制合并最相似对
  Layer 3 全局精炼：每 8 样本后 REFINE（merge/generalize/delete）
```
**多路对比 + 层次化合并**是独有设计。

### 13.3 版本管理与历史追溯

| 系统 | 版本模型 | 历史保留 | 回滚支持 |
|------|---------|---------|---------|
| Mem0 | 无版本 | 全部保留（SQLite 审计表） | 靠外部代码 |
| ReMe | 覆盖 | 删除标记 + 频率追踪 | ❌ |
| MemOS TS | 线性递增 | chunk.dedupStatus 标记 | ❌ |
| AutoSkill | 线性递增 | 版本号 | ❌ |
| Hermes | patch 链 | 完整 patch 历史 | ⚠️ 需人工 |
| **OpenSpace** | **DAG**（独有多父 merge） | content_snapshot + content_diff 全保留 | ✅ **可回滚到任意版本** |
| Skill-Creator | `iteration-N/` 目录 | 所有迭代保留 | ✅ 直接 checkout 旧目录 |
| OpenViking | 线性 | archive_NNN/messages.jsonl 顺序归档 | ✅ 从归档恢复 |
| pskoett | 无版本 | 完整保留所有 entry | 靠 git |
| XSkill | 重编号（E0, E1, E2...） | 覆盖 | ❌ |

**OpenSpace 的版本 DAG 是独一档的设计**：
- 支持多父 merge（两个 skill 合并派生新的）
- 保留 snapshot + diff 双形式（人读 + 机器恢复）
- 任意历史版本可激活回滚

### 13.4 质量反馈来源（演进依据）

演进的依据从哪来 —— 决定了演进方向是否"对"：

| 系统 | 主要反馈源 | 反馈类型 |
|------|---------|---------|
| Mem0 | 无 | —（不演进）|
| ReMe | utility / freq 统计 | 量化（使用频次）+ 5 维 LLM 验证（质量）|
| MemOS TS | LLM 0-10 评分 | LLM 主观判断 |
| AutoSkill | **使用审计闭环**（retrieved→relevant→used）| 客观使用数据 + 4 维 LLM |
| Hermes | LLM-as-judge + 基准测试 | 主观 + 基准 |
| **OpenSpace** | **4 比率 + tool penalty**（多维客观）| 最全面：applied / completion / effective / fallback + 工具健康度 |
| Skill-Creator | `claude -p` 黑盒测试 + 子代理评审 | 客观测试 + 人工最终决策 |
| OpenViking | hotness_score（sigmoid 热度衰减）| 使用频次 |
| pskoett | 人工判断 + Recurrence ≥ 3 硬阈值 | 硬统计门槛 |
| XSkill | 对照 ground_truth + 跨 rollout 对比 | 标准答案 + 多路批判 |

**反馈类型的价值排序**：
1. **有标准答案（XSkill）**：最可靠但场景受限
2. **使用审计客观数据（AutoSkill、OpenSpace）**：跨场景通用、免 LLM
3. **LLM 评分（MemOS / Hermes）**：灵活但有谄媚风险
4. **使用频次（ReMe / OpenViking）**：简单但易被误点击污染
5. **人工评审（Skill-Creator / pskoett）**：最可靠但不可规模化

### 13.5 四种演进范式

总结起来，这 10 个系统的演进范式可归为四类：

| 范式 | 特征 | 代表 |
|------|------|------|
| **追加型**（Append-only）| 只增不改，历史保真 | Mem0 |
| **合并型**（Converge）| 相似条目合并成一个"更好版本" | ReMe, MemOS, AutoSkill, XSkill |
| **分支型**（Branch）| 同一知识可分派多个变体 | **OpenSpace**（FIX + DERIVED 双路径）|
| **升格型**（Promote）| 知识分级别，逐级筛选提升 | **pskoett**（三级漏斗）|

**组合型**：多数系统同时有合并 + 删除，OpenSpace 同时有合并 + 分支（最丰富），pskoett 同时有追加 + 升格。

---

## 十四、LLM 调用开销（学习 + 演进）

开销分两段：**学习成本**（单次任务的提取/合并）和**演进成本**（记忆库长期迭代的 refine / evolve / prune）。两者**不能混算**——有些系统学习便宜但演进很贵（Hermes GEPA），有些系统学习就一次到位（Mem0）没有演进。

### 14.1 学习/提取成本（单次任务）

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **单次提取调用** | **1 次** | **3-4 次** | **5-9 次** | **5-8 次/轮** | **1 次**（agent 的 `skill_manage` 工具调用） | agent loop ≤5 轮 + apply-retry ≤3 次（patch-first） | **~2-3M tokens/技能** | 2-5k tokens×N类别 | **无独立调用，摊入 agent 自身**（每轮读 spec + grep + 写入 + 升格判断，+500-2k tokens） | **N+3+K 次/样本**（N=rollout 数） | — |
| **调用明细** | 提取摘要 | 提取(1)+验证(1)+重排(1)+重写(1) | 摘要(1)+去重(1)+分类(1)+评估(1)+生成(1)+验证(1)... | 改写(1)+embed(1)+选择(1)+提取(1)+决策(1)+合并(1)+审计(1) | 生成 SKILL.md(1)，作为 agent 自觉工具调用 | analysis(1) + 确认(1) + fix/derived/captured(1) + apply-retry(≤3) | 内容优化: 6子代理/eval×N轮 + 描述优化: 60×`claude -p`/轮×5轮 | VLM抽取(1)+dedup向量过滤+LLM决策 | **Agent 自身推理承担**：读 SKILL.md 元指令、grep LEARNINGS.md、判断是否升格、写入——全在 agent 思考环里 | 图像分析(M)+Rollout Summary(N)+Critique(1)+Experience合并(0-K)+Skill生成(1) | — |
| **成本特征** | 最低 | 中等 | 较高 | 最高（每轮都跑） | **最低**（但质量依赖模型能力；是 agent 自调工具，非独立 pipeline） | 中高（patch-first token 效率 4-10×，摊薄在触发事件上） | **极高**（精品路线，适合少量关键技能） | 中等（随session大小线性增长） | **无独立 pipeline 但非零**（每轮 +500-2k tokens 摊入 agent 自身，长会话累计可能反超 Mem0 / Hermes） | 较高（但 batch 摊薄，每批 8 样本共享合并精炼） | — |
| **延迟敏感** | 同步/异步均可 | 后台异步 | 后台异步 | 后台异步 | 在线同步（LLM 自主调用工具） | 后台异步（任务完成后触发，不阻塞 agent） | 人机交互（不要求实时） | Phase1同步归档+Phase2异步抽取 | **同步**（agent 思考环内做，不阻塞但增加当轮推理成本） | 离线 batch（不要求实时） | — |

### 14.2 演进/迭代成本（记忆库长期生命周期）

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **有无独立演进 pipeline** | ❌ 无 | ⚠️ 仅老化（规则，0 LLM） | ✅ upgrade 路径（嵌入 task finalize 回调） | ✅ SkillEvo 遗传 + replay（独立离线工具）| ✅ **GEPA 最重** | ✅ **3 触发器**（最全） | ✅ 描述优化 loop | ⚠️ MemoryArchiver（规则，0 LLM） | ❌ 摊入 agent | ✅ 三层合并精炼 | — |
| **单次演进调用** | — | 0 LLM（老化靠规则）| **M 个候选 × (replay + eval)**（遗传代数）| 同 MemOS + 使用审计（规则）| **GEPA 每代 ~N×LLM 变异 + judge + benchmark**（代数叠加） | ANALYSIS 每任务 3-5 次<br>TOOL_DEGRADATION 级联 M 个依赖 skill × 1-3 次<br>METRIC 周期 × M skills | **60 × `claude -p` × 5 轮**（描述优化单次训练）| 0 LLM（hotness 计算 + 归档） | **Agent 自身承担**（每次自觉判断升格 → 额外推理）| 每批 8 样本 REFINE(1) + 容量>120 强制合并(1-3) | — |
| **演进触发频率** | — | 定时低频 | 高（每 task finalize 都触发 SkillEvolver 回调）| 中（SkillEvo 离线批次 replay）| **高**（Gateway flush + Cron + GEPA 持续演化）| **高**（3 触发器并行，每任务都可能触发）| 低（用户主动 / run_loop 批训）| 低（仅归档）| **高**（每次任务都可能升格判断）| 中（batch=8 触发）| — |
| **演进阻塞 agent？** | — | ❌ 异步 | ❌ 异步 | ❌ 异步 | ❌ 异步（但 GEPA 需 PR） | ❌ 异步（任务完成后） | ❌ 离线（人机交互） | ❌ 异步 | **✅ 同步**（思考环内判断）| ❌ 离线 batch | — |
| **长期总成本特征** | 最低（无演进）| 最低（规则老化） | 中（每任务 upgrade 路径 4 次 LLM，不做遗传 replay）| 中高（遗传 replay 成本随 skill 数 × 代数）| **极高**（GEPA 持续消耗，但质量门控最严）| **高但可控**（3 触发器精准触发，patch-first 4-10× 节省）| **极高**（精品路线总摊销大）| 最低（归档免费）| **中高**（agent 每轮都做升格，长期累积大）| 中（每批摊薄）| — |

### 14.3 两段成本对照

| 系统 | 学习成本 | 演进成本 | 总体特征 |
|------|---------|---------|---------|
| **Mem0** | 低 | 无 | **只学不进**，适合历史保真场景 |
| **ReMe** | 中 | 极低（规则）| 学习为主，演进靠老化 |
| **MemOS TS** | 较高 | 中高 | 双段都有成本，靠后台吸收 |
| **AutoSkill** | 最高 | 中高 | 学习每轮都跑，演进叠加 |
| **Hermes** | **极低**（1 次） | **极高**（GEPA 最重） | **"学得快但进化贵"**——极端案例 |
| **OpenSpace** | 中高 | **高但可控** | 学进并重，patch-first 节省 |
| **Skill-Creator** | 极高 | 极高 | 学进都贵，精品路线 |
| **OpenViking** | 中 | 极低（归档免费）| 演进弱 |
| **pskoett SI** | 摊入 agent | 摊入 agent | **隐性成本**，学进都藏在 agent 推理里 |
| **XSkill** | 较高 | 中 | 演进靠 batch 摊薄 |

**关键观察**：
- **Hermes 是典型的"学习极简 + 演进极重"**——agent 写一次 skill_manage 很便宜，但 GEPA 持续演化随时间成本爆炸
- **OpenSpace 是学进最均衡**——3 触发器精准触发（非定时扫描），patch-first 节省 4-10× token
- **耦合型系统（Hermes / Skill-Creator / pskoett）的演进成本都比看起来高**——要么有独立 GEPA pipeline（Hermes），要么摊入 agent 推理（pskoett），要么人工 + LLM 双重消耗（Skill-Creator）

---

## 十五、独特优势与劣势

### Mem0

**生成/学习**
| 优势 | 劣势 |
|------|------|
| 实现最简单，1 次 LLM 调用 | 无质量门控，可能积累噪声 |
| 历史保真，中断恢复最精确 | 产物不可复用，每次都从头开始 |
| 通用基础设施，不绑定上层结构 | 无通用化，跨任务无法迁移 |
| 支持 20+ 向量 DB 后端 | — |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| — | **无离线演进**（追加型，记忆只增不减） |
| — | 无合并/删除/升级操作 |

### ReMe

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **失败轨迹提取**（独有） | 经验建议不可直接执行 |
| **when_to_use 向量化**检索精准 | 产物形态松散，无结构化约束 |
| LoCoMo SOTA（86.23） | 去重只靠向量相似度，无 LLM 语义判断 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **RewriteMemory** 合并时统一重写，注入适配度高 | **无离线主动演进** |
| **utility/freq 双维度老化**防退化 | 只能老化删除，不能合并优化 |

### MemOS TS

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **产物最完整**（Skill + 脚本 + 评估集） | LLM 调用开销大（5-9 次） |
| **两级去重**（hash + 语义 LLM） | 实现复杂度高 |
| **任务自动分割**（LLM 判断话题切换） | — |
| 质量评分 0-10 精细化 | — |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **Task-finalize upgrade 路径**（7 维评估 + LLM 合并 + 附属文件重建，**非遗传算法**）| **被动升级**（用户再做才触发） |
| 版本号递增线性演进 | 无使用审计闭环 |
| 每次触发只跑一次 LLM 评估，开销可控 | 无变体竞争机制（不像 AutoSkill 能在多个变体里选最优）|

### AutoSkill

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **查询改写**解决代词引用问题 | Sidecar 每轮 5-6 次 LLM 调用，开销大 |
| **LLM Skill 选择**二次筛选保精准 | Embedded 版功能裁剪严重 |
| 证据溯源严格（仅 user 消息） | **仅成功轨迹**（`successOnly=true`），无失败轨迹学习 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **SkillEvo 真·遗传进化**（独立离线工具：变异 + 3 轮 replay + dev/test 分集防过拟合 + 晋升）| 遗传 replay 成本可观（champion + N 变体 × 3 轮 × |dev|+|test| 次 Responder 调用） |
| **使用审计闭环**（retrieved→relevant→used）驱动自动淘汰 | 审计仅记录使用状态，不反馈给 SkillEvo 做梯度演进 |

### Hermes

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **实现最简洁**（无独立管线，1 次 `skill_manage` 工具调用）| 质量完全依赖 LLM 能力 |
| **冻结快照**保 prefix cache 性能 | 无自动质量门控 |
| — | 无使用审计 |
| — | 是 agent 自觉工具调用，弱模型可能漏记 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **GEPA** 是目前最完整的进化闭环 | **GEPA 成本最高**（每代 N 次 LLM 变异 + judge + benchmark）|
| **多层 Dreaming**（6 层离线整理）| GEPA 需人工 PR 审查 |
| **知识图谱**支持跨记忆综合推理 | 约束门仅"新版 ≥ 旧版"，缺分支/派生能力 |
| **容量压力驱动整合**（类人脑遗忘）| 配置复杂度高 |
| 约束门：新版 ≥ 旧版，防过拟合 | — |

### OpenSpace

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **委派执行模式**（独有）：宿主 0 记忆负担，OpenSpace 内部有独立执行 agent 消费 skill | 委派模式 = 多一层 agent-to-agent 调用，不适合延迟敏感的短任务 |
| **Apply-retry 回喂磁盘真实内容** → 避免 LLM 幻觉 patch | 宿主失去对 skill 使用的可见性（完全黑盒） |
| **Patch-first 策略 + 4 级降级匹配 + Unicode 规范化** → token 4-10× 节省 | 任务完成信号模糊时 analyzer 的 `task_completed` 可能不准 |
| **MCP 统一接入多 host**（Claude Code / Codex / OpenClaw / nanobot / Cursor）跨生态 | 依赖 OpenAI embedding（text-embedding-3-small）跨端兼容 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **三演进 × 三触发的 9 元组空间**（FIX/DERIVED/CAPTURED × ANALYSIS/TOOL_DEGRADATION/METRIC_MONITOR）| 架构复杂度最高（skill_engine 目录 ~7800 行），学习曲线陡 |
| **Tool 质量 → Skill 级联修复**（跨层闭环，独有）| Trigger 2/3 每次多一次 LLM 确认调用，延迟增加 |
| **LLM 确认门 + 双反 loop**（状态驱动 + 数据驱动）规避误报→反复演进陷阱 | 规则阈值需人工调优（_FALLBACK_THRESHOLD=0.4 等常量） |
| **版本 DAG（多父 merge 支持）+ content_diff + content_snapshot** → 兼顾人读与机器恢复，支持任意版本回滚 | 纯文本 skill，不支持多模态（vs XSkill 视觉锚定） |
| **4 比率 + tool penalty 使用审计最完整**（applied/completion/effective/fallback 原子递增） | — |
| **云端协作**（public/private/group + diff-based 改进凭证）| — |
| **GDPVal 实测**：50 任务 4.2× 收入 / Phase2 -54% tokens（Qwen 3.5-Plus 同骨干对照）| — |
| **IM 网关**（WhatsApp Baileys + Feishu webhook）突破 MCP 边界 | — |

### Skill-Creator

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **4 子代理并行测试**（Grader/Comparator/Analyzer） | **无自动提取能力**——依赖人工提需求 |
| **`claude -p` 黑盒端到端测试**触发率 | 需要人工全程参与 |
| eval-viewer SPA 精心设计的人工评审体验 | 仅限 Claude Code 生态 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **优化即核心流程**（创建=测试=改进循环） | **token 消耗极高**（~2-3M/技能） |
| **描述优化有防过拟合**（60/40 train/test 拆分 + blinded history） | 不适合批量生产 |
| `iteration-N/` 目录保留所有迭代，可回滚 | 描述优化 60 × `claude -p` × 5 轮，重型演进 |

### XSkill

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **双流分离**（Experience 行动级 + Skill 任务级，消融实验证明互补） | 需要 ground_truth 标准答案 |
| **多路 Rollout 对比**（N=4 成功/失败自然对比） | 计算开销较大（N 次独立执行） |
| **视觉锚定**（图像+轨迹联合分析，其他系统均为纯文本）| 仅适用于多模态视觉推理场景 |
| **跨模型知识迁移**（MLLMexec ≠ MLLMkb） | 框架相对较新，生态不成熟 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **嵌入驱动合并**（粗筛快 + 精合并准，平衡成本质量）| Experience 建议性注入，Agent 可能忽略 |
| **三层合并精炼**（逐条 / 容量缩减 / 全局 REFINE）| 无版本管理（重编号覆盖） |
| **64 词硬约束**系统性对抗知识膨胀 | 演进依赖有 ground_truth 的场景 |
| 每批 8 样本后 REFINE，成本摊薄 | 容量硬限 120 条可能导致早期优质条目被误合并 |

### OpenViking

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **文件系统范式**（viking:// URI 统一标识）直观可治理 | Phase 2 抽取 fire-and-forget，**无重试无 SLA**，崩溃即永久丢失 |
| **L0/L1/L2 三级粒度**存储（abstract / overview / full） | "progressive loading"仅是文档口号，**运行时需 Agent 显式调用**每一层 |
| **8 类单标签分类**（User / Agent 双侧分离） | **单标签**吃掉跨类语义；**缺失 Prospective** 类别 |
| **异步两阶段 commit**（同步归档 + 异步抽取）不阻塞主循环 | 手动 commit 容易忘 → 消息丢；无自动触发、无 incremental extraction |
| ToolPart 保留工具调用语义 | ToolPart 输出**截断 500 字符**，长日志关键信息会丢 |
| **目录递归检索**（IntentAnalyzer→优先队列，α=0.5 score 传播） | score 传播 50/50 过激 → 深层 fact 必降分 |
| — | **Dedup SKIP 硬丢弃无审计**，被丢的候选无法追溯 |
| — | 无显式失败轨迹学习机制；无质量门控 |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **MemoryArchiver 热度老化**（hotness × exp(-decay·age)）冷记忆下沉到 `_archive/` | **无 dreaming / 一致性维护**，仅归档不合并 |
| `archive_NNN/messages.jsonl` 顺序归档可恢复 | 不可变类别（EVENTS/CASES）**只加不删**，长会话必然膨胀 |
| 归档全程零 LLM（规则计算） | 无真正"演进"——知识不会被改写或抽象 |

> 注：Comparison 早期版本曾写过 "LoCoMo10 benchmark 降 83% token" / "检索轨迹可视化" / "OpenClaw 集成" 等条目，但在 [OpenViking_分析.md](./OpenViking_分析.md) 中均**未找到佐证**，已从本对比中移除。如有补充信息（如官方 README / benchmark 报告），请直接更新分析文档后再同步。

### pskoett SI

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **无独立 LLM pipeline** — 完全靠 agent 按 spec 写 Markdown，部署零基础设施 | **成本非零**：开销摊入 agent 自身（每轮读 spec + grep + 写入 + 升格判断，+500-2k tokens），长会话累计可能反超 Mem0 / Hermes |
| **极简实现**（SKILL.md 573 行 + 2 个 bash hook，总代码 <100 行） | 无自动检索（靠 pre-flight-check 主动 surface） |
| **Pattern-Key 稳定去重键**（人工分配，绕过 LLM 改写漂移） | Pattern-Key 需人工约定，两 agent 分别用不同 key 仍会重复 |
| **极高可移植性** — 跨 Claude Code / Codex / Copilot / OpenClaw | 依赖 agent 理解 SKILL.md 元 spec，弱模型可能读不懂 |
| **skill-as-system 范式** — procedural memory 从"后端服务"降维成"前端能力" | 不适合企业级审计场景（无结构化存储、无审计表） |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **三级升格漏斗**（Learning → Project Memory → Skill）层次清晰 | **Markdown 线性增长**，无容量管理，持续膨胀 |
| **Recurrence ≥ 3 + 跨 2 任务 + 30 天**多维硬阈值防过早升格 | error-detector.sh 是粗糙 grep，误报率高 |
| **Inner/Outer Loop 概念化** — 架构思路最清晰的一套 | **升格判断主观性强**（"是否广泛适用"靠 agent 自评） |
| 与 simplify-and-harden / learning-aggregator / harness-updater 形成完整 outer loop 生态 | 升格每次都要 agent 参与推理，长期累计成本可观 |
| — | 无跨项目共享（`.learnings/` 是本地目录）|

### memory-tencentdb

**生成/学习**
| 优势 | 劣势 |
|------|------|
| **四层分层架构**（L0 对话 → L1 结构化 → L2 场景叙事 → L3 用户画像）层次清晰 | **程序记忆折叠进 `instruction`**，无独立 procedural 类型 |
| **三类 L1 记忆**（persona / episodic / **instruction**）明确分档 | **`priority=-1` 声明但未实际生效**（召回注入未硬置顶）|
| **双后端**（本地 SQLite+sqlite-vec+FTS5 / 腾讯云 TCVDB+BM25 sparse） | 工具调用限制（每轮 3 次）是**软约束**，TODO 未实现硬限 |
| **Hybrid 检索**（FTS5 + 向量 + RRF 融合），5s 超时兜底 | **L1 `looksLikePromptInjection()` 已实现但被注释掉**，目前不拒绝疑似注入 |
| **L2 workspace 硬隔离**（scene_blocks/）+ 软删除 | instruction 缺 scope / expiresAt / supersedes 等生命周期字段 |
| **Skill wrapper sanitizer**（过滤 `¥¥[...]¥¥` 防污染）| 失败轨迹学习缺失 |
| **clean-context LLM runner** 防主对话上下文污染 | 无独立 procedural 测试 / benchmark |
| 结构化 reporter（agent_turn / tool_call / l1/l2/l3_extraction 等）| — |
| **CleanContextRunner** 空上下文跑 LLM，避免召回记忆再次进新记忆 | — |

**演进/迭代**
| 优势 | 劣势 |
|------|------|
| **L2 场景可 UPDATE / MERGE**（LLM 作 "Memory Consolidation Architect"）| **instruction 本身不演进**（只有 store/skip/update/merge 四动作决策）|
| **L3 persona 四层深度扫描**（基础锚点 / 兴趣 / 交互协议 / 认知内核）汇总 instruction 和偏好 | persona 全量重生成，没有增量更新 |
| L2/L3 触发条件多元（PERSONA_UPDATE_REQUEST 信号 / 冷启动首次 / memories_since_last_persona ≥ triggerEveryN） | 无 dedup 历史审计——**SKIP 动作硬丢弃不留痕** |
| **Profile Sync 机制**（本地文件 ↔ 远端向量库，baseline version 乐观锁） | L2 场景数量约束仅靠 prompt，工程侧无兜底 |
| **MemoryArchiver 式归档**：L0/L1 retentionDays 清理（可配 allowAggressiveCleanup）| Seed CLI 不等 L2/L3 idle，seed 输出可能不含最新 artifacts |
| 有备份目录 `.backup/` 滚动保留 | JSONL append-only 与 VectorStore 可能短暂不一致 |

**所属档位**：类比 **Mem0 / OpenViking** ——**通用记忆基础设施**中**弱程序记忆支持**的代表。`instruction` 是最接近 GSPD Fb 的类型，但缺强度分级、跨任务触发、候选池升格。

---

## 十六、架构耦合度

> **分类判据**：**记忆写入的执行主体是否是 agent 自己？**
> - **解耦**：记忆操作由外部系统处理（hook / 定时器 / 独立服务），agent 不参与记忆决策，也无需了解记忆系统存在
> - **耦合**：记忆写入发生在 agent 思考环内（自觉调工具 / 自觉 append 文件 / 读元指令自执行），依赖 agent 自身能力

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **OpenSpace** | **Skill-Creator** | **OpenViking** | **pskoett SI** | **XSkill** | **memory-tencentdb** |
| ------ | ---------- | ---------- | -------------- | --------------- | ------------ | ------------ | ------------------- | --- | --- | ------------ | ----- |
| **耦合度** | **解耦-开放协议** | **解耦-框架 hook** | **解耦-框架 hook** | **解耦-框架 hook** | **耦合-agent 自觉** | **解耦-委派模式**（MCP，宿主 0 记忆负担） | **耦合-agent 执行** | **解耦-专有接口** | **耦合-agent 自觉** | **解耦-独立框架** | **解耦-框架 hook**（OpenClaw 插件）|
| **记忆执行主体** | 外部调用方 | 外部 hook 触发管线 | 外部 TS 模块（timer+hook） | 外部 JS/Python（hook 触发） | **Agent 自己**（`skill_manage` 工具） | **OpenSpace 内部的独立执行 agent**（宿主委派整任务过来，skill 由 OpenSpace 自己消费并演进） | **Agent 自己**（读 SKILL.md 元指令并执行评估循环） | 外部服务（`session.commit()`） | **Agent 自己**（Hook 提醒后自觉 append Markdown） | 外部 batch pipeline | 外部 `MemoryPipelineManager`（serial queue + checkpoint）|
| **运行方式** | 独立服务 / HTTP API | HTTP 服务 / 文件系统版 | TypeScript 内置模块（有 API 边界）| OpenClaw 插件（Embedded/Sidecar） | Agent 内置功能 | MCP Server（跨 host：Claude Code / Codex / OpenClaw / nanobot / Cursor） | Skill-Creator SKILL.md + agent 执行 | 独立 Python 库 / HTTP 服务 | Markdown + bash hook + agent 执行 | 独立 Python 框架（可换骨干） | OpenClaw 插件 + 后台 L1/L2/L3 调度器 |
| **依赖关系** | 零依赖（任意 agent 可调用） | 依赖 AgentScope 或独立 HTTP | 依赖 TS-end 框架 hook 系统 | 依赖 OpenClaw hook 系统 | **依赖 agent LLM 能力**（要会主动记忆） | 仅需 MCP 协议 + LLM | **依赖 Claude Code agent 能力**（要会读元指令自运行）| 依赖 VLM + 向量后端（agent 框架无关） | **依赖 agent 理解 SKILL.md 并自觉记录** | 依赖 OpenAI embedding，骨干模型可换 | 依赖 OpenClaw hook + SQLite/TCVDB 后端 + OpenAI-compatible embedding |
| **框架迁移** | **最容易**（纯 API） | 容易（文件版人类可读） | 困难（深度绑定 TS-end）| 困难（OpenClaw hook 专属） | **不可能**（记忆内嵌 agent）| **最容易**（MCP 通用，host 即插即用） | 困难（Claude 生态 + agent 能力依赖） | 较容易（`viking://` URI 可迁移）| **困难**（需 agent 完整理解 573 行 SKILL.md 元 spec） | 较容易（知识 JSON/Markdown 可迁移）| 困难（OpenClaw hook 专属）|
| **多 Agent 共享** | **支持**（通过 `agent_id` / `user_id`） | 支持（通过 workspace） | 支持（通过作用域） | 支持（user / library scope） | 支持（通过 skill 目录） | **原生支持**（云端 public/private 池 + 跨 MCP host） | 支持（`.skill` 文件分发） | 支持（通过 `user_space`） | 可手动复制 `.learnings/` 目录 | **支持**（experiences.json + SKILL.md 跨模型迁移） | 支持（通过 session scope + TCVDB 云端共享）|

### 耦合度分析

```
解耦（外部处理，宿主不感知）   │   委派（宿主 0 负担）   │   耦合（宿主 agent 思考环内）
                              │                         │
Mem0  XSkill  ReMe  AutoSkill │      OpenSpace          │  Skill-Creator  pskoett   Hermes
(HTTP)(独立) (HTTP) (OpenClaw │   (MCP 委派整任务)       │  (读 SKILL.md   (自觉     (skill_manage
             MemOS TS  OpenViking│  OpenSpace 内部有       │   自跑评估)     append    工具)
             (TS hook) (viking://│   独立执行 agent)        │                 Markdown)
                                │                         │
                                │                         │
       ← 外部 pipeline →        │  ← 完整外包 →         │  ← 宿主内部执行 →
                                │                         │
                             核心判据：
                           宿主 agent 在做记忆操作时
                            是否必须"在场"参与？
```

**三类的核心差异**：

| | **解耦**（6 家） | **委派**（OpenSpace 独有） | **耦合**（3 家） |
|---|----------------|------------------------|---------------|
| **宿主 agent 负担** | 无（外部写入，透明）| **零**（任务都外包）| 高（自己做记忆决策）|
| **智能决策** | 较低（hook 定时器兜底）| 高（OpenSpace 内部 agent 智能）| 高（宿主 agent 自己判断）|
| **Prompt 成本** | 低 | **最低**（宿主只传任务描述）| 高（要引导记忆决策）|
| **可移植性** | 好 | **最好**（换宿主无影响）| 差（换 agent = 记忆逻辑失效）|
| **LLM 能力门槛** | 低 | 宿主低，OpenSpace 内部高 | 高（需强模型）|
| **典型代表** | Mem0（即插即用）| OpenSpace（委派专业 agent）| Hermes（思考环内建）|

**一句话总结三类**：
- **解耦** = "**外部系统自动记录**"——稳定、可移植，智能上限受规则限制
- **委派** = "**把活儿整个外包给带记忆的专业 agent**"——宿主 0 负担，但多一次 agent-to-agent 调用
- **耦合** = "**让宿主 agent 自己会自省**"——更智能但成本高、强依赖 agent 能力

---

## 十七、设计光谱

```
原始保真 ←────────────────────────────────────────────────────────────────→ 通用抽象

Mem0   ReMe    pskoett SI   MemOS    AutoSkill  Hermes  OpenSpace   Skill-Creator   XSkill
(快照) (经验)  (日志漏斗)   (技能包)  (方法论)   (进化)   (自演进)    (精品课程)      (双流)
 │       │         │          │         │         │         │            │            │
 ▼       ▼         ▼          ▼         ▼         ▼         ▼            ▼            ▼
verbatim 建议性  Markdown    可执行    可执行+  可执行+  可执行+    测试验证的   Experience≤64词
记录     bullet  三级升格    SOP      审计闭环 离线进化  三演进闭环 可执行SOP    + Skill SOP


单路径提取 ←────────────────────────────────────────────→ 多路径对比

Mem0 ReMe MemOS AutoSkill Hermes OpenSpace pskoett SI  v5设计      Skill-Creator    XSkill
(无) (无)  (无)   (无)    (无)   (无)       (无)      (会话内检测)  (刻意构造A/B)  (N=4 Rollout)
                                                         │              │              │
                                                         ▼              ▼              ▼
                                                    零LLM正则扫     子代理并行spawn  独立执行对比
                                                    切分Path A/B    with/without     成功/失败


依赖 LLM 抽取 ←───────────────────────────────────────────→ 依赖 Spec 让 Agent 自己做

Mem0   MemOS  AutoSkill  XSkill   ReMe    Hermes       OpenSpace     Skill-Creator   pskoett SI
(API)  (代码)  (代码)    (batch)  (代码)  (LLM+工具)    (规则+LLM)     (人机迭代)      (纯 Spec)
                                                         ▲                                ▲
                                                         │                                │
                                                 三触发×三演进闭环            skill-as-system 范式：
                                                 (唯一跨层 Tool→Skill 级联)    把 procedural memory 从
                                                                               "后端服务"降成"前端能力"


演进闭环完整度 ←────────────────────────────────────────────→ 跨层级联（独有）

pskoett SI  Mem0  Skill-Creator  ReMe   MemOS       AutoSkill    Hermes        OpenSpace
(升格漏斗)  (无)   (人机评审)    (老化) (upgrade路径) (SkillEvo遗传) (GEPA+Dream)  (Tool→Skill 级联)
                                                                              ▲
                                                                              │
                                                                    唯一做"工具质量下降
                                                                    → 依赖它的 skill 批量修"
                                                                    的跨层闭环系统
```

---

## 十八、更新/学习触发时机详解

> §三 给了触发方式的快速标签（显式 API / `agent_end` hook / LLM 自主 / 用户主动等），§九 给了更新策略（ADD/MERGE/UPDATE/DELETE 的默认操作）。这一节把**触发源**、**决策者**、**判断条件**三个正交维度摊开，系统回答"什么时候更新"这个问题。

### 18.1 触发信号的 6 大类

把所有系统的触发源归纳为 6 类信号：

| 信号类型 | 特征 | 代表系统 |
|---------|------|---------|
| **事件型（Event）** | 任务/会话结束、话题切换等离散事件 | Mem0(API) / ReMe(`agent_end`) / MemOS(`agent_end` + 话题切换) / AutoSkill(滑窗 + `agent_end`) / memory-tencentdb(`agent_end`) |
| **阈值型（Threshold）** | 调用数 / token 数 / recurrence 达标 | Hermes(5+ tool calls) / pskoett(Recurrence≥3 + 跨 2 任务 + 30 天) / OpenViking(`auto_commit_threshold=8000`) |
| **LLM 自主（LLM-driven）** | LLM 自己判断"这次值得记" | Hermes(`skill_manage` 工具) / OpenSpace(LLM 二次确认门) |
| **周期扫描（Periodic）** | 后台定时器扫描 | OpenSpace(METRIC_MONITOR 周期) / AutoSkill(SkillEvo 离线 replay) / XSkill(每 8 样本 batch) / OpenViking(MemoryArchiver) |
| **指标异常（Metric）** | 4 比率 / fallback_rate / 工具降级等客观指标越线 | OpenSpace(TOOL_DEGRADATION + fallback_rate) / AutoSkill(retrieved≥40 & used==0) |
| **手动（Manual）** | 用户主动触发 / Agent 自觉 | Skill-Creator(用户说"做个技能") / pskoett(Agent 自觉 + `activator.sh` 提醒) / OpenViking(`session.commit()`) |

### 18.2 每个系统的触发全景

| 系统 | 首次学习触发 | 更新触发 | 淘汰/合并触发 |
|------|-------------|---------|-------------|
| **Mem0** | 调用方 `add()` | ❌ 不支持 UPDATE | ❌ 无 |
| **ReMe** | `agent_end` hook + score ≥ threshold | 向量余弦命中 → RewriteMemory 重写 | utility/freq 双维度老化（定时）|
| **MemOS TS** | `agent_end` + chunk/轮数达标 | Task-finalize 回调 → 7 维 LLM 评估 | dedupStatus 标记（不自动删）|
| **AutoSkill** | 滑窗 `messages[-6:]` + `agent_end` | 4 维身份匹配 → merge 决策 | **使用审计**：retrieved ≥ 40 & used == 0 → delete |
| **Hermes** | **LLM 自主**（5+ calls / 修复错误 / 发现 workflow）| LLM 判断 outdated → patch | 容量压力 → LLM 合并 |
| **OpenSpace** | **3 独立触发器**：<br>① ANALYSIS（每任务完成）<br>② TOOL_DEGRADATION（工具降级事件）<br>③ METRIC_MONITOR（周期扫 4 比率） | fallback_rate 越线 / 工具降级 → FIX 覆写<br>`min_selections ≥ 5` 数据门 + LLM 确认门 | ❌ 无硬删（DAG 保留历史，靠检索排序淘汰）|
| **Skill-Creator** | 用户说"做个技能"/"优化这个技能" | 用户 A/B feedback → 人工改进 | 人工控制 |
| **OpenViking** | `session.commit()` 手动 或 `auto_commit_threshold=8000` | MERGE（向量命中 + LLM 四分类）| MemoryArchiver：age > 7d + hotness_score < 0.1 → `_archive/` |
| **pskoett SI** | `activator.sh` 每轮提醒 / `error-detector.sh` grep 命中 | Recurrence-Count++（Pattern-Key 人工分配）| `wont_fix` 状态标记（不删）|
| **XSkill** | N 次 rollout 全完成 → batch pipeline | 嵌入余弦 ≥ 0.70 → LLM 合并 / modify 引用 ID | 全局 REFINE delete 低价值 / 容量 > 120 强制合并最相似对 |
| **memory-tencentdb** | `agent_end` hook → L0→L1→L2→L3 管线 | LLM 四动作（store/skip/update/merge）| L0/L1 `retentionDays` 规则清理；SKIP 直接硬丢（无审计）|

### 18.3 触发决策者：三种模式

```
代码硬编码（规则触发）          LLM 自主决策              人工决策
─────────────────────────────────────────────────────────────────────
Mem0 / ReMe / MemOS TS         Hermes (skill_manage)    Skill-Creator
AutoSkill / OpenViking         OpenSpace                pskoett (Agent 自判)
XSkill / memory-tencentdb      (规则 + LLM 二次确认)
```

- **代码硬编码**：稳定可预测，但需要精心调阈值；OpenSpace 的 `_FALLBACK_THRESHOLD=0.4`、AutoSkill 的 `retrieved ≥ 40`、XSkill 的 `余弦 ≥ 0.70` 都是典型硬编码阈值
- **LLM 自主**：灵活但依赖模型能力，弱模型可能漏记或乱记；Hermes 完全依赖 LLM 判断"值得记"
- **人工决策**：最可靠但不可规模化；Skill-Creator 把人放在循环里，pskoett 让 agent 自己判断升格

### 18.4 怎么判断"什么时候更新"——UPDATE 的具体判据

对于支持 UPDATE 的系统，"该不该更新"的具体决策信号：

| 系统 | UPDATE 触发条件 | 合并/覆盖判据 | 防误报设计 |
|------|---------------|-------------|----------|
| **ReMe** | 向量余弦 ≥ 阈值 0.5 | LLM RewriteMemory 重写合并 | 无（纯向量距离）|
| **MemOS TS** | hash 相同 / LLM 语义判断相似 | **7 维评估**（Faster / Elegant / Convenient / Fewer tokens / Accurate / Robust / New scenario / Fixes outdated）| 任务 finalize 回调才触发，不周期扫 |
| **AutoSkill** | **4 维能力身份匹配**（core job / output contract / constraints / context）| LLM 决策 add/merge/discard | 仅成功轨迹（`successOnly=true`），使用审计 |
| **Hermes** | LLM 判断 outdated（无硬阈值）| LLM 自主 edit patch | 约束门：新版 ≥ 旧版 |
| **OpenSpace** | `fallback_rate > 阈值` + `min_selections ≥ 5` | patch-first 生成 diff + apply-retry 3 次（回喂磁盘真实内容）防幻觉 | **三门结构**：数据门（min_selections）+ LLM 确认门 + `_addressed_degradations` 状态门（防反复演进）|
| **OpenViking** | 向量 top-5 预过滤 + LLM 四分类 | 类别硬规则：PROFILE 总是 MERGE；CASES/EVENTS 不可 MERGE | 类别白/黑名单 |
| **XSkill** | 嵌入余弦 ≥ 0.70（粗筛）| LLM 合并（MERGE_PROMPT：包含所有信息点，≤ 64 词）| 跨 rollout 批判 + 全局 REFINE 去噪 |
| **memory-tencentdb** | LLM 四动作判断（store/skip/update/merge）| 向量优先 + FTS fallback 找相似；update 直接替换 | ❌ SKIP 硬丢无审计 |

**关键观察**：

- **纯 LLM 判据**（Hermes）最灵活但最不稳定；**纯规则判据**（ReMe 向量阈值 / XSkill 0.70 / AutoSkill 4 维身份）最稳定但容易漏判细粒度差异
- **OpenSpace 的三门结构**（数据门 + LLM 确认门 + 状态门）是防误报-防反复演进-防漂移的最完备设计
- **XSkill 的"嵌入粗筛 + LLM 精合并"**是"成本-质量"平衡的最佳实践——嵌入先把明显无关的挡掉（零 LLM），相似的再交给 LLM 精判
- **最弱的**是 Mem0（不更新）和 memory-tencentdb SKIP（硬丢无审计）——前者避开了问题，后者忽略了问题

### 18.5 触发设计的三个陷阱

1. **误报陷阱**：指标波动就触发演进 → 演进反而把好 skill 改坏。OpenSpace 用 `min_selections ≥ 5` 数据门 + LLM 确认门双保险
2. **反复演进陷阱**：同一问题反复触发，每次 patch 后又因指标轻微波动再 patch。OpenSpace 用 `_addressed_degradations` 状态集记录已处理的问题
3. **触发-淘汰不对称**：有 ADD 触发但无 DELETE 触发 → 记忆库线性膨胀。XSkill 用容量硬限 120 + 强制合并解决；memory-tencentdb 只靠 `retentionDays` 规则清理

---

## 十九、抽取内容的类别光谱

> §二 讲了产物形态（文本快照 / SKILL.md / 双流 / 分层等载体），但**产物里装的语义内容是什么类别**没有横向对比。这一节补齐"抽取到的是**什么类型**的知识"。

### 19.1 内容类别分类法（6 类）

按**内容语义**把各系统抽取的记忆归为 6 类：

| 类别 | 定义 | 典型内容 | 时间性 |
|------|------|---------|-------|
| **FACT**（事实）| 用户/环境的相对不变事实 | 用户邮箱、项目常量、API 端点 | 长期稳定 |
| **EVENT**（事件）| 发生过的具体交互历史 | 对话片段、工具调用记录、时间戳 | 一次性 |
| **EXPERIENCE**（经验）| 对过去行为的反思 / 教训 | "这样做会失败"、"下次换个工具" | 半稳定 |
| **SKILL / SOP**（技能）| 可执行的工作流程指令 | "抓取分页：1. 先... 2. 再..." | 可迭代 |
| **PROFILE / PERSONA**（画像）| 用户身份、偏好、协议 | "用户是数据科学家"、"喜欢简洁回答" | 长期演化 |
| **TOOL / ENV**（工具/环境）| 工具签名、环境配置、资源链接 | MCP server URL、工具质量指标 | 动态 |

### 19.2 十一个系统的内容覆盖矩阵

| 系统 | FACT | EVENT | EXPERIENCE | SKILL | PROFILE | TOOL | 单/多类别 |
|------|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| **Mem0** | ✅ | ✅（verbatim）| ⚠️（隐式）| ❌ | ⚠️ | ⚠️ | **单一**（事实快照）|
| **ReMe** | ❌ | ❌ | ✅（含失败）| ⚠️（建议性）| ❌ | ❌ | **单一**（经验）|
| **MemOS TS** | ❌ | ❌ | ❌ | ✅（完整 SOP + 脚本）| ❌ | ⚠️ | **单一**（技能）|
| **AutoSkill** | ❌ | ❌ | ❌ | ✅（方法论抽象）| ❌ | ❌ | **单一**（技能）|
| **Hermes** | ❌ | ❌ | ⚠️（隐式）| ✅ | ❌ | ❌ | **单一**（技能）|
| **OpenSpace** | ❌ | ❌ | ✅（tool_issues）| ✅ | ❌ | ✅（tool 质量指标）| **多类别**（skill + tool 跨层）|
| **Skill-Creator** | ❌ | ❌ | ❌ | ✅（精品测试过）| ❌ | ❌ | **单一**（技能）|
| **OpenViking** | ✅ | ✅ | ⚠️ | ✅ | ✅ | ✅ | **多类别 8 桶**（User/Agent 双侧分桶）|
| **pskoett SI** | ❌ | ❌ | ✅（LEARNINGS/ERRORS）| ✅（升格后）| ❌ | ❌ | **多类别 2 层**（Learning + Skill）|
| **XSkill** | ❌ | ❌ | ✅（Experience）| ✅（Skill）| ❌ | ❌ | **双流**（Experience + Skill）|
| **memory-tencentdb** | ✅ | ✅（episodic）| ⚠️ | ✅（instruction）| ✅（persona）| ❌ | **多类别 4 层**（L0/L1/L2/L3，L1 含 persona / episodic / instruction 三子类）|

### 19.3 按内容类别看光谱

```
单一 FACT    单一 EXPERIENCE   单一 SKILL                    多类别融合
──────────────────────────────────────────────────────────────────────────────
Mem0         ReMe              MemOS / AutoSkill / Hermes / OpenViking (8 桶)
(verbatim)   (经验建议)         Skill-Creator                memory-tencentdb (L0-L3)
                                                             XSkill (Experience + Skill)
                                                             pskoett (Learning + Skill)
                                                             OpenSpace (Skill + Tool)
```

**关键观察**：

- **多类别系统天然需要"分类器"**——OpenViking 用 VLM 单标签八分类，memory-tencentdb 用 LLM 分 persona/episodic/instruction 三档。分类器的准确度直接决定系统质量
- **单一类别系统实现最简单但覆盖窄**——Mem0 只存 fact，ReMe 只存 experience，无法捕获"复合型"知识需求
- **XSkill 的双流（Experience + Skill）是粒度分层的典范**——Action 级 64 词 Experience 负责"遇到 X 就做 Y"，Task 级完整 Skill 负责"整个任务的 SOP"
- **memory-tencentdb 的 `instruction` 是"扁平化的 SKILL"**——所有程序性知识塞进一个类型，缺少动作级/工具级/任务级三粒度分层（详见总览大表第 20 行对此的批评）

### 19.4 抽取粒度（时间尺度）

内容类别之外，还有一个正交维度是**时间粒度**：

| 粒度 | 时间尺度 | 典型内容 | 代表 |
|------|---------|---------|------|
| **Action 级**（单步）| 一次工具调用或单个决策 | "用 `rg` 而不是 `grep` 搜代码" | XSkill Experience / pskoett LEARNINGS |
| **Task 级**（单任务）| 一次完整任务的工作流 | "调 FastAPI 的完整流程" | MemOS / AutoSkill / Hermes / OpenSpace / Skill-Creator / XSkill Skill |
| **Session 级**（会话）| 一段长对话的摘要 | "本次会话主要在调试认证中间件" | Mem0 / OpenViking / memory-tencentdb L2 |
| **User 级**（跨会话）| 用户长期画像 | "用户偏好简洁回答" | memory-tencentdb L3 persona |

**多粒度并存是高阶设计**——XSkill（Action + Task）和 memory-tencentdb（Session + User 加 Task 级 instruction）支持；多数系统局限在单一粒度。

### 19.5 内容类别与注入方式的关系

| 类别 | 注入时机 | 典型注入方式 |
|------|---------|-------------|
| FACT | 早（常驻）| System Prompt 前置 |
| PROFILE | 早（常驻）| System Prompt 前置（memory-tencentdb `<user-persona>`）|
| EXPERIENCE | 按需（相关时）| LLM 适配召回 |
| SKILL | 按需（触发时）| Tool 响应 / hook 追加 |
| EVENT | 按需（查询时）| Agent 主动 search |
| TOOL | 持续（监控）| 跨层反馈（OpenSpace 独有）|

内容类别决定了注入策略——**FACT/PROFILE 适合前置常驻**，**EXPERIENCE/SKILL 适合按需召回**，**TOOL 需要跨层监控**。

---

## 二十、技术路线分类（离线学 / 边干边学）

> 按**知识何时被提炼**这一维度，把各系统归为 4 条技术路线。这决定了系统的**实时性**、**成本结构**和**工程复杂度**。

### 20.1 四条技术路线

| 路线 | 特征 | 时延 | 成本结构 | 代表 |
|------|------|------|---------|------|
| **1. 离线批学（Offline Batch）** | 批量数据 → 定期 pipeline 提炼 | 分钟~小时 | 集中但可摊薄 | XSkill (batch=8) / AutoSkill SkillEvo / Hermes GEPA |
| **2. 在线异步（Online Async）** | 事件触发 → 后台 pipeline 提炼（不阻塞 agent）| 秒~分钟 | 分散但稳定（独立 pipeline）| Mem0 / ReMe / MemOS TS / AutoSkill Sidecar / OpenSpace / OpenViking / memory-tencentdb |
| **3. 在线同步（Online Sync / 边干边学）** | Agent 思考环内自觉记忆 | 毫秒内（摊入当前推理）| 摊入 agent 自身 token | Hermes (`skill_manage` 工具) / pskoett SI（Agent 自觉 append）|
| **4. 人机协作（Human-in-the-loop）** | 人主导 + LLM 辅助迭代 | 分钟~小时 | 人工 + LLM 双重成本 | Skill-Creator |

### 20.2 各系统的技术路线归属

| 系统 | 主要路线 | 说明 |
|------|---------|------|
| **Mem0** | 在线异步 | `add()` API 异步写入向量库 |
| **ReMe** | 在线异步 | `agent_end` hook 后台 pipeline |
| **MemOS TS** | 在线异步 + 离线 upgrade | 在线抽取 + Task-finalize 的离线升级路径 |
| **AutoSkill** | **在线异步 + 离线批（双轨）**| Sidecar 在线抽取 + SkillEvo 离线 replay |
| **Hermes** | **在线同步 + 离线批**| `skill_manage` 同步写 + GEPA 离线进化 |
| **OpenSpace** | 在线异步（3 触发器并行）| ANALYSIS（任务完成）/ TOOL_DEGRADATION（事件）/ METRIC_MONITOR（周期）全异步 |
| **Skill-Creator** | **人机协作** | 用户主导，子代理 + `claude -p` 辅助测试 |
| **OpenViking** | 在线异步 | Phase 1 同步归档 + Phase 2 异步抽取（fire-and-forget）|
| **pskoett SI** | **在线同步（边干边学）** | 成本摊入 agent 自身推理（每轮 +500-2k tokens）|
| **XSkill** | **离线批学** | 每 batch=8 样本后触发 pipeline，不在线 |
| **memory-tencentdb** | 在线异步 | `agent_end` hook + 后台 L1→L2→L3 管线（serial queue + checkpoint）|

### 20.3 四条路线的工程特征对比

| 维度 | 离线批学 | 在线异步 | 在线同步（边干边学）| 人机协作 |
|------|---------|---------|---------------------|---------|
| **对 agent 侵入性** | 零 | 零（外部 hook）| **高**（侵入思考环）| 中（人参与）|
| **agent 能力要求** | 低 | 低 | **高**（agent 需理解记忆 spec）| 中 |
| **实时性** | 最低（批次间隙）| 中（秒级）| **最高**（即时写入）| 最低（人工节奏）|
| **Token 成本结构** | 集中摊薄（batch 内复用）| 独立 pipeline（隔离）| 摊入 agent（隐性累积）| 人工 + LLM 双重 |
| **可观测性** | 高（batch 日志清晰）| 高（独立 pipeline）| **低**（嵌 agent 内部）| 最高（人审）|
| **可移植性** | 好 | 最好 | 差（强依赖 agent 能力）| 差（流程依赖人）|
| **基础设施需求** | batch runner + 存储 | hook + pipeline + 存储 | **几乎零**（agent 自带）| UI + 评审工具 |
| **典型场景** | 研究 / benchmark / 少量高质量 skill | 生产 / 多租户 / 企业级 | 个人开发 / 极简部署 | 精品少量 skill |
| **代表系统** | XSkill / Hermes GEPA | Mem0 / OpenSpace / memory-tencentdb | Hermes (skill_manage) / pskoett | Skill-Creator |

### 20.4 离线 vs 在线的选型权衡

**什么时候选离线批学**：
- 有 ground_truth 可做 A/B 对照（XSkill 的 N=4 rollout）
- 场景天然有"回放空间"（AutoSkill SkillEvo 用历史 user 消息当考题 replay）
- 追求最高质量门控（Hermes GEPA 约束门：新版 ≥ 旧版才晋升）
- **代价**：记忆"滞后"——新场景要等下一轮 batch 才能学到

**什么时候选在线异步**：
- 生产环境，要求记忆**下次任务立即可用**
- 宿主 Hook 机制成熟（OpenClaw / AgentScope / MCP）
- 宿主 agent 能力不强（不依赖 agent 自觉——Mem0 / MemOS / memory-tencentdb 全走这条）
- **代价**：需要维护独立 pipeline 基础设施

**什么时候选在线同步（边干边学）**：
- 个人开发场景，不想维护独立服务（pskoett 的 skill-as-system 极简部署）
- Agent 模型能力很强（能读懂元 spec，Hermes 和 pskoett 都依赖这个）
- 追求**零基础设施**部署（pskoett 甚至连数据库都不要，纯 Markdown + bash hook）
- **代价**：长会话累计 token 成本可能反超独立 pipeline；agent 能力弱时会漏记

**什么时候选人机协作**：
- 少量**关键**技能需精雕细琢（Skill-Creator 一个技能 2-3M tokens 成本）
- 企业级 skill 库需合规审查
- 有稳定 QA 流程可承担人工评审
- **代价**：完全不可规模化

### 20.5 混合路线（成熟系统的共同选择）

多数成熟系统走**混合路线**，在线负责"立即可用"，离线负责"深度提炼"：

| 系统 | 在线部分 | 离线部分 | 互补关系 |
|------|---------|---------|---------|
| **AutoSkill** | Sidecar 在线抽取 | SkillEvo 遗传 replay | 在线抽草稿，离线选最优 |
| **Hermes** | `skill_manage` 同步写 | GEPA 离线进化 | agent 写初版，GEPA 演化成品 |
| **MemOS TS** | `agent_end` 在线抽取 | Task-finalize upgrade 评估 | 抽取即可用，升级走质量门 |
| **OpenSpace** | ANALYSIS（任务完成触发）| METRIC_MONITOR（周期扫）| 单次事件触发 + 周期体检互补 |

**混合路线的核心逻辑**：**两条路线不冲突而是互补**——在线解决"下次任务能用"，离线解决"长期质量爬升"。

### 20.6 边干边学的隐性成本

**特别提醒**：在线同步（边干边学）看起来"零基础设施"很美，但隐性成本容易被低估：

```
pskoett SI 单轮成本分解（每轮 +500-2k tokens）：
  ├─ 读 SKILL.md 元 spec（固定 ~500 tokens）
  ├─ grep LEARNINGS.md / ERRORS.md（变量，随文件增长）
  ├─ 判断是否升格（~300-500 tokens 推理）
  └─ 写入 Markdown（~200 tokens）
───────────────────────────────────────────────
  100 轮会话累计：50k ~ 200k tokens
  对比：Mem0 独立抽取 ~5k tokens / 100 条记忆
```

**在线同步在短会话下最便宜，长会话下反而最贵**——这是 pskoett 最容易被忽视的代价。

---

## 二十一、选型建议

| 你的需求 | 推荐系统 |
|---------|---------|
| 需要**任务断点恢复**（长任务防中断）| **Mem0** — 历史保真最简单 |
| 需要**失败学习** + 情境化建议 | **ReMe** — 失败轨迹 + RewriteMemory 独有 |
| 需要**企业级技能管理**（版本/评估/脚本）| **MemOS** — 产物最完整 |
| 需要**检索精准** + 使用审计闭环 | **AutoSkill** — 查询改写 + Skill 选择 + 审计 |
| 需要**极简实现** + 信任 LLM 能力 | **Hermes** — 无管线，系统提示引导 |
| 需要**离线自动进化** | **Hermes (GEPA)** 或 **AutoSkill (SkillEvo)** |
| 需要**基础设施层**（不绑定上层）| **Mem0** — 通用存储 API |
| 需要**记忆防退化**（老化清理）| **ReMe** — utility/freq 双维度 |
| 需要**少量关键技能精雕细琢** | **Skill-Creator** — 测试驱动 + 人工评审 + 描述优化 |
| 需要**多模态视觉推理**记忆积累 | **XSkill** — 双流 + 视觉锚定 + 跨 Rollout 对比 |
| 需要**多路径对比提取**经验 | **XSkill**（独立 Rollout）或自建（v5 §7.1a 会话内检测） |
| 需要**双流知识**（行动级+任务级分离）| **XSkill** — 消融实验证明互补性 |
| 需要**跨 agent 共享自演进技能** | **OpenSpace** — MCP 统一接入，本地 + 云端 public/private/group |
| 需要**工具质量下降自动修依赖 skill**（级联闭环）| **OpenSpace** — TOOL_DEGRADATION 触发器，独有 |
| 需要**完整演化链可审计**（版本 DAG + 多父 merge）| **OpenSpace** — skill_id sidecar + content_snapshot/diff 全版本保留 |
| 需要**4 比率使用审计**（selections→applied→completions→fallbacks） | **OpenSpace** — SQL 原子递增，并发安全 |
| 需要**patch-first token 效率** | **OpenSpace** — FIX 只输出 diff，节省 4-10× tokens vs 全文重写 |
| 需要**多 host 统一技能管理**（Claude Code / Codex / OpenClaw / nanobot / Cursor） | **OpenSpace** — MCP 协议即插即用 |
| 需要**上下文统一管理层**（记忆+资源+技能合一）| **OpenViking** — viking:// 文件系统 + 8 类分桶（User/Agent 双侧） |
| 需要**长程会话 Agent**（手动 commit 控制粒度）| **OpenViking** — 两阶段 commit，archive_NNN 顺序归档 |
| 需要**工具调用语义完整保留** | **OpenViking** — ToolPart(tool_input/output/status/duration_ms，输出截断 500 字符) |
| 需要**零基础设施** | **pskoett SI** — 无独立 pipeline，纯 Markdown + bash（成本摊入 agent 自身上下文） |
| 需要**跨编辑器通用技能** | **pskoett SI** — Claude Code / Copilot / OpenClaw 通吃 |
| 需要**教训→规则的升格漏斗** | **pskoett SI** — Recurrence≥3 + 三级升格 |
| 需要**作为 outer loop 参考设计** | **pskoett SI** — Inner/Outer loop 概念最清晰 |
| 需要**零基础设施 + 跨 agent 兼容**（Claude Code / Codex / Copilot 混用） | **pskoett SI** — 纯 Markdown + bash，skill-as-system |
| 需要**个人开发场景的轻量教训管理** | **pskoett SI** — 三级升格漏斗，Pattern-Key 人工去重 |
| 需要**把高频规则"结晶"到 system prompt** | **pskoett SI** — Level 2 升格到 CLAUDE.md / AGENTS.md |
| 需要**skill-as-system 范式参考**（prompt 而非代码实现记忆系统）| **pskoett SI** — 无独立 pipeline，但成本摊入 agent 自身 token |
| 需要**OpenClaw 分层长期记忆**（persona + 事件 + 交互规则）| **memory-tencentdb** — L0-L3 四层，SQLite/TCVDB 双后端 |
| 需要**L3 用户画像自动生成** | **memory-tencentdb** — 四层深度扫描（基础锚点/兴趣/交互协议/认知内核）|
| 需要**Hybrid 检索**（FTS5 + 向量 + RRF）| **memory-tencentdb** — 5s 超时兜底，三种策略可切换 |
| 需要**云端向量检索**（腾讯云 TCVDB + BM25 sparse）| **memory-tencentdb** — 支持 profile sync 双向同步 |

---

## 参考资料

- [mem0ai/mem0](https://github.com/mem0ai/mem0) | [arXiv 2504.19413](https://arxiv.org/abs/2504.19413)
- [agentscope-ai/ReMe](https://github.com/agentscope-ai/ReMe) | [arXiv 2512.10696](https://arxiv.org/abs/2512.10696)
- [MemOS Skill 实现分析](./MemOS-Skill-Implementation.md)
- [AutoSkill 管线分析](./AutoSkill-Pipeline-Analysis.md)
- [Hermes Dreaming 分析](./Hermes-Dreaming-Analysis.md)
- [Mem0 程序记忆分析](./Mem0-Procedural-Memory-Analysis.md)
- [ReMe 程序记忆分析](./ReMe-Procedural-Memory-Analysis.md)
- [Skill-Creator 分析](./Skill-Creator-Analysis.md) | [anthropics/skills](https://github.com/anthropics/skills/tree/main/skills/skill-creator)
- [XSkill Experience 分析](./XSkill-Experience-Analysis.md) | [arXiv 2603.12056](https://arxiv.org/abs/2603.12056) | [XSkill-Agent/XSkill](https://github.com/XSkill-Agent/XSkill)
- [OpenSpace 分析](./OpenSpace-Analysis.md) | [HKUDS/OpenSpace](https://github.com/HKUDS/OpenSpace) | [open-space.cloud](https://open-space.cloud)
- [OpenViking 分析](./OpenViking_分析.md) | [volcengine/OpenViking](https://github.com/volcengine/OpenViking)
- [pskoett Self-Improvement Skill 分析](./Self-Improvement-Skill-Analysis.md) | [pskoett/pskoett-ai-skills](https://github.com/pskoett/pskoett-ai-skills/tree/main/skills/self-improvement)
- [memory-tencentdb 代码分析](./memory-tencentdb-analysis.md) | OpenClaw 插件（四层 L0-L3 长期记忆 + SQLite/TCVDB 双后端）
