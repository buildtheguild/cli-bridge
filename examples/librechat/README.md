# LibreChat + CLIBridge

**Status: Recommended configuration example — not yet project-verified**

LibreChat supports custom OpenAI-compatible endpoints and, importantly for CLIBridge, can drop request parameters before forwarding them.

That makes LibreChat a strong candidate for CLIBridge because the endpoint can be configured to stay close to CLIBridge's currently documented Chat Completions request surface.

---

## Values you need

```text
Base URL: https://your-domain.com/v1
API key:  YOUR_BRIDGE_TOKEN
```

On a shared Docker network:

```text
Base URL: http://cli-bridge:3900/v1
```

---

## 1. Add the token to LibreChat `.env`

```env
CLIBRIDGE_API_KEY=YOUR_BRIDGE_TOKEN
```

Do not commit the real token.

---

## 2. Add CLIBridge to `librechat.yaml`

```yaml
version: 1.3.13

endpoints:
  custom:
    - name: "CLIBridge"
      apiKey: "${CLIBRIDGE_API_KEY}"
      baseURL: "https://your-domain.com/v1"

      models:
        default:
          - "gpt-5.5"
        fetch: true

      modelDisplayLabel: "CLIBridge"

      titleConvo: false

      dropParams:
        - "user"
        - "temperature"
        - "top_p"
        - "max_tokens"
        - "max_completion_tokens"
        - "frequency_penalty"
        - "presence_penalty"
        - "stop"
        - "seed"
        - "tools"
        - "tool_choice"
        - "parallel_tool_calls"
```

Replace the example model with a model currently returned by:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

`models.fetch: true` lets LibreChat use CLIBridge's `/v1/models` endpoint for discovery.

---

## 3. Docker users: mount `librechat.yaml`

Make sure the LibreChat API container can read your custom configuration.

A typical override contains a bind mount similar to:

```yaml
services:
  api:
    volumes:
      - type: bind
        source: ./librechat.yaml
        target: /app/librechat.yaml
```

Restart LibreChat after changing the configuration.

---

## 4. Select CLIBridge

Open LibreChat and select:

```text
CLIBridge
```

from the endpoint/provider selector.

Choose a model and send a basic text prompt.

---

## Why `dropParams` is included

CLIBridge currently documents these Chat Completions request fields:

```text
model
messages
stream
response_format
```

LibreChat supports a broader OpenAI parameter set.

The `dropParams` list prevents common unsupported fields from being forwarded to CLIBridge.

This example also drops:

```text
tools
tool_choice
parallel_tool_calls
```

because CLIBridge does not currently document native OpenAI tool calling.

---

## Image input

CLIBridge supports OpenAI-style image content parts.

Image behavior through LibreChat still depends on how the selected LibreChat flow formats the request, so validate image input separately after plain text chat is working.

---

## Agents and tools

This configuration is for **normal chat**.

Do not assume LibreChat Agents, MCP-driven model tool calling, or OpenAI function calling will work through CLIBridge.

Those features can require `tools` / `tool_choice`, which this example intentionally drops.

---

## Troubleshooting

### `Unknown model`

Check:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

and update `models.default`.

CLIBridge can also fall back to the active backend's default model when model fallback is enabled.

### LibreChat sends a parameter CLIBridge rejects

Add that parameter to:

```yaml
dropParams:
```

provided the parameter is not essential to the feature you are trying to use.

### LibreChat cannot reach a local CLIBridge container

Put both containers on the same Docker network and use:

```text
http://cli-bridge:3900/v1
```

---

## Compatibility scope

This example is designed for:

- model discovery
- normal text chat
- streaming
- the subset of structured response formatting supported by CLIBridge

It intentionally avoids:

- Responses API
- embeddings
- native model tool calling
- OpenAI image generation
