<h1 align="center">Strata</h1>

[English](README.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · **Deutsch** · [Français](README.fr.md) · [Español](README.es.md) · [Português](README.pt-BR.md)

<p align="center"><b>在你自己的游戏 PC 上运行 1250 亿参数的 AI 模型</b><br>
NVIDIA 或 AMD 显卡（12 GB 起）· Windows 或 Linux · 免费且开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="模型在 Strata 中编写的一个体素宝塔花园，正在浏览器中运行"></a><br>
<sub>一个由单条提示词生成的体素宝塔花园，在 RTX 5070 上使用 Strata 运行（IQ3_S，128K 上下文）·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

Strata 让 **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)** 在一台普通 PC 上运行。
这是一个大型、聪明的 AI 模型，通常需要服务器才能运行。它可以聊天、编写代码、理解图像，并与你的应用和
编程 Agent 协同工作。一切都不会离开你的 PC。

## 它有多快？

我们在两台普通的游戏 PC 上进行了测量。一个 Token 大约相当于 ¾ 个单词。

- **生成回答：** 短对话中回答出现的速度。每秒 60 个 Token 比你阅读的速度还快。
- **读取你的提示词：** 它接收你发送内容的速度（这里指一份 32K Token 的文档、代码或聊天记录）。

<table>
<tr><th>NVIDIA：RTX 5070（12 GB），Ryzen 5 7600，64 GB RAM</th><th>AMD：RX 9070 XT（16 GB），Ryzen 9 3900X，47 GB RAM</th></tr>
<tr><td>

| 大小 | 生成回答 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 94 Tokens/s | 2,650 Tokens/s |
| **IQ2_XS** | 79 Tokens/s | 2,090 Tokens/s |
| **IQ3_XXS** | 62 Tokens/s | 1,750 Tokens/s |
| **IQ3_S** | 53 Tokens/s | 1,620 Tokens/s |
| **Coder** | 55 Tokens/s | 2,180 Tokens/s |

</td><td>

| 大小 | 生成回答 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 60 Tokens/s | 1,160 Tokens/s |
| **IQ2_XS** | 52 Tokens/s | 1,110 Tokens/s |
| **Coder** | 44 Tokens/s | 1,420 Tokens/s |

</td></tr>
</table>

NVIDIA：Q2_0 使用引擎 0.1.36，其他行使用 0.1.26（4K 回答，32K 提示词）。完整表格见
[DETAILS.md](docs/DETAILS.md#speed-measured)。显存更大的显卡更快：RTX 3090（24 GB）预计每秒可生成约
100-140 个 Token。长对话和其他显卡：
[每个模型的速度](docs/MODELS.md#how-fast-is-each-size)、[社区测试结果](docs/COMMUNITY_BENCHMARKS.md)。

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata 是免费的。如果它在你的 PC 上运行良好，一杯咖啡能让这项工作继续下去。</sub></p>

## 你需要什么

| | |
| --- | --- |
| **显卡** | **NVIDIA** GeForce RTX 20、30、40 或 50 系列，或 **AMD** Radeon RX 7900 XT / XTX、RX 7800 XT / 7700 XT、RX 9060 XT、RX 9070 / 9070 XT、Radeon AI PRO R9700 或 RX 6800 / 6900 系列。需要 **12 GB 显存或更多**。 |
| **内存** | 32 GB 或更多。你的内存决定[哪个模型](#welches-modell-soll-ich-nehmen)合适。64 GB 可以运行所有大小。 |
| **硬盘** | 约 80 GB 可用空间。尽可能使用 SSD：首次启动会快得多。 |
| **系统** | Windows 10 / 11 或 Linux，以及最新的 NVIDIA 或 AMD 显卡驱动。 |

其他一切由安装程序处理。两张或三张显卡可以共享模型（[多 GPU](docs/MULTI_GPU.md)）。

实验性功能，由社区成员在自己的机器上编写和测试：

- **较旧的显卡**（Tesla P40 / V100、GTX 10、Radeon VII / MI50、RX 6700 XT、RX 5500 XT）：[较旧的 GPU](docs/OLDER_GPUS.md)。
- **Intel Arc**，在 Linux 下从源码构建：[Intel Arc](docs/INTEL_ARC.md)。
- **AMD Ryzen AI Max（Strix Halo）**，在 Linux 下从源码构建：[Strix Halo](docs/STRIX_HALO.md)。
- **不支持 AVX2 的较旧处理器**：可以运行，但很慢。[较旧的 CPU](docs/INSTALL.md#older-cpus-experimental)。

完整列表：[docs/INSTALL.md](docs/INSTALL.md#what-you-need)。

## 安装

### 让你的 AI 来设置

你使用 AI 编程助手（Claude Code、Cursor、Codex、GitHub Copilot……）吗？在那里粘贴以下文本：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它会检查你的显卡、内存和硬盘，并选择合适的模型。然后它会安装并启动它，并告诉你如何连接你的应用。
AI 工具也可以通过其 [MCP 服务器](docs/MCP_SERVER.md)安装、启动和停止 Strata。

### 或者自己动手

[下载 Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) 并解压（或使用
`git clone`）。**Windows：** 双击 **`START-HERE.bat`**。**Linux：** 在 Strata 文件夹中运行 **`./setup.sh`**。

NVIDIA 和 AMD 的步骤相同。安装程序会检测你的显卡并为其设置合适的引擎。它会问你几个问题：

- 哪个模型和哪个大小，
- 多少上下文（模型能记住多少文本），
- 是否应该读取图像。

每次按 Enter 采用推荐的答案。然后它会下载模型（约 70 GB）并启动它。如果下载中断，只需重新启动：
它会从上次中断的地方继续。你的浏览器会在 `http://127.0.0.1:8080` 打开 Strata 应用。

> **在模型启动期间，你的 PC 可能会变慢或无响应 1-3 分钟**（首次时间最长）。Strata 会将 35-55 GB 加载到
> 你的内存中，并为显卡预留其中一部分。这是正常的。请等待，不要关闭窗口。窗口中会显示 Strata 正在做什么。

**下次**你再次启动 `START-HERE.bat`（或 `./setup.sh`）。它会立即启动，不会重复下载任何内容。关闭窗口即可
停止模型。`UPDATE.bat`（`./update.sh`）会更新 Strata 而不启动它。更新、Docker、多张显卡、文件存放位置
以及所有选项：[docs/INSTALL.md](docs/INSTALL.md)。

## 我应该选择哪个模型？

安装程序会根据你的内存推荐一个。同一个模型有多种大小，压缩程度各不相同。较小的尺寸更快。较大的尺寸
稍微更聪明。

| 你的内存 | 选择 | 原因 |
| --- | --- | --- |
| **32 GB** | **Coder** | 适合 32 GB，专为代码打造（使用 24 GB 显卡也可以运行 Q2_0 和 IQ2_XS） |
| **48 GB** | **IQ2_XS**（或 Q2_0，最快的） | 更大的尺寸放不下 |
| **64 GB** | **IQ2_XS**（推荐）或 IQ3_XXS / IQ3_S | 所有尺寸都合适；IQ3_S 最好但也最慢 |
| **96 GB 或更多** | **IQ3_S** 或 Unsloth 的 UD-IQ4_XS（约 4 位） | 有空间容纳最大的尺寸，即使其他一切都在运行 |

- **[Coder](docs/MODELS.md#coder)：** 一个用于代码的版本，移除了一半的专家。它达到了完整模型
  SWE-bench-Verified 分数的 91%（由其作者测量），并且适合 32 GB 内存。在代码之外它较弱，包括中文和其他
  CJK 文本（#438）。为此请使用 Q2_0、IQ2_XS 或 IQ3_S，它们保留了所有专家。
- **[Swift 1.5](docs/MODELS.md#swift-15)：** 一个微调版本，在回答前思考时间短得多。你能更早得到答案，
  质量大致相同。
- **[Unsloth UD-IQ4_XS](docs/MODELS.md#unsloth-ud-iq4_xs)：** Unsloth 的约 4 位版本，质量介于 IQ3_S 和
  UD-Q4_K_XL 之间。94 GB 下载量。如果内存少于约 80 GB，Strata 会在回答期间从 SSD 读取其中一部分，因此
  会较慢（NVMe SSD 有帮助）。
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)**（实验性）：最接近完整模型。
  但 Strata 会在回答期间从 SSD 读取其中大部分内容，因此在 64 GB 的 PC 上每秒只能生成 7-8.5 个 Token。
- **[OrcaRouter 的 Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)：** 需要手动设置。
  它不在安装程序的菜单中。

大小、下载量以及什么适合什么：[docs/MODELS.md](docs/MODELS.md)。要稍后添加另一个模型，请运行
`SETUP.bat`（Linux：`./setup.sh --setup`）。

## 如何使用

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="Strata 应用的 Monitor 标签页旁边是一个编程 Agent"><br>
<sub>Strata 应用的 <b>Monitor</b>（左侧），同时一个编程 Agent 正在编写视频中的宝塔花园（右侧）</sub></p>

- **在浏览器中：** 打开 `http://127.0.0.1:8080`。那里有 **Chat**、用于模型和你的 GPU/CPU/RAM 的实时
  **Monitor**，以及包含设置和地址的 **About**。
- **你的应用和编程 Agent：** 添加一个“OpenAI 兼容”提供商，基础 URL 为
  **`http://127.0.0.1:8080/v1`**。任何 API Key 和任何模型名称都可以。
  - 使用 Anthropic API 的应用：`http://127.0.0.1:8080/v1/messages`（Claude Code：
    `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`）。
  - Codex CLI 和其他使用 OpenAI Responses API 的应用：`/v1/responses`
    （[设置](docs/DETAILS.md#the-responses-api-and-codex-cli)）。
- **思考：** 在聊天菜单或你的应用中的“Reasoning Effort”下选择 **关闭、低、中或高**。关闭最快。高最适合
  困难的问题。
- **图像：** 在设置中对“Images?”回答是。然后在聊天中点击 **Picture** 或在你的应用中附加图像。AMD 显卡
  在 Linux 下通过处理器读取图像；在 Windows 下还不行。
- **从手机或其他 PC：** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>`。始终设置一个 Key。
- **一次一个请求：** 默认情况下，Strata 一次回答一个请求，其他请求等待。要同时回答多个请求，请设置
  `"parallel": 2`（[BATCHING.md](docs/BATCHING.md)）。在 12 GB 显卡上，这会使每个回答变慢。
- **长提示词：** Strata 会完整读取聊天的第一条消息，每 30,000 个 Token 约需 1 分钟。后续消息在几秒内
  启动。

更多：[你的聊天存储在哪里](docs/INSTALL.md#where-things-are-stored)、[API](docs/DETAILS.md#using-it)。

## 出问题了？

- **我的 PC 在 Strata 首次启动时冻结了。** 这在模型加载期间是正常的。请等待，不要关闭窗口。10 分钟后
  仍然冻结？重启 PC，关闭其他程序再试一次，或选择较小的尺寸。
- **下载或安装时停止了。** 再次启动 `START-HERE.bat`（或 `./setup.sh`）。它会从上次中断的地方继续。
- **它非常慢，硬盘灯不断闪烁，或者报告“the engine stopped unexpectedly”。** 你的 PC 没有足够的可用
  内存。关闭其他程序（浏览器占用很多）或选择较小的尺寸（Q2_0 或 IQ2_XS）。
- **它报告端口 8080 已被占用。** Strata 已经在运行。寻找它的窗口。

更多问题及其解决方案：[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。还是无法解决？
打开一个 [Issue](https://github.com/Niko1221/Strata/issues) 并附上 Strata 文件夹中的
`strata-<model>.log`。你发现了安全问题？请保密报告：[SECURITY.md](SECURITY.md)。

## 它是如何工作的？

像这样的模型通常运行在拥有数百 GB 显存的服务器上。你的显卡只有 12-24 GB。Strata 仍然能容纳这个模型，
方法是**将工作分配到你整台 PC 上**。想象一个厨房：你经常需要的东西留在台面上，其余的放在储藏室里。

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="模型的 24,576 个专家：最常用的在显卡上，全部在内存中，SSD 上有一个查找表"></p>

- **模型是一个由 24,576 个小专家组成的团队。** 每个词只需要其中 10 个。
- **你的显卡**保存最常用的几千个专家。**你的内存**保存所有专家，**你的处理器**同时计算其余的。
  **你的 SSD**保存一个大型查找表。

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="一个小助手猜测接下来的词；大模型一次性检查所有词并保留正确的"></p>

- **猜测，然后检查：** 一个小助手猜测接下来的几个词。大模型一次性检查它们。你得到相同的答案，只是快了
  1.6-1.8 倍。
- **长文本以大批次读取**（一次最多 8,192 个 Token），每秒超过 1,000 个 Token。

更长的解释：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md)。每个部分及其数字：
[详细信息](docs/DETAILS.md#how-it-works)和[论文](docs/paper/Strata-Paper.pdf)。

## 致谢和许可证

该模型是来自 Qwen 团队的 [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)。压缩由
[ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)、UkisAI（Swift 1.5）
和 Unsloth 完成。Strata 使用了 [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) 的部分内容。
所有致谢：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits)。Strata 是在 [MIT 许可证](LICENSE)下的
开源软件。一些部分和所有模型有自己的许可证（[哪些](docs/HOW_IT_WORKS.md#license)）。

## 支持 Strata

Strata 是免费且开源的。如果它对你有帮助，你可以支持开发：

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>