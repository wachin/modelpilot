# SECURITY.md — ModelPilot

ModelPilot runs locally, holds **your provider API keys**, and exposes an endpoint that a coding
agent will send source code through. This document states the threat model, the controls, and
the operations you must perform. It is updated whenever a control changes.

**Security contact:** open a private security advisory on the repository
(`https://github.com/wachin/modelpilot` → Security → Advisories). Do not open a public issue for
a vulnerability involving credential handling.

---

## 1. Assets and threat model

| Asset | Why it matters | Primary threat |
| --- | --- | --- |
| Provider API keys | Direct financial/abuse cost if leaked; often tied to a personal account | Exfiltration via the DB, a log, a backup, or an error message |
| The unified API token | Grants use of your endpoint (and your provider quota) | Leaked in an agent config, committed to a dotfile repo, pasted into a chat |
| Prompt/response content | Contains source code and possibly secrets | Stored too aggressively, logged, or exposed to an unintended provider |
| Request/quota history | Reveals what you work on and when | Local file permissions, backups, accidental publication |
| Encryption key material | Protects every stored provider key | Stored beside the ciphertext in a world-readable location |

**In scope:** a local attacker on the same machine with your user account; a malicious or
compromised dependency; a provider returning unexpected data; a coding agent that logs its own
configuration; accidentally committing a data directory.

**Out of scope:** a remote attacker with existing root/Administrator access; physical access with
full-disk encryption disabled; CPU side channels; a compromised provider (we can only report what
it tells us — see invariant I1 in `AGENT_HANDOFF.md`).

---

## 2. Controls

### 2.1 Provider keys at rest

- Encryption: **AES-256-GCM** (authenticated). Every credential row stores `ciphertext`, `nonce`,
  and `key_version` separately.
- Key derivation: **`scrypt`** from a passphrase, with a per-install random salt. Parameters live
  in the source and their measured cost is recorded in §2.7.
- Key material precedence:
  1. `MODELPILOT_SECRET_KEY` environment variable (recommended), or
  2. an auto-generated key file stored **beside, not inside, the database** — because a key
     stored with the ciphertext it protects provides no protection if the data directory is
     copied or committed.
- Key-file handling: written to a temporary file, `fsync`ed, permissioned `0600` **before** the
  rename, then renamed into place atomically. On Windows the ACL is restricted instead.
- Failure mode: a missing or malformed key causes a **fail-fast startup error** with an actionable
  message. ModelPilot never silently generates a replacement key over an existing database,
  because that would render every stored credential undecryptable and could look like data loss.

### 2.2 Decrypt-at-use

A credential is decrypted inside the single call that needs it and discarded immediately after.
No long-lived plaintext cache exists. The credential type exposes plaintext only through a scoped
accessor, so a caller cannot accidentally retain it beyond the request.

### 2.3 Unified token vs provider keys

Different secrets with different primitives, deliberately:

| | Provider credential | Unified API token |
| --- | --- | --- |
| Direction | Outbound (we present it to the provider) | Inbound (a caller presents it to us) |
| Storage | AES-256-GCM ciphertext | SHA-256 hash |
| Recoverable? | Yes, by design (we must send it) | **No** — the plaintext is shown once at creation |
| Visible to a coding agent? | **Never** | Yes (it is the agent's credential) |

A coding agent can never obtain a provider key through ModelPilot. There is no endpoint, header,
error message, or log line that returns one.

### 2.4 Logging and redaction

- A redaction pass runs **at the logging boundary**, so every record is sanitised regardless of
  which module emitted it. Redaction is code, not a review-time habit.
- Redacted: `Authorization` / `x-api-key` / `api-key` header values, `Bearer …` tokens, values
  matching a registered key pattern, and any secret explicitly registered at runtime.
- Provider error bodies are redacted before they are stored on a `requests` row or returned to a
  caller.
- A test asserts that a known test credential never appears in captured log output.

### 2.5 Network exposure

- Default bind: **`127.0.0.1`** only. A fresh install is unreachable from the network.
- Binding to a non-loopback address requires **both** an explicit configuration flag and an admin
  token. Startup refuses otherwise, with a message explaining why. This is a deliberate speed bump:
  the endpoint spends real money and quota, and should never be exposed by accident.
- The admin/dashboard surface is separate from `/v1` and requires the admin token.

### 2.6 Custom endpoint validation (SSRF)

`custom`/local endpoints are user-supplied URLs and can be pointed anywhere, including cloud
metadata services. Validation on add and on use:

- scheme must be `http` or `https`;
- no embedded credentials in the URL;
- the host must resolve;
- an optional allowlist of hosts/ports can be configured, and is recommended if the endpoint is
  shared or scripted;
- loopback and private-range targets are permitted (that is the point — Ollama, LM Studio), but
  are never *implied*: the user must type them;
- documented explicitly: ModelPilot will fetch the URL you configure. Do not configure URLs you
  would not fetch yourself.

### 2.7 Cryptography parameters (measured, not assumed)

| Parameter | Value | Note |
| --- | --- | --- |
| Cipher | AES-256-GCM | 96-bit random nonce per encryption |
| KDF | `scrypt` | `n=2**14`, `r=8`, `p=1` (initial) |
| Derivation time | *to be measured in Phase 2 on the target hardware and recorded here* | Record the measured cost so the parameter choice is auditable rather than folklore. |

### 2.8 Filesystem permissions

| Path | Mode | Contents |
| --- | --- | --- |
| data directory | `0700` | everything below |
| database (`modelpilot.db`) | `0600` | providers, models, quota, requests, encrypted credentials |
| database `-wal` / `-shm` | `0600` | SQLite WAL sidecars — these contain recent data and are easy to forget |
| key file (if used) | `0600` | the KDF passphrase or key material |
| backups | `0600` | ciphertext only |

A `modelpilot doctor` check verifies these and reports any drift.

### 2.9 Backups

- Backups are **explicit**, never automatic.
- A backup contains **ciphertext only** — never a plaintext provider key.
- A backup is restorable only with the same key material. Losing the key means losing the stored
  credentials; this is documented in the install guide so it is not a surprise.
- `modelpilot doctor` warns if the key file is missing while the database contains credentials.

---

## 3. Credential rotation and removal (runbook)

### 3.1 Rotate a provider API key (the provider reissued it)

1. Obtain the new key from the provider.
2. `modelpilot credential remove --provider <id> --label <label>`
3. `modelpilot credential add --provider <id> --label <label>` and paste the new key.
4. `modelpilot provider test <id>` — confirm it works.
5. If the old key is still valid at the provider, **revoke it there**. Removing it from ModelPilot
   does not revoke it upstream.

### 3.2 Rotate the encryption key material

1. `modelpilot credential rotate` — re-encrypts every stored credential under new key material,
   incrementing `key_version`, inside a transaction.
2. Verify: `modelpilot provider test <id>` for each provider, and `modelpilot doctor`.
3. Securely destroy the old key material (and every copy of it).

A rotation must be **atomic**: either every credential is re-encrypted or none is. A partial
rotation is worse than either outcome, and a test asserts this.

### 3.3 Suspected credential leak

1. **Revoke the leaked key at the provider first.** That stops the bleeding; anything local is
   secondary. This is true whether the leak was a provider key or the unified token.
2. If it was a **provider key**: rotate as in §3.1. Then search your history for where it leaked
   (`modelpilot doctor` checks for a tracked secret file; `git log -p` for a committed secret).
3. If it was the **unified token**: `modelpilot token remove <id>` and issue a new one. Update the
   agent configuration.
4. Assume any content that traversed the endpoint during the exposure window is exposed.
5. If the leak was a **committed secret**: rotate it, then remove it from history. Note that
   rewriting Git history is not sufficient on its own — once pushed, assume the secret is public.

### 3.4 Full reset

Delete the data directory. There is no remote copy and no account to close. Document that the
provider keys still exist at the providers and must be revoked there if that is the intent.

---

## 4. What ModelPilot deliberately does NOT do

| Not done | Reason |
| --- | --- |
| Store provider keys in a plaintext config file | The entire point of §2.1. |
| Provide a "reveal my provider key" API | A coding agent shares this endpoint. There is no legitimate need for it. |
| Send telemetry anywhere | Local-first. No phone-home, no crash-reporting endpoint. |
| Bind to `0.0.0.0` by default | A default that spends your quota should be reachable only from your machine. |
| Download a model catalog from the internet | A remote catalog is a trust dependency and an offline-mode failure (`ARCHITECTURE.md` §3.2, D9). |
| Retain prompt/response bodies by default | Only metadata (model, tokens, latency, status) is persisted. Bodies are opt-in, and the setting states the trade-off. |
| Log request or response bodies by default | They contain source code. Opt-in, with a warning. |

---

## 5. Operator checklist

- [ ] Set `MODELPILOT_SECRET_KEY` (rather than relying on an auto-generated key file), or back up
      the key file securely and separately from the database.
- [ ] Confirm the data directory is `0700` and the DB is `0600` (`modelpilot doctor`).
- [ ] Keep the bind address on `127.0.0.1` unless you have a specific reason and have enabled the
      admin token.
- [ ] Verify your `.env` and data directory are gitignored before your first commit.
- [ ] Treat the unified token like a password: it grants use of your provider quota.
- [ ] Re-run `modelpilot doctor` after any change to the bind address, the data directory, or the
      key material.
- [ ] Review each provider's terms of service before enabling it (`PROVIDERS.md` §5).

---

## 6. Reporting a vulnerability

Open a private security advisory on the repository with: affected version/commit, a minimal
reproduction, the impact, and whether credentials are involved. Do not include real API keys in
the report — use the fake adapter (`adapters/fake.py`) to demonstrate.
