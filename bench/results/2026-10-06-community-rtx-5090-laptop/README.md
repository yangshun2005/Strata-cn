# 社区基准测试：RTX 5090 Laptop GPU（24 GB），Ryzen 9 9955HX3D

由 [wolffahrer](https://github.com/wolffahrer) 于 2026-10-06 在一台 Windows 11 游戏笔记本（Schenker XMG NEO 16，A25）上测得。本测试针对 Strata 0.1.40.1（引擎 0.1.40），使用 Flash-Next **Q2_0**、单块移动 GPU，以及 131,072 token 的上下文限制。它复用了 [2026-09-30 桌面 RTX 5090 报告](../2026-09-30-community-rtx-5090/README.md) 的工作负载（相同的合成提示词、提示词长度、贪婪解码、关闭推理、256 token 输出上限），以便两者可以并排阅读。这**并非**受控对比：量化方式、引擎版本、操作系统、VRAM、功耗限制和 PCIe 链路均不相同（见下文）。

中位解码吞吐量为 **4,096 提示词 token 时 136.9 tok/s，32,768 时 137.7 tok/s，128,000 时 134.6 tok/s**。中位提示词吞吐量为 1,952.5、2,877.8 和 2,764.1 tok/s。全部九次速度请求均成功，且零复用 token。这些是合成的代码解释请求；它们不能确立通用的回答质量。

## 硬件与软件

- NVIDIA GeForce RTX 5090 **Laptop** GPU；报告 VRAM 为 24,463 MiB。功耗：电流限制 175.00 W（默认 95.00 W，最大 175.00 W，笔记本 Dynamic Boost）；运行期间采样功耗中位数 162.6 W，峰值采样 191.8 W。**PCIe 链路：负载下为 Gen 4、x8**（空闲时为 Gen 2 x8；`nvidia-smi` 报告最大宽度为 x16）。引擎的启动传输探测未被记录。GPU 时钟未固定。
- AMD Ryzen 9 9955HX3D，16 核 / 32 逻辑 CPU。本次运行使用 `--pool-workers 10`（10 个专家池工作线程加上核心 0 上的宿主线程），该值由本地手动扫描选定。
- 已安装 64 GB RAM（2 x 32 GB DDR5-5600）；Windows 报告可用 61.68 GiB。
- 存储：模型文件和 pack 位于 Samsung SSD 990 PRO 1TB（NVMe）；操作系统位于 WD Blue SN5100 500GB（NVMe）。
- Windows 11 Home，build 26200。NVIDIA 驱动 610.88（CUDA UMD 13.3）。
- Strata **v0.1.40.1**，标签提交 `82f46a8c8f475f001ad76d92f58f4a4f8ffb0253`，从发布源代码归档构建（不是 git 检出）。引擎 0.1.40，使用 CUDA toolkit 13.3（nvcc 13.3.73）为 CUDA 架构 120 本地编译，包含 GPU 视觉辅助程序。参见 [BUILD.json](BUILD.json)。
- 后台：一个 WSL2 VM（内存上限 32 GB）保持空闲的本地 agent 栈运行；常规本地推理服务器已为本次运行停止，因此 GPU 专用于 Strata 及其视觉辅助程序。并非隔离的操作系统。

## 模型与配置

- 模型：`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`，**Q2_0**：`Q2_0/Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00001-of-00002.gguf`、`Q2_0/Qwen3.8-Flash-Next-GSQ-RCO-Q2_0-00002-of-00002.gguf`、`mmproj-Qwen3.8-Flash-Next-BF16.gguf`。仓库修订版本和文件哈希：本报告**未测量**。（桌面报告使用 IQ2_XS。）
- Pack 和 MTP 草稿专家由未修改的安装程序（`setup.py --build`）准备；捆绑的专家配置文件。
- 上下文 131,072；INT8 KV；每个注意力层 32,768 个 KV 单元常驻 GPU（K/V 位于 1.55 GiB 固定内存中）。
- 专家缓存 `auto`：**11,742 个槽位，15.12 GiB VRAM**（桌面为 17,463 个槽位 / 23.44 GiB），配置文件预填充，无驱逐。`--vram-reserve-mib 1000`；全部加载后剩余 497 MiB VRAM。`--mmap-experts`。
- 预填充 `auto`：8,192 token 分块，在提示词处理期间借用 2,082 个专家缓存槽位（2.68 GiB）。
- MTP `--spec 4 --spec-min-p 0.7`（桌面：0.5）和 `--pcie-frac 0`，均来自本地手动扫描；启用内置后缀草稿。无内置校准运行，实验性速度预测关闭，低内存模式关闭。
- GPU 视觉已启用（1,024 图像 token）；基准测试请求不包含图像。
- 已开启但不影响这些请求的选项：对话缓存 `--conversation-cache-mib 6144 --conversation-cache-slots 2`、`reasoning_budget_tokens: 12000`、`reasoning_loop_recovery: "stop"`。
- 速度运行：关闭推理（`reasoning_effort: none`），温度 0，最多生成 256 token。

完整配置（本地路径已替换为占位符）见 [strata-q2_0.json](strata-q2_0.json)；启动选择以及每个请求的计时和草稿统计见 [engine.log](engine.log)。

```text
.venv\Scripts\python.exe serve\server.py --engine strata --config strata-q2_0.json --port 18096
```

## 方法

[benchmark.py](benchmark.py) 是桌面报告的脚本，有三处改动：它使用 Strata 自带的 Windows venv 运行；失败或取消的请求会连同其错误一起记录，并从统计中排除，而不是停止整个测试套件；在每个请求期间每 0.5 秒采样一次峰值宿主 RAM（系统范围内总量减去可用量）和 GPU 内存。提示词、nonce、token 目标以及与服务器进行的 token 计数检查均未改变。

```text
.venv\Scripts\python.exe benchmark.py --root <strata> --pack <strata-data>\packs\q2_0 --url http://127.0.0.1:18096 --out results
.venv\Scripts\python.exe tools\needle_bench.py --url http://127.0.0.1:18096 --lengths 32k,128k --depths 10,50,90 --out needles.json
.venv\Scripts\python.exe coding_check.py --url http://127.0.0.1:18096 --out coding-check.json
```

一次简短的热身请求被排除。每个长度三次运行在同一已加载引擎上按长度递增顺序串行执行。全部九次速度请求都处理了其完整提示词（**零复用 token**）。专家缓存在启动时填充，并在请求之间保持；没有重启或页缓存重置将它们分隔开。加载时间被排除（服务器在启动后 15.1 秒即健康）。吞吐量仅来自引擎的计时值：提示词 tok/s = 新读取的 token / `prompt_ms`，解码 tok/s = `engine_generated / decode_ms`。TTFT 是客户端流式传输到第一个非空文本增量的时间（答案文本，推理关闭）；总计是直到流结束。两者都包含通过回环的 HTTP 和前端 tokenization。

## 结果

三次运行的中位数 **[最小值–最大值]**。每个请求生成 256 token 并在输出限制处停止；没有失败或取消的请求。

| 提示词 token | 复用 | 提示词 tok/s | 解码 tok/s | TTFT 秒 | 总计秒 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 1,952.5 [1,746.8–1,955.5] | 136.9 [125.0–137.4] | 2.124 [2.117–2.376] | 3.977 [3.977–4.430] |
| 32,768 | 0 | 2,877.8 [2,867.3–2,900.1] | 137.7 [134.2–140.0] | 11.458 [11.367–11.509] | 13.352 [13.184–13.357] |
| 128,000 | 0 | 2,764.1 [2,755.1–2,797.3] | 134.6 [122.1–141.9] | 46.498 [45.973–46.666] | 48.289 [47.863–48.751] |

原始逐次运行记录：[results.json](results.json)（引擎计时、草稿统计、输出文本、请求哈希、每请求内存峰值）。汇总：[summary.json](summary.json)。九次请求的解码专家缓存命中率为 90.2–98.5%（首次 4K 运行也拥有最低的解码速度，保留在该范围内）。

**与桌面报告并排对比**（并非受控对比）：解码达到桌面中位数（179.4 / 175.7 / 165.0 tok/s）的 76%（4K）、78%（32K）和 82%（128K）；提示词吞吐量达到 46%、52% 和 48%（4,269.8 / 5,543.2 / 5,778.7 tok/s）。可能的贡献因素，此处均未单独隔离：175 W vs 575 W 功耗限制，PCIe Gen 4 x8 vs Gen 5 x16，24 vs 32 GB VRAM（15.1 vs 23.4 GiB 专家缓存），Q2_0 vs IQ2_XS，Windows vs Linux，引擎 0.1.40 vs 0.1.29，`--spec-min-p` 0.7 vs 0.5。

内存，来自 [monitor.py](monitor.py) 从模型加载前到检查结束每秒采样一次（[telemetry.jsonl](telemetry.jsonl)，397.5 秒内 369 个样本，汇总见 [memory-summary.json](memory-summary.json)）：

- GPU 内存：**峰值 23,219 MiB**（22.67 GiB），单 GPU 系统范围内。
- 使用中的宿主 RAM（系统范围内总量减去可用量，包括 WSL2 VM 和其他进程；不是进程 RSS）：加载前 23.63 GiB，**峰值 55.59 GiB**。
- 使用中的交换/页面文件：0.64 GiB，运行期间未变化。未测量缺页率。无内存不足错误。

## 正确性与局限性

- **长上下文召回：** 未修改的 `tools/needle_bench.py` 找到了**全部六根针**（`32k` 和 `128k` 下深度 10/50/90%）。实际提示词长度为 32,308–32,309 和 125,703–125,705 token。128K 案例中的两个和一个 32K 案例复用了 16,384 个提示词 token；因此召回延迟不属于速度表的一部分。参见 [needles.json](needles.json)。
- **编码检查：** [coding_check.py](coding_check.py) 要求实现带错误处理的 `roman_to_int(s)`，并对返回的代码运行固定的本地测试套件：**10/10 测试通过**（7 个转换，3 个预期的 `ValueError`），推理强度 `low`，生成 1,160 token，10.6 秒。参见 [coding-check.json](coding-check.json)。
- 未评估：长输出、采样解码、思考速度、视觉、工具使用、并发、持续热运行。笔记本 GPU 的时钟会随温度和电源状态变化；本次短运行中 GPU 温度峰值为 83 °C。128,000 token 的提示词并未填满完整的 131,072 token 窗口。

作者在 KI SOUVERÄN 频道（德语）记录消费级硬件上的本地 AI：[YouTube](https://www.youtube.com/channel/UCE6Ch6g6Bo8v4ROpYDzOOxA) 和 [Telegram](https://t.me/lokale_ki)。