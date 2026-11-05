# AGENT_HANDOFF.md — ModelPilot

**Read this first if you are an AI agent continuing this project.**
It contains the architectural decisions, the current repo state, the invariants you must not
break, and the exact next step. Update it at the end of every work session.

**Last updated:** 2025-11-05 (Phase 1 — audit, architecture, documentation)
**Repo:** `https://github.com/wachin/modelpilot`
**Working directory:** repository root
**Current phase:** Phase 1 complete — Phase 2 not started.

---

## 1. What this project is (30 seconds)

A local-first app that connects multiple AI providers behind one OpenAI-compatible endpoint
(`http://127.0.0.1:PORT/v1`), for the purpose of surviving a provider's exhausted free quota.

The differentiator is **not** routing. It is that **the user must always know exactly which
provider and model produced a given response**, and the app must never silently change the
model mid-session.

## 2. Current repository state

```
modelpilot/
├── README.md                     entry point, vision, quick start (planned), doc index
├── ARCHITECTURE.md               stack, modules, data model, routing, quota, security   [DONE]
├── ROADMAP.md                    phases 1-11 with [ ]/[~]/[x] markers                   [DONE]
├── AGENT_HANDOFF.md              this file                                              [DONE]
├── PROVIDERS.md                  provider plan + verification policy                    [DONE]
├── SECURITY.md                   threat model, controls, rotation runbook               [DONE]
├── CONTRIBUTING.md               dev workflow, invariants, review checklist             [DONE]
├── THIRD_PARTY_NOTICES.md        FreeLLMAPI MIT attribution + our reuse policy          [DONE]
├── LICENSE                       MIT                                                    [DONE]
├── .gitignore                    secrets, DBs, venvs, caches, OS noise, external/       [DONE]
├── docs/
│   └── AUDIT_FREELLMAPI.md       source-level audit of the reference                    [DONE]
└── external/                     reference checkout (NOT tracked; see .gitignore)
```

**No application code exists yet.** That is intentional: the brief requires the architecture to
be clear before implementation.

### Phase status

| Phase | Status | Evidence |
| --- | --- | --- |
| 1 — Audit, architecture, requirements | `[x]` | `docs/AUDIT_FREELLMAPI.md`, `ARCHITECTURE.md`, `ROADMAP.md`, this file |
| 2 — Core app, storage, credentials, provider abstraction | `[ ]` | not started |
| 3 — Initial providers + connection testing | `[ ]` | not started |
| 3b — Extended providers | `[ ]` | not started |
| 4 — Model catalog + exact identification | `[ ]` | not started |
| 5 — Usage, quota, reset policies | `[ ]` | not started |
| 6 — Unified OpenAI-compatible API | `[ ]` | not started |
| 7 — Dashboard + activity history | `[ ]` | not started |
| 8 — Recommendations + user switching | `[ ]` | not started |
| 9 — Coding-agent compatibility | `[ ]` | not started |
| 10 — Security audit, perf, reliability | `[ ]` | not started |
| 11 — Cross-platform packaging | `[ ]` | not started |

---

## 3. Decisions already made (do not re-litigate without recording why)

| # | Decision | Rationale | Where |
| --- | --- | --- | --- |
| D1 | **Python 3.11+ / FastAPI / SQLite / httpx** | Adapter economics, no build step, best fixture-test ergonomics. Rejected alternatives recorded. | `ARCHITECTURE.md` §3 |
| D2 | **No bandit / no randomized routing in the default path.** Deterministic, ties broken on `(provider_id, model_id)`. | The reference's Thompson-sampling bandit re-selects the model every request, which *is* model opacity. | `ARCHITECTURE.md` §7 |
| D3 | **`auto` resolves once and pins to the session.** | A user's model choice is a decision, not a per-request lottery. | `ARCHITECTURE.md` §6 step 5 |
| D4 | **An ineligible pinned model returns a choice payload; it never silently switches.** | Core product requirement. | `ARCHITECTURE.md` §6 step 6 |
| D5 | **`sessions.allow_fallback` defaults to `0`; when enabled, max 1 alternate attempt.** | The reference allows up to 20 invisible attempts — the failure mode we design against. | `ARCHITECTURE.md` §7 rule 5 |
| D6 | **The response `model` field echoes the model that ACTUALLY served**, with the requested id in `metadata.modelpilot.requested_model`. | The reference overwrites `model` with the routed/alias id, destroying the evidence (its own migration comment says so; issue #534). | `ARCHITECTURE.md` §2.3 |
| D7 | **`Verification` is a closed enum; `PLACEHOLDER` and `UNVERIFIED` never render as a model name.** | Never invent an identity. | `ARCHITECTURE.md` §2.2 |
| D8 | **`QuotaProvenance` is a closed enum (6 values); `NULL` never renders as `0`.** | Never present an estimate as exact provider-reported quota. | `ARCHITECTURE.md` §8 |
| D9 | **Model metadata is local and versioned (`data/catalog.yaml`); no remote catalog.** | Local-first must work offline. | `ARCHITECTURE.md` §3.2 |
| D10 | **Capability flags are tri-state (`None` = unknown), never defaulted to `False`.** | "Unknown" and "no" drive different UI and routing behaviour. | `ARCHITECTURE.md` §9 rule 2 |
| D11 | **No capability score may be derived from a model's name.** | "coder" in a name proves nothing; a test enforces this. | `ROADMAP.md` Phase 4 |
| D12 | **One `OpenAICompatAdapter` parameterized by options; one file per provider.** | Proven in the reference: Groq/Cerebras are ~5 lines each. | audit §3.1 |
| D13 | **AES-256-GCM + scrypt KDF; key artifact beside (not inside) the DB; atomic `0600` write.** | Adopted from the reference's reasoning, implemented independently. | `ARCHITECTURE.md` §10 |
| D14 | **Adapters land one at a time, each with a hermetic `respx` contract test.** No adapter without a test; no test that touches a real provider. | Quality over breadth. | `PROVIDERS.md` |
| D15 | **Historical attribution is frozen at write time; no FK from `requests` to `models`.** | History must survive model deletion and never be re-attributed. | `ARCHITECTURE.md` §5.1 |
| D16 | **Redaction is a logging-boundary concern, not a reviewer habit.** | The reference has dedicated redaction libs for exactly this. | `ARCHITECTURE.md` §10 |

---

## 4. The invariants (breaking any of these is a bug, not a design choice)

- **I1 — Justified attribution.** A response is attributed to a concrete `(provider_id, model_id)`
  only with evidence. Otherwise `verification = unverified` and nothing else is claimed.
- **I2 — No silent switch.** A session's model does not change without an explicit user action,
  ever, in any code path, including error handling and retries.
- **I3 — One source of truth.** The `Attribution` object is built once and fanned out to headers,
  metadata, logs, and the DB. Nothing re-derives it downstream.
- **I4 — Unknown is a value, not a default.** `None`/`NULL` means unknown in limits, tokens,
  context windows, capabilities, and reset times. It is never coerced to `0` or `False`.
- **I5 — Estimates are labelled.** An estimated number is never rendered as provider-reported.
- **I6 — No secret leaves the credential layer.** Not in logs, not in API responses, not in
  errors, not in the DB in plaintext.

If a requested change conflicts with an invariant, stop and flag it rather than implementing it.

---

## 5. Reference project — what to look at, and what to ignore

The audited reference lives at `external/freellmapi` (gitignored, not vendored). Full audit:
`docs/AUDIT_FREELLMAPI.md`.

**Look at it when you want a proven shape for:**

| Need | Reference file |
| --- | --- |
| Adapter interface with capability flags | `server/src/providers/base.ts` |
| OpenAI-compatible adapter parameterized by options | `server/src/providers/openai-compat.ts` |
| Registry pattern | `server/src/providers/index.ts` |
| AES-GCM credential handling + key-file reasoning | `server/src/lib/crypto.ts` |
| Decrypt-at-use discipline | `server/src/services/provider-credential.ts` |
| Back-off parsing from header/body/prose | `server/src/providers/base.ts` (`parseRetryAfterMs`, `parseStatedRetryMs`) |
| Drift detection between requested and served model | `server/src/lib/served-model.ts` |
| Freezing request identity in history | `server/src/db/migrations/20260913_000001_request_model_attribution.ts` |
| Windowed quota accounting | `server/src/services/ratelimit.ts`, `provider-quota.ts` |
| Redaction at the logging boundary | `server/src/lib/log-redaction.ts`, `error-redaction.ts` |
| A working agent config | `opencode.json` |

**Do NOT copy from it:**

- `server/src/services/router.ts` + `scoring.ts` as a whole — the bandit is the design we
  explicitly reject (D2). Read it to understand the trade-off, do not port it.
- The practice of overwriting the response `model` field (D6).
- The remote signed catalog (`services/catalog-sync.ts`) — we ship a local `data/catalog.yaml` (D9).
- Any adapter file as a whole — write our own and record what we learned.

**Licence rule:** MIT (`Copyright (c) 2026 Tashfeen Ahmed`). No code is currently reused. If you
copy *any* code verbatim, you must (a) keep the upstream MIT header in that file, (b) update
`THIRD_PARTY_NOTICES.md`, and (c) say so in the commit message.

---

## 6. Exact next steps (Phase 2)

Work in this order. Do not skip ahead — each step's tests are the foundation for the next.

1. **`pyproject.toml`** — PEP 621, `requires-python = ">=3.11"`, runtime deps
   (`fastapi`, `uvicorn[standard]`, `pydantic`, `pydantic-settings`, `httpx`, `cryptography`,
   `typer`), dev extras (`pytest`, `pytest-asyncio`, `respx`, `ruff`), and a `modelpilot` console
   script entry point. Configure `pytest` (`asyncio_mode = "auto"`) and `ruff`.
2. **`src/modelpilot/infra/config.py`** — pydantic-settings model. Fields: `data_dir`,
   `host` (default `127.0.0.1`), `port` (default `8080`), `secret_key`, `db_path`,
   `log_level`, `admin_token`, `allow_non_loopback` (default `False`). Layering: defaults →
   optional `~/.config/modelpilot/config.toml` → env (`MODELPILOT_*`) → CLI flags.
   **Refuse to start** if a non-loopback host is set without `allow_non_loopback` + `admin_token`.
3. **`src/modelpilot/infra/crypto.py`** — `encrypt(plaintext, key) -> (ciphertext, nonce)`,
   `decrypt(ciphertext, nonce, key) -> plaintext`, `derive_key(passphrase, salt)` via
   `hashlib.scrypt` (n=2**14, r=8, p=1 — document the parameters), `load_or_create_key_file()`
   with temp+fsync+rename and `chmod 0600`. Fail fast with an actionable message on a bad key.
4. **`src/modelpilot/infra/storage.py`** — `connect()` with `PRAGMA journal_mode=WAL`,
   `PRAGMA foreign_keys=ON`; `migrate()` applying numbered migrations; `schema_version` in `meta`.
   Write schema v1 exactly as specified in `ARCHITECTURE.md` §5.
5. **`src/modelpilot/infra/credentials.py`** — `add(provider_id, label, plaintext)`,
   `list_masked()`, `get_for_use(id)` (returns plaintext inside a context manager, never stored),
   `remove(id)`, `rotate_all(new_passphrase)`.
6. **`src/modelpilot/infra/logging.py`** — structured logger + a `redact()` applied to every
   record's message/args; register known secrets so they are masked in output.
7. **`src/modelpilot/adapters/base.py`** — the contract from `ARCHITECTURE.md` §9.
8. **`src/modelpilot/adapters/registry.py`** — `register(adapter)`, `get(provider_id)`,
   `all()`; raise on duplicate ids.
9. **`src/modelpilot/adapters/fake.py`** — a deterministic adapter with scripted outcomes
   (success, specific model echo, drift echo, placeholder echo, 429 with/without `Retry-After`,
   5xx, malformed JSON, stall). **Every later test depends on this file**, so build it early and
   make it thorough.
10. **`src/modelpilot/api/app.py`** — `create_app(settings)` with lifespan init and `/health`.
11. **`src/modelpilot/cli.py`** — `serve`, `doctor`, `credential add|list|remove|rotate`.
12. **Tests** — as itemized in `ROADMAP.md` Phase 2.
13. Run `pytest`; then run `modelpilot serve` and `modelpilot credential add` manually and
    inspect the DB file to confirm ciphertext.
14. Update `ROADMAP.md` Phase 2 markers and this file's §2 phase table.

### Opening moves, concretely

```bash
python -m venv .venv && . .venv/bin/activate
uv pip install -e '.[dev]'     # or: pip install -e '.[dev]'
pytest -q
modelpilot doctor
```

---

## 7. Environment facts (as of Phase 1)

| Item | Value |
| --- | --- |
| OS | Linux |
| Python | 3.13.5 |
| `uv` | available (`~/.local/bin/uv`) |
| `poetry` | available (not required) |
| Node | v22.23.4 (irrelevant to ModelPilot; the reference needs it) |
| CPUs / RAM | 4 cores / ~7.6 GiB |

Verified working install set (used to validate the toolchain during Phase 1):
`fastapi 0.162.0`, `pydantic 2.14.3`, `cryptography 50.0.2`, `httpx 0.28.1`,
`pytest 9.2.0`, `pytest-asyncio 1.4.1`, `respx 0.23.1`, `uvicorn 0.54.0`.
Pin compatible ranges in `pyproject.toml` rather than these exact versions.

---

## 8. Things that have already gone wrong (do not repeat)

| Symptom | Cause | Fix applied / rule |
| --- | --- | --- |
| `uv venv` installs no `pip` | `uv`-created venvs omit `pip` by design | Use `uv pip install` into the venv, or `python -m ensurepip`. Do not assume `pip` exists. |
| Editor rejects large single writes | Tooling limit (~6000 chars per call) | Write docs in several appended sections rather than one large block. |
| Two previous attempts produced no artifacts | Work was started but never committed | Commit documentation as soon as a phase's docs are complete; do not batch across sessions. |
| `external/` reference is large and irrelevant to builds | It is a full checkout of another project | It is gitignored. Never reference it from `src/` or `tests/`. |

---

## 9. Definition of done (use this checklist before marking anything `[x]`)

- [ ] Code exists and is wired into the app (not dead code).
- [ ] Tests exist in the same commit and pass (`pytest -q` green).
- [ ] The tests cover the failure path, not only the happy path.
- [ ] If the change touches model identity, quota reporting, or error surfaces: a test covers the
      `unknown` / `unverified` path specifically.
- [ ] No test contacts a real provider.
- [ ] No secret appears in logs or responses (proven by a test where practical).
- [ ] Relevant `docs/` updated in the same commit.
- [ ] `ROADMAP.md` marker updated (`[ ]` → `[~]` → `[x]`).
- [ ] `AGENT_HANDOFF.md` §2 phase table and §8 updated at the end of the session.
- [ ] Manually run once (serve / CLI) and the result observed.

---

## 10. Open questions for Phase 2 (resolve and record the answer here)

1. **Session identity fallback order.** Proposed: `X-ModelPilot-Session` → SHA-256 of the first
   system message → `X-Request-ID` → request-scoped. Confirm against OpenCode, Codex CLI, and
   Kiro request shapes in Phase 9 and update this file.
2. **`model` field on a permitted fallback.** Current decision: echo the **actual** serving
   model (D6). Re-verify against real agent behaviour; if an agent breaks on an unexpected model
   id, the fallback of choice is to also send the requested id in a header the agent can read —
   **not** to lie in `model`.
3. **SQLite write strategy under streaming.** Confirm that no write transaction is held across an
   `await` and that WAL handles the streaming concurrency in practice (Phase 6).
4. **`/v1/models` handling of unreachable models.** Omit vs. flag. Decide in Phase 4, document,
   and test.
5. **`scrypt` parameter calibration.** n=2**14/r=8/p=1 is a starting point; measure on a modest
   CPU (4 cores here) and record the resulting derivation time in `SECURITY.md`.
