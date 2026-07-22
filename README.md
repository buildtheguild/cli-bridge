# CLIBridge

**HTTP gateway that turns the OpenAI Codex CLI or Anthropic Claude Code CLI into a standard OpenAI-compatible API.**

Use your own **ChatGPT Plus** or **Claude Pro** subscription. CLIBridge handles authentication, streaming, image inputs, rate limiting, health monitoring, and SaaS-based activation.

**Docker Hub:** https://hub.docker.com/r/thebuildguild/cli-bridge

**HTTP endpoints:** [here](#http-endpoints)

## Tags

| Tag                      | CLI backends   | Size    |
| ------------------------ | -------------- | ------- |
| `latest`, `1.6.2`        | Codex + Claude | ~930 MB |
| `claude`, `1.6.2-claude` | Claude only    | ~590 MB |
| `codex`, `1.6.2-codex`   | Codex only     | ~420 MB |

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
    image: thebuildguild/cli-bridge:1.6.2-codex
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
    image: thebuildguild/cli-bridge:1.6.2-claude
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
    image: thebuildguild/cli-bridge:1.6.2
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
APP_DESCRIPTION=OpenAI-compatible gateway for Codex and Claude CLI
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

All endpoints except `/v1/license/*` also require an activated CLIBridge license.

### OpenAI-compatible endpoints

| Method | Path                          | Description                                              |
| ------ | ----------------------------- | -------------------------------------------------------- |
| `GET`  | `/v1/models`                  | List available models                                    |
| `POST` | `/v1/models/refresh`          | Probe the active CLI and refresh the model cache         |
| `POST` | `/v1/chat/completions`        | Create a chat completion using JSON or SSE streaming     |
| `POST` | `/v1/chat/completions/upload` | Create a chat completion with one multipart image upload |

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
| `GET`  | `/v1/health?details=true` | Expanded CLI, watchdog, and license health status      |
| `GET`  | `/v1/metrics`             | Live request and token counters                        |
| `GET`  | `/v1/watchdog/status`     | Watchdog configuration and unhealthy-check count       |
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
* Use image version `1.6.2` or newer.
* Use a versioned Docker image tag in production instead of relying on `latest`.
* The combined image contains both CLIs but only one backend can process requests at a time.
* Protect `BRIDGE_TOKEN` as you would protect an API key.
* Place CLIBridge behind HTTPS before exposing it publicly.
