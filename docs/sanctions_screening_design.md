# Sanctions Screening — Insertion Point & Design

## 1. Pipeline Position

Derived from live `get_compliance_pipeline` MCP call against the banking-api server:

Current pipeline: `kyc_check → aml_analysis → [NEW: sanctions_screening] → approval`

| Step | Name | Description |
|------|------|-------------|
| 1 | `kyc_check` | Verify customer identity |
| 2 | `aml_analysis` | Anti-money laundering screening |
| **2.5** | **`sanctions_screening`** | **Real-time OFAC/UN sanctions list check (NEW)** |
| 3 | `approval` | Final compliance approval |

**Rationale:** Sanctions screening must run after AML analysis (which confirms the transaction is not structurally suspicious) and before approval. The OFAC/UN list check is a hard-stop gate — any match with `confidence >= 0.75` must be routed to manual review before an approval decision is made.

## 2. Data Inputs Required

| Field | Type | Source | Notes |
|-------|------|--------|-------|
| `customer_id` | string | Banking platform transaction context | Internal customer identifier; must be masked in logs (PCI DSS Req 3) |
| `counterparty_details` | object | Transaction payload | Must include `name` key; optional: `country`, `account_number` |
| `transaction_amount` | float | Transaction payload | Used for risk-weighting (high-value transactions require elevated confidence threshold); not used in name-match logic |

## 3. Output Schema

```json
{
  "screened": true,
  "match_list": [
    {
      "list_source": "OFAC",
      "matched_name": "Acme Trading Ltd",
      "score": 0.91
    }
  ],
  "confidence": 0.91
}
```

- `screened` (bool) — `true` if the screening step executed successfully; `false` on API failure or validation error.
- `match_list` (list) — zero or more OFAC/UN matches, each with `list_source` ("OFAC"|"UN"), `matched_name`, and `score` (0.0–1.0).
- `confidence` (float, 0–1) — the highest match score found, or `0.0` if no matches. Drives the approval-gate decision.

**Failure mode:** if the sanctions API is unreachable, `screened` must be `false` and the transaction routed to manual review — never silently passed through.

## 4. Integration Contract with the AML Module

- **Input from AML module:** `aml_result` (status, risk_score) passes through unchanged to the approval step; sanctions screening runs independently of the AML verdict.
- **Output to Approval step:** any non-empty `match_list` with `confidence >= 0.75` is a hard stop requiring manual compliance review before approval may proceed.
- **Calling convention:** `screen_transaction(customer_id: str, counterparty_details: dict, transaction_amount: float) → dict`
- **Error handling:** on `InvalidScreeningInputError` or API unreachable, return `{"screened": False, "match_list": [], "confidence": 0.0}` and route to manual review.
- **Logging:** `customer_id` and counterparty names are masked to first 2 + last 2 characters in all log lines (PCI DSS Req 3).
- **Storage:** screening results are persisted via the encrypted-storage call site (`_store_result_encrypted`) before returning — never stored in plaintext.

## 5. Open Questions / Risks

- **False-positive rate:** fuzzy name matching may flag legitimate counterparties with names similar to sanctioned entities. A `confidence >= 0.75` threshold is used; review cadence should be established with the compliance team.
- **List refresh cadence:** OFAC/UN lists are updated daily; the production implementation must refresh its local cache on a schedule (current lab uses a deterministic stub).
- **Dual-list precedence:** when a counterparty appears on both OFAC and UN lists, `confidence` takes the maximum score across both lists.
