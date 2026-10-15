---
title: "05. 量化金融场景（`rdagent/scenarios/qlib`）"
date: 2026-10-15T10:05:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

量化金融场景是 RD-Agent 的"旗舰"场景——项目最早就是从这里出发的，围绕开源量化投资平台 [Qlib](https://github.com/microsoft/qlib) 构建。它把 02 篇介绍的通用骨架（`Scenario` / `Task` / `Experiment` / `Developer(Coder/Runner)` / `HypothesisGen` / `Hypothesis2Experiment` / `Trace` / `RDLoop`）具体实例化成了"挖因子"和"挖模型"两件事，并在此基础上派生出"因子-模型联合优化"和"从研报抽因子"两个变体。四个 CLI 入口都定义在 `rdagent/app/qlib_rd_loop/` 下：

| 入口 | 文件 | Loop 类 |
| --- | --- | --- |
| `rdagent fin_factor` | `app/qlib_rd_loop/factor.py` | `FactorRDLoop(RDLoop)` |
| `rdagent fin_model` | `app/qlib_rd_loop/model.py` | `ModelRDLoop(RDLoop)` |
| `rdagent fin_quant` | `app/qlib_rd_loop/quant.py` | `QuantRDLoop(RDLoop)`（自定义了 `direct_exp_gen`/`coding`/`running`/`feedback`） |
| `rdagent fin_factor_report` | `app/qlib_rd_loop/factor_from_report.py` | `FactorReportLoop(FactorRDLoop)` |

四者共享同一套执行环境：`rdagent/scenarios/qlib/experiment/workspace.py` 的 `QlibFBWorkspace`（`execute()` 会在 Docker 或 Conda 环境里跑 `qrun <config>.yaml` 再跑 `read_exp_res.py` 解析结果），以及 `rdagent/scenarios/qlib/docker/Dockerfile`（基于 `pytorch/pytorch:2.2.1-cuda12.1-cudnn8-runtime`，`git clone` 官方 Qlib 仓库并 `pip install -e .` 固定到某个 commit）。是否用 Docker 由 `MODEL_COSTEER_SETTINGS.env_type`（`docker`/`conda`）决定，`QTDockerEnv`/`QlibCondaEnv` 定义在 `rdagent/utils/env.py`（文档08）。

所有具体类是通过 `rdagent/app/qlib_rd_loop/conf.py` 里的四个 `BasePropSetting` 子类（`FactorBasePropSetting`/`ModelBasePropSetting`/`FactorFromReportPropSetting`/`QuantBasePropSetting`）用**类路径字符串**注入到 `RDLoop` 里的，这是整篇文档最值得先看懂的一张"地图"，后文逐场景展开时都会对照它。

## 1. 因子挖掘（`fin_factor`）

### 1.1 整体接线（`FactorBasePropSetting`，`app/qlib_rd_loop/conf.py`）

```
scen               = scenarios.qlib.experiment.factor_experiment.QlibFactorScenario
hypothesis_gen      = scenarios.qlib.proposal.factor_proposal.QlibFactorHypothesisGen
hypothesis2experiment = scenarios.qlib.proposal.factor_proposal.QlibFactorHypothesis2Experiment
coder               = scenarios.qlib.developer.factor_coder.QlibFactorCoSTEER   # = components.coder.factor_coder.FactorCoSTEER
runner              = scenarios.qlib.developer.factor_runner.QlibFactorRunner
summarizer          = scenarios.qlib.developer.feedback.QlibFactorExperiment2Feedback
```

`train_start/train_end/valid_start/valid_end/test_start/test_end` 这几个日期字段（默认 2008-2014 训练、2015-2016 验证、2017-2020-08 测试）也在这个配置里，直接决定了 Qlib 回测配置里 `segments` 的取值。

`FactorRDLoop`（`app/qlib_rd_loop/factor.py`）只对标准 `RDLoop` 做了两处定制：
- `skip_loop_error = (FactorEmptyError, CoderError)`：这两类异常会被当作"这轮失败"，直接跳到 `feedback` 步生成一个 `decision=False` 的反馈，循环继续，而不是整体崩溃。
- 重写了 `running()`：调用 `runner.develop` 后，如果返回 `None` 就主动 `raise FactorEmptyError("Factor extraction failed.")`——因为 Runner 在"所有新因子都算不出有效数值"时，约定返回 `None` 而不是抛异常。

### 1.2 假设生成：对什么做推理

`QlibFactorHypothesisGen`（`scenarios/qlib/proposal/factor_proposal.py`，继承 `components/proposal` 的 `FactorHypothesisGen`，再往上是 `core/proposal.HypothesisGen`）的 `prepare_context` 把以下内容拼进 Prompt：
- `hypothesis_and_feedback`：整条 `trace.hist` 的历史假设+反馈摘要（第一轮为空）。
- `last_hypothesis_and_feedback`：上一轮的假设和反馈。
- `RAG`（这里不是知识库检索，只是一段写死的策略提示）：`trace.hist` 少于 15 轮时提示"先尝试简单快速的因子"，之后提示"要尝试能拿到高 IC 的因子（如机器学习类因子）"——即一个人为设计的、按轮次切换的探索策略。
- `hypothesis_output_format`/`hypothesis_specification`：来自 `scenarios/qlib/prompts.yaml` 的 `factor_hypothesis_output_format`（只要求输出 `hypothesis`+`reason` 两个字段的 JSON）和 `factor_hypothesis_specification`（规定"每轮生成 1-5 个因子""先简单再复杂""连续多轮没超过 SOTA 就换方向"等具体写作规则）。

`convert_response` 把 LLM 返回的 JSON 解析成一个纯文本的 `Hypothesis`（`QlibFactorHypothesis = Hypothesis`，没有子类化，只是类型别名）。注意：这一步**只产生"因子思路"的文字描述，不产生具体因子公式**——具体因子是下一步 `Hypothesis2Experiment` 做的。

### 1.3 假设→实验：`FactorTask` 的字段 & 去重

`QlibFactorHypothesis2Experiment.prepare_context` 把 `hypothesis` 文本、场景描述（`scen.get_scenario_all_desc()`）、历史假设反馈一起喂给 LLM，要求按 `factor_experiment_output_format`（`scenarios/qlib/prompts.yaml`）输出 JSON：每个因子一个条目，含 `description`（要求以 `[Momentum Factor]` 这类类型标签开头）、`formulation`（LaTeX 公式）、`variables`（公式里符号的说明字典）。

`convert_response` 把每条 JSON 转成 `FactorTask`（`rdagent/components/coder/factor_coder/factor.py`），构造参数为 `factor_name/factor_description/factor_formulation/variables`（还有可选的 `resource`）。`FactorTask` 继承 `CoSTEERTask`（进而是 `core/experiment.Task`），`get_task_information()` 把这四项拼成一段文本，作为 CoSTEER 内循环里"这个任务是什么"的唯一标识。

生成的 `QlibFactorExperiment(tasks, hypothesis=hypothesis)`（`scenarios/qlib/experiment/factor_experiment.py`）还做了两件重要的事：
- `exp.based_experiments = [QlibFactorExperiment(sub_tasks=[])] + [历史上所有 decision=True 的 FactorExperiment]`——即"继承所有历史 SOTA 因子"，为后面增量叠加因子库做准备。
- **跨轮去重**：遍历新生成的 `tasks`，如果某个因子名字（`factor_name`）已经出现在任何历史 `based_experiments` 的 `sub_tasks` 里，就把它从本轮任务列表里剔除（纯字符串比较因子名，不是语义去重）。

`QlibFactorExperiment.__init__` 还会创建 `self.experiment_workspace = QlibFBWorkspace(template_folder_path=.../factor_template)`——这是整合阶段（Runner 阶段）要用的"可运行工作区模板"，和下面 Coder 阶段每个 `FactorTask` 各自的 `FactorFBWorkspace` 是两个不同的东西。

### 1.4 Coder 生成什么代码：`factor.py` 接口约定

`coder = QlibFactorCoSTEER`，就是通用的 `rdagent.components.coder.factor_coder.FactorCoSTEER`（一个 `Developer`），内部驱动的是 02 篇讲的 CoSTEER evolve/evaluate 内循环（`evolving_strategy.py` 生成/修订代码，`evaluators.py` 评估）。对每个 `FactorTask`，LLM 需要写出一个单文件 `factor.py`，接口约定（`scenarios/qlib/experiment/prompts.yaml` 的 `qlib_factor_interface`/`qlib_factor_output_format`）非常具体：

- 文件必须包含 import 部分 + 一个 `calculate_{function_name}` 主函数 + `if __name__ == "__main__":` 调用它；**不允许写 try/except**（异常要原样抛出，由框架捕获当反馈）。
- 用户会直接 `python factor.py` 执行该文件，代码要把计算结果写到同目录下的 `result.h5`——一个 HDF5 文件，内容是一个 `pandas.DataFrame`，索引是 `(datetime, instrument)` 的 MultiIndex，唯一一列就是因子值。
- `qlib_factor_strategy` 额外要求"每一步数据处理都要用注释说明数据结构（是否多级索引、怎么取某一级等）"，这是为了让 LLM 自己在生成代码时少犯索引错误。

实际执行发生在 `FactorFBWorkspace.execute()`（`components/coder/factor_coder/factor.py`）：把 `factor.py` 和数据文件夹（`FACTOR_COSTEER_SETTINGS.data_folder`/`data_folder_debug`，来自 `scenarios/qlib/experiment/utils.py:generate_data_folder_from_qlib()`，用 Qlib 导出的 `daily_pv.h5` 价量数据）链接进工作区，然后 `subprocess` 直接跑 `python factor.py`（**不是在 Docker 里跑**——这一步是本机 Python 子进程，有超时控制 `file_based_execution_timeout`，并用 `FileLock` 防止并发写冲突），读回 `result.h5`。CoSTEER 的评估器（`components/coder/factor_coder/evaluators.py` 的 `FactorEvaluatorForCoder`）依次用 `FactorValueEvaluator`（对比生成的因子值是否和已知正确实现数值接近/高度相关）、`FactorCodeEvaluator`（代码质量点评）、`FactorFinalDecisionEvaluator`（综合给出 `final_decision`）——这是"一段代码的多轮修订收敛"内循环，和下面"整体跑 Qlib 回测"是两层不同粒度的验证。

### 1.5 Runner 怎么跑 Qlib：数据合并、增量 SOTA、LGBM/SOTA 模型二选一

`QlibFactorRunner`（`scenarios/qlib/developer/factor_runner.py`，继承 `components/runner.CachedRunner`，结果按任务信息做了 pickle 缓存）是因子场景里最复杂的一段逻辑：

1. 如果 `exp.based_experiments` 非空且最后一个历史实验还没有 `result`（比如 `withdraw_loop_error` 场景下的基线实验），先递归 `develop` 它，保证"基线"一定有结果可比。
2. `process_factor_data`（`scenarios/qlib/developer/utils.py`）并行（`multiprocessing_wrapper`）执行历史 SOTA 因子工作区和本轮新因子工作区的 `.execute("All")`，把各自的 `result.h5` 拼接成一张大表，并做了索引规范化（统一成 `(datetime, instrument)` 两级 MultiIndex，容忍重复的索引 level）。
3. **去重**：`deduplicate_new_factors` 计算新因子和已有 SOTA 因子两两之间的 IC（相关系数），IC ≥ 0.99 的新因子列被剔除——避免"新因子其实和老因子高度冗余"污染因子库。
4. 把 `SOTA_factor` 与去重后的 `new_factors` 拼接、按列名去重、包进 `("feature", 因子名)` 的多级列，存成 `combined_factors_df.parquet`（放进 `exp.experiment_workspace.workspace_path`，供 Qlib 配置里的 `StaticDataLoader` 读取）。
5. 判断历史里有没有一个"被接受的模型实验"（`QlibModelExperiment`，`decision=True`）：
   - 有 → 把该 SOTA 模型的 `model.py` 注入工作区，并把它的 `training_hyperparameters`/`model_type` 一并带过去，用 `conf_combined_factors_sota_model.yaml` 跑（即"用目前最好的模型架构来检验新因子"）；
   - 没有 → 用 `conf_combined_factors.yaml`（固定用 `LGBModel`/LightGBM，`qlib.contrib.model.gbdt`，超参写死在配置里）跑。
   - 如果这是第一轮（`based_experiments` 为空），则看 `exp.base_feature_codes` 是否有预置基线特征，决定用 `conf_combined_factors.yaml` 还是 `conf_baseline.yaml`（纯基线 Alpha158 特征 + LGBM，完全不含 LLM 生成的因子，用来建立第一条基线）。

`QlibFBWorkspace.execute(qlib_config_name, run_env)`（`scenarios/qlib/experiment/workspace.py`）是真正的落地执行：在 `QTDockerEnv`/`QlibCondaEnv` 里依次跑 `qrun <config>.yaml`（Qlib 官方命令行，会用 Jinja2 渲染好的 yaml 跑完整的"构建数据集→训练模型→策略回测→生成分析记录"流程）和 `python read_exp_res.py`（`scenarios/qlib/experiment/factor_template/read_exp_res.py`：用 Qlib 的 `mlflow` 记录器接口 `R.list_experiments()/list_recorders()` 找到最新一次运行，导出所有 `metrics` 到 `qlib_res.csv`，并把 `portfolio_analysis/report_normal_1day.pkl` 另存成 `ret.parquet` 给前端画图）。最终 `exp.result` 就是 `qlib_res.csv` 对应的一个 `pandas.Series`（index 是指标名）。

### 1.6 反馈与 SOTA 判定：看哪些指标

`QlibFactorExperiment2Feedback.generate_feedback`（`scenarios/qlib/developer/feedback.py`）只截取 `IMPORTANT_METRICS = ["IC", "1day.excess_return_with_cost.annualized_return", "1day.excess_return_with_cost.max_drawdown"]` 三项，和 `exp.based_experiments[-1].result`（即当前 SOTA）对比，连同假设文本、各因子任务描述一起交给 LLM（`scenarios.qlib.prompts:factor_feedback_generation`），要求输出 `Observations/Feedback for Hypothesis/New Hypothesis/Reasoning/Replace Best Result`（最后一项经 `convert2bool` 转成 `decision`）。注意：**是否采纳新 SOTA 是 LLM 主观判断的结果**（给了三项关键指标做参考），不是写死的数值阈值规则。`decision=True` 的实验会在下一轮被 `Trace.get_sota_experiment`/`based_experiments` 逻辑当作新的基线继续累加因子。

## 2. 模型挖掘（`fin_model`）

### 2.1 整体接线（`ModelBasePropSetting`）

```
scen        = scenarios.qlib.experiment.model_experiment.QlibModelScenario
hypothesis_gen = scenarios.qlib.proposal.model_proposal.QlibModelHypothesisGen
hypothesis2experiment = scenarios.qlib.proposal.model_proposal.QlibModelHypothesis2Experiment
coder       = scenarios.qlib.developer.model_coder.QlibModelCoSTEER   # = components.coder.model_coder.ModelCoSTEER
runner      = scenarios.qlib.developer.model_runner.QlibModelRunner
summarizer  = scenarios.qlib.developer.feedback.QlibModelExperiment2Feedback
```

`ModelRDLoop`（`app/qlib_rd_loop/model.py`）比 `FactorRDLoop` 还简单——只声明了 `skip_loop_error = (ModelEmptyError,)`，没有重写任何 step。

### 2.2 假设生成：对模型结构的推理

`QlibModelHypothesisGen.prepare_context`（`scenarios/qlib/proposal/model_proposal.py`）比因子版多了一路 `SOTA_hypothesis_and_feedback`：从历史里倒序找第一个 `decision is True` 的节点，专门喂给模型。`RAG` 字段同样是写死的领域知识（不是检索出来的），内容是："市场数据是时间序列，GRU/LSTM 适合；暂时不要生成 GNN"，以及明确的训练/验证样本规模数量级，提示"如果你认为模型本身不错，只是超参不对，可以返回同一个模型只调超参"。`hypothesis_specification` 对应 `scenarios/qlib/prompts.yaml` 的 `model_hypothesis_specification`：要求假设**聚焦模型架构**（层配置、激活函数、正则化手段），明确"不要做特征工程相关的假设"，并鼓励"参照顶会（NeurIPS/ICLR/ICML/SIGKDD）水平的创新"。

### 2.3 假设→实验：`ModelTask` 的字段

`QlibModelHypothesis2Experiment.convert_response` 按 `model_experiment_output_format`（`scenarios/qlib/prompts.yaml`，要求"这一轮只设计一个模型"）解析 JSON，构造 `ModelTask`（`components/coder/model_coder/model.py`），字段包括：`name/description/formulation(LaTeX)/architecture/variables/hyperparameters/training_hyperparameters/model_type`。其中 `model_type` 取值为 `"Tabular"`/`"TimeSeries"`（代码注释里还提到 `Graph`/`XGBoost`，但 Qlib 场景实际只用前两种）决定了后面 Runner 用哪套 Qlib dataset 类和 Jinja2 配置分支。`exp.based_experiments` 只保留历史上 `decision=True` 且类型是 `ModelExperiment` 的实验（模型场景不像因子场景那样做"跨轮去重"，因为每轮只产出一个候选模型，直接和当前 SOTA 比较）。

### 2.4 Coder 生成什么代码：`model.py` 的 `model_cls` 约定

`qlib_model_interface`（`scenarios/qlib/experiment/prompts.yaml`）规定生成的 `model.py` 必须：import 必要库 → 定义一个 `torch.nn.Module` 子类（`__init__` + `forward(tensor) -> tensor`）→ 把类赋给变量 `model_cls`。`ModelFBWorkspace.execute()`（`components/coder/model_coder/model.py`）用这个约定做"是否能跑通"的轻量自检：按 `model_type` 构造一个假的输入张量（`Tabular` → `(batch, num_features)`；`TimeSeries` → `(batch, num_timesteps, num_features)`；代码里还保留了 `Graph` 分支但 Qlib 场景不会触发），把 `model_cls` 所有参数填成固定值后跑一次前向传播，检查输出 shape 和是否报错——执行脚本模板是 `components/coder/model_coder/model_execute_template_v1.txt`，通过 `QTDockerEnv`/`QlibCondaEnv.dump_python_code_run_and_get_results` 在 Docker/Conda 里跑（这一步和因子场景不同——模型的"能否跑通"自检默认在 Docker/Conda 环境里执行，而不是本机子进程）。这只是 CoSTEER 内循环的"语法/形状自检"，真正的效果验证要到 Runner 把它接进 Qlib 训练流程才知道。

### 2.5 Runner 怎么跑 Qlib

`QlibModelRunner`（`scenarios/qlib/developer/model_runner.py`）逻辑比因子 Runner 简单：
1. 若历史基线实验还没结果，递归先跑它。
2. 若 `based_experiments` 里有 `QlibFactorExperiment`（即存在 SOTA 因子库），用 `process_factor_data` 合并出 `combined_factors_df.parquet`，并把 `exist_sota_factor_exp` 标记为真；否则只用 Qlib 内置的 Alpha158 基线特征。
3. 把 `exp.sub_workspace_list[0].file_dict["model.py"]` 注入 `exp.experiment_workspace`（如果 Coder 没写出 `model.py` 就抛 `ModelEmptyError`）。
4. 把 `ModelTask.training_hyperparameters` 里的 `n_epochs/lr/early_stop/batch_size/weight_decay` 转成环境变量传给 Qlib 配置（`conf_sota_factors_model.yaml` / `conf_baseline_factors_model.yaml`，二者都用 `GeneralPTNN`——Qlib 提供的"用任意 PyTorch 模型做训练/推理"的通用包装器，`pt_model_uri: "model.model_cls"` 正是去读 Coder 生成的那个变量），按 `model_type` 是 `TimeSeries` 还是 `Tabular` 决定 `dataset_cls` 用 `TSDatasetH` 还是 `DatasetH`，以及是否需要 `step_len`。
5. 剩下的执行路径（`qrun` → `read_exp_res.py` → `exp.result`）和因子场景完全一致，复用的是同一个 `QlibFBWorkspace.execute`。

### 2.6 反馈

`QlibModelExperiment2Feedback.generate_feedback` 用 `trace.get_sota_hypothesis_and_experiment()`（`core/proposal.Trace`）拿到当前 SOTA 的假设/任务描述/`model.py` 代码/`IMPORTANT_METRICS` 指标，和本轮结果一起交给 LLM（`scenarios.qlib.prompts:model_feedback_generation`），解析出 `Observations/Feedback for Hypothesis/New Hypothesis/Reasoning/Decision`。**代码里有一处明显的冗余/疑点**：该函数先调用一次 `APIBackend().build_messages_and_create_chat_completion(...)` 得到 `response` 并解析成 `response_json`，但随后又对**同一组 prompt**再调用一次得到 `response_hypothesis`，最终返回值完全基于第二次调用的结果（`response_json_hypothesis`），第一次调用的 `response`/`response_json` 被整体丢弃未使用——这更像是遗留的重复代码（多花一次 LLM 调用但没有实际作用），而不是有意为之的"两次投票"机制（没有任何投票/融合逻辑）。文档特此标出，供项目维护者确认是否为 bug。

## 3. 联合优化（`fin_quant`，即 RD-Agent(Q)）

### 3.1 整体接线（`QuantBasePropSetting`）

```
scen = scenarios.qlib.experiment.quant_experiment.QlibQuantScenario
quant_hypothesis_gen        = scenarios.qlib.proposal.quant_proposal.QlibQuantHypothesisGen
factor_hypothesis2experiment = scenarios.qlib.proposal.factor_proposal.QlibFactorHypothesis2Experiment
model_hypothesis2experiment  = scenarios.qlib.proposal.model_proposal.QlibModelHypothesis2Experiment
factor_coder / model_coder   = 同 fin_factor / fin_model 的 CoSTEER 实现
factor_runner / model_runner = 同 fin_factor / fin_model 的 Runner
factor_summarizer / model_summarizer = 同 fin_factor / fin_model 的 Experiment2Feedback
action_selection = "bandit"  # 可选 "llm" / "random"
```

`fin_quant` 不是简单复用 `RDLoop`：`QuantRDLoop`（`app/qlib_rd_loop/quant.py`）**重写了 `__init__`**，同时持有一整套"因子版"和一整套"模型版"的 `Hypothesis2Experiment`/`Coder`/`Runner`/`Summarizer`（上面六个配置项都在这里被 `import_class` 实例化），并用自己的 `QuantTrace(Trace)`（`scenarios/qlib/proposal/quant_proposal.py`）记录历史——`QuantTrace` 额外持有一个 `EnvController` 实例（见下）。

### 3.2 "交替"的具体机制：`action` 字段 + 三种 action-selection 策略

这是 `fin_quant` 与另外两个场景最大的结构性差异：每一轮循环，先通过 `QlibQuantHypothesisGen`（`FactorAndModelHypothesisGen` 子类）决定**这一轮到底是做因子研究还是模型研究**（`hypo.action ∈ {"factor", "model"}`），再分别路由到对应的 `Hypothesis2Experiment`/`Coder`/`Runner`/`Summarizer`。决定 `action` 的策略由 `QUANT_PROP_SETTING.action_selection` 控制，三种实现都在 `prepare_context` 里（`scenarios/qlib/proposal/quant_proposal.py`）：

1. **`bandit`（默认）**：`EnvController`（`scenarios/qlib/proposal/bandit.py`）维护一个双臂线性 Thompson Sampling 老虎机（`LinearThompsonTwoArm`，`arm ∈ {"factor","model"}`，8 维特征：`ic, icir, rank_ic, rank_icir, arr, ir, -mdd, sharpe`，`prior_var=10, noise_var=0.5`）。每轮先用上一轮的结果 `extract_metrics_from_experiment(trace.hist[-1][0])` 算出一个标量 reward（`weights · metric_vector`，权重向量硬编码为 `(0.1, 0.1, 0.05, 0.05, 0.25, 0.15, 0.1, 0.2)`，年化收益权重最高）更新上一次选择的那条臂的贝叶斯线性回归后验，再采样决定下一步的 `action`（`next_arm` 对两条臂都采样一个预期 reward，取较大者）。**这不是严格意义上的"轮流交替"（strict alternation），而是一个根据历史收益自适应选择下一步做什么的强化学习式决策**——README 里"alternating factor-model optimization"的说法在字面上略有误导，实际是"自适应选择"而非"严格轮换"，值得向项目方确认措辞本意。
2. **`llm`**：把历史假设反馈喂给 LLM，用 `scenarios.qlib.prompts:action_gen` 的 system/user prompt 直接问"接下来该做因子还是模型"，JSON 返回里取 `action` 字段（默认回退到 `"factor"`）。
3. **`random`**：`random.choice(["factor", "model"])`，纯随机，应该是用于消融实验/测试用的对照组。

选定 `action` 之后，上下文构造上也做了"专属视角裁剪"：如果本轮是 `"factor"`，历史里只会塞入"所有因子实验 + 最近一个被接受的模型实验"；如果是 `"model"`，则相反（"所有模型实验 + 最近一个被接受的因子实验"）——这样 LLM 在决定"下一个因子怎么设计"时不会被无关的模型架构历史干扰,反之亦然。

`QlibQuantHypothesis(Hypothesis)` 比普通 `Hypothesis` 多了一个 `action` 字段，`QlibQuantHypothesisGen.convert_response` 解析 `hypothesis_output_format_with_action`（`scenarios/qlib/prompts.yaml`，要求 JSON 额外带 `action` 字段）。

`direct_exp_gen`（`QuantRDLoop`，覆盖了 `RDLoop` 默认实现）拿到 `hypo` 后，按 `hypo.action` 调用 `factor_hypothesis2experiment.convert` 或 `model_hypothesis2experiment.convert`——这两者就是 1.3/2.3 节介绍的那两个类，**完全复用**，没有针对 quant 场景做特化。`coding`/`running`/`feedback` 三步同理，按 `action` 分支调用对应的 factor/model 版 Coder/Runner/Summarizer（见 `app/qlib_rd_loop/quant.py` 第 92-128 行）。

### 3.3 场景描述的"双视角"：`QlibQuantScenario`

`QlibQuantScenario`（`scenarios/qlib/experiment/quant_experiment.py`）把 `QlibFactorScenario`/`QlibModelScenario` 的 `background/output_format/interface/simulator` 四个方法都改成了带 `tag ∈ {None,"factor","model"}` 参数的版本，`get_scenario_all_desc(..., action=...)` 会根据当前 `action` 只拼出对应那一半描述（`factor_proposal.py`/`model_proposal.py` 里凡是 `isinstance(trace.scen, QlibQuantScenario)` 的分支都会传 `action=` 参数走这条路）——这就是因子/模型两条子流程复用同一套 Prompt 模板类，却能各自看到"合身"的场景描述的原因。

### 3.4 共享状态：`self.plan` 里的 Alpha20 基线特征

`QuantRDLoop.__init__` 里有 `self.plan: ExperimentPlan = {"features": ALPHA20, "feature_codes": {}}`（`ALPHA20` 来自 `rdagent/utils/qlib.py`，是一组预定义的 Qlib Alpha158 子集表达式，作为"始终存在的基础特征"，不需要 LLM 重新发明）。这个 `plan` 在 `direct_exp_gen` 里被写回每个新生成的 `exp.base_features`/`exp.based_experiments[-1].base_features`，保证因子 Runner/模型 Runner 在合并特征时始终带着这组基线特征。

## 4. 研报因子抽取（`fin_factor_report`）

### 4.1 整体接线与 Loop 结构

`FactorFromReportPropSetting`（继承 `FactorBasePropSetting`）只覆盖了 `scen = scenarios.qlib.experiment.factor_from_report_experiment.QlibFactorFromReportScenario`（仅仅是改了 `rich_style_description` 文案，其余 background/interface/output_format/simulator 全部继承自 `QlibFactorScenario`，因子接口约定与 `fin_factor` 完全一致），并新增三个该场景专属配置：`report_result_json_file_path`（默认 `git_ignore_folder/report_list.json`，一份"研报列表"JSON）、`max_factors_per_exp`（每份研报最多实现 6 个因子）、`report_limit`（最多处理 20 份研报）。

`FactorReportLoop`（`app/qlib_rd_loop/factor_from_report.py`，继承 `FactorRDLoop`）重写了 `direct_exp_gen` 和 `coding`，`running`/`feedback`/`record` 三步**原样继承自 `FactorRDLoop`/`RDLoop`**——也就是说一旦因子从 PDF 里被抽出来，剩下的"跑 Qlib 回测、生成反馈、写入 Trace"流程和 `fin_factor` 完全相同。`loop_n` 被设成 `min(研报数量, report_limit)`：**每一轮循环对应一份 PDF 研报**，不是"同一份研报多轮迭代假设"。

### 4.2 PDF → 因子的抽取链路（复用文档reader，不重复展开）

PDF 文本抽取本身依赖 `rdagent/components/document_reader/document_reader.py`（`load_and_process_pdfs_by_langchain`、`extract_first_page_screenshot_from_pdf`，这部分的实现细节见文档04/08，这里不重复），`fin_factor_report` 在此基础上的业务逻辑都在 `scenarios/qlib/factor_experiment_loader/pdf_loader.py`：`FactorExperimentLoaderFromPDFfiles.load()` 串联了一条相当长的多步 LLM 流水线：

1. `load_and_process_pdfs_by_langchain` 把 PDF 转成文本字典。
2. `classify_report_from_dict`：用 LLM 判断这份研报是否属于"量化/因子"相关（0/1 二分类，可投票多次，`vote_time` 参数控制）。
3. `extract_factors_from_report_dict` → 对每份"有用"的研报，先用 `__extract_factors_name_and_desc_from_content` 让 LLM 循环抽取"因子名+描述"（session 式多轮抽取，直到某轮返回空），再用 `__extract_factors_formulation_from_content` 针对已抽出的因子名补抽公式和变量说明（同样支持"缺哪个因子就追问"的多轮纠错）。
4. `merge_file_to_factor_dict_to_factor_dict`：多份研报可能抽出同名因子，简单取"公式描述最长"的那条作为合并结果（注意：这只是研报抽取阶段内部的合并逻辑，发生在 `fin_factor_report` 自己的多因子场景里，不同于 §1.3 提到的、`fin_factor`/`fin_quant` 在 `Hypothesis2Experiment.convert_response` 阶段按因子名做的"跨轮去重"）。
5. `check_factor_viability`：再用一次 LLM 判断每个因子"是否可实现/有意义"（`viability` 布尔值），只保留可行的。
6. 代码里还实现了基于 Embedding + K-Means 聚类 + LLM 判重的 `deduplicate_factors_by_llm`/`__deduplicate_factor_dict`，但在 `load()` 方法的最终调用链里**这一步被注释掉了**（`# factor_dict, duplication_names_list = deduplicate_factors_by_llm(...)`），当前实际生效的流程里没有执行这个去重，只做了"相关性检查"（`check_factor_relevance`，但这个函数在 `load()` 里同样没有被调用）——这是读代码才能发现的细节，值得向项目方确认是"功能暂时禁用"还是"遗留的死代码"。
7. 最终把 `filtered_factor_dict` 交给 `FactorExperimentLoaderFromDict().load()`（`scenarios/qlib/factor_experiment_loader/json_loader.py`）转成标准的 `QlibFactorExperiment`（内部就是逐条构造 `FactorTask`）。

### 4.3 "事后"补写假设（Hypothesis）

和 `fin_factor`/`fin_model`/`fin_quant` 不同（那三者是先生成 `Hypothesis` 再落地成 `Experiment`），`fin_factor_report` 的顺序反过来了：`extract_hypothesis_and_exp_from_reports`（`app/qlib_rd_loop/factor_from_report.py`）先用上面的 PDF 流水线拿到 `exp`（已经有了具体的 `FactorTask` 列表），**再**调用 `generate_hypothesis`（用 `app/qlib_rd_loop/prompts.yaml` 里极简的 `hypothesis_generation` system/user prompt，把抽出的因子描述和研报原文喂给 LLM）去反推出一段"看起来像是先有假设才做实验"的 `Hypothesis` 文本，塞进 `exp.hypothesis`。这是一种"为已有结果配一个合理叙事"的事后生成，而不是真正驱动因子设计的前置推理——文档特此指出这个结构性差异，避免读者误以为它和前三者走的是同一种"先思考再实现"的流程。

`direct_exp_gen` 里如果某份研报抽不出任何有效因子（`exp is None`），会 `self.shift_report += 1` 跳过这份报告并把 `self.loop_n -= 1`（这里有个计数细节：注释写着"每一步都会先减一次 `loop_n`"，用于防止"全是无效研报"时死循环，`loop_n` 减到负数就抛 `LoopTerminationError` 主动结束）。此外还会截断 `exp.sub_workspace_list`/`exp.sub_tasks` 到 `max_factors_per_exp`（默认 6 个），并把 `exp.based_experiments` 设置成"一个空壳 `QlibFactorExperiment` + 历史上所有被接受的实验"——之后 Coder/Runner 的行为就和 `fin_factor` 完全一致（同一个 `QlibFactorCoSTEER`、同一个 `QlibFactorRunner`、同一个 `QlibFactorExperiment2Feedback`）。

## 5. 与其他模块的关系

- **对 `core/`（文档02）的具体化**：四个入口分别把 `Scenario`/`HypothesisGen`/`Hypothesis2Experiment`/`Developer`/`Experiment2Feedback` 这几个抽象接口具体成了 `QlibFactorScenario`/`QlibModelScenario`/`QlibQuantScenario`、`QlibXxxHypothesisGen`、`QlibXxxHypothesis2Experiment`、`QlibXxxRunner`、`QlibXxxExperiment2Feedback`；循环骨架本身（`direct_exp_gen → coding → running → feedback → record`）只有 `fin_quant` 因为要在"因子/模型"两条子流程间路由而重写了四个 step，其余三个入口都是对 `RDLoop` 的最小化定制（`skip_loop_error` + 偶尔重写 `running`）。
- **对 `components/coder`（文档03）的复用**：`QlibFactorCoSTEER`/`QlibModelCoSTEER` 本身只是 `components/coder/factor_coder.FactorCoSTEER`/`model_coder.ModelCoSTEER` 的别名（零定制），说明 Qlib 场景没有给 CoSTEER 内循环写任何场景特化的 evolve/evaluate 策略，全部差异都体现在 Prompt 模板（`scenarios/qlib/experiment/prompts.yaml`）和外层 Runner 的数据拼装逻辑里。
- **对 `components/runner`（文档03）的复用**：`QlibFactorRunner`/`QlibModelRunner` 都继承自通用的 `components/runner.CachedRunner`，复用了它基于"任务信息文本 md5"做的结果缓存机制（`cache_with_pickle` + `get_cache_key`/`assign_cached_result`）。
- **对 `components/document_reader`（文档04/08）的依赖**：`fin_factor_report` 的 PDF 文本抽取和截图抽取直接调用该组件，自身只负责"抽取后怎么变成因子/假设"这一段业务逻辑。
- **对 `utils/env.py`（文档08）的依赖**：`QlibFBWorkspace.execute`、`FactorFBWorkspace`/`ModelFBWorkspace` 内部用到的 `QTDockerEnv`/`QlibCondaEnv`/`KGDockerEnv` 都定义在这里，决定了代码到底是在 Docker 容器还是本机 Conda 环境里跑；本场景里因子代码自身的自检（`FactorFBWorkspace.execute`）走的是本机 `subprocess`，而模型代码自检（`ModelFBWorkspace.execute`）和两者的 Qlib 整体回测（`QlibFBWorkspace.execute`）都走 `QTDockerEnv`/`QlibCondaEnv`，这是本场景内部一个容易被忽略的不一致点。
- **对 `log/`（文档08）的依赖**：本文引用的所有关键对象（`scen`/`hypothesis_gen`/`coder`/`runner`/`feedback`/`exp.result`/`combined_factors_df` 等）在各 Loop 里都通过 `rdagent_logger.log_object(..., tag=...)` 落盘，是 Streamlit/Web UI 回放这几个场景时的数据来源。
