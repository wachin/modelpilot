# Audit — FreeLLMAPI (reference project)

**Audited revision:** working copy at `external/freellmapi`, HEAD = `fb07845f`
("fix(cloudflare): meter Workers AI neurons so the free 10K/day stays free (#1817)").
**Upstream:** https://github.com/tashfeenahmed/freellmapi
**Audit date:** 2025-11-05
**License of the reference:** MIT — `Copyright (c) 2026 Tashfeen Ahmed` (see `external/freellmapi/LICENSE`).

This document is a source-level audit, not a description of the README marketing. Every claim
below was verified by reading the file paths cited next to it.

---

## 1. License review and attribution obligations

| Item | Finding |
| --- | --- |
| License file present | Yes, `external/freellmapi/LICENSE`, MIT. |
| Copyright holder | `Copyright (c) 2026 Tashfeen Ahmed` |
| Sub-package licenses | `cli/LICENSE` also declared (MIT). |
| Copyleft / share-alike | None. MIT is permissive. |
| Required notice | The copyright notice and the permission notice must be included in all copies or substantial portions of the Software. |

**Consequence for ModelPilot.** ModelPilot is a **clean-room, independently designed**
implementation. We do not copy source files from FreeLLMAPI. We study its architecture and
adopt *generic engineering patterns* (an adapter registry, AES-GCM credential encryption,
windowed rate-limit counters, an OpenAI-compatible proxy surface) that are not themselves
copyrightable and that we implement against our own design goals.

Where an idea, constant, or data shape is genuinely derivative, it must be attributed.

Current attribution state: **no code is reused.** Therefore:

- No MIT notice is required inside `src/`.
- `THIRD_PARTY_NOTICES.md` records the reference project, its license, and what we learned
  from it, so the provenance of design decisions is auditable.
- **Rule for all future work:** the moment any FreeLLMAPI source is copied verbatim (even a
  small helper), that file MUST carry the upstream MIT header, `THIRD_PARTY_NOTICES.md`
  MUST be updated, and the commit message MUST say so. Until then, do not paste upstream code.

Also noted: the reference's `docs/` repeatedly states that provider free tiers are not
guaranteed ("subject to change", "promotional availability is catalog-managed"). ModelPilot
treats every quota number as **unverified unless the provider itself reported it**.

---

## 2. Verified repository structure

Monorepo, npm workspaces (`package.json` → `shared`, `server`, `client`, `cli`).
~810 `.ts` + ~130 `.tsx` files. Node `>=20.18 <25`, `.nvmrc` = 22.

```
shared/types.ts                      Domain types + the Platform union
server/src/providers/                ~57 adapters (base.ts, index.ts, openai-compat.ts, google.ts, ...)
server/src/services/                 ~50 services (router, ratelimit, provider-quota, scoring, ...)
server/src/routes/                   HTTP surface (proxy, keys, models, analytics, fallback, logs, ...)
server/src/db/migrations/            ~57 timestamped, reversible SQL migrations
server/src/lib/                      cross-cutting helpers (crypto, credential, served-model, error-classify, ...)
server/src/middleware/               auth, rateLimit, allowedHosts, errorHandler
client/src/                          React 19 + Vite dashboard (shadcn/ui, TanStack Query, Recharts)
cli/src/                             setup generators + config-file merge
desktop/                             Electron wrapper (electron-builder)
```

Measured size of the core (from `wc -l server/src/services/*.ts`): **~21,651 lines** across
services alone; `router.ts` is 2,562 lines and `ratelimit.ts` 1,619 lines. This is the single
most important datum in this audit — see §7.

---

## 3. Patterns worth learning from (verified)

### 3.1 Provider adapter contract

`server/src/providers/base.ts` — `abstract class BaseProvider` declares:

- `platform`, `name` (readonly) — identity.
- `chatCompletion()`, `streamChatCompletion()` — chat, buffered + streaming.
- `validateKey()` → `boolean | { valid: false, error }` — connection testing.
- `fetchQuota()` — optional provider-reported quota probe; default does **no network** and
  returns `false`. A separate getter `hasQuotaProbe` exists because `OpenAICompatProvider`
  overrides `fetchQuota` for every instance whether or not a probe spec exists.
- `supportsKeyValidation` / `supportsKeyPolling` — opt-outs so health checks do not burn
  metered quota or trip abuse detection (LLMTR, Exa, SerpSearch).
- `keyless` flag for anonymous providers (sentinel key row so routing still sees them
  "configured").

**Lesson taken:** the adapter interface should carry *capability flags*, not just methods,
because providers differ in what they permit (probing, polling, parallel tool calls).

`server/src/providers/index.ts` keeps a `Map<Platform, BaseProvider>` and a `register()`
function; `OpenAICompatProvider` is configured by options (`platform`, `name`, `baseUrl`,
`extraHeaders`, `validateUrl`, `timeoutMs`, `keyless`, `forceSingleToolCall`, `quotaProbe`).
Groq and Cerebras are ~5 lines each. **This is the right shape and ModelPilot adopts it.**

### 3.2 Credential encryption

`server/src/lib/crypto.ts` — AES-256-GCM, 32-byte key, hex-encoded (64 chars).

- Key precedence: `ENCRYPTION_KEY` env → key file next to the DB → legacy `settings` row →
  freshly generated (dev only).
- The dev-fallback key lives in `.encryption-key` **beside, not inside, the database**, on
  the explicit reasoning that storing a key next to its ciphertext protects nothing if the
  DB is copied. `mode 0o600`, written temp-file-then-rename so it is never briefly
  world-readable.
- Length/hex validation up front with an actionable error message.
- Production refuses to boot without an explicit key.

`server/src/services/provider-credential.ts` shows the decrypt-at-use discipline: keys are
decrypted in memory per request, never held in a long-lived plaintext structure, and
consumed with a `release()` callback for budget reservations.

**Lesson taken:** copy the *properties* (separate key artifact, atomic chmod-600 write,
fail-fast validation, decrypt-at-use). ModelPilot implements its own version with scrypt
key derivation from a passphrase, which the reference does not do.

### 3.3 Response metadata / transparency headers

This is the most directly relevant part of the audit. Verified in
`server/src/routes/proxy.ts`:

```
res.setHeader('X-Routed-Via', routedViaValue(route.platform, route.modelId));   // streaming + non-stream
res.setHeader('X-Provider',   safeHeaderValue(result.platform));                // media routes
res.setHeader('X-Model',      safeHeaderValue(result.modelId));                 // media routes
res.setHeader('X-Request-ID', requestGroupId);
```

and in `server/src/db/migrations/20260726_000005_request_served_model.ts`:

> "The proxy overwrites the upstream `model` field with the routed id before the caller sees
> it (the gateway contract), which also destroyed the only evidence when a provider silently
> serves a DIFFERENT model than requested (#534 — verified live: kilo-auto/small serving
> Gemma)."

`server/src/lib/served-model.ts` implements `observeServedModel()`: it captures the raw
upstream `model` string *before* the overwrite, normalizes cosmetic differences
(case, one `:free`-style tier suffix, last `/` segment, separator runs), and persists only
genuine drift into `requests.served_model`. A structured `[ServedModel]` warning is logged once
per `(platform, requested, served)` tuple, with a bounded cache (1000 tuples) so a hostile
upstream cannot grow it without limit. `PLACEHOLDER_MODEL_VALUES = {'', 'default', 'unknown'}`
avoids comparing meaningless values.

**This is exactly the requirement ModelPilot is built around, and in the reference it was
added reactively, after a live incident, as a forensic side-channel.** Also from the same
migration comment: the response `model` field is still overwritten with the routed id, so *the
client still sees the alias, not the real served model.* That is the gap ModelPilot closes by
design (§5).

### 3.4 Request attribution

`server/src/db/migrations/20260913_000001_request_model_attribution.ts` adds
`requests.model_db_id` so history points at a concrete `models` row, because
`(platform, model_id)` is not unique once custom relays exist. Explicit design note:
"no foreign key on models(id) — request history must survive the user deleting the model
row that once served it", and unattributable custom requests stay `NULL` rather than being
mis-attributed to the wrong relay.

**Lesson taken:** historical attribution must be *frozen at write time* and must prefer
`NULL` over a guess. ModelPilot adopts both rules.

### 3.5 Quota tracking with provenance

`server/src/services/provider-quota.ts` records `provider_quota_observations` rows and
distinguishes observation sources; `services/ratelimit.ts` (1,619 lines) enforces
RPM/RPD/TPM/TPD plus provider-wide caps against a persistent ledger; `router.ts` asks
`canMakeRequest` / `canUseTokens` / `isOnCooldown` / `canUseProvider*` before dispatch.

Cooldowns: `base.ts` parses back-off from three sources — `Retry-After` header (delta-seconds
or HTTP-date), structured body fields (`google.rpc.RetryInfo.retryDelay` = `"17s"`, or bare
seconds under `retryDelay`/`retryAfter`/`retryAfterSeconds`), and finally prose
(`"try again in 1m26s"`). Every value is clamped to `MAX_RETRY_AFTER_MS = 24h` because the
delay feeds cooldown expiry directly and a malformed/hostile value would otherwise bench a key
effectively forever. A depth-capped recursive search prevents a pathological body from spinning.

Error classification lives in `lib/error-classify.ts` with `TIMEOUT_ERROR_MARKERS`.

**Lesson taken:** (a) always clamp provider-stated delays; (b) a stated field is a promise, a
prose sentence is an observation — parse them differently and rank them differently;
(c) timeouts count as reliability failures *and* as speed samples with zero tokens.

### 3.6 Routing engine (and why we diverge)

`server/src/services/router.ts` + `scoring.ts`. Not round-robin: a **contextual bandit**.
`scoring.ts` documents the formula:

```
base      = w_rel·reliability + w_speed·speed + w_intel·intelligence     (convex weights)
effective = base × headroomFactor × rateLimitFactor                      (guardrail multipliers)
```

Reliability is Thompson sampling over Beta posteriors fed by decay-weighted samples
(7-day window, 2-day half-life: `weight = 0.5^(age_days/2)`). Strategies:
`balanced` (default, 0.5/0.25/0.25), `smartest`, `fastest`, `reliable`, `custom`, `priority`,
`decision`, plus non-Manual `auto:<axis>` ordering.

Chain sources, in priority order: active profile (`profile_models`) → global
`fallback_config` → sort alias `auto:smart|fast|cheap|reliable|balanced` → profile by name
`auto:<name>`. `match_tier` is an outer sort key that dominates score so a group-resolved
slug never outranks a direct `model_id` hit. Requests fail over across **up to 20 attempts**
inside a wall-clock budget. `RouteError.diagnostics` collects one reason line per considered
model for the "all exhausted" case.

**This is genuinely impressive engineering — and it is the thing ModelPilot deliberately
does NOT copy as the default.** A bandit that re-picks `(platform, model, key)` every request
is, by construction, *model opacity*. For a coding agent mid-task, an invisible swap between
attempt 3 and attempt 4 is a correctness hazard, not a feature. See §5.

### 3.7 Observability

`server/src/db/migrations/20260823_000001_server_logs.ts` (two-tier log store, live panel,
polling API), `lib/log-redaction.ts` + `lib/error-redaction.ts` (never log secrets),
`lib/request-log.ts`, `lib/attempt-trace.ts`, `routes/analytics.ts` (exposes
`requested_model`, `served_model`, latency percentiles via a dedicated index migration).

**Lesson taken:** redaction must be a library concern applied at the logging boundary, not a
reviewer habit. Latency percentile indexes belong in the schema from the start.

### 3.8 Client compatibility surface

`opencode.json` in the repo root is a working OpenCode config pointing at
`http://127.0.0.1:3001/v1` with `model: freellmapi/auto`. `cli/src/` contains `setup-*`
generators and `config-files.ts` (merge, not clobber). `docs/en/clients/` documents base URLs
per agent, headerless URL tokens, an Ollama emulation surface (`routes/ollama.ts`), and an MCP
server (`routes/mcp.ts`). `routes/anthropic.ts` exposes `/v1/messages`.

**Lesson taken:** ship a machine-readable agent-config snippet per tool and test it; do not
claim compatibility untested.

### 3.9 Testing discipline

`server/vitest.config.ts` runs `--pool=forks --fileParallelism=false` (SQLite + shared process
state). `server/src/__tests__/` is organized by area (`db/`, `routes/`, `services/`,
`providers/`) with ~200 test files, including narrowly-scoped regression tests named after
the incident they prevent (`ratelimit-cooldown-ceiling`, `ratelimit-daily-boundary`,
`router-key-bandit`, `served-nonchat-marks-key-healthy`, ...). Tests use `msw` for transport
mocking and an in-memory DB.

**Lesson taken:** name regression tests after the failure mode, not the function. Use an
in-memory DB and a mock transport so no test ever touches a real paid endpoint.

---

## 4. Limitations and weaknesses observed

1. **The bandit resets model identity constantly.** Failover across up to 20 attempts with
   per-request re-selection means the model that generated *this* response may be the 4th
   choice, and nothing in the default response body says so. Transparency is bolted on
   (`X-Routed-Via`, `served_model`) rather than being the selection contract.
2. **The response `model` field is overwritten** with the routed id before the client sees it.
   Callers therefore cannot detect drift from the payload; they must know to read a custom
   header or query the analytics DB. Issue #534 had to be fixed with a forensic column.
3. **Drift is detected but not surfaced to the user as a decision.** A `[ServedModel]` warn and
   a DB column are diagnostics; there is no in-UI "the provider served something else" event
   with a choice for the operator.
4. **Drift reporting is lossy by design** — `served_model` is stored only when it differs after
   normalization, and only the *first* differing tuple per process is logged (bounded cache).
   Useful for alerting, insufficient as an audit trail.
5. **Complexity is very high for the stated goal.** 2,562-line router, 1,619-line ratelimit,
   1,284-line catalog-sync, ~57 migrations, ~57 adapters, 4 README translations, ~200 test
   files. A new contributor cannot hold the routing semantics in their head. The 20-attempt
   failover budget multiplies the state space that tests must cover.
6. **Provider breadth over verification depth.** ~34 advertised providers, many of them small
   promotional gateways ("catalog-managed", "promotional availability"). Several were removed
   over time (Moonshot, MiniMax, GitHub, Auray — see the `20261017_000002_remove_github_provider`
   and `20261019_000001_remove_auray_provider` migrations). Breadth creates a maintenance
   treadmill and makes "is this integration actually working?" hard to answer.
7. **Quota truth is mixed in one place.** The code does distinguish observation provenance,
   but the README ships headline numbers like "7.4 billion tokens/month of listed capacity",
   which invites users to read catalog *claims* as guarantees — precisely what the project's
   own docs warn against.
8. **Most catalog data is remote.** Since the June 2026 "Premium live catalog", model/limit
   *data* is delivered by a signed hosted catalog (`services/catalog-sync.ts`, gated on
   `hasProvider`). `20260101_000000_legacy_baseline.ts` states V25 is the last model-data
   migration. A local-first tool that depends on a remote catalog for its model metadata is an
   availability and trust dependency ModelPilot should avoid.
9. **Free-tier mutability is structural.** Providers revoke, gate, or promote tiers constantly.
   Any design that assumes a stable free quota will rot.

---

## 5. What ModelPilot must do differently (the core divergence)

FreeLLMAPI's implicit contract is **"give me the best available free model right now."**
ModelPilot's contract is **"tell me, verifiably, which exact provider+model produced this
response — and never change it behind my back."**

Concretely, ModelPilot inverts these defaults:

| Dimension | FreeLLMAPI | ModelPilot |
| --- | --- | --- |
| Per-request model choice | Bandit re-selects each request | **Sticky**: a session resolves a model once and keeps it |
| Mid-session model change | Silent failover, up to 20 attempts | **Refused by default.** `model_locked` error carrying a choice payload |
| Response `model` field | Overwritten with routed/alias id | **Echoes the model that actually served**, requested id kept in metadata |
| Truth channel | Custom header + DB column | Header **and** metadata **and** `X-Model-Source: requested\|fallback` **and** log **and** Activity row |
| Alias handling | Normalize away, store NULL | **Explicitly mapped or marked `unverified`; never invented** |
| Quota numbers | Catalog claims + observations | Five-level provenance enum; estimates never rendered as reported |
| Model metadata | Remote signed catalog | Local, versioned, seeded from the repo, offline-first |
| Provider count | ~34, breadth-first | **Hand-picked, each one tested; no unverified integrations** |
| Reset policy | "resets at UTC midnight" in places | Per-provider documented policy + timezone carried as data |

The single most important rule: **a response is only attributed to a model if ModelPilot can
justify that attribution.** If the upstream echoed a placeholder (`default`, empty, `unknown`)
or echoed an alias that we cannot resolve to a concrete id, the response is marked
`unverified` — we do not fall back to "it was probably what we asked for."

---

## 6. Concrete technical decisions inherited from this audit

Adopted (implemented independently in ModelPilot):

- Adapter registry keyed by provider id, with capability flags on the adapter.
- One `OpenAICompatAdapter` configured by options, reused by most providers.
- AES-256-GCM credential encryption with a key artifact separate from the DB, atomic
  chmod-600 write, fail-fast validation, decrypt-at-use.
- `Retry-After` / structured `retryDelay` / prose parsing, all clamped (24h).
- Windowed RPM/RPD/TPM/TPD counters over a persistent ledger.
- Request rows that freeze provider+model identity at write time and prefer "unverified" over
  a guess.
- Redaction at the logging boundary.
- Mock-transport tests, in-memory DB, regression tests named after failure modes.

Deliberately rejected:

- Bandit / Thompson-sampling routing as the *default*. ModelPilot's router is deterministic
  and explicable: explicit preference order → eligibility filter → score → pick one. No
  per-request randomization. A user must be able to read a decision and reproduce it.
- Overwriting the caller's `model` field with a gateway-internal id.
- Remote catalog as the source of truth for model metadata.
- 20-attempt invisible failover.
- Shipping adapters that have not been tested against a real endpoint.

---

## 7. Size and complexity benchmark

The reference's own core is the strongest argument for ModelPilot's staging discipline.
ModelPilot's Phase 2 budget is a **single-process FastAPI app with a handful of modules**:

- `router.py` — deterministic, < 300 lines
- `quota.py` — windowed counters + reset policies, < 400 lines
- `catalog.py` — local model metadata + capability flags, < 300 lines
- `adapters/` — one file per provider, ~60–120 lines each
- `crypto.py` / `credentials.py` — < 200 lines

If a module starts approaching the reference's line counts, that is a signal to split the
module, not to celebrate the feature count.

---

## 8. Open questions carried into Phase 2

1. **Session identity.** Coding agents do not always send a conversation id. ModelPilot must
   derive session stickiness from something reliable — candidate keys: the unified API key plus
   a caller-supplied `X-ModelPilot-Session` header, else a fingerprint of the first system
   message, else the `X-Request-ID` chain. The fallback must be documented, because getting it
   wrong means either unwanted sticky models or accidental swaps.
2. **`model` field semantics on a fallback.** When a fallback *is* permitted and occurs, do we
   echo the fallback id in `model` (OpenAI-honest) or the requested id (client-compatible)?
   Decision for Phase 2: echo the **actual** serving model, and put the requested id in
   `metadata.modelpilot.requested_model`, so the payload itself cannot lie.
3. **Embedding/model-listing parity** — `/v1/models` must list only models that are actually
   reachable with current credentials, or clearly mark the rest.
4. **Key rotation UX** — provider keys rotate; the re-encrypt path must be a documented,
   tested operation, not a manual DB edit.

---

## 9. Sources read for this audit

- `external/freellmapi/LICENSE`, `package.json`, `.nvmrc`
- `shared/types.ts` (`Platform` union, `Model`, `ApiKey`, `KeyStatus`, `ApiKeyCooldown`)
- `server/src/providers/base.ts`, `openai-compat.ts`, `index.ts`
- `server/src/services/provider-credential.ts`, `provider-quota.ts`, `router.ts`, `scoring.ts`
- `server/src/lib/crypto.ts`, `served-model.ts`
- `server/src/routes/proxy.ts` (headers, media routes)
- `server/src/db/migrations/20260101_000000_legacy_baseline.ts`,
  `20260726_000005_request_served_model.ts`, `20260913_000001_request_model_attribution.ts`
- `docs/en/OVERVIEW.md`, `docs/en/architecture/*`, `docs/en/providers/OVERVIEW.md`,
  `docs/en/clients/OVERVIEW.md`, `opencode.json`
- `client/package.json`, `server/package.json`
