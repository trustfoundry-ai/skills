# Endpoint Reference

## POST /public/v1/agentic-search

AI-planned multi-source search. The agent determines which content types to search, builds a plan, reviews results.

**Request:**
```json
{
  "query": "What are the penalties for HIPAA violations?",
  "default_state": "FED"
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `query` | string | yes | Max 2000 chars |
| `default_state` | string | yes | 2-letter uppercase state code or `FED` |

**Response:** NDJSON stream (`application/x-ndjson`). See SKILL.md for event types.

**Product category:** `agentic_search`

---

## POST /public/v1/search

Direct single-source search. You specify content type explicitly. Fastest option.

**Request:**
```json
{
  "query": "exhaust emission standards for diesel engines",
  "state": "FED",
  "model_type": "reg_question"
}
```

| Field | Type | Required | Constraints |
|-------|------|----------|-------------|
| `query` | string | yes | Max 2000 chars |
| `state` | string | yes | 2-letter uppercase state code or `FED` |
| `model_type` | string | yes | One of: `case_question`, `law_question`, `reg_question`, `case_key_fact` |

**`model_type` values:**
- `case_question` — case law, question-style query
- `law_question` — statutes, question-style query
- `reg_question` — regulations, question-style query
- `case_key_fact` — case law, fact-pattern-style query

**Response:** NDJSON stream. Same event types as agentic-search (without `thinking_delta` and `confused`).

**Product category:** `fast_citation_search`

---

## GET /public/v1/search/results/{uuid}

Retrieve a previously saved search result set.

**Parameters:**
- `uuid` (path, required): UUID of the search set (from `citations_ready` event's `content.uuid`)

**Response (JSON):**
```json
{
  "uuid": "e29a934c-...",
  "query": "original search query",
  "created_at": "2025-10-06T17:15:40.125604",
  "search_results": [
    {
      "uuid": "d416bfc7-...",
      "header": "40 CFR § 89.112",
      "citation_tag": "[40 CFR § 89.112](https://www.ecfr.gov/...)",
      "url": "https://www.ecfr.gov/...",
      "excerpt": "Exhaust emission standards...",
      "relevance_score": 0.95,
      "result_type": "reg",
      "first_level_geo": "USA",
      "second_level_geo": "FED",
      "created_at": "2025-10-06T17:15:40.125604"
    }
  ]
}
```

**SearchSetResult fields:**
| Field | Type | Description |
|-------|------|-------------|
| `uuid` | string (UUID) | Result item ID — use with describe endpoint |
| `header` | string | Citation header (e.g., "40 CFR § 89.112") |
| `citation_tag` | string | Markdown citation with hyperlink |
| `url` | string (URI) | Direct link to source document |
| `excerpt` | string | Relevant text excerpt |
| `relevance_score` | number (0-1) | Relevance to query |
| `result_type` | string | `"law"`, `"reg"`, or `"case"` |
| `first_level_geo` | string | Country (e.g., "USA") |
| `second_level_geo` | string | Jurisdiction (e.g., "FED", "CA") |
| `created_at` | string (ISO 8601) | Timestamp |

**Product category:** `search_result_retrieval`

---

## GET /public/v1/search/results/items/describe/{uuid}

Get a summarized legal description of a search result item.

**Parameters:**
- `uuid` (path, required): UUID of the search result item

**Response (JSON):** Uses a `type` discriminator:

### Case law response (`type: "case"`)
```json
{
  "type": "case",
  "description": {
    "primary_holding": "...",
    "context_summary": "...",
    "key_facts_that_mattered": "...",
    "legal_principle_created_modified": "...",
    "legal_test_established": "...",
    "scope_of_application": "...",
    "exceptions_created": "...",
    "categories": ["..."],
    "cases_overruled_distinguished_by_this_case": ["..."]
  }
}
```

### Statute/regulation response (`type: "law_reg"`)
```json
{
  "type": "law_reg",
  "description": {
    "context_summary": "...",
    "key_points": "...",
    "categories": ["..."]
  }
}
```

All description fields are nullable. Dispatch on `type` to determine which fields are available.

**Product category:** `document_description`
