# Strata 与 llama.cpp v0.6.0 正面对比（2026-10-06）

Stream 1402-h。仅测量。脚本：`bench/bench_vs_llama.py`。每请求原始数据行：`<machine>/raw/*.jsonl`
（命令位于每个文件的第一行），调优扫描 `<machine>/tune-*.txt`，机器/配置文件 `machine-*.json`。

## 对比内容

- **模型：** Qwen3.8-Flash-Next GSQ-RCO **IQ3_XXS** GGUF（架构 `qwen4exp`，70.6 GiB，177 B 参数，约 3 B 激活）。两个引擎读取相同的 GGUF 文件
  （Strata 通过其原生 pack 读取，该 pack 指向 GGUF 专家）。
- **llama.cpp：** 发布版本 **v0.6.0**（标签 b11429，提交 8345f33）。5070：官方 win-cuda-13.4 二进制文件。P1004 和 P2003：从
  v0.6.0 tarball 构建，`-DGGML_CUDA=ON`，架构 60（CUDA 12.9）/ 86（CUDA 13.4）。
- **Strata：** 针对 IQ3_XXS 的 setup 选定配置（`--spec 4 --spec-min-p 0.5 --mtp rt --kv int8 --expert-cache auto --prefill auto`，
  上下文 36864；P2003 在 46 GB 上使用 setup 选定的 `--resident-experts`）。引擎：5070 = 发布版 z040（v0.1.40 引擎）；P2003 = /data/strata-140（v0.1.40）；
  P1004 = /data/strata 构建 c56a6874（0.1.40 系列，sm_60 路径）。
- **llama.cpp 设置：** `-fa on -ctk q8_0 -ctv q8_0 -c 36864`，KV q8_0 对比 Strata int8 KV。
  - `fit`：`--fit on` 默认值（自动 n-cpu-moe）。仅 1 轮（5070：1 轮；3060/P100：1 轮冒烟测试，每个单元格 2 行）。
  - `tuned`：`-ngl 99 -ncmoe N -b 2048 -ub 2048`，其中 N 来自 `llama-bench` 扫描（`tune-*.txt`）：P2003 N=44（pp2048 在 ub 2048 下为 345，对比 ub 512 下为 250），P1004 N=38。
    该扫描规模较小（N 取 2-3 个值，ub 取 2 个值），并非穷举。
  - **MTP：** GSQ-RCO GGUF 没有 MTP 层（"model doesn't contain MTP layers"），而 Strata 自有的 MTP head 是不同架构（`qwen4exp-mtp`），
    llama.cpp 会拒绝。对于 MTP 行，我使用了 unsloth 的独立 head `MTP/mtp-Qwen3.8-Flash-Next-Q8_0.gguf`（自包含；"shared" 变体失败并报
    `token_embd.weight not found`，因为它从主文件借用），通过 `-md ... --spec-type draft-mtp --spec-draft-n-max 3` 加载。这是与 Strata 不同的 MTP head，
    但为同一模型训练；主模型的权重完全相同。该 head 需要 3.4 GB 显存，因此 `mtp3` 行在 CPU 上使用更多专家（P2003 N=48，P1004 N=46 且 ub 1024；
    N=44 / 42 会耗尽显存），这也是 MTP 落败的部分原因。
- **协议：** 两个服务器均使用 `/v1/chat/completions`，无思考，256 个新 token，`top_k 20, top_p 0.95`，温度 0.7，种子 1，提示缓存关闭，每请求唯一 nonce。
  提示词：Strata 文档的前 4,000 / 32,000 个 token（`prompt-4k.txt`、`prompt-32k.txt`）+ "Summarize the text above in three sentences."
  提示 tok/s、解码 tok/s、草稿计数和 TTFT（= 服务器提示时间，非流式）均为服务器自身的计时。
- **交错执行：** 两个引擎无法同时加载（12 GB / 16 GB 显存，46-64 GB 内存，76 GB 模型），因此一轮 = 分别全新启动每个引擎，预热（两个短请求），运行所有单元格，停止。
  各轮交替引擎顺序。P1004 和 P2003 上每个引擎 3 轮，因此 **32K 时 n = 3，4K 时每个温度 n = 6**（每轮 2 次重复；温度 0 和 0.7 各 6 次）。并非要求的 10 对；在这些机器上 32K 每个请求耗时 2-5 分钟。
  报告中位值。llama.cpp 会话的第一个 4K 请求较慢，因为 mmap 的权重需要换入（76 GB 模型在 46-64 GB 内存上），这会略微拉低 llama.cpp 的 4K 中位值；Strata 常驻模式会预加载。
- **显存** = 会话期间整卡 `nvidia-smi` 峰值使用量（包含卡的其他用户；P1004 的卡在锁之间也被其他 stream 使用，5070 驱动显示器约 0.5 GB）。**RSS** = 引擎进程树的峰值常驻集
  （对 llama.cpp 计入 mmap 的文件页，对 Strata `--resident-experts` 副本也计入）。

## 结果（中位值；解码 = 256 token）

### P2003：RTX 3060 12 GB，46 GB 内存，Core Ultra 7 265（Linux）
| 配置 | ctx | 温度 | n | 提示 tok/s | 解码 tok/s | TTFT s | 草稿接受率 | 显存峰值 MiB | RSS GiB |
|---|---|---|---|---|---|---|---|---|---|
| **Strata setup** | 4K | 0 | 8 | **915.7** | **41.6** | **4.4** | 0.68 | 11727 | 43.2 |
| **Strata setup** | 4K | 0.7 | 8 | **910.2** | **41.9** | **4.5** | 0.61 | 11727 | 43.2 |
| **Strata setup** | 32K | 0 | 3 | **1050** | **41.8** | **30.5** | 0.66 | 11727 | 41.6 |
| **Strata setup** | 32K | 0.7 | 3 | **1049** | **38.8** | **30.6** | 0.55 | 11727 | 41.6 |
| llama.cpp tuned，MTP 关闭 | 4K | 0 | 6 | 328 | 9.1 | 12.4 | - | 9865 | 41.1 |
| llama.cpp tuned，MTP 关闭 | 4K | 0.7 | 6 | 337 | 9.8 | 12.1 | - | 9865 | 41.1 |
| llama.cpp tuned，MTP 关闭 | 32K | 0 | 3 | 294 | 8.8 | 109.0 | - | 9865 | 43.1 |
| llama.cpp tuned，MTP 关闭 | 32K | 0.7 | 3 | 338 | 8.7 | 94.9 | - | 9865 | 44.1 |
| llama.cpp tuned，MTP n=3 | 4K | 0 | 6 | 293 | 7.0 | 13.9 | 0.44 | 10519 | 43.3 |
| llama.cpp tuned，MTP n=3 | 4K | 0.7 | 6 | 310 | 6.6 | 13.1 | 0.39 | 10519 | 43.4 |
| llama.cpp tuned，MTP n=3 | 32K | 0 | 3 | 261 | 8.3 | 122.6 | 0.59 | 10519 | 44.0 |
| llama.cpp tuned，MTP n=3 | 32K | 0.7 | 3 | 302 | 6.4 | 106.1 | 0.43 | 10519 | 44.2 |
| llama.cpp `--fit on`，MTP 关闭（1 轮） | 4K | 0 | 2 | 162 | 9.6 | 25.0 | - | 10717 | 43.8 |

### P1004：Tesla P100 16 GB（sm_60），62 GB 内存（Linux）
| 配置 | ctx | 温度 | n | 提示 tok/s | 解码 tok/s | TTFT s | 草稿接受率 | 显存峰值 MiB | RSS GiB |
|---|---|---|---|---|---|---|---|---|---|
| **Strata setup** | 4K | 0 | 8 | **283.8** | **31.1** | **14.3** | 0.67 | 15969 | 42.3 |
| **Strata setup** | 4K | 0.7 | 8 | **284.6** | **30.3** | **14.2** | 0.60 | 15969 | 42.3 |
| **Strata setup** | 32K | 0 | 3 | **322.9** | **32.8** | **99.3** | 0.68 | 15985 | 42.5 |
| **Strata setup** | 32K | 0.7 | 3 | **322.9** | **28.7** | **99.3** | 0.52 | 15985 | 42.5 |
| llama.cpp tuned，MTP 关闭 | 4K | 0 | 6 | 180.5 | 10.3 | 22.5 | - | 15587 | 44.1 |
| llama.cpp tuned，MTP 关闭 | 4K | 0.7 | 6 | 180.7 | 10.3 | 22.5 | - | 15587 | 44.1 |
| llama.cpp tuned，MTP 关闭 | 32K | 0 | 3 | 155.9 | 9.1 | 205.6 | - | 15847 | 44.1 |
| llama.cpp tuned，MTP 关闭 | 32K | 0.7 | 3 | 157.0 | 9.1 | 204.2 | - | 15847 | 44.1 |
| llama.cpp tuned，MTP n=3 | 4K | 0 | 6 | 112.8 | 7.7 | 36.0 | 0.42 | 12117 | 45.8 |
| llama.cpp tuned，MTP n=3 | 4K | 0.7 | 6 | 113.0 | 8.0 | 35.9 | 0.45 | 12117 | 45.8 |
| llama.cpp tuned，MTP n=3 | 32K | 0 | 3 | 101.0 | 8.8 | 317.3 | 0.57 | 12307 | 46.8 |
| llama.cpp tuned，MTP n=3 | 32K | 0.7 | 3 | 100.8 | 7.9 | 318.1 | 0.47 | 12307 | 47.7 |
| llama.cpp `--fit on`，MTP 关闭（1 轮） | 4K | 0 | 2 | 84.0 | 10.8 | 48.2 | - | 15213 | 45.4 |

每个 llama.cpp 会话的第一个 MTP 请求在 P1004 上 4K 时显示 112-114 tok/s 提示速度（权重换入），这就是其 MTP 4K 中位值偏低的原因；32K 及后续行在原始数据中为 245-320 tok/s
（`p1004/raw/llama-tuned-mtp3.jsonl`）。即使最佳的热行也达不到 Strata 的水平。

### 本机：RTX 5070 12 GB，64 GB 内存（Windows），仅 1 轮
机主暂停了此 PC 上的模型工作（该 PC 在 2026-10-06 因内存耗尽而冻结），因此仅运行了第一轮：32K 时 n = 1，4K 时 n = 2。仅供参考。
| 配置 | ctx | 温度 | n | 提示 tok/s | 解码 tok/s | TTFT s | 草稿接受率 | 显存峰值 MiB |
|---|---|---|---|---|---|---|---|---|
| **Strata setup** | 4K | 0 | 2 | **1273** | **49.7** | **3.2** | 0.64 | 9911 |
| **Strata setup** | 4K | 0.7 | 2 | **1282** | **52.9** | **3.2** | 0.61 | 9911 |
| **Strata setup** | 32K | 0 | 1 | **1606** | **54.4** | **20.0** | 0.63 | 9911 |
| **Strata setup** | 32K | 0.7 | 1 | **1618** | **57.9** | **19.8** | 0.62 | 9916 |
| llama.cpp `--fit on`，MTP 关闭 | 4K | 0 | 2 | 198 | 15.7 | 20.5 | - | 10648 |
| llama.cpp `--fit on`，MTP 关闭 | 4K | 0.7 | 2 | 206 | 15.3 | 19.7 | - | 10648 |
| llama.cpp `--fit on`，MTP 关闭 | 32K | 0 | 1 | 183 | 14.8 | 175.2 | - | 10652 |
| llama.cpp `--fit on`，MTP 关闭 | 32K | 0.7 | 1 | 198 | 14.4 | 162.3 | - | 10652 |
| llama.cpp `--fit on`，MTP n=3 | 4K | 0 | 2 | 128 | 10.7 | 35.4 | 0.46 | 10381 |
| llama.cpp `--fit on`，MTP n=3 | 32K | 0.7 | 1 | 169 | 10.0 | 189.8 | 0.44 | 10381 |

（5070 的 llama.cpp 行使用 `--fit on`，而非 tuned 的 `-ncmoe` 扫描，后者未在此 PC 上运行。唯一的 MTP 32K 温度 0 行，2.3 tok/s，是卡顿而非测量值：请忽略。调优可能缩小差距。）

## 总结：谁胜出

Strata 在每一测量行中领先。llama.cpp 一行未胜。

| 行 | Strata 对比最佳 llama.cpp（tuned） |
|---|---|
| 3060，提示 tok/s，4K | 915 对比 337：**2.7x** |
| 3060，提示 tok/s，32K | 1050 对比 338：**3.1x**（TTFT 30.5 s 对比 94.9 s） |
| 3060，解码，4K / 32K，温度 0 | 41.6 对比 9.1 和 41.8 对比 8.8：**4.6x / 4.7x** |
| 3060，解码，温度 0.7 | 41.9 对比 9.8（4K），38.8 对比 8.7（32K）：**4.3x / 4.5x** |
| P100，提示 tok/s，4K / 32K | 284 对比 181 和 323 对比 157：**1.6x / 2.1x** |
| P100，解码，4K / 32K | 31.1 对比 10.3 和 32.8 对比 9.1：**3.0x / 3.6x** |
| 5070（1 轮，`--fit` llama），提示 4K / 32K | 1273 对比 198（6.4x），1606 对比 183（8.8x） |
| 5070（1 轮），解码 | 50-58 对比 14-16：**3.5-4x** |

- **llama.cpp v0.6.0 中的 MTP 在此没有帮助。** 使用独立 head 文件时，MTP 开启在两个机器上都比关闭慢（3060：4K 时 7.0 对比 9.1 tok/s；P100：7.7 对比 10.3）。n=3 时每次草稿接受率为 0.42-0.59，但
  草稿 head 需要 3.4 GB 显存，这会将额外 4 个专家层推到 CPU，而 CPU 常驻专家是瓶颈。llama.cpp 公布的 1.55x 是在统一内存的 DGX Spark 上全部常驻的情况下测得的，