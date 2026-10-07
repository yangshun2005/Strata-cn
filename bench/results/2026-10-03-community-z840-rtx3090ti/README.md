# RTX 3090 Ti + 双路 Xeon E5-2699 v3（HP Z840）上的社区基准测试

由 pertain99 于 2026-10-03 测量。测试内容：在双路 Haswell-EP 工作站（仅 AVX2、PCIe 3.0、384 GB DDR4-2133）中的单块 RTX 3090 Ti 上，以 262,144 上下文运行 IQ3_S，外加三方 NUMA 放置对比和 `--calibrate` 扫描。主要局限：一个短提示词和一个约 5K 的提示词，贪婪解码，输出上限 512 个 token；除引擎自身的检查外没有质量测试套件。这是首个双路 Xeon / Z840 数据点（#508 中要求的）。

## 硬件与软件

- GPU：NVIDIA GeForce RTX 3090 Ti 24 GB，PCIe 3.0 x16（引擎探测主机到设备 12.4 GB/s），功耗限制设为 320 W（默认 450 W）。GPU 位于 NUMA 节点 0。
- CPU：2x Intel Xeon E5-2699 v3 @ 2.30 GHz（Haswell-EP，每颗 18C/36T，共 72 逻辑核心）。AVX2 + FMA，**无 AVX-512**。引擎在 AVX2 路径上运行其专家内核，使用 17 个池工作线程（仅节点 0，见 NUMA）。
- 内存：384 GB DDR4-2133 ECC（可用 378 GB），每路 4 通道，两个 NUMA 节点。
- 存储：Intel DC P4608 6.4 TB HHHL（PCIe 3.0 x8，两个 3.2 TB 控制器；一半为 XFS 用于模型文件，另一半为 swap），连接到 NUMA 节点 1。系统盘为 Samsung 870 QVO 2 TB SATA。
- 操作系统：Ubuntu 22.04.5 LTS，内核 6.8.0-138-generic（HWE）。NVIDIA 驱动 610.57.04（开源内核模块）。源码构建（nvcc）使用 CUDA toolkit 12.3，gcc 12.3。
- Strata：v0.1.38，2026-10-03 的 `main` 检出，源码构建（`CMAKE_CUDA_ARCHITECTURES=86`）。
- 后台：同一 GPU 上的 GNOME 桌面（Xorg + gnome-shell，空闲约 15 MiB 显存），dockerd，JupyterLab 空闲。专家 arena 位于 2 MB hugetlb 页（`vm.nr_hugepages=28000`，全部在节点 0），`LimitMEMLOCK=infinity`，`kernel.numa_balancing=0`。

## 模型与配置

- 模型：ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF，IQ3_S，由 setup 拉取时固定的修订版本（`Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`、`-00002-of-00002.gguf`），由 setup 构建的原生 pack。
- 视觉编码器在 GPU 上（`--vision`，mmproj BF16）；随附的专家配置 `data/expert-profile.bin`。
- 上下文 262,144；KV int8；KV 流式开启（`--kv-resident 32768`，KV 缓存在 RAM 中，3.6 GB）；专家缓存自动：加载视觉编码器时 **7,911 个专家 / 15.0 GiB 在显存中**（不加载时为 8,605 / 16.3 GiB）；专家 arena 46.84 GiB 在 RAM 中；`--prefill auto`；非低内存模式。
- MTP 开启（`--spec 4`）；所有测量均关闭思考（`reasoning_effort: none`）；贪婪解码（`temperature 0`）；校准运行一次（见下文）；实验性速度预测关闭。

```text
numactl --cpunodebind=0 --membind=0 engine/strata --serve \
  --pack /data/Strata-data/packs/iq3_s --native .../IQ3_S-00001-of-00002.gguf --ple-gguf .../IQ3_S-00002-of-00002.gguf \
  --expert-profile data/expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.70 \
  --mtp /data/Strata-data/mtp/rt --max-context 262144 --kv int8 --kv-resident 32768 \
  --vision --vram-reserve-mib 700 --pcie-frac 0.20
```

服务器：由 setup 写入的 `serve/server.py`，配置为 `anthropic_thinking: on_request`、`fit_max_tokens: true`、`reasoning_budget_tokens: 8192`。

## 方法

通过本地服务器上的 `POST /v1/chat/completions` 发起请求，贪婪解码，`reasoning_effort: none`，解码运行使用 `max_tokens` 512，长提示词运行使用 32。提示词和解码 tok/s 是引擎自身在服务器日志中的每请求数据（`prompt ... read in ... ms (N tok/s), ... generated in ... ms (N tok/s)`），而非总请求时间。

- **解码：** 一个固定的 31 token 中文编码提示词（"写一个带单元测试的 LRU 缓存类并逐行解释"），每次运行生成 512 个 token。启动后的第一个请求报告为冷启动（随附配置，无自适应）；从第二个请求起，提示词前缀被复用（31 个 token 中的 24 个），自适应专家层级已换入专家。
- **长提示词：** 一个约 5,000 token 的提示词（一段中英文混合段落重复 70 次），每次运行使用不同的开头句，使提示词缓存不匹配；生成 32 个 token。
- **NUMA A/B**（`strata-numa-ab.sh`，已附）：对每种放置方式，引擎均全新启动（停止 systemd 服务），重新分配 hugepages 以匹配（节点绑定时 28000/0，否则 14000/14000），2 次预热解码请求，然后 5 次测量解码运行和 5 次测量长提示词运行；报告中位数和范围。模型加载时间从服务启动到 `/health` 报告已加载。
- **校准：** 在节点 0 绑定下运行 `./setup.sh --calibrate`；其自身测量数据按原样引用（每轮冷缓存，校准期间不加载视觉编码器）。
- 内存：服务运行时 `MemTotal - MemAvailable` 约 64 GB（56 GB hugetlb 池，3.6 GB KV，引擎和服务器进程）。峰值显存 23.85 GB。

## 结果

解码，IQ3_S，节点 0 绑定，校准后设置（`--pcie-frac 0.20 --spec-min-p 0.70`）：

| 配置 | 实际提示词 token 数 | 复用 token 数 | 生成 token 数 | 运行次数 | 提示词 tok/s 中位数和范围 | 解码 tok/s 中位数和范围 | TTFT 秒中位数和范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 冷启动，第一个请求 | 31 | 0 | 512 | 1 | 54.2 | 67.1 | 0.57 |
| 热启动，请求 2-3（校准前） | 31 | 24 | 512 | 2 | 70.5-83.9 | 89.0 (87.0-91.1) | 0.08-0.10 |
| 热启动，A：`--cpunodebind=0 --membind=0` | 31 | 24 | 512 | 5 | 未测量 | 94.9 (93.7-95.9) | 未测量 |
| 热启动，B：`--interleave=all`（35 个工作线程） | 31 | 24 | 512 | 5 | 未测量 | 98.8 (95.9-100.6) | 未测量 |
| 热启动，C：无 numactl | 31 | 24 | 512 | 5 | 未测量 | 90.1 (88.2-93.4) | 未测量 |
| 长提示词，A | ~5,000 | 0 | 32 | 5 | 1,304 (1,296-1,308) | 未测量 | ~3.8 |
| 长提示词，B | ~5,000 | 0 | 32 | 5 | 1,239 (1,233-1,243) | 未测量 | ~4.0 |
| 长提示词，C | ~5,000 | 0 | 32 | 5 | 1,268 (1,262-1,272) | 未测量 | ~3.9 |

解码专家缓存命中率：冷启动第一个请求为 82.4%，热启动为 90.6-92.3%（三种 NUMA 放置方式彼此相差在 0.1 个百分点以内）。草稿接受率 77-85%。

真实 agent 流量（Claude Code / OpenClaw / Hermes 通过 Anthropic 和 OpenAI 端点，开启思考）：解码 56-80 tok/s，命中率 76-85%；一个 13,493 token 的 agent 提示词在 7 秒内读取完毕（约 1,900 tok/s）。

模型加载：从 NVMe 以 5.2 GiB/s 读取 46.84 GiB 专家数据，服务加载耗时 32 秒（节点 0 绑定）、36 秒（默认）、42 秒（interleave）。

`--calibrate` 扫描（其自身的冷缓存数据，关闭视觉，节点 0 绑定）：

| 设置 | 尝试的值（tok/s） | 保留值 |
| --- | --- | --- |
| `--pcie-frac` | 0.00: 63.1, **0.20: 65.6**, 0.34（探测自动值）: 64.8, 0.35: 65.5, 0.55（默认）: 58.3, 0.75: 51.8 | 0.20 |
| `--spec-min-p` | 0.30: 63.9, 0.50（默认）: 64.9, **0.70: 70.9** | 0.70 |
| `--pool-workers` | **17: 66.8**, 11: 62.1, 8: 57.0 | 17（默认 = 节点 0 物理核心数减 1） |

要点：在 PCIe 3.0 上，默认 0.55 的 PCIe 份额损失 11%；`--spec-min-p 0.70` 是最大的单项增益（+9%）；更多 CPU 工作线程一直有帮助，直到该路全部 17 个核心，但第二路的 18 个核心加内存交错仅带来 +4% 解码提升，却在提示词读取上损失 5%，因此保留了节点 0 绑定（这也让第二路保持空闲）。

每次运行的原始数据和 A/B 脚本在本文件夹中。

## 正确性与局限

- 模型正确回答了中文编码提示词（有效的 Python，基于 `OrderedDict` 的 LRU 带单元测试）；Claude Code 通过 Anthropic 端点端到端完成了文件创建任务。
- 未运行 needle 测试，也未运行质量测试套件；`tools/needle_bench.py` 是显而易见的后续工作。
- 单 GPU，一次一个请求，贪婪解码，速度运行关闭思考。
- NUMA 解码行的提示词和 TTFT 列未被脚本记录（仅解码）；长提示词行仅记录提示词速度。
- 解码数字随草稿接受率波动，因此不同提示词会使其变动几个百分点。
- Hugetlb 页放置对 NUMA 对比很重要：当全部 28,000 页都在节点 0 时，`--interleave=all` 运行会静默回退到节点 0 的页用于 arena；脚本在每次运行前重新分配它们。