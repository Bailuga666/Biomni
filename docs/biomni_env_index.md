# biomni_env 目录说明

本文档简要说明仓库中 `biomni_env/` 目录下的配置、脚本与清单文件的用途、常用安装命令和维护建议，便于在不同机器上重现开发/运行环境。

目录文件一览（简短说明）

- `README.md`：本目录的使用说明，包含各环境的创建命令与安装流程摘要（推荐步骤、资源与注意事项）。
- `environment.yml`：主 Conda 环境配置（`biomni_e1`），包含 Python 3.11、常用 Python 包与若干 langchain/LLM 客户端依赖，适合完整开发与交互式使用。示例创建命令：

```bash
conda env create -f biomni_env/environment.yml
conda activate biomni_e1
```

- `fixed_env.yml`：较精确且更完整的依赖清单（大量 pin 的 bioconda/系统级包），用于尽量可复现的环境安装或离线情况（体积大、安装耗时）。可用作替代或补充 `environment.yml`。创建命令同上，使用文件名替换。

- `bio_env_py310.yml`：为需要 Python 3.10 的工具单独提供的环境（例如部分拷贝数/基因组学工具）。仅当某些工具不兼容 Python >=3.11 时使用。创建命令：

```bash
conda env create -f biomni_env/bio_env_py310.yml
conda activate biomni_py310
```

- `r_packages.yml`：包含 R 运行时与 `r-essentials` 的简易 Conda 配置，便于在 Conda 中安装并使用 R。可与 `install_r_packages.R` 配合使用。

- `cli_tools_config.json`：命令行生物信息学工具的元数据清单（例如 PLINK、IQ-TREE、GCTA、BWA、FastTree、MUSCLE、HOMER 等），包含下载 URL、期望二进制路径与版本检测命令；由安装脚本引用以自动化下载/编译/安装。

- `install_cli_tools.sh`：用于下载、编译和安装上面列出的命令行工具的安装脚本。支持不同平台的下载、源码编译（例如 FastTree、BWA）、以及为 HOMER 等做特殊处理。通常配合 `cli_tools_config.json` 使用：

```bash
bash biomni_env/install_cli_tools.sh
# 或非交互式（示例）
NON_INTERACTIVE=1 bash biomni_env/install_cli_tools.sh
```

- `install_r_packages.R`：用于在 R 中安装 CRAN 与 Bioconductor 包的脚本（如 `DESeq2`, `edgeR`, `clusterProfiler`, `WGCNA` 等），当 Conda 中 R 包缺失或需要单独安装时运行：

```bash
Rscript biomni_env/install_r_packages.R
```

- `new_software_v008.sh`：增量安装脚本，用于随版本发布添加/安装新的 Python 包与工具（通过 `pip` 或系统包管理器）。用于已装基础环境时的快速更新。

- `setup.sh`：总控安装脚本，负责按序安装 Conda 环境、CLI 工具与 R 包（交互式或非交互式模式），适合在干净机器上一次性部署完整环境（耗时并占用较多磁盘）。运行示例：

```bash
bash biomni_env/setup.sh
```

---

使用建议与注意事项

- 磁盘与时间：完整安装（`setup.sh` + `environment.yml` + CLI 工具）可能需要数十 GB 的磁盘空间与数小时到十几小时的时间，请在具备条件的机器上运行。`README.md` 中强调了 Ubuntu 22.04 的验证兼容性。
- 交互与非交互模式：若要在 CI 或自动化脚本中运行，设置环境变量 `NON_INTERACTIVE=1` 可跳过交互提示。CLI 工具安装目录可由 `BIOMNI_TOOLS_DIR` 指定。
- 权限与编译：部分工具需要 `gcc`/`make`/`git` 等编译工具或 `sudo`（取决于目标目录），建议在用户可写目录下安装 CLI 工具（脚本默认 `./biomni_tools`）。
- R 依赖：若 Conda 提供的 R 包不完整，可运行 `install_r_packages.R` 逐项补装 Bioconductor/CRAN 包。
- 精简与调试：`fixed_env.yml` 提供了一个更可复现但更臃肿的替代清单；开发时可先使用 `environment.yml` 快速进入并在需要时添加额外包。

---

维护提示

- 定期更新 `cli_tools_config.json` 中工具的下载链接与版本检测命令，避免因上游发布格式变化导致安装失败。
- 将经常改动的“增量安装”脚本（例如 `new_software_v008.sh`）按版本归档，并在 `README.md` 中记录变更。
- 在自动化部署（例如容器化）场景，优先把环境转换为可重现的 Dockerfile 或 mamba 脚本，避免长时间的在线安装步骤。

---

如果你希望，我可以：

- 把 `biomni_env` 的关键安装命令加入项目根 `README.md` 的 “Quick Start” 小节（并提交补丁）；或
- 为 `install_cli_tools.sh` 添加一个 `--dry-run` 模式来在实际下载前列出将安装的工具清单。你想先做哪一个？
