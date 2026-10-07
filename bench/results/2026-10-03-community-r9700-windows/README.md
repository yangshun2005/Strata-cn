# 社区基准测试：Radeon AI PRO R9700 (gfx1201)、Windows 11、Ryzen 9 9950X

测量于 2026-10-03，由 [HarukiOtaku](https://github.com/HarukiOtaku) 完成。Strata **0.1.38**
（预构建 Windows HIP 引擎），原始 Flash-Next **IQ2_XS**，单 GPU，上下文上限
262,144 tokens。贪心采样，256-token 上限，关闭推理，每个长度运行三次；提示和
解码数据为引擎自身的日志值。

| 提示 tokens | 复用 | 生成 | 运行次数 | 提示 tok/s（中位数，范围） | 解码 tok/s（中位数，范围） | TTFT 秒（中位数，范围） | 总计 秒 |
| ---: | ---: | --- | ---: | --- | --- | --- | ---: |
| 4,198 (4,195–4,199) | 0 | 256 (236–256) | 3 | **855.8** (820.7 – 863.5) | **99.8** (95.3 – 101.3) | 4.95 (4.89 – 5.16) | 7.4 |
| 33,467 | 0 | 256 | 3 | **1,182.1** (1,181.1 – 1,191.2) | **97.8** (95.3 – 101.2) | 28.39 (28.18 – 28.43) | 30.9 |
| 131,388 | 0 | 256 (249–256) | 3 | **1,231.7** (1,225.0 – 1,233.3) | **89.9** (86.2 – 91.0) | 106.90 (106.77 – 107.49) | 109.7 |
| 247,990 | 0 | 256 (250–256) | 3 | **1,130.1** (1,127.2 – 1,132.0) | **87.5** (75.1 – 87.5) | 219.84 (219.49 – 220.42) | 222.7 |

**请以区间来理解中位数。** 同一引擎会话内的三次重复在
32K/128K/248K 处一致到 <1%，在 4K 处约 5%，但同一配置在新会话中重新测量时
波动可达 **32K 处 11.5%**（中位数 1,182.1 vs 1,060.6）和 **4K 处 14.2%**（855.8 vs
749.3）——若比较单次重复则为 13.4% 和 18.4%——因此请将中位数视为 ±10–15%，
而非固定数值。只有 4K 和 32K 有重复会话：**131,388 和 247,990 各自仅在一个会话中运行**
（唯一其他 128K 请求位于被排除的无效尝试中）。

表中的每次运行都读取了完整提示（`reused = 0`），日志中未出现停滞行。在约 33.5K 和约 131.4K 提示 tokens 处，
深度 10/50/90% 共发现六次针检查（**6/6**）——每次运行各一次，因此这是一个结果而非成功率。一次较早的尝试
几乎没有产生可用答案（其 15 行中有 13 行为空），在下方报告为
失败，而非数据。

`docs/AMD_HIP.md` 将超过 16K 的长上下文在 RDNA4 上标记为**未验证**，维护者的 R9700 验证为 Linux 上 4K/16K；
我能找到的 Windows AMD 报告
（[#499](https://github.com/Niko1221/Strata/issues/499)，RX 9070 XT——仍是一个**开放的、
未合并**的 PR）测量了到 31,059 tokens 的提示。此处的新颖之处在于 **Windows + gfx1201 达到 248K**，
而非硬件本身。

值得明确说明上游声称和不声称的内容。`docs/AMD_HIP.md` 记录了一个
已验证的 Windows AMD 案例——RX 9070 XT 在 Windows 11 上，引擎**在该
PC 上编译**，32K，29.2 tok/s 解码 / 约 181 tok/s 预填充（0.1.34）——然后说：
"现成的 zip 本身尚未在独立显卡上运行过模型——请报告。"本次运行就是那份
报告，在不同的显卡（R9700，gfx1201）上并通过现成的 zip。它**不是**
项目验证的支持：`docs/AMD_HIP_PERFORMANCE.md` 将其声称中的"Windows HIP、其他 AMD
架构"排除在外，而 `docs/AMD_HIP.md` 称其 RDNA4 表格"不得被解读为
对每一次后续变基的基准测试"，并且"长上下文压力测试、广泛的答案质量
等价性、其他 AMD 显卡以及混合厂商推理在此均未验证"。

**这不是一项受控研究：** 一个操作系统会话、一台机器、一种量化；提示为合成
代码；GPU 时钟和功耗限制未记录；桌面环境未隔离。

## 硬件和软件

- **GPU：** Radeon AI PRO R9700 32 GiB（gfx1201；Windows 报告 32 GB，引擎识别为 31.9 GiB），
  默认时钟，**功耗限制未记录**。显示器运行在集成 GPU 上，因此该
  显卡仅用于计算。
- **PCIe：** 引擎探测 `54.7 GB/s host->device (best of 45.5 54.4 54.7 54.6)`；
  **宽度/代数未测量**。
- **CPU / RAM / 存储：** Ryzen 9 9950X 16 核（AVX-512；引擎运行其默认的 15
  个专家池工作线程 + 主机线程）；Windows 报告 **61.5 GiB 总物理内存**
  （DIMM 套件未记录，因此不声称套件容量）；`D:` 上的 2 TB 级 NVMe
  （**设备型号未记录**）。
- **操作系统 / 驱动：** Windows 11 Pro，版本 **10.0.26300**，AMD 驱动 **32.0.31036.15**
  （iGPU 32.0.11024.2）——引自 [PROVENANCE.txt](PROVENANCE.txt) 中的机器信息。
- **引擎：** 仓库 `main` `99f3dbd0b21d1401b3769e0c0d963913607f380b`；预构建 Windows HIP
  引擎 **0.1.38**（每个会话记录 `session is up (engine 0.1.38)`；其 `engine/BUILD.json`
  显示 prebuilt、hip、0.1.38、ROCm 10.2.0a20260930、hipBLASLt 100500、gfx1201——**逐字
  引用在 [PROVENANCE.txt](PROVENANCE.txt) 中**，无构建哈希）。现成的
  `strata-windows-x64-hip.zip`；HIP 运行时 `engine/amdhip64_7.dll`。
- **未捕获：** `docs/AMD_HIP.md` 要求的 Windows-AMD 捆绑信息
  （`strata-device --list-devices`、`--selftest`、引擎日志尾部）、hipBLASLt
  `STRATA_HIPBLASLT_VERBOSE=1`/`fallbacks=0` 检查、时钟/功耗/温度、每次运行的峰值
  内存、分页/OOM 计数器、每次运行的答案文本，以及 IQ2_XS 包和
  MTP 张量的哈希。使用的调优表是
  上游自己的 `tools/hip/gfx1201-hipblaslt-100500.txt`（32 行，hipBLASLt 1.5.0 /
  ROCm 10.2.0a 代——该表基于 2026-09-14 的 nightly 校准，而引擎
  报告的是 2026-09-30 的，上游警告不同的 1.5.0 构建可能对其
  解决方案编号不同），日志显示其已启用，但未
  通过 `STRATA_HIPBLASLT_VERBOSE=1`/`fallbacks=0` 验证，且上游测量的增益
  为在 4,210-token 提示上 +3.9%。

所测量会话的引擎状态，摘自 `engine-0.1.38.log`（逐字
摘录）：

```text
expert cache auto: 25.78 GiB free, 700 MiB reserved (+143 MiB for the draft head) -> 17739 slots
expert cache 18575 slots, 24.95 GiB of VRAM
pre-filled 18575 of 18575 slots from the profile; slot 0 verified
FileExpertSource: RAM budget 16.00 GiB: 6001 of the 8.07 GiB of experts the GPU cache does not hold
the file tier reads unbuffered (5 of 16 probe reads from the file cache; 46.0 GiB available, 36.5 GiB of files)
Windows budgets 31704 of this card's 32624 MiB for this process; free VRAM is counted within that
KV streaming: 32768 of 262144 cells per QSA layer in VRAM, the K/V in 3.09 GiB of pinned RAM
the prompt path borrows 3203 CUDA0 cache slots (4.34 GiB)
PLE on, table 320001536 rows
15 expert-pool workers + the host thread
```

之前的引擎（试点，131,072 上下文）记录了 `large pages refused for 35456548864 B
(GetLargePageMinimum=2097152, VirtualAlloc error 1314); using 4 KB pages`，保存在
[pilot-0.1.36-engine-startup.log](pilot-0.1.36-engine-startup.log) 中；"Lock pages in memory" 未
启用，#42 将其记录为错误 1314 的预期桌面结果。

## 模型和配置

- `ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF` @
  `ed59f92082b1e93c0e96d60a8b11aab089b52f09`（自 2026-09-29 起未变，早于本
  机器 2026-10-02 的下载）。分片
  `IQ2_XS/Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf` 和 `...-00002-of-00002.gguf`。
  **两个分片及 `data/expert-profile.bin` 的 SHA-256 在
  [PROVENANCE.txt](PROVENANCE.txt) 中**（包和 MTP 草稿张量无哈希）。
- MTP 草稿头来自 `Qwen/Qwen3.8-Flash-Next`（修订版本未记录）；日志称其为
  `experimental native Q5_K head, 337715200 bytes` 并加载 `835 MiB` 的草稿层。
  投机解码已开启（`--spec 4`、`--spec-min-p 0.5`）；草稿接受率在
  十二次头条运行中为 62.1–83.8%，在所有 36 个测量行中为 57–100%（低
  端是一次 13-token 针生成）——计数仅存在于日志中。
- 视觉关闭（无 `--vision`，无 mmproj；Windows AMD 无图像），无 `--low-ram`，无引擎
  校准，无自定义控制向量，无 API 密钥。`--kv int8` 和贪心 256-token
  采样来自启动配置；日志显示 KV 单元数但不显示
  dtype。

`strata-iq2_xs.json` 中写入的引擎参数（启动器启动 `serve/server.py`，
后者用这些参数生成引擎）：

```text
--pack  D:\AI\Strata\Strata-main\Strata-data\packs\iq2_xs
--native  D:\AI\Strata\Strata-main\Strata-data\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00001-of-00002.gguf
--ple-gguf D:\AI\Strata\Strata-main\Strata-data\models\IQ2_XS\Qwen3.8-Flash-Next-GSQ-RCO-IQ2_XS-00002-of-00002.gguf
--expert-profile D:\AI\Strata\Strata-main\Strata-main\data\expert-profile.bin
--expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
--mtp D:\AI\Strata\Strata-main\Strata-data\mtp\rt
--max-context 262144 --kv int8 --kv-resident 32768 --mmap-experts --resident-budget-gib 16
env STRATA_HIPBLASLT_TUNING=...\tools\hip\gfx1201-hipblaslt-100500.txt
```

## 方法

- [benchmark.py](benchmark.py)（仅标准库）从合成的
  类 Python 填充内容构建每个请求，以**字符**为单位确定大小（校准测得 2.52 字符/token，因此
  目标超出 2–3.5%），并在**每个提示中放置一个全新的 nonce**，使任何运行都无法复用
  缓存前缀——因此 `reused = 0` 是构造使然，而非因为缓存缺失。
- 每个系列先运行一次校准和一次 2,082-token 预热；两者均从
  表中排除，引擎加载亦然。推理关闭，因此第一个 token 即答案文本。
- **提示/解码 tok/s、token 计数和缓存状态为引擎的日志值；TTFT 和
  总延迟为客户端测量**（包含 HTTP + 分词 + 前端）。未使用
  "生成 tokens ÷ 总时间" 的数值。作为交叉检查，客户端计时器包围
  引擎自身的时钟且**从不更快**：TTFT ≥ `read_ms` 达 32–414 ms，
  总计 ≥ `read_ms + gen_ms` 达 17–395 ms，在每个表内行上均如此，因此引擎的计时
  被佐证为下界而非独立验证（其 token 计数完全没有
  外部检查）。
- 确切提示无法从此包中重建：填充内容是确定性的，但
  每请求的种子和 nonce 未保存（仅保存 `chars`），且速度行不保留
  答案文本。[benchmark.py](benchmark.py) 是指南所要求的可复现指令。
- 同一长度的三次重复是三个**不同**的、标称大小相同的提示（填充
  种子每次重复递增），因此每个中位数混合了提示间变化与
  运行间噪声。头条运行还使用**默认提示内核路径**——融合
  内核仅在 A/B 分支中。
- `docs/DETAILS.md` 为字节级相同重复和干净 A/B 分支推荐的开关
  （`--prompt-cache 0 --adapt-swaps 0 --pcie-frac 0`，加上 `STRATA_IQ_MT_MIN=1`）**未**
  使用，因此缓存适配和 PCIe 分数舍入可能在请求之间和
  分支之间移动结果——这是 10.8% 默认对默认差异的一个可能贡献因素。
- 日志**没有时间戳或请求 id**，因此一行通过 token
  计数绑定到日志行；测试框架的 `pairing_ok` 检查在这些运行之后才添加，且在这些
  文件中缺失。手动重新运行：所有 69 个带 `usage` 块的行（36 个测量、15 个无效、18 个
  校准/预热）在 ±1 内匹配，且无 `read_ms` 重复。
- `model` 字符串在标准/240K/无效文件中为 `qwen3.8-flash-next-iq2_xs`，
  在六个 A/B 文件中为 `strata`（测试框架的回退别名）。
- 提示目标是标签，而非大小；测量的 `engine_prompt_tokens` 按行
  报告。针放置在**字符**位置的 10/50/90% 处，且仓库的
  `tools/needle_bench.py` **未**使用，因此召回检查是本测试框架自己的。

完整方法细节、标签/会话注意事项及所有披露：见
[METHOD-AND-DISCLOSURES.md](METHOD-AND-DISCLOSURES.md)。

## 结果

每次运行的 JSON：[bench-output-standard.json](bench-output-standard.json)（4K/32K/128K +
针），[bench-output-240k.json](bench-output-240k.json)（248K）。"表内" = 九个 JSON 文件中
测量系列的 36 行（两个头条文件中 18 行，六个 A/B
分支中 18 行）；表格使用前 18 行。日志
（[engine-0.1.38.log](engine-0.1.38.log)）包含 71 个请求行：36 个表内、15 行
下方失败尝试、18 个校准/预热，以及 2 个其他（一个测试框架未产生的 246,933-token 请求，
和一个遗留的 0.1.36 试点行）。

**失败：** [bench-output-standard-first-attempt-void.json](bench-output-standard-first-attempt-void.json)
在测试框架禁用推理之前产生——其 15 行中有 13 行未返回答案文本
（7 个速度行在 256 生成 tokens 处，6 个针行在 32-token 上限处），所有六个针
均未命中，