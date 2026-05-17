# Agent Bridges: CrofAI, Vertex AI, LiteLLM, OpenClaw, and OpenCode

These notes capture the practical patterns for using Cloudflare AI Gateway as the central provider gateway for agent tools. Everything here is placeholder-only. Do not commit account IDs, gateway IDs, API tokens, provider keys, service account JSON, project IDs, local paths, hostnames, IPs, or personal names.

## Scope

Use Cloudflare AI Gateway for:

- Custom OpenAI-compatible providers such as CrofAI.
- Google Vertex AI through BYOK provider keys.
- Agent clients that can speak OpenAI-compatible chat completions.
- Optional LiteLLM routing when a client needs LiteLLM-specific virtual keys, model aliases, spend controls, or fallback logic.

Cloudflare AI Gateway centralizes provider access and key handling. It does not keep a local agent session alive through a local machine power-off or internet outage. For that, run the agent process itself on a cloud VM, container, or hosted worker.

## Placeholder Environment

```bash
export CF_ACCOUNT_ID="<ACCOUNT_ID>"
export CF_GATEWAY_ID="<GATEWAY_ID>"
export CF_AIG_TOKEN="<CLOUDFLARE_AIG_RUNTIME_TOKEN>"
export CROFAI_PROVIDER_SLUG="custom-crofai"
export VERTEX_BYOK_ALIAS="<VERTEX_BYOK_ALIAS>"
export LITELLM_API_KEY="<LITELLM_VIRTUAL_KEY>"
```

Use the documented gateway auth header for runtime calls:

```http
cf-aig-authorization: Bearer <CLOUDFLARE_AIG_RUNTIME_TOKEN>
```

Use the normal `Authorization: Bearer <TOKEN>` header for Cloudflare REST API management calls. Do not put upstream provider keys in app requests when BYOK is configured.

## Core URL Shapes

```text
OpenAI-compatible compat base:
https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat

Compat chat endpoint:
https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat/chat/completions

Custom provider-native endpoint:
https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/custom-<SLUG>/<PROVIDER_PATH>

Vertex provider-native endpoint:
https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/google-vertex-ai/v1/projects/<GCP_PROJECT_ID>/locations/<REGION>
```

For `/compat`, include the provider prefix inside `model`.

Examples:

```text
custom-crofai/kimi-k2.6
custom-crofai/glm-5.1
google-vertex-ai/google/gemini-2.5-pro
google-vertex-ai/google/gemini-2.5-flash
```

For Vertex through `/compat`, keep the `google/` publisher segment in the model string. `google-vertex-ai/gemini-...` is not the same as `google-vertex-ai/google/gemini-...`.

## CrofAI as a Custom Provider

Create a Cloudflare custom provider with:

```text
name: CrofAI
slug: custom-crofai
base_url: https://crof.ai/v1
```

Store the CrofAI provider key in Cloudflare BYOK. Then call it through `/compat`:

```bash
curl -sS "https://gateway.ai.cloudflare.com/v1/$CF_ACCOUNT_ID/$CF_GATEWAY_ID/compat/chat/completions" \
  -H "cf-aig-authorization: Bearer $CF_AIG_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "custom-crofai/kimi-k2.6",
    "messages": [
      { "role": "user", "content": "Return exactly: gateway-ok" }
    ],
    "max_tokens": 100
  }'
```

Notes:

- Use `/compat` first; it is the simplest path for OpenAI-compatible agent clients.
- Some reasoning models may produce `reasoning_content` or internal reasoning fields before final `content`. Use realistic `max_tokens` in smoke tests and make the client tolerant of empty final content on length-limited responses.
- If a model is newly added or renamed by the provider, Cloudflare custom provider routing usually does not need a new Cloudflare provider, only the right `custom-crofai/<model>` string.

## Vertex AI through BYOK

In Cloudflare dashboard, add a Google Vertex AI provider key using service account JSON. Use a unique alias and keep the alias in config as a placeholder such as `<VERTEX_BYOK_ALIAS>`.

Basic `/compat` smoke test:

```bash
curl -sS "https://gateway.ai.cloudflare.com/v1/$CF_ACCOUNT_ID/$CF_GATEWAY_ID/compat/chat/completions" \
  -H "cf-aig-authorization: Bearer $CF_AIG_TOKEN" \
  -H "cf-aig-byok-alias: $VERTEX_BYOK_ALIAS" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "google-vertex-ai/google/gemini-2.5-flash",
    "messages": [
      { "role": "user", "content": "Return exactly: vertex-ok" }
    ],
    "max_tokens": 100
  }'
```

Operational notes:

- `cf-aig-byok-alias` selects the stored provider key alias.
- A missing or wrong Vertex provider key commonly appears as a Google credentials error, not a Cloudflare token error.
- Prefer a specific Vertex region for provider-native calls. The `/compat` route uses the model prefix and the stored BYOK provider key configuration.
- Keep direct Vertex or direct provider aliases as rollback paths until Cloudflare smoke tests pass for the agent clients.

## LiteLLM Bridge

Use LiteLLM when you need an intermediate gateway for:

- Virtual keys per agent.
- Model aliases that hide provider-specific model names.
- Budget or spend controls.
- LiteLLM-specific routing and fallback behavior.
- Clients that cannot set the Cloudflare headers directly.

Cloudflare can replace LiteLLM for clients that can set:

- `baseURL` to the Cloudflare `/compat` base.
- `cf-aig-authorization` for gateway auth.
- `cf-aig-byok-alias` when using non-default BYOK aliases.

Example LiteLLM model list entries:

```yaml
model_list:
  - model_name: cf-crofai-kimi
    litellm_params:
      model: openai/custom-crofai/kimi-k2.6
      api_base: https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat
      api_key: os.environ/CF_AIG_TOKEN
      extra_headers:
        cf-aig-authorization: Bearer <CLOUDFLARE_AIG_RUNTIME_TOKEN>

  - model_name: cf-vertex-gemini-flash
    litellm_params:
      model: openai/google-vertex-ai/google/gemini-2.5-flash
      api_base: https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat
      api_key: os.environ/CF_AIG_TOKEN
      extra_headers:
        cf-aig-authorization: Bearer <CLOUDFLARE_AIG_RUNTIME_TOKEN>
        cf-aig-byok-alias: <VERTEX_BYOK_ALIAS>
```

Keep secrets out of the YAML. If your process manager can template headers from environment variables, use that. Otherwise keep this config private and out of source control.

## OpenClaw

OpenClaw has an official `cloudflare-ai-gateway` provider path for Anthropic through Cloudflare. For CrofAI or Vertex through Cloudflare `/compat`, use an OpenClaw custom provider with `api: "openai-completions"`.

Minimal config shape:

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
            name: "CrofAI Kimi K2.6",
            reasoning: true,
            input: ["text"],
            contextWindow: 262144,
            maxTokens: 262144
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
            maxTokens: 65536
          }
        ]
      }
    }
  },
  agents: {
    defaults: {
      model: {
        primary: "cloudflare-crofai/custom-crofai/kimi-k2.6"
      }
    }
  }
}
```

Verification commands:

```bash
openclaw config validate
openclaw models list --provider cloudflare-crofai
openclaw models list --provider cloudflare-vertex
```

Copy the exact provider/model reference shown by `openclaw models list`. Some catalogs normalize model display names or preview suffixes.

## OpenCode

OpenCode supports custom OpenAI-compatible providers with `@ai-sdk/openai-compatible`, `options.baseURL`, `options.apiKey`, and `options.headers`.

Example `opencode.jsonc` provider block:

```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "crofai-cloudflare": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "CrofAI via Cloudflare",
      "options": {
        "baseURL": "https://gateway.ai.cloudflare.com/v1/<ACCOUNT_ID>/<GATEWAY_ID>/compat",
        "apiKey": "{env:CF_AIG_TOKEN}",
        "headers": {
          "cf-aig-authorization": "Bearer {env:CF_AIG_TOKEN}"
        }
      },
      "models": {
        "custom-crofai/kimi-k2.6": {
          "name": "CrofAI Kimi K2.6",
          "limit": { "context": 262144, "output": 262144 }
        },
        "custom-crofai/glm-5.1": {
          "name": "CrofAI GLM 5.1",
          "limit": { "context": 202752, "output": 202752 }
        }
      }
    }
  }
}
```

If your OpenCode build does not expand environment placeholders inside `headers`, keep a private local config or use an intermediate gateway such as LiteLLM to inject Cloudflare headers. Do not commit a filled token.

## Troubleshooting

- `Invalid provider`: check the custom provider slug. Cloudflare custom providers are addressed as `custom-<slug>`.
- `CREDENTIALS_MISSING` or Google 401: check Vertex BYOK provider key, alias, region, and service account permissions.
- Vertex model not found: use `google-vertex-ai/google/<MODEL_ID>` through `/compat`.
- Empty final `content`: increase `max_tokens` and inspect reasoning fields.
- Client cannot send `cf-aig-authorization`: use a client header option, LiteLLM, a small Worker wrapper, or a private config that can inject the header.
- Local internet or power outage: Cloudflare keeps the provider gateway available, but the local agent process still stops unless it runs in cloud.

## Public References

- Cloudflare BYOK: https://developers.cloudflare.com/ai-gateway/configuration/bring-your-own-keys/
- Cloudflare authenticated gateway: https://developers.cloudflare.com/ai-gateway/configuration/authentication/
- Cloudflare custom providers: https://developers.cloudflare.com/ai-gateway/configuration/custom-providers/
- Cloudflare Vertex AI provider: https://developers.cloudflare.com/ai-gateway/usage/providers/vertex/
- LiteLLM proxy config: https://docs.litellm.ai/docs/proxy/configs
- OpenClaw Cloudflare AI Gateway provider: https://docs.openclaw.ai/providers/cloudflare-ai-gateway
- OpenClaw custom providers/config tools: https://docs.openclaw.ai/gateway/config-tools
- OpenClaw LiteLLM provider: https://docs.openclaw.ai/providers/litellm
- OpenCode custom providers: https://opencode.ai/docs/providers
