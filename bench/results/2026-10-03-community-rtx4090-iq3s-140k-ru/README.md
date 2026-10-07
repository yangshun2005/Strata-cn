# 社区基准测试：NVIDIA GeForce RTX 4090

由 [Dmitry-B](https://github.com/Dmitry-B) 于 2026-10-03 测量。本测试使用 Strata 0.1.38，搭配 qwen3.8-flash-next-iq3_s 和 143360 token 上下文。提示词为俄语散文；贪婪解码，输出上限 256 token，每种配置运行三次；TTFT 通过流式传输测量。这些是合成工作负载；它们不能代表通用的回答质量。

## 硬件与软件

- GPU：NVIDIA GeForce RTX 4090；报告显存 23028 MiB；功耗限制 480.00 W；PCIe 总线 00000000:01:00.0；PCIe 链路速度和宽度：未测量。GPU 频率未锁定。
- CPU：AMD Ryzen 9 7950X 16 核处理器（32 个逻辑 CPU）。
- 内存：已安装 46464 MiB；存储布局：见 env.json（lsblk 输出）。
- Ubuntu 26.04.1 LTS，内核 7.0.0-38-generic；NVIDIA 驱动 610.57.04；版本 13.4，V13.4.92。
- Strata 提交 `99f3dbd0b21d1401b3769e0c0d963913607f380b`（分支 main）；引擎 0.1.38，发布二进制文件来自仓库的 engine/ 目录。
- 后台工作负载：一个 dsh/Authentik/Caddy Web 技术栈以及 Ubuntu 自带服务；GPU 专用于 Strata，但操作系统未隔离。

## 模型与配置

- 模型：qwen3.8-flash-next-iq3_s（Strata 服务器的模型名称）；GGUF 文件名、大小和修改时间见 env.json（无哈希值）。
- 上下文 143360；INT8 KV；常驻专家；GPU 视觉——见下方配置副本。
- 草稿词表子集：`draft_vocab=cyrillic`（英文/代码子集加上整个西里尔字母，约 106k 行）。这比默认的 `en` 子集更宽；英文文本上的草稿接受率未受影响（78-80%），但可能损失几个百分点的解码速度。

```text
/home/dgbox/Strata/engine/strata --serve --pack /home/dgbox/Strata-data/packs/iq3_s --native /home/dgbox/Strata-data/models/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf --ple-gguf /home/dgbox/Strata-data/models/IQ3_S/Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf --expert-profile /home/dgbox/Strata/data/expert-profile-learned-iq3s.bin --expert-cache auto --prefill auto --spec 4 --mtp /home/dgbox/Strata-data/mtp/rt --max-context 143360 --kv int8 --resident-experts --vision --vram-reserve-mib 989 --control-vector-scaled /home/dgbox/Strata/data/experimental-speed-projection/Qwen3.8-Flash-Next-experimental-speed-projection.gguf:1.0 --control-vector-layer-range 4 44 --cvec-mode project --cvec-dir per-layer --spec-min-p 0.5 --expert-profile-save /home/dgbox/Strata/data/expert-profile-learned-iq3s.bin --expert-profile-save-every 10
```

完整服务器配置：[config.json](config.json)（原位于 /home/dgbox/Strata/strata-iq3s-140k-resident.json），环境详情：[env.json](env.json)。

## 方法

- 预热：一个 4K token 的提示词（生成 64 个 token），不计入测量。
- 每个被测提示词都带有随机标记，因此提示词前缀缓存不会被复用；表格中的 reused 列报告引擎实际复用的 token 数量。
- 吞吐量来自引擎的计时字段（prompt_per_second / predicted_per_second）。TTFT 是首个非空流式增量的时间，忽略 keep-alive。总延迟（wall）是在客户端测量的整个请求时间。
- temperature=0，reasoning_effort=none，输出上限 256 token（仅生成场景为 1024）。模型在重复文本上经常提前停止；实际生成长度见表格。
- 专家缓存由预热运行和更早的会话预热；专家配置状态见配置副本。
- 内存：在测量运行期间每 2 秒采样一次峰值显存/内存，外加一个启动快照。

## 结果

| 配置 | 实际提示词 token 数 | 复用 token 数 | 生成 token 数 | 运行次数 | 提示词 tok/s 中位数及范围 | 解码 tok/s 中位数及范围 | TTFT 秒中位数及范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 4096-prompt | 4028 | 0 | 102 | 3 | 2084.2（范围 2078.8-2086.5，n=3） | 76.1（范围 62.4-91.3，n=3） | 1.96 s（范围 1.96-1.97，n=3） |
| 32768-prompt | 31828 | 0 | 102 | 3 | 2991.1（范围 2987.3-2997.6，n=3） | 102.5（范围 101.9-104.2，n=3） | 10.78 s（范围 10.75-10.79，n=3） |
| 131072-prompt | 127228 | 0 | 102 | 3 | 3077.9（范围 3075.9-3079.6，n=3） | 102.2（范围 97.7-105.2，n=3） | 41.81 s（范围 41.8-41.86，n=3） |
| gen-only | 228 | 0 | 826 | 3 | 312.6（范围 305.6-314.0，n=3） | 65.9（范围 54.6-66.5，n=3） | 0.75 s（范围 0.74-0.77，n=3） |

- 总延迟（客户端，wall）：4096-prompt - 3.29 s（范围 3.07-3.59，n=3）；32768-prompt - 11.77 s（范围 11.72-11.78，n=3）；131072-prompt - 42.82 s（范围 42.8-42.83，n=3）；gen-only - 13.18 s（范围 5.31-13.49，n=3）。
- 内存：{"start_snapshot": {"vram_used_mib": 22116, "ram_used_kib": 46273424}, "peak_vram_used_mib": 22116, "peak_ram_used_kib": 46918576, "note": "peak = maximum of 2-second samples taken during the measured runs"}
- 每次运行及其草稿统计（接受/总数）：[runs.json](runs.json)。
- 召回检查（needle）：[needles.json](needles.json)。

## 正确性与局限性

- 速度测量不能代表通用的回答质量。召回检查在 32K 和 128K 下、深度 10/50/90 全部通过 6/6；见 needles.json。
- GPU 未完全隔离：后台服务可能引入了少量噪声。
- 提示词是合成的（带随机标记的重复文本）；真实工作负载会表现出不同的前缀复用和草稿接受率。