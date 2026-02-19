
# Load Meter Abstraction

## Purpose

The **Load Meter Abstraction** defines a uniform set of channels for representing the electrical consumption (load) of a site. Whether the load is measured by a dedicated physical meter (e.g., a submeter on a manufacturing line) or calculated as a virtual meter from other measurements, the rest of the EMS sees a **standardized interface** for consumption data.

This abstraction enables:

- **Controllers** (self‑consumption, peak shaving) to understand how much power is being used on site.

- **Protocol adapters** to translate vendor‑specific registers into a common format.

- **Simulators** to generate realistic load profiles during development.

- **Net/gross analysis** by comparing load with generation (PV) and grid exchange.

The load meter is typically a **read‑only** device – it measures consumption but does not accept commands. (Some advanced meters may support remote disconnect, but that is out of scope for the basic abstraction.)

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Load** | Electrical power consumed by on‑site equipment (lights, motors, HVAC, etc.). |
| **Consumption** | Energy used over time (kWh). |
| **Net vs. Gross Analysis** | **Gross load** is the total consumption measured at the load meter. **Net load** is the power seen at the grid connection after accounting for on‑site generation and storage. Comparing load with generation enables optimization. |
| **Demand** | The average power over a specified interval (e.g., 15‑minute demand). Used for demand charges. |
| **Peak Load** | The maximum demand over a billing period – a key metric for demand charge reduction. |
| **Per‑Phase Load** | In three‑phase systems, loads may be unbalanced. Per‑phase measurement helps identify imbalance issues. |

---

## Standardized Load Meter Channels

All load meter channels reside in the **Device Abstraction Layer** and follow the naming convention `load_{instance_id}/{channel_name}`. 

For example: `load_main/ActivePower`, `load_line1/EnergyTotal`. If only one load meter exists, simply `load/ActivePower` is used.

### Input Channels (Telemetry – read‑only)

| Channel Name | Unit | Description | MVP / Future |
|--------------|------|-------------|--------------|
| `ActivePower` | W | Instantaneous total active power consumption. **Positive = power flowing into loads (consumption).**
| `EnergyTotal` | kWh | Accumulated total energy consumed (lifetime or since last reset).
| `EnergyImport` | kWh | Some meters may separate import (consumption) – same as `EnergyTotal`.
| **`Type`** | – | Indicates origin of data: `"PHYSICAL"` (real meter), `"VIRTUAL"` (computed), or `"ESTIMATED"` (e.g., from simulation).
| `Timestamp` | – | Time of last update (added by DAL).

**Important:** The load meter is assumed to measure **consumption only**. If a meter is bidirectional (e.g., a grid meter used as load meter in some configurations), the sign convention must be aligned: positive = consumption, negative = generation (export) – but such a scenario is better handled by a grid meter abstraction. Therefore, we recommend that a load meter channel **never be negative**. Protocol adapters should clamp or report an error if negative values appear.

---

## How It Fits into the DAL

Load meter channels are just another set of channels in the Device Abstraction Layer.

```
[Modbus Adapter] ── writes ──→  DAL load channels  ←── reads ── [Controllers]
       ↑                               ↑
[Physical Meter]                 [Simulated Load]
```

- **Protocol adapters** read from physical load meters (e.g., a submeter on a production line) and write to DAL load channels.
- **Simulated load** (during development) generates synthetic consumption data and writes to DAL.
- **Controllers** (e.g., self‑consumption) read these channels (via snapshot) to understand site demand and make decisions.
- **Cloud telemetry** and **alarm engine** also consume load channels for monitoring and alerts.

---

## Sign Convention

- **Active Power:** **Positive = consumption** (power flowing from the electrical panel into loads). Negative values are not expected from a dedicated load meter. If a meter reports negative (e.g., due to reverse power flow from generation behind the meter), it should be treated as an error or clamped to zero, as this likely indicates misconfiguration or a bidirectional meter better suited as a grid meter.
- **Energy Total:** Always positive, cumulative.
- **Reactive Power (future):** Positive = inductive load (typical for motors), negative = capacitive load (uncommon). But this can be defined when implementing.

All protocol adapters must convert vendor‑specific signs to this standard.

---

## Virtual Load Meter (No Physical Meter)

If a site does not have a dedicated load meter, you can create a **virtual load meter** that computes load from other measurements.

The correct formula, given the established sign conventions (grid import = positive, PV generation = positive, battery charging = positive), is:

```
load/ActivePower = grid/ActivePower + pv/ActivePower - battery/ActivePower
```

**Why this formula works:**

| Scenario | Grid | PV | Battery | Formula | Correct Load |
|----------|------|----|---------|---------|--------------|
| Night, import to charge battery | +5 | 0 | +5 | 5 + 0 - 5 = 0 | 0 |
| Day, PV covers load, export | -5 | +10 | 0 | -5 + 10 - 0 = 5 | 5 |
| Day, PV excess charges battery | -3 | +10 | +7 | -3 + 10 - 7 = 0 | 0 |
| Night, import supplies load | +8 | 0 | 0 | 8 + 0 - 0 = 8 | 8 |

A virtual meter service (which could be a controller or a separate service) would:

1. Read the relevant channels from DAL (`pv/ActivePower`, `battery/ActivePower`, `grid/ActivePower`). Note that `battery/ActivePower` is the measured power (positive = charging), not the setpoint.
2. Compute load using the formula above.
3. Write the result to `load/ActivePower` and, by integrating over time, to `load/EnergyTotal`.
4. Optionally set the channel `Type` to `"VIRTUAL"` so that UI and other components know the data is derived.

This derived load channel then becomes available to all other components just like a physical meter. This is a powerful way to enable net/gross analysis even without direct load measurement.

---

## Example: End‑to‑End Flow

**Scenario:** A site has a load meter showing 8 kW consumption. PV is generating 5 kW, battery is idle.

1. **Physical load meter** (via Modbus) reports:
   - Active power = +8000 W
   - Energy total = 123456 kWh
2. **Modbus adapter** writes to DAL:
   - `load/ActivePower = 8000`
   - `load/EnergyTotal = 123456`
   - `load/Type = "PHYSICAL"`
3. **Edge Core Cycle** snapshot captures these.
4. **Self‑consumption controller** runs:
   - load = 8000, pv = 5000, net = 3000 (need from grid)
   - battery has SOC 80%, so controller decides to discharge to cover part of the load.
   - It computes discharge = 2000 W (respecting limits) and writes `battery_1/SetActivePower = -2000`.
5. Battery discharges, reducing grid import.
6. Next cycle, load may change, and the process repeats.

If no physical load meter existed, a virtual meter would compute `load/ActivePower` from the other channels and set `Type = "VIRTUAL"`.

---

## Important Notes

- **Timestamping:** Every load channel update must include a timestamp. This is crucial for aligning load with generation and grid data.
- **Quality flags:** In production, you may add a `quality` field to indicate if the meter data is `GOOD`, `STALE`, or `ERROR` (e.g., due to communication loss).
- **Per‑phase values:** For industrial sites with unbalanced loads, per‑phase power channels (`ActivePower_L1`, etc.) are valuable. Include them in future extensions.
- **EnergyTotal rollover:** Some meters have a 32‑bit counter that rolls over. The adapter should handle this by detecting rollovers and reporting a monotonically increasing value if possible.
- **Type channel:** Adding a `Type` field (PHYSICAL / VIRTUAL / ESTIMATED) helps the UI display appropriate icons and informs operators about data provenance.
- **Load shedding:** If the EMS supports load control (e.g., via relays or smart breakers), those commands would be handled by separate device abstractions (e.g., a relay board). The load meter itself is only for measurement.

---

## Summary

The Load Meter Abstraction provides a **single, consistent interface** for site consumption data. By defining a fixed set of channel names and a clear sign convention (positive = consumption), it enables:

- **Interoperability** – any meter that measures load can be integrated.
- **Testability** – simulate load profiles during development, with correct energy integration.
- **Unified control** – controllers can rely on `load/ActivePower` to understand demand.
- **Net/gross analysis** – compare load with generation and grid to optimize self‑consumption.