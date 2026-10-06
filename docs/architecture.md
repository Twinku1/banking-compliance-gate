# Heritage National Bank — AI Banking Platform: Architecture

## 1. System Overview

Heritage National Bank's AI Banking Platform is a unified, production-ready system that integrates large-language-model reasoning, MCP-based microservices, compliance automation, fraud detection, and customer-service capabilities into a single coherent workflow. The platform accepts inbound transaction and customer-service requests, routes each request to the appropriate model tier (Haiku, Sonnet, or Opus) based on cost and risk, enforces a sequential KYC → AML → Sanctions → Approval compliance pipeline before any money moves, and emits a full, tamper-evident audit trail covering every MCP tool hop, model selection decision, and transaction outcome. A batch fraud pipeline processes synthetic transaction sets at the cheapest model tier, while a streaming chatbot provides guardrailed customer support backed by the same MCP tools and skill library used by the compliance engine.

---

## 2. Layer Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                        Entry Point                          │
│                      bfsi/main.py                           │
└──────────────────────────┬──────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          │                                 │
          ▼                                 ▼
┌──────────────────┐              ┌──────────────────────┐
│  Transaction     │              │  Batch Fraud         │
│  Processing      │              │  Pipeline            │
│  transactions.py │              │  batch_fraud.py      │
└────────┬─────────┘              └──────────┬───────────┘
         │                                   │
         ▼                                   │
┌──────────────────┐                         │
│  Compliance      │                         │
│  Pipeline        │◄────────────────────────┤
│  pipeline.py     │                         │
└────────┬─────────┘                         │
         │                                   │
         ▼                                   ▼
┌─────────────────────────────────────────────────────────────┐
│                      MCP Registry                           │
│                    mcp/registry.py                          │
│          (audit every hop → mcp_tool_calls.jsonl)           │
└──────────────────────────┬──────────────────────────────────┘
                           │
          ┌────────────────┴────────────────┐
          ▼                                 ▼
┌──────────────────┐              ┌──────────────────────┐
│  customer-info   │              │  banking-api         │
│  MCP Server      │              │  MCP Server          │
│  get_customer    │              │  get_balance         │
│  check_history   │              │  transfer_funds      │
│  verify_ip       │              │  apply_loan          │
└──────────────────┘              └──────────────────────┘

Cross-cutting concerns (used by multiple layers above):
┌────────────────┐  ┌───────────────┐  ┌──────────────────┐  ┌──────────────────┐
│ Model Router   │  │ Fraud Engine  │  │ Chatbot          │  │ Skills Loader    │
│ model_router   │  │ fraud_engine  │  │ chatbot.py       │  │ skills_loader.py │
│ .py            │  │ .py           │  │ (streaming,      │  │ (prompt cache)   │
│ Haiku/Sonnet/  │  │ (MCP signals  │  │  sanitized)      │  │                  │
│ Opus + log     │  │  + skill AI)  │  │                  │  │                  │
└────────────────┘  └───────────────┘  └──────────────────┘  └──────────────────┘
```

---

## 3. Component Descriptions

### bfsi/main.py — Entry Point
`main.py` is the top-level orchestrator that bootstraps the application by calling `load_dotenv()` to inject secrets from the `.env` file, then drives an end-to-end demonstration workflow. It calls `process()` from `transactions.py` for two representative transactions (one that should settle, one that should be blocked by sanctions), then invokes `run_batch()` from `batch_fraud.py` to score a synthetic set of transactions. It is the single executable surface of the platform and the target of the `/security-review` slash command.

### bfsi/transactions.py — Transaction Processing
`transactions.py` is the production money-movement layer. For each request it selects a model tier via `model_router.route()`, runs the full compliance pipeline, and — only on an APPROVE decision — executes the `banking-api` `transfer_funds` MCP tool. Every outcome (SETTLED, FAILED, or REJECTED) is appended as a JSON record to `audit_log/transactions.jsonl`, providing the authoritative transaction audit trail. Known gap: records do not include an ISO-8601 timestamp field.

### bfsi/compliance/pipeline.py — Compliance Pipeline
`pipeline.py` enforces the mandatory four-stage gate: KYC, AML, Sanctions, and Approval. Each stage is evaluated in order; a BLOCK at any stage halts the pipeline immediately and returns a structured result identifying the failing stage. KYC delegates to the `customer-info` MCP server, AML uses the fraud engine's risk score, Sanctions calls `sanctions_check()`, and Approval is implicit when all prior stages pass. Known gap: PARTIAL sanctions hits are currently passed through rather than escalated for manual review.

### bfsi/fraud_engine.py — Fraud Engine
`fraud_engine.py` combines signals from two MCP calls (`check_customer_history`, `verify_ip_location`) with a Claude Haiku inference call using the `fraud-triage` skill to produce a `{"risk", "reason", "signals"}` verdict. If the API call fails, a deterministic fallback rule fires: HIGH risk if the amount is more than 3× the customer's average or above $5,000 with a geo-mismatch, LOW otherwise. The engine is the AML signal source for the compliance pipeline and the per-item scorer for the batch fraud pipeline.

### bfsi/chatbot.py — Chatbot
`chatbot.py` delivers a compliance-guardrailed, streaming customer-service interface. It sanitizes every inbound message (stripping control characters and prompt-injection phrases such as "ignore previous") and truncates input to 2,000 characters before any model call. It fetches live fraud signals via the audited MCP registry, then calls Claude Sonnet with the `customer-support` skill as a cached system prompt. Both streaming and non-streaming modes are supported.

### bfsi/batch_fraud.py — Batch Fraud Pipeline
`batch_fraud.py` simulates the Module 5 Batch API by iterating over a set of synthetic transactions, routing each to the `classify` tier (Haiku) via `model_router.route()`, and delegating scoring to `fraud_engine.score_transaction()`. Results are collected, written to `audit_log/batch_results.json`, and returned as a list for display by `main.py`. This is the primary high-volume, cost-optimized processing path.

### bfsi/mcp/registry.py — MCP Registry
`registry.py` is the in-process MCP dispatch layer. It maps `(server, tool)` tuples to the actual Python handler functions in the two MCP server modules and provides a single `call(server, tool, **args)` API for the rest of the platform. Before recording each hop it runs `_mask()` to redact full card numbers to the last four digits (PCI DSS compliance). Every call is appended to `audit_log/mcp_tool_calls.jsonl` with a monotonically increasing sequence number and a pass/fail indicator. The `github` MCP server is declared in `.claude/mcp.json` for Claude Code tooling use but is not wired into the in-process registry.

### bfsi/model_router.py — Model Router
`model_router.py` implements cost-aware model selection across three named tiers: `classify`, `reason`, and `decide`. For each routing decision it resolves the model slug (preferring environment-variable overrides), appends a JSON record to `audit_log/model_routing.jsonl` with the request ID, task class, selected model, and rationale string, and returns the model slug to the caller. This ensures every LLM call is associated with an explicit, auditable routing decision.

### bfsi/skills_loader.py — Skills Loader
`skills_loader.py` reads every `skills/*/SKILL.md` file, parses the YAML front matter, and assembles the skill bodies into a single system-prompt block marked with `cache_control: ephemeral` for Anthropic prompt caching. Callers pass an optional `active` list to select only the skills relevant to a given context (e.g., `["fraud-triage"]` for the fraud engine). This ensures the large, stable skill library is billed once per cache warm-up rather than on every API call.

---

## 4. Model Routing Strategy

| Task Class | Tier    | Default Model                  | Rationale                                                    |
|------------|---------|--------------------------------|--------------------------------------------------------------|
| `classify` | Haiku   | `anthropic/claude-3.5-haiku`   | High-volume, low-stakes classification; batch fraud scoring, intent tagging. Minimises per-request cost. |
| `reason`   | Sonnet  | `anthropic/claude-sonnet-4.5`  | Default reasoning tier for chatbot responses, KYC extraction, compliance narrative generation. |
| `decide`   | Opus    | `anthropic/claude-opus-4.1`    | Highest-stakes decisions: final transaction approval, Claude-as-Judge evaluation, ambiguous fraud adjudication. |

The active model for each tier can be overridden at runtime via the `MODEL_HAIKU`, `MODEL_SONNET`, and `MODEL_OPUS` environment variables. Every routing decision is logged to `audit_log/model_routing.jsonl`.

---

## 5. MCP Server Catalogue

| Server Name     | Transport          | Tools Available                                                     | Purpose                                                                 |
|-----------------|--------------------|---------------------------------------------------------------------|-------------------------------------------------------------------------|
| `customer-info` | In-process / stdio | `get_customer`, `check_customer_history`, `verify_ip_location`      | Customer master data, transaction history baselining, geo-IP mismatch detection for fraud and KYC. |
| `banking-api`   | In-process / stdio | `get_balance`, `transfer_funds`, `apply_loan`                       | Account balances, money movement between accounts (validated), indicative loan underwriting. |
| `github`        | npx / stdio        | `create_branch`, `create_commit`, `create_pull_request`             | Source control operations for CI/CD automation; used by Claude Code tooling, not the in-process registry. |

---

## 6. Compliance Pipeline

The pipeline in `bfsi/compliance/pipeline.py` is strictly sequential. A transaction must pass each stage before the next is evaluated. Any BLOCK terminates the pipeline immediately; the result includes the name of the failing stage for audit.

1. **KYC** — Calls `customer-info` → `get_customer`. Passes if `kyc_verified == True`. Blocks if the customer record is absent or the KYC flag is not set.
2. **AML** — Calls `fraud_engine.score_transaction()` (which uses `check_customer_history` and `verify_ip_location` MCP tools plus a Haiku LLM inference). Passes if `risk != "HIGH"`. Blocks on HIGH risk.
3. **Sanctions** — Calls `sanctions_check(name)` against an internal watchlist (production equivalent: OFAC/UN/EU lists). Passes if there is no hit, or if the hit strength is `PARTIAL`. Blocks on `STRONG` hits. _Known gap: PARTIAL hits are silently passed rather than escalated for manual review._
4. **Approval** — Reached only if KYC, AML, and Sanctions all pass. Sets `decision = "APPROVE"` and returns the full stage trace including fraud signals and sanctions result.

---

## 7. Audit Trail

| Log File                            | Written By                                                      | Content                                                                                   |
|-------------------------------------|-----------------------------------------------------------------|-------------------------------------------------------------------------------------------|
| `audit_log/mcp_tool_calls.jsonl`    | `mcp/registry.py` + `PreToolUse` hook in `settings.json`       | One JSON record per MCP tool invocation: sequence number, server name, tool name, masked arguments, pass/fail indicator. |
| `audit_log/model_routing.jsonl`     | `model_router.route()`                                          | One JSON record per model selection: request ID, task class, resolved model slug, rationale string. |
| `audit_log/transactions.jsonl`      | `transactions.process()`                                        | One JSON record per transaction: request ID, settlement status, compliance stage trace, transfer result. _Known gap: no timestamp field._ |
| `audit_log/bandit_scan.txt`         | `PostToolUse` hook in `settings.json`                           | Output of `bandit -r bfsi` run automatically after every file Edit or Write operation.    |
| `audit_log/batch_results.json`      | `batch_fraud.run_batch()`                                       | Full risk/engine verdict for each synthetic batch transaction.                            |

---

## 8. Security Controls

### PAN Masking
`mcp/registry._mask()` inspects all argument dictionaries passed to MCP tools. Any key whose name contains the substring `card` with a string value longer than 4 characters is replaced with `"****" + last_4_digits` before the record is written to the audit log. This prevents full Primary Account Numbers from appearing in log files.

### Input Sanitization
`chatbot.sanitize()` strips all non-printable characters from user input, replaces known prompt-injection phrases (`ignore previous`, `system prompt`, `full card number`) with `[removed]`, and truncates the result to 2,000 characters. The sanitized string is what reaches the model and the MCP tools.

### Secrets via Environment Variables
API keys and model slug overrides are consumed exclusively from environment variables (`OPENROUTER_API_KEY`, `ANTHROPIC_BASE_URL`, `MODEL_HAIKU`, `MODEL_SONNET`, `MODEL_OPUS`). No credentials are hard-coded in any source file. The `.env` file is loaded at startup by `python-dotenv`.

### Automated Security Scanning
A `PostToolUse` hook in `.claude/settings.json` runs `bandit -q -r bfsi` after every file edit and writes the report to `audit_log/bandit_scan.txt`. The Definition of Done requires zero medium-or-higher findings.

### MCP Audit Hook
A `PreToolUse` hook in `.claude/settings.json` matches all `mcp__.*` tool calls and appends a timestamped record to `audit_log/mcp_tool_calls.jsonl`, providing a second, independent audit path alongside the in-process registry logging.

---

## 9. Known Gaps (Security Review Findings)

| Gap | Affected Component | Risk |
|-----|--------------------|------|
| **No per-session rate limiting** on money-movement operations | `transactions.py`, `mcp/registry.py` | A compromised session or runaway client can submit unlimited `transfer_funds` calls without throttling, enabling large-scale fund exfiltration. |
| **Missing timestamps in transaction log** | `transactions.py` | `audit_log/transactions.jsonl` records lack an ISO-8601 `ts` field, making forensic sequencing and regulatory reporting imprecise. |
| **Sanctions PARTIAL-hit bypass** | `compliance/pipeline.py` | Transactions where `sanctions_check()` returns `strength == "PARTIAL"` are passed to Approval without human review or escalation, potentially allowing partially-matched watchlist entities through. |
