# Heritage National Bank — AI Banking Platform: Deployment Guide

## 1. Prerequisites

### Python Version
Python 3.10 or later is required. The codebase uses `list[dict]` and `X | Y` union type hints that require Python 3.10+.

### Required Packages
Install all dependencies with pip:

```bash
pip install anthropic python-dotenv "mcp[cli]" bandit pytest
```

| Package        | Used By                                              |
|----------------|------------------------------------------------------|
| `anthropic`    | `fraud_engine.py`, `chatbot.py`, `kyc_vision.py` — Anthropic SDK for LLM calls |
| `python-dotenv`| `main.py` — loads `.env` secrets at startup          |
| `mcp[cli]`     | `banking_api_server.py`, `customer_info_server.py` — FastMCP server framework |
| `bandit`       | Security scanning (PostToolUse hook and manual scan) |
| `pytest`       | Test runner for `tests/`                             |

### Required Environment Variables

| Variable             | Required | Default                           | Description                              |
|----------------------|----------|-----------------------------------|------------------------------------------|
| `OPENROUTER_API_KEY` | Yes      | —                                 | API key for OpenRouter (used as the Anthropic SDK `auth_token`) |
| `ANTHROPIC_BASE_URL` | No       | `https://openrouter.ai/api`       | Override the API base URL (e.g., to use Anthropic directly) |
| `MODEL_HAIKU`        | No       | `anthropic/claude-3.5-haiku`      | Model slug for the `classify` tier       |
| `MODEL_SONNET`       | No       | `anthropic/claude-sonnet-4.5`     | Model slug for the `reason` tier         |
| `MODEL_OPUS`         | No       | `anthropic/claude-opus-4.1`       | Model slug for the `decide` tier         |
| `GITHUB_TOKEN`       | No       | —                                 | Personal access token for the `github` MCP server (Claude Code tooling only) |

---

## 2. Environment Setup

Create a `.env` file in the project root (`/home/labuser/claude-bfsi/m4-capstone/`) with the required variables:

```bash
# .env — never commit this file
OPENROUTER_API_KEY=sk-or-your-key-here

# Optional overrides
# ANTHROPIC_BASE_URL=https://api.anthropic.com
# MODEL_HAIKU=anthropic/claude-3.5-haiku
# MODEL_SONNET=anthropic/claude-sonnet-4.5
# MODEL_OPUS=anthropic/claude-opus-4.1
# GITHUB_TOKEN=ghp_your_token_here
```

`main.py` calls `load_dotenv()` at startup, which reads the `.env` file and injects the variables into `os.environ` before any module imports them. If the `.env` file is absent, variables must be exported in the shell environment before running.

---

## 3. Running the Platform

Execute the end-to-end workflow from the project root:

```bash
python -m bfsi.main
```

Or equivalently:

```bash
python bfsi/main.py
```

This runs two representative transactions through the compliance pipeline (one SETTLED, one REJECTED via sanctions) and then executes the batch fraud pipeline. Output is printed to stdout; audit records are written to `audit_log/`.

---

## 4. Running Batch Fraud

The batch fraud pipeline can be invoked independently:

```bash
python -c "from bfsi.batch_fraud import run_batch; import json; print(json.dumps(run_batch(), indent=2))"
```

This scores the four synthetic transactions in `batch_fraud._SYNTHETIC` at the Haiku tier, logs routing decisions to `audit_log/model_routing.jsonl`, and writes the full results to `audit_log/batch_results.json`.

---

## 5. Audit Log Locations

All audit files are written to the `audit_log/` directory, which is created automatically on first run.

| File                              | Created By                          | Contents                                                                               |
|-----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------|
| `audit_log/mcp_tool_calls.jsonl`  | `mcp/registry.py` + PreToolUse hook | One JSON line per MCP tool call: sequence number, server, tool, masked args, ok flag. |
| `audit_log/model_routing.jsonl`   | `bfsi/model_router.py`              | One JSON line per model selection: request_id, task class, model slug, rationale.     |
| `audit_log/transactions.jsonl`    | `bfsi/transactions.py`              | One JSON line per transaction: request_id, status, compliance stage trace, transfer.  |
| `audit_log/bandit_scan.txt`       | PostToolUse hook / manual scan      | Plain-text bandit report. Re-generated after every file edit in Claude Code.           |
| `audit_log/batch_results.json`    | `bfsi/batch_fraud.py`               | JSON array of batch risk verdicts: id, risk level, engine (claude or fallback).       |

---

## 6. Running Tests

From the project root:

```bash
pytest
```

To run with verbose output and stop on first failure:

```bash
pytest -v -x
```

The Definition of Done requires `pytest` to pass with zero failures.

---

## 7. Running the Security Scan

Run bandit against the `bfsi/` package and write the report to the standard audit location:

```bash
bandit -r bfsi -f txt -o audit_log/bandit_scan.txt
```

View the report:

```bash
cat audit_log/bandit_scan.txt
```

The Definition of Done requires no medium-or-higher severity findings. The same scan runs automatically via the `PostToolUse` hook in `.claude/settings.json` after every file edit made through Claude Code.

---

## 8. Directory Structure

```
m4-capstone/
├── .claude/
│   ├── mcp.json              # MCP server registry (customer-info, banking-api, github)
│   └── settings.json         # Hooks: PreToolUse audit, PostToolUse bandit scan
├── .env                      # Local secrets (not committed)
├── CLAUDE.md                 # Authoritative development context
│
├── bfsi/                     # Core application package
│   ├── __init__.py
│   ├── main.py               # Entry point — end-to-end workflow
│   ├── transactions.py       # Production transaction processing + audit trail
│   ├── batch_fraud.py        # High-volume batch fraud scoring (Haiku tier)
│   ├── chatbot.py            # Guardrailed streaming customer-service chatbot
│   ├── fraud_engine.py       # Fraud detection: MCP signals + LLM skill
│   ├── kyc_vision.py         # Vision-based KYC document extraction
│   ├── model_router.py       # Cost-aware model selection + routing log
│   ├── skills_loader.py      # Cached skill library loader
│   ├── compliance/
│   │   ├── __init__.py
│   │   ├── pipeline.py       # KYC -> AML -> Sanctions -> Approval pipeline
│   │   └── sanctions_check.py# Watchlist screening
│   └── mcp/
│       ├── __init__.py
│       ├── registry.py       # In-process MCP dispatch + PAN masking + audit
│       ├── customer_info_server.py  # customer-info MCP server
│       └── banking_api_server.py   # banking-api MCP server
│
├── skills/                   # Skill library (one subdirectory per skill)
│   ├── compliance-narrative/
│   │   └── SKILL.md
│   ├── customer-support/
│   │   └── SKILL.md
│   ├── fraud-triage/
│   │   └── SKILL.md
│   ├── kyc-extraction/
│   │   └── SKILL.md
│   ├── risk-scoring/
│   │   └── SKILL.md
│   └── sanctions-summary/
│       └── SKILL.md
│
├── audit_log/                # Runtime audit output (git-ignored)
│   ├── mcp_tool_calls.jsonl
│   ├── model_routing.jsonl
│   ├── transactions.jsonl
│   ├── batch_results.json
│   └── bandit_scan.txt
│
├── reports/                  # Compliance and judge evaluation reports
│
├── docs/                     # Project documentation
│   ├── architecture.md       # This system's architecture reference
│   └── deployment.md         # This deployment guide
│
└── tests/                    # pytest test suite
```
