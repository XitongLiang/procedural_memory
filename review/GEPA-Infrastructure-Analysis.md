# GEPA: Reflective Prompt Evolution 基础设施全解析

> 基于论文 [arXiv:2507.19457](https://arxiv.org/abs/2507.19457) (ICLR 2026 Oral) + [gepa-ai/gepa](https://github.com/gepa-ai/gepa) 源码

---

## 一、核心思想

GEPA（Genetic-Pareto）把 **prompt 当作可优化参数**，用进化算法 + 自然语言反思来迭代改进，不动模型权重。

类比：

| 模型训练 | GEPA |
|---------|------|
| 参数 = 权重矩阵 W（浮点数） | 参数 = Prompt 文本（自然语言字符串） |
| 前向传播 = model(W, input) | 前向传播 = LLM(prompt, input) |
| 损失函数 = cross-entropy | 损失函数 = LLM-as-judge / exact match |
| 梯度 = dL/dW（数值向量） | "梯度" = 自然语言反思（"这个 prompt 缺少边界条件处理"） |
| 优化器 = Adam | 优化器 = GEPA（反思 → 变异 → Pareto 选择） |

---

## 二、整体架构

```
输入：系统 Phi、训练数据 D_train、评估指标 mu、反馈函数 mu_f、预算 B

Step 0: 数据分割
  D_train → D_feedback（收集学习信号）+ D_pareto（候选选择验证）

Step 1: 初始化
  候选池 P ← [原始系统 Phi]
  在 D_pareto 上评估初始系统

Step 2: 主循环（直到预算耗尽）
  ┌─ 2a. SelectCandidate：从 Pareto 前沿采样一个候选
  │  2b. 选模块：Round-robin 选择要更新的模块 j
  │  2c. 采样：从 D_feedback 取 minibatch（默认 3 条）
  │  2d. 执行+收集反馈：跑系统，收集轨迹和分数
  │  2e. UpdatePrompt：LLM 反思轨迹，生成新 prompt
  │  2f. 快速验证：在 minibatch 上比较新旧分数
  │  2g. 如果改进 → 加入候选池，在 D_pareto 上完整评估
  └─ 2h. 可选：Merge（交叉操作，最多 5 次）

Step 3: 返回 D_pareto 上均分最高的系统
```

---

## 三、六个核心步骤详解

### Step 2a. SelectCandidate — Pareto 前沿采样

**作用**：从候选池中选一个"有潜力"的 prompt 变体作为本轮进化的起点。

**算法**：

```
输入：候选池 P，每个候选在每个验证实例上的分数 S

1. 对每个验证实例 i：
     找到最高分 s*[i] = max_k S[k][i]
     P*[i] = {所有在实例 i 上达到最高分的候选}

2. 提取候选集 C = 所有出现在某个 P*[i] 中的候选

3. 支配性剪枝：
     如果候选 A 出现在的 Pareto 集合 ⊂ 候选 B 出现的集合
     → A 被 B 支配，移除 A

4. 计算频率 f[k] = 候选 k 出现在 Pareto 前沿的实例数
   按概率 proportional to f[k] 采样

输出：一个候选
```

**示例**：

假设有 3 个候选在 5 个验证实例上的分数：

```
         实例1  实例2  实例3  实例4  实例5
候选 A:   0.9    0.3    0.8    0.5    0.7
候选 B:   0.7    0.9    0.6    0.8    0.5
候选 C:   0.6    0.4    0.5    0.4    0.3   ← 每项都不如 A 或 B

Pareto 前沿（每实例最优者）：
  实例1: A    实例2: B    实例3: A    实例4: B    实例5: A

频率：A 出现 3 次，B 出现 2 次，C 出现 0 次（被淘汰）
采样概率：A = 3/5 = 60%，B = 2/5 = 40%
```

**为什么不直接选均分最高的？** 论文实验表明 Pareto 采样比 SelectBest 高 8.17%，因为它保留了多样性——某些候选在特定类型任务上更强。

---

### Step 2b. 选择目标模块 — Round-robin

**作用**：决定本轮优化 prompt 的哪个部分。

```
一个 Compound AI 系统可能有多个 LLM 模块，每个有自己的 prompt：

  模块 1: 查询改写 prompt
  模块 2: 检索 prompt
  模块 3: 生成回答 prompt

Round-robin = 轮流选：第 1 轮改模块 1，第 2 轮改模块 2，第 3 轮改模块 3，第 4 轮又回到模块 1...
```

**示例（HotpotQA 多跳问答）**：

```
系统有两个模块：
  hop_1_prompt: "根据问题生成搜索查询"
  hop_2_prompt: "根据第一跳结果，生成第二个查询"

第 1 轮 → 选 hop_1_prompt 来优化
第 2 轮 → 选 hop_2_prompt 来优化
第 3 轮 → 回到 hop_1_prompt
...
```

---

### Step 2c-2d. 采样 + 执行 + 收集反馈

**作用**：从训练集中取少量样本，跑系统，记录完整轨迹和反馈。

```
输入：D_feedback 中随机取 minibatch（默认 b=3 条）

对每条样本执行：
  1. 跑系统 → 收集完整轨迹：
     - 推理链（CoT）
     - 工具调用和返回值
     - 中间输出
     - 最终输出

  2. 评估 → 收集反馈：
     - 分数（0.0 ~ 1.0）
     - 反馈文本（错误信息、编译输出、评估日志）

输出：[(轨迹_1, 分数_1, 反馈_1), (轨迹_2, 分数_2, 反馈_2), (轨迹_3, 分数_3, 反馈_3)]
```

**反馈函数 mu_f 的具体实现取决于任务**：

| 任务 | 反馈来源 |
|------|---------|
| HotpotQA（问答） | 检索到的文档、推理链、与标准答案的对比 |
| IFBench（指令遵循） | 哪些约束满足了、哪些没满足 |
| AIME（数学） | 计算过程、中间步骤验证 |
| NPUEval（代码优化） | 编译错误、性能 profiling 数据 |

**示例（HotpotQA）**：

```
样本：Q = "Who directed the film that Tom Hanks starred in about a FedEx employee?"

执行轨迹：
  hop_1: 搜索 "Tom Hanks FedEx employee film" → 返回 Cast Away 页面
  hop_2: 搜索 "Cast Away" → 返回电影信息
  answer: "Robert Zemeckis"

反馈：
  score: 1.0（正确）
  feedback_text: "正确检索到 Cast Away，正确识别导演"

另一个样本失败：
  hop_1: 搜索 "Tom Hanks movie" → 返回 Forrest Gump 页面（错误方向）
  hop_2: 搜索 "Forrest Gump director" → Robert Zemeckis
  answer: "Robert Zemeckis"（碰巧也对，但推理路径错误）

反馈：
  score: 1.0
  feedback_text: "答案正确但第一跳检索不精确，搜索词缺少 FedEx 关键词"
```

---

### Step 2e. UpdatePrompt — 核心：反思式变异

**作用**：这是 GEPA 的灵魂步骤——让 LLM 读取执行轨迹，反思失败原因，生成改进后的 prompt。

**代码结构**（源码 `proposer/reflective_mutation/reflective_mutation.py` + `strategies/instruction_proposal.py`）：

```python
# 伪代码，基于源码结构还原
class ReflectiveMutationProposer:
    def propose_new_texts(self, component_to_update, base_instruction, dataset_with_feedback):
        """
        输入：
          - component_to_update: 要优化的模块名
          - base_instruction: 当前 prompt 文本
          - dataset_with_feedback: [(样本, 轨迹, 分数, 反馈文本), ...]
        
        输出：
          - new_instruction: 改进后的 prompt 文本
        """
        result = InstructionProposalSignature.run_with_metadata(
            current_instruction_doc=base_instruction,
            dataset_with_feedback=dataset_with_feedback,
            prompt_template=self.reflection_prompt_template
        )
        return result.new_instruction
```

**发给 reflection_lm 的 meta-prompt 结构**（基于论文附录 B 和源码还原）：

```markdown
你是一个 prompt 优化专家。下面是一个 AI 系统中某个模块当前的指令，
以及它在一批任务上的执行轨迹和评估反馈。

## 当前指令

<curr_param>
{当前 prompt 文本}
</curr_param>

## 执行轨迹与反馈

<side_info>
### 样本 1
输入：{问题/任务}
执行轨迹：
  - Step 1: {推理/工具调用}
  - Step 2: {推理/工具调用}
  - ...
输出：{系统输出}
评分：{0.0-1.0}
反馈：{具体反馈文本}

### 样本 2
...

### 样本 3
...
</side_info>

## 你的任务

分析上述执行轨迹，找出当前指令的不足之处：
1. 哪些样本失败了？为什么？
2. 成功的样本有什么共同模式？
3. 当前指令中缺少什么关键引导？

然后输出一个改进后的指令版本，用 ``` 包裹。
改进应该：
- 解决观察到的具体失败模式
- 保留已经有效的策略
- 加入可泛化的新策略（不是针对单个样本的特例修复）
```

**示例（HotpotQA hop_2 prompt 优化过程）**：

优化前的 prompt：
```
Given the context from the first search, generate a follow-up search query.
```

反思 LLM 收到的反馈：
```
样本 1: 第二跳搜索词太泛，返回无关文档。score=0.0
样本 2: 成功利用第一跳信息缩小范围。score=1.0
样本 3: 重复搜索第一跳已有的信息，浪费检索。score=0.0
```

反思 LLM 输出的改进 prompt：
```
You are a multi-hop search specialist. Given the first hop's results (summary_1),
your job is to generate a precise follow-up query that:

1. Identify what is STILL MISSING - look for implicit connections or broader
   context hinted at but not directly stated in summary_1
2. DO NOT repeat known facts - never re-query information already present
3. Bridge the gap - your query should connect what you know to what you need
4. Be specific - include entity names, dates, or distinguishing details
   from summary_1 to narrow results

Output only the search query, nothing else.
```

---

### Step 2f. 快速验证 — Minibatch 比较

**作用**：在正式加入候选池之前，先在 minibatch 上快速检查新 prompt 是否真的更好。

```
在同样的 3 条 minibatch 样本上：
  旧 prompt 分数：[0.0, 1.0, 0.0] → 均分 0.33
  新 prompt 分数：[1.0, 1.0, 0.0] → 均分 0.67

0.67 > 0.33 → 通过，加入候选池
否则 → 丢弃这个变体，不浪费完整评估的预算
```

**为什么需要这步？** 完整评估（在 ~300 条 D_pareto 上跑）很贵。minibatch 快速筛选可以过滤掉明显退化的变体，节省 rollout 预算。

---

### Step 2g. 完整评估 + 加入候选池

```
通过 minibatch 验证的新候选：
  → 在 D_pareto（~300 条）上完整评估
  → 记录每个实例的分数
  → 加入候选池 P
  → 更新 Pareto 前沿
  → 下一轮 SelectCandidate 就可以采样到它
```

---

### Step 2h. Merge — 系统感知交叉（可选）

**作用**：把不同进化分支学到的互补策略合并。

**触发条件**：候选池中存在独立进化的分支，各自在不同模块上表现好。

```
进化树示例：

  原始系统
    ├── 分支 A（优化了 hop_1_prompt，hop_2 没动）
    │     hop_1: 很强
    │     hop_2: 一般
    │
    └── 分支 B（优化了 hop_2_prompt，hop_1 没动）
          hop_1: 一般
          hop_2: 很强

Merge：取 A 的 hop_1 + B 的 hop_2 → 新候选 C
  hop_1: 很强（来自 A）
  hop_2: 很强（来自 B）
```

**算法**（源码 `proposer/merge.py`）：

```
1. find_common_ancestor_pair()：
   - 随机选两个候选
   - 递归找它们的共同祖先
   - 确保它们在不同模块上做了改进（互补性）

2. sample_and_attempt_merge()：
   - 对比两个候选与祖先在每个模块上的差异
   - 选每个模块的最优版本组合

3. 验证：
   - 在验证子集上评估合并结果
   - 仅当合并后分数 >= 两个父本的最大值时才接受
```

**约束**：最多调用 5 次 / 每次优化运行。论文实验显示 GPT-4.1 Mini 上额外提升 2-5%，但 Qwen3 8B 上效果不稳定。

---

## 四、代码结构

```
src/gepa/
├── core/adapter.py                    # GEPAAdapter 基础接口
├── adapters/                          # 不同场景的适配器
│   ├── default_adapter/               # 默认 prompt 优化
│   ├── dspy_full_program_adapter/     # DSPy 集成
│   ├── generic_rag_adapter/           # RAG 系统优化
│   ├── mcp_adapter/                   # MCP 工具优化
│   └── terminal_bench_adapter/        # 终端任务优化
├── proposer/                          # 变异提案器
│   ├── reflective_mutation/           # 核心：反思式变异
│   │   ├── base.py
│   │   └── reflective_mutation.py     # UpdatePrompt 实现
│   ├── base.py
│   └── merge.py                       # Merge 交叉操作
├── strategies/                        # 选择策略
│   ├── candidate_selector.py          # Pareto / TopK / EpsilonGreedy
│   ├── component_selector.py          # Round-robin 模块选择
│   ├── batch_sampler.py               # Minibatch 采样
│   ├── acceptance.py                  # 快速验证接受准则
│   ├── eval_policy.py                 # 评估策略
│   └── instruction_proposal.py        # Meta-prompt 构造与解析
├── api.py                             # gepa.optimize() 入口
├── optimize_anything.py               # gepa.optimize_anything() 入口
└── lm.py                              # LLM 调用封装
```

---

## 五、三种使用方式

### 方式 1: 直接 API（最简）

```python
import gepa

result = gepa.optimize(
    seed_candidate={"system_prompt": "Extract contact information from text."},
    trainset=train_examples,
    valset=val_examples,
    task_lm="openai/gpt-4.1-mini",        # 执行任务的模型
    reflection_lm="openai/gpt-5",          # 做反思的模型（推荐用强模型）
    max_metric_calls=150,                   # 总评估预算
)
print(result.best_candidate)
```

### 方式 2: 通用优化（optimize_anything）

```python
result = gepa.optimize_anything(
    seed_candidate="你的初始 prompt 或任意文本工件",
    evaluator=lambda candidate: my_eval(candidate),  # 返回 float 分数
    objective="优化这个 prompt 使得数学推理准确率最高",
    config=GEPAConfig(engine=EngineConfig(max_metric_calls=100))
)
```

### 方式 3: DSPy 集成（推荐用于 AI pipeline）

```python
import dspy

# 定义系统
class MultiHopQA(dspy.Module):
    def __init__(self):
        self.hop1 = dspy.ChainOfThought("question -> search_query")
        self.hop2 = dspy.ChainOfThought("question, context -> search_query")
        self.answer = dspy.ChainOfThought("question, context -> answer")

    def forward(self, question):
        q1 = self.hop1(question=question)
        ctx1 = search(q1.search_query)
        q2 = self.hop2(question=question, context=ctx1)
        ctx2 = search(q2.search_query)
        return self.answer(question=question, context=f"{ctx1}\n{ctx2}")

# 定义评估指标（可返回 float 或 dict）
def metric(gold, pred, trace=None):
    score = float(gold.answer == pred.answer)
    feedback = f"Expected: {gold.answer}, Got: {pred.answer}"
    return {"score": score, "feedback": feedback}

# GEPA 优化
optimizer = dspy.GEPA(
    metric=metric,
    auto='medium',                                    # 预设预算
    reflection_lm=dspy.LM('openai/gpt-5', temperature=1.0, max_tokens=32000),
    candidate_selection_strategy='pareto',             # 默认 Pareto
    reflection_minibatch_size=3,                       # 每轮反思样本数
    use_merge=True,                                    # 启用交叉
    max_merge_invocations=5,
)

optimized_qa = optimizer.compile(
    student=MultiHopQA(),
    trainset=train_examples,
    valset=val_examples,
)
```

---

## 六、关键超参数

| 参数 | 默认值 | 含义 |
|------|--------|------|
| `reflection_minibatch_size` | 3 | 每轮反思使用的样本数 |
| `auto` | 'light' / 'medium' / 'heavy' | 预算级别 |
| `max_metric_calls` | 取决于 auto | 总评估调用次数 |
| `candidate_selection_strategy` | 'pareto' | 候选选择策略 |
| `use_merge` | True | 是否启用 Merge 交叉 |
| `max_merge_invocations` | 5 | 最大 Merge 次数 |
| `skip_perfect_score` | True | 跳过已满分的样本 |
| `failure_score` | 0.0 | 执行失败的默认分数 |
| `perfect_score` | 1.0 | 最高分数 |
| `reflection_lm` | 需指定 | 反思用的模型（推荐强模型） |

---

## 七、实验结果

### vs GRPO（强化学习）

| 任务 | GRPO (24000 rollouts) | GEPA (~300 rollouts) | 差距 |
|------|----------------------|---------------------|------|
| HotpotQA | 基线 | +6% | 35x 更少推理 |
| AIME-2025 | 基线 | +20% | 最大提升 |
| 平均 | 基线 | +6% | — |

### vs MIPROv2

| 对比 | 结果 |
|------|------|
| GEPA vs MIPROv2 | 平均高 10% |
| GEPA+Merge vs MIPROv2 | 平均高 14% |
| AIME-2025 | +12% |
| Prompt 长度 | GEPA 短 9.2 倍 |

### 推理时搜索（Inference-time）

| 任务 | 基线 | GEPA |
|------|------|------|
| NPUEval（AMD XDNA2 代码优化） | GPT-4o: 4.25% | 30.52% 向量利用率 |

---

## 八、局限性

1. **不能替代 RL/SFT** — 模型权重决定能力上限，prompt 优化只能在上限内调度
2. **依赖强反思模型** — reflection_lm 推荐 GPT-5 / Claude Opus 级别，用弱模型做反思效果差
3. **Merge 不稳定** — 在 Qwen3 8B 上效果不一致
4. **任务限制** — 在模型已有能力覆盖的任务上效果好，需要新能力的任务无能为力

---

## 九、与相关工作的定位

```
                    优化什么？
                    │
        ┌───────────┼───────────┐
        权重                   Prompt
        │                      │
    ┌───┼───┐            ┌─────┼─────┐
   SFT  RL  Pretrain    DSPy框架     独立工具
   │                     │            │
   GRPO                ┌─┼─┐        gepa.optimize()
   PPO              MIPROv2 GEPA    gepa.optimize_anything()
   DPO              Bootstrap
                    FewShot
```

Sources:
- 论文: https://arxiv.org/abs/2507.19457
- 代码: https://github.com/gepa-ai/gepa
- DSPy API: https://dspy.ai/api/optimizers/GEPA/overview/
- Pydantic 示例: https://pydantic.dev/articles/prompt-optimization-with-gepa