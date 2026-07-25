# Changelog

All notable changes to this project are documented here.

---

## [2.1.0] – 2026-07-25

Follow-up release on top of `2.0.0`'s Whisper support: turns the single baked-in model into a full model-management system (download/switch/delete from the dashboard or API, per-request override), adds cancellation for both downloads and in-flight transcriptions, adds a Logs tab for diagnosing failures, and fixes several bugs found while exercising all of that against real hardware and real network conditions.

### Added

- **In-dashboard Whisper model switching** — `GET /v1/audio/models` lists a curated catalog (`tiny`/`base`/`small`/`medium`/`large-v3-turbo`/`large-v3`) with size and downloaded/active state; `POST /v1/audio/models/:name/select` switches immediately if the model is already available, or downloads it from https://huggingface.co/ggerganov/whisper.cpp straight to the persistent `/data` volume and switches automatically once done (`GET /v1/audio/models` reports live progress for polling); `DELETE /v1/audio/models/:name` frees disk space. Downloaded models and the active selection live on `/data`, so they survive container recreation and image updates — `WHISPER_MODEL_PATH` is now only the fallback used before any selection has been made. The dashboard's Whisper card gained a model dropdown with a live download progress bar and a confirmation modal before deleting.
- **Per-request model override on `POST /v1/audio/transcriptions`** — the `model` field (previously accepted-but-ignored, only there for OpenAI-client compatibility) now actually selects which downloaded model transcribes that one request, without changing the dashboard's globally active ("default") model. Only accepts already-downloaded catalog names — never triggers a download mid-request, since that could block the caller for minutes; an undownloaded or unknown model returns a clear `400` instead. Omitting `model` (or sending OpenAI's fixed `"whisper-1"` placeholder) uses whichever model is currently active. Swagger now shows this as a dropdown of catalog names instead of a free-text field marked "ignored."
- **`POST /v1/audio/cancel`** — aborts whatever transcription request(s) are currently running (via Node's native `AbortSignal` support on the spawned `ffmpeg`/`whisper-cli` child processes), freeing the capacity slot immediately instead of waiting out `WHISPER_TRANSCRIBE_TIMEOUT_MS` (now up to 10 minutes by default). The cancelled request itself receives a clear `400 "Transcription was cancelled."` response rather than a timeout error. The dashboard's Capacity tile shows a Cancel button whenever a job is active.
- **`POST /v1/audio/models/cancel`** — aborts an in-progress model download (cleans up the partial file, leaves the previously-active model unchanged) instead of the dashboard just disabling the picker until it finished or failed on its own.
- **`WHISPER_TRANSCRIBE_TIMEOUT_MS`** — the actual whisper-cli inference step now has its own, far more generous timeout (default 600000ms/10min) separate from `WHISPER_TIMEOUT_MS` (now decode-only, still 120000ms). Transcription time scales with both audio length and model size; a larger/more-accurate model transcribing 1-2 minutes of audio can take several minutes on modest CPU hardware, and reusing the short decode timeout for that was cutting off legitimate large-model transcriptions with `whisper-cli timed out after 120000ms` rather than just catching genuine hangs.
- **`GET /v1/logs`** — new in-memory ring buffer (last 200 entries) of notable operational events, separate from `docker logs`/stdout so an operator can see *why* something failed straight from the dashboard instead of just a bare failure count. Wired into the Whisper failure points: decode failures, missing-model errors, whisper-cli failures/timeouts (including the exact `whisper-cli timed out after ...` message), and model-download failures/retries. Resets on restart — it's a recent-events view, not persistent log storage. The dashboard has a new "Logs" tab showing these as a scrollable table (time/level/source/message), polled on the existing 20s cycle.
- **Mobile-responsive dashboard navigation** — the tab row (now five items: Account, Claude, Codex, Logs, API Docs) collapses into a hamburger-triggered dropdown menu below a 640px viewport width instead of relying on horizontal scroll, which was the only affordance before and didn't make it obvious more tabs existed off-screen.
- User-facing text (dashboard, Swagger docs, and API error/status messages) no longer says "local" or "whisper.cpp" — rephrased as "Whisper service" throughout, since the implementation detail isn't relevant to API consumers or dashboard operators. The dashboard's "Model" label is now "Default model," since a request can now override it per-call.

### Fixed

- **The Whisper model picker could silently disappear from the dashboard after a page reload** — `refreshDashboardData()` gated the `/v1/audio/models` fetch behind the *previous* `/v1/audio/status` call's forbidden-check, so any hiccup on one request could blank the other. The two are now fetched independently, each determining "forbidden" from its own response.
- **Large model downloads (1.5GB+) could fail with a bare `terminated` error and no retry** — a dropped connection partway through a multi-minute download is an expected, transient failure mode, not a bug. Downloads now retry up to 3 times with backoff before surfacing an error.
- **A shorter card (e.g. Claude's) left a dead gap at the bottom when a sibling card (e.g. Whisper's, with its variable-height model picker) stretched both to equal height** — card bodies are now flex columns with the trailing action block (button / model picker) pinned to the bottom via `margin-top:auto`, so the slack collects invisibly above it instead of appearing as empty space below.

---

## [2.0.0] – 2026-07-24

Major version bump: this release adds a new local transcription capability (not just a bridge to Codex/Claude) and grows the Docker image with a compiled `whisper.cpp` binary and a baked-in model — a large enough surface and image-composition change to warrant a major version rather than a minor one.

### Added

- **`POST /v1/audio/transcriptions`** — OpenAI-compatible audio transcription endpoint backed by a **100% locally-run** `whisper.cpp` — uploaded audio is decoded and transcribed entirely inside the container and is never sent to OpenAI, Anthropic, or any other external service, and there's no external API cost. Accepts the same multipart `file`/`model`/`language`/`prompt`/`response_format` shape as OpenAI's Whisper endpoint (`json`, `text`, and `verbose_json` with segment timestamps are supported), so existing Whisper clients can point at this bridge by changing only the base URL. Uploaded audio is decoded to the 16kHz mono PCM WAV whisper.cpp requires via `ffmpeg` (handles mp3/m4a/ogg/opus/webm/wav and other common voice-note formats) before transcription. A configurable concurrency guard (`WHISPER_MAX_CONCURRENT`, default 1) protects small/shared deployments from a transcription job starving concurrent chat requests. The `language` field is now a documented enum in Swagger (renders as a dropdown of every ISO-639-1 code whisper.cpp supports, `auto` first/default) instead of a freeform string. See `docs/config.md` for the full `WHISPER_*`/`FFMPEG_BIN`/`MAX_AUDIO_BYTES` env var reference, including the build-time `WHISPER_MODEL`/`WHISPER_CPP_VERSION` Dockerfile args that pick which model gets baked into the image.
- **`GET /v1/audio/status`** — ungated Whisper backend status endpoint (binary/model/ffmpeg present and working, live capacity, usage metrics), on the same pattern as `GET /v1/watchdog/status`. The console dashboard now shows a Whisper card next to the Claude/Codex CLI cards, polled on the existing 20s metrics cycle.
- **`WHISPER_LANGUAGE`** — default language used when a transcription request doesn't specify one (default `auto`). Pinning a language skips per-request detection, which can improve both speed and accuracy for single-language deployments.
- **`WHISPER_THREADS`** — pins whisper.cpp's CPU thread count per job. Left unset it auto-detects; set explicitly (e.g. `1`) to cap CPU usage on a small/shared VPS.
- **Runtime model switching without a rebuild** — `WHISPER_MODEL_PATH` can point at any `ggml-*.bin` mounted into the container (e.g. downloaded from https://huggingface.co/ggerganov/whisper.cpp onto the `/data` volume). The `WHISPER_MODEL` Dockerfile build arg only controls which model ships baked in *by default*; it was never required to rebuild just to try a different model size.
- **Whisper is plan-gated** — `POST /v1/audio/transcriptions` and `GET /v1/audio/status` now require a CLIBridge plan that includes Whisper; a plan that doesn't include it gets `403 { error: "whisper_not_entitled" }` on both routes. The dashboard's Whisper card is hidden entirely (not shown locked/grayed) when the plan doesn't include it, distinguishing a `403` from a genuine connectivity/health problem.

### Fixed

- **Claude verification-code input lost anything typed or pasted into it within ~3.5s** — the wizard's session-status poll (`pollProviderSession`) re-renders the whole step's HTML on every cycle via `setHtml`, which fully replaces the DOM node the `<input id="claude-code-input">` lives in, wiping its value and dropping focus. `renderWizard()` now captures the input's value, focus state, and cursor position immediately before that re-render and restores them immediately after, so typing or pasting a code no longer races the poll.

### Docs

- Clarified throughout (README, `docs/dockerhub.md`, `docs/config.md`, Swagger tag/operation descriptions, the app's default `APP_DESCRIPTION`) that audio transcription is fully local and never proxied to any external API — this wasn't obvious enough from the original wording.

---

## [1.6.3] – 2026-07-23

### Fixed

- **Claude backend: every request started failing with `claude failed code=1 stderr=(none)` after updating the Claude CLI to 2.1.218** — `@anthropic-ai/claude-code` raised its minimum Node requirement from `>=18.0.0` (the previously baked-in `2.1.173`) to `>=22.0.0` as of `2.1.218`, but the image's `node:20-alpine` base (both build and runtime stages) no longer met it, so the CLI failed to run and produced no diagnostic output at all. The image now builds on `node:22-alpine`, and the baked-in `CLAUDE_CODE_VERSION` build arg is bumped to `2.1.218` to match. Verified directly against a rebuilt image: `node --version` reports `v22.23.1`, and the exact `--print --output-format stream-json --verbose` invocation the bridge sends now returns a clean, well-formed `result` event instead of a silent crash.

### Infrastructure

- **`CLAUDE_CODE_VERSION` build arg bumped to `2.1.218`** — the in-container "Update" button (see 1.6.0) only patches a running container's writable layer; it doesn't change what gets baked into freshly built or redeployed images. Rebuilding with the old `CLAUDE_CODE_VERSION=2.1.173` default on the new `node:22-alpine` base would have worked too, but pinning the baked-in version to what's now current avoids immediately hitting an "Update available" banner on a fresh deploy.

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
