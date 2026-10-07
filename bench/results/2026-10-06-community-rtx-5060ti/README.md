# 社区基准测试：RTX 5060 Ti 16 GB，Ryzen 9 7945HX

测量于 2026-10-06（UTC），由 [avarakin](https://github.com/avarakin) 在本仓库的开发机器上完成，该机器是一台配备一块 NVIDIA GeForce RTX 5060 Ti 的 Linux 工作站。本次测试复现了 [RTX 5090 社区测试](../2026-09-30-community-rtx-5090/README.md) 的方法，使用相同的脚本、相同的合成提示词和相同的召回检查，但运行在不同的 GPU、不同的量化和更小的专家缓存上。

解码吞吐量中位数为 **4,096 提示词 token 时 56.6 tok/s，32,768 时 60.2 tok/s，128,000 时 57.2 tok/s**。在这些长度下，提示词吞吐量中位数分别为 **1,157.7、1,225.3 和 1,175.9 tok/s**。这些是合成代码解释请求，采用贪婪解码，输出上限为 256 token。它们不能代表通用回答质量或在其他工作负载上的性能。

## 硬件和软件

- NVIDIA GeForce RTX 5060 Ti；报告 VRAM 为 16,311 MiB；功耗限制 180 W。`nvidia-smi` 报告链路为 Gen 5，x8。GPU 频率未固定。采样期间观测到的 GPU 内存峰值为 15,702 MiB。
- AMD Ryzen 9 7945HX（含 Radeon Graphics）；32 个逻辑 CPU。引擎选择了 15 个专家池工作线程加上其宿主线程。
- 内存：`/proc/meminfo` 报告 `MemTotal` 为 60.53 GiB，`lsmem` 报告在线内存为 65.5 GiB；未测量已安装 DIMM 的容量（需要 root 权限）。交换空间：31.99 GiB。
- 存储：模型、pack 和 MTP 文件（`/data/ai/Strata-data` 下共 87 GB）位于 **Samsung 970 EVO Plus 2 TB NVMe 上的 ext4**（`/dev/nvme0n1p2`，挂载于 `/ssd`），通过 `/data` 处的 **mergerfs** 联合挂载访问（`/ssd/data:/data1`，选项 `cache.files=off,category.create=ff,func.getattr=newest`）。每个 Strata 文件都解析到 `/ssd/data` 分支；联合挂载的第二个分支（`/data1`，位于 HGST HUH721010ALE600 9.1 TB 7200 rpm 磁盘上的 ZFS 池 `data1`）不包含其中任何文件。引擎通过 `/data/...` 打开 GGUF 文件。交换空间是同一块 NVMe 上 LUKS 卷内 btrfs（`compress=zstd:3`）上的一个 32 GB 文件（`/swap/swapfile`）。
- Arch Linux，内核 7.2.8-arch1-1，glibc 2.44，NVIDIA 驱动 615.71.09。
- 源码提交 `d9ab8435f654c368c586340d490915f6addf56a3`（引擎 0.1.35），于 2026-10-02 在本地为 CUDA 架构 120 构建，包含 GPU 视觉辅助程序。参见 [BUILD.json](BUILD.json)。NVCC 13.4.92 来自 `/opt/cuda`。Python 3.14.7 位于 `.venv`。
- 原生构建，非容器。引擎和视觉辅助程序在运行前已运行约 25 小时；其他 CPU 服务也在运行。

## 模型和配置

模型：`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`，**IQ3_S**（社区测试使用的是 IQ2_XS）：

- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`（54,817,524,224 B）
- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf`（28,800,138,432 B）
- `mmproj-Qwen3.8-Flash-Next-BF16.gguf`

引擎在启动时以 3.11 GiB/s 的速度加载了 46.84 GiB。本次运行未重新校验哈希值，此处也未记录 Hugging Face 修订版本；磁盘上的 GGUF 文件日期为 2026-10-02。IQ3_S 原生 pack（`/data/ai/Strata-data/packs/iq3_s`，1.5 GB）和 Q2_0 MTP 草稿 pack（`/data/ai/Strata-data/mtp/rt`，787 MB）由安装脚本从这些分片准备；`packs/iq3_s/conversions.json` 列出了两个源分片及其字节大小。

完整的测量配置见 [strata-iq3_s.json](strata-iq3_s.json)。启动选项和每请求的引擎日志行见 [engine-excerpt.log](engine-excerpt.log)。

- 上下文 131,072；KV `q4_0`；无 KV 单元常驻（`kv_resident 0`）。
- 专家缓存 `auto`：**3,652 个槽位，6.98 GiB VRAM**，`PROFILE` 策略，预填充 3,652/3,652 个槽位，**无驱逐**。使用了捆绑的专家配置文件（`data/expert-profile.bin`，24,576 个排序对），未进行校准。16 GB 显卡在加载权重后剩余 7.88 GiB 空闲，全部加载后仍剩余 331 MiB。
- 预填充 `auto` 选择了 8,192 token 的分块，在提示词处理期间借用 2,331 个 CUDA0 专家缓存槽位（4.43 GiB）。
- MTP `--spec 4 --spec-min-p 0.70`；内置后缀草稿保持启用。
- `--pcie-frac 0.00`；低内存模式关闭；GPU 视觉启用，保留 700 MiB VRAM 和 1,024 图像 token。基准测试请求不包含图像。
- 推理关闭，温度 0，每次速度运行最多生成 256 token。

## 方法和复现

服务器是本机上已在运行的实例，启动方式为：

```bash
.venv/bin/python serve/server.py --engine strata --config strata-iq3_s.json \
  --port 8080 --open --host 0.0.0.0
```

它可在回环地址之外访问，且没有 API 密钥。它也是本次基准测试所使用的推理端点：驱动基准测试的 agent 在同一服务器上应答。参见 [局限性](#局限性)。基准测试本身与 `127.0.0.1:8080` 通信。

本目录中的 `benchmark.py` 是社区脚本，仅有一处改动，即 [脚本改动](#脚本改动) 中描述的引擎记录匹配修复。下面的运行从社区目录调用同一脚本，该目录当时包含此改动，此后已恢复为发布状态。运行方式为：

```bash
.venv/bin/python bench/results/2026-09-30-community-rtx-5090/benchmark.py \
  --root /data/ai/Strata --pack /data/ai/Strata-data/packs/iq3_s \
  --url http://127.0.0.1:8080 \
  --out bench/results/2026-10-06-community-rtx-5060ti \
  --targets 4096,32768,128000 --runs 3
```

（输出目录在运行期间命名为 `2026-10-06-local-rtx-5060ti`，之后重命名为发布名称。）

运行前检查了 token 计数路径：使用 IQ3_S 分词器对合成填充内容进行二分查找，精确达到 4,096、32,768 和 128,000 个渲染后的聊天 token。

排除了一个短的预热请求。在每个长度上串行执行三次运行，按长度递增顺序在同一已加载引擎上进行。全部九次速度请求都处理了完整提示词：**零复用 token**。专家缓存在启动时填充并在请求之间保留；运行之间没有引擎重启或页缓存重置。

本目录中的 `monitor.py` 是社区脚本，未改动；它从本目录以 `python ../2026-09-30-community-rtx-5090/monitor.py telemetry.jsonl` 运行，仅在召回阶段每秒采样一次——速度套件在采样开始前已运行完毕。[telemetry.jsonl](telemetry.jsonl) 包含 440.1 秒内的 431 个样本，[memory-summary.json](memory-summary.json) 包含峰值。

召回使用了仓库中未改动的 `tools/needle_bench.py`：

```bash
.venv/bin/python tools/needle_bench.py --url http://127.0.0.1:8080 \
  --lengths 32k,128k --depths 10,50,90 \
  --out bench/results/2026-10-06-community-rtx-5060ti/needles.json
```

## 结果

下表中每个单元格是三次运行的中位数 **[最小值–最大值]**。每个请求生成 256 token 并在输出上限处停止，没有失败或取消的速度请求。

| 提示词 token | 复用 | 提示词 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,157.7 [1,145.7–1,157.8] | 56.6 [47.5–58.5] | 3.573 [3.571–3.613] | 8.071 [7.920–8.971] |
| 32,768 | 0 | 1,225.3 [1,223.8–1,227.4] | 60.2 [58.9–60.3] | 26.811 [26.759–26.841] | 31.031 [30.987–31.157] |
| 128,000 | 0 | 1,175.9 [1,175.5–1,175.9] | 57.2 [55.6–59.7] | 109.024 [109.018–109.053] | 113.471 [113.316–113.595] |

原始逐次运行记录：[results.json](results.json)。计算出的聚合数据：[summary.json](summary.json)。逐次运行的请求体和完整流式分块日志保存为 `*-request.json` 和 `*-raw.json`。

这九次请求的解码专家缓存命中率为 **44.9–65.4%**。草稿接受为 163–180 个中接受 124–152 个。

## 召回阶段的内存

- GPU 内存峰值：所选 GPU 上全系统 **15,702 MiB**；431 个样本中有 420 个 GPU 利用率至少为 1%；峰值功耗 146.2 W，低于 180 W 限制。
- 主机 RAM 峰值：**55.26 GiB**，按 `MemTotal - MemAvailable` 计算。这包括 Strata、其他服务和不可回收的系统内存；不是进程 RSS。
- 采样开始时交换空间已使用 **3,874 MiB**（共 31.99 GiB），结束时为 **3,863 MiB**；在召回阶段没有增长。引擎在此之前已服务约 25 小时，因此起始值不是基准测试的产物。未测量交换 I/O 速率。未发生内存不足错误。

这些是采样峰值；样本之间的短暂分配尖峰可能被遗漏。

## 召回

未改动的 `tools/needle_bench.py` 找到了 **全部六根针**：在两个请求长度下深度 10%、50% 和 90%。实际提示词长度为 `32k` 时 31,459–31,460 token，`128k` 时 121,852–121,854。所有答案与预期代码词完全匹配。参见 [needles.json](needles.json)。

最后一个 128K 召回用例复用了 16,384 个提示词 token（103.7 s 对比 90.6 s 墙钟时间）；其他五个复用为零。因此召回延迟未用于全新提示词速度表。脚本按字符位置而非 token 位置选择针的深度。

## 脚本改动

`benchmark.py`（保存在本目录中）选择属于其自身请求的引擎记录，而不是取 `metrics["requests"][0]`：它查找提示词长度匹配且在该请求开始与流完成之间结束的最新记录，并记录 `concurrent_requests`，即该窗口内完成的其他请求数量。在本机上服务器还服务其他客户端，因此原始的 `requests[0]` 可能取到其他客户端的计时并引发错误的 token 计数不匹配。本次运行中唯一被标记的用例是脚本自身的预热请求，它在第一次 4K 运行前 0.9 秒开始，落在 `[start - 1 s, end + 2 s]` 窗口内：`concurrent_requests` 也计入脚本自身的请求，因此应将其视为上界，而非其他客户端的证据。

## 局限性

- **基准测试 agent 共享此服务器。** 引擎一次应答一个请求，扫描第一次 4,096 运行到最后一次 128,000 运行之间的每一行 `prompt … tokens`，只找到九行基准测试记录；六次召回运行同样如此。运行附近唯一的其他流量是 agent 自身的推理：三轮 27,735 / 27,890 / 28,074 token（生成 116 / 127 / 301，约 99% 的提示词被复用）在预热前约一秒结束，因为套件作为一条阻塞命令运行，而 agent 当时未在生成。因此被测请求没有与其他工作重叠，但确实紧接其后开始。第一次 4K 运行（解码 47.5 tok/s，总计 8.97 s）是低值离群点，直接紧随那轮 301 token 之后；未证实其原因，第 2 和第 3 次运行分别为 58.5 和 56.6 tok/s。
- **非隔离机器。** 其他 CPU 服务在运行，GPU 频率未固定，专家缓存在启动时已填充并被先前流量使用，引擎在本次运行前已服务 548 个请求。页缓存是热的。
- 这是一台机器、一种量化、一种配置和一个小型合成工作负载。未评估长输出、采样解码、思考、编码任务正确性、视觉、工具使用、多请求并发和持续热运行。
- 16 GB 显卡持有 **3,652** 个专家缓存槽位，而 RTX 5090 为 **17,463** 个，此处测得的解码命中率（44.9–65.4%）远低于 5090 运行报告的 97.8–99.7%。加上较弱的 GPU，这最可能是解码差距的原因，但未进行受控比较：量化（IQ3_S 对比 IQ2_XS）、KV 格式（`q4_0` 对比 INT8）、引擎版本（0.1.35 对比 0.1.29）、CPU、RAM、PCIe 宽度（x8 对比 x16）和构建也各不相同。两次运行都从 NVMe 读取模型。**不要将两个表格解读为仅 GPU 的对比。**
- 针检查衡量的是这六个输入上的召回，而非通用模型准确率。
- 128,000 token 的速度提示词并未填满 131,072 token 上下文窗口的每个 token。