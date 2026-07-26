# CLIBridge

**HTTP gateway that turns the OpenAI Codex CLI or Anthropic Claude Code CLI into a standard OpenAI-compatible API — with a 100% local, self-hosted Whisper audio-transcription endpoint built in.**

Use your own **ChatGPT Plus** or **Claude Pro** subscription. CLIBridge handles authentication, streaming, image inputs, audio transcription, rate limiting, health monitoring, and SaaS-based activation.

Audio sent to `POST /v1/audio/transcriptions` is transcribed entirely inside the container by a bundled `whisper.cpp` — it is **never** sent to OpenAI, Anthropic, or any other external service.

**Docker Hub:** https://hub.docker.com/r/thebuildguild/cli-bridge

**Source / issues:** https://github.com/buildtheguild/cli-bridge

**HTTP endpoints:** [here](#http-endpoints)

## Tags

| Tag                      | CLI backends   | Notes                                          |
| ------------------------ | -------------- | ----------------------------------------------- |
| `latest`, `2.1.1`        | Codex + Claude | Includes whisper.cpp + baked-in Whisper model   |
| `claude`, `2.1.1-claude` | Claude only    | Includes whisper.cpp + baked-in Whisper model   |
| `codex`, `2.1.1-codex`   | Codex only     | Includes whisper.cpp + baked-in Whisper model   |

As of `2.0.0`, every tag also bundles `whisper.cpp` and a baked-in Whisper model (`base` by default). The `claude` variant measures ~340MB; `codex` and the combined `latest`/`full` image are in a similar range, a bit larger for `full` since it installs both CLIs. Whisper adds roughly the size of the baked-in model on top of that (~150MB for `base`) — see [Audio transcription](#7-audio-transcription-optional-plan-gated) below to change the baked-in model size, or swap models at runtime without rebuilding.

Use a versioned tag in production.

> **Full image and `BACKEND`:** The `latest` / full image has both CLIs pre-installed but still routes all traffic through a single backend selected at runtime via `BACKEND=codex` or `BACKEND=claude`. There is no simultaneous dual-backend routing. The full image is a build-time convenience so you can switch providers without rebuilding the image.

## Requirements

* Docker
* A valid CLIBridge subscription
* A CLI account for the backend you want to use

## Quick start

### 1. Create `.env`

Pick the backend you want to use.

**Codex:**

```dotenv
BRIDGE_TOKEN=your-long-random-secret-here
BACKEND=codex
```

**Claude:**

```dotenv
BRIDGE_TOKEN=your-long-random-secret-here
BACKEND=claude
```

### 2. Create `docker-compose.yml`

If your `.env` was copied from a Windows host, it may contain a Windows binary path, for example:

```dotenv
CODEX_BIN=C:\Users\you\AppData\Roaming\npm\codex.cmd
```

The `environment:` block below overrides that value with the in-container binary name.

Values defined under `environment:` take precedence over values from `env_file`, so the rest of your `.env` remains unaffected.

#### Codex

```yaml
services:
  cli-bridge:
    image: thebuildguild/cli-bridge:2.1.1-codex
    ports:
      - "3900:3900"
    env_file:
      - .env
    environment:
      PORT: 3900
      BACKEND: codex
      CODEX_BIN: codex
      API_DOCS_SERVERS: http://localhost:3900,https://your.domain.com
    volumes:
      - cli_data:/data
    restart: unless-stopped

volumes:
  cli_data:
```

#### Claude

```yaml
services:
  cli-bridge:
    image: thebuildguild/cli-bridge:2.1.1-claude
    ports:
      - "3900:3900"
    env_file:
      - .env
    environment:
      PORT: 3900
      BACKEND: claude
      CLAUDE_BIN: claude
      API_DOCS_SERVERS: http://localhost:3900,https://your.domain.com
    volumes:
      - cli_data:/data
    restart: unless-stopped

volumes:
  cli_data:
```

#### Combined image

Both CLIs are installed, but only one backend is active at a time.

```yaml
services:
  cli-bridge:
    image: thebuildguild/cli-bridge:2.1.1
    ports:
      - "3900:3900"
    env_file:
      - .env
    environment:
      PORT: 3900
      BACKEND: codex
      CODEX_BIN: codex
      CLAUDE_BIN: claude
      API_DOCS_SERVERS: http://localhost:3900,https://your.domain.com
    volumes:
      - cli_data:/data
    restart: unless-stopped

volumes:
  cli_data:
```

Set `BACKEND` to either:

```yaml
BACKEND: codex
```

or:

```yaml
BACKEND: claude
```

You can switch the active provider without rebuilding the image.

### 3. Start

```bash
docker compose pull
docker compose up -d
```

### 4. Activate

Open:

```text
http://localhost:3900
```

Click **Activate**, then approve the device in your browser.

### 5. Authenticate

Authentication is required once for the selected CLI provider.

#### Recommended: web wizard

Open:

```text
http://localhost:3900
```

Go to the provider step and click **Authenticate**.

Complete the login in the browser tab that opens. The dashboard will automatically detect when authentication is complete.

No terminal access is required.

#### Alternative: CLI authentication

For headless or scripted environments, open a shell inside the container.

```bash
docker compose exec cli-bridge sh
```

For Claude:

```bash
claude auth login
```

For Codex:

```bash
codex login --device-auth
```

### 6. Keeping the CLI up to date

The dashboard checks the npm registry for newer Codex and Claude CLI releases.

When a newer version is available, the provider card displays an **Update available** banner and an **Update** button.

Clicking the button runs the appropriate npm installation command inside the running container:

```bash
npm install -g <package>@latest
```

Updates are intentionally manual. CLIBridge does not automatically update the installed CLI.

The update modifies only the running container's writable layer.

The updated CLI version remains available after:

```bash
docker compose restart
```

or:

```bash
docker compose up -d
```

However, the update will be lost after operations that recreate or replace the container, including:

```bash
docker compose up -d --force-recreate
```

Pulling a newer Docker image or deploying on a new server will also restore the CLI version baked into the image.

To make a newer CLI version permanent, rebuild and republish the image using an updated build argument:

```text
OPENAI_CODEX_VERSION
```

or:

```text
CLAUDE_CODE_VERSION
```

### 7. Audio transcription (optional, plan-gated)

`POST /v1/audio/transcriptions` doesn't depend on your Codex/Claude subscription at all — it's backed entirely by the `whisper.cpp` binary and model baked into the image, and nothing is sent to OpenAI, Anthropic, or any other external service.

It does require your CLIBridge plan to include Whisper. If it doesn't, both this endpoint and `GET /v1/audio/status` return `403`, and the dashboard hides the Whisper status card entirely rather than showing it locked.

```bash
curl http://localhost:3900/v1/audio/transcriptions \
  -H "Authorization: Bearer $BRIDGE_TOKEN" \
  -F "file=@voice-note.ogg"
```

Response formats follow OpenAI's Whisper endpoint shape — `json` (default), `text`, or `verbose_json` with per-segment timestamps:

```bash
curl http://localhost:3900/v1/audio/transcriptions \
  -H "Authorization: Bearer $BRIDGE_TOKEN" \
  -F "file=@voice-note.ogg" \
  -F "response_format=verbose_json" \
  -F "language=en"
```

Check readiness anytime with `GET /v1/audio/status` — binary/model/ffmpeg health, live capacity, and usage metrics, independent of the `HEALTH_VERBOSE_ENABLED` flag used by `/v1/health`:

```bash
curl http://localhost:3900/v1/audio/status \
  -H "Authorization: Bearer $BRIDGE_TOKEN"
```

If a transcription hangs or is taking too long, cancel it and free the capacity slot immediately instead of waiting out the timeout:

```bash
curl -X POST http://localhost:3900/v1/audio/cancel \
  -H "Authorization: Bearer $BRIDGE_TOKEN"
```

#### Switching models (no rebuild needed)

As of `2.1.0`, models can be downloaded and switched entirely from the dashboard or API — no rebuild, no manual file handling:

```bash
# List the catalog (tiny/base/small/medium/large-v3-turbo/large-v3) with size and downloaded/active state
curl http://localhost:3900/v1/audio/models \
  -H "Authorization: Bearer $BRIDGE_TOKEN"

# Switch — downloads to the persistent /data volume automatically if not already present,
# then activates once done. Returns immediately; poll GET /v1/audio/models for progress.
curl -X POST http://localhost:3900/v1/audio/models/large-v3-turbo/select \
  -H "Authorization: Bearer $BRIDGE_TOKEN"

# Cancel an in-progress download, or delete a downloaded model to free disk space
curl -X POST http://localhost:3900/v1/audio/models/cancel -H "Authorization: Bearer $BRIDGE_TOKEN"
curl -X DELETE http://localhost:3900/v1/audio/models/tiny -H "Authorization: Bearer $BRIDGE_TOKEN"
```

Downloaded models and the active selection live on the `/data` volume, so they survive container recreation and image updates. The dashboard's Whisper card exposes all of this as a dropdown with a live download progress bar and a confirmation prompt before deleting.

To change which model ships **baked into the image by default** (rather than downloaded afterward), rebuild with `--build-arg WHISPER_MODEL=small` (or `tiny`/`medium`/`large-v3`/`large-v3-turbo`) — bigger models are more accurate but need more RAM/CPU per request, so pick to match your host.

#### Per-request model override

Pass `model` on a transcription request to use a different *already-downloaded* model for just that one request, without changing the dashboard's default:

```bash
curl http://localhost:3900/v1/audio/transcriptions \
  -H "Authorization: Bearer $BRIDGE_TOKEN" \
  -F "file=@voice-note.ogg" \
  -F "model=large-v3-turbo"
```

An undownloaded or unknown model name returns a clear `400` rather than silently falling back or triggering a multi-minute download mid-request. Omitting `model` (or sending OpenAI's fixed `"whisper-1"` placeholder) uses whichever model is currently active.

#### Diagnosing failures

`GET /v1/logs` (and the dashboard's Logs tab) shows recent transcription/download failures with the actual error message — e.g. the exact `whisper-cli timed out after ...` text — instead of just a bare failure count:

```bash
curl http://localhost:3900/v1/logs \
  -H "Authorization: Bearer $BRIDGE_TOKEN"
```

## Configuration

Configuration values can be set in `.env`, loaded through `env_file`, or placed directly under `environment:` in `docker-compose.yml`.

Values under `environment:` take precedence when the same variable is also present in `.env`.

### Required

#### `BRIDGE_TOKEN`

Shared secret used to authenticate HTTP API requests.

```dotenv
BRIDGE_TOKEN=your-long-random-secret-here
```

Use a long, random, private value.

### Server

#### `PORT`

Port on which CLIBridge listens.

Default:

```dotenv
PORT=3000
```

The examples in this document use:

```dotenv
PORT=3900
```

#### `CORS_ORIGINS`

Comma-separated list of allowed browser origins.

Default behavior allows all origins.

Example:

```dotenv
CORS_ORIGINS=https://app.example.com,https://admin.example.com
```

#### `ENABLE_API_DOCS`

Enables or disables Swagger UI at `/api-docs`.

Default:

```dotenv
ENABLE_API_DOCS=true
```

#### `API_DOCS_SERVERS`

Comma-separated server URLs displayed in the Swagger UI server selector.

Example:

```dotenv
API_DOCS_SERVERS=http://localhost:3900,https://your.domain.com
```

#### `API_DOCS_EXPANSION`

Controls how Swagger sections are expanded.

Supported values:

* `full`
* `list`
* `none`

Example:

```dotenv
API_DOCS_EXPANSION=list
```

#### Application branding

The following variables control the service name and descriptions shown in Swagger and health responses:

```dotenv
APP_TITLE=CLIBridge
APP_DESCRIPTION=HTTP gateway for AI CLIs (Codex & Claude) with an OpenAI-compatible API, streaming, image support, and local audio transcription (whisper.cpp — audio never leaves this server).
APP_SERVICE_NAME=cli-bridge
```

#### `HEALTH_VERBOSE_ENABLED`

Enables expanded output for:

```http
GET /v1/health?details=true
```

Default:

```dotenv
HEALTH_VERBOSE_ENABLED=false
```

### Backend selection

#### `BACKEND`

Selects the active CLI provider.

Supported values:

```dotenv
BACKEND=codex
```

or:

```dotenv
BACKEND=claude
```

Default:

```dotenv
BACKEND=codex
```

`BACKEND` is independent of `CODEX_BIN` and `CLAUDE_BIN`.

`BACKEND` determines which provider handles requests. The `*_BIN` variables only define the executable used by that provider.

The binary variable for the inactive backend is ignored.

Whisper audio transcription is independent of `BACKEND` entirely — it's always available (subject to plan entitlement, see below) regardless of which CLI provider is selected, and the Docker image bundles `whisper.cpp` in every variant (`codex`, `claude`, and `latest`/full).

### Codex CLI

These variables are used when:

```dotenv
BACKEND=codex
```

#### `CODEX_BIN`

Codex executable name or path.

Default:

```dotenv
CODEX_BIN=codex
```

#### Other Codex options

```dotenv
CODEX_TIMEOUT_MS=
CODEX_DEFAULT_MODEL=
CODEX_ALLOWED_MODELS=
CODEX_SANDBOX=
CODEX_ASK_FOR_APPROVAL=
CODEX_ALLOW_SEARCH=
```

#### `CODEX_SKIP_GIT_REPO_CHECK`

CLIBridge runs as a service rather than inside a project repository, so the Git repository check is skipped by default.

Default:

```dotenv
CODEX_SKIP_GIT_REPO_CHECK=true
```

#### `OPENAI_MODELS_HIDE_CODEX`

Controls whether Codex-specific internal models are hidden from the model list.

Default:

```dotenv
OPENAI_MODELS_HIDE_CODEX=true
```

### Claude CLI

These variables are used when:

```dotenv
BACKEND=claude
```

#### `CLAUDE_BIN`

Claude executable name or path.

Default:

```dotenv
CLAUDE_BIN=claude
```

#### `CLAUDE_DEFAULT_MODEL`

Default Claude model.

```dotenv
CLAUDE_DEFAULT_MODEL=claude-sonnet-4-6
```

#### `CLAUDE_ALLOWED_MODELS`

Optional comma-separated list of models that may be requested.

Example:

```dotenv
CLAUDE_ALLOWED_MODELS=claude-sonnet-4-6,claude-opus-4-6
```

### Audio transcription (Whisper)

These variables configure `POST /v1/audio/transcriptions` and `GET /v1/audio/status`, both always available regardless of `BACKEND` (subject to plan entitlement — see below).

#### `WHISPER_MODEL_PATH`

Path to the `.bin` ggml model used for transcription.

Default:

```dotenv
WHISPER_MODEL_PATH=/app/models/ggml-base.bin
```

Matches whichever model was baked into the image via the `WHISPER_MODEL` build argument. Point this at a different mounted model file to switch models without rebuilding.

#### `WHISPER_LANGUAGE`

Default ISO-639-1 language used when a request doesn't specify one.

Default:

```dotenv
WHISPER_LANGUAGE=auto
```

`auto` lets whisper.cpp detect the language per request. Pinning a specific language skips detection and can improve both speed and accuracy for single-language deployments.

#### `WHISPER_THREADS`

Pins whisper.cpp's CPU thread count per transcription job.

Left unset, whisper.cpp auto-detects (typically `min(4, core count)`). Set explicitly to cap CPU usage on a small or shared host, for example:

```dotenv
WHISPER_THREADS=1
```

#### `WHISPER_MAX_CONCURRENT`

Concurrent transcription jobs allowed at once.

Default:

```dotenv
WHISPER_MAX_CONCURRENT=1
```

Requests beyond this limit receive `429 Too Many Requests`. Keep this low on small or shared hosts — a transcription job is CPU/RAM-heavy, and this also protects concurrent chat requests running on the same box from resource starvation.

An in-progress job can also be cancelled outright with `POST /v1/audio/cancel`, freeing its slot immediately instead of waiting out the timeout below.

#### `WHISPER_TRANSCRIBE_TIMEOUT_MS`

Timeout for the actual whisper-cli inference step, separate from `WHISPER_TIMEOUT_MS` (which only covers the fast ffmpeg decode step).

Default (10 minutes):

```dotenv
WHISPER_TRANSCRIBE_TIMEOUT_MS=600000
```

Transcription time scales with both audio length and model size — a larger/more-accurate model transcribing 1-2 minutes of audio can take several minutes on modest CPU hardware. Raise this further if you still see `whisper-cli timed out` errors with a large model on slow hardware.

#### `MAX_AUDIO_BYTES`

Maximum upload size for `POST /v1/audio/transcriptions`, in bytes.

Default (25 MB, matching OpenAI's limit):

```dotenv
MAX_AUDIO_BYTES=26214400
```

#### `WHISPER_BIN`, `FFMPEG_BIN`, `WHISPER_TIMEOUT_MS`, `WHISPER_HEALTH_TIMEOUT_MS`

Advanced settings that rarely need changing — binary paths and timeouts for the ffmpeg decode step and the whisper.cpp run itself.

#### Build-time model selection

`WHISPER_MODEL` (default `base`) and `WHISPER_CPP_VERSION` are Docker build arguments, not runtime environment variables — they control which model size and which whisper.cpp release get baked into the image. Bigger models (`small`, `medium`, `large-v3`) are more accurate but need more RAM/CPU per request. See [Manual image build / push](#manual-image-build--push)-style rebuild instructions in `docs/dockerhub.md` if you maintain your own build.

### Plan entitlement

`POST /v1/audio/transcriptions` and `GET /v1/audio/status` require a CLIBridge plan that includes Whisper. If your plan doesn't include it, both endpoints return:

```json
{ "error": "whisper_not_entitled", "message": "Your current plan does not include audio transcription (Whisper). Upgrade your plan to enable it." }
```

with HTTP status `403`, and the dashboard hides the Whisper status card entirely instead of showing it locked. This is unrelated to `BACKEND`/CLI authentication — it's a separate entitlement tied to your CLIBridge subscription plan.

### Advanced configuration

CLIBridge also supports optional advanced settings for:

* Input and output size limits using `MAX_*`
* Summarization thresholds using `SUMMARY_THRESHOLD_TOKENS`
* Token counting using `TIKTOKEN_ENCODING`
* Image token estimation using `IMAGE_TOKEN_*`
* Scheduled insight reports using `INSIGHTS_*`
* SMTP delivery using `SMTP_*`
* Insight report emails using `INSIGHTS_EMAIL_ENABLED`
* CLI update alert emails using `CLI_UPDATE_EMAIL_ENABLED`
* CLI version checks using `CLI_UPDATE_*`
* Watchdog monitoring and self-healing using `WATCHDOG_*`

These options are not required for a basic deployment and can safely be omitted initially.

### Reverse-proxy variables

Variables such as the following are not read directly by CLIBridge:

```dotenv
VIRTUAL_HOST=
VIRTUAL_PORT=
LETSENCRYPT_HOST=
LETSENCRYPT_EMAIL=
```

These variables belong to reverse-proxy containers such as:

* `nginx-proxy`
* `acme-companion`
* `letsencrypt-companion`

They may be included in the CLIBridge service when it runs alongside those containers, but CLIBridge itself does not process them.

## API call example

```bash
curl http://localhost:3900/v1/chat/completions \
  -H "Authorization: Bearer $BRIDGE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "claude-sonnet-4-6",
    "messages": [
      {
        "role": "user",
        "content": "Hello!"
      }
    ]
  }'
```

You can also authenticate using the `X-Bridge-Token` header:

```bash
curl http://localhost:3900/v1/models \
  -H "X-Bridge-Token: $BRIDGE_TOKEN"
```

## HTTP endpoints

All routes below require `BRIDGE_TOKEN`, supplied using either:

```http
Authorization: Bearer <token>
```

or:

```http
X-Bridge-Token: <token>
```

The following route does not require a bridge token:

```http
GET /
```

All endpoints except `/v1/license/*` also require an activated CLIBridge license. Every `/v1/audio/*` endpoint additionally requires a plan that includes Whisper — see [Plan entitlement](#plan-entitlement) above.

### OpenAI-compatible endpoints

| Method   | Path                            | Description                                                |
| -------- | -------------------------------- | ------------------------------------------------------------ |
| `GET`    | `/v1/models`                    | List available models                                        |
| `POST`   | `/v1/models/refresh`            | Probe the active CLI and refresh the model cache             |
| `POST`   | `/v1/chat/completions`          | Create a chat completion using JSON or SSE streaming         |
| `POST`   | `/v1/chat/completions/upload`   | Create a chat completion with one multipart image upload     |
| `POST`   | `/v1/audio/transcriptions`      | Transcribe audio — 100% local whisper.cpp, plan-gated        |
| `POST`   | `/v1/audio/cancel`              | Cancel in-progress transcription(s), freeing capacity immediately |
| `GET`    | `/v1/audio/status`              | Whisper backend health, capacity, and usage metrics          |
| `GET`    | `/v1/audio/models`              | Whisper model catalog — downloaded/active state, live download progress |
| `POST`   | `/v1/audio/models/:name/select` | Switch model, downloading it first if needed                 |
| `POST`   | `/v1/audio/models/cancel`       | Cancel an in-progress model download                          |
| `DELETE` | `/v1/audio/models/:name`        | Delete a downloaded model to free disk space                  |

### CLI authentication endpoints

| Method   | Path                       | Description                                      |
| -------- | -------------------------- | ------------------------------------------------ |
| `GET`    | `/v1/auth/codex`           | Get Codex authentication status                  |
| `GET`    | `/v1/auth/claude`          | Get Claude authentication status                 |
| `POST`   | `/v1/auth/codex/start`     | Start Codex browser authentication               |
| `POST`   | `/v1/auth/claude/start`    | Start Claude browser authentication              |
| `POST`   | `/v1/auth/claude/:id/code` | Submit Claude's verification code                |
| `GET`    | `/v1/auth/sessions/:id`    | Poll an authentication session                   |
| `POST`   | `/v1/auth/codex/update`    | Update Codex CLI in the running container        |
| `POST`   | `/v1/auth/claude/update`   | Update Claude CLI in the running container       |
| `DELETE` | `/v1/auth/codex`           | Log out from Codex and clear stored credentials  |
| `DELETE` | `/v1/auth/claude`          | Log out from Claude and clear stored credentials |

### License endpoints

| Method | Path                        | Description                           |
| ------ | --------------------------- | ------------------------------------- |
| `GET`  | `/v1/license/status`        | Get the current license state         |
| `POST` | `/v1/license/session/start` | Start device activation               |
| `POST` | `/v1/license/refresh`       | Force a live license check            |
| `POST` | `/v1/license/revoke`        | Revoke the current activation session |

### Diagnostic endpoints

| Method | Path                      | Description                                            |
| ------ | ------------------------- | ------------------------------------------------------ |
| `GET`  | `/v1/health`              | Basic health status                                    |
| `GET`  | `/v1/health?details=true` | Expanded CLI, whisper, watchdog, and license health status |
| `GET`  | `/v1/metrics`             | Live request and token counters                        |
| `GET`  | `/v1/watchdog/status`     | Watchdog configuration and unhealthy-check count       |
| `GET`  | `/v1/logs`                | Recent operational log entries (transcription/download failures, timeouts) — resets on restart |
| `GET`  | `/v1/insights/latest`     | Get the latest generated insight report                |
| `GET`  | `/v1/insights/history`    | Get previous generated insight reports                 |
| `POST` | `/v1/insights/generate`   | Generate an insight report outside the normal schedule |

Full request and response schemas are available in:

```text
endpoints.md
```

They are also available through Swagger UI on a running CLIBridge instance:

```text
http://localhost:3900/api-docs
```

The following routes are intentionally excluded from Swagger:

```text
/v1/license/*
GET /
```

## Important notes

* Authentication and activation state are stored in the Docker volume mounted at `/data`.
* Keep the `cli_data` volume persistent between container restarts and upgrades.
* Do not run `docker compose down -v` unless you intentionally want to erase the activation and CLI authentication state.
* Use image version `2.1.1` or newer.
* Use a versioned Docker image tag in production instead of relying on `latest`.
* The combined image contains both CLIs but only one backend can process requests at a time. Whisper audio transcription is available regardless of which backend is selected.
* Protect `BRIDGE_TOKEN` as you would protect an API key.
* Place CLIBridge behind HTTPS before exposing it publicly.
* Audio sent to `POST /v1/audio/transcriptions` never leaves the container — there is no external API call for transcription.
