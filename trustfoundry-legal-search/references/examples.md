# Code Examples

## TypeScript: Full Agentic Search with Error Handling

```typescript
const API_KEY = process.env.TRUSTFOUNDRY_API_KEY; // api_...
const BASE_URL = "https://api.trustfoundry.ai";

interface SearchResult {
  uuid: string;
  header: string;
  citation_tag: string;
  url: string;
  excerpt: string;
  relevance_score: number;
  result_type: "law" | "reg" | "case";
  first_level_geo: string;
  second_level_geo: string;
  created_at: string;
}

async function agenticSearch(query: string, state = "FED"): Promise<SearchResult[]> {
  const res = await fetch(`${BASE_URL}/public/v1/agentic-search`, {
    method: "POST",
    headers: {
      "X-API-Key": API_KEY!,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ query, default_state: state }),
  });

  // Handle rate limit / quota / credit errors
  if (res.status === 429) {
    const retryAfter = parseInt(res.headers.get("Retry-After") ?? "60", 10);
    throw new Error(`Rate/quota limited. Retry after ${retryAfter}s`);
  }
  if (res.status === 402) {
    const body = await res.json();
    throw new Error(`Insufficient credits. Need ${body.credits_required}`);
  }
  if (res.status === 401) {
    throw new Error("Invalid or missing API key. Check X-API-Key header.");
  }
  if (!res.ok) {
    throw new Error(`HTTP ${res.status}: ${await res.text()}`);
  }

  // Parse NDJSON stream
  const reader = res.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split("\n");
    buffer = lines.pop()!;
    for (const line of lines) {
      if (!line.trim()) continue;
      const event = JSON.parse(line);
      switch (event.type) {
        case "citations_ready":
          return event.content.search_results as SearchResult[];
        case "error":
          throw new Error(event.content?.message ?? "Stream error");
        case "confused":
          throw new Error(`Query not understood: ${event.content}`);
      }
    }
  }
  return [];
}

// Retrieve full details for a search result
async function describeResult(uuid: string) {
  const res = await fetch(`${BASE_URL}/public/v1/search/results/items/describe/${uuid}`, {
    headers: { "X-API-Key": API_KEY! },
  });
  if (!res.ok) throw new Error(`HTTP ${res.status}`);
  const data = await res.json();

  // Dispatch on type discriminator
  if (data.type === "case") {
    console.log("Holding:", data.description.primary_holding);
  } else if (data.type === "law_reg") {
    console.log("Summary:", data.description.context_summary);
  }
  return data;
}
```

## Python: Full Agentic Search with Error Handling

```python
import httpx
import json
import os
from dataclasses import dataclass

API_KEY = os.environ["TRUSTFOUNDRY_API_KEY"]  # api_...
BASE_URL = "https://api.trustfoundry.ai"

def agentic_search(query: str, state: str = "FED") -> list[dict]:
    """Execute an agentic search and return results."""
    with httpx.stream(
        "POST",
        f"{BASE_URL}/public/v1/agentic-search",
        headers={"X-API-Key": API_KEY, "Content-Type": "application/json"},
        json={"query": query, "default_state": state},
    ) as res:
        if res.status_code == 429:
            retry = res.headers.get("retry-after", "60")
            raise Exception(f"Rate/quota limited. Retry after {retry}s")
        if res.status_code == 402:
            raise Exception("Insufficient credits")
        if res.status_code == 401:
            raise Exception("Invalid or missing API key")
        res.raise_for_status()

        for line in res.iter_lines():
            if not line.strip():
                continue
            event = json.loads(line)
            if event["type"] == "citations_ready":
                return event["content"]["search_results"]
            if event["type"] == "error":
                raise Exception(event.get("content", {}).get("message", "Stream error"))
            if event["type"] == "confused":
                raise Exception(f"Query not understood: {event['content']}")
    return []


def direct_search(query: str, state: str, model_type: str) -> list[dict]:
    """Execute a direct search with explicit content type."""
    with httpx.stream(
        "POST",
        f"{BASE_URL}/public/v1/search",
        headers={"X-API-Key": API_KEY, "Content-Type": "application/json"},
        json={"query": query, "state": state, "model_type": model_type},
    ) as res:
        if res.status_code == 429:
            retry = res.headers.get("retry-after", "60")
            raise Exception(f"Rate/quota limited. Retry after {retry}s")
        if res.status_code == 402:
            raise Exception("Insufficient credits")
        res.raise_for_status()

        for line in res.iter_lines():
            if not line.strip():
                continue
            event = json.loads(line)
            if event["type"] == "citations_ready":
                return event["content"]["search_results"]
            if event["type"] == "error":
                raise Exception(event.get("content", {}).get("message", "Stream error"))
    return []


def describe_result(uuid: str) -> dict:
    """Get summarized description of a search result."""
    res = httpx.get(
        f"{BASE_URL}/public/v1/search/results/items/describe/{uuid}",
        headers={"X-API-Key": API_KEY},
    )
    res.raise_for_status()
    data = res.json()

    # Dispatch on type discriminator
    if data["type"] == "case":
        print(f"Holding: {data['description']['primary_holding']}")
    elif data["type"] == "law_reg":
        print(f"Summary: {data['description']['context_summary']}")
    return data


# Usage
results = agentic_search("What are the penalties for HIPAA violations?")
for r in results:
    print(f"[{r['result_type']}] {r['header']} (score: {r['relevance_score']})")
    print(f"  {r['excerpt'][:100]}...")
    # Get detailed description
    detail = describe_result(r["uuid"])
```
