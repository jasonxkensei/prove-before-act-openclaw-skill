---
name: xproof
version: 4.3.0
description: Prove Before Act — anchor reasoning (WHY) + planned action (WHAT) on-chain before execution. Public 4W audit trail, trust score, violations layer. REST API, MCP, x402. $0.01/proof flat. No proof = no action.
homepage: https://xproof.app
metadata: {"xproof":{"category":"proof,security,compliance,accountability,prove-before-act","api_base":"https://xproof.app"}}
---

# xProof — Prove Before Act

**Any agent. No account required. $0.01/proof flat.**

Hash reasoning locally → anchor on MultiversX → get immutable proof_id → execute. Full 4W audit trail. Public verification. Violations layer. Trust score.

```bash
openclaw skills install @jasonxkensei/xproof
```

---

## The Core Pattern

```
Hash (WHY) → Anchor → proof_id → Execute (WHAT) → Hash (result) → Anchor
```

Pre-execution commitment + post-execution result = tamper-proof 4W audit trail (Who / What / When / Why).

---

## Pricing

**$0.01 per proof — flat. No tiers, no volume discounts.**

| Scale | Cost |
|:---|:---|
| 1 proof | $0.01 |
| 1,000 proofs | $10 |
| 50 agents × 20 actions/day × 30 days | $300/month |

Payment: USDC on Base (x402, no account) · EGLD on MultiversX (ACP/wallet) · prepaid credits.

---

## Quick Start

```bash
# Option A — MCP register_trial (fastest — if already in an MCP session)
# No Authorization header needed. register_trial requires no auth.
{"name": "register_trial", "arguments": {"agent_name": "my-agent-001"}}
# → { "api_key": "pm_...", "trial_remaining": 10, "next_step": "..." }

# Option B — REST (one curl, no wallet, no card)
curl -X POST https://xproof.app/api/agent/register \
  -H "Content-Type: application/json" \
  -d '{"agent_name": "my-agent-001"}'
# → { "api_key": "pm_...", "trial": { "quota": 10, "remaining": 10 } }
# Store pm_ key securely — treat it like a password, never log or echo it.

# Option C — x402 (USDC on Base, no account, pay per proof)
# No registration needed. POST /api/proof → receive 402 → pay → resend.
# WARNING: autonomous payments. Set a spending cap before enabling in production.

# Option D — Prepaid credits (wallet login)
# Dashboard: https://xproof.app/dashboard
```

```bash
# Reference files (load once at install time, not at runtime)
mkdir -p .agent/skills/xproof/references
for f in certification x402 mcp; do
  curl -sL "https://raw.githubusercontent.com/jasonxkensei/xproof-openclaw-skill/main/xproof/references/${f}.md" \
    > ".agent/skills/xproof/references/${f}.md"
done
```

---

## ⚠️ Security & Privacy

**Read this section in full before enabling this skill in any agent.**

### What leaves your environment

xproof receives only the fields you explicitly send: `file_hash` (SHA-256 hex), `filename` (a label you choose), and optional `metadata` fields. **No raw content, no documents, no binary data are ever transmitted.** Hash computation must happen locally using `hashlib.sha256`, `crypto.subtle.digest`, `sha256sum`, or equivalent before any API call.

The metadata fields (`why`, `what`, `model_hash`, etc.) are under your control. You decide what goes in them. Choose carefully:

- **Safe to include**: opaque IDs, hashed summaries, strategy labels, model version strings, decision category codes.
- **Never include**: raw prompts, chain-of-thought text, user PII, secrets, API keys, proprietary strategy details, confidential filenames, or regulated data. Even when only a hash is anchored on-chain, metadata submitted alongside it is stored and may be returned in public API responses.

### Proof visibility and public records

- **Trial users** (`register_trial` / `POST /api/agent/register`): proofs are **public by default** (`is_public: true`). Anyone can look up your proof by hash or proof_id. Pass `"is_public": false` in the request body to create a private proof even during trial.
- **Authenticated users** (prepaid credits, wallet): proofs default to `is_public: false`. Set `"is_public": true` only when public verifiability is intentional.
- Once a proof is anchored on-chain, its blockchain record is permanent and cannot be deleted. Only the off-chain metadata visibility can be toggled.

### API key (`pm_`) security

A `pm_` key is a bearer credential — treat it exactly like a database password:

- Store it in a secret manager (env var, vault, secret store) — never hardcode it in source code.
- Never log it, echo it in chat output, persist it in agent memory files, or include it in trace exports.
- When `register_trial` returns a key in its response, redact or discard that value from any transcript or log immediately after storing it securely.
- Rotate the key immediately if you suspect it was exposed.

### `audit_agent_session` — external persistent transmission

> **This tool transmits complete session content (input, reasoning steps, output) to a durable, external record at xproof.app.**

Key implications:
- The record is designed to be immutable. Submitted session data cannot be deleted or amended after anchoring.
- Do not use this tool if your session contains user PII, secrets, regulated data (HIPAA, GDPR-sensitive), proprietary strategy, or confidential instructions.
- Prefer submitting a SHA-256 hash of the session + a high-level label rather than raw session content.
- Operator opt-in required: only call this tool when your deployment operator has explicitly configured session-level audit logging for compliance purposes.

### x402 autonomous payments

x402 enables your agent to initiate USDC payments on Base without per-transaction human confirmation. This is intentional by design for autonomous agents, but requires explicit safeguards:

- **Set a per-session spending cap** in your agent framework before enabling x402. Without a cap, a looping agent can trigger repeated payments.
- **Require human approval above your threshold** (e.g. >$1 per session) before enabling x402 in production.
- x402 is opt-in: if your agent uses `Authorization: Bearer pm_...`, it will not trigger x402 payments. x402 only activates on unauthenticated calls that receive a `402` response and proceed to sign a payment payload.
- Isolate the wallet used for x402 from your main treasury. Use a dedicated wallet with a limited USDC balance.

### Tool invocation scope

This skill's tools should be invoked **only when your operator configuration explicitly requires proof anchoring** for a given action type. Do not invoke `certify_file` or `audit_agent_session` by default on every action. Define a policy in your operator config specifying which action categories require anchoring (e.g. financial operations, data writes, external API calls) and gate invocation on that policy check.

### Other security rules

- **NEVER** commit API keys to a public repository. **ALWAYS** add `.env` to your `.gitignore`.
- **x402 and MCP endpoints perform real network calls** to xproof.app. In air-gapped or restricted environments, verify that outbound HTTPS to `xproof.app` is permitted.
- **`llms.txt` and `llms-full.txt`** are static documentation references — load them once at install time, not at runtime on every call. Dynamic fetching creates a prompt-injection surface if the file is ever compromised upstream.

---

## Core Tools

### `certify_file` — Anchor a hash on-chain

**Invoke only when your operator policy requires proof anchoring for this action type.**

```python
import hashlib, json

# Step 1: compute hash LOCALLY — never send raw content
content = json.dumps({"decision": "execute_trade", "rationale_id": "r_abc123"},
                     sort_keys=True).encode()
file_hash = hashlib.sha256(content).hexdigest()

# Step 2: anchor hash + opaque metadata (no raw reasoning text)
proof = client.certify(
    file_hash=file_hash,
    filename="decision_2026_001.json",
    metadata={
        "role": "WHY",
        "action_type": "trade_execution",
        "model": "gpt-4o-mini",
        # DO NOT put raw prompt, strategy text, or PII here
    },
    is_public=False   # default to private unless public verifiability is required
)
proof_id = proof["proof_id"]
```

### `audit_agent_session` — Log a complete session

> **⚠️ Transmits complete session data to a durable external record. Operator opt-in required. Prefer hashing session content and submitting the hash via `certify_file`.**

```python
# Safer alternative: hash the session, anchor the hash
session_hash = hashlib.sha256(json.dumps(session_data, sort_keys=True).encode()).hexdigest()
proof = client.certify(file_hash=session_hash, filename="session_audit.json",
                       metadata={"session_id": "s_xyz", "agent": "my-agent"})
```

### `investigate_proof` — Look up an existing proof

Read-only. No records created, no payments triggered.

```bash
curl https://xproof.app/api/proof/<proof_id>
```

### `register_trial` — Get a free pm_ key (10 proofs, no wallet)

```bash
curl -X POST https://xproof.app/api/agent/register \
  -H "Content-Type: application/json" \
  -d '{"agent_name": "my-agent"}'
# { "api_key": "pm_...", "trial": { "quota": 10, "remaining": 10 } }
# ↑ treat this value as a secret — do not echo, log, or persist in plaintext
```

---

## 4W Audit Trail

| Dimension | Anchor Point | What to hash |
|:---|:---|:---|
| **WHO** | at session start | agent identity / wallet address |
| **WHAT** | after execution | sha256(actual output or result) |
| **WHEN** | blockchain timestamp | automatic — `certified_at` field |
| **WHY** | before execution | sha256(reasoning summary + action label) |

---

## Trust Score & Violations

```bash
GET https://xproof.app/api/agents/{wallet}               # trust profile (public)
GET https://xproof.app/api/agents/{wallet}/violations    # violation audit
GET https://xproof.app/api/agents/{wallet}/incident-report?proof_id={id}  # 4W reconstruction
```

Trust score is computed from certification count, streak, and violation history. Violations are recorded automatically when structural anomalies are detected (time-ordering gaps, missing pre-execution anchors, burst patterns).

---

## Command Cheatsheet

```bash
# Hash locally — original content must never leave your environment
sha256sum myfile.pdf | awk '{print $1}'

# Anchor via REST (is_public: false recommended for non-trial)
curl -X POST https://xproof.app/api/proof \
  -H "Authorization: Bearer pm_..." \
  -H "Content-Type: application/json" \
  -d '{"file_hash":"<sha256hex>","filename":"action_001.json","is_public":false}'

# Anchor via MCP
curl -X POST https://xproof.app/mcp \
  -H "Authorization: Bearer pm_..." \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"certify_file","arguments":{"file_hash":"...","filename":"action.json","is_public":false}}}'

# Verify
curl https://xproof.app/api/proof/<proof_id>

# Badge (embed in README)
![xProof](https://xproof.app/badge/<proof_id>)

# Batch (max 100 per call)
curl -X POST https://xproof.app/api/batch \
  -H "Authorization: Bearer pm_..." \
  -d '{"files":[{"file_hash":"...","filename":"a.json"},{"file_hash":"...","filename":"b.json"}]}'

# Health
curl https://xproof.app/api/acp/health
```

---

## Discovery Endpoints

| Endpoint | Description |
|:---|:---|
| `GET /.well-known/agent.json` | Agent Protocol manifest |
| `GET /.well-known/mcp.json` | MCP server manifest |
| `GET /ai-plugin.json` | OpenAI ChatGPT plugin manifest |
| `GET /llms.txt` | LLM-friendly summary |
| `GET /llms-full.txt` | Complete LLM reference |
| `POST /mcp` | MCP JSON-RPC 2.0 endpoint |
| `GET /api/standard` | Agent Proof Standard specification |

Full specification: [AGENT_PROOF_STANDARD.md](https://github.com/jasonxkensei/xProof/blob/main/AGENT_PROOF_STANDARD.md)

