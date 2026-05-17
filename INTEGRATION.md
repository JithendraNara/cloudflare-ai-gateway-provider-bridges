# AI Agent Integration

## Overview

This document shows generic patterns for integrating Cloudflare AI Gateway with AI agents and applications. MiniMax is the worked example for provider-native endpoints; see [AGENT_BRIDGES.md](AGENT_BRIDGES.md) for CrofAI, Vertex AI, LiteLLM, OpenClaw, and OpenCode.

## Common Architecture

```
Your AI Agent (any framework)
    │
    ├─► MiniMax Direct ──► MiniMax API
    │       (voice/TTS, fast path)
    │
    └─► AI Gateway ──────► Custom Provider: MiniMax
            (chat, caching, observability)
```

## Integration Pattern: Cached Chat

For broadcast-style content (news alerts, sports scores, market updates):

```python
import requests
import os
from typing import Optional

class AIGatewayClient:
    def __init__(self, account_id: str, gateway_id: str, aig_token: str, provider_slug: str = "custom-minimax"):
        self.base_url = f"https://gateway.ai.cloudflare.com/v1/{account_id}/{gateway_id}"
        self.token = aig_token
        self.provider_slug = provider_slug

    def chat(
        self,
        messages: list,
        model: str = "MiniMax-M2.7",
        max_tokens: int = 500,
        cache_ttl: Optional[int] = None,
        skip_cache: bool = False,
    ) -> tuple[str, str]:
        """Returns (response_text, cache_status)"""
        headers = {
            "cf-aig-authorization": f"Bearer {self.token}",
            "Content-Type": "application/json",
        }

        if cache_ttl:
            headers["cf-aig-cache-ttl"] = str(cache_ttl)
        if skip_cache:
            headers["cf-aig-skip-cache"] = "true"

        payload = {
            "model": model,
            "messages": messages,
            "max_tokens": max_tokens,
        }

        response = requests.post(
            f"{self.base_url}/{self.provider_slug}/v1/chat/completions",
            headers=headers,
            json=payload,
            timeout=30,
        )

        cache_status = response.headers.get("cf-aig-cache-status", "UNKNOWN")
        data = response.json()

        return data["choices"][0]["message"]["content"], cache_status

    def messages(
        self,
        messages: list,
        system: str = "You are concise.",
        model: str = "MiniMax-M2.7",
        max_tokens: int = 500,
    ) -> dict:
        """Anthropic-compatible route; preferred for MiniMax-M2.7 thinking/text blocks."""
        response = requests.post(
            f"{self.base_url}/{self.provider_slug}/anthropic/v1/messages",
            headers={
                "cf-aig-authorization": f"Bearer {self.token}",
                "Content-Type": "application/json",
            },
            json={
                "model": model,
                "system": system,
                "messages": messages,
                "max_tokens": max_tokens,
            },
            timeout=30,
        )
        response.raise_for_status()
        return response.json()


# Initialize
client = AIGatewayClient(
    account_id=os.environ["CF_ACCOUNT_ID"],
    gateway_id=os.environ["CF_GATEWAY_ID"],
    aig_token=os.environ["AIG_TOKEN"],
)

# Broadcast pattern: first request MISS, subsequent requests HIT
content, cache = client.chat(
    messages=[{"role": "user", "content": "Latest CSK vs RCB score"}],
    cache_ttl=300,  # 5 minute cache for live scores
)
print(f"Cache: {cache}, Response: {content}")
```

## Integration Pattern: Text-to-Speech via AI Gateway

```python
def text_to_speech(
    text: str,
    voice_id: str = "male-qn-qingse",
    speed: float = 1.0,
) -> dict:
    """Generate speech via AI Gateway T2A endpoint."""
    response = requests.post(
        f"{base_url}/{provider_slug}/v1/t2a_v2",
        headers={
            "cf-aig-authorization": f"Bearer {aig_token}",
            "Content-Type": "application/json",
        },
        json={
            "model": "speech-2.8-hd",
            "text": text,
            "stream": False,
            "voice_setting": {
                "voice_id": voice_id,
                "speed": speed,
                "vol": 1.0,
                "pitch": 0,
            },
            "audio_setting": {
                "sample_rate": 32000,
                "bitrate": 128000,
                "format": "mp3",
                "channel": 1,
            },
        },
        timeout=30,
    )
    response.raise_for_status()
    return response.json()  # data.audio is a URL or hex string depending on output_format
```

## Integration Pattern: Image Generation

```python
def generate_image(
    prompt: str,
    model: str = "image-01",
    aspect_ratio: str = "1:1",
) -> dict:
    """Generate image via AI Gateway."""
    response = requests.post(
        f"{base_url}/{provider_slug}/v1/image_generation",
        headers={
            "cf-aig-authorization": f"Bearer {aig_token}",
            "Content-Type": "application/json",
        },
        json={
            "model": model,
            "prompt": prompt,
            "aspect_ratio": aspect_ratio,
        },
        timeout=60,
    )
    return response.json()
```

## When to Use Each Path

| Use Case | Path | Why |
|----------|------|-----|
| Broadcast content | AI Gateway + cache | One API call, many cache hits |
| Live data (scores) | AI Gateway, skip cache | Fresh data required |
| TTS voice output | AI Gateway T2A | Caching + observability |
| Simple requests | MiniMax Direct | Faster, no gateway overhead |
| Streaming responses | AI Gateway | Real-time token delivery |

## Environment Variables

```bash
# Cloudflare / AI Gateway
CF_ACCOUNT_ID=your_account_id
CF_GATEWAY_ID=your_gateway_id
AIG_TOKEN=cfut_your_runtime_token

# Optional: for direct MiniMax calls
MINIMAX_API_KEY=your_minimax_api_key
```

## Error Handling

```python
import time

def chat_with_retry(messages, max_retries=3, backoff_base=2):
    for attempt in range(max_retries):
        try:
            return client.chat(messages)
        except requests.exceptions.HTTPError as e:
            if e.response.status_code == 429:  # Rate limited
                time.sleep(backoff_base ** attempt)
                continue
            raise
    raise Exception("Max retries exceeded")
```

## Metrics to Track

| Metric | How to Get |
|--------|------------|
| Cache hit rate | Count `HIT` / total requests |
| Latency | Response time per request |
| Token usage | From response `usage` field |
| Error rate | Non-200 responses / total |

## Multi-Agent Broadcasting

For agents that serve multiple users simultaneously (e.g., a sports bot serving 1000 users asking the same question):

```python
# User A asks "CSK vs RCB score" → MISS (calls MiniMax)
# User B asks "CSK vs RCB score" → HIT (served from cache)
# User C asks "CSK vs RCB score" → HIT
# ... up to cache TTL

# Set cache TTL high for stable content
content, status = client.chat(
    messages=[{"role": "user", "content": "CSK vs RCB score"}],
    cache_ttl=3600,  # 1 hour — good for pre-match predictions
)
```

The cache key is based on the request body, so identical prompts from any user hit the same cached response.
