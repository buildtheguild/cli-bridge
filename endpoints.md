# Endpoints

There are two independent guards, and it matters which ones a route has:

- **`AuthGuard`** — requires `BRIDGE_TOKEN`, applied per-controller via
  `@UseGuards(AuthGuard)`. Checked via either header:
  - `Authorization: Bearer <BRIDGE_TOKEN>`
  - `X-Bridge-Token: <BRIDGE_TOKEN>`
- **`LicenseGuard`** — a *global* guard (registered as `APP_GUARD`) that
  blocks every route unless the license is `valid`/`grace`, **except** routes
  explicitly marked `@SkipLicenseCheck()`. This is independent of `AuthGuard`
  — a route can require the token but still be reachable pre-activation if
  it's marked skip.

Only `GET /` has neither guard — it's the only truly public route.
Everything else, including every `/v1/license/*` route, requires
`BRIDGE_TOKEN`.

| Route | Requires `BRIDGE_TOKEN` | Requires valid license |
|---|---|---|
| `GET /` | No | No (`@SkipLicenseCheck`) |
| `GET /v1/license/*`, `POST /v1/license/*` | **Yes** | No (`@SkipLicenseCheck`) |
| `GET /v1/auth/*`, `GET /v1/auth/sessions/:id` | Yes | Yes |
| Everything under `/v1` on `CodexController` (health, metrics, watchdog, insights, models, chat) | Yes | Yes |

`GET /v1/license/status` being reachable without a valid license (but still
needing the token) is what lets the operator console show license state
before/after activation, and is why the docs previously implying it was
tokenless was wrong — check `src/auth/auth.guard.ts` and
`src/license/license.guard.ts` if this drifts again.

## Not in Swagger UI

`LicenseController` and `SetupController` are both `@ApiExcludeController()`
— all `/v1/license/*` routes and `GET /` are real, working endpoints, but
deliberately don't show up at `/api-docs`. Everything below "OpenAI-compatible
API" is what Swagger actually documents; treat this section as the source of
truth for the excluded routes.

## `GET /`

Serves the operator console (HTML). Prompts for `BRIDGE_TOKEN` client-side
before it will call any authenticated endpoint (bootstrap, license status,
auth status, etc.) on your behalf.

## License (`/v1/license`, `AuthGuard` + `@SkipLicenseCheck` on every route)

- `GET /v1/license/bootstrap`
  - single call the operator console uses on unlock: license state + which
    CLI providers this deployment supports + whether docs are enabled
- `GET /v1/license/status`
  - current license state (`pending_activation` / `activating` / `valid` /
    `grace` / `denied`)
- `POST /v1/license/refresh`
  - forces a live check against the SaaS backend instead of waiting for the
    12-hour cache timer
- `POST /v1/license/session/start`
  - starts device activation; returns `activationUrl` / `deviceCode` /
    `expiresIn`
  - returns HTTP 409 if the bridge is already activated
- `POST /v1/license/revoke`
  - revokes the current remote session and clears local state; bridge
    returns to `pending_activation`

## CLI auth (`/v1/auth`, `AuthGuard`, license required)

**Codex:**
- `GET /v1/auth/codex` — auth status, credentials path, active session, CLI/latest version
- `POST /v1/auth/codex/update` — runs `npm install -g @openai/codex@latest` in the running container (container-only, doesn't persist across image rebuilds)
- `POST /v1/auth/codex/start` — spawns `codex login --device-auth`, returns a session with `loginUrl`; poll `GET /v1/auth/sessions/:id`
- `DELETE /v1/auth/codex` — removes the stored auth file and kills any in-progress session

**Claude:**
- `GET /v1/auth/claude` — auth status, credentials path, active session, CLI/latest version
- `POST /v1/auth/claude/update` — runs `npm install -g @anthropic-ai/claude-code@latest` in the running container
- `POST /v1/auth/claude/start` — spawns `claude auth login`, returns a session with `loginUrl`
- `POST /v1/auth/claude/:id/code` — submits the browser-returned verification code to finish login
- `DELETE /v1/auth/claude` — removes the stored credentials file and kills any in-progress session

**Shared:**
- `GET /v1/auth/sessions/:id` — status of a specific Codex or Claude login session (`awaiting_browser` / `awaiting_code` / `authenticated`)

All CLI-auth and CLI-update routes are additionally rate-limited per source IP.

## Diagnostics

- `GET /v1/health` — basic `{ ok, time }`; `?details=true` (requires `HEALTH_VERBOSE_ENABLED=true`) adds version, memory, `cli`, `watchdog`, `license`. `cli.quotaLeft` is always `null` — neither CLI exposes machine-readable remaining quota.
- `GET /v1/watchdog/status` — watchdog config + consecutive unhealthy check count
- `GET /v1/metrics` — live in-memory counters: `totalRequests`, `chatRequests`, `streamRequests`, `failedRequests`, `totalInputTokens`, `totalOutputTokens`, `activeSessions`, `sessionLimit`. Lightweight — no AI calls.
- `GET /v1/insights/latest` — most recent insight report, or `null`
- `GET /v1/insights/history?limit=10` — last N reports, newest first
- `POST /v1/insights/generate` — triggers an out-of-schedule insight report, returns it

## OpenAI-compatible API

### `GET /v1/models`

Returns model list in OpenAI-compatible shape (`object: "list", data: [...]`).

**Codex backend:**
- Source: Codex `models_cache.json` when available.
- Fallback: `CODEX_ALLOWED_MODELS` / `CODEX_DEFAULT_MODEL` / `gpt-5-codex`.
- By default, any `-codex` marker is hidden in advertised ids (for example `gpt-5-codex-mini` -> `gpt-5-mini`) when `OPENAI_MODELS_HIDE_CODEX=true`.
- Requests accept both forms (with or without `-codex`) and map to the internal Codex model id.

**Claude backend:**
- Source: `CLAUDE_ALLOWED_MODELS` / `CLAUDE_DEFAULT_MODEL` / `claude-sonnet-4-6`.
- No cache file; model list is always derived from env vars.

### `POST /v1/models/refresh`

Runs a small CLI probe to verify the backend is authenticated and responsive, then returns the current OpenAI-compatible model list plus:

- `ok` (probe success/failure)
- `message`
- `cachePath` (Codex backend only — resolved cache file path when available; `null` for Claude)

For Claude, this is a useful post-login probe to confirm the CLI is authenticated and responsive.

### `POST /v1/chat/completions`

OpenAI-compatible chat completions endpoint.

Supported request fields:

- `model` (optional)
- `messages` (required) — `role` is one of `system` / `user` / `assistant` / `tool` / `developer`; `content` is either a string or an array of parts (`text`, `input_text`, or `image_url`/`input_image`)
- `stream` (optional)
- `response_format` (`json_object` / `json_schema`)

Image inputs (multiple images supported):

- `messages[].content` may be an array of parts. Include a part with `type: "image_url"` (or `input_image`) and `image_url: { "url": "https://..." }` or a data URI.
- Each image is downloaded server-side (http/https or data URI).
- Max size per image: 10 MB.
- Multipart upload (single image) for testing:
  - `POST /v1/chat/completions/upload`
  - `messages` is a JSON string
  - `imageFile` is a binary file field
  - `stream` is not supported on this endpoint

Streaming returns OpenAI-style SSE chunks (`chat.completion.chunk`) and ends with `data: [DONE]`.

Non-streaming errors and stream-setup errors both return the OpenAI error shape: `{ error: { message, type, param: null, code: null } }`.

Auto-summary:

- If input exceeds `SUMMARY_THRESHOLD_TOKENS`, the response includes a top-level `summary` string (non-streaming only).

Model validation is strict: if `model` is provided and does not map to a known cached/allowed model, the bridge returns HTTP 400.
