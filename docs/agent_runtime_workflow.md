**Agent 运行时流程：从提问到最终响应（详尽版）**

目的：详细而真实地说明当你对 `A1` 智能体调用一次问答（`a.go(prompt)`）时，代码在内部按什么顺序做了哪些工作、哪些模块会被触发、以及有哪些可配置或需注意的点。

参考代码：`biomni/agent/a1.py`、`biomni/llm.py`、`biomni/model/retriever.py`、`biomni/tool/tool_registry.py`、`biomni/utils.py` 中的执行/运行辅助函数。

---

## 高层概览

1. 资源检索（可选）——根据问题选出相关工具 / datalake / know‑how 文档（`ToolRetriever`）。
2. 更新系统提示——把选定资源（工具描述、数据项、know‑how）注入系统提示和上下文。 
3. 发起 LLM 推理流（stream）——调用主推理模型或子模型（`get_llm` 返回的客户端），流式获得中间与最终输出。 
4. 解析并执行“可执行片段”——如果 LLM 输出含有工具调用/执行代码（如 `<execute>` 标签、或结构化工具调用），Agent 解析并执行（本地执行 / 调用注册工具 / 调用 MCP）。
5. 收集执行结果并反馈给 LLM（可多轮）——把执行输出或工具返回值作为新的观察/消息再次送入 LLM，直至找到解决方案或终止。 
6. 返回并保存会话日志——`go()` 返回最终文本与逐步日志，必要时可导出为 PDF/Markdown 保存。 

---

## 详细逐步流程（代码级对应）

1) 构建查询输入
- 调用者执行：`log, final_output = a.go(prompt)`（或 `go_stream`）。
- `A1.go()` 会先设置 `self.critic_count`、`self.user_task`、并在启用工具检索时调用 `_prepare_resources_for_retrieval(prompt)`。

2) 资源检索（ToolRetriever） — 可选但常用
- 函数：`biomni/model/retriever.py::ToolRetriever.prompt_based_retrieval`。
- 输入：用户 `prompt` + `resources`（包含 `tools`, `data_lake`, `libraries`, `know_how` 列表）。
- 行为：构建检索型提示（把资源按编号列出），调用短文本 LLM（可用自带或临时创建的检索模型）让模型返回“应选择哪些索引”。
- 输出：各类资源的索引集合 → `A1` 用这些索引选出具体资源并传给 `update_system_prompt_with_selected_resources`。

3) 更新系统提示与上下文
- `A1.update_system_prompt_with_selected_resources(...)` 会把选中的工具描述、data‑lake 项与 know‑how 文档拼入 system prompt 模板中（便于 LLM 在推理时知晓可用工具和本地证据）。
- 这一步对可执行性与可验证性很关键：系统提示会包含工具的名称、参数和简要说明。

4) 发起流式推理（self.app.stream）
- `A1` 构造 `inputs = {"messages": [HumanMessage(content=prompt)], "next_step": None}` 和 `config = {"recursion_limit": 500, ...}`，然后调用 `for s in self.app.stream(inputs, stream_mode="values", config=config):`。
- `self.app.stream`（项目使用的执行框架，如 langgraph/StateGraph）会逐步运行 agent 的状态图：
  - 每一步中，agent 可以调用 `self.llm.invoke(...)`（`self.llm` 来自 `get_llm()`），或调用子工具、执行 Python/R/系统命令，或进行检索。 
  - `stop_sequences`（例如 `</execute>`, `</solution>`）用于界定生成片段的边界，Agent 在调用 `get_llm` 时把这些作为参数传入（见 `biomni/llm.py`）。

5) LLM 输出的解析与动作执行
- LLM 输出可能包含：纯文本回答、结构化 JSON、工具调用指令、或带执行标签的代码块（agent 使用 `biomni/utils.py` 中的解析器函数，如 `parse_tool_calls_from_code`, `parse_tool_calls_with_modules`, `has_execution_results` 等来识别）。
- 常见执行路径：
  - 生成“工具调用” → Agent 将调用 `ToolRegistry` 中相应工具（registry 提供工具 schema、id、name）；工具实现可能是本地 Python 函数、外部服务的 REST wrapper、或 MCP 暴露的工具。工具调用接口由 `ToolRegistry` 管理注册、由 agent 负责调用并捕获返回。 
  - 生成“可执行代码”（Python/R/bash）被标记为要执行 → Agent 使用 `run_python_repl`, `run_r_code`, `run_bash_script` 等封装函数在子进程或隔离环境中执行，并把 stdout/stderr/返回值捕获为观察。
  - 生成检索或数据库查询 → Agent 可能调用内部抽象（例如 `biomni/tool/database.py`），而这些模块自身可能再次调用 `get_llm`（例如做结构化 API 生成），从而触发对不同模型的调用（“子任务路由”）。

6) 将执行结果反馈给 LLM（多轮循环）
- Agent 把工具返回或代码执行输出格式化（`format_observation_as_terminal`, `format_lists_in_text` 等）并作为新的消息或观察注入到后续 LLM 調用中，触发新一轮 reasoning / synthesis。
- 该循环可重复多次，直到达到终止条件（LLM 输出终结标记、超时、或找到满足条件的解）。

7) 流式输出收集与返回
- 在 `for s in self.app.stream(...)` 的每个迭代中，Agent 用 `pretty_print(message)` 格式化当前消息并追加到 `self.log`，最终把最后一条消息内容作为 `final_output` 返回。
- `A1.go()` 返回 `(self.log, message.content)`；`A1.go_stream()` 则以 generator 的形式逐步产出每一步的输出，适合实时前端显示。

8) 会话保存与导出（可选）
- 若需要保存，Agent 提供 `save_conversation_history(...)` 等工具把 `self._conversation_state` 转为 Markdown/PDF 并写盘（需要外部依赖如 `weasyprint`、`pandoc` 等）。

---

## 重要模块说明（在哪儿看代码）

- `biomni/agent/a1.py`：主流程、资源准备、`go()` / `go_stream()`、系统提示拼接、工具调用协调。  
- `biomni/llm.py`：`get_llm()` 实現（根据 `model` 推断 `source`，並从对应环境变量读取 API key）；传入 `stop_sequences` 控制生成边界与特殊 gpt-5 Responses API 的差异化处理。 
- `biomni/tool/tool_registry.py`：保存工具元数据、按 id/name 查找工具、提供 `document_df` 以供检索索引。 
- `biomni/model/retriever.py`：提示式资源检索实现（把资源编号列表作为 prompt，要求 LLM 返回相关索引）。
- `biomni/utils.py`：包含执行/解析/格式化等实用函数（解析 LLM 输出、运行代码、格式化终端输出等）。

---

## 常见故障與调试建議

- 若 LLM 报错“未提供 API key”或客户端初始化失败：检查 `.env` 或系统环境变量（`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `OPENAI_ENDPOINT`, `GEMINI_API_KEY`, `GROQ_API_KEY` 等）。
- 若 agent 卡在下载/检索步骤：看 `A1` 初始化日志（会打印 "Checking and downloading..."）、或在调用 `go()` 时观察控制台流式输出。 
- 若工具调用报错：确认 `ToolRegistry` 中工具 schema 的 `required_parameters` 是否正确，工具实现是否可用（本地函数 / MCP 服务是否可达）。
- 若多轮对话上下文丢失：确保你在同一 agent 实例内维护 `history`（或在调用中注入历史消息），并注意模型上下文窗口与 token 成本。 

---

## 安全与运行环境注意事項

- Agent 可能执行 LLM 生成的代码並发起网络请求：在生产环境请务必使用沙箱/隔离环境（容器、受限用户、网络策略）。
- 小心凭据管理：不要把 `.env` 提交到仓库，使用 CI/云 secret 或平台提供的密钥管理。 

---

如果需要，我可以：
- 在 `a1.py` 中把文档中的关键调用处打注释（行注）以便调试；
- 生成一个不调用真实 API 的本地 mock 演示脚本，演示完整“检索→工具调用→执行→反馈→总结”的交互流程。
