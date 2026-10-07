# 社区基准测试：2× RTX 3060 12 GB（Xeon E5-2678 v3），Qwen3.8-Flash-Next IQ3_S

由 [zimuhuan-code](https://github.com/zimuhuan-code) 于 2026-10-04 在一台无头 Linux 服务器上测量。本测试针对 **Strata 0.1.37** 与 **Qwen3.8-Flash-Next GSQ-RCO IQ3_S**，使用两块 RTX 3060 进行层拆分，并采用 **524,288 token** 上下文。这是对最初发布于 #602 中数据的正式整理，现补充了每次运行的数据、首 token 时间、内存观测以及构建产物哈希。

**首先说明主要局限性：** 提示词是**合成的**（一段固定的英文填充段落重复至目标长度，每次运行带有唯一的前导标签），并非真实工作负载，且此处仅重新测量了**一种量化**。#602 中的 Q2_0 对比作为**历史单次运行**收录——其权重和 pack 已不在磁盘上，未重新运行。

每种大小取三次运行的中位数，贪婪解码，输出上限 256 token，无提示词前缀复用（每次运行 `cache_n = 0`）：

| 提示词大小（实际 token 数） | 提示词 tok/s（中位数，范围） | 解码 tok/s（中位数，范围） | TTFT 秒（流式） | 总耗时 秒（中位数） |
| --- | --- | --- | ---: | ---: |
| 3,197 | 603.0 (601.2–604.8) | 43.6 (42.2–44.0) | 5.4 | 10.4 |
| 24,798 | 1,156.3 (1,154.4–1,159.3) | 42.5 (41.4–43.1) | 21.8 | 25.4 |
| 98,599 | 1,323.3 (1,321.0–1,324.7) | 40.0 (37.1–40.6) | 75.5 | 79.6 |

此处提示词吞吐量随提示词长度**上升**（603 → 1,323 tok/s）：在几千 token 时，每请求的固定开销占主导，随着提示词增长而被摊薄。#602 中"prefill 38–43 tok/s"的数据来自五提示词混合中约 50–200 token 的提示词，与上述数字**不**具可比性。

## 硬件与软件

- 2× **NVIDIA GeForce RTX 3060，各 12,288 MiB**（共 24 GiB），驱动 **595.84**，功耗
  限制 **170 W**（最大 187 W），**无超频，无锁频**。PCIe 能力
  **Gen 3 ×16**；空闲链路状态读数为 **Gen 1 ×16**（NVIDIA 链路节能），我们
  在生成期间测得**两块 GPU 均为 Gen 3 ×16**（`nvidia-smi`，请求运行期间以 250 ms 采样）。
  这些运行中未锁定链路。
- **Intel Xeon E5-2678 v3 @ 2.50 GHz**，2 路 × 12 核 = **24 核 / 48 线程**
  （AVX2，无 AVX-512）。引擎侧专家 worker 保持引擎默认值。
- 操作系统可见 **121.5 GiB RAM**（`free`）。⚠️ 在比较 prefill 数据时需要注意的平台说明：
  这是一块**改装 X99 主板（华南金牌），在 LGA2011-3 Xeon E5 v3 上运行 DDR3-1867**——
  该插槽上的 DDR3 是主板级特性而非 CPU 特性，因此其内存带宽未必与原生 DDR4 X99 平台一致。
  引擎加载了 **46.84 GiB 专家权重，速率 3.42 GiB/s（43 秒）**，并报告常驻"约 55 GB"。
- **单位：** 下文所有容量均为二进制（MiB / GiB）；GPU 标称"12 GB"，即同样的 12,288 MiB。
- 存储：模型位于 **NVMe** 设备（`nvme1n1`，1.4 TB，ext4）；另有两块空闲 HDD，未被使用。
- **其他工作负载：无。** 该机器上共享这两块 GPU 的另一个本地引擎（`ftllm`）在 Strata 运行期间已禁用。
  测量开始前引擎已空闲约 32 分钟，且该机器的交互客户端（一个 DSH 会话）连接的是**远程**模型，
  因此未占用这两块 GPU。
- 操作系统 **Ubuntu 26.04.1 LTS**，内核 **7.0.0-30-generic**，glibc **2.43**。
- **CUDA 12.8**（nvcc V12.8.93；12.9 也已安装但在此处构建失败——见 #601）。
  默认 `g++` 为 15.2.0；引擎使用 **g++-14** 构建，并应用了本地 `setup.py` 补丁，
  通过 `STRATA_NVCC` 选择工具包。
- **Strata 提交 `db4f91a1171d697928b0d2e1f50ef95e25559d4c` — v0.1.37，从源码构建**
  （没有 Linux 发布二进制）。llama.cpp 由 setup 固定在
  `3cf03257f219afbe7334045ff7c6a06ac68c627d`。Python **3.14.4**（setup venv）。

## 模型与配置

- 模型仓库 **`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`**，修订版本
  **`ed59f92082b1e93c0e96d60a8b11aab089b52f09`**。
- 量化 **IQ3_S**：
  - `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` — 54,817,524,224 B
  - `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf` — 28,800,138,432 B
  - 每个已发布构建产物的 SHA-256 见 [`artifact-sha256.txt`](artifact-sha256.txt)
    （**本地计算；我们未独立核对仓库的 LFS 哈希**）。该 pack 还包含 `dense.bin`、`index.txt`、
    `native_experts.txt`、`conversions.json` 及 tokenizer 目录，均列于该文件中。
- 视觉编码器**关闭**；pack、专家 profile 和 MTP 草稿 pack 由仓库的 `setup.sh` 生成，
  **无手工修改**；实验性速度预测**关闭**。
- 上下文 **524,288**；**yarn rope 缩放 ×2**（超出训练的 262,144）；KV **int8**，
  `--kv-resident 32768` 个单元保留在 VRAM 中。
- 专家缓存 **auto** → 本次启动填充了 **3,579 个专家（6.38 GiB VRAM）**。
  ⚠️ 记录在案：此前在该机器上的启动（2026-10-03）在另外两种配置下分别报告了 **5,549** 和
  **2,574** 个专家，我们的笔记中还包含一次 64K 上下文运行的 **5,857**；该数字取决于配置，
  因此只有上述 3,579 属于本报告中的运行。
- 低内存模式**关闭**；prefill **auto**；MTP / 投机解码**开启**
  （`--spec 4 --spec-min-p 0.5`）。
- 采样：**temperature 0**，无 top-p/top-k 覆盖（引擎默认值），`max_tokens 256`，
  推理保持引擎默认值。

精确启动配置（路径已缩短；完整文件为 `strata-config.json`）：

```text
serve/server.py --engine strata --config <config>.json --port 8081

args:
  --pack <data>/packs/iq3_s
  --native <data>/models/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf
  --ple-gguf <data>/models/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf
  --expert-profile <work>/data/expert-profile.bin
  --expert-cache auto --prefill auto
  --spec 4 --spec-min-p 0.5 --mtp <data>/mtp/rt
  --max-context 524288 --rope-scaling yarn --rope-scale 2
  --kv int8 --kv-resident 32768
```

## 方法

- 运行器：[`bench.py`](bench.py)（仅标准库）。它收集主机元数据，每 2 秒采样一次
  `nvidia-smi` 和 `/proc/meminfo`，并将每个原始响应写入 `raw/`。
- **提示词**为 `FILLER`（一段固定段落——确切文本是
  [`bench.py`](bench.py) 中的 `FILLER` 常量；其 SHA-256 和字符数记录在 `results.json` 的
  `meta.protocol.filler_sha256` / `filler_chars` 下）重复至目标大小，前缀为 `[run <tag>]`，
  后缀为固定任务行。唯一标签意味着**前缀从不复用**——九次测量运行全部 `cache_n = 0` 已证实。
  请求大小为 4k/32k/128k；**实际**提示词 token 数为 3,197 / 24,798 / 98,599，所有地方均报告这些实际值。
- **计时来源：** 主要数据由运行器发送**非流式**请求并读取引擎自身的 `timings` 块
  （`prompt_n`、`cache_n`、`prompt_ms`、`prompt_per_second`、`predicted_n`、`predicted_ms`、
  `predicted_per_second`、`draft_n`）。解码吞吐量为引擎的 `predicted_per_second`，
  **而非**生成 token 数除以总请求时间。
- **TTFT** 在此版本上无法通过该方式获得：流式路径不输出 `usage`/`timings`。因此它在
  **每种大小额外一次流式运行**中测量，使用相同的提示词构造，测量从发送请求到第一个非空
  `delta` 的时间（此处始终为 `reasoning_content`，即第一个 token 是思考 token）。
  保活注释和空 delta 被忽略。这三次运行仅用于 TTFT，不属于中位数表格。
- **运行顺序与状态：** 九次非流式运行按大小顺序（每种大小 3 次），然后是三次流式运行。
  运行之间服务器**未重启**，我们有意**未**重置或清空 KV 缓存或专家缓存——因此专家缓存是
  **热的**。保持运行可比性的是每次运行都带有**不同的提示词前缀**，因此没有提示词 token 被复用：
  九次测量运行全部 `cache_n = 0`。每请求的专家缓存命中率由引擎记录（此处观察到 58–82 %）。
- **预热与加载：** **模型加载不计入任何计时**——引擎在首次测量运行前约 55 分钟已加载权重和专家池。
  它在此之前还已响应过一次健康检查和四次短探测请求（包括一次用于验证运行器的 512 token 提示词）。
- 整个会话的引擎侧计时行见 [`engine.log`](engine.log)。

## 结果

每次运行的完整数据见 [`results.json`](results.json)（包括完整的引擎 `timings` 块和每次运行的
GPU/RAM 观测），未修改的 API 响应见 `raw/`，2 秒内存采样见 [`samples.jsonl`](samples.jsonl)。

| 配置 | 实际提示词 token 数 | 复用 token 数 | 生成 token 数 | 运行次数 | 提示词 tok/s（中位数，范围） | 解码 tok/s（中位数，范围） | TTFT 秒（中位数） | 总延迟 秒（中位数，范围） |
| --- | ---: | ---: | ---: | ---: | --- | --- | ---: | --- |
| IQ3_S, 524,288 ctx, 贪婪, 256 上限 | 3,197 | 0 | 160–220 | 3 | 603.0 (601.2–604.8) | 43.6 (42.2–44.0) | 5.4 | 10.4 (9.0–10.4) |
| IQ3_S, 524,288 ctx, 贪婪, 256 上限 | 24,798 | 0 | 150–174 | 3 | 1,156.3 (1,154.4–1,159.3) | 42.5 (41.4–43.1) | 21.8 | 25.4 (25.2–25.7) |
| IQ3_S, 524,288 ctx, 贪婪, 256 上限 | 98,599 | 0 | 152–185 | 3 | 1,323.3 (1,321.0–1,324.7) | 40.0 (37.1–40.6) | 75.5 | 79.6 (78.8–80.2) |

- **内存：** VRAM 峰值 **11,549 MiB（GPU 0）/ 11,691 MiB（GPU 1）**，共 12,288 MiB；这些是
  整个基准测试期间的峰值，每 2 秒采样一次，而非启动快照。系统 RAM 峰值
  **已用 68.68 GiB**（共 121.5 GiB），引擎约 55 GiB 专家池常驻。
- **失败：** 无。每个请求均返回 `finish_reason: length`（256 token 上限），
  无错误，无取消请求。
**数据格式**（指南要求每份报告记录自身的字段和单位）：

- `results.json` → `meta`（运行器捕获的主机 / 引擎 / 协议）、`runs[]`、
  `summary`（每种大小的中位数 + 范围）、`gpu_peak_mib`、`ram_peak_used_gb`。
  每个 `runs[]` 条目：`run`（标签）· `mode`（`nonstream` / `stream`）· `target_prompt_tokens` ·
  `filler_repeats` · `prompt_chars` · `wall_seconds`（客户端侧，请求发送 → 最后一字节）·
  `prompt_tokens_reported` / `completion_tokens_reported`（引擎 `usage`）· `finish_reason` ·
  `content_chars` / `reasoning_chars` · **`timings`**（引擎的块，逐字：`prompt_n`、
  `cache_n`、`prompt_ms`、`prompt_per_second`、`predicted_n`、`predicted_ms`、
  `predicted_per_second`、`draft_n`）· `prompt_tok_s` / `decode_tok_s` / `cache_n`（为方便起见
  从 `timings` 中提取）· `gpu_mib_after` / `ram_after`。流式运行携带
  `ttft_seconds`、`content_chunks`、`reasoning_chunks` 而非 `timings`。
- `samples.jsonl` → 每 2 秒一个对象：`{t, gpu_mib: {"0": MiB, "1": MiB}, ram: {...}}`。
  ⚠️ 命名怪癖，特此说明以免误导：运行器的 `ram.*_gb` 键使用基于 1024 的算术计算，
  即尽管后缀为 `_gb`，它们实际是 **GiB**。本 README 中引用的每个内存数字均为二进制（MiB / GiB）。
- `artifact-sha256.txt` → 纯 `sha256sum` 输出，`<hash>  <relative path>`。
- `needles.json` → 仓库脚本自身的输出格式。
- `strata-config.json` → 加载时的服务器配置文件，本地路径已缩短。

- **历史对比（未重新运行）。** 同一机器和引擎使用 **Q2_0** 在 2026-10-03 测量了单次记录运行：
  **45.0 tok/s 解码**（1,015 token / 22.6 秒加权平均）和 **5,549** 个专家缓存槽，
  比 IQ3_S 快约 25 %。这些权重及匹配的 pack 之后被删除（回收 37 GiB），**不**属于本报告的
  测量数据；Q2_0 长上下文数据（144k 时 prefill 1,452 tok/s，511k 时 855 tok/s，
  512K needle 通过）来自同一单次记录会话，同样属于**历史数据**。

## 正确性与局限性

**召回检查** — 仓库脚本 `tools/needle_bench.py`，相同配置，在速度运行之后
（因此：专家缓存热，且此处的提示词前缀在深度之间*确实*被复用，这就是其计时不属于速度表的原因）：

| 长度 | 深度 | 结果 | 提示词 token 数 | 耗时 |
| --- | --- | --- | ---: | ---: |
| 32k | 10 % / 50 % / 90 % | 找到 / 找到 / 找到 | 31