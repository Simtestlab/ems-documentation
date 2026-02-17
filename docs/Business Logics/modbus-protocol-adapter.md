# Modbus Protocol Adapter

The Modbus Protocol Adapter is the **foundation of all real‑time data acquisition and control** in the EMS.

- **Translates** raw Modbus register values into standardized, typed EMS channels.
- **Manages** the communication lifecycle with hundreds of heterogeneous field devices (inverters, meters, BMS).
- **Guarantees** temporal coherence and deterministic update rates required for closed‑loop control.

---

## Layer Descriptions & Critical Enhancements

### Layer 1 – Hardware Abstraction Layer (HAL)

**Role:** Semantic decoupling.  
**Standard function:** Map `State_of_Charge` → register `40091`.  
**Real‑world challenge (IEEE 2023):** Sites mix devices from different vendors with **incompatible data models** – a Siemens meter uses different addresses and scaling than a Sungrow inverter.

**Enhancements:**
- **Dynamic profile loading** – profiles stored in cloud, pushed to edge on startup or device change.
- **Dialect plugins** – handle non‑standard variants (e.g., 32‑bit floats stored in two 16‑bit registers with vendor‑specific endianness).

---

### Layer 2 – Optimized Scan Engine

**Role:** Deterministic data heartbeat.  
**Standard function:** Poll registers periodically.  
**Real‑world challenge (MDPI 2022):** Modbus can read multiple **consecutive** registers in one request. **Non‑consecutive registers require separate requests**, dramatically increasing acquisition time.

**Enhancements – Adaptive Frame Aggregation:**
The scan engine dynamically analyses the requested register list and selects the optimal read strategy:

| Scenario | Register Pattern | Requests | Efficiency |
|----------|------------------|----------|------------|
| **A** | Consecutive block (`40001–40010`) | **1** | High |
| **B** | Two consecutive blocks (`40001–40005`, `40050–40055`) | **2** | Optimal |
| **C** | Highly scattered (individual addresses) | **N** | **Warning – redesign profile** |

**Priority‑Based Poll Groups:**
- `FAST` group (e.g., `ActivePower`, `Soc`) – 200–500 ms interval.
- `SLOW` group (e.g., `Temperature`, `CumulativeEnergy`) – 5–60 s interval.
- Groups execute independently; critical data is always fresh.

**Connection Management:**
- TCP keep‑alive with exponential backoff on failure.
- RTU multi‑drop support with configurable inter‑frame delay.

---

### Layer 3 – Data Normalization & Temporal Coherence

**Role:** Standardization + time‑truth.  
**Standard function:** Scale raw integers (`2305` → `230.5 V`).  
**Real‑world challenge (MDPI 2022):** In distributed polling, register A and register B are read **at different moments**. Using them together without timestamps yields mathematically invalid power calculations.

**Enhancement – Acquisition Cycle Timestamping:**
Every channel update carries the **timestamp of the Modbus request that fetched it**, not the system processing time.

```
Channel:    meter0/ActivePower
Value:      12.34 kW
Timestamp:  2026-02-12T10:15:30.050Z  ← request sent
Quality:    GOOD
```

**Benefit:** Downstream controllers and historians know **exactly how stale** each data point is. Temporal alignment becomes explicit.

---

### Time Synchronization & Clock Skew Monitoring

**The Gap:** High‑precision timestamps are useless if the edge device's clock drifts. Even a 5‑second skew invalidates temporal alignment with cloud timestamps and grid signals.

**Solution – Mandatory Time Synchronization:**
- All edge devices **must** run an NTP client (Network Time Protocol) synchronized to a reliable time source (e.g., cloud‑based NTP servers or local GPS).
- For applications requiring microsecond precision (e.g., synchrophasors), **PTP (Precision Time Protocol, IEEE 1588)** may be used with hardware timestamping support.

**Monitoring & Alarming:**
- The EMS cloud periodically compares its own time with the edge's reported timestamp (from heartbeats or telemetry).
- If the deviation exceeds **100 ms**, a **Clock Skew Alarm** is raised.
- The alarm is visible in the dashboard and can trigger notifications (email, SMS) to operators.
- Edges with persistent skew may be flagged for maintenance (e.g., faulty RTC battery, NTP misconfiguration).

**Implementation Requirements:**
- Edge heartbeat messages must include a timestamp generated immediately before transmission.
- Cloud records arrival time and calculates offset.
- Offset is tracked over time; sustained offset >100 ms triggers alarm.
- Optionally, edges can self‑monitor their NTP synchronization status and report it as a diagnostic channel (e.g., `edge/NTP_Status`).

**Impact:** Ensures that all timestamps across distributed edges are mutually coherent, enabling accurate event correlation and historical analysis.

---

## Device Profile Specification

Profiles are stored as JSON (or YAML) and include **optimization directives** for the scan engine.

```json
{
  "profile_id": "sma_sunny_tripower_60",
  "protocol": "Modbus_TCP",
  "connection": {
    "port": 502,
    "timeout_ms": 2000,
    "retries": 2
  },
  "optimization": {
    "strategy": "auto_aggregate",
    "max_registers_per_frame": 125,
    "fast_poll_ms": 250,
    "slow_poll_ms": 10000
  },
  "channels": [
    {
      "tag": "ActivePower",
      "address": 30775,
      "type": "UINT32",
      "scale": 0.01,
      "unit": "kW",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "GridVoltage_L1",
      "address": 30799,
      "type": "UINT16",
      "scale": 0.1,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "HeatSinkTemp",
      "address": 30917,
      "type": "INT16",
      "scale": 0.1,
      "unit": "°C",
      "priority": "LOW",
      "poll_group": "slow"
    }
  ]
}
```

**How the engine uses this:**
1. Separates `fast` and `slow` groups into independent poll threads.
2. Within `fast`, sees addresses `30775` (UINT32 = 2 registers) and `30799` (UINT16 = 1 register).  
   → **Consecutive?** `30775`+2 = `30777`, next is `30799` → gap.  
   → Issues **two** frames: `30775-30777` and `30799`.
3. Schedules `fast` every 250 ms, `slow` every 10 s.
4. Attaches the **request send time** to every channel update.

---

## Implementation Recommendations

| Component          | Technology Choice | Rationale                                                                 |
| ------------------ | ----------------- | ------------------------------------------------------------------------- |
| **Edge Language**  | Python 3.11+      | `pymodbus` mature, rapid development, easy profile parsing.               |
| **Scan Engine**    | `asyncio`         | Concurrent polling of multiple devices; non‑blocking I/O.                 |
| **Profile Store**  | JSON + Git       | Human‑readable, version‑controlled, cloud‑sync ready.                      |
| **Message Bus**    | In‑memory dict   | Redis/ZeroMQ.                                                              |
| **Timestamping**   | `time.perf_counter()` | Nanosecond precision, monotonic.                                      |

**Optional (IEEE 2023 reference):** For extremely heterogeneous sites, a **Node‑RED** bridge can be deployed alongside the main engine to translate exotic protocols into Modbus TCP, which the adapter then ingests .

---

## Metrics & Acceptance Criteria

| Metric                          | Target                          | How Measured                     |
| ------------------------------- | ------------------------------- | -------------------------------- |
| **Poll cycle deviation**        | ≤ 10% of configured interval    | Logged request‑response jitter   |
| **Successful read rate**        | ≥ 99.5% (stable network)        | Successful / total requests      |
| **Non‑consecutive detection**   | 100% of fragmented maps flagged | Engine alert on profile load     |
| **Timestamp accuracy**          | ±1 ms of request send time      | Compare timestamp vs. wall clock |
| **Connection recovery**         | ≤ 30 s after link restore       | Simulated outage                 |

---


# Example Device Profiles

The following appendices provide concrete examples of device profiles derived from real‑world manuals. They illustrate how to translate manufacturer documentation into the JSON format used by the Modbus Protocol Adapter.

---

## TRUMPF TruConvert AC 3025 Inverter

Based on the *[TRUMPF TruConvert AC 3025 Operator's Manual](../static/TRUMPF_Manual_TruConvert_AC3025.pdf)*, this profile includes a subset of the most important Modbus registers for monitoring and control. The full register map is extensive; here we focus on the key parameters needed for basic operation and energy management.

### Device Overview

- **Device Type:** Bidirectional AC/DC inverter (grid‑forming and grid‑following capable).
- **Protocol:** Modbus TCP (default port 502) or Modbus RTU over RS‑485.
- **Key Features:** Supports up to 16 parallel units, island mode, droop control, and extensive diagnostics.

### JSON Profile

```json
{
  "profile_id": "trumpf_truconvert_ac3025",
  "manufacturer": "TRUMPF",
  "protocol": "Modbus_TCP",
  "connection": {
    "port": 502,
    "timeout_ms": 2000,
    "retries": 2
  },
  "optimization": {
    "strategy": "auto_aggregate",
    "max_registers_per_frame": 125,
    "fast_poll_ms": 500,
    "slow_poll_ms": 10000
  },
  "channels": [
    {
      "tag": "Status",
      "address": 4000,
      "type": "UINT16",
      "scale": 1.0,
      "unit": "",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Power stage configuration (bit0: enable)"
    },
    {
      "tag": "ActivePower",
      "address": 4195,
      "type": "INT16",
      "scale": 1.0,
      "unit": "kVA",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Signed power set value (sign influences cos φ)"
    },
    {
      "tag": "PowerFactor",
      "address": 4217,
      "type": "INT16",
      "scale": 0.01,
      "unit": "",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Set value cos φ for L1-L3 (range -100 to +100)"
    },
    {
      "tag": "GridVoltage_L1",
      "address": 5002,
      "type": "INT32",
      "scale": 0.1,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Grid voltage L1 (actual value)"
    },
    {
      "tag": "GridCurrent_L1",
      "address": 5004,
      "type": "INT32",
      "scale": 0.1,
      "unit": "A",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Grid current L1"
    },
    {
      "tag": "DCLinkVoltage",
      "address": 5006,
      "type": "INT32",
      "scale": 0.1,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "DC link voltage"
    },
    {
      "tag": "Frequency",
      "address": 5010,
      "type": "INT32",
      "scale": 0.01,
      "unit": "Hz",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Grid frequency"
    },
    {
      "tag": "Temperature",
      "address": 5020,
      "type": "INT16",
      "scale": 1.0,
      "unit": "°C",
      "priority": "LOW",
      "poll_group": "slow",
      "description": "Internal temperature"
    },
    {
      "tag": "AlarmCode",
      "address": 2810,
      "type": "UINT16",
      "scale": 1.0,
      "unit": "",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "First active alarm code"
    }
  ],
  "commands": [
    {
      "name": "enable_power_stage",
      "description": "Enable or disable power transmission",
      "register": 4000,
      "type": "UINT16",
      "values": {
        "disable": 0,
        "enable": 1
      }
    },
    {
      "name": "set_active_power",
      "description": "Set active power (signed)",
      "register": 4195,
      "type": "INT16",
      "scale": 1.0,
      "unit": "kVA"
    },
    {
      "name": "set_power_factor",
      "description": "Set power factor cos φ (range -100 to +100)",
      "register": 4217,
      "type": "INT16",
      "scale": 0.01
    }
  ]
}
```

### Derivation Notes

- Register addresses are taken from the Modbus register map in the manual (pages 77–96).
- All floating‑point values are transmitted as 32‑bit IEEE 754 over two consecutive 16‑bit registers; the adapter must handle this automatically.
- The profile defines both read‑only channels (actual values) and writable command registers.
- Fast poll group is assigned to critical operational data (power, voltage, current, frequency); slow group for temperature and less time‑sensitive data.

---

## SDM630 Energy Meter

Based on the *[SDM630 Modbus Protocol v2.0](../static/sdm630_modbus_protocol_v2.0.pdf)* document, this profile defines the most common energy‑metering parameters. The SDM630 is a widely used three‑phase energy meter with Modbus RTU interface.

### Device Overview

- **Device Type:** Three‑phase multifunction energy meter.
- **Protocol:** Modbus RTU (RS‑485).
- **Data Format:** All measured values are 32‑bit IEEE 754 floating point, stored in two consecutive 16‑bit registers.
- **Key Parameters:** Voltage, current, power, energy, frequency, power factor, THD.

### JSON Profile

```json
{
  "profile_id": "eastron_sdm630_v2",
  "manufacturer": "Eastron",
  "protocol": "Modbus_RTU",
  "connection": {
    "baud_rate": 9600,
    "data_bits": 8,
    "stop_bits": 1,
    "parity": "none",
    "unit_id": 1,
    "timeout_ms": 1000,
    "retries": 2
  },
  "optimization": {
    "strategy": "auto_aggregate",
    "max_registers_per_frame": 80,
    "fast_poll_ms": 1000,
    "slow_poll_ms": 60000
  },
  "channels": [
    {
      "tag": "Voltage_L1",
      "address": 0x0000,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Phase 1 line to neutral volts"
    },
    {
      "tag": "Voltage_L2",
      "address": 0x0002,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "Voltage_L3",
      "address": 0x0004,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "V",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "Current_L1",
      "address": 0x0006,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "A",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "Current_L2",
      "address": 0x0008,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "A",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "Current_L3",
      "address": 0x000A,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "A",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "ActivePower_Total",
      "address": 0x0034,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "W",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Total system power"
    },
    {
      "tag": "ApparentPower_Total",
      "address": 0x0038,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "VA",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "ReactivePower_Total",
      "address": 0x003C,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "VAR",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "PowerFactor_Total",
      "address": 0x003E,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "",
      "priority": "CRITICAL",
      "poll_group": "fast",
      "description": "Total power factor (positive = capacitive)"
    },
    {
      "tag": "Frequency",
      "address": 0x0046,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "Hz",
      "priority": "CRITICAL",
      "poll_group": "fast"
    },
    {
      "tag": "ImportEnergy_Total",
      "address": 0x0048,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "kWh",
      "priority": "MEDIUM",
      "poll_group": "slow",
      "description": "Total imported active energy since last reset"
    },
    {
      "tag": "ExportEnergy_Total",
      "address": 0x004A,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "kWh",
      "priority": "MEDIUM",
      "poll_group": "slow"
    },
    {
      "tag": "THD_Voltage_L1",
      "address": 0x00EA,
      "type": "FLOAT32",
      "scale": 1.0,
      "unit": "%",
      "priority": "LOW",
      "poll_group": "slow",
      "description": "Phase 1 voltage total harmonic distortion"
    }
  ],
  "commands": [
    {
      "name": "reset_energy",
      "description": "Reset accumulated energy values (requires password)",
      "register": 0x00D8,
      "type": "UINT16",
      "values": {
        "reset_energy": 1
      }
    }
  ]
}
```

### Derivation Notes

- Register addresses are given in hexadecimal in the manual; the profile uses the same values.
- The `type: "FLOAT32"` tells the adapter to read two consecutive 16‑bit registers and interpret them as IEEE 754 floating‑point.
- RTU connection parameters (baud rate, parity, etc.) are set to typical defaults; they can be overridden per installation.
- The meter also has holding registers for configuration; those are omitted here for brevity but could be included if needed.
- Fast poll group includes real‑time values; slow group is for accumulated energy and harmonic data.

### References

1. Enhancing the Modbus Communication Protocol to Minimize Acquisition Times Based on an STM32-Embedded Device [Modbus Communication](https://www.mdpi.com/2227-7390/10/24/4686)

2. Modbus TCP Bridging for Interconnecting Non-Compatible Devices in the Energy Sector Using Node-RED and Edge Computing [IEEE Site](https://ieeexplore.ieee.org/document/10517535)