# SchemaSense — AI-Powered Database Knowledge & Governance Layer

## 1. Problem Statement

Organizations running production relational databases accumulate massive amounts of undocumented schema knowledge over time. Engineers who understand *why* a table exists, what a cryptically-named column actually means, or which query patterns are safe leave the company — taking that knowledge with them. This causes:

- New engineers spending weeks reverse-engineering schemas before becoming productive
- Analysts repeatedly interrupting engineering teams for basic data pulls
- Silent breakage of downstream reports/dashboards when schema migrations happen without impact analysis
- Security/compliance risk from ungoverned, unaudited ad-hoc querying

**SchemaSense** is a self-hosted layer that sits on top of any relational database and provides automatic documentation, safe natural-language querying, schema drift detection, and query auditing — combining classical DBMS engineering with a constrained, schema-grounded LLM component.

This is not an attempt to build a general-purpose LLM. It is a narrow, grounded AI system — the same category of tool used in production by companies like Snowflake (Cortex Analyst) and Databricks (Genie) — built from scratch to demonstrate applied DBMS + AI engineering.

## 2. Objectives

1. Auto-generate human-readable documentation for any connected schema
2. Allow non-technical users to query data in plain English, safely
3. Detect and alert on schema drift that could break existing queries/reports
4. Maintain a full audit trail of who queried what, with anomaly flags
5. Demonstrate depth across core DBMS concepts, not just an API wrapper

## 3. System Architecture

```
                     ┌─────────────────────────┐
                     │      Frontend (UI)      │
                     │  React / Streamlit      │
                     │  - NL query box         │
                     │  - Schema docs browser  │
                     │  - Audit dashboard      │
                     └───────────┬─────────────┘
                                 │
                     ┌───────────▼───────────────┐
                     │   Application Backend     │
                     │   (Python / FastAPI)      │
                     │  - Auth & RBAC            │
                     │  - Query orchestration    │
                     │  - Validation & sandboxing│
                     └───────────┬───────────────┘
                    ┌────────────┼─────────────┐
                    ▼            ▼             ▼
          ┌──────────────┐ ┌──────────┐ ┌──────────────┐
          │  LLM Layer   │ │ Metadata │ │ Target DB    │
          │ (schema-     │ │ Store    │ │ (Postgres —  │
          │ grounded     │ │ (schema  │ │ the DB being │
          │ NL→SQL)      │ │ catalog, │ │ documented/  │
          │              │ │ logs,    │ │ queried)     │
          │              │ │ users)   │ │              │
          └──────────────┘ └──────────┘ └──────────────┘
```

**Key design principle:** the LLM never executes SQL directly against the target database. It proposes a query; the backend validates it against the live schema catalog, checks it against RBAC rules, runs it through `EXPLAIN` for cost/safety, and only then executes it inside a read-only, row-limited, timeout-bound transaction.

## 4. Core Modules

### Module A — Schema Ingestion & Documentation Generator
- Reads `information_schema` / `pg_catalog` for tables, columns, types, foreign keys, constraints
- Samples representative rows (with PII masking) per table
- Sends schema + samples to LLM to generate plain-English table/column descriptions
- Stores descriptions in the metadata store, versioned

### Module B — Natural Language to SQL Engine
- User asks a question in plain English
- LLM prompt is grounded with: relevant schema subset (via embedding search over table/column docs), few-shot examples of past validated queries, and RBAC-permitted tables only
- Generated SQL is parsed (not blindly executed) and checked:
  - Does it reference only tables/columns that exist?
  - Does it touch only tables the user's role can access?
  - Is it read-only (no INSERT/UPDATE/DELETE/DROP)?
- Query is shown to the user in plain English + SQL before running, with estimated cost from `EXPLAIN`

### Module C — Schema Drift & Impact Detection
- Triggers on the metadata store fire when the target schema changes (column dropped/renamed, type changed)
- Cross-references the change against stored historical queries/report definitions
- Flags which saved reports/dashboards would break, before they silently fail

### Module D — Audit & Anomaly Layer
- Every query (human or LLM-generated) is logged: user, timestamp, tables touched, rows returned, execution time
- Flags anomalies: unusual table access for a role, spikes in data volume pulled, off-hours access
- RBAC and row-level security enforced at the Postgres level, not just app level

## 5. DBMS Concepts Demonstrated (map this directly to your evaluation rubric)

| Concept | Where it appears |
|---|---|
| ER Modeling & Normalization | Metadata store schema: users, roles, tables_catalog, columns_catalog, query_log, drift_alerts |
| Transactions & ACID | Query execution wrapped in transactions; audit log write is atomic with query execution |
| Indexing | Indexes on query_log(user_id, timestamp) for fast audit lookups; discussion of B-tree vs hash index choice |
| Query Optimization | EXPLAIN/EXPLAIN ANALYZE used to cost-check LLM-generated SQL before execution |
| Views | Pre-approved "safe query templates" exposed as views for common analyst questions |
| Stored Procedures/Triggers | Trigger on schema change to populate drift_alerts; procedure for anonymized sampling |
| Concurrency Control | Multiple simultaneous NL queries handled via connection pooling + isolation levels |
| Security | RBAC, row-level security (RLS) policies, parameterized queries to prevent injection from LLM output |
| Backup/Recovery (stretch) | Versioned schema snapshots stored in metadata store for point-in-time doc comparison |

## 6. Tech Stack

- **Database:** PostgreSQL (chosen for RLS, triggers, rich `EXPLAIN`, JSON support)
- **Backend:** Python + FastAPI
- **LLM:** Any API (Claude/GPT) called with schema-grounded prompts; embeddings (pgvector) for retrieving relevant schema chunks
- **Frontend:** Streamlit (fast to build) or React (more polish for demo)
- **Auth:** JWT-based, roles mapped to Postgres roles

## 7. What Makes It Unique (for your project defense)

- Most student "NL to SQL" projects stop at "LLM writes SQL, we run it." This project's differentiator is the **validation/governance layer** — grounding, RBAC enforcement, cost-checking, and drift detection — which is the actual hard, interesting DBMS engineering problem, and mirrors how real companies deploy this safely.
- It directly answers a documented real-world pain point (tribal schema knowledge, silent report breakage) rather than being a generic chatbot-over-database demo.

## 8. Suggested Build Order (for a semester timeline)

1. Design & normalize the metadata store schema (Week 1–2)
2. Build schema ingestion + doc generation (Week 3–4)
3. Build NL→SQL with validation layer (Week 5–7)
4. Add RBAC + RLS + audit logging (Week 8–9)
5. Add drift detection with triggers (Week 10)
6. Frontend + demo polish + write-up (Week 11–12)

## 9. Evaluation Metrics to Report

- NL→SQL accuracy on a held-out set of test questions (execution accuracy, not just exact string match)
- Query validation catch rate (how many unsafe/invalid LLM outputs get correctly blocked)
- Drift detection precision/recall on simulated schema changes
- Latency of end-to-end NL query (embedding retrieval + LLM call + validation + execution)
