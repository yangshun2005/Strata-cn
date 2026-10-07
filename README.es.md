<h1 align="center">Strata</h1>

[English](README.md) · [简体中文](README.zh-CN.md) · [日本語](README.ja.md) · [Deutsch](README.de.md) · [Français](README.fr.md) · **Español** · [Português](README.pt-BR.md)

<p align="center"><b>在你自己的游戏 PC 上运行一个 1250 亿参数的人工智能模型</b><br>
NVIDIA 或 AMD 显卡（12 GB 或以上）· Windows 或 Linux · 自由且开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="一座由体素构成的宝塔花园，由 Strata 模型编写，在浏览器中运行"></a><br>
<sub>一座由体素构成的宝塔花园，在 RTX 5070 上用 Strata（IQ3_S，128K 上下文）通过单条提示词生成 ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

Strata 在普通 PC 上运行 **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)**。这是一个
通常需要服务器才能运行的大型智能人工智能模型。它可以对话、编写代码、理解图像，
并能与你的应用程序和编程代理配合使用。一切都不会离开你的 PC。

## 速度有多快？

我们在两台普通游戏 PC 上进行了测量。一个 token 大约相当于 ¾ 个单词。

- **生成回复：** 在简短聊天中回复出现的速度。每秒 60 个 token 比你阅读的速度还快。
- **读取你的提示词：** 处理你发送内容的速度（此处为 32K token 的文档、代码或聊天记录）。

<table>
<tr><th>NVIDIA：RTX 5070（12 GB），Ryzen 5 7600，64 GB RAM</th><th>AMD：RX 9070 XT（16 GB），Ryzen 9 3900X，47 GB RAM</th></tr>
<tr><td>

| 大小 | 生成回复 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 94 tokens/s | 2,650 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s |
| **Coder** | 55 tokens/s | 2,180 tokens/s |

</td><td>

| 大小 | 生成回复 | 读取你的提示词 |
| --- | ---: | ---: |
| **Q2_0** | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 52 tokens/s | 1,110 tokens/s |
| **Coder** | 44 tokens/s | 1,420 tokens/s |

</td></tr>
</table>

NVIDIA：Q2_0 使用 0.1.36 引擎，其余各行使用 0.1.26（4K 回复，32K 提示词）。完整表格
见 [DETAILS.md](docs/DETAILS.md#speed-measured)。显存更大的显卡更快：一块
RTX 3090（24 GB）应该能每秒生成约 100-140 个 token。长聊天和其他显卡：
[各模型的速度](docs/MODELS.md#how-fast-is-each-size)、[社区测试结果](docs/COMMUNITY_BENCHMARKS.md)。

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata 是免费的。如果它在你的 PC 上运行良好，一杯咖啡能帮助继续开发它。</sub></p>

## 你需要什么

| | |
| --- | --- |
| **显卡** | **NVIDIA** GeForce RTX 20、30、40 或 50 系列，或 **AMD** Radeon RX 7900 XT / XTX、RX 7800 XT / 7700 XT、RX 9060 XT、RX 9070 / 9070 XT、Radeon AI PRO R9700 或 RX 6800 / 6900 系列。需要 **12 GB 或以上显存**。 |
| **RAM** | 32 GB 或以上。你的 RAM 决定[哪个模型](#哪个模型适合我)能装下。64 GB 可以运行所有大小。 |
| **磁盘** | 约 80 GB 可用空间。如果可以，请使用 SSD：首次启动会快得多。 |
| **系统** | Windows 10 / 11 或 Linux，以及最新的 NVIDIA 或 AMD 显卡驱动。 |

安装程序会准备好其他一切。两块或三块显卡可以分担模型（[多 GPU](docs/MULTI_GPU.md)）。

实验性功能，由社区成员在自己的设备上编写和测试：

- **旧显卡**（Tesla P40 / V100、GTX 10、Radeon VII / MI50、RX 6700 XT、RX 5500 XT）：[旧显卡](docs/OLDER_GPUS.md)。
- **Intel Arc**，在 Linux 上从源代码编译：[Intel Arc](docs/INTEL_ARC.md)。
- **AMD Ryzen AI Max（Strix Halo）**，在 Linux 上从源代码编译：[Strix Halo](docs/STRIX_HALO.md)。
- **不支持 AVX2 的旧处理器**：可以运行，但很慢。[旧 CPU](docs/INSTALL.md#older-cpus-experimental)。

完整列表：[docs/INSTALL.md](docs/INSTALL.md#what-you-need)。

## 安装

### 让你的 AI 来安装

你使用带 AI 的编程助手（Claude Code、Cursor、Codex、GitHub Copilot……）吗？把这段粘贴进去：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

它会检查你的显卡、RAM 和磁盘，并选择能装下的模型。然后它会安装、启动，并向你说明
如何连接你的应用程序。AI 工具也可以通过其
[MCP 服务器](docs/MCP_SERVER.md)安装、启动和停止 Strata。

### 或者自己动手

[下载 Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) 并解压（或使用 `git clone`）。
**Windows：** 双击 **`START-HERE.bat`**。**Linux：** 在 Strata 文件夹中运行 **`./setup.sh`**。

NVIDIA 和 AMD 的步骤相同。安装程序会检测你的显卡并为其准备合适的引擎。
它会问你几个问题：

- 哪个模型和哪个大小，
- 多少上下文（模型能记住多少文本），
- 是否要理解图像。

每次按 Enter 使用推荐答案。之后它会下载模型（约 70 GB）并启动它。如果
下载中断，请重新运行：它会从上次中断的地方继续。你的浏览器会打开 Strata 应用，
地址为 `http://127.0.0.1:8080`。

> **模型启动时，你的 PC 可能会变慢或无响应 1-3 分钟**（首次更久）。
> Strata 会将 35-55 GB 加载到你的 RAM 中，并为显卡预留一部分。这是正常的。请等待，不要关闭
> 窗口。窗口会显示 Strata 正在做什么。

**下次**，再次运行 `START-HERE.bat`（或 `./setup.sh`）。它会立即启动，不会重复下载任何内容。
关闭其窗口即可停止模型。`UPDATE.bat`（`./update.sh`）可以在不启动的情况下更新 Strata。
更新、Docker、多显卡、文件保存位置以及所有选项：[docs/INSTALL.md](docs/INSTALL.md)。

## 哪个模型适合我？

安装程序会根据你的 RAM 推荐一个。同一个模型有多种大小，压缩程度不同。较小的
大小更快。较大的大小稍微更聪明。

| 你的 RAM | 选择 | 原因 |
| --- | --- | --- |
| **32 GB** | **Coder** | 能装进 32 GB，专为编程打造（使用 24 GB 显卡时 Q2_0 和 IQ2_XS 也可以） |
| **48 GB** | **IQ2_XS**（或 Q2_0，最快） | 更大的大小装不下 |
| **64 GB** | **IQ2_XS**（推荐），或 IQ3_XXS / IQ3_S | 所有大小都能装下；IQ3_S 最好也最慢 |
| **96 GB 或以上** | **IQ3_S**，或 Unsloth 的 UD-IQ4_XS（约 4 位） | 有空间容纳最大的大小，同时还能运行其他一切 |

- **[Coder](docs/MODELS.md#coder)：** 一个用于编程的版本，去掉了一半的专家。它达到了
  完整模型 SWE-bench Verified 得分的 91%（由其作者测量），并且能装进 32 GB RAM。它在代码之外较弱，
  中文和其他 CJK 文本也是如此（#438）。对于这些，请选择 Q2_0、IQ2_XS 或 IQ3_S，它们
  保留了所有专家。
- **[Swift 1.5](docs/MODELS.md#swift-15)：** 一个微调版本，在回答前思考的时间少得多。你能
  更快得到回复，质量大致相同。
- **[Unsloth UD-IQ4_XS](docs/MODELS.md#unsloth-ud-iq4_xs)：** Unsloth 的约 4 位版本，质量介于 IQ3_S 和
  UD-Q4_K_XL 之间。下载大小为 94 GB。RAM 少于约 80 GB 时，Strata 会在回复时从 SSD 读取一部分，
  因此在那里会更慢（NVMe SSD 有帮助）。
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)**（实验性）：最接近完整模型。
  但 Strata 在回复时会从 SSD 读取大部分内容，因此在 64 GB 的
  PC 上每秒只能生成 7-8.5 个 token。
- **[OrcaRouter 的 Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)：** 需要手动安装。它不在
  安装程序菜单中。

大小、下载以及各自适合哪里：[docs/MODELS.md](docs/MODELS.md)。要稍后添加另一个模型，请运行
`SETUP.bat`（Linux：`./setup.sh --setup`）。

## 如何使用

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="Strata 应用的 Monitor 标签页与一个编程代理并排显示"><br>
<sub>Strata 应用的 <b>Monitor</b>（左侧），同时一个编程代理正在编写视频中的宝塔花园（右侧）</sub></p>

- **在浏览器中：** 打开 `http://127.0.0.1:8080`。它有 **Chat**、模型和你的
  GPU/CPU/RAM 的实时 **Monitor**，以及包含设置和地址的 **About**。
- **你的应用程序和编程代理：** 添加一个"兼容 OpenAI"的提供商，基础 URL 为
  **`http://127.0.0.1:8080/v1`**。任何 API 密钥和任何模型名称都可以。
  - 使用 Anthropic API 的应用程序：`http://127.0.0.1:8080/v1/messages`（Claude Code：
    `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`）。
  - Codex CLI 和其他使用 OpenAI Responses API 的应用程序：`/v1/responses`
    （[配置](docs/DETAILS.md#the-responses-api-and-codex-cli)）。
- **推理：** 在聊天菜单或你的应用程序的"reasoning effort"中选择 **off、low、medium 或 high**。
  Off 最快。High 最适合困难的问题。
- **图像：** 在安装时对"Images?"回答是。然后在聊天中点击 **Picture**，或在你的应用程序中附加
  图像。AMD 显卡在 Linux 上通过处理器读取图像；在 Windows 上还不行。
- **从你的手机或其他 PC：** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>`。请务必设置密钥。
- **一次一个请求：** 默认情况下 Strata 一次回复一个请求，其他请求等待。要同时回复多个
  请求，请设置 `"parallel": 2`（[BATCHING.md](docs/BATCHING.md)）。在 12 GB 的显卡上，每个回复会更慢。
- **长提示词：** Strata 会完整读取聊天的第一条消息，大约每 30,000 个 token 需要 1 分钟。后续
  消息会在几秒内开始。

更多：[你的聊天保存在哪里](docs/INSTALL.md#where-things-are-stored)、[API](docs/DETAILS.md#using-it)。

## 出问题了？

- **Strata 首次启动时我的 PC 卡住了。** 在加载模型时这是正常的。请等待，不要关闭
  窗口。10 分钟后仍然卡住？重启 PC，关闭其他程序再试一次，或者
  选择更小的大小。
- **下载或安装过程中停止了。** 再次运行 `START-HERE.bat`（或 `./setup.sh`）。它会从
  中断的地方继续。
- **运行很慢，磁盘灯不停闪烁，或者显示"the engine stopped unexpectedly"。** 你的 PC 没有
  足够的可用 RAM。关闭其他程序（浏览器占用很多），或选择更小的大小（Q2_0 或
  IQ2_XS）。
- **显示端口 8080 已被占用。** Strata 已经在运行。找到它的窗口。

更多问题及解决方案：[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。还是卡住了？开一个
[issue](https://github.com/Niko1221/Strata/issues) 并附上 Strata 文件夹中的 `strata-<model>.log`。你
发现了安全问题？请私下报告：[SECURITY.md](SECURITY.md)。

## 它是如何工作的？

像这样的模型通常运行在拥有数百 GB 显存的服务器上。你的显卡
只有 12-24 GB。Strata 通过**将工作分散到你的整个 PC** 来让模型装下。想象一个厨房：你
一直用的东西放在台面上，其余的在储藏室里等着。

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="模型的 24,576 个专家：最常用的在显卡上，全部在 RAM 中，SSD 上有一个查找表"></p>

- **这个模型是一个由 24,576 个小专家组成的团队。** 每个词只需要其中 10 个。
- **你的显卡**保存最常用的几千个专家。**你的 RAM** 拥有全部专家，而**你的
  处理器**同时处理其余的。**你的 SSD** 保存一个大型查找表。

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="一个小助手猜测接下来的词；大模型同时检查所有词并保留正确的"></p>

- **猜测与检查：** 一个小助手猜测接下来的词。大模型同时检查所有词。
  你得到相同的回复，但快 1.6-1.8 倍。
- **长文本以大批次读取**（每次最多 8,192 个 token），每秒超过 1,000 个 token。

详细解释：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md)。每个部分及其数字：
[详细信息](docs/DETAILS.md#how-it-works)和[论文](docs/paper/Strata-Paper.pdf)。

## 致谢与许可

模型是 [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)，来自 Qwen 团队。它由
[ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)、UkisAI（Swift 1.5）
和 Unsloth 压缩。Strata 使用了 [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) 的部分内容。所有致谢：
[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits)。Strata 是开源的，采用 [MIT 许可证](LICENSE)。
某些部分和每个模型都有自己的许可证（[哪些](docs/HOW_IT_WORKS.md#license)）。

## 支持 Strata

Strata 是自由且开源的。如果你觉得它有用，可以支持它的开发：

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>