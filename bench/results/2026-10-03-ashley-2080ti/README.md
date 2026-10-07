# Ashley-PC：RTX 2080 Ti / i9-9900KF，Qwen3.8-Flash-Next IQ3_XXS

Strata 0.1.38（release 构建，archs 75/86/89/120，CUDA 13.0，便携版），Windows 11，
测量于 26 年 10 月 3 日。issue #165 谱系的 Turing 级数据点：目前树中最老的 GPU
（sm_75，2019 年芯片），11 GB VRAM，64 GB DDR4-3200。

## 硬件 / 软件

- GPU：RTX 2080 Ti 11 GB（驱动 591.47），PCIe gen3 x16
- CPU：Intel i9-9900KF（8C/16T）
- RAM：64 GB DDR4-3200 双通道（理论约 51 GB/s）
- 操作系统：Windows 11（10.0.26200），Python 3.12
- 模型：Qwen3.8-Flash-Next-GSQ-RCO-IQ3_XXS（47.0 GB 包 + PLE 拆分）
- 服务参数：expert-cache auto，prefill auto，spec 4，MTP rt，max-context 131072，
  kv int8，kv-resident 32768，vision，vram-reserve 700，spec-min-p 0.5。
  引擎自动选择：expert_cache 2275 MiB / 1350 slots，pcie_frac 0.35，
  conversation_cache_mib 0。
- 测量路径：引擎绑定在 Windows 机器的 127.0.0.1:8080；测试工具通过 SSH 隧道在
  第二台机器（Linux，RTX 5090 机器）上运行，局域网 RTT 约 2 ms。
  引擎日志通过 SSH 镜像以获取引擎侧指标。API 侧与引擎侧
  数字在约 1% 以内一致。

## 方法

与 2026-10-03-dev-5090 使用相同的测试工具（补丁副本位于 data/），强制冷 prefill
（全部 12 个单元格均为 `reused: 0`），每个单元格重复 3 次，256 token 解码。运行期间
引擎上没有其他流量。128k 第 1 次运行的引擎侧解码在镜像日志窗口中缺失；
API 侧已记录。

## 速度（3 次中位数）

| ctx  | TTFT (s) | prefill tok/s | decode tok/s（引擎） | decode（API） |
|------|----------|---------------|-----------------------|--------------|
| 1k   | 3.79     | ~295          | 36.9                  | 36.9         |
| 4k   | 9.43     | ~445          | 33.8                  | 33.8         |
| 32k  | 60.2     | 525           | 34.8                  | 34.9         |
| 128k | 266.1    | ~491          | 33.7（API）           | 33.8         |

草稿接受率（256 token 运行）：约 72-75%（例如 1k 时 167/217，32k 时 154/214）。
解码期间 GPU 利用率保持 99-100%；RAM 稳定在约 59.8/64 GB；GPU 在 260 W 上限下
约 200 W 时温度为 70-73 C。

## 解读

- 解码在 1k..128k 范围内平稳保持 33-38 tok/s：47 GB 专家流依赖 DDR4
  带宽和 2.3 GB 专家缓存，而非 VRAM。在此级别上上下文长度几乎不影响
  解码。
- Prefill 是 2019 年的代价：32k 时为 525 tok/s，而 5090 机器上为 6,300。128k
  提示需要 266 s 来读取。可用性来自热复用跳过 prefill，
  而冷基准测试刻意不进行这一操作。
- 11 GB VRAM 在 128k int8 KV 下容纳 KV + 常驻专家，剩余 114 MiB 空闲；
  未观察到 SSD 溢出（运行期间磁盘读取为 MB 级）。
- 此机器上 conversation_cache_mib 0：默认情况下热多轮复用支持不足；
  添加 conversation-cache 对是显而易见的下一项配置测试。

## 注意事项

- 单机，每个单元格单次通过（3 次重复），无 needle 运行。
- 仅第 1 次运行的 128k 引擎侧解码缺失（日志镜像窗口）；第 2-3 次
  引擎侧存在（34.5 / 33.7）。
- Windows 遥测通过引擎自身的 /metrics 硬件块（Windows 上的 psutil）。

文件：`IQ3_XXS-1x2080Ti/matrix.{md,json}`，`data/speed-<ctx>-<run>.json`（12 个），
`hw.txt`，补丁测试工具位于 `data/strata-bench.py`。