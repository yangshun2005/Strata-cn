# 社区基准测试：2x NVIDIA GeForce RTX 2080 Ti，Ryzen 9 5950X

由 [cezartolvai](https://github.com/cezartolvai) 于 2026-10-06 在一台 Ubuntu 22.04 服务器上测量。这是针对 Qwen3.8-Flash-Next GSQ-RCO IQ3_XXS 的初步社区报告，配置为 131,072 token 上下文，并在两张 RTX 2080 Ti 显卡上自动进行层拆分。

对于 4,096 token 的提示词，吞吐量中位数为 **411.1 prompt tok/s 和 52.8 decode tok/s**；对于 32,768 token 的提示词，吞吐量中位数为 **477.2 prompt tok/s 和 42.5 decode tok/s**。每个数据点均为三次串行、全新提示词的请求，采用贪婪解码，输出上限为 256 token。本报告不涵盖 128K 提示词、召回率、散热、视觉、并发或长时间稳定性。

## 硬件与软件

- **使用的 GPU：** 2x NVIDIA GeForce RTX 2080 Ti，每张报告 22,528 MiB 显存；`nvidia-smi` 索引为 0 和 2。另有一张 RTX 3070 8 GB 存在但未被选用。配置的功耗限制为每张 GPU 100 W。负载下的 PCIe 带宽未测量。
- **CPU 与内存：** AMD Ryzen 9 5950X（16 核 / 32 逻辑 CPU，AVX2）；62 GiB 可用内存；无 swap。
- **存储：** 模型 GGUF 分片位于本地 ext4 卷；Strata 包和 MTP 草稿数据位于 NFS 存储。本次基准测试期间未发生模型下载。
- **操作系统与运行时：** Ubuntu 22.04.5 LTS，内核 6.8.0-138-generic；NVIDIA 驱动 580.178.04；CUDA 12.8。
- **源码与引擎：** Strata 提交 `82f46a8c8f475f001ad76d92f58f4a4f8ffb0253`；本地编译的引擎 0.1.40，针对 CUDA SM 75，源码 id `6f298dff754d187f`。
- **其他工作负载：** Strata 启动时不存在其他 GPU 计算进程。这并非完全隔离的主机。

## 模型与配置

- **模型：** `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`，原始 Qwen 系列，IQ3_XXS 原生 GGUF。本报告未独立验证上游修订版本或 SHA-256 哈希。
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`（47,039,860,096 字节）
  - `Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`（28,800,138,432 字节）
- **上下文与缓存：** 131,072 token 上下文，INT8 KV，KV 流式关闭，`--expert-cache auto`，`--prefill auto`。
- **多 GPU：** 自动层拆分，先 GPU 0 后 GPU 2，启用 `--remote-expert-opt`。
- **草稿：** 捆绑的 MTP 草稿层，`--spec 4 --spec-min-p 0.5`。
- **其他设置：** 视觉关闭；推理强度 `none`；温度 0；无校准、速度预测、控制向量或显式功耗/频率锁定。

## 方法

来自 `bench/results/2026-09-30-community-rtx-5090/` 的上游社区 `benchmark.py` 针对回环服务器运行：

```text
python benchmark.py --targets 4096,32768 --runs 3
```

它发送确定性的合成 Python 源码，每个填充符前带有不同的 nonce，使用 Strata 的分词器统计完整渲染的聊天提示词，流式传输每个响应，并记录引擎指标。一次短暂预热被排除。六次测量请求在一个已加载的引擎上串行运行；所有请求均以 **零复用 token** 处理完整提示词，并精确生成 256 token。加载时间不计入。

Prompt tok/s 是新处理的提示词 token 数除以引擎的 `prompt_ms`；decode tok/s 是 `engine_generated / decode_ms`。TTFT 和总时长为客户端回环测量值。原始请求、流式响应、引擎指标和汇总摘要均包含在本目录中。

## 结果

每个单元格为三次运行的中位数 **[最小值-最大值]**。

| 提示词 token 数 | 复用 | 生成 token 数 | Prompt tok/s | Decode tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 256 | 411.1 [404.0-413.9] | 52.8 [52.6-54.2] | 10.00 [9.94-10.19] | 14.84 [14.75-14.87] |
| 32,768 | 0 | 256 | 477.2 [435.0-574.0] | 42.5 [42.5-42.8] | 68.77 [57.19-75.41] | 74.74 [63.17-81.36] |

32K 提示词吞吐量的离散范围被保留而非取平均消除。没有请求失败或被取消。六次测量请求的解码专家缓存命中率为 98.0-99.7%。

## 局限性

- 这是单一主机和单一 IQ3_XXS 配置。
- 未测量 128K 提示词、`needle_bench.py`、视觉、工具使用、并发请求、热稳定性，以及与其他引擎的同工作负载对比。
- 测试套件期间未采样峰值 RAM/VRAM 遥测；启动时显示两张选定的 GPU 几乎满载，且主机立即可用内存有限。
- 模型来源仅通过文件名和大小标识；本报告未独立验证上游修订版本或 SHA-256。