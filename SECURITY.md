# 安全

## 报告漏洞

请通过 GitHub 的私密漏洞报告功能**私下**报告任何在修复前不应公开的问题：仓库的 **Security** 标签页，**Report a vulnerability**（[直接链接](https://github.com/Niko1221/Strata/security/advisories/new)）。请说明你发现了什么、如何复现（请求、涉及的配置键、Strata 版本），以及攻击者能获得什么。我们会在那里回复，在发布版本中修复，并在安全公告中致谢——除非你要求我们不这样做。

普通的加固想法以及可以安全公开讨论的发现，欢迎作为 [Issue](https://github.com/Niko1221/Strata/issues)（#544 收集审计发现）或 Pull Request 提交，我们会像审查自己的代码一样审查它们。

支持范围：最新发布版本。修复会进入下一个发布版本，而不会进入更早的版本。

## 服务器暴露了什么

Strata 在你的 PC 上运行模型，并通过 HTTP 提供服务（`serve/server.py`）。开箱即用时，只有本机可以访问它。详细信息和每一项设置都在 [docs/DETAILS.md](docs/DETAILS.md) 中（“From other devices”、“Host names”、“Web pages without an API key”、“Tools from MCP servers”）。

- **监听位置。** 默认是 `127.0.0.1`。`--host 0.0.0.0`（或 `strata-<model>.json` 中的 `"host"`）会将其开放到你的网络，此时如果未设置 API key，服务器会发出警告。
- **API key。** 运行配置中的 `"api_key"`（或 `STRATA_API_KEY`）在 `/v1/*` 以及所有显示模型状态、请求或应答的端点上都是必需的（`/status`、`/metrics`、`/settings`、`/mcp`、`/props`、`/slots`、`/api/requests`、`/config`），并且所有 `POST` 请求也都需要。比较使用常量时间。在将服务器开放到网络或在其前面放置隧道之前，请先设置一个。
- **Host 检查（DNS 重绑定，0.1.38）。** 未设置 API key 时，服务器只应答 `Host` 为已知名称的请求（`localhost`、IP 地址、它监听的地址，以及当它监听范围超出本机时本机的名称）。其他请求会收到 403。`"allowed_hosts"` 可以添加名称；设置了 API key 时，由 key 决定。
- **Origin 检查（0.1.38）。** 未设置 API key 时，来自其他网站页面的浏览器 `POST` 请求到 `/v1/*`（携带 `Origin` 头）会收到 403，除非该 origin 是 Strata 自己的页面、`localhost`、允许的 host，或列在 `"trusted_origins"` / `"cors_origins"` 中。更改设置（`/settings` 以及 `/config`：页面中 Model settings 可以更改的少数运行配置键——绝不包含网络、key、MCP 或程序相关键）、`/load` 和 `/unload` 以及 MCP 工具，只接受来自 Strata 自己页面（或 `"trusted_origins"`）的 JSON，因此其他地方的页面无法更改设置或运行工具。
- **CORS** 默认关闭，除非 `"cors_origins"` 列出了 origin，且仅对 `/v1/*` 生效。
- **MCP 工具** 是选择性启用的：只有你放入运行配置中的服务器，且只有来自 Strata 自己页面并请求它们的请求才会使用。它们以你的用户权限运行，由模型决定何时调用它们。
- **请求监视器**（`/api-monitor`，在内存中保留最近的提示和应答）默认关闭，除非设置了 `"api_monitor": true`。

Strata 尚未经过外部安全审计。