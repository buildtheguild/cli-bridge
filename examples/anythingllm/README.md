# AnythingLLM + CLIBridge

**Status: Experimental — do not advertise as verified yet**

AnythingLLM includes a **Generic OpenAI** LLM provider that accepts a custom OpenAI-compatible Base URL.

The connection shape is correct for CLIBridge, but current AnythingLLM Generic OpenAI code sends:

```text
model
messages
temperature
max_tokens
```

for normal chat, with `stream`-related fields for streaming.

CLIBridge currently documents `model`, `messages`, and `stream`, but does not document `temperature` or `max_tokens`.

That makes native AnythingLLM chat an integration candidate rather than a confirmed integration for the current CLIBridge release.

---

## UI configuration to test

In AnythingLLM:

1. Open **Settings**.
2. Open the LLM/provider configuration.
3. Select **Generic OpenAI**.
4. Configure:

```text
Base URL: https://your-domain.com/v1
API Key:  YOUR_BRIDGE_TOKEN
Model:    a model returned by CLIBridge
```

On a shared Docker network:

```text
Base URL: http://cli-bridge:3900/v1
```

---

## Environment-variable configuration

AnythingLLM also supports configuration similar to:

```env
LLM_PROVIDER='generic-openai'

GENERIC_OPEN_AI_BASE_PATH='https://your-domain.com/v1'
GENERIC_OPEN_AI_MODEL_PREF='gpt-5.5'
GENERIC_OPEN_AI_MODEL_TOKEN_LIMIT='4096'
GENERIC_OPEN_AI_API_KEY='YOUR_BRIDGE_TOKEN'
```

Replace the model with one returned by:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

---

## Important request-body caveat

AnythingLLM currently sends `temperature` and `max_tokens` in its Generic OpenAI chat requests.

Because CLIBridge does not currently document those fields, the native integration must be tested.

If CLIBridge rejects the request because of unsupported fields, AnythingLLM should remain marked unsupported/experimental until CLIBridge accepts or safely ignores those common OpenAI parameters.

---

## RAG / embeddings

AnythingLLM normally uses an embedding provider for document retrieval.

Do not point AnythingLLM's Generic OpenAI embedding engine at CLIBridge because CLIBridge does not currently expose:

```text
POST /v1/embeddings
```

Use AnythingLLM's local embedder or another embedding provider.

---

## Speech-to-text

AnythingLLM supports OpenAI-compatible speech-to-text providers.

CLIBridge exposes:

```text
POST /v1/audio/transcriptions
```

through local `whisper.cpp` when the CLIBridge plan includes Whisper.

This can be tested separately from the LLM/chat integration.

---

## Compatibility scope to verify

Before changing this guide to “Verified”, test:

- provider connection
- non-streaming chat
- streaming chat
- whether `temperature` is accepted/ignored
- whether `max_tokens` is accepted/ignored
- optional STT through CLIBridge Whisper

Do not claim CLIBridge supplies AnythingLLM embeddings.
