---
title: "PoYo API with LiteLLM: Python SDK and Gateway"
sidebar_label: "PoYo"
description: Call GPT, Claude, Gemini, Grok and Kimi models on PoYo through LiteLLM with the Python SDK or Gateway. Configure your API key, model name, and optional API base.
---

# PoYo

[PoYo](https://poyo.ai) is an AI model API aggregator that serves GPT, Claude, Gemini, Grok and Kimi models behind one OpenAI-compatible chat completions API. Use the `poyo/` prefix with a model name from the PoYo model catalog.

<SinceVersion v="1.113.0" />

{/* TODO: replace TODO below with the BerriAI/litellm PR number that adds the poyo provider */}
Provider support was added in [PR #TODO](https://github.com/BerriAI/litellm/pull/TODO). Use this version or a later build that includes the provider.

## Overview

| Property | Details |
|----------|---------|
| Provider prefix | `poyo/` |
| API key environment variable | `POYO_API_KEY` |
| Default API base | `https://api.poyo.ai/v1` |
| Optional API base override | `POYO_API_BASE` or the `api_base` parameter |
| Upstream endpoint | `/chat/completions` |
| Provider documentation | [PoYo API manual](https://docs.poyo.ai/api-manual/overview) |

## Setup

Create an API key on the [PoYo API key page](https://poyo.ai/dashboard/api-key) and export it before you start the SDK or proxy:

```bash
export POYO_API_KEY="your-poyo-api-key"
```

Use `poyo/<model-name>` for a model in the PoYo catalog, for example `poyo/claude-sonnet-5-5` or `poyo/gpt-6.2-sol`. LiteLLM removes the `poyo/` prefix before it sends the model value to PoYo. LiteLLM ships prices and context windows for the PoYo models in its model cost map, so spend tracking works for them without extra configuration.

## LiteLLM Python SDK

```python keep-model-ids
from litellm import completion

response = completion(
    model="poyo/claude-sonnet-5-5",
    messages=[{"role": "user", "content": "Say hello in five words"}],
    max_tokens=2000,
)

print(response.choices[0].message.content)
```

For streaming, pass `stream=True` and iterate over the response chunks:

```python keep-model-ids
from litellm import completion

response = completion(
    model="poyo/claude-sonnet-5-5",
    messages=[{"role": "user", "content": "Say hello in five words"}],
    max_tokens=2000,
    stream=True,
)

for chunk in response:
    print(chunk.choices[0].delta.content or "", end="")
```

Reasoning models such as `poyo/gemini-4-neon` and `poyo/kimi-k3` count their reasoning tokens toward `max_tokens`. Leave enough room for both, or the response can end before any visible text.

## LiteLLM Gateway

Save this configuration as `config.yaml`:

```yaml keep-model-ids
model_list:
  - model_name: claude-sonnet-5-5
    litellm_params:
      model: poyo/claude-sonnet-5-5
      api_key: os.environ/POYO_API_KEY
```

Start the proxy with the API key exported in its environment:

```bash
litellm --config config.yaml
```

Send a request using the configured model alias:

```bash keep-model-ids
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-5-5",
    "messages": [{"role": "user", "content": "Say hello in five words"}],
    "max_tokens": 2000
  }'
```

The proxy returns an OpenAI-format chat completion:

```json
{
  "choices": [{"message": {"role": "assistant", "content": "Hello, nice to meet you!"}}],
  "usage": {
    "completion_tokens": 9,
    "prompt_tokens": 14,
    "total_tokens": 23
  }
}
```

If your proxy requires authentication, also send `Authorization: Bearer <your-litellm-key>`. This is your LiteLLM key, not your PoYo API key.

To expose every PoYo model in the cost map at once, set both `model_name` and `litellm_params.model` to `poyo/*` and send requests with the full `poyo/<model-name>` value.

LiteLLM also accepts [Anthropic Messages](/docs/anthropic_unified) and [Responses API](/docs/response_api) requests for PoYo models through its translation layer. These requests use PoYo's chat completions endpoint, not native PoYo Messages or Responses endpoints.
