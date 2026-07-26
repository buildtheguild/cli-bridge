# Changelog

All notable changes to this project are documented here.

---

## [2.1.1] – 2026-07-26

Mobile-responsive fixes for the operator console. The dashboard nav and activation wizard worked fine on desktop/tablet but were never actually exercised at phone width until now.

### Fixed

- **Topbar cramped on mobile** — the logo, refresh, notifications, and Sign out were all fighting for space in one row below ~640px, with Sign out often clipped. The topbar now shows a hamburger menu on the left (holding Sign out), the logo/text centered in the remaining space between it and the refresh/bell icons, and the dashboard tabs' own mobile toggle is icon-only (no "Menu" label) to match.
- **Activation wizard unusable on mobile** — the step list (License/Claude/Codex) was a fixed 16rem-wide left sidebar that squeezed the main content into an unreadably narrow column with every word wrapping. Below 640px, the step list now renders as a compact horizontal row above the content instead, and the whole screen scrolls normally instead of using independently-scrolling panes.
- **Wizard step list rendered as a staircase in the new horizontal layout** — the underlying stepper library strips top padding from the first step and bottom padding from the last, which is correct for a vertical list but pushed the first/last items up and down once forced into a row. Padding is now reset symmetrically for all three items on mobile.
- **Wizard footer (Previous/Next) scrolled away with the page** on mobile — it's now pinned to the bottom of the viewport, compacted, with enough reserved space in the content area so it never covers the last bit of text.

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

- **A false positive could revoke a still-valid license after a transient backend hiccup** — a still-paying customer could be forced into a full re-activation by a one-off blip in the license server's entitlement check. Enforcement now tolerates a brief, isolated failure instead of revoking instantly, while a genuine cancellation is still enforced.

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
- **License activation session no longer times out** while waiting for approval — the wizard no longer dead-ends back at "no active license found" if checkout takes a while.
- **Cancel button on the activation screen** — lets an in-progress activation be deliberately aborted instead of waiting it out.
- **Auto-open activation/login URLs** — starting license activation or a Codex/Claude login now opens the approval/login URL in a new tab automatically. Falls back to the existing "Open" button + copy-link if the tab is blocked by a popup blocker.
- **Manual license refresh from the wizard** — added an on-demand refresh action instead of relying solely on the periodic background check.

### Fixed

- **Cancelled subscriptions were not being enforced, and the displayed expiry could be inaccurate** — the bridge could keep reporting a valid license after a subscription was cancelled, and the expiry date shown wasn't reliably tied to the actual billing period. Both are now corrected so cancellation is enforced and the displayed expiry reflects the real subscription period.
- **Watchdog restart loop flooding email** — `WATCHDOG_RESTART_ON_CRITICAL=true` treated "not logged in" as a critical condition and restarted the process, but a restart can never restore a credential that was never there — it just crash-looped roughly every 3 minutes. Each restart re-triggered `INSIGHTS_RUN_ON_START`'s startup email with no cooldown, flooding the configured inbox. The watchdog now only restarts for conditions a restart can plausibly fix (idle timeout, high failure rate); "not logged in" still surfaces in status/alerts but never triggers a process exit.
- **Insight emails now have a cooldown floor** — added `INSIGHTS_EMAIL_COOLDOWN_MIN` (default 15 minutes) as defense-in-depth so no future rapid-restart scenario can flood the inbox, independent of the watchdog fix above.

### Changed

- **License active card recolored** — the "License active" badge icon and days-remaining progress bar now use the app's purple brand color instead of green, matching the rest of the console.

### Infrastructure

- **Dockerfile no longer reinstalls dependencies on every publish** — build arguments that change on every release were being declared too early in the Dockerfile, busting Docker's layer cache for the CLI installs and production dependency install on every single build even when nothing else changed. Those arguments are now declared immediately before the step that consumes them, so rebuilding with unchanged dependencies and CLI versions reuses cache instead of redoing them from scratch.

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

- Fixed Docker Hub bridge images that were incompatible with the current SaaS license tokens, which could leave the bridge stuck in `pending_activation` or `activating` even after a successful browser approval.

### Changed

- Clarified in code and docs that the SaaS backend is authoritative for license refresh and entitlement checks.
- Switched release documentation to `docker buildx` multi-platform publishing as the standard path.
- Documented the known release pitfall from `1.4.0-claude` so it is not repeated.

### Verification

- Reproduced the bug locally with `thebuildguild/cli-bridge:1.4.0-claude` and confirmed the patched source resolves it against the current SaaS backend.

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

- **Strengthened license activation trust boundaries** — closed a gap that allowed local configuration to influence license server trust at runtime, and tightened session restoration after restart to require a fresh online check.

### Changed

- **Docker deployment model**:
  - `docker-compose.yml` is now image-only and uses `CLI_BRIDGE_IMAGE`
  - normal operator flow is `docker compose pull && docker compose up -d`
  - release image publishing is documented separately via manual `docker build` / `docker push`
- **Pinned bundled CLI versions in Docker image builds**:
  - `@openai/codex@0.139.0`
  - `@anthropic-ai/claude-code@2.1.173`

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
- **Concurrency limit now sourced from the license plan** — changing the plan in the admin portal takes effect automatically on the next scheduled license check.

### Security

- **Concurrency limits are now enforced authoritatively by the license server**, not local configuration, so a self-hosted deployment cannot be configured to exceed the limit permitted by its active plan.

---

## [1.2.0] – 2026-05-19

### Added

- **License enforcement** — introduced device-based license activation with periodic online entitlement checks. Unlicensed installations are restricted to the activation flow until approved.
- **Grace period** — the bridge tolerates a temporary SaaS backend outage after a successful activation rather than failing immediately.
- **License status in health response** — `GET /v1/health?details=true` now includes basic license status, plan, and expiry information.

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
