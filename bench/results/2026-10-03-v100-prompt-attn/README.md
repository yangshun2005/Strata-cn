# V100 (sm_70)：张量核心 QSA 提示注意力（2026-10-03）

在 sm_70 上，提示路径在解码内核上运行 QSA 注意力，每次处理一个查询，使用 FP32 FMA。`prompt_attn_v70_kernel`
（`src/kernels/cuda/qsa_prompt_attn.cu`）在 Volta 的张量核心（`mma.sync.m8n8k4`）上执行两个矩阵乘积。
`STRATA_PROMPT_ATTN_OLD=1` 是旧路径，因此下面的每个数字都是同一个二进制文件在有无该开关时的结果。

## 测试平台

- Tesla V100-PCIE-32GB（sm_70，PCIe gen3 x16），驱动 550.120，使用一张卡；PC 中还有第二张 V100 和一张 Quadro RTX 4000（sm_75），
  通过 `CUDA_VISIBLE_DEVICES` 隐藏。Intel Core i9-7960X（AVX-512），125 GB RAM，Linux 6.8，CUDA 12.8，GCC 13。
- 引擎：`origin/main` 位于 `99f3dbd`（0.1.35 + 83 个提交）加上此更改，使用
  `-DCMAKE_CUDA_ARCHITECTURES=70 -DSTRATA_EXPERIMENTAL_SM60=ON` 构建。
- 模型：Unsloth `Qwen3.8-Flash-Next-UD-IQ4_XS`（3 个分片），使用 `tools/iq_pack.py --compat-bf16` 打包，MTP 头来自
  `tools/mtp_fetch.py`。不是 setup 的模型之一（见限制）。`--max-context 204800`，`--kv int8`，`--spec 2`。

## 提示速度

每个配置启动一次服务器（冷启动），一个 2K 预热请求，然后三个随机单词提示，每个都以贪婪方式回答
（温度 0，48 个 token）。数字是服务器自己的输出行 `strata serve: prompt N tokens = 0 reused + N read in T ms`。每个
单元格运行一次；同一配置的重复启动给出了相同的秒数。

| 提示 token 数 | 旧路径 | 新内核 | 增益 | 读取时间，旧 -> 新 |
| ---: | ---: | ---: | ---: | ---: |
| 7,194 | 1,010 tok/s | 1,164 tok/s | +15% | 7.1 s -> 6.2 s |
| 28,650 | 1,063 tok/s | 1,251 tok/s | +18% | 26.9 s -> 22.9 s |
| 114,338 | 973 tok/s | 1,123 tok/s | +15% | 117.5 s -> 101.8 s |

解码在噪声范围内保持不变（38-51 tok/s，每个请求 48 个 token）：它不使用此内核。

## 时间去向（nsys，一张 V100，28,650 token 提示，整个捕获的 GPU 内核时间：预填充加上 128-193 个解码 token）

`strata-1gpu-32k-*.nsys-rep`；“之前”的捕获是 0.1.35（`d9ab843`），“之后”的是同一基础加上此更改以及 #540 / 已关闭的 #593 的 BF16 更改
（通过 FP16 的 BF16 GEMM；那个只移除了 `magma_sgemmEx`，不在此表中）。

| | 之前 | 之后 |
| --- | ---: | ---: |
| 解码内核上的 QSA 注意力（`attn_chunk_kernel<1>`） | 5.78 s (19.2%) | 0.18 s（仅解码步骤） |
| `prompt_attn_v70_kernel` | - | 2.16 s (9.0%；提示部分的 12%) |

## 精度

`qsa_prompt_attn_parity`（合成，无模型；使用 `-DSTRATA_PARITY_PROMPT_ATTN=ON` 构建），V100，32K 上下文，每块 2,048 个查询，输出缩放 3.6：

| KV | 与 FP64 的误差，旧内核 | 与 FP64 的误差，新内核 | 每块时间，旧 -> 新 |
| --- | ---: | ---: | ---: |
| int8 | 2.4e-6 | 5.1e-6 | 38.1 -> 13.4 ms (2.8x) |
| fp16 | 1.9e-6 | 5.3e-6 | 37.3 -> 28.3 ms (1.3x) |

1,500 和 2,100 token 的上下文也通过（int8：3.2x 和 2.2x）。该测试的 Q4_0 用例在 sm_80 以下被跳过，此时调度器保留
旧内核（上游的模式 4 仅限 sm_80+）。

## 输出检查

- **大海捞针召回**（`tools/needle_bench.py --lengths 8k,32k,128k --depths 10,30,50,70,90`，贪婪，关闭思考，每个
  配置一个全新服务器；提示为 7.9K、32.0K 和 122.7K token）：**旧路径找到 15/15，新内核找到 15/15。** 较后深度的读取时间
  更短，因为服务器复用了针之前文本的检查点。
- **贪婪输出**（48 个 token，上述三个提示）：28,650 和 114,338 token 的提示结果相同。对于 7,194 token 的提示，
  答案开头相同，**之后不同**：内核以另一种顺序求和，贪婪解码可能在别处跟随一个接近平局的结果。使用代码词的召回测试
  无法区分输出分布的微小偏移，且未运行教师强制的 top-1 比较。

## 分布检查（教师强制，对照旧路径和对照组）

`docs/UNSLOTH_Q4.md` 的方法：在 serve 引擎上使用 `STRATA_LOGPOS=<file> STRATA_LOGPOS_TOPK=256`（`--short-read 640 --adapt-every 100000`，贪婪，
`--max-context 204800`）。每个提示是一个聊天：一个长的第一条消息（本仓库文档和代码的 8,543 / 32,906 / 99,040 token，由
批处理提示路径读取，即由被测内核读取），一个短回复，以及一条约 600 token 的最后消息，内容为其他仓库文本，经过验证
窗口，其中每个 token 都被评分：每个提示 617 个位置。KL 是旧 || 其他，基于旧运行的 top 256 加上其余部分的桶。

| 比较 | KL 均值（8.5K / 33K / 99K） | argmax 相同 | top-10 重叠 |
| --- | --- | --- | --- |
| 旧路径 vs 旧路径（第二次运行） | 0 / 0 / 0 | 100% | 100% |
| 新内核 vs 新内核（第二次运行） | 0 / 0 / 0 | 100% | 100% |
| **旧路径 vs 新内核** | 0.028 / 0.040 / 0.116 | 97.7 / 93.2 / 86.4% | 93.3 / 91.9 / 88.4% |
| 对照 A：旧路径，BF16 GEMM 无 FP16 绕行（`STRATA_BF16_VIA_F16=0`） | 0.026 / 0.051 / 0.090 | 97.2 / 91.4 / 91.2% | 92.9 / 91.4 / 88.5% |
| 对照 B：旧路径，`--prefill 2048` 而非自动（8192） | 0.023 / 0.035 / 0.038 | 97.9 / 94.8 / 95.6% | 92.9 / 92.2 / 89.7% |

旧 vs 新的 KL 中位数：0.0007 / 0.0088 / 0.022；p99：0.35 / 0.39 / 1.12（重尾）。真实 token 的困惑度在每个
更改下都变化很大（旧 365 / 127 / 24；新 346 / 192 / 12；对照 A 395 / 161 / 18），因此它在这里不能说明质量。

解读：模型放大了微小的数值差异，而两个对照组除了舍入顺序外没有改变任何东西，其分布移动程度与内核一样大。
在 99K 时，内核略高于两个对照组；这是一份文档和 617 个位置，因此那里的真实效应既未被确认也未被排除。

**模型自身数据上的内核。** 一项临时检查（不在此 PR 中）在同一输入上并排运行解码内核和新内核，针对每第 8 批
32 个查询，在所有 12 个 QSA 层中，在上述三个提示上（位置高达 98K，约 500 万 (query, head) 行，每行 256）：每行的 ||new - old|| / ||old||
平均为 4.4e-7，最差行为 4.8e-6；最大绝对差为 4e-5，对应值高达 31。各层或上下文长度上没有趋势。
这是 FP32 舍入水平，因此上述差异来自模型的敏感性，而非内核的算术。

## 限制

- 仅限 sm_70（`__CUDA_ARCH__` 700-749）。Q4_0 KV（`--kv q4_0`）和 QSA 块分数（`block_scores_kernel`，sm_80 以下的 warp 内核）在 Volta 上仍使用其
  旧内核。
- 一个请求，一张卡。Unsloth IQ4_XS 包不是 setup 模型；GSQ-RCO 量化未在此平台上运行。
- 上述分析是一个提示的一次捕获。分布检查是每个长度一份文档；99K 行是最不确定的。

## 复现

```sh
cmake -S . -B build -DSTRATA_ENABLE_CUDA=ON -DSTRATA_BUILD_TESTS=OFF -DSTRATA_PARITY_PROMPT_ATTN=ON \
      -DCMAKE_CUDA_ARCHITECTURES=70 -DSTRATA_EXPERIMENTAL_SM60=ON
cmake --build build --target strata qsa_prompt_attn_parity -j
CUDA_VISIBLE_DEVICES=0 ./build/qsa_prompt_attn_parity 32768 2048 5       # 精度表
STRATA_PROMPT_ATTN_OLD=1 <start script>                                   # 旧路径；不带该变量：新内核
python tools/needle_bench.py --url http://127.0.0.1:8081 --lengths 8k,32k,128k --depths 10,30,50,70,90
```