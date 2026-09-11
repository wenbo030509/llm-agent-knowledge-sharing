# 专题拆解：ReWatch 视频推理数据构造 Pipeline（可复用蓝图）

> 来源：ReWatch-R1（Alibaba，arXiv:2509.23652v1）§2 + 附录 A/D ｜ 拆解日期：2026-09-11
> 定位：把原文数据合成管线**单独抽出**，做成"我 vlm-agent 靶向数据 pipeline"可直接对标/借用的工程蓝图
> 配套：完整精读见同目录 [`笔记.md`](./笔记.md)（含结果/消融/超参）；本篇只讲**怎么造数据**

---

## 0. 一句话总览

**先把视频转成"带绝对时间戳的结构化文本底座（caption）"，之后出题、造 CoT、算奖励全部在这个文本底座上进行**——用文本操作替代反复解码视频，既降本又可控。三阶段：

```text
Stage 1 分层 Captioning ──→ Stage 2 对比出题+三层过滤 ──→ Stage 3 多智能体 ReAct 造 CoT
   (10,989 视频)              (170k QA，过滤后 85k→改写多选)      (135k CoT)
        │                            │                                │
   带时间戳事件流 C_detail      反捷径、强视频依赖的 QA          可追溯"先定位→取证→推理"的 CoT
```

核心设计哲学（3 条，最值得偷师）：
1. **文本底座先行**：一次性把视频"翻译"成带时间戳的事件流，后续所有环节复用它 → 极致降本。
2. **成本/质量分层**：便宜模型做粗活（分段、摘要、转写），贵模型做关键判断（细描述、出题、验证）。
3. **反捷径是第一原则**：不是事后质检，而是把"必须依赖视频"做成出题阶段的**硬过滤 + 可量化验收**。

---

## 1. Stage 1：分层动态帧率 Captioning（造"文本底座"）

**目标**：把长视频转成高保真、带绝对时间戳的事件描述，避免长视频直接喂模型的幻觉。

```text
输入：视频 V（来自 5 个公开源）
 │
 ├─[1] 语义分段  M_seg（低帧率、便宜模型）
 │      把 V 切成 k 个语义连贯片段 S={s_1..s_k}，每段带 [t_start, t_end]，保事件完整
 │
 ├─[2] 细粒度描述  M_cap（高帧率、强模型）
 │      对每段 s_i 生成事件描述 + 相对时间戳 {(c_ij, τ_ij)}
 │
 └─[3] 时间戳realign  t_ij = t_i_start + τ_ij（相对→绝对）

输出：C_detail(V) = 全片带绝对时间戳的事件流
```

**用的模型**：M_seg = M_cap = Gemini-2.5-Flash（non-thinking）。
**caption prompt（Fig 17）关键约束**：按剧情分段、每段 `[MM:SS-MM:SS]`+描述、**时间戳必须连续覆盖全片**、聚焦动作/关键物体/交互。

★ 对我项目的映射：这一步 = 我"抽帧判断前置"的加强版。我是**抽帧判断要不要抓**，它是**先把整片转成结构化文本再复用**。我的 pipeline 可以加一层"视频→带时间戳事件流"的入库产物，让出题和质检都基于它。

---

## 2. Stage 2：对比出题 + 三层过滤（反捷径核心）★★

**目标**：造"必须看视频、且看细节才能答"的高难 QA，从源头掐掉语言捷径和粗粒度捷径。

### 2.1 对比式出题（Contrastive Prompting）

```text
C_detail ──M_sum──→ C_sum（精简摘要，便宜模型）
                       │
C_detail + C_sum ──M_qa──→ (Q,A)_raw
   目标：造出"能从 detailed caption 答、但从 summary 答不出"的题
   → 用"详细 vs 摘要"的信息差，逼出细粒度、非概览性问题
   10 种题型保多样性（见 §4）
```

### 2.2 三层过滤级联（Three-Layer Filtering）

```text
F1 答案验证       M_verify 核对 A 能否被 C_detail 支持     → 去错标
   通过条件：M_verify(Q,A,C_detail)=True

F2 文本偏置消除   一组 LLM 不看视频裸答，命中率 < θ_text   → 去语言先验捷径
   通过条件：mean(命中) < θ_text

F3 摘要偏置消除   一组 LLM 只看摘要作答，命中率 < θ_sum    → 逼细粒度视频依赖
   通过条件：mean(命中) < θ_sum

过滤后 85k 题 ──M_rewrite──→ 改写成多选，扩到 170k QA
```

**真实参数**：
- **θ_text = θ_sum = 1**（严格版：探测模型**全部**能裸答/只看摘要答对，才判"有捷径"淘汰）。
- 探测模型组 M_probe = **Qwen3-235B-A22B-Instruct + Qwen2.5-VL-72B-Instruct**。
- M_sum = Gemini-2.5-Flash-Lite；M_qa = Gemini-2.5-Flash(thinking)；M_verify = GPT-4.1；M_rewrite = Gemini-2.5-Flash。

### 2.3 反捷径的量化铁证（这就是"必须依赖视觉"的验收指标）★★

```text
指标                ReWatch-QA   Video-R1-QA   随机基线
Text-Only 准确率      29.4%        68.9%        25%
推理步数              3.31         1.82          -
回答长度              398.75       205.74        -
```

**Video-R1-QA 有 68.9% 的题不看视频就能答对；ReWatch-QA 只有 29.4%（逼近 25% 随机）**——三层过滤真的把文本捷径挤没了。

★ 对我项目的映射：我"必须依赖视觉"过去靠 judge+人工定性；可直接借这套**可量化验收**——
   拿一组纯文本 LLM 去答我的题，**Text-Only 准确率越接近随机越好**；对比式出题（详细能答/摘要答不出）也可搬进我的出题环节，主动造难，前移到源头治难度坍缩。

---

## 3. Stage 3：多智能体 ReAct 造 CoT（把"回看视频"外化成可追溯轨迹）★★

**目标**：造"视觉锚定、可追溯"的推理链，教模型"先定位→再取证→后推理"的框架，而非答案。

```text
输入：问题 Q、文本底座 C_detail
两个 agent 在 C_detail 上循环（模拟人回看视频找证据）：

  Reasoner A_R：产出 thought T_t + action Act_t
        │
  Observer A_O：在 C_detail 上执行 Act_t，返回观察 Obs_t
        │
  循环直到 Reasoner 给出 final answer

两个核心动作（模拟视觉查找）：
  segment_retrieval(query)   按语义找事件的时间戳（模拟"找片段在哪"）
  segment_query(timestamp)   按时间戳取事件细节（模拟"定格看细节"）

输出：轨迹 T ──M_convert──→ 带 <action>/<observation>/<answer> 标签的自然语言 CoT
```

**用的模型**：Reasoner = Gemini-2.5-Flash(thinking)；Observer = GPT-4.1；M_convert = Gemini-2.5-Flash-Lite。
**为什么全程在文本上跑**：Observer 查的是 C_detail 而非重新解码视频 → **成本极低**，这是"合成"能规模化的关键。

★ 对我项目的映射：这就是我"多视频 CoT = 逐视频证据+对应关系+汇总结论"结构的**动作序列化版本**。
   把"证据段落"写成 `segment_retrieval / segment_query` 两个可执行动作，CoT 就天然可追溯、可校验，也天然为后续 RL 的过程奖励备好了结构。

---

## 4. 交付数据规格（Table 2，可当我数据 schema 的对标）

```text
总视频 10,989（MiraData 15.9% / VideoEspresso 18.0% / VideoMarathon 30.0%
              / Video-R1 18.0% / Vript 18.1%）
时长：短(<3min) 3970 / 中(3-20min) 5472 / 长(20-60min) 1547
QA 170,862：多选 50.2% / 开放 49.8%
CoT 135,346：平均推理步数 2.3（最多11）、平均推理 token 332.5（最多2045）

10 种题型（Table 4，= 能力维度定义，可直接对标我的 L1/L2/L3）：
  事件定位 12.4% / 时间定位 10.4% / 计数 11.0% / 因果 9.5% / 阅读OCR 8.5%
  / 空间感知 9.6% / 物体识别 10.7% / 状态变化 8.9% / 数值推理 11.3% / 反事实推理 7.8%
```

题型定义要点（Table 4，挑几个和我 L1/L2/L3 对应的）：
- 事件定位：输出特定事件的精确起止时间（≈ 我 L1 时序感知）
- 状态变化：识别对象属性/位置/行为/情绪的时序变化（≈ 我 L2 状态对比）
- 因果：识别事件间直接因果，一个事件直接导致另一个（≈ 我 L3 推理）
- 反事实推理：假设某事件没发生/以不同方式发生，推断可验证后果（≈ 我 L3 最难档）

---

## 5. 三阶段"造什么模型分工"总表（成本/质量分层一览）

```text
阶段        角色/子步骤         模型                          think?
Stage1     语义分段 M_seg      Gemini-2.5-Flash              non-think
Stage1     细描述 M_cap        Gemini-2.5-Flash              non-think
Stage2     摘要 M_sum          Gemini-2.5-Flash-Lite         non-think
Stage2     出题 M_qa           Gemini-2.5-Flash              thinking
Stage2     答案验证 M_verify   GPT-4.1                       -
Stage2     探测 M_probe        Qwen3-235B-A22B + Qwen2.5-VL-72B  -
Stage2     改写多选 M_rewrite  Gemini-2.5-Flash              non-think
Stage3     Reasoner A_R        Gemini-2.5-Flash              thinking
Stage3     Observer A_O        GPT-4.1                       -
Stage3     转写 M_convert      Gemini-2.5-Flash-Lite         non-think
```

规律：**粗活/格式活用 Flash-Lite/Flash（便宜），关键判断（细描述、出题、验证、取证）用 thinking 或 GPT-4.1（贵）**。

---

## 6. 直接搬进我 vlm-agent pipeline 的 5 个动作项

```text
① 增加"文本底座"产物：视频入库时同时产出带绝对时间戳的事件流 caption，
   让出题/质检/CoT 都基于它，避免反复解码视频（对标 Stage1）。
② 出题改成对比式：先生成详细caption+摘要，造"详细能答/摘要答不出"的题（对标 Stage2.1）。
③ 反捷径做成硬过滤 + 量化验收：一组纯文本LLM裸答，Text-Only 准确率越近随机越好；
   阈值可先宽松（非严格=1），逐步收紧（对标 Stage2.2/2.3）。
④ CoT 动作序列化：把"逐视频证据"写成 segment_retrieval/segment_query 两动作，
   CoT 天然可追溯，也为后续 RL 过程奖励备好结构（对标 Stage3）。
⑤ 数据 schema 加题型维度：像 Table 4 那样给每条数据打 10 类考点标签，
   对齐我的 L1/L2/L3，让覆盖率可统计、可归因（对标 §4）。
```

---

## 7. 边界与未公开细节（诚实标注）

- Stage2 的 **θ 值原文取 1（严格）**，未给敏感性分析（换 0.6/0.8 会怎样未知）。
- Stage3 CoT 的**证据忠实自动判分（r_obs）prompt 未公开**（附录 D 只有答案一致性判分 Fig 18）——
  说明"观察是否被证据支持"的自动判分他们也没标准化，是我可深耕的差异点。
- 全流程是**文本底座上的合成**：优点是降本可控，隐患是**caption 本身的错误/遗漏会向下游传导**（原文 Stage1 用分层动态帧率来压这个风险，但未给 caption 质量的独立评测数字）。
