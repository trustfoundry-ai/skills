---
name: trustfoundry-legal-search
description: >
  Integrate with the TrustFoundry legal research API. Provides authentication patterns,
  endpoint contracts, NDJSON streaming parsers, and error handling for legal search,
  citations, case law, statutes, and regulations. Use when building against
  TrustFoundry APIs or when the user mentions legal search, legal research,
  citations, TrustFoundry, or agentic search.
license: MIT
compatibility: Requires network access. Works with any language that supports HTTP streaming.
metadata:
  author: trustfoundry
  version: "1.0"
  homepage: "https://trustfoundry.ai"
---

# TrustFoundry Legal Research API

AI-powered legal research API. Search US laws, regulations, and case law with natural language.

## Authentication

- Header: `X-API-Key: api_...` (36-char key, `api_` prefix)
- Created in dashboard: Settings > Developers > API Keys
- Do NOT use `Authorization: Bearer` (reserved for OAuth Word Add-in only)
- Do NOT send `Session-Uuid` with API key auth (returns 400)

## Endpoints

| Method | Path | Request Body | Response |
|--------|------|-------------|----------|
| POST | /public/v1/agentic-search | `{query, default_state}` | NDJSON stream |
| POST | /public/v1/search | `{query, state, model_type}` | NDJSON stream |
| GET | /public/v1/search/results/{uuid} | — | JSON (SearchSet) |
| GET | /public/v1/search/results/items/describe/{uuid} | — | JSON (Description) |

For detailed request/response contracts, see [references/endpoints.md](references/endpoints.md).

## NDJSON Streaming

Search endpoints return `application/x-ndjson`. Each line is a complete JSON object. Parse line-by-line and dispatch on the `type` field. Do NOT use `res.json()` — it will fail.

**Event sequence:** `start` → `thinking_delta`* → `search_start`/`search_end`* → `citations_ready` → `end`

| Event | Payload | Notes |
|-------|---------|-------|
| `start` | `{type, query}` | Stream begins |
| `thinking_delta` | `{type, content}` | Status text (agentic only) |
| `search_start` / `search_end` | `{type, content}` | Sub-search lifecycle |
| `citations_ready` | `{type, content}` | Results — main payload |
| `confused` | `{type, content}` | Query not understood (agentic only) |
| `error` | `{type, content: {message}}` | Error occurred |
| `end` | `{type}` | Stream finished |

## Error Handling

MUST implement these handlers for stable integrations:

| Code | Condition | Response | Action |
|------|-----------|----------|--------|
| 429 | Rate limit exceeded | `{"error":"Rate limit exceeded"}` | Respect `Retry-After` header, back off |
| 429 | Quota exceeded | `{"error":"Quota exceeded"}` | Wait for reset per `Retry-After` |
| 402 | Insufficient credits | `{"error":"Insufficient credits","credits_required":N}` | Acquire credits or wait for quota reset |
| 401 | Invalid/missing API key | `{"error":"invalid_token"}` | Check `X-API-Key` header |

Response headers for monitoring: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `X-Quota-Limit`, `X-Quota-Remaining`, `X-Request-Id`.

## Hallucination Protection

Ground truth — do NOT generate alternatives:

- Auth header: `X-API-Key` only. Not `Authorization: Bearer`.
- Key prefix: `api_` (36 chars). No other prefix exists.
- `Session-Uuid` + API key = 400 error. Never combine them.
- `default_state`/`state`: 2-letter uppercase or `FED`. Not full state names.
- `model_type`: exactly `case_question`, `law_question`, `reg_question`, `case_key_fact`.
- Search responses are NDJSON, not JSON. Cannot use `res.json()`.
- Quota only increments on 2xx. Failed requests don't count.
- HTTP 402 = insufficient credits, not a payment method issue.
- Describe endpoint returns `type: "case"` or `type: "law_reg"` — dispatch on this.

## Code Examples

For complete TypeScript and Python examples with error handling, see [references/examples.md](references/examples.md).

### Minimal TypeScript

```typescript
const res = await fetch("https://api.trustfoundry.ai/public/v1/agentic-search", {
  method: "POST",
  headers: { "X-API-Key": process.env.TRUSTFOUNDRY_API_KEY!, "Content-Type": "application/json" },
  body: JSON.stringify({ query: "HIPAA violation penalties", default_state: "FED" }),
});

if (res.status === 429) throw new Error(`Retry after ${res.headers.get("Retry-After")}s`);
if (res.status === 402) throw new Error("Insufficient credits");

const reader = res.body!.getReader();
const decoder = new TextDecoder();
let buf = "";
while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buf += decoder.decode(value, { stream: true });
  const lines = buf.split("\n");
  buf = lines.pop()!;
  for (const line of lines) {
    if (!line.trim()) continue;
    const evt = JSON.parse(line);
    if (evt.type === "citations_ready") return evt.content.search_results;
    if (evt.type === "error") throw new Error(evt.content?.message);
  }
}
```

### Minimal Python

```python
import httpx, json, os
with httpx.stream("POST", "https://api.trustfoundry.ai/public/v1/agentic-search",
    headers={"X-API-Key": os.environ["TRUSTFOUNDRY_API_KEY"], "Content-Type": "application/json"},
    json={"query": "HIPAA penalties", "default_state": "FED"},
) as r:
    if r.status_code == 429: raise Exception(f"Retry after {r.headers['retry-after']}s")
    if r.status_code == 402: raise Exception("Insufficient credits")
    r.raise_for_status()
    for line in r.iter_lines():
        if not line.strip(): continue
        evt = json.loads(line)
        if evt["type"] == "citations_ready":
            for hit in evt["content"]["search_results"]:
                print(f"{hit['header']}: {hit['excerpt'][:80]}")
```
