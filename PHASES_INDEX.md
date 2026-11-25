# Where everything is, phase by phase

Start here: **`docs/HANDOVER.md`** (how to run it), then **`docs/COMPLETE_PROJECT_REPORT.pdf`** (everything in one
document, with diagrams and screenshots). The `.md` and `.html` editions have the same content.

| Phase | What was built | Report | Main code / artefacts | Evidence folder | Re-run it |
|---|---|---|---|---|---|
| 1 | Repository structure and database architecture (23 tables, integrity triggers, least privilege, seed) | `docs/PHASE1_REPORT.md` | `src/wallet/*/models.py`, `src/migrations/`, `data/seed.py`, `infra/` | `evidence/phase1/` | `python evidence/scripts/verify_phase1.py` |
| 2 | FastAPI backend: JWT/RBAC, state machine, idempotency, ledger, compliance, simulated Raast, outbox, RabbitMQ, reversals, reconciliation, finance, AI triage, audit | `docs/PHASE2_REPORT.md`, `docs/API_EXAMPLES.md` | `src/wallet/` | `evidence/phase2/` | `python evidence/scripts/verify_phase2.py` |
| 3 | React + TypeScript frontend for every role | `docs/PHASE3_REPORT.md` | `frontend/` | `evidence/phase3/` (incl. 26 screenshots) | `python evidence/scripts/verify_phase3.py` |
| 4 | Automated test suite (rubric, unit, integration, concurrency, security, recovery) | `docs/PHASE4_REPORT.md` | `tests/` | `evidence/phase4/` | `python evidence/scripts/verify_phase4.py` |
| 5 | OpenAPI 3.1, event schema, Postman, C4, sequences, BPMN, state machine, ERD, threat model, ADR-001..007 | `docs/PHASE5_REPORT.md` | `api/`, `docs/architecture/` | `evidence/phase5/` (rendered diagrams, newman) | `python evidence/scripts/verify_phase5.py` |
| 6 | Evidence pack: benchmark (p95), logs, traces, restart recovery, reconciliation, trial balance, AI metrics, screenshots | `docs/PHASE6_REPORT.md`, `evidence/phase6/EVIDENCE_REPORT.md`, **`evidence/phase6/benchmark/benchmark_report.md`** | `evidence/generate_evidence.py`, `evidence/scripts/` | `evidence/phase6/` | `python evidence/generate_evidence.py` |
| 7 | Final integration and handover: static checks, clean venvs, live stack (PostgreSQL + RabbitMQ + nginx + all workers), fixes, traceability | `docs/PHASE7_REPORT.md`, `docs/HANDOVER.md`, `docs/FINAL_TRACEABILITY_REPORT.md` | `evidence/scripts/verify_phase7.py`, `stack_verification.py` | `evidence/phase7/` | `python evidence/scripts/verify_phase7.py` |

Requirements: `docs/requirements_traceability_matrix.md` (52 requirements) → `docs/FINAL_TRACEABILITY_REPORT.md`
(each requirement with the tests that ran and their outcome). AI use: `AI_USE_LOG.md`.
Your own screenshots go in `evidence/manual_screenshots/` (checklist inside).

The verify and evidence scripts **reset the database named by `MIGRATION_DATABASE_URL`**. Use a development database.
