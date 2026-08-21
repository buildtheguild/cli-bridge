# Dify + CLIBridge

**Status: Experimental — do not advertise as verified yet**

Dify has an official **OpenAI-API-compatible** model provider and lets you configure a custom API Base URL.

The integration architecture is correct for CLIBridge, but current Dify provider validation/inference code can send token-control parameters such as `max_tokens` or `max_completion_tokens`.

CLIBridge does not currently document those request fields, so compatibility depends on how the running CLIBridge release handles them.

---

## Values to test

```text
API Base URL: https://your-domain.com/v1
API Key:      YOUR_BRIDGE_TOKEN
Model:        a model returned by /v1/models
Mode:         Chat
```

On a shared Docker network:

```text
API Base URL: http://cli-bridge:3900/v1
```

---

## Suggested Dify model configuration

In Dify, add/configure the **OpenAI-API-compatible** provider.

Use settings similar to:

```text
Model Name:             <CLIBridge model ID>
API Key:                YOUR_BRIDGE_TOKEN
API Base URL:           https://your-domain.com/v1
Completion mode:        Chat
Compatibility mode:     Strict / OpenAI compatible
Function Call Type:     Not Support
Stream Function Calling: Not Support
```

For the first test, leave advanced capabilities disabled.

Get the model ID from:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

---

## Important validation caveat

Dify may validate an OpenAI-compatible chat model by sending a request that includes a token limit such as:

```text
max_tokens
```

or:

```text
max_completion_tokens
```

Those fields are not currently listed in CLIBridge's supported Chat Completions request fields.

If Dify fails while saving/validating the model, inspect the returned error and CLIBridge logs.

Do not label Dify as compatible until this validation succeeds on a real deployment.

---

## Function calling

For this CLIBridge profile, configure Dify as if the model does not support function calling.

CLIBridge does not currently document:

```text
tools
tool_choice
```

---

## Embeddings

Do not configure CLIBridge as Dify's OpenAI-compatible embedding model.

CLIBridge does not currently expose:

```text
POST /v1/embeddings
```

Use another embedding provider for Dify knowledge/RAG.

---

## Vision

CLIBridge supports OpenAI-style image content parts, but Dify's exact multimodal request format should be validated separately.

For the first compatibility test, leave vision disabled and verify text chat first.

---

## Compatibility scope to verify

Before changing the status from Experimental, test:

1. Provider credential validation
2. Plain non-streaming chat
3. Streaming chat
4. Structured response format
5. Image input

Then document any Dify fields that must be disabled.

Do not include agents/function calling in the verified scope unless `tools` support is added and tested.
