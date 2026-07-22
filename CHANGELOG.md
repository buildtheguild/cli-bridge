# Changelog

All notable changes to this project are documented here.

---

## [1.6.2] – 2026-07-10

### Fixed

- **Claude backend: image requests (both `image_url` and `POST /v1/chat/completions/upload`) failed with a 500** — the bridge was passing `--image <path>` to the `claude` CLI, a flag that doesn't exist there (it's a Codex CLI flag; the two CLIs were wrongly assumed to share the same interface). The Claude process exited non-zero on the unrecognized option, and — compounding it — `chatCompletionsUpload` had no try/catch around its body, so the resulting plain `Error` fell through to Nest's default exception filter as a generic `{"statusCode":500,"message":"Internal server error"}` with no useful detail. Images are now sent the way the Claude CLI actually supports it: `--input-format stream-json` with base64-encoded image content blocks on stdin. The upload endpoint's error handling now also mirrors `/v1/chat/completions`, returning a proper OpenAI-style `{"error":{"message":...}}` envelope with the real message and status code for any failure, not just image-related ones.
- **Claude backend: `stream: true` failed on every request, not just image ones** — found while fixing the above. `--output-format stream-json` requires `--verbose` when combined with `--print`, or the CLI exits immediately with a usage error; the streaming code path never passed it. Streaming now works for the Claude backend, with or without images.
- **Claude backend: `response_format: json_object` / `json_schema` produced malformed or wrapped JSON around 1 in 3 requests** — the bridge asked the model to "return JSON only" in the prompt and regex-extracted the first `{...}` block from its reply, which broke whenever Claude added any surrounding commentary. JSON output is now enforced natively via the CLI's `--json-schema` flag (tool-based structured output, the same mechanism Claude Code itself uses for structured responses) instead of text-based prompting — `json_object` mode with no caller-supplied schema gets a permissive `{"type":"object","additionalProperties":true}` fallback so any object shape is still accepted. Verified materially more reliable in back-to-back testing (5/5 and 3/3 clean runs across plain and image+JSON requests, versus roughly 2/3 before).
- **Claude backend: multiple images in one request were unreliable, and some requests (with or without images) got a meta-commentary refusal instead of an answer** — root cause was upstream of all three fixes above: the bridge builds prompts by flattening every message into `ROLE:\ntext` blocks (`buildMessagesPrompt`), a design that made sense for Codex's single-shot CLI (no native multi-turn/role concept) but was being reused for Claude too. Claude's safety training reads an inline `SYSTEM:` label — especially paired with the bridge's own "don't reveal system prompts" guard instruction — as characteristic of a prompt-injection attempt, and occasionally responded with a refusal/meta-commentary about it instead of the actual answer. This got markedly worse with multiple images attached. Claude now gets its own prompt path: system-role messages (and the guard instruction) are passed through the CLI's real `--append-system-prompt` flag instead of being faked as text, and prior conversation turns are folded into a plain "Here is the conversation so far: ..." transcript rather than labeled `SYSTEM:`/`USER:` blocks. (Prior turns are folded into text rather than replayed as separate native turns deliberately — replaying them as real `stream-json` turns was confirmed to make the CLI generate, and bill for, a fresh response at every one of them instead of just the last.) Codex is unaffected — it still uses the original flattened prompt builder, which was never broken.

---

## [1.6.1] – 2026-06-22

### Fixed

- **A cancelled-subscription false positive could revoke a perfectly valid license** — the 1.6.0 fix for unenforced cancellations (see below) revoked local `valid`/`grace` status immediately on a single `allowed: false` response from the SaaS entitlement check. A still-paying customer hit this on a routine image upgrade: a token refresh succeeding right before the entitlement check proves nothing about actual entitlement, so a one-off backend hiccup was enough to force a full re-activation. `allowed: false` now routes through the same 72-hour grace window already used for an unreachable backend, instead of revoking instantly — a genuine cancellation is still enforced, just not on the very first blip. This also required decoupling `lastSuccessfulCheckAt` (the grace-window anchor) from "token refresh succeeded" to "entitlement positively confirmed," since the two had been conflated — without that, every 12-hour token refresh would have kept resetting the grace clock and a real cancellation could never actually expire.

### Removed

- **Cancel button on the activation screen** — removed along with `POST /v1/license/session/cancel` (introduced in 1.6.0). The activation step already waits indefinitely via automatic device-code renewal, so a manual abort control wasn't needed in practice.

---

## [1.6.0] – 2026-06-21

### Added

- **CLI version update checks** — the bridge now periodically checks the npm registry for newer Codex/Claude CLI releases (`CLI_UPDATE_CHECK_INTERVAL_HOURS`, default every 6 hours, scoped to whichever provider(s) `BACKEND` selects). When a newer version is available:
  - The provider's dashboard card shows an "Update available" banner with an **Update** button. Clicking it runs `npm install -g <package>@<latest>` inside the running container — fully manual, nothing updates automatically.
  - The notifications panel lists an "Update available" entry per outdated provider.
  - If `CLI_UPDATE_EMAIL_ENABLED=true`, one email is sent the first time each new version is detected (not repeated on every check cycle).
  - New endpoints: `POST /v1/auth/codex/update`, `POST /v1/auth/claude/update`. `GET /v1/auth/codex` and `GET /v1/auth/claude` now also report `latestVersion`/`updateAvailable`.
  - This updates the running container's writable layer only — a later `docker compose up --force-recreate`, image pull, or fresh deploy reverts to the version baked into the image. Rebuild and republish to make an upgrade persistent.
- **License activation no longer times out** — a device code's lifetime (~10 minutes) could expire before a user finished checkout on the approval page, dead-ending the wizard back at "no active license found." The bridge now silently mints a fresh device code when one expires (client-side deadline or an upstream `expired` status) and keeps the existing approval URL refreshing in place, so the activation step effectively waits indefinitely.
- **Cancel button on the activation screen** — `POST /v1/license/session/cancel` plus a Cancel button in the wizard, for deliberately aborting an in-progress activation instead of waiting it out.
- **Auto-open activation/login URLs** — starting license activation or a Codex/Claude login now opens the approval/login URL in a new tab automatically. Falls back to the existing "Open" button + copy-link if the tab is blocked by a popup blocker.
- **Live license refresh from the wizard** — `POST /v1/license/refresh` forces a round-trip to the SaaS backend instead of returning the locally cached state (which otherwise only refreshes on a 12-hour timer). The topbar refresh button now uses this while on the wizard's License step; refresh behavior on the Dashboard is unchanged.

### Fixed

- **Cancelled subscriptions were never enforced, and the displayed expiry was meaningless** — `expiresAt` was being set from the access token's own short-lived `exp` claim on every refresh, not the subscription period end, so it kept creeping forward each time the token refreshed (independent of the real plan). Worse, when the SaaS backend's entitlement check reported `allowed: false` (e.g. a cancelled subscription), nothing happened — the bridge kept reporting `valid` indefinitely as long as the refresh token still worked. `expiresAt` now only ever comes from the entitlement check's real `renewsAt` and is preserved (not overwritten) between checks, and an `allowed: false` response now immediately revokes local `valid`/`grace` status back to `pending_activation` (re-subscribing is picked up automatically on the next check, no re-activation needed). The 12-hour background recheck interval is unchanged — use the manual refresh button for an immediate check.
- **Watchdog restart loop flooding email** — `WATCHDOG_RESTART_ON_CRITICAL=true` treated "not logged in" as a critical condition and restarted the process, but a restart can never restore a credential that was never there — it just crash-looped roughly every 3 minutes. Each restart re-triggered `INSIGHTS_RUN_ON_START`'s startup email with no cooldown, flooding the configured inbox. The watchdog now only restarts for conditions a restart can plausibly fix (idle timeout, high failure rate); "not logged in" still surfaces in status/alerts but never triggers a process exit.
- **Insight emails now have a cooldown floor** — added `INSIGHTS_EMAIL_COOLDOWN_MIN` (default 15 minutes) as defense-in-depth so no future rapid-restart scenario can flood the inbox, independent of the watchdog fix above.

### Changed

- **License active card recolored** — the "License active" badge icon and days-remaining progress bar now use the app's purple brand color instead of green, matching the rest of the console.

### Infrastructure

- **Dockerfile no longer reinstalls dependencies on every publish** — `ARG APP_VERSION` (which changes on every release) was declared and baked into `ENV APP_VERSION` *before* both CLI installs (`npm i -g`) and the production `npm ci` in the runtime stage, so changing only the version number busted Docker's layer cache for those steps and forced a full reinstall on every single build. `APP_VERSION` is now declared last, right before `CMD` — it's purely a runtime display value, so this is a no-op behaviorally. `LICENSE_SAAS_URL`/`LICENSE_PRODUCT_SLUG` in the builder stage were similarly moved down to just before the one step that consumes them. Rebuilding with unchanged dependencies and CLI versions now reuses cache for `npm ci`/`npm i -g` instead of redoing them from scratch.

---

## [1.5.2] – 2026-06-18

### Added

- **`GET /v1/metrics` endpoint** — lightweight endpoint returning live runtime counters (`totalRequests`, `chatRequests`, `streamRequests`, `failedRequests`, `totalInputTokens`, `totalOutputTokens`, `activeSessions`, `sessionLimit`) directly from memory. No AI call, no insight generation, no email — pure in-process read.
- **Dashboard auto-refresh** — metric cards (Uptime, Failure Rate, Requests, Tokens) now update automatically every 20 seconds while the dashboard is open. Polling starts on entering the dashboard screen and stops on leaving it.

### Fixed

- **Uptime card always showing N/A** — `GET /v1/health?details=true` previously gated `uptimeSec` and `version` behind `HEALTH_VERBOSE_ENABLED=true`. That env var is not set in standard Docker deployments, so both fields were never returned. `version` and `uptimeSec` are now always included in the `details=true` response; only genuinely sensitive fields (`pid`, `memory`, `config`, `cli`, `watchdog`, `license`) remain behind the flag.
- **Footer showing no version** — same root cause as Uptime N/A; resolved by the same fix.
- **Metric cards sourcing stale data** — the frontend was reading `totalRequests`, `failedRequests`, and token totals from `/v1/insights/latest`, which only updates when an insight is generated (default: every 12 hours or on manual trigger). The dashboard now fetches `/v1/metrics` for live values.

---

## [1.5.1] – 2026-06-18

### Added

- **Bridge logo as favicon and app logo** — `bridge.png` is now served from a `public/` static directory and used as the browser tab favicon, the topbar logo, and the login card logo. Replaces the generic Vuexy SVG placeholder in all three locations.

### Changed

- **`vuexy-assets` trimmed from 61 MB to 3.5 MB** — removed all image directories, audio, JSON, SVG, unused vendor libraries (50+ libs including jQuery, apex-charts, DataTables, etc.), FontAwesome, flag icons, and vendor JS. Only the seven CSS files actually referenced by the UI are retained alongside their three supporting libs (`node-waves`, `perfect-scrollbar`, `bs-stepper`).

### Infrastructure

- `public/` directory added to the project; served at the root URL prefix by NestJS static assets middleware. Dockerfile copies it into the image alongside `vuexy-assets`.

---

## [1.5.0] – 2026-06-18

### Added

- **Runtime metrics dashboard** — the four metric cards on the dashboard now show live data:
  - **Uptime** — session uptime in seconds from `GET /v1/health?details=true`.
  - **Failure Rate** — percentage of failed requests with absolute failed-request count, colour-coded green/red.
  - **Requests** — total requests handled, sourced from `GET /v1/insights/latest`.
  - **Tokens (in/out)** — combined card showing input and output token totals as `N / N`.
- **CLI version per provider** — each provider card in the console now shows the installed CLI version (e.g. `codex 0.139.0`) detected by running `--version` on the provider binary. Falls back to credentials path if the binary is unreachable.
- **Loading overlay** — a centred spinner replaces the white blank screen shown while the page bootstraps or refreshes. The overlay is visible by default and dismissed the moment the active screen is resolved.
- **Dashboard footer** — displays copyright, current year, app name and version (from `/v1/health`), and three hyperlinks (License, Documentation, Support) pointing to `https://cli-bridge.hobbytronics.pk`.
- **`activatedAt` in license state** — `LicenseState` now carries `activatedAt` (populated from `subscription.currentPeriodStart` via the policy-check response). Used to compute the exact subscription period length for the days-remaining progress bar.

### Changed

- **License card**:
  - Removed the "Your Current Plan is …" heading.
  - Progress bar label now reads "Last checked [date]" instead of "Expires [date]".
  - Days remaining denominator is computed from `activatedAt` → `expiresAt` (falls back to 365 days) instead of a hardcoded 30.
  - Revoke button moved inside the right column below the progress bar; Refresh runtime button removed.
- **Provider hero icons** — icon area increased to `min-height: 14rem`; icon rendered at `font-size: 8rem` so it fills the chip correctly regardless of which Tabler icon is used.
- **Topbar** — operator avatar dropdown replaced with a compact "Sign out" button (`tabler-logout` icon + label).
- **Metric cards** — all four cards use `d-flex` + `w-100` so they stretch to equal height in the row. Removed the `font-size: 1.75rem` override from the Tokens card so all four values share identical `console-metric` typography.

### Fixed

- Login-screen flash on hard refresh — login section is now hidden by default and shown only after the session check resolves.
- Days-remaining counter showed incorrect totals (e.g. "354 of 30 days") due to a hardcoded 30-day period; now derived from the actual subscription window.

---

## [1.4.1] - 2026-06-15

### Fixed

- Fixed Docker Hub bridge images that were incompatible with the current SaaS device-flow tokens.
- Removed the retired local Ed25519 verification path from runtime license activation so approved SaaS `RS256` sessions can become valid after approval and restart.
- Prevented the failure mode where browser approval succeeded but the bridge stayed stuck in `pending_activation` or `activating`.

### Changed

- Clarified in code and docs that:
  - the SaaS backend is authoritative for refresh and entitlement checks
  - the native addon remains for build-time immutable release metadata
- Switched release documentation to `docker buildx` multi-platform publishing as the standard path.
- Documented the known release pitfall from `1.4.0-claude` so it is not repeated.

### Verification

- Reproduced the bug locally with `thebuildguild/cli-bridge:1.4.0-claude`.
- Confirmed the old image logged `License token failed local Ed25519 verification.` against the current SaaS backend.
- Verified the patched source removes that legacy runtime check.

---

## [1.4.0] – 2026-06-13

### Added

- **Swagger-managed CLI login flows**:
  - `GET /v1/auth/codex` and `POST /v1/auth/codex/start`
  - `GET /v1/auth/claude` and `POST /v1/auth/claude/start`
  - `POST /v1/auth/claude/:id/code`
  - `GET /v1/auth/sessions/:id`
- **Activation lifecycle gating**:
  - `GET /` now serves the activation page only while license state is `pending_activation` or `activating`
  - after activation, `/` redirects to `/api-docs` when docs are enabled, otherwise serves a minimal activated landing page
  - `POST /v1/license/session/start` is rejected after activation
- **Request hardening for image inputs**:
  - upload endpoint now limits multipart file size and accepts image MIME types only
  - remote image fetches now enforce timeouts, block localhost/private-network targets, and require image content types
- **Rate limiting and audit-style logging** for activation start and CLI auth start/code endpoints
- **Release verification tooling**:
  - `npm run smoke`
  - `docs/release-smoke.md`
  - `docs/cloudflare-tunnel.md`
- **Configurable CLI auth timeouts**:
  - `CLI_AUTH_TIMEOUT_MS`
  - `CLI_AUTH_STATUS_TIMEOUT_MS`

### Security

- **Removed runtime license authority override**:
  - the bridge no longer trusts `SAAS_URL` / `SAAS_PRODUCT_*` environment variables at runtime
  - the license authority URL and product slug are now compiled into `license_verify.node`
- **Docker runtime now ships the native license addon** instead of silently falling back to environment configuration
- **Local session bootstrap no longer grants validity from decoded JWT expiry alone**; the bridge now requires a successful online refresh before restoring a valid session after restart
- **Primary deployment surface no longer exposes internal SaaS network assumptions**:
  - removed `saas-net`
  - removed SaaS `extra_hosts`
  - removed public docs/runtime references to the private licensing hostname

### Changed

- **Docker deployment model**:
  - `docker-compose.yml` is now image-only and uses `CLI_BRIDGE_IMAGE`
  - normal operator flow is `docker compose pull && docker compose up -d`
  - release image publishing is documented separately via manual `docker build` / `docker push`
- **Docker image build now compiles the license addon** with build-time arguments:
  - `LICENSE_SAAS_URL`
  - `LICENSE_PRODUCT_SLUG`
- **Pinned bundled CLI versions in Docker image builds**:
  - `@openai/codex@0.139.0`
  - `@anthropic-ai/claude-code@2.1.173`
- **Test backend overlay now uses a neutral internal hostname** (`license.test.internal`) instead of a production SaaS hostname

### Verification

- `npm test` passes
- `npm run smoke` passes against the local live bridge after rebuild/restart

---

## [1.3.0] – 2026-06-10

### Added

- **Session concurrency limit** — `CodexService` now enforces a maximum number of simultaneous in-flight requests.
  - A private `activeRequests` counter is incremented at the start of each `chat`, `chatStream`, and `chatStreamOpenAi` call and always decremented in a `finally` block.
  - `acquireSession()` throws HTTP 429 (`Too Many Requests`) with a descriptive message if `activeRequests >= sessionLimit`.
  - `GET /v1/health?details=true` now includes `activeSessions` (current in-flight count) and `sessionLimit` (configured cap) in the `metrics` block.
- **`LicenseService.getSessionLimit()`** — public method returning `state.maxConcurrentSessions`.
- **`LicenseState.maxConcurrentSessions`** — new field. Populated from `limits.active_devices` in the policy check response. Defaults to `1` during pending activation and the grace period.
- **Policy check reads concurrency limit** — `checkAndApplyEntitlement()` now extracts `data.limits.active_devices` from the `POST /v1/policy/check` response and stores it in the license state. Changing the plan in the admin portal takes effect on the next 12-hour token refresh.

### Security

- **Removed `MAX_CONCURRENT_SESSIONS` env var.** Because customers self-host the bridge and have full access to their `.env` file, storing the session limit there allowed trivial bypass. The limit is now authoritative from the license server's policy check response only. Customers cannot exceed the limit permitted by their active plan regardless of any local configuration.

### Changed

- `CodexRuntimeMetrics` type extended with `activeSessions: number` and `sessionLimit: number`.
- `applyValidToken()` and `enterPendingActivation()` updated to initialise `maxConcurrentSessions` (preserving existing value on token refresh; resetting to `1` on re-activation).

---

## [1.2.0] – 2026-05-19

### Added

- **License enforcement — device authorization flow (RFC 8628)**:
  - On first use, customers are directed to `GET /` — an inline HTML activation page.
  - Clicking "Activate" opens `https://your-license-server.example.com/activate?code=...` in a new tab.
  - The app polls the backend every 3 seconds until the session is approved or times out (5-minute window).
  - On approval, an Ed25519-signed JWT is received, verified locally, and saved to `/data/.codex-bridge-session.json`.
  - All non-activation endpoints return `HTTP 403` with `{ error: "license_required", activationPage: "/" }` until activation succeeds.
  - `GET /v1/license/status` and `GET /` are always reachable (no auth required).
  - `POST /v1/license/session/start` starts a new device flow and returns the activation URL.
- **Grace period** — if the SaaS backend is unreachable after a successful session, the app remains operational for up to `LICENSE_GRACE_HOURS` (default 72 h). Status shows `gracePeriodActive: true`.
- **Session persistence** — JWT and refresh token are saved in `/data/.codex-bridge-session.json` (mode 600). On restart, the session is reloaded; expired JWTs are refreshed automatically.
- **Periodic token refresh** — every `LICENSE_CHECK_INTERVAL_HOURS` (default 12 h) the JWT is silently renewed via `POST {SAAS_URL}/api/device/refresh`. Revoked sessions return 401 and trigger re-activation.
- **C++ native addon** (`build/Release/license_verify.node`) — Ed25519 JWT verification compiled into a native binary using `node-addon-api` + OpenSSL EVP API:
  - XOR-obfuscated public key (not visible as plaintext in the binary).
  - Verifies signature, expiry, and device fingerprint in compiled C++ code.
  - Built inside the Docker container via `node-gyp rebuild` during image build.
- **Bytenode V8 bytecode protection** — `scripts/protect.js` compiles `license.service.js` and `license.guard.js` into `.jsc` bytecode files during Docker build, then replaces the `.js` files with 2-line stubs. Patching the license logic requires C++ reverse-engineering skills.
- **Device fingerprint binding** — JWT payload includes `device: sha256(hostname:platform:arch)`. Sessions created on one machine are rejected on another.
- **`@SkipLicenseCheck()` decorator** — marks routes that bypass the global license guard (activation page, license status endpoint).
- **`LICENSE_ENABLED` env var** — set to `false` to disable enforcement entirely (dev/personal use).
- **`SAAS_URL` env var** — defaults to the URL compiled into the native addon at build time. Configurable for local testing; security is not weakened because the C++ addon verifies the JWT Ed25519 signature regardless of origin.
- **Test backend** (`test-backend/`) — minimal Express server implementing the full SaaS contract for local development:
  - `GET /activate` — HTML approval page with plan selector and customer ID field.
  - `GET /api/device/poll?code=` — returns `pending / approved / denied`.
  - `POST /api/device/refresh` — issues a renewed JWT or 401 on revoked sessions.
  - `GET /sessions` / `DELETE /sessions/:prefix` — admin session list and revocation.
  - `TOKEN_TTL_SEC` env var (default 120) for rapid expiry testing.
- **`test-backend/docker-compose.test.yml`** — compose overlay that adds the test backend as `mock-saas` (network alias `license.test.internal`), sets `LICENSE_ENABLED=true`, and exposes port 3001 to the host:
  ```
  docker compose -f docker-compose.yml -f test-backend/docker-compose.test.yml up -d --build
  ```
- **License state in health response** — `GET /v1/health?details=true` now includes a `license:` block with status, plan, expiry, and grace period info.
- **`node-addon-api`** and **`bytenode`** added to dependencies; `node-gyp` added to devDependencies.

### Changed

- `SAAS_URL` is now read from env (was hardcoded). Default is the URL compiled into the native addon at build time. Security is unchanged — the Ed25519 signature check prevents token forgery regardless of server URL.

---

## [1.1.0] – 2026-05-18

### Added

- **Dual-backend support** — select `BACKEND=codex` or `BACKEND=claude` in `.env` (or `docker-compose.yml`). No image rebuild required to switch.
- **Claude Code CLI backend** (`BACKEND=claude`):
  - Non-streaming chat via `claude --print --output-format text`.
  - Streaming chat via `claude --print --output-format stream-json` with line-buffered NDJSON parsing.
  - JSON schema output injected into the prompt text (Claude CLI has no `--output-schema` flag).
  - Backend-aware model selection via `CLAUDE_DEFAULT_MODEL` and `CLAUDE_ALLOWED_MODELS` env vars.
  - Auth status detected by checking `$HOME/.claude/.credentials.json` — no tokens spent on health checks.
- **Health endpoint improvements**:
  - `codex:` field renamed to `cli:` to correctly reflect whichever backend is active.
  - `cli.backend` sub-field indicates `"codex"` or `"claude"`.
  - Config section hides Codex-only fields (`sandbox`, `searchAllowed`) when Claude is active.
  - Two detailed Swagger health examples: `detailed-codex` and `detailed-claude`.
- **CORS configuration** — `CORS_ORIGINS` env var accepts a comma-separated list of allowed origins. Empty value allows all origins (default).
- **Multiple Swagger servers** — `API_DOCS_SERVERS` accepts a comma-separated list of base URLs, populating the server dropdown in Swagger UI for both local and production access.
- **Dockerfile** installs both CLIs in the same image (`@openai/codex` and `@anthropic-ai/claude-code`), with runtime backend selection via env var.
- **Watchdog false-positive fix for Claude** — watchdog now checks `loggedIn === false` (explicit boolean) rather than `status !== "logged_in"`, preventing false alerts when Claude's auth state is `null` (not applicable).
- **Insight reports renamed** — `InsightReport.codex` field renamed to `cli`; email text label updated from `Codex:` to `CLI:` to match the active backend.
- **Env vars added** to `.env` and `.env.example`:
  - `BACKEND` — selects the active CLI backend.
  - `CLAUDE_BIN` — path to the Claude Code CLI binary.
  - `CLAUDE_DEFAULT_MODEL` — default model for Claude backend.
  - `CLAUDE_ALLOWED_MODELS` — comma-separated model allowlist for Claude backend.
- **Documentation** — all files in `docs/` updated to cover dual-backend setup, Claude auth, port 3900, CORS, and Swagger access.

### Fixed

- Docker port mapping corrected from `3900:3000` to `3900:3900`.
- `proxy-net` Docker network documented as requiring manual creation (`docker network create proxy-net`) before first `docker compose up`.

---

## [1.0.0] – initial release

### Added

- NestJS HTTP gateway wrapping the OpenAI Codex CLI.
- OpenAI-compatible chat completion endpoint (`POST /v1/chat/completions`) with non-streaming and streaming (`stream: true`) support.
- JSON schema enforcement via `--output-schema` Codex CLI flag.
- Image upload endpoint (`POST /v1/images/upload`) with base64 pass-through to Codex.
- Health endpoint (`GET /v1/health`) with optional verbose details.
- Models endpoint (`GET /v1/models`) with live cache refresh.
- Bearer-token authentication via `BRIDGE_TOKEN` env var.
- Insights scheduler — periodic AI-generated status reports with optional SMTP email delivery.
- Watchdog service — self-healing loop that monitors failure rate and consecutive unhealthy checks.
- Token counting and input/output limits via tiktoken.
- Auto-summarisation of long conversations when token threshold is exceeded.
- Swagger / OpenAPI docs at `/api-docs` with configurable server list.
- Docker Compose setup with persistent `/data` volume for CLI auth and data storage.
- `.env.example` documenting all configuration options.
