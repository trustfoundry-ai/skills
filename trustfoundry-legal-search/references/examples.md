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

async function agenticSearch(
  query: string,
  state = "FED",
  maxRetries = 3
): Promise<SearchResult[]> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = await fetch(`${BASE_URL}/public/v1/agentic-search`, {
      method: "POST",
      headers: {
        "X-API-Key": API_KEY!,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ query, default_state: state }),
    });

    // Handle rate limit (429) with exponential backoff
    if (res.status === 429) {
      const retryAfter = parseInt(res.headers.get("Retry-After") ?? "10", 10);
      const body = await res.json();

      // Quota exceeded = stop retrying (resets at billing period boundary)
      if (body.error === "Quota exceeded") {
        throw new Error(`Quota exceeded. Resets in ${retryAfter}s`);
      }

      // Rate limit = back off and retry
      if (attempt === maxRetries) {
        throw new Error("Rate limited after max retries");
      }
      const wait = retryAfter * Math.pow(2, attempt) * 1000;
      console.warn(`Rate limited. Waiting ${wait / 1000}s (attempt ${attempt + 1}/${maxRetries})`);
      await new Promise((r) => setTimeout(r, wait));
      continue;
    }

    // Non-retryable errors
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

    // Monitor remaining capacity from response headers
    const rlRemaining = res.headers.get("X-RateLimit-Remaining");
    const quotaRemaining = res.headers.get("X-Quota-Remaining");
    const quotaWarning = res.headers.get("X-Quota-Warning");
    if (rlRemaining && parseInt(rlRemaining) < 5) {
      console.warn(`Rate limit low: ${rlRemaining} requests remaining this minute`);
    }
    if (quotaWarning === "true") {
      console.warn(`Quota warning: ${quotaRemaining} remaining for ${res.headers.get("X-Quota-Category")}`);
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

def agentic_search(query: str, state: str = "FED", max_retries: int = 3) -> list[dict]:
    """Execute an agentic search with exponential backoff on rate limits."""
    import time

    for attempt in range(max_retries + 1):
        with httpx.stream(
            "POST",
            f"{BASE_URL}/public/v1/agentic-search",
            headers={"X-API-Key": API_KEY, "Content-Type": "application/json"},
            json={"query": query, "default_state": state},
        ) as res:
            # Handle 429 — distinguish rate limit vs quota exceeded
            if res.status_code == 429:
                retry_after = int(res.headers.get("retry-after", "10"))
                body = res.json()
                error_msg = body.get("error", "")

                # Quota exceeded = stop retrying (resets at billing period boundary)
                if error_msg == "Quota exceeded":
                    raise Exception(f"Quota exceeded. Resets in {retry_after}s")

                # Rate limit = back off and retry
                if attempt == max_retries:
                    raise Exception("Rate limited after max retries")
                wait = retry_after * (2 ** attempt)
                print(f"Rate limited. Waiting {wait}s (attempt {attempt + 1}/{max_retries})")
                time.sleep(wait)
                continue

            # Non-retryable errors
            if res.status_code == 402:
                raise Exception("Insufficient credits")
            if res.status_code == 401:
                raise Exception("Invalid or missing API key")
            res.raise_for_status()

            # Monitor remaining capacity from response headers
            rl_remaining = res.headers.get("x-ratelimit-remaining")
            quota_remaining = res.headers.get("x-quota-remaining")
            quota_warning = res.headers.get("x-quota-warning")
            if rl_remaining and int(rl_remaining) < 5:
                print(f"Warning: Rate limit low ({rl_remaining} remaining this minute)")
            if quota_warning == "true":
                print(f"Warning: Approaching quota limit ({quota_remaining} remaining)")

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
