# CLIBridge integration examples

This directory contains setup examples for applications that can connect to an OpenAI-compatible API.

The examples use a CLIBridge Base URL such as:

```text
https://your-domain.com/v1
```

For local use:

```text
http://localhost:3900/v1
```

For two containers on the same Docker network:

```text
http://cli-bridge:3900/v1
```

Use your CLIBridge `BRIDGE_TOKEN` as the API key/Bearer token.

---

## Compatibility status

| Integration | Status | Guide |
|---|---|---|
| n8n | **Tested** | [n8n](./n8n/) |
| OpenAI Node.js/Python SDKs | **Direct API match** | [openai-sdk](./openai-sdk/) |
| Custom applications | **Direct API match** | [custom-app](./custom-app/) |
| LibreChat | **Recommended configuration example** | [librechat](./librechat/) |
| Open WebUI | **Basic-chat candidate** | [open-webui](./open-webui/) |
| Flowise | **Experimental** | [flowise](./flowise/) |
| Langflow | **Experimental** | [langflow](./langflow/) |
| Dify | **Experimental** | [dify](./dify/) |
| AnythingLLM | **Experimental** | [anythingllm](./anythingllm/) |

“Experimental” does not mean the application cannot work. It means the client is known to support custom OpenAI-compatible endpoints but may send request fields that CLIBridge does not currently document.

---

## Current CLIBridge Chat Completions request surface

`POST /v1/chat/completions` currently documents:

- `model`
- `messages`
- `stream`
- `response_format`

Image content parts are also supported.

Do not assume the bridge supports every OpenAI option just because a client calls itself OpenAI-compatible.

In particular, CLIBridge does not currently document:

- `/v1/responses`
- `/v1/embeddings`
- `temperature`
- `top_p`
- `max_tokens`
- `max_completion_tokens`
- `tools`
- `tool_choice`
- `frequency_penalty`
- `presence_penalty`
- `stop`

For best results, configure clients to send only the supported fields.

---

## Find your model ID

Before configuring a client, check what CLIBridge currently advertises:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

Use one of the returned IDs in your client.

---

## A note about Docker networking

If the application and CLIBridge are separate containers, do not use:

```text
http://localhost:3900/v1
```

unless CLIBridge is actually running inside the same container.

When both containers share a Docker network, use the CLIBridge service name:

```text
http://cli-bridge:3900/v1
```

---

## Help improve these examples

If you test one of the integrations marked **Experimental**, please open an issue with:

- application name and version
- CLIBridge version
- backend (`codex` or `claude`)
- whether streaming was enabled
- sanitized request/error
- whether basic chat worked
- whether advanced features were attempted

That helps turn an experimental guide into a verified integration.
