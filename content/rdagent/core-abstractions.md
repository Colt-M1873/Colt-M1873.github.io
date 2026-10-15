---
title: "02. 核心抽象层（`rdagent/core`）"
date: 2026-10-15T10:02:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

`rdagent/core` 定义了与具体业务场景无关的"骨架"类：场景怎么描述、任务怎么表示、实现怎么存放、怎么产生想法、怎么评价结果、怎么记录历史、怎么把这些串成一个可持久化的循环。理解这一层，是理解任何一个具体场景代码的前提。

本篇按"数据在循环中流动的顺序"介绍这些类，而不是按文件顺序。

## 1. `Scenario`（`core/scenario.py`）：场景上下文

```python
class Scenario(ABC):
    background: str                 # 背景信息
    rich_style_description: str     # 给用户展示用的说明
    def get_source_data_desc(task=None) -> str: ...     # 数据描述（可因 task 不同而不同）
    def get_scenario_all_desc(task=None, filtered_tag=None, simple_background=None) -> str: ...  # 汇总成一段完整描述，通常直接拼进 LLM Prompt
    def get_runtime_environment() -> str: ...  # 运行环境描述（比如用的什么镀像/依赖）
    experiment_setting: str | None  # 实验设置说明（可选）
```

`Scenario` 本质上是"喂给 LLM 的上下文"的结构化来源——几乎所有 Prompt 模板都会插入 `scen.get_scenario_all_desc()`。每个具体场景（Qlib 因子、Kaggle、General Model...）都会实现自己的 `Scenario` 子类，告诉 Agent "我们现在在做什么、数据长什么样、代码跑在什么环境里"。`Scenario` 本身不包含任何"方法/策略"相关的配置（比如 RAG 要不要开），这是有意的设计约束（见类注释）。

## 2. `Task` / `AbsTask`（`core/experiment.py`）：一个可执行的子任务

```python
class AbsTask(ABC):
    name: str
    version: int
    def get_task_information() -> str  # 任务的唯一描述，用于做 key / 去重

class Task(AbsTask):
    description: str
    user_instructions: UserInstructions | None  # 用户追加的高优先级指令
```

一个 `Experiment`（见下）可以包含多个 `Task`（`sub_tasks`），比如"实现 3 个新因子"就是 3 个 `FactorTask`。各场景会继承出自己的 Task 子类（`FactorTask`、`ModelTask`、`PipelineTask` 等），挂上该场景特有的字段（比如因子的公式文本）。

## 3. `Workspace` / `FBWorkspace`（`core/experiment.py`）：任务实现的落地容器

`Workspace` 是抽象基类，`FBWorkspace`（File-Based Workspace）是通用实现——**用一个文件系统目录来表示"一个任务的代码实现"**：

- `file_dict: dict[str, str]`：文件名 → 内容的映射（代码在内存中的"真身"，方便序列化/对比/回滚）。
- `workspace_path`：实际落盘目录（`RD_AGENT_SETTINGS.workspace_path / <uuid>`）。
- `inject_files(**files)`：把 `file_dict` 写入磁盘（值为 `"__DEL__"` 表示删除该文件）；内部会校验路径不能用 `..`/绝对路径"逃出"工作区（防止路径穿越）。
- `execute(env, entry)` / `run(env, entry)`：先 `prepare()` + 把 `file_dict` 落盘，再调用传入的 `Env.run(entry, workspace_path)` 在隔离环境（通常是 Docker，见文档08的 `utils/env.py`）里真正执行代码，返回 `EnvResult`（stdout/exit_code/运行时长）。
- `create_ws_ckp()` / `recover_ws_ckp()`：把当前工作区目录打成 zip 存在内存里做"检查点"，之后可以还原——用于"失败了想回滚到某个历史状态"的场景。
- `copy()`：`deepcopy` 整个工作区，得到一个独立快照（因为 `Experiment` 的历史记录里要保存"当时的实现"，不能被后续修改污染）。

**一句话总结**：`Developer`（Coder/Runner，见下）操作的对象就是 `Workspace`；它们通过 `inject_files` 写代码，通过 `execute`/`run` 把代码丢进 Docker 里跑。

## 4. `Experiment`（`core/experiment.py`）：一轮"实验"的完整记录

```python
class Experiment(Generic[Task, Workspace, Workspace]):
    hypothesis: Hypothesis | None        # 这个实验是为了验证哪个假设
    sub_tasks: Sequence[Task]            # 包含的子任务
    sub_workspace_list: list[Workspace | None]  # 每个子任务对应的实现（初始为 None，由 Coder 填充）
    experiment_workspace: Workspace | None      # 整合后的、可运行的完整工作区（由 Runner 使用/填充）
    based_experiments: Sequence[Workspace]      # 继承自哪些历史实验（增量开发）
    running_info: RunningInfo            # 执行结果（result 字段）
    plan: ExperimentPlan | None          # 规划信息（多阶段场景用）
    local_selection: tuple[int, ...] | None  # 本实验挂在 Trace 的哪个分支节点下（并行多分支用）
```

`Experiment` 把"要做什么（sub_tasks）"和"做出来的东西（sub_workspace_list / experiment_workspace）"绑在一起，并带着"为什么要做（hypothesis）"的上下文，是整个循环里传递的核心数据对象——从 ExpGen 生成，经 Coder 填充代码，经 Runner 填充执行结果，最后被 Experiment2Feedback 消费产出反馈。

## 5. 想法生成链路：`Hypothesis` → `Experiment`（`core/proposal.py`）

```
HypothesisGen.gen(trace, plan) -> Hypothesis
          │
          ▼
Hypothesis2Experiment.convert(hypothesis, trace) -> Experiment
```

- **`Hypothesis`**：`hypothesis`（这次想验证的点子）+ `reason`（为什么这么想）+ 一组 `concise_*` 字段（给后续 Prompt 复用的精简版描述）。是纯文本/结构化对象，不含代码。
- **`HypothesisGen.gen(trace, plan)`**：抽象接口，具体场景用 LLM 读 `trace`（历史上哪些假设成功/失败）生成新的 `Hypothesis`。
- **`Hypothesis2Experiment.convert(hypothesis, trace)`**：把 `Hypothesis` 落地成具体的 `Experiment`（即生成若干 `Task`）。
- **`ExpGen`**：把以上两步合并的高层接口（`gen(trace) -> Experiment`），语义上等价于 `Hypothesis2Experiment().convert(HypothesisGen().gen(trace))`；有些场景（比如结构更复杂的 data_science）会直接重写 `ExpGen.gen`，不严格区分这两步。`ExpGen` 还提供了 `async_gen`，用于"当前并行跑的 loop 数已达上限时，先等一等再生成新实验"的限流逻辑。
- **`ExpPlanner.plan(trace) -> ExperimentPlan`**：可选的前置规划步骤（`ExperimentPlan` 只是个 `dict`），用于需要多阶段规划的复杂场景。

## 6. 反馈链路：执行结果 → `ExperimentFeedback`

```python
class ExperimentFeedback(Feedback):
    decision: bool          # 这次实验是否被采纳（bool(feedback) 就返回这个）
    reason: str
    code_change_summary: str | None
    exception: Exception | None   # 如果实验执行失败，异常会被塞进来

class HypothesisFeedback(ExperimentFeedback):
    observations, hypothesis_evaluation, new_hypothesis, acceptable  # 更丰富的结构化反馈，供下一轮生成假设参考
```

`Experiment2Feedback.generate_feedback(exp, trace, exception=None) -> ExperimentFeedback`：读取已经执行完的 `Experiment`（里面有运行结果、和历史 SOTA 的对比），用 LLM 总结出"这个假设到底对不对、要不要采纳、下一步可以怎么改"。`decision=False` 的实验不会被当作新的 SOTA，但依然会被记录到 Trace 里（失败经验本身也是信息）。

## 7. `Trace`（`core/proposal.py`）：支持分支的研发历史

`Trace` 不是一条链表，而是一个**显式维护父子关系的 DAG**：

- `hist: list[tuple[Experiment, ExperimentFeedback]]`：按"记录顺序"存的历史节点。
- `dag_parent: list[tuple[int, ...]]`：第 `i` 个节点的父节点下标；`()` 表示它是一棵新树的根。
- `current_selection: tuple[int, ...]`：当前"准备往哪个节点后面接新实验"的游标；`SEL_LATEST_SOTA = (-1,)` 是默认值（接在最新节点后面），`NEW_ROOT = ()` 表示另起一棵新树。
- 关键方法：
  - `get_sota_experiment(node_id=None)`：从某节点往上找最近一个 `decision=True` 的实验，即"当前最优解"。
  - `get_parent_exps(selection)`：取出某分支从根到当前的完整路径（供 Prompt 里展示"历史沿革"）。
  - `sync_dag_parent_and_hist(exp_and_fb, loop_id)`：每轮循环的 `record` 步调用，把新结果挂到正确的父节点下面。

之所以设计成 DAG 而不是链表，是为了支持**多分支并行搜索**（同时从不同的历史节点出发尝试不同方向，类似 beam search/MCTS），`CheckpointSelector`（选"从哪个历史节点出发"）和 `SOTAexpSelector`（选"最终提交哪个实验"）就是服务于更高级搜索策略的可插拔接口。

## 8. `Developer`（`core/developer.py`）：把实验"实现"或"运行"的统一接口

```python
class Developer(Generic[Experiment]):
    def develop(self, exp: Experiment) -> Experiment: ...
```

接口非常简单，但约定了一个重要原则：**原地修改 `exp`，而不是只靠返回值**（因为即使中途抛异常，调用方也想要已经写出来的中间结果）。

`Developer` 有两种典型角色（都是它的子类，具体实现在 `components/coder`、`components/runner`，见文档03）：
- **Coder**：只管"把 Task 变成代码"，填充 `exp.sub_workspace_list`，不负责真正跑出数值结果。
- **Runner**：把 Coder 产出的代码整合、执行，填充 `exp.experiment_workspace` 和 `exp.result`。

`rdagent/components/workflow/rd_loop.py` 里的标准循环就是依次调用 `self.coder.develop(exp)` 再 `self.runner.develop(exp)`。

## 9. 循环引擎：`LoopBase` / `LoopMeta`（`utils/workflow/loop.py`）

这是把以上所有抽象"粘"起来、实际跑起来的引擎，设计上有几个关键点：

1. **`LoopMeta` 元类自动收集 "steps"**：子类里定义的公开方法（不以 `_` 开头、非 `load`/`dump`）会被自动按定义顺序收集进 `steps` 列表，不需要手写"注册"代码。约定**最后一个 step 永远是写历史的 `record`**。
2. **`loop_prev_out[loop_idx][step_name]`**：每一步的返回值会存进这个字典，下一步函数接收上一步的完整输出字典作为唯一参数——这是步骤间传参的方式（而不是用成员变量），方便做持久化快照。
3. **异步 + 多条 loop 并行**：`kickoff_loop()` 不断启动新的 `loop_idx`（受 `RD_AGENT_SETTINGS.step_semaphore` 限流），`execute_loop()`（可以开多个协程）从队列里取 loop 继续跑未完成的步骤；每个 step 可以配置自己的并发信号量（通过 `get_semaphore`），`record`/`feedback` 步骤强制并发度为 1（避免并行写 Trace 产生竞态）。
4. **会话持久化**：每跑完一步就调用 `self.dump(session_folder/<loop_idx>/<step_idx>_<step_name>)`，用 `core/serialization.py` 的**带 HMAC 签名的 pickle**存盘（防止 pickle 反序列化攻击：没有签名或签名不对的文件会被拒绝加载，除非显式打开 `ALLOW_UNSAFE_LEGACY_PICKLE`）。`LoopBase.load(path, checkout=...)` 可以从任意一步的快照恢复，`checkout=True` 还会顺带清理掉"比这个快照更晚"的日志，方便"回到过去重新跑"。
5. **三种异常处理策略**（类属性可被子类覆盖）：
   - `skip_loop_error`：命中的异常会跳过当前 loop 剩余步骤，直接跳到 `feedback`（或 `skip_loop_error_stepname` 指定的步骤），把异常塞进 `HypothesisFeedback` 里当作失败反馈记录下来，循环继续。
   - `withdraw_loop_error`：命中的异常会回退整个 loop 到上一个快照状态，并抛出 `LoopResumeError`，上层 `run()` 捕获后重置 `loop_idx=0` 重新调度（用于"状态已经脏了，必须回滚重来"的严重错误）。
   - 其他未声明的异常：直接向上抛出，终止整个 `run()`。
6. **超时/步数控制**：`timer`（`RDAgentTimer`）支持整体运行时长限制；`step_n`/`loop_n` 支持"只跑 N 步/N 轮"用于调试或续跑。

## 10. 标准循环实现：`RDLoop`（`components/workflow/rd_loop.py`）

`RDLoop(LoopBase)` 是绝大多数场景复用的"标准五步循环"：

```
direct_exp_gen  →  coding  →  running  →  feedback  →  record
(propose+转Experiment)  (Coder.develop)  (Runner.develop)  (Experiment2Feedback)  (写入Trace)
```

- `direct_exp_gen`：内部拆成 `_propose()`（调 `hypothesis_gen.gen`，支持 `_interact_hypo` 把假设发给用户确认/修改）和 `_exp_gen()`（调 `hypothesis2experiment.convert`）。
- `feedback`：如果前面步骤抛出了被 `skip_loop_error` 捕获的异常，直接构造一个 `decision=False` 的 `HypothesisFeedback`；否则调用 `summarizer.generate_feedback`。同样支持 `_interact_feedback` 把反馈发给用户确认/修改。
- `record`：调用 `trace.sync_dag_parent_and_hist`。
- 各场景的具体 Loop 类（如 `app/qlib_rd_loop/factor.py` 的 `FactorRDLoop`）通常只需要：声明 `skip_loop_error = (XxxEmptyError, CoderError)`，以及偶尔重写某一步（如 `FactorRDLoop.running` 额外做了"如果没抽出因子就报错"的判断）。

依赖哪些具体类（`hypothesis_gen`、`coder`、`runner`、`summarizer`...）由 `BasePropSetting`（各场景的 `conf.py`）以**类路径字符串**的形式配置，`RDLoop.__init__` 用 `core/utils.py:import_class` 动态 `importlib` 加载——这是整个框架"插件化"的关键机制：换场景只需要改配置指向不同的实现类，循环骨架代码完全不用动。

## 11. 代码进化内循环：CoSTEER 思想（`core/evolving_framework.py` + `core/evolving_agent.py`）

这套机制解决的问题是："LLM 一次写的代码常常跑不通/效果不好，怎么让它在一个任务内部反复修订，直到收敛？"它和上面的 R&D Loop 是**两个不同粒度的循环**，通常被 Coder 组件在内部调用（详见文档03）。

- **`EvolvableSubjects`**：被进化的对象（典型是"这组代码实现"），`clone()` 产生独立快照。
- **`EvolvingStrategy.evolve_iter(evo, queried_knowledge, evolving_trace) -> Generator`**：核心策略，是一个生成器——每 `yield` 一次表示完成了一小步修订（允许多个子任务分批、逐步演进，而不是一次性生成全部）。
- **`IterEvaluator.evaluate_iter() -> Generator`**：配套的生成器式评估——通过 `generator.send(evo)` 把每一步的半成品喂给它，拿到这一步的局部反馈，最后 `return` 一个总体反馈；`evaluate()` 是对这个生成器协议的同步封装。
- **`RAGStrategy`**：检索增强——维护一个 `EvolvingKnowledgeBase`，在每轮进化前 `query()` 相关经验（比如"这类报错以前怎么修的"），进化/评估之后 `generate_knowledge()` 把新经验写回知识库（`dump_knowledge_base`/`load_dumped_knowledge_base` 支持多进程并行编码时共享知识库，配合文件锁 `enable_filelock`）。
- **`RAGEvoAgent.multistep_evolve(evo, eva)`**：驱动整个内循环的"指挥官"——每一轮：(1) 可选地查知识库 (2) 调 `evolving_strategy.evolve_iter` 逐步演进 (3) 同步用 `eva.evaluate_iter` 评估每一小步 (4) 把这一轮的 `(evo, knowledge, feedback)` 存进 `evolving_trace` (5) 可选地让 RAG 自我更新知识库 (6) `yield` 当前状态给外部做日志/早停判断 (7) 如果反馈显示"全部任务已完成"（`feedback.finished()`）则提前跳出，否则最多跑 `max_loop` 轮。

一句话类比：**外层 R&D Loop 像"换一个研究方向"，内层 CoSTEER 循环像"把这个方向的代码改到能跑通为止"**。

## 12. 其它基础设施

- **知识库抽象**（`core/knowledge_base.py`）：`KnowledgeBase`（有 `load`/`dump`）是 `EvolvingKnowledgeBase` 的父类，被 `RAGStrategy` 使用；具体实现（向量库/图谱等）在 `components/knowledge_management`（文档04）。
- **`Interactor`**（`core/interactor.py`）：human-in-the-loop 的抽象接口 `interact(exp, trace) -> exp`。`RDLoop._interact_hypo` / `_interact_feedback` 就是它的一种具体落地方式——通过 `multiprocessing.Queue` 把假设/反馈发给前端（比如 `rdagent ds_user_interact` 的 Streamlit 页面），等用户编辑后再返回继续跑。
- **配置 `RD_AGENT_SETTINGS`**（`core/conf.py`，基于 pydantic-settings）：工作区路径、`step_semaphore`（并行度/每步限流）、pickle 缓存开关与路径、`artifact_signing_key`（会话快照签名密钥）等。`ExtendedBaseSettings` 让子类配置类可以"继承"父类配置类的环境变量来源，这是各场景自己的 `conf.py` 普遍继承 `RDAgentSettings`/`BasePropSetting` 的基础。
- **异常体系 `core/exception.py`**：`WorkflowError`（大类）下有 `FormatError`/`CodeBlockParseError`（LLM 输出格式解析失败）、`CoderError` 及其子类 `CodeFormatError`/`CustomRuntimeError`/`NoOutputError`（代码实现/执行各阶段的错误，`FactorEmptyError`/`ModelEmptyError` 是它的别名）；独立的 `RunnerError`、`KaggleError`、`PolicyError`（触发内容审核策略）、`EvaluatorDidNotTerminateError`。这些异常被各场景的 `skip_loop_error` 声明捕获，转化成"失败反馈"而不是直接崩溃退出。
- **安全序列化 `core/serialization.py`**：给 pickle 数据加 HMAC-SHA256 签名前缀，签名密钥来自配置或首次运行时在 `~/.rdagent/artifact_signing.key` 自动生成（带文件锁防并发写冲突）；加载时校验签名，防止"篡改过的/不可信的 pickle 文件被反序列化执行任意代码"这个经典安全风险。会话快照（`LoopBase.dump/load`）和一些结果缓存（`core/utils.py:cache_with_pickle`）都基于它。
- **`core/utils.py` 杂项**：`SingletonBaseClass`（基于 kwargs 哈希的单例，故意设计成不可 pickle，避免反序列化出"假单例"）、`import_class`（按字符串路径动态加载类，驱动上文提到的插件化配置）、`multiprocessing_wrapper`（带"确定性缓存种子传递"的多进程并行调用封装，保证并行跑和串行跑在开启 LLM 缓存时能复现同样的调用序列）、`cache_with_pickle` 装饰器（给任意函数加基于参数哈希的安全 pickle 缓存）。

## 13. 小结：一张图看懂依赖关系

```
Scenario  ──────────────────┐ (提供上下文描述，被几乎所有下游类持有 self.scen)
                             │
Trace(History of Experiment+Feedback)
   │  ▲
   │  │ record
   ▼  │
HypothesisGen.gen ──► Hypothesis ──► Hypothesis2Experiment.convert ──► Experiment(sub_tasks=[Task,...])
                                                                            │
                                                               Coder(Developer).develop
                                                                            │ 填充 sub_workspace_list（内部可能跑 CoSTEER 小循环）
                                                                            ▼
                                                               Runner(Developer).develop
                                                                            │ 填充 experiment_workspace / result（在 Docker 中执行）
                                                                            ▼
                                                     Experiment2Feedback.generate_feedback ──► HypothesisFeedback
                                                                            │
                                                                            ▼
                                                                 Trace.sync_dag_parent_and_hist（回到顶部）
```

整套骨架由 `LoopBase`/`RDLoop` 编排执行、持久化、并行化、异常处理。下一篇（[03. 研发组件](/rdagent/components-development/)）会具体展开 Coder/Runner/Proposal 在 `components/` 下是怎么实现的。
