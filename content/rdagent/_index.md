---
title: "RD-Agent 源码解读"
date: 2026-10-15T10:00:00+08:00
draft: false
ShowToc: true
TocOpen: false
---

本文档集面向希望理解 [microsoft/RD-Agent](https://github.com/microsoft/RD-Agent) 代码库原理与结构的读者，从整体设计思想逐步深入到各个子模块的实现细节。建议按顺序阅读。

| 序号 | 文档 | 内容 |
| --- | --- | --- |
| 01 | [项目概览与整体原理](/rdagent/overview/) | RD-Agent 是什么、R&D 大循环的思想、目录结构总览、CLI 入口、配置与监控方式 |
| 02 | [核心抽象层（rdagent/core）](/rdagent/core-abstractions/) | Scenario / Task / Workspace / Experiment / Developer / Hypothesis / Trace / LoopBase 等基础类，以及它们如何拼成一轮“提出想法→实现→验证→反馈→记录”的循环 |
| 03 | [研发组件：Coder / Runner / Proposal / Workflow](/rdagent/components-development/) | `rdagent/components` 中负责“把想法变成代码”“把代码跑起来”“产生新想法”的核心组件，以及 CoSTEER 式的代码进化机制 |
| 04 | [支撑组件：知识库 / Benchmark / Loader / 文档阅读](/rdagent/components-support/) | 知识管理、基准评测、任务加载器、论文/研报解析等辅助组件 |
| 05 | [场景一：Qlib 量化金融](/rdagent/scenario-qlib-finance/) | 因子挖掘、模型挖掘、因子-模型联合优化（RD-Agent(Q)）、财报因子抽取 |
| 06 | [场景二：Data Science / Kaggle](/rdagent/scenario-data-science-kaggle/) | 通用数据科学智能体与 Kaggle 竞赛智能体，是当前项目主力发展方向 |
| 07 | [场景三：通用模型复现 / 强化学习 / LLM 微调](/rdagent/scenario-others/) | General Model（论文复现）、RL（Agent² RL-Bench）、FT-Agent（LLM 微调） |
| 08 | [基础设施：LLM 后端 / 日志与追踪 / 运行环境](/rdagent/infrastructure/) | `oai`（统一 LLM 调用层）、`log`（追踪、Streamlit UI、Web UI）、`utils/env`（Docker 执行环境）等 |

## 阅读建议

- 如果只想快速建立整体概念，读 01 + 02 即可，这两篇覆盖了项目的设计哲学与骨架代码。
- 如果要深入某个具体场景（比如量化因子挖掘），建议先读 02，再读对应场景文档；场景文档会大量引用 02 中定义的术语（`Scenario`、`Hypothesis`、`Experiment`、`Developer` 等），不重复解释。
- 文档中引用的文件路径均相对 `RD-Agent/` 目录（即仓库根目录）。
