---
title: "04. 支撑型组件（`rdagent/components/*`，非主循环部分）"
date: 2026-10-15T10:04:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

本篇介绍四个"支撑型"（supporting）组件目录：`knowledge_management`、`benchmark`、`loader`、`document_reader`。它们和文档03要讲的 Coder/Runner/Proposal 不同——**不是** `Scenario → HypothesisGen → ... → Trace.record` 这条主循环骨架的必经环节，而是各个具体场景按需"opt-in"选用的基础设施：

- 想让 CoSTEER 编码内循环具备"记忆"能力 → 用 `knowledge_management`；
- 想量化评价 RD-Agent 自身的生成质量，或评测训练出来的模型 → 用 `benchmark`；
- 想从已有的因子库/模型库/JSON 种子数据构造出 `Task`/`Experiment` 来启动一轮研发 → 用 `loader`；
- 想从 PDF 研报/论文里抽取结构化信息（公式、模型描述）喂给 LLM → 用 `document_reader`。

去掉这四个目录，`rdagent/core` + `components/workflow`（`RDLoop`）+ `components/coder`/`runner` 仍然能跑完整的研发循环；但很多场景（финансовый因子报告挖掘、从论文实现模型、Kaggle 领域知识积累、RL/LLM 自身能力评测）离不开它们。

## 1. `rdagent/components/knowledge_management/`：向量库与知识图谱

这是 `core/knowledge_base.py` 中 `KnowledgeBase`/`EvolvingKnowledgeBase` 抽象接口的**通用具体实现**，被 CoSTEER（文档03里讲的代码进化内循环）的 `RAGStrategy` 直接复用，用来让"写代码—报错—改代码"这个过程具备跨任务、跨进程的检索增强记忆。

### 1.1 向量库：`vector_base.py`

- `KnowledgeMetaData`（别名 `Document`）：知识的最小单元，`content` 是文本，`id` 由 `uuid.uuid3(NAMESPACE_DNS, content)` 从内容本身派生（相同内容天然得到相同 id，天然去重）；`create_embedding()` 调 `APIBackend().create_embedding()` 生成整体的 embedding；`split_into_trunk(size, overlap)` 把长文本按固定长度切片（朴素的字符级分片，不考虑语义边界），再批量生成每个分片的 embedding，用于后续"分片级"检索。
- `contents_to_documents(contents, label)`：批量把字符串列表转成 `Document` 列表，考虑到 OpenAI embedding 接口单次最多 16 条输入，按 16 为一批调用 `APIBackend().create_embedding`。
- `VectorBase`（继承 `KnowledgeBase`）：抽象基类，只声明 `add()`/`search()`。
- `PDVectorBase`：用一个 **pandas DataFrame**（列：`id/label/content/trunk/embedding`）做向量存储的朴素实现——没有专门的向量索引（如 FAISS/HNSW），`search()` 是对 DataFrame 里每一行的 embedding 调 `scipy.spatial.distance.cosine` 算余弦相似度，再 `nlargest(topk_k)` 取前 K。支持 `similarity_threshold` 过滤和 `constraint_labels` 按标签过滤。**这是一个全量暴力检索（O(N) 扫描），没有做近似最近邻优化**，适合中小规模知识库（量级是"历史上失败/成功过的编码经验"，不是海量文档）。

### 1.2 知识图谱：`graph.py`

- `UndirectedNode`（继承 `Node = KnowledgeMetaData`）：图节点，`neighbors: set[UndirectedNode]`，`add_neighbor`/`remove_neighbor` 是双向维护邻接关系。
- `Graph`（继承 `KnowledgeBase`）：`nodes: dict[id, Node]`，`get_all_nodes_by_label_list`、`find_node(content, label)`（按内容+标签精确查找）、`batch_embedding`（批量生成一组节点的 embedding，同样是 16 条一批调 `APIBackend`）。
- `UndirectedGraph`：真正可用的实现，内部持有一个 `PDVectorBase` 做"节点内容 → 节点"的语义检索通道（图的"连接关系"和"语义相似度"是两套互补的检索方式）：
  - `add_node(node, neighbor, same_node_threshold=0.95)`：按 id 或"内容+标签完全相同"去重（去重逻辑里注释掉了一段基于语义相似度阈值 0.95 判重的代码，**当前版本并未真正按语义相似度合并节点，只按精确内容匹配**——这是一处值得注意的"未完全实现"的设计意图）。
  - `get_nodes_within_steps(start_node, steps, constraint_labels, block)`：BFS 遍历，`block=True` 表示除起点外只能走经过 `constraint_labels` 类型节点的边（限定搜索必须"穿过"某类中间节点）。
  - `get_nodes_intersection` / `graph_query_by_intersection`（在 CoSTEER 侧子类中）：多个起点各自 BFS 后求交集，交集出现频率越高排序越靠前——用于"多个 component 节点共同关联的 task_description 节点"这类查询。
  - `semantic_search(node, similarity_threshold, topk_k, constraint_labels)`：委托给内部的 `vector_base.search()`，再把搜到的 `Document.id` 映射回图节点——**图的语义检索本质是向量库检索 + id 反查**。
  - `query_by_node`/`query_by_content`：组合"语义检索找起点"+"BFS 按连接关系扩展"+"可选的 constraint_node 距离约束"，是图谱检索的主入口。
  - 模块底部还有一组与图谱检索无关的纯工具函数（`graph_to_edges`、`assign_random_coordinate_to_node`、`assign_isometric_coordinate_to_node`、`curly_node_coordinate`）：给前端可视化知识图谱布局用的坐标生成器，和检索逻辑本身无关。

### 1.3 被谁使用：CoSTEER 的 `RAGStrategy` 具体实现

`rdagent/components/coder/CoSTEER/knowledge_management.py` 是这套向量库/图谱的最重要消费者（文档03会详细讲 CoSTEER 本身，这里只讲它怎么用知识库）：

- `CoSTEERKnowledgeBaseV2`（继承 `core/evolving_framework.py` 的 `EvolvingKnowledgeBase`）内部持有一个 `UndirectedGraph`（`self.graph`），节点按 `label` 区分四类语义：`component`（预置的代码组件/知识点，如"如何处理缺失值"）、`task_description`（任务描述文本）、`task_trace`（某次失败尝试的代码+反馈文本）、`error`（归一化后的报错类型/报错行信息）、`task_success_implement`（最终成功的实现）。`update_success_task()` 在某任务首次成功时，把整条"失败轨迹→成功实现"串成一条图上的邻接链，并把轨迹中出现的报错都挂到 `error` 节点上——这就是"记住哪类报错曾经怎么被修复"的具体落地方式。
- `CoSTEERRAGStrategyV2`（继承 `core/evolving_framework.py` 的 `RAGStrategy`）：
  - `generate_knowledge(evolving_trace)`：每轮进化后扫描新增的 `EvoStep`，调用 LLM 做 `analyze_component`（把任务描述归类到已有的 component 节点列表，走 `json_mode` 的 chat completion）和 `analyze_error`（用正则从 traceback/校验失败文本里抽出 `ErrorType`/出错行，或匹配几个硬编码的数值校验错误类型字符串），然后调 `knowledgebase.update_success_task` 落图。
  - `query(evo, evolving_trace)` 分三步组合查询结果（`CoSTEERQueriedKnowledgeV2`）：`former_trace_query`（这个任务自己之前失败过几次，带上最近几次失败轨迹）、`component_query`（按 component 节点的图交集 + embedding 相似度，找"做法相似的历史成功任务"，并保证至少一半结果来自"有 ground-truth 验证过"的知识）、`error_query`（按当前错误类型在图上找"曾经犯过同样错误、后来又成功了"的案例对）。
  - 配置开关集中在 `CoSTEERSettings`（`components/coder/CoSTEER/config.py`）：`fail_task_trial_limit`（默认20，超过这个次数的任务被标记为"放弃重试"）、`v2_query_component_limit`/`v2_query_error_limit`/`v2_query_former_trace_limit`（各类查询返回条数上限，默认都很小：1/1/3）、`v2_knowledge_sampler`（对查到的知识做随机采样比例，默认1.0即不采样）、`v2_add_fail_attempt_to_latest_successful_execution`（是否把"最近一次修正尝试"也附带给 Agent，默认关闭）。
  - `CoSTEERRAGStrategy.dump_knowledge_base()`/`load_dumped_knowledge_base()`：用 `core/serialization.py` 的安全 pickle 落盘/加载——支持多个并行编码进程共享同一份知识库文件（配合文件锁，呼应文档02提到的 `enable_filelock`）。
  - 另外还留有 `CoSTEERKnowledgeBaseV1`/`CoSTEERRAGStrategyV1`，源码里明确标注"**it is deprecated**"、方法体直接 `raise NotImplementedError`，只作为历史兼容占位，不应再使用。

### 1.4 场景级扩展：Kaggle 知识图谱

`rdagent/scenarios/kaggle/knowledge_management/graph.py` 的 `KGKnowledgeGraph` 直接继承 `components/knowledge_management/graph.py` 的 `UndirectedGraph`，加上 Kaggle 场景特有的领域知识加载逻辑（从 `KAGGLE_IMPLEMENT_SETTING.domain_knowledge_path` 读取文档）和按时间戳生成新知识库文件名的策略。说明 `components/knowledge_management` 是作为**可被场景继承扩展的通用基类**设计的，而不是只能原样使用。

## 2. `rdagent/components/benchmark/`：RD-Agent 自身能力的评测框架

这个目录下其实承载了**两类不完全相同的"评测"**，共享配置/工具代码但服务于不同的评测对象：

### 2.1 因子生成质量评测（`eval_method.py` + `conf.py`）

用于回答"给定一个 `Developer`（比如某个 Coder 实现），它生成因子代码的正确率/相关性到底如何"，是 RD-Agent 项目组用来横向比较不同编码 Agent 实现效果的内部基准，入口脚本在 `rdagent/app/benchmark/factor/eval.py`。

- `BenchmarkSettings`（`conf.py`，环境变量前缀 `BENCHMARK_`）：`bench_data_path`（默认 `./example.json`，即同目录下的 `example.json`——真实的"题库"，每条形如 `{因子名: {description, formulation, variables, gt_code, Category, Difficulty}}`，`gt_code` 是人工写好的标准答案 pandas 实现）、`bench_test_round`（重复跑几轮，默认10，因为 LLM 输出有随机性）、`bench_method_cls`（要测试的方法类路径，默认 `rdagent.components.coder.factor_coder.FactorCoSTEER`，和 `core/utils.py:import_class` 的插件化加载机制一致）、`bench_result_path`。
- `TestCase`/`TestCases`（`eval_method.py`）：把题库里的每一条包装成 `(target_task: FactorTask, ground_truth: FactorFBWorkspace)`，`get_exp()` 把所有 `target_task` 打包成一个 `Experiment` 丢给被测的 `Developer.develop()`。
- `BaseEval`：持有一组 `FactorEvaluator`（具体指标类来自 `components/coder/factor_coder/eva_utils.py`：`FactorSingleColumnEvaluator`/`FactorRowCountEvaluator`/`FactorIndexEvaluator`/`FactorEqualValueRatioEvaluator`/`FactorCorrelationEvaluator` 等，分别检查"是否单列输出""行数是否一致""索引是否一致""数值相等比例""与标准答案的相关系数"），`eval_case(case_gt, case_gen)` 逐个评估器跑一遍，异常会被 `catch_eval_except` 控制是否吞掉。
- `FactorImplementEval(BaseEval)`：`develop()` 循环跑 `test_round` 轮 `generate_method.develop(...)`，`eval()` 用 `core/utils.py:multiprocessing_wrapper` 并行跑 `eval_case`，`summarize_res()` 把结果整理成一个 `pandas.DataFrame`（行是"因子名+该次生成的唯一标识"，列是各个评估器名字，值是具体指标）方便后续统计分析（`app/benchmark/factor/analysis.py` 消费这份结果做进一步画图/统计）。
- 模型侧的对应评测在 `rdagent/app/benchmark/model/eval.py`（配套 `README.md`），思路类似，但针对 `ModelTask`/`ModelFBWorkspace`，这里不单独展开。

### 2.2 LLM/训练产出模型的外部能力评测（OpenCompass 集成）

`configs/opencompass_template.yaml`、`configs/models.yaml`、`utils.py` 以及 `__init__.py` 里的 `BENCHMARK_CONFIGS_DIR` 是另一条线——服务于 `rdagent/scenarios/rl/autorl_bench/core/opencompass.py` 的 `OpenCompassEvaluator`，用来评测"RL/微调场景训练出来的模型，在 gsm8k/bbh/ARC 等公开数据集上表现如何"：

- `opencompass_template.yaml`：一个 Jinja2 风格的 OpenCompass 配置文件模板（显式 `from ... import ...` 而不是 `import *`，避免把 `os`、临时变量等不可序列化对象带进 mmengine 的配置系统），模板变量包括 `dataset_imports`（数据集模块路径+要导入的变量名列表）、`test_range`（如 `"[:100]"` 做验证集切片）、`num_runs`/`pass_k`（多次采样/pass@k 评测）、模型侧的 `tensor_parallel_size`/`gpu_memory_utilization`/`max_seq_len` 等 vLLM 推理参数。
- `models.yaml`：按模型名覆盖默认推理超参（`temperature`/`top_p`/`top_k`/`max_seq_len`/`enable_thinking` 等），比如 Qwen3 系列需要打开 `enable_thinking`。
- `utils.py:build_dataset_imports_explicit(dataset_imports)`：实际 `importlib.import_module` 每个数据集配置模块，自动找出其中名为 `datasets` 或以 `_datasets` 结尾的 list 变量作为显式导入名；找不到时按约定（`<父包名>_datasets`）猜测并打印警告——这是为了绕开 OpenCompass 配置习惯用 `import *` 导出数据集变量、但显式导入更安全这个矛盾。
- `OpenCompassEvaluator.run_eval()`（在 `scenarios/rl/autorl_bench` 下，不属于 `components/benchmark` 本身但是其主要调用方）：校验模型路径存在、拒绝未合并的 LoRA adapter（要求先 `merge_and_unload()`）、拼好配置后用 `subprocess` 实际拉起 OpenCompass 跑评测。

**小结**：`components/benchmark` 与其说是单一"基准类"，更像是"评测基础设施的公共目录"——`eval_method.py`/`conf.py` 服务于"评价 RD-Agent 编码方法本身"，`configs/`+`utils.py` 服务于"用标准 LLM benchmark（OpenCompass）评价场景产出的模型"，两者共享 `BENCHMARK_` 配置前缀的设计风格，但调用链路彼此独立。

## 3. `rdagent/components/loader/`：从外部数据构造 Task/Experiment 的通用接口

对应文档02提到的抽象接口 `Loader[TaskOrExperiment].load(*args, **kwargs)` 和 `WsLoader[Task, Workspace].load(task)`，`components/loader/` 里只放了**跟具体场景无关的最通用部分**（任务/实验级别的抽象占位 + 一个从 JSON 加载模型任务的现成实现），真正"有意思"的 Loader 实现大多下沉到了具体场景目录（`scenarios/qlib/factor_experiment_loader/`、`components/coder/model_coder/task_loader.py`）。

- `task_loader.py`：
  - `FactorTaskLoader`/`ModelTaskLoader`：分别是 `Loader[FactorTask]`/`Loader[ModelTask]` 的空占位子类，起"类型锚点"作用，具体场景再继续往下继承。
  - `ModelTaskLoaderJson`：真正有实现的类——从一个 JSON 文件（格式 `{model_name: {description, formulation, variables, model_type}}`）批量构造出 `ModelTask` 列表。源码里保留了一大段注释掉的旧版本代码和一条 `# FIXME: the model in the json file is not right due to extraction error` 的注释，说明这条路径（JSON 种子库）目前主要靠人工/半自动方式维护，抽取出来的公式偶有错误需要人工修正。
  - `ModelWsLoader(WsLoader[ModelTask, ModelFBWorkspace])`：给定一个目录 `path`，按 `task.name` 找同名 `.py` 文件读进来，`inject_files` 塞进一个新建的 `ModelFBWorkspace`——用于"已经有现成实现代码，只是要包装成标准 Workspace 对象"的场景（比如加载历史已验证过的模型实现）。
- `experiment_loader.py`：`FactorExperimentLoader`/`ModelExperimentLoader` 同样是 `Loader[FactorExperiment]` 的空占位子类（`ModelExperimentLoader` 的类型参数写成 `FactorExperiment` 看起来像是复制粘贴遗留的小瑕疵，而非有意为之，目前代码中未见对它的直接实例化使用，影响有限）。

### 3.1 真正被使用的具体 Loader 实现（在场景目录里，但靠 `components/loader` 的基类打通类型）

- `rdagent/scenarios/qlib/factor_experiment_loader/json_loader.py`：
  - `FactorExperimentLoaderFromDict`：把 `{factor_name: {description, formulation, variables}}` 的 dict 转成一组 `FactorTask`，包进 `QlibFactorExperiment`——是整条"从 JSON/PDF 抽取因子"链路的落脚点。
  - `FactorExperimentLoaderFromJsonFile`/`FactorExperimentLoaderFromJsonString`：在 `FactorExperimentLoaderFromDict` 外面包一层文件/字符串解析。
  - `FactorTestCaseLoaderFromJsonFile`：专门给 2.1 节的 benchmark 用——除了造 `FactorTask`，还把题库里的 `gt_code` 注入一个 `FactorFBWorkspace` 当作 ground truth，组装成 `TestCases`。
- `rdagent/scenarios/qlib/factor_experiment_loader/pdf_loader.py` 的 `FactorExperimentLoaderFromPDFfiles`：**打通 `document_reader` → LLM 抽取 → `loader` 的完整链路**（详见第4节），最终也是调 `FactorExperimentLoaderFromDict().load(...)` 收尾。
- `rdagent/components/coder/model_coder/task_loader.py` 的 `ModelExperimentLoaderFromDict`/`ModelExperimentLoaderFromPDFfiles`：模型侧的对应实现，逻辑结构和因子侧几乎对称（从 PDF 抽取模型描述 → 合并去重 → 转成 `ModelTask`）。

这些具体 Loader 在 `rdagent/app/qlib_rd_loop/factor_from_report.py`（财报因子挖掘场景的核心循环，见文档05/06）、`rdagent/app/general_model/general_model.py`（通用模型复现场景，见文档07）里被直接调用，是"种子数据 → 可执行 Experiment"这一步的具体承担者。

## 4. `rdagent/components/document_reader/`：PDF 解析与结构化抽取

只有一个文件 `document_reader.py`，提供两条互不依赖的 PDF 读取路径，外加一个取首页截图的小工具：

- **路径一：LangChain + PyPDF（默认，无需额外密钥）**
  - `load_documents_by_langchain(path)`：`path` 是目录则用 `PyPDFDirectoryLoader(path, silent_errors=True)`（单个坏文件不中断整体加载），是单个文件则用 `PyPDFLoader(path)`。
  - `process_documents_by_langchain(docs)`：把 LangChain 返回的按"页"切分的 `Document` 列表，按来源文件名（`doc.metadata["source"]`，若文件存在则 resolve 成绝对路径作 key）合并成 `{文件路径: 全文内容}` 的 dict。
  - `load_and_process_pdfs_by_langchain(path)`：上面两步的组合，是**全项目最常用的 PDF 读取入口**（因子/模型 PDF 抽取链路、`factor_from_report.py` 的研报正文读取都用它）。这条路径走纯文本抽取（`PyPDFLoader` 底层是 `pypdf`），**不解析表格结构、不识别公式图片**，只拿得到可被 pypdf 提取的文本层。
- **路径二：Azure Document Intelligence（可选，需要配置密钥）**
  - `load_and_process_one_pdf_by_azure_document_intelligence(path, key, endpoint)`：先用 `PyPDFLoader` 数一下页数，再调 `azure.ai.formrecognizer.DocumentAnalysisClient.begin_analyze_document("prebuilt-document", file, pages=f"1-{pages}")`，用 Azure 的通用文档理解模型（能更好地处理版式、表格）抽取 `.content`。
  - `load_and_process_pdfs_by_azure_document_intelligence(path)`：对目录递归找所有 `.pdf` 调上面的函数；入口处 `assert RD_AGENT_SETTINGS.azure_document_intelligence_key is not None` 和对应 `endpoint` 的断言——这两个配置项定义在 `rdagent/core/conf.py`（`azure_document_intelligence_key`/`azure_document_intelligence_endpoint`，默认空字符串），**不配置的话这条路径直接断言失败，无法使用**；代码库里目前没有发现其它地方调用这两个 Azure 函数（搜索范围内只有默认的 LangChain 路径被实际的场景代码使用），更像是"预留的、效果更好但依赖外部付费服务的备选路径"。
  - `extract_first_page_screenshot_from_pdf(pdf_path)`：用 `fitz`（PyMuPDF）打开 PDF（支持本地路径或先用 `requests.get` 下载远程 URL 到内存），取第一页渲染成 `PIL.Image`——用于日志/前端展示"这份报告长什么样"，和正文抽取无关。

### 4.1 下游怎么用：PDF → 结构化因子/模型描述 → `Loader`

`document_reader` 本身只返回"纯文本"，真正的"结构化抽取"（公式、变量、分类相关性判断）是在调用方用一系列 **LLM 多轮对话 + JSON 模式**完成的：

- 因子侧，`rdagent/scenarios/qlib/factor_experiment_loader/pdf_loader.py` 的 `FactorExperimentLoaderFromPDFfiles.load()` 串了一条相当长的流水线：`load_and_process_pdfs_by_langchain` 读正文 → `classify_report_from_dict`（LLM 判断这份研报是否属于"金工因子"类，支持多数投票 `vote_time`）→ `extract_factors_from_report_dict`（对筛出来的研报，用 `multiprocessing_wrapper` 并行调 LLM，分两步：先抽因子名+描述，再单独一轮抽公式+变量，都用 `session.build_chat_completion(json_mode=True)` 循环直到抽完或达到重试上限）→ `merge_file_to_factor_dict_to_factor_dict`（多文件间按公式长度简单去重合并）→ `check_factor_viability`（LLM 判断因子是否可实现）→ 最终交给 `FactorExperimentLoaderFromDict().load(...)`（即第3节的 `loader`）转成 `QlibFactorExperiment`。模块里还实现了一套基于 embedding + KMeans 聚类 + LLM 判重的 `deduplicate_factors_by_llm`（但在 `FactorExperimentLoaderFromPDFfiles.load()` 里这行调用目前被注释掉，未接入主流程）。
- 模型侧对称的实现在 `rdagent/components/coder/model_coder/task_loader.py`：`ModelExperimentLoaderFromPDFfiles.load()` = `load_and_process_pdfs_by_langchain` → `extract_model_from_docs`（逐文档调 LLM，用正则 `` ```json(.*)``` `` 从回复里抠出 JSON 块，而不是 `json_mode`）→ `merge_file_to_model_dict_to_model_dict` → `ModelExperimentLoaderFromDict().load(...)`。这条路径被 `rdagent/app/general_model/general_model.py`（直接把一篇论文 PDF 转成可运行模型实现）和 `rdagent/scenarios/rl/autorl_bench/core/opencompass.py` 一侧的模型信息抽取复用。
- `rdagent/app/qlib_rd_loop/factor_from_report.py` 的 `FactorReportLoop`（财报因子挖掘场景的标准 `RDLoop`）在 `direct_exp_gen` 步骤里直接调用 `extract_hypothesis_and_exp_from_reports()`：先用 `FactorExperimentLoaderFromPDFfiles().load(report_file_path)` 拿到因子 `Experiment`，再用 `load_and_process_pdfs_by_langchain` 把全文喂给 LLM 生成这份报告对应的 `Hypothesis`（研报里提出的因子"为什么应该有效"），把 PDF 解析、因子抽取、假设生成三件事串成了场景循环的一步。

## 与其他模块的关系

- 这四个组件都依赖并复用 [02. 核心抽象层](/rdagent/core-abstractions/) 中定义的接口与数据结构：`KnowledgeBase`/`EvolvingKnowledgeBase`（`knowledge_management` 的父类）、`Loader`/`WsLoader`（`loader` 的父类）、`Task`/`Experiment`/`FBWorkspace`（`loader` 产出的目标类型，也是 `document_reader` 抽取结果最终要落地成的形态）。
- `knowledge_management` 的核心消费方是 [03. 研发组件](/rdagent/components-development/) 里讲的 CoSTEER 代码进化内循环——具体落地类 `CoSTEERRAGStrategyV2`/`CoSTEERKnowledgeBaseV2` 在 `rdagent/components/coder/CoSTEER/knowledge_management.py`，是 `core/evolving_framework.py` 的 `RAGStrategy`/`EvolvingKnowledgeBase` 接口的具体实现，也是 Coder 组件内部真正"用"知识库的地方。
- `loader` 和 `document_reader` 主要被财报因子挖掘、论文模型复现这类"需要从外部材料里种出第一个 `Experiment`"的场景使用，具体场景文档见 05/06/07（Qlib 因子场景、通用模型复现场景等）——这些文档会讲到 `FactorReportLoop`、`GeneralModelScenario` 等场景类是怎么把本篇介绍的 Loader/PDF 解析拼进一个完整可跑的 `RDLoop` 的。
- `benchmark` 相对独立，不在任何场景的标准运行路径上，是项目组用来离线评价"编码 Agent 效果"或"训练出的模型在公开数据集上的能力"的工具链，入口在 `rdagent/app/benchmark/`。
