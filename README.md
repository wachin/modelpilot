# ModelPilot

**Multi-provider AI model manager and router — with mandatory model transparency.**

ModelPilot connects several AI API providers behind a single OpenAI-compatible endpoint
(`http://127.0.0.1:PORT/v1`) so you are not stuck when one provider's free daily quota, token
allowance, or rate limit runs out — while making sure you *always* know which provider and which
exact model produced a given response.

> **The one rule everything else serves:**
> **"Which exact AI provider and model generated this response?"** must be answerable
> immediately, reliably, and verifiably. Not inferred. Not "probably". Not hidden behind a
> generic "Auto Model" label.

**Status: Phase 1 complete — architecture and documentation. No application code yet.**
See [`ROADMAP.md`](ROADMAP.md) for the phased plan and [`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) for
the current project state.

---

## Why this exists

API routers that silently pick "the best available model" are fine for a chatbot and dangerous
for a coding agent. Different models have different reasoning ability, coding skill, context
window, tool-calling behaviour, and reliability. A model swap mid-task can quietly degrade your
code.

ModelPilot makes the opposite trade: **transparency over invisible routing.**

| Principle | How ModelPilot enforces it |
| --- | --- |
| Never hide the actual model | Every response carries `X-ModelPilot-Provider`, `X-ModelPilot-Model`, `X-ModelPilot-Verification`, and a `metadata.modelpilot` block |
| Never silently switch models | A session's model is pinned; if it becomes unavailable you get a **choice**, not a swap |
| Never invent an identity | If the provider does not tell us what served the response, it is marked **unverified** |
| Never invent quota figures | Every number carries its provenance: provider-reported, observed, estimated, user-configured, or **unknown** |
| Never assume a reset time | Reset policy and timezone are data; unknown policies are reported as unknown |

---

## Documentation

| Document | What it covers |
| --- | --- |
| [`ARCHITECTURE.md`](ARCHITECTURE.md) | Stack and rationale, module boundaries, data model, routing engine, quota provenance model, security architecture |
| [`ROADMAP.md`](ROADMAP.md) | Phases 1–11 with `[ ]` / `[~]` / `[x]` markers and exit criteria per phase |
| [`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) | Current state, fixed decisions (D1–D16), the six invariants, next steps |
| [`PROVIDERS.md`](PROVIDERS.md) | Provider plan, the eight-step "add a provider" checklist, free-tier reality check |
| [`SECURITY.md`](SECURITY.md) | Threat model, controls, credential rotation and leak runbooks, operator checklist |
| [`CONTRIBUTING.md`](CONTRIBUTING.md) | Setup, the rules that matter, PR expectations, review priorities |
| [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md) | Licence obligations and the attribution of studied design ideas |
| [`docs/AUDIT_FREELLMAPI.md`](docs/AUDIT_FREELLMAPI.md) | Source-level audit of the reference project: what we adopted, what we rejected, and why |

---

## Planned quick start (once Phases 2–6 land)

> This section describes the intended experience. **None of it exists yet** — it is here so the
> target is explicit and testable. Do not treat it as working instructions.

```bash
# install
git clone https://github.com/wachin/modelpilot
cd modelpilot
python -m venv .venv && . .venv/bin/activate
pip install -e '.[dev]'

# store a provider key (encrypted at rest)
modelpilot credential add --provider groq --label personal

# verify it works
modelpilot provider test groq

# see what is available and how much quota is left
modelpilot models list
modelpilot quota show

# issue a token for your coding agent (separate from provider keys)
modelpilot token create --label opencode

# run the local endpoint
modelpilot serve        # http://127.0.0.1:8080/v1
```

Then point any OpenAI-compatible client at it:

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:8080/v1", api_key="mp-...")
resp = client.chat.completions.create(
    model="groq/llama-3.3-70b-versatile",
    messages=[{"role": "user", "content": "hi"}],
)
print(resp.choices[0].message.content)

# the truth, always available:
print(resp.model)                                    # the model that actually served it
print(resp.metadata["modelpilot"]["verification"])   # verified / alias_mapped / placeholder / drift_flagged / unverified
print(resp.metadata["modelpilot"]["provider_id"])
```

### Transparency channels

Every request's attribution is recorded and exposed through all of these, from a single source
of truth:

| Channel | How |
| --- | --- |
| Response headers | `X-ModelPilot-Provider`, `-Model`, `-Model-Source`, `-Verification`, `-Fallback`, `-Request-Id` |
| Response metadata | `metadata.modelpilot` in non-streaming bodies; a trailing metadata frame in streams |
| Application logs | One structured, redacted line per request |
| Activity history | A frozen `requests` row (never recomputed later) |
| Dashboard | A live "currently serving" banner plus an Activity inspector that shows the raw upstream model echo when it differs from what was requested |

---

## When a model runs out of quota

ModelPilot does **not** move you to another model quietly. It tells you what happened and offers
four explicit choices:

1. **Keep the current model and wait** for its quota to reset (with the reset estimate when the
   provider's reset policy is known, and an honest "unknown" when it is not).
2. **Switch to a suggested model now** — with the provider, exact model id, context window,
   capability flags, and quota status for each suggestion, plus the reason it is being
   recommended.
3. **Save the suggested model for later.**
4. **Pick another configured model manually.**

Automatic fallback exists, but is **off by default** for every session, and when enabled it is
limited to one alternate attempt and is announced in the response headers, the logs, and the
dashboard.

---

## What ModelPilot is not

- **Not a way to get more free quota.** It helps you keep working when a quota runs out; it does
  not circumvent any provider's limits, and it does not claim any quota figure it cannot justify.
- **Not a latency optimizer.** Its router is deterministic and explicable precisely so that a
  routing decision can be read and reproduced — not a black box that reshuffles models per
  request.
- **Not a hosted service.** It is local-first: loopback by default, no telemetry, no remote model
  catalog, your provider keys encrypted at rest and never exposed to connected agents.

---

## Design principles (from the project brief)

1. Transparency is more important than invisible automatic routing.
2. Never hide the actual model used.
3. Never silently switch models during a coding task.
4. Never invent quota figures or availability.
5. Clearly distinguish reported quota from estimated usage.
6. Preserve conversation and session consistency whenever possible.
7. Keep provider integration modular.
8. Keep the application local-first and secure.
9. Prefer reliable, tested implementations over a large number of unverified integrations.
10. Keep the project open-source and maintainable.

---

## Contributing

Start with [`CONTRIBUTING.md`](CONTRIBUTING.md), then read the six invariants in
[`AGENT_HANDOFF.md`](AGENT_HANDOFF.md) §4. Adding a provider has an eight-step checklist in
[`PROVIDERS.md`](PROVIDERS.md) §4 — a provider is not "supported" until every step is done.

## Licence

MIT — see [`LICENSE`](LICENSE). Third-party attribution and the provenance of the design ideas
studied from the reference project are recorded in
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
