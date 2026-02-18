# Edge Core Cycle & Scheduler

## Purpose

The **Edge Core Cycle & Scheduler** is the heartbeat of the EMS Edge node. Think of it as the main loop of a real‑time application – it runs continuously at a fixed interval (e.g., once per second) and orchestrates everything that needs to happen in that time frame.

In an EMS, this component is responsible for:

- **Timing:** Ensuring all control actions happen at predictable, regular intervals.
- **Data consistency:** Providing a stable, unchanging view of the world to all decision‑making modules (controllers).
- **Orderly execution:** Running control algorithms (like "peak shaving" or "self‑consumption") in a defined sequence, so they don't interfere with each other.
- **Health monitoring:** Detecting if something goes wrong (e.g., a controller takes too long) and raising alarms.
- **Fault isolation:** Preventing a single crashed controller from taking down the entire system.
- **Safety:** Resetting outputs to safe defaults when no controller is active.

---

## Role in the EMS Architecture

The Edge Core Cycle sits in the middle of the edge software stack:

```
┌─────────────────────────────────────────────┐
│           Cloud / Dashboard                  │
│  (User interface, configuration, storage)    │
└─────────────────────┬───────────────────────┘
                      │ (WebSocket / API)
┌─────────────────────▼───────────────────────┐
│           Edge Core Cycle & Scheduler        │ ← You are here
├─────────────────────────────────────────────┤
│  • Reads inputs from Device Abstraction Layer│
│  • Runs controllers in order                 │
│  • Sends commands to devices                 │
└─────────────────────┬───────────────────────┘
                      │ (Normalized channels)
┌─────────────────────▼───────────────────────┐
│      Device Abstraction Layer                 │
│  (Holds latest values from all devices)       │
└─────────────────────┬───────────────────────┘
                      │ (Protocol‑specific data)
┌─────────────────────▼───────────────────────┐
│   Protocol Adapters (Modbus, CAN, SunSpec)   │
│   & Simulators                                │
└─────────────────────────────────────────────┘
```

The cycle is the conductor of the orchestra – it tells everyone when to play.

---

## Key Concepts

| Term | Meaning |
|------|---------|
| **Cycle** | A fixed‑duration loop (e.g., 1 second). Every cycle, we do the same sequence of steps.
| **Scheduler** | The component that decides **which controllers run and in what order**. |
| **Channel** | A named piece of data, like `battery_1/Soc` (state of charge) or `grid/ActivePower`. | A variable in a global store (e.g., Redux state). |
| **Snapshot** | A copy of all channel values taken at the **start** of a cycle. Controllers see this frozen snapshot, not live‑updating values. |
| **Controller** | A piece of logic that reads channels and optionally writes setpoints (commands). **This is where high‑level EMS logic lives – including State Machines (IDLE → CHARGING), Peak Shaving, Balancing, etc.** |
| **Setpoint** | A command to change something, e.g., `battery_1/SetPower = 5000` (charge at 5 kW). |
| **Jitter** | The variation in how long a cycle actually takes. Ideally zero; in reality, small. | The variability in server response times.  This is bad because the next cycle will be delayed.

## Functional Requirements

1. **Run at a fixed frequency** – by default once per second (1000 ms), configurable per edge.
2. **Reset all output setpoints to a safe default** at the start of each cycle (e.g., 0 or None).
3. **Take a snapshot** of all input channels at the very beginning of each cycle.
4. **Execute a list of controllers** in a user‑configurable order.
5. **Isolate controller failures** – if a controller crashes, log it and continue with the next.
6. **Apply time budgets** to each controller – if a controller runs too long, log a warning or disable it.
7. **Handle setpoints** – controllers may write to output channels. The last controller to write to a channel "wins".
8. **Apply ramp limits** to smooth abrupt setpoint changes.
9. **Detect overruns** – if the whole cycle takes longer than the configured period, raise an alarm.
10. **Expose diagnostics** – cycle timing statistics (min, max, average, jitter) as channels that can be sent to the cloud.
11. **Support simulation mode** – when no real devices are present, the cycle still runs and can interact with simulated device models.

---

## How It Works

Imagine the cycle as a loop that runs forever. Each iteration does:

### Wait for the next cycle start
- The cycle keeps track of an **absolute wake‑up time** (e.g., start at time T, then T+1s, T+2s, …).
- It sleeps until that exact moment, compensating for any delays from the previous cycle.

### Reset outputs to safe defaults
- **At the very beginning of the cycle (before any controllers run), all output setpoints are reset to a safe default (e.g., `0` or `None`). This ensures that if no controller writes to a channel during the cycle, the safe default is sent to the device.**

### Take a snapshot
- At the moment the cycle starts, it reads **all input channels** from the Device Abstraction Layer and makes a frozen copy.
- All controllers in this cycle will see this same snapshot – they won't see values that change halfway through the cycle.

### Execute controllers in order
- The scheduler has a list of controller IDs with an `execution_order` number.
- For each controller in ascending order:
  - Start a timer.
  - **Wrap the controller execution in a `try/catch` block. If a controller throws an unhandled exception:**
    - **Log the error with full stack trace.**
    - **Mark the controller as "Faulted" for this cycle (it will not write any setpoints).**
    - **Immediately move to the next controller. The cycle must never crash.**
  - Call the controller's logic, passing it the snapshot.
  - The controller reads snapshot values and may write setpoints to output channels.
  - If the controller exceeds its `time_budget_ms`, log a warning and optionally disable it for future cycles.
- Because controllers write to a shared output area, the **last one to write** to a particular channel determines the final setpoint.

### Apply ramp limits
- After all controllers have run, the system has a set of desired setpoints (e.g., `battery_power = 5000`).
- Before sending them to the device, a **ramp limiter** checks the last commanded value and ensures the change isn't too abrupt.
- If the requested change exceeds the allowed ramp rate (e.g., 100 kW/s), it is capped to a safe increment.

### Send commands to devices
- The final, limited setpoints are written to the appropriate output channels.
- The Device Abstraction Layer passes them to the protocol adapters (or simulators) for execution.

### Update statistics
- Record the actual cycle duration (from snapshot to just before sleeping).
- Update min/max/avg/jitter metrics.
- If duration > cycle period, increment an overrun counter and raise an alarm.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Device Abstraction Layer (DAL)** | The cycle reads a snapshot from DAL at the start of each cycle and writes final setpoints back to DAL at the end. |
| **Controllers** | Each controller is a separate module that implements an `execute(snapshot)` method. Controllers are registered with the scheduler. |
| **Command Dispatcher** (part of cycle) | The step that applies ramp limits and sends final setpoints to DAL. In a simple implementation, this can be inside the cycle. |
| **Cloud Telemetry** | Cycle statistics (jitter, overruns) are published as channels and sent to the cloud for monitoring. |
| **Configuration (Edge Config Sync)** | The scheduler's controller list, cycle period, and ramp rates are loaded from a local `config.json` (initially) or from the cloud later. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How to Measure |
|--------|--------|----------------|
| **Cycle period accuracy** | Within ±5% of configured period (99th percentile) | Log the actual wake‑up times and compare to ideal schedule. |
| **Cycle jitter** | ≤20% of period (e.g., 200ms jitter for a 1s cycle) | Standard deviation of wake‑up delays. |
| **Snapshot consistency** | All controllers in same cycle see identical data | Unit test: have one controller modify a channel and another read it – they should see the old value. |
| **Controller time‑budget compliance** | ≥99% of executions finish within budget | Per‑controller execution timers. |
| **Exception isolation** | 100% – a crashing controller never stops the cycle | Inject a `throw` in a test controller and verify cycle continues. |
| **Output reset** | All setpoints reset to safe default at cycle start | Verify that with no active controllers, safe defaults are sent. |
| **Overrun detection** | 100% of overruns ≥1 ms are logged | Compare actual cycle end time to period. |
| **Ramp limiting** | No setpoint changes faster than configured rate | Simulate abrupt changes and verify output is smoothed. |
| **Configuration changes** | Changes to `config.json` are applied within 1 cycle | Modify file and watch for controller order change. |

---

## Configuration & Management

For initial development, we'll use a simple `config.json` file placed on the edge (e.g., in `/etc/ems/config.json`). The edge reads it at startup and can optionally watch for changes.

**Example `config.json`:**
```json
{
  "edge_id": "sim_site_1",
  "cycle": {
    "period_ms": 1000,
    "safe_defaults": {
      "battery_1/SetActivePower": 0,
      "pv_inverter/SetCurtailment": 0
    },
    "ramp_rates": {
      "battery_1/SetActivePower": 100   // kW per second
    }
  },
  "controllers": [
    {
      "id": "balancing",
      "enabled": true,
      "execution_order": 1,
      "time_budget_ms": 50
    },
    {
      "id": "peak_shaving",
      "enabled": true,
      "execution_order": 2,
      "time_budget_ms": 80
    }
  ]
}
```

Later, when we implement Edge Configuration Sync, the edge will receive updates from the cloud, but we can keep the local file as a fallback.

---

## Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| **Sleep drift** | Use absolute deadline scheduling, not `sleep(1)`. Calculate next wake time and sleep until then. |
| **Snapshot inconsistency** | Always copy data at the start; don't let controllers read live values during the cycle. |
| **Controller conflicts** | Enforce "last write wins" and make the order configurable. Document that controllers must not rely on others' outputs in the same cycle. |
| **Unbounded controller execution** | Always enforce a time budget; otherwise a faulty controller can freeze the system. |
| **Crashed controller takes down the system** | **Wrap every controller in `try/catch`. Log and continue – never let the cycle die.** |
| **Stale setpoints from disabled controllers** | **Reset all outputs to safe defaults at cycle start. If no controller writes, safe default is sent.** |
| **Abrupt setpoint changes** | Implement ramp limiting – it's a safety feature that also makes simulation more realistic. |
| **Overrun not visible** | Expose overrun counts as channels and send them to the cloud so operators can see if the edge is overloaded. |

---

## Glossary

- **Channel:** A named variable representing a measurement or setpoint (e.g., `battery/Soc`, `inverter/SetPower`).
- **Controller:** A module that implements EMS logic (e.g., peak shaving). It reads channels and may write setpoints. **This includes State Machines and other high‑level logic.**
- **Cycle:** One iteration of the main loop.
- **Jitter:** Variation in cycle start times.
- **Overrun:** When a cycle takes longer than its allocated period.
- **Ramp rate:** Maximum allowed change in a setpoint per second (kW/s).
- **Safe default:** A predefined safe value (e.g., `0`) sent when no controller writes to a channel.
- **Snapshot:** A frozen copy of all channel values taken at cycle start.
- **Setpoint:** A command value written to a device (e.g., charge power).

---