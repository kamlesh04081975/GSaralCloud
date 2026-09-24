# GSaralCloud MRA Agentic Platform

An agentic **Migration Readiness Assessment (MRA)** platform. A LangGraph coordinator dispatches seven specialist agents across two dependency-ordered waves, turning a raw on-premises workload inventory into:

- a right-sizing plan
- a 7R strategy register
- a three-year cost model
- an AWS CAF readiness score
- a gap register
- a dependency-sequenced wave plan
- a client-ready Excel workbook

One repository. Runs in VS Code, then Docker, then AWS EKS — same code, only configuration changes.

---

## Table of Contents

- [Architecture](#architecture)
- [Features](#features)
  - [Model Routing](#model-routing)
  - [Guardrails](#guardrails)
  - [Live Execution Tracking](#live-execution-tracking)
- [Output (Layer 6)](#output-layer-6)
- [Evaluation](#evaluation)
- [Getting Started](#getting-started)
- [Configuration](#configuration)
- [Testing](#testing)
- [Deployment](#deployment)
- [Security](#security)
- [Repository Layout](#repository-layout)
- [License](#license)

---

## Architecture

Six layers, implementing `gsaralcloud_mra_platform_architecture.svg`.

```text
 L1  INGESTION      Excel/CSV → alias-resolved canonical records → validation gate
                    core/ingestion/normalizer.py
                              │
 L2  COORDINATOR    LangGraph StateGraph. The hub. Single writer of state.
                    core/orchestrator/coordinator.py
                              │
      validate ──► wave1 ──► promote ──► wave2 ──► plan ──► report ──► END
          │                     │
          └─────────────────────┴──────────────► hitl_review ─────────┘
                              │
 L3  WAVE 1         right_sizing · dependency_mapping · bucketing_7r
     (parallel)     inputs are independent, so they fan out concurrently
                              │
 L4  WAVE 2         tco_calculation · assessment · gap_analysis
     (serial)       each consumes Wave 1 verdicts
                              │
 L5  PLANNING       DependencySequencer · PriorityScorer (MPS) · MigrationFactory
                    core/planning/engine.py
                              │
 L6  REPORTING      Excel workbook (8 sheets) + HTML executive summary
                    core/output/report_compiler.py
```

---

## Features

| Concern | Module | Notes |
|---|---|---|
| Orchestration | `core/orchestrator/coordinator.py` | LangGraph StateGraph, hub-and-spoke |
| Agents | `mra/agents/spokes.py` | Seven spokes, one typed verdict each |
| A2A contracts | `mra/schemas/` | `AgentOutput` envelope + per-agent verdicts |
| Model routing | `core/llm/router.py` | Per-agent tier policy with escalation |
| Providers | `core/llm/providers.py` | Anthropic live + deterministic mock |
| Guardrails | `core/guardrails/engine.py` | PII masking, injection detection, schema validation |
| Memory | `core/memory/store.py` | Namespaced pgvector, Redis context, episodic |
| MCP tools | `core/mcp/registry.py` | Six tools: EC2, EOL, rate card, CAF rubric |
| Planning (L5) | `core/planning/engine.py` | Topological sort, MPS scoring, effort model |
| Reporting (L6) | `core/output/report_compiler.py` | Excel + HTML deliverables |
| Evaluation | `core/evaluation/harness.py` | 13 domain metrics with a CI gate |
| Observability | `core/observability/telemetry.py` | Trace store, Prometheus, optional Langfuse |
| Live tracking | `core/observability/eventbus.py` | SSE stream + outbound webhook |
| API + UI | `app/` | FastAPI, JWT auth, ten-tab dashboard |

### Model Routing

Right-sizing is a deterministic mapping exercise; gap analysis is a judgement task. Routing each agent to an appropriate tier is the single largest cost lever in an agentic system.

| Tier | Agents | Rationale |
|---|---|---|
| `fast` | `right_sizing`, `tco_calculation` | Schema-bound work over a fixed rubric |
| `balanced` | `dependency_mapping`, `bucketing_7r`, `wave_planning` | Domain reasoning against a rubric |
| `deep` | `assessment`, `gap_analysis` | Open-ended, executive-facing synthesis |

> A verdict that fails schema validation retries once, one tier up.

### Guardrails

Uploaded inventories are untrusted input — a `Notes` column is a realistic injection vector in this exact product. A 488-workload assessment over one poisoned cell is the wrong trade-off.

### Live Execution Tracking

Every node entry, edge traversal, agent verdict, MCP tool call, and guardrail event is emitted to an event bus and streamed to the browser over Server-Sent Events (SSE).

The **Live execution** tab renders the graph with the active node pulsing and completed nodes filled, so you can see exactly where execution is at any instant — including the three Wave 1 spokes running in parallel.

> Runs finish in about a second on the mock provider, so the stream **replays the whole run** before going live; a late-connecting browser misses nothing.

---

## Output (Layer 6)

Generated automatically at the end of every run into `reports/`.

### `MRA_<client>_<run_id>.xlsx`

Eight sheets mirroring a standard GSaralCloud MRA deliverable, so it drops into an existing engagement without reformatting.

| Sheet | Contents |
|---|---|
| `01_Executive Summary` | Headline metrics with the basis for each |
| `02_Workload Inventory` | Every workload with its agent verdicts joined |
| `03_TCO Analysis` | Per-strategy rollup over the shared rate card |
| `04_Wave Planning` | Waves with MPS score, effort, and runbook detail |
| `05_CAF Assessment` | Six-perspective readiness scorecard |
| `06_Gap Analysis` | Current versus target register |
| `07_Right-Sizing` | Per-workload instance recommendations |
| `08_Quality Metrics` | The platform's evaluation of its own output |

**Totals are live formulas, not baked-in numbers.** Change a rate and every dependent figure recalculates. A workbook of hardcoded values is a screenshot with extra steps, and it will not survive a committee that wants to test an assumption.

### `MRA_<client>_<run_id>.html`

Self-contained, branded executive summary.

---

## Evaluation

Generic LLM metrics do not tell you whether a migration assessment is any good. The harness scores domain metrics against ground truth in the source inventory.

> **Note:** Three metrics are informational, on purpose.

---

## Getting Started

Start with [`SETUP-VSCODE.md`](SETUP-VSCODE.md) for local development and debugging. See [Deployment](#deployment) for Docker and AWS EKS.

---

## Configuration

Everything is environment-driven; `.env.example` is fully annotated.

| Variable | Default | Purpose |
|---|---|---|
| `LLM_PROVIDER` | `mock` | `mock` or `anthropic` |
| `ANTHROPIC_API_KEY` | — | Required only when provider is `anthropic` |
| `WEBHOOK_URL` | — | Optional outbound execution webhook |
| `REPORTS_DIR` | `reports` | Where Layer 6 writes |
| `DEMO_USER` / `DEMO_PASSWORD` | `Admin` / `Admin` | Demo login |
| `ALLOW_UPLOADS` | `true` | `false` locks every assessment to the bundled inventory and rejects uploads with a `403`, server-side (see note below) |
| `JWT_SECRET` | placeholder | **Change before any real deployment** |
| `HITL_CONFIDENCE_THRESHOLD` | `0.65` | Below this, route to human review |
| `LANGFUSE_ENABLED` | `false` | Mirror spans to self-hosted Langfuse |

> **`ALLOW_UPLOADS`** is enforced in `app/main.py::start_run`, not just hidden in the UI. This is what the AWS demo profile sets.

Postgres and Redis both **degrade gracefully** to in-process stores, so the platform runs on a laptop with nothing but Python.

---

## Testing

No Postgres, no Redis, no API key required. A test suite that needs infrastructure is a test suite that stops being run.

```bash
pytest
```

---

## Deployment

| Stage | Guide | What it adds |
|---|---|---|
| 1. VS Code | [`SETUP-VSCODE.md`](SETUP-VSCODE.md) | Local development and debugging |
| 2. Docker | [`RUNBOOK-LOCAL.md`](RUNBOOK-LOCAL.md) | Real pgvector and Redis |
| 3. AWS EKS | [`RUNBOOK-AWS.md`](RUNBOOK-AWS.md) | Public demo URL, IRSA, Secrets Manager, ALB — mock inference and fixed dataset by default, cost-minimised (~$156/mo) |

---

## Security

> [!WARNING]
> This is a **demonstration environment**. Complete the steps below before using real client data.

1. Replace the single shared demo account with Cognito or your IdP — there is no user store, no rotation, no MFA, and no per-user audit trail.
2. Set a long random `JWT_SECRET`.
3. Move secrets to AWS Secrets Manager (already wired for EKS — see `core/aws/secrets.py`).
4. Tighten the CORS policy in `app/main.py`.
5. Put the API behind TLS.

The app logs a warning at startup while the credentials or JWT secret are at their shipped defaults.

---

## Repository Layout

```text
.
├── SETUP-VSCODE.md     # Start here
├── RUNBOOK-LOCAL.md    # Docker Desktop guide
├── RUNBOOK-AWS.md      # AWS EKS guide
├── api.http            # REST Client scratchpad
├── .vscode/            # Launch, tasks, settings, extensions
├── app/                # FastAPI + static dashboard
├── core/               # Platform layers (see Features)
├── mra/                # Domain: agents, schemas, workflow
├── data/               # Bundled GreenTech inventory
├── reports/            # Generated deliverables
├── scripts/            # Fixtures, debug harness, AWS deployment
├── k8s/                # EKS manifests
├── iam/                # IAM policy
└── tests/              # 77 tests
```

---

## License

© GSaralCloud. Internal reference implementation.
