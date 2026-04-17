<!-- Auto-generated: biomni 模块与文件说明索引 -->
# biomni 包模块索引

本文件概述 `biomni/` 包内的主要模块与子包，简短说明每个文件/模块的作用，方便快速定位源代码职责与使用场景。

注意：本索引基于源码头部 docstring 与文件名语义整理，如需更细粒度函数级说明，请参见 `docs/a1_functions.md` 或请求针对具体文件的深入分析。

## 顶层模块

- `biomni/__init__.py`
  - 包版本导出（`__version__`）。
- `biomni/config.py`
  - 全局配置类 `BiomniConfig` 与 `default_config`，从环境变量支持覆盖，包含路径、超时、LLM 默认设置等。
- `biomni/llm.py`
  - LLM 工厂 `get_llm()`：根据 model 名称或 `source` 选择并构造对应的 LangChain LLM 包装器（OpenAI、Anthropic、Ollama、Gemini、Bedrock、Custom 等），处理 stop sequences、Responses API 特殊性与 provider 特性。
- `biomni/utils.py`
  - 通用工具函数集合：运行 R/Bash/CLI 的封装、带超时的执行（`run_with_timeout`）、将函数代码转为 API schema 的辅助（`function_to_api_schema`）、以及多种文本/执行结果格式化与 I/O 帮助函数等。
- `biomni/version.py`
  - 包版本字符串。

## 子包：`agent/`（智能代理相关）

- `biomni/agent/a1.py`
  - 核心 Agent 实现 `A1`：初始化配置、加载数据 lake 与 know-how、构建 system prompt、创建 StateGraph 工作流（`generate` / `execute` 节点）、执行 LLM-驱动的代码执行循环、支持 `go()` 与 `go_stream()`。详见 `docs/a1_functions.md`。
- `biomni/agent/__init__.py`
  - Agent 子包导出（初始化入口）。
- `biomni/agent/qa_llm.py`
  - 与问答/检索相关的 LLM 帮助器（QA 流程、answer formatting 等），供 agent 在檢索或结果汇总时使用。
- `biomni/agent/react.py`
  - 基于 ReAct 风格或交互式策略的辅助实现（策略/路由/简单决策示例）。
- `biomni/agent/function_generator.py`
  - 帮助生成函数包装、将函数源码转 schema/描述，配合 `add_tool` 和工具注册使用。
- `biomni/agent/env_collection.py`
  - 环境收集/采样工具，可能用于收集上下文、样例或数据以扩充 prompt 或检索索引。

## 子包：`tool/`（工具集合与描述）

此子包按学科或功能组织一组“工具”模块（每个文件通常导出若干可由 agent 调用的函数/工具），并包含工具描述用于生成 system prompt 与检索。

- `biomni/tool/tool_registry.py`
  - 简单的工具注册/索引类 `ToolRegistry`，维护工具列表、id、查找与持久化（pickle）。
- `biomni/tool/support_tools.py`
  - 支撑性工具（例如 `run_python_repl` 等）用于在 agent 执行环境中运行 Python 代码并捕获输出。
- 领域工具（按文件名）：
  - `biochemistry.py`、`bioengineering.py`、`bioimaging.py`、`biophysics.py`、`cancer_biology.py`、`cell_biology.py`、`database.py`、`genetics.py`、`genomics.py`、`glycoengineering.py`、`immunology.py`、`lab_automation.py`、`literature.py`、`microbiology.py`、`molecular_biology.py`、`pathology.py`、`pharmacology.py`、`physiology.py`、`protocols.py`、`synthetic_biology.py`、`systems_biology.py`
  - 说明：这些模块封装了学科特定的辅助函数、数据处理与检索适配器，供 agent 在生成代码或调用工具时使用并列入 system prompt 的函数字典。
- `biomni/tool/example_mcp_tools/pubmed_mcp.py`
  - MCP 示例工具：展示如何把外部 MCP 服务工具包装并注册到 agent 中用于调用。
- `biomni/tool/tool_description/` 下的多个文件
  - 每个对应学科的工具描述文本（用于填充 system prompt 的工具说明部分），例如 `biochemistry.py`、`genomics.py` 等。

## 子包：`model/`

- `biomni/model/retriever.py`
  - `ToolRetriever`：基于 prompt 的检索器实现，用于根据用户查询选择最相关的工具、data-lake 条目与 know-how 文档，辅助生成更小、更相关的 system prompt（当 `use_tool_retriever=True` 时启用）。

## 子包：`task/`

- `biomni/task/base_task.py`
  - 任务基类：定义任务生命周期与接口，供具体任务实现继承。
- `biomni/task/hle.py`、`biomni/task/lab_bench.py`
  - 具体任务实现示例（例如 HLE：patient gene detection 相关 benchmark 任务、lab bench 操作封装等），用于评估或示例自动化工作流。

## 子包：`know_how/`

- `biomni/know_how/loader.py`
  - `KnowHowLoader`：加载 `know_how/` 下的 markdown 文档，抽取标题、描述与 metadata，并把文档作为可注入 system prompt 的资源管理（支持 add_custom_document 等接口）。
- `biomni/know_how/__init__.py`
  - 子包导出。

## 其他脚本与辅助工具

- `biomni/biorxiv_scripts/`：
  - `extract_biorxiv_tasks.py`, `generate_function.py`, `process_all_subjects.py`：用于从 bioRxiv 或论文集合中提取任务、自动生成函数描述或批量处理学科数据的脚本（离线数据准备与实验）。

- `biomni/eval/biomni_eval1.py`
  - 评估脚本：包含 benchmark/评测相关的评价逻辑与指标汇总代码，用于衡量 agent 在指定任务集上的表现。

## 使用建议与下一步

- 这是一个高层索引，覆盖了 `biomni/` 包的主要文件与职责。如果你想要：
  - 针对某个文件（例如 `biomni/agent/a1.py` 或 `biomni/tool/support_tools.py`）生成函数级别说明或把示例插入 docstring，我可以继续把更详细内容写入对应文件或把片段加入到 docs。 
  - 把此索引追加到 README 或生成 cross-linked文件引用（带行号），我也可以进行插入或创建引用链接。

要我现在把这个索引保存到仓库并打开 `biomni/agent/a1.py` 做 docstring 注入吗？

---
_自动生成于代码库扫描，若需更新请重新运行索引生成任务。_
