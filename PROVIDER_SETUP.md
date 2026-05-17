# Provider Setup: Custom Providers in AI Gateway

## Overview

Cloudflare AI Gateway supports custom providers via BYOK (Bring Your Own Keys). This guide uses MiniMax as the worked example, and the same pattern applies to OpenAI-compatible custom providers such as CrofAI.

## Prerequisites

- Cloudflare account with AI Gateway enabled (Workers Paid plan or higher)
- MiniMax API key (with active token plan)
- Two API tokens:
  - **AI Gateway Run** (`cfut_...`) — for runtime requests through the gateway
  - **AI Gateway Edit** (`cfat_...`) — for management/metadata (optional for this guide)

## Step 1: Create an API Token (Cloudflare)

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → **My Profile** → **API Tokens**
2. Create Token → **Create Custom Token**
3. Add permissions:
   - `AI Gateway > AI Gateway Run > Edit`
   - `AI Gateway > AI Gateway Edit > Edit` (optional, for management)
4. Save the token — you'll see `cfut_...` for runtime and `cfat_...` for management

## Step 2: Add MiniMax Provider Key (BYOK)

1. Go to **AI** → **AI Gateway** → Select your gateway
2. Navigate to **Settings** → **Provider Keys**
3. Click **Add Key**:
   - Provider: `Custom`
   - Name: `minimax` (this becomes your provider slug)
   - API Key: your MiniMax API key
4. Save

## Step 3: Configure the Custom Provider

Still in gateway settings, navigate to **Providers** → **Custom Providers** → **Add Provider**:

| Field | Value | Notes |
|-------|-------|-------|
| **Provider Name** | `minimax` | Lowercase, used in URL paths |
| **base_url** | `https://api.minimax.io` | API root; request paths include `/anthropic/...` or `/v1/...` |
| **Auth Type** | `API Key` | Select the Provider Key added in Step 2 |

### Working Route Map

MiniMax supports multiple API surfaces. With the custom provider slug `minimax`, call it through AI Gateway as `custom-minimax`.

| Use case | Gateway path | Notes |
|----------|--------------|-------|
| Chat, recommended | `/custom-minimax/anthropic/v1/messages` | Anthropic-compatible; separates thinking/text blocks |
| Chat, OpenAI-compatible | `/custom-minimax/v1/chat/completions` | Works, but M2.7 may include reasoning in `message.content` |
| Text-to-Speech | `/custom-minimax/v1/t2a_v2` | HTTP TTS |
| Image generation | `/custom-minimax/v1/image_generation` | Text-to-image |

Do not send the MiniMax key from your app. Store it as a Cloudflare provider key and use `cf-aig-authorization` for runtime requests.

Cloudflare appends everything after `/custom-minimax/` to this base URL:

```
Gateway path:  /custom-minimax/anthropic/v1/messages
base_url:      https://api.minimax.io
Upstream:      https://api.minimax.io/anthropic/v1/messages
```

## Step 4: Verify the Setup

```bash
export AIG_TOKEN="cfut_YOUR_RUN_TOKEN"
export ACCOUNT_ID="your_account_id"
export GATEWAY="your_gateway_id"

# Test Anthropic-compatible Messages
curl -s -w "\nStatus: %{http_code}\n" -X POST \
  "https://gateway.ai.cloudflare.com/v1/$ACCOUNT_ID/$GATEWAY/custom-minimax/anthropic/v1/messages" \
  -H "cf-aig-authorization: Bearer $AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "MiniMax-M2.7",
    "system": "Respond with only the requested final text.",
    "messages": [{
      "role": "user",
      "content": [{"type": "text", "text": "Return exactly: gateway-ok"}]
    }],
    "max_tokens": 20
  }'
```

Expected output includes a `content` array with a final text block: `gateway-ok`.

## Common Errors

| Error | Cause | Fix |
|-------|-------|-----|
| `code: 2008, Invalid provider` | Provider slug wrong in URL | Use `custom-minimax` not `minimax` |
| `login fail (401)` | MiniMax provider key not attached or not forwarded | Store the MiniMax key in Cloudflare BYOK and select it for the provider |
| `unknown model 'm2.7'` | Model name format wrong | Use full name: `MiniMax-M2.7` |
| 401 on gateway requests | Wrong runtime auth header | Use `cf-aig-authorization`, not `Authorization` |
| 404 on `/custom-minimax/v1/messages` | Missing Anthropic path prefix | Use `/custom-minimax/anthropic/v1/messages` |

## Provider URL Formats

Once configured, two URL formats work:

### Provider-Native (Full Path Control)
```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/custom-minimax/{provider-path}
```
Use when the endpoint path differs from OpenAI's standard `/v1/...` format.

### Unified API (OpenAI-Compatible)
```
https://gateway.ai.cloudflare.com/v1/{account}/{gateway}/compat/chat/completions
```
Model name format: `custom-minimax/MiniMax-M2.7`
