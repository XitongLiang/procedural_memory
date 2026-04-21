# pskoett Self-Improvement Skill 分析

> 基于 https://github.com/pskoett/pskoett-ai-skills/tree/main/skills/self-improvement 源码
> 作者：pskoett（个人 skill 测试集）
> 类别：Markdown 日志型"手工自改进"系统
> 分析日期：2026-04-21

---

## 一、定位

**极简主义 procedural memory**——**无独立 LLM pipeline**，靠 agent 按 spec 把教训追加到 Markdown 文件，外加两个 shell hook 做轻触发。是所有系统里最轻、最可移植、最贴近 Claude Code 原生能力的一种。

> **成本模型澄清**：常见误读是"零 LLM 调用"，但这只对"独立管线"成立。实际成本被**摊入 agent 自身**——每轮需要：(1) 读 SKILL.md 规范；(2) grep 既有 entry 找 Pattern-Key；(3) 按格式写 Markdown；(4) 判断是否达到升格条件。折合每轮 +500~2k tokens，长会话累积可能**反超 Mem0（1 次专用调用）或 Hermes（1 次）**。它的真正优势是"零基础设施"而非"零成本"。

跟其他系统的根本差异：

| 维度 | pskoett self-improvement | Hermes | OpenSpace | XSkill | Mem0 |
|------|--------------------------|--------|-----------|--------|------|
| **提取主体** | **Agent 自己按 spec 写 Markdown** | Agent 自己调 `skill_manage` 工具 | Analyzer LLM 独立分析 | 独立 MLLMkb | 调用方显式 API |
| **结构化程度** | Markdown + YAML-like 字段 | SKILL.md | SKILL.md + scripts | Exp(≤64词) + SKILL.md | 文本快照 |
| **去重** | **手动 grep + 人工 See Also** | 无 | 嵌入 + 目录级 | 嵌入 ≥ 0.70 + LLM | 无 |
| **检索** | **无**（靠 pre-flight-check 提醒） | 动态加载 | BM25+向量+LLM | embed top-k | 语义向量 |
| **演进** | 手动状态机（pending→resolved→promoted） | LLM patch | FIX/DERIVED/CAPTURED | add/modify/refine | ADD |
| **质量门控** | Recurrence ≥ 3 + 跨 2 任务 + 30 天 | 无 | 4 比率+tool penalty | 64 词硬约束 | 无 |
| **多 agent 生态** | **Claude Code / Codex / Copilot / OpenClaw** | Hermes 专属 | MCP 跨 host | 独立框架 | 独立服务 |
| **代码量** | **~10 文件 + 2 bash hook（~600 行 md）** | 大 | ~7800 行 | 中等 | 小 |

**一句话**：Claude Code 时代的"手工版 procedural memory"——架构简洁到极致，但功能也最薄。

---

## 二、产物形态

### 2.1 三类 Markdown 文件

```
.learnings/
  ├─ LEARNINGS.md        # 用户纠正 / 知识缺口 / 最佳实践
  ├─ ERRORS.md           # 命令或工具失败
  └─ FEATURE_REQUESTS.md # 用户想要但还没有的能力
```

这个三分**与 GSPD 的 Fb/TP/TSE 看似相似，但切分维度不同**：

| 系统 | 切分维度 | 分类 |
|------|---------|------|
| **pskoett** | **现象分类**（What happened） | 错误 / 纠正 / 缺失 |
| **GSPD** | **强度分类**（How binding） | 硬约束 / 偏好 / 软建议 |

### 2.2 Learning Entry 格式

```markdown
## [LRN-YYYYMMDD-XXX] category

**Logged**: 2026-04-21T12:00:00Z
**Priority**: low | medium | high | critical
**Status**: pending | in_progress | resolved | wont_fix | promoted
**Area**: frontend | backend | infra | tests | docs | config

### Summary
One-line description

### Details
Full context

### Suggested Action
Concrete fix

### Metadata
- Source: conversation | error | user_feedback
- Related Files: path/to/file.ext
- Tags: tag1, tag2
- See Also: LRN-20260110-001
- Pattern-Key: simplify.dead_code | harden.input_validation
- Recurrence-Count: 1
- First-Seen: 2026-01-15
- Last-Seen: 2026-01-15
```

**关键字段**：
- **Pattern-Key**：稳定的 dedupe 键，人工分配（如 `simplify.dead_code`）
- **Recurrence-Count**：同 key 的出现次数
- **See Also**：关联已有 entry，维护问题图谱
- **Status 状态机**：`pending → in_progress → resolved → promoted`（或 `wont_fix`）

### 2.3 ID 生成规则

`TYPE-YYYYMMDD-XXX` 格式：
- TYPE：`LRN`（learning）/ `ERR`（error）/ `FEAT`（feature）
- YYYYMMDD：当日
- XXX：序号或 3 字符随机（`001` / `A7B`）

---

## 三、三级升格漏斗（核心设计）

```
┌─────────────────────────────────────────────────────────┐
│ Level 1: Learning Entry                                 │
│   .learnings/LEARNINGS.md / ERRORS.md / FEATURE_REQUESTS.md │
│                                                          │
│   升格条件：                                              │
│     • Recurrence-Count ≥ 3                              │
│     • 跨 ≥ 2 个不同任务                                   │
│     • 在 30 天窗口内                                      │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Level 2: Project Memory                                  │
│   CLAUDE.md       → 项目事实、约定、gotcha                │
│   AGENTS.md       → Agent 工作流、工具用法、自动化规则     │
│   .github/copilot-instructions.md → Copilot 项目上下文   │
│   SOUL.md (OpenClaw) → 行为准则、沟通风格                 │
│   TOOLS.md (OpenClaw) → 工具能力、集成 gotcha             │
│                                                          │
│   升格条件（进一步）：                                     │
│     • recurring + resolved + 广泛适用                    │
│     • 用户说 "save this as a skill"                      │
│     • Category: best_practice with broad applicability   │
└─────────────────────┬───────────────────────────────────┘
                      │
                      ▼
┌─────────────────────────────────────────────────────────┐
│ Level 3: Independent Skill                               │
│   skills/<name>/SKILL.md                                │
│   通过 extract-skill.sh 脚手架创建                        │
│   原 Learning 状态更新为 `promoted_to_skill`              │
└─────────────────────────────────────────────────────────┘
```

### 3.1 为什么三级

这对应作者在 README 自述的 **"inner loop + outer loop"** 理念：

```
Inner loop（会话内）：
  verify-gate / context-surfing / intent-framed-agent
  → 会话内检测问题、自修、防漂移

Outer loop（跨会话）：
  self-improvement / learning-aggregator / harness-updater
  → 累积教训 → 结晶为规则 → 注入新会话
```

**核心洞察**：agent 有"context 峰值"时刻——规划后、执行中、完成时、学习后——**不同 skill 针对不同峰值**。

### 3.2 升格阈值的量化设计

`Recurrence ≥ 3 + 跨 ≥ 2 任务 + 30 天窗口` 三个条件同时满足才升格，与 GSPD 的 `support_score = 0.50·mention_ratio + 0.25·recency + 0.25·hard_ratio` 思路一致，**都是多维度联合门控**，但 pskoett 是硬规则，GSPD 是连续分数。

---

## 四、Hook 驱动的"轻触发"

三个 hook 构成全部自动化：

| Hook | 绑定事件 | 作用 | 代码位置 |
|------|---------|------|---------|
| `activator.sh` | `UserPromptSubmit` | 每轮用户输入后注入 50-100 token 提醒 | `scripts/activator.sh` |
| `error-detector.sh` | `PostToolUse(Bash)` | grep 16 种错误特征，命中提醒 | `scripts/error-detector.sh` |
| `handler.ts` | `agent:bootstrap`（OpenClaw） | 启动时注入 learning 检查提醒 | `hooks/openclaw/handler.ts` |

### 4.1 activator.sh（全部代码）

```bash
#!/bin/bash
cat << 'EOF'
<self-improvement-reminder>
After completing this task, evaluate if extractable knowledge emerged:
- Non-obvious solution discovered through investigation?
- Workaround for unexpected behavior?
- Project-specific pattern learned?
- Error required debugging to resolve?

If yes: Log to .learnings/ using the self-improvement skill format.
If high-value (recurring, broadly applicable): Consider skill extraction.
</self-improvement-reminder>
EOF
```

**作用**：不做任何判断，只在每次用户输入后提醒 agent 自己评估。质量完全依赖 agent 是否严格按 spec 执行。

### 4.2 error-detector.sh（核心逻辑）

```bash
ERROR_PATTERNS=(
    "error:" "Error:" "ERROR:"
    "failed" "FAILED"
    "command not found" "No such file" "Permission denied"
    "fatal:" "Exception" "Traceback"
    "npm ERR!" "ModuleNotFoundError" "SyntaxError" "TypeError"
    "exit code" "non-zero"
)

OUTPUT="${CLAUDE_TOOL_OUTPUT:-}"
for pattern in "${ERROR_PATTERNS[@]}"; do
    if [[ "$OUTPUT" == *"$pattern"* ]]; then
        # 输出 <error-detected> 提醒 agent 考虑记录
        break
    fi
done
```

**简单粗暴**：纯字符串匹配，会误报（如日志里出现 `error: 0` 也命中），但实现零成本。

### 4.3 OpenClaw handler.ts

```typescript
const handler: HookHandler = async (event) => {
  if (event.type !== 'agent' || event.action !== 'bootstrap') return;

  event.context.bootstrapFiles ??= [];
  event.context.bootstrapFiles.push({
    name: 'SELF_IMPROVEMENT_REMINDER.md',
    content: REMINDER_CONTENT,
    virtual: true,
  });
};
```

在 OpenClaw 生态里，通过 `agent:bootstrap` 事件把 reminder 作为虚拟文件注入 bootstrap 流程。

---

## 五、extract-skill.sh 脚手架

第三级升格的落地工具：

```bash
./skills/self-improvement/scripts/extract-skill.sh docker-m1-fixes --dry-run
./skills/self-improvement/scripts/extract-skill.sh docker-m1-fixes
```

**做了什么**：
1. 校验 skill-name 格式（小写 + 连字符）
2. 检查目标目录不存在
3. 从 `assets/SKILL-TEMPLATE.md` 生成新 SKILL.md 骨架（含 TODO 占位符）
4. 提示后续手工填充步骤

**注意**：这**不是自动提取**——只是脚手架，实际内容还是要 agent 手工从 Learning entry 搬到新 SKILL.md 里。

### 5.1 Skill 抽取判据

满足任一即可：

| 判据 | 说明 |
|------|------|
| **Recurring** | 有 2+ See Also 链接到类似问题 |
| **Verified** | Status = resolved 且 fix 已验证 |
| **Non-obvious** | 需要实际 debug/调研才能发现的 |
| **Broadly applicable** | 不限特定项目、跨 codebase 有用 |
| **User-flagged** | 用户说 "save this as a skill" |

### 5.2 质量门

抽取前检查清单：
- [ ] Solution tested and working
- [ ] Description clear without original context
- [ ] Code examples self-contained
- [ ] No project-specific hardcoded values
- [ ] Follows skill naming conventions

---

## 六、Simplify-and-Harden 反馈环（生态协作）

这个 skill **不是独立工作的**，它与同一仓库的其他 skill 形成反馈循环：

```
simplify-and-harden（末端质量审查）
    │
    │  产出 candidates，每条含 pattern_key
    ▼
self-improvement.ingestion_workflow
    │
    │  for each candidate:
    │    grep "Pattern-Key: <pattern_key>" .learnings/LEARNINGS.md
    │    found → Recurrence-Count++, Last-Seen 更新
    │    not found → 新建 LRN entry，Source: simplify-and-harden
    ▼
learning-aggregator（跨会话聚合）
    │
    │  发现 Recurrence ≥ 3 && 跨 2 任务 && 30 天
    ▼
harness-updater → 晋升到 CLAUDE.md / AGENTS.md / SOUL.md
    │
    ▼
eval-creator → 把晋升规则变成回归测试
    │
    ▼
pre-flight-check → 下次会话开始时把所有上述内容 surface
```

**这才是完整故事**：self-improvement 只是**漏斗的入口**，后面还有 4 个 skill 承接。单独看 self-improvement 会觉得"太弱"，看整个 pipeline 才能体会设计思路。

---

## 七、独特优势

| 优势 | 说明 |
|------|------|
| **极简主义** | 无独立 LLM pipeline，靠 agent 自身的写文件能力（成本摊入 agent 主循环） |
| **极高可移植性** | 纯 Markdown + bash，跨 Claude Code / Codex / Copilot / OpenClaw |
| **Inner/Outer loop 概念化** | README 明确划分两层闭环，架构思路最清晰 |
| **Recurrence ≥ 3 + 2 任务 + 30 天** | 多维硬阈值升格规则，防止一次失败被当教训 |
| **Pattern-Key 稳定去重** | 人工分配 key（如 `simplify.dead_code`），避免 LLM 改写漂移 |
| **三级漏斗层次清晰** | Learning → Project Memory → Skill 路径明确 |
| **生态协作** | 与 simplify-and-harden / learning-aggregator / harness-updater / eval-creator / pre-flight-check 形成完整 outer loop |
| **状态机完整** | pending / in_progress / resolved / wont_fix / promoted / promoted_to_skill 覆盖所有生命周期 |
| **零运行成本** | 不需任何外部服务，仓库内完成一切 |

---

## 八、局限与劣势

| 劣势 | 说明 |
|------|------|
| **质量完全依赖 agent 严格性** | 没有 LLM 辅助提取，agent 偷懒少填字段就直接降质 |
| **无自动检索** | 教训只在 pre-flight-check 主动读或 agent 自觉 grep 时激活 |
| **Markdown 线性增长** | 无容量管理，`.learnings/*.md` 会持续膨胀到难以维护 |
| **Pattern-Key 需人工约定** | 没有自动聚类，同类问题用不同 key 会分散计数 |
| **error-detector.sh 粗糙** | 16 个字符串特征容易误报（`error: 0` / `testing error paths` 都会命中） |
| **无语义去重** | See Also 靠人手动链接，跨 entry 语义相似度无法自动发现 |
| **无跨项目共享** | `.learnings/` 是本地目录，跨项目经验不流通（对比：OpenSpace 有云端） |
| **升格决策依赖人** | 虽然有 3+2+30 硬阈值，但最终还是 agent 判断"是否广泛适用"，主观性高 |

---

## 九、对 GSPD 的启发

### 9.1 值得借鉴

**1. 三级升格漏斗**

```
GSPD 目前：Fb/TP/TSE 各自升格（候选池 → active）
可加一层：active 条目高频复现 → 考虑提升为 system-prompt 硬规则 / skill
```

对应到 v5 设计：`support_score ≥ 0.7 + cross_session_count ≥ N` 时提示用户"要不要把这条放进 project CLAUDE.md？"

**2. Pattern-Key 稳定去重键**

GSPD 目前 Fb 的去重靠 hybrid（向量 + LLM），但 LLM 改写可能漂移 content 字符串。**可以在 Fb 上加一个 `pattern_key` 字段**（用户可指定，或 LLM 从首个 Fb 提取时生成），作为稳定 dedupe 锚点。

```sql
-- 建议的 schema 增补
ALTER TABLE feedback ADD COLUMN pattern_key TEXT;
CREATE INDEX idx_fb_pattern_key ON feedback(pattern_key);
```

**3. 30 天窗口 + 2 任务**作为跨任务泛化判据

GSPD 现在的 `recency` 是连续衰减，**可以额外加一个"跨任务出现次数"**作为 Fb/TP 升格的门槛。单任务内复现 10 次 ≠ 跨 3 任务各出现 1 次，后者更能体现"可泛化"。

**4. Inner/Outer Loop 概念**

GSPD 已经有类似设计（在线学习 + 离线 dreaming），但可以在文档里**显式用 "Inner/Outer Loop" 的说法**，比"在线/离线"更贴近 agent 工程语言。

### 9.2 可以忽略

- 纯 Markdown 存储和 grep 检索 — GSPD 向量检索已经超越
- `extract-skill.sh` 脚手架 — GSPD 的 TSE 本身就是 skill-like 产物
- 三类文件物理分离 — GSPD 的三类已经在 schema 层分表

---

## 十、与本目录其他系统对比定位

```
自动化程度 ←──────────────────────────────────────────→ 人工主导

Mem0   MemOS  AutoSkill  XSkill   ReMe    Hermes   OpenSpace  Skill-Creator  pskoett
(API)  (代码)  (代码)    (batch)  (代码)  (LLM自主) (规则+LLM)  (人机迭代)     (全人工+hook)
                                                                                    ▲
                                                                                    │
                                                                        最右：agent 按 spec 手写
```

### 10.1 在"设计光谱"中的位置

```
原始保真 ←────────────────────────────────────────────→ 通用抽象

Mem0   ReMe   MemOS   AutoSkill   Hermes   OpenSpace   Skill-Creator   XSkill   pskoett
(快照) (经验) (技能)   (方法论)    (进化)    (自演进)     (精品课程)      (双流)    (日志漏斗)
                                                                                   ▲
                                                                                   │
                                                                           特别：不试图"通用"
                                                                           而是"记录→升格→结晶"
                                                                           的工程学漏斗
```

pskoett 的位置特殊——它不沿着"保真 vs 抽象"光谱定位，而是另一个维度：**"依赖 LLM vs 依赖 Spec"**。

---

## 十一、选型建议

| 场景 | 是否推荐 pskoett |
|------|------------------|
| **个人开发 + Claude Code** | **强推荐**——轻量、无依赖、跨工具 |
| **小团队 + 多 agent 混用** | 推荐——Claude Code / Codex / Copilot 通吃 |
| **需要云端共享技能** | 不推荐——选 OpenSpace |
| **需要自动提取不愿让 agent 手写** | 不推荐——选 Hermes / XSkill / OpenSpace |
| **需要严格质量门控** | 不推荐——选 Skill-Creator / OpenSpace |
| **作为"outer loop 参考设计"** | **强推荐**——概念清晰，容易移植 |
| **企业级系统 + 合规审计** | 不推荐——无结构化存储、无审计表 |

---

## 十二、关键文件速查

| 文件 | 作用 |
|------|------|
| `skills/self-improvement/SKILL.md` | 主文档（573 行），完整 workflow 定义 |
| `skills/self-improvement/scripts/activator.sh` | UserPromptSubmit hook（20 行 bash） |
| `skills/self-improvement/scripts/error-detector.sh` | PostToolUse(Bash) hook（55 行 bash） |
| `skills/self-improvement/scripts/extract-skill.sh` | Skill 脚手架生成器（204 行 bash） |
| `skills/self-improvement/hooks/openclaw/handler.ts` | OpenClaw agent:bootstrap hook（47 行 TS） |
| `skills/self-improvement/hooks/openclaw/HOOK.md` | OpenClaw hook 文档 |
| `skills/self-improvement/references/hooks-setup.md` | Hook 配置详解 |
| `skills/self-improvement/references/openclaw-integration.md` | OpenClaw 集成方式 |
| `skills/self-improvement/references/examples.md` | 完整 Learning/Error/Feature 示例 |
| `skills/self-improvement/assets/LEARNINGS.md` | 空模板 |
| `skills/self-improvement/assets/SKILL-TEMPLATE.md` | Skill 生成模板 |

### 生态内其他协作 skill

| Skill | 作用 |
|-------|------|
| `simplify-and-harden` | 任务末端的质量/安全审查，产出 recurring pattern candidates |
| `learning-aggregator` | 跨会话聚合 `.learnings/` |
| `harness-updater`（README 提到但未在仓库看到） | 把聚合规则写入 CLAUDE.md / AGENTS.md |
| `eval-creator` | 把晋升规则变成回归测试 |
| `pre-flight-check` | 会话开始时 surface 已有 learnings |
| `skill-pipeline` | 路由任务到合适的 skill 组合 |

---

## 参考资料

- [pskoett/pskoett-ai-skills](https://github.com/pskoett/pskoett-ai-skills) — 源码仓库
- [self-improvement skill 目录](https://github.com/pskoett/pskoett-ai-skills/tree/main/skills/self-improvement)
- [Agent Skills 规范](https://agentskills.io/specification)
- [Procedural-Memory-System-Comparison.md](./Procedural-Memory-System-Comparison.md) — 本目录主对比文档
