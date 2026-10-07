# 社区基准测试结果，索引

本目录中每个社区文件夹一行（为 0.1.40.2 从社区基准测试 PR 移植，作者已在提交中署名）。原始转储、日志和二进制文件已从文件夹中裁剪；每个裁剪后的文件夹都有一个 `TRIMMED.md`，列出被省略的内容。每个文件夹中的 README 包含完整方法。

如何阅读此表：

- **Prompt** 是提示（预填充）tok/s；**Output** 是解码 tok/s。带有 `/` 的单元格给出 4K / 32K / 128K 运行结果，除非单元格另有说明。
- 数字是提交者自己的中位数，取自每个文件夹的 README。不同的机器、提示和引擎版本：请谨慎比较各行。我们自己对同一引擎的数字在上方带日期的报告中（`2026-09-29-speed-0126` 及其他）。
- Strata 是提交者运行的引擎版本。
- PR 是原始 Pull Request（已被 2026-10-06 历史重写关闭；工作内容在 0.1.40.2 中）。

| 文件夹 | 硬件 | 模型 | Prompt tok/s | Output tok/s | Strata | PR |
|---|---|---|---|---|---|---|
| [2026-10-01-0.1.31-release](2026-10-01-0.1.31-release/) | RX 7900 XTX 24 GB, Ryzen 7 7700X | Coder IQ1_M | 545 到 900 中位数（0.1.30 到 0.1.31） | 16-34 到 17-65 | 0.1.30 / 0.1.31 | #404 |
| [2026-10-01-community-2x-rtx-pro-4500](2026-10-01-community-2x-rtx-pro-4500/) | 2x RTX PRO 4500 32 GB, Threadripper 7960X | Swift IQ3_XXS | 1,428 / 2,454 / 2,785 | 115 / 120 / 93 | 0.1.30 | #418 |
| [2026-10-02-community-2x-rtx-pro-4500-engine-0.1.36](2026-10-02-community-2x-rtx-pro-4500-engine-0.1.36/) | 2x RTX PRO 4500 32 GB, Threadripper 7960X | Swift IQ3_XXS | 2,960 / 5,065 / 5,762 | 127 / 130 / 105 | 0.1.36 | #418 |
| [2026-10-01-community-2x-titan-rtx](2026-10-01-community-2x-titan-rtx/) | 2x TITAN RTX 24 GB (NVLink), Xeon E5-2696 v4 | IQ3_S, 262K | 865 / 1,612 / 1,785 | 68 / 60 / 62 | 0.1.31 | #389 |
| [2026-10-01-community-rtx-pro-4500](2026-10-01-community-rtx-pro-4500/) | RTX PRO 4500 32 GB, Threadripper PRO 3975WX | UD-Q4_K_XL / UD-IQ4_XS / IQ3_S | 见 README | UD-Q4_K_XL 61.5 / 72.1 / 70.2; UD-IQ4_XS 82.0 / 95.8 / 91.4; IQ3_S 106 / 120 / 117 (2.7K / 64K / 128K) | 0.1.31 | #417 |
| [2026-10-01-community-rtx-5090-5950x](2026-10-01-community-rtx-5090-5950x/) | RTX 5090, Ryzen 9 5950X (AVX2), 96 GB | UD-Q4_K_XL, IQ3_S, Swift IQ3_XXS | UD-Q4_K_XL 14.7K: 2,161 (0.1.31) | UD-Q4_K_XL 14.7K: 88.3 | 0.1.31 到 0.1.39 | #433 |
| [2026-10-01-ctx-ladder](2026-10-01-ctx-ladder/) | RTX 4070 Ti SUPER 16 GB (docker) | IQ2_XS | 167 / 128 / 133 (SSE 预填充, 32K / 64K / 128K) | 97 / 84 / 74 (32K / 64K / 128K, 128K 时 KV q4_0) | 见 README | #995 |
| [2026-10-05-community-rtx-4070-ti-super](2026-10-05-community-rtx-4070-ti-super/) | RTX 4070 Ti SUPER 16 GB, Ryzen 7 9800X3D | IQ2_XS / IQ3_XXS | 冷预填充见 README（32K-256K 时约 2,500） | IQ2_XS 97.9 / 94.3 / 90.0 / 82.4; IQ3_XXS 76 / 71 / 70 / 56 (32K / 64K / 128K / 256K) | 0.1.39 (6f32ec0) | #995 |
| [2026-10-01-iq4xs-64gb](2026-10-01-iq4xs-64gb/) | RTX 5060 Ti 16 GB (PCIe 4.0 x8), Ryzen 7 5700X3D, 64 GB | UD-IQ4_XS（60.9 GiB 专家层） | 不适用（每轮草稿毫秒数） | 33.9 固定 arena, 16.9-39 mmap 变体 | 0.1.31 / 0.1.32 | #416 |
| [2026-10-02-community-2x5060ti](2026-10-02-community-2x5060ti/) | 2x RTX 5060 Ti 16 GB (PCIe gen3), i9-9900KF | IQ3_S | 1,108 (20.9K 冷) | 56.4 (20.9K); 62.9 短 | 0.1.34 | #483 |
| [2026-10-02-community-rtx-5060-ti-x2](2026-10-02-community-rtx-5060-ti-x2/) | 2x RTX 5060 Ti 16 GB, Xeon E5-2690 v4 | IQ3_XXS, 262K | 808 / 1,931 / 2,199 | 77.5 / 72.7 / 72.4 | 0.1.35 | #508 |
| [2026-10-05-community-rtx-5060-ti-x2](2026-10-05-community-rtx-5060-ti-x2/) | 2x RTX 5060 Ti 16 GB, Xeon E5-2690 v4 | IQ3_XXS, 262K（重跑） | 911 / 2,087 / 2,440 | 84.8 / 86.5 / 83.8 | 0.1.39 | #508 |
| [2026-10-02-community-rtx-5090-iq3s](2026-10-02-community-rtx-5090-iq3s/) | RTX 5090, Ryzen 9 9950X3D, 96 GB | IQ3_S, 262K | 2,473 / 4,168 / 4,450 (4K / 32K / 131K) | 141.8 / 142.2 / 127.2 | 0.1.33 | #440 |
| [2026-10-02-community-rtx2080ti-11gb](2026-10-02-community-rtx2080ti-11gb/) | RTX 2080 Ti 11 GB, Threadripper 3960X, 128 GB | IQ3_S, 262K | 497 / 637 / 566 | 39.6 / 38.0 / 34.1 | 0.1.33 | #469 |
| [2026-10-02-community-rtx4000ada-layer-split](2026-10-02-community-rtx4000ada-layer-split/) | 2x RTX 4000 Ada (+ 1x SFF Ada), EPYC 7313P | IQ3_S | 2,110 (2 GPU) vs 779 (3 GPU), 16.6K | 73 vs 74 中位数 | 0.1.35 | #521 |
| [2026-10-02-community-rtx5090-1m-agent](2026-10-02-community-rtx5090-1m-agent/) | RTX 5090, Ryzen 9 7950X | IQ3_S, 1M 上下文 (YaRN) | 约 2,500 / 3,300 / 3,400 (4K / 32K / 128K) | 94-105 | 0.1.31 | #466 |
| [2026-10-02-community-rx9070xt-windows](2026-10-02-community-rx9070xt-windows/) | RX 9070 XT 16 GB, Windows 11 | IQ3_S | 约 600 | 45-46（首次运行 37） | 0.1.35 (AMD 引擎) | #499 |
| [2026-10-03-ashley-2080ti](2026-10-03-ashley-2080ti/) | RTX 2080 Ti, i9-9900KF, Windows 11 | IQ3_XXS | 32K 时 525 | 33-38 (1K 到 128K) | 0.1.38 | #698 |
| [2026-10-03-dev-5090](2026-10-03-dev-5090/) | RTX 5090, Ryzen 9 9950X3D | IQ3_S (0.1.34 / 0.1.38 / `STRATA_PF_FUSED=1`) | 4K: 4,308 / 4,846 / 5,361 | 4K: 178 / 193 / 189; 128K: 144 / 152 / 165 | 0.1.34, 0.1.38 | #698 |
| [2026-10-03-community-r9700-windows](2026-10-03-community-r9700-windows/) | Radeon AI PRO R9700 (gfx1201), Ryzen 9 9950X, Windows 11 | IQ2_XS | 856 (4K) / 1,182 (32K) / 1,232 (131K) / 1,130 (248K) | 99.8 (4K) / 97.8 / 89.9 / 87.5 | 0.1.38 | #618 |
| [2026-10-03-community-rtx4090-iq3s-140k-code](2026-10-03-community-rtx4090-iq3s-140k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_S, 140K, 代码提示 | 2,059 / 2,948 / 3,048 | 77.5 / 97.2 / 84.4 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3s-140k-ru](2026-10-03-community-rtx4090-iq3s-140k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_S, 140K, 俄语散文 | 2,084 / 2,991 / 3,078 | 76.1 / 102.5 / 102.2 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3xxs-200k-code](2026-10-03-community-rtx4090-iq3xxs-200k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, 代码提示 | 1,844 / 2,976 / 2,962 | 118 / 128 / 108 | 0.1.38 | #624 |
| [2026-10-03-community-rtx4090-iq3xxs-200k-ru](2026-10-03-community-rtx4090-iq3xxs-200k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, 俄语散文 | 1,824 / 2,912 / 2,984 | 94.2 / 85.6 / 90.3 | 0.1.38 | #624 |
| [2026-10-04-community-rtx4090-iq3xxs-200k-code](2026-10-04-community-rtx4090-iq3xxs-200k-code/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, 代码提示 | 1,966 / 2,974 / 3,085 | 124 / 135 / 120 | 0.1.39 | #624 |
| [2026-10-04-community-rtx4090-iq3xxs-200k-ru](2026-10-04-community-rtx4090-iq3xxs-200k-ru/) | RTX 4090, Ryzen 9 7950X | IQ3_XXS, 200K, 俄语散文 | 1,908 / 3,128 / 3,126 | 81.1 / 90.5 / 87.8 | 0.1.39 | #624 |
| [2026-10-03-rx6900xt-coder](2026-10-03-rx6900xt-coder/) | RX 6900 XT, Threadripper 3990X (Windows) | Coder IQ1_M | 313 (4K) / 350 (32K) | 45.3 (4K) 使用 31 个 worker vs 63 个时 11.6 | 0.1.38 | #628 |
| [2026-10-03-windows-groups](2026-10-03-windows-groups/) | RX 6900 XT, Threadripper 3990X (Windows) | Coder IQ1_M | 见 README | 12.7 到 46.1（短），13.1 到 47.4（4K）保留处理器组 | 0.1.38 + 修复 | #628 |
| [2026-10-03-community-rx7900xtx-rocm102-nightly](2026-10-03-community-rx7900xtx-rocm102-nightly/) | RX 7900 XTX 24 GB, ROCm 7.1.1 vs 10.2 nightly | IQ3_S | 921 vs 1,535（无 / 有匹配的 hipBLASLt） | 62.0 vs 61.0 | 0.1.38 | #745 |
| [2026-10-04-community-rx7900xtx-hipblaslt-100500](2026-10-04-community-rx7900xtx-hipblaslt-100500/) | RX 7900 XTX 24 GB, ROCm 10.2 nightly | IQ3_S, 128K | 926 到 1,687（gfx1100 表） | 62.3 | 0.1.38 谱系 | #745 |
| [2026-10-03-community-v100](2026-10-03-community-v100/) | Tesla V100-PCIE 32 GB, 2x Xeon E5-2696 v3, Windows | Flash-Next（见 README） | 见 README | 校准后 16.5 到 33.5；散文 22.0 | 0.1.38，为 0.1.39 更新 | #707 |
| [2026-10-03-community-z840-rtx3090ti](2026-10-03-community-z840-rtx3090ti/) | RTX 3090 Ti, 2x Xeon E5-2699 v3 (HP Z840) | IQ3_S, 262K | 1,239-1,304（约 5K） | 90-99（NUMA 变体） | 0.1.38 | #674 |
| [2026-10-04-community-2x-rtx3060](2026-10-04-community-2x-rtx3060/) | 2x RTX 3060 12 GB, Xeon E5-2678 v3 | IQ3_S | 603 / 1,156 / 1,323 (3.2K / 24.8K / 98.6K) | 43.6 / 42.5 / 40.0 | 0.1.37（源码） | #721 |
| [2026-10-04-community-rx-7900-gre](202