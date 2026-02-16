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

### Layer 1 – Deterministic Cycle Timer & Runtime Environment

**Role:** Generate a precise, periodic wake‑up signal and maintain a deterministic execution environment.

**Standard function:** `time.sleep(interval)` on a standard OS.

**Real‑world challenge:** 
- Standard sleep relies on the system "Wall Clock" (susceptible to NTP shifts) and the default OS scheduler (susceptible to preemption by background tasks).
- Python's internal Garbage Collector (GC) and OS memory paging can introduce random latencies of 10–100ms ("Jitter"), violating control loop requirements.

#### 1. Monotonic Deadline Scheduling

Instead of relative sleeps, the timer uses a **Monotonic Clock** (e.g., `time.perf_counter`) which is immune to system time changes (NTP).

- **Absolute Deadline Calculation:** The next wake-up time is calculated algebraically: `target_wake_time = cycle_start_time + (cycle_count * period_sec)`.
- **Drift Compensation:** By sleeping until an absolute point in time, any execution jitter from the previous cycle is mathematically eliminated in the next, preventing cumulative drift.

#### 2. Soft Real-Time OS Priority (`SCHED_FIFO`)

The Edge Core process elevates its own priority to bypass standard time-sharing scheduling.

- **Mechanism:** Uses the `sched_setscheduler` syscall to request **SCHED_FIFO** policy with high priority (e.g., 99).
- **Benefit:** This ensures the kernel preempts almost all other user-space processes (logging, updates, networking) immediately when the Core Cycle needs to run.
- **Hard Real-Time Option:** For sub-millisecond requirements, the software supports deployment on a **PREEMPT_RT** patched Linux kernel.

#### 3. Deterministic Memory Management

To prevent non-deterministic pauses caused by memory operations, the runtime environment is hardened:

- **Memory Locking (`mlockall`):** The process forces all current and future memory pages to stay in physical RAM, preventing the OS from "swapping" memory to disk (which causes massive latency spikes).
- **Manual Garbage Collection (GC):** Automatic Python GC is **disabled**. Instead, the cycle supervisor manually triggers `gc.collect()` only during the "Slack Time" (idle time) at the end of a cycle, ensuring no "Stop-the-World" pauses occur during critical control logic.

#### 4. Jitter Monitoring

The supervisor continuously measures the delta between the _ideal_ wake-up time and the _actual_ execution start time.

- **Diagnostic Channel:** `sys_core/CycleJitter` (microseconds).
- **Alarming:** If Jitter consistently exceeds a safety threshold (e.g., 10% of cycle period), a **High Jitter Alarm** is raised to indicate CPU starvation.

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