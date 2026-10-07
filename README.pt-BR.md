<h1 align="center">Strata</h1>

[English](README.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · [Deutsch](README.de.md) · [Français](README.fr.md) · [Español](README.es.md) · **Português**

<p align="center"><b>在你自己的游戏 PC 上运行 1250 亿参数 AI 模型</b><br>
NVIDIA 或 AMD 显卡（12 GB 或以上）· Windows 或 Linux · 免费且开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="Strata 模型编写的体素宝塔花园，正在浏览器中运行"></a><br>
<sub>一个体素宝塔花园，在 RTX 5070 上用 Strata（IQ3_S，128K 上下文）通过单个提示词生成 ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

Strata 在普通 PC 上运行 **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)**。这是一个
通常需要服务器才能运行的大型智能 AI 模型。它能对话、编写代码、理解图像，并可与你的应用和编程智能体
配合使用。一切都不会离开你的 PC。

## 速度有多快？

我们在两台普通游戏 PC 上进行了测量。一个 token 约等于 ¾ 个单词。

- **生成回复：** 短对话中回复出现的速度。每秒 60 个 token 比你阅读的速度还快。
- **读取提示词：** 它吸收你发送内容的速度（此处为 32K token 的文档、代码或对话历史）。

<table>
<tr><th>NVIDIA：RTX 5070（12 GB），Ryzen 5 7600，64 GB 内存</th><th>AMD：RX 9070 XT（16 GB），Ryzen 9 3900X，47 GB 内存</th></tr>
<tr><td>

| 大小 | 生成回复 | 读取提示词 |
| --- | ---: | ---: |
| **Q2_0** | 94 tokens/s | 2,650 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s |
| **Coder** | 55 tokens/s | 2,180 tokens/s |

</td><td>

| 大小 | 生成回复 | 读取提示词 |
| --- | ---: | ---: |
| **Q2_0** | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 52 tokens/s | 1,110 tokens/s |
| **Coder** | 44 tokens/s | 1,420 tokens/s |

</td></tr>
</table>

NVIDIA：Q2_0 使用 engine 0.1.36，其他行使用 0.1.26（4K 回复，32K 提示词）。完整表格见
[DETAILS.md](docs/DETAILS.md#speed-measured)。显存更大的显卡更快：RTX 3090（24 GB）预计每秒可生成约
100-140 个 token。长对话和其他显卡：
[各模型速度](docs/MODELS.md#how-fast-is-each-size)、[社区测试结果](docs/COMMUNITY_BENCHMARKS.md)。

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata 是免费的。如果它在你的 PC 上运行良好，一杯咖啡能帮助维持它的开发。</sub></p>

## 你需要什么

| | |
| --- | --- |
| **显卡** | **NVIDIA** GeForce RTX 20、30、40 或 50 系列，或 **AMD** Radeon RX 7900 XT / XTX、RX 7800 XT / 7700 XT、RX 9060 XT、RX 9070 / 9070 XT、Radeon AI PRO R9700 或 RX 6800 / 6900 系列。需要 **12 GB 或以上显存**。 |
| **内存** | 32 GB 或以上。你的内存决定[哪个模型](#qual-modelo-devo-escolher)能装下。64 GB 可运行任何大小。 |
| **磁盘** | 约 80 GB 可用空间。尽量使用 SSD：首次启动会快得多。 |
| **系统** | Windows 10 / 11 或 Linux，以及最新的 NVIDIA 或 AMD 显卡驱动。 |

安装程序会处理其余一切。两块或三块显卡可以分担模型（[多 GPU](docs/MULTI_GPU.md)）。

实验性支持，由社区成员在自己的机器上编写和测试：

- **较旧的显卡**（Tesla P40 / V100、GTX 10、Radeon VII / MI50、RX 6700 XT、RX 5500 XT）：[较旧 GPU](docs/OLDER_GPUS.md)。
- **Intel Arc**，在 Linux 上从源代码编译：[Intel Arc](docs/INTEL_ARC.md)。
- **AMD Ryzen AI Max（Strix Halo）**，在 Linux 上从源代码编译：[Strix Halo](docs/STRIX_HALO.md)。
- **不支持 AVX2 的较旧处理器**：可以运行，但很慢。[较旧 CPU](docs/INSTALL.md#older-cpus-experimental)。

完整列表：[docs/INSTALL.md](docs/INSTALL.md#what-you-need)。

## 安装

### 让你的 AI 来安装

你使用带 AI 的编程助手（Claude Code、Cursor、Codex、GitHub Copilot……）？把这段粘贴给它：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它会检查你的显卡、内存和磁盘，并选择能装下的模型。然后安装、启动并说明如何连接你的应用。AI 工具
还可以通过它的 [MCP 服务器](docs/MCP_SERVER.md)安装、启动和停止 Strata。

### 或者自己动手

[下载 Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) 并解压（或使用 `git clone`）。
**Windows：** 双击 **`START-HERE.bat`**。**Linux：** 在 Strata 文件夹中运行 **`./setup.sh`**。

NVIDIA 和 AMD 的步骤相同。安装程序会找到你的显卡并为其配置正确的 engine。它会问几个问题：

- 哪个模型和哪个大小，
- 多少上下文（模型在内存中保留多少文本），
- 是否要理解图像。

每个问题按 Enter 即可使用推荐答案。然后它会下载模型（约 70 GB）并启动。如果下载中断，再次运行即可：
它会从断点继续。浏览器会打开 Strata 应用，地址为 `http://127.0.0.1:8080`。

> **模型启动时，你的 PC 可能会变慢或无响应 1-3 分钟**（首次更久）。
> Strata 会将 35-55 GB 加载到你的内存中，并为显卡预留一部分。这是正常的。请等待，不要
> 关闭窗口。窗口会显示 Strata 正在做什么。

**下次**，再次运行 `START-HERE.bat`（或 `./setup.sh`）。它会立即启动，不会重复下载任何内容。关闭窗口
即可停止模型。`UPDATE.bat`（`./update.sh`）会更新 Strata 而不启动它。
更新、Docker、多显卡、文件存放位置及所有选项：[docs/INSTALL.md](docs/INSTALL.md)。

## 我应该选择哪个模型？

安装程序会根据你的内存推荐一个。同一个模型有多种大小，压缩程度不同。较小的尺寸更快。较大的尺寸
稍微更智能。

| 你的内存 | 选择 | 原因 |
| --- | --- | --- |
| **32 GB** | **Coder** | 能装进 32 GB，专为代码设计（搭配 24 GB 显卡，Q2_0 和 IQ2_XS 也能运行） |
| **48 GB** | **IQ2_XS**（或 Q2_0，最快） | 更大的尺寸装不下 |
| **64 GB** | **IQ2_XS**（推荐），或 IQ3_XXS / IQ3_S | 所有尺寸都能装下；IQ3_S 最好也最慢 |
| **96 GB 或以上** | **IQ3_S**，或 Unsloth 的 UD-IQ4_XS（约 4-bit） | 有空间容纳最大尺寸，其余全部打开 |

- **[Coder](docs/MODELS.md#coder)：** 一个移除了半数专家的编程版本。它在 SWE-bench Verified 上达到完整
  模型 91% 的得分（由作者测量），并能装进 32 GB 内存。它在代码之外较弱，包括中文和其他 CJK 文本
  (#438)。为此，请选择 Q2_0、IQ2_XS 或 IQ3_S，它们保留所有专家。
- **[Swift 1.5](docs/MODELS.md#swift-15)：** 一个在回答前思考时间少得多的微调版本。你能更早得到回复，
  质量几乎相同。
- **[Unsloth UD-IQ4_XS](docs/MODELS.md#unsloth-ud-iq4_xs)：** Unsloth 的约 4-bit 版本，质量介于 IQ3_S 和
  UD-Q4_K_XL 之间。下载量 94 GB。内存少于约 80 GB 时，Strata 会在回复时从 SSD 读取其中一部分，
  因此会变慢（NVMe SSD 有帮助）。
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)**（实验性）：最接近完整模型。
  但 Strata 在回复时会从 SSD 读取其大部分内容，因此在 64 GB 的 PC 上每秒只能生成 7-8.5 个 token。
- **[OrcaRouter 的 Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)：** 需手动配置。它
  不在安装程序菜单中。

尺寸、下载量及各自能装在哪里：[docs/MODELS.md](docs/MODELS.md)。之后要添加其他模型，请运行
`SETUP.bat`（Linux：`./setup.sh --setup`）。

## 如何使用

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="Strata 应用的 Monitor 标签页旁边是一个编程智能体"><br>
<sub>Strata 应用的 <b>Monitor</b>（左侧），同时一个编程智能体正在编写视频中的宝塔花园（右侧）</sub></p>

- **在浏览器中：** 打开 `http://127.0.0.1:8080`。那里有 **Chat**、模型和你的 GPU/CPU/内存的实时
  **Monitor**，以及包含配置和地址的 **About**。
- **你的应用和编程智能体：** 添加一个 "OpenAI-compatible" 提供商，基础 URL 为
  **`http://127.0.0.1:8080/v1`**。任何 API 密钥和任何模型名称都可以使用。
  - 使用 Anthropic API 的应用：`http://127.0.0.1:8080/v1/messages`（Claude Code：
    `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`）。
  - Codex CLI 和其他使用 OpenAI Responses API 的应用：`/v1/responses`
    （[配置](docs/DETAILS.md#the-responses-api-and-codex-cli)）。
- **推理：** 在聊天菜单或应用的 "reasoning effort" 中选择 **off、low、medium 或 high**。Off 最快。High
  最适合难题。
- **图像：** 在安装时对 "Images?" 回答是。然后在聊天中点击 **Picture**，或在你的应用中附加图像。
  AMD 显卡在 Linux 上使用处理器理解图像；在 Windows 上尚不支持。
- **从手机或其他 PC 访问：** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>`。务必设置密钥。
- **一次一个请求：** 默认情况下，Strata 一次回复一个请求，其他请求等待。要同时回复
  多个请求，请设置 `"parallel": 2`（[BATCHING.md](docs/BATCHING.md)）。在 12 GB 的显卡上，这会让
  每个回复变慢。
- **长提示词：** Strata 会一次性读取整个对话的第一条消息，每 30,000 个 token 约需 1 分钟。后续消息
  几秒内开始。

更多：[你的对话保存在哪里](docs/INSTALL.md#where-things-are-stored)、[API](docs/DETAILS.md#using-it)。

## 出问题了？

- **Strata 首次启动时我的 PC 卡住了。** 在加载模型时这是正常的。请等待，不要
  关闭窗口。10 分钟后仍然卡住？重启 PC，关闭其他程序再试，或
  选择更小的尺寸。
- **下载或安装过程中停止了。** 再次运行 `START-HERE.bat`（或 `./setup.sh`）。它会从
  断点继续。
- **非常慢，磁盘灯不停闪烁，或出现 "the engine stopped unexpectedly"。** 你的 PC
  没有足够的可用内存。关闭其他程序（浏览器占用很多），或选择更小的尺寸（Q2_0
  或 IQ2_XS）。
- **提示端口 8080 已被占用。** Strata 已经在运行。找到它的窗口。

更多问题及解决方法：[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。仍未解决？提交一个
[issue](https://github.com/Niko1221/Strata/issues)，并附上 Strata 文件夹中的 `strata-<model>.log`。发现
安全问题？请私下报告：[SECURITY.md](SECURITY.md)。

## 它是如何工作的？

这样的模型通常运行在拥有数百 GB 显存的服务器上。你的显卡只有 12-24 GB。Strata 通过**将工作分配到
整台 PC** 上来让模型装下。想象一个厨房：你经常用的东西放在台面上，其余的放在储藏室里等待。

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="模型的 24,576 个专家：最常用的在显卡上，全部在内存中，SSD 上有一张查找表"></p>

- **模型是一个由 24,576 个小专家（"experts"）组成的团队。** 每个词只需要其中 10 个。
- **你的显卡**保存最常用的几千个专家。**你的内存**保存全部专家，**你的处理器**同时处理其余部分。
  **你的 SSD**保存一张大型查找表。

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="一个小助手猜测接下来的词；大模型一次性核对全部并保留正确的"></p>

- **猜测与核对：** 一个小助手猜测接下来的词。大模型一次性核对全部。你得到相同的回复，但提前
  1.6-1.8 倍。
- **长文本以大批次读取**（每次最多 8,192 个 token），速度超过每秒 1,000 个 token。

更长的解释：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md)。每个部分及其数据：
[详细信息](docs/DETAILS.md#how-it-works)和[论文](docs/paper/Strata-Paper.pdf)。

## 致谢与许可

模型是 [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)，由 Qwen 团队开发。它由
[ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)、UkisAI（Swift 1.5）
和 Unsloth 压缩。Strata 使用了 [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) 的部分内容。所有致谢：
[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits)。Strata 是在 [MIT License](LICENSE) 下的开源项目。
部分组件和所有模型有自己的许可（[哪些](docs/HOW_IT_WORKS.md#license)）。

## 支持 Strata

Strata 是免费且开源的。如果它对你有用，你可以支持开发：

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>