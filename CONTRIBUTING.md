# CONTRIBUTING.md — ModelPilot

Thanks for helping. ModelPilot has an unusually strict core requirement, so read §2 before you
write code — most review feedback will come from there.

---

## 1. Getting set up

```bash
git clone https://github.com/wachin/modelpilot
cd modelpilot
python -m venv .venv && . .venv/bin/activate
uv pip install -e '.[dev]'     # or: pip install -e '.[dev]'
pytest -q
```

Common commands (once Phase 2 lands):

```bash
pytest -q                       # full suite
pytest tests/unit -q            # fast, pure-function tests
pytest tests/contracts -q       # provider adapter fixtures (hermetic)
ruff check . && ruff format .   # lint + format
modelpilot doctor               # environment + permission self-check
```

**No test may contact a real provider.** Provider tests use `respx` fixtures. If a test needs a
live key, mark it and ensure it is skipped by default.

---

## 2. The rules that matter here

Before anything else, read the six invariants in `AGENT_HANDOFF.md` §4. In practice:

1. **Never invent a model identity.** If the upstream did not tell us what served the response,
   the result is `verification = unverified`. Do not fall back to "it was probably what we asked
   for." This is the whole point of the project.
2. **Never silently change the model.** Any code path that could dispatch to a different model
   than the session's pinned one is a bug, including retries and error handling.
3. **`None` means unknown.** Never coerce an unknown limit, token count, context window, or
   capability to `0` or `False`. "Unknown" and "no" are different facts.
4. **Never present an estimate as a provider-reported value.** Every quota number carries its
   provenance.
5. **Never log or return a secret.** Redaction is applied at the logging boundary; do not rely on
   being careful.
6. **One source of truth.** Build the `Attribution` once and fan it out. Do not recompute it in a
   second place, and do not recompute it later from history.

If a feature request conflicts with one of these, open an issue to discuss it rather than
implementing it quietly.

---

## 3. Adding a provider

Follow the eight-step checklist in [`PROVIDERS.md`](PROVIDERS.md) §4. The short version:

1. One file in `src/modelpilot/adapters/`, preferring the shared `OpenAICompatAdapter`.
2. Declare capability flags honestly.
3. Map errors to `ErrorKind` and clamp any provider-stated back-off to 86400 s.
4. Return the upstream `model` value verbatim — never rewrite it.
5. A `tests/contracts/` file with fixtures including a **"provider lies"** case that echoes a
   different model id.
6. Seed catalog entries with a `source_url` and a `verified` date.
7. Add a row to `PROVIDERS.md` §1 with the live-verification date.
8. Keep `core/` untouched — an architectural test enforces this.

**Do not send a PR with several new providers at once.** One provider per pull request, each with
its tests, so a reviewer can actually verify the claims.

---

## 4. Pull request expectations

A PR is ready to review when:

- [ ] `pytest -q` passes locally.
- [ ] `ruff check .` is clean.
- [ ] New behaviour has tests in the same commit, covering failure paths as well as the happy path.
- [ ] Any change touching model identity, quota reporting, or error surfaces has a test for the
      unknown/unverified path.
- [ ] No secret is logged or returned (and a test proves it where practical).
- [ ] Relevant `docs/` files are updated in the same commit.
- [ ] `ROADMAP.md` markers are updated with the right status (`[ ]` → `[~]` → `[x]`).
- [ ] `AGENT_HANDOFF.md` §2 and §8 are updated if the session changed project state.
- [ ] The PR description states what was verified manually, and how.

**A marker is not `[x]` until tests pass and docs are written.** Reviewers will ask for evidence
rather than accepting "it works".

---

## 5. Commit conventions

Conventional commits, scoped by module:

```
feat(adapters): add Groq adapter with contract fixtures
fix(quota): do not render an unknown limit as zero
docs(architecture): record the fallback policy decision
test(transparency): cover the placeholder-echo case
```

Rules:

- One logical change per commit.
- If you copy code from the reference project, say so in the commit message and update
  `THIRD_PARTY_NOTICES.md` (see §6).
- Never commit a real API key, a data directory, or a `.env` file. If you do it by accident,
  rotate the key first, then remove the file — assume it is public.

---

## 6. Third-party code and licensing

- ModelPilot is **MIT** (`LICENSE`).
- The reference project (FreeLLMAPI) is **MIT**, `Copyright (c) 2026 Tashfeen Ahmed`. It is
  audited in `docs/AUDIT_FREELLMAPI.md` and used as an architectural reference, **not** vendored.
- **No FreeLLMAPI code is currently reused.** We implement our own version of the patterns we
  learned from it.
- If you want to copy code from any third-party project:
  1. keep the original licence header in the file,
  2. add an entry to `THIRD_PARTY_NOTICES.md` (project, licence, copyright holder, files affected),
  3. say so in the commit message.
- Prefer re-implementation when the borrowed material is a design idea rather than a
  meticulously-tuned implementation. Attribution is cheap; a licensing mess is not.

---

## 7. Documentation rules

- Every doc claim should be checkable. "Supports X" needs a test or a dated manual verification.
- Provider support claims need a **date**.
- Keep `ARCHITECTURE.md` decisions current; when a decision changes, record the change and the
  reason rather than editing the old text into something it never said.
- Cross-link rather than duplicate. If two docs need the same content, one owns it and the other
  links to it.

---

## 8. Review priorities

When reviewing, check in this order:

1. Could this cause a silent model change? (highest severity)
2. Could this invent or misattribute a model identity?
3. Could this misreport a quota as exact when it is an estimate, or as `0` when it is unknown?
4. Could this leak a secret?
5. Is there a test for the failure path?
6. Is the code in the right layer (`core/` vs. `adapters/` vs. `infra/`)?
7. Style and naming.

---

## 9. Code of conduct

Be direct about technical problems and generous about people. Assume good faith, ask questions
before assuming an error, and keep discussion focused on the code and the user's problem.

---

## 10. Questions

Open a discussion issue. If you are an AI agent continuing this project, read
[`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) first — it has the current state and the exact next step.
