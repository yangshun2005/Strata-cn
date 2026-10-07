# 社区基准测试（重跑）：2x RTX 5060 Ti（16 GB），Xeon E5-2690 v4

由 R6DJO 于 2026-10-05 对 [2026-10-02 报告](../2026-10-02-community-rtx-5060-ti-x2/README.md) 进行的重跑，此前引擎已更新至 0.1.39，并更改了启动配置（见下方硬件与软件）。硬件与之前相同；本次重跑也澄清了其描述：1 个 CPU（1 个插槽）和四通道内存（4x 32 GB DDR4-2133 ECC，依据 `dmidecode -t memory`——2026-10-02 报告中的“已安装 125 GB”与 MemTotal 所报告的 4x 32 GiB 相同）。模型相同：原始的 Flash-Next GSQ-RCO IQ3_XXS，两块 GPU 采用固定的层拆分，以及 262,144 token 的上下文限制。

解码吞吐量中位数为 **4,096 提示 token 时 84.8 tok/s，32,768 时 86.5 tok/s，128,000 时 83.8 tok/s**；提示吞吐量中位数为 **4K 时 911 tok/s，32K 时 2,087 tok/s，128K 时 2,440 tok/s**（上一次运行：解码 77.5 / 72.7 / 72.4，提示 808 / 1,931 / 2,199）。这些是合成的代码解释请求，采用贪婪解码，输出上限为 256 token。它们不能证明一般答案质量或其他工作负载下的性能。由于两次运行之间引擎版本和启动配置发生了变化，而硬件没有变化，因此差异不能归因于任何单一更改。

## 硬件与软件

- 2x NVIDIA GeForce RTX 5060 Ti，每块报告 VRAM 为 16,311 MiB，功耗限制 180 W，计算能力 12.0。本次测试未固定 GPU 时钟。
- Intel Xeon E5-2690 v4 @ 2.60 GHz；1 个 CPU（1 个插槽），14 核 / 28 线程；无 AVX-512（引擎报告其在 AVX-2 上运行专家内核）。13 个专家池工作进程。主板：HUANANZHI X99-BD4，BIOS 5.11。
- 已安装 128 GB 内存：4x 32 GB DDR4-2133 ECC，四个通道全部占用（依据 `dmidecode -t memory`，通道组 1 上的 DIMM_A1/B1，通道组 2 上的 DIMM_C1/D1），四通道；MemTotal 报告为 125.7 GiB，2026-10-02 运行安装的是相同内存。基准测试开始时约 70 GB 可用。交换空间：2 GB swapfile。存储：仓库和 pack 位于 Netac NVMe SSD 1 TB 上；GGUF 分片位于 /mnt/data 的 SATA SSD 上。
- Linux Mint 22.3，内核 7.0.0-31-generic，NVIDIA 驱动 610.57.04。
- 引擎 0.1.39（由 `/metrics` 报告）。本地编译，`CMAKE_BUILD_TYPE=Release`，`CUDA_ARCHITECTURES=120`，CUDA toolkit 13.4，gcc 13.3.0。
- 其他 GPU 工作负载：无。同一 CPU 上的常规桌面服务仍在运行；这不是一个完全隔离的操作系统。

## 模型与配置

模型：`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`（文件名和大小见 [model-files.txt](model-files.txt)）：

- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`
- `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`

视觉编码器：无（`images: false`）。自定义 pack（`Strata-data/packs/iq3_xxs`）；MTP 草稿头为已发布的那个（`Strata-data/mtp/rt`）。

- 上下文限制 262,144；KV int8，常驻窗口为 32,768 token。
- 专家缓存自动（`expert_slots` 14,547；23,893 MiB，其中 12,593 MiB 在主卡上），预填充自动。
- MTP 开启（`--spec 4 --spec-min-p 0.70`；引擎报告 `spec: 6, mtp_max: 4`）；请求中推理强度为 `none`。
- 层拆分固定为跨两块 GPU 的 25/…（`--layer-split 25`；2026-10-02 运行使用 `auto`）。与 2026-10-02 相比，启动还增加了 `--remote-expert-opt --conversation-cache-mib 8192 --trim-stage-weights`，并使用学习到的专家配置文件（`expert-profile-learned.bin`，由 `--expert-profile-save` 保存）；`--pcie-frac 0.19`（原为 0.20）。
- 采样：temperature 0。

```text
<strata-dir>/.venv/bin/python <strata-dir>/serve/server.py \
  --engine strata --config <strata-dir>/strata-iq3_xxs.json \
  --port 31000 --host 0.0.0.0
# config strata-iq3_xxs.json passes to the engine:
#   --pack .../packs/iq3_xxs --native <shard 1> --ple-gguf <shard 2>
#   --expert-profile expert-profile-learned.bin --expert-cache auto --prefill auto
#   --spec 4 --mtp .../mtp/rt --max-context 262144 --kv int8 --kv-resident 32768
#   --remote-expert-opt --pcie-frac 0.19 --spec-min-p 0.70
#   --conversation-cache-mib 8192 --trim-stage-weights --layer-split 25
#   --expert-profile-save expert-profile-learned.bin
# (the full JSON, with local paths, is in strata-iq3_xxs.json)
```

## 方法

基准测试脚本是 [RTX 5090 社区报告中的串行全新提示脚本](../2026-09-30-community-rtx-5090/benchmark.py)，针对此服务器的 tokenizer pack 运行。对于 4,096 / 32,768 / 128,000 提示 token 中的每一个，它执行 3 次运行；每个请求都带有一个唯一的 nonce，因此对话缓存永远不会重用之前的提示（全部九次运行均为 `reused: 0`，由引擎的每请求指标确认）。在测量的运行之前进行了一次预热请求（16 个输出 token）；模型加载不计入任何计时。

请求为贪婪（`temperature 0`），`reasoning_effort: "none"`，输出上限为 256 token；所有运行都在上限处结束（`finish_reason: "length"`，每次生成 256 token）。提示和解码 tok/s 根据引擎的每请求计时（`prompt_ms`、`decode_ms`）计算，而非总请求时间。TTFT 和耗时是客户端流式测量（排除 keep-alive 和空增量；第一个 token 是答案文本，推理已关闭）。运行期间未采样内存（`not measured`）。测量运行的每请求解码命中率为 0.98-0.99（值见 [results.json](results.json)）。取消或失败的请求：无。

每次运行的数据：[results.json](results.json)（每次运行一行，包含引擎计时、用量和响应文本），每次运行的 `tokens-*-raw.json` 和 `tokens-*-request.json`，引擎指标快照见 [engine-timing-lines.txt](engine-timing-lines.txt)。

## 结果

| 配置 | 实际提示 token | 重用 token | 生成 token | 运行次数 | 提示 tok/s 中位数（最小-最大） | 解码 tok/s 中位数（最小-最大） | TTFT s 中位数（最小-最大） |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| IQ3_XXS，2 GPU，4,096 token 提示 | 4,096 | 0 | 256 | 3 | 911 (911-915) | 84.8 (74.3-88.9) | 4.54 (4.52-4.54) |
| IQ3_XXS，2 GPU，32,768 token 提示 | 32,768 | 0 | 256 | 3 | 2,087 (2,087-2,099) | 86.5 (80.1-87.2) | 15.82 (15.73-15.83) |
| IQ3_XXS，2 GPU，128,000 token 提示 | 128,000 | 0 | 256 | 3 | 2,440 (2,438-2,457) | 83.8 (81.4-84.0) | 52.87 (52.47-52.91) |

客户端总请求时间（中位数）：4K 时 7.5 s，32K 时 18.7 s，128K 时 55.9 s。每次测量的 128K 运行都接受了推测性草稿（例如第 2 次运行中 180 个中接受了 145 个）。推理期间的 RAM/VRAM 使用：未测量。

## 正确性与局限性

长上下文召回使用 [`tools/needle_bench.py`](../../../tools/needle_bench.py) 在 32K 和 128K 提示 token 下检查，needle 位于 10% / 50% / 90% 深度：**全部六项检查都找到了代码词**（每次都是精确答案）；见 [needles.json](needles.json)。每请求时间：32K 时 15.9-18.7 s，128K 时 45.2-50.0 s，包括生成。

局限性：一种量化（IQ3_XXS），一种工作负载（合成代码解释，贪婪），输出上限为 256 token，GPU 时钟未固定，且同一台机器上运行着桌面服务。两次运行之间引擎版本（0.1.35 -> 0.1.39）和启动配置一起发生了变化，因此本次重跑并未隔离速度差异的原因。这些数字不能证明 needle 检查之外的答案质量或其他工作负载下的性能。