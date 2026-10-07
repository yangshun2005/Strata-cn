# 社区基准测试：RTX 4070 Ti SUPER，Ryzen 7 9800X3D

由 [lfontanez](https://github.com/lfontanez) 于 2026-10-05 在一台名为 Shin-BlackMamba 的 Linux 机器上测量。本测试在上游主分支提交 `6f32ec07` 上测试 Strata，使用 Qwen3.8-Flash-Next IQ2_XS 和 IQ3_XXS 两种量化，单块 GPU，上下文上限从 32 768 到 262 144 tokens。该基准测试同时测量**挂钟预填充（冷启动）**、**流式预填充（TTFT 对齐）**和**解码吞吐量**，外加 32 k 下的长上下文召回检查。

中位数**解码吞吐量**（引擎报告的 `predicted_per_second`，在温度 0、max_tokens 192、`reasoning_effort="minimal"` 下进行三次 SSE 流调用）：

| 模型    | KV    | 32 k 上下文  | 64 k 上下文  | 128 k 上下文 | 256 k 上下文 |
|----------|-------|----------:|----------:|----------:|----------:|
| IQ2_XS   | int8  |  **97.90** |  **94.30** |  **90.00** |  **82.40** |
| IQ3_XXS  | 不定 |  76.00    |  71.30    |  70.00    |  56.00    |

冷预填充吞吐量（`POST /unload` 后执行 `max_tokens=1` 非流式调用）在所有 8 种配置下均保持在 2 400-2 750 t/s；README 中 2 000+ t/s 的标题数据在这块卡上依然成立。流式预填充（TTFT 对齐）为 100-160 t/s——这两个指标含义不同，参见方法论。

## 硬件与软件

- **GPU**：NVIDIA GeForce RTX 4070 Ti SUPER，16 376 MiB 显存（16 GiB），计算能力 8.9（`sm_89`）。PCIe 能力：Gen 4，x16；引擎的启动传输探测报告了该插槽的主机到设备带宽。GPU 功耗限制为出厂默认的 285 W；本次测试未固定时钟频率。
- **CPU**：AMD Ryzen 7 9800X3D，16 个逻辑 CPU，AVX2 + AVX-512。引擎选择了 23 个专家池工作线程加上其主机线程。
- **内存**：已安装 94 GiB；Linux 报告可用 90.43 GiB。8 GiB 系统交换空间。
- **存储**：Samsung 990 PRO 2 TB NVMe（`/dev/nvme0n1p2`，ext4，根文件系统）。本地模型卷：名为 `strata-data` 的 Docker 卷挂载在 `/data`。
- **操作系统**：Ubuntu 24.04.5 LTS，内核 6.8.0-146-generic。
- **技术栈**：NVIDIA 驱动 595.91.07（CUDA 13.2）；nvidia-container-toolkit 1.20.1，`nvidia` 运行时作为 Docker 默认；Docker 29.8.1。
- **Strata 提交**：`6f32ec070f23ced9f50e704d854d775da52591ab`（[Niko1221/Strata](https://github.com/Niko1221/Strata) main，2026-10-05）。
- **引擎构建**：本地编译 CUDA 架构 **89**（仅 RTX 40，单 cubin 的 fat-binary），`BUILD_VISION=0`，因此图像编码器**未**编译。基础镜像：`nvidia/cuda:13.0.0-devel-ubuntu24.04`。Python 3.12.3；NVCC 13.0.x。
- **工作负载隔离**：主机上还有 8 个其他容器在运行（voicebox、supabase 栈、n8n、qdrant、langfuse、open-webui、portainer、flowise）。除 voicebox 外均为 CPU 和内存密集型；voicebox 空闲时为 TTS 密集型。整个运行期间 GPU 专用于 Strata 容器。这不是一个完全隔离的操作系统。

## 模型与配置

模型来自 [ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF)，版本由安装程序在运行时解析：

- IQ2_XS：`IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf`（39.07 GiB）+ `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf`（26.81 GiB PLE）。
- IQ3_XXS：`IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf`（43.81 GiB）+ `IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf`（共享 PLE 分片）。

MTP 由安装程序从 [Qwen/Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) 获取；张量范围清单位于 `/data/mtp/mtp-manifest.json`（以及本 PR 审计文件夹中的 [`mtp-manifest.json`](mtp-manifest.json)）。选择了 Q2_0 MTP 专家，如 install.sh 默认配置中所设置。

未经修改的安装程序（`-e VISION=no`）准备了原生 IQ2_XS 和 IQ3_XXS 包。可执行文件和准备好的包位于镜像中的 `/opt/strata/engine/strata` 以及 `/data` Docker 卷中。

测试的 8 种配置：

|  配置 # | 模型    | 上下文  | KV    | KV 流式 |
|--------|----------|---------:|------:|:------------:|
|  1     | IQ2_XS   |    32768 | int8  | 不适用（KV 可容纳） |
|  2     | IQ2_XS   |    65536 | int8  | `--kv-resident 32768` |
|  3     | IQ2_XS   |   131072 | int8  | `--kv-resident 32768` |
|  4     | IQ2_XS   |   262144 | int8  | `--kv-resident 32768` |
|  5     | IQ3_XXS  |    32768 | int8  | 不适用（KV 可容纳） |
|  6     | IQ3_XXS  |    65536 | int8  | （KV 在 RAM 中仍然很小） |
|  7     | IQ3_XXS  |   131072 | q4_0  | （KV 流式 + q4_0） |
|  8     | IQ3_XXS  |   262144 | k8v4  | （KV 流式 + k8v4） |

KV 选择反映了显存范围：IQ3_XXS 常驻 47 GiB 专家，因此其 KV 表在更高上下文时必须缩小。IQ2_XS 在所有情况下都保持 `int8`，因为 `--kv-resident 32768` 将活跃 KV 切片保留在 RAM 中，而非显存中。

所有 8 种配置的通用设置：

- `--expert-cache auto`、`--prefill auto`、`--spec 4 --spec-min-p 0.5`。
- `--expert-profile /opt/strata/data/expert-profile.bin`（捆绑，随镜像发布）。
- MTP 草稿层位于 `/data/mtp/rt`（Q2_0 张量范围，默认）。
- 推理 `minimal`，温度 0，流式运行使用 `max_tokens 192`。
- 冷预填充探测在 `POST /unload` 后使用 `max_tokens 1` 非流式调用。
- 视觉解码器未编译（`BUILD_VISION=0`）；`VISION=no`。
- 低内存检测器在启动时选择了**常驻专家**（94 GiB > 模型专家）。

完整的测量配置位于 [`strata-iq2_xs.json`](strata-iq2_xs.json)（CONTEXT=262144，KV=int8，KV 流式）和 [`strata-iq3_xxs.json`](strata-iq3_xxs.json)（CONTEXT=262144，KV=k8v4）。所有其他 CONTEXT 在重新启动时通过 `-e REINSTALL=1` 应用。

## 方法与复现

使用附带的 [`strata-iq2_xs.json`](strata-iq2_xs.json) 和 [`strata-iq3_xxs.json`](strata-iq3_xxs.json)——它们是有效的，因为每个容器启动时都会在 `/data/config/` 上重新生成配置，而带 `REINSTALL=1` 的 `docker run` 会重新应用模型设置而无需重新下载。

```bash
# 从仓库根目录，使用空闲的 GPU + toolkit（已验证）
docker build -t strata \
  --build-arg CUDA_ARCHITECTURES=89 \
  --build-arg BUILD_VISION=0 \
  .

docker volume create strata-data

# 以 IQ2_XS @ 32k 为例。其他 7 种配置请替换 MODEL/CONTEXT/KV。
docker rm -f strata 2>/dev/null
docker run -d --name strata --network host --gpus all \
  --ulimit memlock=-1 --shm-size=4g \
  -v strata-data:/data \
  -e FAMILY=qwen -e MODEL=IQ2_XS -e CONTEXT=32768 \
  -e VISION=no -e KV=int8 -e REINSTALL=1 \
  -e HOST=127.0.0.1 -e PORT=8090 \
  --restart unless-stopped strata

# 每行 1 次冷预填充探测 + 3 次 SSE 流调用 → 矩阵
pip install requests
python3 bench/results/2026-10-01-ctx-ladder/run.py \
  --model qwen3.8-flash-next-iq2_xs --contexts 32768 \
  --out bench/results/2026-10-01-ctx-ladder/32k.json
```

**为什么使用 `--network host`？** 在桥接网络上发布 `-p 127.0.0.1:HOST:CT` 会被 DNAT 到容器的桥接接口，但如果引擎绑定 `127.0.0.1`（其默认值），内核会拒绝转发到自身的回环。`--network host` 将两个命名空间合并，使引擎的回环就是主机的回环。容器也仅限 localhost，因此在单用户开发机上无需 API 密钥即可安全使用。

每个运行行包含三次 SSE 流调用加一次冷预填充探测。冷预填充是全新的 `/unload` 后执行非流式 `max_tokens=1` 调用：答案不会与之前解码的 KV 复用混淆。每次容器启动都是一次全新的加载（约 50-90 秒）；三次流调用之间引擎未重启或 `/unload`——调用之间提示词会变化，且服务器端禁用了对话缓存。

## 结果（完整矩阵，每格 3 次运行 + 冷预填充）

| 标签        | 模型    | KV    | 提示词 tok | 平均 TPS | 峰值    | 引擎 TPS | **冷预填充** | SSE 预填充 | 草稿 | TTFT   |
|--------------|----------|-------|-----------:|--------:|--------:|-----------:|-----------------:|------------:|------:|-------:|
| 32 k         | IQ2_XS   | int8  |    29 966  |   95.62 |  136.34 |     93.47  |    **2 695**     |       141.3 | 75.8% | 183 ms |
| 64 k         | IQ2_XS   | int8  |    60 270  |   91.80 |  123.20 |     89.90  |    **2 678**     |       140.9 | 74.5% | 271 ms |
| 128 k        | IQ2_XS   | int8  |   120 748  |   87.95 |  117.52 |     86.67  |    **2 670**     |       132.6 | 76.8% | 422 ms |
| 256 k        | IQ2_XS   | int8  |   240 912  |   79.63 |  107.36 |     78.50  |    **2 484**     |       127.3 | 69.6% | 726 ms |
| IQ3 @32 k    | IQ3_XXS  | int8  |    29 966  |   73.32 |   97.96 |     71.73  |    **2 747**     |       116.5 | 69.8% | 217 ms |
| IQ3 @64 k    | IQ3_XXS  | int8  |    60 270  |   69.16 |   98.66 |     68.36  |    **2 545**     |       107.0 | 70.3% | 295 ms |
| IQ3 @128 k   | IQ3_XXS  | q4_0  |   120 748  |   65.51 |   84.42 |     64.10  |    **2 478**     |       105.2 | 67.7% | 461 ms |
| IQ3 @256 k   | IQ3_XXS  | k8v4  |   240 912  |   53.70 |   71.09 |     52.97  |    **2 404**     |        91.9 | 66.6% | 768 ms |

质量与速度的代价（IQ3_XXS 对比 IQ2_XS，相同 KV）：

| 上下文 | KV | IQ2_XS 引擎 TPS | IQ3 引擎 TPS | IQ3 解码代价 |
|-----|-----|-----------------:|---------------:|---------------:|
| 32 k | int8 | 93.47 | 71.73 | **-23.3 %** |
| 64 k | int8 | 89.90 | 68.36 | **-24.0 %** |
| 128 k | int8（iq2）对比 q4_0（iq3） | 86.67 | 64.10 | **-26.0 %**（KV 不同，存在混淆） |
| 256 k | int8（iq2）对比 k8v4（iq3） | 78.50 | 52.97 | **-32.5 %**（KV 不同，存在混淆） |

128 k/256 k 行并非干净的同类对比（IQ3 引擎在其一侧被迫降低 KV），但它们反映了希望在 256 k 上下文下获得 IQ3 质量的用户在单块 16 GiB 卡上实际会看到的情况。

### 长上下文召回（`tools/needle_bench.py`）

对运行中的容器执行单次 `needle_bench.py --lengths 32k --depths 10,50,90`。来源：`docs/`、`include/`、`src/`、`third_party/llama.cpp/docs`，按固定顺序，截断至约 32 768 × 3.2 字符 = 约 104 858 字符。脚本从 16 个词的列表中随机选取一个 18 像素的词，将其嵌入所选深度，并要求返回。

| 标签       | 长度 | 深度   | 找到 | 提示词 tokens | 秒数 |
|-------------|:------:|--------:|:----:|--------------:|--------:|
| IQ2_XS @32k | 32 k   | 10 %    | ✅   |       32 171  |    13   |
| IQ2_XS @32k | 32 k   | 50 %    | ✅   |       32 171  |    12   |
| IQ2_XS @32k | 32 k   | 90 %    | ✅   |       32 172  |     6   |
| IQ3_XXS@32k | 32 k   | 10 %    | ✅   |       32 171  |    12   |
| IQ3_XXS@32k | 32 k   | 50 %    | ✅   |       32 171  |    12   |
| IQ3_XXS@32k | 32 k   | 90 %    | ✅   |       32 172  |     6   |

在 32 k 上下文下，IQ2_XS 和 IQ3_XXS 均为 `()   6 of 6 FOUND`。

原始 JSON：[`recall_iq2_xs_32k.json`](recall_iq2_xs_32k.json)、[`recall_iq3_xxs_32k.json`](recall_iq3_xxs_32k.json)。

## 方法论

表格中出现两个预填充指标；它们含义不同：

- **冷预填充**——引擎报告的 `timings.prompt_per_second`，来自 `POST /unload` 后的 `max_tokens=1` 非流式调用。真正的提示词结束吞吐量，即 2 000+ t/s 标题数据所测量的数字。
- **SSE 预填充**——引擎报告的 `timings.prompt_per_second`，来自流式调用。较低