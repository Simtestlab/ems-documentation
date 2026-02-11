# Sim EMS — Implementation Roadmap (Python + FastAPI + PostgreSQL + React)

**Document Type**: Roadmap + Technical Specification (Implementation-Ready)  
**Date**: January 20, 2026  
**Primary Tech Stack**: Python, FastAPI, PostgreSQL (TimescaleDB optional), React  

---

## 0) Executive Summary

**Sim EMS** is a best-in-class Energy Management System that combines:

- **Real-time edge control** (deterministic cycle, state machines, safety-first control)
- **Enterprise analytics** (multi-tenancy, hierarchy, reporting, billing, carbon)
- **Predictive intelligence** (forecasting + optimization via MPC)

This README defines:

- A **phased roadmap** from MVP to production
- The **Python-first architecture** for both Edge and Backend
- The **algorithms** (control, estimation, forecasting, optimization)
- The **design patterns** and standards used across the system

---

## 1) Canonical Tech Stack

### 1.1 Edge (On-Prem / Device)

**Language**: Python 3.11+  
**Runtime**: Linux (Raspberry Pi / IPC), systemd  
**Control Loop**: asyncio-based cycle (deterministic)  

Recommended libraries:
- Async: `asyncio`, `anyio`
- Messaging (optional): `paho-mqtt`, `pyzmq`
- Modbus: `pymodbus`
- CAN: `python-can`, `cantools`
- Local storage: `sqlite3` (local config/state), optional `duckdb`
- Validation: `pydantic`
- Logging: Python `logging` with JSON formatter

### 1.2 Backend (Cloud / Data Platform)

**API**: FastAPI (Python 3.11+)  
**DB**: PostgreSQL 15+ (TimescaleDB extension recommended for time-series)  
**Caching/Queue (optional)**: Redis + Celery/RQ  

Recommended libraries:
- FastAPI + `uvicorn`
- Auth: `python-jose`, `passlib`, OIDC integration (enterprise)
- DB: `SQLAlchemy` + `alembic` (migrations)
- Async DB: `asyncpg` (or SQLAlchemy async)
- Observability: `prometheus-client`, OpenTelemetry (optional)

### 1.3 UI (Web)

**Frontend**: React 18 + TypeScript  
**Visualization**: Apache ECharts (or Recharts)  
**State**: Redux Toolkit (or React Query + Zustand)  

### 1.4 Integration Fabric (Recommended)

- Telemetry ingestion: MQTT/HTTPS/WebSocket
- Command path: HTTPS + MQTT/WebSocket to edge
- Schema: versioned JSON payloads + strict validation

---

## 2) System Architecture (Python-First)

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                           BACKEND (FastAPI)                                   │
│  • Auth/RBAC • Tenants • Org Hierarchy • Device Registry                       │
│  • Ingestion APIs • Reports • Billing • Carbon • Optimization Schedules        │
│  • WebSocket for Live Data • Admin APIs                                        │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │ TLS
                                │
┌───────────────────────────────▼───────────────────────────────────────────────┐
│                               DATA (PostgreSQL)                                │
│  • Metadata tables • Timeseries hypertables (TimescaleDB optional)              │
│  • RLS policies (multi-tenant) • Audit logs                                    │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────────────┐
│                         EDGE (Python Control Core)                              │
│  • IPO cycle + Process Image + Scheduler                                        │
│  • Controllers (Strategy plugins)                                               │
│  • Bridges (Modbus/CAN/MQTT/HTTP/OCPP)                                          │
│  • Local buffer • Offline-first                                                 │
└───────────────────────────────┬───────────────────────────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────────────────────────┐
│                          PHYSICAL DEVICES                                       │
└───────────────────────────────────────────────────────────────────────────────┘
```

---

## 3) Design Patterns (What We Use and Why)

### 3.1 Edge Control Patterns

1. **IPO Cycle (Input–Process–Output)**
   - Ensures predictable read/compute/write sequencing.

2. **Process Image (Immutable Snapshot)**
   - Input writes to `next_value`; at cycle start, snapshot swaps into `value`.
   - Controllers read only from the frozen snapshot.

3. **Priority Scheduler**
   - Controllers run in strict order: Manual override → Safety → Grid → Optimization → Aux.

4. **State Machine (Hierarchical FSM)**
   - Top-level states: `INIT`, `STANDBY`, `RUN`, `FAULT`, `SHUTDOWN`.
   - Control-mode states: `PEAK_SHAVING`, `SELF_CONSUMPTION`, `DROOP`, `BACKUP`, `ISLAND`.

5. **Strategy Pattern (Controllers)**
   - Each control strategy implements a common interface (`Controller.execute(ctx)`).

6. **Adapter Pattern (Device Drivers)**
   - Protocol-specific mapping to standardized device interfaces (ESS, Meter, PV, EVCS).

7. **Circuit Breaker + Retry/Backoff (Bridges)**
   - Prevents cascading failures from unstable devices.

### 3.2 Backend Patterns

1. **API Gateway (FastAPI)**
   - Consolidates authentication, authorization, and routing.

2. **Domain-Driven Boundaries**
   - Separate domains: telemetry, billing, carbon, reporting, optimization.

3. **Event-Driven Processing (Optional)**
   - Ingestion triggers aggregation jobs, anomaly checks, report scheduling.

4. **Multi-Tenant Isolation**
   - Recommended: PostgreSQL Row-Level Security (RLS) + tenant_id everywhere.

5. **CQRS (Optional at scale)**
   - Separate write model (ingestion) from read model (dashboards/reports).

---

## 4) Algorithms (Control, Estimation, Forecasting, Optimization)

This section lists the canonical algorithms Sim EMS uses. Every algorithm must be:
- bounded-time on edge (or moved to backend)
- observable (metrics)
- testable (scenario replay)

### 4.1 Control Algorithms (Edge)

1. **Peak Shaving (Reactive + Hysteresis + Ramp Limit)**
   - Objective: keep grid import below threshold to reduce demand charges.
   - Key elements:
     - hysteresis band to avoid oscillation
     - ramp-rate limiting to reduce stress

2. **Self-Consumption Balancing (Grid-Zero / Export-Limited)**
   - Objective: maximize onsite PV usage by charging/discharging BESS.

3. **Frequency Droop Control (Primary Frequency Support)**
   - Objective: respond to frequency deviations.
   - Enhancements:
     - deadband
     - SOC reserve window

4. **Virtual Inertia (Optional)**
   - Adds dF/dt component for fast frequency response.

5. **Reactive Power / Power Factor Control**
   - Objective: maintain PF targets or voltage support where supported.

6. **Backup Reserve + Island Mode Sequencing**
   - Maintains SOC floor for backup.
   - Island sequences: detect grid loss, isolate, stabilize, resynchronize.

7. **PID / Ramp Tracking (Optional)**
   - For smooth setpoint tracking where inverter dynamics require it.

### 4.2 Battery State Estimation (Edge + Backend)

1. **SOC by Coulomb Counting**
   - Integrate current over time; drift corrected periodically.

2. **OCV Correction (Rest-State Recalibration)**
   - Only valid when current ~0 and battery at rest.

3. **Extended Kalman Filter (EKF) + Equivalent Circuit Model (ECM)**
   - Fuses current + voltage + temperature to reduce drift.

4. **SOH Estimation (Backend-first)**
   - Physics-informed ML (e.g., XGBoost) using:
     - cycle count (DoD-weighted)
     - calendar aging features
     - temperature stress
     - C-rate stress

### 4.3 Forecasting (Backend)

1. **Persistence Baseline**
   - Very short horizon forecast.

2. **Similarity Model (kNN)**
   - Finds similar historical days based on time/season/weather features.

3. **LSTM (24h Load/PV Forecast)**
   - Learns non-linear time dependencies.

4. **Ensemble**
   - Weighted combination of persistence + kNN + LSTM.

5. **Price Ingestion + Prediction**
   - Day-ahead market ingestion; predictive model optional.

### 4.4 Optimization (Backend + Edge Execution)

1. **Model Predictive Control (MPC)**
   - Horizon: 24h, step: 15 min.
   - Objective options:
     - minimize cost
     - reduce demand charges
     - reduce carbon
     - reduce battery degradation

2. **Real-Time Constraint Arbitration (Edge)**
   - Applies hard constraints and caps schedule tracking.
   - Optional linear solver:
     - resolves conflicting requests from controllers

---

## 5) Data Model (PostgreSQL)

### 5.1 Core Tables (Conceptual)

- `tenants`
- `users`, `roles`, `permissions`
- `org_nodes` (enterprise/site/building/floor/space/equipment)
- `devices` (type, vendor, protocol, config)
- `channels` (device points: unit, scaling, metadata)
- `measurements` (time-series; Timescale hypertable recommended)
- `events` (alarms, faults, operator actions)
- `tariffs`, `billing_runs`, `invoices`
- `carbon_factors`, `emissions`
- `commands` (backend → edge), `command_acks`
- `audit_log` (immutable)

### 5.2 Multi-Tenant Strategy

- Add `tenant_id` to every row that is tenant-owned.
- Use PostgreSQL RLS policies to enforce isolation.

---

## 6) Roadmap (Phases, Deliverables, Acceptance Criteria)

Time estimates are indicative. You can compress/expand based on team size.

### Phase 0 — Repo & Foundations (Weeks 1–2)

Deliverables:
- Repo structure (edge/backend/ui)
- Local dev stack (docker compose: postgres + backend + ui)
- CI pipeline (lint/test/build)

Acceptance:
- `backend` starts, connects to Postgres
- `ui` builds and loads
- health endpoints and basic logging exist

### Phase 1 — Edge MVP + Telemetry (Weeks 3–6)

Scope:
- Edge IPO cycle + Process Image + channel model
- One bridge: Modbus TCP
- One device mapping: Meter + ESS minimal
- Local buffer + offline-first queue
- Backend ingestion API + time-series storage

Acceptance:
- Stable cycle timing within configured bounds
- Telemetry stored in PostgreSQL and visible in UI
- Basic alarms for stale data

### Phase 2 — Safety + Core Controllers (Weeks 7–10)

Scope:
- Safety controllers: SOC limits, temp derating, voltage protection
- Peak shaving controller (hysteresis + ramp)
- Manual override path
- Fault state machine and recovery procedures

Acceptance:
- Safety limits always override optimization
- Faults latch and require explicit clear
- Setpoints never violate configured hard constraints

### Phase 3 — Enterprise Core (Weeks 11–16)

Scope:
- Multi-tenancy + RBAC
- Org hierarchy
- Device registry + provisioning workflow
- Reporting basics (daily/weekly energy summaries)

Acceptance:
- Tenant isolation verified
- Role restrictions enforced
- Reports reproducible with audit trails

### Phase 4 — Billing + Carbon (Weeks 17–22)

Scope:
- Tariff engine (TOU + demand charge baseline)
- Invoices/export
- Carbon factors + emissions reports

Acceptance:
- Billing results match test fixtures
- Carbon reports are traceable and consistent

### Phase 5 — Forecasting + MPC (Weeks 23–30)

Scope:
- Forecasting service (baseline + kNN + LSTM)
- MPC schedule generation (CVXPY)
- Edge schedule tracker controller

Acceptance:
- MPC produces feasible schedules under constraints
- Edge follows schedule but never violates safety
- Drift monitoring for forecasts

### Phase 6 — EVCS + Advanced Grid Services (Weeks 31–38)

Scope:
- OCPP integration
- Smart charging + surplus charging
- Droop + virtual inertia options

Acceptance:
- EVCS cluster stays within site power cap
- Grid services respond within configured time

### Phase 7 — Production Hardening (Weeks 39+)

Scope:
- Full observability (metrics, tracing)
- Security hardening (cert auth, secrets mgmt)
- Load tests + failure injection
- Upgrade/rollback strategy

Acceptance:
- SLOs defined and measured
- Documented operational runbooks
- Disaster recovery and backup tested

---

## 7) Testing & Validation Strategy

### Edge
- Unit tests for controllers
- Scenario replay tests (recorded telemetry → deterministic outputs)
- Hardware-in-the-loop (HIL) where possible

### Backend
- Contract tests for ingestion payloads
- Billing fixtures and golden tests
- Multi-tenant security tests (RLS)

### UI
- Component tests
- End-to-end flows for provisioning, dashboards, billing

---

## 8) “Add New Feature” Rules (Engineering Governance)

Every new feature must define:
- type (controller/bridge/backend/ml/ui)
- owner and acceptance criteria
- safety impact analysis
- test plan
- observability (metrics + logs)
- fallback behavior

---

## 9) Next Actions

If you want, I can generate:
- a matching folder scaffold (edge/backend/ui) aligned to this roadmap
- an initial API contract (OpenAPI) for ingestion + commands
- a minimal database schema (Alembic migrations)
