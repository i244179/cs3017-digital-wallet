# CS3017 Digital Wallet — Wallet-to-Bank Transfer via a Simulated Raast Rail

**CS3017 Enterprise Systems and Applications — Fall 2026 semester project**
NayaPay-context reference architecture (proposed, not NayaPay's actual systems). **Synthetic data only**: no real
CNICs, IBANs, cards, credentials or network calls to any payment system.

> **Status: v1.0.0, all seven phases delivered.** Start with **`docs/HANDOVER.md`** for setup, start, migrate, seed
> and test commands, URLs, demo accounts and known limitations. The whole project (every phase report, architecture,
> ADRs, evidence, benchmark, requirements and traceability) is in **`docs/COMPLETE_PROJECT_REPORT.md`** (and `.html` / `.pdf`
> editions with diagrams and screenshots). Final verification: `python evidence/scripts/verify_phase7.py`.

## Problem

A customer sends money from a digital wallet to a beneficiary bank account through an instant-payment rail.
The platform must never double-debit on retries, never lose money when the rail times out, keep a balanced
ledger, hold risky transfers for a human, and reconcile its records against the rail's settlement file.

## Architecture

One deployable FastAPI service plus three worker processes (rail dispatcher, outbox relay, RabbitMQ consumers)
over one PostgreSQL database, split into bounded-context
packages under `src/wallet/` — each owns its tables:

| Context | Owns (system of record) |
|---|---|
| `transactions` | transaction status, immutable reference, status history |
| `ledger` | balances, postings, append-only entries |
| `idempotency` | idempotency outcome (key + request hash + response) |
| `compliance` | versioned rule decisions, human-review cases |
| `rail` | adapter message log; simulated switch state and settlement files |
| `events` | transactional outbox, consumer de-duplication, notifications (derived) |
| `reconciliation` | runs, item classification, break ownership/resolution |
| `ai_triage` | advisory suggestions log (no write access elsewhere) |
| `identity` / `audit` | logins + RBAC roles; hash-chained audit log |

Key controls already enforced **inside PostgreSQL** (migration `0002_integrity_controls`):
append-only ledger/audit tables; balances changed only by ledger entries; every posting balanced and
shape-checked (P1/P2/P3) at commit; legal state transitions only, each with a history row; no ledger
posting while `ON_HOLD`; one reversal per transaction with distinct maker/checker; hash-chained audit log;
least-privilege application role with no UPDATE/DELETE on ledger or audit tables.

## Prerequisites

- **Docker Desktop** (with Compose v2), or
- **Python 3.11+** and **PostgreSQL 16** installed locally

## Setup and run

### Option A — Docker (one command after configuring)

```bash
python infra/make_env.py                 # or: cp infra/.env.example infra/.env and edit every change-me value
docker compose -f infra/docker-compose.yml --env-file infra/.env up --build
```

This starts PostgreSQL and RabbitMQ, runs `alembic upgrade head`, loads the synthetic seed data, then starts the
API (`http://localhost:8000/docs`), the rail dispatcher, the outbox relay and the event consumers.

### Option B — Local Python

```bash
python -m venv .venv && source .venv/bin/activate      # Windows: .venv\Scripts\activate
pip install -r requirements-dev.txt                    # runtime: requirements.txt; + test tools
python infra/make_env.py                               # writes infra/.env with random local secrets
# one-time, as a PostgreSQL superuser: create the owner login + database, then the app login:
#   CREATE ROLE wallet_owner LOGIN SUPERUSER PASSWORD '...';  CREATE DATABASE wallet OWNER wallet_owner;
#   then run infra/postgres/init/01-create-app-login.sh (or its SQL) to create wallet_app
alembic upgrade head
python data/seed.py
# without RabbitMQ: set BROKER=memory in infra/.env (events are consumed via POST /v1/operations/workers/run)
uvicorn wallet.api.app:app --app-dir src --reload          # API on http://localhost:8000/docs
python -m wallet.workers dispatcher                        # (separate terminals, PYTHONPATH=src)
python -m wallet.workers relay
python -m wallet.workers consumers                         # requires RabbitMQ
```

### Frontend (React + TypeScript + Vite)

```bash
cd frontend
npm ci
npm run dev          # http://localhost:5173 — proxies /v1 to the API on http://localhost:8000
# or: npm run build && npm run preview   (http://localhost:4173)
```

Sign in with a seeded username (listed under "Demo accounts" on the login page) and `SEED_DEMO_PASSWORD`.
With Docker, the web app is served at http://localhost:8080.

### Demo flow in one command

```bash
python evidence/scripts/generate_api_examples.py   # resets the local DB, starts uvicorn, records 42 real calls
```

## Test

```bash
pytest -q                                   # 304 tests on throw-away PostgreSQL databases (never the demo DB)
pytest -m rubric -q                         # the 10 mandatory rubric scenarios, one fresh database each
pytest -m "unit or concurrency or security or recovery" -q
python evidence/scripts/verify_phase4.py    # full suite + per-category JUnit counts + 5x concurrency + frontend tests
python evidence/scripts/generate_api_docs.py # regenerate api/*.yaml|json and generated diagrams (--check for drift)
python evidence/scripts/verify_phase5.py    # spec/schema validation, contract coverage, diagram rendering, newman run
python evidence/scripts/verify_phase7.py    # final verification: static checks, clean venvs, frontend, live stack, traceability
python evidence/generate_evidence.py        # full evidence pack -> evidence/phase6/EVIDENCE_REPORT.md (resets the local DB)
python evidence/scripts/verify_phase2.py    # backend acceptance run; writes evidence/phase2/*
python evidence/scripts/verify_phase3.py    # frontend: typecheck, unit, build, live client tests, Chrome E2E
cd frontend && npm test                     # frontend unit tests only
```

The verify scripts and `generate_api_examples.py` **drop and recreate all tables** in the database named by `MIGRATION_DATABASE_URL` —
use a local development database only.

## API and architecture documentation

- `api/openapi.yaml`: OpenAPI 3.1 for all 46 operations, identical to the live `GET /openapi.json` (Swagger UI at `/docs`).
- `api/event_schema.json`: JSON Schema 2020-12 for the 12 event types on the outbox and RabbitMQ.
- `api/postman_collection.json`: import into Postman, then set `baseUrl`, `demoPassword` (SEED_DEMO_PASSWORD) and `railApiKey`.
  Run it in order on a freshly seeded database, or with `newman run api/postman_collection.json --env-var …`.
- `docs/architecture/`: C4, sequences, BPMN-style process, state machine, ERD, threat model and ADR-001…007 (index in its README).

## Synthetic seed data

| Login | Role | Notes |
|---|---|---|
| `ayesha.rauf` | CUSTOMER | Rs 250,000 funded; normal-path demo |
| `bilal.qureshi` | CUSTOMER | second customer for object-level access tests |
| `zara.hussain` | CUSTOMER | Rs 15,000 — concurrent double-spend tests |
| `kamran.aslam` | CUSTOMER | HIGH risk — triggers compliance HOLD |
| `noor.fatima` | CUSTOMER | KYC BLOCKED — triggers DECLINE |
| `support.agent`, `ops.reviewer`, `compliance.officer`, `finance.maker`, `finance.checker`, `finance.checker2`, `platform.admin` | one per back-office role | |

All logins share `SEED_DEMO_PASSWORD` from `infra/.env`. Eight historical transfers for 2026-11-19 and the
Raast settlement file `data/settlement_files/raast_settlement_2026-11-19.csv` contain five intentional
reconciliation breaks (duplicate, amount mismatch, missing internal, missing external, status mismatch).

## Limitations

- `docker compose up` was not executed in the build environment (no Docker Hub access). Every service's compose command
  was run as a process against real PostgreSQL 16, RabbitMQ 3.12 and nginx (`evidence/phase7/stack/`, 19/19 checks).
- Latency p95 target met at concurrency 1/4/8, missed at 16/32 (measured bottleneck; see `evidence/phase6/benchmark/`).
- The AI triage is a deterministic rule-based "mock AI decision service" (permitted by the guidelines), not a learned model.
- Full reversals only (partial refunds and post-settlement refunds are out of scope by design).
- Synthetic compliance rules are control-placement demonstrations, not legal KYC/AML screening.

## Repository layout

```
/README.md  /AI_USE_LOG.md  /alembic.ini  /pyproject.toml  /requirements.txt  /requirements-dev.txt
/src/wallet/          bounded-context packages (models, services, API, workers)
/src/migrations/      Alembic env + versions (0001 schema, 0002 integrity controls, 0003)
/tests/               pytest suites: phase1-3 (integration), rubric, unit, concurrency, security, recovery; support/ = fixtures
/data/                seed.py + synthetic settlement files
/infra/               docker-compose.yml, Dockerfile, .env.example, postgres init
/evidence/            scripts/ (generators) and phase outputs
/frontend/            React + TypeScript + Vite app (src/api, auth, components, pages, styles)
/api/                 openapi.yaml, event_schema.json, postman_collection.json
/docs/                HANDOVER, COMPLETE_PROJECT_REPORT, PHASE1-7 reports, RTM, FINAL_TRACEABILITY_REPORT, architecture/ (C4, sequences, BPMN, state machine, ERD, threat model, ADRs)
```
