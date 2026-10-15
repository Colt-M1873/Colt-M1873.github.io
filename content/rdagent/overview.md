---
title: "01. 项目概览与整体原理"
date: 2026-10-15T10:01:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

## 1. RD-Agent 要解决什么问题

RD-Agent（R&D-Agent）是微软开源的一套 LLM Agent 框架，目标是**自动化数据驱动的工业研发（R&D）流程**。项目名字里的 "R&D" 被拆成两个能力：

- **R（Research / 想法提出）**：像数据科学家一样，观察数据/场景、提出新的假设（Hypothesis），比如"换一种模型结构可能效果更好""加入这个因子可能提升收益"。
- **D（Development / 想法实现）**：把抽象的想法转成可运行的代码（因子、模型、特征工程脚本等），运行它，拿到真实的数值反馈（而不是让 LLM 自己判断好坏）。

项目的核心论点是：R 和 D 应该**循环迭代、互相促进**——想法要靠真实执行结果验证，执行结果又会启发下一个想法，类似人类科研专家的成长过程。这个闭环在代码里体现为一个可以无限跑下去的"大循环"（下面称为 **R&D Loop**）。

## 2. R&D Loop：贯穿全项目的核心循环

不管是量化因子挖掘、Kaggle 竞赛，还是论文复现、LLM 微调，所有场景最终都会落到同一种循环结构上（具体实现在 `rdagent/utils/workflow/loop.py` 的 `LoopBase` 和 `rdagent/components/workflow/rd_loop.py` 的 `RDLoop`，详见 [02-核心抽象层](/rdagent/core-abstractions/)）：

```
            ┌─────────────────────────────────────────────┐
            ▼                                               │
  ① 提出假设(Hypothesis)                                    │
            │                                               │
            ▼                                               │
  ② 转化为实验(Experiment / Task)                            │
            │                                               │
            ▼                                               │
  ③ 编码实现(Coder: Developer.develop)  ← 内部可能有"CoSTEER"式多轮代码进化小循环
            │                                               │
            ▼                                               │
  ④ 执行验证(Runner: Developer.develop) ← 在隔离环境(Docker)中真实跑代码
            │                                               │
            ▼                                               │
  ⑤ 生成反馈(Experiment2Feedback → HypothesisFeedback)        │
            │                                               │
            ▼                                               │
  ⑥ 记录到历史(Trace，一个支持分支的 DAG)──────────────────────┘
```

每一轮循环都会把"假设 + 实验 + 反馈"三元组追加到 `Trace` 里，下一轮的假设生成会参考这个历史（哪些做法被验证有效/无效），从而让 Agent 越跑越"懂行"。这个循环被设计为**可中断、可持久化、可并行跑多条**（session 用签名 pickle 存盘，可以从任意一步继续跑；也支持同时并行推进多个假设分支）。

## 3. 两层循环：R&D 大循环 与 CoSTEER 代码进化小循环

实际实现中有两层粒度的循环：

1. **外层：R&D Loop**（上图）——粒度是"一个假设/一次实验"，由 `Trace` 记录历史。
2. **内层：代码进化循环（CoSTEER 思想）**——粒度是"一段代码的多次修订"。因为 LLM 一次生成的代码往往跑不通或效果不佳，`rdagent/core/evolving_framework.py` + `evolving_agent.py` 提供了一套"evolve → evaluate → 根据反馈再 evolve"的内循环（`RAGEvoAgent.multistep_evolve`），并可以借助一个知识库（RAG）记住"之前这样写报错，应该怎么改"。这套机制被 `rdagent/components/coder` 广泛复用，用来实现一个任务从"写不出代码"到"代码能跑通且符合要求"的收敛过程。

详见 [03-研发组件](/rdagent/components-development/)。

## 4. 应用场景矩阵

RD-Agent 是一个框架，具体落地到不同领域时，只需要实现该领域的 `Scenario`（场景描述）+ 若干 `Developer`/`ExpGen` 子类，剩下的循环骨架复用。目前已经落地的场景：

| 领域 | 场景 | 说明 | 对应目录 |
| --- | --- | --- | --- |
| 💹 量化金融 | 因子挖掘 (`fin_factor`) | 基于 [Qlib](https://github.com/microsoft/qlib) 不断提出新因子并验证 | `scenarios/qlib`, `app/qlib_rd_loop/factor.py` |
| 💹 量化金融 | 模型挖掘 (`fin_model`) | 不断尝试新模型结构 | 同上, `model.py` |
| 💹 量化金融 | 因子-模型联合优化 (`fin_quant`, 即 RD-Agent(Q)) | 交替优化因子库与模型 | 同上, `quant.py` |
| 💹 量化金融 | 研报因子抽取 (`fin_factor_report`) | 从财报 PDF 中抽取因子公式并实现 | 同上, `factor_from_report.py` |
| 🩺 医疗 / 🤖 通用数据科学 | Data Science Agent (`data_science`) | 面向通用数据科学/医疗预测任务的全流程智能体 | `scenarios/data_science`, `app/data_science` |
| 🏭 通用 | Kaggle Agent (`data_science --competition ...`) | 自动调参 + 特征工程打 Kaggle 比赛 | `scenarios/kaggle`, 复用 data_science 入口 |
| 🏭 通用 | General Model Copilot (`general_model`) | 自动读论文/研报并复现模型结构 | `scenarios/general_model`, `app/general_model` |
| 🧪 研究 | Agent² RL-Bench (`rl`) | 评测 Agent 自主构建强化学习后训练流程的能力 | `scenarios/rl`, `app/rl` |
| 🧪 研究 | FT-Agent LLM 微调 (`llm_finetune`) | 自主完成 LLM 微调全流程（数据处理/训练/评测/反馈迭代） | `scenarios/finetune`, `app/finetune` |

详见 [05](/rdagent/scenario-qlib-finance/) / [06](/rdagent/scenario-data-science-kaggle/) / [07](/rdagent/scenario-others/)。

## 5. 顶层目录结构

```
RD-Agent/
├── rdagent/
│   ├── core/            # 框架级抽象基类（与具体场景无关），见文档02
│   ├── components/      # 可复用的研发组件：coder/runner/proposal/workflow/knowledge_management/...，见文档03、04
│   ├── scenarios/        # 各具体业务场景的 Scenario 实现 + 场景专属的 Coder/Runner/ExpGen，见文档05-07
│   ├── app/              # 每个场景的 CLI 入口（main 函数）与具体 Loop 子类，见文档05-07及cli.py
│   ├── oai/               # LLM 调用的统一后端（LiteLLM 封装、缓存、Embedding），见文档08
│   ├── log/               # 全链路追踪日志、Streamlit UI、Flask Web UI 后端，见文档08
│   └── utils/             # 通用工具：workflow/loop.py（循环引擎）、env.py（Docker执行环境）等，见文档02、08
├── web/                 # Web UI 前端（React，配合 rdagent server_ui 使用）
├── test/                # 单测
├── docs/                # readthedocs 官方文档源（Sphinx），本 documentation/ 是面向仓库使用者的补充说明
└── pyproject.toml        # 定义 CLI 入口 `rdagent = "rdagent.app.cli:app"`
```

## 6. 命令行入口

安装后统一通过 `rdagent` 命令驱动（由 [typer](https://typer.tiangolo.com/) 构建，定义在 `rdagent/app/cli.py`）：

| 命令 | 作用 |
| --- | --- |
| `rdagent fin_factor` / `fin_model` / `fin_quant` / `fin_factor_report` | 启动对应的 Qlib 量化场景 |
| `rdagent data_science --competition <name>` | 启动数据科学 / Kaggle 场景 |
| `rdagent general_model <paper_url_or_path>` | 启动论文/研报模型复现 |
| `rdagent llm_finetune --base-model <model>` | 启动 LLM 微调场景 |
| `rdagent ui` / `rdagent server_ui` | 启动 Streamlit 日志查看器 / Flask 实时 Web UI 后端 |
| `rdagent health_check` | 自检 Docker、端口占用、环境变量配置是否正常 |
| `rdagent grade_summary <log_folder>` | 汇总 MLE-bench 打分结果 |

每个场景的 `main()`（如 `rdagent/app/qlib_rd_loop/model.py:main`）都支持：
- `path`：从某个 session 快照继续跑（断点续跑，依赖 `LoopBase.load`，见文档02）。
- `step_n` / `loop_n` / `all_duration`：控制跑多少步/多少轮/多长时间后自动停止。

## 7. 配置与运行环境

- 配置统一通过 `.env` 文件 + [pydantic-settings](https://docs.pydantic.dev/latest/concepts/pydantic_settings/) 管理：框架级配置是 `rdagent/core/conf.py` 的 `RD_AGENT_SETTINGS`（工作区路径、并行度 `step_semaphore`、缓存开关等），LLM 相关配置是 `rdagent/oai/llm_conf.py` 的 `LLM_SETTINGS`，各场景还有自己的 `conf.py`（如 `DS_*` 系列环境变量控制 data_science 场景）。
- 代码的实际执行（跑因子脚本、跑模型训练等）默认在 **Docker 容器**里进行，以保证可重复、隔离、不污染主机环境；因此文档开头强调"RD-Agent 目前只支持 Linux，且需要本机能无 sudo 跑 docker"。执行环境抽象见 `rdagent/utils/env.py`（文档08 详细介绍）。
- LLM 调用默认通过 **LiteLLM** 统一后端，因此理论上可以接入 OpenAI / Azure OpenAI / DeepSeek / 任何 LiteLLM 支持的供应商，只需配置 `.env` 中的 `CHAT_MODEL` / `EMBEDDING_MODEL` 等变量。

## 8. 监控与可观测性

每一轮 Loop 的每一步都会通过 `rdagent/log` 写入结构化日志/追踪对象（prompt、LLM 回复、生成的代码、执行 stdout、反馈等），可以用两种方式查看：

- **Streamlit UI**（`rdagent ui`）：离线查看某个 log 目录下的完整 trace，支持回放 data_science 场景的详细过程（`--data-science`）。
- **Web UI**（`rdagent server_ui` + `web/` 前端）：Flask 后端 + 实时 API，支持一边运行一边在浏览器里看 trace，目前暂不支持 data_science 场景的全部细节。Web UI 默认只监听本地地址，且强制要求配置鉴权 token（`UI_SERVER_AUTH_TOKEN`），因为它暴露的是"能触发代码生成与执行"的操作面——这是一个需要认真对待的安全边界，详见 README 中"Web UI security and remote access"章节与文档08。

## 9. 后续阅读

想理解"循环骨架"本身是怎么写的 → [02. 核心抽象层](/rdagent/core-abstractions/)。
想理解"一个假设怎么变成代码、代码怎么被反复打磨到能跑通" → [03. 研发组件](/rdagent/components-development/)。
想看某个具体场景的落地细节 → [05](/rdagent/scenario-qlib-finance/) / [06](/rdagent/scenario-data-science-kaggle/) / [07](/rdagent/scenario-others/)。
