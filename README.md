# CLI 代理 API

一个为 CLI 提供 OpenAI/Gemini/Claude/Codex/Grok 兼容 API 接口的代理服务器。

您可以通过任何与 OpenAI（包括 Responses）、Gemini（包括 Interactions）或 Claude 兼容的客户端或 SDK，以本地方式或多 CLI 账户访问以下提供商。

<table>
<tbody>
    <tr>
        <th align="center" width="100">提供商</th>
        <th align="center">说明</th>
    </tr>
    <tr>
        <td align="center"><a href="https://www.kimi.com/code/"><img src="./assets/logo/kimi.svg" alt="Kimi" width="28" height="28" /></a></td>
        <td>Kimi 系列模型，可通过 OAuth 或兼容 API 接入 CLIProxyAPI。</td>
    </tr>
    <tr>
        <td align="center"><a href="https://platform.openai.com/docs/"><img src="./assets/logo/openai.svg" alt="OpenAI" width="28" height="28" /></a></td>
        <td>OpenAI GPT 系列模型，可通过 Codex OAuth 或兼容 API 接入 CLIProxyAPI。</td>
    </tr>
    <tr>
        <td align="center"><a href="https://www.anthropic.com/claude"><img src="./assets/logo/claude.svg" alt="Anthropic" width="28" height="28" /></a></td>
        <td>Anthropic Claude 系列模型，可通过 Claude Code OAuth 或兼容 API 接入 CLIProxyAPI。</td>
    </tr>
    <tr>
        <td align="center"><a href="https://antigravity.google/"><img src="./assets/logo/antigravity.svg" alt="Antigravity" width="28" height="28" /></a></td>
        <td>Google Gemini 系列模型，可通过 Gemini CLI、AI Studio 或兼容 API 接入 CLIProxyAPI。</td>
    </tr>
    <tr>
        <td align="center"><a href="https://x.ai/grok"><img src="./assets/logo/xai.svg" alt="xAI" width="28" height="28" /></a></td>
        <td>xAI Grok 系列模型，可通过 Grok Build OAuth 或兼容 API 接入 CLIProxyAPI。</td>
    </tr>
</tbody>
</table>

## 功能特性

- 为 CLI 模型提供 OpenAI/Gemini/Claude/Codex/Grok 兼容的 API 端点
- OpenAI Codex（GPT 系列）OAuth 登录
- Claude Code OAuth 登录
- Grok Build OAuth 登录
- 支持流式、非流式响应，以及受支持场景下的 WebSocket 响应
- 函数调用/工具支持
- 多模态输入（文本、图片）
- 多账户支持与轮询负载均衡
- Gemini AI Studio API 密钥支持
- 通过配置接入 OpenAI 兼容上游（例如 OpenRouter）
- 可复用的 Go SDK（见 `docs/sdk-usage_CN.md`）

## 新手入门

CLIProxyAPI 用户手册：[https://help.router-for.me/cn/](https://help.router-for.me/cn/)

## 多账号场景下保持 Codex Prompt Cache 稳定

当 New API 等 OpenAI 兼容网关把 `/v1/responses` 请求转发给 CLIProxyAPI，并由 CLIProxyAPI 管理多个 Codex OAuth 账号时，请使用本节配置。

如果不启用会话亲和，`round-robin` 可能在每一轮请求时选择不同的 Codex 账号。Prompt Cache 通常不能跨账号/会话命名空间迁移，因此即使 `prompt_cache_key` 相同，缓存也可能反复退回冷缓存或只命中较短的前缀。

### 必需配置

编辑正在使用的 `config.yaml`（或 `--config` 指定的文件），修改其中已有的 `routing` 和 `codex` 段：

```yaml
routing:
  strategy: "round-robin"
  session-affinity: true
  session-affinity-ttl: "1h"

codex:
  identity-confuse: true
```

> [!IMPORTANT]
> 请修改已有的 `routing:` 和 `codex:` 段，不要添加重复的 YAML 顶层键。重复键可能被编辑器或 YAML 解析器覆盖，导致其中一段配置实际上没有生效。

各配置项作用：

- `routing.strategy: "round-robin"`：只在**新会话**首次进入时，把它分配给一个可用凭据。
- `routing.session-affinity: true`：在选择凭据之前读取稳定会话标识，包括 Responses body 中的 `prompt_cache_key`，然后把该会话绑定到同一个凭据。
- `routing.session-affinity-ttl: "1h"`：空闲绑定最多保留一小时；每次成功命中亲和绑定都会刷新过期时间。
- `codex.identity-confuse: true`：针对选中的 Codex 凭据，对缓存/跟踪标识做确定性重映射。启用 session affinity（或使用 `fill-first`）时生效。

对于带稳定缓存键的 Responses 请求，Codex 上游最终会收到一致的标识：

```text
body.prompt_cache_key = <按所选凭据隔离的稳定 ID>
Session_id            = <相同 ID>
Conversation_id       = <相同 ID>
```

这样同时建立两层亲和：

1. CLIProxyAPI 使用 `prompt_cache_key` 将会话固定到同一个 Codex OAuth 凭据；
2. CLIProxyAPI 使用一致的 `prompt_cache_key`、`Session_id` 和 `Conversation_id` 告诉 Codex 上游这是同一个会话。

如果请求没有缓存键，也不能从其他来源得到稳定会话标识，CLIProxyAPI不会仅为了 Responses 请求凭空创建这些缓存/会话标识。

### 在 New API 后面使用 CLIProxyAPI

请在 New API 中把 CLIProxyAPI 配置为 **OpenAI 兼容渠道**：

```text
渠道类型：OpenAI 兼容
Base URL：http://<cli-proxy-host>:8317
密钥：    CLIProxyAPI 配置中 api-keys 列表里的一个密钥
接口：    /v1/responses
模型：    CLIProxyAPI 对外提供的 Codex 模型
```

当上游是 CLIProxyAPI 时，不要把该渠道配置成 New API 的“ChatGPT Subscription/Codex”直连类型：该适配器面向 ChatGPT 私有的 `/backend-api/codex/responses`，而 CLIProxyAPI 对外提供的是标准 `/v1/responses`。

前置网关必须保留 Responses JSON body 中的 `prompt_cache_key`。客户端不需要自行生成 `Session_id` 或 `Conversation_id`；CLIProxyAPI会在请求 Codex 上游前，根据稳定缓存标识生成这两个值。

如果 New API 与 CLIProxyAPI 在同一台机器上，优先使用私有容器网络或宿主机网关地址，不要绕公网域名和 TLS 再回到本机。容器内的 `127.0.0.1` 指向该容器自身，并不指向宿主机或另一个容器。

### 应用配置与验证

CLIProxyAPI正常启动方式会监听 `config.yaml` 并自动热重载。保存后，请在日志中确认配置已成功重新加载。只有嵌入式部署或特殊启动方式禁用了 watcher 时，才需要重启进程。

然后至少串行发送三次请求，并保持：

- 模型相同；
- Prompt 前缀/请求 body 相同；
- 使用同一个高熵 `prompt_cache_key`；
- 输入 token 足够达到上游 Prompt Cache 门槛。

预期表现：

1. 第一个成功请求允许是冷缓存；
2. 后续成功请求应保持使用同一凭据，`cached_tokens` 通常会升高并保持稳定；
3. 日志应先出现一次 `session-affinity` 新绑定，随后同一会话持续出现 cache hit。

### 运维说明

- 会话亲和绑定保存在内存中。CLIProxyAPI重启会清空绑定；热更新 `routing.strategy`、`routing.session-affinity` 或 `routing.session-affinity-ttl` 也会替换 selector 并清空已有绑定，因此变更后的首个请求可能冷缓存。
- 如果已绑定凭据被禁用、限流、配额耗尽或暂时不可用，系统会自动故障转移到另一个凭据；切换后的首次缓存 miss 属于正常现象。
- 亲和键包含 provider、会话标识和模型；更换模型可能建立另一条绑定。
- 每个会话应使用唯一且高熵的 `prompt_cache_key`。不要让无关用户或无关对话共用一个静态 key；相同 key 会共享同一个亲和/缓存命名空间。
- `identity-confuse` 会按选中的 Codex 凭据隔离上游标识，但不能修复客户端没有 cache key 或每次请求都更换 key 的情况。
- 上游 overloaded 等拥塞错误与缓存亲和相互独立，启用本配置后仍可能发生。

如果缓存仍在高命中与 0/短前缀之间跳变，请依次确认：

1. 配置中只有一份实际生效的 `routing:` 和 `codex:`；
2. 运行时配置确实是 `session-affinity: true` 和 `identity-confuse: true`；
3. 前置网关没有删除 `prompt_cache_key`；
4. 客户端保持 key、模型和 Prompt 前缀稳定；
5. 已绑定的 Codex 凭据期间没有进入 cooldown 或发生故障转移。

## 管理 API 文档

请参见 [Management API 文档](https://help.router-for.me/cn/management/api)。

## SDK 文档

- 使用文档：[docs/sdk-usage_CN.md](docs/sdk-usage_CN.md)
- 高级（执行器与翻译器）：[docs/sdk-advanced_CN.md](docs/sdk-advanced_CN.md)
- 认证：[docs/sdk-access_CN.md](docs/sdk-access_CN.md)
- 凭据加载/更新：[docs/sdk-watcher_CN.md](docs/sdk-watcher_CN.md)
- 自定义 Provider 示例：`examples/custom-provider`

## 许可证

本项目使用 MIT 许可证，详情参见 [LICENSE](LICENSE)。
