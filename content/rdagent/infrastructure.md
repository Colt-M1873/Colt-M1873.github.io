---
title: "08. 基础设施：LLM 调用层 / 日志追踪与 UI / 代码执行环境"
date: 2026-10-15T10:08:00+08:00
draft: false
series: ["RD-Agent 源码解读"]
ShowToc: true
TocOpen: false
---

前面几篇文档讲的是"骨架怎么搭"（`core`）、"研发组件怎么实现"（`components`）、"具体场景怎么落地"（`05`-`07`）。这些内容全都要落在三块真正干"脏活"的基础设施上：**怎么调 LLM**、**怎么把每一步跑了什么记下来并展示给人看**、**代码到底在哪里被执行**。本篇就是讲这三块，外加几个零散但值得知道的小工具目录。

## 1. LLM 调用层 `rdagent/oai/`

### 1.1 统一配置：`LLM_SETTINGS`（`oai/llm_conf.py`）

`LLMSettings(ExtendedBaseSettings)` 是基于 pydantic-settings 的全局配置对象（实例 `LLM_SETTINGS`），关键字段：

- **后端选择**：`backend: str = "rdagent.oai.backend.LiteLLMAPIBackend"`——一个**类路径字符串**，和 `core/utils.py` 的 `import_class` 插件机制一脉相承：想换后端，改这一个字符串即可，不用改调用代码。
- **模型**：`chat_model`（默认 `"gpt-4-turbo"`）、`embedding_model`（默认 `"text-embedding-3-small"`）、`chat_model_map`（一个 `dict`，可以按"日志 tag"把不同调用路由到不同模型/温度/`reasoning_effort`，比如给某个特定 Coder 步骤单独配置更强的模型）。
- **采样参数**：`chat_temperature`（默认 0.5）、`chat_max_tokens`、`chat_stream=True`、`chat_token_limit=100000`、`system_prompt_role`（默认 `"system"`，部分模型如 o1 系列需要改成别的角色名）、`reasoning_effort`、`enable_response_schema=True`（是否允许走结构化输出）、`reasoning_think_rm=False`（是否剥离回复里的 `<think>` 标签）。
- **重试与失败阈值**：`max_retry=10`、`retry_wait_seconds=1`、`timeout_fail_limit=10`、`violation_fail_limit=1`（内容审核触发次数上限）。
- **缓存**：`use_chat_cache`/`dump_chat_cache`/`use_embedding_cache`/`dump_embedding_cache`（四个开关默认全为 `False`，读/写分离控制）、`prompt_cache_path`（默认当前目录下的 `prompt_cache.db`）、`use_auto_chat_cache_seed_gen=False`、`init_chat_cache_seed=42`。
- **Embedding 相关**：`embedding_max_str_num=50`、`embedding_max_length=8192`。
- **历史遗留字段**：文件里有大量 Azure（`chat_use_azure`/`embedding_use_azure`/`managed_identity_client_id`...）、离线 Llama2、GCR 端点相关字段，源码注释明确写着"这些基本只给 `deprec.DeprecBackend` 用"——即它们是为了兼容旧后端留下的，LiteLLM 默认路径基本不用。
- `LiteLLMSettings(LLMSettings)`（`backend/litellm.py`）在此基础上加了 `env_prefix="LITELLM_"`。

### 1.2 调用入口：`APIBackend`

场景/组件代码调用 LLM 永远走同一个口子：

```python
from rdagent.oai.llm_utils import APIBackend
resp = APIBackend().build_messages_and_create_chat_completion(
    system_prompt=..., user_prompt=..., json_mode=True,
)
```

`APIBackend` 其实是 `llm_utils.py` 里 `get_api_backend()` 的别名：调用时内部用 `import_class(LLM_SETTINGS.backend)(...)` 动态加载当前配置的后端类并实例化——所以 `APIBackend()` 永远拿到的是"当前配置选中的那个具体后端"，但对调用方而言是统一接口。抽象基类是 `rdagent.oai.backend.base.APIBackend(ABC)`，提供缓存、重试、自动续写、输出解析等**所有后端共享**的逻辑；具体后端只需实现 `supports_response_schema`、`_calculate_token_from_messages`、`_create_embedding_inner_function`、`_create_chat_completion_inner_function` 这几个钩子。全仓库有 20 多处直接调用 `APIBackend()`，覆盖 `scenarios/qlib`、`scenarios/kaggle`、`scenarios/rl`、`scenarios/finetune`、`scenarios/data_science` 以及 `utils/agent/workflow.py`——确认了它是全项目唯一的 LLM 入口。

除了一次性问答，还有：
- `build_chat_session()` → `ChatSession`：多轮对话，历史存在 `SessionChatHistoryCache` 里。
- `create_embedding(input_content)`：批量/单条文本转向量。
- `json_mode=True` 会设置 `response_format={"type": "json_object"}`；如果传入一个 pydantic `BaseModel` 作为 `response_format`，且后端支持（`supports_response_schema()`，LiteLLM 实现里靠 `litellm.supports_response_schema(model=...)` 并同时要求 `LLM_SETTINGS.enable_response_schema=True`），则走真正的结构化输出，返回时用 `response_format(**json.loads(resp))` 做校验。

### 1.3 缓存机制：为什么要缓存、怎么缓存

**为什么**：一是省钱——同样的 prompt 不用反复付费调用；二是**可复现性**——开发/调试/CI 时反复跑同一段代码，缓存能让"同样的输入永远得到同样的输出"，方便定位问题。

**怎么缓存**：`SQliteLazyCache`（`backend/base.py`，继承 `SingletonBaseClass`）用一个 SQLite 文件（`LLM_SETTINGS.prompt_cache_path`）存三张表：`chat_cache(md5_key, chat)`、`embedding_cache(md5_key, embedding)`、`message_cache(conversation_id, message)`。缓存 key 是对 `chat_cache_prefix + json.dumps(messages) + "<seed=.../>"` 做 MD5。四个 `use_*`/`dump_*` 开关互相独立（读缓存和写缓存可以分别开关）。Embedding 缓存支持"部分命中"：一批字符串里只有没缓存过的那些才会被真正打包发一次 embedding 请求。

### 1.4 缓存种子机制：`CacheSeedGen`

缓存带来一个新问题：如果开着缓存、同一个 prompt 连续调用两次（比如想让 LLM 对同一个问题采样出 N 个不同的假设），拿到的会是**完全相同**的缓存答案，丧失了想要的多样性。`CacheSeedGen`（`rdagent/core/utils.py`）就是解这个矛盾的：

```python
class CacheSeedGen:
    def __init__(self):
        self.set_seed(LLM_SETTINGS.init_chat_cache_seed)   # 默认 42
    def set_seed(self, seed): random.seed(seed)
    def get_next_seed(self): return random.randint(0, 10000)

LLM_CACHE_SEED_GEN = CacheSeedGen()
```

当 `use_auto_chat_cache_seed_gen=True` 且调用时没显式指定 seed，每次调用会消耗 `LLM_CACHE_SEED_GEN` 产生的下一个随机数并拼进缓存 key——于是"同一次程序运行内，连续两次相同 prompt 落进不同的缓存槽位（有多样性）"，而"重新跑一遍整个程序，由于生成器本身从固定的 `init_chat_cache_seed` 重新播种，产生的种子序列是确定的（整体可复现）"。

这和并行执行还有一层联动：`multiprocessing_wrapper`（同样在 `core/utils.py`）在把一批调用分发到多进程 `Pool` 之前，会先用 `LLM_CACHE_SEED_GEN.get_next_seed()` 给每个子进程分配一个种子，子进程启动后先 `LLM_CACHE_SEED_GEN.set_seed(seed)` 再执行目标函数——这样"并行跑"和"串行跑"在开着缓存的前提下，能拿到完全一致的种子消耗顺序，保证并行化不会破坏可复现性。这正是文档 02 提到的"`LoopBase` 并行跑多条 loop 时希望确定性复现"的底层支撑之一。

顺带一提，`core/utils.py` 里还有一个和 LLM 无关、更通用的 `cache_with_pickle` 装饰器：给任意函数的返回值做基于参数哈希的缓存，落盘路径在 `RD_AGENT_SETTINGS.pickle_cache_folder_path_str`，读写都走 `core/serialization.py` 的**签名 pickle**（`secure_pickle_dump`/`secure_pickle_load`），一旦检测到 `UntrustedArtifactError`（文件被篡改或没签名）就直接删掉重算，不会去反序列化不可信内容。

### 1.5 重试与错误处理

`APIBackend` 里是一套手写的重试循环（不是 tenacity），最多 `max_retry`（10）次，针对不同错误分别处理：
- 报错里提示 "messages must contain the word 'json'" → 自动在 prompt 里补一句提示词再重试；
- embedding 超长 → 先用 `truncate_content_list` 截断一次再重试，再超就抛 `RuntimeError` 提示调低 `embedding_max_length`；
- 内容审核拦截（`litellm.BadRequestError`/`ContentPolicyViolationError` 或 Azure 过滤消息）→ 计入 `violation_fail_limit`（默认 1 次就放弃），抛 `core.exception.PolicyError`；
- 超时/被 Azure 拦截 → 计入 `timeout_fail_limit`（默认 10），超限后继续往外抛；
- `openai.RateLimitError` → 尝试从报错文本里解析 "Please retry after N seconds" 来定睡眠时长，否则用 `retry_wait_seconds` 做固定时长 `time.sleep`（没有指数回退）。

每次失败都会喂给全局的 `RD_Agent_TIMER_wrapper`（`log/timer.py`）：累加 `api_fail_count`、记录 `latest_api_fail_time`，失败耗时也会计入定时器的 `all_duration` 预算里——也就是说 LLM 重试浪费的时间，会算进整个 Loop 的超时预算。另外还有一个独立于错误重试的机制：`_create_chat_completion_auto_continue`，当 `finish_reason == "length"`（回复被截断）时自动把已输出内容拼回 prompt 续写，最多续 6 次（`try_n=6`）。输出解析上，`JSONParser` 有四级兜底策略（直接解析/提取代码块/用 `tokenize` 修复 Python 字面量写法/组合策略），`CodeBlockParser` 负责按语言标签提取围栏代码块，解析不出来时抛 `core.exception.CodeBlockParseError`（除非 `fallback_to_raw=True`）。

### 1.6 后端实现：LiteLLM 默认 / 遗留后端 / pydantic-ai 适配器

- **`LiteLLMAPIBackend`**（`oai/backend/litellm.py`，默认后端）：底层调 `litellm.completion()` / `litellm.embedding()`，借助 `litellm.token_counter`/`get_model_info`/`completion_cost`/`supports_response_schema` 做计数、成本估算、能力探测；模块级全局变量 `ACC_COST` 累计花费，通过 `logger.log_object(tag="token_cost")` 写进日志；支持通过 `chat_model_map` 按当前日志 tag 覆盖模型/温度/`max_tokens`/`reasoning_effort`；还会给 `litellm.exceptions.BadRequestError`/`Timeout` 注册 `copyreg.pickle`，让这些异常能被安全地塞进 `HypothesisFeedback.exception` 并随会话快照一起签名 pickle 存盘。
- **`DeprecBackend`**（`oai/backend/deprec.py`）：LiteLLM 引入之前的遗留后端，直接用 `openai` SDK + Azure（`azure.identity`/`azure.ai.inference`）+ 离线 Llama2 + 通过裸 `urllib.request` 访问的 GCR 托管 llama2/llama3/phi 端点。只要把 `LLM_SETTINGS.backend` 指过去就能用，但不是默认；`LLM_SETTINGS` 里绝大多数 azure/llama2/gcr 字段其实就是为它准备的。
- **`backend/pydantic_ai.py`**：不是第三个独立后端，而是一个**适配器**——`get_agent_model()` 断言当前激活的后端必须是 `LiteLLMAPIBackend`，复用它已解析好的模型名/凭据（通过硬编码的 `PROVIDER_TO_ENV_MAP` 把 LiteLLM 的 provider 名映射到对应的 `{PREFIX}_API_KEY`/`{PREFIX}_API_BASE` 环境变量），构造出一个真正的 `pydantic_ai.models.openai.OpenAIChatModel`（经由 `LiteLLMProvider`）。用在需要 `pydantic-ai` 的 `Agent(model)` 工具调用循环的地方（如 `utils/agent/workflow.py`）。
- 三者的选择机制和整个框架的插件化思路完全一致：靠 `LLM_SETTINGS.backend` 这个字符串 + `import_class`。

### 1.7 Embedding 与 RAG

`APIBackend().create_embedding()` 走 `_create_embedding_with_cache`（支持部分命中合并）→ 具体后端的 `_create_embedding_inner_function`（LiteLLM 下就是 `litellm.embedding(...)`）。`oai/utils/embedding.py` 提供 `get_embedding_max_tokens`/`trim_text_for_embedding`/`truncate_content_list`（三级兜底：LiteLLM 的 `get_max_tokens` → 内置的 `EMBEDDING_MODEL_LIMITS` 常见模型表 → 配置里的 `embedding_max_length`），既被主动调用，也被 1.5 节提到的"embedding 超长自动截断重试"逻辑复用。`llm_utils.py` 还有 `calculate_embedding_distance_between_str_list()` 算余弦相似度矩阵。实际消费方例如 `rdagent/scenarios/kaggle/knowledge_management/vector_base.py` 就是调 `APIBackend()` 来算向量——这正是 02 文档提到的 `RAGStrategy`/`EvolvingKnowledgeBase` 做知识检索增强时的底层实现（具体知识库设计见文档 04，这里不展开）。

## 2. 日志与追踪 + UI `rdagent/log/`

### 2.1 核心日志对象：`rdagent_logger`

`rdagent/log/logger.py` 定义 `RDAgentLog(SingletonBaseClass)`，模块级单例实例在 `rdagent/log/__init__.py` 里暴露为 `rdagent_logger`（惯用别名 `from rdagent.log import rdagent_logger as logger`）。

**`tag()` 上下文管理器**：

```python
@contextmanager
def tag(self, tag: str):
    new_tag = tag if not self._tag_ctx.get() else f"{self._tag_ctx.get()}.{tag}"
    token = self._tag_ctx.set(new_tag)
    try:
        yield
    finally:
        self._tag_ctx.reset(token)
```

关键点：`_tag_ctx` 是一个 `contextvars.ContextVar`，**不是**手写的列表栈——这让它天然支持协程并发安全（`LoopBase` 是用 asyncio 跑多条 loop 的）。嵌套的 `tag()` 会把新 tag 用点号拼在当前 tag 后面，所以 `rdagent/utils/workflow/loop.py` 里 `with logger.tag(f"Loop_{li}.{name}"):` 包住每一步执行后，日志条目的 tag 天然带上"第几个 loop、第几步"的层级信息。

**`log_object(obj, tag="")`**：把当前上下文 tag、传入的局部 tag、以及 `get_pids()`（从当前进程沿父进程链一直走到主进程，拼成 `123-456-789` 这样的 PID 链，用来区分多进程/子进程产生的日志）拼接成最终 tag，交给 `self.storage`（一个 `FileStorage`）以及 `self.other_storages` 里的每个存储后端去落盘/推送。

**落盘格式**（`rdagent/log/storage.py` 的 `FileStorage`）：`log()` 把 `tag` 里的点号替换成路径分隔符（`cur_p = self.path / tag.replace(".", "/")`），再在这个目录下用微秒级时间戳命名文件（默认 `save_type="pkl"`，走 `core/serialization.py` 的签名 pickle；也支持 `"json"`/`"text"`）。所以磁盘上的真实结构是：

```
<trace_path>/<上下文tag各级目录>/<局部tag各级目录>/<pid链目录>/<时间戳>.pkl
```

`iter_msg()` 反向遍历这些目录重建 `Message` 对象（`rdagent/log/base.py` 定义：`tag`/`level`/`timestamp`/`caller`/`pid_trace`/`content`），会跳过 `debug_llm.pkl`；`truncate(time)` 删除某时间点之后的所有文件并清理空目录——这就是 `LoopBase.load(path, checkout=True)`（文档02提到）"回滚会话时顺带清理掉更晚的日志"背后调用的东西。

**需要特别说明的是**：这套 `FileStorage` 轨迹（记录假设/反馈/实验/LLM问答等"追踪事件"）和 `LoopBase.dump()`/`.load()` 产生的**会话快照**（`__session__/<loop_idx>/<step_idx>_<step_name>`，整个 Loop 对象的签名 pickle）是**两套不同的目录树**，只是共享同一个根路径 `LOG_SETTINGS.trace_path`（`rdagent/log/conf.py` 的 `LogSettings`，`env_prefix="LOG_"`，默认 `./log/<UTC时间戳>`）。前者是"发生了什么"的流水日志，后者是"整个状态如何恢复"的快照。

### 2.2 `RDAgentTimer`（`rdagent/log/timer.py`）

`RDAgentTimer` 保存 `started`/`target_time`/`all_duration`/`_remain_time_duration`。`reset(all_duration)` 能解析 `"12h"`/`"30m"`/`"10s"`/`"2d"` 这样的字符串，算出 `target_time = now() + all_duration`。`is_timeout()` 判断是否超过 `target_time`。全局单例是 `RD_Agent_TIMER_wrapper`（`RDAgentTimerWrapper(SingletonBaseClass)`，顺带维护 `api_fail_count`/`latest_api_fail_time`，就是 1.5 节提到的那个计数器）。

`LoopBase`（`utils/workflow/loop.py`）如何用它：`self.timer = RD_Agent_TIMER_wrapper.timer`；`run(all_duration=...)` 首次设置时调用 `timer.reset()`；每一步执行前 `_check_exit_conditions_on_step()` 检查 `timer.is_timeout()`，一旦超时就抛 `LoopTerminationError("Timer timeout")`，被 `run()` 捕获后清理子进程并跳出循环——这就是 `all_duration` 超时控制的真正落地点。会话保存时（`LoopBase.dump()`）会先 `update_remain_time()` 把"剩余时间"精确记录下来；恢复会话时（`LoopBase.load(..., replace_timer=True)`）用 `restart_by_remain_time()` 接着之前剩下的时间继续跑，而不是重新给满额度——这样"断点续跑"不会变相延长总预算。源码里也坦诚写了一条已知限制：当某一步被强制丢进子进程执行（`force_subproc=True`，经由 `ProcessPoolExecutor`）时，全局计时器不会被实时同步到子进程。

### 2.3 Streamlit 离线/交互 UI（`rdagent/log/ui/`）

- **`rdagent ui`**（`ui/app.py`）：读取某个 `--log_dir` 下的 `FileStorage`，逐条回放消息，按 "hypothesis generation" 等 tag 切分"第几轮"，渲染假设/任务/反馈/代码。它导入的是 Qlib/Kaggle/General Model 场景的类型（`QlibModelScenario`、`KGScenario`、`FactorTask`、`ModelSingleFeedback` 等），**不包含 data_science 场景的类型**——印证了 README 里"通用 `rdagent ui` 不覆盖 data_science 细节"的说法。
- **`rdagent ui --data-science` 对应的 `ui/dsapp.py`**：专门的多页 Streamlit 应用（`st.navigation` 挂载 `ds_summary.py`/`ds_trace.py`/`aide.py` 三个页面），导入 `DataScienceRDLoop`，专门用来补上述通用 UI 看不到的 data_science 细节。
- **`ui/ds_user_interact.py`**：data_science 场景下 human-in-the-loop 交互页面，用的是**文件轮询握手协议**（不是 Flask 的 queue）——每秒轮询 `DS_RD_SETTING.user_interaction_mid_folder` 下的 `.pkl` 文件（里面是待确认的假设/任务/用户指令等），用户编辑后写一个同名的 `<session>_RET.json` 响应文件，再删掉原始 `.pkl`；超时未处理的会话会被自动清理。这与 `core/interactor.py` 的 `Interactor`/`RDLoop._interact_hypo`/`_interact_feedback` 抽象（文档02提到）是对应的落地实现之一，但与下面 Flask Web UI 用的 IPC 机制是**两条独立的路径**（一个走磁盘文件轮询，一个走 `multiprocessing.Queue`）。
- **`ui/storage.py` 的 `WebStorage(Storage)`**：这是让 Streamlit UI 之外、给 Flask Web UI 服务端"实时推送"用的存储后端。`log()` 把对象转成前端友好的 JSON（`_obj_to_json`，按类型分支处理 `Hypothesis`、PDF 截图、`FactorTask`/`ModelTask` 列表、`DSExperiment`、CoSTEER 单步反馈、Qlib 回测图表等），再 POST 到 `http://localhost:{port}/receive`（即下面 Flask 服务的 `/receive` 接口），如果配置了 `UI_SETTING.server_auth_token` 会带上 `Authorization: Bearer` 头；连不上时静默失败（不会导致主流程崩溃）。它是如何被启用的：`rdagent/log/conf.py` 的 `LogSettings.set_ui_server_port(port)` 往 `storages` 字典里加一条 `"rdagent.log.ui.storage.WebStorage": [port, trace_path]`，再调用 `rdagent_logger.refresh_storages_from_settings()`（内部用 `import_class` 动态实例化）——于是跑 `server_ui` 时启动的子进程，会一边写本地 `FileStorage`，一边把同样的事件实时推给 Flask 后端。
- 其余文件只做简要用途说明：`ds_summary.py`（结果汇总图表）、`ds_trace.py`（DS 场景的 trace 树查看器）、`llm_st.py`/`qlib_report_figure.py`（Plotly 图表绘制，后者专门画 Qlib 回测收益/回撤曲线）、`web.py`（通用 trace 渲染小组件）、`st_fixed_container.py`（Streamlit 固定定位容器的小工具）、`utils.py`（基于 networkx 的 trace 依赖图可视化）。

### 2.4 Flask Web UI 后端（`rdagent/log/server/`）：README 安全声明逐项核验

`rdagent server_ui` 背后是 `rdagent/log/server/app.py`（Flask），配置来自 `rdagent/log/ui/conf.py` 的 `UIBasePropSetting`（`env_prefix="UI_"`，实例 `UI_SETTING`，同一份配置同时服务 Streamlit 和 Flask 两端）：

```
server_host = "127.0.0.1"
server_auth_token = ""
cors_allowed_origins = []
max_upload_mb = 20
load_legacy_pickle_traces = False
static_path = "./git_ignore_folder/static"
trace_folder = "./git_ignore_folder/traces"
upload_folder = "./git_ignore_folder/uploads"
```

对照 README「Web UI security and remote access」一节的声明，逐条核验结果：

| README 声明 | 核验结果 | 代码证据 |
| --- | --- | --- |
| 默认只监听 `127.0.0.1` | **属实** | `app.py: main(port=19899, host=UI_SETTING.server_host)`；`--host` CLI 参数的具体 typer 接线未独立追踪确认，但大概率如实 |
| 不配置 `UI_SERVER_AUTH_TOKEN` 服务拒绝启动，哪怕是本机 | **属实** | `main()` 里无 token 直接 `raise ValueError`；`before_request` 钩子 `_require_authentication` 还有一道运行期兜底：token 没配置则对所有非公开路由返回 503 |
| Token 经由重定向到 `/` 从 URL 里去掉，存成 HttpOnly、SameSite cookie | **属实，且更严格** | `index()`：校验通过后 `redirect(url_for("index"))` 清空查询串，`set_cookie("rdagent_auth", token, httponly=True, samesite="Strict", secure=request.is_secure)`——SameSite 具体是 `Strict`，不只是泛泛的"same-site" |
| 默认禁用 CORS，仅允许精确 origin，拒绝通配符/路径 | **属实** | 默认 `cors_allowed_origins=[]`；`security.py` 的 `normalize_origin()` 拒绝 `*`、路径/query/fragment、带用户信息的 URL；非法配置在应用启动时直接抛 `ValueError` 崩溃，而不是悄悄放宽 |
| Cookie 鉴权的 POST 必须是同源或白名单 Origin/Referer | **属实** | 仅当走 cookie（而非显式 Bearer）且方法非 GET/HEAD 时触发校验，Origin 缺失则退而看 Referer，必须属于 `CORS_ALLOWED_ORIGINS ∪ {本服务自身origin}` |
| 上传接口拒绝 `.dill/.pickle/.pkl/.py/.pyc/.pyo` 后缀 | **属实，完全一致** | `security.py`: `_UNSAFE_UPLOAD_SUFFIXES = {".dill",".pickle",".pkl",".py",".pyc",".pyo"}`，在 `validate_upload_filename()` 里强制检查 |
| `UI_MAX_UPLOAD_MB` 默认 20，按整个请求体积限制 | **属实** | `app.config["MAX_CONTENT_LENGTH"] = UI_SETTING.max_upload_mb * 1024 * 1024`，Werkzeug 层面的整请求体积上限，非按单文件 |
| `UI_LOAD_LEGACY_PICKLE_TRACES` 默认关闭，控制是否反序列化历史 pickle trace | **属实，但有一处值得注意的细节** | 默认 `False`；它具体控制的是**服务启动时**的批量扫描反序列化 `_load_existing_traces()`。在当前代码里，这是文件中唯一会触发批量历史 pickle 反序列化的路径，所以效果上等同于"全局开关"，但严格说它只是挂在启动扫描这一个调用点上 |
| 上传文件与 trace 目录物理隔离（`UI_UPLOAD_FOLDER` vs `UI_TRACE_FOLDER`） | **属实** | `log_folder_path = UI_SETTING.trace_folder`、`upload_folder_path = UI_SETTING.upload_folder` 是两个独立的根路径；列 trace 列表时还会显式排除路径里带 `uploads` 的目录，双重保险 |
| `/upload` 只接受真实文件，拒绝把 URL/服务器路径当文本字段传入 | **属实，完全一致** | 代码里专门判断 `request.form.getlist("files")`（同名文本字段）非空就直接 400 拒绝，报错信息就是"Upload files directly; URLs and server paths are not accepted" |

**额外发现（README 未明确提及，但值得在文档/使用建议里提一句）**：

- 鉴权/CORS/Origin 的判断逻辑全部集中在同一个 `before_request` 钩子里，执行顺序是"先查 Origin 合法性（哪怕是公开路由也要查）→ 公开路由直接放行 → 查 token → 如果是 cookie 鉴权的写操作再查 Origin/Referer 白名单"，顺序本身有讲究。
- **`rdagent/log/server/debug_app.py` 是一个没有任何鉴权、没有 CORS 防护的调试用 Flask 变体**：没有 `before_request` 鉴权钩子、没有 token 概念、`/control` 只是回一句假的"成功"而不真的管进程，`/upload` 路径硬编码成了某个开发者本机路径（`/home/bowen/workspace/new_traces`）且实际上根本没有保存上传的文件。这个文件明显是开发期遗留的调试辅助，**绝不能当成生产可用的服务来暴露**，README 没提到它，这里额外提醒。
- `UI_LOAD_LEGACY_PICKLE_TRACES` 关闭之后，之前已跑过的 trace 在历史列表里仍会出现名字（因为列表只看目录是否存在 `.pkl` 文件），但其内容在服务重启后不会被加载进内存——这与 README 文字描述一致，值得在文档里强调，避免用户以为"看到列表项=内容能看"。

### 2.5 其它日志相关小文件

`rdagent/log/mle_summary.py`：backing `rdagent grade_summary` 命令（`fire.Fire({"grade":..., "summary":..., "grade_summary":...})`），读取 `FileStorage` 里打了 `competition`/`running`/`feedback` tag 的消息，跑 MLE-bench 评分脚本并汇总奖牌统计，输出 `summary.pkl`。`rdagent/log/utils/folder.py` 提供两个分析用的辅助函数：`get_first_session_file_after_duration`（按已耗时找到对应的会话快照文件）、`first_li_si_after_one_time`（找到运行满 N 小时时所在的 loop/step），都是直接在 `__session__` 快照树上做遍历。

## 3. 代码执行环境 `rdagent/utils/env.py`

`FBWorkspace.execute(env, entry)` / `.run(env, entry)`（文档02）最终调的就是这里的 `Env` 抽象。

### 3.1 抽象层：`Env` / `EnvResult` / `EnvConf`

`Env[ASpecificEnvConf]`（泛型基类）约定：`prepare()`（准备镜像/环境，抽象方法）+ `_run(entry, local_path, env, running_extra_volume, ...) -> (stdout, exit_code)`（真正执行，子类实现）。公开方法 `run(...)` 才是外部调用入口，它会：

1. 合并 `conf.env_dict` 和调用方传入的 `env`；
2. 把 `entry` 包进 `timeout --kill-after=10 {running_timeout_period} {entry}`（见 3.3）；
3. 根据 `conf.enable_cache` 决定走结果缓存（`cached_run`）还是直接重试执行（`__run_with_retry`）；
4. 把返回 stdout 里出现的宿主机绝对工作区路径替换成占位符 `<WORKSPACE_PATH>`（避免把文件系统布局泄露进 LLM 上下文）；
5. 包装成 `EnvResult` 返回。

`EnvResult`（`@dataclass`）字段：`full_stdout`（完整原始输出）、`exit_code`、`running_time`（秒）、`stored_full_stdout_to_truncated_stdout`。**`.stdout` 是一个属性而非存储字段**：它会先用 `rdagent.utils.filter_redundant_text()`（去 ANSI 转义码、去 tqdm 进度条模式、合并连续空行、删除重复超过 `max(len/10,10)` 次的行，再用 LLM 生成的正则做最多 3 轮过滤）清洗，再用 `rdagent.utils.fmt.shrink_text(context_lines=RD_AGENT_SETTINGS.stdout_context_len, line_len=RD_AGENT_SETTINGS.stdout_line_len)` 做长度裁剪——真正喂给 LLM/反馈环节的是这个清洗/裁剪后的版本，整个计算过程还套了 `@cache_with_pickle`。

`EnvConf(ExtendedBaseSettings)` 公共字段：`default_entry`、`env_dict`、`extra_volumes`、`running_timeout_period=3600`、`redirect_stdout_to_file=False`、`enable_cache=True`、`retry_count=5`、`retry_wait_seconds=10`、`exclude_chmod_paths=[]`。

### 3.2 两条执行路线：本地子进程 vs Docker

- **`LocalEnv(Env[ASpecificLocalConf])`**：用 `subprocess.Popen(entry, shell=True, ...)` 在宿主机上跑。`live_output=True`（默认）时用 `select.poll()` 实时交织输出 stdout/stderr 到终端；`False` 时用 `process.communicate()` 一次性拿结果（避免管道卡死）。
  - 各场景的 Conda 环境（`QlibCondaEnv`、`FTCondaEnv`、`BenchmarkCondaEnv`）都**继承自 `LocalEnv`**，区别只在 `prepare()`（创建/校验 conda 环境、装依赖）和各自的 `*CondaConf` 配置类——代码里**没有**一个单独叫 `CondaEnv` 的类，Conda 场景复用的是 `LocalEnv` 这同一个执行引擎。
- **`DockerEnv(Env[DockerConf])`**：用 **docker-py SDK**（`docker.from_env()`），不是 shell 出去调 `docker run`。各场景的具体类：`QTDockerEnv`（Qlib）、`KGDockerEnv`（Kaggle）、`MLEBDockerEnv`（MLE-bench）、`FTDockerEnv`（LLM 微调）、`BenchmarkDockerEnv`（OpenCompass）。**没有名为 `DSDockerEnv` 的类**——data_science/Kaggle 场景是直接把 `DSDockerConf()` 配置对象传给通用的 `DockerEnv` 来用（见 `rdagent/components/coder/data_science/conf.py`），如果之前以为有专门的 `DSDockerEnv` 类，这里纠正一下。

### 3.3 Docker 镜像选择与资源限制

`DockerConf(EnvConf)` 核心字段：`build_from_dockerfile`、`dockerfile_folder_path`、`image`、`mount_path`、`extra_volumes`、`extra_volume_mode="ro"`（除 `mount_path` 外默认只读挂载）、`network="bridge"`、`shm_size`、`enable_gpu=True`、`mem_limit="48g"`、`cpu_count=None`（默认不限 CPU 数）、`running_timeout_period=3600`、`save_logs_to_file=True`、`terminal_tail_lines=20`。各场景子类（`QlibDockerConf`/`KGDockerConf`/`DSDockerConf`/`MLEBDockerConf`/`FTDockerConf`/`BenchmarkDockerConf`）都用 `pydantic_settings` 的 `env_prefix`（如 `DS_DOCKER_`）定制镜像名（`local_ds:latest` 等本地构建的 tag，**不是**从公共仓库拉取已发布镜像，全部走 `build_from_dockerfile=True` 本地构建）、挂载点、超时时长等。两个值得注意的例外：`FTDockerConf`/`BenchmarkDockerConf` 把 `network` 显式设成 `"host"`（为了能连到本机的 litellm 代理），相比默认的 `"bridge"` 隔离网络，这是一个**明显更宽松**的网络暴露面，值得在使用这两个场景时留意。

**超时怎么落地**：靠在容器/进程内包一层 `timeout --kill-after=10 {period} {entry}` 这个 shell 命令，不是 Python 侧的 watchdog 线程，也不是 docker 的某个运行时参数——所以依赖镜像/宿主机里有 GNU coreutils 的 `timeout` 命令；`running_timeout_period=None` 则完全不设超时。超时发生后 `__run_with_retry` 还会额外做一次事后的墙钟时间校验，补一条日志说明"进程被判定超时"，但真正的 kill 动作已经由 `timeout` 完成。

**重试**：`Env.__run_with_retry`（私有方法）在 `_run()` 失败时最多重试 `conf.retry_count + 1` 次（本地默认 0 次不重试，Docker 默认 5 次），每次间隔 `retry_wait_seconds`。

**资源限制**：传给 `docker.containers.run()` 的只有 `mem_limit`、`shm_size`、`cpu_count`、`network`，以及通过 `docker.types.DeviceRequest` 控制的 GPU 访问（默认整机所有 GPU，若宿主机设了 `CUDA_VISIBLE_DEVICES` 则对应传具体 `device_ids`）。**没有**设置 `cpuset_cpus`、`pids_limit`、`cap_drop`/`cap_add`、`security_opt`（如 no-new-privileges）、`read_only` 根文件系统、非 root `user=` 这些更细粒度的硬化手段——容器用的是 Docker 默认能力集，实际运行身份取决于各场景 Dockerfile 里的 `USER` 设置（未在 `env.py` 范围内核实）。

**stdout/exit_code 采集**：`LocalEnv` 靠管道 + `select.poll()`/`communicate()`；`DockerEnv` 用 `container.logs(stream=True)` 拿实时日志流（`terminal_tail_lines>0` 时用 `rich.live.Live` 只滚动显示最后 N 行，避免刷屏；`save_logs_to_file=True` 时把**完整**日志额外写到 `<workspace>/logs/docker_execution_<timestamp>.log`，并维护一个 `docker_execution_latest.log` 软链接），退出码来自 `container.wait()["StatusCode"]`。另外 `redirect_stdout_to_file` 开关提供了第二种采集路径：把 entry 改写成重定向到工作区内一个按 entry 内容哈希命名的 `.log` 文件，跑完读出来再删掉。

**镜像构建/拉取**：`DockerEnv.prepare()` 若 `build_from_dockerfile=True` 且对应 Dockerfile 目录存在，就用 `client.api.build(...)` 本地构建（带进度条），否则 `client.images.get()` 找不到时走 `client.api.pull()` 拉取。**没有**显式的 registry 登录/凭据代码，完全依赖宿主机 `~/.docker/config.json` 的既有登录状态。

### 3.4 安全隔离相关要点（对应 README 的"不挂载宿主机凭据/Docker socket"原则）

- `env.py` 里**没有**任何地方把 `/var/run/docker.sock` 挂载进被执行代码所在的容器——唯一出现 `docker.sock` 的地方是 RD-Agent 主进程自己连接本机 Docker daemon 去拉镜像（`pull_image_with_progress()`），这与 README 所述"生成代码执行时不暴露 Docker socket"的原则是一致的。
- 默认会挂进容器的卷：任务工作区（`local_path`，可写）、`extra_volumes`（默认只读，个别场景如 Qlib 把 `~/.qlib/` 显式设成可写以便复用行情缓存）、调用方传入的 `running_extra_volume`。`env.py` 自身默认**不会**挂载宿主机的凭据目录（`~/.aws`、`~/.ssh`、`.env` 等）；但由于 `extra_volumes`/`running_extra_volume` 是调用方可控的字典，不能排除某个场景代码在别处传入了敏感挂载——这一点没有在全仓库范围内逐一核实，只能确认 `env.py` 自身的默认行为是干净的。
- 默认网络模式是 `"bridge"`（标准隔离网桥，容器仍有出网能力，不是完全断网的 `"none"`），`FTDockerConf`/`BenchmarkDockerConf` 显式放宽为 `"host"`（见 3.3）——这是文档里值得单独提醒使用者的一点。
- 每次运行结束后会在容器内对挂载目录执行 `chmod -R 777`（排除 `exclude_chmod_paths`），目的是让宿主机能正常读写容器（往往以 root 身份）写出的文件，副作用是工作区目录事后对宿主机所有用户可写——影响范围只在工作区目录内，不是全局性的权限问题，但也值得知道。

## 4. 其它工具与辅助脚本

- **`rdagent/utils/agent/`**（注意和 `rdagent/components/agent/` 不是一回事——后者是"工具调用型 Agent"的具体实现，比如 `context7/` 对接 Context7 文档检索、`mcp/` 对接 Model Context Protocol、`rag/` 是检索增强策略，这些是文档04范畴内"组件"；`utils/agent/` 则是更底层的、被各处 prompt/LLM 调用代码共享的基础设施）：`tpl.py` 定义 `T`（即 `RDAT` 类的简写别名）——一套按"`a.b.c:x.y.z`"这种 URI 语法从 YAML 加载 Jinja2 模板片段并渲染的小框架，支持按"当前目录优先于包内默认模板"的顺序覆盖，渲染结果还会被记录成 `tag="debug_tpl"` 的调试日志；`ret.py` 定义 `AgentOut`/`PythonAgentOut`/`MarkdownAgentOut`/`BatchEditOut`/`PythonBatchEditOut`/`PythonBatchPatchOut` 一组"如何从 LLM 回复里解析出结构化结果"的策略类（提取代码块、JSON、多文件批量编辑、diff/patch 等）；`apply_patch.py` 实现了一套类似 OpenAI "apply_patch" 工具的文本补丁解析与应用逻辑；`workflow.py` 的 `build_cls_from_json_with_retry` 封装了"让 LLM 产出 JSON、解析失败就把错误信息追加回 prompt 再试一次（最多 `retry_n` 次）"的通用模式，内部就是调 1.2 节的 `APIBackend().build_messages_and_create_chat_completion(json_mode=True)`。
- **`rdagent/utils/repo/`**：`repo_utils.py` 的 `RepoAnalyzer` 能遍历一个目录生成"仓库摘要"（文件树 + 每个 `.py` 文件的签名/docstring，分级详略），用于把"参考代码库"的结构喂给 LLM 做上下文；`diff.py` 的 `generate_diff(dir1, dir2, file_pattern)` 对两个目录做类似 `diff -durN` 的逐文件比较，生成统一 diff 文本——用于展示"代码相对上一版改了什么"。
- **`rdagent/utils/blob/`**：只有两个 shell 脚本（`gen_token.sh`/`azsync.sh`），不是 Python 基础设施，而是面向 LLM 微调（finetune）场景的运维辅助工具——用 Azure CLI 生成一个有时效的 SAS Token，再用 `azcopy sync` 把本地的 `log/`、workspace 目录或 litellm 日志目录同步到 Azure Blob 容器，方便团队成员之间同步长跑任务的产出。属于边缘小工具，和核心研发流程无关。
- **`rdagent/app/CI/`**：`run.py`（800+行）实现了一个"用 LLM 自动修代码风格问题"的小闭环——跑 `ruff`/`mypy` 收集 `CIError`，包装成 `CIFeedback`（继承 `core/evolving_framework.py` 的 `Feedback`），借助 `tree_sitter_python` 做语法级定位，驱动 LLM 生成修复，产出 `FixRecord`（区分"直接修复"/"人工修复"/"跳过"/"需要人工处理的说明"）。本质上是把 02 文档介绍的 CoSTEER 式"进化-评估"内循环，复用到了"代码规范检查"这个场景上。
- **`rdagent/app/benchmark/`**：`factor/eval.py`、`model/eval.py` 是离线评测脚本——读取 `BenchmarkSettings`，加载标准测试用例（如因子复现的 benchmark JSON），跑指定的生成方法（通过 `import_class` 加载），再用 `FactorImplementEval`/对应的 Eval 类打分，结果通过 `logger.log_object` 记下来；`factor/analysis.py` 做结果的统计分析。这套东西是用来衡量"某个 Coder/生成方法在标准测试集上到底行不行"的离线评测工具，不是在线 R&D Loop 的一部分。
- **`rdagent/app/utils/`**：`health_check.py`（backing `rdagent health_check`）做三件事——`check_docker_status()`（拉 `hello-world` 镜像跑一下验证 Docker 能用）、`check_and_list_free_ports()`（检查默认的 19899 端口是否被占用，列出可用端口）、`env_check()`（按 `DEEPSEEK_API_KEY`/`OPENAI_API_KEY` 真实发一次 chat/embedding 请求验证 LLM 配置是否生效）；`info.py`（`collect_info`）收集系统/Python版本/最近一个 Docker 容器状态/RD-Agent 版本与依赖包版本，用于提交 issue 时打印诊断信息；`ape.py` 是一个实验性的"自动 Prompt 工程"脚本雏形（读取 `debug_llm` 日志、调 LLM 分析问答对提出 prompt 改进建议）；`ws.py`/`ws_ft.py` 是两个小 CLI（`rdagent.app.utils.ws`/`ws_ft`），分别用于手动起一个 data_science/LLM 微调场景的 Docker 环境并跑一条命令（常用于"进容器调试"场景，比如 `sleep 3600` 占住容器再 `docker exec -it` 进去看现场）。

## 5. 与其他模块的关系

把本篇和文档02对照起来看，调用链路大致是：

- **谁调 `oai/`**：几乎所有产出"文本/决策"的类都直接或间接调 `APIBackend()`——`HypothesisGen.gen`、`Hypothesis2Experiment.convert`、`Experiment2Feedback.generate_feedback`、Coder 内部的 `EvolvingStrategy.evolve_iter`、`utils/agent/workflow.py` 的 `build_cls_from_json_with_retry`，以及几乎所有场景自己的 prompt 拼装代码。可以说 `core/proposal.py` 和 `core/evolving_framework.py` 里定义的每一个"靠 LLM 做决定"的抽象方法，具体实现基本都要落到 `APIBackend` 这一个入口上。
- **谁调 `log/`**：`LoopBase`（`utils/workflow/loop.py`）本身贯穿使用 `logger.tag()`/`logger.log_object()` 记录每一步；`RD_Agent_TIMER_wrapper` 被它用来做超时控制；`core/serialization.py` 的签名 pickle 同时被 `LoopBase.dump/load`（会话快照）和 `FileStorage`（追踪日志）复用，是两边共同的持久化底座。`rdagent ui`/`server_ui` 则是读这些日志/快照的下游消费者。
- **谁调 `env.py`**：`core/experiment.py` 的 `Workspace.execute(env, entry)`/`FBWorkspace.run(env, entry)` 是唯一入口，各场景 Runner（文档03）负责选择合适的 `Env` 子类（通常靠一个 `get_xxx_env()` 工厂函数，如本文提到的 `get_ds_env`/`get_ft_env`）并把代码丢进去执行，拿到的 `EnvResult` 会被塞进 `Experiment.running_info`/`result`，进而流向 `Experiment2Feedback`。

从这个角度看，本篇介绍的三块基础设施，其实是文档05/06/07 里每一个具体场景能跑起来的**共同地基**：不管场景差异多大，"怎么问 LLM"、"怎么记录过程"、"代码在哪跑"这三件事，全部复用的是这里讲的同一套代码。
