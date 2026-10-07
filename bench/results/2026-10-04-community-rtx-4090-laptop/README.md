# 社区基准测试：RTX 4090 Laptop GPU，Core i9-13900HX

由 [30crows](https://github.com/30crows) 于 2026-10-04 在一台 Lenovo Legion 笔记本（型号 82WQ）上测得。本测试针对 Strata 0.1.38，使用原始 Flash-Next IQ3_S 和单块 GPU，上下文限制为 131,072 个 token，并在第二次会话中测试了 262,144 个 token 的限制。

在 131,072 个 token 限制下，解码吞吐量中位数为 **4,096 个提示 token 时 51.2 tok/s，32,768 时 49.9 tok/s，128,000 时 48.1 tok/s**；提示吞吐量中位数为 1,345 / 2,506 / 2,386 tok/s。在 262,144 个 token 限制下，**250,000 个 token 的提示以 2,116 tok/s 读取（首 token 耗时 118.5 秒），解码速度为 40.3 tok/s**；在较短长度下，解码速度比 131,072 个 token 限制时慢 13-17%（[见下文](#262144-token-context)）。这些是使用贪心解码和 256 个 token 输出上限的合成代码解释请求。它们不能代表一般答案质量或其他工作负载下的性能。

## 硬件和软件

- NVIDIA GeForce RTX 4090 Laptop GPU (AD103)；报告显存为 16,376 MiB。nvidia-smi 报告当前和最大功耗限制为 175 W（默认 80 W），`nvidia-powerd`（Dynamic Boost）处于活动状态。PCIe 能力：Gen 4，x16。引擎的启动传输探测报告主机到设备速度为 19.3 GB/s。GPU 频率未固定。
- Intel Core i9-13900HX；32 个逻辑 CPU。无 AVX-512：引擎使用 AVX2，配备 23 个专家池工作线程及其主机线程。来自 Lenovo 固件属性的 CPU 限制：PL1 125 W，PL2 190 W。
- 已安装 64 GB 内存；Linux 报告可用 61.28 GiB。存储：WD_BLACK SN850X 2 TB NVMe（模型文件和数据目录）。512 GiB 交换文件。
- 平台配置文件 `performance`（`lenovo-wmi-gamezone` 驱动还提供 `max-power` 和 `custom`；未使用）。
- Ubuntu 26.04.1 LTS，内核 7.0.0-38-generic，NVIDIA 驱动 610.57.04（报告 CUDA 13.3）。
- 源码提交 `99f3dbd0b21d1401b3769e0c0d963913607f380b`；引擎 0.1.38。
  **源码构建：** setup 未找到适用于 v0.1.38 的现成引擎（发布版本下载返回 HTTP 404），因此使用 CUDA Toolkit 13.4（NVCC V13.4.92）和 GCC 15.2.0 为 CUDA 架构 89 在本地编译了一个。Python 3.14.4。引擎二进制文件和专家配置文件的哈希值见 [artifact-hashes.json](artifact-hashes.json)。
- 运行期间，Firefox 和其他大型应用程序已关闭；桌面会话保持运行。机器未做其他隔离。

## 模型和配置

模型：`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`，修订版本 `ed59f92082b1e93c0e96d60a8b11aab089b52f09`（setup 固定的修订版本）：

- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`（54,817,524,224 字节）
- `IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf`（28,800,138,432 字节）

这些文件由未经修改的安装程序下载并校验；本报告未再次计算哈希值。MTP 草稿层由安装程序从 `Qwen/Qwen3.8-Flash-Next` 获取。无视觉编码器。

Setup 命令：

```bash
./setup.sh --yes --family qwen --model IQ3_S --context 131072 --kv int8 --vision no --no-start
```

- 上下文 131,072；INT8 KV。启动参数中不包含 `--kv-resident`，因此 KV 流式传输处于关闭状态（setup 针对此内存大小的选择）。
- 专家缓存 `auto`：3,865 个槽位，7.38 GiB 显存，配置文件预填充，无驱逐。使用了捆绑的专家配置文件，未进行校准。
- 预填充 `auto` 选择了 8,192 个 token 的块，在提示处理期间借用 2,333 个专家缓存槽位（4.43 GiB）。
- 46.84 GiB 的专家数据以 4.96 GiB/s 的速度加载到内存；4 KB 页面（无大页）。
- MTP `--spec 4 --spec-min-p 0.5`。
- 低内存模式关闭；视觉关闭；实验性速度预测关闭。
- 推理禁用，温度 0，每次速度运行最多生成 256 个 token。

完整配置见 [strata-iq3_s.json](strata-iq3_s.json)。测量会话的引擎日志（从引擎启动到最后一次召回检查）见 [engine.log](engine.log)。服务器使用 `BROWSER=true ./run-iq3_s.sh` 启动，因此不会打开浏览器。

## 方法和复现

基准测试和监控脚本来自 [RTX 5090 报告](../2026-09-30-community-rtx-5090/README.md)，未做修改，从仓库根目录针对已安装的服务器运行：

```bash
B=bench/results/2026-09-30-community-rtx-5090
.venv/bin/python $B/monitor.py ~/strata-bench/telemetry.jsonl &
.venv/bin/python $B/benchmark.py --root . --pack ../Strata-data/packs/iq3_s \
  --url http://127.0.0.1:8080 --out ~/strata-bench
kill %1
.venv/bin/python tools/needle_bench.py --url http://127.0.0.1:8080 \
  --lengths 32k,128k --depths 10,50,90 --out ~/strata-bench/needles.json
```

在基准测试之前，同一个已加载的引擎回答了一个 1,000 个 token 的请求（功耗检查，不属于结果的一部分）。随后排除了一个短的预热请求。每个长度执行三次运行，按长度递增顺序串行执行。所有九个速度请求都处理了其完整提示：**零复用 token**。GPU 专家缓存在启动时填充并在请求之间保持；加载时间不计入。

TTFT 和总延迟由客户端通过回环测量。引擎提示吞吐量为新读取的 token 数除以 `prompt_ms`；解码吞吐量为 `engine_generated / decode_ms`。

## 结果

每个单元格是三次运行的中位数 **[最小值–最大值]**。每个请求生成 256 个 token 并在输出限制处停止，没有失败或取消的请求。

| 提示 token 数 | 复用 | 提示 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,345.0 [1,280.0–1,367.7] | 51.2 [42.7–53.9] | 3.084 [3.032–3.235] | 8.004 [7.802–9.205] |
| 32,768 | 0 | 2,505.5 [2,496.1–2,511.8] | 49.9 [49.0–53.1] | 13.150 [13.135–13.201] | 18.305 [17.919–18.345] |
| 128,000 | 0 | 2,385.8 [2,376.5–2,412.7] | 48.1 [47.6–48.8] | 53.818 [53.217–54.043] | 59.164 [58.432–59.333] |

每次运行的原始记录：[results.json](results.json)；聚合数据：[summary.json](summary.json)。解码专家缓存命中率为 68.2–80.3%，模型在每个请求中接受了 203–234 个 MTP 草稿中的 136–162 个。最低的解码运行（42.7 tok/s）是第一次 4K 运行，其命中率也最低（68.2%）。

[telemetry.jsonl](telemetry.jsonl) 包含 267 个样本，覆盖 273.5 秒，仅涵盖速度测试套件（不包括召回检查）。观察到的峰值（[memory-summary.json](memory-summary.json)）：

- GPU 内存：系统范围内 **15,698 MiB**（共 16,376 MiB）。
- 主机内存：系统范围内已用 **54.74 GiB**（`MemTotal - MemAvailable`，非进程 RSS）；MemAvailable 从未低于 6.55 GiB。
- 交换：启动时和峰值时分配了 698.5 MiB（运行期间无增长）。交换 I/O 未测量。
- GPU 功耗：在利用率超过 50% 的 250 个样本中，中位数为 **170 W**，峰值为 176 W；峰值温度 86 °C。提示处理在功耗限制下运行。

同一会话的单独功耗检查（一个 1,000 个 token 的回答，25 个 token 的提示，不属于表格的一部分）在 39 个忙碌样本中解码速度为 57.0 tok/s，GPU 功耗中位数为 116 W（峰值 141 W）：解码未达到功耗限制。

## 召回

未修改的 `tools/needle_bench.py` 在 10%、50% 和 90% 深度处找到了**全部六根针**。`32k` 的实际提示长度为 31,969–31,970 个 token，`128k` 为 122,753–122,755。参见 [needles.json](needles.json)。每个长度的第三个用例复用了 16,384 个提示 token（引擎 `/metrics`）；其他用例复用为零。

## 262,144-token 上下文

同一天晚些时候（重启后）在同一台机器、提交、引擎构建、模型文件和电源模式下测量。仅上下文发生变化：

```bash
./setup.sh --setup --yes --family qwen --model IQ3_S --context 262144 --kv int8 --vision no --no-start
```

生成的配置与 131,072-token 配置的唯一区别是 `--max-context 262144`（[context-262144/strata-iq3_s.json](context-262144/strata-iq3_s.json)）。KV 流式传输保持关闭，因此更大的 INT8 KV 缓存驻留在显存中，专家缓存从 3,865 个槽位（7.38 GiB）缩减到 **2,856 个槽位（5.47 GiB）**。预填充仍使用 8,192 个 token 的块并借用 2,319 个槽位（4.43 GiB）。PCIe 探测报告 20.6 GB/s。此会话的引擎日志：[context-262144/engine.log](context-262144/engine.log)。

相同的基准测试脚本使用 `--targets 4096,32768,128000,250000` 运行，监控程序同时运行。预热之前没有其他请求。所有十二个速度请求都读取了其完整提示（**零复用 token**），生成 256 个 token 并在输出限制处停止。

| 提示 token 数 | 复用 | 提示 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,343.0 [1,266.5–1,397.2] | 43.4 [37.4–44.1] | 3.097 [2.973–3.278] | 9.056 [8.961–9.783] |
| 32,768 | 0 | 2,457.5 [2,434.0–2,469.2] | 43.3 [43.0–43.3] | 13.412 [13.349–13.539] | 19.295 [19.228–19.457] |
| 128,000 | 0 | 2,291.6 [2,290.5–2,327.3] | 39.7 [39.5–42.6] | 56.027 [55.166–56.054] | 62.031 [61.612–62.448] |
| 250,000 | 0 | 2,116.4 [2,115.7–2,132.3] | 40.3 [40.2–41.0] | 118.457 [117.552–118.483] | 124.661 [123.891–124.798] |

原始记录：[context-262144/results.json](context-262144/results.json)；聚合数据：[context-262144/summary.json](context-262144/summary.json)。解码专家缓存命中率为 67.1–74.6%（131,072-token 限制下为 68.2–80.3%），模型在每个请求中接受了 203–246 个 MTP 草稿中的 134–155 个。与 131,072-token 配置相比，解码中位数在 4,096 个 token 时低 15%，32,768 时低 13%，128,000 时低 17%；提示吞吐量差异在 4% 以内。

遥测数据（[context-262144/telemetry.jsonl](context-262144/telemetry.jsonl)，662 个样本，覆盖 679.5 秒，仅速度套件）显示峰值为 **15,718 MiB** GPU 内存和 **55.11 GiB** 主机内存使用；MemAvailable 从未低于 6.17 GiB，交换保持在 0.7 MiB。在利用率超过 50% 的 626 个样本中，GPU 功耗中位数为 **172 W**，一个样本为 184 W，峰值温度为 89 °C（[context-262144/memory-summary.json](context-262144/memory-summary.json)）。

针测试使用 `--lengths 32k,128k,256k` 运行，找到了**全部九根针**。`32k` 的实际提示长度为 32,034–32,035 个 token，`128k` 为 122,798–122,800，`256k` 为 249,309–249,310（[context-262144/needles.json](context-262144/needles.json)）。所有三个 `256k` 用例以及第三个 `32k` 和 `128k` 用例复用了 16,384 个提示 token；其他用例复用为零。

## 局限性

这是一台机器、一种量化、两个上下文限制和一个小型合成工作负载。未评估长输出、采样解码、思考、编码任务正确性、工具使用和持续热运行。笔记本的功耗和热行为取决于其电源模式、散热和环境温度；未测试其他电源模式。README 中其他机器的数据不是受控比较。