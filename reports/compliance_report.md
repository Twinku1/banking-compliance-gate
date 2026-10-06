# Heritage National Bank — Compliance Report

**Generated:** 2026-10-06  
**Scope:** AI Banking Platform — end-to-end audit log review  
**Audit logs reviewed:** `mcp_tool_calls.jsonl`, `model_routing.jsonl`, `transactions.jsonl`, `batch_results.json`, `bandit_scan.txt`

---

## Executive Summary

The platform's core compliance pipeline (KYC → AML → Sanctions → Approval) is operational and correctly blocks high-risk transactions. All MCP tool calls are audited, model routing rationale is logged for every request, and the static security scan (bandit) finds zero medium+ issues across 387 lines of code. Two high-severity gaps require remediation before production: per-session rate limiting on money-movement operations is entirely absent, and a logic defect in the sanctions check silently passes PARTIAL watchlist matches.

---

## PCI DSS Compliance Status

| Constraint | Status | Evidence |
|---|---|---|
| Secrets from env only | **PASS** | All modules use `os.environ`; `load_dotenv()` in `main.py`; no hardcoded keys found |
| PAN masking to last 4 | **PARTIAL** | `mcp/registry.py _mask()` masks keys containing "card"; keys named `pan`/`pan_number` not covered; transaction log writes raw `transfer` dict without masking |
| Input sanitization | **PASS** | `chatbot.sanitize()` strips non-printable chars and prompt-injection patterns; confirmed by `test_input_sanitization` |
| Per-session rate limiting | **FAIL** | No rate limiting found anywhere in codebase; `transactions.process()` and `chatbot.reply()` are unbounded |
| Audited money movement | **PARTIAL** | All transactions written to `transactions.jsonl`; no timestamps in any record; REJECTED records omit `amount`/`from_account`/`to_account` |

---

## Compliance Pipeline Coverage

Based on `audit_log/transactions.jsonl` (6 records across 4 unique request IDs):

| Request | Amount | Country | KYC | AML | Sanctions | Approval | Final Status |
|---|---|---|---|---|---|---|---|
| R1001 | $750.00 | IN | PASS | LOW | NONE | PASS | **SETTLED** |
| R1002 | $5,000.00 | RU | PASS | HIGH (geo_mismatch) | — blocked | — blocked | **REJECTED** |
| REDGE | $10,000,000.00 | IN | PASS | HIGH (amount 17,391× avg) | — blocked | — blocked | **REJECTED** |
| RAUD | $10.00 | IN | PASS | LOW | NONE | PASS | **SETTLED** |

**Note:** All four fraud engine calls used the `fallback` engine (rule-based), not the Claude model, due to `NotFoundError` — indicating the OpenRouter model endpoint was unreachable during testing. Fallback rules produced correct risk classifications in all observed cases.

**Compliance logic defect:** `pipeline.py:32` — `sanc_ok = not sanc["hit"] or sanc["strength"] == "PARTIAL"` — PARTIAL watchlist matches pass silently. Confirmed untriggered in current logs but represents a live compliance gap.

---

## Model Routing Audit

`audit_log/model_routing.jsonl` — 13 records:

| Task class | Model | Requests | Rationale logged |
|---|---|---|---|
| `reason` (Sonnet) | `anthropic/claude-sonnet-4.5` | R1001, R1002, REDGE, RAUD (×2 each) | Yes — all records |
| `decide` (Opus) | `anthropic/claude-opus-4.1` | R1001, RAUD (approved only) | Yes — all records |
| `classify` (Haiku) | `anthropic/claude-3.5-haiku` | B1, B2, B3, B4 | Yes — all records |

Routing behaviour is correct: Opus is invoked only for APPROVE decisions (money movement), not for blocked transactions. Haiku handles all batch scoring. All 13 records contain `rationale` field.

**Gap:** No timestamps in routing records — cannot establish ordering relative to transaction records.

---

## MCP Tool Call Audit

`audit_log/mcp_tool_calls.jsonl` — 50 records across multiple test runs:

| Server | Tool | Call count | All audited | Any failures |
|---|---|---|---|---|
| customer-info | `get_customer` | 10 | Yes | No |
| customer-info | `check_customer_history` | 14 | Yes | No |
| customer-info | `verify_ip_location` | 14 | Yes | No |
| banking-api | `get_balance` | 4 | Yes | No |
| banking-api | `transfer_funds` | 8 | Yes | No |

All 50 records carry `seq`, `server`, `tool`, `args`, and `ok` fields. High-risk country codes observed in args: `RU` (seq 7, 11), `NG` (seq 15) — no escalation flag present in audit record.

**Gap:** The `github` MCP server (configured in `.claude/mcp.json`) has no calls in the audit log — coverage cannot be confirmed.

---

## Security Scan Summary

From `audit_log/bandit_scan.txt` (run 2026-10-06):

```
Total lines of code: 387
Total lines skipped (#nosec): 0

Total issues (by severity):
    Low: 0  |  Medium: 0  |  High: 0
Total issues (by confidence):
    Low: 0  |  Medium: 0  |  High: 0
Files skipped: 0
```

**Result: CLEAN.** No suppressed findings (`#nosec` count = 0).

---

## Batch Fraud Pipeline

`audit_log/batch_results.json` — 4 synthetic transactions scored at Haiku tier:

| ID | Customer | Amount | Country | Risk | Engine |
|---|---|---|---|---|---|
| B1 | C5001 | $300 | IN | LOW | fallback |
| B2 | C5001 | $9,000 | RU | HIGH | fallback |
| B3 | C5002 | $15,000 | SG | LOW | fallback |
| B4 | C5002 | $60,000 | NG | HIGH | fallback |

All four routed to Haiku via model_routing.jsonl. Fallback engine active for all (same `NotFoundError` as transaction pipeline).

---

## Open Findings

| # | Finding | Severity | Location |
|---|---|---|---|
| F-1 | No per-session rate limiting on money-movement | **High** | `bfsi/transactions.py`, `bfsi/chatbot.py` |
| F-2 | Sanctions PARTIAL match silently passes compliance | **High** | `bfsi/compliance/pipeline.py:32` |
| F-3 | No timestamps in any audit log record | **Medium** | All three `.jsonl` logs |
| F-4 | REJECTED transaction records omit `amount`, `from_account`, `to_account` | **Medium** | `bfsi/transactions.py:23-24` |
| F-5 | `_mask()` only covers keys containing "card"; `pan`/`pan_number` unprotected | **Medium** | `bfsi/mcp/registry.py:29` |
| F-6 | Fraud engine always falls back to rule-based scoring (model unreachable) | **Medium** | `bfsi/fraud_engine.py` — OpenRouter endpoint |
| F-7 | `github` MCP server audit coverage unconfirmed | **Low** | `audit_log/mcp_tool_calls.jsonl` |
| F-8 | Sanctions watchlist is a demo fixture (3 names) | **Low** | `bfsi/compliance/sanctions_check.py` |

---

## Recommended Remediations

| Priority | Action |
|---|---|
| **High** | Add token-bucket rate limiter in `transactions.process()` — max 5 transfers per customer per 60s |
| **High** | Fix `pipeline.py:32`: `sanc_ok = not sanc["hit"]` — route PARTIAL hits to Opus for human-in-the-loop review |
| **Medium** | Add `"timestamp": datetime.utcnow().isoformat() + "Z"` to every audit record at write time |
| **Medium** | Include full transaction fields (`customer_id`, `amount`, `from_account`, `to_account`, `ip_country`) in REJECTED records |
| **Medium** | Extend `_mask()` to match `pan` in key names and scan string values for 13–19 digit PAN patterns |
| **Medium** | Investigate OpenRouter connectivity for Claude model endpoint — fallback-only fraud scoring reduces detection quality |
| **Low** | Add `github` MCP test coverage; replace demo sanctions watchlist with a production-grade source |
