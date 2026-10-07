# Strata R9700 / Windows 11 — 文件读取扩展性（实测，来自 strata-iq2_xs.log 的增量）

计数器：`strata serve: expert tiers: ... since the start RAM <n> blobs, files <n> blobs <MB> MB read (the GGUF in place)`
同一引擎会话内连续请求之间的增量。

| prompt tokens | new tokens read | file MB read this request | MB per new token |
|---|---|---|---|
| 406 | 406 | +581 | 1.43 |
| 1,042 | 296 | +727 | 2.46 |
| 1,997 | 959 | +1,639 | 1.71 |
| 11,042 | 9,050 | +12,707 | 1.40 |
| 53,470 | 42,433 | +31,666 | 0.75 |
| 125,247 | 71,782 | +45,238 | 0.63 |
| 228,869 | 103,402 | +65,285 | 0.63 |

（更早的小请求：406 tok → +581 MB，492 tok/91 new → +581 MB）

## 发现
- 文件读取量**随 prompt 长度**扩展（每个新 token 约 0.6–1.4 MB）→ PLE/expert 数据是按每个 prompt token 从 SSD 流式读取的（与文档一致："每个 token 通过 OS 缓存读取几行"）。这将 prefill 上限限制在约 1,000–1,300 tok/s。
- 并非每次引擎启动的一次性开销（此前的假设——已修正）。
- 真正的一次性开销：每次引擎启动后的第一个请求（93 tok 耗时 39.0 s，而之后一个几乎相同的约 91-tok 请求仅耗时 4.7 s）。
- 每请求固定开销 ≈ 4.4 s（线性拟合 a=4.4 s，边际速率 ≈ 1,300 tok/s；若将冷启动离群点纳入朴素拟合，则 a=12.4 s）。

## 冷启动证据
`strata generate: resident RAM mode: 6.95 GiB of experts in RAM (page-locked), 19403 in the GPU cache`
`FileExpertSource: mapped pinned cache complement ready: resident 6.95 GiB, pinned 6.95 GiB`（使用 --resident-budget-gib 16 和 30 时结果相同）