# Architecture — Cloudflare AI Gateway Provider Bridges

## System Overview

```
                           ┌──────────────────────────┐
                           │    Your Application       │
                           │    (AI Agent, CLI, App)     │
                           │     Scripts, Cron Jobs)   │
                           └────────────┬─────────────┘
                                        │ HTTP/REST
                    ┌───────────────────┼───────────────────┐
                    │                   │                    │
                    ▼                   ▼                    │
         ┌──────────────────┐  ┌─────────────────┐         │
         │ Provider Direct   │  │  AI Gateway     │         │
         │ (optional escape) │  │  (Chat, Cache)  │         │
         └──────────────────┘  └────────┬────────┘         │
                                        │                   │
                    ┌───────────────────┴───────────────────┐
                    │                                       │
                    ▼                                       ▼
         ┌──────────────────┐                   ┌─────────────────────┐
         │ Provider API     │                   │ Cloudflare Network   │
         │                  │                   │ (Edge Caching,       │
         │ MiniMax/CrofAI/  │                   │  Rate Limiting,      │
         │ Vertex/etc.      │                   │  Analytics)          │
         └──────────────────┘                   └──────────┬──────────┘
                                                            │
                                        ┌───────────────────┴───────────┐
                                        │       BYOK Provider Keys         │
                                        │   Custom + native providers      │
                                        └───────────────────┬───────────┘
                                                            │
                                    ┌─────────────────────────┴─────────────┐
                                    │                                     │
                                    ▼                                     ▼
                         ┌────────────────────┐              ┌──────────────────────┐
                         │   Chat Completions │              │   Text-to-Audio (T2A)│
                         │   /v1/chat/       │              │   /v1/t2a_v2        │
                         │   MiniMax-M2.7     │              │   speech-2.8-hd     │
                         └────────────────────┘              └──────────────────────┘
```

## Data Flow: Cached Request

```
User Request
    │
    ▼
Cloudflare Edge (AI Gateway)
    │
    ├─► Cache lookup (by request body hash)
    │       │
    │       ├── HIT ──► Return cached response (0.17s)
    │       │
    │       └── MISS ──► Forward to MiniMax API (1.5s)
    │                       │
    │                       ▼
    │                   MiniMax API
    │                       │
    │                       ▼
    │                   Response cached at edge
    │                       │
    └───────────────────────┘
                │
                ▼
            Response returned to user
```

## Directory Structure

```
cloudflare-ai-gateway-provider-bridges/
├── README.md              # Overview & quick start
├── PROVIDER_SETUP.md      # Adding MiniMax as a custom-provider example
├── AGENT_BRIDGES.md       # CrofAI, Vertex AI, LiteLLM, OpenClaw, OpenCode
├── ROUTING.md             # Unified API vs provider-native routing
├── CACHING.md             # Deep dive on caching behavior
├── ENDPOINTS.md           # All tested endpoints with examples
├── INTEGRATION.md         # Generic AI agent integration patterns
├── TESTING.md             # Raw test data and methodology
├── ARCHITECTURE.md        # This file
└── .env.example           # Environment variable template
```

## Key Design Decisions

### 1. Custom Provider over Native Integration

MiniMax is not a built-in provider in AI Gateway. We use the **Custom Provider** (BYOK) mechanism:
- Store MiniMax API key in Cloudflare (Provider Keys)
- Configure `base_url: https://api.minimax.io`
- Route all requests through `/custom-minimax/` path prefix

### 2. Per-Request Caching over Gateway-Level Default

Caching is opt-in per request via `cf-aig-cache-ttl` header rather than enabling it at the gateway level. This gives:
- Fine-grained TTL control per use case
- Explicit skip-cache for real-time queries
- Custom cache keys for segmentation

### 3. Provider-Native over Unified API

We use `/custom-minimax/v1/...` paths rather than `/compat/chat/completions` because:
- MiniMax has non-standard response formats
- Some params (like `texts` for embeddings) don't translate through compat layer
- Full access to MiniMax-specific features (T2A, image gen)

### 4. Two Auth Headers

| Layer | Header |
|-------|--------|
| AI Gateway proxy | `cf-aig-authorization: Bearer cfut_...` |
| Cloudflare REST API | `Authorization: Bearer cfat_...` |

Using the wrong header causes silent 401s. Both tokens start with `cf` but have different prefixes and purposes.

## Failure Modes

| Component | Failure | Impact | Mitigation |
|-----------|---------|--------|-----------|
| MiniMax API | Down | All requests fail | Dynamic routing to fallback |
| AI Gateway | Down | All requests fail | Direct MiniMax as fallback |
| Network | Latency spike | Slow responses | Caching reduces provider calls |
| Cache | Cold start | First request slow | Pre-warm cache for known queries |

## Cost Model

| Component | Cost |
|-----------|------|
| AI Gateway | $5/month (Workers Paid plan) |
| MiniMax | Token-based (from MiniMax plan) |
| Cloudflare egress | Variable by region |
| Workers AI fallback | Per-token pricing |

No extra cost for caching itself — it's a gateway feature.
