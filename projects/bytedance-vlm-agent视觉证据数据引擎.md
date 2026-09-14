# Benchmark 驱动的 VLM-Agent 视觉证据数据引擎

> Benchmark-driven Visual Evidence Data Engine for VLM-Agent
>
> 项目类型：VLM-Agent 训练数据构造与能力闭环
>
> 项目周期：2026.08 - 至今
>
> 文档定位：基于真实项目事实，借鉴 Alibaba ReWatch-R1 的数据构造方法，对原 vlm-agent 项目进行系统化重构
>
> 方法参考：[ReWatch-R1 精读](../papers/15.alibaba-vlm-rewatch-training-pipeline/笔记.md) / [数据构造 Pipeline 精析](../papers/15.alibaba-vlm-rewatch-training-pipeline/数据构造pipeline精析.md)

---

## 零、事实边界

本文同时包含两类内容，必须严格区分：

| 类型 | 含义 | 使用方式 |
|------|------|----------|
| **已发生事实** | 原项目真实的需求、数据交付、Pipeline、训练方式和结果口径 | 可作为项目经历直接陈述 |
| **重构方案** | 用 ReWatch-R1 方法对原流程进行标准化、补全和升级 | 应表述为“复盘后形成的方法”或“下一轮方案”，不能倒推成当时已经完整实施 |

已确认事实：

```text
1. 项目从 ClawEval / WildClawBench 等 Agent 评测的失分问题出发。
2. 首轮聚焦视觉理解、跨片段对齐和推理等视觉原子能力。
3. 数据交付不是简单 VQA，而是包含视频素材、caption、时间戳证据、
   question/instruction、answer、CoT 和能力标签的复合训练数据。
4. 约 74 个 topic、约 400 条素材；内部 161 题中覆盖 72 题，约 45%。
5. 算法侧采用 SFT；内部覆盖子集 +2.8，ClawEval Overall +0.5，
   WildClawBench +2.3。
6. Agent 基准收益来自共享基座上的多条数据线，尚无完整消融，
   不能把 +0.5 / +2.3 全部归因给本项目。
```

借鉴 ReWatch-R1 后新增的标准化设计：

```text
1. 把带绝对时间戳的 caption 明确为共享证据底座。
2. 用 detailed caption 与 summary 的信息差做对比式出题。
3. 用答案支持、Text-Only、Summary-Only、跨视频必要性做级联过滤。
4. 把 CoT 改造成“定位 → 取证 → 对齐 → 推理 → 回答”的可追溯动作序列。
5. 用组件消融和能力分桶评测，验证 caption、过滤器和 CoT 各自的增益。
```

---

## 一、项目四段式

### 1. 项目名称

**Benchmark 驱动的 VLM-Agent 视觉证据数据引擎**

副标题：

> 基于时间戳 Caption、反捷径任务构造与证据锚定 CoT 的靶向能力提升

这个名称比“vlm-agent 数据项目”更准确，因为项目的核心不是泛化地做 Agent，也不是批量生产简单 VQA，而是：

```text
Agent Benchmark 失分
→ 定位视觉原子能力缺口
→ 构造可追溯的视觉证据数据
→ 通过 SFT 改善模型
→ 回到能力评测和 Agent 任务验证收益
```

### 2. 项目背景

ClawEval 和 WildClawBench 中包含需要读取视频、识别画面状态、跨片段查找证据并基于视觉信息完成后续任务的 Agent 场景。对失分任务复盘后发现，一部分失败不是 Agent 框架不会规划或调用工具，而是更早发生在视觉观察阶段：

```text
关键片段没有被定位
→ 物体、动作、文字或状态识别错误
→ 跨片段关系没有建立
→ 后续规划建立在错误观察上
→ Agent 端到端任务失败
```

原项目的早期叙事和部分通用视觉数据存在四个问题。这里的问题不是“没有交付 Caption”，而是没有把 Caption 明确为贯穿全链路的共享证据资产：

1. **证据关系不显式**：Caption、问题、答案和 CoT 虽然存在，但缺少统一的证据 ID 和依赖关系。
2. **任务过于简单**：不少问题依赖语言常识或视频摘要即可回答，不能形成强视觉训练信号。
3. **CoT 不可追溯**：推理过程没有明确引用视频、片段和时间点，容易夹带幻觉。
4. **评测与数据脱节**：Benchmark 只给总分，无法说明哪类数据改善了哪项能力。

因此需要建设一套从 Benchmark 缺口出发、以视觉证据链为核心的数据构造与验证系统。

### 3. 项目动作

项目动作可以概括为七步：

```text
① Benchmark badcase 归因
   定位首次错误环节，区分视觉输入、感知、对齐、推理、Agent 执行和 Judge 问题。

② 能力与难度建模
   建立 L1 感知 / L2 对齐比较 / L3 多证据推理的能力体系，
   同时记录时长、证据跨度、干扰量、实体数和推理步数。

③ 靶向寻源与资产化
   从能力缺口反推关键词和素材关系，完成抽帧预检、抓取、TOS 入库、
   去重和元数据登记。

④ 构造时间戳 Caption 底座
   先做语义分段，再生成片段级细描述，并统一为绝对时间戳事件流。

⑤ 生成并过滤任务
   基于 detailed caption 与 summary 的信息差出题，再通过答案支持、
   Text-Only、Summary-Only 和跨视频必要性过滤，去除错标与捷径题。

⑥ 生成证据锚定 CoT
   把推理写成“定位片段 → 查询细节 → 跨片段/跨视频对齐 → 推理 → 回答”，
   每个关键观察都能回指 video_id 和 timestamp。

⑦ 质检、SFT 与分层复测
   人工终审后交付算法训练；按能力、覆盖、难度和任务类型对比训练前后结果，
   再把剩余 badcase 回流到下一轮数据策略。
```

### 4. 项目结果

#### 已确认结果

```text
数据资产：
├── 74 个 topic
├── 约 400 条视频素材
├── 数据形态包含 caption、时间戳证据、question/instruction、
│   answer、CoT、能力与难度标签
└── 内部 161 题中靶向覆盖 72 题，覆盖率约 45%

模型结果：
├── 内部 72 题覆盖子集：+2.8
├── ClawEval Overall：+0.5
└── WildClawBench：+2.3

工程结果：
├── 已跑通“能力驱动寻源 → 抽帧预检 → 抓取/TOS → 去重
│   → Prompt/CoT 候选 → 作业表 → 人工终审”的半自动主链路
└── 本次重构进一步把时间戳 Caption、级联过滤和证据绑定
    定义为主链路中的标准节点，自动化程度待下一轮实施验证
```

#### 结果的正确解释

- `+2.8` 是内部 161 题中 72 题覆盖子集的变化，是靶向方向有效的直接证据。
- `+0.5` 是 ClawEval 全量 Agent 指标，包含大量与视觉无关的任务，不能等价为视觉能力提升幅度。
- `+2.3` 是 WildClawBench 的端到端结果，说明共享基座整体在真实长链路任务上改善，但尚不能独立归因给本项目。
- 首轮最重要的策略结论是：**数据方向有效，但覆盖率只有约 45%，下一轮的主要瓶颈是能力覆盖，而不是继续堆同类样本。**

---

## 二、项目核心定义

### 2.1 项目不是简单 VQA

本项目交付的基本单位不是：

```text
video + question + answer
```

而是一份围绕同一视觉证据源组织的复合数据包：

```text
Visual Evidence Package
├── 原始视频或多视频组合
├── 视频元数据与来源信息
├── 带绝对时间戳的 detailed caption
├── 全局 summary
├── question / task instruction
├── reference answer
├── evidence spans
├── 证据锚定 CoT
├── 能力标签与难度标签
├── 质量 Gate 结果
└── 数据版本与追踪字段
```

其中，**timestamped caption 是共享证据底座**：

```text
Caption 支撑出题
Caption 校验答案
Caption 为 CoT 提供证据
Caption 为 Judge 提供核验上下文
Caption 为后续检索动作提供索引
```

这使 QA、CoT 和质检不再各自生成、各自判断，而是围绕同一份证据对齐。

### 2.2 项目目标

项目有三个目标：

| 目标 | 要解决的问题 | 验收方式 |
|------|--------------|----------|
| O1 视觉能力提升 | 模型看不准、找不到、对不齐、推不出 | 内部能力集按 L1/L2/L3 分桶变化 |
| O2 Agent 任务迁移 | 视觉子步骤改善能否带动任务完成 | ClawEval 多模态子项、WildClawBench |
| O3 数据供给效率 | 如何稳定生产强视觉依赖、可训练的数据 | 有效样本率、人时、覆盖率、各 Gate 淘汰率 |

---

## 三、能力模型：从 Agent 失败映射到数据缺口

### 3.1 首次错误定位

Agent 任务最终失败可能由多个环节造成。不能看到最终答案错误，就直接归因为视觉模型能力不足。

```text
输入层：关键帧是否进入上下文？
  ↓
感知层：对象、动作、OCR、状态是否识别正确？
  ↓
对齐层：跨时间、跨片段、跨视频关系是否建立？
  ↓
推理层：是否从正确观察推出正确结论？
  ↓
执行层：规划、工具、参数和状态反馈是否正确？
  ↓
评测层：Judge 是否正确判定结果？
```

判定方法：

- 回放 badcase 完整轨迹，定位第一次偏离正确路径的位置。
- 检查关键证据是否进入模型输入。
- 用人工补充关键帧或 Oracle caption 做对照。
- 如果补充正确视觉证据后任务恢复，视觉链路是主要瓶颈。
- 如果视觉事实正确但动作仍错，则应转向规划或工具调用数据。

### 3.2 能力 Taxonomy

| 层级 | 能力 | 典型任务 | 典型错误 |
|------|------|----------|----------|
| L1 感知与定位 | 物体、动作、OCR、计数、事件/时间定位 | 找到某动作发生时间，读取画面文字 | 漏看、看错、时间点偏移 |
| L2 对齐与比较 | 实体匹配、状态变化、步骤对齐、跨视频比较 | 比较两个片段中的状态或做法 | 对错实体、关系错配 |
| L3 多证据推理 | 因果、数值、迁移、反事实、综合判断 | 融合多个片段证据后得出结论 | 跳步、因果倒置、证据不足 |

### 3.3 难度模型

难度不能只用题型描述，必须落到可记录字段：

```text
video_duration          视频时长
evidence_span           首尾证据时间跨度
evidence_count          必要证据数量
entity_count            相关实体数量
distractor_level        干扰信息量
cross_video_dependency  是否必须跨视频
reasoning_steps         最少推理步数
ocr_density             文字密度
visual_ambiguity        遮挡、视角、分辨率等视觉歧义
```

这样可以识别“难度坍缩”：示例要求一小时视频中的跨时段检索，实际数据却变成十秒片段内的直接识别；答案虽然正确，但训练信号已经变弱。

---

## 四、数据构造 Pipeline

### 4.1 总体流程

```text
Stage 0  Benchmark 归因与数据规格
    ↓
Stage 1  靶向寻源、抽帧预检、抓取、入库与去重
    ↓
Stage 2  分层时间戳 Captioning，形成共享证据底座
    ↓
Stage 3  对比式任务生成与四层过滤
    ↓
Stage 4  多 Agent / 多角色生成证据锚定 CoT
    ↓
Stage 5  自动 Quality Gate + 人工终审
    ↓
Stage 6  数据打包、版本化、交付 SFT
    ↓
Stage 7  分层评测、消融、badcase 回流
```

ReWatch-R1 的三阶段方法适合作为核心骨架，但本项目需要增加两个适配：

1. 在前面增加 Benchmark 能力归因和靶向寻源，因为本项目不是从通用公开视频池随机合成。
2. 对多视频任务增加 `video_id + timestamp` 双重证据锚点和“跨视频必要性过滤”。

### 4.2 Stage 0：Benchmark 归因与数据规格

输入：

```text
Benchmark task
+ base model trajectory
+ ground truth / task verifier
+ screenshot/video/tool observations
```

处理：

1. 标记任务是否依赖视觉。
2. 定位首次错误环节。
3. 为错误绑定能力标签和难度标签。
4. 判断应补理解数据、Grounding 数据还是 Agent 轨迹数据。
5. 形成按优先级排序的数据缺口清单。

输出示例：

```text
gap_id: GAP-L2-STATE-017
target_skill: L2.state_change
required_evidence: two_video_segments
difficulty:
  duration: medium
  distractor_level: high
  evidence_span: long
target_volume: 20
acceptance:
  - answer_supported_by_evidence
  - text_only_unsolvable
  - cross_video_required
```

### 4.3 Stage 1：能力驱动寻源与资产化

```text
能力缺口
→ 场景描述
→ AI 扩展检索词
→ 平台检索
→ 候选 URL 写表
→ 远程抽帧/低成本预览
→ Rubric 预检
→ 合格后抓取
→ TOS 入库
→ 去重和元数据登记
```

关键设计：

- **检索与抓取分离**：先低成本判断内容是否符合能力和难度要求，再下载完整视频。
- **关系优先**：多视频任务不是找两个“好视频”，而是找能建立同实体、同过程、不同状态或不同环境关系的视频组合。
- **难度前置**：时长、干扰、证据跨度在寻源阶段验收，不把问题留到标注末端。
- **评测隔离**：素材、任务模板和语义近邻都需要与测试集做隔离。

### 4.4 Stage 2：分层时间戳 Captioning

目标：把视频转成高保真、可定位、可复用的结构化文本证据。

```text
视频 V
  │
  ├── 低帧率语义分段
  │   输出 segment_id / start / end / event boundary
  │
  ├── 高帧率片段描述
  │   输出对象、动作、状态、OCR、交互和相对时间点
  │
  └── 时间戳统一
      relative timestamp + segment start
      → absolute timestamp

输出：C_detail(V)
```

单视频事件：

```text
[00:31-00:38] 男子将红色箱子从桌面移动到地面。
```

多视频事件：

```text
[video_A | 00:31-00:38] 红色箱子由桌面移动到地面。
[video_B | 01:12-01:19] 同类箱子始终位于桌面，未发生移动。
```

Caption 质量 Gate：

- 时间戳连续性和合法性；
- 关键事件覆盖率；
- OCR 与物体描述准确性；
- 不添加画面外因果或身份信息；
- 多视频中的实体命名保持一致；
- 关键 caption 片段回看原视频抽检。

注意：Caption 是整个 Pipeline 的单点风险。Caption 漏证据或写错，错误会传递到问题、答案和 CoT，因此不能只验证最终 QA。

### 4.5 Stage 3：对比式任务生成与四层过滤

先从 `C_detail` 生成压缩摘要 `C_sum`：

```text
C_detail → C_sum
C_detail + C_sum → 生成任务 Q 与答案 A
```

任务生成约束：

> 问题必须能由 detailed caption 中的一个或多个证据回答，但不能只凭 summary、语言常识或单个无关片段回答。

四层过滤：

| Gate | 判定 | 拦截风险 |
|------|------|----------|
| F1 答案支持 | `A` 是否被 `C_detail` 明确支持 | 错标、证据幻觉 |
| F2 Text-Only | 不看视频与 caption 能否回答 | 语言先验捷径 |
| F3 Summary-Only | 只看摘要能否回答 | 粗粒度捷径 |
| F4 Cross-Evidence | 去掉任一关键片段/视频后能否回答 | 伪跨片段、伪多视频 |

F4 是本项目相对 ReWatch 的重要适配：

```text
完整证据可答
+ 去掉 video_A 不可答
+ 去掉 video_B 不可答
= 真正依赖跨视频关系
```

任务类型建议与 L1/L2/L3 对齐：

```text
L1：物体识别 / OCR / 计数 / 事件定位 / 时间定位
L2：实体匹配 / 状态变化 / 时间对齐 / 步骤比较
L3：因果 / 数值推理 / 操作迁移 / 反事实 / 多证据综合
```

### 4.6 Stage 4：证据锚定 CoT

CoT 的目标不是写得长，而是让推理步骤能被证据验证。

标准动作：

```text
segment_retrieval(video_id, query)
  → 找到与问题相关的候选时间段

segment_query(video_id, timestamp)
  → 读取该时间段的细粒度事件

evidence_align(evidence_ids)
  → 建立跨时间或跨视频的实体、状态和步骤对应

reason(evidence_ids)
  → 基于已确认事实完成比较、因果或迁移推理

answer()
  → 输出最终答案
```

标准轨迹：

```text
<thought>需要比较两个视频中箱子最终所处的位置。</thought>
<action>segment_retrieval(video_A, "箱子最终位置")</action>
<observation evidence_id="E1">[video_A|00:31-00:38] 箱子被移到地面。</observation>
<action>segment_retrieval(video_B, "箱子最终位置")</action>
<observation evidence_id="E2">[video_B|01:12-01:19] 箱子仍在桌面。</observation>
<action>evidence_align(E1, E2)</action>
<thought>两个视频中的最终状态不同。</thought>
<answer>视频 A 中箱子在地面，视频 B 中仍在桌面。</answer>
```

CoT 硬要求：

- 每个关键观察包含 `video_id + timestamp + evidence_id`。
- 推理只能引用已出现的 observation。
- 最终答案必须能由 observation 推出。
- 不写与作答无关的冗长过程。
- 答案正确但证据错误的样本不得进入 SFT。

### 4.7 Stage 5：Quality Gate 与人工终审

#### 自动 Gate

```text
G1 Schema：字段完整、JSON 可解析、时间戳合法
G2 Caption：关键事件覆盖、时间连续、实体命名一致
G3 Answer：答案被证据支持、格式符合任务要求
G4 Dependency：Text-Only / Summary-Only / Cross-Evidence 检测
G5 CoT：引用合法、无悬空证据、结论可由证据推出
G6 Dedup：视频、caption、问题和语义近重复检测
G7 Leakage：与评测素材、问题模板和答案模式隔离
```

#### 人工终审

人工负责自动化最不稳定的四类判断：

1. Caption 是否遗漏影响答案的细节。
2. 任务是否真正命中目标能力和难度。
3. 跨视频关系是否真实成立，而非表面相似。
4. CoT 是否忠实、必要、可学习。

硬门槛采用不可补偿规则：

```text
答案错误             → Reject
关键证据不真实       → Reject
不依赖视觉即可回答   → Reject / Rewrite
伪跨视频             → Reject / 降级为单视频任务
CoT 引用不存在的证据 → Reject
```

表达、格式和冗余可以修复，但不能用“整体得分高”抵消证据错误。

### 4.8 Stage 6：数据打包与版本化

建议数据 Schema：

```json
{
  "sample_id": "vlma_l2_state_00017",
  "source": {
    "video_ids": ["video_A", "video_B"],
    "tos_links": ["tos://...", "tos://..."],
    "license": "internal_checked"
  },
  "captions": {
    "detail": [
      {
        "evidence_id": "E1",
        "video_id": "video_A",
        "start": "00:31",
        "end": "00:38",
        "text": "红色箱子由桌面移动到地面。"
      }
    ],
    "summary": "两个视频展示了箱子在不同操作后的最终位置。"
  },
  "task": {
    "instruction": "比较两个视频中箱子的最终位置。",
    "answer": "视频 A 中在地面，视频 B 中仍在桌面。",
    "cot": [],
    "required_evidence_ids": ["E1", "E2"]
  },
  "taxonomy": {
    "level": "L2",
    "skill": "state_change",
    "cross_video_dependency": true
  },
  "difficulty": {
    "evidence_count": 2,
    "distractor_level": "medium",
    "reasoning_steps": 2
  },
  "quality": {
    "caption_pass": true,
    "answer_support_pass": true,
    "text_only_pass": true,
    "summary_only_pass": true,
    "cross_evidence_pass": true,
    "human_final_pass": true
  },
  "version": {
    "dataset": "v1.0",
    "rubric": "v1.2",
    "generator": "recorded_model_version"
  }
}
```

必须保留：

- 原始模型与 Prompt 版本；
- Rubric 和 Judge 版本；
- 每个 Gate 的结果和失败原因；
- 人工修改前后内容；
- 数据集版本与训练批次；
- 从 Benchmark gap 到训练样本的映射。

---

## 五、Pipeline 工程化

### 5.1 状态机

```text
DISCOVERED
→ PREVIEW_PASSED
→ INGESTED
→ DEDUPED
→ CAPTIONED
→ TASK_GENERATED
→ FILTERED
→ COT_GENERATED
→ AUTO_QC_PASSED
→ HUMAN_APPROVED
→ DELIVERED
```

失败状态需要携带原因，不直接删除：

```text
REJECT_SOURCE
REJECT_DIFFICULTY
REJECT_CAPTION
REJECT_SHORTCUT
REJECT_EVIDENCE
REJECT_DUPLICATE
NEEDS_REPAIR
```

### 5.2 工程要求

- **幂等**：同一 `sample_id` 重跑不会重复入库。
- **断点续跑**：单节点失败后从该节点恢复。
- **错误隔离**：单条视频失败不阻塞整批。
- **背压**：Caption/生成模型吞吐低时，上游不无限堆积。
- **可观测**：统计各节点耗时、成功率和淘汰率。
- **可追踪**：任一训练样本可回溯到来源、证据、生成模型和人工修改记录。

### 5.3 效率指标

```text
Source Precision       = 预检合格候选 / 检索候选
Caption Pass Rate      = Caption Gate 通过数 / 入库视频数
Task Yield             = 合格任务数 / Caption 完成视频数
Visual Dependency Rate = 通过 F2/F3/F4 的任务数 / 生成任务数
Human Acceptance Rate  = 人工终审通过数 / 自动 Gate 通过数
Data Efficiency        = 目标能力指标增量 / 有效训练样本数
Label Efficiency       = 目标能力指标增量 / 人工小时
```

---

## 六、训练消费设计

### 6.1 SFT 数据配方

数据不应只混入长 CoT，建议保留三类响应路径：

| 数据线 | 训练目标 | 作用 |
|--------|----------|------|
| Caption / Understanding | 从视频生成结构化事件描述 | 保持视觉理解、OCR、状态识别 |
| Direct QA / Grounding | 基于视频直接回答或定位证据 | 保持答案效率和视觉定位 |
| Evidence-grounded CoT | 定位、取证、对齐、推理后回答 | 增强复杂推理与可追溯性 |

混合三类数据的原因：

- 只训长 CoT 可能让模型输出冗长，并损害原有 Grounding。
- 只训 Direct QA 学不到跨片段取证过程。
- 只训 Caption 无法保证任务完成和推理迁移。

具体配比、学习率、冻结策略、epoch 和教师模型属于算法侧未确认信息，本文不虚构。

### 6.2 SFT 学到什么

```text
Caption 数据：
提高 p(结构化视觉描述 | 视频)

Direct QA：
提高 p(正确答案 | 视频, 问题)

Evidence-grounded CoT：
提高 p(下一步定位/观察/推理 | 视频, 问题, 已有轨迹)
```

数据能否转化为能力取决于：

1. 样本是否命中真实能力缺口；
2. 视觉证据是否进入模型输入；
3. 输出是否提供正确且一致的监督；
4. 难度是否处于模型可学习边界；
5. 数据配比是否避免过拟合和能力遗忘；
6. 独立评测是否出现泛化收益。

---

## 七、评测与实验设计

### 7.1 四层评测

| 层级 | 回答的问题 | 主要指标 |
|------|------------|----------|
| 数据质量 | 构造的数据可靠吗 | Caption 准确率、证据支持率、捷径率、人工通过率 |
| 视觉能力 | 模型具体学会了什么 | L1/L2/L3 分桶准确率、时间定位、跨视频对齐 |
| 通用回归 | 专项训练是否导致降智 | 未覆盖子集、通用视频理解、Grounding |
| Agent 端到端 | 视觉提升能否迁移到任务完成 | ClawEval 多模态子项、WildClawBench、任务完成率 |

### 7.2 正确的对照

训练前后必须固定：

```text
base checkpoint
评测集版本
视频采样和帧预算
prompt template
解码参数
judge model
rubric version
```

训练、开发和测试素材需按视频来源与语义近邻隔离，不能只按 `sample_id` 随机切分。

### 7.3 最小消融矩阵

| 实验组 | Caption | 反捷径过滤 | 证据 CoT | 目的 |
|--------|---------|------------|----------|------|
| A Base | - | - | - | 原始基线 |
| B QA-only | - | - | - | 验证简单 QA 的增益上限 |
| C Caption + filtered QA | ✓ | ✓ | - | 验证证据底座与反捷径 |
| D Full | ✓ | ✓ | ✓ | 验证证据锚定 CoT 增量 |

如资源允许，再增加：

```text
D1 去掉 Text-Only Filter
D2 去掉 Summary-Only Filter
D3 去掉 Cross-Evidence Filter
D4 只有长 CoT、不混 Caption/Direct QA
```

这组实验才能回答：

- 收益来自更多数据，还是来自更强视觉依赖？
- Caption 是辅助资产，还是对理解能力有独立贡献？
- CoT 提升了推理，还是只增加输出长度？
- 跨视频任务是否真的依赖两个视频？
- 专项训练是否损害原有 Grounding 和直接回答能力？

### 7.4 当前结果口径

| 指标 | 当前结果 | 能证明什么 | 不能证明什么 |
|------|----------|------------|--------------|
| 内部覆盖子集 | +2.8 | 靶向数据方向有效 | 不能代表内部全量或所有视觉能力 |
| ClawEval Overall | +0.5 | 共享模型端到端整体有改善 | 不能直接等价为本项目净贡献 |
| WildClawBench | +2.3 | 真实长链路 Agent 整体改善 | 无消融时不能排除其他数据线贡献 |

下一轮最需要补的不是新总分，而是：

```text
Capability Label
× Training Coverage
× Pre/Post Delta
× Held-out Delta
```

---

## 八、项目角色与个人贡献

推荐表述：

> 我负责把 Agent Benchmark 暴露的视觉失分，转化为可交付、可训练、可追踪的数据规格和生产流程。具体包括能力与难度拆解、靶向寻源、时间戳 Caption 和证据结构设计、Question/Answer/CoT 质量标准、半自动 Pipeline、人工作业规范，以及训练后评测结果的业务归因。算法侧负责实际 SFT、训练配置和最终评测执行。

不要表述为：

```text
“我独立训练了模型并证明提升 +2.3”
“我完整复现了 ReWatch-R1”
“项目只交付了一批 VQA”
“所有 Agent 指标上涨都来自我的数据”
```

更准确的价值：

```text
Benchmark badcase
→ 能力缺口
→ 视觉证据数据规格
→ 可规模化 Pipeline
→ Training Signal Quality Gate
→ 训练交付
→ 结果解释与下一轮策略
```

---

## 九、与 ReWatch-R1 的对应关系

| ReWatch-R1 | 本项目适配 | 差异 |
|------------|------------|------|
| 分层 Captioning | 视频分段 + 绝对时间戳事件流 | 多视频增加 `video_id` 锚点 |
| Detailed vs Summary 出题 | 构造必须依赖细节的任务 | 从通用生成改为 Benchmark gap 定向生成 |
| F1/F2/F3 过滤 | 答案支持、Text-Only、Summary-Only | 增加 Cross-Evidence Filter |
| ReAct CoT | 定位、查询、对齐、推理、回答 | 增加跨视频实体与状态对齐 |
| SFT + GRPO | 当前项目已知止于 SFT | RL 和 O&R 奖励属于后续方向 |
| 五个视频推理基准 | 内部 161 题 + Agent 基准 | 本项目还需证明视觉能力到 Agent 的迁移 |

结论：

> ReWatch-R1 的数据构造思想高度适用于本项目，尤其适合规范 Caption、时间戳、任务和 CoT 的证据关系；但它不能原样替代本项目的 Benchmark 归因、多视频必要性验证和 Agent 端到端评测。

---

## 十、下一轮实施路线

### Phase 1：把既有数据标准化

- 为现有样本补齐 `video_id + timestamp + evidence_id`。
- 统一 detailed caption 与 summary。
- 将 QA 和 CoT 显式绑定到 required evidence。
- 补充能力、难度、来源和版本字段。

### Phase 2：在小样本上验证过滤器

- 抽取一批现有样本运行 Text-Only 和 Summary-Only 测试。
- 人工核验 Cross-Evidence Filter 是否能识别伪多视频题。
- 统计过滤前后人工通过率和题型分布。
- 校准误杀率与漏放率，再决定是否扩到全量。

### Phase 3：形成可归因训练实验

- 按最小消融矩阵准备四组数据。
- 锁定训练和评测配置。
- 在内部 161 题上补齐能力标签和覆盖映射。
- 分别报告覆盖能力、未覆盖能力和通用回归。

### Phase 4：扩展到视觉 Agent 数据

在视频理解数据稳定后，再补三类 Agent 数据：

```text
GUI Understanding：截图 caption、OCR、布局和状态语义
GUI Grounding：指令到 point / bbox / action
Agent Trajectory：观察、计划、动作、反馈、恢复与 verifier
```

三条数据线需要持续混合，避免只训练长轨迹导致视觉理解或 Grounding 退化。

---

## 十一、2 分钟项目陈述

> 这个项目我重新定义为“Benchmark 驱动的 VLM-Agent 视觉证据数据引擎”。背景是我们在 ClawEval 和 WildClawBench 的任务中发现，一部分 Agent 失败并不是规划或工具调用问题，而是更早的视觉观察出了错，比如没有定位到关键片段、误读状态，或者无法建立跨视频关系，导致后续决策建立在错误事实之上。
>
> 我的核心工作不是简单补一批 VQA，而是把 Benchmark badcase 转成完整的视觉证据训练数据。首先，我把能力拆成 L1 感知定位、L2 对齐比较和 L3 多证据推理，再按能力缺口靶向寻源。数据入库后，先生成带绝对时间戳的 detailed caption，作为共享证据底座；问题、答案和 CoT 都围绕这份底座构造。出题时采用 detailed caption 和 summary 的信息差，保证问题必须依赖细节；质检时检查答案支持、Text-Only、Summary-Only 和跨视频必要性；CoT 则固定成“定位、取证、对齐、推理、回答”的结构，每一步都能回指具体视频和时间点。最后通过半自动 Pipeline 生成候选，由人工守住证据真实性、难度和可学习性的最终 Gate。
>
> 首轮实际交付覆盖约 74 个 topic、约 400 条素材，内部 161 题中覆盖 72 题。算法侧 SFT 后，内部覆盖子集提升 2.8，ClawEval Overall 提升 0.5，WildClawBench 提升 2.3。这里我会严格说明，两个 Agent 基准的收益来自共享基座，缺少完整消融，不能全部归因给本项目。首轮真正得到的策略结论是：数据方向有效，但覆盖率只有约 45%，下一轮应优先扩能力覆盖，并通过 Caption、过滤器和证据 CoT 的组件消融，把“什么数据带来什么能力”证明清楚。

---

## 十二、待确认清单

- [ ] 原数据中 caption、timestamp、QA、CoT 各自的准确交付数量。
- [ ] Caption 是人工生成、模型生成还是人机协作，以及具体质量标准。
- [ ] 实际进入 SFT 的样本量、数据配比和去重后有效量。
- [ ] SFT 的 base model、训练模块、冻结策略、epoch 和学习率。
- [ ] 内部 161 题的能力标签、内部全量变化和未覆盖 89 题变化。
- [ ] ClawEval Multimodal / Video QA 子项的训练前后结果。
- [ ] WildClawBench 中视觉相关任务与非视觉任务的分项结果。
- [ ] 本项目与其他训练数据线的单独/联合消融。
- [ ] Text-Only、Summary-Only、Cross-Evidence 三类过滤的真实试验结果。
- [ ] Caption 错误向 QA/CoT 传播的抽检数据。

---

**最后更新：2026-09-14**
