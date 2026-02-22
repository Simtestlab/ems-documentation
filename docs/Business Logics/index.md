# EMS Core Features

## EMS MVP Features

After completing the six phases, your Minimum Viable Product (MVP) will include a fully functional, production‑ready **Device Abstraction Layer (DAL)** and **Command Dispatcher**, running on a simulated edge (Raspberry Pi) with realistic battery, grid, PV, and load simulators. The system will be secure, observable, and capable of handling manual commands with full reliability. All core components are built and integrated; only hardware protocols, advanced controllers (balancing, peak shaving), and multi‑site management are deferred to later phases.

Below is the complete list of features delivered in the MVP.

---

## 1. Device Abstraction Layer (DAL)

- **Core Channel Store** – Thread‑safe in‑memory dictionary with `set_channel`, `get_channel`, `snapshot()`.
- **Channel Categories:**
  - **Input channels** – telemetry from devices (e.g., `battery_1/Soc`, `grid/ActivePower`).
  - **Output channels** – setpoints to devices (e.g., `battery_1/SetActivePower`).
  - **Request channels** – user commands from UI (`ui_requests/battery_1_power`).
  - **Acknowledgment channels** – final execution status (`ack/battery_1_power`).
- **Naming Convention** – consistent hierarchical naming (`device_type_instance/channel`).
- **Timestamping** – every channel update carries a timestamp.
- **Atomic Snapshot** – consistent view of all input channels at cycle start.
- **Thread Safety** – supports concurrent producers/consumers.

---

## 2. Smart Simulators (Edge)

- **Battery Simulator** – stateful model with SOC, capacity, power limits, efficiency. Responds to `SetActivePower`, updates telemetry, respects Min/Max SOC.
- **Grid Meter Simulator** – generates realistic import/export patterns (configurable).
- **PV Inverter Simulator** – generates solar production based on time of day.
- **Load Meter Simulator** – generates consumption patterns (optional, can be derived).
- **Simulator Framework** – all simulators run as separate threads, updating DAL every second.

---

## 3. Edge Core Cycle & Arbitration

- **Deterministic 1‑Second Loop** – async loop with absolute deadline scheduling.
- **Snapshot** – captures DAL snapshot at cycle start.
- **Scheduler** – loads controllers from config, runs them in priority order.
- **Manual Override Controller** – reads request channels, revalidates, writes to output channels, clears request, writes final status to ack channel.
- **Last‑Write‑Wins Arbitration** – if multiple controllers write same output, last one prevails.
- **Telemetry Uplink** – sends full DAL snapshot to cloud every cycle via WebSocket; cloud stores in TimescaleDB.

---

## 4. Command Dispatcher – Basic (Phase 4)

- **Cloud‑Side Dispatcher (Basic)**
  - WebSocket server for UI connections (query‑string or first‑message auth).
  - Static validation against device profiles (static limits, enums).
  - Routing to online edge (simple connection registry).
  - Immediate acknowledgments (`received`/`rejected`).
- **Edge‑Side Dispatcher (Basic)**
  - WebSocket client with first‑message API key auth.
  - Lightweight validation (device existence, request channel mapping).
  - Writes commands to request channels in DAL.
  - Sends immediate `received` ack to cloud.
- **Device Profiles (Static)**
  - JSON definitions with static limits, enums, channel names.
  - Fetched by edge at startup (simple HTTP).
- **Command Payload Schema** – standard JSON format for all commands.
- **UI Hookup** – existing Next.js dashboard connects to cloud WebSocket, sends commands, displays immediate acks.

---

## 5. Command Dispatcher – Advanced (Phase 5)

- **Dynamic Limit Validation**
  - Cloud queries TimescaleDB for latest telemetry (e.g., `MaxChargePower`).
  - Staleness check (e.g., 30s) – rejects if telemetry is stale.
  - Computes effective limits and validates command.
- **Acknowledgment Channels** – controllers write final status to `ack/` channels.
- **Final Acknowledgment Forwarding** – edge dispatcher monitors `ack/` channels and forwards final status to cloud.
- **In‑Flight Watchdog**
  - Redis sorted set tracks commands sent to online edges.
  - Background worker sends `failed: edge_timeout` if no final ack within 5s.
- **Offline Command Queue**
  - Redis LIST per edge (FIFO) stores commands when edge offline.
  - Per‑command TTL enforced at dequeue.
  - Mandatory background expiry worker sends `failed: timeout_in_queue` when commands expire while offline.
- **Queue Overflow Protection** – rejects commands when queue length exceeds limit (e.g., 10,000).

---

## 6. Security & Audit Logging (Phase 6)

- **Authentication**
  - User JWT (Django‑issued) validated on connection and per command.
  - Edge API keys (hashed) authenticated via first‑message frame.
  - Session revocation via Redis pub/sub (Django publishes user ID, cloud kills WebSockets).
- **Rate Limiting**
  - Redis token buckets: 10 commands/sec per user, 100 acks/sec per edge.
  - IP‑based limits for HTTP endpoints.
- **Replay Protection**
  - Redis cache of `command_id`s with 60s TTL.
  - Timestamp freshness check (≤ 60s).
- **Input Sanitization** – Pydantic models with strict schema (extra fields rejected).
- **CORS** – restricted to UI domain.
- **Audit Logging**
  - Structured JSON logs at all waypoints (cloud, queue, edge) with `command_id` correlation.
  - Edge logs use rotating files, shipped via Filebeat to central aggregator (ELK/Datadog).
  - NTP requirement for edges; clock drift warnings logged.

---

## 7. UI Dashboard (Next.js)

- **Live Single‑Line Diagram** – WebSocket connection to cloud telemetry stream.
- **Command Forms** – send setpoints (battery power), mode changes, view immediate and final status.
- **Queue Notifications** – shows `queued`, `timeout_in_queue`, `edge_timeout`, and final execution status.
- **Telemetry Display** – real‑time charts for grid, PV, battery SOC, load.
- **Login Page** – JWT authentication integrated with Django.

---

## 8. Cloud Infrastructure

- **FastAPI WebSocket Servers** – for UI commands and edge telemetry.
- **TimescaleDB** – stores all telemetry (used for dynamic validation and history).
- **Redis** – used for offline queues, in‑flight watchdog, rate limiting, replay cache, session revocation.
- **Django Auth Service** – issues JWT, manages users, publishes revocation events.
- **Log Aggregator** – ELK or Datadog stack for audit logs.

---

## What Is NOT in the MVP (Future Phases)

- Real hardware protocol adapters (Modbus, CAN, SunSpec) – these will replace simulators later.
- Multiple controllers (balancing, peak shaving) – the controller team will integrate them after MVP using the established interfaces.
- Multi‑tenancy (organisation/site hierarchy) – Phase 7 optional.
- Edge configuration sync (cloud‑managed config) – Phase 7 optional.
- Advanced SOC estimation (Kalman, ML) – future.
- Reactive power control (Volt‑VAR) – future.
- Predictive controllers (ToU, forecasting) – future.
- Real‑time alarms (threshold alarms, email notifications) – future.
- Edge heartbeat monitoring and Raft‑based HA – future.

---

## Summary

The MVP delivered after Phase 6 is a complete, secure, and observable EMS edge and cloud system with simulated devices, manual control, and a polished UI. It includes every feature documented for the Device Abstraction Layer and Command Dispatcher (static/dynamic validation, offline queues, acknowledgments, watchdog, audit logging, rate limiting). This foundation is ready to integrate real hardware protocols and advanced control logic in subsequent phases.

## Core Infrastructure & Device Connectivity

1. **[Modbus Protocol Adapter](./Core/modbus-protocol-adapter.md)** - Read/write registers via Modbus TCP/RTU; device-profile based.

2. **[Edge Core Cycle](./Core/edge-core-cycle.md)** - Deterministic 1s (configurable) execution loop (scheduler).

3. **[Edge Configuration Sync](./Core/edge-configuration-sync.md)** - Edge pulls latest device mappings and controller parameters from cloud.

4. **[Command Dispatch](./Core/command_dispatch.md)** - Bidirectional WebSocket: send setpoints from cloud/dashboard to edge.

5. **[CAN Bus Support](./Core/can-bus-support.md)** - Driver for BMS communication

6. **[SunSpec Compilance](./Core/sunspec-compliance.md)** - Auto-discovery and parsing of SunSpec-compilant inverters.

## Device Abstraction Layer & Virtualization

1. **Definition of [Device Abstraction Layer](./Abstraction/index.md)**

2. **[ESS Abstraction](../Business%20Logics/Abstraction/ess_abstraction.md)** - Standardised battery channels: SOC, power, limits, setpoints.

3. **[Grid/Site Meter](../Business%20Logics/Abstraction/grid_meter.md)** - Unified import/export meter;

4. **[PV Inverter Abstraction](../Business%20Logics/Abstraction/pv_inverter.md)** - Active power, energy, voltage and current.

5. **[Load Meter Abstraction](../Business%20Logics/Abstraction/load-meter.md)** - Consumption meter, enables net/gross analysis.

6. **[Virtual Meter](./Abstraction/virtual-meter.md)** - Computer channels (sum of PV, wind, net load)

#### Planned features for next phase

7. **Advanced SOC Estimation** - Kalman filter (EKF) with voltage correction and improved accuracy.

8. **Machine Learning SOC** - LSTM/GRU model trained on site-specific BMS data; adaptive to ageing.

9. **Reactive Power Capability** - Inverter channels for Q setpoint, power factor, Volt-VAR curves.

## Command Dispatcher

1. **[Command Dispatcher](../Business%20Logics/Command%20Dispatcher/index.md)** - Responsible for routing **user-initiated commands** from the cloud UI to the appropriate edge device.

2. **[Cloud-Site Dispatcher Service](../Business%20Logics/Command%20Dispatcher/cloud-side-dispatcher.md)**

3. **[Edge-Site Dispatcher Service](../Business%20Logics/Command%20Dispatcher/edge-side-dispatcher.md)**

4. **[Request Channel](../Business%20Logics/Abstraction/ui_request_channel.md)**

5. **[Command Validation](../Business%20Logics/Command%20Dispatcher/command-validation.md)**

6. **[Offline Command Queue and TTL](../Business%20Logics//Command%20Dispatcher/offline-command-queue.md)**

7. **[Acknowledgement Flow](../Business%20Logics/Command%20Dispatcher/acknowledgement-flow.md)**

8. **[Security and Rate Limiting](../Business%20Logics/Command%20Dispatcher/security-and-rate-limit.md)**

9. **[Controller Consumption & Arbitration](../Business%20Logics/Command%20Dispatcher/controller_consumption.md)**

10. **[Command Payload Schema](../Business%20Logics/Command%20Dispatcher/command-payload.md)**

11. **[Audit logging & Command Traceability](../Business%20Logics/Command%20Dispatcher/audit-logging.md)**

## Control & Optimisation

### Basic Controllers

1. **Self-consumption (Balancing)** - Zero grid target; charge from excess PV/wind, discharge to meet load.

2. **Peak Shaving** - Discharge battery when grid import exceeds threshold; reduce deman charges.

3. **Manual Override** - Operator can set fixed charge/discharge power via dashboard.

4. **Load Following** - Maintain predefined grid import/export profile; follow utility signal.

5. **Fixed Power Factor** - Maintain constant PF at grid connection point.

6. **Volt-VAR Control** - Autonomous reactive power response to voltage deviations.

7. **Frequency-Watt** - Active power curtailment during over-frequency events.

## Predictive Controllers

1. **Static Time-of-Use (ToU)** - Schedule, charge during off-peak, discharge during on-peak.

2. **Load & PV Forecasting** - Persistence model; 24h horizon, 15min resolution.

3. **ML-Based Forecasting** - XGBoost/LightGBM trained on site data + weather API;

4. **Optimal ToU Dispatch** - Linear programming (LP) scheduler; minimises energy cost; respects battery degradation.

5. **Dynamic Peak shaving** - Forecast-aware threshold adjustment; avoid peaks proactively.

6. **Demand Response** - Accept external curtailment signals (OpenADR 2.0b).

## Monitoring & Visualization

1. **Live Single-Line Diagram** - Real-time power flow; grid, solar, wind, battery, loads; updates via WebSocket.

2. **Multi-Site Overview** - Cards with key metrics (total solar, battery SOC, grid import, alarms)

3. **Device Detail Page** - Real-time channel values + mini-trends (last hour).

4. **Historical Energy Charts** - Consumed, generated, stored energy over user-selected range.

5. **Alarm list & Timeline** - Active/acknowledged/cleared alarms with severity and timestamp.

5. **Energy mix Pie Chart** - Share of solar, wind, grid, battery in supply.

## Analytics & Reporting

1. **Daily/Weekly/Monthly Energy Report** - PDF/CSV; consumption, generation, battery throughput, grid exchange.

2. **Peak Demand Report** - Top 5 peak 15-min intervals per billing period.

3. **Estimated Savings** - Compare actual grid import with baseline (flat threshold).

4. **Carbon Avoidance** - Co2 emissions avoided (based on grid mix factors)

5. **Power Quality Report** - Voltage deviations, frequency, harmonics.

6. **Battery Degradation Report** - Throughput vs SOH estimate; remaining useful life.

7. **Anomaly Detection** - Unsupervised ML to identify unsual consumption/generation patterns.

8. **Automated Email reports** - Schedules delivery to stakeholders.

## Management & Administration

1. **Multi-Tenant Hierarchy** - Organisation -> Site -> Edge -> Device; strict data isolation.

2. **Role-based access control** - Admin, Operator, Viewer roles per organisation.

3. **Audit Log** - Track user actions: configuration changes, manual commands.

4. **Bulk Device Provisioning** - CSV import; auto-assignment to sites.

## Alarms & Events

1. **Threshold Alarm Rules** - User-definable high/low limits on any numeric channel.

2. **Communication Loss Alarm** - Edge or device disconnect detected; auto-raised.

3. **Alarm Acknowledgement** - User can acknowledge; status reflected in UI.

4. **Email Notifications** - Send alarm emails to configured receipients.

5. **Alarm Escalation** - If not acknowledged, notify higher-level user.

6. **Predictive Alarms** - ML-based early warning (e.g battery may fail in 7 days.)

## Reliability & Scalability

1. **Edge Heartbeat Monitoring** - Cloud tracks last seen timestamp; alerts on stale.

2. **TimescaleDB Compression** - Reduce Storage costs for historical data.

3. **Edge Local Storage** - Cache telemetry locally if cloud unreachable; backfill on reconnect.

4. **Raft-Based Edge HA** - Active-passive cluster with automatic failover (no control gap).