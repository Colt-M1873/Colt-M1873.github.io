---
title: "06. 场景详解：Data Science Agent / Kaggle Agent（`rdagent data_science`）"
date: 2026-10-15T10:06:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

> 阅读前提：本篇沿用 [01-概览](/rdagent/overview/) 与 [02-核心抽象层](/rdagent/core-abstractions/) 的词汇（`Scenario`/`Task`/`Workspace`/`Experiment`/`Developer`(Coder/Runner)/`Hypothesis`/`HypothesisGen`/`Hypothesis2Experiment`/`ExpGen`/`Trace`/`LoopBase`/`RDLoop`、CoSTEER 代码进化内循环），不再重复定义。

## 0. 这是什么、为什么复杂

`rdagent data_science --competition <name>` 是 README 中反复强调的"当前旗舰场景"（"top ML engineering agent on MLE-bench"）。它要解决的问题是：**给定一个数据科学/机器学习竞赛（Kaggle 竞赛、MLE-bench 任务，或任何本地整理好的预测类数据集，包括医疗预测场景），自动读懂数据与评价指标、写出一个端到端可运行的训练+推理+提交流水线代码，并通过"测分数 → 改进"的循环不断逼近更高分数**。

相比 [05-Qlib 金融场景](/rdagent/scenario-qlib-finance/)（单一"因子"或"模型"组件、相对固定的任务结构），Data Science 场景的复杂度体现在三层：

1. **任务结构更复杂**：一个完整方案可以被拆成 5 个组件（数据加载/特征工程/模型/集成/工作流），也可以合并成"整体流水线"一个任务（由 `DS_CODER_ON_WHOLE_PIPELINE` 开关控制，当前默认开启）。
2. **Runner 内部也有一层 CoSTEER 内循环**：Qlib 场景里 Runner 一般只是"跑一次拿结果"，而这里的 `DSCoSTEERRunner` 本身继承自 `CoSTEER`，执行失败/有警告时会反复修代码直到跑通或到达 `runner_max_loop`。
3. **提案（Proposal）层做了大量工程化扩展**：支持多分支并行搜索（`Trace` 的 DAG 能力被真正用起来）、起草（draft）/常规提出想法（idea）/多分支合并（merge）三种策略之间的路由、EDA 自动摘要注入、知识库/想法池、假设的"批评-重写"自我修正等。

由于该场景体量巨大（`rdagent/app/data_science` + `rdagent/components/coder/data_science` + `rdagent/scenarios/data_science` + 遗留的 `rdagent/scenarios/kaggle`，合计数十个 Python 文件、上百个竞赛模板），**本文档对核心链路（配置、Scenario、任务结构、PipelineCoSTEER 编码、Runner 执行评分、DSProposalV2ExpGen 主提案逻辑、多分支调度骨架）做了逐行级精读；对一些次要/可选分支（`DSProposalV1ExpGen` 旧版提案、`merge.py` 合并策略细节、`idea_pool.py`/知识库内部实现、`diversity_strategy.py`、各 `Scheduler`/`Selector` 的具体算法、notebook 转换 `share/notebook.py`、MCP 文档搜索 `components/agent/context7`）仅做了结构性略读，标注在对应小节末尾，使用时请以实际代码为准。**

## 1. 场景与配置

### 1.1 入口与两级配置继承

CLI 入口是 `rdagent/app/cli.py` 里的 `data_science` 命令（底层调用 `rdagent/app/data_science/loop.py:main`），一路向下依赖的配置对象是 `DS_RD_SETTING`（`rdagent/app/data_science/conf.py` 的 `DataScienceBasePropSetting`，pydantic-settings，环境变量前缀 **`DS_`**）。

有一个容易踩坑的设计细节：`DataScienceBasePropSetting` **继承自** `rdagent/app/kaggle/conf.py` 的 `KaggleBasePropSetting`（环境变量前缀 `KG_`，服务于下面 1.3 节提到的*遗留* Kaggle 场景）。代码里原话吐槽：

```python
class DataScienceBasePropSetting(KaggleBasePropSetting):
    # TODO: Kaggle Setting should be the subclass of DataScience
    ...
```

也就是说继承方向本来"写反了"——`local_data_path`、`competition`、`template_path`、`if_using_mle_data` 等字段实际定义在 `KaggleBasePropSetting` 里，被 `DataScienceBasePropSetting` 继承后改用 `DS_` 前缀暴露（如 `DS_LOCAL_DATA_PATH`、`DS_IF_USING_MLE_DATA`）。理解这一点，才能看懂为什么 data_science 场景的部分配置字段的"祖籍"在看起来不相关的 `app/kaggle/conf.py` 里。

### 1.2 关键 `DS_*` 环境变量（按用途分组，均来自 `rdagent/app/data_science/conf.py` 实际字段）

- **场景选择**：`DS_SCEN`（字段名 `scen`，默认值 `"rdagent.scenarios.data_science.scen.KaggleScen"`）。可以切到 `"rdagent.scenarios.data_science.scen.DataScienceScen"` 用于非 Kaggle（如本地整理好的医疗/竞赛数据）场景——两者的区别见第 2 节。
- **数据来源**：`DS_LOCAL_DATA_PATH`（竞赛数据根目录）、`DS_IF_USING_MLE_DATA`（是否使用 MLE-bench 标准数据/评测格式）、`DS_SAMPLE_DATA_BY_LLM`（是否按新流程做"调试态小样本数据 + LLM 估算全量耗时"，否则走旧的 `create_debug_data`/`sample.py` 采样方式）、`DS_USE_RAW_DESCRIPTION`、`DS_SHOW_NAN_COLUMNS`。
- **编码策略**：`DS_CODER_ON_WHOLE_PIPELINE`（字段 `coder_on_whole_pipeline`，**默认 `True`**，决定走"整体流水线编码"还是"分组件编码"，见第 3 节）、`DS_SPEC_ENABLED`、`DS_CODER_MAX_LOOP` / `DS_RUNNER_MAX_LOOP`（CoSTEER 内循环最大轮数，默认 10/3）。
- **超时控制**：`DS_DEBUG_TIMEOUT` / `DS_FULL_TIMEOUT`（默认 600s / 3600s）及其"建议值" `*_recommend_timeout`、`DS_ALLOW_LONGER_TIMEOUT` 及一组 `*_longer_timeout_multiplier_upper`/`*_timeout_increase_stage` 字段——用于"连续因超时失败就逐步放宽超时上限"的自适应机制。
- **多分支搜索（multi-trace）**：`DS_MAX_TRACE_NUM`（并行分支数上限）、`DS_TRACE_SCHEDULER`（默认 `RoundRobinScheduler`，还可选 `MCTSScheduler`/`SOTABasedScheduler`）、`DS_SELECTOR_NAME`（默认 `LatestCKPSelector`，决定新实验挂在哪个历史节点后）、`DS_SOTA_EXP_SELECTOR_NAME`（默认 `GlobalSOTASelector`，决定最终提交哪个实验）、`DS_MERGE_HOURS`（临近总时限时触发"多分支合并"阶段）。
- **知识库/想法池**：`DS_ENABLE_KNOWLEDGE_BASE`、`DS_KNOWLEDGE_BASE_PATH`、`DS_IDEA_POOL_JSON_PATH`。
- **假设生成的工程化开关**：`DS_ENABLE_HYPO_CRITIQUE_REWRITE`（假设"批评-重写"自我修正）、`DS_ENABLE_SIMPLE_HYPOTHESIS`、`DS_LLM_SELECT_HYPOTHESIS`（用 LLM 而非加权打分选假设，与 `DS_ENABLE_CROSS_TRACE_DIVERSITY` 互斥，`conf.py` 末尾有一条 `assert` 强制校验）、`DS_ENABLE_DRAFT_BEFORE_FIRST_SOTA`、`DS_ENABLE_PLANNER`。
- **日志归档**：`DS_ENABLE_LOG_ARCHIVE`、`DS_LOG_ARCHIVE_PATH`——跑完每个 loop 后把 log/workspace 打 tar 归档（见 `scenarios/data_science/loop.py:record` 末尾一大段 `backup_folder`/`tar` 逻辑）。

### 1.3 `KaggleScen` 与 `DataScienceScen`（`rdagent/scenarios/data_science/scen/__init__.py`）

两者都在 `scenarios/data_science/scen/__init__.py` 里，`KaggleScen` 是 `DataScienceScen` 的子类：

- **`DataScienceScen.__init__`**：假设数据已经整理在 `{DS_LOCAL_DATA_PATH}/{competition}` 下；若目录不存在直接 `FileNotFoundError`。读取 `description.md`（或 `{competition}.json`）作为原始描述；调用 `describe_data_folder_v2`（`scen/utils.py`）生成"数据目录结构描述"；然后用 LLM 做一次 `_analysis_competition_description()`，把原始描述+数据目录描述喂给 `prompts.yaml:competition_description_template`，解析出结构化字段：`task_type`、`data_type`、`brief_description`、`dataset_description`、`submission_specifications`、`metric_name`、`metric_description`、`metric_direction_guess`,以及"是否需要更长超时"的判断。这些字段共同拼成 `get_scenario_all_desc()`（几乎所有下游 Prompt 都会插入这段文本）。
- **调试数据**：若 `DS_SAMPLE_DATA_BY_LLM=False`，走 `create_debug_data`（`scenarios/data_science/debug/data.py`）或竞赛自带的 `sample.py` 脚本生成小样本调试数据集（`local_data_path/sample/<competition>`）；若为 `True`（默认），不单独采样，而是让生成的代码自身支持 `--debug` 参数跑小样本（见第 3 节 Pipeline 评测逻辑）。
- **`reanalyze_competition_description()`**：供 `DataScienceRDLoop.record` 在"连续多次 Coder/Runner 失败"时调用，重新走一遍上面的分析流程（可能 LLM 第一次理解错了任务类型）。
- **超时计算**：`real_debug_timeout()`/`real_full_timeout()` 会结合 `timeout_increase_count`（每次长时间失败 `+1`）动态放大超时上限，直到 `*_multiplier_upper` 封顶。
- **`KaggleScen`**：在 `__init__` 前先调用 `scenarios/kaggle/kaggle_crawler.py:download_data()`（用 Kaggle API 下载竞赛数据到本地），`_get_description()` 改为用 `crawl_descriptions()` 抓取 Kaggle 网页上的竞赛描述，`_get_direction()` 改为用 `get_metric_direction()`（基于排行榜分数走势猜测指标方向）。**除了数据来源与描述获取方式不同，`KaggleScen` 与 `DataScienceScen` 共享完全相同的下游流程**（同一套 Coder/Runner/Proposal）。
- `DS_SCEN` **默认值就是 `KaggleScen`**，所以不显式设置时，`rdagent data_science --competition <name>` 默认按"这是个 Kaggle 竞赛"的方式去抓取数据/描述；如果是自定义/医疗数据集（没有 Kaggle 页面，数据已经本地放好），需要显式把 `DS_SCEN` 切到 `DataScienceScen`。

（略读）`scen/utils.py:describe_data_folder_v2` 具体怎么遍历文件/采样列名没有逐行核对，只确认了其输出被用作 LLM 输入。

## 2. 遗留的 `rdagent/scenarios/kaggle/` 目录：不是当前 Kaggle Agent 的实现

容易混淆的一点：仓库里还存在一套**独立、更早期**的 Kaggle 实现，位于 `rdagent/scenarios/kaggle/`（`experiment/scenario.py:KGScenario`、`developer/coder.py`、`developer/runner.py`、`proposal/proposal.py:KGHypothesisGen`，以及 `app/kaggle/conf.py`/`app/kaggle/loop.py`，环境变量前缀 `KG_`）。它有自己的一套概念：基于 UCB（置信上界）算法在"特征处理/特征工程/模型特征选择/模型调参"四个动作间做探索-利用权衡（`KGScenario.__init__` 里的 `action_counts`/`reward_estimates`/`confidence_parameter`），还带有针对 ~20 个具名竞赛（`spaceship-titanic`、`playground-series-*`、`digit-recognizer` 等）的模板代码（`experiment/templates/`）、图谱/向量知识库（`knowledge_management/`）。

**但经核实，`rdagent/app/cli.py` 当前并未注册任何 `kaggle` 子命令**（只注册了 `data_science`），`app/kaggle/loop.py` 在整个代码库里也没有被其它入口引用——也就是说这套实现目前**不是**通过 `rdagent` 命令可达的代码路径，更像是保留下来的历史产物/参考实现。真正服务于"Kaggle 竞赛"的，是上面第 1.3 节的 `scenarios/data_science/scen.py:KaggleScen`（复用了 `scenarios/kaggle/kaggle_crawler.py` 的数据下载/描述抓取/排行榜工具函数，以及 `scenarios/kaggle/security.py` 里的 `validate_competition_slug`，用正则 `^[a-z0-9][a-z0-9-]{0,99}$` 校验竞赛名，防止竞赛名被拼进路径/命令时注入）。写文档/读代码时应以 `scenarios/data_science/` 为准，`scenarios/kaggle/` 仅在"数据下载/描述抓取/安全校验"这几个工具函数层面被复用。

（本节未深入阅读 `scenarios/kaggle/developer/*`、`proposal/proposal.py`、`knowledge_management/*` 的具体算法实现，因为确认其不在当前主干调用路径上。）

## 3. 任务与流水线结构

### 3.1 `DSExperiment` 与 `COMPONENT`

`rdagent/scenarios/data_science/experiment/experiment.py` 定义：

```python
COMPONENT = Literal["DataLoadSpec", "FeatureEng", "Model", "Ensemble", "Workflow", "Pipeline"]

class DSExperiment(Experiment[Task, FBWorkspace, FBWorkspace]):
    def __init__(self, pending_tasks_list: list, hypothesis_candidates: list | None = None, ...):
        ...
        self.pending_tasks_list = pending_tasks_list   # list[list[Task]]
        self.hypothesis_candidates = hypothesis_candidates
        self.format_check_result = None
    def is_ready_to_run(self) -> bool:
        return self.experiment_workspace is not None and "main.py" in self.experiment_workspace.file_dict
```

关键点：

- 一个方案被拆成 5 个可能的"组件"（`DataLoadSpec`/`FeatureEng`/`Model`/`Ensemble`/`Workflow`），再加一个代表"整体流水线"的 `Pipeline`。`DSTrace.COMPLETE_ORDER = ("DataLoadSpec", "FeatureEng", "Model", "Ensemble", "Workflow")` 定义了分组件模式下必须按顺序补齐的组件集合。
- `pending_tasks_list` 是**任务列表的列表**（不是单层 `sub_tasks`）：每次 `gen()` 产出的 `DSExperiment` 里可能同时装着"主任务列表"和"工作流更新任务列表"两组，`DataScienceRDLoop.coding` 会依次取出每一组塞进 `exp.sub_tasks`，交给对应的 Coder。
- "是否 ready to run" 的判定很朴素——只看 `experiment_workspace.file_dict` 里有没有 `main.py`，这是整个流水线的统一入口文件名。

### 3.2 两种编码粒度：整体流水线（Pipeline） vs. 分组件（DataLoadSpec/FeatureEng/Model/Ensemble/Workflow）

由 `DS_RD_SETTING.coder_on_whole_pipeline`（即 `DS_CODER_ON_WHOLE_PIPELINE`，**默认 `True`**）二选一：

- **整体流水线模式**（默认）：每次只生成/修改一个 `PipelineTask`（`components/coder/data_science/pipeline/exp.py`，本质就是 `CoSTEERTask` 的空子类，仅用于 `isinstance` 分派），描述里要求 LLM 直接写/改一份完整的 `main.py`（数据读取→特征→模型→集成→产出 `submission.csv`+`scores.csv` 一步到位）。这是 README 所说"top ML engineering agent on MLE-bench"的工作方式：**一次 LLM 调用管整条流水线**，更符合"端到端打磨一份代码"的直觉，也是当前重点优化方向。
- **分组件模式**（`coder_on_whole_pipeline=False`，一般只在教学/对比实验或更早版本的流程里使用）：任务按 `DataLoaderTask`（`raw_data_loader/exp.py`）→`FeatureTask`（`feature/exp.py`）→`ModelTask`（`model/exp.py`）→`EnsembleTask`（`ensemble/exp.py`）→`WorkflowTask`（`workflow/exp.py`，负责把前面几个组件串成 `main.py`）逐步"起草"（见 `proposal/exp_gen/draft/draft.py:DSDraftExpGen`，在 `trace.next_incomplete_component()` 还没返回 `None` 时，先把这 5 个组件依次补全），每个组件对应各自的 CoSTEER 编码器与 spec 文件（`scenarios/kaggle/tpl_ex/*/spec/*.md` 一类的规范文档）。
- `DataScienceRDLoop.coding`（`scenarios/data_science/loop.py`）用 `isinstance(exp.sub_tasks[0], ...)` 对 6 种 Task 类型做分派,分别调用 `self.data_loader_coder` / `feature_coder` / `model_coder` / `ensemble_coder` / `workflow_coder` / `pipeline_coder`（全部是 `CoSTEER` 子类实例，构造时都只传入 `scen`）。

（略读）分组件模式下各 Task 的具体 spec 文件内容（`spec/data_loader.md` 等，分布在 `scenarios/kaggle/tpl_ex/*/spec/` 下的若干模板目录里）与 `workflow/exp.py` 的详细字段未逐行核对。

## 4. 代码生成（Coder）

### 4.1 统一基类：`DSCoSTEER`

`rdagent/components/coder/data_science/share/ds_costeer.py`：

```python
class DSCoSTEER(CoSTEER):
    def get_develop_max_seconds(self) -> int | None:
        return int(self.scen.real_debug_timeout() * self.settings.max_seconds_multiplier)
```

6 个 Coder（`DataLoaderCoSTEER`/`FeatureCoSTEER`/`ModelCoSTEER`/`EnsembleCoSTEER`/`WorkflowCoSTEER`/`PipelineCoSTEER`，分别位于 `components/coder/data_science/{raw_data_loader,feature,model,ensemble,workflow,pipeline}/__init__.py`）都是 `DSCoSTEER` 的薄子类，复用 02 文档里介绍的 `RAGEvoAgent.multistep_evolve` 内循环骨架（`evolve_iter` + `evaluate_iter`），区别主要在各自的 `eval.py`（怎么判定这一步代码是否合格）与 `prompts.yaml`（怎么描述该组件的任务/规范）。

### 4.2 以 `PipelineCoSTEER` 为例看编码-评测循环（`components/coder/data_science/pipeline/eval.py`）

默认模式下真正起作用的是 `PipelineCoSTEEREvaluator.evaluate()`，每一轮编码后的验证流程大致是：

1. 在 Docker 环境（`get_ds_env()`，见第 5 节）里先清空上一轮产物（`get_clear_ws_cmd()`），再执行：
   `strace -e trace=file -f -o trace.log python -m coverage run main.py [--debug]`
   （`--debug` 仅当 `DS_SAMPLE_DATA_BY_LLM=True` 时附加，让生成代码自己用小样本跑一遍，同时用 `strace` 记录文件访问轨迹，`coverage` 记录代码覆盖率）。
2. **防作弊检查**：扫描 `trace.log`，如果代码在调试/训练阶段 `openat` 打开了"样例提交文件"（sample submission），直接判定不合格（避免 LLM 作弊，直接抄 `sample_submission.csv` 里的值）。
3. 检查 `scores.csv` 是否存在、index 是否唯一、是否包含 `ensemble` 这一行、列名（大小写不敏感）是否等于 `scen.metric_name`、是否有 NaN。
4. 若 `DS_SAMPLE_DATA_BY_LLM=True` 且该竞赛配置了评测脚本（见 `test_eval.py`），调用 `TestEvalBase.valid()` 再做一次提交格式校验。
5. 把上面所有 stdout/检查结果拼进 Prompt（`prompts.yaml:pipeline_eval`），LLM 返回结构化的 `PipelineSingleFeedback`（`DSCoderFeedback` 的别名，含 `execution`/`return_checking`/`code`/`final_decision`，以及是否需要去查第三方库文档 `requires_documentation_search`——如果需要，会触发 `components/agent/context7` 的 MCP 文档检索 Agent 把检索结果追加进 `error_message` 供下一轮修复参考，开关是 `DS_ENABLE_MCP_DOCUMENTATION_SEARCH`）。
6. 任何一步格式检查失败都会强制把 `final_decision` 置为 `False`（即便 LLM 自己判断通过）。

`DS_RD_SETTING.coder_max_loop`（默认 10）限制了这个"写代码→跑→看报告→改"的内循环最多跑多少轮。

### 4.3 EDA 自动摘要是如何产生的

生成的 `main.py` 被要求在代码里打印一段被 `=== Start of EDA part === ... === End of EDA part ===` 包裹的探索性数据分析文本；Runner/Coder 的评测器用正则把这段抠出来存成工作区里的 `EDA.md` 文件，随后的 `scen.get_scenario_all_desc(eda_output=...)` 会把它重新注入给下一轮的提案/编码 Prompt，形成"自己写的 EDA 总结反哺后续迭代"的闭环；`HypothesisFeedback.eda_improvement` 字段（02 文档已提及）就是 LLM 在生成反馈时顺带给出的"EDA 还可以怎么改进"的建议，供下一轮任务生成时参考（见 `proposal/exp_gen/proposal.py:task_gen` 里的 `eda_improvement=fb_to_sota_exp.eda_improvement`）。

## 5. 执行与评分（Runner）

### 5.1 执行环境：Docker 配置（`rdagent/utils/env.py` + `components/coder/data_science/conf.py:get_ds_env`）

`get_ds_env(conf_type="kaggle"|"mlebench", ...)` 根据 `DSCoderCoSTEERSettings.env_type`（默认 `"docker"`）选择：

- **`DSDockerConf`**（环境变量前缀 `DS_DOCKER_`）：`build_from_dockerfile=True`，Dockerfile 在 `rdagent/scenarios/kaggle/docker/DS_docker/Dockerfile`（基础镜像是官方 **`gcr.io/kaggle-gpu-images/python:latest`**，额外装了 `curl`/`vim`/`git`/`build-essential`/`strace`），构建出的镜像名 `local_ds:latest`，容器内挂载路径 `/kaggle/workspace`，默认入口 `python main.py`，内存上限 `48g`（注释提到类似 `new-york-city-taxi-fare-prediction` 这种竞赛数据量大，需要这么大内存）。
- **`MLEBDockerConf`**（前缀 `MLEB_DOCKER_`）：Dockerfile 在 `scenarios/kaggle/docker/mle_bench_docker/`，镜像 `local_mle:latest`，用于 MLE-bench 标准化评测（`mlebench grade-sample ...`）。
- 也支持纯 Conda 本地环境（`CondaConf`/`MLECondaConf`，`env_type="conda"`）作为不用 Docker 的退路。
- 真正把竞赛数据目录挂进容器是通过 `extra_volumes={f"{DS_RD_SETTING.local_data_path}/{competition}": <容器内输入路径>}` 实现的（容器内路径定义在 `scenarios/data_science/share.yaml:scen.input_path`）。

另外注意 `rdagent/scenarios/data_science/sing_docker/Dockerfile` 是一个**不同用途**的 Dockerfile——它打的是"运行 RD-Agent 自身 + MLE-bench 工具链 + litellm 代理"的开发/部署环境镜像，不是上面 `DSDockerConf` 执行生成代码所用的沙箱镜像,两者不要混淆。

### 5.2 `DSCoSTEERRunner`（`scenarios/data_science/dev/runner/__init__.py`）

Runner 本身也是一个 `CoSTEER` 子类（这是与 Qlib 场景的显著差异——Qlib 的 Runner 通常只跑一次）：

```python
class DSCoSTEERRunner(CoSTEER):
    def __init__(self, scen, ...):
        eval_l = [DSRunnerEvaluator(scen=scen)]
        if DS_RD_SETTING.enable_model_dump:
            eval_l.append(ModelDumpEvaluator(scen=scen, data_type="full"))
        ...
        super().__init__(..., max_loop=DS_RD_SETTING.runner_max_loop, ...)  # 默认 3
```

`develop(exp)` 会先把 `exp.pending_tasks_list` 备份，临时塞进一个人工构造的 `CoSTEERTask("Debug running solution", ...)`，然后调用父类 `CoSTEER.develop`，也就是走一遍 `evolve → evaluate` 的内循环（`DSRunnerMultiProcessEvolvingStrategy.implement_one_task`：如果上一轮反馈 `acceptable=False` 走 "debugger" 提示词修 bug，否则走 "refine" 提示词做超参调优）。跑完之后：

- 必须存在 `scores.csv`，否则抛 `RunnerError("Metrics file (scores.csv) is not generated")`（`RunnerError` 是 02 文档提到的异常体系的一种，会被 `DataScienceRDLoop.skip_loop_error` 捕获，转成失败反馈而不中断整个大循环）。
- 若 `DS_IF_USING_MLE_DATA=True`，额外读取 `test/mle_submission_format_test.output` 存进 `exp.format_check_result`。

### 5.3 `DSRunnerEvaluator.evaluate()`（`scenarios/data_science/dev/runner/eval.py`）——真正"打分"的地方

关键步骤（在全量数据而非调试样本上跑，超时用 `scen.real_full_timeout()`）：

1. 执行入口固定为 **`python -m coverage run main.py`**（全量数据，非 `--debug`）。
2. 同样用正则抠出 `EDA.md`，并把 `stdout.txt`（根据 `DS_Runner_CoSTEER_dump_stdout_type` 决定存全量还是裁剪版）注入工作区，便于调试复盘。
3. **分数文件校验**：`scores.csv` 必须有 `ensemble` 这一行；若走分组件模式还会校验 index 与"实际生成了哪些 `model_*.py` 文件"一致；列名需与 `scen.metric_name` 一致（大小写不敏感）。
4. **MLE-bench 提交格式校验**：若 `DS_IF_USING_MLE_DATA=True`，调用 `get_test_eval()` 得到的 `MLETestEval.valid()`，本质是在 `mlebench` conda/Docker 环境里跑一段格式校验脚本。
5. **超参调优开关的四个门控条件**（`c1`~`c4`，代码里都有编号注释）：是否是该任务第一次评测（`only_first_loop_enable_hyperparameter_tuning`）、本次运行耗时占超时上限的比例是否低于阈值、是否只在"多分支合并阶段"才允许调参、全局剩余时间占比是否仍然充足——四者都满足才会把"是否建议超参调优"这件事交给 LLM 判断（避免在时间紧张时还浪费一轮在调参上）。
6. LLM（`prompts.yaml:DSCoSTEER_eval`）返回 `DSRunnerFeedback`（含 `acceptable`、`hyperparameter_tuning_decision`、`hyperparameter_tuning_suggestion`、`score`），`final_decision = acceptable and not hyperparameter_tuning_decision`（即"代码可接受，且不需要再做一轮调参"才算这次 Runner 内循环真正结束）。
7. **分组件模式下的覆盖率联动清理**：用 `coverage json` 产出的覆盖率报告判断"哪些脚本文件实际被 `main.py` 引用过"——如果 `model_*.py` 一个都没被用到，直接判不合格；如果 `load_data.py`/`feature.py`/`ensemble.py` 这类必需文件没被覆盖到，也判不合格；其余"写了但没用到"的脚本会被自动从工作区删除（避免历史脚本堆积污染）。

最终分数怎么和"排行榜/竞赛指标"对齐：`scores.csv` 里 `ensemble` 行、列名为 `scen.metric_name` 的那个单元格数值，就是这次实验的 `exp.result`（一个 `DataFrame`），后续 `DSExperiment2Feedback`（见第 7 节）会拿它与 `trace.sota_experiment().result` 比较（结合 `scen.metric_direction` 判断"分数更高/更低是否更好"）决定是否 `decision=True`（即是否取代当前 SOTA）。

（略读）`get_test_eval()` 支持的"非 MLE-bench"版本 `TestEval`（直接读取 `grade.py`/`submission_test.csv`）用于用户自备评测脚本的场景，流程类似但未逐行比对细节；`ModelDumpEvaluator`（`DS_ENABLE_MODEL_DUMP`，模型权重/产物导出检查）与 `notebook` 转换功能（`DS_ENABLE_NOTEBOOK_CONVERSION`）仅确认存在，未深入阅读实现。

## 6. 提出新想法的策略（Proposal / ExpGen）

这是该场景里工程量最大的一块，由 `rdagent/scenarios/data_science/proposal/exp_gen/` 下十余个文件构成。该目录自带一份 `README.md`，概括了设计思路：把"优化策略"分为 `draft`（起草全新方案）、`idea`（提出改进点）、`merge`（合并多条分支方案）三类，由一个 `router`（当前实现是 `ParallelMultiTraceExpGen`）在它们之间路由；另外提供 `select` 工具集：`expand`（选"从哪个历史节点继续扩展"）与 `submit`（选"最终提交哪个实验"）。

### 6.1 顶层路由：`ParallelMultiTraceExpGen`（`proposal/exp_gen/router/__init__.py`，默认 `DS_HYPOTHESIS_GEN`）

这是 `DataScienceRDLoop.__init__` 里真正实例化的 `self.exp_gen`（配置项 `hypothesis_gen`，默认值就是它）。它本身不直接生成实验，而是在 `async_gen(trace, loop)` 里做调度：

1. **限流等待**：轮询 `loop.get_unfinished_loop_cnt(loop.loop_idx) < RD_AGENT_SETTINGS.get_max_parallel()`，没有空闲并行槽位就 `asyncio.sleep(1)` 继续等。
2. **选一个分支继续扩展**：
   - 若还没到总时限的"合并窗口"（`DS_MERGE_HOURS`），调用 `self.trace_scheduler.next(trace)`（`DS_TRACE_SCHEDULER`，默认 `RoundRobinScheduler`，还有 `MCTSScheduler`/`SOTABasedScheduler` 等实现，位于 `trace_scheduler.py`）选一个 `local_selection`（即 `Trace` 的 DAG 节点游标，可以是已有分支的叶子节点，也可以是 `trace.NEW_ROOT` 表示"另起一棵新树"）。
   - 若已进入合并窗口（剩余时间 `< DS_MERGE_HOURS`），改为优先挑选"包含当前最佳提交分支"的叶子节点，为"多分支合并"做准备。
3. **三选一生成具体实验**：
   - 如果该分支还没有任何 SOTA、且 `DS_ENABLE_DRAFT_BEFORE_FIRST_SOTA=True` → 用 `DSDraftV2ExpGen`（起草，`draft/draft.py`）。
   - 如果已进入合并窗口且已有 ≥2 条分支 → 用 `ExpGen2Hypothesis`（`merge.py`，把多条分支的优点合并成一个新假设/实验），此时还会临时把 `coding_fail_reanalyze_threshold`/`consecutive_errors` 设成极大值（10 万）避免合并阶段被"连续失败重置 Trace"的保护逻辑打断。
   - 否则 → 用默认的 `DSProposalV2ExpGen`（即 `DataScienceRDLoop.default_exp_gen`，见下）常规地"提出改进想法"。
4. 把选中的 `local_selection` 和可选的 `ExperimentPlan`（`DS_ENABLE_PLANNER` 开启时由 `DSExpPlannerHandCraft` 产出）写入生成的 `exp`，再通过 `trace.register_uncommitted_exp(exp, loop.loop_idx)` 登记（供其它并行分支的"同辈实验"查询，见下方多样性注入）后返回。

### 6.2 默认提案核心：`DSProposalV2ExpGen`（`proposal/exp_gen/proposal.py`）

`gen(trace, plan)` 的主干流程：

1. **识别问题（identify_problem）**：区分两类问题来源——
   - `identify_scenario_problem`：对比当前 SOTA 实现与"场景/数据本身"的契合度（SOTA 对齐分析、差距识别、领域-实现一致性检查等四个分析维度，结构化为 `ScenarioChallenges`），产出若干 `SCENARIO_PROBLEM`。
   - `identify_feedback_problem`：从历史反馈/实现复盘/实验轨迹中挖掘问题（结构化为 `TraceChallenges`），产出若干 `FEEDBACK_PROBLEM`。
   - 两者的混合比例由 `self.scen_prob_multiplier`（随着当前子分支里"成功实验数×3 + 失败实验数×2"的加权计数衰减,从 3 线性降到 0）动态调节——**分支越"年轻"（尝试次数越少），越偏向从场景本身找问题；分支越"成熟"，越偏向从历史反馈里找问题**。
2. **（可选）从想法池采样**：`DS_ENABLE_KNOWLEDGE_BASE=True` 时，`trace.knowledge_base.sample_ideas()` 会把外部/历史积累的"想法"混入问题列表。
3. **生成假设（hypothesis_gen）**：把所有候选问题丢给 LLM，一次性产出一组 `HypothesisDetail`（每条假设都带 `alignment`/`impact`/`novelty`/`feasibility`/`risk_reward_balance` 五维打分，`HypothesisEvaluationReasoningScore`）。
4. **（可选）批评-重写**：`DS_ENABLE_HYPO_CRITIQUE_REWRITE=True` 时，先用 `hypothesis_critique` 让 LLM 挑刺，再用 `hypothesis_rewrite` 依据挑刺结果重写假设（失败则静默回退到原假设，不阻塞主流程）。
5. **选择一个假设**：
   - 默认路径 `hypothesis_rank`：用加权打分（`alignment*0.2 + impact*0.4 + novelty*0.2 + feasibility*0.1 + risk_reward_balance*0.1`）取 Top-5，再按"是否来自想法池 / 问题标签是 SCENARIO 还是 FEEDBACK"做加权随机抽签（用 `md5_hash` 对分数字符串做种子，保证可复现）从 Top-5 里选一个——**不是单纯贪心取最高分，而是在高分候选里按权重随机，兼顾探索与利用**。
   - 可选路径 `DS_LLM_SELECT_HYPOTHESIS=True`：改为直接让 LLM 在候选假设里选一个（同时会用 embedding 余弦相似度 + 历史分数做"采样历史相似案例"辅助 LLM 判断，`_prob_dis_torch`），与"跨分支多样性注入"互斥。
6. **生成任务（task_gen）**：把选中的假设转成具体 `Task`（整体流水线模式下固定是 `PipelineTask`；要求 LLM 先给出 `CodingSketch`——包含"当前 main.py 现状总结/具体改动点/新架构骨架/一段 Markdown 格式的实现步骤叙述/预计要用的第三方包列表"——再落地成任务描述文本）。若涉及非 `Workflow` 组件，还会让 LLM 顺带给出 `workflow_update`,生成一个追加的 `WorkflowTask`。
7. 新生成的 `DSExperiment` 会把 SOTA 工作区的代码通过 `inject_code_from_file_dict` 继承过来（**增量开发**：不是从零开始写,而是在上一版最优代码基础上改）。

### 6.3 跨分支多样性注入与"同辈实验"

`DSTrace.should_inject_diversity()` / `get_sibling_exps()`：在多分支并行搜索时，生成新假设前会查询"同一父节点下其它分支正在尝试/已经尝试过的假设"（包括还没写入 `hist` 的 `uncommitted_experiments`），作为 Prompt 的一部分（`sibling_hypotheses`）告诉 LLM"不要重复别人已经在试的方向"，由 `DS_DIVERSITY_INJECTION_STRATEGY`（默认 `InjectUntilSOTAGainedStrategy`）决定什么时候触发。

### 6.4 多分支搜索的收尾：Checkpoint 选择与 SOTA 提交选择

- `DS_SELECTOR_NAME`（默认 `LatestCKPSelector`，`select/expand.py`）：每轮 `DataScienceRDLoop.direct_exp_gen` 开头调用 `ckp_selector.get_selection(trace)`,决定（在非并行场景下）这一轮该从 `Trace` 的哪个节点继续长出新实验。
- `DS_SOTA_EXP_SELECTOR_NAME`（默认 `GlobalSOTASelector`，`select/submit.py`）：每轮 `record` 步末尾调用 `get_sota_exp_to_submit(trace)`,决定"如果现在要停止，应该提交哪个实验"。`GlobalSOTASelector` 的实现很直接——`trace.sota_experiment(search_type="all")`，即全历史里最后一个 `decision=True` 的实验；文件里还定义了更复杂的 `AutoSOTAexpSelector`（结合 `sota_count_window`/`sota_count_threshold`/`max_sota_retrieved_num` 等配置，用 LLM 在多个候选 SOTA 间做二次挑选）供多分支场景使用。

（略读）`merge.py:ExpGen2Hypothesis` 的具体合并提示词与实现、`idea_pool.py`（`DSKnowledgeBase`/`DSIdea` 想法池的存储格式与采样算法）、`diversity_strategy.py`、`planner/__init__.py`（`DSExpPlannerHandCraft`）、`trace_scheduler.py` 里 `MCTSScheduler`/`SOTABasedScheduler`/`ProbabilisticScheduler` 的具体公式，均只读了类/函数签名与所在上下文，未逐行核对算法细节。另外还有一条**已废弃但代码仍在**的旧版提案 `DSProposalV1ExpGen`（同文件内，组件选择与假设生成分两步走，不支持多分支/批评重写等新特性），当前默认不使用。

## 7. 主循环如何串起以上所有部分：`DataScienceRDLoop`

`rdagent/scenarios/data_science/loop.py:DataScienceRDLoop` 是 `RDLoop` 的子类，但几乎每一步都做了针对本场景的重写（而不是像 Qlib 场景那样只声明 `skip_loop_error` 就完事）：

- `skip_loop_error = (CoderError, RunnerError)`，`withdraw_loop_error = (PolicyError,)`，`skip_loop_error_stepname = "record"`（出错直接跳到 `record`，而不是常规的 `feedback`——因为 `record` 里自带了"异常情况下如何构造失败反馈"的分支逻辑，见下）。
- `direct_exp_gen`：先用 `ckp_selector` 定位 `trace.current_selection`，再 `await self.exp_gen.async_gen(trace, self)`（即第 6.1 节的 `ParallelMultiTraceExpGen`），最后过一遍 `interactor.interact()`（默认 `SkipInteractor`，即不做人工介入；02 文档提到的 human-in-the-loop 接口在这里的具体落地）。
- `coding`：按 `pending_tasks_list` 逐组分派给 6 种 Coder 之一（见 3.2 节）。
- `running`：仅当 `exp.is_ready_to_run()`（存在 `main.py`）才调用 `self.runner.develop(exp)`；`DS_ENABLE_DOC_DEV=True` 时额外跑一个 `DocDev`（`components/coder/data_science/share/doc.py`，为代码生成文档，未深入阅读）。
- `feedback`：仅当"分组件模式下组件已全部补齐"或"整体流水线模式"时才调用 `summarizer.generate_feedback`（见下 `DSExperiment2Feedback`）；否则（分组件模式的起草阶段）直接给一个 `decision=True` 的占位反馈，表示"这一步只是把某个组件补上，不是在验证假设"。
- `record`：
  - 正常情况：把 `(exp, feedback)` 按 `local_selection` 挂到 `Trace` DAG 的正确父节点下。
  - 异常情况：根据异常是 `CoderError`/`RunnerError` 决定用 `direct_exp_gen` 还是 `coding` 步的产物记录；若命中"连续失败且是超时类 `CoderError`"且 `DS_ALLOW_LONGER_TIMEOUT=True`，调用 `scen.increase_timeout()`；若整条 Trace 还没有任何 SOTA 且连续失败次数达到 `coding_fail_reanalyze_threshold`（整体流水线模式）或 `consecutive_errors`（分组件模式），分别触发"重新分析竞赛描述"（`scen.reanalyze_competition_description()`）或"**整条 Trace 推倒重来**"（新建一个空 `DSTrace`，并调用 `exp_gen.reset()` 清空调度器状态）——这是一种"卡住了就换个起点"的保护机制。
  - 末尾统一调用 `sota_exp_selector.get_sota_exp_to_submit(trace)` 更新"当前应提交的最优实验"，并在 `DS_ENABLE_LOG_ARCHIVE=True` 时把本轮 log/workspace 打 tar 归档、清理工作区里的大文件（只保留 `.py`/`.md`/`.csv`）。
- 重写了 `_check_exit_conditions_on_step`：跳过对 `running`/`feedback` 步骤的超时检查（因为这两步本来就"预期很快结束"）。
- 重写了 `load`/`dump`：知识库对象体积大，`dump` 前临时摘掉 `trace.knowledge_base`，`load` 后重新挂回去，避免每次会话快照都把整个知识库序列化一遍。

### `DSExperiment2Feedback`（`scenarios/data_science/dev/feedback.py`）

对应 02 文档里的 `Experiment2Feedback.generate_feedback`。拿当前实验结果与 `trace.sota_experiment()` 的 `ensemble` 分数对比（结合 `scen.metric_direction` 判断方向），生成代码 diff（`generate_diff_from_dict`），连同 EDA 输出一起喂给 LLM（`prompts.yaml:exp_feedback`），解析出 `HypothesisFeedback` 的各字段；特别地，若 LLM 判断"评测代码与任务本身不对齐"（`Evaluation Aligned With Task == no`），会强制把 `exp.result` 置空并把 `decision` 设为 `False`（防止一个"跑出了分数但评测方式其实有问题"的实验被误判为 SOTA）。若启用知识库，还会把这次的假设/任务/问题标签打包成 `DSIdea` 存入 `trace.knowledge_base`。

## 8. 与其他模块的关系

- **复用 `rdagent/core` 的全部骨架**：`Scenario`/`Task`/`FBWorkspace`/`Experiment`/`Developer`/`Hypothesis`/`Trace`/`LoopBase` 的语义与 02 文档完全一致；本场景的"特殊性"几乎全部体现在**子类重写**与**配置项**上，没有突破核心抽象。
- **复用 `rdagent/components/coder/CoSTEER` 的代码进化内循环**：6 个 Data Science Coder + 1 个 Runner（共 7 处）全部基于 `CoSTEER`/`MultiProcessEvolvingStrategy`/`CoSTEERMultiEvaluator` 搭建，是全项目里对 CoSTEER 思想用得最重的场景（Runner 把它用在了"修运行期 bug/调超参"而不仅是"写初版代码"上，这点与 Qlib 场景的用法不同）。
- **复用 `rdagent/utils/env.py` 的 Docker 执行环境抽象**：`DockerConf`/`DockerEnv`/`CondaConf`/`LocalEnv` 是框架级基础设施（见文档08），本场景只是提供了 `DSDockerConf`/`MLEBDockerConf` 两份具体配置。
- **与 `rdagent/scenarios/kaggle/` 的关系**：如第 2 节所述，仅复用其 `kaggle_crawler.py`（数据下载/描述抓取/排行榜）与 `security.py`（竞赛名校验）两个工具模块；该目录下的 `KGScenario`/`KG*CoSTEER`/UCB 探索-利用策略/图谱知识库是一套未被当前 CLI 暴露的历史实现，与当前 Kaggle Agent 的实际工作方式无关。
- **与日志/UI 模块的关系**（见文档08）：`rdagent ui --data-science` 专门支持回放本场景的详细 Trace（包括多分支 DAG、EDA、各阶段 stdout），是目前唯一有定制可视化支持的场景，侧面印证了它在项目里的"旗舰"地位。
- **与 05（Qlib 金融场景）的对比**：两者共享同一套 `RDLoop`/CoSTEER 骨架，但 Qlib 场景的任务结构更扁平（因子/模型二选一，不存在"整体流水线 vs 分组件"的二元切换）、Trace 基本是单链（没有大规模多分支并行搜索与合并策略）、Runner 不内嵌 CoSTEER 内循环——这些都是 Data Science 场景为了应对"任意开放式竞赛"这种更弱结构化问题而额外发展出的工程机制。

## 附：本文档的精读/略读范围说明

- **精读（逐行核对过逻辑）**：`app/data_science/conf.py`、`app/data_science/loop.py`、`app/kaggle/conf.py`、`scenarios/data_science/loop.py`、`scenarios/data_science/scen/__init__.py`、`scenarios/data_science/experiment/experiment.py`、`scenarios/data_science/proposal/exp_gen/base.py`、`scenarios/data_science/proposal/exp_gen/proposal.py`（`DSProposalV2ExpGen` 全部方法）、`scenarios/data_science/proposal/exp_gen/router/__init__.py`、`scenarios/data_science/dev/runner/__init__.py`、`scenarios/data_science/dev/runner/eval.py`、`scenarios/data_science/dev/feedback.py`、`scenarios/data_science/test_eval.py`、`components/coder/data_science/conf.py`、`components/coder/data_science/share/ds_costeer.py`、`components/coder/data_science/pipeline/exp.py`、`components/coder/data_science/pipeline/eval.py`、`components/coder/data_science/raw_data_loader/exp.py`、`scenarios/kaggle/experiment/scenario.py`（用于对比确认差异）、`scenarios/kaggle/kaggle_crawler.py`（函数签名级）、`scenarios/kaggle/security.py`、`utils/env.py` 中 `DSDockerConf`/`MLEBDockerConf`/`CondaConf`/`MLECondaConf` 定义、两份 Dockerfile。
- **略读（仅看结构/签名，未逐行核对算法）**：`proposal/exp_gen/draft/draft.py`、`proposal/exp_gen/merge.py`、`proposal/exp_gen/idea_pool.py`、`proposal/exp_gen/diversity_strategy.py`、`proposal/exp_gen/planner/__init__.py`、`proposal/exp_gen/trace_scheduler.py`（除 `TraceScheduler` 抽象基类外，各具体 Scheduler 算法）、`proposal/exp_gen/select/expand.py`、`proposal/exp_gen/select/submit.py`（除 `GlobalSOTASelector` 外）、`components/coder/data_science/{feature,model,ensemble,workflow}` 四个组件的 `eval.py`/`prompts.yaml`（假定与 `pipeline/eval.py` 思路一致，未逐一比对差异）、`components/coder/data_science/share/{doc.py,notebook.py}`、`components/agent/context7`（MCP 文档搜索）、`scenarios/kaggle/` 下除上述文件外的全部内容（`developer/*`、`proposal/proposal.py`、`knowledge_management/*`、近 20 个竞赛模板目录）、`scenarios/data_science/scen/utils.py:describe_data_folder_v2` 的具体实现、`scenarios/data_science/debug/data.py` 的采样算法。
