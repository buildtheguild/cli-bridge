# n8n + CLIBridge

**Status: Tested**

CLIBridge can be used with n8n through n8n's OpenAI credential and **OpenAI Chat Model** node by pointing it to a custom OpenAI-compatible Base URL.

The most important setting on recent n8n versions is:

> **Use Responses API: OFF**

CLIBridge currently exposes `/v1/chat/completions`; it does not expose `/v1/responses`.

---

## Values you need

```text
Base URL: https://your-domain.com/v1
API key:  YOUR_BRIDGE_TOKEN
Model:    a model returned by CLIBridge /v1/models
```

If n8n and CLIBridge are containers on the same Docker network:

```text
Base URL: http://cli-bridge:3900/v1
```

---

## 1. Create the OpenAI credential

In n8n:

1. Open **Credentials**.
2. Create an **OpenAI API** credential.
3. Set the API key to your CLIBridge `BRIDGE_TOKEN`.
4. Set the Base URL to:

```text
https://your-domain.com/v1
```

Save the credential.

---

## 2. Add an OpenAI Chat Model

Add an **OpenAI Chat Model** sub-node to your AI workflow.

Select the CLIBridge OpenAI credential.

Choose a model from the list or enter a model ID returned by:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

---

## 3. Disable the Responses API

On n8n OpenAI Chat Model versions that show the option:

```text
Use Responses API = OFF
```

If it is enabled, n8n may try to use:

```text
/v1/responses
```

which CLIBridge does not currently expose.

Do not enable OpenAI built-in tools that require the Responses API.

---

## 4. Start with a simple workflow

For the first test, use a basic text-generation flow and avoid optional OpenAI parameters.

For example:

```text
Chat Trigger
    ↓
Basic LLM Chain
    ↓
OpenAI Chat Model (CLIBridge)
```

or another simple n8n chain that uses the OpenAI Chat Model.

Once basic chat is working, add your surrounding n8n workflow logic.

---

## Recommended initial settings

```text
Base URL:          https://your-domain.com/v1
Use Responses API: OFF
Streaming:         default/optional
Response Format:   Text
Extra Body:        empty
```

Do not add optional parameters unless you know the current CLIBridge version supports them.

---

## Using n8n AI Agent

CLIBridge can provide the language-model connection for n8n, but do not assume OpenAI-native tool/function calling is available.

The current CLIBridge documentation does not list:

```text
tools
tool_choice
```

as supported Chat Completions request fields.

If your n8n Agent depends on model-native tool calling, test it separately before using it in production.

---

## Concurrent executions

If a trigger can fire several executions at once — a webhook receiving a burst of messages,
or a Split In Batches node — they all reach CLIBridge simultaneously.

How many run at once is governed by your plan's request-concurrency limit. Requests beyond
it **wait for a free slot rather than failing**, so a burst drains at the licensed rate
instead of erroring.

You only receive `429 Too Many Requests` if no slot frees within the server's queue wait, or
the queue is full. Those responses include a `Retry-After` header.

Two settings matter on the n8n side:

- **Do not set an aggressive retry** on the HTTP/OpenAI node. Requests already queue
  server-side, so retrying a slow-but-pending burst only adds more work. If you do retry,
  honour `Retry-After` and add jitter so parallel executions do not retry in lockstep.
- **Raise the node timeout** above the expected queue wait. A queued request can legitimately
  sit for a while before it starts, and a short client timeout will cancel it while it is
  still waiting its turn.

To see whether the limit is your bottleneck, call `GET /v1/metrics`:

```json
{ "activeSessions": 10, "queuedRequests": 4, "sessionLimit": 10 }
```

Sustained `queuedRequests > 0` means executions are waiting on the concurrency limit.

---

## Troubleshooting

### n8n shows a `/responses` error

Disable:

```text
Use Responses API
```

and retry.

### Models do not appear

Test CLIBridge directly:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

If this succeeds, check the Base URL in the n8n credential.

### `401 Unauthorized`

Verify the n8n API key exactly matches:

```env
BRIDGE_TOKEN=...
```

### n8n and CLIBridge are both in Docker

Do not use `localhost` unless both processes are in the same container.

Use a shared Docker network and a URL such as:

```text
http://cli-bridge:3900/v1
```

### A workflow works until tools are attached

The model may be receiving OpenAI `tools` / `tool_choice` fields, which CLIBridge does not currently document.

Test with a basic chain first.

---

## Compatibility scope

This guide covers standard **Chat Completions** usage through n8n's OpenAI Chat Model.

It does not claim support for:

- OpenAI Responses API
- OpenAI built-in tools
- OpenAI embeddings
- OpenAI image generation
- model-native function/tool calling
