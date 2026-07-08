---
name: xproof
version: 4.2.0
description: "No API key needed. Any agent can anchor a proof and pay per call via x402 — one HTTP request, no account, no setup. Prove Before Act: anchor reasoning (WHY) + planned decision (WHAT) on-chain before execution. Anchor actual result after for a full 4W audit trail on MultiversX. MCP native."
homepage: https://xproof.app
metadata: {"xproof":{"category":"proof,security,compliance,accountability,x402,mcp","api_base":"https://xproof.app","x402":true,"mcp":true,"prove_before_act":true}}
---

# xProof — Prove Before Act for AI Agents

> **No API key needed.** Any agent can anchor a proof and pay per call via x402 — one HTTP request, no account, no setup.

The accountability layer for autonomous agents. Instead of being a black box, your agent becomes transparent, auditable, and verifiable.

**Prove Before Act**: anchor reasoning (WHY) + planned decision/intention (WHAT) on-chain *before* execution. Anchor actual result/output *after* for a full 4W audit trail. Hash is computed locally — no raw content ever leaves the agent.

**Proven in production**: xproof_agent_verify (Moltbook) — 4,418 on-chain proofs, 100% confirmation rate, 16-week streak, trust score 43,326. $0.01/proof, ~$2.76/week for a continuously accountable agent.

---

## Quick Start — 3 steps

```bash
# 1. Get API key — no wallet, no card (10 free proofs)
curl -X POST https://xproof.app/api/agent/register -H "Content-Type: application/json" -d '{"agent_name": "my-agent"}'
# → { "api_key": "pm_...", "trial": { "quota": 10 } }

# 2. Hash reasoning locally (nothing leaves your machine)
FILE_HASH=$(python3 -c "import hashlib,json; print(hashlib.sha256(json.dumps({'why':'RSI=38','what':'BUY BTC'},sort_keys=True).encode()).hexdigest())")

# 3. Anchor BEFORE executing — Prove Before Act
curl -X POST https://xproof.app/api/proof -H "Authorization: Bearer pm_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d "{\"file_hash\":\"$FILE_HASH\",\"filename\":\"reasoning.json\",\"metadata\":{\"who\":\"my-agent\",\"what\":\"BUY BTC\",\"why\":\"RSI=38\"}}"
# → { "proof_id": "prf_...", "verify_url": "/proof/...", "status": "pending" }
```

---

## Going to production? — 4 required patterns

- **Batch anchoring** — POST /api/batch, up to 100 files per call, 50× fewer requests than one-by-one
- **Retry policy** — exponential backoff (1s→2s→4s), handle 409 dedup, respect Retry-After on 429
- **Monitoring** — alert if daily proof volume drops below expected minimum
- **No proof = no action** — for high-stakes agents (trading, legal, medical): block action if anchoring fails after 3 retries

```bash
# Batch anchoring — 1 call instead of N
curl -X POST https://xproof.app/api/batch -H "Authorization: Bearer pm_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"certifications":[{"file_hash":"hash1","filename":"action1.json"},{"file_hash":"hash2","filename":"action2.json"}]}'
# → {"results":[{"proof_id":"prf_...","status":"pending"},{"proof_id":"prf_...","status":"pending"}]}
```

```python
def anchor_with_retry(file_hash: str, filename: str, api_key: str, max_retries=3):
    """Production-grade anchor — retry + no proof = no action gate."""
    import time, requests
    backoff = [1, 2, 4]
    for attempt in range(max_retries):
        try:
            resp = requests.post("https://xproof.app/api/proof",
                headers={"Authorization": f"Bearer {api_key}"},
                json={"file_hash": file_hash, "filename": filename}, timeout=10)
            if resp.status_code == 200: return resp.json()["proof_id"]
            if resp.status_code == 409: return resp.json()["existing_proof_id"]
            if resp.status_code == 429: time.sleep(int(resp.headers.get("Retry-After", 5))); continue
            if resp.status_code >= 500: time.sleep(backoff[attempt]); continue
        except requests.Timeout:
            time.sleep(backoff[attempt])
    raise PolicyError("Proof anchoring failed — action blocked (no proof = no action)")
```

---

## Use-case examples — copy-paste ready

| Use case | Domain | What gets anchored |
|----------|--------|--------------------|
| **Trading agent** | Finance · High-value decisions | BUY/SELL reasoning before order execution |
| **Research agent** | Content · Reports · Analysis | Reasoning + sources before publishing |
| **Support agent** | Customer service · Compliance | Decision rationale before sending response |

### Trading agent — Finance · High-value decisions

Prove a BUY/SELL decision before executing — full 4W audit trail anchored on-chain.

```python
import hashlib, json, requests

# 1. Document your reasoning
reasoning = {
    "who": "trading-agent-v2", "what": "BUY BTC 0.5",
    "why": "RSI=38 (below 40 threshold); allocation=2.1% (below 3% cap)",
    "model": "gpt-4o-mini", "session_id": "sess_001"
}
h = hashlib.sha256(json.dumps(reasoning, sort_keys=True).encode()).hexdigest()

# 2. Anchor BEFORE executing — Prove Before Act
resp = requests.post("https://xproof.app/api/proof",
    headers={"Authorization": "Bearer pm_YOUR_KEY"},
    json={"file_hash": h, "filename": "trade_decision.json", "metadata": reasoning})
proof_id = resp.json()["proof_id"]  # returned in ~1.1s, on-chain in ~6s

# 3. Execute only after proof is anchored
execute_trade("BUY", "BTC", 0.5)
print(f"Audit trail: https://xproof.app/proof/{proof_id}")
```

### Research agent — Content · Reports · Analysis

Anchor reasoning + sources before publishing — verifiable provenance for readers.

```python
import hashlib, json, requests

# 1. Summarize reasoning and sources
reasoning = {
    "who": "research-agent-v1", "what": "Publish Q2 crypto market outlook",
    "why": "5 sources reviewed, confidence=0.87, no contradictions detected",
    "sources": ["arxiv:2406.12345", "bloomberg:BTC-Q2", "coindesk:2026-07-01"]
}
h = hashlib.sha256(json.dumps(reasoning, sort_keys=True).encode()).hexdigest()

# 2. Anchor hash — report content never leaves the agent
resp = requests.post("https://xproof.app/api/proof",
    headers={"Authorization": "Bearer pm_YOUR_KEY"},
    json={"file_hash": h, "filename": "research_reasoning.json", "metadata": reasoning})
proof_id = resp.json()["proof_id"]

# 3. Publish with verifiable provenance link
publish_report(report_content, audit_ref=proof_id)
print(f"Readers can verify: https://xproof.app/proof/{proof_id}")
```

### Support agent — Customer service · Compliance

Certify decision before sending response — dispute-proof audit record.

```python
import hashlib, json, requests

# 1. Document the decision rationale
decision = {
    "who": "support-agent-v3", "what": "Refund $47.50 approved",
    "why": "Policy §3.2: purchase <30 days, credits unused, first request",
    "ticket_id": "TKT-98231", "confidence": 0.95
}
h = hashlib.sha256(json.dumps(decision, sort_keys=True).encode()).hexdigest()

# 2. Certify before sending — creates dispute-proof audit record
resp = requests.post("https://xproof.app/api/proof",
    headers={"Authorization": "Bearer pm_YOUR_KEY"},
    json={"file_hash": h, "filename": "support_decision.json", "metadata": decision})
proof_id = resp.json()["proof_id"]

# 3. Send response with proof_id as audit reference
send_to_customer(ticket_id, response_text, audit_ref=proof_id)
```

---

## Why it matters in 2026

- **AI Act compliance** — traceable decision audit trail before every action
- **Legal & reputational accountability** — immutable evidence of what the agent decided and why
- **User & enterprise trust** — public verifiable proofs, not internal logs anyone can edit

---

## Key features

- **Privacy-first** — only SHA-256 hash anchored on-chain, raw content stays local
- **x402 native** — pay per call with USDC on Base, no API key, no account, no human
- **MCP compatible** — JSON-RPC 2.0 endpoint, tools: `certify_file`, `audit_agent_session`, `investigate_proof`, `register_trial`
- **4W audit trail** — Who, What, When, Why — publicly verifiable at `xproof.app/proof/{id}`
- **Free trial** — 10 free proofs via `register_trial` MCP tool or `POST /api/agent/register`, no wallet

---

## Install

```bash
# Via Hermes
hermes skills install clawhub/xproof

# Via OpenClaw
openclaw skills install xproof
```

---

## Code examples

### 1. Basic (Python + API key)

```python
from xproof import xproof

async def prove_before_act(task):
    reasoning = await agent.think(task)

    # Prove Before Act — anchor reasoning BEFORE execution
    proof = await xproof.anchor(
        content=reasoning,
        metadata={
            "who": "my-agent-v1",
            "what": f"execute task: {task}",
            "why": reasoning,
            "purpose": "decision_audit"
        }
    )

    # Only execute after proof is confirmed
    result = await agent.execute(task, proof_id=proof.id)
    return {"result": result, "proof": proof.verify_url}
```

### 2. x402 — no API key, fully autonomous

```python
import hashlib, json, base64, requests

def anchor_x402(reasoning: dict, filename: str, wallet_signer) -> dict:
    content = json.dumps(reasoning, sort_keys=True).encode()
    file_hash = hashlib.sha256(content).hexdigest()
    payload = {"file_hash": file_hash, "filename": filename}

    r = requests.post("https://xproof.app/api/proof", json=payload)
    assert r.status_code == 402
    payment_info = r.json()["payment"]
    signed = wallet_signer.sign_x402(payment_info)
    x_payment = base64.b64encode(json.dumps(signed).encode()).decode()

    proof = requests.post("https://xproof.app/api/proof",
        headers={"X-PAYMENT": x_payment}, json=payload)
    return proof.json()
```

### 3. MCP (Claude / Cursor / any MCP client)

```json
{
  "mcpServers": {
    "xproof": {
      "url": "https://xproof.app/mcp",
      "headers": { "Authorization": "Bearer pm_YOUR_KEY" }
    }
  }
}
```

Available tools: `certify_file`, `audit_agent_session`, `verify_proof`, `investigate_proof`, `register_trial`

### 4. Full MCP server integration

```python
from mcp import MCPServer, Tool
from xproof import xproof_x402

class xProofTool(Tool):
    name = "xproof_anchor"
    description = "Anchor agent reasoning (WHY) and planned action (WHAT) BEFORE execution. No API key — pays via x402."

    async def execute(self, reasoning: str, action: str, who: str):
        proof = await xproof_x402.anchor(
            content=f"Reasoning:\n{reasoning}\n\nAction:\n{action}",
            metadata={"who": who, "what": action, "why": reasoning},
            payment={"method": "x402", "network": "base", "max_amount_usd": 0.01}
        )
        return {
            "status": "anchored",
            "proof_id": proof.id,
            "verify_url": proof.verify_url,
            "message": "Reasoning anchored. Proceed with action."
        }

server = MCPServer("xproof-accountability")
server.register_tool(xProofTool())
```

---

## Pricing

| Pricing | Amount |
|---------|--------|
| Per proof | $0.01 (flat) |
| 100 proofs | $1 |
| 1,000 proofs | $10 |
| 10,000 proofs | $100 |

Free trial: 10 proofs via `register_trial` MCP tool — no wallet, no credit card.

---

## Links

- Platform: https://xproof.app
- Agent context doc: https://xproof.app/agent-context
- MCP endpoint: https://xproof.app/mcp
- Live agent profile (Moltbook): https://xproof.app/agent/erd1hlx4xanncp2wm9aly2q6ywuthl2q9jwe9sxvxpx4gg62zcrvd0uqr8gyu9
- x402 guide: https://xproof.app/agent-context#x402
