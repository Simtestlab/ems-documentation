
# ESS Abstraction – Standardized Battery Channels

## Purpose

The **ESS Abstraction** defines a **uniform set of channels** for representing any Energy Storage System (battery) in the EMS. Whether the physical battery communicates via Modbus, CAN, or is simulated, the rest of the system sees the same **standardized interface**.

This abstraction allows:

- **Controllers** (peak shaving, balancing) to work with any battery without modification.

- **Protocol adapters** to translate vendor‑specific registers into a common format.

- **Simulators** to mimic battery behavior during development.

- **Advanced estimators** (Kalman, ML) to read raw data and write back improved SOC.

Think of it as defining a **contract** between the battery and the rest of the EMS: *if you provide these channels, we can control you.*  

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **State of Charge (SOC)** | The remaining energy in the battery, expressed as a percentage (0–100%). 0% = empty, 100% = full. |
| **Power** | Instantaneous power flow into (charging) or out of (discharging) the battery. Positive typically means charging, negative discharging (but the sign convention must be defined). |
| **Limits** | Maximum allowed charge/discharge power (often dependent on SOC, temperature, and BMS limits). Also minimum/maximum SOC for operational safety (e.g., never discharge below 10% to protect battery life). |
| **Setpoints** | Commands to control the battery: desired power, current limits, operating mode (charge/discharge/idle). |
| **State of Health (SOH)** (optional) | Indicates battery degradation over time (e.g., 80% means capacity has reduced to 80% of original). |
| **Temperature** | Battery pack temperature – important for safety and derating. |
| **Voltage & Current** | Instantaneous measurements – used for power calculation and by advanced estimators. |

---

## Standardized ESS Channels

All ESS channels reside in the **Device Abstraction Layer** and follow the naming convention `battery_{instance_id}/{channel_name}`.

### Input Channels (Telemetry – from device to EMS)

| Channel Name | Unit | Description | Typical Source |
|--------------|------|-------------|----------------|
| `Soc` | % | State of Charge | BMS or estimator |
| `Power` | W | Instantaneous power (+ = charging, – = discharging) | BMS / inverter |
| `Voltage` | V | Battery pack voltage | BMS |
| `Current` | A | Battery current (+ = charging, – = discharging) | BMS |
| `Temperature` | °C | Average pack temperature (or individual cell temps) | BMS |
| `MaxChargePower` | W | Maximum allowed charge power at current conditions | BMS / inverter |
| `MaxDischargePower` | W | Maximum allowed discharge power | BMS / inverter |
| `MinSoc` | % | User‑configured minimum SOC (safety limit) | Configuration |
| `MaxSoc` | % | User‑configured maximum SOC (safety limit) | Configuration |
| `Soh` | % | State of Health | BMS / estimator |
| `Status` | – | Operational status (e.g., "idle", "charging", "fault") | BMS |
| `CycleCount` | – | Number of charge/discharge cycles | BMS |
| `Timestamp` | – | Time of last update (added by DAL) | DAL |

### Output Channels (Setpoints – from EMS to device)

| Channel Name | Unit | Description |
|--------------|------|-------------|
| `SetActivePower` | W | Desired power (+ = charge, – = discharge). The device should follow this as closely as possible, respecting limits. |
| `SetChargeCurrent` | A | Maximum charge current limit (alternative to power setpoint). |
| `SetDischargeCurrent` | A | Maximum discharge current limit. |
| `SetMode` | – | Command: `"charge"`, `"discharge"`, `"idle"`, `"standby"`. |
| `SetMinSoc` | % | Update the minimum SOC limit (if adjustable). |
| `SetMaxSoc` | % | Update the maximum SOC limit. |
| `SetChargeVoltage` | V | Target charge voltage (for some chemistries). |
| `Enable` | boolean | Master enable/disable for the battery (optional). |

---

## How It Fits into the DAL

The ESS Abstraction is **not a separate component** – it is a **set of channel definitions** that live inside the Device Abstraction Layer.

```
[Modbus Adapter] ── writes ──→  DAL battery channels  ←── reads ── [Controllers]
       ↑                               ↑                               ↑
[Real Battery]                     [Simulated Battery]           [Command Dispatcher]
```

- **Protocol adapters** read from physical batteries and write to DAL input channels (e.g., `battery_1/Soc`, `battery_1/Power`).
- **Simulated batteries** (during development) also write to DAL, but their values come from a mathematical model.
- **Controllers** read these input channels (via snapshot) and write to output channels (e.g., `battery_1/SetActivePower`).
- **Command Dispatcher** reads output channels and sends the commands to the appropriate adapter (which then writes to the physical device or simulator).

---
## Example: End‑to‑End Flow

**Scenario:** Peak shaving controller wants to discharge battery at 5 kW.

1. **Physical battery** (via Modbus) sends telemetry: SOC = 70%, MaxDischargePower = 10 kW.
2. **Modbus adapter** writes to DAL:
   - `battery_1/Soc = 70`
   - `battery_1/MaxDischargePower = 10000`
3. **Edge Core Cycle** takes a snapshot containing these values.
4. **Peak shaving controller** runs, reads snapshot, computes setpoint = -5000 W (discharge), writes `battery_1/SetActivePower = -5000`.
5. **Ramp limiter** (within cycle) checks last commanded value and applies ramp rate if needed.
6. **Command Dispatcher** reads `battery_1/SetActivePower` and sends it to Modbus adapter.
7. **Modbus adapter** writes the setpoint to the physical battery.
8. Battery responds, new telemetry flows in next cycle.

---

## Important Notes

- **Sign convention:** We use **positive = charging, negative = discharging** for power and current. This must be consistent across all adapters and controllers.
- **Limits:** Always respect `MaxChargePower` and `MaxDischargePower`. A controller should never exceed these.
- **MinSoc/MaxSoc:** These are safety limits. If SOC reaches MinSoc, discharging must stop (setpoint forced to 0 or positive). Similarly at MaxSoc, charging stops.
- **Timestamping:** Every channel update includes a timestamp. This is crucial for detecting stale data and for temporal coherence in controllers.
- **Quality flags:** In production, you may add a `quality` field to indicate if a value is `GOOD`, `STALE`, or `ERROR`.

---

## Summary

The ESS Abstraction provides a **single, consistent interface** for all battery‑related data and control. By defining a fixed set of channel names, units, and semantics, it enables:

- **Interchangeability** – swap one battery brand for another without changing controllers.
- **Testability** – simulate batteries during development.
- **Extensibility** – add new battery types by writing new protocol adapters, leaving the rest of the system untouched.