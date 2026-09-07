# Agent Note: pi-ai 路由指定携带会话 id 的头部

Status: implemented

[English](2026-09-07-pi-ai-configurable-session-header.md) | 中文

## 问题

适配器已经把持久化会话 id 作为 `StreamOptions.sessionId` 交给 pi-ai，但 pi-ai 只在模型的 compat 启用会话亲和时才把该值变成请求头部：`sendSessionAffinityHeaders` 默认为 false，`sessionAffinityFormat` 只能在封闭的厂商约定集合（`openai`、`openai-nosession`、`openrouter`，以及 Anthropic 的 `x-session-affinity`）中选择。harness 在 profile 的 `compat` 上扣留这两个字段，因为它安装的 pi-ai catalog 已为其中具名的厂商设置好它们（[catalog.ts](../../../../packages/llm/llm-pi-ai/src/catalog.ts)）。手工声明的路由按定义就是该 catalog 未描述的端点，因此一个按自有会话字段做关联、路由或缓存的私有网关在线路上收不到任何会话 id，配置里也无从表达。

## 决定

provider profile 可以设置 `sessionHeader`：发往该路由的请求上携带当前会话 id 的 HTTP 字段。[adapter.ts](../../../../packages/llm/llm-pi-ai/src/adapter.ts) 中的 `requestHeaders` 把 `{ [sessionHeader]: String(options.sessionId) }` 合并到 profile 的静态 `headers` 字典之上，再按大小写不敏感地丢弃与 harness 归属集合冲突的名称，并最后追加归属，因此把该字段指向归属字段的 profile 仍然发送 harness 的值。逐请求值优先于同名的静态 `headers` 条目，而请求不携带 `GenerateOptions.sessionId` 时该静态条目原样保留。

profile 解析拒绝空名称、仅空白名称，以及 Fetch 无法表示的名称，用的是 `headers` 字典所过的同一个 `new Headers` 检查，因此写错的字段名会以插件挂载失败、设置写入失败或启动时存储区段失败的形式暴露，而不是在每条请求上悄悄丢掉该头部。

该字段是发往路由 `baseURL` 的传输元数据：它不是会话事件，也不进入请求体、提示词、token 计量或 KV cache 身份。pi-ai 自己的 `sessionId` 选项语义不变，因此具名的 catalog 路由继续发送其 compat 声明的亲和头部。

## 验证

- `tests/adapter.spec.ts` 断言配置的字段以请求的会话 id 到达 mock 服务器，静态 `headers` 值在不携带会话 id 的请求上保留，以及把该字段指向保留的归属字段时 `User-Agent` 仍为 harness 值。
- profile 解析 spec 断言空、仅空白与 Fetch 无法表示的名称都被拒绝。
- 无需修改 keyless snapshot：该字段是传输元数据，[DeepSeek 请求身份决策](2026-08-11-deepseek-request-user-id-header.zh.md)对其头部的判断同样适用，它从不进入模型可见或用户可见的 transcript 内容。

## 考虑过的替代方案

| 已否决 | 原因 |
|---|---|
| 开放 pi-ai 的 `compat.sendSessionAffinityHeaders` 与 `sessionAffinityFormat` | 两者被扣留是因为安装的 catalog 已为具名厂商设置它们。开放会把厂商绑定的开关放回手工声明的路由，而格式枚举仍然无法命名网关自己的字段；想要 pi-ai 亲和行为的 profile 应当是承载该行为的 catalog 路由。 |
| 把值放进 `headers` 字典 | 该字典是在任何会话存在之前就解析完的静态配置，值只能写进文件，从而把该路由上的每个会话钉死在一个 id 上。在配置里命名字段、由请求提供取值，把这两种角色分开。 |
| 从提供方无关的归属辅助函数发送会话 id | [强制归属决策](../architecture/2026-06-21-mandatory-app-attribution-headers.zh.md)把该辅助函数限定为静态产品身份，并禁止其字段携带会话 id；那样每个适配器都会把 id 发给每个提供方，包括没有会话概念的端点。 |
| 使用一个 harness 全局的头部名称 | 名称属于接收请求的网关，而同一个组合里存在指向不同网关的路由。单一名称会发送多数网关并未索要的字段，而 DeepSeek 自己的路由已经携带 `x-deepseek-harness-session-id`。 |
| 从 `baseURL` 推导名称 | 探测回答的是 pi-ai 自带的端点，而手工声明的路由恰恰不是其中之一；由运维方声明该字段，而不是让 harness 去猜。 |

## 后果

- 会话 id 会到达路由 `baseURL` 所指向的任何位置，包括记录未知字段的网关或代理。该字段按路由选择性启用，未命名它的路由不会新增任何发送内容。
- 要求自有会话字段的网关现在可以接入，而不必为手工声明的路由开放 harness 所扣留的 compat 开关。
- 具名的 catalog 路由不受影响：pi-ai 继续从同一个 `sessionId` 选项发送 catalog 声明的亲和头部。
