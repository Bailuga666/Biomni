<!-- Auto-generated: biomni/agent/react.py 说明 -->
# `biomni.agent.react` 模块说明

简介
- 文件: `biomni/agent/react.py`
- 目的: 提供一个基于 ReAct（或类似工具调用）模式的轻量 agent 实现，封装了工具注册、超时包装、prompt 构建以及基于 `StateGraph` 的循环执行流程。

主要类
- `react`
  - 职责: 初始化 LLM 与工具集合、为工具添加超时保护、构建系统 prompt 并创建可执行的工作流（LangGraph `StateGraph`），对外暴露 `configure()`、`go()` 与 `add_tool()` 等方法。

构造函数与初始化
- __init__(self, path=None, llm=None, use_tool_retriever=None, timeout_seconds=None)
  - 使用 `default_config` 的值作为默认参数。
  - 读取 `module2api`（工具描述表），将各工具通过 `api_schema_to_langchain_tool` 转为 LangChain tool 对象集合。
  - 如果 `use_tool_retriever` 为 True，会创建 `ToolRegistry` 与 `ToolRetriever` 以支持基于 prompt 的工具/数据检索。
  - 对工具集合调用 `_add_timeout_to_tools`，为每个工具的执行函数添加一个进程级别的超时包装（见下）。

超时包装
- `_add_timeout_to_tools(self, tools)`
  - 把每个 tool 的 `.func` 替换为 `timed_func`：在子进程中执行原始函数，等待 `timeout` 秒，超时则终止子进程并返回超时错误字符串。
  - 使用 `multiprocessing.Process` + `Queue` 实现隔离，确保长耗时或阻塞的工具不会卡住主进程。
  - 优点：强制隔离与可杀死进程；缺点：跨进程通信开销、传参/返回需要可序列化。

扩展工具注册
- `add_tool(self, api)`
  - 使用 `inspect.getsource(api)` 与 `function_to_api_schema`（LLM 辅助）生成工具 schema，然后转为 LangChain tool 并通过 `_add_timeout_to_tools` 包装后加入 `self.tools`。

配置与 prompt
- `configure(self, plan=False, reflect=False, data_lake=False, react_code_search=False, library_access=False)`
  - 根据传入 flag 组装 `prompt_modifier`（system prompt 文本），可选择包含：计划模板（plan）、反思逻辑（reflect）、数据湖条目（data_lake）与库访问说明。
  - 当 `react_code_search` 为 True 时，prompt 简化为仅暴  露 `run_python_repl` 与 `search_google` 两个工具的用法示例。
  - 将 prompt 封装为 `ChatPromptTemplate`，并保存在 `self.system_prompt` / `self.prompt`。
  - 调用 `_create_custom_react_agent` 构建 `self.app`（StateGraph workflow）。

工作流实现（LangGraph）
- `_create_custom_react_agent(self, llm, tools, prompt)`
  - 绑定工具到 LLM：`llm_with_tools = llm.bind_tools(tools)`，这样 LLM 在生成时可以产生 tool_calls 字段。
  - 定义两个主要节点：
    - `agent`：向 LLM 发送 SystemMessage + 状态消息，接收 LLM 响应（可能含 `tool_calls`）。
    - `tools`：遍历最近消息中的 `tool_calls`，调用对应工具（通过 `tools_by_name` 映射），将执行结果封装为 `ToolMessage` 返回给状态。
  - 条件路由 `should_continue`：如果最后一条消息没有 `tool_calls` 则结束（END），否则跳到 `tools` 节点。工具节点执行完后回到 `agent` 节点，形成循环（agent → tools → agent ...）。
  - 返回编译后的 workflow（可流式执行）。

运行入口
- `go(self, prompt)`
  - 若启用了 `use_tool_retriever`：收集可用工具、data-lake 与库描述，调用 `self.retriever.prompt_based_retrieval(prompt, resources, llm=self.llm)` 选择相关资源，并打印选中列表；随后用检索到的工具重建 agent workflow（仅保留相关工具，且确保包含 `run_python_repl`）。
  - 创建 `inputs = {"messages": [("user", prompt)]}` 并通过 `self.app.stream(inputs, stream_mode="values")` 运行工作流，收集并返回日志与最终消息内容。

辅助方法
- `result_formatting(self, output_class, task_intention)`：使用 LLM 的 structured output 能力对 `self.log`（agent 历史）做格式化/校验，返回解析结果。

重要实现细节与注意事项
- 工具超时是基于子进程的强制终止；这可以中断任意阻塞调用，但要求工具参数与返回值可被 multiprocessing 序列化。
- 使用 LLM 的 `bind_tools` 与 tool_calls 机制：依赖于所用 LLM/客户端支持工具调用序列化（如 `tool_calls` 字段）；若 provider 不支持该字段，工作流可能退化为纯文本交互。
- `use_tool_retriever` 提供基于 prompt 的工具与数据检索，用于缩小上下文与工具集，减少 prompt token 且提高相关性。
- `StateGraph` 的循环依赖基于是否存在 `tool_calls` 判断结束条件：如果 LLM 一次性产出没有 tool_call，工作流将终止并返回结果。

使用建议
- 若工具会返回复杂对象（非 JSON），考虑在工具实现中转换为 JSON-serializable 结构，确保 `ToolMessage` 内容能被 downstream 解析。
- 对需跨进程传递的大对象（如大型 numpy 数组），应避免直接通过 `Process` 返回；改为写文件并返回路径或摘要。
- 在不信任外部工具时，先通过 `configure()` 限制激活工具集合并在 `go()` 前运行检索以只启用必要工具。

示例快速浏览
- 初始化并执行（伪代码）:

```python
from biomni.agent.react import react
agent = react(use_tool_retriever=True)
agent.configure(plan=True, data_lake=True)
log, final = agent.go('给我找出与基因X相关的文献并汇总')
```

文件位置：`biomni/agent/react.py`

---
_自动生成：如果需要我可以把此内容合并到 `docs/a1_functions.md` 或把示例插入 `react.py` docstring。_

## 补充：工作流实现细节、与 `A1` 的 LLM 关系、以及检索选择逻辑

下面是对你关心的三部分要点的精炼总结，代码位于 `biomni/agent/react.py` 与 `biomni/model/retriever.py`：

- 工作流实现（更详细）
  - 结构：基于 `StateGraph` 的两节点循环：`agent`（调用 LLM）→ `tools`（执行 tool_calls）→ `agent`。
  - `agent` 节点：构造 SystemMessage（`self.system_prompt`）并把当前消息发给绑定了工具的 LLM（`llm.bind_tools(tools)`），LLM 可能返回带 `tool_calls` 的结构化响应。
  - `tools` 节点：遍历 `tool_calls`，用 `tools_by_name[name].invoke(args)` 调用工具并把结果封装为 `ToolMessage` 返回；工具执行前已被 `_add_timeout_to_tools` 用子进程包装以防阻塞。
  - 路由：`should_continue` 检查最后一条消息是否包含 `tool_calls`；若无则结束（END），否则继续到 `tools` 节点，形成循环。

- `react` 使用的 LLM 与 `A1` 使用的 LLM 有何关系
  - 同源不同实例：两者都可通过 `get_llm()` 创建，可能使用相同底层模型，但通常为独立实例、各自持有自己的 prompt 与配置。
  - 设计目标不同：
    - `A1`（`a1.py`）采用基于文本标签的协议（`<execute>` / `<observe>` / `<solution>`），agent 自己解析标签并在本地运行代码（`generate()` / `execute()` 节点）。
    - `react` 倾向使用 provider/LangChain 的工具调用接口（`tool_calls`），由 LLM 直接产出调用计划，agent 执行并回填结果。
  - 因此两者可互换底层模型，但交互模式、解析逻辑与容错策略不同。

- `prompt_based_retrieval`（谁来选？如何选？）
  - 实现位置：`biomni/model/retriever.py::ToolRetriever.prompt_based_retrieval`。
  - 流程：
    1. `ToolRetriever` 把所有候选资源（tools、data_lake、libraries、know_how）格式化为一个长 prompt，连同用户 query 一起发给传入的 `llm`（如 `self.llm`）。
    2. Prompt 要求 LLM 返回每类资源的“索引列表”（例如 `TOOLS: [0,3,5]`），并包含选择准则以指导模型（例如优先数据库工具、包含文献检索工具等）。
    3. `ToolRetriever` 解析 LLM 的文本响应（正则提取 `TOOLS:
[...]` 等），将索引映射回资源并返回实际的资源子集。
  - 结论：资源的“选择”由 LLM 根据 prompt 决策（LLM 输出索引），而 `ToolRetriever` 负责 prompt 构造、解析、边界检查与最终映射（即“LLM 选 + retriever 解析/执行”）。

如果你愿意，我可以把一个小的示例：完整的 retrieval prompt（截断版）与一个可能的 LLM 返回示例，加入到本文件以便调试与复现。要我现在加吗？

**A1 vs React 对比（简洁）**

- **本质**: 智能层相同，均由 LLM 提供理解与决策；差别在输出接口与执行边界。
- **输出形式**: `A1` 输出带 `<execute>` 标签的可执行代码；`react` 输出结构化的 `tool_calls`（name + args）。
- **执行与安全**: `A1` 直接运行模型生成的代码（任意代码执行风险）；`react` 通过已注册工具调用，便于参数校验与沙箱隔离。
- **可审计性与可控性**: `react` 更易日志化、审计与权限控制；`A1` 更灵活但难以静态验证。
- **错误模式**: `A1` 易出现语法/运行时或越权命令；`react` 更常见工具选择或参数错误，通常更易重试与修正。
- **适用场景**: 实验性、快速原型与复杂脚本适合 `A1`；生产化、合规与可复用组件优先 `react`。
- **实践建议**: 若需安全、可重复与审计首选 `react`；需灵活执行脚本且可接受隔离成本时可选 `A1`，并加强沙箱与监控。

相关源码位置（便于细读）:

- `biomni/agent/a1.py` — `generate` / `execute` 节点（解析 `<execute>`、执行代码并回填 `<observation>`）。
- `biomni/agent/react.py` — `call_model` / `tool_node`（LLM 产生 `tool_calls`，agent 遍历并调用对应工具）。

