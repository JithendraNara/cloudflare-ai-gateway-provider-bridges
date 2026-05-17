# OpenClaw Integration

This file is the OpenClaw-specific path for using Cloudflare AI Gateway as a provider bridge.

Use it for:

- CrofAI through Cloudflare `/compat`.
- Vertex AI through Cloudflare `/compat` with a BYOK alias.
- Replacing LiteLLM when OpenClaw can send the required Cloudflare headers directly.
- Keeping LiteLLM when OpenClaw needs virtual keys, spend limits, detailed proxy logs, or LiteLLM fallback routing.

Everything here uses placeholders. Do not commit real account IDs, gateway IDs, tokens, provider keys, BYOK aliases, project IDs, service account JSON, hostnames, or local paths.

## What Official OpenClaw Supports

OpenClaw has an official `cloudflare-ai-gateway` provider for Anthropic models routed through Cloudflare AI Gateway. That official provider uses a Cloudflare Anthropic gateway URL and an upstream provider API key.

For CrofAI and Vertex AI through Cloudflare `/compat`, use OpenClaw custom providers:

- `models.providers.<id>`
- `baseUrl: "https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat"`
- `api: "openai-completions"`
- explicit `headers` for Cloudflare gateway auth and BYOK alias selection

OpenClaw's custom provider docs also support safe additive updates with `openclaw config set ... --strict-json --merge`.

## Environment

Put these in the environment that the OpenClaw gateway process actually sees. A key in an interactive shell does not help a launchd/systemd daemon unless that environment is imported into the daemon.

```bash
export CF_ACCOUNT_ID="<ACCOUNT_ID>"
export CF_GATEWAY_ID="<GATEWAY_ID>"
export CF_AIG_TOKEN="<CLOUDFLARE_AIG_RUNTIME_TOKEN>"
export VERTEX_BYOK_ALIAS="<VERTEX_BYOK_ALIAS>"
```

Gateway requests use:

```http
cf-aig-authorization: Bearer <CLOUDFLARE_AIG_RUNTIME_TOKEN>
```

Vertex BYOK requests also use:

```http
cf-aig-byok-alias: <VERTEX_BYOK_ALIAS>
```

## Direct Cloudflare Providers

Add providers with `models.mode: "merge"` so bundled OpenClaw providers remain available.

```js
{
  models: {
    mode: "merge",
    providers: {
      "cloudflare-crofai": {
        baseUrl: "https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat",
        api: "openai-completions",
        apiKey: "${CF_AIG_TOKEN}",
        authHeader: true,
        headers: {
          "cf-aig-authorization": "Bearer ${CF_AIG_TOKEN}"
        },
        models: [
          {
            id: "custom-crofai/kimi-k2.6",
            name: "CrofAI Kimi K2.6 via Cloudflare",
            reasoning: true,
            input: ["text"],
            contextWindow: 262144,
            contextTokens: 196608,
            maxTokens: 65536,
            compat: {
              requiresStringContent: true
            }
          },
          {
            id: "custom-crofai/glm-5.1",
            name: "CrofAI GLM 5.1 via Cloudflare",
            reasoning: true,
            input: ["text"],
            contextWindow: 202752,
            contextTokens: 131072,
            maxTokens: 65536,
            compat: {
              requiresStringContent: true
            }
          }
        ]
      },
      "cloudflare-vertex": {
        baseUrl: "https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat",
        api: "openai-completions",
        apiKey: "${CF_AIG_TOKEN}",
        authHeader: true,
        headers: {
          "cf-aig-authorization": "Bearer ${CF_AIG_TOKEN}",
          "cf-aig-byok-alias": "${VERTEX_BYOK_ALIAS}"
        },
        models: [
          {
            id: "google-vertex-ai/google/gemini-2.5-flash",
            name: "Vertex Gemini 2.5 Flash via Cloudflare",
            reasoning: false,
            input: ["text", "image"],
            contextWindow: 1048576,
            contextTokens: 262144,
            maxTokens: 65536
          },
          {
            id: "google-vertex-ai/google/gemini-2.5-pro",
            name: "Vertex Gemini 2.5 Pro via Cloudflare",
            reasoning: true,
            input: ["text", "image"],
            contextWindow: 1048576,
            contextTokens: 262144,
            maxTokens: 65536
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "cloudflare-crofai/custom-crofai/kimi-k2.6",
        fallbacks: [
          "cloudflare-vertex/google-vertex-ai/google/gemini-2.5-flash"
        ]
      }
    }
  }
}
```

Notes:

- Keep the model reference as `provider-id/model-id`.
- For Vertex through `/compat`, keep the publisher segment: `google-vertex-ai/google/<MODEL_ID>`.
- `authHeader: true` satisfies clients that expect an OpenAI-style bearer key. The Cloudflare gateway auth header is still explicit in `headers`.
- If your OpenClaw build accepts the config without `authHeader`, the explicit `cf-aig-authorization` header is the important part for Cloudflare gateway authentication.
- Use smaller `contextTokens` than the native context window when you want a practical runtime cap.

## Safe Additive Config Update

Use a config file edit when it is clearer. If scripting, prefer additive `config set` calls instead of replacing the whole provider catalog.

Example provider-only update:

```bash
openclaw config set models.providers.cloudflare-crofai '<JSON_OBJECT>' --strict-json --merge
openclaw config set models.providers.cloudflare-vertex '<JSON_OBJECT>' --strict-json --merge
```

Example model-list update:

```bash
openclaw config set models.providers.cloudflare-crofai.models '<JSON_ARRAY>' --strict-json --merge
```

Avoid `models.mode: "replace"` unless you intentionally want to drop bundled providers.

## Verification

Run these after every config edit:

```bash
openclaw config validate
openclaw models list --provider cloudflare-crofai
openclaw models list --provider cloudflare-vertex
```

Then copy the exact model reference shown by OpenClaw. Some model catalogs or local builds normalize display names and preview suffixes.

Recommended smoke path:

```bash
# Confirm the gateway process can read env vars.
openclaw config validate

# Confirm providers are visible.
openclaw models list --provider cloudflare-crofai
openclaw models list --provider cloudflare-vertex

# Start a fresh OpenClaw session with the exact model ref shown by models list.
```

If OpenClaw runs as a background gateway, restart that gateway after changing its environment.

## When to Keep LiteLLM

Keep LiteLLM between OpenClaw and Cloudflare when you need:

- Per-agent virtual keys.
- Spend caps and budgets.
- Detailed proxy logs.
- Centralized routing policy independent of OpenClaw config.
- Client-side compatibility when OpenClaw cannot send custom Cloudflare headers.

OpenClaw's LiteLLM provider uses:

```js
{
  models: {
    providers: {
      litellm: {
        baseUrl: "http://localhost:4000",
        apiKey: "${LITELLM_API_KEY}",
        api: "openai-completions",
        models: [
          {
            id: "cf-crofai-kimi",
            name: "CrofAI Kimi via Cloudflare via LiteLLM",
            reasoning: true,
            input: ["text"],
            contextWindow: 262144,
            maxTokens: 65536
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: { primary: "litellm/cf-crofai-kimi" }
    }
  }
}
```

In that setup, LiteLLM injects Cloudflare headers and OpenClaw only talks to LiteLLM.

## Troubleshooting

- `Invalid provider`: the Cloudflare custom provider slug is wrong. Custom providers use `custom-<slug>` in the model id.
- Google credentials error: Vertex BYOK provider key is missing, wrong, not attached, or the alias header is wrong.
- Vertex model not found: use `google-vertex-ai/google/<MODEL_ID>`, not `google-vertex-ai/<MODEL_ID>`.
- OpenClaw model not listed: check `models.mode`, provider id, model id, and `openclaw config validate`.
- Shell tests work, daemon fails: put env vars where the OpenClaw daemon can read them and restart it.
- Empty response content on reasoning models: increase `maxTokens` and inspect provider reasoning fields.
- Local outage still kills sessions: Cloudflare is only the provider gateway. Run OpenClaw itself on a cloud VM or persistent host if sessions must survive local power or internet cuts.

## References

- OpenClaw Cloudflare AI Gateway provider: https://docs.openclaw.ai/providers/cloudflare-ai-gateway
- OpenClaw custom providers and config tools: https://docs.openclaw.ai/gateway/config-tools
- OpenClaw LiteLLM provider: https://docs.openclaw.ai/providers/litellm
- Cloudflare authenticated gateway: https://developers.cloudflare.com/ai-gateway/configuration/authentication/
- Cloudflare BYOK provider keys: https://developers.cloudflare.com/ai-gateway/configuration/bring-your-own-keys/
