# ROADMAP.md — ModelPilot

**Status markers:** `[ ]` not implemented · `[~]` partially implemented · `[x]` implemented **and verified** (tests green + docs updated + manually validated where applicable).

> **Rule (non-negotiable):** a task is only `[x]` when its tests exist and pass, its docs are
> updated, and it has been run. Code existing is `[~]` at best.

Companion documents: [`ARCHITECTURE.md`](ARCHITECTURE.md) · [`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) ·
[`docs/AUDIT_FREELLMAPI.md`](docs/AUDIT_FREELLMAPI.md) · [`PROVIDERS.md`](PROVIDERS.md)

---

## Phase 1 — Audit, architecture, requirements

**Goal:** do not write application code until the architecture is agreed and the divergence
from the reference project is explicit.

- [x] Read the FreeLLMAPI reference and confirm its license (MIT, `Copyright (c) 2026 Tashfeen Ahmed`).
- [x] Source-level audit of adapters, OpenAI-compatible API, catalog, credential encryption, quota/rate-limit tracking, routing/fallback, dashboard, analytics, error handling, tests.
- [x] Document which patterns to adopt and which to reject, with file-level evidence → [`docs/AUDIT_FREELLMAPI.md`](docs/AUDIT_FREELLMAPI.md).
- [x] Choose the technology stack and record why (including rejected alternatives) → [`ARCHITECTURE.md`](ARCHITECTURE.md) §3.
- [x] Define the transparency model (`Attribution`, `ModelSource`, `Verification`) → `ARCHITECTURE.md` §2.
- [x] Define the data model encoding the invariants → `ARCHITECTURE.md` §5.
- [x] Define the deterministic routing engine → `ARCHITECTURE.md` §7.
- [x] Define the five-level quota provenance model → `ARCHITECTURE.md` §8.
- [x] Create `ROADMAP.md`, `AGENT_HANDOFF.md`, `SECURITY.md`, `PROVIDERS.md`, `CONTRIBUTING.md`,
      `THIRD_PARTY_NOTICES.md`, `LICENSE`, `.gitignore`, and `README.md`.
- [x] Repository hygiene: `.gitignore` covers secrets, DBs, venvs, caches, OS noise; `external/`
      (reference checkout) excluded from version control.

**Exit criteria met:** architecture document reviewed; no application code written before this
point; license obligations recorded; phased plan published.

**Phase 1 deviations from the brief:** none. The brief's Phase 1 matches.

---

## Phase 2 — Core application, storage, credentials, provider abstraction

**Goal:** a running process with an encrypted credential store, a real schema, and an adapter
interface — no providers yet, but the abstraction is proven by an in-repo fake adapter.

- [ ] `pyproject.toml` (PEP 621) with pinned ranges: `fastapi`, `uvicorn[standard]`, `pydantic`, `pydantic-settings`, `httpx`, `cryptography`, `typer`; dev extras: `pytest`, `pytest-asyncio`, `respx`, `ruff`.
- [ ] `src/modelpilot/infra/config.py` — layered settings (defaults → config file → env → CLI flags); explicit `data_dir`; loopback-only default bind.
- [ ] `src/modelpilot/infra/crypto.py` — AES-256-GCM encrypt/decrypt, `scrypt` KDF, key-version field, atomic `0600` key-file write, fail-fast validation with actionable errors.
- [ ] `src/modelpilot/infra/storage.py` — SQLite (WAL, `foreign_keys=ON`), `meta(schema_version)`, migration runner, typed query helpers.
- [ ] Schema v1 implementing `ARCHITECTURE.md` §5 in full (all tables, all indexes).
- [ ] `src/modelpilot/infra/credentials.py` — add / list (masked) / remove / rotate; decrypt-at-use; never returns plaintext outside the single call site.
- [ ] `src/modelpilot/infra/logging.py` — structured logging + redaction pass at the boundary; redaction registry.
- [ ] `src/modelpilot/adapters/base.py` — `Adapter`, `ModelInfo`, `ChatResult`, `Chunk`, `Usage`, `ProviderError`, `ErrorKind`, `ValidationResult`.
- [ ] `src/modelpilot/adapters/registry.py` — registration + lookup; duplicate-id detection.
- [ ] `src/modelpilot/adapters/fake.py` — in-repo fake adapter (deterministic, scriptable responses/errors) used by all later tests.
- [ ] `src/modelpilot/api/app.py` — app factory, lifespan (init storage + key), health endpoint.
- [ ] `src/modelpilot/cli.py` — `serve`, `doctor`, `credential add|list|remove|rotate`.
- [ ] **Tests:** crypto round-trip + tamper detection + wrong-passphrase failure; key-file permissions (`0600`); migration up on empty DB; credentials CRUD; plaintext never logged; `doctor` smoke; fake adapter contract test.
- [ ] Docs: installation, configuration, and credential-rotation instructions in `README.md` / `SECURITY.md`.

**Exit criteria:** `pip install -e .` then `modelpilot serve` starts; `modelpilot credential add` stores ciphertext (verifiable: the raw DB contains no plaintext key); `pytest` green; no provider adapter exists yet.

---

## Phase 3 — Initial provider integrations and connection testing

**Goal:** a small number of **verified** providers, each with a hermetic contract test. Quality
over breadth — deliberately not 30 providers.

- [ ] `adapters/openai_compat.py` — one adapter parameterized by `provider_id`, `display_name`, `default_base_url`, `extra_headers`, `error_map`. Shared by most providers.
- [ ] `adapters/groq.py` (OpenAI-compatible; free tier).
- [ ] `adapters/cerebras.py` (OpenAI-compatible; free tier).
- [ ] `adapters/nvidia_nim.py` (OpenAI-compatible; note the single-tool-call quirk).
- [ ] `adapters/google_ai_studio.py` (native `generateContent` wire format, distinct error shape).
- [ ] `adapters/mistral.py` (OpenAI-compatible).
- [ ] `adapters/openrouter.py` (OpenAI-compatible; `:free` suffix handling).
- [ ] `adapters/local_openai.py` (Ollama / LM Studio / llama.cpp / vLLM via a user-supplied base URL; no key required).
- [ ] `adapters/custom.py` — user-defined OpenAI-compatible endpoint with validated URL.
- [ ] Connection testing: `validate_key()` per adapter, exposed as `modelpilot provider test <id>`; result persisted in `providers.last_test_*`.
- [ ] Model discovery: `list_models()` where supported; upsert into `models` with `capability_source='discovery'`; never overwrite curated metadata.
- [ ] Error normalization per adapter → `ErrorKind`; `Retry-After` and structured/prose back-off parsed and **clamped to 86400 s**.
- [ ] `PROVIDERS.md` entry per provider: auth, free-tier status, discovery support, quota-probe support, quirks, and the **date the integration was last verified against the live endpoint**.
- [ ] **Tests:** one `tests/contracts/<provider>_test.py` each using `respx` fixtures — happy path, 401, 429 with `Retry-After`, 5xx, malformed body, and a **"provider lies" fixture that echoes a different model**.
- [ ] **Test:** an architectural test asserting that adding a provider does not import from `core/`.

**Exit criteria:** ≥6 providers with green contract tests; every `PROVIDERS.md` row has a verification date; no provider is claimed as supported without a passing fixture test.

**Deviation from the brief:** the brief lists Cloudflare, Hugging Face, and Cohere in Phase 3.
They move to Phase 3b because each has a materially different auth/response shape and would
dilute the "small number of verified providers" goal.

### Phase 3b — Extended providers
- [ ] `adapters/huggingface.py`
- [ ] `adapters/cloudflare.py` (account-id + token auth; document its request-unit accounting)
- [ ] `adapters/cohere.py` (native chat shape)
- [ ] Contract tests for all three.

---

## Phase 4 — Model catalog and exact model identification

**Goal:** a local, offline-capable catalog with exact ids, honest capability flags, and no
name-based guessing.

- [ ] `src/modelpilot/data/catalog.yaml` — versioned seed: per provider, exact `model_id`s, display names, context windows where documented, capability flags, aliases, and reset policy + timezone. Every entry carries a `source_url` and a `verified` date.
- [ ] `src/modelpilot/core/catalog.py` — load seed, merge with the `models` table, discovery upserts, curated-wins merge rule.
- [ ] Tri-state capability handling (`None` = unknown) end to end, including API responses.
- [ ] Suitability scores (`code_score`, `reasoning_score`, `speed_score`) set **only** from documented metadata or an explicitly cited benchmark; a name containing "coder" must never set a score. A unit test asserts this.
- [ ] Alias resolution: `model_aliases` + catalog `aliases`, with `source` recorded; an alias with no mapping resolves to `unverified`, never to a guess.
- [ ] `src/modelpilot/core/transparency.py` — `Attribution`, `normalize_model_id`, `build_attribution(...)`, and the full `Verification` decision table.
- [ ] `/v1/models` — lists only models reachable with current credentials; unreachable ones are omitted or clearly flagged as unavailable (decide + document + test).
- [ ] CLI: `modelpilot models list [--provider X] [--json]`, `modelpilot models show <provider>/<model>`.
- [ ] **Tests:** every `Verification` outcome has a case; alias mapping; placeholder handling (`''`, `default`, `unknown`); cosmetic-difference normalization (case, `:free`, `/`-prefix, separator runs); a name-based-scoring guard test.
- [ ] Docs: `docs/MODELS.md` describing the catalog format and how to add or override a model.

**Exit criteria:** a model asked for by exact id is the model reported back; an aliased or unverifiable response is marked as such; no code path derives capability from a model's display name.

---

## Phase 5 — Usage tracking, quota management, reset policies

- [ ] `src/modelpilot/core/quota.py` — window keying (`rpm`, `rpd`, `tpm`, `tpd`, `monthly`), insert-or-increment into `usage_windows`, provenance-aware `QuotaReading`.
- [ ] Reset computation from `reset_policy` + `reset_timezone` (`utc_midnight`, `pacific_midnight`, `fixed_window`, `rolling`, `calendar_month`, `unknown`); `resets_at` is `None` when the policy is unknown.
- [ ] Record observed usage from every request (requests + prompt/completion tokens) with `source='observed'`.
- [ ] Provider-reported quota probe hook: `Adapter.fetch_quota()` + `supports_quota_probe`; results stored with `provenance='provider_api'`.
- [ ] Cooldown tracking: on 429/5xx, store an expiry derived from the parsed, clamped back-off; the router skips the model until expiry.
- [ ] Quota events: `rate_limit` / `daily_exhausted` / `unknown` classification written to `events`, including the reset estimate (or `None`).
- [ ] Manual limits: `modelpilot quota set --provider X --window rpd --limit N --reset-policy ... --timezone ...` with `provenance='user_configured'`.
- [ ] CLI: `modelpilot quota show [--json]`, `modelpilot quota history --provider X --model Y`.
- [ ] **Tests:** window rollover for each reset policy (including a non-UTC timezone boundary and a DST transition); the unknown-quota path returns `UNKNOWN`, never a number; an estimate is never labelled exact; a `NULL` limit never renders as `0`; 429 classification temporary vs. daily; clamped back-off (a hostile `Retry-After: 99999999999` must not bench a key forever).
- [ ] Docs: `docs/QUOTAS.md` — the provenance model and the per-provider reset-policy table.

**Exit criteria:** every displayed quota number carries a provenance; all reset boundaries are covered by tests; an unknown quota is displayed as unknown.

---

## Phase 6 — Unified OpenAI-compatible API

**Goal:** a working `/v1` endpoint that any OpenAI client can use, with the transparency
contract enforced.

- [ ] `POST /v1/chat/completions` — non-streaming and streaming (`stream: true`).
- [ ] `POST /v1/completions` (legacy) — thin wrapper, same attribution path.
- [ ] `GET /v1/models` — OpenAI `list` shape, enriched with ModelPilot metadata.
- [ ] `POST /v1/embeddings` — only for providers/models that actually support it; otherwise a clear `400`.
- [ ] Tool/function calling pass-through, including the single-tool-call quirk flag.
- [ ] Structured output (`response_format: {type: json_schema|json_object}`) pass-through where supported.
- [ ] Unified bearer auth: `Authorization: Bearer mp-...` verified against `api_tokens.token_hash`; OpenAI-compatible error shape on failure.
- [ ] **Attribution fan-out:** `X-ModelPilot-Provider`, `-Model`, `-Model-Source`, `-Verification`, `-Fallback`, `-Request-Id` headers on every response; `metadata.modelpilot` block in non-streaming bodies; a trailing metadata frame in streams.
- [ ] Error normalization to OpenAI error shape: `{error: {message, type, param, code}}` with a `modelpilot` extension for quota/choice payloads.
- [ ] `src/modelpilot/core/session.py` — session-id derivation (header → system-message fingerprint → request-scoped), pin storage and lookup.
- [ ] **Pin enforcement:** a pinned session whose model becomes ineligible returns a `model_unavailable` error carrying alternatives; it does **not** silently switch.
- [ ] Request persistence: one `requests` row per request with frozen attribution, tokens, latency, attempts.
- [ ] **Tests:** streaming chunk shape matches OpenAI; tool-call round-trip; structured output; `/v1/models` shape; auth failure shape; **headers present on 200, 4xx, and 5xx**; a pinned session cannot be switched (written before the feature, must fail first for the right reason); tokens recorded when reported and `NULL` when not.
- [ ] Docs: `docs/API.md` — full reference including every ModelPilot-specific header and metadata field.

**Exit criteria:** the official `openai` Python SDK can complete a chat (streaming and not), list models, and call a tool against the local endpoint; every response carries verifiable attribution headers.

**Deviation from the brief:** the brief places the dashboard (its Phase 7) after the API. Order
preserved — the API must exist before a UI can display anything meaningful.

---

## Phase 7 — Dashboard and activity history

**Goal:** the transparency contract is visible without reading headers or SQL.

- [ ] `api/routes_admin.py` + static assets served by the same process (no build step).
- [ ] **Dashboard:** current active provider + model (prominent), provider connection status, available model count, models approaching quota, models with exhausted quotas, recent requests, API server status.
- [ ] **Providers page:** add / edit / enable / disable / test / remove; API keys (masked); custom base URLs; connection status; supported capabilities.
- [ ] **Models page:** provider + exact `model_id` + display name; context window (or "unknown"); capability flags with tri-state rendering; quota status; enable/disable; pin.
- [ ] **Quotas page:** per provider/model usage; daily and monthly; remaining when known; explicit "estimated" labelling; reset schedule with timezone; history.
- [ ] **Activity page:** every request with **actual provider + model**, tokens, latency, status, error; fallback events; model changes; an inspect view showing the full `Attribution` including `served_model_raw` when it differs.
- [ ] **Recommendations page:** alternatives when a model is exhausted/limited, with per-candidate reasons, availability, quota status, and one-click switching.
- [ ] Live "currently serving" indicator showing provider + model for the most recent request.
- [ ] Query API for the dashboard (`/admin/api/...`) returning the same data as the CLI `--json` output.
- [ ] **Tests:** UI rendering contract for every `QuotaProvenance` and every `Verification` value (including `UNKNOWN` and `UNVERIFIED` must render as "unverified", never as a model name); dashboard picks up a request row written by the API.
- [ ] Docs: dashboard guided tour in `README.md`.

**Exit criteria:** a user can answer "which provider and model served my last request?" from the dashboard alone, in one glance, with the evidence visible.

---

## Phase 8 — Alternative-model recommendations and user-controlled switching

**Goal:** when a model is exhausted, the user chooses — with reasons.

- [ ] `src/modelpilot/core/recommend.py` — candidate generation: enabled, credentialed, capability-compatible, not rate-limited, not cooling down, distinct from the exhausted model.
- [ ] **Reason strings are generated, not templated blindly.** Each candidate carries at least one concrete, checkable reason (e.g. "same provider family, 118k context still available"; "tool-calling verified; 412/1000 daily requests used"; "0 errors in 30 days"). A candidate with no defensible reason must not be proposed.
- [ ] Ranking: capability fit → headroom → reliability → speed → preference rank. Deterministic.
- [ ] `GET /admin/api/recommendations?provider=&model=` and a switch endpoint to apply a choice.
- [ ] Four explicit user choices, mirroring the brief: **keep current model and wait** (with the reset estimate), **switch now**, **save for later** (persist as a preference), **select manually** from configured models.
- [ ] Session-pinning integration: "switch now" writes `sessions.pinned_*`; "keep and wait" leaves the pin and returns a reset estimate; neither ever happens implicitly.
- [ ] Automatic fallback (opt-in only): `sessions.allow_fallback = 1`, max **1** alternate attempt by default, `model_source='fallback'`, an `events` row, immediate user-visible notification, and a header trail. Off by default for every session, always.
- [ ] **Tests:** recommendations exclude ineligible candidates; a candidate with no reason is excluded; "keep and wait" never dispatches; "switch now" pins and the *next* request is served by the new model (asserted via DB); automatic fallback is disabled unless explicitly enabled; when enabled it records the fallback event accurately and never exceeds the configured attempt cap; a recommended model that turns out to be unavailable at dispatch fails loudly rather than looping.
- [ ] Docs: `docs/RECOMMENDATIONS.md` — the reason taxonomy and the switching contract.

**Exit criteria:** a user can always see why each alternative is recommended; switching is authenticated, explicit, persisted, and reflected in the next request's attribution.

---

## Phase 9 — Coding-agent compatibility and integration tests

**Goal:** prove compatibility, do not assert it.

- [ ] `docs/INTEGRATIONS.md` with a tested, copy-pasteable config for each tool.
- [ ] OpenCode — config snippet (base URL `http://127.0.0.1:PORT/v1`), session header, and a recorded verification date.
- [ ] Codex CLI — config + verification date.
- [ ] Kiro — config + verification date.
- [ ] VS Code extensions (OpenAI-compatible config) — snippet + verification date.
- [ ] Python `openai` SDK example, run in CI as an e2e test.
- [ ] Other agents (e.g. Aider, Zed) documented only after a successful run.
- [ ] `X-ModelPilot-Session` propagation guidance so agent sessions pin correctly.
- [ ] An **agent-session e2e test**: a multi-turn conversation where the target model is deliberately exhausted mid-session, asserting the response is an explicit choice payload (not a silent switch).
- [ ] **Tests:** each documented snippet must correspond to a passing test or a recorded manual verification with date and version; CI fails if a documented tool has no evidence.
- [ ] Docs: per-tool troubleshooting, including "why did my agent report a different model than the one I picked?" (answer: it didn't — check `X-ModelPilot-Model` or the Activity page).

**Exit criteria:** every tool listed as compatible has dated evidence; the agent-session exhaustion test passes.

---

## Phase 10 — Security audit, documentation, performance, reliability

- [ ] Security review against the `SECURITY.md` checklist: crypto, key handling, redaction, loopback binding, SSRF on custom endpoints, dependency audit.
- [ ] `pip-audit` (or equivalent) in CI with a documented policy for findings.
- [ ] Chaos tests: a provider that hangs, returns malformed JSON, returns a wrong model, returns 200 with an empty body, streams then stalls, streams then drops.
- [ ] Reliability metrics: per-provider success rate, latency percentiles (p50/p95), drift count, surfaced in the dashboard.
- [ ] Performance: request-overhead budget documented and measured; a benchmark test asserts the proxy adds < 30 ms p95 excluding upstream time.
- [ ] Signal handling and graceful shutdown (no in-flight corruption, no lost `requests` rows).
- [ ] Backup/restore command with a restore test that verifies credentials still decrypt.
- [ ] Full documentation pass: every doc cross-linked, every command has an example, every claim has evidence.
- [ ] **Tests:** the chaos suite; a restore-drives-a-successful-request test; all audit regression tests named after their failure mode.

**Exit criteria:** security checklist signed off with evidence; chaos suite green; no unredacted secret in logs (asserted by a test that greps captured log output for a known test key).

---

## Phase 11 — Cross-platform packaging and distribution

- [ ] Build a wheel; verify `pip install modelpilot` then `modelpilot serve` in a clean environment.
- [ ] Linux: verify install and run on at least one distro; document the simple service approach.
- [ ] Windows: verify install and run in PowerShell; confirm path handling, key-file ACLs, and SQLite behaviour; document Windows-specific notes.
- [ ] macOS: verify install and run; confirm key-file `0600` and any notarization needs if a binary is shipped.
- [ ] Optional single-file binaries (PyInstaller) per OS; document size and trade-offs.
- [ ] `Dockerfile` (non-root, loopback-bound, volume for the data dir) and a `docker-compose.yml` example.
- [ ] Release checklist: version bump, changelog, licence and `THIRD_PARTY_NOTICES.md` review, security re-check, docs updated, tags signed.
- [ ] **Tests:** a CI matrix on Linux + Windows + macOS running the full suite; a smoke test that runs the packaged artifact and answers a `/v1/chat/completions` from the fake adapter.

**Exit criteria:** a user on each of the three OSes can install from a published artifact and reach a working endpoint without a toolchain.

---

## Cross-cutting requirements (apply to every phase)

- [ ] Every new behavior has a test in the same commit.
- [ ] Every user-visible string that reports a model, a quota, or an error has a rendering test covering the unknown/unverified path.
- [ ] `docs/` is updated in the same commit as the feature.
- [ ] `AGENT_HANDOFF.md` is updated at the end of every work session.
- [ ] No test contacts a real provider.
- [ ] No secret is logged, and a test proves it.
- [ ] A feature is `[~]`, never `[x]`, until tests pass and docs are written.

---

## Milestone summary

| Milestone | Phase | Demonstrable outcome |
| --- | --- | --- |
| M1 | 1 | Architecture, audit, and plan published; licence obligations recorded |
| M2 | 2 | `modelpilot serve` runs with an encrypted credential store and a real schema |
| M3 | 3 | ≥6 providers, each with a passing hermetic contract test |
| M4 | 4 | Exact model identity preserved; aliases mapped or marked unverified |
| M5 | 5 | Every quota number carries a provenance; resets are policy-driven |
| M6 | 6 | The official `openai` SDK works end-to-end with attribution headers |
| M7 | 7 | The dashboard answers "which model served this?" at a glance |
| M8 | 8 | Exhaustion produces ranked, reasoned alternatives and a user choice |
| M9 | 9 | Agent compatibility is proven per tool, with dated evidence |
| M10 | 10 | Security audit and chaos suite green; docs complete |
| M11 | 11 | Installable on Linux, Windows, macOS from a published artifact |
