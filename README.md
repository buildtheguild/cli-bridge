# CLIBridge

**Turn OpenAI Codex CLI or Anthropic Claude Code CLI into an OpenAI-compatible HTTP API.**

CLIBridge is a self-hosted Docker gateway that lets applications talk to the **Codex CLI** or **Claude Code CLI** through familiar OpenAI-style endpoints such as:

- `GET /v1/models`
- `POST /v1/chat/completions`
- `POST /v1/audio/transcriptions`

Use it with automation platforms, chat interfaces, SDKs, scripts, and custom applications that can point to a custom OpenAI-compatible Base URL.

CLIBridge also includes optional **100% local Whisper transcription** powered by `whisper.cpp`. Audio submitted to the Whisper endpoint is processed inside the container and is not sent to OpenAI, Anthropic, or another transcription provider.

> CLIBridge does not provide OpenAI or Anthropic API credits. It uses the CLI account authenticated inside your CLIBridge deployment. Provider terms, subscription limits, rate limits, and availability still apply.

**Docker Hub:** https://hub.docker.com/r/thebuildguild/cli-bridge  
**GitHub:** https://github.com/buildtheguild/cli-bridge  
**Endpoint reference:** [endpoints.md](./endpoints.md)  
**Integration examples:** [examples/](./examples/)

---

## Why CLIBridge?

Many applications already know how to talk to an OpenAI-compatible API.

CLIBridge gives those applications a simple HTTP interface in front of the Codex CLI or Claude Code CLI, so you can connect compatible software using a Base URL such as:

```text
https://your-domain.com/v1
```

or locally:

```text
http://localhost:3900/v1
```

Your application sends an OpenAI-style Chat Completions request, and CLIBridge routes it through the active CLI backend.

---

## Highlights

- OpenAI-compatible `POST /v1/chat/completions`
- OpenAI-compatible `GET /v1/models`
- JSON and SSE streaming responses
- System, developer, user, assistant, and tool message roles
- Image input through URL or data URI
- Multipart image-upload endpoint for testing
- Codex CLI backend
- Claude Code CLI backend
- Docker-first deployment
- Browser-based activation and provider authentication
- Dashboard with health, usage, logs, and CLI update status
- Configurable model fallback after switching backends
- Optional emergency OpenRouter fallback
- Rate limiting and operational safeguards
- Optional local Whisper transcription
- Downloadable/switchable Whisper models with persistent storage

---

## Client compatibility

CLIBridge is designed around the OpenAI **Chat Completions** protocol.

| Client / platform | Status | Notes |
|---|---|---|
| n8n | **Tested** | Use a custom OpenAI Base URL and disable the Responses API. |
| OpenAI Node.js SDK | **Direct API match** | Use `chat.completions.create()` with a custom `baseURL`. |
| OpenAI Python SDK | **Direct API match** | Use `chat.completions.create()` with a custom `base_url`. |
| Custom applications | **Direct API match** | Call `/v1/chat/completions` directly. |
| LibreChat | **Integration guide available** | Custom endpoints plus `dropParams` can keep requests inside CLIBridge's supported request surface. |
| Open WebUI | **Basic-chat candidate** | Uses OpenAI Chat Completions, but may send optional OpenAI parameters CLIBridge does not currently document. Test before relying on advanced features. |
| Flowise | **Experimental** | Native ChatOpenAI integration can send parameters such as `temperature`. |
| Langflow | **Experimental** | OpenAI Compatible provider is a good protocol fit, but model components may send additional parameters. |
| Dify | **Experimental** | Its OpenAI-compatible provider may send token-control parameters during validation and inference. |
| AnythingLLM | **Experimental** | Generic OpenAI currently sends `temperature` and `max_tokens`. |

See [examples/README.md](./examples/README.md) for setup guides and compatibility notes.

### What “OpenAI-compatible” means here

CLIBridge currently documents these request fields for `POST /v1/chat/completions`:

- `model`
- `messages`
- `stream`
- `response_format`

Image parts are also supported inside `messages[].content`.

CLIBridge does **not currently document** support for the full OpenAI API surface. In particular, do not assume support for:

- `/v1/responses`
- `/v1/embeddings`
- `tools`
- `tool_choice`
- `temperature`
- `top_p`
- `max_tokens`
- `max_completion_tokens`
- `frequency_penalty`
- `presence_penalty`
- `stop`
- image generation
- text-to-speech

A client may still be usable if it only sends the fields CLIBridge supports, or if it can remove unsupported parameters before sending the request.

---

## Docker tags

| Tag | CLI backends | Notes |
|---|---|---|
| `latest`, `2.2.2` | Codex + Claude | Includes both CLIs plus `whisper.cpp`; one CLI backend is active at a time. |
| `codex`, `2.2.2-codex` | Codex only | Smaller single-backend image. |
| `claude`, `2.2.2-claude` | Claude only | Smaller single-backend image. |

Use a **versioned tag** in production.

> The combined image contains both CLIs, but CLIBridge routes chat traffic through only one active backend at a time. Select it with `BACKEND=codex` or `BACKEND=claude`.

---

## Requirements

- Docker with Docker Compose
- A valid CLIBridge subscription/license
- An account that can authenticate the CLI backend you want to use
- For Codex: a supported ChatGPT/Codex CLI account
- For Claude: a supported Claude/Claude Code account
- HTTPS if the bridge will be exposed outside a trusted local network

---

# Quick start

## 1. Create `.env`

For Codex:

```env
BRIDGE_TOKEN=replace-with-a-long-random-secret
BACKEND=codex
```

For Claude:

```env
BRIDGE_TOKEN=replace-with-a-long-random-secret
BACKEND=claude
```

`BRIDGE_TOKEN` protects the HTTP API. Treat it like an API key.

---

## 2. Create `docker-compose.yml`

The combined image is convenient if you may switch between Codex and Claude later:

```yaml
services:
  cli-bridge:
    image: thebuildguild/cli-bridge:2.2.2
    container_name: cli-bridge
    ports:
      - "3900:3900"
    env_file:
      - .env
    environment:
      PORT: 3900
      BACKEND: ${BACKEND:-codex}
      CODEX_BIN: codex
      CLAUDE_BIN: claude
      API_DOCS_SERVERS: http://localhost:3900,https://your-domain.com
    volumes:
      - cli_data:/data
    restart: unless-stopped

volumes:
  cli_data:
```

If you only need one backend, use the `2.2.2-codex` or `2.2.2-claude` image instead.

---

## 3. Start CLIBridge

```bash
docker compose pull
docker compose up -d
```

Open:

```text
http://localhost:3900
```

---

## 4. Activate CLIBridge

Open the operator console and complete the activation flow.

The activation state is stored in the persistent `/data` volume.

Do not remove that volume unless you intentionally want to reset activation and stored CLI authentication.

---

## 5. Authenticate the CLI backend

The recommended method is the browser-based authentication flow in the operator console.

You can also authenticate from inside the container.

For Codex:

```bash
docker compose exec cli-bridge sh
codex login --device-auth
```

For Claude:

```bash
docker compose exec cli-bridge sh
claude auth login
```

---

## 6. Test model discovery

```bash
curl http://localhost:3900/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

You can also use:

```text
X-Bridge-Token: YOUR_BRIDGE_TOKEN
```

instead of the Bearer header.

---

## 7. Send a chat request

```bash
curl http://localhost:3900/v1/chat/completions \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.5",
    "messages": [
      {
        "role": "user",
        "content": "Hello from CLIBridge"
      }
    ]
  }'
```

Use a model returned by:

```text
GET /v1/models
```

If the requested model is not valid for the active backend and model fallback is enabled, CLIBridge can use the backend's default model instead.

---

# Use CLIBridge with other applications

Most integrations use these three values:

```text
Base URL: https://your-domain.com/v1
API key:  your BRIDGE_TOKEN
Model:    a model returned by GET /v1/models
```

Examples are included for:

- [n8n](./examples/n8n/)
- [Open WebUI](./examples/open-webui/)
- [LibreChat](./examples/librechat/)
- [Flowise](./examples/flowise/)
- [Langflow](./examples/langflow/)
- [Dify](./examples/dify/)
- [AnythingLLM](./examples/anythingllm/)
- [OpenAI Node.js and Python SDKs](./examples/openai-sdk/)
- [Custom applications](./examples/custom-app/)

For applications running in Docker, remember that:

```text
localhost
```

means **the current container**, not the Docker host.

If CLIBridge and the client are on the same Docker network, use the CLIBridge service/container name, for example:

```text
http://cli-bridge:3900/v1
```

---

# Streaming

Set:

```json
{
  "stream": true
}
```

CLIBridge returns OpenAI-style Server-Sent Events using `chat.completion.chunk` objects and ends the stream with:

```text
data: [DONE]
```

---

# Image input

CLIBridge supports image parts in `messages[].content`.

Example:

```bash
curl http://localhost:3900/v1/chat/completions \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-5.5",
    "messages": [
      {
        "role": "user",
        "content": [
          {
            "type": "text",
            "text": "Describe this image"
          },
          {
            "type": "image_url",
            "image_url": {
              "url": "https://example.com/image.jpg"
            }
          }
        ]
      }
    ]
  }'
```

Data URIs are also supported.

A separate multipart endpoint is available for testing a single uploaded image:

```text
POST /v1/chat/completions/upload
```

See [endpoints.md](./endpoints.md) for the exact request format and limits.

---

# Local Whisper transcription

CLIBridge can expose an OpenAI-compatible transcription endpoint backed by local `whisper.cpp`:

```text
POST /v1/audio/transcriptions
```

Example:

```bash
curl http://localhost:3900/v1/audio/transcriptions \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN" \
  -F "file=@voice-note.ogg"
```

Supported response formats include:

- `json`
- `text`
- `verbose_json`

Whisper is independent of the selected Codex/Claude backend.

It is also **plan-gated**. If your CLIBridge plan does not include Whisper, audio endpoints return `403`.

Check status:

```bash
curl http://localhost:3900/v1/audio/status \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

List locally available/downloadable models:

```bash
curl http://localhost:3900/v1/audio/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

Downloaded models and the active model selection are stored on the persistent `/data` volume.

---

# Switching between Codex and Claude

With the combined image, change:

```env
BACKEND=codex
```

to:

```env
BACKEND=claude
```

or vice versa, then recreate/restart the service as required by your deployment.

The active backend determines which CLI handles chat requests.

Whisper transcription is independent of this setting.

---

# Model fallback after switching backends

Automations sometimes keep an old model ID after you change the active backend.

For example, a workflow may continue sending a Claude model after switching to Codex.

CLIBridge can fall back to the active backend's default model:

```env
MODEL_FALLBACK_TO_DEFAULT_ON_UNKNOWN=true
```

This behavior is enabled by default.

Set:

```env
MODEL_FALLBACK_TO_DEFAULT_ON_UNKNOWN=false
```

for strict model validation.

When fallback occurs, non-streaming responses can include a `model_fallback` object so callers can see what happened.

---

# Emergency OpenRouter fallback

CLIBridge can optionally retry a user-facing chat request through OpenRouter when the active CLI backend fails because of a known emergency condition such as provider rate limiting, quota exhaustion, missing CLI authentication, timeout, or provider unavailability.

It is disabled by default.

Example:

```env
OPENROUTER_ENABLE_EMERGENCY_FALLBACK=true
OPENROUTER_API_KEY=your-openrouter-key
OPENROUTER_DEFAULT_MODEL=openai/gpt-5
OPENROUTER_FALLBACK_MODELS=anthropic/claude-sonnet-4.5,google/gemini-2.5-pro
OPENROUTER_TIMEOUT_MS=45000
```

This fallback uses **OpenRouter API credits**. It does not use your Codex or Claude subscription quota.

Leave it disabled unless you intentionally want a paid emergency path.

---

# Main HTTP endpoints

## OpenAI-compatible

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/models` | List available models |
| `POST` | `/v1/models/refresh` | Probe the CLI and refresh model information |
| `POST` | `/v1/chat/completions` | Chat completion, including SSE streaming |
| `POST` | `/v1/chat/completions/upload` | Chat completion with one multipart image upload |
| `POST` | `/v1/audio/transcriptions` | Local Whisper transcription, plan-gated |

## Diagnostics and operations

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/v1/health` | Basic health status |
| `GET` | `/v1/health?details=true` | Expanded health when enabled |
| `GET` | `/v1/metrics` | Request/token counters |
| `GET` | `/v1/logs` | Recent operator-relevant failures |
| `GET` | `/v1/watchdog/status` | Watchdog status |
| `GET` | `/v1/audio/status` | Whisper health and capacity |
| `GET` | `/v1/audio/models` | Whisper model catalog and state |

For the complete endpoint list and request/response schemas, see:

**[endpoints.md](./endpoints.md)**

A running instance can also expose Swagger UI at:

```text
http://localhost:3900/api-docs
```

when API docs are enabled.

---

# Important environment variables

| Variable | Purpose |
|---|---|
| `BRIDGE_TOKEN` | Secret used to authenticate API requests |
| `PORT` | HTTP listening port |
| `BACKEND` | `codex` or `claude` |
| `CODEX_DEFAULT_MODEL` | Default Codex model |
| `CODEX_ALLOWED_MODELS` | Optional allowed Codex model list |
| `CLAUDE_DEFAULT_MODEL` | Default Claude model |
| `CLAUDE_ALLOWED_MODELS` | Optional allowed Claude model list |
| `MODEL_FALLBACK_TO_DEFAULT_ON_UNKNOWN` | Fall back when a client sends an invalid/stale model |
| `ENABLE_API_DOCS` | Enable Swagger UI |
| `API_DOCS_SERVERS` | Server URLs displayed in Swagger |
| `CORS_ORIGINS` | Allowed browser origins |
| `HEALTH_VERBOSE_ENABLED` | Allow expanded health details |
| `OPENROUTER_ENABLE_EMERGENCY_FALLBACK` | Enable emergency OpenRouter retry path |
| `OPENROUTER_API_KEY` | OpenRouter API key |
| `OPENROUTER_DEFAULT_MODEL` | Primary emergency fallback model |
| `WHISPER_LANGUAGE` | Default Whisper language or `auto` |
| `WHISPER_THREADS` | CPU threads for transcription |
| `WHISPER_MAX_CONCURRENT` | Maximum simultaneous Whisper jobs |

See the existing configuration and endpoint documentation for additional advanced variables.

---

# Security recommendations

- Use a long random `BRIDGE_TOKEN`.
- Never commit `.env` or provider credentials.
- Put CLIBridge behind HTTPS before exposing it to the internet.
- Restrict access at the reverse proxy/firewall where practical.
- Keep the `/data` volume persistent and protected.
- Use versioned Docker image tags in production.
- Review logs and health status after CLI updates.
- Do not expose the operator console publicly without appropriate network controls.
- Treat OpenRouter fallback as a paid external fallback and keep it disabled unless required.

---

# Updating the bundled CLIs

The operator dashboard checks for newer Codex and Claude CLI versions.

CLI updates are intentionally manual.

An update performed inside the running container modifies that container's writable layer and may survive a normal restart, but it will be lost when the container is recreated from the image.

Use a newer CLIBridge image/build when you want an updated CLI version to become part of the deployment permanently.

---

# Troubleshooting

## `401` / authentication error

Check that your client sends:

```text
Authorization: Bearer YOUR_BRIDGE_TOKEN
```

or:

```text
X-Bridge-Token: YOUR_BRIDGE_TOKEN
```

## Client tries `/v1/responses`

CLIBridge currently exposes Chat Completions, not the OpenAI Responses API.

Configure the client to use:

```text
POST /v1/chat/completions
```

For n8n, disable **Use Responses API**.

## Client cannot load models

Test:

```bash
curl https://your-domain.com/v1/models \
  -H "Authorization: Bearer YOUR_BRIDGE_TOKEN"
```

If that works, verify the client is using the Base URL:

```text
https://your-domain.com/v1
```

and not:

```text
https://your-domain.com
```

when the client expects an OpenAI-style `/v1` base.

## Docker client cannot reach `localhost:3900`

Inside a container, `localhost` points to that same container.

Put both services on the same Docker network and use:

```text
http://cli-bridge:3900/v1
```

or use a reachable host/domain address.

## Client fails on `temperature`, `max_tokens`, `tools`, or another optional OpenAI field

The current CLIBridge Chat Completions documentation lists only:

```text
model
messages
stream
response_format
```

plus supported image content parts.

Some OpenAI-compatible clients send additional parameters automatically. Check the integration guide in [examples/](./examples/) and disable/drop unsupported parameters where the client allows it.

---

# Support and documentation

- [Integration examples](./examples/)
- [HTTP endpoint reference](./endpoints.md)
- [Changelog](./CHANGELOG.md)
- Docker Hub: https://hub.docker.com/r/thebuildguild/cli-bridge
- GitHub issues: https://github.com/buildtheguild/cli-bridge/issues

---

## Disclaimer

CLIBridge is an independent gateway project and is not an official OpenAI or Anthropic product.

OpenAI, ChatGPT, Codex, Anthropic, Claude, and Claude Code are trademarks or product names of their respective owners.

Your use of those services remains subject to their applicable terms, subscription rules, rate limits, and availability.
