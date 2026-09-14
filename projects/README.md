# 项目索引

> 按项目独立记录，持续填充。每个项目一个文件，记录背景、技术解剖、知识连接、面试故事。
>
> 更新：2026-09-14（最新口径）：字节项目分**业务/开发**两类。业务=CoT evalset 构造 / VLM-Agent 视觉证据数据引擎 / 长视频复杂推理；开发=数据构造 Pipeline、数据质检 Pipeline。VLM-Agent 当前主文档为 `bytedance-vlm-agent视觉证据数据引擎.md`，以时间戳 Caption 为共享证据底座，串联任务生成、证据锚定 CoT、质量 Gate、SFT 与分层评测；原 `bytedance-vlm-multivideo-capability-loop.md` 保留为历史详版。

---

## 当前项目

### 字节 · 业务项目（数据构造 / 评测集构造）

| 项目 | 类型 | 文件 | 状态 | 时间 |
|------|------|------|------|------|
| CoT 基础学科长尾 + 推理 | **evalset 构造** | `bytedance-cot-compressed-evalset.md` | 🟢 已交付，持续深挖 | 2026-06 ~ 07 |
| VLM-Agent 视觉证据数据引擎（Agent 失分 → Caption/时间戳/证据 CoT → SFT） | 训练数据构造 | `bytedance-vlm-agent视觉证据数据引擎.md`（主文档）+ `bytedance-vlm-agent能力建设.md`（全链路纲要）+ `bytedance-vlm-multivideo-capability-loop.md`（历史详版） | 🟢 进行中（首轮闭环已跑通，方法已重构） | 2026-08 ~ |
| 长视频复杂推理（含多视频=拼接后多跳） | 训练数据构造 | `bytedance-长视频复杂推理能力建设.md`（纲要/能力地图） | 🟢 进行中 | 2026-08 ~ |

### 字节 · 开发项目（Pipeline）

| 项目 | 类型 | 文件 / 位置 | 状态 |
|------|------|------------|------|
| 数据构造 Pipeline（寻源→入库→去重→自动 Prompt/CoT→人工终审 DAG） | 开发 | `bytedance-数据构造pipeline.md` | 🟢 已成文 |
| 数据质检 Pipeline（多层 Quality Gate + LLM Repair） | 开发 | 内容见 CoT 5 层 Gate + 详版 §6，尚无独立文档 | 🟡 待抽出成文 |

### 其他项目

| 项目 | 文件 | 状态 | 时间 | 简历位置 |
|------|------|------|------|---------|
| 百度 · 一键百看投放 Agent | `baidu-agent-migration.md` | 🟡 历史项目，故事可用 | 2025.08 ~ 2026.04 | 实习经历 · 百度 |
| 第四范式 · 航空机务 MaaS | `fourth-paradigm-maas.md` | 🟡 历史项目，故事可用 | 2024.10 ~ 2025.02 | 实习经历 · 第四范式 |
| 第四范式 · 军事 Multi-Agents（Manus 方向） | `fourth-paradigm-multi-agents.md` | 🟡 历史项目，暂不深入 | 2025.03 ~ 2025.08 | 实习经历 · 第四范式 |
| Price-Agent · 商品对比助手 | `price-agent-project-readme.md` + `price-agent-复盘文档.md` | 🟡 自驱项目，故事可用 | 持续维护 | 项目经历 |
| Price-Agent · 评测增强方案 | `price-agent-eval-upgrade.md` | 📋 方案待执行 | 2026-08-21 | 项目经历 |
| Kaggle · 广告欺诈检测 | `kaggle-talkingdata.md` | 🟢 已完成 | — | 项目经历 |

---

## 项目文档结构

每个项目文档包含：

1. **项目概览** — 背景、目标、周期、数据规模
2. **核心工作内容** — 职责、设计决策、量化成果
3. **Technical Anatomy** — 按主题逐步解剖（持续追加）
4. **四视角展开** — PM / 算法 / 工程 / 策略
5. **面试故事** — 30 秒 / 2 分钟 / 5 分钟三层版本
6. **面试官追问准备** — 8-10 个高频追问
7. **知识连接** — 本项目涉及的知识树干节点
8. **待深挖方向** — 还没拆到八层深度的点

---

## 约定

- 一个项目一个 `.md` 文件
- 每次在这个项目上有新工作或新理解 → 追加到这个文件
- Daily 文档通过链接引用项目文档，不重复复制内容
- 面试故事统一索引在 `interview-prep/story-bank.md`，项目文档只写本项目
- 简历经历更新时，同步检查对应项目文档是否过期（如百度 92% 修正）
