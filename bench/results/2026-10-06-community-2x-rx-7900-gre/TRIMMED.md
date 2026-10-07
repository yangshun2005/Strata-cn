# 精简文件

从 PR #1140 移植到 0.1.40.2。为保持仓库精简，原始转储、日志和二进制文件已省略；
README、摘要、配置和每次运行的结果保持不变。以下文件曾包含在原始 PR 中。

53 个文件，1.3 MB：

| 文件 | 字节数 | 原因 |
|---|---:|---|
| `data/ab-batch2-A.json` | 9,280 | 超出文件夹大小预算 |
| `data/ab-batch2-B.json` | 9,777 | 超出文件夹大小预算 |
| `data/ab-batch2-run.out` | 17,050 | 超出文件夹大小预算 |
| `data/ab-batch2-server-A.out` | 3,588 | 超出文件夹大小预算 |
| `data/ab-batch2-server-B.out` | 3,654 | 超出文件夹大小预算 |
| `data/ab-batch4-A.json` | 9,042 | 超出文件夹大小预算 |
| `data/ab-batch4-B.json` | 9,515 | 超出文件夹大小预算 |
| `data/ab-batch4-run.out` | 16,550 | 超出文件夹大小预算 |
| `data/ab-batch4-server-A.out` | 3,765 | 超出文件夹大小预算 |
| `data/ab-batch4-server-B.out` | 3,903 | 超出文件夹大小预算 |
| `data/ab-pipeline-off-A.json` | 9,678 | 超出文件夹大小预算 |
| `data/ab-pipeline-off-B.json` | 9,502 | 超出文件夹大小预算 |
| `data/ab-pipeline-off-run.out` | 17,195 | 超出文件夹大小预算 |
| `data/ab-pipeline-off-server-A.out` | 3,537 | 超出文件夹大小预算 |
| `data/ab-pipeline-off-server-B.out` | 3,590 | 超出文件夹大小预算 |
| `data/ab-resident-A.json` | 9,629 | 超出文件夹大小预算 |
| `data/ab-resident-B.json` | 9,716 | 超出文件夹大小预算 |
| `data/ab-resident-run.out` | 17,356 | 超出文件夹大小预算 |
| `data/ab-resident-server-A.out` | 3,649 | 超出文件夹大小预算 |
| `data/ab-resident-server-B.out` | 3,768 | 超出文件夹大小预算 |
| `data/ab-stage-trim-off-A.json` | 9,553 | 超出文件夹大小预算 |
| `data/ab-stage-trim-off-B.json` | 9,645 | 超出文件夹大小预算 |
| `data/ab-stage-trim-off-run.out` | 17,215 | 超出文件夹大小预算 |
| `data/ab-stage-trim-off-server-A.out` | 3,597 | 超出文件夹大小预算 |
| `data/ab-stage-trim-off-server-B.out` | 3,972 | 超出文件夹大小预算 |
| `data/batch-concurrent-batch2.engine.log` | 9,408 | 日志/二进制文件 |
| `data/batch-concurrent-batch2.server.out` | 3,193 | 超出文件夹大小预算 |
| `data/batch-concurrent-batch4.engine.log` | 9,789 | 日志/二进制文件 |
| `data/batch-concurrent-batch4.json` | 3,417 | 超出文件夹大小预算 |
| `data/batch-concurrent-batch4.server.out` | 8,004 | 超出文件夹大小预算 |
| `data/engine-ab-batch2.log` | 22,754 | 日志/二进制文件 |
| `data/engine-ab-batch4.log` | 28,942 | 日志/二进制文件 |
| `data/engine-ab-pipeline-off.log` | 23,963 | 日志/二进制文件 |
| `data/engine-ab-resident.log` | 33,455 | 日志/二进制文件 |
| `data/engine-ab-stage-trim-off.log` | 29,554 | 日志/二进制文件 |
| `data/engine-interleave.log` | 170,347 | 日志/二进制文件 |
| `data/engine.log` | 283,947 | 日志/二进制文件 |
| `data/hardware.txt` | 11,601 | 超出文件夹大小预算 |
| `data/interleave500-server.out` | 33,909 | 超出文件夹大小预算 |
| `data/interleave500.json` | 19,569 | 超出文件夹大小预算 |
| `data/run-all.out` | 14,490 | 超出文件夹大小预算 |
| `data/soak-engine.log` | 168,668 | 日志/二进制文件 |
| `data/soak-server.out` | 60,607 | 浸泡测试转储 |
| `data/soak.json` | 87,931 | 浸泡测试转储 |
| `scripts/ab-run.sh` | 3,861 | 超出文件夹大小预算 |
| `scripts/batch_concurrent.py` | 6,703 | 超出文件夹大小预算 |
| `scripts/collect_hardware.sh` | 3,190 | 超出文件夹大小预算 |
| `scripts/interleave500.py` | 10,474 | 超出文件夹大小预算 |
| `scripts/run-all.sh` | 2,806 | 超出文件夹大小预算 |
| `scripts/sample_mem.py` | 3,225 | 超出文件夹大小预算 |
| `scripts/soak.py` | 9,469 | 超出文件夹大小预算 |
| `scripts/soak_summary.py` | 4,785 | 超出文件夹大小预算 |
| `scripts/summarize.py` | 7,082 | 超出文件夹大小预算 |