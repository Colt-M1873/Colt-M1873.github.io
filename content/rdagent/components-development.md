---
title: "03. 开发组件（Components / Development）"
date: 2026-10-15T10:03:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

本篇文档覆盖 `rdagent/components/` 下六个与"做研发"直接相关的可复用组件目录：`coder`、`runner`、`proposal`、`workflow`、`agent`、`interactor`。它们共同构成了 `RDLoop`（见 `rdagent/components/workflow/rd_loop.py`）五个步骤的具体实现层：

- `direct_exp_gen` 步骤由 `proposal` 目录中的 `LLMHypothesisGen` / `LLMHypothesis2Experiment` 具体化（并经由 `interactor` 做人机交互的可插拔钩子）；
- `coding` 步骤由 `coder` 目录中的 `CoSTEER` 框架具体化，它是 `core/developer.py:Developer` 的子类，内部驱动 `core/evolving_agent.py:RAGEvoAgent` 做多轮"写代码 → 跑 → 评 → 再写"的内层进化循环；
- `running` 步骤由 `runner` 目录（目前只提供了一个通用的缓存基类 `CachedRunner`，真正的 Docker 执行与 SOTA 比较逻辑在各 Scenario 包里，会在 05/06/07 文档中讲）具体化；
- `feedback`/`record` 步骤依赖 `proposal` 里对应的 `Experiment2Feedback` 实现和 `core.proposal.Trace`；
- `agent` 目录提供的是一个与 RDLoop 的五步模型平行、更底层的工具——一个基于 `pydantic-ai` 的 Agent 封装，被 `coder`（以及未来可能的其它组件）用来做"错误信息 → 查文档"这类有工具调用能力的子任务；
- `workflow` 目录里的 `conf.py:BasePropSetting` 和 `rd_loop.py:RDLoop` 是把上面这些组件用"配置里写类路径字符串 + `import_class` 动态加载"的方式粘合起来的胶水层，也是整个框架的插件机制所在地。

下文中 `coder` 部分内容最多最重要，拆成多个 H3 小节介绍。

## `rdagent/components/coder/` —— CoSTEER 代码实现框架

CoSTEER（Collaborative evolving Strategy for Automatic Data-Science Development，源自论文同名方法）是 RD-Agent 里"把一个 `Task` 变成能跑的代码"的标准实现。它不是一个场景专属的组件，而是一套可以被不同场景（因子库、模型结构、Kaggle 数据科学管线、强化学习后训练、LLM 微调……）复用的通用 Coder 基础设施。

### 3.1 总体结构与文件组织

`rdagent/components/coder/CoSTEER/` 是框架本体：

- `__init__.py`：`CoSTEER` 类本身，`core.developer.Developer` 的子类，是每个场景 Coder 的父类。
- `config.py`：`CoSTEERSettings`（pydantic settings，环境变量前缀 `CoSTEER_`），定义了 `max_loop`、`fail_task_trial_limit`、v1/v2 查询相关的限额、`knowledge_base_path`/`new_knowledge_base_path`、`enable_filelock`、`coder_use_cache` 等全局旋钮。
- `task.py`：`CoSTEERTask(Task)`，目前只加了一个几乎被废弃的 `base_code` 字段（注释里作者自己说"think we don't need the base_code anymore"）。
- `evolvable_subjects.py`：`EvolvingItem(Experiment, EvolvableSubjects)` —— 多重继承把 `core.experiment.Experiment` 和 `core.evolving_framework.EvolvableSubjects` 粘在一起，是内层进化循环操作的"进化主体"。`EvolvingItem.from_experiment(exp)` 从外层的 `Experiment` 构造出内层要进化的副本（只拷贝 `sub_tasks`/`based_experiments`/`experiment_workspace`，不拷贝 `sub_workspace_list`，意味着每次 `develop()` 都从空白 workspace list 开始）。
- `evolving_strategy.py`：`MultiProcessEvolvingStrategy(EvolvingStrategy)` —— 所有具体 Coder 的进化策略基类，核心是把"写代码"这一步拆成可在多进程里并行跑的 `implement_one_task`。
- `evaluators.py`：`CoSTEERSingleFeedback`/`CoSTEERSingleFeedbackDeprecated`/`CoSTEERMultiFeedback`/`CoSTEEREvaluator`/`CoSTEERMultiEvaluator`，评价体系的基础类型。
- `knowledge_management.py`：`CoSTEERRAGStrategy` 及其 V1（已废弃）/V2 两版实现，知识库的查询与生成逻辑都在这里。

具体场景的 Coder（`factor_coder/`、`model_coder/`、`data_science/*`、`rl/`、`finetune/`）都遵循同一套"骨架"：一个继承 `CoSTEERTask` 的 Task 子类、一个继承 `CoSTEEREvaluator`（或直接实现 evaluate）的 Evaluator、一个继承 `MultiProcessEvolvingStrategy` 并重写 `implement_one_task`/`assign_code_list_to_evo` 的 Strategy，最后用一个继承 `CoSTEER` 的类把三者拼起来（在 `__init__` 里构造 `settings`/`eva`/`es` 并传给父类）。`data_science/pipeline/__init__.py` 顶部的注释把这个文件组织约定写得很清楚：`__init__.py` 是入口/Agent、`eval.py` 是评价器、`conf.py` 是配置、`exp.py` 装 Task/Experiment/Workspace、`test.py` 是单独测试入口。

### 3.2 `CoSTEER.develop()`：外层驱动与"回退"机制

`CoSTEER` 继承 `Developer[Experiment]`，它的构造函数根据 `evolving_version`（1 或 2，默认 2；V1 的 `CoSTEERRAGStrategyV1` 在代码里标注"it is deprecated"且很多方法直接 `raise NotImplementedError`）选择 `CoSTEERRAGStrategyV1`/`V2` 作为 RAG 策略。

`develop(exp)` 的关键流程（`rdagent/components/coder/CoSTEER/__init__.py:93-155`）：

1. 用 `EvolvingItem.from_experiment(exp)` 构造内层进化主体 `evo_exp`。
2. 构造一个新的 `core.evolving_agent.RAGEvoAgent`（`self.evolve_agent`），把 `max_loop`、`evolving_strategy`、`rag`、`with_knowledge`、`knowledge_self_gen`、`enable_filelock`、`stop_eval_chain_on_fail` 都传进去。
3. 对 `self.evolve_agent.multistep_evolve(evo_exp, self.evaluator)` 这个生成器做 `for` 循环，每一轮都会：
   - 通过 `should_use_new_evo()`（默认实现：只要新反馈 `is_acceptable()` 就接受）判断是否要把当前这一轮的结果设为"回退候选" `fallback_evo_exp`/`fallback_evo_fb`，并调用 `fallback_evo_exp.create_ws_ckp()` 对 workspace 文件做快照（因为后续的 in-place 修改会污染同一个 workspace 对象）。
   - 检查两个熔断条件：`get_develop_max_seconds()`（子类可覆盖，例如 `DSCoSTEER` 用场景的 debug 超时乘以 `max_seconds_multiplier`）超时，或全局计时器 `RD_Agent_TIMER_wrapper.timer` 超时，命中则提前 `break`。
4. 循环结束后，若存在 `fallback_evo_exp` 就用 `recover_ws_ckp()` 把它恢复为最终结果——**这意味着"最后一轮"的代码未必是最终提交的代码，框架会在历史中选一个曾经被接受过的最优/最近可接受版本回退使用**，这是防止"越改越差"的保险。
5. `_exp_postprocess_by_feedback`：如果所有子任务的 feedback 都不可接受，直接抛 `CoderError`（并标记 `caused_by_timeout`）。

这里能看到一个与 `core/evolving_agent.py:RAGEvoAgent.multistep_evolve` 的清晰分工：**外层 `CoSTEER.develop()` 负责"要不要继续迭代、迭代出错怎么办、最终选哪个版本"，内层 `RAGEvoAgent` 负责"每一轮怎么从知识库查、怎么调用 strategy 评价、知识库怎么自我更新"**。

### 3.3 `MultiProcessEvolvingStrategy`：任务级并行与增量实现

`evolve_iter()`（`rdagent/components/coder/CoSTEER/evolving_strategy.py:108-172`）是每一轮"写代码"的核心：

1. 先按知识库把 `evo.sub_tasks` 分成三类：已在知识库里标记"成功"的任务（直接复用历史实现，不再调 LLM）、已被标记"失败次数超限"的任务（跳过）、以及真正需要本轮调用 LLM 实现的任务（`to_be_finished_task_index`）。`improve_mode=True` 时还有一个特殊规则：只有上一轮反馈里该任务确实存在失败反馈才会被安排实现（即"只修复失败的部分"，用于 Runner 场景——见下文 `assign_code_list_to_evo` 的注释提到"for runner, coder already generated full training config"）。
2. 对 `implement_func_list()` 返回的每个实现函数（默认只有一个 `implement_one_task`），用 `core.utils.multiprocessing_wrapper` 对 `to_be_finished_task_index` 中的任务**并行调用**，并发度由 `RD_AGENT_SETTINGS.multi_proc_n` 控制。
3. 调用子类实现的 `assign_code_list_to_evo()` 把生成的 `{filename: content}` 字典写回对应的 `sub_workspace_list[index]`（通过 `FBWorkspace.inject_files`），并 `yield evo`——这个 `yield` 让外层 `RAGEvoAgent` 可以在每个"实现函数"执行完后立即送去评价，天然支持"分阶段实现 → 分阶段评价"的流水线。

`implement_one_task()` 是留给子类覆盖的抽象方法：输入目标任务、查询到的知识、workspace、上一轮该任务的反馈,输出要写入 workspace 的文件字典。基类提供的默认 `assign_code_list_to_evo` 实现假定"没有真正修改要写"（主要是给 Runner 这种只是复用/微调已有代码的场景用），具体 Coder（factor/model/pipeline/workflow 等）都重写了这两个方法。

### 3.4 评价体系：`CoSTEERSingleFeedback` / `CoSTEERMultiFeedback` / `CoSTEERMultiEvaluator`

`CoSTEERSingleFeedback`（dataclass）是一个任务的评价结果，字段设计明确对应"执行 → 返回值检查 → 代码质量 → 最终决定"四个阶段：`execution`、`return_checking`、`code`、`final_decision`（可能为 `None`，表示还未决定），还有 `raw_execution`（给 UI 用的完整 stdout）和 `source_feedback`（记录这条 feedback 是从哪些来源合并来的）。`__bool__` 直接返回 `final_decision`，所以可以直接 `if feedback:` 判断是否通过。`merge()` 是一个类方法，用于把"同一个任务、多个 Evaluator"给出的多条反馈合并成一条（多个 `execution`/`return_checking`/`code` 字段用空行拼接，`final_decision` 取逻辑 AND）。

`CoSTEERSingleFeedbackDeprecated` 是老版本字段命名（`execution_feedback`/`shape_feedback`/`value_feedback`/`code_feedback`/`final_feedback`），通过 property 把新旧字段名做了双向映射以兼容旧场景代码（`factor_coder` 目前仍在用这个别名 `FactorSingleFeedback = CoSTEERSingleFeedbackDeprecated`）。

`CoSTEERMultiEvaluator`（继承 `core.evolving_agent.RAGEvaluator`）实现了 `evaluate_iter`，这是喂给 `RAGEvoAgent.multistep_evolve` 的生成器协议的具体版本：

- 第一次 `yield` 一个空 `CoSTEERMultiFeedback([])`，用于"接收"外层送进来的 `evo`（生成器协议的惯用写法）。
- 对 `self.single_evaluator`（可以是单个 Evaluator 也可以是一个列表，支持"多个评价器各自评价同一批任务后再合并"）逐个执行：用 `multiprocessing_wrapper` 并行对每个子任务调用 `ev.evaluate(task, workspace, gt_implementation, queried_knowledge)`，然后 `yield` 这批反馈。
- 最终把同一任务在不同 evaluator 下的反馈通过 `CoSTEERSingleFeedback.merge()` 合并，并把 `final_decision=True` 的任务标记到 `sub_tasks[i].factor_implementation = True`（历史遗留字段名，注释里写明"this is to be compatible with factor_implementation"）。

`CoSTEERMultiFeedback` 则是一个 list-like 容器，`is_acceptable()` 要求所有子反馈都可接受，`finished()` 则容忍 `None`（表示该任务被跳过但不计入失败）。

### 3.5 知识管理（RAG）：V1 vs V2

`knowledge_management.py` 里 `CoSTEERRAGStrategy` 基类提供知识库的加载/落盘（`load_or_init_knowledge_base`/`dump_knowledge_base`/`load_dumped_knowledge_base`，用 `core.serialization` 的签名 pickle `dump`/`load`，而不是裸 pickle，防止被篡改的知识库文件注入恶意代码）。

- **V1**（`CoSTEERKnowledgeBaseV1`/`CoSTEERRAGStrategyV1`）：按注释是**已废弃**的实现，`generate_knowledge`/`query` 方法体第一行就是 `raise NotImplementedError(...)`，后面的代码虽然还留着（基于简单的 dict + embedding 相似度检索），但永远不会被执行到。文档中需要明确告知：**不要参考 V1 的具体检索逻辑，只需知道它的接口形状**。
- **V2**（`CoSTEERKnowledgeBaseV2`/`CoSTEERRAGStrategyV2`，默认启用）：是真正在用的实现，底层用一个无向图 `rdagent/components/knowledge_management/graph.py:UndirectedGraph`（节点类型包括 `component`/`task_description`/`task_trace`/`task_success_implement`/`error`）来组织知识，具体检索/生成逻辑：
  - `generate_knowledge()`：遍历新增的进化轨迹（`evolving_trace`），对每个任务：若本轮成功，调用 LLM 分析该任务涉及哪些"组件"（`analyze_component`，用一个单独的 LLM 调用把任务描述映射到知识图谱里已有的 component 节点列表上），再调用 `update_success_task()` 把整条实现轨迹（从头到成功）写入图谱，建立 `task_description --- task_trace* --- task_success_implement` 的链路，并把每一步的报错也解析成 `error` 节点挂在对应 `task_trace` 节点上；若本轮失败，调用 `analyze_error()` 用正则从 traceback/返回值检查文本中提取"错误类型 + 出错代码行"或预定义的几类"数值检查错误"，存入 `working_trace_error_analysis` 供下次查询使用。
  - `query()` 由三个子查询组成，各自有独立的数量上限配置（`v2_query_former_trace_limit`/`v2_query_component_limit`/`v2_query_error_limit`）：
    1. `former_trace_query`：取该任务自己历史上的失败轨迹（做了一个启发式裁剪——"如果上一次有返回值检查但这一次没有，说明退化了，删掉这条"，代码注释把这类比为"类似错误的梯度下降"）；
    2. `component_query`：按"组件"在知识图谱里查相似的**成功**任务实现，再用 embedding 相似度（`calculate_embedding_distance_between_str_list`）补充候选，并有一条"至少一半结果来自有 Ground Truth 验证的成功实现"的采样规则；
    3. `error_query`：按"上一次出的错误节点"在图谱里查"曾经出现同样错误、后来修好了"的成功案例配对，供 LLM 参考"别人是怎么从这个错误里改好的"。
  - `v2_knowledge_sampler`（0~1 的概率）可以对查询结果做随机采样，避免每次都把全部命中结果塞进 prompt。

注意：V2 的知识库是**跨进程共享的单一 pickle 文件**，`CoSTEER` 构造时传入 `enable_filelock`/`filelock_path`，`RAGEvoAgent.multistep_evolve` 在"知识自生成"步骤会用 `filelock.FileLock` 包一层 `load_dumped_knowledge_base() → generate_knowledge() → dump_knowledge_base()`，这是支持多 Loop 并行跑（`RD_AGENT_SETTINGS` 的并行度设置）时知识库不被脏写的关键机制。

### 3.6 场景化 Coder 实例

- **`factor_coder/`**（`factor.py`/`evolving_strategy.py`/`evaluators.py`/`eva_utils.py`）：面向 Qlib 量化因子。`FactorTask` 携带 `factor_formulation`/`variables`/`resource` 等金融特定字段；`FactorFBWorkspace.execute()` 用 `@cache_with_pickle(hash_func)` 按"因子代码内容的 md5"做结果缓存，执行方式是把因子代码和数据链接到 workspace 后起子进程跑 `python factor.py`（或 v2 版本用一个模板脚本间接 import），超时/异常都转成结构化的 `CustomRuntimeError`/`NoOutputError`，产出 `result.h5`。`FactorMultiProcessEvolvingStrategy.implement_one_task` 展示了一个"动态裁剪 prompt 长度"的通用模式：构造 system/user prompt 后用 `APIBackend().build_messages_and_calculate_token` 检查是否超过 `chat_token_limit`，超了就依次丢弃最旧的失败知识/成功知识/错误知识直到塞得下（最多重试 10 次），这个模式在 `model_coder`/`data_science` 的多个 Coder 里重复出现，是本项目处理"长上下文但 token 有限"的标准写法。`FactorEvaluatorForCoder`（三段式：`FactorValueEvaluator` 比较生成因子值与 Ground Truth 的数值/相关性、`FactorCodeEvaluator` 做代码层面批评、`FactorFinalDecisionEvaluator` 综合判断）展示了"有 GT 就优先用数值比较，没有 GT 才退化到代码审查"的评价优先级。
- **`model_coder/`**：面向 Qlib 模型结构代码（`ModelTask`/`ModelFBWorkspace`/`ModelMultiProcessEvolvingStrategy`），结构与 `factor_coder` 基本同构。额外带了一个 `benchmark/` 子目录（`eval.py` + `gt_code/*.py` 几个真实 GNN 模型实现如 `dirgnn.py`/`gpsconv.py`/`visnet.py`），其中 `ModelImpValEval` 用"固定若干组随机初始化参数+随机输入，比较生成模型和 GT 模型隐层输出的相关系数"来量化"模型结构是否等价"——这是一种不依赖字符串/AST 比较、而是**行为等价性**的评测思路，专门用于离线 benchmark（见 `rdagent/components/benchmark/eval_method.py`，不在本文档范围）而非线上 RDLoop。另有 `one_shot/__init__.py:ModelCodeWriter`，是一个**不走 CoSTEER 进化循环**的极简 `Developer`：对每个子任务只调一次 LLM 把代码写出来，没有评价/重试,是 CoSTEER 之外"最简 Coder"的对照实现。
- **`data_science/`**：面向 Kaggle/MLE-bench 式数据科学竞赛,是目前最复杂、子模块最多的 Coder 群。关键共享基础设施：
  - `share/ds_costeer.py:DSCoSTEER(CoSTEER)`：唯一重写了 `get_develop_max_seconds()`，用 `scen.real_debug_timeout() * settings.max_seconds_multiplier` 做超时预算，所有 data_science 下的具体 Coder 都继承它。
  - `conf.py:DSCoderCoSTEERSettings` + `get_ds_env()`：按配置在 `docker`/`conda` 两种 `Env`（见 `rdagent/utils/env.py` 的 `DockerEnv`/`DSDockerConf`/`MLEBDockerConf`/`CondaConf`/`MLECondaConf`）之间切换,`conf_type` 区分 "kaggle" 和 "mlebench" 两套基准;还提供 `get_clear_ws_cmd()` 生成"训练前/推理前清场"的 shell 命令。
  - 六个子任务类型均继承 `CoSTEERTask` 但基本不加字段，只是用来做 `isinstance` 区分：`raw_data_loader/exp.py:DataLoaderTask`、`feature/exp.py:FeatureTask`、`model/exp.py:ModelTask`、`ensemble/exp.py:EnsembleTask`、`workflow/exp.py:WorkflowTask`、`pipeline/exp.py:PipelineTask`。一个值得注意的设计：**这些子任务通常共享同一个 `experiment_workspace`**（`PipelineMultiProcessEvolvingStrategy.assign_code_list_to_evo`/`WorkflowMultiProcessEvolvingStrategy` 里都是 `evo.sub_workspace_list[index] = evo.experiment_workspace` 后再 `inject_files`），即多个"子任务"其实是在同一份 `main.py`（或多文件管线）上做不同方面的增量编辑，而不是像 `factor_coder` 那样每个任务各自独立的 workspace。
  - `pipeline/` 是当前（single-file pipeline，`main.py`）主推路线，`PipelineCoSTEER.__init__` 展示了"按配置动态叠加 Evaluator"的模式：基础的 `PipelineCoSTEEREvaluator`，若 `DS_RD_SETTING.enable_model_dump` 再叠加 `ModelDumpEvaluator`，再从 `settings.extra_evaluator`/`extra_eval`（字符串类路径列表）用 `core.utils.import_class` 动态加载任意额外评价器——这是比 `BasePropSetting` 更细粒度的"配置里写类路径字符串"插件点。
  - `PipelineCoSTEEREvaluator.evaluate()`（`pipeline/eval.py`）是理解 data_science 评价流程的最佳样本：先清场,用 `strace` 包一层跑 `python -m coverage run main.py [--debug]`拿到 stdout,再做一串**硬编码规则检查**（是否偷看了 sample_submission、`scores.csv` 是否存在且格式正确、是否包含 `ensemble` 行、提交文件格式检查）叠加**LLM 判断**（`build_cls_from_json_with_retry` 构造结构化的 `DSCoderFeedback`，带 `requires_documentation_search` 字段），若 LLM 认为需要查文档且开了 `enable_mcp_documentation_search`,会调用 `rdagent/components/agent/context7` 的 `DocAgent` 去查第三方库文档并把结果追加进 `error_message` 里再返回——这是 `components/agent` 被 `components/coder` 消费的具体例子。硬编码检查和 LLM 判断的结果是"任一方否决即否决"：硬编码检查永远可以把 LLM 原本给出的 `True` 翻转成 `False`,但反过来不行。
  - `share/doc.py:DocDev`、`share/notebook.py:NotebookConverter` 是两个"代码之外的产出物"生成器：前者给 workspace 写 `README.md`,后者把 `main.py` 转成 Jupyter Notebook（要求代码里 `main()` 函数用 `print("Section: ...")` 做分节标记,否则 `validate_code_format` 会报错拒绝转换）。
  - `utils.py:remove_eda_part` 是一个很小但到处用到的工具：从 stdout 里删掉 EDA（探索性数据分析）输出的那一段,避免不相关任务的评价 prompt 被无关的 EDA 文本撑爆。
- **`rl/`**（`costeer.py:RLCoSTEER`）：强化学习后训练的 Coder,目前明显是一个**半成品/占位实现**——`RLCoderEvaluator.evaluate()` 直接 `# TODO: 实现真正的评估逻辑` 后无条件返回 `final_decision=True` 的 mock 反馈;`RLEvolvingStrategy` 里也留了一个没被调用的 `_mock_code()` 兜底方法。构造时 `max_loop=1`、`with_knowledge=False`、`knowledge_self_gen=False`,相当于"只跑一轮、不用知识库"的最简配置,是 CoSTEER 框架可以优雅退化为"单轮生成不验证"模式的证明,但目前不能真正保证生成代码可用。
- **`finetune/`**：LLM 微调场景的环境与路径管理（`conf.py`）,比其它场景多了一层"Docker vs Conda 双模式路径前缀"的抽象（`FTPathConfig`,按 `is_docker` 切换 `/assets/models` 与本地目录,以及 workspace 前缀 `/workspace/` vs `./`），并按操作阶段（`data_processing`/`debug_data_processing`/`micro_batch`/`full_training`）各自配置超时;`get_benchmark_env()` 专门为跑 OpenCompass 基准准备了独立的 Docker/Conda 环境和挂载卷;`get_data_processing_cache_key()` 展示了"缓存键只取决于数据处理脚本+数据集配置,而不受训练配置改动影响"的细粒度缓存策略,避免因为改了训练超参就重跑一遍昂贵的数据预处理。

## `rdagent/components/runner/` —— 结果执行与缓存

这个目录目前只有一个文件 `__init__.py`,内容也很薄：`CachedRunner(Developer[ASpecificExp])` 提供两个方法：

- `get_cache_key(exp)`：把 `exp.based_experiments` 里全部历史任务 + 当前 `exp.sub_tasks` 的 `get_task_information()` 拼接起来做 md5,作为缓存键——即"只要任务描述完全相同（包括其依赖的历史实验),就认为可以复用之前跑过的结果"。
- `assign_cached_result(exp, cached_res)`：把缓存里的 `result` 写回 `exp.result`,并顺带把 `based_experiments[-1].result` 也补上（如果之前没有的话）。

真正"在 Docker 里跑实验、从输出里解析指标、与当前 Trace 里的 SOTA 比较"的逻辑**不在这个目录里**,而是在各 Scenario 包自己的 Runner 子类中（继承 `CachedRunner` 并实现 `develop()`),会在场景文档（05/06/07）中讲。所以这里的"通用组件"本质上只贡献了一个可复用的**缓存 mixin**,而不是完整的执行引擎。

## `rdagent/components/proposal/` —— 通用的假设生成与实验转化

`rdagent/components/proposal/__init__.py` 提供了 `core.proposal.HypothesisGen`/`Hypothesis2Experiment` 两个抽象接口的**LLM 化通用实现**：

- `LLMHypothesisGen(HypothesisGen)`：把"生成假设"这件事拆成两个留给子类实现的钩子——`prepare_context(trace) -> (context_dict, json_flag)`（场景相关：决定塞进 prompt 的历史假设/反馈/RAG 片段,以及输出格式说明）和 `convert_response(response) -> Hypothesis`（把 LLM 的文本/JSON 输出解析成 `Hypothesis` 对象）。`gen()` 方法本身是场景无关的"模板装配 + 调用 `APIBackend` + 解析"的固定流程，并支持传入 `plan.user_instruction` 把外部（人工）指令塞进 system prompt。三个子类 `FactorHypothesisGen`/`ModelHypothesisGen`/`FactorAndModelHypothesisGen` 只是设置了不同的 `self.targets` 字符串（影响用哪个 prompt 模板片段、用哪个 `filtered_tag` 截取场景描述）。
- `LLMHypothesis2Experiment(Hypothesis2Experiment[Experiment])`：结构对称,`prepare_context`/`convert_response` 留给子类,`convert()` 用 `@wait_retry(retry_n=5)` 装饰（见 `rdagent/utils/workflow`),即对 LLM 调用失败/解析失败有 5 次自动重试。对应的三个子类同样只是设置 `self.targets`。

这里体现的设计模式是**模板方法（Template Method）**：把"调用大模型、拼 prompt、解析输出"这类与场景无关的胶水代码写一次放在基类,场景特定的"读哪些历史信息""输出什么格式"留给子类实现——这也是整个 `rdagent/scenarios/*` 目录下大量 `XxxHypothesisGen(LLMHypothesisGen)` 类之所以能保持短小的原因。需要注意：这个文件里**没有**包含"把 Trace 总结成 markdown 喂给 LLM"这类具体的 trace 摘要策略的实现细节,那些通常写在各场景自己的 proposal 模块里（通过 `prepare_context` 的具体实现）,`components/proposal` 本身只给了骨架。

## `rdagent/components/workflow/` —— 胶水层与标准主循环

- `conf.py:BasePropSetting(ExtendedBaseSettings)`：定义了 RDLoop 需要的全部"组件插槽",全部是**字符串类型**（点分类路径,如 `"rdagent.scenarios.xxx.YyyCoder"`),运行时由 `core.utils.import_class` 动态 `import` 并实例化：`scen`/`knowledge_base`/`knowledge_base_path`/`hypothesis_gen`/`interactor`/`hypothesis2experiment`/`coder`/`runner`/`summarizer`,外加一个整数 `evolving_n`（默认 10）。各场景的具体配置类（如 `rdagent/app/data_science/conf.py:DS_RD_SETTING`）继承它并把这些字段填成具体的类路径,这就是整个框架"改配置就能换场景"的插件机制实体。
- `rd_loop.py:RDLoop(LoopBase, metaclass=LoopMeta)`：标准五步循环的具体实现,构造函数按 `PROP_SETTING` 里的字符串逐一 `import_class(...)()` 实例化 `hypothesis_gen`/`hypothesis2experiment`/`coder`/`runner`/`summarizer`,并建一个 `core.proposal.Trace(scen=scen)`。
  - 五个公开方法对应五个步骤：`direct_exp_gen`（异步,内部用 `while True` + `asyncio.sleep(1)` 轮询 `self.get_unfinished_loop_cnt(loop_idx) < RD_AGENT_SETTINGS.get_max_parallel()` 来做**并行 Loop 的限流**,拿到配额才真正调用 `_propose()`+`_exp_gen()`）→ `coding`（`self.coder.develop`）→ `running`（`self.runner.develop`）→ `feedback`（异常则直接构造 `decision=False` 的否定反馈,否则调用 `self.summarizer.generate_feedback`）→ `record`（`self.trace.sync_dag_parent_and_hist`）。
  - **值得指出的一点**：这个文件虽然名义上是"通用" RDLoop,但实际内容里混入了不少 Qlib 场景专属的逻辑——`self.plan`（`ExperimentPlan`,初始值里硬编码了 `ALPHA20` 特征集）、`_init_base_features`/`_interact_init_params` 里对 `base_factors.json`/`validate_qlib_features` 的处理、`from rdagent.utils.qlib import ALPHA20, validate_qlib_features` 的顶层 import。这意味着 `RDLoop` 并非对所有场景都"纯粹通用",**数据科学/金融量化场景在复用它时需要留意这部分 Qlib 专属分支是否适用**,这是一个值得项目维护者关注、可能需要进一步抽象或拆分的点。
  - 另外三个 `_interact_*` 方法（`_interact_hypo`/`_interact_feedback`/`_interact_init_params`）实现了一套**基于 `multiprocessing.Queue` 的人机交互协议**：往 `user_request_q` 放请求、从 `user_response_q` 阻塞等响应,任何 `EOFError`/`OSError`/`TypeError` 都会被捕获并静默退回"不交互,用原值"。这套队列只有被外部显式调用 `_set_interactor()` 注入后才会生效,否则 `hasattr` 检查会跳过——这是和 `components/interactor`（下面会讲）平行的另一套、更底层的"人在回路"机制,两者的关系和边界在代码里没有特别说明,是一个可能需要向维护者确认的点。

## `rdagent/components/agent/` —— 基于 pydantic-ai 的工具调用 Agent 封装

这个目录和 RDLoop 的五步模型没有直接对应关系,它提供的是一个**更底层、可被任何组件复用的"LLM + MCP 工具"封装**：

- `base.py:BaseAgent`（抽象接口,`__init__(system_prompt, toolsets)` + `query(query) -> str`)和 `PAIAgent(BaseAgent)`（具体实现,基于 `pydantic_ai.Agent`,`toolsets` 是一串 MCP（Model Context Protocol）服务器地址,统一转成 `MCPServerStreamableHTTP`）。`PAIAgent` 支持通过 Prefect（`prefect.task(cache_policy=INPUTS, persist_result=True)`)做**跨进程持久化缓存**,需要用户另外起一个 `prefect server start` 并设置 `PREFECT_API_URL`才会生效(`enable_cache` 开关控制是否套用缓存版的 `_cached_query`)。代码里特别注明 `nest_asyncio.apply()` 是"very important",因为 `pydantic-ai` 内部用 asyncio 而调用方（`query()`)是同步接口,需要靠 `nest_asyncio` 打补丁才能在已有事件循环里嵌套运行。
- `context7/__init__.py:Agent(PAIAgent)`：对接 [Context7](https://context7.com/) 文档检索 MCP 服务,专门用于"报错信息 → 查第三方库最新 API 文档"场景。`_build_enhanced_query()` 会把原始 error message 包进专门设计的 prompt 模板,还有一个写死的启发式规则——如果错误信息里 "timm" 这个词出现 ≥3 次,会额外拼接一段"timm 特殊处理"的提示文本(`timm_special_case`模板),说明开发者在实践中专门为 `timm` 库踩过的坑做了特判。这个类正是前面 `data_science/share/eval.py:PipelineCoSTEEREvaluator` 在 `requires_documentation_search=True` 时实际调用的 `DocAgent`。
- `rag/__init__.py:Agent(PAIAgent)`：对接一个通用 RAG MCP 服务,默认 system prompt 是"用检索到的文档回答用户问题",目前在仓库里还看不到被具体场景消费的调用点,更像是预留的通用能力。
- `mcp/__init__.py`：**只有一段文档字符串,没有任何代码**,说明这是预留给"各个 MCP server 的 Settings/health_check 定义"的目录骨架,目前尚未填充具体实现——如果用户在这个目录下找具体 MCP server 配置会发现是空的,属于"预留但未完成"的部分。

## `rdagent/components/interactor/` —— Interactor 接口的默认实现

`rdagent/components/interactor/__init__.py` 只有一个类：`SkipInteractor(Interactor[ASpecificExp])`,其 `interact(exp, trace)` 方法什么都不做,直接原样返回 `exp`。也就是说,`core.interactor.Interactor` 这个人机协同接口目前在 `components` 层**只有一个"什么都不干"的占位实现**,真正可用的人机交互逐字段协议反而是在 `rd_loop.py` 里那套基于 `Queue` 的 `_interact_*` 方法(见上文)。换句话说,`BasePropSetting.interactor` 配置项和 `RDLoop._interact_hypo/_interact_feedback` 是**两条并行存在、尚未统一**的"人在回路"路径,`SkipInteractor` 更像是"关闭 Interactor 功能"时的默认占位符,而不是真正被广泛使用的交互实现。这是文档中值得向维护者确认的又一处设计不一致。

## 本模块与其他模块的关系

- **向上（已有文档）**：本篇大量复用并实例化了 [`02-核心抽象`](/rdagent/core-abstractions/) 中定义的 `Scenario`/`Task`/`Workspace`/`Experiment`/`Developer`/`Hypothesis*`/`Trace`/`LoopBase`/`RDLoop` 等概念——例如 `CoSTEER` 就是 `Developer` 的具体子类、`EvolvingItem` 是 `Experiment`+`EvolvableSubjects` 的多重继承、`RAGEvoAgent` 是 CoSTEER 内层循环的驱动器。阅读本篇前建议先读 02。
- **向下（场景文档，05/06/07）**：本篇介绍的都是"可被复用的骨架/基类"——真正把 `BasePropSetting` 填成具体类路径、把 `CoSTEER` 的 Task/Evaluator/Strategy 填上具体 prompt 与业务规则、把 `CachedRunner` 扩展成真正调用 Docker 跑实验并与 SOTA 比较的逻辑,都发生在各个 Scenario 包（Qlib 量化、Kaggle/MLE-bench 数据科学、LLM 微调等）里,会在后续 05/06/07 文档中具体展开。
- **横向（04-组件支持）**：`rdagent/components/knowledge_management/`（CoSTEER V2 知识库底层依赖的无向图 `UndirectedGraph`）、`rdagent/components/benchmark/`（含 `model_coder/benchmark` 这类"与 Ground Truth 比较"的评测方法)、`rdagent/components/loader/` 与 `rdagent/components/document_reader/` 均在 [`04-components-support.md`](/rdagent/components-support/) 中详细介绍,本篇只在用到的地方做了最少量的提及,不重复展开。
