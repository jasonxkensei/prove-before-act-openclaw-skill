--s--BCL25,BASE64
name: xproof
version: 4.1.0
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

## Quick Start — 30 seconds to your first proof

```bash
# 1. Get 10 free proofs — no wallet, no card
curl -X POST https://xproof.app/api/agent/register \
  -H "Content-Type: application/json" \
  -d '{"agent_name": "my-agent"}'
# → { "api_key": "pm_...", "trial": { "quota": 10, "remaining": 10 } }

# 2. Hash your reasoning locally — nothing leaves your machine
python3 -c "import hashlib,json; d={'why':'RSI=38, below threshold','what':'BUY BTC 0.5'}; print(hashlib.sha256(json.dumps(d,sort_keys=True).encode()).hexdigest())"
# → a1b2c3...64hex

# 3. Anchor proof BEFORE executing the action (Prove Before Act)
curl -X POST https://xproof.app/api/proof \
  -H "Authorization: Bearer pm_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"file_hash":"a1b2c3...64hex","filename":"reasoning.json","metadata":{"who":"my-agent","what":"BUY BTC 0.5","why":"RSI=38"}}'
# → { "proof_id": "...", "verify_url": "/proof/...", "status": "pending" }
# → Execute your action ONLY after receiving proof_id
```

---

## Use-case examples — copy-paste ready

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
    print(f"Proof: https://xproof.app{proof.verify_url}")
    return result
```

### 2. Via x402 (no API key — fully autonomous)

```python
import hashlib, json, base64, requests

def prove_before_act_x402(reasoning: dict, wallet_signer) -> dict:
    """No API key, no account — pure machine-to-machine."""
    # 1. Hash locally
    file_hash = hashlib.sha256(
        json.dumps(reasoning, sort_keys=True).encode()
    ).hexdigest()

    # 2. POST without auth → HTTP 402 with price ($0.01 USDC on Base)
    r = requests.post("https://xproof.app/api/proof",
        json={"file_hash": file_hash, "filename": "reasoning.json"})
    assert r.status_code == 402  # ← x402 challenge

    # 3. Sign USDC on Base (eip155:8453)
    signed = wallet_signer.sign_x402(r.json()["payment"])
    x_payment = base64.b64encode(json.dumps(signed).encode()).decode()

    # 4. Resend with X-PAYMENT → proof_id returned immediately
    proof = requests.post("https://xproof.app/api/proof",
        headers={"X-PAYMENT": x_payment},
        json={"file_hash": file_hash, "filename": "reasoning.json"})

    return proof.json()  # { proof_id, verify_url }
```

### 3. Via MCP tool call

```json
// Call certify_file before any significant action
{
  "name": "certify_file",
  "arguments": {
    "file_hash": "sha256_of_reasoning",
    "filename": "decision_reasoning.json",
    "metadata": {
      "who": "my-agent-v2",
      "what": "execute trade BUY BTC 0.5",
      "why": "RSI=38, below oversold threshold, risk/reward 1:3",
      "purpose": "prove_before_act"
    }
  }
}
// Returns: { proof_id, verify_url, status: "anchored" }
// → Now execute the action with proof_id attached
```

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
