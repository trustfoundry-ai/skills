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

## Rate Limiting & Quota Management

MUST implement proper rate limit handling. Without it, the API will reject most requests with 429s.

### Two types of 429 — distinguish by error message

| Error body | Cause | `Retry-After` means | Action |
|-----------|-------|---------------------|--------|
| `{"error":"Rate limit exceeded"}` | Too many requests/minute | Seconds until rate limit window resets | Back off, then retry |
| `{"error":"Quota exceeded"}` | Monthly/lifetime quota used up | Seconds until quota period resets | Stop calling this category until reset |

### Response headers — check on EVERY response (including 2xx)

| Header | Value | When present |
|--------|-------|-------------|
| `X-RateLimit-Limit` | Requests/minute allowed | Always |
| `X-RateLimit-Remaining` | Requests left this minute | Always |
| `X-RateLimit-Reset` | Unix timestamp when minute window resets | Always |
| `X-Quota-Category` | Product category (e.g. `fast_citation_search`) | Metered endpoints |
| `X-Quota-Limit` | Total quota for billing period | Metered endpoints |
| `X-Quota-Remaining` | Remaining quota | Metered endpoints |
| `X-Quota-Reset` | ISO 8601 timestamp when quota period resets | Metered endpoints |
| `X-Quota-Warning` | `true` when usage >= 75% of quota | Metered endpoints |
| `Retry-After` | Seconds to wait before retrying | 429 responses only |
| `X-Request-Id` | Unique request ID for debugging | Always |

### Required: Exponential backoff with Retry-After

```
on 429:
  1. Read Retry-After header (seconds)
  2. Wait AT LEAST that many seconds
  3. Retry with exponential backoff: wait * 2^attempt (cap at 60s)
  4. Max 3 retries, then surface the error

on 2xx:
  1. Check X-RateLimit-Remaining — if < 5, slow down request rate
  2. Check X-Quota-Warning — if "true", alert user they're near quota limit
  3. Check X-Quota-Remaining — if 0, stop making requests to this category
```

### Best practices for high-volume integrations

- **Pre-check remaining capacity**: Read `X-RateLimit-Remaining` and `X-Quota-Remaining` from every response. Do NOT fire requests blindly.
- **Pace requests**: If rate limit is 60/min, space requests ~1s apart. Bursting causes cascading 429s.
- **Distinguish rate limit vs quota**: Rate limits reset every minute (retry soon). Quota resets at the billing period boundary (could be days — stop retrying).
- **Log X-Request-Id on errors**: Include it when reporting issues to TrustFoundry support.
- **Never retry 401/402**: These are auth/billing errors, not transient. Fix the root cause.

### Error reference

| Code | Condition | Response | Action |
|------|-----------|----------|--------|
| 429 | Rate limit exceeded | `{"error":"Rate limit exceeded"}` | Read `Retry-After`, backoff, retry |
| 429 | Quota exceeded | `{"error":"Quota exceeded"}` | Read `Retry-After`, stop until quota resets |
| 402 | Insufficient credits (overage) | `{"error":"Insufficient credits","credits_required":N}` | Purchase credits or wait for quota reset |
| 401 | Invalid/missing API key | `{"error":"invalid_token"}` | Check `X-API-Key` header |

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
- API base: `https://api.trustfoundry.ai/public/v1/`. Do NOT invent paths on this domain.
- Dashboard (API keys, billing, account): `https://dashboard.trustfoundry.ai`. Do NOT invent other URLs.

## Code Examples

For complete TypeScript and Python examples with error handling, see [references/examples.md](references/examples.md).

### Minimal TypeScript (with retry)

```typescript
const API_KEY = process.env.TRUSTFOUNDRY_API_KEY!;
const BASE = "https://api.trustfoundry.ai";

async function searchWithRetry(query: string, state = "FED", maxRetries = 3) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = await fetch(`${BASE}/public/v1/agentic-search`, {
      method: "POST",
      headers: { "X-API-Key": API_KEY, "Content-Type": "application/json" },
      body: JSON.stringify({ query, default_state: state }),
    });

    if (res.status === 429) {
      const retryAfter = parseInt(res.headers.get("Retry-After") ?? "10", 10);
      const isQuota = (await res.json()).error === "Quota exceeded";
      if (isQuota) throw new Error(`Quota exceeded. Resets in ${retryAfter}s`);
      if (attempt === maxRetries) throw new Error("Rate limited after max retries");
      const wait = retryAfter * Math.pow(2, attempt) * 1000;
      console.warn(`Rate limited. Waiting ${wait / 1000}s (attempt ${attempt + 1})`);
      await new Promise((r) => setTimeout(r, wait));
      continue;
    }
    if (res.status === 402) throw new Error("Insufficient credits");
    if (res.status === 401) throw new Error("Invalid API key");
    if (!res.ok) throw new Error(`HTTP ${res.status}`);

    // Log remaining capacity
    const rlRemaining = res.headers.get("X-RateLimit-Remaining");
    const quotaRemaining = res.headers.get("X-Quota-Remaining");
    if (rlRemaining && parseInt(rlRemaining) < 5) console.warn(`Rate limit low: ${rlRemaining} remaining`);
    if (quotaRemaining) console.log(`Quota remaining: ${quotaRemaining}`);

    // Parse NDJSON stream
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
    return [];
  }
}
```

### Minimal Python (with retry)

```python
import httpx, json, os, time

API_KEY = os.environ["TRUSTFOUNDRY_API_KEY"]
BASE = "https://api.trustfoundry.ai"

def search_with_retry(query: str, state: str = "FED", max_retries: int = 3):
    for attempt in range(max_retries + 1):
        with httpx.stream("POST", f"{BASE}/public/v1/agentic-search",
            headers={"X-API-Key": API_KEY, "Content-Type": "application/json"},
            json={"query": query, "default_state": state},
        ) as r:
            if r.status_code == 429:
                retry_after = int(r.headers.get("retry-after", "10"))
                error = r.json().get("error", "")
                if error == "Quota exceeded":
                    raise Exception(f"Quota exceeded. Resets in {retry_after}s")
                if attempt == max_retries:
                    raise Exception("Rate limited after max retries")
                wait = retry_after * (2 ** attempt)
                print(f"Rate limited. Waiting {wait}s (attempt {attempt + 1})")
                time.sleep(wait)
                continue
            if r.status_code == 402:
                raise Exception("Insufficient credits")
            r.raise_for_status()

            # Log remaining capacity
            rl = r.headers.get("x-ratelimit-remaining")
            quota = r.headers.get("x-quota-remaining")
            if rl and int(rl) < 5: print(f"⚠ Rate limit low: {rl} remaining")
            if quota: print(f"Quota remaining: {quota}")

            for line in r.iter_lines():
                if not line.strip(): continue
                evt = json.loads(line)
                if evt["type"] == "citations_ready":
                    return evt["content"]["search_results"]
                if evt["type"] == "error":
                    raise Exception(evt.get("content", {}).get("message", "Stream error"))
    return []
```
