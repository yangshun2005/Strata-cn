# RX 7900 XTX 社区基准测试（Windows 11）

由 nytasuk-commits 于 2026-10-06 测量。Strata 引擎 0.1.40，原始 Flash-Next IQ3_S，262,144 token 上下文，在 Windows 上使用现成的 AMD 引擎（非源码构建）。测试内容：在约 100 和约 38K 提示 token 下各进行三次全新运行，外加在 32K 和 128K 下的六次大海捞针召回检查。主要局限：单台机器、每种配置三次运行、非流式请求（无首 token 时间），且未测量 VRAM、PCIe 链路宽度和 GPU 功耗。

## 硬件与软件

- GPU 与 VRAM：AMD Radeon RX 7900 XTX，24 GB（gfx1100）。还存在一个 AMD 集成 GPU（gfx1036）；安装程序将其列为不受支持，并使用了 7900 XTX（GPU 0）。
- CPU：AMD Ryzen 9 7950X3D 16 核（安装程序检测到 AVX-512）
- 已安装内存：192 GB，5200 MT/s（Strata 状态报告总计 191.2 GiB）
- 存储：两块三星 990 PRO 4TB NVMe SSD。Windows 位于其中一块；模型文件（`Strata-data`）位于另一块。
- PCIe 链路：未记录链路速度和宽度。Strata 自身的启动探测测得主机到设备 28.3 GB/s（本机较早一次启动的引擎日志）。
- 操作系统：Windows 11 Pro，版本 10.0.26200（内部版本 26200）
- 驱动：AMD 显示驱动 32.0.31041.1004，日期为 17/08/2026（如任务管理器所示）
- ROCm / HIP：运行时随 Strata 引擎捆绑（`amdhip64_7.dll`）；未单独安装 ROCm
- Strata 提交：`1735d6471df29b42c26170efaac1f1446a58640f`（提交于 2026 年 10 月 6 日周二 03:12 +0200）；引擎版本 0.1.40（来自 `/v1/status`）；由 `START-HERE.bat` 安装的现成 AMD 引擎，非源码构建
- 后台工作负载：运行期间我没有启动任何工作负载。LM Studio 已卸载；没有 Strata 客户端连接（`/v1/status` 在第一个请求前显示 0 个请求）。GPU 功耗限制：未测量。

## 模型与配置

- 模型：Qwen3.8-Flash-Next，原始版本，由 ISTA-DASLab 进行 GSQ-RCO 量化，大小 IQ3_S
- GGUF 文件：`Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf`（54,817,524,224 字节）和 `...-00002-of-00002.gguf`（28,800,138,432 字节）。两个文件在运行前均完整读取且无错误。仓库修订版本：未记录。
- 视觉编码器：关闭。自定义包或配置文件：无。该包由安装程序准备（`packs\iq3_s`），并使用随附的 `data/expert-profile.bin`。
- 上下文：262,144。KV：int8，使用 `--kv-resident 32768`（KV 流式传输；安装程序报告在 128K 时 1.8 GB 的 KV 位于 RAM 中）。预填充：`auto:32768`。专家缓存：`auto`，启动后缓存了 7,942 个专家（15.06 GiB 的 VRAM）。低内存模式：未使用。对话缓存：8192 MiB。
- MTP 草稿层：开启（`--spec 4 --spec-min-p 0.5`）。推理：服务器默认。采样：服务器默认（未发送采样参数）。校准：未运行（根据 INSTALL.md 仅限 NVIDIA）。实验性速度预测：关闭。
- hipBLASLt 调优表：`gfx1100-hipblaslt-100500.txt`，通过配置中的 `STRATA_HIPBLASLT_TUNING` 设置。

```text
D:\AI\Strata\run-iq3_s.bat
```

运行配置 `strata-iq3_s.json`（也已附上；其中无凭据）：

```json
{
 "exe": "D:\\AI\\Strata\\engine\\strata.exe",
 "args": [
  "--pack", "D:\\AI\\Strata-data\\packs\\iq3_s",
  "--native", "D:\\AI\\Strata-data\\models\\IQ3_S\\Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00001-of-00002.gguf",
  "--ple-gguf", "D:\\AI\\Strata-data\\models\\IQ3_S\\Qwen3.8-Flash-Next-GSQ-RCO-IQ3_S-00002-of-00002.gguf",
  "--expert-profile", "D:\\AI\\Strata\\data\\expert-profile.bin",
  "--expert-cache", "auto",
  "--prefill", "auto:32768",
  "--spec", "4",
  "--spec-min-p", "0.5",
  "--mtp", "D:\\AI\\Strata-data\\mtp\\rt",
  "--max-context", "262144",
  "--kv", "int8",
  "--kv-resident", "32768",
  "--conversation-cache-mib", "8192"
 ],
 "cwd": "D:\\AI\\Strata",
 "tokenizer": "D:\\AI\\Strata-data\\packs\\iq3_s\\tokenizer",
 "model_name": "qwen3.8-flash-next-iq3_s",
 "log": "D:\\AI\\Strata\\strata-iq3_s.log",
 "lib_dirs": ["D:\\AI\\Strata\\engine\\rocm\\bin"],
 "port": 8080,
 "backend": "hip",
 "env": {
  "STRATA_HIPBLASLT_TUNING": "D:\\AI\\Strata\\tools\\hip\\gfx1100-hipblaslt-100500.txt"
 },
 "gpu": 0,
 "gpus_asked": true
}
```

`--prefill auto:32768` 和 `--conversation-cache-mib 8192` 是在安装后按照安装程序打印的两条提示手动添加到配置中的。之后运行 `START-HERE.bat --setup` 会将两者重置。

## 方法

- 请求：非流式 `POST /v1/chat/completions`，`max_tokens` 为 256。每个提示以全新的 GUID（`Run id: ...`）开头，因此没有较早的请求共享其前缀。
- 计时：在每个请求后直接从 `/v1/status` 的 `last_timings` 读取（`prompt_n`、`cache_n`、`prompt_ms`、`predicted_n`、`predicted_per_second`、草稿计数器）。总时间为 HTTP 调用周围的客户端挂钟时间，因此包含请求开销。
- 顺序：重启 Strata，然后进行一次短预热请求和一次长预热请求（均不计入），接着进行 3 次短运行和 3 次长运行。
- 短提示："Write a short story about a lighthouse keeper."（包括运行 id 和模板约 100 token）。
- 长提示：一份包含 1,100 条编号条目的合成日志（以 `System.Random 42` 为种子），在第 550 位有一个口令条目，末尾有一个两部分问题（约 38,170 token）。完整脚本见附录。
- 复用与缓存状态：每次计入的运行中 `cache_n` 均为 0，因此所有提示 token 都是全新读取的。专家缓存在启动时填充，并已服务了两次预热请求。
- 模型加载不计入。内存：使用的系统 RAM，每次运行后从 `/v1/status` 读取（69.0 至 70.2 GiB）。运行期间的 VRAM：未测量（Strata Monitor 不显示 AMD 的 GPU 统计信息）。
- 运行期间没有向 Strata 发送其他内容。

## 结果

| 配置 | 实际提示 token | 复用 token | 生成 token | 运行次数 | 提示 tok/s 中位数与范围 | 解码 tok/s 中位数与范围 | TTFT 秒 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| 短 | 98 至 103 | 0 | 256（上限） | 3 | 84.5（77.7 至 85.9） | 77.5（73.8 至 78.7） | 未测量 |
| 长，38K | 38,170 至 38,174 | 0 | 131 至 138 | 3 | 1,612.0（1,608.7 至 1,619.0） | 79.7（66.7 至 83.2） | 未测量 |

短提示的"提示 tok/s"在约 100 token 的提示上主要由固定的每请求开销主导，并非预填充速度。长提示的数字才是。

总挂钟时间：短中位数 4.5 秒（4.4 至 4.8）；长中位数 25.4 秒（25.4 至 25.7）。

每次运行数据（也在 `runs.csv` 中）：

| 配置 | prompt_n | cache_n | prompt_ms | 提示 tok/s | 生成 | 解码 tok/s | 草稿接受率 % | 总计秒 | 使用 RAM GiB |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| short | 103 | 0 | 1325.0 | 77.7 | 256 | 73.8 | 64.1 | 4.8 | 69.0 |
| short | 98 | 0 | 1160.1 | 84.5 | 256 | 78.7 | 65.9 | 4.4 | 69.2 |
| short | 99 | 0 | 1152.8 | 85.9 | 256 | 77.5 | 63.3 | 4.5 | 69.4 |
| long-38K | 38,174 | 0 | 23,579.2 | 1,619.0 | 138 | 66.7 | 74.1 | 25.7 | 69.5 |
| long-38K | 38,170 | 0 | 23,727.9 | 1,608.7 | 131 | 79.7 | 80.0 | 25.4 | 69.5 |
| long-38K | 38,173 | 0 | 23,680.8 | 1,612.0 | 138 | 83.2 | 78.5 | 25.4 | 70.2 |

长运行生成了 131 至 138 个 token，低于 256 上限。停止原因未记录。来自较短样本的解码数字比短提示的波动更大。

### 附加观察（单次运行，不属于上述基准测试）

这些来自 Strata Monitor 标签页和聊天，使用不同的提示，因此仅供参考。

- **共享 GPU 的模型会缩小专家缓存。** 当 LM Studio 中同时加载了一个 17.74 GB 的模型时，聊天中相同的编码提示给出了 1,771 个缓存专家（3.4 GiB）、49.0 tok/s 和 52.1% 的 VRAM 命中率（564 个输出 token）。卸载并重启后：8,243 个专家（15.6 GB）、70.5 tok/s 和 86.1%（1,053 个输出 token）。在聊天中使用 38K 日志提示：之前为 44.0 tok/s 和 53.8%，之后为 66.2 tok/s 和 80.3%。
- **对话缓存。** 在 Claude Code 中，针对约 40K token 前缀启动新会话，不使用 `--conversation-cache-mib 8192` 时耗时 28.1 秒（0 复用），使用后耗时 1.0 秒（40,413 中复用 40,408）。
- **预填充设置。** `--prefill auto:32768` 与 `auto` 相比，各运行一次分别给出约 1,630 与 1,540 提示 tok/s，因此在噪声范围内。上述长提示基准测试使用 `auto:32768`。
- **128K 与 256K 上下文。** 专家缓存在 128K 时为 8,159 个专家，在 256K 时为 7,942 个。

## 正确性与局限

`tools/needle_bench.py` 针对运行中的服务器（也在 `needles.json` 中）：

```text
.\.venv\Scripts\python.exe tools\needle_bench.py --url http://127.0.0.1:8080 --lengths 32k,128k --depths 10,50,90 --out needles.json
```

| 长度 | 深度 | 结果 | 提示 token | 时间 |
| --- | ---: | --- | ---: | ---: |
| 32K | 10% | 找到 | 32,308 | 19 秒 |
| 32K | 50% | 找到 | 32,308 | 18 秒 |
| 32K | 90% | 找到 | 32,309 | 19 秒 |
| 128K | 10% | 找到 | 125,705 | 78 秒 |
| 128K | 50% | 找到 | 125,703 | 78 秒 |
| 128K | 90% | 找到 | 125,703 | 61 秒 |

6 次全部找到。该脚本不打印复用 token 计数。大海捞针测试衡量的是这些输入上的召回率，而非整体模型质量。

38K 日志提示在聊天中也返回了正确的口令，并且模型通过 Claude Code 完成了智能体循环（读取文件、列出目录、运行 shell 命令），后续工具调用的前缀复用率超过 99%。

局限与未测试项：

- 单台机器，每种配置三次运行，仅中位数和范围。
- 无流式测量，因此无首 token 时间。总延迟为客户端侧。
- 未记录 VRAM 使用、GPU 负载、温度和功耗（Monitor 在 AMD 上不显示 GPU 统计信息）。未记录 PCIe 链路宽度和 GPU 功耗限制。
- 未对答案质量和智能体可靠性进行基准测试。未测试图像（关闭；AMD Windows 仅在 CPU 上读取图片）。
- 未测试其他大小（Q2_0、IQ2_XS、IQ3_XXS）、Coder、Swift 和 Unsloth 变体。
- Windows HIP 在 `AMD_HIP.md` 中被描述为该文档验证范围之外。本报告显示 `START-HERE.bat` 在 Windows 11 上使用 RX 7900 XTX 设置并运行了 AMD 引擎，这是关于该差距的一个数据点，而非对其的验证。

## 附录：基准测试脚本

PowerShell，在 Strata 运行时于第二个窗口运行。它写入 `runs.csv`。

```powershell
$u = "http://127.0.0.1:8080"
$null = New-Item -ItemType Directory -Force D:\AI\bench
$sb = New-Object System.Text.StringBuilder
$rnd = New-Object System.Random 42
1..1100 | ForEach-Object {
  if ($_ -eq 550) { [void]$sb.AppendLine("Entry 550: NOTE the deployment passphrase is copper-lantern-7731.") }
  else { [void]$sb.AppendLine("Entry ${_}: service-$($rnd.Next(1,50)) handled request $($rnd.Next(100000,999999)) in $($rnd.Next(5,900)) ms with status $(@('OK','OK','OK','RETRY','SLOW')[$rnd.Next(0,5)]) on node-$($rnd.Next(1,20)).") }
}
$long = $sb.ToString() + "`nQuestion: What is the deployment passphrase mentioned in the log above, and in one sentence what does the log mostly show?"
$short = "Write a short story about a lighthouse keeper."
function Run($label, $text) {
  $content = "Run id: $([guid]::NewGuid())`n" + $text
  $body = @{ model="x"; max_tokens=256; messages=@(@{role="user"; content=$content}) } | ConvertTo-Json -Depth 5
  $sw = [Diagnostics.Stopwatch]::StartNew()
  $null = Invoke-RestMethod "$u/v1/chat/completions" -Method Post -ContentType "application/json; charset=utf-8" -Body ([Text.Encoding]::UTF8.GetBytes($body))
  $sw.Stop()
  $s = Invoke-RestMethod "$u/v1/status"
  $t = $s.last_timings
  [pscustomobject]@{ config=$label; prompt_n=$t.prompt_n; cache_n=$t.cache_n; prompt_ms=$t.prompt_ms; prompt_tps=$t.prompt_per_second; out_n=$t.predicted_n; decode_tps=$t.predicted_per_second; draft_acc_pct=[math]::Round(100*$t.draft_n_accepted/[math]::Max(1,$t.draft_n),1); total_s=[math]::Round($sw.Elapsed.TotalSeconds,1); ram_gib=$s.machine.ram.used_gib }
}
$null = Run "warmup-long" $long
$rows = @()
1..3 | ForEach-Object { $rows += Run "short" $short }
```