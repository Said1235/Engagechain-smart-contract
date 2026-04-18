# ◈ EngageChain — Intelligent Contract

> A GenLayer Intelligent Contract that validates decisions and opinions using decentralized AI consensus. Multiple validator nodes independently call an LLM, compare results for semantic equivalence, and store the agreed verdict permanently on-chain.

**Network:** GenLayer Studionet  
**Language:** Python + GenLayer SDK  
**Contract address:** `0x99e24b7246DeD27634bBD8659b4D9915ac700EdD`

---

## What it does

EngageChain records any text input on-chain, triggers LLM-based analysis through GenLayer's Optimistic Democracy consensus mechanism, and finalizes a trustless verdict — all without a trusted third party.

The contract supports two validation paths:

- **GenLayer AI** — the built-in LLM runs on every validator node. Results are compared for semantic equivalence before being stored.
- **External AI** — the caller provides their own AI analysis. GenLayer validators verify the JSON structure is valid and normalize it to the standard schema before storing.

---

## State Machine

Every opinion follows a strict three-stage lifecycle enforced by on-chain assertions:

```
submit_opinion()          evaluate_opinion()            finalize_opinion()
      │                         │                              │
      ▼                         ▼                              ▼
  [ pending ]  ──────────▶  [ evaluated ]  ──────────▶  [ finalized ]
```

Calling `finalize_opinion` on a `pending` opinion will revert. Calling `evaluate_opinion` on an already-evaluated opinion will revert. The order is enforced by the contract — not the client.

---

## Storage

All persistent fields are `TreeMap[str, str]`. GenLayer does not support `int` or `float` in storage — only `str`, `bool`, `bigint`, and sized integers (`u8`, `u16`, `u32`, `u64`).

| Field | Type | Description |
|-------|------|-------------|
| `submissions` | `TreeMap[str, str]` | `opinion_id → raw text` |
| `ai_responses` | `TreeMap[str, str]` | `opinion_id → AI result as JSON string` |
| `verdicts` | `TreeMap[str, str]` | `opinion_id → final verdict string` |
| `authors` | `TreeMap[str, str]` | `opinion_id → wallet address` |
| `statuses` | `TreeMap[str, str]` | `opinion_id → pending \| evaluated \| finalized` |
| `sources` | `TreeMap[str, str]` | `opinion_id → genlayer \| external` |
| `metadata` | `TreeMap[str, str]` | `opinion_id → arbitrary JSON string` |
| `next_id` | `str` | Auto-increment counter stored as string |

---

## Methods

### Write Methods

Write methods modify state, require a wallet signature, and cost gas.

---

#### `submit_opinion(text, metadata="")`

Records a new opinion on-chain.

| Parameter | Type | Required | Constraints |
|-----------|------|----------|-------------|
| `text` | `str` | Yes | 1–2000 characters |
| `metadata` | `str` | No | Max 4000 characters. Pass any JSON string for custom fields. |

**Returns:** `opinion_id` (string) — included in the transaction receipt.

**Sets:** `status = "pending"`, `source = "genlayer"`

**Example — minimal:**
```python
contract.submit_opinion("Decentralized AI is the future of trust")
# → "0"
```

**Example — with metadata:**
```python
contract.submit_opinion(
    "This proposal should be approved by the DAO",
    '{"category": "governance", "priority": "high", "app": "my-dapp"}'
)
# → "1"
```

---

#### `evaluate_opinion(opinion_id)`

Runs LLM analysis on a submitted opinion using multi-validator consensus.

Each validator node independently calls `gl.nondet.exec_prompt()` with the same prompt. `gl.eq_principle.strict_eq` coordinates consensus — validators compare results for semantic equivalence before the agreed output is stored.

| Parameter | Type | Required |
|-----------|------|----------|
| `opinion_id` | `str` | Yes |

**Requires:** `status == "pending"`  
**Sets:** `status = "evaluated"`, `source = "genlayer"`  
**Returns:** AI result dict (also stored in `ai_responses[opinion_id]`)

**AI response schema:**
```json
{
  "summary":           "brief summary in 1-2 sentences",
  "sentiment":         "positive | negative | neutral | mixed",
  "category":          "proposal | opinion | dispute | question | other",
  "key_points":        ["point 1", "point 2", "point 3"],
  "ai_recommendation": "concrete recommendation or verdict",
  "confidence_score":  "0.85"
}
```

> **⚠️ `confidence_score` is always a string.** GenLayer's calldata encoder cannot encode Python `float`. The contract casts it with `str(parsed.get("confidence_score", "0"))` before returning. If the LLM ignores the instruction and returns a bare number, this cast prevents the transaction from failing.

**⏱ Timing:** 30–120 seconds. AI consensus is not instant. Wait for `FINALIZED` status.

---

#### `submit_with_external_ai(opinion_id, external_analysis)`

Validates an opinion using the caller's own AI analysis instead of GenLayer's LLM.

GenLayer validators do not re-run the AI. They verify that `external_analysis` is valid JSON matching the expected schema, normalize any missing or misnamed fields to defaults, and store the result on-chain with `source = "external"`.

| Parameter | Type | Required | Constraints |
|-----------|------|----------|-------------|
| `opinion_id` | `str` | Yes | Must be `pending` |
| `external_analysis` | `str` | Yes | Valid JSON string, max 10000 chars |

**Accepted field aliases:**
- `recommendation` or `verdict` → mapped to `ai_recommendation`
- `points` → mapped to `key_points`
- Missing fields → filled with sensible defaults

**Sets:** `status = "evaluated"`, `source = "external"`

**Example:**
```json
{
  "summary": "The proposal is well-structured and actionable.",
  "sentiment": "positive",
  "category": "proposal",
  "key_points": ["Clear scope", "Feasible timeline", "Strong rationale"],
  "ai_recommendation": "Approve",
  "confidence_score": "0.92"
}
```

---

#### `finalize_opinion(opinion_id, verdict)`

Records the final verdict on-chain. The opinion lifecycle is complete after this call — no further writes are possible on this ID.

| Parameter | Type | Required |
|-----------|------|----------|
| `opinion_id` | `str` | Yes |
| `verdict` | `str` | Yes, non-empty |

**Requires:** `status == "evaluated"`  
**Sets:** `status = "finalized"`  
**Returns:** `{"id": opinion_id, "status": "finalized"}`

---

### View Methods

View methods are read-only. They require no gas, no wallet, and can be called by any address including the zero address via `gen_call`.

All view methods return `typing.Any` to avoid the `"Value must be an instance of str"` error that `genlayer-js` throws when strict-checking annotated return types.

---

#### `get_resolution_data(opinion_id)`

Returns the complete on-chain record for one opinion.

```json
{
  "id":          "0",
  "text":        "Decentralized AI is the future of trust",
  "ai_response": "{\"summary\": \"...\", \"sentiment\": \"positive\", ...}",
  "verdict":     "Approved",
  "status":      "finalized",
  "author":      "0xabc...def",
  "source":      "genlayer",
  "metadata":    "{\"category\": \"governance\"}"
}
```

Note: `ai_response` is stored as a JSON string. Parse it client-side with `JSON.parse()`.

---

#### `get_all_opinions()`

Returns all submitted opinions as `{opinion_id: text}`.

```json
{
  "0": "Decentralized AI is the future of trust",
  "1": "This proposal should be approved by the DAO"
}
```

---

#### `get_total_submissions()`

Returns the total number of submissions as a string.

```json
"2"
```

---

#### `get_status(opinion_id)`

Returns just the status of one opinion: `"pending"`, `"evaluated"`, or `"finalized"`.

---

## Calling the contract

### Reading (no wallet, no gas)

```javascript
const res = await fetch('https://studio.genlayer.com:8443/api', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    jsonrpc: '2.0', id: 1, method: 'gen_call',
    params: [{
      from:   '0x0000000000000000000000000000000000000000',
      to:     '0x99e24b7246DeD27634bBD8659b4D9915ac700EdD',
      type:   'read',
      data:   { function: 'get_resolution_data', args: ['0'] },
      status: 'accepted',
    }]
  })
});
const { result } = await res.json();
```

### Writing (requires wallet + genlayer-js)

```javascript
import { createClient, createAccount } from 'genlayer-js';
import { studionet } from 'genlayer-js/chains';

const client = createClient({
  chain:    studionet,
  account:  createAccount(),          // or use MetaMask address
  endpoint: 'https://studio.genlayer.com:8443/api',
});

const txHash = await client.writeContract({
  address:      '0x99e24b7246DeD27634bBD8659b4D9915ac700EdD',
  functionName: 'submit_opinion',
  args:         ['My opinion text', '{"app":"my-dapp"}'],
  value:        BigInt(0),
});

const receipt = await client.waitForTransactionReceipt({
  hash: txHash, status: 'FINALIZED', interval: 3000, retries: 120,
});

const opinionId = String(receipt.result);
```

---

## Deploying your own instance

1. Open [studio.genlayer.com](https://studio.genlayer.com)
2. Click **New Contract** → paste `engagechain.py`
3. Click **Deploy**
4. Copy the contract address from the deployment panel

> Studionet accounts are auto-created and auto-funded. No wallet setup or real tokens required.

---

## Known constraints

| Constraint | Detail |
|------------|--------|
| `float` not encodable | GenLayer calldata only supports `str`, `int`, `bool`, `list`, `dict`. The contract casts `confidence_score` to `str` to prevent `TypeError: not calldata encodable 0.85: float` |
| `int` not in storage | Storage declarations cannot use plain Python `int`. Use `str` or sized integers (`u64`, etc.) |
| Read storage before `nondet` | `self.field` access inside `def get_analysis()` may be unreliable. Always read to a local variable before entering the nondet block |
| `typing.Any` on views | View methods annotated `-> str` cause `"Value must be an instance of str"` in `genlayer-js`. Use `-> typing.Any` instead |
| Consensus time | `evaluate_opinion` and `submit_with_external_ai` both use `gl.eq_principle.strict_eq`. Expect 30–120 seconds for `FINALIZED` |

---

## Related

- [EngageChain App](https://engagechain.vercel.app)
- [API Demo](https://engagechain-api-use.vercel.app)
- [API Repository](https://github.com/Said1235/Engagechain-api)
- [Documentation](https://engagechaindocs.netlify.app)
- [GenLayer Docs](https://docs.genlayer.com)
- [GenLayer Studio](https://studio.genlayer.com)
