```
## 一、基础信息

**项目时间**：2026.06-至今

**岗位**：AI 数据与安全 / AI数据工程师

### 二、项目背景

针对 VLM 长视频复杂理解与推理能力训练，围绕“实践操作类”场景构建长视频训练数据。项目目标是获取 **500 条、30～90 分钟的高质量实践操作视频**，建立从数据需求定义、候选检索、自动筛选、视频采集到 TOS 交付的端到端数据生产 Pipeline，为后续视频标注与训练数据构建提供稳定数据源。

### 三、技术架构

yt-dlp、FFmpeg、自研检索平台、自研 Fetch 平台、Agent、VLM、TOS、并发任务处理

### 四、主要职责

1. **训练数据 Pipeline 设计**：将训练需求拆解为 Data Rubric 与可执行数据规则，搭建“需求定义→候选检索→多级筛选→并发采集→格式标准化→TOS 交付”的端到端长视频数据 Pipeline，统一管理 URL、Metadata、筛选状态、抓取状态及存储地址。

2. **候选数据检索与 Query Expansion**：基于自研检索平台构建多关键词并发召回，同时引入 Agent 根据 Data Rubric 自动扩展长尾检索 Query，扩大实践操作类视频覆盖范围；对召回结果进行 URL/Metadata 标准化与去重，形成候选视频池。

3. **多级数据质量筛选**：设计“标题规则→Metadata 校验→VLM 内容质检”的分层过滤机制；通过媒体流 Metadata 校验视频时长、分辨率等硬约束，并基于 yt-dlp + FFmpeg 对视频流进行解码抽帧，将代表性帧送入 VLM 进行内容理解与目标场景判定，输出通过/淘汰/人工复核结果，降低无效视频进入高成本采集与处理环节的比例。

4. **视频采集与数据交付自动化**：基于自研 Fetch 平台搭建并发视频采集任务，设计任务状态管理与失败重试机制；完成视频格式标准化并写入 TOS，实时回写抓取、转码及存储状态，最终交付带 TOS 链接及完整 Metadata 的训练数据表。

### 五、项目成效

1. **完成 3000 条候选视频召回、520 条内容筛选通过，最终成功抓取并转存 TOS 500 条**，达到项目既定数据量目标；

2. 通过“低成本规则过滤→Metadata 校验→VLM 内容质检→完整采集”的 Cost-aware 数据处理策略，减少无效视频进入完整下载及后处理环节，提升数据生产效率；

3. 建立可复用的**长视频训练数据获取 Pipeline**，实现从数据需求到数据交付的标准化、自动化处理，为后续视频标注、Caption 生成及 VLM 训练数据构建提供稳定数据基础。
```
--- 
```
项目名称：长视频复杂推理视频获取pipeline
项目背景：为训练vlm在长视频复杂推理推理能力，分目的类型的构造训练数据，需要获取500条实践操作类视频，样例视频为化学操作实验
项目动作：
完整数据流：
需求-检索-筛选-抓取-交付
    1. 明确需求：
        1. 实践操作类视频，时长30min-90min，样例：化学实验操作视频。
        2. 根据需求文档，细化目标数据规则
    2. 定义检索rubric并检索返回：
        1. 通用类检索：化学、生物、物理、实验操作
            自研检索平台，通过并发检索，返回视频url及相关信息
        2. 拓展类检索：auto research，由agent读取目标数据规则，自主扩展关键词进行检索.
            使用agent平台，读取rubric，执行并发检索（或接入自研检索平台的接口），返回视频url及相关信息
    3.预筛检索数据
        step1. 定义视频标题筛选规则
            检索的视频中可能有公开课视频而非实践操作类，设置关键词过滤层
            输出：合格（进入step2）/不合格（废弃）
        step2. 定义画质筛选规则
            根据url读取视频的最高画质，最高1080p，最低720p，否则废弃
        step3. 定义视频内容过滤层
            以哔哩哔哩为例：
            B站视频页面 URL
                    ↓
                yt-dlp
                    ↓
            找到真正的媒体流入口
                    ↓
                ffmpeg
                    ↓
            不断从服务器/CDN获取媒体数据
                    ↓
            解码 H.264/H.265...
                    ↓
                视频帧
                    ↓
                间隔抽取需要的帧
                    ↓
                JPEG
                    ↓
                接入VLM解析，形成caption，判断是否属于目标数据
                输出并写回表格筛选结果：合格/不合格/人工校验
    4. 抓取合格数据并转tos
        step1.通过自研fetch平台，对表格中合格的url视频源进行并发抓取保存，实时写回抓取结果（成功/失败，设置失败重试机制）
                    ↓
        step2.下载后的视频实时写回aidp平台并返回TOS链接，转格式结果（成功/失败，设置失败重试机制）
            TOS链接便于后续的标注作业与训练
    5. 交付带有tos链接的数据表格
项目结果：需求目标数据500条，检索结果3000条，筛选结果520条，抓取并转格式成功结果500条
项目思考：
    1. 检索与抓取分离，减少抓取的时间损耗，并发检索。
    2. 检索筛选，保证数据符合需求定义，提高数据可用率。
    3. 抽帧逻辑定义，用较粗的帧率实现捕获视频caption，在保证成功读取视频内容的同时降低推理消耗。
    4. 不足：筛选与抓取是断点式操作，可以做成异步并发提高时间效率。
```


# 项目名称：长视频复杂推理视频获取 Pipeline

## 一、项目背景

为训练 VLM 在**长视频场景下的复杂理解与推理能力**，需要围绕目标能力构建规模化训练数据。

本项目聚焦于**实践操作类长视频数据的自动化获取与预处理**，以化学实验、生物实验、物理实验等操作型视频为主要数据来源，目标获取 **500 条、时长 30～90 分钟的高质量长视频**，为后续视频标注、Caption 构建、训练数据生产提供原始数据。

项目核心并非单纯的视频下载，而是建立一套：

> **需求定义 → 候选召回 → 多级质量筛选 → 视频采集 → 格式标准化 → 对象存储 → 数据交付**

的长视频数据生产 Pipeline。

---

# 二、完整数据流

```text
需求定义
    ↓
数据 Rubric
    ↓
候选数据检索
    ↓
URL / Metadata 标准化
    ↓
多级预筛
    ├── 标题规则过滤
    ├── Metadata / 画质过滤
    └── 内容级筛选（抽帧 + VLM）
    ↓
Qualified Dataset
    ↓
视频采集 / Fetch
    ↓
转码 / 格式标准化
    ↓
对象存储（TOS）
    ↓
数据表回写
    ↓
交付标注 / 训练
```

从 AI 数据工程视角，将整个流程拆分为：

```text
Data Discovery
      ↓
Data Validation
      ↓
Data Acquisition
      ↓
Data Processing
      ↓
Data Storage
      ↓
Data Delivery
```

---

# 三、项目动作

## 1. 明确数据需求并转化为 Data Rubric

### 1.1 需求定义

根据模型训练目标，将业务需求转化为**机器可执行的数据规则（Data Rubric）**。

目标数据定义：

| 维度   | 数据规则            |
| ---- | --------------- |
| 数据类型 | 实践操作类视频         |
| 典型场景 | 化学 / 生物 / 物理实验  |
| 视频时长 | 30～90 min       |
| 视频质量 | 720p～1080p      |
| 核心内容 | 存在连续、可观察的实际操作过程 |
| 数据用途 | VLM 标注、训练数据构建   |

其中重点约束：

> 视频需要具备连续操作过程，而不是单纯的理论课程、实验原理讲解或实验结果展示。

### 1.2 Rubric 结构化

将需求文档拆解为：

```text
Target Definition
    ↓
Positive Criteria
    ↓
Negative Criteria
    ↓
Hard Constraints
    ↓
Review Criteria
```

例如：

**Positive Criteria**

```text
存在实际实验操作
存在连续步骤
存在操作对象 / 仪器 / 试剂
能够观察到过程变化
```

**Negative Criteria**

```text
纯理论课程
PPT 讲解
实验结果展示
与实验相关但不存在实际操作
```

**Hard Constraints**

```text
30min ≤ duration ≤ 90min
resolution >= 720p
```

**Review Criteria**

```text
模型判断不确定
标题与实际内容不一致
存在较长无关片段
```

这样可以保证后续**检索、筛选、人工审核使用的是同一套数据标准**。

---

# 2. 候选数据检索与召回

## 2.1 通用关键词检索

基于 Data Rubric 构建初始 Query Set：

```text
化学实验
化学实验操作
实验全过程
化学实验演示
生物实验操作
物理实验操作
laboratory experiment
experiment procedure
```

通过自研检索平台执行**多 Query 并发召回**。

### 数据流

```text
Query Set
    ↓
并发检索
    ↓
Candidate URLs
    ↓
Metadata Extraction
    ↓
Candidate Dataset
```

候选数据至少保留：

```text
video_url
title
author
source
duration
publish_time
query
```

其中 `query` 建议保留，用于后续分析：

> 哪些 Query 的召回质量高、哪些 Query 噪声高。

---

## 2.2 Agent 驱动的 Query Expansion

在传统关键词召回基础上，引入 Agent 扩展检索空间。

Agent 输入：

```text
Data Rubric
```

Agent 输出：

```text
Expanded Query Set
```

例如：

```text
“化学实验操作”
        ↓
Agent
        ↓
化学滴定实验全过程
有机合成实验操作
大学化学实验演示
酸碱滴定实验步骤
laboratory chemical procedure
...
```

随后通过检索 API 并发执行：

```text
Rubric
   ↓
Agent Query Expansion
   ↓
Concurrent Retrieval
   ↓
Candidate URLs
```

因此形成：

> **Rule-based Retrieval + Agent-based Retrieval**

两路召回。

### 数据工程上的价值

不是单纯“让 Agent 搜索”，而是利用 Agent 做：

> **Query Expansion / Search Space Expansion**

提高长尾数据覆盖能力。

---

# 3. Candidate Dataset 标准化与去重

这一层建议在你原来的流程里补出来。

不同检索源可能返回：

* 同一个视频不同页面 URL
* 同一个视频不同清晰度 URL
* 重复搜索结果
* URL 参数不同但实际资源相同

因此需要做：

```text
URL Normalize
    ↓
URL Deduplication
    ↓
Metadata Normalize
```

例如统一：

```text
source
video_id
title
duration
resolution
video_url
```

并建立唯一数据 ID：

```text
data_id
```

形成统一的 Candidate Dataset。

---

# 4. 多级预筛 Pipeline

筛选策略遵循：

> **低成本规则优先，高成本模型判断后置。**

整体：

```text
Candidate Dataset
        ↓
Rule Filter
        ↓
Metadata Filter
        ↓
Content Filter
        ↓
Qualified Dataset
```

---

## Step 1：标题规则过滤

标题属于成本最低的特征，因此首先进行规则过滤。

### Positive / Negative Keyword Filter

例如：

```text
Positive:
实验
实验操作
实验过程
实验演示
experiment
procedure
laboratory
```

```text
Negative:
公开课
理论
课程
考试
习题
知识讲解
review
lecture
```

输出：

```text
PASS
REJECT
```

这里建议你在项目里使用：

> **Rule-based Candidate Filtering**

而不是简单说“标题筛选”。

---

# Step 2：Metadata / 画质过滤

根据视频 URL 获取媒体元信息，而非直接下载完整视频。

流程：

```text
video_url
    ↓
Media Metadata Probe
    ↓
duration
resolution
codec
format
bitrate
fps
    ↓
Rule Validation
```

例如：

```text
duration ∈ [30min, 90min]
resolution >= 720p
```

不满足硬约束的数据直接淘汰。

### 技术表述建议

不要写：

> “读取视频最高画质。”

建议写：

> **解析媒体 Manifest / Format Metadata，获取可用视频流的分辨率、编码格式、时长等媒体属性，并执行规则校验。**

因为“最高画质”本质上是在读取媒体 Format / Stream Metadata。

---

# Step 3：内容级筛选

这是整个 Pipeline 中成本最高的预筛步骤。

由于：

> **Metadata 无法判断视频内容是否真正属于实践操作类。**

因此增加：

> **Content-level Filtering**

采用：

```text
视频流解析
    ↓
低成本抽帧
    ↓
关键帧序列
    ↓
VLM Inference
    ↓
Content Caption / Classification
    ↓
Rule-based Decision
```

---

## 3.1 视频流解析

以 Bilibili 为例：

```text
Bilibili Page URL
        ↓
yt-dlp
        ↓
Resolve Media Formats / Stream URL
        ↓
DASH / FLV
        ↓
ffmpeg
        ↓
Decode
        ↓
Video Frames
```

这里更准确的工程描述是：

> 使用 yt-dlp 解析页面层资源信息并获取可访问的视频媒体流 / Format 信息，再通过 FFmpeg 对媒体流进行解码和抽帧。

---

## 3.2 边拉流边抽帧

对于长视频，不需要先把完整视频下载下来再做内容筛选。

Pipeline 可以采用：

```text
Media Stream
      ↓
FFmpeg
      ↓
Decode
      ↓
Frame Sampling
      ↓
JPEG
```

即：

> **Streaming Decode + Frame Sampling**

降低：

* 中间存储开销
* 无效视频下载量
* 视频预筛耗时

---

# 3.3 长视频抽帧策略

30～90 分钟视频如果逐帧分析，VLM 推理成本非常高。

因此采用：

> **Temporal Sampling**

根据视频时间长度按照固定时间间隔抽取代表帧。

例如：

```text
90 min video
      ↓
Temporal Sampling
      ↓
N representative frames
```

这些 Frame 不用于最终训练数据，而主要用于：

> **Candidate Video Content Verification**

即判断：

```text
是否存在实际操作？
是否属于目标场景？
是否具有连续操作过程？
是否存在大量无关内容？
```

---

# 3.4 VLM 内容分析

将抽样得到的 Frame 输入 VLM。

可以设计统一 Prompt / Rubric：

```text
Input:
视频关键帧序列

Output:
{
    "is_practical_operation": true,
    "scene": "chemical_experiment",
    "has_continuous_operation": true,
    "confidence": 0.91,
    "reason": "..."
}
```

这里建议不要把流程描述成：

> “VLM Caption，然后判断。”

更专业的说法是：

> **利用 VLM 对视频关键帧进行视觉语义理解，通过 Caption / Classification 等方式提取内容语义，并基于 Data Rubric 执行内容级质量判定。**

最终输出：

```text
PASS
REJECT
REVIEW
```

其中 `REVIEW` 进入人工兜底。

---

# 5. Qualified Dataset → 视频采集

经过多级筛选后，只对 Qualified Dataset 执行完整视频采集。

```text
Qualified Dataset
       ↓
Fetch Task Queue
       ↓
Concurrent Download
       ↓
Local / Temporary Storage
```

这里建议把“抓取”描述成：

> **Data Acquisition / Fetch**

而不是单纯“下载”。

---

## 5.1 并发 Fetch

通过自研 Fetch 平台，将 Qualified URL 转换成异步 Fetch Task：

```text
video_url
    ↓
Task Queue
    ↓
Worker Pool
    ↓
Concurrent Fetch
```

每条任务维护状态：

```text
PENDING
RUNNING
SUCCESS
FAILED
RETRY
```

这样整个数据采集过程具备：

> **任务状态管理 + 并发处理 + Fail Retry**

能力。

---

## 5.2 失败重试

针对：

* 网络超时
* CDN 异常
* Source unavailable
* 下载中断

设计重试机制：

```text
FAILED
  ↓
Retry
  ↓
Retry Limit
  ↓
SUCCESS / FINAL_FAILED
```

并记录失败原因：

```text
error_code
error_message
retry_count
```

便于后续统计失败率和定位 Source 问题。

---

# 6. 视频标准化处理

视频抓取之后进入标准化处理阶段。

```text
Raw Video
    ↓
Format Validation
    ↓
Transcoding
    ↓
Standard Video
```

根据下游标注 / 训练要求统一：

```text
container format
codec
resolution
fps
audio
```

并记录处理状态：

```text
transcode_status
```

---

# 7. 对象存储与 TOS 写入

标准化视频上传至 TOS：

```text
Standard Video
      ↓
TOS Object Storage
      ↓
Object URL
      ↓
Metadata Write-back
```

最终形成：

```text
data_id
video_url
duration
resolution
content_check
fetch_status
transcode_status
tos_url
```

其中 TOS URL 作为下游标注及训练环节的视频访问入口。

---

# 8. 数据交付

最终交付的数据并非单纯的 TOS URL，而是：

> **Metadata + Quality Status + Storage Location**

组成的数据表。

建议最终 Schema：

| 字段               | 类型     | 含义         |
| ---------------- | ------ | ---------- |
| data_id          | string | 数据唯一 ID    |
| source           | string | 数据来源       |
| video_url        | string | 原始页面 URL   |
| title            | string | 视频标题       |
| duration         | float  | 视频时长       |
| resolution       | string | 视频分辨率      |
| retrieval_query  | string | 命中的 Query  |
| title_check      | string | 标题筛选结果     |
| content_check    | string | VLM 内容筛选结果 |
| fetch_status     | string | 视频采集状态     |
| transcode_status | string | 转码状态       |
| tos_url          | string | TOS 地址     |
| review_status    | string | 人工复核状态     |
| remark           | string | 异常说明       |

最终交付：

> **500 条具备稳定存储地址、完整 Metadata 和质量状态标记的长视频数据。**

---

# 四、项目结果

```text
目标数据
500 条
   ↓
候选召回
3000 条
   ↓
多级质量筛选
520 条
   ↓
完整采集 + 标准化 + TOS
500 条
```

核心结果：

| 阶段   |  数据量 |
| ---- | ---: |
| 需求目标 |  500 |
| 候选召回 | 3000 |
| 筛选通过 |  520 |
| 最终交付 |  500 |

候选数据规模达到目标量的 **6 倍**，经过规则筛选、Metadata 校验和 VLM 内容筛选后，最终完成 500 条视频的采集、标准化和对象存储交付。

---

# 五、从 AI 数据工程角度的项目思考

## 1. 将“找数据”转化为 Data Discovery Pipeline

核心不是人工搜索，而是建立：

```text
Rubric
 ↓
Query Set
 ↓
Retrieval
 ↓
Candidate Dataset
```

将自然语言数据需求转化成机器可执行的数据召回策略。

同时引入 Agent 做 Query Expansion，扩大长尾场景覆盖。

---

## 2. 采用 Cost-aware 的多级数据过滤

不同数据处理环节成本不同，因此设计：

```text
Rule Filter
      ↓
Metadata Filter
      ↓
VLM Filter
      ↓
Human Review
      ↓
Full Fetch
```

遵循：

> **低成本判断优先，高成本判断后置。**

这样可以显著减少无效视频进入 VLM 推理和完整下载阶段。

---

## 3. 将 VLM 用作 Data Quality Gate

VLM 在这个 Pipeline 中并不是最终训练模型，而是作为：

> **Data Quality Gate / Content Validator**

对传统 Metadata 无法判断的语义属性进行自动审核。

例如：

```text
Metadata：
“化学实验”
```

无法证明实际存在操作过程。

而 VLM 可以从抽样视频帧中判断：

```text
是否真实实验？
是否存在连续操作？
是否包含目标场景？
```

实现：

> **Rule-based QC + Model-based QC**

结合。

---

## 4. 抽帧策略本质上是数据处理中的 Cost-Quality Trade-off

抽帧不是简单的“减少图片数量”，而是在：

```text
Sampling Cost
       ↕
Semantic Coverage
```

之间进行权衡。

采样过低：

```text
↓
遗漏关键操作
↓
误判率提升
```

采样过高：

```text
↑
VLM Token / Inference Cost
↑
处理时间
```

因此后续可以继续优化：

> **Fixed Sampling → Scene-aware Sampling → Information-aware Sampling**

进一步实现根据场景变化和信息密度动态调整采样率。

---

## 5. 当前 Pipeline 的主要瓶颈：阶段式串行

当前流程更接近：

```text
Retrieval
   ↓
Filtering
   ↓
Fetching
   ↓
Transcoding
   ↓
Storage
```

各阶段存在明显的 Batch Barrier。

例如：

```text
3000 条检索全部完成
        ↓
520 条筛选全部完成
        ↓
开始抓取
```

因此后续可以升级为：

```text
                ┌→ Filter Worker
Retrieval Queue ├→ Filter Worker
                └→ Filter Worker
                       ↓
                  Fetch Queue
                       ↓
                Fetch Worker Pool
                       ↓
                 Transcode Queue
                       ↓
                Transcode Worker
                       ↓
                    TOS
```

通过：

> **Queue + Worker Pool + Async Processing**

形成流水线并行执行。

这样可以让：

```text
第 1 批数据 → 筛选 → 抓取 → 转码
```

与：

```text
第 2 批数据 → 检索
```

同时运行，提高端到端吞吐。

---

# 六、AI 数据工程能力沉淀

该项目最终沉淀的不只是视频采集脚本，而是一套面向 VLM Training Data 的数据生产能力：

```text
                 Data Requirement
                        ↓
                    Data Rubric
                        ↓
               Query / Agent Search
                        ↓
                Candidate Dataset
                        ↓
             ┌──────────┴──────────┐
             ↓                     ↓
        Rule-based QC        Metadata QC
             └──────────┬──────────┘
                        ↓
                  VLM Content QC
                        ↓
                  Human Review
                        ↓
                 Qualified Dataset
                        ↓
                  Fetch Pipeline
                        ↓
                    Transcode
                        ↓
                      TOS
                        ↓
                    Data Delivery
```

从 AI 数据工程视角，项目核心能力可以总结为：

> **围绕 VLM 训练目标，将数据需求结构化为 Data Rubric，并建立从数据发现、候选召回、规则 QC、模型 QC、采集、标准化到对象存储交付的端到端训练数据 Pipeline。**
