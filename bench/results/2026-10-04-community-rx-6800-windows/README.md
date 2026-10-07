# 社区基准测试：RX 6800（Windows），0.1.39 对比 0.1.38

由 [did-technomancer](https://github.com/did-technomancer) 于 2026-10-04 测量。

```text
GPU      AMD Radeon RX 6800 16 GB (gfx1030), also drives two monitors
PCIe     engine probe 28.2 GB/s (pcie_frac 0.55)
CPU      AMD Ryzen 7 5700X3D, 8 cores / 16 threads, AVX2, no AVX-512
RAM      64 GB DDR4-3200 (4x16)
Storage  NVMe SSD (PCIe 3.0), not the system drive
OS       Windows 11 Pro for Workstations 26H2 (build 26300), system-managed page file
Driver   AMD 32.0.21043.10005
ROCm     10.2.0a20260930, hipBLASLt 1.5.0 (no tuning table for gfx1030)
Load     no other workloads, no power limits
0.1.38   release strata-windows-x64-hip.zip (99f3dbd); the same commit built here gave the same numbers
0.1.39   tag v0.1.39 (6f32ec0) built here, tools\hip\build_windows.bat, STRATA_HIP_ARCHS=gfx1030
Model    ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF @ ed59f92: IQ2_XS, IQ3_XXS, Q2_0
```

tok/s。包含多个值的单元格列出了单独的运行结果，中位数在括号中。"0.1.39s" = 0.1.39 加上 `STRATA_SH_STREAM=0`。

**各版本，IQ2_XS**

| 引擎 | 运行次数 | 英语 | 乌克兰语 | 代码 | 提示词 7.9K | 提示词 30.3K |
| --- | ---: | --- | --- | --- | ---: | ---: |
| 0.1.38，发布版 zip | 2 | 50.5 / 55.2 (52.9) | 40.8 / 42.3 (41.6) | 57.2 / 55.8 (56.5) | 319 / 326 | 309 / 310 |
| 0.1.38，本地构建 | 1 | 54.2 | 41.7 | 56.6 | 319 | 310 |
| 0.1.39 | 2 | 42.6 / 42.5 (42.6) | 32.9 / 32.5 (32.7) | 49.2 / 50.2 (49.7) | 326 / 326 | 310 / 311 |
| 0.1.39 + `STRATA_SH_STREAM=0` | 4 | 56.4 / 57.0 / 55.5 / 54.2 (56.0) | 43.5 / 44.3 / 43.2 / 43.0 (43.4) | 62.7 / 62.9 / 60.5 / 61.5 (62.1) | 326 | 311 |

专家缓存 5,361-5,486 个槽位（7.2-7.4 GiB），全部加载后剩余 1,729-1,761 MiB 显存。

局限性：

- 仅一台 PC；大多数配置只运行了一次（少于三次）；重复运行差异最高达 9%（英语）、4%（乌克兰语、代码）
- 0.1.39 发布版 zip 未运行
- 未保存每次请求生成的 token 数量；未测量 RAM
- 未测试：`"parallel"` 槽位中的长提示词、对话缓存、图像、0.1.39 上超过 128K 的上下文、IQ3_S

<details>
<summary>配置与方法</summary>

```text
strata.exe --pack <data>\packs\iq2_xs
  --native <data>\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf
  --ple-gguf <data>\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf
  --expert-profile data\expert-profile.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
  --mtp <data>\mtp\rt --max-context 131072 --kv int8 --kv-resident 32768 --vision --vram-reserve-mib 2048
```

- Pack、MTP 层、专家配置文件和草稿词表：使用捆绑的版本，由 setup 准备就绪
- 低内存模式关闭；7 个池工作线程；校准和速度预测关闭；CPU 图像编码器已加载但未使用
- [ab.py](ab.py)：每个测试臂通过一项更改重启服务器；专家缓存按配置文件填充
- 请求：`/v1/chat/completions`，`reasoning_effort none`，`temperature 0`，每个提示词以随机 id 开头
  （0 个复用 token）
- 解码：3 次英语解释的中位数（420 token 上限）、2 次乌克兰语（520）、1 个 Python 模块（700），
  `timings.predicted_per_second`
- 提示词：从 `serve/server.py` 截取的 7,858 和 30,348 个 token，40 token 答案上限；"仅解码"测试臂跳过此项
- 内存：专家缓存槽位和空闲显存来自引擎的启动日志（快照，非峰值）；模型加载
  不计入任何计时
- [par.py](par.py)：N 个流式请求一起发送，300 token 上限（生成 246-287）；首个 token = 客户端收到的第一个
  非空答案增量

</details>

<details>
<summary>各版本，IQ3_XXS 加上 `--draft-vocab cyrillic`（仅解码，各运行 1 次）</summary>

| 引擎 | 英语 | 乌克兰语 | 代码 | 专家缓存槽位 |
| --- | ---: | ---: | ---: | ---: |
| 0.1.38，发布版 zip | 42.2 | 45.7 | 42.8 | 3,895 |
| 0.1.39 + `STRATA_SH_STREAM=0` | 49.2 | 55.2 | 52.0 | 4,219 |

</details>

<details>
<summary>0.1.39 解码开关，逐一关闭（IQ2_XS，仅解码，各运行 1 次）</summary>

| 更改 | 英语 | 乌克兰语 | 代码 |
| --- | ---: | ---: | ---: |
| 无 | 42.5 | 32.5 | 50.2 |
| `STRATA_SH_STREAM=0` | 56.4 | 43.5 | 62.7 |
| `STRATA_IQ_STAGE_GRID=0` | 43.5 | 34.6 | 51.8 |
| `STRATA_PLE_BATCH=0` | 41.6 | 33.0 | 49.0 |
| `STRATA_OLD_IQ_MMVQ=1` | 43.3 | 32.5 | 47.9 |
| `STRATA_DEC_BATCH=0` | 40.0 | 31.8 | 45.4 |

</details>

<details>
<summary>设置（各运行 1 次；"—" 表示同一引擎的基准）</summary>

| 引擎，模型 | 设置 | 英语 | 乌克兰语 | 代码 | 接受的草稿 |
| --- | --- | ---: | ---: | ---: | ---: |
| 0.1.38, IQ2_XS | —（2 次运行） | 50.5 / 55.2 | 40.8 / 42.3 | 57.2 / 55.8 | 63% / 65% |
| 0.1.38, IQ2_XS | `--draft-vocab cyrillic` | 56.6 | 61.4 | 57.3 | 75% |
| 0.1.38, IQ2_XS | `--pool-workers 11` | 50.6 | 40.1 | 57.6 | 62% |
| 0.1.38, IQ2_XS | `--pool-workers 15` | 37.7 | 28.6 | 39.1 | 63% |
| 0.1.39s, IQ2_XS | — | 57.0 | 44.3 | 62.9 | 63% |
| 0.1.39s, IQ2_XS | `--draft-vocab cyrillic` | 58.0 | 63.2 | 63.4 | 73% |
| 0.1.39s, IQ2_XS | `--spec 6` | 51.4 | 40.6 | 59.3 | 55% |
| 0.1.39s, IQ2_XS | `--spec-min-p 0.7` | 56.0 | 45.3 | 63.1 | 70% |
| 0.1.39s, IQ2_XS | `--pcie-frac 0` | 54.7 | 42.4 | 58.3 | 67% |
| 0.1.39s, IQ2_XS | `STRATA_ADAPT_NOWAIT=1` | 55.8 | 43.5 | 62.1 | 64% |
| 0.1.39s, IQ2_XS | `STRATA_RING_BYTES=0` | 58.2 | 43.3 | 61.6 | 63% |
| 0.1.39s, IQ3_XXS | — | 46.7 | 38.5 | 52.5 | 60% |
| 0.1.39s, IQ3_XXS | `--draft-vocab cyrillic` | 49.2 | 55.2 | 52.0 | 68% |
| 0.1.39s, IQ3_XXS | `STRATA_IQ256_GATHER=1` | 46.5 | 38.5 | 50.9 | 60% |

使用西里尔词表时，额外保留了 160-180 MiB 显存。`STRATA_RING_BYTES=0`：30.3K 提示词的读取速度为
304 tok/s，对比 311。

</details>

<details>
<summary>并发请求（0.1.39s，IQ2_XS，各 1 轮）</summary>

| | 队列 | `"parallel": 2` | `"parallel": 4` |
| --- | --- | --- | --- |
| 4 个请求：全部完成耗时 | 21.9 s | 23.5 s | 22.9 s |
| 4 个请求：合计 | 46.6 tok/s | 46.0 tok/s | 46.6 tok/s |
| 4 个请求：首个 token | 0.7 / 6.6 / 11.8 / 17.7 s | 0.8 / 1.6 / 12.3 / 13.6 s | 0.8 / 3.3 / 3.3 / 3.3 s |
| 2 个请求：全部完成耗时 | 11.9 s | 12.8 s | 13.3 s |
| 2 个请求：首个 token | 0.8 / 7.0 s | 1.1 / 2.0 s | 1.0 / 1.9 s |
| 1 个请求：英语 / 乌克兰语 / 代码 | 57.0 / 44.3 / 62.9 | 54.2 / 42.2 / 59.5 | 49.6 / 40.5 / 55.3 |
| 1 个请求：提示词 7.9K / 30.3K | 326 / 311 | 324 / 308 | 270 / 306 |
| 专家缓存槽位 | 5,484 | 4,394 | 3,278 |

</details>

<details>
<summary>模型大小（0.1.39s，默认草稿词表；IQ2_XS 运行 4 次，其余 1 次）</summary>

| | Q2_0 | IQ2_XS | IQ3_XXS |
| --- | --- | --- | --- |
| 英语 / 乌克兰语 / 代码 | 55.8 / 45.1 / 65.6 | 54.2-57.0 / 43.0-44.3 / 60.5-62.9 | 46.7 / 38.5 / 52.5 |
| 提示词 7.9K / 30.3K | 336 / 321 | 326 / 311 | 324 / 309 |
| 专家缓存槽位 | 5,656 (7.28 GiB) | 5,361-5,484 (7.2-7.4 GiB) | 4,218 (6.86 GiB) |

</details>

<details>
<summary>上下文大小（0.1.38，IQ2_XS，2026-10-03）</summary>

| 上下文 | 显存预留 | 缓存槽位 | 解码（3 次运行） | 提示词 2.2K | 提示词 7.9K | 提示词 30.3K |
| --- | ---: | ---: | --- | ---: | ---: | ---: |
| 8K | 700 MiB | 6,288 | 52.8 / 59.7 / 61.0 | 206 | 未测量 | 未测量 |
| 128K | 2,048 MiB | 5,583 | 47.1 / 53.0 / 54.9 | 223 | 327 | 306 |
| 256K | 2,048 MiB | 4,791 | 45.5 / 53.0 / 55.0 | 203 | 325 | 302 |
| 512K (yarn, x2) | 2,048 MiB | 4,420 | 46.9 / 48.5 / 50.2 | 205 | 328 | 304 |

长提示词（五个"秘密代码"分别位于长度的 10/30/50/70/90% 处，1 次运行，0 个复用 token）：

| 上下文 | 提示词 token | 读取时间 | 读取 tok/s | 解码 | 找到的代码 |
| --- | ---: | ---: | ---: | ---: | ---: |
| 128K | 118,410 | 375 s | 316 | 36.2 | 5 / 5 |
| 256K | 249,365 | 825 s | 302 | 48.7 | 5 / 5 |
| 512K | 512,194 | 1,968 s | 260 | 33.0 | 5 / 5 |

复用前缀（128K，一个 64,885 token 的文件）：第一个问题 198.1 s（328 tok/s），五个后续问题各新增 25-33 个 token，
每个耗时 0.3-0.5 s。

没有请求失败或被取消。

</details>

<details>
<summary>正确性任务（0.1.39s，推理关闭，1 次运行，`strata_eval2.py hard js sql regex`）</summary>

| 集合 | 任务 | Q2_0 | IQ2_XS | IQ3_XXS |
| --- | --- | --- | --- | --- |
| Python | topo_sort | ✓ 8.0 s | ✓ 9.2 s | ✓ 9.8 s |
| Python | wildcard_match | ✓ 5.7 s | ✓ 6.6 s | ✓ 7.0 s |
| Python | min_window | ✓ 3.7 s | ✓ 4.0 s | ✓ 5.5 s |
| Python | json_pointer | ✗ 8.8 s | ✓ 18.9 s | ✓ 22.9 s |
| Python | nqueens | ✓ 3.3 s | ✓ 3.6 s | ✓ 5.8 s |
| Python | semver_sort | ✓ 14.6 s | ✓ 7.4 s | ✓ 22.7 s |
| Python | edit_distance | ✓ 3.5 s | ✓ 2.7 s | ✓ 4.4 s |
| Python | gather_limited | ✓ 3.9 s | ✓ 4.9 s | ✓ 6.2 s |
| Python | trie | ✓ 6.1 s | ✓ 6.7 s | ✓ 9.4 s |
| Python | bowling | ✗ 5.8 s | ✓ 5.8 s | ✓ 7.3 s |
| Python | palindrome | ✓ 3.0 s | ✓ 3.2 s | ✓ 4.8 s |
| Python | normpath | ✓ 6.7 s | ✓ 19.4 s | ✓ 11.0 s |
| JavaScript | groupBy | ✓ | ✓ | ✓ |
| JavaScript | deepEqual | ✗ | ✗ | ✓ |
| JavaScript | parseQuery | ✓ | ✓ | ✓ |
| JavaScript | chunk | ✓ | ✓ | ✓ |
| SQL | customers without orders | ✓ | ✓ | ✓ |
| SQL | the two customers with the highest total | ✓ | ✓