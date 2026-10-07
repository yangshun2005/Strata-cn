# 2026-10-01：64 GB PC 上的 IQ4_XS —— 固定 arena vs `experts.bin` mmap vs RAM 预算

**问题。** 一台 64 GB 内存的机器上有 60.94 GiB 的路由专家。引擎服务它们的三种方式中，哪一种在这里实际上最快：固定 arena（默认）、对包内 `experts.bin` 使用 `--mmap-experts`，还是 0.1.31 的路径——就地读取 GGUF 并为最热的专家固定 `--resident-budget-gib`？

**机器。** Ryzen 7 5700X3D（Zen 3，AVX2，无 AVX-512），64 GB DDR4-2666 双通道，RTX 5060 Ti 16 GB 走 PCIe 4.0 x8（引擎自身的探测：主机→设备 14.1 GB/s），NVMe 顺序读取实测 2.41–2.46 GB/s，Windows 11。

**模型。** `orcarouter/Qwen3.8-Flash-Next-Uncensored-GGUF`，IQ4_XS：60.94 GiB 路由专家，48 层 × 512 专家，每个专家 blob 2.66 MB，gate/up IQ4_XS + down IQ4_NL。引擎 0.1.31 和 0.1.32 从源码构建（带 AVX-2 IQ4_XS gate 内核）。

**方法。** 离线 `strata generate --tokens-file <frozen prompt> --max-new 300 --stats`，各臂交错运行，**每轮将臂顺序轮换一位**，一次不计入测量的预热运行。主要指标是**在相同 draft 轮数下每 draft 轮的毫秒数**。仅凭 tok/s 无法跨运行比较：GPU 命中路径与 CPU 的取整方式不同（引擎在启动时说明了这一点），因此 draft 接受率会逐次运行变化——同一臂、同一参数在五次运行中产生了 112–115 轮，在一次中产生了 133 轮，仅此一项就会让 tok/s 变动约 10%。

## 三条路径

| 路径 | 冷 | 热 | 每轮读取的专家字节数 |
|---|---:|---:|---:|
| 固定 arena（无 `--mmap-experts`） | 27.90 | 33.85 | —（每次运行都以 1.88–2.19 GiB/s 加载 60.94 GiB） |
| 对 `experts.bin` 使用 `--mmap-experts`，setup 的参数 | 14.06 | 16.88 | ~1.2 GB |
| 同上 + `--kv-resident 20480` + `STRATA_IQ_PREFETCH=16384` | 33.3 | **37.5–39.0** | ~1.2 GB，行读取 28 GB/s |
| 就地读取 GGUF（不含 `experts.bin` 的包） | 9.63 | 9.74 | **1325 MB** |
| 就地 GGUF + `--resident-budget-gib 40` | 26.7 | 26.0 | 42 MB |
| 就地 GGUF + 预算 40 + 上述两个旋钮 | 26.7 | 27.5 | 34 MB |
| `experts.bin` + `--resident-budget-gib 40` | 24.7 | 25.4 | 0（全部来自 RAM 副本） |
| 参考：同一模型的 IQ3_S，arena | 50.6 | 51.5 | ~0.94 GB |

因此在这台机器上顺序是 **对 `experts.bin` 的 mmap > 固定 arena > 常驻预算 > 就地 GGUF**，而运行同一文件的最佳与最差方式之间的差距是 **4 倍**。

## 每个数字的来源

**1. arena 并不固定，而在这个规模下这翻转了排名。** `generate.cpp` 中的注释在**一台 63 GB 机器上的 34 GB `experts.bin`** 上测得 arena *比热 mmap 好 1.79 倍，比冷 mmap 好 3.7 倍*，并且它已经说明那里的 arena 并未固定（对 31.64 GiB 执行 `cudaHostRegister` 失败并报 "out of memory"——"你无法固定 63 GB 中的 34 GB"）。这里在更大的规模下出现了同样的失败：

```
expert arena: SetProcessWorkingSetSizeEx(14814 MiB) failed (error 1450); cudaHostRegister of the
whole arena FAILED (out of memory); 37 slices pinne[d]
```

（该行被测试框架截断在 165 列，因此尾部未被记录）。大页也被拒绝，错误 1450，对 61 GiB 和 50.3 GB 的 arena 都是如此。在 60.94 GiB 时改变的是后果：arena 仍然能运行——33.85 tok/s——但它在**每次**启动时都要花约 30 秒以约 2 GiB/s 加载 60.94 GiB，而且现在它**输给**了热映射运行（37.5–39.0）。`cudaHostRegister` 对 IQ3_S 的 arena **确实成功**（对 50,295,996,416 B 执行 `cudaHostRegister PORTABLE ok`），这就是为什么 arena 在低一个量化级别时是正确答案，而在高一个级别时是错误答案。

**2. RAM 预算相对纯 mmap 损失约 30%，而且这两项成本在日志中可见。** `eb_sp`（对 `experts.bin` 的 mmap）38.06 tok/s vs `eb_rb40sp` 27.45：

```
adaptive tier   2688 experts swapped into the VRAM tier (every 4 rounds, 7.580 ms/round)
```

映射模式没有的每轮 7.6 ms PCIe 流量，以及：

| | 行读取 GB/s |
|---|---:|
| mmap，页缓存 | 27.4 |
| 常驻预算，页锁定补集 | **15.5** |

页锁定补集对设备有利，但对读取它的 CPU 池更慢。

**3. 页锁定 RAM 会驱逐其他所有人的页缓存。** 同一臂、同一参数、两轮，在轮次中的位置不同：`eb_sp` 在第 1 轮为 38.06 tok/s / 27.4 GB/s，在第 2 轮为 18.59 / **10.6**——因为其间运行了两次 40 GiB 的页锁定分配，且它们不可驱逐。任何在同一轮中混合映射臂和预算臂的比较，测量的是顺序，而不是模式。

**4. 预取距离不是杠杆；`--kv-resident` 才是。** 本工作早期的一轮报告在映射路径上 2048 B 比 8–24 KB 落后约 9%。**那个比较被混淆了**：2048 B 臂还使用了默认的 `--kv-resident 32768`，而其他臂使用 20480，而 kv-resident 在这个模型上价值约 15%（VRAM 中更少的 KV 槽意味着更少的专家槽，意味着 CPU 池读取更多专家）。在 kv-resident 保持不变的情况下重跑，三个模型，三轮交错且顺序轮换，以 300 token 的总时间比较：

| 模型（专家路径） | 预取关闭 | 2048（默认） | 8192 |
|---|---:|---:|---:|
| IQ3_S（arena） | 5885 ms | 5980 | 5859 |
| IQ3_XXS（arena） | 5472 | **5449** | 5503 |
| IQ4_XS（映射） | 7617 | **7525** | 7748 |

差距 0–3% 且无顺序性：完全关闭行预取没有任何代价，而在映射路径上当前默认是三者中最好的。真正改变数字的是行读取速率：慢的运行是那些处于 21–22 GB/s 而非 25–29 的，即页缓存被部分驱逐，而这每轮落在不同的臂上。涉及的行大小为 656 B（IQ2_S）、784 B（IQ3_XXS）和 1088 B（IQ4_XS），所以 2048 B 是 2–3 行——但在这些读取速率下，硬件预取器已经覆盖了遍历，在它前面加一个显式提示毫无收益。

关于这一点有一个限制：三个模型都是**热**测的（行读取 25–29 GB/s，即由页缓存服务）。一次以 2.4 GB/s 从 NVMe 每轮读取 1.3 GB 的运行是另一种机器状态，在那里缺页前的提示可能有收益；未测试。

**5. 前瞻路由预取：在这个规模下无可测量效果。** `--resident-budget-gib 40` 带与不带 `STRATA_LOOKAHEAD=0`：26.74/25.94 vs 26.25/26.00 tok/s。每轮 42 MB 的文件读取，几乎没有什么可预热的了。

**6. 复现了两次拒绝。** 在映射模式上使用 `--resident-cpu-experts`：*"resident complement 53.40 GiB exceeds available RAM minus the 8 GiB safety headroom"*——这正是 `--resident-budget-gib` 存在的意义。而预算 50 从另一侧触发了 #403：

```
FileExpertSource: resident complement 46.70 GiB exceeds available RAM (50.70 GiB) minus the 4 GiB safety headroom
```

而此前同一预算的一次运行已经成功。40 是这里 64 GB 上的可用上限。

**7. `--expert-cache-per-layer` 在 mmap 中的原生 IQ 包上无法启动**（#369）：
`ExpertCache::verify_slot: slot 0 differs from the arena at byte 0 (of 2662400)`。

**8. 用于 IQ4_XS gate/up 行的 AVX-2 多 token 内核在这里价值约 3%。** 用 `STRATA_NO_IQ256=1` 做 A/B（它会丢弃该路径；IQ4_NL 的 `down` 行有自己的开关，在两臂中都保持开启），四轮交错，顺序轮换，以运行总时间比较：该内核**4 对全胜**，中位数 **−3.2%**（去掉一个离群轮后为 −2.6%），gate/up 阶段 −7.6%，持续行读取 28.6 对 25.6 GB/s。之所以小，是因为两臂都处于内存墙——收益在于每个专家行在每个验证窗口遍历一次，而不是每个 token 一次。

## 我会据此怎么做

- 源码注释中的 arena-vs-映射排名是在 63 GB 上 34 GB 专家时测得的，那里未固定的 arena 仍然获胜。在 64 GB 上 61 GiB 时它输了，而且每次启动的加载成本（约 30 秒）变得可见。第二个数据点值得放在第一个旁边。
- 预取默认值无需更改：在 `--kv-resident` 保持不变的情况下，关闭 / 2048 / 8192 在三个模型和两条专家路径上都落在 0–3% 以内。值得在文档中写一行的是 `--kv-resident`：在专家无法装入 RAM 的模型上，用 20480 代替默认的 32768 价值约 15%，因为它为更多专家槽留出了空间。
- 当页锁定预算大到足以挤出同一模型正在读取的文件缓存时给出警告，可以为下一个人省去一次令人困惑的基准测试。

## 注意事项

一台机器，一个模型，离线模式（无服务器，无跨请求的 KV 流式传输），每次运行 300 token，贪心 + `--spec 4`。tok/s 列在固定 draft 轮数下可复现到 ±1.5%；冷列取决于上一次运行在 RAM 中留下了多少文件。