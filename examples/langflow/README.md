# Langflow + CLIBridge

**Status: Experimental**

Langflow 1.11 introduced an **OpenAI Compatible** model provider that can point to a custom OpenAI-compatible endpoint and discover models through `/v1/models`.

That is a strong protocol match for CLIBridge.

The remaining compatibility question is the request body: Langflow model components may send optional OpenAI parameters such as temperature and token limits, while CLIBridge currently documents a smaller Chat Completions request surface.

---

## Requirements

Use a Langflow release that includes the **OpenAI Compatible** provider.

You need:

```text
Base URL: https://your-domain.com/v1
API Key:  YOUR_BRIDGE_TOKEN
```

On a shared Docker network:

```text
Base URL: http://cli-bridge:3900/v1
```

---

## Setup to test

1. Open Langflow.
2. Open the model/provider configuration area.
3. Add an **OpenAI Compatible** provider.
4. Enter:

```text
API Base URL: https://your-domain.com/v1
API Key:      YOUR_BRIDGE_TOKEN
```

5. Let Langflow discover available models through:

```text
GET /v1/models
```

6. Select one of the discovered CLIBridge models.
7. Use the configured model in a simple prompt/chat flow.
8. Avoid Agent/tool features for the first test.
9. Avoid using CLIBridge as an embedding provider.

---

## Suggested first flow

Start with the simplest possible model invocation:

```text
Chat Input
    ↓
Language Model
    ↓
Chat Output
```

Use plain text only for the first test.

---

## Embeddings

Langflow can configure OpenAI-compatible embedding endpoints.

CLIBridge does **not currently expose**:

```text
POST /v1/embeddings
```

Use another embedding provider for vector search/RAG.

---

## Agents and tools

Do not assume Langflow Agent components will work through CLIBridge.

Agent components can depend on model-native function/tool calling, while CLIBridge does not currently document:

```text
tools
tool_choice
```

---

## Why this is marked Experimental

A model provider can be OpenAI-compatible at the URL level but still send request fields outside CLIBridge's current compatibility surface.

If Langflow sends:

```text
temperature
max_tokens
seed
```

or other unsupported options and CLIBridge rejects them, basic native integration will require either:

- a CLIBridge update that accepts/ignores those fields, or
- a Langflow configuration that prevents them from being sent.

---

## Compatibility scope to verify

Before changing this guide to “Verified”, test:

- `/v1/models` discovery
- non-streaming chat
- streaming chat
- image input
- structured output

Test agents/tool calling separately.
