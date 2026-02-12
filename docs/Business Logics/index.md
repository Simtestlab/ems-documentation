# EMS Core Features

## Core Infrastructure & Device Connectivity
1. **[Modbus Protocol Adapter](./protocol-adapter.md)** - Read/write registers via Modbus TCP/RTU; device-profile based.

2. **[Edge Core Cycle](./edge-core-cycle.md)** - Deterministic 1s (configurable) execution loop (scheduler).

3. **Edge Configuration Sync** - Edge pulls latest device mappings and controller parameters from cloud.

4. **Command Dispatch** - Bidirectional WebSocket: send setpoints from cloud/dashboard to edge.

5. **CAN Bus Support** - Driver for BMS communication

6. **SunSpec Compilance** - Auto-discovery and parsing of SunSpec-compilant inverters.

## Device Abstraction & Virtualisation

1. **ESS Abstraction** - Standardised battery channels: SOC, power, limits, setpoints.

2. **Grid/Site Meter** - Unified import/export meter;

3. **PV Inverter Abstraction** - Active power, energy, voltage and current.

4. **Load Meter Abstraction** - Consumption meter, enables net/gross analysis.

5. **Advanced SOC Estimation** - Kalman filter (EKF) with voltage correction and improved accuracy.

6. **Machine Learning SOC** - LSTM/GRU model trained on site-specific BMS data; adaptive to ageing.

7. **Reactive Power Capability** - Inverter channels for Q setpoint, power factor, Volt-VAR curves.

8. **Virtual Meter** - Computer channels (sum of PV, wind, net load)

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