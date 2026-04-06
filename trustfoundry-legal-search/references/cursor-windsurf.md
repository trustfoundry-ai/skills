# IDE-Specific Configurations

## Cursor (.mdc rule)

Save this file as `.cursor/rules/trustfoundry.mdc` in your project:

```yaml
---
description: TrustFoundry API integration guardrails for legal search endpoints
alwaysApply: false
globs: "**/*.{ts,js,py}"
---
```

```markdown
# TrustFoundry API Rules

## Authentication
- Header: `X-API-Key` — NOT `Authorization: Bearer` (Bearer is OAuth-only)
- API keys: `api_` prefix, 36 characters. Do not fabricate other prefixes.
- NEVER send `Session-Uuid` header with API key auth (causes 400)
- Store key in env var: `TRUSTFOUNDRY_API_KEY`

## Endpoints
- `POST /public/v1/agentic-search` — body: `{ "query": string, "default_state": string }`
- `POST /public/v1/search` — body: `{ "query": string, "state": string, "model_type": string }`
- `GET /public/v1/search/results/{uuid}` — retrieve saved search set
- `GET /public/v1/search/results/items/describe/{uuid}` — summarized legal description

## Streaming (NDJSON)
- Search endpoints return `application/x-ndjson` — one JSON object per line
- MUST parse line-by-line. `await res.json()` WILL FAIL on these endpoints
- Dispatch on `type` field: start, thinking_delta, search_start, search_end, citations_ready, confused, error, end
- Results are in `citations_ready` event at `content.search_results[]`

## Jurisdiction Codes
- `default_state` / `state`: 2-letter uppercase (CA, NY, TX) or `FED` for federal
- Do NOT use full state names

## model_type (POST /search only)
- Exactly: `case_question`, `law_question`, `reg_question`, `case_key_fact`
- No other values exist

## Describe Response (GET .../describe/{uuid})
- Discriminator: `type` field — either `"case"` or `"law_reg"`
- All description fields are nullable

## Error Handling (MUST implement)
- 429 + `Retry-After` header: rate limit or quota exceeded — back off and retry
- 402 + `{"error":"Insufficient credits","credits_required":N}`: need credits
- 401: invalid or missing API key
```

---

## Windsurf (.windsurfrules)

Save this file as `.windsurf/rules/trustfoundry.md` in your project:

```markdown
# TrustFoundry API Integration

## Auth: `X-API-Key` header (api_ prefix, 36 chars). NOT Bearer.

## Endpoints
- POST /public/v1/agentic-search — {query, default_state} → NDJSON stream
- POST /public/v1/search — {query, state, model_type} → NDJSON stream
- GET /public/v1/search/results/{uuid} → JSON SearchSet
- GET /public/v1/search/results/items/describe/{uuid} → JSON (type: "case"|"law_reg")

## Key Rules
- NDJSON: parse line-by-line, dispatch on `type`. Do NOT use res.json().
- model_type: case_question, law_question, reg_question, case_key_fact
- state/default_state: 2-letter uppercase or FED. Not full names.
- 429 → respect Retry-After. 402 → insufficient credits.
- Never send Session-Uuid with API key auth.
```
