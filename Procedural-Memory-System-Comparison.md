# 程序记忆系统横向对比

> 对比范围：Mem0 / ReMe / MemOS / AutoSkill / Hermes / Skill-Creator / XSkill
> 分析日期：2026-04-21（v5：新增 Skill-Creator、XSkill）

---

## 一、核心定位对比

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **设计哲学** | Memory as Facts | Memory as Experience | Memory as Knowledge Assets | Memory as Skills | Memory as Evolution Capital | Memory as Quality-Tested Skills | Memory as Dual-Stream Knowledge |
| **核心目标** | 任务中断后精确恢复 | 提供情境化经验建议 | 生成可执行技能资产 | 提取可复用方法论 | Agent 随时间自动进化 | 精雕细琢高质量技能 | 双流互补持续积累 |
| **产物本质** | 执行历史"录像" | 经验"建议书" | 可执行"技能包" | 可复用"方法论" | 可进化"知识资本" | 经测试验证的"精品课程" | 行动级"笔记" + 任务级"SOP" |
| **与 Agent 系统耦合度** | **完全解耦**（独立服务/API） | **松耦合**（AgentScope 集成但可分离） | **紧耦合**（TS-end 内置） | **紧耦合**（OpenClaw 插件） | **完全内置**（Hermes 专属） | **紧耦合**（Claude Code 插件） | **独立框架**（可换骨干模型） |
| **适用场景** | 通用记忆基础设施 | 长对话 Agent | 企业级任务管理 | OpenClaw 插件 | 自进化个人助手 | Claude Code 技能开发 | 多模态视觉推理 |

---

## 二、记忆产物形态

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **产物结构** | 逐步骤文本快照 | `when_to_use` + `experience` + tags | SKILL.md + scripts/ + references/ + evals/ | SKILL.md (YAML frontmatter + Markdown) | SKILL.md + scripts/ + references/ | SKILL.md + scripts/ + agents/ + evals/ + eval-viewer/ | **双流**：Experience(≤64词 condition-action-embedding) + Skill(Markdown SOP) |
| **内容示例** | Step 1: Opened URL... Result: HTML loaded... | {"when_to_use": "当需要分页抓取时", "experience": "使用 BFS 而非 DFS..."} | # Goal / # Steps / # Pitfalls / evals/test_*.json | # Goal / # Constraints & Style / # Workflow | 同标准 SKILL.md 格式 | 同标准 SKILL.md + 4子代理评审产物 | Experience: "When handling dark images, use histogram equalization" / Skill: # Workflow + ## Tool Templates |
| **去标识化** | **无**（保留 URL/ID/参数） | 中等（提取模式） | **高**（placeholder 替换） | **高**（placeholder 替换） | 高（LLM 自主判断） | 高（用户主动控制） | **高**（占位符 `[TARGET]` `[QUERY]` + 64词强制泛化） |
| **可执行性** | 不可执行（纯记录） | 不可执行（建议性） | **可执行**（含脚本和评估集） | 可执行（Prompt 指令） | 可执行（Skill 三级加载） | **可执行**（含脚本+评估集+子代理） | Experience 建议性 / Skill 可执行（含代码模板） |

---

## 三、提取触发机制

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **触发方式** | 显式 API 调用 | 自动（`agent_end` hook） | 自动（`agent_end` + 话题切换） | 自动（滑动窗口 + `agent_end`） | **LLM 自主判断**（系统提示引导） | **用户主动触发** | 自动（batch 训练后） |
| **触发条件** | `add(agent_id=..., memory_type="procedural")` | 任务完成，score >= threshold | 任务完成，chunk 数/轮数达标 | 每轮 `messages[-6:]` 延迟提取 | 5+ tool calls / 修复错误 / 发现工作流 | 用户说"做个技能"/"优化这个技能" | N 次 rollout 全部完成后 |
| **谁决定提取** | 调用方（外部代码） | 代码硬编码 | 代码硬编码 | 代码硬编码 | **LLM 自己**（`skill_manage` 工具） | **用户自己** | 代码硬编码（batch pipeline） |
| **人类类比** | 手动按录像键 | 自动剪辑精彩集锦 | 自动归档项目文档 | 每轮自动扫描笔记 | 自觉记笔记的学习者 | 手工打造精品课程 | 考完试后批量总结错题 |

---

## 四、提取 Prompt 设计

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **Prompt 数量** | 1 个专用 | 4 个（成功/失败/对比/验证） | 6-8 个（分段管线） | 3-4 个（提取/决策/合并） | **无独立提取 Prompt**（仅靠系统提示指令） | **无提取 Prompt**（人机迭代创建） | **10 个**（P1-P10 覆盖积累+推理两阶段） |
| **Prompt 长度** | ~800 字 | ~400 字 × 4 | ~500-2000 字 × N | **~4000 字**（8 章节） | ~500 字（系统提示中的行为指令） | ~900 行 SKILL.md（元技能指令） | ~200-600 字 × 10 |
| **核心指令** | "记录每步 Action + Result，保留 verbatim" | "提取可复用模式、决策点、失败防范" | "生成任务摘要 → 评估可复用性 → 生成 SKILL.md" | "提取 HOW 非 WHAT，去标识化，用户输入为证据" | "5+ calls 后保存为 skill，发现过时立即 patch" | "Draft → Test → Review → Improve → Repeat" | "Review attempts and extract practical lessons (≤64 words)" |
| **证据来源** | 完整对话（verbatim） | 轨迹（messages + score） | 任务摘要（LLM 生成） | **仅 user 消息**（assistant 仅参考） | LLM 自主判断 | 子代理执行转录 + 人工评审 | **N 条轨迹 + 图像**（视觉锚定） |
| **失败轨迹** | 无 | **有**（专用失败提取 + 对比分析） | 无 | 无（`successOnly=true`） | 无（隐式包含） | **有**（without_skill 基线对比） | **有**（N 路 rollout 含成功+失败，跨轨迹对比） |

---

## 五、多路径对比能力

这是 v5 新增的核心对比维度。不同系统处理"同一任务多种做法"的能力差异巨大。

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **多路径来源** | 无 | 无 | 无 | 无 | 无 | **刻意构造**（with/without skill 并行 spawn） | **自然产生**（N=4 独立 rollout） |
| **对比方式** | — | — | — | — | — | Grader 评分 + Comparator 盲测 A/B + Analyzer 事后分析 | **Cross-Rollout Critique**（成功/失败轨迹对比提取经验） |
| **对比粒度** | — | — | — | — | — | 整体输出质量（rubric 1-10） | **逐步骤**（哪步有效、哪步出错、工具组合效果） |
| **谁做对比** | — | — | — | — | — | 3 个子代理（Grader/Comparator/Analyzer） | MLLMkb（知识模型，可多模态） |
| **对比产物** | — | — | — | — | — | feedback.json → 人工改进技能 | Experience 更新操作 [{add, e}, {modify, id, e'}] |
| **真实执行** | — | — | — | — | — | **是**（子代理真实执行任务） | **是**（N 次独立 rollout 真实执行） |
| **视觉对比** | — | — | — | — | — | 否（纯文本） | **是**（图像与轨迹联合分析） |

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

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **去重策略** | **无**（有意为之） | 向量余弦相似度（阈值 0.5） | 两级：hash + 语义 LLM | 向量 + LLM 4 维判断 | 无独立机制 | 无自动去重（人工控制） | **嵌入余弦 ≥0.70 → LLM 合并** |
| **合并策略** | 无 | RewriteMemory 重写合并 | DUPLICATE/UPDATE/NEW 决策 | add / merge / discard 决策 | LLM 自主 patch/edit | 人工评审后改进 | **三层合并**（逐条/容量缩减/全局精炼） |
| **判断维度** | — | 向量相似度（无 LLM） | 语义 LLM 评估 | **4 维能力身份** | — | 人工判断 | 嵌入相似度（粗筛） + LLM（精合并） |
| **批内去重** | 无 | 有（同批次互相比） | 有（IngestWorker 批处理） | 有（滑动窗口重叠 4 条） | 无 | 无（迭代式改进非批量） | **有**（每批 8 样本后全局精炼） |
| **容量管理** | 无限制 | utility/freq 老化删除 | 无明确限制 | retrieved≥40 & used==0 删除 | 2200 字符硬限制 | 无（靠人控制） | **硬限制 120 条 + 强制合并最相似对** |
| **设计意图** | 历史保真 | 减少噪声积累 | 避免重复 Skill | 渐进完善 | 信任 LLM 判断力 | 手工打磨不需去重 | 嵌入粗筛+LLM精合并，平衡成本和质量 |

---

## 七、质量门控

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **质量评估** | **无** | **5 维验证** | **0-10 评分** | **4 维能力身份** | **无自动门控** | **4 子代理 + 人工评审** | **跨 Rollout 对比 + 64 词硬约束** |
| **评估维度** | — | ACTIONABILITY / ACCURACY / RELEVANCE / CLARITY / UNIQUENESS | Repeatable / Transferable / Technical depth / 完整度 | core job / output contract / constraints / context | — | Grader 断言 + Comparator rubric + Analyzer 模式 + 人工 feedback | 成功/失败轨迹对比 + REFINE 去低质量 |
| **评分方式** | — | 独立 LLM 打分（0.0-1.0） | 独立 LLM 打分（0-10） | LLM 决策（add/merge/discard） | LLM 自觉 + 安全扫描 | 子代理打分 + **人工最终决策** | LLM 对比批判 + 嵌入相似度筛选 |
| **淘汰阈值** | — | **score < 0.3 → 剔除** | qualityScore < 6 → draft | discard 直接丢弃 | 50+ 正则威胁检测 | 人工说"不满意" → 重做 | 全局精炼 delete 低价值条目 |
| **使用审计** | 无 | 无（有 utility/freq 追踪） | 无 | **有**（retrieved→relevant→used 闭环） | 无 | 有（eval pass_rate 追踪） | 无（但有跨任务迁移实验验证泛化性） |
| **自动清理** | 无 | **utility/freq 双维度老化** | 无 | retrieved>=40 & used==0 → 删除 | 容量压力逼合并 | 无（人控制生命周期） | **全局精炼** merge/delete（目标 80-100 条） |

---

## 八、通用化与复用

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **通用化程度** | **无** | 中等 | **高** | **高** | 高 | **最高**（测试驱动验证） | **高**（64 词硬约束 + 占位符） |
| **去标识化方法** | 保留所有具体值 | 抽象为模式描述 | placeholder 替换 | placeholder 替换 | LLM 自主判断 | 用户主动控制 | `[TARGET]` `[QUERY]` 占位符 + 64 词限制强制泛化 |
| **跨任务复用** | 不适合（太具体） | 适合（经验建议） | **适合（可执行 SOP）** | 适合（方法论抽象） | 适合（Skill 格式标准） | **适合**（经 A/B 测试验证） | **适合**（零样本迁移实验验证，跨 benchmark 2-3 分提升） |
| **复用测试** | 无 | 无 | **有（evals/ 评估集）** | **有（SkillEvo replay 评估）** | 有（GEPA 测试套件） | **有**（run_eval.py 触发率 + 子代理 A/B + 人工评审） | **有**（5 benchmark × 4 backbone + 零样本迁移实验） |
| **版本管理** | 无 | 无 | 有（版本号递增） | 有（版本号递增） | 有（patch/edit 历史） | 有（iteration-N/ 目录结构） | 有（batch 迭代，全局文档版本） |

---

## 九、更新策略

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **默认操作** | **ADD**（总是新增） | RewriteMemory 合并更新 | **decide** → add/merge/upgrade | **decide** → add/merge/discard | **LLM 自主选择** create/patch/edit | **人工迭代改进** | **add/modify** → 嵌入去重 → LLM 合并 |
| **UPDATE 条件** | 不支持 | 向量相似命中 → 重写合并 | 语义相似 → 合并内容 | 4 维身份匹配 → merge | LLM 判断 outdated → patch | 人工评审不满意 → 改进 | 嵌入余弦 ≥0.70 → LLM 合并 / modify 引用 ID 直接替换 |
| **DELETE 条件** | 不支持 | utility/freq 老化删除 | 无自动删除 | 使用审计未使用 → 删除 | LLM 判断 → delete | 人工决定 | 全局精炼 delete 低价值 / 容量超限强制合并 |
| **历史保留** | SQLite 审计表全保留 | 删除标记 + 频率追踪 | Chunk 标记 dedupStatus | 版本号递增覆盖 | patch 覆盖 | iteration-N/ 目录保留所有迭代 | 重编号覆盖（E0, E1, E2...） |
| **合并质量** | — | LLM 重写为统一建议 | LLM 语义合并 | LLM 语义合并（防回归） | LLM 自主 edit | 人工把关 | LLM 合并（MERGE_PROMPT：包含所有信息点，≤64词） |

---

## 十、存储与检索

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **主存储** | 向量 DB（20+ 后端） | 向量 DB（Memory/ES/Chroma） | SQLite + 文件系统 | 文件系统 + 向量索引 | 文件系统（~/.hermes/skills/） | 文件系统（skill-name/） | **文件系统**（experiences.json + SKILL.md） |
| **辅助存储** | SQLite（审计历史） | 文件系统（ReMeLight） | SKILL.md 磁盘文件 | SkillBank/ + Mirror | SQLite（Session Archive） | workspace/ 迭代目录 | numpy 嵌入缓存 |
| **向量字段** | 完整摘要 | **`when_to_use`（非 content）** | Chunk summary | Skill description + prompt | 无（纯文件系统） | 无（`claude -p` 黑盒测试触发） | **Experience 的 `name: description`**（非 bullet_points） |
| **检索方式** | 语义向量搜索 | 语义向量 + LLM 重排 | FTS + 向量混合 | **向量 + BM25 混合（0.9:0.1）** | 系统提示动态加载 Skill 索引 | `claude -p` 端到端触发测试 | **任务分解 → 嵌入 top-k=3 余弦相似度** |
| **检索后处理** | 无（上层手动拼接） | **RewriteMemory**（重写为统一建议） | 直接注入 SKILL.md | LLM Skill 选择（二次筛选） | 三级加载（L0/L1/L2） | 无（触发后读完整 SKILL.md） | **Experience Rewrite + Skill Adaptation**（MLLMkb 根据当前任务和图像适配） |
| **注入位置** | 由调用方决定 | System Prompt | before_prompt_build hook | before_prompt_build hook | System Prompt（动态构建） | 触发时读取 SKILL.md | **System Prompt**（非强制注入："consider applying... your own ideas"） |

---

## 十一、离线演进

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **有无离线演进** | **无** | **无**（仅老化删除） | **有**（SkillEvo） | **有**（SkillEvo） | **有**（多层 Dreaming） | **有**（描述自动优化） | **有**（层次化合并精炼） |
| **触发方式** | — | — | 后台定时扫描 | 后台定时扫描 | Gateway flush + Cron + GEPA | 用户主动 + run_loop.py 自动 | 每批 8 样本后自动 |
| **演进机制** | — | — | 遗传算法 + replay | 遗传算法 + replay | **GEPA** | **描述优化**（eval→improve→loop, train/test 拆分） | **三层合并精炼**（逐条/容量缩减/全局 REFINE） |
| **评估方式** | — | — | replay 评分 + 测试套件 | replay 评分 + 规则检查 | LLM-as-judge + 测试套件 + 基准门控 | `claude -p` 黑盒触发率测试 + test score 选最佳 | 对照 ground_truth + 成功/失败轨迹对比 |
| **防过拟合** | — | — | 无 | 无 | 约束门：新版 ≥ 旧版 | **60/40 train/test 拆分 + blinded history** | **64 词硬约束 + "避免具体例子" + 全局精炼去过窄条目** |
| **人工参与** | — | — | 无 | 无 | GEPA 需 PR 审查 | **内容优化全程人工参与**，描述优化全自动 | 无 |

---

## 十二、LLM 调用开销

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **单次提取调用** | **1 次** | **3-4 次** | **5-9 次** | **5-8 次/轮** | **1 次** | **~2-3M tokens/技能** | **N+3+K 次/样本**（N=rollout 数） |
| **调用明细** | 提取摘要 | 提取(1)+验证(1)+重排(1)+重写(1) | 摘要(1)+去重(1)+分类(1)+评估(1)+生成(1)+验证(1)... | 改写(1)+embed(1)+选择(1)+提取(1)+决策(1)+合并(1)+审计(1) | 生成 SKILL.md(1) | 内容优化: 6子代理/eval×N轮 + 描述优化: 60×`claude -p`/轮×5轮 | 图像分析(M)+Rollout Summary(N)+Critique(1)+Experience合并(0-K)+Skill生成(1) |
| **成本特征** | 最低 | 中等 | 较高 | 最高（每轮都跑） | **最低**（但质量依赖模型能力） | **极高**（精品路线，适合少量关键技能） | 较高（但 batch 摊薄，每批 8 样本共享合并精炼） |
| **延迟敏感** | 同步/异步均可 | 后台异步 | 后台异步 | 后台异步 | 在线同步（LLM 自主调用工具） | 人机交互（不要求实时） | 离线 batch（不要求实时） |

---

## 十三、独特优势与劣势

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

### Skill-Creator
| 优势 | 劣势 |
|------|------|
| **优化即核心流程**（创建=测试=改进循环） | token 消耗极高（~2-3M/技能） |
| **4 子代理并行测试**（Grader/Comparator/Analyzer） | 需要人工全程参与 |
| **描述优化有防过拟合**（train/test 拆分） | 不适合批量生产 |
| **`claude -p` 黑盒端到端测试**触发率 | 仅限 Claude Code 生态 |
| eval-viewer SPA 精心设计的人工评审体验 | 无自动提取能力 |

### XSkill
| 优势 | 劣势 |
|------|------|
| **双流分离**（Experience 行动级 + Skill 任务级，消融实验证明互补） | 需要 ground_truth 标准答案 |
| **多路 Rollout 对比**（N=4 成功/失败自然对比） | 计算开销较大（N 次独立执行） |
| **视觉锚定**（图像+轨迹联合分析，其他系统均为纯文本） | 仅适用于多模态视觉推理场景 |
| **嵌入驱动合并**（粗筛快+精合并准，平衡成本质量） | 框架相对较新，生态不成熟 |
| **64 词硬约束**系统性对抗知识膨胀 | Experience 建议性注入，Agent 可能忽略 |
| **跨模型知识迁移**（MLLMexec ≠ MLLMkb） | |

---

## 十四、架构耦合度

| 维度 | **Mem0** | **ReMe** | **MemOS TS** | **AutoSkill** | **Hermes** | **Skill-Creator** | **XSkill** |
|------|----------|----------|--------------|---------------|------------|-------------------|------------|
| **耦合度** | **完全解耦** | **松耦合** | **紧耦合** | **紧耦合** | **完全内置** | **紧耦合** | **独立框架** |
| **运行方式** | 独立服务 / Python 库 / HTTP API | 可选：文件系统版或向量服务版 | TypeScript 内置模块 | OpenClaw 插件（Embedded/Sidecar） | Hermes Agent 内置功能 | Claude Code 插件 | 独立 Python 框架（可换骨干模型） |
| **依赖关系** | 零依赖（任何 Agent 框架可调用） | 依赖 AgentScope 或独立 HTTP 服务 | 依赖 MemOS TS 框架生命周期 | 依赖 OpenClaw Hook 系统 | 完全专属，不可分离 | 依赖 Claude Code + `claude -p` CLI | 依赖 OpenAI embedding API，骨干模型可换 |
| **框架迁移** | **最容易**（纯 API） | 容易（文件版人类可读） | 困难（深度绑定） | 困难（OpenClaw 专属） | **不可能**（完全内置） | 困难（Claude 生态专属） | **较容易**（知识 JSON/Markdown 可迁移，骨干可换） |
| **多 Agent 共享** | **支持**（通过 `agent_id` / `user_id`） | 支持（通过 workspace） | 支持（通过作用域） | 支持（user scope + library scope） | 支持（通过 skill 目录） | 支持（.skill 文件分发） | **支持**（experiences.json + SKILL.md 跨模型迁移） |

### 耦合度分析

```
完全解耦 ◄────────────────────────────────────────────────────────► 完全内置

Mem0    ReMe(reme_ai)  XSkill   ReMe(Light)  Skill-Creator  AutoSkill  MemOS TS  Hermes
 │           │           │          │             │             │          │        │
 ▼           ▼           ▼          ▼             ▼             ▼          ▼        ▼
独立服务   独立服务    独立框架    文件系统      Claude插件     OpenClaw    TS-end   Hermes
可插拔     可插拔      可换骨干    可迁移        生态绑定       插件        内置     专属
```

---

## 十五、设计光谱

```
原始保真 ←──────────────────────────────────────────────→ 通用抽象

Mem0      ReMe      MemOS      AutoSkill   Hermes   Skill-Creator  XSkill
(快照)    (经验)    (技能包)    (方法论)    (进化)    (精品课程)     (双流)
 │         │         │          │          │          │             │
 ▼         ▼         ▼          ▼          ▼          ▼             ▼
verbatim  建议性    可执行     可执行+    可执行+   测试验证的     Experience≤64词
记录      bullet    SOP       审计闭环   离线进化  可执行SOP     + Skill SOP


单路径提取 ←────────────────────────────────────────────→ 多路径对比

Mem0  ReMe  MemOS  AutoSkill  Hermes    v5设计      Skill-Creator    XSkill
(无)  (无)  (无)    (无)      (无)    (会话内检测)  (刻意构造A/B)  (N=4 Rollout)
                                         │              │              │
                                         ▼              ▼              ▼
                                     零LLM正则扫    子代理并行spawn  独立执行对比
                                     切分Path A/B   with/without     成功/失败


自动化 ←──────────────────────────────────────────────→ 人工主导

Mem0   MemOS  AutoSkill  XSkill   ReMe    Hermes   Skill-Creator
(API)  (代码)  (代码)    (batch)  (代码)  (LLM自主)  (人机迭代)
```

---

## 十六、选型建议

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
