# 社区基准测试：2 块 RTX 4060 Ti 16 GB，Threadripper PRO 3975WX

测量于 2026-10-05。Strata 0.1.39（`6f32ec0`），Unsloth UD-IQ4_XS，131,072 token 上下文，
层拆分到两块 RTX 4060 Ti 16 GB 显卡上，专家仅部分位于 VRAM 中。两种配置
仅在 MTP 草稿词表上有所不同：一个是意大利语构建的（`prod`，即这台机器日常提供服务的配置），另一个是
setup 的默认配置（`stock`）。工作负载为 RTX 5090 报告中的合成代码解释提示词
（英文），采用贪婪解码，输出上限为 256 token。它不能确立在其他工作负载上的回答质量。

**中位数，`prod`：** 提示 **880 / 1,917 / 2,221 tok/s**，解码 **54.4 / 56.6 / 54.7 tok/s**，分别在
4,096 / 32,768 / 128,000 提示 token 下；128K 下首 token 耗时 **58 s**。6/6 个针全部找到（32K 和 128K）。

## 硬件与软件

- 2 块 NVIDIA GeForce RTX 4060 Ti 16 GB（AD106，34 SM，每块 16,380 MiB），每块 PCIe Gen 4 x8（该卡的
  最大宽度），165 W 功耗限制，默认频率。未使用 NVLink 或 P2P：纯层拆分。
- AMD Ryzen Threadripper PRO 3975WX（32 核 / 64 线程，AVX2，无 AVX-512）；引擎使用了 AVX2
  i-quant 内核。已安装 128 GB DDR4（可用 125.6 GiB）；未检查 DIMM 配置和通道数。
- 模型文件位于 Crucial P310 500 GB NVMe 上（所有专家在启动时已加载到 RAM，因此磁盘不在
  解码路径上）。
- Pop!_OS 24.04 LTS，内核 7.1.5，NVIDIA 驱动 610.57.04。
- Strata `6f32ec070f23ced9f50e704d854d775da52591ab`，引擎 0.1.39，针对 sm_89 使用 CUDA 12.6 的**源码构建**
  （setup 自己的构建；[BUILD.json](BUILD.json)）。图像编码器为 CPU 构建（`vision: cpu`）。
- 后台：桌面会话（GPU 0 上约 100 MiB VRAM）和空闲服务。该机器通常使用的
  Web 前端在整个运行期间已停止，因此没有其他请求到达服务器。通常驻留在 GPU 1 上的 Ollama runner
  在运行开始时已卸载（Ollama 日志），运行期间不存在。

## 模型与配置

- `unsloth/Qwen3.8-Flash-Next-GGUF`，修订版 `38bb39ee97821de2c9009abb7e93950eec396e66`（setup 固定的
  修订版），`UD-IQ4_XS/Qwen3.8-Flash-Next-UD-IQ4_XS-0000{1,2,3}-of-00003.gguf`。所有三个 SHA-256 哈希
  均已由 setup 对照其固定值（`5ce89370…`、`577a38a2…`、`d4634e6d…`）检查。
- 由 setup 准备的包（`iq_pack.py`，原生专家）；捆绑的 `data/expert-profile.bin`；无校准；
  实验性速度预测关闭。
- 视觉：`mmproj-Qwen3.8-Flash-Next-BF16.gguf` 在 CPU 编码器上，`--vision --vram-reserve-mib 700`
  （本次基准测试期间未发送图像）。
- 上下文 131,072；INT8 KV；`--kv-resident 32768`；专家缓存 `auto`（`prod`：CUDA0 上 4,083 个槽位 / 9.20 GiB，
  CUDA1 上 3,678 个槽位 / 8.30 GiB）；预填充 `auto`；层拆分 `auto`（CUDA0 层 0-22，CUDA1
  层 23-47）；`--remote-expert-opt`；`--pcie-frac 0.00`；一次一个请求（无 `"parallel"`）。
- MTP `--spec 4 --spec-min-p 0.50`：
  - `prod`：`--mtp Strata-data/mtp/rt-it`，同一个 Q2_0 草稿层，其草稿词表从
    意大利语维基百科文本重建（`draft_vocab_it.bin`，约 7 MB 的 itwiki 文本），用于意大利语部署；
  - `stock`：`--mtp Strata-data/mtp/rt`，setup 的默认草稿词表。
- 请求：`temperature 0`，`reasoning_effort "none"`，`max_tokens 256`（基准测试脚本的默认值）。

确切的运行配置为 [prod/strata-config.json](prod/strata-config.json) 和
[stock/strata-config.json](stock/strata-config.json)（保留了本地路径；未配置 API 密钥）。

## 方法

[benchmark.py](benchmark.py) 和 [monitor.py](monitor.py) 是 RTX 5090 报告的脚本，**未更改**。
对于每种配置：全新启动服务器，一次预热请求，然后在 4,096、32,768 和
128,000 提示 token 下各运行三次。每个提示词都以自己的 nonce 开头，因此不重用任何前缀：引擎报告
每次测量运行的重用 token 均为 0。提示和解码速率是引擎自身的计时（`/metrics`），
TTFT 是客户端侧基于流式的（第一个非空增量；推理关闭，因此它是回答文本）。
模型加载不包含在内。`monitor.py` 每秒采样一次 RAM 和 GPU 内存。
[`tools/needle_bench.py`](../../../tools/needle_bench.py) 仅在 `prod` 上运行：`--lengths 32k,128k --depths 10,50,90`。

每次运行的结果（含引擎指标和生成的文本）位于 `*/results.json`，摘要位于
`*/summary.json`，引擎日志位于 `*/engine.log`，遥测数据位于 `*/telemetry.jsonl`，机器快照位于
[machine.txt](machine.txt)，运行日志位于 [run.log](run.log)。

## 结果

| 配置 | 提示 token | 重用 | 生成 | 运行次数 | 提示 tok/s 中位数 [范围] | 解码 tok/s 中位数 [范围] | TTFT s 中位数 [范围] |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| prod（意大利语草稿词表） | 4,096 | 0 | 256 | 3 | 880 [821-881] | 54.4 [52.4-55.0] | 4.7 [4.7-5.0] |
| prod | 32,768 | 0 | 256 | 3 | 1,917 [1,904-1,928] | 56.6 [55.4-57.4] | 17.2 [17.1-17.3] |
| prod | 128,000 | 0 | 256 | 3 | 2,221 [2,220-2,245] | 54.7 [54.0-56.2] | 58.0 [57.3-58.0] |
| stock（默认草稿词表） | 4,096 | 0 | 256 | 3 | 906 [870-911] | 51.9 [51.2-52.4] | 4.6 [4.5-4.7] |
| stock | 32,768 | 0 | 256 | 3 | 1,934 [1,932-1,944] | 55.3 [53.6-56.1] | 17.0 [17.0-17.1] |
| stock | 128,000 | 0 | 256 | 3 | 2,234 [2,198-2,247] | 55.7 [55.4-55.9] | 57.6 [57.3-58.6] |

- 每次测量运行都在 256 token 上限处结束（`finish: length`）。没有请求失败或停滞。
- 解码专家缓存命中率 0.80-0.90（每种配置的首次 4,096 token 运行最低）。
  MTP 草稿接受：`prod` 214-243 中接受 151-164，`stock` 206-244 中接受 141-162。
- 内存，整个运行期间的峰值：`prod` 的 VRAM 为 15,691 / 15,822 MiB（GPU 0 / GPU 1），`stock` 为 15,705 / 15,794 MiB；
  系统 RAM 使用量（所有进程）为 125.6 GiB 中的 80.8 GiB；未使用交换空间。GPU 功耗峰值
  为 146-149 W，温度为 74-76 °C。

## 正确性与局限性

- 针（`prod`）：**6/6 找到**，位于 32,171-32,172 和 125,449-125,451 提示 token，深度 10 / 50 / 90 %
  （[prod/needles.json](prod/needles.json)）。
- 意大利语草稿词表并未拖慢这些英文/代码提示词：解码在 32K 和 128K 下处于运行间
  范围内，在 4K 下略快（54.4 vs 51.9 tok/s）。仅一台机器且各运行三次，
  因此这是一个观察结果，而非普遍结论。
- 并发不属于本次基准测试的一部分。本机上 `"parallel": 2` 的单独测量见
  #857 的评论。
- 未测量：图像（编码器已加载，但未发送图像）、开启思考的解码、采样解码、
  其他量化，以及开放的提示路径 PR（#835、#849、#854）。