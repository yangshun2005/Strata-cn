# 在 4x Tesla P100 16 GB（Pascal，sm_60）上的社区基准测试

由 apollo-mg 于 2026-10-06 测量。这是在 Pascal 上的实验性 CUDA 12 引擎，docs/OLDER_GPUS.md 将其列为 P100 的“未测量”项。
- **模型与设置：** 安装程序针对 GSQ-RCO IQ3_XXS 模型在全部四张卡上的配置。
- **测量内容：** 短提示词，约 4.4K 和约 36.4K token。
- **主要限制：** 单台机器，GPU 时钟锁定在 1063 MHz，且引擎使用 CUDA 12.4 构建，而非你测试所用的 12.8 / 12.9。

## 硬件与软件

- **GPU：** 4x Tesla P100-PCIE-16GB（计算能力 6.0），150 W 功耗限制。
  - 每次运行期间 SM 时钟均为 1063 MHz，来自每秒采样的 nvidia-smi。
- **CPU 与内存：** 2x Intel Xeon E5-2650 v3（AVX2，无 AVX-512）；128 GB DDR4-2133 ECC（可见 121 GB）。
- **存储：** Samsung SSD 860（SATA），本次会话期间大块顺序读取约 76 MB/s。这仅影响首次
  加载。
- **PCIe：** 未测量。
- **操作系统与驱动：** Ubuntu 26.04 LTS（内核 7.0.0-34-generic），NVIDIA 驱动 580.173.02。
- **CUDA：** 12.4（`/usr/bin/nvcc`，V12.4.131），使用 gcc-13 作为 CUDA 主机编译器。系统默认是 gcc 15，
  CUDA 12.4 不支持。
- **Strata：** v0.1.39（标签位于 `a1641e9`），由 setup 从源码构建。
  - 此设置没有现成的 CUDA 12 引擎（“no ready-made engine ... compiling instead”）。
  - 引擎：`engine-cuda12/strata`，由 setup 使用 `STRATA_NVCC=/usr/bin/nvcc CUDAHOSTCXX=/usr/bin/g++-13`、
    `--cuda 12` 构建。
- **后台工作负载：** 无。除基准测试外，机器处于空闲状态。

## 模型与配置

- **模型：** `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` IQ3_XXS，位于 setup 固定的修订版本（`ed59f92`）。
  - 分片 `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`（sha256 `219ea929...56d15`）和 `-00002`
    （`316b46f3...1e113`），均已验证。
  - 文件已在磁盘上，通过 `--gguf-dir` 传入。
- **视觉：** 关闭（`--vision no`）。
- **包与配置档：** 包和 MTP 草稿层（q2_0 专家）由 setup 构建，使用其默认
  专家配置档。
- **上下文与 KV：** 最大上下文 65536，`--kv int8`，KV 流式开启（`--kv-resident 32768`）。
- **专家缓存：** 自动，启用 `--remote-expert-opt`，层拆分跨 GPU 0-3（自动）。`--prefill auto`。
- **MTP 与采样：** `--spec 4 --spec-min-p 0.5`；推理强度 `none`；温度 0。
- **未启用：** 校准和实验性速度预测。

```text
./setup.sh --setup --family qwen --model IQ3_XXS --gguf-dir <folder with both shards> --gpus 0,1,2,3 --cuda 12 --vision no --yes
engine args (from strata-iq3_xxs.json):
--pack .../packs/iq3_xxs --native ...IQ3_XXS-00001-of-00002.gguf --ple-gguf ...IQ3_XXS-00002-of-00002.gguf
--expert-profile data/expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp .../mtp/rt --max-context 65536 --kv int8 --kv-resident 32768 --remote-expert-opt
```

## 方法

- **每个请求都是全新的：** 每个请求都以唯一的 nonce 行开头，因此不会复用任何前缀。引擎日志
  对每个请求都报告 `0 reused`。
- **请求设置：** 输出上限 256 token；流式；`reasoning_effort: none`（未产生推理 token）。
- **短提示词：** 两个 LocalMaxxing 规范提示词（`canonical_reasoning-v1.txt`、`canonical_code-v1.txt`），
  由 LocalMaxxing `lmx` v0.1.48 CLI 发送。
  - 每次服务器启动时，先发一个不计时的预热请求，然后发三个计时请求。
  - reasoning-v1 跨两次服务器启动，code-v1 跨一次。
- **长提示词：** Strata 自身的文档（测量提交处的 `docs/*.md` + `README.md`）外加一个问题，由
  `strata_bench.py` 构建。
  - 各运行三次，没有单独的预热；首次约 4.4K 运行是加载后的第一个请求。
- **计时：**
  - 提示词和解码吞吐量来自引擎自身的 `strata serve: prompt ... generated ...` 行，全部
    记录在 `runs.json` 中。
  - TTFT 为客户端侧：从发送请求到首个流式 token，即答案文本。
- **内存：** 加载后的快照（`free`、nvidia-smi）。

## 结果

| 配置 | 实际提示词 token | 复用 token | 生成 token | 运行次数 | 提示词 tok/s 中位数与范围 | 解码 tok/s 中位数与范围 | TTFT 秒中位数与范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| reasoning-v1（短） | 301-307 | 0 | 256 | 6 | 94.0 (92.9-94.7) | 33.5 (32.0-35.3) | 3.30 (3.27-3.34) |
| code-v1（短） | 239-242 | 0 | 256 | 3 | 86.1 (86.0-88.9) | 32.6 (32.2-33.2) | 2.83 (2.75-2.86) |
| Strata 文档，约 4.4K | 4441 | 0 | 256 | 3 | 239.4 (222.3-240.8) | 29.0 (27.5-29.4) | 18.63 (18.52-20.07) |
| Strata 文档，约 36.4K | 36356-36359 | 0 | 233-256 | 3 | 593.5 (583.5-594.9) | 28.0 (27.6-29.4) | 61.59 (61.38-62.58) |

- **MTP 草稿接受 / 起草：**
  - 短 reasoning 167-174 / 215-240；
  - 短 code 167-168 / 228-238；
  - 4.4K 135-146 / 208-226；
  - 36.4K 131-148 / 205-209。
- **解码专家缓存命中率：** 在 12 个短提示词请求中为 98.7-99.7 %。约 0 % 的路由专家经过
  PCIe；整个 IQ3_XXS 专家区域可容纳于四张卡中。
- **加载后内存：** 使用 49 GB 内存（可用 72 GB）；GPU 0-3 的 VRAM 分别为 13.1 / 13.4 / 15.5 / 15.8 GB。
- **两次 36.4K 运行在上限前结束**（233 和 235 token；引擎生成少于 256，因此模型自行停止；未记录结束原因）。
- **失败：** 无。每个请求均完成。

**仅供参考（不同引擎）：** 相同权重在 llama.cpp 上（fork 位于 `0b2789f23`，张量拆分，Flash-Next MTP 头草稿 3）解码两个短规范提示词的速度相近（中位数 33.4 tok/s，由同一 `lmx` CLI 测量）。

## 正确性与限制

- 每个答案都是连贯的文本，回应了提示词。答案开头在每次运行的数据中；一个短提示词
  示例以“### Part 1: Budget Calculation”开头。
- 未运行准确性检查、大海捞针测试、工具调用或图像测试。
- **未测试：**
  - 其他包，包括 IQ3_S 和 Unsloth 4-bit；
  - 超过约 36K 的提示词；
  - 并发；
  - 未锁定时钟；
  - CUDA 12.8 / 12.9 工具包。
- Pascal 模拟 `__dp4a`（docs/OLDER_GPUS.md）。本报告未单独区分该开销。

文件：`runs.json`（引擎计时行和表格行）、`client_timing_long_prompts.jsonl`、`strata_bench.py`，
以及两个规范提示词。

（测试由我和我的 agent Claude 设计并运行；我在发布前进行了审查。）