# Q8 prefill：使用 8,192 token 分块提速 6.7 倍

**未修改的 Strata v0.1.40，Unsloth Qwen3.8-Flash-Next Q8_0，RTX PRO 6000
Blackwell Workstation Edition 96 GB，Ryzen 9 7950X，128 GB RAM。** GPU 功耗
限制：400 W。测量日期：2026-10-06。

![使用 1024 与 8192 token 分块时的 Q8 prefill 和总请求时间](overview.png)

所测量的变更是从 `--prefill 1024` 改为 **`--prefill 8192`**。两种
配置均保持 FP16 KV、MTP T4、ngram 关闭、相同的权重和 pack、
15,472 个 GPU expert 槽位（75.25 GiB）、56 GiB 常驻 expert 预算、
`--no-prefill-borrow`，以及相同的适配设置。没有更改任何引擎代码，也没有将
优化历史导入此分支。

| 实际输入 | 起始 prefill | 8192 prefill | Prefill 加速比 | 总请求时间，起始 -> 8192 |
|---|---:|---:|---:|---:|
| 32,768 tokens | 52.87 s | **7.90 s** | **6.70x** | 59.71 -> **14.85 s** |
| 131,072 tokens | 213.18 s | **31.98 s** | **6.67x** | 220.25 -> **39.13 s** |

| 实际输入 | 起始输入 tok/s | 8192 输入 tok/s | 起始输出 tok/s | 8192 输出 tok/s | 有效输出 tok/s，起始 -> 8192 |
|---|---:|---:|---:|---:|---:|
| 32,768 | 619.8 | **4149.8** | 149.8 | 147.3 | 17.15 -> **68.98** |
| 131,072 | 614.9 | **4098.9** | 145.1 | 143.5 | 4.65 -> **26.18** |

每个请求生成了 **1,024 个 token**。总请求时间包含 prefill，不包含加载。
有效吞吐量为输出数量除以引擎 prefill 加解码时间。每个计时请求都重新读取了
其完整 prompt，prompt 复用为零。32K 请求分配的上下文为 40,960，128K 请求
为 139,264。

## 为什么更大的分块有帮助

最初的 32K 原生 Nsight trace 记录到 prefill 期间约 **1.50 TB 的
主机到设备拷贝**，速率为 **28.55 GB/s**。启动时的 PCIe 探测测得约
28.9 GB/s。仅这些字节就意味着 51.86 秒的传输下限，接近该 trace 中
53.06 秒的 prefill 标记。GPU 缓存之外的 expert 权重在 1,024-token
分块之间被反复流式传输。

更大的分块让更多输入 token 使用每次上传。更快配置的
硬件 profile 仍在收集中；此处不声称其新的传输量和上限。此结果比较的是
两种显式配置，而非默认自动预设与新的引擎实现。

## 冻结配置

[config.example.json](../../../configs/rtxpro-q8-prefill8192/config.example.json)
包含运行时参数。请根据你的机器编辑其资源路径。其
默认分配的上下文为 139,264，覆盖所测试的 128K 输入加输出。
[launch.py](../../../configs/rtxpro-q8-prefill8192/launch.py) 接受 `--engine`
和 `--context` 覆盖，并将额外参数传递给原生引擎：

```sh
python configs/rtxpro-q8-prefill8192/launch.py --dry-run
# For the native stdin/stdout protocol, not an HTTP listener:
sudo prlimit --pid $$ --memlock=unlimited:unlimited
python configs/rtxpro-q8-prefill8192/launch.py --serve
```

启动器会清除继承的 Strata 实验变量并设置所记录的环境。基准测试
覆盖 EOS 以强制 1,024 个输出；运行时示例保留正常停止。

确切的引擎标签为 v0.1.40，位于
`1cbcacbcae2953f3be9edc46369f0c875bc6ab8b`。
原生模型是六分片的 Unsloth Q8_0 GGUF 集合。现有的兼容
pack 包含用于小型投影的 BF16 转换。两组均使用相同的
pack、profile、draft 运行时和冻结的请求客户端。编码任务要求
一个优先队列模块和测试，并填充普通维护说明以达到
所记录的输入数量。思考已关闭，温度为零。

## 记录的检查

两种配置在两种输入长度下均完成，并通过了小型
`triangular(n)` 生成函数检查。输出流首次出现差异的位置为
**32K 的 token 17** 和 **128K 的 token 85**。长生成模块未
执行。这些是固定输出上限下的单次观测。

[results.csv](results.csv) 保存图表数据；
[measurements.json](measurements.json) 保存四条计时记录、二进制
哈希、启动参数和已提交的 token ID。使用 matplotlib 运行 `python plot.py`
以重新生成 PNG/SVG。此配置实验仍在调查中。