# NanoGate

> **Every AI request takes the cheapest safe route — with proof.**

NanoGate is a local-first AI control plane for enterprise applications, built to run on the **HP ZGX Nano**. It sits between your application and AI models. For every request it decides what is safe, what is allowed, what is cheapest, and what is reliable enough, then records a tamper-evident receipt explaining the decision.

Existing apps adopt NanoGate by changing two values: the OpenAI-compatible `base_url` and the API key.

---

## How it works

```text
Application (OpenAI SDK)
    │
    ▼
Authenticate ─► Derive identity (tenant, department, policy)
    │
    ▼
DLP scan ─► Classify data (Public / Internal / Confidential / Restricted / Secret)
    │
    ▼
Hard policy ─► Allowed routes for this request
    │
    ▼
Verified semantic cache ──── verified hit ───► Response
    │ miss
    ▼
Local model (small) ─► Router computes calibrated p_error
    │
    ├─ p_error ≤ threshold ─► keep local answer
    ├─ else, if allowed      ─► Local large
    ├─ else, if allowed      ─► Remote provider
    └─ else                  ─► Abstain
    │
    ▼
Output DLP scan ─► Budget settlement ─► Sealed receipt ─► OpenAI-compatible response
```

Hard policy always runs before routing. The router can prefer a route, but it can never override policy.

> **Note on streaming:** Because the router and output scan judge the full answer, responses on escalation-eligible routes are generated, checked, and then sent. Streaming is only passed through directly on routes where no escalation is possible.

---

## Models

NanoGate uses several small, specialized models rather than one.

| Component | Model | Purpose |
|---|---|---|
| Local Small LLM | Configurable (`LOCAL_MODEL_NAME`) | First attempt at every uncached request |
| Local Large LLM | Configurable (`LOCAL_LARGE_MODEL_NAME`) | Escalation when `p_error` is too high |
| Embeddings | `BAAI/bge-small-en-v1.5` | Cache candidate retrieval |
| Cache verifier | Local LLM as judge | Confirms two requests are truly equivalent |
| Router | Logistic regression + calibration (scikit-learn) | Predicts the probability the local answer is unacceptable |
| DLP | Presidio + spaCy NER + regex/secret patterns | Detects PII, credentials, and secrets |
| Remote (optional) | Any OpenAI-compatible provider | Used only when policy permits |

LLM choices are set in `.env`; nothing is hardcoded. The actual model name, revision, and runtime in use are shown in the dashboard and in every receipt.

---

## Quickstart

### Requirements

- HP ZGX Nano (or another CUDA machine for development)
- Python 3.11+
- Node 20+
- Docker / container runtime
- A local OpenAI-compatible inference server (e.g. vLLM)

### Setup

```bash
git clone <repo-url> nanogate && cd nanogate
cp .env.example .env        # fill in model endpoints and keys
make doctor                 # checks hardware, runtime, model, ports, artifacts
make setup                  # installs backend + frontend dependencies
make seed-data              # loads datasets and demo tenants/policies
make train-router           # trains router + calibrator, writes artifacts/
make start                  # starts gateway and dashboard
```

- Gateway: `http://localhost:8000`
- Dashboard: `http://localhost:5173`

### Configuration

```bash
# Local inference
LOCAL_MODEL_BASE_URL=http://localhost:8001/v1
LOCAL_MODEL_NAME=
LOCAL_MODEL_API_KEY=
LOCAL_MODEL_FAMILY=
LOCAL_MODEL_REVISION=

LOCAL_LARGE_MODEL_BASE_URL=
LOCAL_LARGE_MODEL_NAME=

# Remote provider (optional; policy-gated)
REMOTE_ENABLED=false
REMOTE_BASE_URL=
REMOTE_API_KEY=

# Embeddings
EMBEDDING_MODEL=BAAI/bge-small-en-v1.5

# Storage
DATABASE_URL=sqlite:///./nanogate.db

# Receipts
RECEIPT_HMAC_KEY=
```

Policies, model aliases, and pricing live in `config/policies.yaml`, `config/model_aliases.yaml`, and `config/pricing.yaml`.

---

## Using NanoGate from an application

```python
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="ng-it-dept-key",
)

resp = client.chat.completions.with_raw_response.create(
    model="default",
    messages=[{"role": "user", "content": "How do I reset the VPN client?"}],
)

print(resp.parse().choices[0].message.content)
print(resp.headers["x-nanogate-route"])    # e.g. local_small
print(resp.headers["x-nanogate-reason"])   # e.g. LOCAL_CONFIDENT
```

Every response includes these headers:

| Header | Meaning |
|---|---|
| `x-nanogate-request-id` | Unique request ID |
| `x-nanogate-receipt-id` | ID of the sealed decision receipt |
| `x-nanogate-route` | `cache`, `local_small`, `local_large`, `remote`, or `abstain` |
| `x-nanogate-reason` | Reason code (see below) |
| `x-nanogate-policy-version` | Policy version applied |
| `x-nanogate-data-class` | Data classification |
| `x-nanogate-estimated-cost-usd` | Estimated cost of this request |

### Reason codes

```text
CACHE_VERIFIED  CACHE_HARD_NEGATIVE  CACHE_NAMESPACE_MISMATCH  CACHE_STALE
LOCAL_CONFIDENT  LOCAL_LARGE_SELECTED
REMOTE_ALLOWED  REMOTE_DISABLED  REMOTE_UNAVAILABLE
SENSITIVE_LOCAL_ONLY  SECRET_BLOCKED  POLICY_BLOCK  BUDGET_DENY
ROUTER_ABSTAIN  OUTPUT_BLOCKED  MODEL_UNAVAILABLE  RATE_LIMITED
TENANT_SPOOF_REJECTED  DEPARTMENT_SPOOF_REJECTED  DLP_UNAVAILABLE_FAIL_CLOSED
```

---

## Verified semantic cache

A plain semantic cache asks *"does this look similar?"* NanoGate asks *"is this equivalent, authorized, and still valid?"*

1. **Retrieve:** embed the request with bge-small and search only the caller's tenant/department namespace.
2. **Verify:** the local LLM judges whether the candidate matches on intent, subject, ownership, authorization, scope, and timeframe.
3. **Accept** only if every check passes and the entry is not stale or revoked.

| Request | Expected result |
|---|---|
| "How do I reset the VPN client?" | Cached after first answer |
| "What steps restore my VPN connection?" | `CACHE_VERIFIED` |
| "How do I reset another employee's VPN password?" | `CACHE_HARD_NEGATIVE` |
| Same canonical request from another tenant | `CACHE_NAMESPACE_MISMATCH` |

---

## Router (intelligence layer)

The router estimates the probability that the local answer is unacceptable.

- **Training data:** questions from a public QA dataset are answered by the local model and graded against reference answers (`y_error = 1` if unacceptable).
- **Features:** prompt size, department, data class, intent, cache similarity signals, token log-probabilities, output length, stop reason, and system load. Missing features have explicit indicators.
- **Split:** 60% train / 20% calibration + threshold selection / 20% frozen test, grouped by canonical intent so paraphrases never leak across splits.
- **Calibration:** isotonic regression, or Platt scaling if the calibration set is small.
- **Threshold:** selected on validation data, never on the test set.

`p_error` is a calibrated probability estimate, not certainty.

---

## API

| Area | Endpoints |
|---|---|
| OpenAI-compatible | `POST /v1/chat/completions`, `GET /v1/models` |
| Receipts | `GET /api/receipts`, `GET /api/receipts/{receipt_id}` |
| Requests | `GET /api/requests/live` |
| Policies | `GET /api/policies`, `GET /api/policies/{id}`, `POST /api/policies/test`, `POST /api/policies/publish` |
| Router | `GET /api/router/model-card`, `GET /api/router/metrics`, `POST /api/router/evaluate`, `POST /api/router/threshold-preview` |
| Cache | `GET /api/cache/metrics`, `GET /api/cache/candidates`, `POST /api/cache/test-pair` |
| FinOps | `GET /api/finops/summary`, `GET /api/finops/departments`, `GET /api/finops/rates` |
| Infrastructure | `GET /api/infrastructure`, `GET /api/infrastructure/services`, `GET /api/infrastructure/models` |
| Health | `GET /healthz`, `GET /readyz`, `GET /metrics` |

---

## Dashboard

| Route | What it shows |
|---|---|
| `/overview` | Live decision stream, key metrics, decision-flow view |
| `/requests` | Filterable request explorer |
| `/requests/:receiptId` | Full decision receipt with integrity verification |
| `/policies` | Policy Studio with a live prompt tester |
| `/router-lab` | Router metrics, reliability and risk-coverage charts, threshold studio |
| `/cache` | Cache metrics and semantic pair tester |
| `/finops` | Measured costs and scenario projections (kept separate) |
| `/infrastructure` | ZGX Nano telemetry, models, services, trust boundary |

All live data arrives from the backend over Server-Sent Events. The frontend holds no business logic and no hardcoded metrics.

---

## Decision receipts

Every answered or denied request produces a receipt containing identity, data classification, cache decision, model details, router scores, chosen route, cost, device telemetry, and integrity hashes. Receipts are SHA-256 hash-chained; modifying any field breaks verification.

```bash
make test-security   # includes the receipt-tampering test
```

---

## Commands

| Command | Purpose |
|---|---|
| `make doctor` | Diagnose hardware, runtime, model, ports, and artifacts |
| `make setup` | Install dependencies |
| `make dev` / `make start` / `make stop` | Run the stack |
| `make seed-data` / `make seed-demo` | Load datasets and demo data |
| `make train-router` | Train router and calibrator |
| `make test` / `make test-security` | Run test suites |
| `make benchmark` | Run router, cache, DLP, security, and load benchmarks |
| `make offline-test` | Verify core functionality with internet disconnected |
| `make demo` / `make clean-demo` | Run or reset the demo |
| `make redact-check` | Confirm no sensitive values appear in logs or the repo |

---

## Repository structure

```text
nanogate/
  apps/          gateway/ (FastAPI), dashboard/ (React + TypeScript)
  nanogate/      auth, policy, dlp, cache, inference, router, budget, audit, telemetry, security
  config/        policies.yaml, model_aliases.yaml, pricing.yaml
  datasets/      knowledge base, intents, requests, PII spans, splits, data cards
  artifacts/     trained router, calibration, benchmark runs
  bench/         evaluation and load-test scripts
  tests/         unit, integration, security, e2e
  scripts/       setup, start, demo, training, benchmark helpers
  docs/          architecture, threat model, evaluation, results, demo runbook
```

---

## Data

- **Live operational data:** generated by the running system.
- **Public datasets:** provenance, license, and checksums recorded in `datasets/DATA_SOURCES.md`.
- **Synthetic security data:** deterministic, clearly labeled test cases for PII, secrets, and attacks. No real personal data or credentials are used.
- **Scenario data:** planning assumptions (electricity rates, annual volume), always labeled `Scenario` and never mixed with measured values.

---

## Results

Measured results live in [`docs/RESULTS.md`](docs/RESULTS.md) and in timestamped runs under `artifacts/benchmarks/`. Each run records its git commit, dataset hash, model revisions, threshold, and hardware summary.

No numbers are reported in this README until they have been measured.

---

## Honesty rules

- Telemetry that cannot be measured is shown as **Unavailable** with a reason, never as a placeholder number.
- Scenario estimates are always separate from measured totals.
- We do not claim perfect privacy, zero hallucinations, or guaranteed savings.
- Known limitations are documented in [`docs/LIMITATIONS.md`](docs/LIMITATIONS.md).

---

## Documentation

[Architecture](docs/ARCHITECTURE.md) · [Threat model](docs/THREAT_MODEL.md) · [AI evaluation](docs/AI_EVALUATION.md) · [Model card](docs/MODEL_CARD.md) · [Results](docs/RESULTS.md) · [Demo runbook](docs/DEMO_RUNBOOK.md) · [Limitations](docs/LIMITATIONS.md)
