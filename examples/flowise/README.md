# Flowise + CLIBridge

**Status: Experimental**

Flowise's OpenAI Chat Model supports a custom **Base Path**, so it can be pointed at an OpenAI-compatible server such as CLIBridge.

However, current Flowise ChatOpenAI components can send additional OpenAI parameters — including a default `temperature` value — that CLIBridge does not currently document.

For that reason, do not advertise this integration as verified until it has been tested against the exact CLIBridge and Flowise versions you plan to support.

---

## Connection values

```text
Base Path: https://your-domain.com/v1
API Key:   YOUR_BRIDGE_TOKEN
Model:     a model returned by /v1/models
```

On a shared Docker network:

```text
Base Path: http://cli-bridge:3900/v1
```

---

## Native ChatOpenAI setup to test

1. Create/select an OpenAI API credential in Flowise.
2. Use your CLIBridge `BRIDGE_TOKEN` as the API key.
3. Add an **OpenAI Chat Model / ChatOpenAI** component.
4. Set **Base Path** to:

```text
https://your-domain.com/v1
```

5. Select or enter a model returned by CLIBridge.
6. Enable or disable streaming as required.
7. Leave advanced OpenAI options empty wherever Flowise allows it.
8. If the Temperature field can be cleared in your Flowise version, clear it for the first test.
9. Do not use a Tool Agent for the first compatibility test.

---

## Suggested first flow

Use the smallest possible chat flow, for example:

```text
Prompt
  ↓
OpenAI Chat Model
  ↓
Output
```

The goal is to verify basic `/v1/chat/completions` traffic before adding agent behavior.

---

## Why this is marked Experimental

Flowise's ChatOpenAI component can send fields such as:

```text
temperature
max_tokens
top_p
frequency_penalty
presence_penalty
stop
```

CLIBridge does not currently document those fields.

Flowise Tool Agent workflows also rely on model function/tool calling, while CLIBridge does not currently document:

```text
tools
tool_choice
```

---

## If native ChatOpenAI fails

Check the error returned by CLIBridge.

If the failure is caused by an unsupported request parameter, treat the native Flowise integration as unsupported for that CLIBridge release.

A direct HTTP call to:

```text
POST /v1/chat/completions
```

using only CLIBridge-supported fields can still be used from custom workflow logic, but that is different from claiming full native ChatOpenAI compatibility.

---

## Compatibility scope to test

Test these separately:

1. Model selection
2. Basic non-streaming chat
3. Streaming chat
4. Image input
5. Structured response format

Do not mark these as supported without dedicated testing:

- Tool Agent
- function calling
- embeddings/RAG through CLIBridge
- Responses API
