# DIET: Dual Intent and Entity Transformer 架构全解析

> 基于论文 [arXiv:2004.09936](https://arxiv.org/abs/2004.09936) + [RasaHQ/rasa](https://github.com/RasaHQ/rasa) 源码

---

## 一、核心思想

DIET（Dual Intent and Entity Transformer）把对话 NLU 看作一个**多任务联合学习问题**，用一个轻量 transformer 同时预测意图和实体，并允许稀疏特征和预训练 embedding 自由组合。核心目标：**在不依赖大规模预训练 LM 的前提下，用 1/6 的训练时间达到 BERT fine-tuning 级别的精度**。

类比：

| 重量级 NLU（BERT fine-tune） | DIET |
|---|---|
| 输入 = token ids | 输入 = 稀疏特征（char/word n-gram + BoW）+ 可选稠密 embedding（ConveRT/BERT/GloVe）|
| 主干 = 预训练 LM（110M+ 参数） | 主干 = 2 层轻量 transformer（~300K 参数 + 特征映射） |
| 意图 head = softmax on [CLS] | 意图 head = **[CLS]-token 与 intent embedding 的相似度**（基于对比学习） |
| 实体 head = 每 token softmax | 实体 head = **CRF over transformer 输出**（全局解码） |
| 训练 = 1 个 supervised loss | 训练 = **意图 loss + 实体 CRF loss + 可选 masked token loss** 联合优化 |
| 迭代时间 = 小时级 | 迭代时间 = **分钟级**（6× 加速） |
| 依赖 = 必须预训练 | 依赖 = **可选**——稀疏特征单独即可工作 |

---

## 二、整体架构

```
输入：用户话语 utterance = "book a flight from Boston to Denver tomorrow"

Step 1: Tokenize
  tokens = [__CLS__, "book", "a", "flight", "from", "Boston", "to", "Denver", "tomorrow"]
  （__CLS__ 是加在句首的特殊 token，用来聚合意图表示）

Step 2: 特征提取（每个 token 独立做）
  ┌─ 2a. 稀疏特征（一定启用）
  │    - char n-gram（1-5）: 字符级子串的 one-hot/count 向量
  │    - word n-gram（1-2）: 词级 n-gram
  │    - BoW on lemmatized tokens
  │    → sparse_vec (稀疏高维)
  │
  └─ 2b. 稠密特征（可选）
       - ConveRT / BERT / GloVe / spaCy 向量
       → dense_vec（固定低维）

Step 3: 特征融合
  每个 token 的 sparse_vec + dense_vec → feed-forward → input_vec（统一维度）

Step 4: Transformer 编码
  2 层 transformer（256 hidden, 4 heads）→ 每个 token 一个上下文化表示 h_i

Step 5: 三个任务头并行
  ┌─ 5a. 意图分类头（利用 __CLS__ 位置）
  │    h_CLS → 与每个候选 intent 的 embedding 做点积
  │    → 意图概率分布
  │
  ├─ 5b. 实体识别头（利用除 __CLS__ 外所有 token）
  │    [h_1, h_2, ..., h_n] → CRF 层 → 每个 token 的 BIOES 标签
  │
  └─ 5c. （可选）Masked Token 头
       随机遮盖 15% 输入 token，预测被遮盖位置 → 自监督正则

Step 6: 联合损失
  L = L_intent + L_entity + λ * L_mask

Step 7: 预测时
  return {intent: argmax_CLS, entities: CRF_decode([h_1...h_n])}
```

---

## 三、六个核心组件详解

### Step 2a. 稀疏特征 — 无预训练也能跑的保底信号

**作用**：提供字符级和词级的表面特征，即使没有预训练模型也能让 DIET 工作。

**具体特征类型**：

```
1. Char-level n-gram（n=1..5）
   "Boston" → {'B', 'o', 's', 't', 'n', 'Bo', 'os', 'st', 'to', 'on', 'Bos', ...}
   每个 n-gram 变成 one-hot 的一个维度

2. Word-level n-gram（n=1..2）  
   "book a flight" → {'book', 'a', 'flight', 'book a', 'a flight'}

3. BoW on lemmatized tokens
   "flying" → 'fly' 的 one-hot 维度

4. 内置正则特征（可选）
   \d+ → digit_flag
   [A-Z]\w+ → capital_flag
```

**为什么重要**：char n-gram 对**未登录词（OOV）和拼写错误**有天然鲁棒性——"Bostn"（拼错）仍能通过与"Boston"共享的 char n-gram 被识别为实体。

**示例**：

```
"book a flight to Bsotn"（用户打错）

char n-gram 共享：
  "Boston" 的 char-gram: B, o, s, t, n, Bo, os, st, to, on, ...
  "Bsotn"  的 char-gram: B, s, o, t, n, Bs, so, ot, tn, Bso, ...
  
重叠度高 → 稀疏特征给出相似向量 → 仍能正确识别为 city 实体
（纯词向量模型会把 "Bsotn" 视为 OOV，彻底失败）
```

---

### Step 2b. 稠密特征 — 可选的预训练加成

**作用**：如果预算允许，可以叠加预训练 embedding 获得更强的语义表示。

**支持的来源**：

| Embedding 类型 | 维度 | 训练数据 | 适用场景 |
|---|---|---|---|
| GloVe / word2vec | 300 | 通用语料 | 基线增强 |
| spaCy word vectors | 300 | 通用新闻 | 内置便利 |
| **ConveRT** | 512 | **对话语料**（Reddit） | **论文推荐**，对话场景最优 |
| BERT / RoBERTa | 768/1024 | 通用语料 | 精度上限，但速度慢 |
| Fine-tuned BERT | 768 | 任务微调 | 最强但训练贵 |

**关键洞察**（论文核心发现）：

```
在 NLU-Benchmark 上：
  无 dense features（仅稀疏）    micro-F1  ~87%
  + GloVe                         ~88%
  + ConveRT（对话预训练）         ~89.6%   ← 最优
  + BERT（fine-tune）              ~89.9%   ← 仅高 0.3%
  BERT alone（fine-tune）          ~89.9%
  
但训练时间：
  DIET + ConveRT  = 基准
  BERT fine-tune  = 6× 慢
```

**意义**：ConveRT 在对话场景上比 BERT 更合适（因为训练数据是对话）且更小。DIET 让**小而合适**的预训练模型发挥出大模型的精度。

---

### Step 3. 特征融合 — 对齐稀疏与稠密

**作用**：把高维稀疏向量和低维稠密向量映射到同一空间。

```
对每个 token_i：
  sparse_i  ∈ R^D_s  （D_s 很大，比如 10000+）
  dense_i   ∈ R^D_d  （D_d 小，比如 512）

  concat = [sparse_i; dense_i]
  input_i = FFN(concat) ∈ R^D_t  （D_t = 256，transformer hidden size）

FFN = 单层线性 + dropout + layer_norm
```

**没有稠密特征时**：`input_i = FFN(sparse_i)`，直接稀疏映射。

---

### Step 4. Transformer 编码 — 轻量而非巨型

**配置**：

```
num_layers     = 2        （BERT: 12 或 24）
hidden_size    = 256      （BERT: 768）
num_heads      = 4        （BERT: 12）
attention_dropout = 0.1
relative_positional_encoding = True  （不是绝对位置）
```

**为什么这么小够用**：对话 NLU 的 utterance 通常很短（5-20 个 token），不需要深层长程建模。2 层 transformer 足以建模 token 间的局部关系。

---

### Step 5a. 意图分类头 — 基于相似度的开放分类

**作用**：把意图预测建模为"utterance 表示"和"intent 标签 embedding"的相似度匹配。

**传统做法**：
```
h_CLS → linear(|intents|) → softmax → P(intent)
问题：添加新意图需要重训输出层
```

**DIET 做法**（StarSpace / StarSpace-inspired）：
```
h_CLS → utterance_embedding u  ∈ R^d

每个候选 intent：
  intent_label → intent_embedding e_i  ∈ R^d  （可学习矩阵）

similarity:
  sim_i = u · e_i  （点积）

loss（对比式）：
  L_intent = max(0, margin - sim_positive + sim_negative)
  采样 K 个负样本（其他 intent label）
```

**优势**：

1. **新意图可动态添加** — 只需新增一行 embedding，不必改模型
2. **少样本学习友好** — 对比损失对类别不平衡更鲁棒
3. **可解释** — embedding 可视化能看出语义相近的 intent

**示例**：

```
utterance: "I want to book a flight"
  u = [0.1, 0.8, -0.2, 0.5, ...]

intent embeddings:
  book_flight: [0.12, 0.78, -0.15, 0.52, ...]   sim = 0.96 ← 最高
  cancel_flight: [0.08, 0.45, 0.3, -0.1, ...]   sim = 0.31
  weather_query: [-0.5, 0.2, 0.8, -0.4, ...]    sim = -0.42

预测 = book_flight
```

---

### Step 5b. 实体识别头 — CRF 全局解码

**作用**：把每个 token 分类为 BIOES 实体标签，但用 CRF 做**全序列联合解码**（避免"B-city 后跟 I-person"这种非法序列）。

**架构**：

```
transformer 输出：[h_1, h_2, ..., h_n]  （不含 __CLS__）

每个 h_i → linear(|entity_tags|) → emission_logits_i  （每个标签的分数）

CRF 层：
  transition_matrix T[|tags|, |tags|]  （可学习）
  对序列 y = [y_1, ..., y_n]：
    score(y) = Σ emission_i[y_i] + Σ T[y_{i-1}, y_i]

训练 loss：
  L_entity = -log P(y* | x) = -score(y*) + log Σ_y exp(score(y))
  （用 forward 算法算归一化项）

推理：
  y* = argmax Viterbi decode
```

**BIOES 标注示例**：

```
"book a flight from Boston to Denver"

tokens:   book  a  flight  from  Boston  to  Denver
BIOES:    O     O  O       O     S-city  O   S-city

S = Single（单 token 实体）
B = Begin, I = Inside, E = End（多 token 实体）
O = Outside（非实体）
```

---

### Step 5c. Masked Token 头 — 自监督正则（可选）

**作用**：借鉴 BERT 的 MLM 任务，在训练中随机遮盖输入 token 让模型恢复，作为**辅助正则**。

**流程**：

```
训练时，对 input_i 随机以 15% 概率替换：
  80% → 替换为 [MASK] 向量
  10% → 替换为随机 token 向量
  10% → 保持不变

transformer 编码后，被遮盖位置的 h_i → 预测原始 token 的稀疏特征

L_mask = CE(predicted_sparse, original_sparse)
```

**关键点**：预测目标是**稀疏特征**（char+word n-gram），不是 token id。这样在开放词表场景下仍可用。

**消融结果**（论文 Table 3）：

```
任务                     无 mask     有 mask
NLU-Benchmark (intent)   89.2%      89.5%   (+0.3%)
NLU-Benchmark (entity)   86.0%      86.6%   (+0.6%)
```

结论：mask loss 带来小但一致的提升，作为**可选**的正则手段。

---

### Step 6. 联合损失 — 共享编码器的多任务优化

```
L_total = L_intent + L_entity + λ_mask × L_mask

λ_mask 默认 0.7（可调），intent 和 entity 权重均为 1
```

**为什么多任务有效**：

- 意图分类学到的 utterance 级语义 → 帮助实体 head 区分"相似词在不同意图下指向不同实体类型"
- 实体识别学到的 span 结构 → 帮助意图 head 聚焦关键槽位词
- mask loss 提供的自监督信号 → 让表示更平滑，对标注噪声更鲁棒

---

## 四、代码结构

```
rasa/nlu/classifiers/diet_classifier.py       # 主分类器类
rasa/nlu/featurizers/
  ├── sparse_featurizer/
  │   ├── count_vectors_featurizer.py         # BoW / word n-gram
  │   ├── lexical_syntactic_featurizer.py     # char n-gram
  │   └── regex_featurizer.py                 # 正则特征
  └── dense_featurizer/
      ├── convert_featurizer.py               # ConveRT
      ├── lm_featurizer.py                    # BERT/RoBERTa
      └── spacy_featurizer.py                 # spaCy

rasa/utils/tensorflow/
  ├── models.py                               # TransformerRasaModel 基类
  ├── transformer.py                          # Transformer 层
  ├── layers.py                               # Dense layer + embedding + CRF
  └── model_data.py                           # 特征 pipeline

核心 forward 函数位于：
  diet_classifier.DIET._train_losses_scores()
  - 输入：batch features (sparse + dense)
  - 过程：feature-combine → transformer → 三个 head
  - 输出：losses dict
```

---

## 五、三种使用方式

### 方式 1: 仅稀疏特征（零预训练依赖，快速原型）

```yaml
# config.yml
pipeline:
  - name: WhitespaceTokenizer
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: CountVectorsFeaturizer
    analyzer: char_wb
    min_ngram: 1
    max_ngram: 4
  - name: DIETClassifier
    epochs: 100
```

**适用**：冷启动项目、标注数据少、需要快速实验

---

### 方式 2: 稀疏 + ConveRT（论文推荐，对话场景最优）

```yaml
pipeline:
  - name: ConveRTTokenizer
  - name: ConveRTFeaturizer
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: CountVectorsFeaturizer
    analyzer: char_wb
    min_ngram: 1
    max_ngram: 4
  - name: DIETClassifier
    epochs: 300
    use_masked_language_model: true
```

**适用**：英语对话助手，要精度也要训练速度

---

### 方式 3: 稀疏 + BERT（精度上限，成本高）

```yaml
pipeline:
  - name: LanguageModelTokenizer
    model_name: "bert"
  - name: LanguageModelFeaturizer
    model_name: "bert"
    model_weights: "rasa/LaBSE"
  - name: LexicalSyntacticFeaturizer
  - name: CountVectorsFeaturizer
  - name: DIETClassifier
    epochs: 300
```

**适用**：多语种、对精度极致追求、可接受更长训练时间

---

## 六、关键超参数

| 参数 | 默认值 | 含义 |
|---|---|---|
| `epochs` | 300 | 训练轮数 |
| `batch_size` | [64, 256] | 动态批次大小（从 64 线性增到 256） |
| `hidden_layer_sizes` | `{text: [256, 128]}` | 特征映射 FFN 维度 |
| `number_of_transformer_layers` | 2 | Transformer 层数 |
| `transformer_size` | 256 | 隐藏维度 |
| `num_heads` | 4 | 注意力头数 |
| `use_masked_language_model` | False | 是否启用 mask loss |
| `BILOU_flag` | True | 是否用 BIOES 标注 |
| `entity_recognition` | True | 是否开启实体 head |
| `intent_classification` | True | 是否开启意图 head |
| `embedding_dimension` | 20 | intent/label embedding 维度 |
| `num_neg` | 20 | 对比损失负采样数 |
| `similarity_type` | "auto" | inner / cosine |
| `loss_type` | "cross_entropy" | cross_entropy / margin |
| `drop_rate` | 0.2 | Dropout |
| `constrain_similarities` | True | 相似度归一化 |

---

## 七、实验结果

### NLU-Benchmark (micro-F1)

| 模型 | Intent | Entity | 训练时间 |
|---|---|---|---|
| Mrkl (LSTM baseline) | 84.1% | 72.8% | — |
| DIET（仅稀疏） | 87.4% | 84.5% | 基准 |
| DIET + GloVe | 88.0% | 85.1% | +10% |
| DIET + ConveRT | **89.6%** | **86.6%** | +15% |
| DIET + BERT | 89.9% | 86.0% | 6× 基准 |
| BERT fine-tune | 89.9% | — | 6× 基准 |

**关键观察**：
- DIET + ConveRT **达到 BERT fine-tune 级精度，同时速度快 6×**
- 仅稀疏特征版本（87.4%）已经**强于 LSTM 基线 3 个点**

---

### ATIS & SNIPS（两个标准 NLU 基准）

| 模型 | ATIS Intent | ATIS Slot F1 | SNIPS Intent | SNIPS Slot F1 |
|---|---|---|---|---|
| Joint BERT | 97.9 | 96.0 | 98.6 | 97.0 |
| **DIET + BERT** | **97.7** | **96.2** | **98.8** | **96.8** |

DIET 在两个经典数据集上与 Joint BERT 基本打平，**证明轻量架构不损失精度**。

---

### 消融实验（NLU-Benchmark）

| 配置 | Intent | Entity |
|---|---|---|
| 完整 DIET | 89.6 | 86.6 |
| −mask loss | 89.2 | 86.0 |
| −entity head（纯意图）| 89.0 | — |
| −intent head（纯实体）| — | 85.3 |
| −transformer（直连 FFN）| 86.5 | 82.4 |

**结论**：
- 多任务联合训练比单任务好 0.6-1.3 个点
- Transformer 是精度关键（−3% 如果移除）
- mask loss 贡献小但一致

---

## 八、局限性

1. **输入长度受限** — 2 层 transformer 对长 utterance（>50 token）建模能力有限；对话 NLU 场景不是问题，但不适合文档级任务
2. **依赖标注数据** — 意图和实体都是监督学习，冷启动需要每意图 10-20 样本
3. **稠密特征预处理耦合** — 使用 BERT 时需要对应的 tokenizer，pipeline 配置复杂
4. **单语言部署** — 每个语言要训练独立模型（除非用 LaBSE 等多语 embedding）
5. **不做对话历史建模** — 纯句子级 NLU，多轮依赖由上层对话管理器处理

---

## 九、与相关工作的定位

```
                           NLU 架构
                              │
           ┌──────────────────┼──────────────────┐
           独立任务                         联合任务
           │                                │
      ┌────┼────┐                      ┌────┼────┐
      CRF  LSTM  BERT slot           Joint    DIET
      slot fill classifier           BERT     │
                                              │
                                    ┌─────────┼──────────┐
                                    轻量      折中        重量
                                    │        │           │
                                仅稀疏   + ConveRT    + BERT
                                (基线)   (论文推荐)  (精度上限)
```

**与 Joint BERT 对比**：
- Joint BERT 把 intent 和 slot 共享 BERT 编码器，DIET 同理但**编码器是自训练的轻量 transformer**
- DIET 支持**无预训练模型**的场景，Joint BERT 强依赖 BERT

**与 Rasa 老方案（MITIE/CRF + sklearn）对比**：
- 老方案是两个独立模型（意图 + 实体），不共享表示
- DIET 统一到一个模型，参数更少、一致性更好

---

## 十、核心贡献与影响

1. **证伪"必须用 BERT"** — 通过 ConveRT 实验证明，在对话 NLU 上**小而合适**的预训练模型足以达到 SOTA
2. **多任务联合的工程可行性** — 意图 + 实体 + mask 三任务共享 transformer 在生产环境稳定可用
3. **模块化 Pipeline 范式** — 稀疏 / 稠密特征可插拔，适配不同预算和场景
4. **落地影响** — 成为 Rasa Open Source 默认 NLU 组件，驱动大量生产级对话助手

---

Sources:
- 论文: https://arxiv.org/abs/2004.09936
- 代码: https://github.com/RasaHQ/rasa（`rasa/nlu/classifiers/diet_classifier.py`）
- Rasa 官方博客: https://rasa.com/blog/introducing-dual-intent-and-entity-transformer-diet-state-of-the-art-performance-on-a-lightweight-architecture/
- ConveRT (对话 embedding): https://arxiv.org/abs/1911.03688
- NLU-Benchmark 数据集: https://github.com/xliuhw/NLU-Evaluation-Data
