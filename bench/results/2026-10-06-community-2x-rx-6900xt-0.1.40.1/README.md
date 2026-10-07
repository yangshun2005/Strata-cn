# 社区基准测试：2x AMD Radeon RX 6900 XT 16 GB (gfx1030)、Ryzen 5 5600X、128 GB RAM - Strata 0.1.40.1

于 2026-10-06 在 Linux 桌面机器上测量。本测试以两种显卡的三种使用方式——单卡、跨双卡的层拆分，以及一张卡搭配另一张卡作为专家辅助（`--remote-expert-opt`）——测试原始 Qwen3.8-Flash-Next（GSQ-RCO IQ3_S）在 131,072 token 上下文下的表现，分别运行于已发布的 0.1.40.1 引擎、带有该卡族所附带两个开关（`STRATA_HIP_PROMPT_F16=1 STRATA_SH_STREAM=1`）的 0.1.40.1，以及在这些开关之上应用了三个开放的 gfx103x Pull Request #1150、#1151 和 #1167 的 0.1.40.1。所有配置中自适应专家交换均为**开启**（默认值）；本机此前的 0.1.39 报告因 #884 将其关闭。

在原始 0.1.40.1 上，单卡读取提示词约为 **460 tok/s**，解码为 47-48 tok/s；使用两个开关后，同一张卡读取速度为 **811 / 973 / 913 tok/s**（4K / 32K / 128K），解码为 49-51；应用 Pull Request 后读取速度为 **969 / 1,152 / 1,126**。应用 Pull Request 后的层拆分读取 128,000 token 提示词的速度为 **2,003 tok/s**（首 token 耗时 64 秒，相比之下原始拆分为 156 秒，单张原始卡为 279 秒），解码为 70-75 tok/s；辅助模式在所有分支中解码为 64-75 tok/s。全部 54 项召回检查（每个配置六个针）均被找到。这些请求是合成的代码解释请求，采用贪婪解码和 256 token 输出上限；它们不能确立答案质量或在其他工作负载上的性能。

**在阅读双卡数据之前需要了解的一件事：** 在自适应交换开启且 GPU 的 GFX 电源门控（GFXOFF）处于内核默认值的情况下，原始版和仅开关版的层拆分都在其首个 128,000 token 请求上停滞（引擎在读取提示词时停止推进；服务器将其重启）。因此，下面的双卡配置是在通过 amdgpu debugfs 旋钮禁用 GFXOFF 的情况下测量的，在本机上这消除了停滞且不改变速度（停滞运行的 4K 和 32K 行保留在 `*-gfxoffon` 文件夹中，与 GFXOFF 关闭的行逐 token 匹配）。单卡配置在默认值下运行且未停滞。详情见下文"停滞"部分；该问题是 #884。

## 硬件和软件

- **GPU：** 2x AMD Radeon RX 6900 XT 16 GB（Navi 21，gfx1030，wave32，无矩阵核心），公版卡，245 W 功耗
  上限，无频率更改，未连接显示器。两者均接在 X570 主板的 CPU PCIe 4.0 通道上，每张卡获得 x8（引擎
  自身的探测：每张卡主机到设备 13.6-14.1 GB/s）。下文任何模式均未使用卡间 P2P。
- **CPU 和 RAM：** AMD Ryzen 5 5600X（6 核 / 12 线程，AVX2，无 AVX-512；引擎运行了 5 个专家池工作线程加
  其宿主线程）。128 GB DDR4-3200（4 条 DIMM）。8 GiB 交换文件。
- **存储：** 模型文件和 pack 位于一块 NVMe SSD（Samsung 970 EVO 500 GB）上，同时也是系统盘。PLE
  表从其上的 GGUF 分片读取（`--ple-io direct`，默认值）。
- **操作系统和运行时：** Ubuntu 26.04，内核 7.0.0-38-generic 及其自带的 amdgpu 驱动（无 DKMS），ROCm 10.0.0（HIP
  7.15，rocBLAS 5.6.0.8d1ae90e）。hipBLASLt 不提供 gfx1030 内核，因此运行的是普通 hipBLAS/rocBLAS 路径。
- **源码，原始版：** [Niko1221/Strata](https://github.com/Niko1221/Strata) `main` 分支 `82f46a8`（引擎 0.1.40.1）。
- **源码，"PRs"：** `82f46a8` 加上 #1150（PR #540 在 gfx103x 上的注意力内核，位精确）、#1151（在 rocBLAS SGEMM
  上的 QSA 块分数，需启用 `STRATA_SELECT_SGEMM=1`）和 #1167（用于 FP16 提示词
  GEMM 的 rocBLAS 解表，`STRATA_ROCBLAS_TUNING`）在本地合并，再加上 #1148（一个测试跳过）和 #1149（测试及一个默认关闭的
  FP16 路径范围计数器）；每个 `BUILD.txt` 中的树为 `341922e`。#1148 和 #1149 的引擎源码
  不改变此处的任何数字。
- **构建：** 两棵树均使用 cmake 和 ROCm 的 clang 手工构建：`-DCMAKE_BUILD_TYPE=Release -DSTRATA_ENABLE_HIP=ON
  -DCMAKE_HIP_ARCHITECTURES=gfx1030 -DSTRATA_PREFILL_MMQ=ON`（setup 用于 AMD 的选项），llama.cpp 位于固定的
  提交。每个配置的 `BUILD.txt` 包含树、提交、二进制的 SHA-256、环境以及
  启动时 GFXOFF 旋钮的读数。
- **其他负载：** 运行期间没有其他程序使用 GPU（遥测见下文）。机器的其他服务（一个空闲的
  llama-swap 路由器、一个家庭自动化栈、LACT 风扇曲线守护进程）继续在 CPU 上运行。

## 模型和配置

**模型：** [`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)，
IQ3_S——与 setup 为其 IQ3_S 选项下载的相同两个文件，使用仓库的 `tools/iq_pack.py` 打包（无
`--compat-bf16`）；SHA-256 哈希与仓库的 `.sha256` 文件匹配（它们列在本机 2026-10-05 的报告中）。MTP 草稿层使用仓库的工具准备。专家配置文件是仓库
自带的 `data/expert-profile.bin`。无视觉编码器、无校准、无速度预测。

**配置**（每个配置的 `config.json` 在其文件夹中；`/m` 是模型目录，`/work/stock` 和
`/work/prs` 是两棵树）。所有配置共有：上下文 131,072，`--kv int8 --kv-resident 32768`（每个
注意力层在 VRAM 中 32,768 个 K/V 单元，其余从固定内存流式传输），`--expert-cache auto`，`--prefill auto`（8,192 token
块），MTP 使用 `--spec 4 --spec-min-p 0.5`，后缀草稿开启，自适应专家交换为默认值
（`--adapt-every 4`，未传入）。

| 名称 | 显卡 | 引擎 | 环境 |
| --- | --- | --- | --- |
| `single-stock` | 一张（PCI 0f） | 原始版 | 无 |
| `single-sw` | 一张 | 原始版 | `STRATA_HIP_PROMPT_F16=1 STRATA_SH_STREAM=1` |
| `single-prs` | 一张 | PRs | 上述两个 + `STRATA_ROCBLAS_TUNING=/work/prs/tools/hip STRATA_SELECT_SGEMM=1 STRATA_SELECT_SGEMM_VERBOSE=1 STRATA_IO_THREADS=128` |
| `split-stock` | 两张，`layer_split: auto` | 原始版 | 无 |
| `split-sw` | 两张 | 原始版 | 两个开关 |
| `split-prs` | 两张 | PRs | 同 `single-prs` |
| `helper-stock` | 一张 + 另一张作为辅助（`--expert-cache-device1 auto --remote-expert-opt`），探测到的 PCIe 份额 0.39 | 原始版 | 无 |
| `helper-sw` | 同上 | 原始版 | 两个开关 |
| `helper-prs` | 同上 | PRs | 同 `single-prs` |
| `split-stock-gfxoffon`、`split-sw-gfxoffon` | 两张 | 原始版 | 同上述两个拆分分支，GFXOFF 为内核默认值；在 128K 处停滞（见下文） |

引擎自身的日志确认每个开关：`STRATA_HIP_PROMPT_F16=1: the prompt's 16-bit GEMMs run FP16 in and out`
（每张卡一次），`prefill gemm: rocBLAS tuning enabled (112 rows, 76 with a solution, gfx1030, rocBLAS 5.6.0.8d1ae90e)`
以及 `qsa select: SGEMM reach<=... ` 测量行（每个文件夹中的 `engine.log`）。

**GFXOFF。** `/sys/kernel/debug/dri/{0,1}/amdgpu_gfxoff`（4 字节，小端）在三个
单卡分支和两个 `*-gfxoffon` 分支中为 1（默认值），在六个双卡分支中为 0，在 `split-stock`
之前以 root 写入，并在 `helper-prs` 之后写回 1；每次启动时旋钮的读数在 `BUILD.txt` 中。该旋钮在 amdgpu 中是一个引用
计数（`amdgpu_gfx_off_ctrl`）：0 只能被后续的 1 撤销。

## 方法

[benchmark.py](benchmark.py) 是
[RTX 5090 报告](../2026-09-30-community-rtx-5090/README.md)的脚本，未更改。它构建确定性的合成 Python
填充内容，在每个请求开头附近放置不同的 nonce，并使用 Strata 的
分词器计算渲染后的聊天提示词。[monitor.py](monitor.py) 每秒从 sysfs 采样一次 RAM、交换和两张卡的 VRAM、GTT、忙碌百分比、功耗和温度。

每个配置启动自己的服务器（从其树运行 `serve/server.py --engine strata --config <config.json> --port 8089`），并按以下顺序运行：

```bash
python3 benchmark.py --root /work/<tree> --pack /m/packs/iq3_s --url http://127.0.0.1:8089 --out <folder> --targets 4096,32768,128000 --runs 3
python3 /work/<tree>/tools/needle_bench.py --url http://127.0.0.1:8089 --lengths 32k,128k --depths 10,50,90 --out <folder>/needles.json
```

- 一个短的预热请求被排除。
- 每个长度三次运行串行执行，按长度递增顺序，在同一个已加载的引擎上。
- 九个完整配置的全部 81 个速度请求都读取了其完整提示词：**零复用 token**；每一个都生成了 256 个 token 并在输出上限处停止（`results.json`）。
- 专家缓存在启动时从配置文件填充，并在请求之间保留。加载时间被排除。
- TTFT 为流式，从 HTTP 请求之前到第一个非空文本增量，通过回环。
- 提示词吞吐量为新读取的 token / `prompt_ms`；解码吞吐量为引擎自身的 `decode_tok_s`，两者均来自
  引擎自身的统计。
- `reasoning_effort: none`，温度 0。
- 每个文件夹包含其会话的引擎完整日志（`engine.log`）、基准测试的输出、`results.json`
  （请求哈希和每个输出文本）、`summary.json`、`needles.json` 和 `needles.out`、`telemetry.jsonl`、
  服务器结束时的状态以及 `BUILD.txt`。

十一个服务器在本地时间 14:19 至 17:51 之间依次运行：三个单卡分支，然后是两个
停滞的拆分分支（15:30-15:56），然后是六个禁用 GFXOFF 的双卡分支。

## 结果

每个单元格是三次运行的**中位数 [最小值-最大值]**。九个完整配置中没有速度请求
失败、被取消或停滞。

### 单卡（PCI 0f），GFXOFF 默认

| | 提示词 token | 提示词 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 原始版 | 4,096 | 458.0 [451.4-458.8] | 47.5 [42.2-49.3] | 8.98 [8.97-9.11] | 14.33 [14.15-15.15] |
| 原始版 | 32,768 | 473.4 [472.9-473.7] | 46.9 [46.5-50.7] | 69.30 [69.26-69.36] | 74.77 [74.28-74.79] |
| 原始版 | 128,000 | 458.5 [458.3-458.9] | 48.3 [45.0-48.3] | 279.37 [279.10-279.51] | 284.77 [284.37-285.03] |
| 开关 | 4,096 | **810.9** [780.2-811.5] | 50.5 [45.1-54.9] | 5.09 [5.09-5.29] | 10.13 [9.72-10.94] |
| 开关 | 32,768 | **973.3** [971.9-974.2] | 48.5 [48.3-51.1] | 33.74 [33.71-33.79] | 38.99 [38.72-39.04] |
| 开关 | 128,000 | **912.8** [912.1-913.0] | 51.3 [49.8-52.7] | 140.42 [140.40-140.53] | 145.38 [145.23-145.63] |
| PRs | 4,096 | **969.4** [954.8-982.8] | 51.8 [44.6-54.2] | 4.26 [4.21-4.33] | 9.12 [8.96-10.04] |
| PRs | 32,768 | **1,152.4** [1,150.7-1,152.7] | 50.9 [50.1-54.3] | 28.51 [28.50-28.55] | 33.50 [33.20-33.63] |
| PRs | 128,000 | **1,125.6** [1,123.2-1,126.2] | 50.0 [49.2-50.1] | 113.92 [113.86-114.16] | 119.03 [119.01-119.24] |

### 跨双卡的层拆分，GFXOFF 禁用

| | 提示词 token | 提示词 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 原始版 | 4,096 | 446.5 [439.2-447.3] | 62.1 [61.9-70.0] | 9.21 [9.19-9.36] | 13.29 [12.84-13.48] |
| 原始版 | 32,768 | 723.4 [721.6-724.5] | 67.0 [61.1-69.4] | 45.37 [45.30-45.48] | 49.28 [48.96-49.53] |
| 原始版 | 128,000 | 821.2 [820.7-821.9] | 59.0 [58.2-59.1] | 156.06 [155.93-156.16] | 160.43 [160.24-160.46] |
| 开关 | 4,096 | **790.9** [752.2-793.7] | **70.4** [67.6-75.3] | 5.21 [5.20-5.48] | 8.81 [8.59-9.24] |
| 开关 | 32,768 | **1,443.3** [1,442.6-1,443.3] | **73.3** [69.1-75.4] | 22.77 [22.77-22.78] | 26.24 [26.15-26.46] |
| 开关 | 128,000 | **1,631.5** [1,629.7-1,634.1] | **64.7** [62.2-68.6