# 精简文件

从 PR #1270 移植到 0.1.40.2。为保持仓库精简，原始转储、日志和二进制文件已省略；
README、摘要、配置和每次运行的结果保持不变。以下文件原本位于该 PR 中。

52 个文件，5.4 MB：

| 文件 | 字节数 | 原因 |
|---|---:|---|
| `benchmark.py` | 7,132 | 超出该文件夹的大小预算 |
| `helper-prs/BUILD.txt` | 1,020 | 超出该文件夹的大小预算 |
| `helper-prs/benchmark.out` | 7,889 | 超出该文件夹的大小预算 |
| `helper-prs/final-status.json` | 1,168 | 超出该文件夹的大小预算 |
| `helper-prs/results.json` | 21,972 | 超出该文件夹的大小预算 |
| `helper-prs/telemetry.jsonl` | 373,931 | 大型原始转储 |
| `helper-stock/benchmark.out` | 7,879 | 超出该文件夹的大小预算 |
| `helper-stock/final-status.json` | 1,169 | 超出该文件夹的大小预算 |
| `helper-stock/needles.json` | 1,011 | 超出该文件夹的大小预算 |
| `helper-stock/results.json` | 21,802 | 超出该文件夹的大小预算 |
| `helper-stock/telemetry.jsonl` | 860,372 | 大型原始转储 |
| `helper-sw/benchmark.out` | 7,877 | 超出该文件夹的大小预算 |
| `helper-sw/final-status.json` | 1,168 | 超出该文件夹的大小预算 |
| `helper-sw/needles.json` | 1,011 | 超出该文件夹的大小预算 |
| `helper-sw/results.json` | 22,006 | 超出该文件夹的大小预算 |
| `helper-sw/telemetry.jsonl` | 449,391 | 大型原始转储 |
| `monitor.py` | 1,487 | 超出该文件夹的大小预算 |
| `single-prs/BUILD.txt` | 1,092 | 超出该文件夹的大小预算 |
| `single-prs/benchmark.out` | 7,891 | 超出该文件夹的大小预算 |
| `single-prs/final-status.json` | 1,179 | 超出该文件夹的大小预算 |
| `single-prs/results.json` | 21,839 | 超出该文件夹的大小预算 |
| `single-prs/telemetry.jsonl` | 373,453 | 大型原始转储 |
| `single-stock/benchmark.out` | 7,882 | 超出该文件夹的大小预算 |
| `single-stock/final-status.json` | 1,179 | 超出该文件夹的大小预算 |
| `single-stock/needles.json` | 1,011 | 超出该文件夹的大小预算 |
| `single-stock/results.json` | 21,741 | 超出该文件夹的大小预算 |
| `single-stock/telemetry.jsonl` | 855,976 | 大型原始转储 |
| `single-sw/benchmark.out` | 7,886 | 超出该文件夹的大小预算 |
| `single-sw/final-status.json` | 1,179 | 超出该文件夹的大小预算 |
| `single-sw/results.json` | 21,754 | 超出该文件夹的大小预算 |
| `single-sw/telemetry.jsonl` | 448,979 | 大型原始转储 |
| `split-prs/BUILD.txt` | 1,019 | 超出该文件夹的大小预算 |
| `split-prs/benchmark.out` | 7,876 | 超出该文件夹的大小预算 |
| `split-prs/final-status.json` | 1,182 | 超出该文件夹的大小预算 |
| `split-prs/results.json` | 21,669 | 超出该文件夹的大小预算 |
| `split-prs/telemetry.jsonl` | 239,191 | 大型原始转储 |
| `split-stock-gfxoffon/benchmark.out` | 4,705 | 超出该文件夹的大小预算 |
| `split-stock-gfxoffon/final-status.json` | 1,182 | 超出该文件夹的大小预算 |
| `split-stock-gfxoffon/results.json` | 14,843 | 超出该文件夹的大小预算 |
| `split-stock-gfxoffon/telemetry.jsonl` | 375,870 | 大型原始转储 |
| `split-stock/benchmark.out` | 7,885 | 超出该文件夹的大小预算 |
| `split-stock/final-status.json` | 1,183 | 超出该文件夹的大小预算 |
| `split-stock/results.json` | 21,863 | 超出该文件夹的大小预算 |
| `split-stock/telemetry.jsonl` | 522,697 | 大型原始转储 |
| `split-sw-gfxoffon/benchmark.out` | 4,715 | 超出该文件夹的大小预算 |
| `split-sw-gfxoffon/final-status.json` | 1,180 | 超出该文件夹的大小预算 |
| `split-sw-gfxoffon/results.json` | 14,809 | 超出该文件夹的大小预算 |
| `split-sw-gfxoffon/telemetry.jsonl` | 240,763 | 大型原始转储 |
| `split-sw/benchmark.out` | 7,875 | 超出该文件夹的大小预算 |
| `split-sw/final-status.json` | 1,182 | 超出该文件夹的大小预算 |
| `split-sw/results.json` | 21,852 | 超出该文件夹的大小预算 |
| `split-sw/telemetry.jsonl` | 281,987 | 大型原始转储 |