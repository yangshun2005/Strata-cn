# 精简文件

从 PR #628 移植到 0.1.40.2。为保持仓库精简，原始转储、日志和二进制文件已省略；
README、摘要、配置和每次运行的结果保持不变。以下文件原本位于该 PR 中。

13 个文件，2.7 MB：

| 文件 | 字节数 | 原因 |
|---|---:|---|
| `default/engine.txt` | 7,941 | 日志 |
| `default/raw.jsonl` | 904,198 | 日志/原始转储 |
| `default/warmup.json` | 110,485 | 原始转储 |
| `workers-15/engine.txt` | 5,448 | 日志 |
| `workers-15/raw.jsonl` | 196,504 | 日志/原始转储 |
| `workers-15/warmup.json` | 107,156 | 原始转储 |
| `workers-31-long/engine.txt` | 13,051 | 日志 |
| `workers-31-long/followup.json` | 196,801 | 较大（>100 KB） |
| `workers-31-long/raw.jsonl` | 704,282 | 日志/原始转储 |
| `workers-31-long/warmup.json` | 125,334 | 原始转储 |
| `workers-31/engine.txt` | 5,587 | 日志 |
| `workers-31/raw.jsonl` | 197,885 | 日志/原始转储 |
| `workers-31/warmup.json` | 108,019 | 原始转储 |