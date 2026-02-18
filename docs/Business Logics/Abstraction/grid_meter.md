# Grid/Site Meter

## Purpose

The **Grid/Site Meter Abstraction** defines a uniform set of channels for representing the electrical connection between the site and the utility grid. Whether the meter is a physical device (e.g., SDM630 over Modbus) or a virtual aggregation of other measurements, the rest of the EMS sees a **standardized interface** for grid data.

This abstraction enables:

- **Controllers** (peak shaving, balancing) to make decisions based on grid import/export power.

- **Protocol adapters** to translate vendor‑specific registers into a common format.

- **Simulators** to generate realistic grid data during development.

- **Virtual meters** to synthesize grid data when a physical meter is absent (e.g., by summing generation and load).

The grid meter is typically a **read‑only** device – the EMS monitors it but does not send commands to it. (Some advanced meters may support configuration, but that is out of scope for the basic abstraction.)

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Grid Meter** | A device that measures power flow at the point of common coupling (where the site connects to the utility grid). |
| **Import / Export** | **Import** = power flowing from the grid into the site (buying electricity). **Export** = power flowing from the site back to the grid (selling or feeding in). The sign convention must be consistent. |
| **Active Power** | Real power (watts) that does useful work. |
| **Reactive Power** | Imaginary power (VAR) that sustains electric and magnetic fields. Important for power factor correction. |
| **Apparent Power** | Vector sum of active and reactive power (VA). |
| **Power Factor** | Ratio of active power to apparent power (cos φ). Indicates how efficiently power is used. |
| **Phases** | Single‑phase or three‑phase systems. Three‑phase meters provide per‑phase values (voltage, current, power). |
| **Energy** | Accumulated active energy (kWh) imported and exported over time. Used for billing and reporting. |
| **Frequency** | Grid frequency (Hz). |
| **Rate of Change of Frequency (RoCoF)** | Derivative of frequency (Hz/s). Critical for detecting islanding and grid disturbances. |

---

## Standardized Grid Meter Channels

All grid meter channels reside in the **Device Abstraction Layer** and follow the naming convention `grid/{channel_name}`. (If multiple grid meters exist, e.g., main and sub‑meter, use `grid_main/...` and `grid_sub/...`.)

### Input Channels (Telemetry – read‑only)

| Channel Name | Unit | Description |
|--------------|------|-------------|
| `ActivePower` | W | Instantaneous **net** active power (sum of all phases). **Positive = import, negative = export.** |
| `ActivePower_L1` | W | Instantaneous active power on phase L1. |
| `ActivePower_L2` | W | Instantaneous active power on phase L2 (if three‑phase). |
| `ActivePower_L3` | W | Instantaneous active power on phase L3 (if three‑phase). |
| `ReactivePower` | VAR | Instantaneous **net** reactive power. Sign follows IEEE 1459 load convention: **positive = inductive (lagging), negative = capacitive (leading).** |
| `ReactivePower_L1` | VAR | Instantaneous reactive power on phase L1. |
| `ReactivePower_L2` | VAR | Instantaneous reactive power on phase L2. |
| `ReactivePower_L3` | VAR | Instantaneous reactive power on phase L3. |
| `ApparentPower` | VA | Instantaneous net apparent power. |
| `PowerFactor` | – | Net power factor (cos φ). Typically positive for import, negative for export, or absolute value. The sign convention should align with reactive power definition. |
| `Frequency` | Hz | Grid frequency. |
| `Frequency_RoCoF` | Hz/s | Rate of change of frequency (optional, advanced). Useful for islanding detection and fast grid services. |
| `Voltage_L1` | V | Phase 1 line‑to‑neutral voltage (or line‑to‑line depending on system). |
| `Voltage_L2` | V | Phase 2 voltage (if three‑phase). |
| `Voltage_L3` | V | Phase 3 voltage (if three‑phase). |
| `Current_L1` | A | Phase 1 current. |
| `Current_L2` | A | Phase 2 current. |
| `Current_L3` | A | Phase 3 current. |
| `EnergyImport` | kWh | Accumulated imported active energy since last reset (or device installation). |
| `EnergyExport` | kWh | Accumulated exported active energy. |
| `EnergyNet` | kWh | Net energy (import - export) – often computed, not directly metered. |
| `ReactiveEnergyImport` | kVARh | Accumulated imported reactive energy (optional). |
| `ReactiveEnergyExport` | kVARh | Accumulated exported reactive energy (optional). |
| `Timestamp` | – | Time of last update (added by DAL). |

**Why per‑phase power matters:**

- **Phase imbalance** – total power may be within limits, but a single phase could be overloaded, tripping breakers.

- **Per‑phase zero export** – some grid codes require zero export on each phase individually, not just the sum.

### Output Channels (Commands)

Grid meters are typically **read‑only** and do not accept commands. However, some meters may support:
- Resetting accumulated energy registers (requires password/protected access).
- Configuring demand period, etc.

If such features are needed, they can be exposed via additional output channels (e.g., `grid/ResetEnergy`), but this is beyond the basic abstraction and should be handled by device‑specific profiles and the command dispatcher.

---

## How It Fits into the DAL

The grid meter channels are just another set of channels in the Device Abstraction Layer.

```
[Modbus Adapter] ── writes ──→  DAL grid channels  ←── reads ── [Controllers]
       ↑                               ↑
[Physical Meter]                 [Simulated Meter]
```

- **Protocol adapters** read from physical meters (e.g., SDM630) and write to DAL grid channels.
- **Simulated meters** (during development) generate synthetic grid data and write to DAL.
- **Controllers** read these channels (via snapshot) to make decisions (e.g., peak shaving: if `grid/ActivePower` > threshold, discharge battery).
- **Cloud telemetry** and **alarm engine** also consume grid channels.

---

## Sign Convention (Critical)

**Active Power:**
- **Positive = import** (power flowing from grid into site)
- **Negative = export** (power flowing from site to grid)

This convention must be strictly enforced by all protocol adapters. Any meter that reports the opposite must be inverted in the adapter.

**Reactive Power (IEEE 1459 Load Convention):**
- **Positive = inductive (lagging)** – typical for motors, transformers, induction furnaces.
- **Negative = capacitive (leading)** – typical for over‑compensated capacitor banks, cable charging.

This ensures that a power factor correction controller sees a positive reactive power when the load is inductive and needs compensation, and negative when the system is capacitive and should reduce capacitor banks. All protocol adapters must convert their meter's native sign to this standard.

**Why this must be locked down:**
- If the sign convention is ambiguous, a power factor controller will work incorrectly when swapping between a generic Modbus meter and a SunSpec meter, or when moving between sites with different meter brands. The DAL is the **source of truth** – adapters must conform.

---

## Example: End‑to‑End Flow

**Scenario:** Site imports 10 kW on total, but phase L1 is at 15 kW (overloaded), L2 at -3 kW (exporting), L3 at -2 kW (exporting). Peak shaving threshold is 8 kW total, but per‑phase export must be zero.

1. **Physical meter** reads:
   - `ActivePower_L1 = +15000` W (import)
   - `ActivePower_L2 = -3000` W (export)
   - `ActivePower_L3 = -2000` W (export)
   - Total = +10000 W (import)
2. **Modbus adapter** writes to DAL exactly these values (adhering to sign convention).
3. **Edge Core Cycle** snapshot captures these.
4. **Per‑phase zero export controller** runs: sees L2 and L3 negative, so it must charge battery or curtail PV to bring those phases to zero.
5. Controller computes required battery setpoints and writes to battery output channels.
6. **Ramp limiter** and command dispatcher send commands.

---

## Important Notes

- **Timestamping:** Every grid channel update must include a timestamp. This is crucial for detecting stale data and for aligning with other measurements.
- **Quality flags:** In production, you may add a `quality` field to indicate if the meter data is `GOOD`, `STALE`, or `ERROR` (e.g., due to communication loss).
- **Three‑phase systems:** For three‑phase meters, you may also have line‑to‑line voltages (`Voltage_L12`, etc.) depending on the meter. Include them if needed.
- **Virtual grid meter:** If no physical meter exists, you can create a virtual meter that computes grid power as:
  ```
  grid/ActivePower_L1 = load/ActivePower_L1 + battery/Power_L1 - pv/ActivePower_L1
  ```
  (assuming per‑phase measurements are available). The virtual meter service would write to `grid` channels just like a physical adapter.
- **Frequency_RoCoF:** This is an advanced channel. If your meter does not provide it directly, you can compute it in a controller or virtual service by differentiating frequency over time. For MVP, it can be omitted, but include it in the design to avoid breaking changes later.

---

## Summary

The Grid/Site Meter Abstraction provides a **single, consistent interface** for grid connection data. By defining a fixed set of channel names, a clear sign convention, and including per‑phase power, it enables:

- **Interoperability** – any meter that can be adapted speaks the same language.
- **Testability** – simulate grid conditions during development.
- **Unified control** – controllers can rely on `grid/ActivePower` and per‑phase values regardless of the actual hardware.
- **Future‑proofing** – adding RoCoF and per‑phase reactive power prepares for advanced grid services.