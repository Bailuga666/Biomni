**智能体创建与准备流程（从启动到接收第一个问题）**

本文档按代码执行顺序，简明说明原版 Biomni 在你创建 `A1` 智能体并准备好工具/资源后，直到接受第一个用户问题前都发生了哪些关键步骤，便于理解与调试。

**主要参考代码位置**
- `biomni/agent/a1.py` — 智能体初始化与 `go()` 主循环。
- `biomni/config.py` — 全局配置与环境变量覆盖逻辑。
- `biomni/llm.py` — 根据 `llm` 字符串选择/实例化模型客户端的逻辑。

以下所有文件名请在仓库中查看对应实现以获取更详细行号。

---

## 1) 进程与环境变量加载（导入阶段）

- 在导入 `biomni.agent` 模块时，`a1.py` 会检查当前工作目录下是否存在 `.env`：
  - 如果存在，调用 `load_dotenv('.env', override=False)` 将 `.env` 中尚未存在于 `os.environ` 的键加入当前进程环境。`override=False` 会阻止覆盖已有的系统环境变量（更安全）。
- 此时 `biomni/config.py` 的 `default_config = BiomniConfig()` 也会被实例化，`BiomniConfig.__post_init__` 会读取以 `BIOMNI_` 前缀的环境变量（例如 `BIOMNI_DATA_PATH`、`BIOMNI_LLM`、`BIOMNI_TIMEOUT_SECONDS` 等），用于覆盖默认配置。

要点：保证你从项目根（含 `.env` 的目录）启动 Python，或把凭据设为系统/用户级环境变量，否则导入时可能读不到期望配置。

## 2) 构造 `A1(...)`（参数合并与显示）

- 调用：`from biomni.agent import A1`，然后 `a = A1(path=..., llm=..., expected_data_lake_files=...)`。
- `A1.__init__` 会对传入参数与 `default_config` 做合并：
  - 构造器优先使用传入参数；未给定的参数回退到 `default_config`。
  - 构造器会把当前配置打印出来（便于确认）。

要点：如果你想用统一全局设置，修改 `default_config`；若只想临时覆盖某次智能体实例，传入构造器参数。

## 3) 数据目录与 Data‑Lake（本地数据准备）

- `A1` 会在 `path` 下创建 `biomni_data`、`data_lake` 与 `benchmark` 等目录结构。
- 如果 `expected_data_lake_files is None`（默认）：
  - 会根据内置的 `data_lake_dict` 检查哪些数据文件缺失，并调用 `check_and_download_s3_files(...)` 从 S3 下载缺失文件（首次可能下载大量数据，README 说明约 11GB）。
- 如果你传 `expected_data_lake_files=[]`，构造器会跳过自动下载，打印提示 "Skipping datalake download (load_datalake=False)"。

要点：若带宽/存储受限，建议传 `expected_data_lake_files=[]` 并按需放置必要数据到 `path/biomni_data/data_lake`。

## 4) 实例化 LLM 客户端（核心）

- 在 `A1.__init__` 中会调用 `self.llm = get_llm(...)` 来获取主推理模型客户端。
- `get_llm()` 的行为：
  - 若 `source` 未显式传入，会尝试从 `LLM_SOURCE` 环境变量读取，或根据 `model` 字符串前缀推断提供商（如以 `claude-` 开头推断为 Anthropic，以 `gpt-` 开头推断为 OpenAI，以 `azure-` 开头推断为 AzureOpenAI 等）。
  - 根据 provider，`get_llm()` 从正确的环境变量读取 API key（例如 `ANTHROPIC_API_KEY`、`OPENAI_API_KEY`、`GEMINI_API_KEY`、`GROQ_API_KEY`）并返回相应的客户端包装。
  - 支持 `Custom`（自托管）模式：需要 `base_url` 与 `api_key`。

要点：你在 `llm` 写的模型标识会影响 `get_llm()` 如何选择 provider，也可显式传 `source` 强制使用某个后端。

## 5) 工具注册、检索器与 Know‑How 加载

- 如果 `use_tool_retriever` 为 True，`A1` 会初始化 `ToolRegistry` 与 `ToolRetriever`，并加载 `module2api`（即工具描述集合），用于后续检索时把相关工具信息注入给模型。
- 初始化 `KnowHowLoader`，加载项目内的 know‑how 文档库；若 `commercial_mode` 为 True，会过滤部分非商业许可内容。

要点：这些资源让 agent 能在推理前检索并把相关工具 / 文档嵌入系统提示，从而生成更可执行、有据可查的响应。

## 6) 系统提示与准备完成

- 在所有资源准备就绪后，`A1` 会构造/维护一个系统提示模板与 `module2api`、data‑lake 描述、know‑how 列表等辅助信息，供后续 `go()` 调用时注入。

## 7) 接受问题之前的状态总结

在你实际调用 `a.go(prompt)` 之前，智能体实例通常已经完成：
- 环境变量加载与配置合并（`default_config` 已生效）；
- 数据目录建立与（可选）datalake 下载/检查；
- LLM 客户端 `self.llm` 已实例化（若缺失凭据会在此处显式报错或提示）；
- ToolRegistry/ToolRetriever、KnowHowLoader 已初始化并加载本地工具/文档索引。

此时 `A1` 已准备好接收第一个问题（`go()`），并在 `go()` 内执行检索、工具选择、发起 LLM 请求与可选执行步骤。

---

## 常见问题（FAQ）

- Q：如果我从其它目录启动 Python，能否加载项目根的 `.env`？
  - A：不会。`.env` 由相对 cwd 决定。要在任意位置启动并仍加载该 `.env`，可在代码中显式调用 `load_dotenv('/绝对/路径/.env')` 或把凭据写入系统环境变量（`setx`/CI secrets）。

- Q：Agent 会自动把上次对话记住吗？
  - A：默认不会。`A1.go()` 每次调用不会自动把以前调用的对话注入为上下文。要实现多轮对话，需要在同一 agent 实例内维护消息历史并在下一次请求时把历史作为 `messages` 传入（或持久化并在新进程启动时恢复）。

- Q：如何确认 agent 使用了哪个模型/凭据？
  - A：在实例化 `A1` 时控制台会打印配置摘要（包括模型名与部分掩码的 API key 信息），此外 `biomni/llm.py` 的推断逻辑可查证具体采用哪个 provider。

---

如果你希望，我可以：
- 把这份文档加入仓库（已添加到 `docs/agent_creation_workflow.md`）；
- 或把其中关键步骤提取到 `README.md` 的“开发者笔记”小节中；
- 或生成一个小脚本演示“在同一 agent 实例中保持 history 并多轮交互”的最小可运行示例。 

请选择下一步。 
