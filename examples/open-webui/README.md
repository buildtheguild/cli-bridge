# Open WebUI + CLIBridge

**Status: Basic-chat candidate — not yet project-verified**

Open WebUI supports OpenAI-compatible connections and uses the Chat Completions protocol for normal chat.

CLIBridge provides the two main endpoints Open WebUI expects for basic model discovery and chat:

```text
GET  /v1/models
POST /v1/chat/completions
```

However, Open WebUI can also send optional OpenAI parameters such as temperature, token limits, and tool settings. Those fields are not currently documented by CLIBridge, so this integration should be considered **basic-chat / experimental until tested with the exact Open WebUI release you deploy**.

---

## Connection values

```text
URL:     https://your-domain.com/v1
API Key: YOUR_BRIDGE_TOKEN
```

On a shared Docker network:

```text
URL: http://cli-bridge:3900/v1
```

---

## Setup

1. Open the Open WebUI admin/settings area.
2. Go to **Connections**.
3. Add an **OpenAI-compatible** connection.
4. Set the URL to:

```text
https://your-domain.com/v1
```

5. Set the API key to your CLIBridge `BRIDGE_TOKEN`.
6. Save the connection.
7. Let Open WebUI discover models through `/v1/models`.

If model discovery does not work, manually add a model ID returned by:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

---

## Recommended first test

Start with:

- plain text chat
- no tools
- no RAG through CLIBridge
- no image generation
- no TTS
- no OpenAI Responses API features

If Open WebUI lets you disable optional sampling parameters for the model/connection, do so for the first compatibility test.

---

## RAG / embeddings

Open WebUI can use an OpenAI-compatible `/v1/embeddings` endpoint for retrieval.

CLIBridge does **not currently expose**:

```text
POST /v1/embeddings
```

Use a separate embedding provider for Open WebUI RAG.

---

## Tools / function calling

Do not enable native OpenAI tool calling for the CLIBridge model unless you have tested it.

CLIBridge does not currently document:

```text
tools
tool_choice
```

as supported request fields.

---

## Speech-to-text

CLIBridge has a separate OpenAI-compatible transcription endpoint:

```text
POST /v1/audio/transcriptions
```

backed by local `whisper.cpp`.

If your CLIBridge plan includes Whisper, you can separately experiment with using it as Open WebUI's OpenAI-compatible STT provider.

This is independent of the basic chat setup above.

---

## Troubleshooting

### Connection verifies but chat fails

Open WebUI may be sending optional OpenAI parameters that the current CLIBridge version does not document.

Check CLIBridge:

```text
GET /v1/logs
```

and your container logs.

### Models do not appear

Verify:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

### RAG fails

Configure a separate embedding provider. CLIBridge does not currently provide `/v1/embeddings`.

### Tools fail

Disable native function/tool calling for the CLIBridge model.

---

## Compatibility scope

The intended compatibility target for this guide is:

```text
model discovery + plain Chat Completions + streaming
```

Advanced Open WebUI features may require API endpoints or request fields outside CLIBridge's current compatibility surface.
