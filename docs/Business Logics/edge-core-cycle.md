# Edge Core Cycle & Scheduler

## Purpose

The Edge Core Cycle & Scheduler is the **execution engine** of the EMS Edge node. It:

- Establishes a **deterministic, fixed‑frequency heartbeat** (typically 1 second) for all control activities.
- Orchestrates the **sequential execution** of all active controllers (balancing, peak shaving, etc.).
- Provides a **consistent snapshot** of input channels for each cycle, ensuring controllers operate on temporally coherent data.
- Monitors **cycle health** and prevents overruns that could destabilize real‑time control.

**Without this component, control algorithms run chaotically, conflicts are unresolved, and system behaviour is non‑deterministic.**

---

## Layer Descriptions & Critical Enhancements

### Layer 1 – Cycle Timer

**Role:** Generate a precise, periodic wake‑up signal.  
**Standard function:** `time.sleep(interval)`.  
**Real‑world challenge:** Simple sleeps accumulate error; OS scheduling introduces jitter.

**Enhancements:**
- **Deadline‑aware scheduling:** Calculate next wake time absolutely (`target = start + N*period`) and sleep until that instant.
- **Jitter logging:** Record deviation from ideal schedule; expose as diagnostic channel.
- **Period configuration:** Adjustable per edge (100–5000 ms) via cloud configuration.

---

### Layer 2 – Channel Snapshot

**Role:** Freeze all input channels at a single logical instant.  
**Standard function:** Controllers read live channels directly.  
**Real‑world challenge:** Live reads during a controller’s execution see values that may have been updated **mid‑cycle** by the Modbus poller, causing **temporal inconsistency** (e.g., voltage from t=0ms, current from t=500ms).

**Enhancement – Atomic Snapshot:**
- At the **very beginning** of each cycle, the engine takes a **consistent copy** of all input channels from the shared bus.
- Controllers **only** see this snapshot; they cannot see live updates during the cycle.
- All controllers in the same cycle operate on **identical, time‑aligned data**.

**Benefit:** Controllers produce setpoints based on a coherent view of the system, eliminating race conditions.

---

### Layer 3 – Scheduler & Controller Executor

**Role:** Run controllers in a deterministic order.  
**Standard function:** Iterate a list and call `execute()`.  
**Real‑world challenge:** Controllers may conflict (e.g., Balancing wants to discharge, Peak Shaving wants to charge). Without an arbiter, **last writer wins** – which is acceptable only if order is **intentional and configurable**.

**Enhancements:**
- **Configurable execution order:** Users/administrators set the priority order via cloud UI.
- **Execution time budget:** Each controller is given a time limit (e.g., 100 ms). Exceeding it triggers a warning and optionally disables the controller.
- **Setpoint conflict resolution:** Simple deterministic rule: **the controller that writes last to a given channel prevails**. This is explicit, auditable, and matches OpenEMS behaviour.

**Scheduler Configuration (JSON):**
```json
{
  "scheduler_id": "edge0_scheduler",
  "cycle_period_ms": 1000,
  "controllers": [
    {
      "id": "ctrlBalancing0",
      "class": "BalancingController",
      "enabled": true,
      "execution_order": 1,
      "time_budget_ms": 50
    },
    {
      "id": "ctrlPeakShaving0",
      "class": "PeakShavingController",
      "enabled": true,
      "execution_order": 2,
      "time_budget_ms": 80
    }
  ]
}
```

---

### Layer 4 – Cycle Supervisor

**Role:** Ensure the cycle remains healthy and deterministic.  
**Standard function:** None (assume everything runs forever).  
**Real‑world challenge:** A faulty controller may hang or exceed its time budget, delaying the next cycle and breaking real‑time guarantees.

**Enhancements:**
- **Overrun detection:** If cycle execution exceeds `cycle_period_ms`, an `OVERRUN` alarm is raised.
- **Watchdog timer:** If consecutive overruns occur, the edge may automatically restart or enter safe mode.
- **Cycle statistics:** Export min/max/avg execution time, jitter, and overrun count as diagnostic channels (visible in dashboard).

---

## Integration with Other Components

| Component               | Interaction                                                                                     |
| ----------------------- | ----------------------------------------------------------------------------------------------- |
| **Modbus Protocol Adapter** | Provides input channels; Core Cycle consumes a **snapshot** of these channels each cycle.         |
| **ESS / Meter Abstraction** | Channels are part of the snapshot; setpoints written by controllers are sent to these abstractions. |
| **Cloud Telemetry**     | Cycle statistics and overrun alarms are transmitted to cloud for monitoring.                    |
| **User Interface**      | Scheduler configuration (order, enabled/disable) is pushed from cloud to edge.                  |

---

## Metrics & Acceptance Criteria (MVP)

| Metric                          | Target                                      | How Measured                         |
| ------------------------------- | ------------------------------------------- | ------------------------------------ |
| **Cycle period accuracy**       | ±5% of configured period (99th percentile)  | Log wake‑up times                   |
| **Cycle jitter**               | ≤20% of period (p99)                       | Deviation from ideal schedule       |
| **Snapshot consistency**        | All controllers in same cycle see same data | Unit test with concurrent updates   |
| **Controller time‑budget compliance** | ≥99% of executions finish within budget | Per‑controller execution timer      |
| **Overrun detection**           | 100% of overruns ≥1 ms logged              | Time delta > period                |
| **Scheduler configuration**     | Changes applied within 1 cycle             | Test with cloud‑pushed config       |

---

## Configuration & Management

The scheduler configuration is part of the **edge’s device profile** stored in the cloud and synchronised to the edge.

**Example Edge Configuration (merged view):**
```json
{
  "edge_id": "site1_edge",
  "core_cycle": {
    "period_ms": 1000,
    "supervisor": {
      "overrun_alert_threshold": 2,
      "auto_recovery": true
    }
  },
  "scheduler": {
    "controllers": [
      { "id": "ctrlBalancing0", "enabled": true, "order": 1, "budget_ms": 50 },
      { "id": "ctrlPeakShaving0", "enabled": true, "order": 2, "budget_ms": 80 }
    ]
  }
}
```

**Dashboard visualisation:**  
- Real‑time cycle time gauge.  
- Controller execution timeline (per cycle).  
- Overrun count and last overrun timestamp.

---