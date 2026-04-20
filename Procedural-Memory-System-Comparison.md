# 程序记忆系统横向对比

> 对比范围：Mem0 / ReMe / MemOS / AutoSkill / Hermes
> 分析日期：2026-04-20

---

## 一、核心定位对比

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **设计哲学** | Memory as Facts | Memory as Experience | Memory as Knowledge Assets | Memory as Skills | Memory as Evolution Capital |
| **核心目标** | 任务中断后精确恢复 | 提供情境化经验建议 | 生成可执行技能资产 | 提取可复用方法论 | Agent 随时间自动进化 |
| **产物本质** | 执行历史"录像" | 经验"建议书" | 可执行"技能包" | 可复用"方法论" | 可进化"知识资本" |
| **与 Agent 系统耦合度** | **完全解耦**（独立服务/API） | **松耦合**（AgentScope 集成但可分离） | **紧耦合**（TS-end 内置） | **紧耦合**（OpenClaw 插件） | **完全内置**（Hermes 专属） |
| **适用场景** | 通用记忆基础设施 | 长对话 Agent | 企业级任务管理 | OpenClaw 插件 | 自进化个人助手 |

---

## 二、记忆产物形态

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **产物结构** | 逐步骤文本快照 | `when_to_use` + `experience` + tags | SKILL.md + scripts/ + references/ + evals/ | SKILL.md (YAML frontmatter + Markdown) | SKILL.md + scripts/ + references/ + templates/ |
| **内容示例** | Step 1: Opened URL... Result: HTML loaded... | {"when_to_use": "当需要分页抓取时", "experience": "使用 BFS 而非 DFS..."} | # Goal / # Steps / # Pitfalls / evals/test_*.json | # Goal / # Constraints & Style / # Workflow (optional) | 同 MemOS/AutoSkill 标准格式 |
| **去标识化** | **无**（保留 URL/ID/参数） | 中等（提取模式） | **高**（placeholder 替换） | **高**（placeholder 替换） | 高（LLM 自主判断） |
| **可执行性** | 不可执行（纯记录） | 不可执行（建议性） | **可执行**（含脚本和评估集） | 可执行（Prompt 指令） | 可执行（Skill 三级加载） |

---

## 三、提取触发机制

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **触发方式** | 显式 API 调用 | 自动（`agent_end` hook） | 自动（`agent_end` + 话题切换） | 自动（滑动窗口 + `agent_end`） | **LLM 自主判断**（系统提示引导） |
| **触发条件** | `add(agent_id=..., memory_type="procedural")` | 任务完成，score >= threshold | 任务完成，chunk 数/轮数达标 | 每轮 `messages[-6:]` 延迟提取 | 5+ tool calls / 修复错误 / 发现工作流 |
| **谁决定提取** | 调用方（外部代码） | 代码硬编码 | 代码硬编码 | 代码硬编码 | **LLM 自己**（`skill_manage` 工具） |
| **人类类比** | 手动按录像键 | 自动剪辑精彩集锦 | 自动归档项目文档 | 每轮自动扫描笔记 | 自觉记笔记的学习者 |

---

## 四、提取 Prompt 设计

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **Prompt 数量** | 1 个专用 | 4 个（成功/失败/对比/验证） | 6-8 个（分段管线） | 3-4 个（提取/决策/合并） | **无独立提取 Prompt**（仅靠系统提示指令） |
| **Prompt 长度** | ~800 字 | ~400 字 × 4 | ~500-2000 字 × N | **~4000 字**（8 章节） | ~500 字（系统提示中的行为指令） |
| **核心指令** | "记录每步 Action + Result，保留 verbatim" | "提取可复用模式、决策点、失败防范" | "生成任务摘要 → 评估可复用性 → 生成 SKILL.md" | "提取 HOW 非 WHAT，去标识化，用户输入为证据" | "5+ calls 后保存为 skill，发现过时立即 patch" |
| **证据来源** | 完整对话（verbatim） | 轨迹（messages + score） | 任务摘要（LLM 生成） | **仅 user 消息**（assistant 仅参考） | LLM 自主判断 |
| **失败轨迹** | 无 | **有**（专用失败提取 + 对比分析） | 无 | 无（`successOnly=true`） | 无（隐式包含） |

---

## 五、去重与合并机制

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **去重策略** | **无**（有意为之） | 向量余弦相似度（阈值 0.5） | 两级：hash + 语义 LLM | 向量 + LLM 4 维判断 | 无独立机制 |
| **合并策略** | 无 | RewriteMemory 重写合并 | DUPLICATE/UPDATE/NEW 决策 | add / merge / discard 决策 | LLM 自主 patch/edit |
| **判断维度** | — | 向量相似度（无 LLM） | 语义 LLM 评估 | **4 维能力身份**：job-to-be-done / output contract / constraints / context | — |
| **批内去重** | 无 | 有（同批次互相比） | 有（IngestWorker 批处理） | 有（滑动窗口重叠 4 条） | 无 |
| **设计意图** | 历史保真，合并会破坏完整性 | 减少噪声记忆积累 | 避免重复 Skill，渐进进化 | 渐进完善（相邻轮次 merge 升级） | 信任 LLM 判断力 |

---

## 六、质量门控

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **质量评估** | **无** | **5 维验证** | **0-10 评分** | **4 维能力身份** | **无自动门控** |
| **评估维度** | — | ACTIONABILITY / ACCURACY / RELEVANCE / CLARITY / UNIQUENESS | Repeatable / Transferable / Technical depth / 完整度 | core job / output contract / constraints / context | — |
| **评分方式** | — | 独立 LLM 打分（0.0-1.0） | 独立 LLM 打分（0-10） | LLM 决策（add/merge/discard） | LLM 自觉 + 安全扫描 |
| **淘汰阈值** | — | **score < 0.3 → 剔除** | qualityScore < 6 → draft | discard 直接丢弃 | 50+ 正则威胁检测 |
| **使用审计** | 无 | 无（有 utility/freq 追踪） | 无 | **有**（retrieved→relevant→used 闭环） | 无 |
| **自动清理** | 无 | **utility/freq 双维度老化** | 无 | retrieved>=40 & used==0 → 删除 | 容量压力逼合并（2200 字符上限） |

---

## 七、通用化与复用

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **通用化程度** | **无** | 中等 | **高** | **高** | 高 |
| **去标识化方法** | 保留所有具体值 | 抽象为模式描述 | placeholder 替换 | placeholder 替换 | LLM 自主判断 |
| **跨任务复用** | 不适合（太具体） | 适合（经验建议） | **适合（可执行 SOP）** | 适合（方法论抽象） | 适合（Skill 格式标准） |
| **复用测试** | 无 | 无 | **有（evals/ 评估集）** | **有（SkillEvo replay 评估）** | 有（GEPA 测试套件） |
| **版本管理** | 无 | 无 | 有（版本号递增） | 有（版本号递增） | 有（patch/edit 历史） |

---

## 八、更新策略

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **默认操作** | **ADD**（总是新增） | RewriteMemory 合并更新 | **decide** → add/merge/upgrade | **decide** → add/merge/discard | **LLM 自主选择** create/patch/edit |
| **UPDATE 条件** | 不支持 | 向量相似命中 → 重写合并 | 语义相似 → 合并内容 | 4 维身份匹配 → merge | LLM 判断 outdated → patch |
| **DELETE 条件** | 不支持 | utility/freq 老化删除 | 无自动删除 | 使用审计未使用 → 删除 | LLM 判断 → delete |
| **历史保留** | SQLite 审计表全保留 | 删除标记 + 频率追踪 | Chunk 标记 dedupStatus | 版本号递增覆盖 | patch 覆盖（old_string/new_string） |
| **合并质量** | — | LLM 重写为统一建议 | LLM 语义合并 | LLM 语义合并（防回归） | LLM 自主 edit |

---

## 九、存储与检索

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **主存储** | 向量 DB（20+ 后端） | 向量 DB（Memory/ES/Chroma） | SQLite + 文件系统 | 文件系统 + 向量索引 | 文件系统（~/.hermes/skills/） |
| **辅助存储** | SQLite（审计历史） | 文件系统（ReMeLight） | SKILL.md 磁盘文件 | SkillBank/ + Mirror | SQLite（Session Archive） |
| **向量字段** | 完整摘要 | **`when_to_use`（非 content）** | Chunk summary | Skill description + prompt | 无（纯文件系统） |
| **检索方式** | 语义向量搜索 | 语义向量 + LLM 重排 | FTS + 向量混合 | **向量 + BM25 混合（0.9:0.1）** | 系统提示动态加载 Skill 索引 |
| **检索后处理** | 无（上层手动拼接） | **RewriteMemory**（重写为统一建议） | 直接注入 SKILL.md | LLM Skill 选择（二次筛选） | 三级加载（L0/L1/L2） |
| **注入位置** | 由调用方决定 | System Prompt | before_prompt_build hook | before_prompt_build hook | System Prompt（动态构建） |

---

## 十、离线演进

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **有无离线演进** | **无** | **无**（仅老化删除） | **有**（SkillEvo） | **有**（SkillEvo） | **有**（多层 Dreaming） |
| **触发方式** | — | — | 后台定时扫描 | 后台定时扫描 | Gateway flush + Cron + GEPA |
| **演进机制** | — | — | 遗传算法 + replay | 遗传算法 + replay | **GEPA（Gradient-free Evolutionary Prompt Adaptation）** |
| **评估方式** | — | — | replay 评分 + 测试套件 | replay 评分 + 规则检查 | **LLM-as-judge + 测试套件 + 基准门控** |
| **数据结构** | — | — | 平文本 SKILL.md | 平文本 SKILL.md | **知识图谱（事实+实体+关系）** |
| **人工参与** | — | — | 无 | 无 | GEPA 需 PR 审查 |
| **人类类比** | — | — | 后台自动复习 | 后台自动复习 | **睡觉时大脑整理记忆** |

---

## 十一、LLM 调用开销

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **单次提取调用** | **1 次** | **3-4 次** | **5-9 次** | **5-8 次/轮** | **1 次** |
| **调用明细** | 提取摘要 | 提取(1) + 验证(1) + 重排(1) + 重写(1) | 摘要(1) + 去重(1) + 分类(1) + 评估(1) + 生成(1) + 验证(1)... | 改写(1) + embed(1) + 选择(1) + 提取(1) + 决策(1) + 合并(1) + 审计(1) | 生成 SKILL.md(1) |
| **成本特征** | 最低 | 中等 | 较高 | 最高（每轮都跑） | **最低**（但质量依赖模型能力） |
| **延迟敏感** | 同步/异步均可 | 后台异步 | 后台异步 | 后台异步 | 在线同步（LLM 自主调用工具） |

---

## 十二、独特优势与劣势

### Mem0
| 优势 | 劣势 |
|------|------|
| 实现最简单，1 次 LLM 调用 | 无质量门控，可能积累噪声 |
| 历史保真，中断恢复最精确 | 产物不可复用，每次都从头开始 |
| 通用基础设施，不绑定上层结构 | 无通用化，跨任务无法迁移 |
| 支持 20+ 向量 DB 后端 | 无离线演进，记忆只增不减 |

### ReMe
| 优势 | 劣势 |
|------|------|
| **失败轨迹提取**（独有） | 经验建议不可直接执行 |
| **RewriteMemory** 注入适配度高 | 无离线主动演进 |
| **when_to_use 向量化**检索精准 | 去重只靠向量相似度，无 LLM 语义判断 |
| **utility/freq 双维度老化**防退化 | 产物形态松散，无结构化约束 |
| LoCoMo SOTA（86.23） | |

### MemOS TS
| 优势 | 劣势 |
|------|------|
| **产物最完整**（Skill + 脚本 + 评估集） | LLM 调用开销大（5-9 次） |
| **两级去重**（hash + 语义 LLM） | 被动升级（用户再做才触发） |
| **任务自动分割**（LLM 判断话题切换） | 实现复杂度高 |
| 质量评分 0-10 精细化 | 无使用审计闭环 |

### AutoSkill
| 优势 | 劣势 |
|------|------|
| **查询改写**解决代词引用问题 | Sidecar 每轮 5-6 次 LLM 调用，开销大 |
| **LLM Skill 选择**二次筛选保精准 | Embedded 版功能裁剪严重 |
| **使用审计闭环**（retrieved→relevant→used） | 仅成功轨迹（successOnly=true） |
| **SkillEvo 离线进化**（replay + 晋升） | 无失败轨迹学习 |
| 证据溯源严格（仅 user 消息） | |

### Hermes
| 优势 | 劣势 |
|------|------|
| **实现最简洁**（无独立管线） | 质量完全依赖 LLM 能力 |
| **冻结快照**保 prefix cache 性能 | 无自动质量门控 |
| **多层 Dreaming**（6 层离线整理） | 无使用审计 |
| **GEPA** 是目前最完整的进化闭环 | GEPA 需人工 PR 审查 |
| **知识图谱**支持跨记忆综合推理 | 配置复杂度高 |
| 容量压力驱动整合（类人脑遗忘） | |

---

## 十三、架构耦合度

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** |
|------|----------|----------|--------------|---------------|------------|
| **耦合度** | **完全解耦** | **松耦合** | **紧耦合** | **紧耦合** | **完全内置** |
| **运行方式** | 独立服务 / Python 库 / HTTP API | 可选：文件系统版（ReMeLight）或向量服务版（reme_ai） | TypeScript 内置模块（TS-end） | OpenClaw 插件（Embedded/Sidecar） | Hermes Agent 内置功能 |
| **依赖关系** | 零依赖（可被任何 Agent 框架调用） | 依赖 AgentScope ReActAgent（文件版）或独立 HTTP 服务（向量版） | 依赖 MemOS TS 框架生命周期 | 依赖 OpenClaw Hook 系统 | 完全专属，不可分离 |
| **接入成本** | 最低（一行代码 `memory.add()`） | 低（pip install + 配置） | 高（需集成 TS-end 框架） | 中（OpenClaw 插件安装） | 最高（需使用完整 Hermes Agent） |
| **框架迁移** | **最容易**（纯 API） | 容易（文件版人类可读） | 困难（深度绑定） | 困难（OpenClaw 专属） | **不可能**（完全内置） |
| **多 Agent 共享** | **支持**（通过 `agent_id` / `user_id`） | 支持（通过 workspace） | 支持（通过作用域） | 支持（user scope + library scope） | 支持（通过 skill 目录） |
| **人类可读** | 一般（JSON 行记录） | **最好**（Markdown 文件） | 好（SKILL.md） | 好（SKILL.md） | 好（SKILL.md） |

### 耦合度分析

```
完全解耦 ◄────────────────────────────────────────► 完全内置

Mem0       ReMe(reme_ai)   ReMe(ReMeLight)   AutoSkill   MemOS TS   Hermes
 │              │                │               │          │         │
 ▼              ▼                ▼               ▼          ▼         ▼
独立服务      独立服务         文件系统         OpenClaw    TS-end    Hermes
可插拔        可插拔           可迁移           插件        内置      专属
```

**关键洞察**：
- **Mem0** 是唯一的"记忆即服务"——可被任何 Agent 框架调用，甚至可被其他记忆系统当作底层存储（Hermes L4 Provider 就支持 Mem0 插件）
- **ReMeLight** 虽然与 AgentScope 集成，但产物是纯 Markdown 文件，人类可直接编辑，迁移成本最低
- **AutoSkill / MemOS / Hermes** 都是特定 Agent 框架的功能模块，无法脱离原框架运行
- 如果你正在设计自己的系统，**Mem0 的解耦思路**最值得借鉴——把记忆层抽象为独立服务，上层 Agent 通过 API 消费

---

## 十四、设计光谱

```
原始保真 ←————————————————————————————→ 通用抽象

Mem0        ReMe        MemOS        AutoSkill     Hermes
(快照)      (经验)      (技能包)      (方法论)      (进化资本)
 │           │           │            │            │
 ▼           ▼           ▼            ▼            ▼
 verbatim   建议性      可执行       可执行+      可执行+
 记录        bullet      SOP          审计闭环     离线进化
            points

自动化 ←————————————————————————————→ LLM 自主

Mem0        MemOS       AutoSkill    ReMe         Hermes
(代码控制)   (代码控制)   (代码控制)    (代码控制)    (LLM自主)

简单 ←————————————————————————————→ 复杂

Mem0        ReMe        Hermes       AutoSkill    MemOS
(1次LLM)    (3-4次)     (1次+系统)   (5-8次/轮)   (5-9次)
```

---

## 十四、选型建议

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

---

## 参考资料

- [mem0ai/mem0](https://github.com/mem0ai/mem0) | [arXiv 2504.19413](https://arxiv.org/abs/2504.19413)
- [agentscope-ai/ReMe](https://github.com/agentscope-ai/ReMe) | [arXiv 2512.10696](https://arxiv.org/abs/2512.10696)
- [MemOS Skill 实现分析](./MemOS-Skill-Implementation.md)
- [AutoSkill 管线分析](./AutoSkill-Pipeline-Analysis.md)
- [Hermes Dreaming 分析](./Hermes-Dreaming-Analysis.md)
- [Mem0 程序记忆分析](./Mem0-Procedural-Memory-Analysis.md)
- [ReMe 程序记忆分析](./ReMe-Procedural-Memory-Analysis.md)
