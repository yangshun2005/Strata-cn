# AGENTS.md

Strata 在普通 PC 上运行 Qwen3.8-Flash-Next 混合专家模型（及其 Coder、Swift 1.5 和 Unsloth 变体）：
一块 NVIDIA 或 AMD 显卡加上系统内存，运行于 Windows 或 Linux。它包含一个 C++/CUDA/HIP 引擎
（`src/`、`include/`），一个带有 OpenAI 和 Anthropic 兼容 API 及 Web 应用的 Python 服务器（`serve/`），以及一个
一键安装程序（`setup.py`，由 `START-HERE.bat` / `setup.sh` 启动）。

## 为用户安装 Strata

遵循 **[docs/AI_SETUP.md](docs/AI_SETUP.md)**：检查 PC、根据内存选择模型、以非交互方式运行安装程序、
启动并验证服务器，然后连接用户的应用。在没有 `--api-key` 的情况下，绝不要将服务器暴露到 `127.0.0.1` 之外。
作为 shell 命令的替代方案，Strata 的 MCP 服务器（[docs/MCP_SERVER.md](docs/MCP_SERVER.md)）
以工具的形式提供相同的步骤。

## 参与代码开发

- 引擎的工作原理、每一个实测数据、API 及所有设置：[docs/DETAILS.md](docs/DETAILS.md) 以及
  [论文](docs/paper/Strata-Paper.pdf)。
- AMD（HIP）构建与验证：[docs/AMD_HIP.md](docs/AMD_HIP.md)；多 GPU：[docs/MULTI_GPU.md](docs/MULTI_GPU.md)。
- 安装程序自身的测试无需 GPU 或下载即可运行：`python tools/test_setup_<name>.py`（例如
  `tools/test_setup_amd.py`、`tools/test_setup_choices.py`）。
- 保持文档的风格：用词平实、实测数据需注明测量环境、没有测量依据就不要下结论。