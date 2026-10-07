# 社区基准测试：AMD Radeon RX 6700 XT 12 GB (gfx1031)，Ryzen 7 5700G

测量于 2026-10-05（MST 17:45-18:40，即 2026-10-06 UTC 00:45-01:40），由 [GPF](https://github.com/GPF) 在一台 Linux 台式机上完成。
本测试在单张 RDNA2 显卡上运行原始 Flash-Next IQ2_XS，上下文长度为 131,072 个 token，该显卡在 setup 中被列为
未验证（gfx1031，#524），运行在基于 ROCm 7.2.4 构建的 Docker 容器中，并让 HIP 运行时将该显卡视为
gfx1030（`HSA_OVERRIDE_GFX_VERSION=10.3.0`）。

解码吞吐量中位数为 **4,096 个提示 token 时 32.1 tok/s，32,768 时 30.7 tok/s，128,000 时 30.2 tok/s**。提示读取速度分别为 302、309 和 298 tok/s。全部六项大海捞针召回检查（32K 和 128K，三个深度）均找到了码字。速度请求是合成的代码解释请求，采用贪婪解码，输出上限为 256 个 token。它们不能证明通用回答质量或其他工作负载下的性能。

## 硬件与软件

- **GPU：** AMD Radeon RX 6700 XT，12 GB（报告为 11.98 GiB），gfx1031（wave32）。由于上述覆盖设置，HIP 运行时将其视为 gfx1030。功耗上限 186 W（从 hwmon 读取）。运行期间
  显卡处于 99% 负载，功耗中位数 173 W（最大 188 W），温度中位数 82 C（最高 85-86 C）。PCIe 链路按 sysfs 报告，
  在运行结束后空闲时读取：16.0 GT/s x16（最大 16.0 GT/s）。负载下的链路速度未检查。
- **CPU 与内存：** AMD Ryzen 7 5700G（8 核，16 线程，AVX2，无 AVX-512）；引擎使用了 7 个专家池工作线程
  加上其宿主线程。已安装 64 GB（Linux 报告可用 62 GiB）；内存速度和通道布局未
  记录。8 GiB 交换空间，运行前和运行期间约使用了 4 GiB。
- **存储：** 模型文件位于一块 4 TB SATA 硬盘上（WDC WD40EZAZ，机械盘，通过 ntfs3 挂载 NTFS；
  使用 `O_DIRECT` 的 `dd` 测得约 175 MB/s）。第二个 GGUF 分片（包含 28.8 GB 的 n-gram 表）是
  一块 1 TB NVMe SSD（T-FORCE TM8FPL1000G）上的副本，并通过 bind mount 覆盖容器中的对应路径。将其保留在硬盘上
  会导致提示停滞（#605）：停滞报告称 35 个引擎线程中有 16 个在等待磁盘，看门狗停止了
  引擎。每次启动时从硬盘加载专家约需 5-6 分钟（已从以下所有
  计时中排除）。
- **操作系统与运行时：** Ubuntu 24.04.5 LTS，内核 7.0.0-34-generic，内核自带的 amdgpu 驱动，Docker 29.8.2。ROCm
  7.2.4 来自镜像 `rocm/dev-ubuntu-24.04:7.2.4-complete`。ROCm 7.0 的 clang 在
  `src/kernels/cuda/iq_kernels.cu:1039` 和 `:1107` 处拒绝 `__shared__ alignas(16)`，因此 7.0 镜像无法构建。
- **源码：** `main` 分支的 6f32ec0，引擎 0.1.39，未修改。容器基于新的 `Dockerfile.hip` 构建（不在本
  Pull Request 中）。它使用系统 ROCm 为 `gfx1030;gfx1031` 编译引擎（`-DSTRATA_ENABLE_HIP=ON
  -DSTRATA_PREFILL_MMQ=ON`，llama.cpp 使用固定的提交），并写入 `engine/BUILD.json`（[BUILD.json](BUILD.json)）。
- **其他服务：** 一个普通的桌面会话。在运行前约三小时使用此服务器的编码代理（pi）在运行期间处于空闲且未连接状态。这不是一台隔离的机器。

## 模型与配置

**模型：** 来自
[`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)
的原始 Qwen3.8-Flash-Next，IQ2_XS，由 setup 下载。仓库修订版本未记录。所用分片的 SHA-256：

- `Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf` `92cee27ae5bbadcd732416a0f7a7f0acc092399dbbe8f5a5efa707c2ec0a49d7`
- `Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf` `316b46f3a2dbd68c900f43136ab9449f9dcc3725dfd8c794847c204bc161e113`

原生包和 MTP 草稿层由 setup 在容器内准备。无视觉编码器，无自定义
包，使用捆绑的专家配置。

**配置**（[strata-iq2_xs.json](strata-iq2_xs.json)；`/data` 是硬盘上的模型文件夹）：

- 上下文 131,072，`--kv int8 --kv-resident 32768`（KV 流式传输：缓存驻留在内存中，1.8 GB）；
- 专家缓存 `auto`：VRAM 中 4,580 个专家槽位（6,293 MiB），加载后剩余 638 MiB VRAM；
- `--prefill auto`：8,192-token 块，384 槽环形缓冲区，提示路径借用 3,222 个缓存槽位（4.31 GiB）；
- MTP `--spec 4`，**`--spec-min-p 0.7`** 和 **`--pcie-frac 0.2`**，均来自校准（见下文）；`--pool-workers` 保持
  默认值（7），校准也保持了该值；
- `fit_max_tokens` 为 true（对这些请求无影响：均未超过上下文）；
- 无视觉，无低内存模式，无速度预测；
- 推理关闭（`reasoning_effort: none`），温度 0，每次速度运行生成 256 个 token。

**校准：** `setup.py --calibrate` 在本报告之前于一个单独的容器中运行了一次，使用此 131,072 上下文。它
测量了 `--pcie-frac` 0.00-0.75、`--spec-min-p` 0.30-0.70 以及 7、5 和 4 个工作线程下的解码速度，并保留了
`--pcie-frac 0.20` 和 `--spec-min-p 0.70`（在其自身提示上为 34.1 tok/s）。其写入配置的内容随该
容器丢失，因此随后手动将相同的两个值应用到配置中（作为字符串；`args` 中的数字会导致
setup 停止）。我没有使用默认值重新运行速度矩阵，因此本报告并未测量校准在此处带来的收益。

启动命令（容器的入口点运行 setup，setup 从保存的配置启动服务器）：

```bash
docker run -d --name strata --ulimit core=0 --ulimit memlock=-1 \
  --device=/dev/kfd --device=/dev/dri --group-add video --group-add render --ipc=host \
  -p 127.0.0.1:8080:8080 \
  -v /media/gpf/DDrive1/dev/models:/data \
  -v /home/gpf/strata-ple/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf:/data/models/IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf:ro \
  -e MODEL=IQ2_XS -e CONTEXT=131072 strata-hip
```

`--ulimit core=0` 很重要：看门狗中止会写入一个与引擎内存大小相当的核心转储文件（约 37 GB），这
曾一度填满此 PC 的磁盘。

## 方法

[benchmark.py](benchmark.py) 是 [RTX 5090 报告](../2026-09-30-community-rtx-5090/benchmark.py)的脚本，未做修改
（SHA-256 `694df81b3d064ca609c73dc8606baf6cbb8a4837ea800933ee6f2eb80dc52d9a`）。它构建确定性的合成 Python
填充内容，在每个请求开头放置不同的 nonce，并使用 Strata 的分词器统计渲染后的聊天提示。
请求哈希和每次运行的输出文本位于 [results.json](results.json)；聚合结果位于
[summary.json](summary.json)。[monitor_amd.py](monitor_amd.py) 是 RTX 5090 报告的 `monitor.py`，将
`nvidia-smi` 调用替换为 amdgpu 的 sysfs 文件；它每秒采样一次内存、交换空间、VRAM、负载、功耗和温度
（速度运行对应 [telemetry.jsonl](telemetry.jsonl)，大海捞针运行对应 [telemetry-needles.jsonl](telemetry-needles.jsonl)）。基准测试脚本在宿主机上使用仓库的 `.venv` 运行：

```bash
python bench/results/2026-10-05-community-rx-6700-xt/benchmark.py --root . --pack /media/gpf/DDrive1/dev/models/packs/iq2_xs \
  --url http://127.0.0.1:8080 --out results/
python tools/needle_bench.py --url http://127.0.0.1:8080 --lengths 32k,128k --depths 10,50,90 --out needles.json
```

- 大海捞针测试先运行（UTC 00:45-01:09），然后是速度矩阵（UTC 01:10-01:39）。第一次速度尝试
  （[benchmark-attempt1-failed.log](benchmark-attempt1-failed.log)）在启动时因宿主机缺少 Python 模块（`regex`）而停止；
  未发送任何请求。安装仓库的 `requirements.txt` 后，速度矩阵运行了一次，未重复。
  使用的两个脚本是 [run_all.sh](run_all.sh)（其大海捞针部分）和 [run_speed.sh](run_speed.sh)。
- 一个短的预热请求被排除。每个长度三次运行在同一已加载引擎上按长度递增顺序串行执行。所有九个请求都读取了完整提示：**零复用 token**。
- 专家缓存在启动时未重新填充。此服务器在大海捞针测试前已回答了 360 个编码代理请求，且六个 32K-128K 大海捞针请求就在速度矩阵之前运行，因此自适应缓存是热的。
- TTFT 是流式的，从 HTTP 请求之前到第一个非空文本增量，通过回环。提示
  吞吐量是新读取的 token / `prompt_ms`。解码吞吐量是 `engine_generated / decode_ms`。加载时间
  已排除。
- [engine.log](engine.log) 是此服务器会话的完整引擎日志，包括运行前的编码代理请求。模型的启动决策位于其顶部。

## 结果

每个单元格是三次运行的 **[最小值-最大值]** 中位数。每个请求生成 256 个 token 并在
输出限制处停止（`finish: length`）。没有速度请求失败或被取消。

| 提示 token | 复用 | 提示 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 301.6 [299.7-301.8] | 32.1 [27.7-34.1] | 13.64 [13.63-13.73] | 21.65 [21.09-22.85] |
| 32,768 | 0 | 309.4 [308.8-310.3] | 30.7 [24.2-32.8] | 106.02 [105.72-106.26] | 114.02 [113.79-116.77] |
| 128,000 | 0 | 298.2 [296.4-298.4] | 30.2 [22.8-31.0] | 429.52 [429.21-432.11] | 437.74 [437.63-443.27] |

解码速度在同一提示的不同运行之间有所变化（27.7-34.1、24.2-32.8 和 22.8-31.0 tok/s）。草稿层的
接受率为 75-85%（162-183 个提供的猜测中有 131-148 个被接受；每次运行的计数位于
[results.json](results.json)）。

**大海捞针召回**（[needles.json](needles.json)，`tools/needle_bench.py`）：**6/6 找到**。

| 长度 | 深度 10% | 深度 50% | 深度 90% |
| --- | --- | --- | --- |
| 32K（32,171-32,172 token） | 找到，105 s | 找到，105 s | 找到，53 s |
| 128K（125,449-125,451 token） | 找到，422 s | 找到，424 s | 找到，371 s |

**内存**（来自遥测的整机值，速度运行；大海捞针运行在 0.3 GiB 以内）：内存使用
（MemTotal - MemAvailable）46.0-48.2 GiB，中位数 46.8，因此至少有 14.5 GiB 可用。引擎容器占用了其中约
36.4 GiB。交换空间使用了 8.0 GiB 中的 3.99-4.09 GiB，持平。VRAM 使用了 11.98 中的 11.82-11.93 GiB，中位数 11.83。缺页和
分页未测量。未发生内存不足事件。

## 正确性与局限性

- 大海捞针测试在两个长度、三个深度上检查一个码字的召回，将引擎的回答与
  预期词匹配。它不衡量回答质量，且这是 2 位量化：此处的速度并不能说明回答质量如何。
- 速度提示是合成的 Python 填充内容；真实的代码代理流量每次请求读取的新 token 要少得多
  （此服务器日志的早期快照，195 个代理请求，显示 98.6% 的提示 token 被复用，解码中位数为
  33.8 tok/s；本文件夹中的日志包含这些请求，但它们不属于上表）。
- 未测试：其他上下文、其他模型或规模、图像、多卡、Windows，以及不带 gfx1030
  覆盖的 gfx1031。此处的显卡是未验证的架构（`docs/AMD_HIP.md#rdna2-gfx1030`）。该覆盖遵循
  此显卡的 llama.cpp ROCm 镜像惯例；我没有测试原生 gfx1031 路径，因此我不知道
  是否需要它。
- 每个设置只运行一次。每个长度的三次运行就是重复次数；没有第二次会话。
- 硬盘设置是结果的一部分：当 n-gram 表位于硬盘上时，同一服务器在
  真实提示上停滞（见硬件），而此处该表位于 SSD 上。