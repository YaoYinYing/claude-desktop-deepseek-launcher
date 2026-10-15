# Running Claude Desktop 3P Through A Self-Hosted Gateway

Notes from wiring Claude Desktop Cowork 3P to a self-hosted Anthropic-compatible
gateway (a relay/proxy in front of DeepSeek or another provider) instead of
`api.deepseek.com` directly.

The gateway URL is written as `https://your-gateway.example.com` throughout. Use
your own endpoint.

## What Claude Desktop Actually Validates

Two independent checks decide whether a gateway profile loads. Both are client-side
name checks — the app never asks your gateway whether a model exists.

**1. `inferenceModels` names must look like Anthropic routes.**

Claude Desktop validates every entry against an allowlist of Anthropic model
families plus a denylist of other vendors. From the app bundle
(`ion-dist/assets/v1/c6f6cf4c2-*.js`, functions `or()` / `fr()`):

```js
// rejects anything matching this first
ar = /ark-code|astron|command-r|deepseek|doubao|gemini|gemma|glm|gpt|grok|hermes|
      hy3|kimi|lfm|\bling\b|llama|longcat|mimo|minimax|mistral|mixtral|moonshot|
      nemotron|openai|phi-|qianfan|qwen|tc-code|\bunic\b|yi-|stepfun|step-3|seed-|
      bytedance|hunyuan|granite|amazon\.nova|nova-|devstral|ministral|ernie|codex|
      arcee|trinity|abab|phi\d|\bk2\.|\bm2\.|jamba|arctic|solar|mercury|zamba|
      kat-coder|\bds-|dpsk/

// then accepts ^(haiku|opus|sonnet|fable)(-[\d.]+)?$ or any name containing
// "claude", "anthropic", or a family word
```

An invalid entry does not fail silently. It is reported as a blocking config error:

```text
"deepseek-flash" is not an Anthropic model for this provider —
  expected a gateway model route referencing an Anthropic model
  (e.g. claude-sonnet-4-5, anthropic/claude-*). Name routes to match the underlying model.
```

Consequences:

- `deepseek-flash`, `deepseek`, `glm-flash`, `Qwen_claude` are all rejected as
  `inferenceModels` names, even though a gateway happily serves them.
- `claude-sonnet-5`, `claude-opus-5`, `claude-fable-5-1` pass.
- This is why `--model-map` uses a hidden Claude route as `name` and puts your
  display text in `labelOverride`: `name` is what Claude Desktop stores and sends,
  `labelOverride` is only the picker label.

**2. The route name also gates Auto mode.**

"Automatically approve" (Auto mode) is disabled per model, and the gate keys off the
route string you put in `inferenceModels`. From the app bundle
(`ion-dist/assets/v1/shared-6-*.js`, functions `Oj()` / `kj()`):

```js
// Oj: "Requires Claude 4.6 or newer"
    rejected if the id contains "claude-3-" or matches
    /claude-[a-z]+-4(?:$|-(?:[0-5](?!\d)|\d{8}))/

// kj: the custom-3p gate, applied on top
    rejected if /opus-4-6(?!\d)/, /sonnet-4-6(?!\d)/,
    or /haiku(?!-5-5(?!\d))(?=-|$)/
```

So the picker ends up like this:

| route name in `inferenceModels` | accepted by the app | Auto mode |
| --- | --- | --- |
| `claude-sonnet-4-5` | yes | **greyed out** |
| `claude-opus-4-6` | yes | **greyed out** |
| `claude-3-5-haiku-20241022` | yes | **greyed out** |
| `claude-sonnet-5` | yes | offered |
| `claude-opus-5` | yes | offered |

A `claude-sonnet-4-5` route looks fine and works for chat, but its Auto mode row
reads "Not available for the selected model" / "Requires Claude 4.6 or newer" — the
name is one version too old for the gate. The launcher now defaults to
`claude-sonnet-5` and `claude-opus-5` for exactly this reason.

Auto mode also needs a model that supports the safety classifier. The app runs a
classifier call on each action; a gateway model only qualifies if the route you name
resolves to something the deployment's provider supports. If your gateway maps the
Anthropic route to a non-Claude model, expect Auto mode to be offered but behave as
"Cowork sessions still start by asking before each action."

## Direct Mode Against A Gateway

`--direct` points Claude Desktop straight at your gateway, with no local proxy:

```bash
python3 launch_claude_desktop_deepseek.py \
  --direct \
  --api-key "sk-..." \
  --base-url "https://your-gateway.example.com" \
  --model-map "Claude Mythos Flash=deepseek-flash" \
  --model-map "Claude Mythos=deepseek-flash" \
  --auth-scheme 'x-api-key' \
  --yes
```

This writes:

```json
"inferenceGatewayBaseUrl": "https://your-gateway.example.com",
"inferenceGatewayAuthScheme": "x-api-key",
"inferenceModels": [
  {"name": "claude-sonnet-5", "labelOverride": "Claude Mythos Flash"},
  {"name": "claude-opus-5",  "labelOverride": "Claude Mythos"}
]
```

The picker shows "Claude Mythos Flash" and "Claude Mythos"; the body carries
`"model": "claude-sonnet-5"`.

Direct mode only works if the gateway itself resolves the Anthropic route to the
upstream model. Either the gateway has a model-alias table (map every Anthropic model
name to your target), or discovery has to return the aliases. A gateway that 404s on
`claude-sonnet-4-5` will fail the same request through the app:

```json
{"error":{"message":"Model \"claude-sonnet-4-5\" is not supported by any configured
 account in this group","type":"model_not_found"}}
```

Mixing the two is a trap: an `--model-map` upstream value that the gateway has never
heard of (for example an upstream name only the local-proxy rewrite path knew) fails
in direct mode. In direct mode the launcher has no rewrite layer — the route name is
the request.

When to drop `--direct` and use the local proxy:

- the gateway rejects Claude Desktop's `user_id` / `userid` values,
- the gateway does not answer `/v1/models` or `/v1/messages/count_tokens`,
- you need to remap routes per request rather than by gateway-side alias,
- authentication needs to be reshaped (Claude Desktop sends what you configure;
  the proxy normalizes to `Authorization: Bearer`).

## Checklist For A New Gateway

1. `curl -sS https://your-gateway.example.com/v1/models -H "x-api-key: sk-..."` —
   note whether the returned ids are Anthropic-shaped.
2. `curl -sS -X POST https://your-gateway.example.com/v1/messages -H 'x-api-key: sk-...' -H 'content-type: application/json' -d '{"model":"claude-sonnet-5","max_tokens":16,"messages":[{"role":"user","content":"hi"}]}'`
   — confirm the route resolves upstream. Repeat for each route you plan to expose.
3. Pick route names the app accepts (`claude-sonnet-5`, `claude-opus-5`, not
   `deepseek-flash`), and put your own labels in `--model-map`'s left side.
4. Decide `--auth-scheme` from what the gateway expects: `x-api-key` for an
   Anthropic-style key header, `bearer` for `Authorization: Bearer`.
5. Run with `--no-launch` first, then inspect the profile before restarting the app.

## Inspecting The Result

```bash
python3 -c "import json;d=json.load(open('$HOME/Library/Application Support/Claude-3p/configLibrary/00000000-0000-4000-8000-00000000d335.json'));print(json.dumps(d['inferenceModels'],indent=1))"
```

Confirm `inferenceGatewayBaseUrl`, `inferenceGatewayAuthScheme`, and every
`inferenceModels[].name` before blaming the gateway. A profile that reads correctly
is not proof the gateway accepts the route — step 2 above is.

Restore normal Claude Desktop at any time:

```bash
python3 launch_claude_desktop_deepseek.py --restore --yes
```
