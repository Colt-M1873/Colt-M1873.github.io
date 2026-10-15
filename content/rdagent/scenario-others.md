---
title: "07. 其他场景：通用模型复现 Copilot / 强化学习 Agent² RL-Bench / LLM 微调 FT-Agent"
date: 2026-10-15T10:07:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

本篇介绍三个相对"小而专"的场景，它们的代码规模、团队背景和成熟度都明显不同于 [05. Qlib 量化场景](/rdagent/scenario-qlib-finance/) 和 [06. Data Science / Kaggle 场景](/rdagent/scenario-data-science-kaggle/)：

- **通用模型复现 Copilot**（`rdagent/scenarios/general_model`）：代码量很小，是一个**单次运行的工具型脚本**，严格来说不构成"循环"。
- **强化学习 Agent² RL-Bench**（`rdagent/scenarios/rl`）：RD-Agent 本体只是一个标准 `RDLoop` 场景，但它被嵌在一个更大的、独立的评测框架 `autorl_bench` 里，身份是"被评测的候选 Agent 之一"，而不是 `autorl_bench` 的一部分。
- **LLM 微调 FT-Agent**（`rdagent/scenarios/finetune`）：是三者中最完整的一个,完整复用 `RDLoop` 五步骨架，且 Coder 与 Runner 都基于 CoSTEER 实现，代码量和工程细节（GPU 内存估算、DeepSpeed、OpenCompass 评测）接近 qlib/data_science 场景的量级。

阅读本篇前建议先读 [01. 项目概览](/rdagent/overview/) 和 [02. 核心抽象层](/rdagent/core-abstractions/)，本篇直接复用 Scenario / Task / Workspace / Experiment / Developer(Coder/Runner) / Hypothesis / Trace / RDLoop / CoSTEER 等术语，不再重复定义。

---

## 1. 通用模型复现 Copilot（General Model Copilot）

### 1.1 入口与定位

- CLI：`rdagent general_model <paper_url_or_path>` → `rdagent/app/cli.py` 的 `general_model_cli` → `rdagent/app/general_model/general_model.py` 的 `extract_models_and_implement(report_file_path)`。
- 这是 README 里明确归类为 **"Copilot"**（遵照指令干活）而非"Agent"（自主迭代）的场景之一，看代码可以确认：它**不创建 `Trace`，不调用 `HypothesisGen`/`ExpGen`/`Experiment2Feedback`，也没有 `Runner`**，是一次性的线性管道：

```python
# rdagent/app/general_model/general_model.py
scenario = GeneralModelScenario()
img = extract_first_page_screenshot_from_pdf(report_file_path)      # 仅截图留档，不参与推理
exp = ModelExperimentLoaderFromPDFfiles().load(report_file_path)    # 读 PDF → 生成 Experiment
exp = QlibModelCoSTEER(scenario).develop(exp)                       # 唯一的"develop"调用：Coder
```

跑一次就结束，没有"下一轮根据反馈再提假设"的环节。

### 1.2 论文如何被摄入、如何变成 Task

- PDF 解析依赖 `rdagent/components/document_reader/document_reader.py` 的 `load_and_process_pdfs_by_langchain`（基于 LangChain 的 PDF loader，按文件切出纯文本）——这部分只是引用，细节不在本篇重复。
- 真正的"抽取模型结构"发生在 `rdagent/components/coder/model_coder/task_loader.py`：
  - `extract_model_from_doc(doc_content)`：把全文丢给 LLM（`prompts.yaml` 里的 `extract_model_formulation_system`），要求以 ```` ```json ```` 代码块返回 `{model_name: {description, formulation, architecture, variables, hyperparameters, training_hyperparameters, model_type}}`；解析失败则重试，最多 10 次。
  - `merge_file_to_model_dict_to_model_dict`：多文件/多次抽取结果按模型名合并去重，同名模型保留 `formulation` 字段最长的那个版本。
  - `ModelExperimentLoaderFromDict.load(model_dict)`：把每个模型字典转成一个 `ModelTask`（`rdagent/components/coder/model_coder/model.py`，是 `CoSTEERTask` 子类，带 `formulation`/`architecture`/`variables`/`hyperparameters`/`model_type` 等字段），再打包成 `QlibModelExperiment(sub_tasks=task_l)`（`rdagent/scenarios/qlib/experiment/model_experiment.py`）。
  - `ModelExperimentLoaderFromPDFfiles` 把以上三步串起来，并用 `@wait_retry(retry_n=5)` 包裹整体重试。

一篇论文如果描述了多个模型，这里会产出多个 `ModelTask`，对应同一个 `Experiment` 里的多个 `sub_tasks`。

### 1.3 Scenario：给 LLM 的"写代码规范"

`rdagent/scenarios/general_model/scenario.py` 的 `GeneralModelScenario` 非常薄——它不像 Qlib/DS 场景那样描述真实数据，而是把 `rdagent/scenarios/general_model/prompts.yaml` 里几段固定文本拼成 `get_scenario_all_desc()`：

- `general_model_interface`：规定生成代码必须是一个 `torch.nn.Module` 子类，并赋给变量 `model_cls`；按 `model_type`（`Tabular`/`TimeSeries`/`Graph`）规定输入张量形状（如 `(batch_size, num_features, num_timesteps)`），输出必须是 `(batch_size, 1)`。
- `general_model_output_format`：输出张量要存成 `output.pth`。
- `general_model_simulator`：明确写道"模型不会被加载去做真实回测"（"The models are not loaded and backtested."）——这是判断它是否"自主迭代"的关键证据之一：没有回测反馈，自然也没有下一轮假设。

### 1.4 Coder：复用 Qlib 的 ModelCoSTEER

```python
# rdagent/scenarios/qlib/developer/model_coder.py
QlibModelCoSTEER = ModelCoSTEER  # 直接是 components/coder/model_coder 下 ModelCoSTEER 的别名
```

即 General Model Copilot **直接借用 Qlib 模型挖掘场景的 Coder**，走标准 CoSTEER 内循环（`core/evolving_framework.py` + `evolving_agent.py` 所述的 evolve→evaluate→再 evolve）：

- `ModelFBWorkspace`（`rdagent/components/coder/model_coder/model.py`）把生成的 `model.py` 落盘，并在沙箱环境（`QTDockerEnv`/`QlibCondaEnv`，取决于 `MODEL_COSTEER_SETTINGS.env_type`）中执行，带缓存的 `execute()` 方法。
- `ModelCoSTEEREvaluator.evaluate`（`rdagent/components/coder/model_coder/evaluators.py`）用**固定的合成输入**（`batch_size=8, num_features=30, num_timesteps=40`，固定 `input_value`/`param_init_value`，避免随机性）跑一遍生成的模型，拿到 `gen_np_array`；再用三个子评估器综合打分：
  - `shape_evaluator`：输出形状是否为 `(8, 1)`；
  - `value_evaluator`：如果有 ground-truth 参考实现（`gt_implementation`，主要用于内部 benchmark 场景），比较数值是否接近；对真实论文复现场景通常没有 ground truth，这一项不生效；
  - `ModelCodeEvaluator`：LLM 读代码 + 读论文抽取出的 `formulation`/`architecture`，判断代码是否忠实实现了描述的结构。
- 多轮修订由 `RAGEvoAgent.multistep_evolve` 驱动（文档02已述），直到评估通过或达到 `max_loop`。

### 1.5 与标准 R&D Loop 的关系：单次管道，而非循环

| 维度 | General Model Copilot | 标准 `RDLoop` 场景 |
| --- | --- | --- |
| 外层 Trace/Hypothesis/Feedback | 无 | 有 |
| 是否迭代"下一轮假设" | 否，跑一次即结束 | 是 |
| Runner（真实执行/回测） | 无——"not loaded and backtested" | 有 |
| 复用的循环机制 | 仅 CoSTEER 内循环（代码改到形状/结构正确为止） | R&D 外循环 + CoSTEER 内循环两层都用 |

所以它名副其实是"Copilot"：**给定一篇论文，尽力把其中描述的模型结构转成一份能跑通、形状正确的 PyTorch 代码**，不负责判断这个模型在真实任务上是否有用，也不会自己提出"换种写法"的新假设。

**相关文件**：`rdagent/app/general_model/general_model.py`、`rdagent/scenarios/general_model/scenario.py`、`rdagent/scenarios/general_model/prompts.yaml`、`rdagent/components/coder/model_coder/task_loader.py`、`rdagent/components/coder/model_coder/model.py`、`rdagent/components/coder/model_coder/evaluators.py`、`rdagent/components/coder/model_coder/eva_utils.py`、`rdagent/scenarios/qlib/developer/model_coder.py`、`rdagent/components/document_reader/document_reader.py`（仅引用）。

---

## 2. 强化学习 Agent² RL-Bench

### 2.1 先澄清一个命令行误区

任务要求里提到"CLI: `rdagent rl`（如果存在）"——实际检查 `rdagent/app/cli.py` 后发现**根本没有注册 `rl` 子命令**（`general_model`/`data_science`/`llm_finetune`/`fin_*` 等都在,但 `rl` 不在)。RL 场景目前只能直接以模块方式运行：

```bash
python -m rdagent.app.rl.loop --base-model Qwen2.5-Coder-0.5B-Instruct --benchmark gsm8k
```

对应 `rdagent/app/rl/loop.py`（typer 构建的独立入口，**没有接入** `rdagent/app/cli.py` 的统一 `app`）。这是文档和代码之间一个值得项目方关注的小落差。

### 2.2 这个场景到底在"评测"什么

关键背景在 `rdagent/scenarios/rl/autorl_bench/README.md`：这是论文 *Agent² RL-Bench: Can LLM Agents Engineer Agentic RL Post-Training?*（arXiv:2604.10547）配套的基准测试框架 **AutoRL-Bench**，其核心问题是——

> 给定一个 Benchmark（GSM8K / HumanEval / ALFWorld / WebShop / AlpacaEval / DeepSearchQA / 自定义 "smith-*" 任务)及一个小模型（如 Qwen2.5-1.5B/7B)的 baseline 分数,让一个"大模型驱动的 Agent"在时限内自主设计并执行一套 RL 后训练（post-training）流程,小模型的分数能否超过 baseline?

`autorl_bench` 本身会去对比**多个候选 Agent**（`autorl_bench/agents/` 下的 `example_agent`、`openhands`、`claude`、`codex`、`gemini`、`opencode`、`openhands`，以及 **`rdagent`**），每个候选 Agent 是一个 `start.sh` 脚本。**RD-Agent 在这里只是被评测的众多候选者之一**，不是评测框架的拥有者：

```bash
# rdagent/scenarios/rl/autorl_bench/agents/rdagent/start.sh
python -m rdagent.app.rl.loop --base-model "$BASE_MODEL" --benchmark "$TASK" \
    --step-n $STEP_N --loop-n $LOOP_N
```

`autorl_bench/run.py`（未在本文详细展开其内部实现，但 README 与 `start.sh`/`scenario.py` 中的环境变量名完全一致，交叉验证可信)在启动某个 Agent 之前,统一完成了：模型/数据下载、workspace 目录+软链接搭建、启动 Flask Grading Server（`core/server.py`,暴露 `/submit` `/set_baseline` `/health`）、并**预先跑一次 baseline 评测**；然后通过环境变量 `BASE_MODEL`/`TASK`/`WORKSPACE`/`MODEL_PATH`/`DATA_PATH`/`OUTPUT_DIR`/`GRADING_SERVER_URL` 把这些信息交给被测 Agent。

### 2.3 RD-Agent 这一侧：标准 RDLoop，但读的是别人准备好的环境

`rdagent/scenarios/rl/scen/scenario.py` 的 `RLPostTrainingScen` docstring 写得很直白：

> "作为 autorl_bench 的 agent 运行时，run.py 已经完成了：资源下载、workspace 创建+软链接、Grading Server 启动+baseline 评测、环境变量传递。本 Scenario 只需读取这些信息，不重复操作。"

它只是从 `os.environ` 读 `BASE_MODEL`/`TASK`/`WORKSPACE`/`MODEL_PATH`/`DATA_PATH`/`OUTPUT_DIR`/`GRADING_SERVER_URL`，缺 `BASE_MODEL`/`TASK` 就直接 `raise ValueError`。

> 注意：`rdagent/app/rl/loop.py` 的注释写着"Scenario.__init__() 中会自动运行 baseline 评测",但 `RLPostTrainingScen.__init__` 实际并不跑 baseline（baseline 是 `run.py` 提前做好的）——这是一处代码注释与实际行为不一致，值得项目方核实/更新注释。

`RLPostTrainingRDLoop`（`rdagent/scenarios/rl/loop.py`）是标准 `RDLoop` 子类，五步骨架完整保留（`direct_exp_gen → coding → running(继承默认实现) → feedback → record`），只是把 `trace` 换成了 `RLTrace = Trace["RLPostTrainingScen", KnowledgeBase]`（`rdagent/scenarios/rl/proposal/trace.py`，纯类型别名,没有额外覆写逻辑）。各组件通过 `RL_RD_SETTING`（`rdagent/app/rl/conf.py`，`RL_` 前缀环境变量）以类路径字符串配置：

```python
hypothesis_gen = "rdagent.scenarios.rl.proposal.proposal.RLPostTrainingExpGen"
coder          = "rdagent.components.coder.rl.RLCoSTEER"
runner         = "rdagent.scenarios.rl.train.runner.RLPostTrainingRunner"
summarizer     = "rdagent.scenarios.rl.dev.feedback.RLExperiment2Feedback"
```

- **Task/Experiment**：`RLTask(Task)` 几乎是空壳（仅携带文本描述);`RLExperiment(Experiment[RLTask, RLWorkspace, RLWorkspace])` 在 `__init__` 里直接创建好 `self.experiment_workspace = RLWorkspace()`。
- **ExpGen**（`RLPostTrainingExpGen.gen`，`rdagent/scenarios/rl/proposal/proposal.py`）：用 LLM 读最近 3 条历史(`trace.hist[-3:]`)生成一个 JSON `{hypothesis, reason, algorithm}`，`algorithm` 字段（如 PPO/GRPO）仅用于日志展示,真正落地是下一步 Coder 自由生成的代码。
- **Coder**（`RLCoSTEER`,`rdagent/components/coder/rl/costeer.py`)：`RLEvolvingStrategy._generate_code` 用 LLM 一次性写出一份 `main.py`（完整的 RL 训练脚本,选什么库/算法/超参全由 LLM 当次决定,代码里甚至留了一个 `_mock_code()` 兜底示例,用 `stable_baselines3` 的 PPO 训练 CartPole）。

  **需要特别指出的一处"占位实现"**：`RLCoderEvaluator.evaluate`（同文件）目前是纯 Mock——

  ```python
  class RLCoderEvaluator:
      def evaluate(self, ...):
          # TODO: 实现真正的评估逻辑
          return CoSTEERSingleFeedback(execution="Mock: executed successfully", ...,
                                        final_decision=True)
  ```

  也就是说 **Coder 内层的 CoSTEER 代码进化循环（`max_loop=1`）并不会真正检验生成代码能否跑通**——它永远判定"成功",代码到底能不能跑、训练是否有效，要等到下一步 Runner 真正执行后才知道。
- **Runner**（`RLPostTrainingRunner`，`rdagent/scenarios/rl/train/runner.py`)：是一个普通的 `Developer`（**不是** CoSTEER）——把 `file_dict` 写到 `$WORKSPACE/code/`，**直接用 `subprocess.run(["python", "main.py"], cwd=code_dir, timeout=self.timeout)` 在当前主机上执行**（默认超时 `360000` 秒 ≈ 100 小时；没有像其他场景那样套一层 Docker `Env` 抽象），再把模型输出目录 POST 给 Grading Server 的 `/submit`，拿到 `score`/`improvement`/`best` 写进 `exp.result`。
- **Feedback**（`RLExperiment2Feedback`，`rdagent/scenarios/rl/dev/feedback.py`)：成功/失败两套 Prompt（`exp_feedback` / `exp_feedback_error`），LLM 输出 `decision`/`reason`/`suggestions`，包成 `HypothesisFeedback` 供下一轮 ExpGen 参考。

### 2.4 计算/时间特殊处理

- Runner 默认超时 100 小时，CLI 支持 `--timeout`（整体 wall-clock 预算，由 `LoopBase`/`RDAgentTimer` 控制）与 `--step-n`/`--loop-n`。
- `RLPostTrainingRDLoop.dump()` 专门覆写了基类的会话快照逻辑：如果 `pickle` 失败（`TypeError`)就跳过保存并打警告，而不是让整个循环崩掉——说明这个场景里确实存在不可序列化的中间对象（比如网络连接/`requests` 会话等），所以牺牲了"断点续跑"的完整性来保证不中断训练。

### 2.5 与标准 R&D Loop 的关系

**完整复用 `RDLoop` 五步骨架 + `Trace`**（只是 `Trace` 的类型别名换成 `RLTrace`，没有定制 DAG 逻辑）。和 qlib/data_science 场景的核心差异是两点：①它是被一个外部评测框架（AutoRL-Bench）当作"一个候选 Agent"来调用和打分的，②它自己的 Coder 内层 CoSTEER 评估目前是占位 Mock，真正的代码正确性验证被推到了 Runner 的真实执行阶段。

**相关文件**：`rdagent/app/rl/loop.py`、`rdagent/app/rl/conf.py`、`rdagent/scenarios/rl/loop.py`、`rdagent/scenarios/rl/scen/scenario.py`、`rdagent/scenarios/rl/experiment/experiment.py`、`rdagent/scenarios/rl/experiment/workspace.py`、`rdagent/scenarios/rl/proposal/proposal.py`、`rdagent/scenarios/rl/proposal/trace.py`、`rdagent/components/coder/rl/costeer.py`、`rdagent/scenarios/rl/train/runner.py`、`rdagent/scenarios/rl/dev/feedback.py`、`rdagent/scenarios/rl/autorl_bench/README.md`、`rdagent/scenarios/rl/autorl_bench/agents/rdagent/start.sh`、`rdagent/scenarios/rl/autorl_bench/core/server.py`。

---

## 3. LLM 微调 FT-Agent

### 3.1 入口与论文背景

- CLI：`rdagent llm_finetune --base-model <model> --benchmark <name> --benchmark-description "<desc>" [--dataset ...] [--upper-data-size-limit N] [--loop-n N] [--timeout 12h]`，在 `rdagent/app/cli.py` 里确实注册为标准子命令（`llm_finetune_cli` → `rdagent/app/finetune/llm/loop.py:main`）。
- 对应论文 *FT-Dojo: Towards Autonomous LLM Fine-Tuning with Language Agents*（ICML 2026，arXiv:2603.01712），`rdagent/app/finetune/llm/README.md` 对流程的五步概括（检查 benchmark/原始数据 → 生成数据处理代码与 LLaMA-Factory 训练配置 → 跑快速失败验证 → 全量微调 → 用 OpenCompass 评测并反馈）与代码实现一致，可信。

### 3.2 `FT_TARGET_BENCHMARK` / `FT_BENCHMARK_DESCRIPTION` 等配置的作用

配置类是 `LLMFinetunePropSetting`（`rdagent/app/finetune/llm/conf.py`，`env_prefix="FT_"`，全局单例 `FT_RD_SETTING`）：

- `target_benchmark`（环境变量 `FT_TARGET_BENCHMARK`，CLI `--benchmark`）：选择哪个 benchmark 适配器,如 `aime24`/`aime25`（数学）、`panorama_par4pc` 等（专利）、`chemcotbench_mol_edit` 等（化学）、`tablebench_*`（表格 QA）、`FinanceIQ_gen`（金融）、`bioprobench_*`（生物）——完整列表见 README 的"Supported Benchmarks"表,注册点在 `rdagent/scenarios/finetune/datasets/__init__.py`（原始数据集）和 `rdagent/scenarios/finetune/benchmark/data/adaptor.py`（`BENCHMARK_CONFIG_DICT`，benchmark→原始数据集的映射与评测方式)。
- `benchmark_description`（`FT_BENCHMARK_DESCRIPTION`,CLI `--benchmark-description`)：一段自然语言描述,告诉 Agent 任务内容和输出格式要求（例如"答案是 0-999 的整数,放在 `\boxed{}` 里"）,会被直接拼进 Scenario 的 Prompt,也会影响数据处理代码要生成什么格式的训练样本。
- `base_model`（`FT_BASE_MODEL`,CLI `--base-model`)：要微调的 HuggingFace 模型 id;目前**必须手动指定**——`main()` 里有一条断言,模型自动选择还是"TODO: not yet supported"。
- `file_path`（`FT_FILE_PATH`,默认 `git_ignore_folder/finetune_files`)：模型/数据集/benchmark 资源的本地根目录,保持稳定可以复用已下载的大文件。
- `upper_data_size_limit`（`FT_UPPER_DATA_SIZE_LIMIT`)：单次实验训练样本数上限。
- `strong_models`/`weak_models`（`FT_STRONG_MODELS`/`FT_WEAK_MODELS`)：**不是** RD-Agent 规划器自己用的模型,而是注入到沙箱环境变量 `STRONG_MODEL_POOL`/`WEAK_MODEL_POOL`,供 Coder 生成的数据处理脚本（`process_data.py`,运行在容器里)调用——比如用强模型做 CoT 生成/复杂推理标注,用弱模型做格式转换/过滤,这是一层"被生成代码在执行期调用的 LLM"，与 RD-Agent 本身的规划 LLM 是两套独立的调用链。
- `judge_model`/`judge_api_key`/`judge_api_base`（`FT_JUDGE_*`)：给 AIME 这类需要"LLM 判题"的 benchmark 配置评分用的模型。

### 3.3 Scenario：复用 `DataScienceScen`，并在初始化时就跑 baseline

`LLMFinetuneScen`（`rdagent/scenarios/finetune/scen/scenario.py`）**继承自 `DataScienceScen`**（而不是直接继承 `core.scenario.Scenario`，这点与 general_model/rl 不同,体现了"微调场景复用 data_science 基础设施"的设计选择)。`__init__` 做了相当多"重活"：

1. 校验/创建 `FT_FILE_PATH`，调用 `prepare_all()` 准备所有已注册数据集，`ensure_ft_assets_exist()` 确保模型权重已下载。
2. `LLaMAFactory_manager.get_info()`（`rdagent/scenarios/finetune/scen/llama_factory_manager.py`)：拉取/解析 LLaMA-Factory 当前支持的训练方法与超参数列表（通过 `extract_parameters.py` 在容器里跑出真实的 CLI 参数定义,并用正则过滤掉推理专用/多模态等不相关参数)，确保 Agent 生成配置时用的参数是"当前版本真实支持的"，不是凭空编的。
3. 生成 `dataset_info.json`（每个数据集的统计信息/样例/README,作为"唯一信息源"），再用 LLM（`--dataset` 未指定时)或直接使用用户指定的数据集,选出与任务相关的数据集子集（`selected_datasets`）。
4. 探测 GPU 数量/显存（`get_runtime_environment_by_env`），用 `MemoryEstimator`（`rdagent/scenarios/finetune/scen/memory_estimator.py`）生成"这个模型+训练方式大概需要多少显存"的 `memory_report`，直接写进 Prompt，帮助 Agent 规避 OOM 配置。
5. **立即跑一次 baseline 评测**（`run_baseline_model_evaluation`，带 `cache_with_pickle` 缓存)：用未微调的原始模型跑一遍目标 benchmark，拆成验证集分数（`benchmark`,Agent 可见,用于后续反馈对比)和测试集分数（`benchmark_test`,**Agent 不可见**,只用于前端展示/最终报告)。

这一步与 RL 场景的"baseline 由外部 `run.py` 提前跑好"形成对照——FT 场景选择把 baseline 评测放进 `Scenario.__init__`，由 RD-Agent **自己**负责。

### 3.4 Task / Experiment：一次微调实验 = 数据处理策略 + 训练配置

- `FTTask(CoSTEERTask)`（`rdagent/components/coder/finetune/exp.py`）字段：`base_model`、`description`（本轮假设对应的任务说明）、`benchmark`、`involving_datasets`（本轮用哪些已选数据集)、`skip_data_processing`（是否复用上一轮/父节点的数据处理脚本，不重新生成)。
- `FTExperiment(Experiment[Task, FTWorkspace, FTWorkspace])`（`rdagent/scenarios/finetune/experiment/experiment.py`）在 `__init__` 里自建 `experiment_workspace = FTWorkspace()`，并提供 `is_ready_to_run()`（检查 `train.yaml` 是否已生成，常量 `FT_YAML_FILE_NAME` 定义于 `components/coder/finetune/conf.py`）。

### 3.5 ExpGen：一次 LLM 会话里同时决定数据策略与训练方法

`LLMFinetuneExpGen`（`rdagent/scenarios/finetune/proposal/proposal.py`）**不**走 `HypothesisGen`+`Hypothesis2Experiment` 两段式，而是直接重写 `ExpGen.gen(trace)`：取 `trace.get_sota_experiment()` 作为父节点，在**同一个 chat session** 里连续问三次（reason → hypothesis → task JSON，后两次复用前面的上下文），一次性决定"这轮要不要换数据处理方式、要不要换微调方法/超参"（`FTHypothesis` 里额外带 `base_model` 字段）。若存在父实验，还会把父节点 `experiment_workspace` 的文件（比如没必要重新生成的 `process_data.py`）注入到新实验里（`inject_from_workspace`），对应 `skip_data_processing` 的"复用上一版数据脚本"语义。

### 3.6 Coder：`LLMFinetuneCoSTEER`——两段式代码生成 + 微批量快速失败验证

`LLMFinetuneCoSTEER`（`rdagent/components/coder/finetune/__init__.py`）是一个**真实**（非占位)的 CoSTEER 实现，`LLMFinetuneEvolvingStrategy.implement_func_list()` 依次执行两个函数：

1. `implement_data`：生成 `process_data.py`——一段会在沙箱里实际执行、且可以调用 `STRONG_MODEL_POOL`/`WEAK_MODEL_POOL` 的 Python 脚本，把原始 HF 数据集转成 Alpaca 风格的 `instruction`/`input`/`output` JSON；如果上一轮的数据脚本已通过评估，或者本轮 `skip_data_processing=True`，则直接跳过、复用旧脚本。
2. `implement_lf_config`：生成 LLaMA-Factory 的 `train.yaml`（+ `test_params.yaml`），Prompt 里会把 `LLaMAFactory_manager` 查出来的"当前支持的方法 + 共享参数 + 各方法专属参数"全部喂给 LLM，让它在真实可用的参数范围内做选择。

评估由 `CoSTEERMultiEvaluator([FTDataEvaluator, FTCoderEvaluator])` 完成，`max_loop = FT_RD_SETTING.coder_max_loop`（默认 10,是一个真正会跑多轮的内循环，不同于 RL 场景的 Mock)。其中最关键的"fail-fast"机制是 `LLMConfigValidator`（`rdagent/components/coder/finetune/unified_validator.py`）：

1. 过滤掉 YAML 里 LLaMA-Factory 不支持的参数；
2. 注入系统强制参数（`SYSTEM_MANAGED_PARAMS`：`output_dir`、`overwrite_cache`、`save_only_model`、`per_device_eval_batch_size=1` 等）；
3. 用一个很小的**微批量（micro-batch）**子集跑一次训练,在真正启动全量训练前,把配置错误(参数不兼容、显存不够、路径错误等)尽早暴露出来——对应 README 里"run fail-fast validation before full training"的描述。

### 3.7 Runner：同样基于 CoSTEER，负责全量训练 + OpenCompass 评测

`LLMFinetuneRunner`（`rdagent/scenarios/finetune/train/runner.py`）是一个有趣的设计——**Runner 本身也是 `CoSTEER` 的子类**，但它的 `FTRunnerEvolvingStrategy.implement_one_task` 什么都不做，直接返回 `{}`（"Coder 已经生成了完整的训练配置，Runner 不做任何修改，直接执行"），真正的工作都在唯一的评估器 `FTRunnerEvaluator`（`rdagent/scenarios/finetune/train/eval.py`）里：

1. **清空旧输出**（`clear_workspace`，只保留 Coder 注入的文件和日志)；
2. **全量数据处理**：不带 `--debug` 标志重跑一次 `process_data.py`，生成完整 `data.json`；
3. **全量训练**：在专用 Docker/conda 环境（`get_ft_env(operation="full_training")`，超时取 `FT_RD_SETTING.full_timeout`，默认 **100 小时**）里执行 LLaMA-Factory 训练，模型/数据目录以只读方式从 `FT_FILE_PATH` 挂载进容器；
4. 从 LLaMA-Factory 产出的 `trainer_state.json` 提取训练/验证 loss 曲线（`extract_loss_history`）；
5. 调用 `run_benchmark`（`rdagent/scenarios/finetune/benchmark/benchmark.py`，底层是**在 Docker 中跑 OpenCompass**）在验证集（Agent 可见）上给微调后的模型打分；AIME 一类题目还会用 `judge_model` 做 LLM 判题。

`LLMFinetuneRunner.get_develop_max_seconds()` 把超时设为 `real_full_timeout() * max_seconds_multiplier`（`FTCoderCoSTEERSettings.max_seconds_multiplier=8`，专门为"LLM 训练比一般代码执行慢得多"放宽的超时倍数），`runner_max_loop` 默认为 1（即每个 RDLoop 迭代只做一次真实的全量训练，不在 Runner 内部反复重训）。

**GPU/算力相关的特殊处理**：
- `MemoryEstimator` 在 Scenario 初始化阶段就把"这个模型+这种训练方法大概需要多大显存"算出来写进 Prompt；
- 支持 DeepSpeed ZeRO-2/ZeRO-3（`rdagent/scenarios/finetune/env/conda/deepspeed/ds_z2_config.json` / `ds_z3_config.json`）做多卡训练；
- 按阶段设置了差异巨大的超时：`debug_data_processing_timeout`（20 分钟,调试用)、`data_processing_timeout`（1 小时)、`micro_batch_timeout`（30 分钟,快速失败验证用)、`full_timeout`（默认 100 小时,全量训练用)——这组超时配置本身就是对"各阶段计算量差异巨大"的直接回应。

### 3.8 Feedback：benchmark 结果如何驱动下一轮假设

`FTExperiment2Feedback`（`rdagent/scenarios/finetune/dev/feedback.py`）：

- 成功时：把 OpenCompass 的 `benchmark` 结果、（采样/截断过的）训练 loss 曲线、`trace.sota_benchmark()`（历史最优的 benchmark 结果）以及 Scenario 初始化时缓存的 `baseline_benchmark_score` 一起喂给 LLM，要求输出 `Decision`/`Reason`/`Code Summary`，打包成 `HypothesisFeedback`（`acceptable=True`）——这份反馈既决定 `FTTrace` 要不要把这次实验记为新的 SOTA,也会被下一次 `LLMFinetuneExpGen` 调用引用。
- 失败时：优先使用 `FTRunnerEvaluator` 留在 workspace 上的结构化诊断（`execution`/`return_checking`/`code` 三段,而不是裸的异常字符串）生成更有针对性的"下一步怎么改"建议。

### 3.9 与标准 R&D Loop 的关系

`LLMFinetuneRDLoop`（`rdagent/scenarios/finetune/loop.py`）就是 `RDLoop` 的直接子类，**完整保留五步骨架**（`direct_exp_gen/coding/feedback/record` 仅做了"从 `prev_out` 里取正确的 exp 对象"这类管道工作，`running` 步沿用基类默认实现去调 `runner.develop`），只是把 `Trace` 换成了 `FTTrace`。与其他场景相比最值得注意的架构特点是：**Coder 和 Runner 都基于 CoSTEER 实现**——这跟"Coder=CoSTEER、Runner=普通 Developer"的常见分工（如 Qlib、RL 场景）不同；Runner 这里用 CoSTEER 只是借壳"跑一次、评一次"的框架，而不是做真正的多轮代码进化。benchmark 驱动的反馈最终又回到下一次 `LLMFinetuneExpGen` 调用,形成完整闭环。

**相关文件**：`rdagent/app/finetune/llm/loop.py`、`rdagent/app/finetune/llm/conf.py`、`rdagent/app/finetune/llm/README.md`、`rdagent/scenarios/finetune/loop.py`、`rdagent/scenarios/finetune/scen/scenario.py`、`rdagent/scenarios/finetune/scen/llama_factory_manager.py`、`rdagent/scenarios/finetune/scen/memory_estimator.py`、`rdagent/scenarios/finetune/experiment/experiment.py`、`rdagent/scenarios/finetune/proposal/proposal.py`、`rdagent/scenarios/finetune/proposal/trace.py`、`rdagent/components/coder/finetune/__init__.py`、`rdagent/components/coder/finetune/conf.py`、`rdagent/components/coder/finetune/exp.py`、`rdagent/components/coder/finetune/unified_validator.py`、`rdagent/scenarios/finetune/train/runner.py`、`rdagent/scenarios/finetune/train/eval.py`、`rdagent/scenarios/finetune/benchmark/benchmark.py`、`rdagent/scenarios/finetune/dev/feedback.py`、`rdagent/scenarios/finetune/env/conda/deepspeed/ds_z2_config.json`、`ds_z3_config.json`。

---

## 4. 与其他模块的关系

- **都建立在 [02. 核心抽象层](/rdagent/core-abstractions/) 之上**：三者都使用 `Scenario`/`Task`/`Experiment`/`FBWorkspace` 等基础类；RL 和 FT-Agent 还完整复用了 `RDLoop` 五步骨架与 `Trace`，General Model Copilot 只借用了 CoSTEER 代码进化内循环，并未使用外层 R&D Loop/Trace。
- **都复用 [03. 研发组件](/rdagent/components-development/) 的 CoSTEER 代码进化机制**，但成熟度不同：General Model Copilot 和 FT-Agent 的 CoSTEER 评估器是真实实现（分别做"形状/数值/代码一致性检查"和"微批量快速失败验证+全量训练"），RL 场景的 Coder 端评估器目前是明确标注 `TODO` 的 Mock,真正的正确性校验被推迟到 Runner 阶段的真实执行。
- **General Model Copilot 直接复用 [05. Qlib 场景](/rdagent/scenario-qlib-finance/) 的模型 Coder**（`QlibModelCoSTEER` 就是 `ModelCoSTEER` 的别名），说明"模型结构代码生成"这一能力是跨场景共享的组件，而不是 General Model 专属。
- **FT-Agent 的 Scenario 直接继承 [06. Data Science 场景](/rdagent/scenario-data-science-kaggle/) 的 `DataScienceScen`**，复用了其"场景描述拼装"等基础设施，是三者中与 data_science 血缘最近的一个。
- **执行环境**均依赖 `rdagent/utils/env.py` 的 `Env`/`DockerEnv` 抽象（文档08），但落地方式不同：General Model 用 `QTDockerEnv`/`QlibCondaEnv`（继承自 Qlib）跑合成输入；FT-Agent 用专属的 `FTDockerEnv`/`FTCondaEnv`/`BenchmarkDockerEnv` 挂载模型/数据集只读卷并区分多档超时；RL 场景的 Runner 反而**没有**套用 `Env` 抽象，是直接 `subprocess.run` 在主机上执行——这是三者里执行隔离程度最弱的一处，值得关注其安全/可重复性含义。
- **可观测性**：三者都通过 `rdagent/log` 写结构化日志（文档08），FT-Agent 和 RL 各自还带了独立的 Streamlit UI（`rdagent/app/finetune/llm/ui/app.py`、`rdagent/app/rl/ui/app.py`），General Model Copilot 没有专属 UI，只能用通用的 `rdagent ui` 查看 trace。
