# Agent Note: A pi-ai route names the header that carries the session id

Status: implemented

English | [中文](2026-09-07-pi-ai-configurable-session-header.zh.md)

## Problem

The adapter already passes the durable session id to pi-ai as `StreamOptions.sessionId`, but pi-ai turns that value into a request header only where the model's compat enables session affinity: `sendSessionAffinityHeaders` defaults to false, and `sessionAffinityFormat` selects among a closed set of vendor conventions (`openai`, `openai-nosession`, `openrouter`, and Anthropic's `x-session-affinity`). The harness withholds both fields from a profile's `compat`, because the pi-ai catalog it installs sets them for the vendors it names ([catalog.ts](../../../../packages/llm/llm-pi-ai/src/catalog.ts)). A hand-declared route is by construction an endpoint that catalog does not describe, so a private gateway that correlates, routes, or caches by a session field of its own received no session id on the wire, and nothing in the configuration could say otherwise.

## Decision

A provider profile may set `sessionHeader`, the HTTP field that carries the current session id on requests to that route. `requestHeaders` in [adapter.ts](../../../../packages/llm/llm-pi-ai/src/adapter.ts) merges `{ [sessionHeader]: String(options.sessionId) }` over the profile's static `headers` dict, drops every name colliding case-insensitively with the harness attribution set, and appends attribution last, so a profile naming an attribution field still sends the harness value. The per-request value wins over a static `headers` entry of the same name, and a request carrying no `GenerateOptions.sessionId` leaves that static entry as written.

Profile resolution refuses a name that is empty or whitespace-only and one Fetch cannot represent, through the same `new Headers` check the `headers` dict passes, so a misspelled field fails the plugin mount, the settings write, or the stored section at startup instead of dropping the header from every request.

The field is transport metadata addressed to the route's `baseURL`: it is not a session event, and it does not enter the request body, prompt, token accounting, or KV-cache identity. pi-ai's own `sessionId` option keeps its meaning, so a named catalog route continues to send the affinity headers its compat declares.

## Verification

- `tests/adapter.spec.ts` asserts that the configured field reaches a mock server carrying the request's session id, that a static `headers` value survives a request carrying no session id, and that naming a reserved attribution field leaves `User-Agent` at the harness value.
- The profile-resolution spec asserts that empty, whitespace-only, and Fetch-invalid names are refused.
- No keyless snapshot changes: the field is transport metadata, as [the DeepSeek request-identity decision](2026-08-11-deepseek-request-user-id-header.md) likewise found for its headers, so it never reaches model-visible or user-visible transcript content.

## Alternatives considered

| Rejected | Reason |
|---|---|
| Offer pi-ai's `compat.sendSessionAffinityHeaders` and `sessionAffinityFormat` | Both are withheld because the installed catalog sets them for a named vendor. Offering them puts a vendor-bound switch back on a hand-declared route, and the format enum still cannot name a gateway's own field; a profile that wants pi-ai's affinity behavior should be the catalog route that carries it. |
| Carry the value in the `headers` dict | That dict is static configuration resolved before any session exists, so the value would have to be written into the file, pinning every session on the route to one id. Naming the field in configuration while the request supplies the value keeps the two roles apart. |
| Send the session id from the provider-neutral attribution helper | [The mandatory-attribution decision](../architecture/2026-06-21-mandatory-app-attribution-headers.md) confines that helper to static product identity and forbids session ids in its fields; every adapter would send the id to every provider, including endpoints with no session concept. |
| One harness-wide header name | The name belongs to the receiving gateway, and one composition holds routes pointing at different gateways. A single name sends a field most of them did not ask for, while DeepSeek's own routes already carry `x-deepseek-harness-session-id`. |
| Derive the name from the `baseURL` | Detection answers for the endpoints pi-ai ships, which is exactly what a hand-declared route is not; the operator states the field instead of the harness guessing it. |

## Consequences

- The session id reaches whatever the route's `baseURL` names, including a gateway or proxy that logs fields it does not recognize. The field is opt-in per route, and a route that names none sends nothing new.
- A gateway requiring its own session field is now connectable without exposing the compat switches the harness withholds from hand-declared routes.
- Named catalog routes are unchanged: pi-ai keeps sending the affinity headers the catalog declares, from the same `sessionId` option.
