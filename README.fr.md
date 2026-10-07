<h1 align="center">Strata</h1>

[English](README.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · [Deutsch](README.de.md) · **Français** · [Español](README.es.md) · [Português](README.pt-BR.md)

<p align="center"><b>在自己的游戏 PC 上运行 1250 亿参数 AI 模型</b><br>
NVIDIA 或 AMD 显卡（12 GB 或以上）· Windows 或 Linux · 免费且开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="由 Strata 模型编写、在浏览器中运行的体素宝塔花园"></a><br>
<sub>一座体素宝塔花园，在 RTX 5070 上使用 Strata（IQ3_S，128K 上下文）通过单次请求生成 ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

Strata 让 **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)** 在普通 PC 上运行。这是一个
能力很强的大型 AI 模型，通常需要服务器才能运行。它可以对话、编写代码、读取图像，并能与你的应用程序和
代码智能体协同工作。一切都不会离开你的 PC。

## 它的速度有多快？

我们在两台普通游戏 PC 上进行了测量。一个 token 大约相当于 ¾ 个单词。

- **生成回复：** 在简短对话中回复显示出来的速度。每秒 60 个 token，比你的阅读速度还快。
- **读取你的提示词：** 它吸收你发送内容的速度（这里指一份文档、代码或 32K token 的对话历史）。

<table>
<tr><th>NVIDIA：RTX 5070（12 GB），Ryzen 5 7600，64 GB 内存</th><th>AMD：RX 9070 XT（16 GB），Ryzen 9 3900X，47 GB 内存</th></tr>
<tr><td>

| 规格 | 生成回复 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 94 tokens/s | 2,650 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s |
| **Coder** | 55 tokens/s | 2,180 tokens/s |

</td><td>

| 规格 | 生成回复 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 52 tokens/s | 1,110 tokens/s |
| **Coder** | 44 tokens/s | 1,420 tokens/s |

</td></tr>
</table>

NVIDIA：Q2_0 使用 0.1.36 引擎，其他行使用 0.1.26（4K 回复，32K 提示词）。完整表格见
[DETAILS.md](docs/DETAILS.md#speed-measured)。显存更大的显卡速度更快：RTX 3090（24 GB）预计每秒可生成约
100-140 个 token。长对话和其他显卡：[各模型速度](docs/MODELS.md#how-fast-is-each-size)、
[社区测试结果](docs/COMMUNITY_BENCHMARKS.md)。

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata 是免费的。如果它在你的 PC 上运行良好，请我喝杯咖啡能帮助这项工作继续下去。</sub></p>

## 你需要什么

| | |
| --- | --- |
| **显卡** | **NVIDIA** GeForce RTX 20、30、40 或 50 系列，或 **AMD** Radeon RX 7900 XT / XTX、RX 7800 XT / 7700 XT、RX 9060 XT、RX 9070 / 9070 XT、Radeon AI PRO R9700 或 RX 6800 / 6900 系列。需要 **12 GB 或以上显存**。 |
| **内存** | 32 GB 或以上。你的内存决定[哪个模型](#quel-modèle-choisir-)能装下。64 GB 可以运行所有规格。 |
| **磁盘** | 约 80 GB 可用空间。尽量使用 SSD：首次启动会快得多。 |
| **系统** | Windows 10 / 11 或 Linux，并安装最新的 NVIDIA 或 AMD 显卡驱动。 |

安装程序会处理其余所有事情。两块或三块显卡可以共享模型（[多 GPU](docs/MULTI_GPU.md)）。

以下为实验性支持，由社区成员在自己的机器上编写和测试：

- **较旧的显卡**（Tesla P40 / V100、GTX 10、Radeon VII / MI50、RX 6700 XT、RX 5500 XT）：[Older GPUs](docs/OLDER_GPUS.md)。
- **Intel Arc**，在 Linux 下从源码编译：[Intel Arc](docs/INTEL_ARC.md)。
- **AMD Ryzen AI Max（Strix Halo）**，在 Linux 下从源码编译：[Strix Halo](docs/STRIX_HALO.md)。
- **不支持 AVX2 的较旧处理器**：可以运行，但很慢。[Older CPUs](docs/INSTALL.md#older-cpus-experimental)。

完整列表：[docs/INSTALL.md](docs/INSTALL.md#what-you-need)。

## 安装

### 让你的 AI 来安装

你在使用 AI 代码助手（Claude Code、Cursor、Codex、GitHub Copilot……）？把下面这段粘贴给它：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它会检查你的显卡、内存和磁盘，并选择合适的模型。然后它会安装、启动，并告诉你如何连接你的应用程序。
AI 工具还可以通过 Strata 的 [MCP 服务器](docs/MCP_SERVER.md)来安装、启动和停止 Strata。

### 或者自己动手

[下载 Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) 并解压（或执行 `git clone`）。
**Windows：** 双击 **`START-HERE.bat`**。**Linux：** 在 Strata 文件夹中运行 **`./setup.sh`**。

NVIDIA 和 AMD 的步骤相同。安装程序会检测你的显卡并为其安装合适的引擎。
它会问你几个问题：

- 选择哪个模型和哪种规格，
- 使用多大的上下文（模型能记住的文本量），
- 是否读取图像。

每次按回车即可保留推荐的选项。然后它会下载模型（约 70 GB）并启动。如果下载中断，重新运行即可：它会从
中断处继续。你的浏览器会打开 Strata 应用，地址为 `http://127.0.0.1:8080`。

> **模型启动期间，你的 PC 可能会变慢或无响应 1-3 分钟**（首次会更久）。
> Strata 会将 35-55 GB 加载到你的内存中，并为显卡预留一部分。这是正常的。请等待，
> 不要关闭窗口。窗口会显示 Strata 正在做什么。

**下次**，重新运行 `START-HERE.bat`（或 `./setup.sh`）。它会立即启动，不会重复下载任何内容。
关闭其窗口即可停止模型。`UPDATE.bat`（`./update.sh`）会更新 Strata 而不启动它。
更新、Docker、多显卡、文件位置及所有选项：[docs/INSTALL.md](docs/INSTALL.md)。

## 选择哪个模型？

安装程序会根据你的内存推荐一个。同一个模型有多种规格，压缩程度各不相同。规格越小速度越快。规格越大
稍微更聪明。

| 你的内存 | 选择 | 原因 |
| --- | --- | --- |
| **32 GB** | **Coder** | 它能装进 32 GB，且专为代码设计（使用 24 GB 显卡时，Q2_0 和 IQ2_XS 也能运行） |
| **48 GB** | **IQ2_XS**（或 Q2_0，最快） | 更大的规格装不下 |
| **64 GB** | **IQ2_XS**（推荐），或 IQ3_XXS / IQ3_S | 所有规格都能装下；IQ3_S 最好也最慢 |
| **96 GB 或以上** | **IQ3_S**，或 Unsloth 的 UD-IQ4_XS（约 4 位） | 有空间容纳最大的规格，同时保持一切可用 |

- **[Coder](docs/MODELS.md#coder)：** 面向代码的版本，专家数量减少了一半。它达到了完整模型 SWE-bench Verified
  分数的 91%（由其作者测量），并能装进 32 GB 内存。它在代码之外的领域表现较差，包括中文和其他 CJK 文本
  （#438）。对于这些情况，请选择 Q2_0、IQ2_XS 或 IQ3_S，它们保留了所有专家。
- **[Swift 1.5](docs/MODELS.md#swift-15)：** 一个在回答前思考时间少得多的微调版本。你能更早得到答案，
  质量大致相同。
- **[Unsloth UD-IQ4_XS](docs/MODELS.md#unsloth-ud-iq4_xs)**：Unsloth 的约 4 位版本，质量介于 IQ3_S 和
  UD-Q4_K_XL 之间。下载量为 94 GB。如果内存少于约 80 GB，Strata 会在回答时从 SSD 读取一部分，因此速度
  会更慢（NVMe SSD 有帮助）。
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)**（实验性）：最接近完整模型。
  但 Strata 会在回答时从 SSD 读取其中大部分内容，因此在 64 GB 的 PC 上每秒只能生成
  7-8.5 个 token。
- **[OrcaRouter's Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)：** 需要手动安装。
  它不在安装程序的菜单中。

规格、下载量及各自能装在哪里：[docs/MODELS.md](docs/MODELS.md)。以后要添加其他模型，
运行 `SETUP.bat`（Linux：`./setup.sh --setup`）。

## 使用

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="Strata 应用的 Monitor 标签页与代码智能体并排显示"><br>
<sub>Strata 应用的 <b>Monitor</b>（左侧）与正在编写视频中宝塔花园的代码智能体（右侧）同时运行</sub></p>

- **在浏览器中：** 打开 `http://127.0.0.1:8080`。你会看到 **Chat**、实时显示模型和 GPU/CPU/内存的
  **Monitor**，以及包含设置和地址的 **About**。
- **你的应用程序和代码智能体：** 添加一个「OpenAI 兼容」提供商，基础 URL 为
  **`http://127.0.0.1:8080/v1`**。任何 API 密钥和任何模型名称都可以使用。
  - 使用 Anthropic API 的应用程序：`http://127.0.0.1:8080/v1/messages`（Claude Code：
    `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`）。
  - Codex CLI 及其他使用 OpenAI Responses API 的应用程序：`/v1/responses`
    （[配置](docs/DETAILS.md#the-responses-api-and-codex-cli)）。
- **思考：** 在聊天菜单或应用程序的「reasoning effort」中选择 **off、low、medium 或 high**。Off 最快。
  High 最适合困难问题。
- **图像：** 安装时对「Images?」回答 yes。然后在聊天中点击 **Picture**，或在应用程序中附加图像。
  AMD 显卡在 Linux 下通过处理器读取图像；在 Windows 下尚不支持。
- **从手机或其他 PC 访问：** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>`。务必设置
  密钥。
- **一次一个请求：** 默认情况下，Strata 一次回答一个请求，其他请求会等待。要同时回答
  多个请求，请设置 `"parallel": 2`（[BATCHING.md](docs/BATCHING.md)）。在 12 GB 显卡上，每个
  回复都会变得更慢。
- **长提示词：** Strata 会完整读取对话的第一条消息，30,000 个 token 大约需要 1 分钟。
  后续消息会在几秒内启动。

更多信息：[你的对话存储在哪里](docs/INSTALL.md#where-things-are-stored)、[API](docs/DETAILS.md#using-it)。

## 遇到问题？

- **Strata 首次启动时我的 PC 卡住了。** 在模型加载期间这是正常的。请等待，不要
  关闭窗口。10 分钟后仍然卡住？重启 PC，关闭其他程序后
  重试，或选择更小的规格。
- **它在下载或安装过程中停止了。** 重新运行 `START-HERE.bat`（或 `./setup.sh`）。它会
  从中断处继续。
- **非常慢，磁盘指示灯不停闪烁，或显示「the engine stopped unexpectedly」。**
  你的 PC 没有足够的可用内存。关闭其他程序（浏览器会占用大量内存），或
  选择更小的规格（Q2_0 或 IQ2_XS）。
- **它说端口 8080 已被占用。** Strata 已经在运行。找到它的窗口。

其他问题及其解决方案：[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。仍然卡住？提交一个
[issue](https://github.com/Niko1221/Strata/issues)，并附上 Strata 文件夹中的 `strata-<model>.log`。你
发现了安全问题？请私下报告：[SECURITY.md](SECURITY.md)。

## 它是如何工作的？

像这样的模型通常运行在拥有数百 GB 显存的服务器上。你的显卡只有 12-24 GB。Strata 通过**将工作分散到整台
PC** 上来装下模型。想象一个厨房：你经常用的东西留在台面上，其余的放在
橱柜里等待。

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="模型的 24,576 个专家：最常用的在显卡上，全部在内存中，SSD 上有一张查找表"></p>

- **模型是一支由 24,576 个小专家组成的团队。** 每个词只需要其中 10 个。
- **你的显卡**保留最常用的几千个专家。**你的内存**包含全部专家，
  而**你的处理器**同时处理其余的。**你的 SSD**包含一张大的查找
  表。

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="一个小助手猜测接下来的词；大模型一次性验证它们并保留正确的"></p>

- **先猜，再验证：** 一个小助手猜测接下来的几个词。大模型一次性验证
  它们。你得到相同的答案，但提前 1.6-1.8 倍。
- **长文本按大块读取**（一次最多 8,192 个 token），速度超过每秒 1,000 个 token。

更详细的解释：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md)。每个部分及其数据：
[详情](docs/DETAILS.md#how-it-works)和[论文](docs/paper/Strata-Paper.pdf)。

## 致谢与许可

模型是 [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)，来自 Qwen 团队。它由
[ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)、UkisAI（Swift 1.5）
和 Unsloth 压缩。Strata 使用了 [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) 的部分内容。所有
致谢：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits)。Strata 是开源项目，采用 [MIT 许可证](LICENSE)。
部分组件和每个模型都有自己的许可证（[哪些](docs/HOW_IT_WORKS.md#license)）。

## 支持 Strata

Strata 是免费且开源的。如果它对你有用，你可以支持它的开发：

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>