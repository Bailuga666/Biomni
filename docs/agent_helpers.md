# 辅助 Agent / 工具类文件汇总

本文档简要说明仓库中三个辅助 agent / 工具实现的作用、关键方法与典型调用位置，便于快速查阅与维护。

**涉及文件**
- [biomni/agent/env_collection.py](biomni/agent/env_collection.py)
- [biomni/agent/function_generator.py](biomni/agent/function_generator.py)
- [biomni/agent/qa_llm.py](biomni/agent/qa_llm.py)

---

**1. `env_collection.py` — PaperTaskExtractor**

- 作用：把学术论文拆成若干文本块，使用 LLM 提取“可编程且通用”的计算任务、数据库与软件，最后合并为结构化 JSON 输出，方便后续自动化处理或工具化实现。
- 关键方法：
  - `configure()`：构建两个严格提示（chunk 分析与合并为 JSON 的 consolidation prompt）。
  - `process_paper()`：按 `chunk_size`/`chunk_overlap` 分块并逐块调用 `_process_chunk()`。
  - `_process_chunk()`：对单块调用 LLM，返回文本结果。
  - `_consolidate_tasks()`：将所有块结果拼接后，用 LLM 要求返回 JSON，并包含多种解析 fallback（直接 JSON、从 code blocks 抽取 JSON、返回错误占位）。
  - `go()` / `save_results()`：流程入口与持久化。
- 典型调用：`biomni/biorxiv_scripts/extract_biorxiv_tasks.py`。
- 建议：在 prompt 中强制 `JSON-only` 输出并添加重试逻辑；对合并结果做去重、置信度或来源片段标注；考虑并行化 chunk 处理并缓存重复结果。

---

**2. `function_generator.py` — FunctionGenerator**

- 作用：基于任务描述调用 LLM 生成 Python 脚本（期望 LLM 将代码放在 fenced code block 中），并生成建议的脚本文件名。
- 关键方法：
  - `configure()`：定义 `system_prompt`（仅输出 Python 代码、包含 main guard、尽量少注释等）。
  - `_generate_code()`：调用 LLM 生成代码文本。
  - `_generate_script_filename()`：从任务描述生成安全的 `.py` 文件名。
  - `_extract_code_block()`：用正则提取第一个 ``` 代码块。
  - `go()`：主入口，返回 `(script_filename, code_str)`。
- 典型调用：`biomni/biorxiv_scripts/generate_function.py`。
- 建议：对提取出的代码做 `ast.parse()` 验证并在失败时请求 LLM 修复或重试；增加裸代码回退、保存到磁盘的可选参数、以及在 prompt 中要求列出外部依赖及 `pip install` 命令。

---

**3. `qa_llm.py` — 轻量 Q&A LLM 封装**

- 作用：提供最小封装的问答型 LLM 使用接口，支持可选的 step-by-step prompt modifier（`lab_bench_reproduce` 模式）以及 `result_formatting()`（结合 `with_structured_output` 校验/格式化输出）。
- 关键方法：
  - `go(input)`：调用 `self.llm.invoke()` 并记录对话日志。
  - `result_formatting(output_class, task_intention)`：使用 `ChatPromptTemplate` + LLM 的 structured output 能力做结果校验与解析。
- 典型调用：主要为示例/文档与轻量 QA 场景使用，仓库中以 docs/DETAILS.md 等处列出；无核心 agent (A1/react) 的直接依赖。
- 建议：若用于自动评估/测验，扩展多轮管理、置信度输出与失败重试策略；可按需将 `prompt_modifier` 参数化。

---

**总体说明（一句话）**

这三者均为辅助/独立脚本所用的 agent 工具：`env_collection` 与 `function_generator` 面向论文处理与脚本生成流程（在 `biorxiv_scripts` 中被脚本调用）；`qa_llm` 为轻量问答封装。它们并不被核心 agent `A1` 或 `react` 直接依赖，修改需谨慎评估外部脚本影响。

如果需要，我可以：
- 把这份文档添加到 `README` 的“工具说明”处；
- 或为 `function_generator` 增加 `ast` 验证并在生成失败时自动重试（我可以实现一个小补丁）。
