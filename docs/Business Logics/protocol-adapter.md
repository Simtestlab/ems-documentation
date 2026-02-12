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

### References

1. Enhancing the Modbus Communication Protocol to Minimize Acquisition Times Based on an STM32-Embedded Device [Modbus Communication](https://www.mdpi.com/2227-7390/10/24/4686)

2. Modbus TCP Bridging for Interconnecting Non-Compatible Devices in the Energy Sector Using Node-RED and Edge Computing [IEEE Site](https://ieeexplore.ieee.org/document/10517535)