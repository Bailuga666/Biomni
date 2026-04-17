# DETAILS.md（中文）

🔍 **由 [Detailer](https://detailer.ginylil.com) 提供支持** — 基于代码上下文的仓库分析


---

## 1. 项目概述

### 项目目的与领域

该项目是一个面向生物医学的综合 AI 工具与研究平台，旨在促进生物医学数据分析、知识抽取与基于 AI 的推理工作。项目集成了大语言模型（LLM）、领域特定生物信息学工具以及科学数据处理流水线，以实现：

- 从文献（例如 bioRxiv 论文）自动提取结构化生物医学知识
- 通过自然语言查询并整合多源生物医学数据库与 API
- 执行领域特定的计算生物学与生理学分析
- 用 AI agent 协同完成复杂的生物医学推理与工具调用
- 基准评估与任务评价


### 目标用户与典型用例

- **生物医学研究者与数据科学家**：自动化文献挖掘、数据检索与分析流程。
- **生物信息学工程师**：需要集成多个生物数据库与计算工具的用户。
- **AI 研究者**：研究将 LLM 与自主 agent 应用于复杂生物学问题的方法学者。
- **开发者与集成者**：构建领域特定 AI 管道与工作流的工程师。

典型用例包括：
- 从科学论文中提取可结构化的生物医学任务与实体。
- 使用自然语言检索基因、蛋白、疾病与通路等数据库信息。
- 运行生物学系统的计算模型（如代谢网络、信号传导建模）。
- 执行图像分析与定量病理学工作流。
- 协调多步 AI 推理流程（工具调用、自我修正）。


### 核心业务逻辑与领域模型

- **生物医学领域模型**：基因 ID、蛋白结构、通路、疾病本体、实验方案等。
- **任务抽象**：以基准任务为单位，采用 prompt/response 的评估方式（例如 HLE、lab_bench）。
- **工具元数据 schema**：声明式描述工具与 API，用于动态调用与验证。
- **AI agent 工作流**：基于 ReAct 风格的推理图，集成 LLM、工具调用、检索与自我批评。
- **数据模型**：以 JSON、pandas DataFrame、numpy 数组等格式表示分析结果与中间数据。

---

## 2. 架构与结构

### 高层架构

系统按模块化层次组织：

- **核心库（`biomni/`）**：主应用逻辑，包括：
  - **Agent 框架（`biomni/agent/`）**：使用 LLM 与工作流图实现自主 agent。
  - **任务定义（`biomni/task/`）**：基准任务抽象与具体实现。
  - **工具实现（`biomni/tool/`）**：领域分析函数、API 客户端与计算工作流。
  - **工具元数据（`biomni/tool/tool_description/`）**：描述工具 API 与参数的声明式 schema。
  - **模型组件（`biomni/model/`）**：用于选择相关工具与数据的检索器。
  - **工具/辅助模块（`biomni/utils.py`, `biomni/llm.py`, `biomni/env_desc.py`）**：LLM 实例化、系统命令、环境描述等帮助函数。
  - **版本（`biomni/version.py`）**：版本管理。

- **环境安装（`biomni_env/`）**：用于重现开发/运行环境的配置与脚本（Conda YAML、安装脚本等）。
- **脚本（`biomni/biorxiv_scripts/`）**：用于文献挖掘与任务抽取的数据处理流水线脚本。
- **文档与配置文件**：顶层 README、CONTRIBUTION、pyproject 等。

---

### 完整仓库结构（概要）

（原文列出了项目树；此处省略长树，以 README 与 docs 为参考）

---

### 核心模块及其职责（要点）

#### `biomni/agent/`

- 实现基于 ReAct 的自主 agent：
  - `react.py`：主要 ReAct agent，管理推理、工具调用、检索与自我批评流程。
  - `env_collection.py`：环境与数据检索的辅助工具（例如 PaperTaskExtractor）。
  - `qa_llm.py`：轻量级 QA LLM 封装。
  - `a1.py`：实验性或补充的 agent 实现（标签式 execute/observe/solution 协议）。

- 使用 `langgraph` 做工作流编排，`langchain` 做 LLM 集成。

#### `biomni/task/`

- 基准任务定义与评估接口：
  - `base_task.py`：抽象基类（`get_example()`、`evaluate()`、`output_class()` 等）。
  - 具体任务如 `hle.py`、`lab_bench.py`。

#### `biomni/tool/`

- 包含领域专用分析函数与 API 客户端（按学科分目录），实现可直接调用的分析/处理函数。
- `tool_description/` 下保存声明式工具描述，便于动态生成接口与校验。

#### `biomni/model/retriever.py`

- 实现 `ToolRetriever`：基于 LLM 的资源选择器，用于从工具/数据/库中选出与问题最相关的子集。

#### `biomni/utils.py` 与 `biomni/llm.py`

- `utils.py`：系统命令、文件操作、schema 生成、日志与辅助工具。
- `llm.py`：用于根据配置创建不同提供商的 LLM 实例的工厂函数。

#### `biomni/env_desc.py`

- 包含环境与数据集描述，作为集中式元数据存储。

---

## 3. 环境设置（`biomni_env/`）

- `setup.sh`：主安装脚本（创建 Conda 环境、安装 R 包与 CLI 工具）。
- `install_cli_tools.sh`：自动化下载/编译/安装常见生物信息学命令行工具（如 PLINK、IQ-TREE 等）。
- `r_packages.yml` / `install_r_packages.R`：R 包安装与管理脚本。
- `environment.yml` / `fixed_env.yml`：Conda 环境描述。

---

## 4. 入口点与执行流程

- **Agent 使用**：从 `biomni.agent.react` 导入 `react`，配置工具与检索后调用 `go(prompt)` 启动工作流。
- **任务评估**：使用 `biomni.task` 下的类加载数据、生成 prompts 并评估 LLM 输出。
- **工具调用**：直接调用 `biomni.tool` 模块中的函数，或通过声明式 API（`tool_description`）进行动态构造。
- **资源发现**：使用 `tool_registry.py` 与检索器选择可用工具与数据。

---

## 5. 开发模式与规范要点

- **模块化设计**：按域与职责划分清晰。
- **函数式编程风格**：多数分析函数为独立函数，输入输出明确。
- **声明式元数据**：工具描述与实现分离，便于动态验证与文档生成。
- **抽象基类**：用于统一任务接口。
- **工厂与策略模式**：`llm.py` 使用工厂模式，检索与任务使用策略可替换。

测试：当前仓库未见完整测试套件；示例与笔记本用于手动验证。

错误处理与日志：对外部调用使用 try-except，LLM 交互有日志回调（`PromptLogger`、`NodeLogger`）。

配置管理：通过环境变量（LLM API Key）、YAML/JSON 管理依赖与工具元数据。

---

## 6. 集成与依赖

- 主要依赖：`langchain_core`、`langchain-openai`、`langchain-anthropic`、`numpy`、`pandas`、`scipy`、`transformers` 等。
- 外部 API/数据源：UniProt、GWAS Catalog、Ensembl、ClinVar、dbSNP、GEO、GnomAD、InterPro 等。
- 常见 CLI 工具：PLINK、IQ-TREE、GCTA、samtools、MACS2 等（通过安装脚本安装）。

---

## 7. 使用与运维指导

### 快速开始

1. 环境准备：运行 `biomni_env/setup.sh` 或按需使用 `environment.yml`。

```bash
bash biomni_env/setup.sh
conda activate biomni_e1
```

2. 配置 API Key：设置 `OPENAI_API_KEY` 或 `ANTHROPIC_API_KEY` 等环境变量。

3. 运行 Agent：

```python
from biomni.agent.react import react
agent = react()
agent.configure(plan=True)
log, final = agent.go('你的生物医学任务描述')
```

### 监控与调试

- 使用日志回调（`PromptLogger`、`NodeLogger`）跟踪 LLM 交互。
- 检查工具与函数输出日志以定位问题。

### 性能与扩展

- 模块化设计便于并行化任务与工具调用。
- agent 的工具调用有超时包装以防挂起。

### 安全注意事项

- API Key 使用环境变量管理；切勿硬编码。
- 项目中存在可执行代码运行功能（注意任意代码执行风险），建议在受控/沙箱环境中运行。

---

## 总结

此项目是一个模块化、可扩展的生物医学 AI 平台，集成了 LLM 驱动的 agent、领域专用工具与声明式 API，旨在帮助研究者与开发者构建、评估并扩展 AI 驱动的生物医学工作流。

---

（完）
