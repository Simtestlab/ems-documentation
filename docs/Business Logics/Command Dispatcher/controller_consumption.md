
# Controller Consumption & Arbitration

## Purpose

The **Controller Consumption & Arbitration** layer defines how active EMS controllers (e.g., Peak Shaving, Balancing, Manual Override, Safety) consume user commands from request channels, validate them against current system state, and determine the final setpoints sent to devices. It also specifies how conflicts between multiple controllers are resolved, ensuring that **safety always has the final say**.

This layer ensures that:
- User commands are safely integrated with automated control logic.
- Conflicting objectives are resolved deterministically.
- Final setpoints respect device limits and safety constraints.
- The outcome of each command (executed, rejected) is reported back via acknowledgment channels.
- **Safety controllers run last** to override any unsafe setpoints.
- **Request channels are cleared** after processing to prevent stale commands from persisting.

Without clear arbitration rules, the system could exhibit unpredictable behavior, race conditions, or unsafe states.

---

## Role in the Command Dispatcher Architecture

The controller layer sits between the request channels (where user commands land) and the output channels (which drive devices). It is part of the Edge Core Cycle.

```
┌─────────────────────────────────────────────┐
│         Edge Core Cycle                      │
├─────────────────────────────────────────────┤
│  • Take DAL snapshot                         │
│  • Run Controllers in order                  │
│    (with safety last)                         │
│  • Apply ramp limits                          │
│  • Write output channels                      │
└───────────────┬─────────────────────────────┘
                │ reads request channels
                │ writes ack channels
                │ clears request channels
┌───────────────▼─────────────────────────────┐
│         Device Abstraction Layer             │
│  • ui_requests/... (user commands)           │
│  • ack/... (final status)                    │
│  • output/... (device setpoints)             │
└─────────────────────────────────────────────┘
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Request Channel** | A DAL channel where user commands are stored (e.g., `ui_requests/battery_1_power`). Written by Edge‑Side Dispatcher, read by controllers. |
| **Controller** | A software module implementing a control strategy (e.g., peak shaving, balancing, safety clamping). It reads the snapshot (including request channels) and may write to output channels and ack channels. |
| **Execution Order** | The sequence in which controllers are run each cycle, defined in the scheduler configuration. Controllers with lower order numbers run first. **Safety controllers must have the highest order number to run last.** |
| **Arbitration** | The process of resolving conflicts when multiple controllers write to the same output channel. The rule is **last write wins** – the controller that runs last determines the final value. Therefore, safety must run last. |
| **Manual Override Controller** | A controller that monitors request channels for user commands and, if present, writes the requested setpoint to the output channel (subject to revalidation). It should run early so that automatic controllers can still adjust, but safety always has final override. |
| **Safety Controller** | A controller that enforces absolute hardware limits (e.g., `MaxChargePower`, `MinSoc`) and clamps any setpoints that exceed them. It must run last to ensure no unsafe value reaches the device. |
| **Acknowledgment Channel** | A DAL channel where controllers write the final outcome of a user command (executed, rejected, with reason). Written by controller, read by Edge‑Side Dispatcher to send final ack to cloud. |
| **Request Clearing** | After processing a user command, the responsible controller must set the request channel value to `None` (or a sentinel indicating no pending request). This prevents the command from being executed repeatedly and returns control to automatic controllers. |
| **Revalidation** | The process of checking a user command against current system state (SOC, limits, etc.) at execution time. This is necessary because conditions may have changed since the command was validated by the cloud. |

---

## Functional Requirements

- During each Edge Core Cycle, all enabled controllers shall be executed in a fixed, configurable order (defined by `execution_order`).
- Each controller shall receive the same DAL snapshot (frozen at cycle start) containing all input channels, including request channels.
- Controllers may write to output channels (e.g., `battery_1/SetActivePower`) and acknowledgment channels (e.g., `ack/battery_1_power`).
- If multiple controllers write to the same output channel, the value written by the controller with the highest `execution_order` (last to run) shall be the final setpoint.
- **Safety controllers (e.g., limit clamping) shall be assigned the highest execution order to ensure they run last and can override any unsafe setpoints.**
- When a controller processes a user command from a request channel, it must revalidate the command against current system state (e.g., SOC, limits, grid conditions). If the command is safe and feasible, it should be executed; otherwise, it should be rejected.
- **After processing a user command (whether executed or rejected), the controller shall clear the request channel by setting its value to `None` (or a null sentinel). This ensures the command is only executed once.**
- For every user command processed, the controller shall write a final status to the corresponding acknowledgment channel. The payload must include `command_id`, `status` (`executed`/`rejected`), `reason` (if rejected), and `executed_at` timestamp.
- A dedicated Manual Override Controller may be implemented to handle user-initiated setpoints. It should run early (e.g., before automatic controllers) so that automatic logic can still apply, but safety still runs last.
- Controllers shall not write directly to request channels (those are only written by the Edge-Side Dispatcher).
- All controller actions (reads, writes, decisions) shall be logged for audit and debugging.

---

## Detailed Flow

### Cycle Start and Snapshot

1. Edge Core Cycle begins.
2. A snapshot of all DAL channels is taken. This snapshot includes:
   - All input channels (telemetry from devices)
   - All request channels (user commands) – note: these may be `None` if no pending command
   - Current values of output channels (last commanded setpoints)
3. The snapshot is passed to the scheduler.

### Controller Execution Order

The scheduler maintains a list of active controllers with their `execution_order`. **Safety must run last**. Example:

| Order | Controller | Purpose |
|-------|------------|---------|
| 10 | Manual Override | Applies user commands (early) |
| 20 | Balancing | Self‑consumption logic |
| 30 | Peak Shaving | Reduces grid import peaks |
| 100 | Safety Limits / Hardware Clamping | Enforces absolute limits – **runs last** |

Rationale:
- Manual Override runs first so that user intent is considered by subsequent automatic controllers (they may adjust based on the user's setpoint).
- Automatic controllers compute their desired setpoints.
- Safety runs last to clamp any setpoint that exceeds hardware limits, regardless of what previous controllers wrote. This ensures safety always prevails.

### Manual Override Controller Logic

1. Reads all request channels from the snapshot (e.g., `ui_requests/battery_1_power`). **Presence is determined by checking if the value is not `None`** (not by non‑zero).
2. If a request is present:
   - Revalidate against current system state:
     - Is SOC within allowed range for charge/discharge?
     - Is requested power within current `MaxChargePower`/`MaxDischargePower`?
     - Are there any grid constraints (e.g., export limits)?
   - If valid, write the requested value to the corresponding output channel (e.g., `battery_1/SetActivePower = requested_value`).
   - If invalid, write to acknowledgment channel with status `rejected` and reason (e.g., "SOC too low").
   - **Regardless of outcome, write the final decision to the acknowledgment channel with the appropriate status.**
   - **Clear the request channel:** Set `ui_requests/battery_1_power = None` to remove the pending command.
3. If no request is present, the controller takes no action (does not write to output or ack channels).

### Automatic Controllers (e.g., Balancing, Peak Shaving)

1. Read relevant telemetry (grid power, PV power, load, battery SOC) from snapshot.
2. Compute desired battery power based on control strategy.
3. Write the computed setpoint to output channel (e.g., `battery_1/SetActivePower`).
   - Note: This may be overwritten by a later automatic controller or the safety controller.

### Safety Controller (Runs Last)

1. Read the current (tentative) output channel values from the snapshot (after previous controllers have written).
2. For each output channel (e.g., `battery_1/SetActivePower`):
   - Compare the value against absolute hardware limits (e.g., `MaxChargePower`, `MaxDischargePower`, `MinSoc` conditions).
   - If the value exceeds limits, clamp it to the maximum safe value.
   - If the value is unsafe (e.g., charging when battery is full), override to `0`.
   - Write the clamped value back to the output channel (overwriting any previous writes).
3. The safety controller **does not** write to acknowledgment channels (those are handled by the controller that processed the user command).

### Arbitration (Last Write Wins with Safety Final)

- After all controllers have run, the final values in output channels are those written by the last controller that touched them – which is the safety controller.
- Example:
  - Manual Override writes `SetActivePower = 50000` (user wants 50kW charge).
  - Balancing writes `SetActivePower = -2000` (discharge 2kW) – but since Manual Override already wrote 50000, Balancing's write might be ignored if it happens after? Actually order matters: Manual Override runs first (order 10), Balancing runs later (order 20), so Balancing would overwrite Manual Override. That's not what we want for manual override. Perhaps we need to reconsider order: If we want manual override to have priority over automatic controllers but still allow safety to clamp, we should run manual override **after** automatic controllers but **before** safety. But then manual override could still be overridden by safety, which is correct.

Let's refine the order to reflect proper priority:

| Order | Controller | Purpose |
|-------|------------|---------|
| 10 | Balancing | Self‑consumption logic |
| 20 | Peak Shaving | Reduces grid import peaks |
| 30 | Manual Override | Applies user commands (overrides automatic) |
| 100 | Safety Limits | Enforces absolute limits – **runs last** |

This way:
- Automatic controllers run first, establishing a baseline.
- Manual Override runs next, potentially overriding automatic decisions with user intent.
- Safety runs last, clamping any unsafe values (including those from manual override).

Thus, the final setpoint is determined by user intent (if present) but always within safe bounds.

### Acknowledgment Handling

- Controllers that process user commands (e.g., Manual Override) write to `ack/` channels immediately after making a decision.
- The payload must include `command_id`, `status`, `reason` (if rejected), and `executed_at`.
- The Edge‑Side Dispatcher (which subscribes to `ack/` channels) forwards these to the cloud.

### Request Clearing

- After a user command is processed (executed or rejected), the responsible controller **must** clear the request channel by setting it to `None`. This ensures that the same command is not executed in subsequent cycles and that automatic controllers regain control.
- If the request channel is not cleared, the command would persist indefinitely, locking out automation.

### Cycle End

- After all controllers run, the final output channels are passed through ramp limiters (if configured) and written back to DAL (or directly to protocol adapters via Command Dispatcher).

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Edge Core Cycle** | Invokes controllers in order, provides snapshot. |
| **Device Abstraction Layer** | Provides request channels, output channels, ack channels. |
| **Edge‑Side Dispatcher** | Writes to request channels; reads from ack channels to forward final status. |
| **Device Profiles** | Provide validation rules (limits) used by controllers during revalidation and safety clamping. |
| **Scheduler Configuration** | Defines which controllers are active and their execution order. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Controller execution time** | < 50 ms per controller (p99) | Log per‑controller execution time. |
| **Arbitration correctness** | Final setpoint always equals last writer's value (safety) | Unit tests with mock controllers. |
| **Revalidation accuracy** | 100% of unsafe commands rejected | Test with edge cases (low SOC, high power). |
| **Request clearing** | 100% of processed commands cleared within the same cycle | Verify request channel becomes `None` after processing. |
| **Ack channel latency** | < 100 ms from decision to ack write | Log timestamp difference. |

---

## Implementation Notes

- **Controller Interface:** Each controller should implement a simple interface, e.g.:
  ```python
  class Controller:
      def __init__(self, config):
          self.config = config
      def execute(self, snapshot):
          # return dict of channels to write (output + ack)
          pass
  ```
- **Scheduler:** The Edge Core Cycle should load controllers dynamically based on configuration, sort by `execution_order`, and call them sequentially, collecting writes. After all controllers run, merge writes (last write wins) and apply to DAL.
- **Manual Override Controller:**
  - Check for presence by testing `value is not None`. Do not use `if value:` because `0` is a valid command.
  - After processing, set the request channel to `None` using `DAL.set_channel(request_channel, None, timestamp)`.
- **Safety Controller:**
  - Should have access to the device's absolute limits (from device profile) and possibly dynamic limits from BMS.
  - It should clamp the setpoint and write the clamped value back to the output channel.
  - It does not need to write to ack channels.
- **Revalidation Logic:** Use the same device profiles and dynamic limit channels as cloud validation. Since the controller has the freshest snapshot, it can apply the same checks.
- **Ack Channel Naming:** Use `ack/{request_channel_name}`. For example, if request channel is `ui_requests/battery_1_power`, the ack channel should be `ack/battery_1_power`. This allows the Edge‑Side Dispatcher to subscribe easily.
- **Logging:** Log each controller's decisions, especially when rejecting a user command, with reason and command_id.

---

## Summary

The Controller Consumption & Arbitration layer ensures that user commands are safely integrated with automated control logic. By running controllers in a fixed order with **safety last**, revalidating commands against real‑time conditions, applying a simple "last write wins" arbitration rule, and **clearing request channels after processing**, the system remains deterministic, safe, and responsive. The dedicated Manual Override Controller, when placed after automatic controllers but before safety, guarantees that user intent can override automatic decisions when appropriate, while still respecting hardware limits and safety constraints. This design provides a solid foundation for both automatic and manual control in the EMS.