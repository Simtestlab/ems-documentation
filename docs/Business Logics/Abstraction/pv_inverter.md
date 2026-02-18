# PV Inverter Abstraction

## Purpose

The **PV Inverter Abstraction** defines a uniform set of channels for representing any photovoltaic (solar) inverter connected to the EMS. Whether the inverter communicates via Modbus (e.g., SMA, Fronius), SunSpec, or is simulated, the rest of the system sees the same **standardized interface** for solar generation data.

This abstraction enables:

- **Controllers** (self‑consumption, peak shaving) to use solar power in their decisions.

- **Protocol adapters** to translate vendor‑specific registers into a common format.

- **Simulators** to generate realistic PV profiles during development.

- **Monitoring and reporting** to display energy production, power, and performance.

The PV inverter is typically a **read‑only** device for basic monitoring, but may support **curtailment** (limiting output) for advanced use cases like demand response or frequency support.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **PV Inverter** | A device that converts DC power from solar panels into AC power synchronized with the grid. |
| **Active Power** | Real power (watts) exported to the grid/site. Positive typically means generation. |
| **Energy Total** | Accumulated energy produced over time (kWh or MWh). Used for reporting and performance tracking. |
| **AC Voltage & Current** | Output voltage and current (per phase or aggregate). Important for monitoring power quality. |
| **Frequency** | Grid frequency as measured by the inverter. |
| **Status** | Operational state (e.g., "producing", "idle", "fault", "curtailed"). |
| **Curtailment** | The ability to limit the inverter's output power (e.g., for zero export or grid support). Not all inverters support this. |
| **Power Factor** | Ratio of active to apparent power – can be controlled in advanced inverters. |

---

## Standardized PV Inverter Channels

All PV inverter channels reside in the **Device Abstraction Layer** and follow the naming convention `pv_{instance_id}/{channel_name}`. For example: `pv_1/ActivePower`, `pv_2/EnergyTotal`.

### Input Channels (Telemetry – read‑only)

| Channel Name | Unit | Description |
|--------------|------|-------------|
| `ActivePower` | W | Instantaneous AC active power output. **Positive = generation.** |
| `EnergyTotal` | kWh | Total accumulated energy produced (lifetime or since last reset). Often a 64‑bit value.
| `Voltage` | V | AC voltage (average or aggregate). For three‑phase, this could be average. |
| `Current` | A | AC current (average or total).
| `Frequency` | Hz | Grid frequency as measured by the inverter.
| `Status` | – | Operational status: `"producing"`, `"idle"`, `"fault"`, `"curtailed"`, etc. |
| `Timestamp` | – | Time of last update (added by DAL).
---

## How It Fits into the DAL

PV inverter channels are just another set of channels in the Device Abstraction Layer.

```
[Modbus Adapter] ── writes ──→  DAL pv channels  ←── reads ── [Controllers]
       ↑                               ↑
[Physical Inverter]              [Simulated PV]
```

- **Protocol adapters** (Modbus, SunSpec) read from physical inverters and write to DAL pv channels.
- **Simulated PV** (during development) generates synthetic data (e.g., from a weather‑based model) and writes to DAL.
- **Controllers** (e.g., self‑consumption) read these channels (via snapshot) to decide whether to charge/discharge the battery.
- **Cloud telemetry** and **alarm engine** also consume pv channels for monitoring and alerts.

---

## Sign Convention

- **Active Power:** **Positive = generation** (power flowing from inverter to the grid/site). This matches intuition and is consistent with most inverters.
- **Energy Total:** Always positive, cumulative.
- **Voltage and Current:** Always positive magnitudes (AC RMS).
- **Power Factor (future):** We'll define a convention when implementing (e.g., positive = lagging, negative = leading, with respect to generation). But for MVP, not needed.

All protocol adapters must convert their vendor‑specific signs to this standard.

---

## Example: End‑to‑End Flow

**Scenario:** A PV inverter is producing 5 kW. The self‑consumption controller wants to charge the battery.

1. **Physical inverter** (via Modbus) reports:
   - Active power = +5000 W
   - Energy total = 12345.6 kWh
   - Voltage = 230 V
   - Current = 21.7 A
   - Frequency = 50.02 Hz
   - Status = "producing"
2. **Modbus adapter** writes to DAL:
   - `pv_1/ActivePower = 5000`
   - `pv_1/EnergyTotal = 12345.6`
   - `pv_1/Voltage = 230`
   - `pv_1/Current = 21.7`
   - `pv_1/Frequency = 50.02`
   - `pv_1/Status = "producing"`
3. **Edge Core Cycle** snapshot captures these.
4. **Self‑consumption controller** runs, sees excess solar, computes battery charge setpoint = 2000 W (battery limit), writes `battery_1/SetActivePower = 2000`.
5. Battery charges.
6. Next cycle, the inverter's power may change, and the process repeats.

---

## Important Notes

- **Timestamping:** Every PV channel update must include a timestamp. This is crucial for detecting stale data and for aligning with other measurements (e.g., comparing PV power with load at the same instant).
- **Quality flags:** In production, you may add a `quality` field to indicate if the inverter data is `GOOD`, `STALE`, or `ERROR` (e.g., due to communication loss).
- **Per‑phase values:** For three‑phase inverters, per‑phase channels (ActivePower_L1, Voltage_L1, etc.) allow finer monitoring and control (e.g., phase balancing). These are recommended for future expansion.
- **EnergyTotal rollover:** Some meters have a 32‑bit counter that rolls over. The adapter should handle this by detecting rollovers and reporting a monotonically increasing value if possible.
- **Curtailment support:** Not all inverters support remote power limiting. The device profile should indicate whether `SetCurtailment` is available. The controller should check before sending commands.

---

## Summary

The PV Inverter Abstraction provides a **single, consistent interface** for solar generation data. By defining a fixed set of channel names and a clear sign convention (positive = generation), it enables:

- **Interoperability** – any inverter with a Modbus or SunSpec profile can be integrated.
- **Testability** – simulate PV output during development.
- **Unified control** – controllers can rely on `pv_1/ActivePower` regardless of the actual hardware.