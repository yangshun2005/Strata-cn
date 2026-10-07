# 社区基准测试：AMD MI50 (gfx906)、Intel i5-12400F

由 [xxDoman](https://github.com/xxDoman) 于 2026-10-04 在 Linux 机器 **LianLi** 上测量。本次测试使用 Strata 0.1.38 搭配原始 Flash-Next IQ2_XS，单块 GPU（MI50），在模型完整的 **131,072 token 上下文**下启用 KV 流式传输。

中位解码吞吐量为 **4,096 提示 token 时 36.0 tok/s，32,768 时 34.0 tok/s，128,000 时 33.5 tok/s**——随着上下文增长几乎不下降。提示处理在 **4K/32K 时保持约 330 tok/s，128K 时约 305 tok/s**。这些是合成代码解释请求，采用贪婪解码，输出上限为 256 token。它们不能代表通用回答质量或其他工作负载下的性能。

据我们所知，这是首次在 AMD MI50（Vega 20，gfx906）上进行的社区实测 Strata 运行。这是一个实验性的、不受支持的架构：构建不包含厂商调优表，数据也反映了这一点。它同时表明，MI50——一块 32 GB 显卡——可以通过 KV 流式传输服务模型完整的 128K 上下文，相比 4K 提示，解码开销仅约 7%。

## 硬件与软件

- AMD Radeon Instinct MI50 32GB（Vega 20，gfx906）；报告 VRAM 为 34,342,961,152 字节；PCIe Gen4 x16（`LnkSta` 16 GT/s，宽度 x16）。TDC GFX 限制在启动时固定为 150 W（由主机服务设置）；除此之外未固定时钟。
- 第 12 代 Intel Core i5-12400F；12 个逻辑 CPU。引擎选择了 AVX2 构建，以及 5 个专家池工作线程加上其宿主线程。
- 已安装 64 GB RAM；Linux 报告总计 61.50 GiB。本地 NVMe 存储：Lexar SSD NM790 2TB。15 GiB 系统交换空间。
- Ubuntu 26.04.1 LTS，内核 7.0.0-38-generic。AMD **ROCm 7.2.1**（`hipcc` 报告 AMD clang 22.0.0）。
- 源码提交 `99f3dbd0b21d1401b3769e0c0d963913607f380b`；引擎 0.1.38。在主机上使用 ROCm 7.2.1 工具链为 **gfx906** 构建，并打包为 Docker 镜像（`strata-mi50:0.1.38`）；服务器在容器内运行，透传了 `/dev/kfd` 和 `/dev/dri`。参见 [BUILD.json](BUILD.json)。
- 运行期间 GPU 专用于 Strata（同机部署的 Ollama 服务器未加载任何模型）。其他 CPU 服务仍在运行；这不是一个完全隔离的操作系统。

## 模型与配置

模型：`ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF`，修订版
`ed59f92082b1e93c0e96d60a8b11aab089b52f09`：

- `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf`
- `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf`

两个本地 GGUF 的 SHA-256 哈希均与该修订版发布的 LFS 哈希匹配
（参见 [model-sha256.txt](model-sha256.txt) 和 [provenance.json](provenance.json)）。
`mmproj-Qwen3.8-Flash-Next-BF16.gguf` 存在于磁盘上，但**未**传递给引擎：本次运行关闭了视觉功能，因此本报告不涵盖它。

- **上下文 131,072；KV 流式传输 `--kv-resident 32768`**——每个 QSA 层有 32,768 个 KV 单元常驻 VRAM，完整的 K/V 位于固定内存中。这正是让完整的 128K 上下文适配显卡、同时为专家留出 VRAM 的原因。
- KV 类型 INT8。
- 专家缓存 `auto`：19,304 个槽位，按 profile 预填充且无驱逐。引擎日志：缓存命中路径开启；每请求 `hit_rate` 为 0.97–0.995。
- 预填充 2,048 token 分块。
- MTP `--spec 3 --spec-min-p 0.9`。
- 视觉关闭；温度 0；每次速度测试最多生成 256 token。

完整的实测配置见 [strata-iq2_xs.json](strata-iq2_xs.json)，引擎自身的启动和每请求日志行见 [engine.log](engine.log)。

## 方法与复现

本测试使用了仓库社区测试工具
（[`bench/results/2026-09-30-community-rtx-5090/benchmark.py`](../2026-09-30-community-rtx-5090/benchmark.py)）的改编版本。
唯一的改动是容器的目标端口/包路径以及必需的
`Authorization` 请求头（[benchmark_mi50.py](benchmark_mi50.py)）。它在容器内运行（从 `/opt/strata` 导入 `strata_tokenizer` 和 `serve.frontend`）。

```bash
docker exec strata python3 /work/bench/benchmark_mi50.py --api-key <key> \
  --url http://127.0.0.1:8085 --out /work/bench/results --targets 4096,32768,128000 --runs 3
```

该脚本生成确定性的合成 Python 函数，在每个请求开头附近添加不同的 nonce，并使用 Strata 的分词器统计完整渲染后的聊天提示，调整填充内容至目标值，随后向服务器验证计数。一次预热请求被排除。在每个长度上各执行三次运行，按长度递增顺序在同一已加载引擎上串行执行。全部九次速度请求都处理了其完整提示：**零复用 token**。加载时间不计入；请求之间保留了专家缓存。

TTFT 在客户端测量，从 HTTP 请求发出前一刻起，直到第一个非空文本增量。引擎提示吞吐量使用新读取的 token 除以 `prompt_ms`；解码吞吐量使用 `engine_generated / decode_ms`。

系统内存和 GPU 内存由
[monitor_mi50.py](monitor_mi50.py)（主机 `rocm-smi` 改编版）每秒采样一次，从加载前直到最后一次运行结束；原始样本见 [telemetry.jsonl](telemetry.jsonl)。

## 结果

每个单元格为三次运行的中位数 **[最小值–最大值]**。每个请求生成 256 token 并在输出上限处停止；没有失败或取消的速度请求。

| 提示 token | 复用 | 提示 tok/s | 解码 tok/s | TTFT 秒 | 总秒数 |
| ---: | ---: | ---: | ---: | ---: | ---: |
| 4,096 | 0 | 331.9 [330.1–332.3] | 36.0 [35.9–37.4] | 12.4 [12.4–12.5] | 19.4 [19.3–19.5] |
| 32,768 | 0 | 327.2 [327.2–327.3] | 34.0 [33.9–34.1] | 100.2 [100.2–100.2] | 107.7 [107.7–107.7] |
| 128,000 | 0 | 305.2 [304.7–305.2] | 33.5 [32.4–34.0] | 419.6 [419.4–419.7] | 427.2 [426.8–430.0] |

每次运行的原始记录：[results.json](results.json)。计算后的汇总：
[summary.json](summary.json)。每请求的引擎值也保存在
`tokens-<N>-run-<R>-raw.json` 中。

会话的第一次请求（此前一次被丢弃的调用）在缓存稳定前显示出一次性的预填充开销（14 个提示 token 约 35 秒）；本次运行中的预热被排除，稳态预填充在三个长度上均为平稳的约 305–330 tok/s。

内存数据，来自 [telemetry.jsonl](telemetry.jsonl)：

- 主机 RAM：峰值 **40.96 GiB**，按所有进程的 `MemTotal - MemAvailable` 计算（并非仅 Strata 的 RSS）。约 41 GiB 是 RAM 中的专家集合加上 128K 上下文的流式 K/V。
- 交换空间：峰值 **约 2.9 GiB 已用**，从约 0 上升——运行期间 Strata 发生换页。
- GPU 内存：MI50 几乎被填满至全部可用 VRAM（约 33.5 GB，32 GB 级显卡）；引擎报告在窗口开启时剩余 `964 MiB`，并为草稿头预留 `1024 MiB`。
- 未发生内存不足错误。

## 正确性与局限性

通过仓库未修改的 `tools/needle_bench.py` 进行长上下文召回测试：
在完整上下文下进行一次探测——`--lengths 128k --depths 50`——在 122,754 token 的提示中**找到了针**（426 秒）。结果见
[needles.json](needles.json)。

_这是一台机器、一种量化、一种配置，以及一个小型合成工作负载。长输出、采样解码、思考、编码任务正确性、视觉、工具使用、多请求并发以及持续热运行均未评估。作为不受支持的 gfx906 构建，没有厂商调优表；在受支持架构上的结果与这些不可比较。未在其他引擎或 GPU 上运行相同工作负载的基线。_