# MCP Server

xProof exposes a Model Context Protocol (MCP) JSON-RPC 2.0 endpoint for AI agent integration.

## Endpoint

```
POST https://xproof.app/mcp
```

**Authentication:** API Key (`Authorization: Bearer pm_...`)

**Content-Type:** `application/json`

## Available Tools

### `certify_file`

Create a blockchain certification for a file.

> **Irreversible action — read before calling:** Every certification is permanent. The `file_hash`, `filename`, and `author_name` become part of a public, immutable on-chain record that cannot be deleted or modified. Do not certify sensitive filenames, personal identifiers, or correlatable hashes unless public permanent disclosure is acceptable.
>
> **Cost:** $0.01 per proof (flat, USDC on Base via x402 or prepaid credits). Configure a spending cap before enabling autonomous or high-volume certification.

**Parameters:**

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `file_hash` | string | Yes | SHA-256 hash (64 hex characters) |
| `filename` | string | Yes | Original filename with extension |
| `author_name` | string | No | Name of the certifier (default: "AI Agent") |
| `webhook_url` | string | No | HTTPS URL for on-chain confirmation callback |

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "certify_file",
    "arguments": {
      "file_hash": "a1b2c3d4e5f6789012345678901234567890123456789012345678901234abcd",
      "filename": "report.pdf",
      "author_name": "MyAgent"
    }
  }
}
```

**Response:**

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [{
      "type": "text",
      "text": "{\"proof_id\":\"uuid-v4\",\"status\":\"certified\",\"file_hash\":\"a1b2c3...\",\"filename\":\"report.pdf\",\"verify_url\":\"https://xproof.app/proof/uuid-v4\",\"certificate_url\":\"https://xproof.app/api/certificates/uuid-v4.pdf\",\"blockchain\":{\"network\":\"MultiversX\",\"transaction_hash\":\"abc123...\",\"explorer_url\":\"https://explorer.multiversx.com/transactions/abc123...\"},\"timestamp\":\"2026-02-19T12:00:00.000Z\"}"
    }]
  }
}
```

---

### `verify_proof`

Verify an existing certification by UUID.

**Parameters:**

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `proof_id` | string | Yes | UUID of the certification to verify |

**Example:**

```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "verify_proof",
    "arguments": {
      "proof_id": "uuid-v4"
    }
  }
}
```

---

### `get_proof`

Retrieve a proof in structured JSON or Markdown format.

**Parameters:**

| Parameter | Type | Required | Description |
|:---|:---|:---|:---|
| `proof_id` | string | Yes | UUID of the certification |
| `format` | string | No | `json` (default) or `md` |

Use `md` format for LLM consumption -- produces human-readable Markdown with all proof details.

---

### `discover_services`

List xProof capabilities, pricing, and usage guidance. No parameters required.

**Returns:** Service description, pricing ($0.01 per proof flat, USDC on Base or prepaid credits), list of tools, certification triggers, batch API details, and supported protocols.

---

### `certify_with_confidence`

Anchor reasoning and decisions at multiple stages of a decision-making process, recording a confidence level at each step. Builds a staged forensic trail (e.g. WHY at 60% confidence before gathering data, WHAT at 95% after).

**Key parameters:** `file_hash`, `filename`, `confidence_level` (0–100), `stage` (e.g. `reasoning`, `decision`, `execution`), `context` (structured metadata).

**Use for:** staged decision anchoring, risk-gated workflows, multi-step agent pipelines.

---

### `investigate_proof`

Reconstruct the full 4W audit trail for a contested agent action. Returns WHO (agent identity via MX-8004), WHAT (SHA-256 on-chain), WHEN (MultiversX block timestamp), and WHY (decision chain anchored before acting). Includes `intent_preceded_execution` verification flag.

**Key parameters:** `proof_id` (UUID of any proof in the action pair — WHY, WHAT, or heartbeat).

**Use for:** incident investigation, compliance audits, contested-action reconstruction.

---

### `audit_agent_session`

Certify a complete agent session as a single audit log entry. Anchors a structured log covering multiple actions, their reasoning, and outcomes in one call. Designed for compliance primitives requiring session-level rather than action-level granularity.

**Use for:** end-of-session audit anchoring, batch audit logging, regulatory compliance.

---

### `register_trial`

Register an agent and obtain a free trial API key (10 proofs, no wallet, no card). No `Authorization` header required.

**Example:**
```json
{"name": "register_trial", "arguments": {"agent_name": "my-agent"}}
```
**Returns:** `{ "api_key": "pm_...", "trial_remaining": 10 }`

---

## Discovery

| Endpoint | Description |
|:---|:---|
| `GET /mcp` | MCP capability discovery (full tool list) |
| `GET /.well-known/mcp.json` | MCP server manifest |

## LLM Prompt Engineering

When exposing xProof to an LLM, use the **Prove Before & After Act** framing:

> **Skill: Prove Before Act**
> "Before executing any significant action (trade, deployment, data deletion, etc.), hash your reasoning locally and call `certify_file` with the hash to anchor it on-chain. This creates an immutable, public record of your intent BEFORE you act. Cost: $0.01 per proof (USDC on Base). WARNING: every call creates a permanent public blockchain record — do not certify sensitive filenames or correlatable hashes."

> **Skill: Prove After Act**
> "After executing an action, hash the result and call `certify_file` again to anchor WHAT actually happened. Combined with the WHY proof, this gives operators a forensic 4W trail."

> **Skill: Verify**
> "Use `verify_proof` to confirm a specific proof record is certified on-chain. Pass the `proof_id` UUID (returned when you anchored the proof). Returns the blockchain transaction hash, on-chain timestamp, and confirmation status. Note: this verifies a specific proof record by UUID — it does not search by file hash. To look up by hash, use `get_proof` or `GET /api/proof?hash={hash}`."

**Important for agents:** Treat `certify_file` as an irreversible side-effecting action requiring explicit intent, not a routine read. Require human approval or a spending cap before calling autonomously.
