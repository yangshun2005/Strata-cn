# 0.1.40.1 如何处理模型写入的 `<tool_call>`，以及等待已死引擎的请求

测量于 2026-10-06，由 [Dmitry-B](https://github.com/Dmitry-B) 在
[2026-10-03-community-rtx4090-iq3xxs-200k](../2026-10-03-community-rtx4090-iq3xxs-200k/README.md)
中描述的机器上进行（RTX 4090、`qwen3.8-flash-next-iq3_xxs`、上下文 204800）。这是一份仅包含结果的报告，关于热修复版本
v0.1.40.1（`82f46a8`）的两项修复：模型写入的 `<tool_call>` 的规则（#804、#1058），以及引擎重启期间等待的请求（#1012）。
引擎二进制文件是为 0.1.40 构建的，且与其字节完全一致
（`196504f2822cbf15fb49b873e0000f41`）；只有 Python 服务器发生了变化。

## 运行了什么

| 文件 | 内容 |
| --- | --- |
| `probe_parser_corpus.py`、`corpus_results.json` | 作者自己的语料库 `serve/fixtures/rcall_specimens.json`，通过服务器所使用的同一个解析器重放 |
| `probe_toolcall.py`、`results.json` | 声明了工具的实时聊天请求，用于端到端地查看 API 返回什么 |
| `probe_restart_waiters.py`、`results_restart.json` | 三个等待引擎的请求，引擎在读取提示词时被杀死 |

## 1. 引用调用与搁浅调用语料库的重放

`serve/fixtures/rcall_specimens.json` 包含 37 个样本：16 个必须变成工具调用，21 个必须保持为文本。每个样本都被
喂给 `serve.frontend.OutputParser`——服务器自身使用的类——以六种喂入宽度（1、2、3、7、随机
1-9 个字符，以及一次性喂入整段文本）并在 `stream_tools` 开启和关闭的情况下进行：与
`serve/test_reasoning_rescue.py` 相同的矩阵，但按样本记录，而不是通过/失败。

```
образцов 37, должны дать вызов: 16, должны остаться текстом: 21, провалились: 0
```

37/37，444 次解析器运行，无不匹配，并且对于每个样本，流式结果都等于整段文本结果。
按样本的详细信息：[corpus_results.json](corpus_results.json)。

## 2. 声明了工具的实时请求

`get_weather` 在 `tools` 中声明；每个提示词要么以示例形式展示一个调用（在代码围栏或行内
代码中），要么请求一个真实的调用。`reasoning_effort` 按用例说明，因为规则关乎文本
所处的位置——在思考中还是在可见回答中。`totals.tool_calls_from_reasoning` 在整组
前后都被读取。

| 用例 | 预期 | 返回 | 调用文本停留的位置 | 判定 |
| --- | --- | --- | --- | --- |
| `1-fenced-example-in-answer`（思考开启） | 无调用 | 1 个调用 | `reasoning_content` | 不确定——见下文 |
| `2-inline-code-in-answer`（思考关闭） | 无调用 | 1 个调用 | 无（模型发出了一个顶层调用） | 不确定 |
| `3-fenced-example-in-thinking`（思考高） | 无调用 | 0 | `content`、`reasoning_content` | 匹配 |
| `4-turn-cut-by-max-tokens`（`max_tokens=60`，思考高） | 无调用 | 0 | `reasoning_content` | 匹配 |
| `5-real-call-top-level-after-sentence`（思考关闭） | 调用 | 1 个调用 | — | 匹配 |
| `6-plain-real-call` | 调用 | 1 个调用 | — | 匹配 |
| `7-streaming-split-fence`（流式，围栏跨块分割） | 无调用 | 0 | `content`、`reasoning_content` | 匹配 |
| `8-fenced-example-in-answer-no-thinking` | 无调用 | 0 | `content` | 匹配 |

`totals.tool_calls_from_reasoning` 在整组中保持 **0 → 0**：这些提示词都没有让模型
把调用搁浅在它的思考中，因此救援路径从未有任何东西可救援。用例 1、2、5 和 6 中的调用
是通过普通路径传入的。

这也是为什么用例 1 和 2 被记录为不确定而非失败。在用例 1 中，模型把
示例放在了其推理中的围栏块内（文本仍然在 `reasoning_content` 中），并产生了一个
单独的顶层调用；在用例 2 中，思考关闭时，它用一个顶层调用作答，而不是复制行内
代码。两者都是模型行为，而非规则被破坏——规则被应用于模型实际
写入的内容，而计数器显示没有执行救援。每个回答的原始文本：[results.json](results.json)。

端到端展示的内容：可见回答中代码围栏内的 `<tool_call>` 保持为文本，包括当
回答流式传输且围栏跨块分割时（用例 7、8），在被 `max_tokens` 截断的一轮思考中写入的调用
保持为文本（用例 4），普通调用继续正常工作（用例 5、6）。

## 3. 等待引擎死亡的请求（#1012）

三个带有约 60 000 token 提示词的请求被一起启动；15 秒后 `pkill` 杀死了引擎进程
（两个 pid，`39647` 和 `39659`）。

| 请求 | HTTP | 首个流式增量 | 请求结束 | 增量数 |
| --- | --- | --- | --- | --- |
| `waiter-1` | 200 | 无 | 启动后 16.6 秒 | 0 |
| `waiter-2` | 200 | 40.4 秒 | 启动后 40.5 秒 | 3 |
| `waiter-3` | 200 | 40.3 秒 | 启动后 40.4 秒 | 3 |

服务器在杀死后 25.5 秒启动了一个新引擎，`/health` 在同一时刻报告 `loaded: true`。
两个等待者用新引擎继续并在约 40 秒时完成；那个流已经打开的等待者在 16.6 秒时
被关闭，没有内容。没有任何东西挂起：没有请求等待 300 秒，也没有出现
`list.remove(x): x not in list`。作为对比，这是在 0.1.40 上约 300 秒后以 1 和 5 个 token
结束的用例——见
[2026-10-06-community-rtx4090-opt-ins](../2026-10-06-community-rtx4090-opt-ins/README.md)。

服务器端详细信息，包括它重启时使用的引擎命令行：[results_restart.json](results_restart.json)。

## 同一提交在这台机器上的单元测试

```
tools.test_setup_golden tools.test_setup_amd tools.test_setup_choices tools.test_setup_config
tools.test_setup_hotfix_tag                                                     -> 87 tests, OK
serve.test_responses serve.test_security serve.test_server serve.test_slots serve.test_runconfig
serve.test_mcp serve.test_monitor serve.test_frontend serve.test_structured serve.test_detok
serve.test_reasoning_tools serve.test_reasoning_loop_recovery serve.test_reasoning_rescue
serve.test_restart_waiters serve.test_parallel serve.test_vram                  -> 430 tests, OK (skipped=5)
serve.test_control_cancel serve.test_fatal_recovery                             -> 10 tests, OK
```

527 个测试，0 个失败。在 0.1.40 上同一组是 492 个；差异是本版本的两个新模块。

## 局限性

- 语料库重放测试的是解析器，而非 HTTP 层。端到端部分（第 2 节）在这个模型上没有产生
  真正的搁浅调用，因此救援路径没有通过 API 被观察到——只观察到了规则的"必须保持为文本"
  这一半。
- 三个等待者发送了相同的填充文本，因此引擎复用了前缀缓存（日志显示
  `prompt 141045 tokens = 140329 reused + 716 read`）。这对挂起测试没问题，对速度测试则是错误的；
  没有从中得出任何吞吐量结论。
- `waiter-1` 以 HTTP 200 且无内容结束，而不是发布说明中描述的 503：它的流
  已经打开，因此服务器无法再回答 503。按实测报告。
- 一台机器、一个模型、一次引擎重启。回合中途取消和批处理窗口 `ERR` 报告
  未被测试。