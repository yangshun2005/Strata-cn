# 社区基准测试：NVIDIA GeForce RTX 4090

由 [Dmitry-B](https://github.com/Dmitry-B) 于 2026-10-03 测量。本测试使用 qwen3.8-flash-next-iq3_xxs 和 204800 token 上下文测试 Strata 0.1.38。提示词为俄语散文；贪心解码，输出上限 256 token，每种配置运行三次；TTFT 通过流式传输测量。这些是合成工作负载；它们不能证明通用的回答质量。

## 硬件和软件

- GPU：NVIDIA GeForce RTX 4090；报告显存 23028 MiB；功耗限制 480.00 W；PCIe 总线 00000000:01:00.0；PCIe 链路速度和宽度：未测量。GPU 频率未固定。
- CPU：AMD Ryzen 9 7950X 16 核处理器（32 个逻辑 CPU）。
- 内存：已安装 46464 MiB；存储布局：见 env.json（lsblk 输出）。
- Ubuntu 26.04.1 LTS，内核 7.0.0-38-generic；NVIDIA 驱动 610.57.04；版本 13.4，V13.4.92。
- Strata 提交 `1d5e1eaa0867b1fb2a531fc62150957f10056b6e`（分支 bench/2026-10-03-rtx4090-community）；引擎 0.1.38，发布二进制文件来自仓库的 engine/ 目录。
- 后台工作负载：一个 dsh/Authentik/Caddy Web 技术栈和 Ubuntu 默认服务；GPU 专用于 Strata，但操作系统未隔离。

## 模型和配置

- 模型：qwen3.8-flash-next-iq3_xxs（Strata 服务器的模型名称）；GGUF 文件名、大小和修改时间见 env.json（无哈希值）。
- 上下文 204800；INT8 KV；常驻专家；GPU 视觉 - 见下方配置副本。
- 草稿词汇子集：`draft_vocab=cyrillic`（英语/代码子集加上整个西里尔字母，约 106k 行）。这比默认的 `en` 子集更宽；英语文本上的草稿接受率未受影响（78-80%），但可能损失几个百分点的解码速度。

```text
/home/dgbox/Strata/engine/strata --serve --pack /home/dgbox/Strata-data/packs/iq3_xxs --native /home/dgbox/Strata-data/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00001-of-00002.gguf --ple-gguf /home/dgbox/Strata-data/models/IQ3_XXS/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS-00002-of-00002.gguf --expert-profile /home/dgbox/Strata/data/expert-profile-learned.bin --expert-cache auto --prefill auto --spec 4 --mtp /home/dgbox/Strata-data/mtp/rt --max-context 204800 --kv int8 --mmap-experts --vision --vram-reserve-mib 989 --control-vector-scaled /home/dgbox/Strata/data/experimental-speed-projection/Qwen3.8-Flash-Next-experimental-speed-projection.gguf:1.0 --control-vector-layer-range 4 44 --cvec-mode project --cvec-dir per-layer --pcie-frac 0.00 --spec-min-p 0.50 --pool-workers 10 --expert-profile-save /home/dgbox/Strata/data/expert-profile-learned.bin --expert-profile-save-every 10
```

完整服务器配置：[config.json](config.json)（位于 /home/dgbox/Strata/strata-200k.json），环境详情：[env.json](env.json)。

## 方法

- 预热：一个 4K token 的提示词（生成 64 个 token），不计入测量。
- 每个被测量的提示词都带有一个随机标记，因此提示词前缀缓存不会被复用；表中的 reused 列报告引擎实际复用的 token 数量。
- 吞吐量来自引擎的计时字段（prompt_per_second / predicted_per_second）。TTFT 是第一个非空流式增量出现的时间，忽略 keep-alive。总延迟（wall）是在客户端测量的整个请求时间。
- temperature=0，reasoning_effort=none，输出上限 256 token（仅生成场景为 1024）。模型在重复文本上经常提前停止；实际生成长度见表。
- 专家缓存由预热运行和之前的会话预热；专家配置状态在配置副本中。
- 内存：在测量运行期间每 2 秒采样一次峰值显存/内存，外加一个启动快照。

## 结果

| 配置 | 实际提示词 token 数 | 复用 token 数 | 生成 token 数 | 运行次数 | 提示词 tok/s 中位数和范围 | 解码 tok/s 中位数和范围 | TTFT 秒中位数和范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 4096-prompt | 4028 | 0 | 142 | 3 | 1823.8（范围 1797.2-1892.1，n=3） | 94.2（范围 80.1-95.2，n=3） | 2.24 s（范围 2.16-2.27，n=3） |
| 32768-prompt | 31828 | 0 | 256 | 3 | 2912.0（范围 2793.9-2962.9，n=3） | 85.6（范围 74.9-86.8，n=3） | 11.06 s（范围 10.87-11.52，n=3） |
| 131072-prompt | 127228 | 0 | 119 | 3 | 2984.2（范围 2957.9-2986.1，n=3） | 90.3（范围 86.2-102.8，n=3） | 43.11 s（范围 43.08-43.5，n=3） |
| gen-only | 228 | 0 | 450 | 3 | 279.7（范围 274.0-297.8，n=3） | 82.9（范围 82.8-85.8，n=3） | 0.83 s（范围 0.78-0.85，n=3） |

- 总延迟（客户端，wall）：4096-prompt - 3.75 s（范围 3.73-3.86，n=3）；32768-prompt - 14.04 s（范围 13.6-14.93，n=3）；131072-prompt - 44.52 s（范围 44.26-44.86，n=3）；gen-only - 6.27 s（范围 5.08-9.74，n=3）。
- 内存：{"start_snapshot": {"vram_used_mib": 22022, "ram_used_kib": 5819728}, "peak_vram_used_mib": 22022, "peak_ram_used_kib": 6529600, "note": "пик — максимум опросов каждые 2 с во время замеров"}
- 每次运行及其草稿统计（接受/总数）：[runs.json](runs.json)。
- 召回检查（needle）：[needles.json](needles.json)。

## 正确性和局限性

- 速度测量不能证明通用的回答质量。召回检查在 32K 和 128K 下、深度 10/50/90 全部通过 6/6；见 needles.json。
- GPU 未完全隔离：后台服务可能增加了少量噪声。
- 提示词是合成的（带随机标记的重复文本）；真实工作负载将显示不同的前缀复用和草稿接受率。