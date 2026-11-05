# ARCHITECTURE.md — ModelPilot

> **Status:** proposal, Phase 1. Reviewed and ratified before Phase 2 code was written.
> Companion documents: [`docs/AUDIT_FREELLMAPI.md`](docs/AUDIT_FREELLMAPI.md) (why these
> choices), [`ROADMAP.md`](ROADMAP.md) (when), [`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) (state).

---

## 1. The one requirement everything else serves

> **"Which exact AI provider and model generated this response?"**

Every design decision below is subordinate to that question being answerable
**immediately, reliably, and verifiably** — not inferred, not guessed, not "probably".

Three invariants fall out of it:

| # | Invariant | Enforcement |
| --- | --- | --- |
| **I1** | A response is attributed to a concrete `(provider_id, model_id)` **only if justified by evidence**. Otherwise it is `unverified`. | `transparency.py:Attribute`, gate in `router.py` |
| **I2** | The model that serves a **session** does not change unless the user explicitly permitted it. | Session pin check before dispatch; `model_locked` error |
| **I3** | Every request carries its attribution through headers, response metadata, logs, and the DB row — consistently, from one source of truth. | `Attribution` object built once, fanned out |

## 2. Transparency model (the core design)

### 2.1 The `Attribution` object

Built **once**, at the moment a response is finalised, then fanned out to every channel.
Nothing downstream re-derives it.

```python
@dataclass(frozen=True)
class Attribution:
    # --- what the caller asked for -------------------------------------------------
    requested_model: str | None        # raw value from the request body
    requested_alias: str | None        # ModelPilot alias if one was used, else None

    # --- what actually served ------------------------------------------------------
    provider_id: str                   # e.g. "groq"
    provider_name: str                 # e.g. "Groq"
    model_id: str                      # exact upstream id sent to the provider
    model_display_name: str | None     # from local catalog, if known
    served_model_raw: str | None       # the `model` field the upstream echoed back
    model_source: ModelSource          # requested | fallback | unverified

    # --- verification --------------------------------------------------------------
    verification: Verification         # verified | alias_mapped | placeholder | drift_flagged | unverified
    drift_note: str | None             # human-readable, only when verification != verified

    # --- the event ------------------------------------------------------------------
    request_id: str
    timestamp: datetime                # UTC, ISO-8601 with offset
    fallback_used: bool
    fallback_chain: list[str]          # ordered (provider:model) attempts actually made
    status: str                        # ok | ok_stream | provider_error | rate_limited | model_locked
    error: dict | None
    usage: Usage | None                # prompt/completion/total tokens, None when unreported
    latency_ms: int | None
```

### 2.2 `ModelSource` and `Verification` — the two enums that make honesty cheap

```python
class ModelSource(str, Enum):
    REQUESTED = "requested"    # the model the caller named (or the pin) served it
    FALLBACK  = "fallback"     # a different model served it, and the caller permitted that
    PINNED    = "pinned"       # served by an explicit session pin

class Verification(str, Enum):
    VERIFIED      = "verified"      # upstream echoed the model id, or reported nothing to contradict it
    ALIAS_MAPPED  = "alias_mapped"  # upstream echoed an alias we have an explicit catalog mapping for
    PLACEHOLDER   = "placeholder"   # upstream echoed '', 'default', 'unknown' — proves nothing
    DRIFT_FLAGGED = "drift_flagged" # upstream echoed a *different concrete* model than we sent
    UNVERIFIED    = "unverified"    # we cannot justify any attribution
```

`PLACEHOLDER` and `UNVERIFIED` **must never be rendered as a concrete model name** in the UI.
The dashboard shows `Unverified — provider did not report the served model` in that case.
There is no fallback-to-requested for display purposes. This directly implements design
principle #4 ("never invent quota figures or availability") for model identity.

### 2.3 Where attribution is surfaced

| Channel | Mechanism |
| --- | --- |
| Response headers | `X-ModelPilot-Provider`, `X-ModelPilot-Model`, `X-ModelPilot-Model-Source`, `X-ModelPilot-Verification`, `X-ModelPilot-Request-Id`, `X-ModelPilot-Fallback` |
| Response body (non-stream) | `model` = **actual serving model**; `metadata.modelpilot` = full `Attribution` |
| Streaming | Stream is OpenAI-shaped; the truth arrives in trailing headers + a final `metadata` frame when the client asked for `stream_options.include_usage`-style extras, and always in the DB + log |
| Logs | One structured redacted line per request via `logging.py` |
| Persistence | `requests` row, frozen at write time (never recomputed) |
| Dashboard | Activity panel + Live "currently serving" banner |

Naming: the reference uses bare `X-Provider`/`X-Model` for media routes only. ModelPilot uses a
namespaced `X-ModelPilot-*` prefix so it can never collide with an intermediary's headers, and
applies it to **every** route including chat.

## 3. Technology stack

| Layer | Choice | Why |
| --- | --- | --- |
| Language | **Python 3.11+** (tested on 3.13) | Best ecosystem for provider SDKs, minimal ceremony for an adapter-heavy app, ubiquitous on all three target OSes. |
| API framework | **FastAPI** + **Uvicorn** | First-class async streaming (`StreamingResponse`), Pydantic validation gives us request-shape safety for free, auto OpenAPI docs for the `/v1` surface. |
| Validation / settings | **Pydantic v2** + `pydantic-settings` | One schema language for wire models, config files, and env. |
| HTTP client | **httpx** | Async, HTTP/2-capable, one client abstraction for all adapters, `MockTransport` for tests. |
| Database | **SQLite** (WAL) | Local-first by definition: no server to install. Portable file, trivially backed up. Same choice the reference validated at scale. |
| DB access | stdlib `sqlite3` behind a thin `storage.py` | Deliberately no ORM: the schema is small, and an ORM would obscure the exact SQL that attribution correctness depends on. |
| Encryption | **cryptography** (AES-256-GCM) + `hashlib.scrypt` | AES-GCM gives authenticated encryption; scrypt derives the key from a passphrase so no raw key file is strictly required. |
| CLI | **Typer** (or argparse) | `modelpilot serve`, `modelpilot key add`, `modelpilot doctor`. |
| Tests | **pytest** + `pytest-asyncio` + **respx** | respx mocks httpx transports → provider tests are hermetic and free. |
| Packaging | `pyproject.toml` (PEP 621) + **uv**/pip | Cross-platform wheels; uv for fast, reproducible dev environments. |
| Dashboard | Phase 7 — static HTML/JS bundle served by the same app, no build step | Add a bundler only if the UI actually outgrows plain ES modules. Avoiding a Node toolchain keeps install to `pip install modelpilot`. |

### 3.1 Why Python instead of the reference's TypeScript

The reference's stack is valid, but for ModelPilot Python is a better fit:

1. **Adapter economics.** The dominant cost of this project is provider adapters. Python's
   HTTP ergonomics and dict handling make a 60–120 line adapter the norm, and provider SDKs
   and example snippets are overwhelmingly Python-first.
2. **No build step.** `pip install` then run. The reference requires Node 20–24, npm
   workspaces, a Vite/tsc build for the client, and an optional Electron wrapper. That is a
   lot of surface for a local tool whose value is transparency, not a GUI.
3. **Testability of the transparency contract.** The attribution rules are pure functions over
   dicts. pytest expresses them directly, and `respx` lets us assert *exactly* what was sent
   upstream and simulate a provider that echoes a different model — the single most important
   test in the suite.
4. **Cross-platform distribution.** Pure-Python wheel + optional PyInstaller one-file binary
   covers Linux/Windows/macOS without a toolchain on the user's machine.

Trade-off accepted: Python's async streaming story is slightly more verbose than Node's, and
packaging a true single binary is fiddlier than `pkg`. Both are Phase 11 problems, and neither
affects correctness. If the dashboard later needs heavy interactivity, it can be added as a
separate frontend without touching the core.

### 3.2 Rejected alternatives

| Option | Verdict | Reason |
| --- | --- | --- |
| Node/TypeScript (same as reference) | Rejected | Would invite pattern-by-pattern mimicry of the very design we are diverging from, and drags in a build toolchain. |
| Go | Rejected | Excellent runtime and binaries, but weaker iteration speed for a metadata/adapter-heavy app and no benefit to the transparency core. |
| LiteLLM as the engine | Rejected as the core | It is a router that already owns model naming, and it abstracts away exactly the provider-level detail we must expose verbatim. Usable later as an *optional* adapter for exotic providers only. |
| Postgres | Rejected | Violates local-first. SQLite with WAL handles single-user concurrency with room to spare. |
| ORM (SQLAlchemy) | Rejected for now | Indirection hides the SQL that attribution and quota correctness depend on. Revisit only if the schema triples. |
| Remote model catalog | Rejected | A local-first tool must work offline; a remote catalog is a trust and availability dependency (see audit §4.8). |

---

## 4. Module boundaries

Strict layering. Arrows point **inward only**; no cycles.

```
                    ┌───────────────────────────────────────────────┐
   HTTP / CLI  ───▶ │  api/            routes.py, cli.py            │  thin: parse → call core → render
                    └───────────────────────┬───────────────────────┘
                                            ▼
                    ┌───────────────────────────────────────────────┐
     routing  ───▶  │  core/           router.py, session.py        │  the ONLY place selection happens
                    └───┬───────────┬───────────┬───────────┬───────┘
                        ▼           ▼           ▼           ▼
                 ┌────────────┐ ┌────────┐ ┌─────────┐ ┌──────────────┐
     domain  ─▶  │ catalog.py │ │quota.py│ │ history │ │transparency  │
                 │            │ │        │ │  .py    │ │    .py       │
                 └─────┬──────┘ └───┬────┘ └────┬────┘ └──────┬───────┘
                       │            │           │             │
                       ▼            ▼           ▼             ▼
                 ┌───────────────────────────────────────────────────────┐
     infra   ─▶  │ storage.py (SQLite)   credentials.py   crypto.py     │
                 └───────────────────────────────────────────────────────┘
                       ▲                                    ▲
                       │                                    │
                 ┌─────┴──────────────┐              ┌──────┴─────────┐
     adapters ▶  │ adapters/base.py   │◀── registry ─│ adapters/*.py  │
                 └────────────────────┘              └────────────────┘
```

| Module | Responsibility | Must NOT |
| --- | --- | --- |
| `api/routes.py` | OpenAI-compatible surface, header emission, error rendering | contain routing or provider logic |
| `api/cli.py` | `serve`, `key add\|list\|remove`, `provider test`, `models`, `doctor`, `rotate-key` | duplicate core logic |
| `core/router.py` | preference order → eligibility → score → pick; pin enforcement; fallback policy | perform I/O to providers directly |
| `core/session.py` | session id derivation and pin storage | know about HTTP |
| `core/catalog.py` | local model metadata: ids, display names, context windows, capability flags, aliases | fetch anything from the network |
| `core/quota.py` | windowed RPM/RPD/TPM/TPD counters, reset policies, provenance-aware readings | guess a number |
| `core/transparency.py` | `Attribution`, drift normalisation, verification decision | emit headers or log |
| `core/history.py` | freeze `requests` rows at write time | recompute attribution later |
| `adapters/base.py` | adapter contract + capability flags | contain provider-specific strings |
| `adapters/<provider>.py` | one provider: chat, stream, validate, discover | contain routing policy |
| `adapters/registry.py` | id → adapter lookup | be edited to add a provider (registration lives in the adapter module) |
| `infra/credentials.py` | add/remove/test/rotate provider keys, decrypt-at-use | log or return plaintext keys |
| `infra/crypto.py` | AES-256-GCM + scrypt KDF, key artifact handling | know about providers |
| `infra/storage.py` | schema + migrations + typed queries | contain business rules |

**Adding a provider must not touch `core/`.** This is a hard rule and it is tested: see
ROADMAP Phase 3 acceptance criteria.

---

## 5. Data model

SQLite, WAL, foreign keys on. Schema version in `meta(schema_version)`; migrations are
numbered, forward-only SQL with a documented down-path where practical.

```sql
-- Providers the user has configured (one row per provider, not per key)
providers (
  id TEXT PRIMARY KEY,              -- 'groq', 'nvidia_nim', 'custom_ollama'
  name TEXT NOT NULL,               -- display name
  kind TEXT NOT NULL,               -- adapter family: 'openai_compat' | 'anthropic' | 'gemini' | ...
  base_url TEXT,                    -- NULL = adapter default; set for custom endpoints
  enabled INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL,
  last_tested_at TEXT,
  last_test_ok INTEGER,             -- NULL = never tested
  last_test_error TEXT,
  docs_url TEXT,
  notes TEXT
);

-- Credentials, encrypted at rest. One provider may have several keys.
provider_credentials (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  provider_id TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  label TEXT NOT NULL,
  ciphertext BLOB NOT NULL,
  nonce BLOB NOT NULL,
  key_version INTEGER NOT NULL DEFAULT 1,
  created_at TEXT NOT NULL,
  last_used_at TEXT,
  disabled INTEGER NOT NULL DEFAULT 0
);

-- Model catalog: LOCAL, seeded from data/ and refreshed by discovery
models (
  provider_id TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
  model_id TEXT NOT NULL,           -- EXACT upstream id. never normalised, never rewritten
  display_name TEXT,
  context_window INTEGER,           -- NULL = unknown, never invented
  max_output_tokens INTEGER,
  -- capability flags: tri-state NULL = unknown, 1 = yes, 0 = no
  cap_tools INTEGER, cap_vision INTEGER, cap_json_mode INTEGER, cap_reasoning INTEGER,
  cap_streaming INTEGER, cap_embeddings INTEGER,
  -- suitability: only set from verified metadata/benchmarks, NOT from the model's name
  code_score REAL, reasoning_score REAL, speed_score REAL,
  capability_source TEXT NOT NULL,  -- 'provider_api' | 'curated' | 'discovery' | 'user'
  aliases TEXT,                     -- JSON array of alternative ids the provider may echo
  enabled INTEGER NOT NULL DEFAULT 1,
  pin_rank INTEGER,
  first_seen_at TEXT, last_seen_at TEXT,
  PRIMARY KEY (provider_id, model_id)
);
```

```sql
-- Usage ledger: one row per meter window, never a mutable counter
usage_windows (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  provider_id TEXT NOT NULL,
  model_id TEXT,                    -- NULL = provider-wide meter
  window_kind TEXT NOT NULL,        -- 'rpm' | 'rpd' | 'tpm' | 'tpd' | 'monthly'
  window_start TEXT NOT NULL,       -- ISO-8601 UTC
  requests INTEGER NOT NULL DEFAULT 0,
  prompt_tokens INTEGER NOT NULL DEFAULT 0,
  completion_tokens INTEGER NOT NULL DEFAULT 0,
  source TEXT NOT NULL,             -- 'observed' | 'provider_reported'
  UNIQUE (provider_id, model_id, window_kind, window_start)
);

-- What the provider TOLD us it allows (distinct from what we counted)
quota_limits (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  provider_id TEXT NOT NULL,
  model_id TEXT,
  window_kind TEXT NOT NULL,
  limit_value INTEGER,              -- NULL = genuinely unknown
  provenance TEXT NOT NULL,         -- 'provider_documented' | 'provider_api'
                                    -- | 'user_configured' | 'unknown'
  reset_policy TEXT,                -- 'utc_midnight' | 'pacific_midnight' | 'rolling'
                                    -- | 'fixed_window' | 'calendar_month' | 'unknown'
  reset_timezone TEXT,              -- IANA, e.g. 'America/Los_Angeles'
  observed_at TEXT NOT NULL,
  source_url TEXT
);

-- Requests: the audit trail. Attribution is FROZEN here at write time.
requests (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  request_id TEXT NOT NULL UNIQUE,
  ts TEXT NOT NULL,
  session_id TEXT,
  endpoint TEXT NOT NULL,           -- '/v1/chat/completions' | ...
  requested_model TEXT,
  provider_id TEXT,                 -- NULL only when nothing was ever dispatched
  model_id TEXT,                    -- NULL only when nothing was ever dispatched
  model_source TEXT NOT NULL,       -- 'requested' | 'fallback' | 'pinned' | 'none'
  served_model_raw TEXT,            -- what the upstream echoed, verbatim
  verification TEXT NOT NULL,
  verification_note TEXT,
  stream INTEGER NOT NULL DEFAULT 0,
  fallback_used INTEGER NOT NULL DEFAULT 0,
  attempts_json TEXT,               -- JSON array {provider_id, model_id, outcome}
  status TEXT NOT NULL,
  http_status INTEGER,
  error_kind TEXT,                  -- rate_limit | quota_exhausted | auth | timeout | ...
  error_message TEXT,               -- redacted
  latency_ms INTEGER,
  prompt_tokens INTEGER, completion_tokens INTEGER, total_tokens INTEGER,
  client JSON                       -- user-agent / SDK hints, no secrets
);
CREATE INDEX idx_requests_ts ON requests(ts DESC);
CREATE INDEX idx_requests_model ON requests(provider_id, model_id, ts DESC);
CREATE INDEX idx_requests_session ON requests(session_id, ts DESC);

-- Alias resolution: an alias is only mapped when we have explicit evidence
model_aliases (
  alias TEXT NOT NULL,              -- what the provider might echo
  provider_id TEXT NOT NULL,
  model_id TEXT NOT NULL,           -- the concrete id we believe it means
  source TEXT NOT NULL,             -- 'catalog' | 'provider_api' | 'user'
  PRIMARY KEY (alias, provider_id)
);

-- Session pins: a pinned model cannot be swapped by the router
sessions (
  session_id TEXT PRIMARY KEY,
  pinned_provider_id TEXT,
  pinned_model_id TEXT,
  pinned_at TEXT,
  allow_fallback INTEGER NOT NULL DEFAULT 0,   -- DEFAULT FALSE. see principle #3
  last_seen_at TEXT,
  expires_at TEXT
);

-- Unified API keys for callers. SEPARATE from provider credentials.
api_tokens (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  label TEXT NOT NULL,
  token_hash TEXT NOT NULL UNIQUE,  -- SHA-256; plaintext shown once, never stored
  created_at TEXT NOT NULL,
  last_used_at TEXT,
  disabled INTEGER NOT NULL DEFAULT 0
);

-- Append-only transparency/gap event log (quota exhausted, drift, fallback, unverified)
events (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts TEXT NOT NULL,
  kind TEXT NOT NULL,
  provider_id TEXT, model_id TEXT, request_id TEXT,
  severity TEXT NOT NULL,           -- 'info' | 'warning' | 'critical'
  detail_json TEXT
);

meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);  -- schema_version, settings
```

### 5.1 Schema rules that encode the product requirement

1. `requests.provider_id` / `model_id` are `NULL` when nothing was dispatched. **A `NULL` there
   means "unknown", and the UI renders "Unverified", never a guess.**
2. `requests` has no FK to `models`. Deleting a model row must not erase or re-attribute
   history (adopted from audit §3.4).
3. `usage_windows` rows are **insert-or-increment on a window key**. No code path may rewrite a
   past window's counts except by adding an observation.
4. `quota_limits.limit_value` is `NULL` when unknown. `NULL` ≠ `0`, and a `NULL` must never be
   rendered as "0 remaining".
5. `quota_limits.provenance` is required and non-defaulted. There is no such thing as a limit
   without a stated origin.
6. `sessions.allow_fallback` defaults to `0`. Automatic fallback is opt-in and per-session.
7. `api_tokens.token_hash` — the unified key is hashed, the provider keys are encrypted.
   Different threats, different primitives.

---

## 6. Request lifecycle (the transparency path)

```
 1.  POST /v1/chat/completions
 2.  auth: unified API token (hashed lookup). provider keys never touch this path.
 3.  validate body (Pydantic). capture requested_model verbatim.
 4.  resolve session_id
       X-ModelPilot-Session  →  else fingerprint(system message)  →  else request-scoped
 5.  resolve target:
       session pinned?         →  that (provider, model); IGNORE other candidates
       explicit model named?   →  resolve alias → concrete (provider, model)
       alias "auto"?           →  run router (deterministic; see §7) and PIN the result
                                  for the session lifetime  ← the key transparency move
 6.  eligibility check for the target:
       provider enabled? credential present? model enabled?
       quota headroom (provenance-aware)? on cooldown?
       if NOT eligible → do NOT silently move on. Raise 409 `model_locked`-style with a
       body listing: reason, exhaustion event, reset estimate, and candidate alternatives.
 7.  dispatch to adapter.chat_completion() with the EXACT model_id.
 8.  capture served_model_raw from the upstream payload BEFORE any rewriting.
 9.  transparency.build_attribution(...) → verified | alias_mapped | placeholder |
       drift_flagged | unverified
10.  record usage (observed) into usage_windows; record quota events
11.  persist the requests row (attribution frozen)
12.  emit X-ModelPilot-* headers + metadata.modelpilot
13.  structured log line (redacted)
14.  respond
```

**Step 5 is the design. Step 6 is the behavioural guarantee.**

An `auto` selection is resolved **once** and then pinned to the session. The user's first
"auto" is a *decision*, not a per-request lottery. If that model later becomes ineligible, the
user is **asked**, not moved.

### 6.1 Streaming

- Client gets an OpenAI-shaped SSE stream from the provider, piped through with minimal
  transformation.
- The truth is available before the first byte: the routing decision is known at step 5, so
  `X-ModelPilot-*` headers can be sent on the initial response — **before** any content.
- The `served_model_raw` capture happens on the first chunk that carries a `model` field; if
  the provider never sends one, verification is `verified` (nothing contradicted us) only when
  we sent an exact concrete id, otherwise `unverified`.
- A final SSE frame `data: {"metadata": {"modelpilot": {...}}}` is appended when the caller
  opts in via `stream_options`, and the DB row is always written.

---

## 7. Routing engine

Deliberately **deterministic and explicable**. No randomization, no per-request bandit.

```
candidates = enabled models, filtered by:
    provider.enabled
    model.enabled
    credential available
    capability satisfies request (tools? vision? json mode?)
    quota headroom > 0 (provenance-aware, and "unknown" is treated per policy below)
    not on cooldown

score = w1 * preference_rank      # explicit user preference order — dominant term
      + w2 * capability_match     # how well capabilities fit the request
      + w3 * headroom_score       # more remaining quota = lower risk of mid-task failure
      + w4 * reliability          # observed success ratio from `requests` history
      + w5 * speed                # observed latency percentiles

pick = min(candidates, key=(score, provider_id, model_id))   # ties broken, never random
```

Rules:

1. **`preference_rank` dominates.** If the user pinned or ranked a model, config wins. Scoring
   never overrides an explicit user preference.
2. **Ties break deterministically** (`(provider_id, model_id)` lexicographic) so a decision is
   reproducible and a test can assert it exactly.
3. **Unknown quota is not treated as "plenty".** `unknown` headroom gets a configurable penalty
   (default: mid-pack), so a provider that never reports limits cannot silently win every race
   and then die mid-task.
4. **No silent re-selection.** The router runs at session start (or on explicit user request).
   Within a session, `session.pinned_*` short-circuits it entirely.
5. **Fallback is a policy, not a reflex.** `session.allow_fallback` defaults to `0`. When it is
   `1`, a retry to a different model is permitted, is recorded in `attempts_json`, forces
   `model_source = fallback`, and is surfaced in headers + log + UI. The cap is **1** alternate
   attempt by default (vs. the reference's 20) — a long invisible chain is the failure mode we
   are designing against.
6. **The chain is always reported.** Even when a fallback succeeds, `attempts_json` records what
   was tried and why it failed, so the Activity panel can explain the response.

### 7.1 Why this is still useful without a bandit

The product goal is *quota resilience*, not *latency optimization*. Resilience comes from having
several configured providers and a user who can switch in one click — which requires knowing
what is exhausted. A deterministic router plus honest quota accounting delivers that. Adaptive
scoring is a Phase 10+ opt-in nicety (behind an explicit setting that also pins sessions), never
the default.

---

## 8. Quota model — five provenance levels

The product requirement ("never present an estimate as an exact provider-reported quota") is
enforced by a closed enum. Every number shown in the UI is a `QuotaReading` carrying its level:

```python
class QuotaProvenance(str, Enum):
    PROVIDER_API        = "provider_api"         # asked the provider, it answered
    PROVIDER_DOCUMENTED = "provider_documented"  # provider's published policy for this tier
    OBSERVED            = "observed"             # WE counted requests/tokens
    ESTIMATED           = "estimated"            # derived from OBSERVED + documented limit
    USER_CONFIGURED     = "user_configured"      # the operator set a limit by hand
    UNKNOWN             = "unknown"              # nothing reliable to say

@dataclass(frozen=True)
class QuotaReading:
    window_kind: str
    limit: int | None            # None == unknown. NEVER coalesced to 0
    used: int | None             # None == unknown
    remaining: int | None        # only set when BOTH limit and used are known
    provenance: QuotaProvenance
    reset_policy: str            # utc_midnight | pacific_midnight | rolling | ...
    reset_timezone: str | None
    resets_at: datetime | None   # computed from policy; None when policy unknown
    note: str | None             # the honest explanation shown in the UI
```

**Rendering rules (UI contract, tested):**

| Provenance | UI wording |
| --- | --- |
| `PROVIDER_API` | `1,234 / 2,000 remaining — reported by Groq` |
| `PROVIDER_DOCUMENTED` | `2,000/day limit — from Groq's documented free tier` |
| `OBSERVED` | `412 requests sent today — counted locally (Groq does not publish a remaining-quota API)` |
| `ESTIMATED` | `≈ 588 remaining — estimated from 412 sent vs. 1,000 documented` |
| `USER_CONFIGURED` | `600 / 1,000 used — your configured limit` |
| `UNKNOWN` | `Remaining quota not reported by this provider` |

A `PROVIDER_API` reading is never mixed with an `ESTIMATED` one in the same field. If a provider
answers for one window and not another, each window carries its own provenance.

**Reset policies** are data, not assumptions. `quota_limits.reset_policy` + `reset_timezone`
carry the documented policy (e.g. Google AI Studio uses Pacific midnight, not UTC), and
`resets_at` is computed from them. When the policy is unknown, we say so and let the operator
configure it. A rate-limit error observed on the wire can *refine* a policy (e.g. a
`Retry-After` gives a concrete timestamp) but never invents a daily reset time.

**Quota failure is an event, not a silent retry.** On `rate_limit` / `quota_exhausted`:

1. classify as `temporary` (rate limit, has `Retry-After`) / `daily` (daily window exhausted) /
   `unknown`;
2. write an `events` row;
3. compute the reset estimate **only from a known policy**, else `None`;
4. return a choice payload to the caller and show it in the dashboard.

---

## 9. Provider adapter contract

```python
# adapters/base.py
@dataclass(frozen=True)
class ModelInfo:
    model_id: str                    # EXACT id as the provider names it
    display_name: str | None = None
    context_window: int | None = None
    max_output_tokens: int | None = None
    supports_tools: bool | None = None      # None = unknown, not False
    supports_vision: bool | None = None
    supports_json_mode: bool | None = None
    supports_streaming: bool | None = None
    aliases: tuple[str, ...] = ()           # ids the provider may echo instead
    meta: dict = field(default_factory=dict)

class ChatResult:
    """Raw upstream result. Adapters MUST NOT rewrite the model field."""
    served_model_raw: str | None
    content: str | None
    tool_calls: list[dict] | None
    finish_reason: str | None
    usage: Usage | None               # None when the provider did not report tokens
    raw: dict                         # kept for debugging/tests

class ProviderError(Exception):
    kind: ErrorKind                   # normalised enum
    status: int | None
    retry_after_s: float | None       # parsed & CLAMPED to <= 86400
    provider_message: str | None      # redacted before it reaches a log
    daily_exhausted: bool | None

class Adapter(ABC):
    provider_id: str                  # class attribute, e.g. "groq"
    display_name: str

    # capability flags — the audit's lesson (base.ts) applied
    supports_key_validation: bool = True
    supports_model_discovery: bool = True
    supports_quota_probe: bool = False
    requires_key: bool = True

    def default_base_url(self) -> str | None: ...

    async def chat(self, cred, model_id, messages, opts) -> ChatResult: ...
    async def stream(self, cred, model_id, messages, opts) -> AsyncIterator[Chunk]: ...
    async def validate_key(self, cred) -> ValidationResult: ...
    async def list_models(self, cred) -> list[ModelInfo]: ...   # default: raise NotSupported
```

Rules:

1. **Adapters never rewrite `served_model_raw`.** The transparency layer owns that judgement.
2. **`None` means unknown, always.** `supports_tools is None` ≠ `False`. The UI and router treat
   them differently.
3. **Error normalization is the adapter's job** — map provider-specific shapes to `ErrorKind`
   with a `Retry-After`-equivalent extracted and clamped (audit §3.5).
4. **Capability flags explain provider quirks** so the router can avoid a wasted call
   (e.g. "this provider rejects parallel tool calls").
5. Adding a provider = one file in `adapters/` + one line in its own `register()` call + a
   hand-written test with recorded fixtures. **No change to `core/`, and a test asserts that.**

---

## 10. Security architecture

| Concern | Control |
| --- | --- |
| Provider keys at rest | AES-256-GCM; key derived via `scrypt` from a passphrase held in `MODELPILOT_SECRET_KEY` or an `0600` key file beside (not inside) the DB. |
| Key file handling | Atomic write (temp + `fsync` + `rename`), `chmod 0600`, restricted before rename so it is never briefly world-readable. |
| Decrypt-at-use | A credential is decrypted inside the single call that needs it; no long-lived plaintext cache. |
| Unified vs. provider keys | Separate tables, separate primitives (hashed vs. encrypted), separate code paths. A caller can never reach a provider key. |
| Logs | A redaction pass runs at the logging boundary (`logging.py`): key-shaped strings, `Authorization`/`x-api-key` values, and any secret in the redaction registry are replaced with `***`. |
| Admin surface | Bound to `127.0.0.1` by default. A non-loopback bind requires an explicit config flag **and** an admin token; startup refuses otherwise. |
| Custom endpoints | URL validation on add: scheme must be `http(s)`, host must resolve, no credentials in the URL, optional host allowlist, documented SSRF note. |
| Secrets in Git | `.gitignore` excludes `.env`, `*.db`, `*.sqlite*`, `var/`, key files, and backups. Tests run against a temp DB. |
| Rotation / removal | `modelpilot credential rotate` re-encrypts every credential (key-versioned, atomic, tested). `credential remove` deletes the ciphertext and records a tombstone event with **no** key material. |
| Backups | Export is explicit and off by default; a backup contains ciphertext only, never plaintext keys. |

Full detail and threat model: [`SECURITY.md`](SECURITY.md).

---

## 11. Repository layout (target)

```
modelpilot/
├── src/modelpilot/
│   ├── api/
│   │   ├── routes_chat.py        /v1/chat/completions, /v1/completions
│   │   ├── routes_models.py      /v1/models, /v1/embeddings
│   │   ├── routes_admin.py       dashboard + admin JSON API
│   │   ├── routes_recommend.py   alternatives + switching
│   │   ├── auth.py               unified token verification
│   │   └── app.py                app factory
│   ├── core/
│   │   ├── router.py             deterministic selection (§7)
│   │   ├── session.py            session id + pins
│   │   ├── catalog.py            local model metadata
│   │   ├── quota.py              windows, provenance, resets
│   │   ├── transparency.py       Attribution + verification   ← the heart
│   │   ├── history.py            frozen request rows
│   │   └── recommend.py          alternatives + reasons
│   ├── adapters/
│   │   ├── base.py               Adapter, ModelInfo, ChatResult, ProviderError
│   │   ├── registry.py           id → adapter
│   │   ├── openai_compat.py      shared by most providers
│   │   ├── groq.py, cerebras.py, nvidia_nim.py, mistral.py, openrouter.py
│   │   ├── google_ai_studio.py, huggingface.py, cloudflare.py, cohere.py
│   │   └── local_openai.py       Ollama / LM Studio / llama.cpp / vLLM
│   ├── infra/
│   │   └── crypto.py, credentials.py, storage.py, logging.py, config.py
│   ├── data/
│   │   └── catalog.yaml          seeded, versioned model metadata + reset policies
│   └── cli.py
├── tests/
│   ├── unit/        crypto, transparency, quota, catalog, router, recommend
│   ├── contracts/   per-adapter fixtures (hermetic; respx)
│   └── e2e/         OpenAI SDK compatibility, streaming, tool calls
├── docs/            AUDIT_FREELLMAPI.md, INTEGRATIONS.md, API.md, QUOTAS.md
├── ARCHITECTURE.md  ROADMAP.md  AGENT_HANDOFF.md  PROVIDERS.md
├── SECURITY.md  CONTRIBUTING.md  README.md  THIRD_PARTY_NOTICES.md  LICENSE
├── pyproject.toml
└── .gitignore
```

---

## 12. Testing strategy

| Tier | Scope | Rule |
| --- | --- | --- |
| `unit/` | Pure functions: normalization, verification, quota window math, reset computation, scoring, redaction | No network, no DB, fast. These encode the invariants. |
| `contracts/` | One test file per adapter against `respx` fixtures | **A provider is not "supported" until its fixture test passes.** Includes a "provider lies" fixture that echoes a different model. |
| `integration/` | Router + quota + transparency over a temp SQLite DB | Asserts the end-to-end attribution path and the no-silent-switch guarantee. |
| `e2e/` | Real HTTP against the running app with the **official `openai` Python SDK** | Proves compatibility claims rather than asserting them. |

Non-negotiables:

- **No test may contact a real provider.** `respx` blocks/asserts. A test that would need a live
  key must be marked and skipped by default.
- Every quota provenance level has a rendering test — including the `UNKNOWN` path.
- Every `Verification` value has a test producing it, including `PLACEHOLDER` and
  `DRIFT_FLAGGED`.
- The "cannot silently switch a pinned session" test is written **before** the feature exists
  and must fail for the right reason first.

---

## 13. Known risks and mitigations

| Risk | Mitigation |
| --- | --- |
| Provider free tiers change without notice | All quota data is provenance-tagged and stale-dated; a `doctor` command re-verifies; docs show the last verification date per provider. |
| Session identity is ambiguous for some agents | Documented derivation order + explicit `X-ModelPilot-Session` header + a config override; ambiguous cases fall back to request-scoped (never sticky-wrong). |
| Provider silently serves another model | `DRIFT_FLAGGED` verification, an `events` row, a dashboard banner, and a "wrong model" count in the provider's panel. |
| Scope creep toward "support all providers" | Hard rule: adapters land one at a time, each with a green fixture test. No adapter without a test. |
| Local dashboard exposed on a LAN | Loopback-only default; non-loopback bind requires an explicit flag + admin token. |
| SQLite concurrency under streaming | WAL mode, short write transactions, and no writes held across an `await`. |

---

## 14. What is deliberately out of scope (for now)

- Image/video/audio generation and transcription. Chat, streaming, tools, structured output,
  embeddings, and model listing come first.
- Multi-user auth, roles, and sharing. ModelPilot is single-operator by design; the admin
  surface is protected but not multi-tenant.
- Anthropic `/v1/messages` and Ollama emulation surfaces. Valuable for agent coverage (Phase 9)
  but not load-bearing for the transparency contract.
- Prompt compression, response caching, and search-augmentation. Each adds a place where the
  served response can drift from expectation, which is the opposite of the goal.
- A hosted/SaaS mode. Local-first is a feature, not a limitation.
