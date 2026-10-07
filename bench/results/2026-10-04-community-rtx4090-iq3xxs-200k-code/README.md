# 社区基准测试：NVIDIA GeForce RTX 4090

由 [Dmitry-B](https://github.com/Dmitry-B) 于 2026-10-04 测量。本测试使用 qwen3.8-flash-next-iq3_xxs 和 204800 token 上下文对 Strata 0.1.39 进行测试。提示词为代码解释文本；贪婪解码，输出上限 256 token，每种配置运行三次；TTFT 通过流式传输测量。这些是合成工作负载；它们不能代表通用的回答质量。

## 硬件和软件

- GPU：NVIDIA GeForce RTX 4090；报告显存 23028 MiB；功耗限制 480.00 W；PCIe 总线 00000000:01:00.0；PCIe 链路速度和宽度：未测量。GPU 时钟未固定。
- CPU：AMD Ryzen 9 7950X 16 核处理器（32 个逻辑 CPU）。
- 内存：已安装 46464 MiB；存储布局：见 env.json（lsblk 输出）。
- Ubuntu 26.04.1 LTS，内核 7.0.0-38-generic；NVIDIA 驱动 610.57.04；release 13.4，V13.4.92。
- Strata 提交 `2900da3ba8de27a467b4b3fcea6a4ca2a9061bea`（分支 bench/2026-10-03-rtx4090-community）；引擎 0.1.39，release 二进制文件来自仓库的 engine/ 目录。
- 后台工作负载：一个 dsh/Authentik/Caddy Web 技术栈和 Ubuntu 默认服务；GPU 专用于 Strata，但操作系统未隔离。

## 模型和配置

- 模型：qwen3.8-flash-next-iq3_xxs（Strata 服务器的模型名称）；GGUF 文件名、大小和修改时间见 env.json（无哈希值）。
- 上下文 204800；INT8 KV；常驻专家；GPU 视觉 - 见下方配置副本。
- 草稿词表子集：`draft_vocab=cyrillic`（英文/代码子集加上整个西里尔字母，约 106k 行）。这比默认的 `en` 子集更宽；英文文本上的草稿接受率未受影响（78-80%），但可能损失几个百分点的解码速度。

```text
/home/dgbox/Strata/engine/strata --serve --pack /home/dgbox/Strata-data/packs/iq3_xxs --native /home/dgbox/Strata-data/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf --ple-gguf /home/dgbox/Strata-data/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf --expert-profile /home/dgbox/Strata/data/expert-profile-learned.bin --expert-cache auto --prefill auto --spec 4 --mtp /home/dgbox/Strata-data/mtp/rt --max-context 204800 --kv int8 --mmap-experts --vision --vram-reserve-mib 989 --control-vector-scaled /home/dgbox/Strata/data/experimental-speed-projection/Qwen3.8-Flash-Next-experimental-speed-projection.gguf:1.0 --control-vector-layer-range 4 44 --cvec-mode project --cvec-dir per-layer --pcie-frac 0.00 --spec-min-p 0.50 --pool-workers 10 --expert-profile-save /home/dgbox/Strata/data/expert-profile-learned.bin --expert-profile-save-every 10
```

完整服务器配置：[config.json](config.json)（原位于 /home/dgbox/Strata/strata-200k.json），环境详情：[env.json](env.json)。

## 方法

- 预热：一个 4K token 的提示词（生成 64 个 token），不计入测量。
- 每个被测提示词都带有一个随机标记，因此提示词前缀缓存不会被复用；表格中的复用列报告引擎实际复用的 token 数。
- 吞吐量来自引擎的计时字段（prompt_per_second / predicted_per_second）。TTFT 是到第一个非空流式增量的时间，忽略 keep-alive。总延迟（wall）是在客户端测量的整个请求时间。
- temperature=0，reasoning_effort=none，输出上限 256 token（仅生成场景为 1024）。模型在重复文本上经常提前停止；实际生成长度见表格。
- 专家缓存由预热运行和之前的会话预热；专家配置文件状态见配置副本。
- 内存：在测量运行期间每 2 秒采样一次峰值显存/内存，外加一个启动快照。

## 结果

| 配置 | 实际提示词 token 数 | 复用 token 数 | 生成 token 数 | 运行次数 | 提示词 tok/s 中位数和范围 | 解码 tok/s 中位数和范围 | TTFT 秒 中位数和范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 4096-prompt | 3971 | 0 | 256 | 3 | 1966.2（范围 1952.4-1967.7，n=3） | 123.6（范围 109.6-134.3，n=3） | 2.04 s（范围 2.04-2.06，n=3） |
| 32768-prompt | 31307 | 0 | 256 | 3 | 2973.7（范围 2952.3-3130.1，n=3） | 134.9（范围 123.8-136.0，n=3） | 10.61 s（范围 10.09-10.69，n=3） |
| 131072-prompt | 125011 | 0 | 183 | 3 | 3084.9（范围 3043.4-3109.1，n=3） | 120.3（范围 91.5-124.1，n=3） | 40.82 s（范围 40.51-91.16，n=3） |
| gen-only | 163 | 0 | 1024 | 3 | 205.3（范围 204.7-219.0，n=3） | 135.8（范围 135.2-138.6，n=3） | 0.81 s（范围 0.76-0.82，n=3） |

- 总延迟（客户端，wall）：4096-prompt - 4.1 s（范围 3.94-4.38，n=3）；32768-prompt - 12.48 s（范围 11.97-12.75，n=3）；131072-prompt - 42.87 s（范围 42.02-92.7，n=3）；gen-only - 8.29 s（范围 8.18-8.37，n=3）。
- 内存：{"start_snapshot": {"vram_used_mib": 22031, "ram_used_kib": 5931132}, "peak_vram_used_mib": 22031, "peak_ram_used_kib": 6667592, "note": "пик — максимум опросов каждые 2 с во время замеров"}
- 每次运行及其草稿统计（接受/总数）：[runs.json](runs.json)。
- 召回检查（needle）：[needles.json](needles.json)。

## 正确性和局限性

- 速度测量不能代表通用的回答质量。召回检查在 32K 和 128K 下、深度 10/50/90 处通过了 6/6；见 needles.json。
- GPU 未完全隔离：后台服务可能增加了少量噪声。
- 提示词是合成的（带随机标记的重复文本）；真实工作负载会表现出不同的前缀复用和草稿接受率。
- 一次测量运行受到该机器同时服务的交互式会话的污染：在两个 131072-prompt 运行之间，另一个客户端的 139,527 token 请求被处理（服务器一次只回答一个序列），这就是该场景的 TTFT 和 wall 范围达到 91 s 和 93 s 的原因。该次运行的引擎侧计时未受影响（125,011 token 以 3043 tok/s 读取，与其两个相邻运行相同）；只有客户端侧的等待被夸大了。


## 与引擎 0.1.38 的比较

同一台 PC、同一模型、同一运行配置，与
[2026-10-03-community-rtx4090-iq3xxs-200k-code](../2026-10-03-community-rtx4090-iq3xxs-200k-code/README.md)
（引擎 0.1.38，Strata 提交 `1d5e1ea`）相同。只有 Strata 版本变化：0.1.38 -> 0.1.39，源码构建，相同的
CUDA 13.4，相同的 `sm_89`，相同的参数，相同的 `expert-profile-learned.bin`。引擎的自动提示词分块在两者中
均未改变（服务器日志中为 `prompt chunk auto: 8192`），因此 #583 的字节预算环不适用于此配置。

| 配置 | 提示词 tok/s 0.1.38 -> 0.1.39 | 解码 tok/s 0.1.38 -> 0.1.39 | TTFT 秒 0.1.38 -> 0.1.39 |
| --- | --- | --- | --- |
| 4096-prompt | 1844.3 -> 1966.2 (+6.6%) | 118.0 -> 123.6 (+4.7%) | 2.18 -> 2.04 |
| 32768-prompt | 2976.2 -> 2973.7 (-0.1%) | 127.9 -> 134.9 (+5.5%) | 10.6 -> 10.61 |
| 131072-prompt | 2962.3 -> 3084.9 (+4.1%) | 108.4 -> 120.3 (+11.0%) | 42.5 -> 40.82 |
| gen-only | 200.0 -> 205.3 (+2.7%) | 133.0 -> 135.8 (+2.1%) | 0.83 -> 0.81 |

- 提示词吞吐量是可比较的数字：它来自引擎自身的计时，且两次运行的范围都很窄
  （131072-prompt 下为 2906-2965 和 3021-3110 tok/s）。
- 解码中位数是最不可比较的一列。每个被测提示词都带有一个随机标记，在这种重复文本上，模型每次运行会在
  不同的位置提前停止，因此两份报告中的生成长度不同（例如 4096-prompt 下为 142 对 256 token）。较短的回答
  和较长的回答不具有相同的草稿接受率。`gen-only` 场景始终运行到 1024 token 上限，是四者中最干净的解码比较。
- 召回：两个版本在 32K 和 128K 下均为 6/6。