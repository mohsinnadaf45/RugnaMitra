# RugnaMitra

# Predictive Hospital Bed Management

> PS #16 · Healthcare & Medical Technology · Software

An EHR-integrated forecasting and decision-support system that predicts **patient admissions**, **length of stay (LOS)** and **discharge times**, then recommends **bed allocation** to cut ED boarding and wait times.

**Status:** pre-development (PRD complete, dated 11 Oct 2026). Targets and thresholds below are proposals to validate against real hospital baselines.

---

## Table of contents

1. [Problem](#problem)
2. [Solution overview](#solution-overview)
3. [Key features](#key-features)
4. [ML models](#ml-models)
5. [Tech stack](#tech-stack)
6. [Deployment](#deployment)
7. [Repository structure](#repository-structure)
8. [Getting started](#getting-started)
9. [Data sources](#data-sources)
10. [EHR integration](#ehr-integration)
11. [Evaluation and success metrics](#evaluation-and-success-metrics)
12. [Roadmap](#roadmap)
13. [8-week MVP plan](#8-week-mvp-plan)
14. [Security, privacy and ethics](#security-privacy-and-ethics)
15. [Risks and open questions](#risks-and-open-questions)
16. [Glossary](#glossary)
17. [Contributing and license](#contributing-and-license)

---

## Problem

Hospital bed supply is fixed in the short term, demand is random and cyclical, and the data needed to match them is scattered. Bed managers work from whiteboards, phone calls and a morning census, so shortages are noticed only after patients are already waiting.

Typical failure points:

| Stage | What goes wrong today |
| --- | --- |
| Arrival planning | Planned from last week's average or gut feel |
| ED to ward handoff | No view of which bed opens in the next 2 to 4 hours (ED boarding) |
| Inpatient stay | LOS estimated once at admission, rarely updated |
| Discharge | Bunched in the afternoon; paperwork, pharmacy and transport done serially |
| Bed turnover | Cleaning requested only after the patient leaves |
| Elective surgery | Booked without checking downstream bed capacity |
| Allocation | Gender, isolation and specialty constraints checked by hand |

## Solution overview

A read-only layer on top of the EHR/HIS that gives a **72-hour forward view** of demand and bed supply and a **recommended action** for every predicted shortfall.

- Predict, then prescribe: forecasts always come with a suggested action.
- Human in the loop: the system recommends, staff decide and can override with a reason.
- Explain every number: top drivers and uncertainty ranges on all predictions.
- Fit the existing workflow: bed huddles, shift handovers, ADT events.
- Start read-only: v1 never writes orders back into the EHR.

## Key features

| ID | Feature | Priority |
| --- | --- | --- |
| F1 | EHR ingestion (HL7 v2 ADT, FHIR R4) | P0 |
| F2 | Live bed board (occupied, clean, dirty, blocked, reserved) | P0 |
| F3 | Admission forecast per ward and hour (4h, 24h, 72h) | P0 |
| F4 | Remaining LOS prediction per inpatient | P0 |
| F5 | Discharge time window with confidence | P0 |
| F6 | Capacity forecast and shortfall probability per ward | P0 |
| F7 | Configurable alerts | P0 |
| F8 | Explanations for every prediction | P0 |
| F9 | Bed assignment recommender | P1 |
| F10 | Discharge barrier tracker | P1 |
| F11 | Override and feedback logging | P1 |
| F12 | What-if scenario simulator | P1 |
| F13 | Elective scheduling capacity check | P1 |
| F14 | EVS and transport pre-notification | P1 |
| F15 | Reports and KPIs | P1 |
| F16 | Multi-hospital support | P2 |
| F17 | Staffing forecast | P2 |
| F18 | Mobile app | P2 |

**Non-goals (v1):** diagnosis or treatment advice, automatic EHR write-back, outpatient scheduling, billing and claims prediction, replacing the bed manager's authority.

## ML models

| Model | Predicts | Suggested approach | Baseline to beat |
| --- | --- | --- | --- |
| M1 Admission forecaster | Arrivals per ward, source, hour | LightGBM with calendar, lag, weather features; Prophet/ETS benchmark; quantile loss for intervals | Same weekday/hour average of last 8 weeks |
| M2 LOS predictor | Remaining LOS (days) | Gradient boosting with survival framing | Mean LOS by diagnosis group and ward |
| M3 Discharge predictor | Discharge probability per 4h window, up to 72h | Discrete-time survival / classifier on hourly snapshots | Ward-level discharge hour distribution |
| M4 Capacity aggregator | Free beds and shortfall probability | Monte Carlo over M1 arrivals and M3 discharges | Census plus planned discharges |
| O1 Bed recommender | Best bed or action | Constraint optimization (OR-Tools) | Manual allocation |

Practices: time-based splits only (no random split), no feature leakage, calibrated probabilities, SHAP explanations, subgroup fairness checks, weekly retraining of forecasters, champion/challenger promotion after at least 2 weeks of shadowing.

## Tech stack

| Layer | Choice |
| --- | --- |
| Integration | HAPI FHIR / custom adapters; a small forwarder at the hospital for HL7 v2 (see ARCHITECTURE.md) |
| Queue | Redis (Render Key Value) with Redis Streams, or a Postgres job table |
| Storage | PostgreSQL (Render managed); S3-compatible object storage (AWS S3 or Cloudflare R2) for datasets and model files |
| Feature store | Postgres feature tables (Feast later if needed) |
| ML | Python, scikit-learn, LightGBM, statsmodels/Prophet, SHAP, OR-Tools |
| MLOps | MLflow (or model files in object storage for the MVP), Evidently |
| Backend | FastAPI on Render (web service, background worker, cron jobs) |
| Frontend | React + TypeScript (Vite or Next.js) on Vercel, ECharts or Recharts |
| Notifications | SMS gateway, SMTP email, web push |
| Local dev | Docker Compose |
| Observability | Render logs and metrics, Sentry, optional Grafana |

## Deployment

| Part | Platform |
| --- | --- |
| Frontend (`web/`) | Vercel |
| API, workers, scheduled jobs (`api/`, `models/`, `mlops/`) | Render |
| Database and queue | Render Postgres and Render Key Value |

Full design, data flow, `render.yaml`, `vercel.json` and environment variables are in [ARCHITECTURE.md](ARCHITECTURE.md).

> Use only synthetic or de-identified data on public cloud platforms until a compliance review approves real patient data.

## Repository structure

Proposed layout (adjust as you build):

```
.
├── README.md
├── ARCHITECTURE.md          # system design and Vercel + Render deployment
├── render.yaml              # Render Blueprint (backend infrastructure)
├── docs/                    # PRD, safety case
├── data/
│   ├── synthetic/           # generated ADT and encounter data
│   └── external/            # holidays, weather
├── ingestion/               # HL7 v2 / FHIR adapters, validators
├── features/                # feature definitions and pipelines
├── models/
│   ├── admissions/          # M1
│   ├── los/                 # M2
│   ├── discharge/           # M3
│   ├── capacity/            # M4 simulation
│   └── optimizer/           # O1 bed recommender
├── api/                     # FastAPI service
├── web/                     # React dashboard
├── mlops/                   # training DAGs, registry, drift monitors
├── infra/                   # docker-compose, k8s manifests
├── tests/
└── notebooks/               # exploration and evaluation
```

## Getting started

> Commands are placeholders until the code exists; update them as you implement each piece.

**Prerequisites:** Python 3.11+, Node 20+, Docker and Docker Compose, PostgreSQL 15+.

```bash
# 1. Clone
git clone <your-repo-url>
cd predictive-bed-management

# 2. Python environment
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# 3. Start infrastructure (database, queue, MLflow)
docker compose -f infra/docker-compose.yml up -d

# 4. Generate synthetic data
python -m ingestion.synthetic --months 24 --out data/synthetic

# 5. Train baseline and first models
python -m models.admissions.train --config configs/m1.yaml

# 6. Run API and dashboard
uvicorn api.main:app --reload
cd web && npm install && npm run dev
```

## Data sources

Verify access terms and licences before use.

| Source | Use |
| --- | --- |
| MIMIC-IV (credentialed access via PhysioNet) and its open demo subset | Admissions, transfers, ICU stays for LOS and discharge models |
| eICU Collaborative Research Database | Multi-hospital ICU stays |
| Synthea | Synthetic patient records exported as FHIR |
| Custom synthetic generator | ED arrival streams with weekday, seasonal and surge effects (public datasets rarely include these) |
| Weather, holidays, local events | External features for M1 |

For a real hospital you need 2 to 3 years of ADT and encounter history (minimum 12 months), ethics approval and a data-sharing agreement before training.

## EHR integration

| Method | Use |
| --- | --- |
| HL7 v2 ADT (A01, A02, A03, A08, A11, A13) | Real-time patient and bed movement |
| FHIR R4 REST and Subscriptions | Encounter, Patient, Observation, Condition, Procedure, MedicationRequest, ServiceRequest, Location, Appointment |
| Bulk export / replica / CSV | Historical training data |
| Scheduling and OT system API | Planned surgeries and elective admissions |
| EVS and transport systems | Cleaning and transport task status |

In India, align with ABDM FHIR profiles where the hospital is onboarded. Pseudonymize identifiers before training and keep the re-identification key inside the hospital.

## Evaluation and success metrics

Targets are proposals; replace with measured baselines from the pilot hospital.

| Metric | Target (6 months post go-live) |
| --- | --- |
| Median ED-to-ward bed wait | 20% reduction |
| ED boarding hours | 25% reduction |
| Discharges before noon | From 10-15% to 30% |
| Bed-related surgery cancellations | 30% reduction |
| 24h ward census forecast error (MAPE) | 8% or less |
| Admission forecast error (weighted MAPE per ward per 4h) | 15% or less and better than baseline |
| Discharges within +/- 4h of prediction | 75% or more |
| LOS error (general wards) | Under 1.0 day |
| Recommendation acceptance | 60% or more |
| Data freshness | Under 5 min (p95) |
| Dashboard uptime in clinical hours | 99.5% or more |

**Guardrails:** 30-day readmissions, patient safety events, staff workload and alert fatigue, forecast fairness across age, sex and payer groups.

**Methods:** rolling-origin backtests, 12-week shadow mode, pilot on two wards versus matched controls (difference-in-differences), usability testing (SUS 75+), monthly safety review.

## Roadmap

| Phase | Weeks | Focus | Gate to exit |
| --- | --- | --- | --- |
| Baseline | 1-4 | Map workflows, measure KPIs, approvals | Approvals granted |
| Integrate | 5-12 | ADT feed, history load, data checks | Feed stable, data clean |
| Shadow run | 13-24 | Train M1 to M3, silent predictions, backtest | Beats baseline, safety case signed |
| Pilot | 25-36 | Two wards live: forecasts and alerts | At least 2 of 3 primary KPIs improve, guardrails flat |
| Scale | 37-52 | All wards, recommender, simulator, retraining cycle | n/a |

## 8-week MVP plan

For a project or hackathon build using public and synthetic data.

| Week | Deliverable |
| --- | --- |
| 1 | Scope, personas, datasets, canonical schema, repo and CI |
| 2 | Synthetic ADT generator, FHIR/HL7 ingestion into the canonical store |
| 3 | M1 admission forecaster with baseline and backtest |
| 4 | M2 LOS and M3 discharge models with calibration |
| 5 | M4 capacity simulation and shortfall alerts |
| 6 | Dashboard: command centre, bed board, ward view |
| 7 | O1 recommender, SHAP explanations, override and audit logging |
| 8 | Evaluation report, demo script, documentation, presentation |

## Security, privacy and ethics

- TLS 1.2+ in transit, AES-256 at rest, secrets in a vault, RBAC with SSO, MFA for admins, immutable audit log.
- Compliance checklist (confirm with counsel per region): India DPDP Act 2023 and ABDM policies, HIPAA (US), GDPR (EU).
- Keep claims to operational decision support; get a regulatory assessment before any patient-level clinical use.
- Predictions must never delay or deny care or pressure early discharge; clinician decisions always override.
- Fairness audit before launch and quarterly; no protected attributes used to deprioritize patients.
- Ethics committee approval before using historical data and before the pilot.

## Risks and open questions

Top risks: slow EHR integration, poor or short history, low staff trust, alert fatigue, model drift, privacy incidents, bias.

Open questions:

- Which EHR/HIS does the target hospital use, and does it expose FHIR or HL7 v2?
- Single hospital or group, and which country's regulation applies first?
- Which wards are in the pilot, and is ICU in scope for v1?
- Who owns the decision when the model and bed manager disagree?
- Is there an existing bed management module to integrate with?
- Deployment: on-premise, region-locked cloud, or hybrid?

## Glossary

| Term | Meaning |
| --- | --- |
| ADT | Admit, discharge, transfer events |
| ED boarding | Admitted patients waiting in the ED for a ward bed |
| LOS | Length of stay |
| EHR / HIS | Electronic health record / hospital information system |
| EVS | Environmental services (bed cleaning) |
| MAPE | Mean absolute percentage error |
| PHI | Protected health information |
| SHAP | Method to explain predictions by feature contribution |
| Census | Number of occupied beds at a point in time |

## Contributing and license

- Branch from `main`, open a pull request, and require tests plus one review.
- Never commit real patient data or credentials.
- License: _choose one (for example MIT or Apache-2.0) before publishing._
