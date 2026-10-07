# 在 RTX 5080 + Ryzen 9 9900X3D 上的社区基准测试

由 kalabaddon 于 2026-10-05 测量。

* **机器：** RTX 5080 16 GB，运行在 **PCIe 5.0 x8**，Ryzen 9 9900X3D，96 GiB DDR5-6000，Windows 11 25H2，Strata 引擎 0.1.39（提交 `6f32ec0`）。
* **测试对象：** 五种配置下的四个模型：IQ3_S、Unsloth UD-IQ4_XS（两次，分别在 `START-HERE.bat --calibrate` 之前和之后）、Unsloth UD-Q4_K_XL 和 Coder IQ1_M。
* **速度测试：** 在 1k、4k、32k、64k、128k 和 262k 提示词 token 下各运行 3 次，输出上限 256 token，贪婪解码，关闭推理。
* **大海捞针测试：** 在 IQ3_S、校准后的 UD-IQ4_XS、UD-Q4_K_XL 和 Coder IQ1_M 上共 9 个用例（32k / 128k / 256k，深度分别为 10 / 50 / 90%）。
* **主要注意事项：** 许多回答在达到 256 token 上限之前就停止了，因此解码数据不如提示词数据可靠。

## 摘要

四个模型都找到了 9 个大海捞针中的 9 个。速度方面，引擎报告的中位数（提示词范围覆盖 4k-262k，1k 单独列出，因为对所有模型来说它都低得多；解码范围覆盖全部六种尺寸）：

| 配置 | 提示词 tok/s，4k-262k | 提示词 tok/s，1k | 解码 tok/s | 262k 时的 TTFT |
|---|---|---|---|---|
| IQ3_S | 2,344-3,603 | 674 | 80-98 | 80.6 s |
| UD-IQ4_XS，校准前 | 1,498-2,424 | 483 | 39-51 | 174.9 s |
| UD-IQ4_XS，校准后 | 1,480-2,410 | 479 | 52-63 | 174.8 s |
| UD-Q4_K_XL | 1,238-1,861 | 382 | 44-51 | 179.8 s |
| Coder IQ1_M（缩减量化，见下文） | 3,175-4,001 | 1,431 | 86-117 | 73.6 s |

* **IQ3_S** 在每种尺寸下都是完整模型中最快的，无论是提示词速度、解码速度还是 TTFT：从 4k 起，其提示词速度约为 UD-IQ4_XS 各次运行的 1.5-2.2 倍，解码速度约为 1.4-2.5 倍，而在 262k 时其 TTFT 不到后者的一半。
* **UD-Q4_K_XL** 在每种尺寸下的提示词速度都是五个中最慢的，TTFT 也最长。其解码速度接近未校准的 UD-IQ4_XS，低于校准后的那次运行。它还具有最大的专家区域（66,482 MiB）和最小的 RAM 余量（引擎读数：93.6 GiB 中已使用 85.5 GiB）。
* **校准（UD-IQ4_XS）** 只改变了 `--pcie-frac`（从 0.55 到 0.35）。解码中位数上升了 13-44%；提示词速度变化在 -2% 到 +8% 之间。
* **Coder IQ1_M** 是 Coder 变体，采用小得多的 IQ1_M 量化（专家区域 23,981 MiB，而其他模型为 47,962-66,482 MiB），面向更受限的系统。测试它是希望获得大幅速度提升，但它相对 IQ3_S 的领先幅度不大：从 32k 起提示词速度约高 9-11%（4k 时高 35%，1k 时高 112%），解码速度参差不齐（大多数尺寸下更高，4k 和 128k 时更低）。它在 `--spec-min-p`（0.50 对 0.70）和专家配置上也与 IQ3_S 不同，因此即便这一差距也不仅仅归因于量化。

**写满 256 token 完整回答的运行**（而非提前停止，通常以类似 "I'll start by exploring the repository" 的一行结束）。这只是遵循任务的一个代理指标：一些完整长度的 IQ3_S 回答开头就说文件被截断了。

| 上下文 | IQ3_S | UD-IQ4_XS，校准前 | UD-IQ4_XS，校准后 | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| 1k-64k（共 12） | 4 | 12 | 12 | 11 | 3 |
| 128k（共 3） | 1 | 2 | 2 中的 1 \* | 1 | 2 |
| 262k（共 3） | 0 | 0 | 0 | 1 | 3 |
| 总计（共 18） | 5 | 14 | 17 中的 13 \* | 13 | 8 |

\* 一次校准后的 128k 运行因引擎错误失败，已排除（见结果）。

**限制比较的条件：** PCIe 5.0 x8 而非 x16；加载 IQ3_S 且上下文为 262,144 时仅有 249 MiB 空闲 VRAM；使用 UD-IQ4_XS 和 UD-Q4_K_XL 时 RAM 几乎占满；每种尺寸只有一个提示词；一次失败的请求（校准后的 UD-IQ4_XS，128k）；短回答使解码数据更加嘈杂。详情见局限性部分。

## 硬件和软件

* **GPU：** NVIDIA GeForce RTX 5080，16 GB（报告为 16,303 MiB），唯一的 GPU。显示器连接在它上面（`nvidia-smi` 显示 Disp.A On），因此桌面占用了一些 VRAM。
* **PCIe 链路：** Gen 5，**x8**（该卡支持 x16）。`nvidia-smi --query-gpu=pcie.link.gen.current,pcie.link.gen.max,pcie.link.width.current,pcie.link.width.max`，在一次请求期间每秒采样一次，持续 15 秒，每次都读到 5、5、8、16。据机主称，原因是与 NVMe 驱动器共享通道（未经独立验证）。
* **CPU：** AMD Ryzen 9 9900X3D（AM5）。
* **RAM：** 96 GiB DDR5（2x48 GiB Corsair Vengeance，4 个插槽中的 2 个），任务管理器显示为 6000 MT/s；引擎显示可用 93.6 GiB。
* **存储：** Samsung 970 EVO Plus 2 TB NVMe，启动盘也是唯一使用的盘；协商链路速度未检查。
* **软件：** Windows 11 25H2（内部版本 26200.9457）；NVIDIA 驱动 610.47（WDDM），`nvidia-smi` 显示 CUDA 13.3；Python 3.14。Strata 引擎 0.1.39（启动日志 "session is up (engine 0.1.39)"，与检出的 `CMakeLists.txt` 一致），提交 `6f32ec0`；发布二进制还是源码构建：未记录；无更改的构建选项。
* **后台：** Firefox 和少数其他应用打开；Steam 和其他游戏服务已停止；内存使用未另行最小化。
* **加载 UD-IQ4_XS 时的内存快照**（任务管理器，单次快照，非推理期间的峰值）：已使用 88.6 GiB（压缩 1.2 GiB），可用 5.1 GiB，缓存 5.0 GiB，已提交 157 GiB 中的 97 GiB，硬件保留 2.4 GiB。大约同一时间的 `nvidia-smi`：16,303 MiB VRAM 中已使用 15,692 MiB，360 W 上限中功耗 51 W。其他配置没有任务管理器快照；所有五个配置的引擎 RAM 读数见模型和配置下的表格。
* **还可用：** 该机器双启动 Linux，配有由两块 Crucial T705 1 TB PCIe Gen5 NVMe SSD（额定高达 13,600 MB/s）组成的软件 RAID（据信为 RAID 0），以及更低的 OS 开销。此处未使用；如果有人想要更快的存储对比，我可以在那里重新运行。

## 模型和配置

模型（均为 Qwen3.8-Flash-Next）：

* **IQ3_S：** 原始模型（非 Coder 或 Swift 变体），2 个分片，setup 模型菜单中的第一项，由 Strata 的 setup 提供。服务名为 `qwen3.8-flash-next-iq3_s`。
* **UD-IQ4_XS：** Unsloth 系列，3 个分片，setup 菜单中的实验性条目，由 Strata 的 setup 提供。main 分支上的 `docs/MODELS.md` 将该系列描述为 UD-Q4_K_XL，因此 UD-IQ4_XS 可能比该页面更新。服务名为 `qwen3.8-flash-next-unsloth-ud-iq4_xs`。
* **UD-Q4_K_XL：** Unsloth 系列，4 个分片，`docs/MODELS.md` 为该系列指定的模型。服务名为 `qwen3.8-flash-next-unsloth-ud-q4_k_xl`。
* **Coder IQ1_M：** Coder 变体，2 个分片，来自 Strata 的官方安装列表。服务名为 `qwen3.8-flash-next-coder-iq1_m`。

模型仓库和修订版本：未记录；所有 GGUF 均由 Strata 的 setup 下载。`<Strata-data>\packs\` 下的包由 setup 生成（未记录哈希）；专家配置为 `data\` 中随附的配置。无自定义草稿词汇表，视觉编码器关闭。

引擎启动标志。路径相对于 `<Strata-data>`（Strata 文件夹旁的数据文件夹）或 Strata 文件夹。

| 标志 | IQ3_S | UD-IQ4_XS，校准前 | UD-IQ4_XS，校准后 | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| `--pack` | `<Strata-data>\packs\iq3_s` | `<Strata-data>\packs\unsloth-ud-iq4_xs` | 相同 | `<Strata-data>\packs\unsloth-ud-q4_k_xl` | `<Strata-data>\packs\coder-iq1_m` |
| `--native` | `<Strata-data>\models\IQ3_S\Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf` | `<Strata-data>\models\unsloth-UD-IQ4_XS\Qwen3.8-Flash-Next-UD-IQ4_XS-00001-of-00003.gguf` | 相同 | `<Strata-data>\models\unsloth-UD-Q4_K_XL\Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf` | `<Strata-data>\models\coder-IQ1_M\Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf` |
| `--ple-gguf` | 模型的第二个分片 | 未使用 | 未使用 | 未使用 | `<Strata-data>\models\coder-IQ1_M\Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00002-of-00002.gguf` |
| `--expert-profile` | `data\expert-profile.bin`（随附） | `data\expert-profile.bin`（随附） | 相同 | `data\expert-profile.bin`（随附） | `data\expert-profile-coder.bin`（随附，Coder 专用） |
| `--expert-cache` | auto | auto | auto | auto | auto |
| `--prefill` | auto | auto | auto | auto | auto |
| `--spec` | 4 | 4 | 4 | 4 | 4 |
| `--mtp` | `<Strata-data>\mtp\rt` | `<Strata-data>\mtp\rt` | 相同 | `<Strata-data>\mtp\rt` | `<Strata-data>\mtp\rt` |
| `--max-context` | 262144 | 262144 | 262144 | 262144 | 262144 |
| `--kv` | int8 | int8 | int8 | int8 | int8 |
| `--kv-resident` | 32768 | 32768 | 32768 | 32768 | 32768 |
| `--resident-budget-gib` | 未设置 | 55 | 55 | 66 | 未设置 |
| `--pcie-frac` | 0.20 | 0.55 † | 0.35 | 0.20 | 0.20 |
| `--spec-min-p` | 0.70 | 0.50 † | 0.50 | 0.70 | 0.50 |

捕获的 UD-Q4_K_XL 引擎参数，本地路径已缩短（其他启动仅在表中所示处不同）：

```text
--pack <Strata-data>\packs\unsloth-ud-q4_k_xl
--native <Strata-data>\models\unsloth-UD-Q4_K_XL\Qwen3.8-Flash-Next-UD-Q4_K_XL-00001-of-00004.gguf
--expert-profile data\expert-profile.bin
--expert-cache auto --prefill auto --spec 4
--mtp <Strata-data>\mtp\rt
--max-context 262144 --kv int8 --kv-resident 32768
--resident-budget-gib 66 --pcie-frac 0.20 --spec-min-p 0.70
```

† 报告为该次运行生效（该标志在其配置中缺失）；只有校准后那次运行的标志列表被直接捕获。所有其他列均为启动时的标志列表。

每个配置首次 1k 运行时引擎报告的状态（来自每次运行 JSON 中保存的指标）：

| 引擎报告 | IQ3_S | UD-IQ4_XS，校准前 | UD-IQ4_XS，校准后 | UD-Q4_K_XL | Coder IQ1_M |
|---|---|---|---|---|---|
| 专家缓存槽位（VRAM） | 4,010（7,835 MiB） | 3,040（7,009 MiB） | 2,941（6,781 MiB） | 2,335（6,967 MiB） | 3,973（7,750 MiB） |
| 专家区域（RAM） | 47,962 MiB | 49,753 MiB | 49,981 MiB | 66,482 MiB | 23,981 MiB |
| 加载后空闲 VRAM | 249 MiB | 99 MiB | 464 MiB | 176 MiB | 383 MiB |
| RAM 使用（共 93.6 GiB） | 63.1 GiB | 68.8 GiB | 67.7 GiB | 85.5 GiB | 45.0 GiB |

其他设置：关闭推理（`reasoning_effort: none`），贪婪（温度 0），关闭视觉，CPU 工作线程 11（引擎报告的 `pool_workers`；未在命令行设置）。未检查低 RAM 模式和实验性速度预测；IQ3_S 启动日志没有控制向量行，Strata 文档称该行在预测开启时出现。

## 方法

* **脚本：** `bench/results/2026-09-29-rtx3090-epyc-milan/data/` 中的 `strata-bench.py`（仅标准库），从 Strata 仓库根目录针对运行中的服务器运行，每个配置运行一次，使用各自的 `--label` 和 `--log`。显示了 UD-Q4_K_XL 命令；其他仅更改了 `--label` 和 `--log`（标签为 `iq3_s`、`unsloth-ud-iq4_xs`、`unsloth-ud-iq4_xs_calibrated`、`coder-iq1_m`）。密钥已移除：

```text
python bench\results\2026-09-29-rtx3090-epyc-milan\data\strata-bench.py --url http://127.0.0.1:8080 --api-key <API_KEY> --label unsloth-ud-q4_k_xl --contexts 1k,4k,32k,64k,128k,262k --log strata-unsloth-ud-q4_k_xl.log --out bench\results\2026-10-05-community-rtx5080-9900x3d
```

* **提示词：** 来自 Strata 仓库自身源码和文档的文本，按字符数截取，后接 "explain what the last file above does, then propose one concrete improvement as a unified diff"。每次运行都有唯一前缀，因此引擎日志显示每次运行都复用了 0 个 token。
* **输出：** 256 token 上限，流式 API，贪婪。
* **重复次数：** 每种尺寸 3 次；表格显示中位数（范围）。没有单独的预热；1k 运行最先进行。专家缓存在运行期间自适应，其状态在运行之间延续。模型加载不计时。
* **计时：** TTFT