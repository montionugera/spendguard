---
name: spendguard
description: Estimate, cap, and govern what an AI agent spends on cloud data warehouses (BigQuery, Snowflake, Databricks) before and during query execution. Use when the agent is about to run SQL against a billable warehouse, when a user asks how much a query will cost, or when enforcing a spend budget on agent-driven analytics.
---

# spendguard — a spend governor for AI agents on data warehouses

An agent with a "run SQL" tool and no spending limit will happily `SELECT *`
a billion-row table and burn real money before you've finished your coffee.
spendguard sits between the agent and the warehouse: it previews the dollar
cost **before** every query, enforces budgets, learns how accurate its
estimates are, and suggests cheaper rewrites when a query blows the budget.

## When to use this skill

- Before running **any** SQL against BigQuery, Snowflake, or Databricks,
  estimate the cost first.
- When the user asks "how much will this query cost?" or sets a spend budget.
- When a query is refused by a budget gate — use the cheaper-rewrite
  suggestions instead of retrying blindly.

## Setup

spendguard is an MCP server, installable with zero config:

```bash
uvx spendguard-mcp
```

Add it to the agent's MCP config (Claude Code, Cursor, Copilot, Codex):

```json
{ "mcpServers": { "spendguard": { "command": "uvx", "args": ["spendguard-mcp"],
  "env": { "BIGQUERY_PROJECT": "my-project",
            "GOOGLE_APPLICATION_CREDENTIALS": "/path/to/sa.json" } } } }
```

Repo: https://github.com/tanveer-arch/spendguard · PyPI: `spendguard-mcp`

## The governed workflow

Never run warehouse SQL directly. Always follow this loop:

1. **Estimate** — `estimate_query_cost(engine, sql)`. Free, no query executed.
   Check the `accuracy_tier`: BigQuery is PRECISE (dry-run based), Snowflake
   is UPPER_BOUND, Databricks is HEURISTIC.
2. **Gate** — `run_query_bounded(engine, sql, max_estimated_cost_usd=...)`.
   This estimates, enforces the budget, executes, then reconciles the actual
   billed cost into a local ledger. If refused (`over_call_cap`), do not
   retry the same query.
3. **Rewrite** — on refusal, call `suggest_cheaper_query` with the SQL and
   apply its suggestions (LIMIT injection, partition filters, column
   pruning) before re-estimating.
4. **Report** — `spend_report()` shows reconciled spend per engine and how
   well-calibrated the estimates are.

## Example

```jsonc
// estimate_query_cost("bigquery", "SELECT * FROM proj.ds.events_2025")
{ "accuracy_tier": "PRECISE", "estimated_cost_usd": 0.0081 }

// run_query_bounded("bigquery", "SELECT * FROM ...", "max_estimated_cost_usd": 5.0)
{ "status": "refused", "reason": "over_call_cap",
  "detail": "Estimated $12.40 exceeds your per-call cap $5.00." }
```

## Rules

- Cost estimates are pre-flight only; the ledger's reconciled `actual_usd`
  is the source of truth for spend reporting.
- Respect the accuracy tiers — never present a HEURISTIC estimate as exact.
- Budgets are enforced by the tool, not by the agent's judgment. Do not
  bypass a refusal by rewording the query; rewrite it to be cheaper.
