# Raw Test Data & Methodology

## Test Environment

- **AI Gateway:** `<GATEWAY_ID>` (name or ID)
- **Account ID:** `<ACCOUNT_ID>`
- **Provider:** `custom-minimax` (base_url: `https://api.minimax.io`)
- **Runtime Token:** `cfut_...` (AI Gateway Run permission)

The original test environment used real account metadata. This public document keeps only placeholder identifiers.

## Test 1: Basic Cache Warm-Up

```bash
# Request 1: Unique string (never seen before)
UNIQ="test-phrase-abc123"
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"model": "MiniMax-M2.7", "messages": [...], "max_tokens": 30}'

# Result:
# HTTP 200
# cf-aig-cache-status: MISS
# Time: 1.45s

# Request 2: Exact same payload
# Result:
# HTTP 200
# cf-aig-cache-status: HIT
# Time: 0.17s
```

**Conclusion:** First request caches, second request hits cache in 0.17s (8.5x speedup).

---

## Test 2: Multi-Turn Conversation

```bash
# Turn 1: Fresh conversation
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"model": "MiniMax-M2.7", "messages": [{"role": "user", "content": "My favorite color is blue."}], "max_tokens": 30}'

# Result: MISS, 0.86s

# Turn 2: Same conversation (exact same 3-message array)
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"model": "MiniMax-M2.7", "messages": [{"role": "user", "content": "My favorite color is blue."}, {"role": "assistant", "content": "I will remember that."}, {"role": "user", "content": "What is my favorite color?"}], "max_tokens": 30}'

# Result: MISS, 4.54s (longer prompt = more compute)

# Turn 3: Exact same as Turn 2
# Result: HIT, 0.17s
```

**Conclusion:** Full message arrays are cached as-is.

---

## Test 3: max_tokens in Cache Key

```bash
# Same prompt, max_tokens=20
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"max_tokens": 20, ...}'

# Result: MISS, 1.17s

# Same prompt, max_tokens=100 (different value)
# Result: MISS, 2.23s (not HIT)
```

**Conclusion:** `max_tokens` is part of the cache key.

---

## Test 4: Cache Without Header

```bash
# Request WITHOUT cf-aig-cache-ttl
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -d '{"model": "MiniMax-M2.7", "messages": [...]}'

# Result: cf-aig-cache-status: MISS (no caching)
```

**Conclusion:** `cf-aig-cache-ttl` header is REQUIRED to enable caching.

---

## Test 5: Skip Cache

```bash
# First: cache normally
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"model": "MiniMax-M2.7", "messages": [...]}'
# Result: MISS, cached

# Second: skip cache
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -H "cf-aig-skip-cache: true" \
  -d '{"model": "MiniMax-M2.7", "messages": [...]}'
# Result: MISS, not served from cache
```

**Conclusion:** `cf-aig-skip-cache: true` bypasses cache even if entry exists.

---

## Test 6: Custom Cache Key

```bash
# Cache with key-A
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -H "cf-aig-cache-key: cache-key-alpha" \
  -d '{"model": "MiniMax-M2.7", "messages": [...]}'
# Result: MISS

# Same with key-A again
# Result: HIT

# Same with key-B (different key)
# Result: MISS (different cache entry)
```

**Conclusion:** `cf-aig-cache-key` overrides default key computation.

---

## Test 7: Cross-User Cache Sharing

```bash
# User A asks question (first ever)
curl -i -X POST ".../custom-minimax/v1/chat/completions" \
  -H "cf-aig-cache-ttl: 300" \
  -d '{"model": "MiniMax-M2.7", "messages": [{"role": "user", "content": "Who won IPL 2024?"}]}'
# Result: MISS, 1.0s

# User B asks exact same question
# Result: HIT, 0.28s
```

**Conclusion:** Cache is shared across users for identical requests.

---

## Latency Benchmarks

| Scenario | MISS (fresh) | HIT (cached) | Speedup |
|----------|-------------|--------------|---------|
| Simple prompt (50 tokens) | 1.0-1.5s | 0.15-0.3s | 5-10x |
| Multi-turn (200 tokens) | 4.5s | 0.17s | 26x |
| Cross-user identical | 1.0s | 0.28s | 3.5x |

---

## Stress Test (Concurrent Requests)

```
Test: 5 parallel requests to chat completions
Result: 5/5 successful
Wall time: ~1.2s (parallel)
Individual avg: ~0.16s

Test: 15 sequential requests
Result: 15/15 successful
Avg time: 0.16s (all cached by then)
```

**Conclusion:** Gateway handles concurrent load well.
