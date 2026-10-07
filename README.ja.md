<h1 align="center">Strata</h1>

[English](README.md) · [简体中文](README.zh-CN.md) · **日本語** · [Deutsch](README.de.md) · [Français](README.fr.md) · [Español](README.es.md) · [Português](README.pt-BR.md)

<p align="center"><b>在手上的游戏 PC 上运行 1,250 亿参数的 AI 模型</b><br>
NVIDIA 或 AMD 显卡（12 GB 以上） · Windows 或 Linux · 免费开源</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="Strata 的模型绘制的体素五重塔庭院，正在浏览器中运行"></a><br>
<sub>体素五重塔庭院。通过一次提示词创建，在 RTX 5070 上使用 Strata（IQ3_S，128K 上下文）运行 ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">完整视频（49 秒）</a></sub></p>

Strata 让 **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)** 在普通 PC 上运行。
这是一个又大又聪明的 AI 模型，通常需要服务器。它可以聊天、写代码、读图片。
还能与应用和编码智能体协作。数据不会离开你的 PC。

## 有多快？

在两台很普通的游戏 PC 上进行了测量。1 个 token 大约是英文单词的 ¾。

- **回答生成速度：** 短对话中回复显示的速度。每秒 60 token 就能比阅读更快地显示出来。
- **提示词读取速度：** 读取你发送内容的速度（这里是 32K token 的文档、代码或聊天记录）。

<table>
<tr><th>NVIDIA: RTX 5070 (12 GB), Ryzen 5 7600, 64 GB RAM</th><th>AMD: RX 9070 XT (16 GB), Ryzen 9 3900X, 47 GB RAM</th></tr>
<tr><td>

| 大小 | 回答生成速度 | 提示词读取速度 |
| --- | ---: | ---: |
| **Q2_0** | 94 tokens/s | 2,650 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s |
| **Coder** | 55 tokens/s | 2,180 tokens/s |

</td><td>

| 大小 | 回答生成速度 | 提示词读取速度 |
| --- | ---: | ---: |
| **Q2_0** | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 52 tokens/s | 1,110 tokens/s |
| **Coder** | 44 tokens/s | 1,420 tokens/s |

</td></tr>
</table>

NVIDIA：Q2_0 使用引擎 0.1.36 测量，其他行使用 0.1.26 测量（回答 4K，提示词 32K）。完整表格见
[DETAILS.md](docs/DETAILS.md#speed-measured)。VRAM 越多的显卡越快。RTX 3090（24 GB）
应该能以每秒约 100-140 token 的速度生成。关于长对话和其他显卡：[各模型的速度](docs/MODELS.md#how-fast-is-each-size)、
[社区测量结果](docs/COMMUNITY_BENCHMARKS.md)。

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata 是免费的。如果它在你的 PC 上运行良好，一杯咖啡的支持就是继续开发的动力。</sub></p>

## 需要什么

| | |
| --- | --- |
| **显卡** | **NVIDIA** GeForce RTX 20、30、40、50 系列，或 **AMD** Radeon RX 7900 XT / XTX、RX 7800 XT / 7700 XT、RX 9060 XT、RX 9070 / 9070 XT、Radeon AI PRO R9700、RX 6800 / 6900 系列。需要 **VRAM 12 GB 以上**。 |
| **RAM** | 32 GB 以上。RAM 的容量决定[可用的模型](#どのモデルを選べばいい)。64 GB 就能运行所有大小。 |
| **磁盘** | 约 80 GB 可用空间。尽量使用 SSD。首次启动会快得多。 |
| **OS** | Windows 10 / 11 或 Linux。NVIDIA 或 AMD 的最新显卡驱动。 |

其他所需的一切都由安装程序准备。也可以用 2 张或 3 张显卡分担模型（[多 GPU](docs/MULTI_GPU.md)）。

实验性支持。由社区成员在自己的机器上编写并测试：

- **较旧的显卡**（Tesla P40 / V100、GTX 10、Radeon VII / MI50、RX 6700 XT、RX 5500 XT）：[Older GPUs](docs/OLDER_GPUS.md)。
- **Intel Arc**（在 Linux 上从源码构建）：[Intel Arc](docs/INTEL_ARC.md)。
- **AMD Ryzen AI Max（Strix Halo）**（在 Linux 上从源码构建）：[Strix Halo](docs/STRIX_HALO.md)。
- **没有 AVX2 的旧处理器**：可以运行，但很慢。[Older CPUs](docs/INSTALL.md#older-cpus-experimental)。

完整列表：[docs/INSTALL.md](docs/INSTALL.md#what-you-need)。

## 安装

### 让 AI 帮你安装

你在使用 AI 编码助手（Claude Code、Cursor、Codex、GitHub Copilot 等）吗？粘贴这段：

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

AI 会检查显卡、RAM 和磁盘，选择合适的模型。然后安装并启动，
并告诉你如何连接应用。AI 工具可以通过 Strata 的 [MCP 服务器](docs/MCP_SERVER.md)
安装、启动、停止 Strata。

### 自己动手

[下载 Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) 并解压（或 `git clone`）。
**Windows：** 双击 **`START-HERE.bat`**。**Linux：** 在 Strata 文件夹中运行 **`./setup.sh`**。

NVIDIA 和 AMD 的步骤相同。安装程序会找到你的显卡，并准备与之匹配的引擎。
它会问几个问题：

- 使用哪个模型、哪个大小
- 上下文设多长（模型能记住的文本量）
- 是否让它读图片

每次都按 Enter 就会得到推荐的答案。之后会下载模型（约 70 GB）并启动。
如果下载中断，请再运行一次。它会从断点继续。浏览器中会打开 Strata 应用，
地址是 `http://127.0.0.1:8080`。

> **模型启动期间，PC 可能会变慢或失去响应 1-3 分钟**（首次最长）。
> Strata 会将 35-55 GB 加载到 RAM，并将其一部分固定给显卡使用。这是正常的。
> 请不要关闭窗口，耐心等待。Strata 正在做什么会显示在窗口中。

**下次**再运行一次 `START-HERE.bat`（或 `./setup.sh`）。它会立即启动，不会重复下载相同内容。
要停止模型，关闭窗口即可。使用 `UPDATE.bat`（`./update.sh`）可以在不启动 Strata 的情况下更新。
关于更新、Docker、多张显卡、文件保存位置以及所有选项：[docs/INSTALL.md](docs/INSTALL.md)。

## 该选哪个模型

安装程序会根据 RAM 推荐一个。同一个模型也有压缩强度不同的多个大小。
越小的越快，越大的稍微更聪明。

| RAM | 选择 | 理由 |
| --- | --- | --- |
| **32 GB** | **Coder** | 能装进 32 GB，且专为代码打造（24 GB 显卡的话 Q2_0 和 IQ2_XS 也能运行） |
| **48 GB** | **IQ2_XS**（或最快的 Q2_0） | 更大的大小装不下 |
| **64 GB** | **IQ2_XS**（推荐），或 IQ3_XXS / IQ3_S | 所有大小都能装下。IQ3_S 最聪明，也最慢 |
| **96 GB 以上** | **IQ3_S**，或 Unsloth 的 UD-IQ4_XS（约 4-bit） | 即使其他程序全部打开，也有余量容纳最大的大小 |

- **[Coder](docs/MODELS.md#coder)：** 去掉一半专家的编码专用版。达到完整模型 SWE-bench Verified 分数的
  91%（作者测量），能装进 32 GB 的 RAM。代码以外较弱，也不擅长中文等 CJK 文本（#438）。
  这些用途请选择保留所有专家的 Q2_0、IQ2_XS、IQ3_S。
- **[Swift 1.5](docs/MODELS.md#swift-15)：** 回答前思考时间大幅缩短的微调版。
  质量几乎相同，回答更快。
- **[Unsloth UD-IQ4_XS](docs/MODELS.md#unsloth-ud-iq4_xs)：** Unsloth 的约 4-bit 版，质量介于 IQ3_S 和 UD-Q4_K_XL 之间。
  下载 94 GB。RAM 低于约 80 GB 时，回答期间 Strata 会从 SSD 读取其中一部分，因此会变慢（NVMe SSD 有效）。
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)**（实验性）：最接近完整模型。
  但回答期间 Strata 会从 SSD 读取其大部分内容，因此在 64 GB 的 PC 上每秒只能生成 7-8.5 token。
- **[OrcaRouter's Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)：** 手动设置。
  不在安装程序菜单中。

关于大小、下载以及各自能装在哪里：[docs/MODELS.md](docs/MODELS.md)。之后要添加其他模型，
运行 `SETUP.bat`（Linux：`./setup.sh --setup`）。

## 使用方法

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="Strata 应用的 Monitor 标签页，以及旁边运行的编码智能体"><br>
<sub>Strata 应用的 <b>Monitor</b>（左）。编码智能体正在编写视频中的五重塔庭院（右）</sub></p>

- **在浏览器中：** 打开 `http://127.0.0.1:8080`。有 **Chat**、实时查看模型和 GPU/CPU/RAM 状态的 **Monitor**、
  以及显示设置和地址的 **About**。
- **从应用或编码智能体：** 添加「OpenAI-compatible」提供商，将基础 URL 指定为
  **`http://127.0.0.1:8080/v1`**。API 密钥和模型名可以随意填写。
  - 使用 Anthropic API 的应用：`http://127.0.0.1:8080/v1/messages`（Claude Code：
    `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`）。
  - Codex CLI 等使用 OpenAI Responses API 的应用：`/v1/responses`
    （[配置方法](docs/DETAILS.md#the-responses-api-and-codex-cli)）。
- **思考（Thinking）：** 在聊天菜单或应用的「reasoning effort」中选择 **off、low、medium、high**。
  off 最快，high 适合难题。
- **图片：** 在安装时对「Images?」回答 yes。之后在聊天中点击 **Picture**，或在应用中附加图片。
  AMD 显卡在 Linux 上使用处理器读取图片。Windows 上暂时还无法读取。
- **从智能手机或另一台 PC：** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>`。请务必设置密钥。
- **一次一个请求：** 默认情况下，Strata 回答一个请求，其他请求等待。要同时回答多个请求，
  设置 `"parallel": 2`（[BATCHING.md](docs/BATCHING.md)）。在 12 GB 的显卡上，每个回答都会变慢。
- **长提示词：** Strata 会读取聊天中的第一条消息的全部内容。每 30,000 token 约需 1 分钟。
  第二条及之后的消息几秒内就会开始。

更多信息：[聊天保存位置](docs/INSTALL.md#where-things-are-stored)、[API](docs/DETAILS.md#using-it)。

## 遇到问题时

- **Strata 首次启动时 PC 卡住了。** 加载模型时很常见。请不要关闭窗口，耐心等待。
  过了 10 分钟还是卡住？重启 PC，关闭其他程序再试一次，或选择更小的大小。
- **下载或安装中途停止了。** 再运行一次 `START-HERE.bat`（或 `./setup.sh`）。
  它会从停止的地方继续。
- **非常慢，磁盘指示灯一直闪烁。或显示「the engine stopped unexpectedly」。** PC 的可用 RAM
  不足。关闭其他程序（浏览器很占内存），或选择更小的大小（Q2_0 或 IQ2_XS）。
- **显示端口 8080 已被占用。** Strata 已经在运行。请找到那个窗口。

其他问题和解决方法：[docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md)。如果仍未解决，
请创建 [issue](https://github.com/Niko1221/Strata/issues)，并附上 Strata 文件夹中的 `strata-<model>.log`。
如果发现安全问题，请私下报告：[SECURITY.md](SECURITY.md)。

## 工作原理

这类模型通常运行在拥有数百 GB 显存的服务器上。你的显卡只有
12-24 GB。Strata 通过 **让整台 PC 分担工作** 来容纳模型。想象一下厨房。
常用的工具放在料理台上，其余的收进储藏室。

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="模型的 24,576 个专家：常用的放在显卡上，全部放在 RAM 中，查找表放在 SSD 上"></p>

- **模型是由 24,576 个小专家（「专家」）组成的团队。** 每个词只需要其中 10 个。
- **显卡** 保存常用的数千个专家。**RAM** 保存所有专家，
  **处理器** 同时处理其余部分。**SSD** 保存大型查找表。

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="小助手预测下一个词，大模型批量确认并保留正确的词"></p>

- **先预测再确认：** 小助手预测接下来的几个词。大模型批量确认它们。
  答案保持不变，返回速度快 1.6-1.8 倍。
- **长文本按大块读取**（一次最多 8,192 token）。速度可达每秒 1,000 token 以上。

更详细的说明：[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md)。各部分及其数值：
[详情](docs/DETAILS.md#how-it-works)和[论文](docs/paper/Strata-Paper.pdf)。

## 致谢与许可

模型来自 Qwen 团队的 [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)。
压缩来自 [ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)、UkisAI（Swift 1.5）、
Unsloth。Strata 使用了 [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp) 的一部分。所有致谢：
[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits)。Strata 是采用 [MIT License](LICENSE) 的开源项目。部分组件和
所有模型都有各自的许可（[详情](docs/HOW_IT_WORKS.md#license)）。

## 支持 Strata

Strata 是免费开源的。如果它对你有帮助，可以支持开发：

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>