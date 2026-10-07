# 社区基准测试：2x Tesla P40 22 GB（Pascal，sm_61），2x Xeon E5-2698 v4

由 [sarge18](https://github.com/sarge18) 于 2026-10-05 在一台 Linux 服务器上测量。本测试在 **面向 Pascal 的实验性 CUDA 12 引擎**（`-DSTRATA_EXPERIMENTAL_SM60=ON`，sm_61）上，以 32,768 token 上下文测试原始 Flash-Next IQ2_XS，分别在单块 P40 和**两块 P40 配合层拆分**（配置 `"gpu": [0, 1]`，而 `docs/MULTI_GPU.md` 将计算能力低于 7.5 的显卡列为不支持该配置）上各运行一次。本报告在维护者的指导下借助 AI 助手（Claude Sonnet 5.5，中等投入）编写；完整研究（还在相同权重上比较了 llama.cpp 和 Ollama）见 [sarge18/p40-llm-engine-bakeoff](https://github.com/sarge18/p40-llm-engine-bakeoff)。

短提示词的中位解码吞吐量为**单块 P40 上 19.6 tok/s，双块上 34.8 tok/s**。提示词读取速度在单卡上为 **348 / 388 tok/s**（4K / 16K 提示词 token），双卡上为 **353 / 501 tok/s**。在 32K 上下文下每个单元格仅运行三次；每种配置的首次运行包含专家缓存预热（见方法）。这些请求是合成的且为贪婪解码；它们不能确立通用回答质量或在其他工作负载上的性能。

## 硬件与软件

- **GPU：** 2x NVIDIA Tesla P40（GP102，24 GB 中可用 22.4 GiB，计算能力 6.1），被动散热，由机箱风扇驱动。功耗限制保持显卡默认值（观测到功耗最高 278 W）；温度达到 84 C（单卡）和 79 C（双卡）。PCIe 3.0 x16。引擎的 PCIe 探测测得 GPU 1 的主机到设备带宽为 11.8-11.9 GB/s，GPU 0 为 10.3 GB/s（见引擎日志）。GPU 0 位于 NUMA 节点 0，GPU 1 位于节点 1；单卡运行绑定到节点 1（`numactl --cpunodebind=1 --preferred=1`），双卡运行未绑定。
- **CPU 与内存：** 2x Intel Xeon E5-2698 v4（2x20 核，80 线程，AVX2，无 AVX-512；引擎使用了 19 个专家池工作线程加上其宿主线程）。153 GiB 内存（Linux MemTotal 160,445,020 kB）。存储：一个与其他用途共享的机械盘 ZFS RAIDZ2 池（仅影响冷加载时间）。8 GiB swap；服务器单元禁用了 swap，宿主机的 swap 使用保持平稳（约 0.4 GiB，来自其他进程）。
- **操作系统与运行时：** Ubuntu 26.04，内核 7.0.0-29-generic，glibc 2.43，NVIDIA 驱动 580.173.02，CUDA 工具包 12.4（Ubuntu 的 `nvidia-cuda-toolkit`）配合 g++ 13.4。
- **源码：** Strata v0.1.39，提交 `6f32ec070f23ced9f50e704d854d775da52591ab`（撰写时的当前 `main`），未修改。
- **构建：** 通过 setup 进行本地源码构建：`CXX=g++-13 CUDAHOSTCXX=g++-13 STRATA_NVCC=/usr/bin/nvcc ./setup.sh --family qwen --model IQ2_XS --context 32768 --vision no --cuda 12 --build --gpu 1 --yes --no-start --no-browser --data-dir <dir> --experimental-speed-projection off`。
  setup 使用 `-DSTRATA_EXPERIMENTAL_SM60=1` 为 `compute_61/sm_61` 编译了引擎。关于 Ubuntu 26.04 上所需的内容，见下文"构建说明"。
- **其他服务：** 机器的其他 GPU 服务（一个 Ollama 实例）在每次运行前停止，之后恢复，因此显卡仅运行 Strata。其他进程（一个文件服务器、一个空闲虚拟机、小型周期性定时器）继续运行，且一个服务在页缓存中锁定了约 22 GB 内存。根据定时器计划（而非日志）判断，运行期间没有到期的长时间运行计划任务；小型周期性内务定时器确实运行了。这不是一个完全隔离的操作系统。
  每次运行都在仅含回环（无互联网）的网络命名空间内启动，以非特权用户身份运行并设有硬性内存限制。

## 模型与配置

**模型：** [`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)，修订版本 `ed59f92082b1e93c0e96d60a8b11aab089b52f09`，IQ2_XS（setup 固定的修订版本）。哈希值是该修订版本下的 Hugging Face LFS SHA-256 值；
文件由 setup 下载，setup 仅检查其结构。它们未在本地重新哈希，且文件之后被删除：

- `Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf` 39,225,954,592 字节 `92cee27ae5bbadcd732416a0f7a7f0acc092399dbbe8f5a5efa707c2ec0a49d7`
- `Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf` 28,800,138,432 字节 `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113`

原生 pack 和 MTP 草稿层由 setup 准备（MTP 张量来自固定修订版本 `de4b8e4d43b917e7706784d8bb445c9af86a3540` 的 `Qwen/Qwen3.8-Flash-Next`）。专家配置为捆绑的 `data/expert-profile.bin`。

**配置**（单卡使用 [strata-iq2_xs.json](strata-iq2_xs.json)，双卡使用 [strata-iq2_xs-dual.json](strata-iq2_xs-dual.json)；`/m` 是模型目录，`/work` 是检出目录）：

- 上下文 32,768，`--kv int8`，专家缓存 `auto`，`--prefill auto`；
- MTP `--spec 4 --spec-min-p 0.5`（原生 IQ pack 要求 `--spec` 至少为 2，因此无法关闭推测）；
- 单卡：`"gpu": 1`；双卡：`"gpu": [0, 1]`，`"layer_split": "auto"`。引擎日志记录"layer split across 2 GPUs: CUDA0, then CUDA1 (split auto)"并选择 K=24（层 0-23 在第一块卡上，24-47 和 head 在第二块上），且"the caches hold 24576 of 24576 profiled pairs (~100.0% of the routed mass)"；
- 无视觉，无校准，无速度投影，无控制向量；推理关闭（`reasoning_effort: none` 和 `chat_template_kwargs.enable_thinking=false`）；温度 0，`top_k 1`，种子 1。

```text
# server, inside the benchmark's network namespace (cwd /work); one card:
python serve/server.py --engine strata --config strata-iq2_xs.json --port 8080
# both cards: --config strata-iq2_xs-dual.json
```

## 方法

[scripts/bench.py](scripts/bench.py)（标准库 Python；与研究仓库中相同的文件）发送 OpenAI 风格的流式 `/v1/chat/completions` 请求。速度任务为 `speed-short`（34 token 提示词，请求 400 token；实际生成恰好 400）和
`speed-prefill-4k` / `speed-prefill-16k`（4,000 / 15,920 token 的合成调查日志填充，请求 64 token）。提示词由脚本从固定种子确定性重新生成，因此无需提示词文件。每种配置
冷启动，然后三个任务按顺序运行三次（`--reps 3`），在同一个服务器进程上，不重启。各次重复的提示词完全相同，因此引擎的对话缓存本可复用 token；引擎计时值报告每次运行的 `cache_n = 0`（无复用），引擎日志也显示如此（"0 reused + N read"）。

- **提示词 tok/s** 和 **解码 tok/s** 是响应中返回的引擎自身计时值（`timings.prompt_per_second`、`timings.predicted_per_second`）。解码仅计生成的 token，不计总延迟。
- **TTFT** 和 **总延迟** 由客户端测量，从发送请求到首个生成 token / 流结束；它们包含引擎的提示词处理及任何排队（无排队：一次一个请求）。首个 token 是回答文本（推理关闭）。
- **预热：** 每种配置的首次重复较慢（专家缓存填充）：见逐次运行表。中位数为三次运行的中位数，括号内为范围。模型加载时间未计入任何速度数字。
- **内存：** 服务器单元的 cgroup 内存峰值（匿名内存加上计入它的页缓存）为 34.8 GB（单卡）和 35.8 GB（双卡）；swap 为 0。运行期间的峰值 VRAM（nvidia-smi，5 秒采样）单卡时为 GPU 1 上 22,601 MiB，双卡时为 20,487 MiB（GPU 0）和 22,377 MiB（GPU 1）。无分页或内存不足事件。
  `data/engine-logs/strata-iq2_xs.log` 和 `strata-iq2_xs-dual.log` 是该安装上每次运行的引擎累积日志（它们还包含一次冒烟测试和质量运行）。峰值为整个运行期间的值；遥测 CSV（每 5 秒：宿主机可用内存、swap、每 GPU 利用率、内存、温度、功耗）位于 `data/telemetry/`。

## 结果

3 次运行的中位数；括号内为范围。TTFT 和总延迟为客户端侧。

| 配置 | 实际提示词 token | 复用 token | 生成 token | 运行次数 | 提示词 tok/s 中位数（范围） | 解码 tok/s 中位数（范围） | TTFT 秒中位数（范围） |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 单块 P40（GPU1），34 token 提示词，生成 400 | 34 | 0 | 400 | 3 | 53 (16–53) | 19.6 (11.9–20.1) | 0.69 (0.68–2.30) |
| 单块 P40（GPU1），4,000 token 提示词，生成 64 | 4000 | 0 | 64 | 3 | 348 (124–349) | 16.1 (10.7–20.0) | 11.54 (11.54–32.24) |
| 单块 P40（GPU1），15,920 token 提示词，生成 64 | 15920 | 0 | 64 | 3 | 388 (314–389) | 23.7 (15.5–35.6) | 41.11 (41.10–50.75) |
| 双块 P40，34 token 提示词，生成 400 | 34 | 0 | 400 | 3 | 55 (15–55) | 34.8 (10.4–34.9) | 0.66 (0.66–2.39) |
| 双块 P40，4,000 token 提示词，生成 64 | 4000 | 0 | 64 | 3 | 353 (120–353) | 37.8 (11.9–37.9) | 11.40 (11.40–33.34) |
| 双块 P40，15,920 token 提示词，生成 64 | 15920 | 0 | 64 | 3 | 501 (376–501) | 37.2 (13.4–37.2) | 31.93 (31.90–42.47) |

逐次运行值（运行 1 / 运行 2 / 运行 3）：

| 配置 | 任务 | 提示词 tok/s | 解码 tok/s | 接受的草稿 / 草稿总数 |
| --- | --- | --- | --- | --- |
| 单块 P40（GPU1） | speed-short | 16 / 53 / 53 | 11.9 / 19.6 / 20.1 | 226 of 369 / 221 of 379 / 210 of 379 |
| 单块 P40（GPU1） | speed-prefill-4k | 124 / 348 / 349 | 10.7 / 20.0 / 16.1 | 30 of 47 / 31 of 47 / 34 of 55 |
| 单块 P40（GPU1） | speed-prefill-16k | 314 / 388 / 389 | 15.5 / 35.6 / 23.7 | 35 of 46 / 35 of 46 / 37 of 48 |
| 双块 P40 | speed-short | 15 / 55 / 55 | 10.4 / 34.8 / 34.9 | 208 of 390 / 208 of 390 / 208 of 390 |
| 双块 P40 | speed-prefill-4k | 120 / 353 / 353 | 11.9 / 37.8 / 37.9 | 35 of 48 / 35 of 48 / 35 of 48 |
| 双块 P40 | speed-prefill-16k | 376 / 501 / 501 | 13.4 / 37.2 / 37.2 | 37 of 49 / 37 of 49 / 37 of 49 |

每种配置的全部 9 个请求均以 `finish_reason` `length` 结束（达到了请求上限，符合定长速度请求的预期）；无一失败或被取消。未测量：32K 和 128K 提示词（此处上下文为 32,768）、视觉、其他量化、并发请求以及冷模型加载（一次首次从磁盘冷加载在单独的冒烟测试中耗时 254 秒；热加载为 33-76 秒）。
完整的逐次运行记录在 [results.json](results.json) 中，连同原始输出在 `data/`（`*.jsonl`）中；汇总统计在 [summary.json](summary.json) 中。

## 正确性与局限性

一个 69 项任务的正确性套件在单卡配置上运行了两次（相同提示词，贪婪解码，推理关闭）：带计算答案的推理 36 of 40 / 37 of 40（40 项任务），通过执行隐藏测试评分的代码任务 11 of 12 / 11 of 12（12），工具调用 8 of 8 / 8 of 8（8；其中两项是不得调用工具的克制用例），
完整性检查 4 of 6 / 5 of 6（6），以及每个提示词三个合成"访问码"的长上下文针检索（longctx-4k：4124/4124 token，召回 100%/100%；longctx-12k：12056/12056 token，召回 100%/100%；longctx-24k：23952/23952 token，召回 100%/100%）。对照原始输出查看的失败为模型错误（日期算术、单词反转、一个颠倒的罗马数字循环、一个停在 96 项的质数列表、要求 `##` 处却用了 `#` 标题）。
任务定义和评分器在 [scripts/bench.py](scripts/bench.py) 中（`python scripts/bench.py --selftest` 检查评分器对参考答案通过、对垃圾答案失败）；该套件存在天花板效应，无法对模型排名。这不是 `tools/needle_bench.py`。

**运行间可重复性：** 在温度 0 且提示词相同的情况下，69 个输出中有 49 个在两次运行之间逐字节相同（且有 2 项通过/失败发生变化），即在此配置上生成并非完全可重复。作为对比，同一台机器上一个 Ollama 配置的两次运行（不同模型）产生了相同的输出；见研究仓库。

局限性：一台机器，一天，合成工作负载；每个单元格的首次运行包含预热；双卡运行未做 NUMA 绑定，而单卡运行做了；GPU 0 和 GPU 1 与持有绑定工作线程的 CPU 的距离并不相同。

## 相关报告

- #875（v0.1.39 上的 P40 + RTX 3070 组合，IQ3_XXS，P40 功耗限制在 130 W，DDR4-2400，PCIe 3.0 x8）和 #395（IQ3_S 上的 P40，引擎 0.1.30）是其他 Pascal 测量。设置不同（量化、功耗限制、内存和 PCIe），因此数字不可直接比较；
  在本报告中，P40 上的解码速率在短生成中约在 10 到 36 tok/s