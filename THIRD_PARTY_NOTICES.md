# THIRD_PARTY_NOTICES.md — ModelPilot

ModelPilot is MIT-licensed (see [`LICENSE`](LICENSE)). This file records third-party projects
that influenced its design or whose code is included, as required by their licences.

**Current status: no third-party source code is included in this repository.**

---

## 1. FreeLLMAPI — architectural reference (no code reused)

| Field | Value |
| --- | --- |
| Project | FreeLLMAPI |
| Repository | https://github.com/tashfeenahmed/freellmapi |
| Licence | MIT |
| Copyright | `Copyright (c) 2026 Tashfeen Ahmed` |
| Files affected | **None.** No source code, data file, or asset from FreeLLMAPI is present in ModelPilot. |
| Interaction | A local checkout was read for a source-level architecture audit only. The checkout is excluded from version control (see `.gitignore`, `external/`). |
| Audit | [`docs/AUDIT_FREELLMAPI.md`](docs/AUDIT_FREELLMAPI.md) |

### 1.1 What we learned from it (attribution of ideas, not code)

The following design ideas were studied in FreeLLMAPI and **independently re-implemented** in
ModelPilot. No source was copied. They are recorded here because honest provenance is part of
this project's stated principles.

| Idea | Reference location | ModelPilot implementation |
| --- | --- | --- |
| Adapter registry keyed by provider id, with capability flags on the adapter | `server/src/providers/base.ts`, `index.ts` | `src/modelpilot/adapters/base.py`, `registry.py` |
| One OpenAI-compatible adapter parameterized by `baseUrl`/headers/timeout, reused by many providers | `server/src/providers/openai-compat.ts` | `src/modelpilot/adapters/openai_compat.py` |
| AES-256-GCM credential encryption with the key artifact kept **beside** the database and restricted to `0600` | `server/src/lib/crypto.ts` | `src/modelpilot/infra/crypto.py` |
| Decrypt-at-use rather than caching plaintext credentials | `server/src/services/provider-credential.ts` | `src/modelpilot/infra/credentials.py` |
| Parsing a back-off from `Retry-After`, structured error-body fields, and prose — with an upper clamp so a hostile value cannot bench a key indefinitely | `server/src/providers/base.ts` (`parseRetryAfterMs`, `parseStatedRetryMs`) | `src/modelpilot/adapters/base.py` (clamped to 86400 s) |
| Comparing the requested model against the model the upstream *actually* echoed, and treating cosmetic differences as non-drift | `server/src/lib/served-model.ts` | `src/modelpilot/core/transparency.py` |
| Freezing request→model attribution at write time, and preferring "unattributable" over guessing | `server/src/db/migrations/20260913_000001_request_model_attribution.ts` | `ARCHITECTURE.md` §5 invariant; `core/history.py` |
| Windowed rate-limit accounting (RPM/RPD/TPM/TPD) over a persistent ledger with provenance on quota observations | `server/src/services/ratelimit.ts`, `provider-quota.ts` | `src/modelpilot/core/quota.py` |
| Applying redaction at the logging boundary rather than relying on review discipline | `server/src/lib/log-redaction.ts`, `error-redaction.ts` | `src/modelpilot/infra/logging.py` |
| Hermetic transport mocking plus an in-memory database so provider tests never touch a paid endpoint | `server/vitest.config.ts`, `server/src/__tests__/` | `tests/contracts/` with `respx` |

### 1.2 Where ModelPilot deliberately diverges

These are documented in full in the audit and in `AGENT_HANDOFF.md` §3 (decisions D2, D5, D6, D9):

- **No randomized / bandit routing.** ModelPilot's router is deterministic and reproducible.
- **`auto` resolves once and pins to the session** rather than being re-decided per request.
- **An ineligible pinned model produces a user choice, never a silent switch.**
- **Automatic fallback defaults to off**, with a maximum of one alternate attempt when enabled
  (the reference permits up to 20 invisible attempts).
- **The response `model` field reports the model that actually served**, instead of being
  overwritten with a gateway-internal id.
- **The model catalog is local and versioned**, not fetched from a remote signed catalog.

---

## 2. Runtime dependencies

Populated when `pyproject.toml` is created (Phase 2). The expected runtime set is
FastAPI, Uvicorn, Pydantic / pydantic-settings, httpx, cryptography, and Typer; development
tooling is pytest, pytest-asyncio, respx, and Ruff. Each will be listed here with its licence
(these are all permissively licensed) once pinned.

Because ModelPilot is distributed as source and installed via `pip`, dependency licences are
also carried by the installed distributions. This file records them for auditability.

---

## 3. Policy for future contributions

If any third-party code is ever copied into this repository:

1. the original licence header must remain in the affected file(s);
2. an entry must be added to §1 or §2 above naming the project, licence, copyright holder, and
   the affected files;
3. the commit message must state that code was copied and from where;
4. if the upstream licence requires the full licence text to be reproduced, it must be included
   verbatim in this file.

Until such an entry exists, assume **no** third-party code has been incorporated, and do not
introduce any without following the four steps above.

See also [`CONTRIBUTING.md`](CONTRIBUTING.md) §6.
