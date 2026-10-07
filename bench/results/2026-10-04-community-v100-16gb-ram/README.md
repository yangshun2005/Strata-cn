# 在 Tesla V100-PCIE-32GB 与 16 GB 系统内存上的社区基准测试

由 christopherrobertbrooks-tech 于 2026-10-04 测量。Coder IQ1_M 运行在一张 Volta 卡上（实验性 CUDA 12 引擎），所在 PC 仅有 16 GB 内存，低于 Coder 通常要求的 32 GB。主要局限：单台机器、单一模型规模；V100 位于 PCIe 3.0 x4 链路上。

## 硬件与软件

- GPU：Tesla V100-PCIE-32GB（sm_70），PCIe 3.0 **x4**（16 通道插槽中装有一张 RTX 4070 12 GB，负责驱动显示器，在测试期间处于空闲状态）；功耗限制 250 W（默认）。CPU：Intel Core i7-13700KF（24 线程，AVX2，无集成 GPU）。内存：**16 GB DDR5**（可见 15 GiB）。模型文件位于 SATA SSD（Samsung 870 EVO 500 GB）。
- 操作系统：Ubuntu 24.04.4 LTS，内核 7.0.0-30-generic；NVIDIA 驱动 580.173.02；CUDA 12.9 工具包。
- Strata 主分支位于 6f32ec0，引擎 0.1.39。引擎由 setup.sh 从源码构建（CUDA 12，`-DSTRATA_EXPERIMENTAL_SM60=ON`，sm_70；构建耗时约 45 秒）。在 `serve/frontend.py` 中有一处本地修改（Anthropic tool_result 块内的图像，PR #819）；它不涉及引擎或这些 OpenAI-API 请求。
- 透明大页：`enabled = madvise`，`defrag = madvise`。
- 后台：服务器运行在 llama-swap 之后（请求通过同一台机器上的其代理转发）；没有其他 GPU 工作。

## 模型与配置

- Qwen3.8-Flash-Next GSQ-RCO **Coder IQ1_M**（setup 的 `--family coder`）：
  `Qwen3.8-Flash-Next-GSQ-RCO-IQ1_M-00001-of-00002.gguf`、`-00002-of-00002.gguf`。视觉编码器开启
  （`mmproj-Qwen3.8-Flash-Next-BF16.gguf`，在 GPU 上）。专家配置 `data/expert-profile-coder.bin`（随附）。
- 上下文 131,072；KV int8 位于 VRAM（setup 关闭了 KV 流式传输：“它需要在 IQ1_M 使用的约 32 GB 之外再占用约 1.8 GB 内存，
  而这台 PC 只有 15”）；专家缓存自动；预填充自动；`--resident-experts`（setup 的低内存选择，`--low-ram on`）。
- 启动时：“常驻内存模式：4.61 GiB 的专家在内存中（页锁定），**11650 在 GPU 缓存中**”；“全部加载后剩余 672 MiB 的 VRAM
  空闲”。在没有视觉编码器的情况下，全部 **12,288** 个专家都在 GPU 缓存中（内存中占用 3.40 GiB）。
- MTP `--spec 4 --spec-min-p 0.5`；这些测试中推理 `none`；温度 0；无校准，无速度预测。
  配置还设置了 `"reasoning_budget_tokens": 8192`（在推理关闭时此处无关）。

```text
./setup.sh --yes --family coder --gpu 1 --cuda 12 --low-ram on --vision gpu --data-dir /mnt/steam/Strata-data --no-start
engine args: --pack .../packs/coder-iq1_m --native ...-00001-of-00002.gguf --ple-gguf ...-00002-of-00002.gguf
  --expert-profile data/expert-profile-coder.bin --expert-cache auto --prefill auto --spec 4 --spec-min-p 0.5
  --mtp .../mtp/rt --max-context 131072 --kv int8 --resident-experts --vision --vram-reserve-mib 700
```

## 方法

来自 `bench/results/2026-10-03-community-2x-mi50/` 的 `benchmark.py`（相同模型），未做改动：使用全新提示的串行请求
（0 个复用 token），一次预热，在 4,096、32,768 和 128,000 个提示 token 下进行三次测量运行，输出上限为
256 个 token，`reasoning_effort: none`，温度 0。提示和解码速率是引擎自身的计时；TTFT 和总
时间在客户端测量，通过 llama-swap，模型已加载。每次运行的 JSON 在此文件夹中。

## 结果

| 配置 | 实际提示 token | 复用 token | 生成 token | 运行次数 | 提示 tok/s 中位数与范围 | 解码 tok/s 中位数与范围 | TTFT 秒中位数与范围 |
| --- | ---: | ---: | ---: | ---: | --- | --- | --- |
| Coder IQ1_M，仅 V100 | 4,096 | 0 | 256 | 3 | 1,233 (1,232–1,234) | 69.3 (67.2–77.1) | 3.35 (3.35–3.36) |
| Coder IQ1_M，仅 V100 | 32,768 | 0 | 256 | 3 | 1,580 (1,580–1,581) | 68.6 (64.6–69.0) | 20.80 (20.80–20.81) |
| Coder IQ1_M，仅 V100 | 128,000 | 0 | 256 | 3 | 1,394 (1,393–1,395) | 67.1 (65.5–71.6) | 92.01 (91.93–92.06) |

- 总请求时间（客户端）：7.0 秒、24.5 秒和 95.8 秒（中位数）。
- GPU 专家缓存命中率 0.997–0.999；每次运行**从模型文件读取 0.0 MB**；PCIe 占比 0.0（引擎日志）。
- MTP 草稿接受率 0.63–0.78。
- 内存：服务期间 V100 使用 31.5–32.2 GB；系统内存在 15 GiB 中使用约 8–9 GB。无分页，无内存不足故障。
- 通过 llama-swap 启动（加载到首次回答）：约 80 秒，模型文件位于 SATA SSD。
- PCIe，在代理工作期间使用 `nvidia-smi dmon -s t` 单独测量：进入显卡 120–150 MB/s，输出 3–19 MB/s——
  约为 x4 链路的 4%，因此在模型常驻的情况下，窄插槽并未限制推理。

## 正确性与局限

- 长上下文召回，`tools/needle_bench.py --lengths 32k,128k --depths 10,50,90`：**6/6 找到**（32,171-32,172 和
  125,449-125,451 个提示 token；每次检查 11-22 秒和 81-90 秒）。结果在 `needles.json` 中。
- HumanEval（164，贪心，思考关闭，我们自己的测试框架，不属于此仓库）：**158/164**，7.1 分钟。
- 作为本地编码代理的构建者（Claude Agent SDK -> `/v1/messages`，中等努力），在现有
  Python/GTK 项目中的 4 个真实任务，3 轮：12/12 完成，所有验收测试和隐藏布局检查均通过。
- 供参考，README 的 Coder 行（其他引擎版本和设置，因此仅大致可比）：RTX 5070
  12 GB 配 64 GB 内存，解码 55 / 43 tok/s（短 / 128K），提示 2,180 tok/s；此 V100 将整个 Coder 保持在
  VRAM 中并解码 69 / 67 tok/s，提示读取较慢（32K 时为 1,580）。
- 未测试：其他规模、与第二张卡的层拆分、速度测试中开启思考、超过三次的运行。
- 在代理使用中观察到一次（不在此基准测试中）：单个思考块运行到 32K 输出上限。一个
  `reasoning_budget_tokens` 上限可以约束这一点：在一个被构建为长时间思考的提示上，无上限则没有回答
  （`finish_reason: length`），400 token 上限给出了服务器的总结行和完整回答。在后续代理运行中
  使用 8K 上限时，该上限从未触发（最长思考约 4K token）。参见 #710/#728。

本报告由 AI 助手（Claude）根据我们的测量起草，并由我核对。