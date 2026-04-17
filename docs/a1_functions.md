<!-- Auto-generated: 详细函数说明 for biomni/agent/a1.py -->
# `biomni.agent.a1` 函数总览

本文档对 `biomni/agent/a1.py` 中主要函数进行细致说明，按源码中出现的顺序列出：目的、参数、返回值、副作用和重要实现细节或注意事项。旨在帮助开发者快速理解每个方法的职责与使用场景。

## 总体说明
- 文件核心类：`A1`，代表 Biomni 的主要 agent 实例，负责初始化配置、加载知识库和工具、管理检索、构建 system prompt、执行交互式工作流并保存会话记录。
- 该文件包含若干辅助函数和内部工具（以 `_` 开头或内部嵌套定义），以及用于扩展工具和 MCP（Model Context Protocol）集成的功能。

---

## 函数索引（按出现顺序）
- `__init__`
- `add_tool`
- `add_mcp`
  - 嵌套：`discover_mcp_tools_sync`、`_discover_async`、`make_mcp_wrapper`、`sync_tool_wrapper`、`async_tool_call`
- `get_custom_tool`
- `list_custom_tools`
- `remove_custom_tool`
- `add_data`
- `get_custom_data`
- `list_custom_data`
- `remove_custom_data`
- `add_software`
- `get_custom_software`
- `list_custom_software`
- `remove_custom_software`
- `_filter_know_how_for_commercial_mode`
- `_generate_system_prompt`
  - 嵌套：`format_item_with_description`
- `configure`
  - 嵌套：`generate`、`execute`、`routing_function`、`routing_function_self_critic`、`execute_self_critic`
- `_prepare_resources_for_retrieval`
- `go`
- `go_stream`
- `update_system_prompt_with_selected_resources`
- `result_formatting`
- `_parse_tool_calls_from_code`
- `_parse_tool_calls_with_modules`
- `_inject_custom_functions_to_repl`
- `create_mcp_server`
- `save_conversation_history`
  - 嵌套：`timeout_handler`
- `_generate_markdown_content`
- `_get_messages_for_processing`
- `_normalize_conversation_state_messages`
- `_normalize_log_messages`
- `_process_message`
- `_process_human_message`
- `_process_ai_message`
- `_process_other_message`
- `_process_execution_with_results`
- `_format_and_add_content`
  - 嵌套：`parse_tool_calls_wrapper`
- `_add_execution_plots`
- `_process_regular_ai_message`
- `_convert_markdown_to_pdf`
- `_clear_execution_plots`
- `_generate_mcp_wrapper_from_biomni_schema`
  - 嵌套：返回的 `wrapper`（无参或有 kwargs 两种签名）
- `launch_gradio_demo`
  - 嵌套：`verify_access_code`、`generate_response`、`like`

---

## 逐一说明

### __init__(...)
- 作用：创建并配置 `A1` agent 实例。
- 主要参数（常用）:
  - `path`: 数据根路径（默认来自 `default_config.path`）。
  - `llm`, `source`, `base_url`, `api_key`: 指定 agent 使用的 LLM 模型/提供方与凭据覆盖。
  - `use_tool_retriever`: 是否启用基于 LLM 的资源检索（ToolRetriever）。
  - `timeout_seconds`: 代码执行超时。
  - `commercial_mode`: 如果为 True，则选择仅允许商业使用的数据集/文档描述。
  - `expected_data_lake_files`: 控制是否下载远端 data lake（传空列表可跳过下载）。
- 返回值：无（初始化实例，设置大量属性）。
- 主要副作用与行为：
  - 读取 `.env`（若存在，文件顶部会执行 load_dotenv）。
  - 根据 `commercial_mode` 导入不同的 env 描述（`env_desc.py` 或 `env_desc_cm.py`）。
  - 创建 `biomni_data` 下的 `data_lake` 与 `benchmark` 子目录，并可自动下载缺失的 data lake 文件（S3），受 `expected_data_lake_files` 控制。
  - 调用 `get_llm(...)` 初始化 agent LLM，默认带 `stop_sequences` `</execute>` 与 `</solution>`。
  - if `use_tool_retriever`：创建 `ToolRegistry` 与 `ToolRetriever`。
  - 初始化 `KnowHowLoader` 并根据商用模式过滤文档。
  - 最后调用 `configure()` 生成初始 system prompt 与工作流（StateGraph）。
- 注意：大量打印用于调试/配置可见性；调用可能涉及网络（下载）与 I/O。

### add_tool(self, api)
- 作用：动态将一个 Python 可调用对象注册为 agent 的“工具”。
- 参数：
  - `api`: 可调用对象（函数），agent 将提取源代码并尝试从中生成 API schema。
- 返回值：成功时返回生成的 `schema`（dict）；失败抛出异常。
- 主要行为：
  - 使用 `inspect.getsource(api)` 读取函数源码，并调用 `function_to_api_schema`（依赖 LLM）生成 schema。
  - 补齐 schema 的缺失字段（`name`, `description`, `required_parameters` 等）。
  - 将工具加入 `tool_registry`（如已启用），并将其加入 `module2api`，以便生成 system prompt 时被列出。
  - 在 `builtins._biomni_custom_functions` 中注册该函数（使其在 REPL 执行时可见）。
  - 更新内部 `_custom_functions` 与 `_custom_tools` 映射并调用 `configure()` 以刷新 system prompt。
- 副作用：修改 agent 内部工具列表、全局 builtins 状态，可能影响后续检索与执行。

### add_mcp(self, config_path: str | Path = "./tutorials/examples/mcp_config.yaml") -> None
- 作用：通过 MCP（Model Context Protocol）配置文件加载并注册外部 MCP 服务器上公开的工具，生成同步 wrapper 以便在 agent 内部直接调用。
- 参数：
  - `config_path`: MCP 配置 YAML 路径，描述多个 MCP 服务器及其参数。
- 返回值：None（强调：会修改 agent 的工具注册状态）。
- 主要行为与实现要点：
  - 解析 YAML 配置，针对每个 server 做发现与注册；使用 `mcp` 库的 `ClientSession` 与 stdio client 与服务器建立连接。
  - 提供 `discover_mcp_tools_sync` 用于同步发现 MCP 工具（内部会封装异步发现逻辑 `_discover_async`）。
  - 为每个外部 MCP 工具生成 `make_mcp_wrapper` → `sync_tool_wrapper`，把异步 MCP 调用包装为同步函数，暴露给 agent。
  - 将这些 wrapper 注册为 custom functions/tools 并更新 `module2api` 与 `tool_registry`。
- 注意：需要 `mcp` 包与可访问的外部 MCP 服务器；实现使用 `nest_asyncio` 来在同步上下文运行 async 任务。

嵌套函数说明：
- `discover_mcp_tools_sync(server_params)`：同步包装，返回 MCP 工具元信息列表。
- `_discover_async()`：内部的 async 发现实现（由 `discover_mcp_tools_sync` 调用）。
- `make_mcp_wrapper(cmd, args, tool_name, doc, env_vars)`：返回同步 wrapper `sync_tool_wrapper`，该 wrapper 在内部执行异步 MCP 调用并返回结果。

### get_custom_tool(self, name)
- 作用：查找并返回已注册的自定义工具函数对象（如果存在）。
- 参数：`name`（str）工具名。
- 返回：函数对象 或 None。
- 行为：从 `self._custom_functions` 中读取（若存在）。

### list_custom_tools(self)
- 作用：列出所有已注册的自定义工具名。
- 返回：字符串列表。

### remove_custom_tool(self, name)
- 作用：移除自定义工具（从内部映射、builtins、tool_registry、module2api 等）。
- 参数：`name`。
- 返回：布尔值（True 表示已删除）。
- 注意：会尝试清理所有注册点，但若其它地方仍引用该函数可能导致残留。

### add_data(self, data)
- 作用：向 data lake 注册/添加自定义数据项（在 agent 的运行时索引与 system prompt 中可见）。
- 参数：`data`：dict，键为相对路径或名称，值为描述字符串。
- 返回：None（成功时内部更新 `_custom_data` 与 `data_lake_dict` 并调用 `configure()`）。

### get_custom_data(self, name)
- 作用：返回已添加的自定义数据条目（若存在）；否则返回 None。

### list_custom_data(self)
- 作用：返回所有自定义数据项的列表（name 与描述）。

### remove_custom_data(self, name)
- 作用：从自定义数据列表和 data_lake_dict 中移除指定条目。
- 返回：布尔值表示是否移除成功。

### add_software(self, software)
- 作用：向软件库注册自定义软件（在 system prompt 中显示可用软件）。
- 参数：`software`：dict，格式类似 `{'tool_name': 'description'}`。
- 返回：None，内部更新 `_custom_software` 与 `library_content_dict`，并调用 `configure()`。

### get_custom_software(self, name)
- 作用：查询并返回某个自定义软件项的描述（或 None）。

### list_custom_software(self)
- 返回：已注册的自定义软件列表。

### remove_custom_software(self, name)
- 作用：删除自定义软件并从 `library_content_dict` 中清理。
- 返回：布尔值。

### _filter_know_how_for_commercial_mode(self)
- 作用：当 `commercial_mode` 为 True 时，过滤掉不允许商业使用的 know-how 文档。
- 行为：遍历 `self.know_how_loader.documents` 并根据文档 metadata 标记移除不合规条目。

### _generate_system_prompt(...)
- 作用：根据当前可用工具、data lake、软件库与 know-how 文档生成 agent 的 system prompt（长文本模版），该 prompt 用于引导 LLM 的行为与工具使用规则。
- 主要参数：
  - `tool_desc`：工具字典（module -> [tool schemas]）
  - `data_lake_content`：data lake 项列表（带描述）
  - `library_content_list`：软件库名称列表
  - `self_critic`、`is_retrieval`：控制是否包含自我审查或检索上下文的特别提示
  - `custom_tools`、`custom_data`、`custom_software`、`know_how_docs`
- 返回：格式化后的 prompt 字符串。
- 重要细节：
  - 包含一个详细的「计划-执行-报告」工作流说明，要求 LLM 先给出分步骤计划，并在每步后更新复选框状态。
  - 明确要求 LLM 使用 `<execute>` / `</execute>` 标签进行可执行代码调用，并用 `<observe>` 标注观察。
  - 提供 `PROTOCOL GENERATION` 段落，限定实验 protocol 必须来自本地资源/检索结果并禁止捏造细节。
  - 将函数字典（function dictionary / tool description）与 data lake、软件库逐项列入 prompt 中。

嵌套：`format_item_with_description(name, description)`：用于把单个 data-lake 或 library 项格式化为可读描述。

### configure(self, self_critic=False, test_time_scale_round=0)
- 作用：构建/刷新 agent 的 system_prompt，并搭建内部 StateGraph 工作流（生成 & 执行节点），为 `go()` 提供运行时引擎。
- 参数：`self_critic`（是否启用自我审查路径），`test_time_scale_round`（用于测试缩放轮次）。
- 返回：无（设置 `self.system_prompt`, `self.app`, `self.checkpointer` 等）。
- 实现要点：
  - 收集 data lake 列表、module2api（工具描述），并生成初始 system_prompt（调用 `_generate_system_prompt`）。
  - 定义 workflow 的节点函数：`generate(state)`（调用 LLM 生成下一步文本/计划）和 `execute(state)`（解析 `<execute>` 标签并执行对应代码/工具），并构造 routing 函数根据状态决定流转。
  - 使用 `StateGraph(AgentState)` 编译生成 `self.app`（可流式执行的状态机），并设置 `MemorySaver` 作为 checkpointer。

嵌套节点说明（概览）：
- `generate(state)`：向 LLM 发送 messages（包含 system prompt 与历史），将 LLM 输出作为 AI 消息追加到 state，并决定下一步（通常进入 `execute` 或结束）。
- `execute(state)`：查找并执行消息中的 `<execute>` 标签内容（支持 Python/R/Bash 等），捕获输出并将 observation 写回 state（作为 AI 的 observation）。
- `routing_function` / `routing_function_self_critic`：基于 state 中 `next_step` 等字段决定下一节点（`generate`/`execute`/`end`）。
- `execute_self_critic(state)`：当启用 self_critic 时使用的执行节点，负责自我审查循环。

### _prepare_resources_for_retrieval(self, prompt)
- 作用：聚合当前可检索资源（tools、data_lake、libraries、know_how summaries），调用 `ToolRetriever.prompt_based_retrieval` 以由 LLM 选择最相关的资源并返回名称集合。
- 参数：`prompt`（用户查询）。
- 返回：`selected_resources_names` dict，包含 `tools`, `data_lake`, `libraries`, `know_how` 四类的已选名称。
- 重要：若 `use_tool_retriever` 为 False，则直接返回空或默认资源。

### go(self, prompt)
- 作用：以同步方式运行 agent 完整交互直到结束（blocking）。适用于脚本或 REPL 调用。
- 参数：`prompt`（用户文本）。
- 返回：None（执行过程中会打印日志并把最终会话状态保存在 `self._conversation_state`）。
- 行为：创建初始 inputs（messages: HumanMessage），并通过 `self.app.stream(inputs, stream_mode="values", ...)` 阶段性地运行 workflow，收集/打印中间日志，最终保存 conversation state。

### go_stream(self, prompt) -> Generator[dict, None, None]
- 作用：与 `go` 类似，但以生成器方式产出每一步结果，适用于实时 UI 或需要逐步消费执行进度的场景。
- 返回：yield 每一 step 的字典（包含当前 message 与状态摘要），并最终把 `self._conversation_state` 设置为最后状态。

### update_system_prompt_with_selected_resources(self, selected_resources)
- 作用：在检索之后，用选中的工具/数据/库/know-how 更新 `self.system_prompt`（通过 `_generate_system_prompt`），以便后续 LLM 调用能获得更聚焦的上下文。
- 参数：`selected_resources`（dict，通常来自 `_prepare_resources_for_retrieval`）。

### result_formatting(self, output_class, task_intention)
- 作用：使用 LLM 将 agent 的内部 `self.log`（历史）格式化为结构化输出，`output_class` 定义结构化输出格式（LangChain structured output）。
- 返回：解析后的字典（LLM 解析结果）。

### _parse_tool_calls_from_code(self, code: str) -> list[str]
- 作用：静态解析代码字符串以提取被导入或调用的工具名（仅名称列表）。
- 返回：工具名字符串列表，依赖 `module2api` 与 `_custom_functions`。

### _parse_tool_calls_with_modules(self, code: str) -> list[tuple[str, str]]
- 作用：类似上面，但返回元组列表 `(tool_name, module_name)`，用于更精确的工具追踪和权限校验。

### _inject_custom_functions_to_repl(self)
- 作用：把 `self._custom_functions` 注入到 Python REPL 的执行环境中（通过 util helper），以便执行代码时直接调用这些工具。

### create_mcp_server(self, tool_modules=None)
- 作用：基于当前 `module2api` 生成一个本地 `FastMCP` 服务器实例，暴露 agent 内部工具为 MCP endpoints，便于外部进程通过 MCP 调用。
- 返回：`FastMCP` 实例（尚未运行）；函数会在内部遍历 module 列表并注册 wrapper。

### save_conversation_history(self, filepath: str, include_images: bool = True, save_pdf: bool = True) -> None
- 作用：将会话历史导出并保存为 PDF（内部先生成临时 markdown，再调用 PDF 渲染器）。
- 参数：
  - `filepath`：目标路径（不必带 .pdf 扩展）；
  - `include_images`：是否把捕获的图像包含进输出；
  - `save_pdf`：是否真的保存（可选仅生成 markdown）。
- 重要实现：使用 `_generate_markdown_content(include_images)` 获取 markdown，再通过 `_convert_markdown_to_pdf`（内部 util）生成 PDF；设有超时处理（60s）并对异常做清理。

嵌套：`timeout_handler(signum, frame)`：内用于处理中断/超时信号（与 PDF 生成超时机制相关）。

### _generate_markdown_content(self, include_images: bool = True) -> str
- 作用：从 `self._conversation_state` 或 `self.log` 中提取并格式化会话内容，返回完整的 markdown 字符串。
- 返回：markdown 文本（包含步骤编号、代码执行片段、观察结果、图像占位，按 agent 的格式化规则）。

### _get_messages_for_processing(self)
- 作用：选择最佳消息来源：优先使用 `self._conversation_state['messages']`（若存在），否则 fallback 到 `self.log`。
- 返回：标准化的消息列表供后续 `_generate_markdown_content` 使用。

### _normalize_conversation_state_messages(self, messages)
- 作用：把 LangChain 风格的消息对象（`HumanMessage`, `AIMessage`, `SystemMessage` 等）转成统一的字典格式 `{'content','type','original'}`，便于后续处理与格式化。

### _normalize_log_messages(self, log_entries)
- 作用：把内部 `self.log`（字符串或特定结构）解析并规范化为与上面相同的格式。

### _process_message(self, message_data, content, step_number, first_human_shown, added_plots, include_images)
- 作用：根据消息类型（human/ai/other）分发到对应的处理器（`_process_human_message` / `_process_ai_message` / `_process_other_message`），并返回更新后的内容与步骤计数等。

### _process_human_message(self, clean_output, content, step_number, first_human_shown)
- 作用：处理用户消息（例如特殊提示“必须包含思考过程”），首次人类消息会以特殊样式展示。

### _process_ai_message(self, clean_output, content, step_number, added_plots, include_images)
- 作用：处理 AI 消息，尤其能识别并拆分含有 `<observation>` 标签或 `<execute>` 执行相关的输出，处理执行结果并插入图像/terminal 块等。

### _process_other_message(self, ...)
- 作用：处理既非 human 也非 ai 的消息类型（例如 observation-only 或系统消息），并进行合适的格式化添加。

### _process_execution_with_results(self, clean_output, content, added_plots, include_images, execution_results)
- 作用：在 AI 消息中找到与执行相关的记录（匹配执行标识），提取执行产生的图像或结果并嵌入到 markdown 中。

### _format_and_add_content(self, clean_output, content)
- 作用：对 AI 文本做最终格式化（处理列表、`<execute>` 标签、工具调用标注等），返回追加后的 markdown 文本块。
- 嵌套：`parse_tool_calls_wrapper(code)`：用于在格式化执行段时识别工具调用并把它们替换成可读形式。

### _add_execution_plots(self, matching_execution, content, added_plots, include_images)
- 作用：将执行结果中捕获的图像/plot 嵌入 markdown（避免重复添加）。

### _process_regular_ai_message(self, clean_output, content)
- 作用：普通 AI 文本的格式化分支，直接调用 `_format_and_add_content`。

### _convert_markdown_to_pdf(self, markdown_path: str, pdf_path: str) -> None
- 作用：封装调用 util `convert_markdown_to_pdf`，提供可靠的转换到 PDF 的路径封装。

### _clear_execution_plots(self)
- 作用：清理先前捕获的图像缓存，防止旧图重用。

### _generate_mcp_wrapper_from_biomni_schema(self, original_func, func_name, required_params, optional_params)
- 作用：基于 biomni 的 schema（必需/可选参数）生成 MCP-compatible 的同步 wrapper 函数，用于 MCP 服务器端点。
- 返回：一个 `wrapper` 函数（如果没有参数，返回无参 wrapper；否则返回接受 kwargs 的 wrapper）。

### launch_gradio_demo(self, thread_id=42, share=False, server_name="0.0.0.0", require_verification=False)
- 作用：启动一个基于 Gradio 的交互 Web UI，包装 agent 的 `go_stream` 或 `go` 功能用于演示与交互。
- 参数：配置 web 服务（是否共享、绑定地址、是否需要访问码验证等）。
- 嵌套：
  - `verify_access_code(code)`：用于可选的访问码验证逻辑。
  - `generate_response(prompt_input, inner_history=None, main_history=None)`：bridge 到 agent 执行并格式化界面输出，包含多轮会话管理、文件上传/下载、图片处理和生成 PDF 下载等功能。
  - `like(data: gr.LikeData)`：简单的交互事件回调用于 UI 的点赞功能等。
- 注意：需要 `gradio` 包，UI 逻辑较复杂且包含文件/图像处理与多线程/异步交互细节。

---

  **示例对话**

  下面是一个简短的示例回合，展示 `A1` 如何解析模型输出、执行代码并把结果回填给模型，最终给出解答。

  - 用户: 请计算 1 到 10 的平均值，并画一个直方图。

  - 模型（LLM）示例输出:
    <think>我将先计算平均值，然后绘制直方图。</think>
    <execute>
    # Python: 计算并画图
    import numpy as np
    import matplotlib.pyplot as plt
    data = np.arange(1,11)
    mean = data.mean()
    print("MEAN:", mean)
    plt.hist(data, bins=10)
    plt.savefig("hist.png")
    print("SAVED: hist.png")
    </execute>

   （说明：因为已在 `stop_sequences` 中注册 `</execute>`，模型在生成到闭标签处会尝试停止；agent 收到响应后进行解析与容错。）

  - Agent 本地行为（对应 `generate()` → `execute()`）：
    - 解析出 `<execute>` 代码段，识别为 Python（默认）。
    - 在受控环境执行代码（应用 `timeout_seconds`、捕获 stdout/stderr、保存生成的图片）。
    - 收集执行结果并构造成观察消息：
      <observe>
      STDOUT:
      MEAN: 5.5
      SAVED: hist.png

      ARTIFACTS:
      - hist.png (保存在 agent 工作目录)
      </observe>
    - 将该 `<observe>` 追加到对话状态并再次调用 LLM。

  - 模型（收到 observation 后）示例输出:
    <think>已看到计算结果和图像。</think>
    <solution>平均值为 5.5。直方图已生成并保存在 hist.png。</solution>

  Agent 将 `<solution>` 返回给用户并结束会话。

  说明要点：stop sequences 是截断辅助；真正的执行/回填是由 agent 的 `generate`/`execute` 节点解析并驱动的，agent 具备自动补标签、fenced-code 兼容与重试/终止等容错机制。

## 使用建议与注意事项
- 该模块高度依赖外部 LLM 客户端、工具库和本地 data lake。进行集成测试时，可通过创建 `A1(..., expected_data_lake_files=[])` 来跳过大文件下载。
- `add_tool` 与 `add_mcp` 会将函数注册到全局 `builtins._biomni_custom_functions`，在安全敏感环境请审查并限制可注册的函数。
- `_generate_system_prompt` 包含严格格式（`<execute>` / `<solution>`），与 agent 工作流深度耦合，修改格式时需同时更新 `generate`/`execute` 节点解析逻辑。

---

如果你希望我把每个函数的关键源码片段（关键逻辑行）也包含进文档，或者把这些说明注入到源码为 docstring，请告诉我，我可以继续补充或直接修改 `a1.py`。

---

## 深入解析：`_generate_system_prompt`（更详细）

目的：构造发送给 LLM 的系统级上下文（system prompt）以规范 agent 的整体行为。该 prompt 不是简单文本，而是严格模板化的上下文，包含：函数字典（function_intro）、工具说明（tool_desc）、data lake 列表、软件库说明、检索强调内容、以及执行/格式化规则。

主要组成（以 format_dict 的键为核心）：
- `function_intro`：简短说明 agent 可用函数的总体约束与调用范例；通常用于提示 LLM 如何调用函数并报告结果。
- `tool_desc`：按模块组织的工具 schema 列表（每个工具含 name/description/parameters），用于指导 LLM 在何种情境下调用哪一个工具。
- `import_instruction`：说明如何在生成代码时导入/引用这些工具（例如直接调用 `my_tool()` 或从模块导入的建议样式）。
- `data_lake_path` 与 `data_lake_content`：列出可用的生物数据集与简短描述，告诉 LLM 本地可以访问哪些文件以便在需要时引用或读取。
- `library_content_formatted`：列出可用软件/库与它们的功能说明，供 LLM 决定使用哪套软件工具链。

核心要求与格式约束（非常关键）：
- 计划先行：Prompt 明确要求 LLM 在动手前给出带复选框的分步骤计划（1. [ ] ...），并在每步完成后更新状态。这保证了可审计的逐步推理与执行记录。
- 执行标记：所有要执行的代码必须包裹在 `<execute>...</execute>` 标签里；支持多种语言（默认 Python；`#!R` 或 `#!BASH` 前缀用于 R/Bash）。
- 观察/输出标记：执行后产生的输出应通过 `<observe>...</observe>` 被 agent 捕获并回传给 LLM，作为下一轮上下文。
- 解答标记：当 LLM 准备给出最终答案时，必须使用 `<solution>...</solution>` 标签结束工作流，便于 agent 识别终止并产出最终结构化结果。

检索与自定义资源强调：
- 当 `is_retrieval=True` 时，Prompt 会更明确地把检索到的工具/数据/know-how 高亮放在前面（以提高相关性）。
- 自定义资源（`custom_tools`, `custom_data`, `custom_software`, `know_how_docs`）会被放在 prompt 的显著位置并加上使用建议或限制（例如商用文档的可用性说明）。

实现细节与工程注意事项：
- 模板化填充：`formatted_prompt = prompt_modifier.format(**format_dict)`，因此所有字段必须可字符串化且大小受 token 限制影响。大型 `know_how_docs` 会显著增加 prompt token 数。
- stop token 配合：类如 `</execute>`、`</solution>` 等需要在 LLM 客户端（`get_llm`）的 `stop_sequences` 中注册，以确保模型在输出到达这些标签时停止，从而便于解析。
- Token 与上下文管理：若 data lake 或 know-how 内容过大，建议只把摘要或检索到的片段注入 `is_retrieval=True` 的 prompt，而非把全部文档塞入初始 prompt。

示例片段（概念性）：
```
System: You have access to functions: analyze_gene(), run_blast()
Environment Resources:
- Function Dictionary:
  - analyze_gene(gene_name): ...
---
<execute>print('Hello world')</execute>
<observe> Hello world </observe>
```

修改提示：若你希望 prompt 更短、更严格或支持额外的标签（例如 `<explain>`），请同时调整 `generate`/`execute` 节点的解析逻辑。

---

## 在源码中 `<think>` 的位置与说明

- 我在 `biomni/agent/a1.py` 中查到了两处与 `<think>` 相关的处理（见行号参考）：
  - 在解析模型输出时，如果检测到开标签但没有闭合标签，代码会自动补全：
    - 代码片段（大意）: `if "<think>" in msg and "</think>" not in msg: msg += "</think>"` （见源码行）
  - 使用正则匹配 `<think>` 内容：
    - 代码片段（大意）: `think_match = re.search(r"<think>(.*?)</think>", msg, re.DOTALL | re.IGNORECASE)`

- 说明：源码并没有在 `prompt_modifier` 模板里强制包含 `<think>` 标签（模板要求的是先给出计划并遵守 `<execute>` / `<solution>`），但 `generate()` 节点为兼容性额外支持 `<think>` 标签。也就是说模型可以选择使用 `<think>` 来显式输出中间思考，agent 会识别并据此继续流转；但 `<think>` 并非必须存在。

## 整合版（把之前的答案浓缩并加入此文件）

以下为整合后的关键点（可直接作为快速参考粘贴使用）：

- `_generate_system_prompt`：
  - 作用：构建给 LLM 的系统上下文模板，包含函数字典、工具说明、data lake、软件列表与使用约束。
  - 强制格式：要求计划优先（带复选框），执行代码用 `<execute>...</execute>`，执行结果用 `<observe>...</observe>`，最终答案用 `<solution>...</solution>`。
  - 工程要点：对大型 know-how 文档应做检索分段注入；`stop_sequences` 需包含 `</execute>`/`</solution>` 以便安全截断输出。

- `generate(state)`（节点逻辑）：
  - 调用 LLM：`response = self.llm.invoke(messages)`，取 `response.content` 并规范化为字符串。
  - 容错与兼容：
    - 自动补闭合标签（`</execute>`,`</solution>`,`</think>`）以减少解析失败。
    - 若无 `<execute>`，尝试把 Markdown 代码块 ```...``` 识别为 execute（兼容部分模型输出风格）。
  - 匹配顺序与路由决策：
    - 若检测到 `<solution>` → `state['next_step'] = 'end'`（结束）。
    - elif 检测到 `<execute>` → `state['next_step'] = 'execute'`（进入执行）。
    - elif 检测到 `<think>` → `state['next_step'] = 'generate'`（继续生成/思考）。
    - else → 进入错误处理（提示模型必须包含 `<execute>` 或 `<solution>` 并重试，最多重试两次，仍失败则结束）。

- `execute(state)`（节点逻辑）：
  - 从最后一条 AIMessage 中提取 `<execute>` 内容并根据语言标记选择执行器（Python/R/Bash/CLI）。
  - Python 执行前会注入自定义函数并清理旧图表，执行通过 `run_with_timeout(run_python_repl, ...)` 以防卡死。
  - 捕获 stdout/stderr、异常与生成的图像（base64），把执行结果包装成 `<observation>...</observation>` 并追加到 state 供下一轮使用。

- 错误处理策略（无 `<execute>` 也无 `<solution>`）：
  - agent 会向 state 追加一条 HumanMessage，提醒模型必须包含 `<execute>` 或 `<solution>`，并将 `next_step` 设回 `generate` 以重试。
  - 若重复错误超过阈值（>=2），agent 会终止会话并附加一条终止说明的 AIMessage。

- `go` vs `go_stream`：
  - `go(prompt)`：同步阻塞式执行整个 workflow，适合脚本/一次性查询，结果保存在 `self._conversation_state`。
  - `go_stream(prompt)`：返回 generator，逐步 yield 每次状态更新，适合 UI 实时渲染与中断控制。

如果你希望，我可以：
- 把上面整合内容插入到 `a1.py` 的适当函数 docstring 中；或
- 在 `docs/a1_functions.md` 中增加行号引用并链接回源码（例如把关键正则与分支的原始代码片段以可跳转链接形式加入）。
请告诉我你的偏好。

## 深入解析：`configure`、工作流（StateGraph）与交互流程

目的：`configure` 将 agent 的静态资源（模块函数、data lake、know-how）与行为模板（system prompt）结合，并创建一个可执行的状态机（`StateGraph`），该状态机定义了“生成（generate）→执行（execute）→生成”的闭环流程。

关键步骤（高层顺序）：
1. 收集资源：从 `self.path` 的 `data_lake`、`module2api`、`library_content_dict` 与 `_custom_*` 中准备 `tool_desc`, `data_lake_with_desc`, `library_content_list`。
2. 生成 initial system prompt：调用 `_generate_system_prompt(..., is_retrieval=False)`。
3. 定义节点函数并构建图：
   - `generate(state)`：把系统 prompt 与会话历史（state['messages']）打包后调用 LLM，得到 AI 消息；该消息通常包含计划、解释、以及可选的 `<execute>` 代码段。生成节点负责把 LLM 输出追加到 state 并设置 `next_step`（例如 'execute' 或 'end'）。
   - `execute(state)`：解析刚刚 AI 消息中是否存在 `<execute>` 标签；若有则按标签内语言执行代码（Python/R/Bash），或调用封装的工具函数（通过 `_parse_tool_calls_with_modules` 识别要调用哪些工具）。执行结果被以 `<observe>` 形式追加回 state，供下一次 `generate` 使用。
   - Routing：`routing_function` 根据 `state['next_step']` 决定下一节点（`'generate'`、`'execute'`、`'end'`）。当 LLM 输出 `<solution>` 或没有后续步骤时，路由到 `end`。
4. 编译图：使用 `workflow.compile()` 得到 `self.app`，该对象支持 `stream(...)` 调用以流式运行状态机并产出中间状态。

关于 `StateGraph` 的工作模型（交互式说明）：
- 把 agent 的一轮交互看成一个“消息队列”与“状态转移”系统：
  1) 起始：用户消息以 HumanMessage 存入 state['messages']，`next_step=None`。
  2) generate：LLM 被调用，输出被追加为 AIMessage（可能含 plan 与 `<execute>`）。
  3) 如果 AIMessage 含 `<execute>`，routing 指向 execute：解析并执行，捕获 stdout、错误、图片等，构造 observation 放回 state；否则直接回到 generate 让 LLM 根据观察继续。
  4) LLM 读取 observation（系统 prompt + 历史 + observation），更新计划或给出最终 `<solution>`，如无更多执行则结束。

执行节点（`execute`）的详细行为：
- 解析 `<execute>` 标签：支持代码块内的语言提示（`#!R`, `#!BASH`），或默认 Python。
- 执行方式：
  - Python：会把 custom functions 注入 REPL（通过 `_inject_custom_functions_to_repl`），使用 `run_python_repl` 或内部安全执行器（带 `timeout_seconds`）执行；捕获返回值、异常与任何图片输出。
  - Bash：通过 `run_bash_script` 执行并捕获 stdout/stderr。
  - R：通过 `run_r_code` 或 `Rscript` 方式执行并抓取输出。
  - 工具函数：若代码显式调用已注册工具（`module2api` 或 `_custom_functions`），agent 会优先将这些调用路由为函数调用而非在 REPL 中动态导入执行（更安全、可审计）。
- 输出回传：所有执行的结果会被格式化成 `<observe>...</observe>` 并写入 state，以便 LLM 的下一次调用能够使用这些观测数据。

持久化与恢复：
- `self.checkpointer = MemorySaver()` 与 `self.app.checkpointer = self.checkpointer`：StateGraph 的检查点可将中间状态保存，用于恢复或调试。

性能与安全考量：
- 并非所有执行都安全，`add_tool` 将自定义函数注入全局会改变运行时权限边界；在受限环境请禁用此类功能或在沙箱中运行。
- prompt 大小：把大量 know-how 全部放入初始 prompt 会导致 token 爆炸，建议在初始 prompt 只放摘要并在检索时动态注入全文片段。

---

## 工作流实例：从用户问题到最终回答（逐步示例）

情景：用户问“如何比较两个基因表达矩阵的差异？”

执行流程：
1. 用户输入（HumanMessage）进入 state。
2. 若 `use_tool_retriever=True`，调用 `_prepare_resources_for_retrieval(prompt)`：
   - 聚合所有工具描述、data lake 列表与 know-how 摘要并把它们连同用户问题一起发给 `ToolRetriever`（又是一个 LLM 驱动的选择器），得到推荐的工具/数据/文档索引。
   - `update_system_prompt_with_selected_resources` 会把这些被选资源注入 system prompt 的显著位置。
3. `generate` 节点调用 LLM，基于最新 system prompt 与历史生成计划和可能的 `<execute>` 代码（例如生成 Python 代码调用 `scipy` 或调用内建 `compare_matrices()` 工具）。
4. 如果 LLM 输出了 `<execute>`：
   - `execute` 解析语言（Python），识别并替换工具调用为直接函数调用或在 REPL 中执行代码（取决于调用形式）。
   - 执行结果（例如统计表、p-value、图片）被捕获并包装成 `<observe>`。
5. 回到 `generate`，LLM 读取 observation 并据此决定下一步（进一步分析、生成解释或输出 `<solution>`）。
6. 若 LLM 输出 `<solution>`，路由到 `end`，会话完成；agent 可调用 `save_conversation_history` 导出完整流程文档。

---

## `go` 与 `go_stream`：区别与使用指南

共同点：
- 两者都以 `self.app.stream(...)` 作为运行入口，最终会把 `self._conversation_state` 设置为最后的 conversation state，并把执行日志保存在 `self.log`。

主要区别：
- `go(prompt)`：
  - 同步阻塞式执行，函数在内部循环 `for s in self.app.stream(...)`，直到状态机结束才返回。
  - 适合脚本与一次性批处理（例如在 CLI 中运行一个问题然后退出），默认在控制台打印中间调试信息，并把最终 state 存在实例上。
  - 返回值为 None（结果保存在实例属性），调用者通常随后调用 `save_conversation_history` 或检查 `self._conversation_state`。

- `go_stream(prompt)`：
  - 返回一个 Python 生成器（generator），在状态机每产出一个步骤时 yield 一个包含状态/消息的字典。
  - 适合实时 UI（如 `Gradio`）或需要在前端逐步渲染 agent 思考与执行过程的场景。
  - 更细粒度控制：调用方可以在每次 yield 时更新界面、显示进度或中断会话（若 UI 支持取消）。

选择建议：
- 若需要交互式界面或流式展示 agent 思考过程，优先使用 `go_stream` 并在前端逐步消费 generator 的输出。
- 若只需执行并在本地得到最终结果，使用 `go` 简单直接。

示例（伪代码）：
```
agent = A1()
# 同步
agent.go('比较两个矩阵的差异')
print(agent._conversation_state)

# 流式（UI）
for step in agent.go_stream('比较两个矩阵的差异'):
    render_step(step)
```

---

已将以上详细内容追加入 `docs/a1_functions.md`，接下来我将完成 TODO：把 `go`/`go_stream` 部分标记为完成并更新任务列表，或者你希望我先把这些说明也注入 `a1.py` 的 docstrings？
