# PROVIDERS.md — ModelPilot provider plan

> **Policy.** A provider appears in the "Supported" table **only** after its adapter has a
> passing hermetic contract test (`tests/contracts/<provider>_test.py`) **and** the integration
> has been exercised against the live endpoint at least once, with the date recorded below.
> An adapter that exists but is untested is listed under "Planned", not "Supported".
>
> Free tiers change without notice. Nothing in this document is a guarantee. Every quota figure
> is treated as *unverified* until the provider itself reports it at runtime.

**Last verified run:** none yet (Phase 3 not started).

---

## 1. Supported providers

*(empty — no adapter has been written or tested yet)*

| Provider | Adapter | Auth | Free tier | Key validation | Model discovery | Quota probe | Last live-verified |
| --- | --- | --- | --- | --- | --- | --- | --- |
| — | — | — | — | — | — | — | — |

---

## 2. Planned providers (Phase 3)

The brief lists a long roster. ModelPilot deliberately sequences it by **shape**, because the
cheapest, most testable integrations come first and each distinct wire format is a separate
class of bug.

### Wave 1 — OpenAI-compatible (one shared adapter, thin per-provider config)

All of these speak the OpenAI `chat/completions` shape. They differ in base URL, headers,
model-id spellings, error bodies, and quirks.

| Provider | Provider id | Base URL (default) | Auth header | Notes / known quirks |
| --- | --- | --- | --- | --- |
| Groq | `groq` | `https://api.groq.com/openai/v1` | `Authorization: Bearer` | Fast inference; free tier is rate-limited rather than quota-limited. |
| Cerebras | `cerebras` | `https://api.cerebras.ai/v1` | `Authorization: Bearer` | Free tier; generous token throughput. |
| NVIDIA NIM | `nvidia_nim` | `https://integrate.api.nvidia.com/v1` | `Authorization: Bearer` | **Rejects parallel tool calls** → the adapter must force `parallel_tool_calls=false` when tools are present. Long cold starts possible (per-provider timeout). |
| Mistral | `mistral` | `https://api.mistral.ai/v1` | `Authorization: Bearer` | Free-tier availability varies by account; probe it, never assume it. |
| OpenRouter | `openrouter` | `https://openrouter.ai/api/v1` | `Authorization: Bearer` | Model ids may carry a `:free` suffix; the suffix is a **tier tag, not part of the model name** → normalize for comparison, never rewrite the id we send. |

### Wave 2 — distinct wire formats

| Provider | Provider id | Wire format | Auth | Notes |
| --- | --- | --- | --- | --- |
| Google AI Studio | `google_ai_studio` | native `generateContent` | API key (query param and/or `x-goog-api-key`) | Different request/response shape **and** error shape (429 carries `error.details[]` with `google.rpc.RetryInfo.retryDelay` like `"17s"`). Long cold starts on some reasoning models → higher default timeout. |
| Hugging Face | `huggingface` | OpenAI-compatible router + native inference endpoints | `Authorization: Bearer` | Some models are on the Inference API and some via the router; discovery is partial. |
| Cloudflare Workers AI | `cloudflare` | OpenAI-compatible endpoint, account-scoped path | `Authorization: Bearer` + account id in the URL | Accounting is in *neurons / request units*, not plain tokens → quota probing must name the unit and must not invent a token equivalence. |
| Cohere | `cohere` | native `chat` | `Authorization: Bearer` | Native message/response shape differs; map carefully and do not lose `finish_reason`. |

### Wave 3 — user-controlled endpoints

| Provider | Provider id | Notes |
| --- | --- | --- |
| Custom OpenAI-compatible | `custom` (user-defined id) | User supplies base URL + optional key + model list. URL validated on add (scheme, no embedded credentials, host resolves). **SSRF note in `SECURITY.md`.** |
| Ollama | `ollama` (via `local_openai`) | Default `http://127.0.0.1:11434/v1`. No key. |
| LM Studio | `lmstudio` (via `local_openai`) | Default `http://127.0.0.1:1234/v1`. No key. |
| llama.cpp server | `llamacpp` (via `local_openai`) | Default `http://127.0.0.1:8080/v1`. No key. |
| vLLM | `vllm` (via `local_openai`) | User-supplied host/port. No key. |

Local endpoints are the easiest to verify end-to-end and are therefore the recommended first
manual validation, even though Groq/Cerebras land first in code (their shape is the one the
shared adapter is built against).

---

## 3. Explicitly out of scope

| Category | Why not |
| --- | --- |
| Media-only gateways (image/video/audio) | ModelPilot is a chat/router for coding agents first. Different transparency semantics (no `model` echo, binary payloads). |
| Search-augmentation APIs | Not model providers; there is no model identity to be transparent about. |
| Providers whose ToS forbid proxying | See §5. |
| Providers requiring a paid plan for any usable access | Not the target use case, and it blurs the quota story. |

---

## 4. Adding a provider — the required checklist

1. Create `src/modelpilot/adapters/<provider>.py` exporting a module-level `register()` that the
   registry picks up. Prefer `OpenAICompatAdapter` with options; write a bespoke adapter only when
   the wire format genuinely differs.
2. Declare the capability flags honestly: `supports_key_validation`, `supports_model_discovery`,
   `supports_quota_probe`, `requires_key`.
3. Map provider errors to `ErrorKind`; extract a back-off from `Retry-After`, structured body
   fields, or prose, and **clamp it to 86400 s**.
4. Never rewrite the upstream `model` value. Return it verbatim as `served_model_raw`.
5. Add `tests/contracts/<provider>_test.py` with `respx` fixtures covering: happy path; 401/403;
   429 with `Retry-After`; 429 without it; 500; malformed JSON; empty body; **a "provider lies"
   case that echoes a different model id**; and a streaming happy path.
6. Seed `src/modelpilot/data/catalog.yaml` with the provider's exact model ids, documented
   context windows, honest (tri-state) capability flags, aliases, and reset policy + timezone.
   Every entry needs `source_url` and a `verified` date.
7. Add the row to §1 of this file with the live-verification date.
8. Run the "add a provider without touching `core/`" architectural test.

**A provider is not supported until all eight steps are done.** Until then it is "Planned".

---

## 5. Free-tier reality check (applies to every entry)

ModelPilot takes no position on whether any provider will keep offering free access. What it
does guarantee is that it will not pretend to know:

- A provider's free tier may be **rate-limited** (requests/minute), **quota-limited**
  (requests or tokens per day), **credit-limited** (a recurring allowance), or none of the
  above — and the limit may change with no notice.
- Where the provider exposes a quota endpoint, ModelPilot records the reading with
  `provenance = provider_api`.
- Where it does not, ModelPilot records **observed** usage, says so in the UI, and lets the
  operator set a manual limit and reset policy.
- Where even the existence of a limit is unknown, the UI says **unknown**. It does not say zero,
  and it does not say plenty.

Operators are responsible for complying with each provider's terms of service. Using a provider
through another client may violate terms that forbid proxying or resale; check before enabling a
provider. This document is informational, not legal advice.

---

## 6. Provider-specific categories ModelPilot will support in its quota model

| Category | Meaning | Example handling |
| --- | --- | --- |
| **RPM / RPD** | Requests per minute / day | Counted locally from `requests`; compared against a documented limit if known. |
| **TPM / TPD** | Tokens per minute / day | Counted from reported `usage`; `NULL` when the provider does not report tokens. |
| **Monthly credits** | A pool consumed by tokens or requests | Tracked as `monthly`; the unit (tokens, credits, neurons) is recorded, never converted silently. |
| **Concurrency** | Simultaneous in-flight requests | Tracked where the provider documents it; surfaced as a separate meter. |
| **Unknown** | Provider documents nothing and answers nothing | Displayed as unknown; operator may set a manual limit. |

Each provider row in §1 will state which categories apply and, crucially, **whether the value is
provider-reported or locally observed**.
