# ⚡ DataVibe — AI-Assisted, Human-in-the-Loop Data Migration

DataVibe turns a plain-English request like
> *"Migrate `warehouse` db from postgres to `orders.csv`"*

into a **safe, reviewable data migration**: it parses the intent with an LLM, figures out what it knows and what it still needs, **asks you** when something's ambiguous, cleans the data, and writes to the destination **only after you approve** — then verifies the result.

Built as a **stateful LangGraph pipeline** with multiple human-in-the-loop gates, a dynamic connector system, and production-grade observability.

---

## The goal

Real data migrations break in unglamorous ways: a tool guesses the wrong table, connects with half-configured credentials and dies mid-run, or silently loads dirty data. I wanted to build the opposite — a system that is **communicative and safe by default**:

- understands the request,
- tells you what it understood vs. what it still needs,
- **asks instead of failing**,
- never silently guesses or drops data,
- and is fully **auditable**.

---

## The flow

```
                       ┌──────────── human gates (interrupt + durable resume) ───────────┐
START → parse(LLM) → 📥 setup sources → connect → 🗂️ select table → discover → validate
        → 🧹 detect cleaning → preview transform → 🔎 review (schema + cleaning)
        → clean → transform → 🎯 destination (where to land) → connect dest → ingest
        → reconcile ──(confidence low?)──► rollback
                     └──(pass)──────────► END
```

1. **Describe** the migration in plain English.
2. **Provide sources** — upload a CSV, or enter DB connection details (the form is *generated from the connector's spec*).
3. **Pick a table** — if a DB source named a database but not a table, it lists the *real* tables and asks.
4. **Review** — see the discovered schema and exactly what cleaning will be applied; choose boolean format / null handling.
5. **Choose destination** — file path, or DB connection (a missing database is auto-created).
6. **Ingest & reconcile** — data is written, then verified; low confidence triggers rollback.

Each gate is a LangGraph `interrupt()` — the graph pauses, **checkpoints its state**, and resumes exactly where it left off (even across restarts).

---

## 🤝 Multi-agent architecture (10 agents, orchestrated by LangGraph)

DataVibe is built as **10 specialized, single-responsibility agents**.
 — **the LangGraph graph *is* the orchestrator**: each agent is a node, edges define the flow, `interrupt()` nodes are the human gates, and a checkpointer holds durable state.

| # | Agent | Responsibility |
|---|-------|----------------|
| 1 | **Requirement** | Parse the plain-English request into a structured plan (LLM · few-shot · JSON mode) |
| 2 | **Connector** | Spawn & test source/destination connectors; auto-create a missing destination DB |
| 3 | **Schema** | Profile each source — types, null rates, sample values |
| 4 | **Dashboard** | Build the human-readable discovery report for review |
| 5 | **Validation** | Run auto-generated + custom data-quality rules |
| 6 | **Cleaning** | Row-preserving fixes — booleans, mixed-format dates, nulls |
| 7 | **Transformation** | Map source fields to the destination schema |
| 8 | **Ingestion** | Batched, retried loading |
| 9 | **Reconciliation** | Verify the load (row counts, checksums, referential integrity) → confidence score |
| 10 | **Rollback** | Revert the destination on failure or low confidence |

> **Why a deterministic graph instead of autonomous agent-to-agent chatter?** Data migration needs auditability, repeatability, and safety. So the agents are coordinated by an explicit state machine with human gates — the LLM is used *within* agents where language understanding helps (parsing, mapping), not to improvise the workflow.

---

## 🛡️ Engineering measures (how it avoids hallucination & silent failures)

This is the heart of the project — the deliberate safeguards:

| Measure | What it prevents | How |
|---|---|---|
| **Few-shot prompting + JSON mode** | LLM hallucinating structure or inventing data | The requirement parser is given worked examples and uses the model's native JSON mode; it's explicitly told to set `table: null` when no table is named — *never guess one* |
| **Direction-aware parsing** | Confusing source vs. destination ("X → Y") | The prompt + parser separate the source side from the destination side |
| **"Understood vs. needed" check** | Connecting with missing config and erroring | Each connector declares required fields (a spec); after parsing, the system computes what's still missing and **asks at a gate** |
| **Pick from reality, not guesses** | Reading the wrong/again-nonexistent table | If a DB source has no table, it connects, lists the **actual** tables, and asks you to choose — never grabs the first one silently |
| **Graceful LLM degradation** | Whole pipeline failing when the LLM is down | Rule-based fallback parser; retries with backoff on transient errors; **fail-fast with a clear message** on auth errors |
| **Safe, row-preserving cleaning** | Corrupting or losing data | Trims, normalizes booleans (`true/1/yes → true`), standardizes **mixed-format dates**, handles nulls — **never drops a row**, so counts still reconcile |
| **Auto-create destination DB** | "database does not exist" failures | Each connector can `ensure_database()`; a missing Postgres db is created instead of failing |
| **Reconciliation + auto-rollback** | Declaring success on a bad load | Row counts, checksums, referential integrity → confidence score; low confidence rolls back |
| **Full traceability** | "Why did it do that?" | Every LLM call logs its **prompt, token usage, and JSON output**; every agent logs start/finish + key results (readable console + JSONL audit) |

---

## 🔌 Dynamic, connector-agnostic design

Connectors **declare what they need** in a spec registry (`kind`, required fields, `needs_table`, `creatable`). The UI forms, the "what's missing?" checks, table selection, and DB auto-create are all driven by that spec — **no per-type `if`-branches** in the UI or pipeline.

> Adding a new backend (Neo4j, Snowflake, …) = a connector class + one spec entry. The forms, gates, and checks light up automatically.

Supported today: **CSV/Excel, SQLite, PostgreSQL, MongoDB, REST**.

---


## 🧱 Architecture notes

- **State** — a typed, serializable dict checkpointed after every node (durable resume).
- **RunContext** — live objects (DB connections, DataFrames) kept *out* of the checkpoint.
- **Deterministic core, LLM at the edges** — orchestration is a deterministic graph (auditable, repeatable); the LLM is used only where natural-language understanding genuinely helps (parsing, mapping).

---

## Tech stack

**Python** · **LangGraph** (stateful agent graph + durable checkpointing) · **OpenAI / OpenAI-compatible LLMs** · **pandas** · **SQLAlchemy / psycopg2** · **pymongo** · **Pydantic** · **Streamlit** · structured logging.

---

## Project structure

```
datavibe/
├── graph/          # LangGraph: state, nodes (skills), gates, pipeline wiring
├── agents/         # the skills (parse, connect, schema, validate, clean, transform, ingest, reconcile, rollback)
├── connectors/     # CSV/Excel, SQLite, Postgres, Mongo, REST + specs.py (the spec registry)
├── tools/          # llm.py (provider-agnostic + logging), cleaning, validation, transform, audit
├── models/         # Pydantic state & schema models
├── ui/             # Streamlit dashboard (drives the graph)
├── config/         # settings (env-driven)
├── sample_data/    # demo CSVs
└── tests/          # connector / agent / pipeline tests
```

---
