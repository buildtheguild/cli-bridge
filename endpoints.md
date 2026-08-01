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
| `GET /v1/logs` | Yes | Yes |
| Everything under `/v1/audio/*` (`AudioController`, `WhisperModelsController`) | Yes | Yes, **plus** a plan-entitlement check — a valid license on a plan that doesn't include Whisper still gets `403 { error: "whisper_not_entitled" }` on every route here. |

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

- `GET /v1/health` — basic `{ ok, time }`; `?details=true` (requires `HEALTH_VERBOSE_ENABLED=true`) adds version, memory, `cli`, `whisper`, `watchdog`, `license`. `cli.quotaLeft` is always `null` — neither CLI exposes machine-readable remaining quota.
- `GET /v1/watchdog/status` — watchdog config + consecutive unhealthy check count
- `GET /v1/metrics` — live in-memory counters: `totalRequests`, `chatRequests`, `streamRequests`, `failedRequests`, `totalInputTokens`, `totalOutputTokens`, `activeSessions`, `sessionLimit`. Lightweight — no AI calls. Reflects real `/v1/chat/completions`-family traffic only; the periodic insights report's own chat call (see `docs/config.md`) is excluded so its success/failure never shows up here — check `GET /v1/logs` for that instead.
- `GET /v1/logs?limit=100` — in-memory ring buffer (max 200) of `{ id, timestamp, level, context, message }`, newest first. Populated by Codex/Claude chat and streaming request failures (`context: "Codex"`/`"Claude"`) and the Whisper module's failure points (decode failures, missing-model errors, whisper-cli failures/timeouts, model-download failures/retries, `context: "Whisper"`) — not a full application log, just these operator-relevant failure points. Resets on restart; not persisted anywhere. Guarded by `AuthGuard` only — no Whisper plan entitlement required, since it's not itself a Whisper feature.
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

When a request sends an unknown `model` and fallback is enabled, successful responses include a top-level `model_fallback` object:

```json
{
  "model_fallback": {
    "requested": "claude-sonnet-4-6",
    "used": "gpt-5.5",
    "reason": "unknown_model",
    "message": "Requested model 'claude-sonnet-4-6' is not available for the active backend, so the server default model was used instead."
  }
}
```

When a known emergency condition forces an internal OpenRouter failover, successful responses include a top-level `provider_fallback` object:

```json
{
  "provider_fallback": {
    "from": "claude",
    "to": "openrouter",
    "reason": "rate_limited",
    "model": "openai/gpt-5"
  }
}
```

Auto-summary:

- If input exceeds `SUMMARY_THRESHOLD_TOKENS`, the response includes a top-level `summary` string (non-streaming only).

Model validation is configurable:

- Default behavior (`MODEL_FALLBACK_TO_DEFAULT_ON_UNKNOWN` omitted, or any value other than `false`): if `model` is provided but does not map to a known cached/allowed model, the bridge silently falls back to the backend default model.
- Strict behavior (`MODEL_FALLBACK_TO_DEFAULT_ON_UNKNOWN=false`): the bridge returns HTTP 400 with `Unknown model ...`.

Emergency provider failover is separate from model validation. When `OPENROUTER_ENABLE_EMERGENCY_FALLBACK=true` and the OpenRouter credentials are configured, the bridge may internally retry the request through OpenRouter only for known emergency conditions such as missing CLI auth, rate limiting, quota exhaustion, or provider unavailability. It does not retry generic application errors.

## Audio / Whisper (`/v1/audio`, requires auth + a plan that includes Whisper)

100% local — audio is decoded and transcribed entirely inside the container via a bundled `whisper.cpp`; nothing is sent to OpenAI, Anthropic, or any other external service. Independent of `BACKEND` — available regardless of which CLI (Codex or Claude) is active. Every route below additionally requires the license to be on a plan that includes Whisper (see the table above); a plan without it gets `403 { error: "whisper_not_entitled" }` on all of them.

### `POST /v1/audio/transcriptions`

OpenAI-compatible transcription. `multipart/form-data`:

- `file` (required) — audio file, any format ffmpeg can decode (mp3/m4a/wav/ogg/opus/webm/...)
- `model` (optional) — per-request override; must be an **already-downloaded** catalog name (see `GET /v1/audio/models`). Omitted, empty, or OpenAI's fixed `"whisper-1"` placeholder uses whichever model is currently active. An unknown or undownloaded name returns HTTP 400 — this never triggers a download mid-request. Does **not** change the globally active model.
- `language` (optional) — ISO-639-1 code; omitted uses `WHISPER_LANGUAGE` (default `auto`, i.e. per-request detection)
- `prompt` (optional) — vocabulary/context hint passed through to whisper.cpp
- `response_format` (optional) — `json` (default, `{ text }`), `text` (raw `text/plain` body), or `verbose_json` (`{ task, language, duration, text, segments }`, segments include per-utterance start/end timestamps)

Concurrency is capped by `WHISPER_MAX_CONCURRENT` (default 1); requests beyond that get HTTP 429. Decode uses `WHISPER_TIMEOUT_MS` (default 120000); the whisper-cli inference step uses the separate, more generous `WHISPER_TRANSCRIBE_TIMEOUT_MS` (default 600000 / 10 min) — transcription time scales with both audio length and model size, so a large model on modest hardware can legitimately take minutes.

### `POST /v1/audio/cancel`

Aborts whatever transcription request(s) are currently running, via `AbortSignal` on the spawned `ffmpeg`/`whisper-cli` processes. Frees the capacity slot immediately. The in-flight request(s) receive `400 { message: "Transcription was cancelled." }` instead of completing or timing out. Returns `400` if nothing is currently running. Response: `{ cancelled: <count> }`.

### `GET /v1/audio/status`

Health/capacity snapshot — not gated by `HEALTH_VERBOSE_ENABLED` (unlike the verbose fields on `GET /v1/health`), only by the Whisper entitlement above.

```json
{
  "available": true,
  "bin": "whisper-cli",
  "binaryOk": true,
  "model": { "name": "base", "path": "/app/models/ggml-base.bin", "exists": true, "sizeBytes": 147964211 },
  "ffmpeg": { "bin": "ffmpeg", "ok": true },
  "maxConcurrent": 1,
  "activeJobs": 0,
  "metrics": { "totalRequests": 12, "successCount": 11, "failedRequests": 1, "avgProcessingMs": 4200 }
}
```

`available` is `true` only if the model file exists, `whisper-cli --help` runs successfully, and `ffmpeg -version` runs successfully.

### `GET /v1/audio/models`

Static catalog (`tiny`/`base`/`small`/`medium`/`large-v3-turbo`/`large-v3`) plus live state.

```json
{
  "activeModel": "base",
  "builtInModel": "base",
  "downloading": null,
  "models": [
    { "name": "tiny", "label": "Tiny — fastest, lowest accuracy", "approxBytes": 78000000, "downloaded": false, "active": false, "builtIn": false },
    { "name": "base", "label": "Base — fast, good baseline", "approxBytes": 148000000, "downloaded": true, "active": true, "builtIn": true }
  ]
}
```

`builtInModel` is whatever `WHISPER_MODEL_PATH` resolves to (the model baked into the image); `activeModel` is whatever was last selected via the endpoint below, defaulting to `builtInModel` if nothing has been selected yet. `downloading` is `null`, or `{ name, status, downloadedBytes, totalBytes, error? }` with `status` one of `downloading` / `done` / `failed` / `cancelled` — poll this endpoint while `status === "downloading"` to track progress.

### `POST /v1/audio/models/:name/select`

If `:name` is already downloaded (or is `builtInModel`), activates it immediately and returns `{ status: "active" }`. Otherwise starts a background download from `https://huggingface.co/ggerganov/whisper.cpp/resolve/main/ggml-:name.bin` to `$HOME/whisper-models/` on the persistent `/data` volume, returns `{ status: "downloading" }` immediately (not awaited — a large model can take minutes), and activates automatically once it completes. Retries the download up to 3 times with backoff on a dropped connection before surfacing a `failed` state. Only one download runs at a time — a second `select()` while one is in flight gets HTTP 429. Unknown `:name` returns HTTP 400.

The active selection is written to `$HOME/whisper-active-model.json` on `/data`, so it survives container recreation and image updates.

### `POST /v1/audio/models/cancel`

Aborts an in-progress model download, cleans up the partial file, and leaves the previously-active model unchanged (`downloading.status` becomes `cancelled`). Returns HTTP 400 if nothing is currently downloading.

### `DELETE /v1/audio/models/:name`

Frees disk space. Returns HTTP 400 if `:name` is `builtInModel` (baked into the image, nothing to delete) or the currently active model (switch away first).
