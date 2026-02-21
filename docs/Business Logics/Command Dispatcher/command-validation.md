# Command Validation

## Purpose

Command validation ensures that every user‑initiated command (setpoint, mode change, override) is safe, feasible, and correctly formatted **before** it reaches the edge device or the control logic. Validation happens at multiple layers:

- **Syntax and basic structure** – is the JSON well‑formed and contains required fields?
- **Access rights** – is the target channel writable?
- **Static limits** – does the requested value fall within the device’s absolute hardware limits?
- **Dynamic limits** – does the requested value respect the current real‑time limits (e.g., current `MaxChargePower` from the BMS), with a check for telemetry staleness?
- **Enumeration checks** – for mode commands, is the requested string one of the allowed values?
- **Contextual validation** – after the command is queued, controllers may re‑validate due to changing conditions.

By catching errors early, we prevent invalid commands from consuming network bandwidth, clogging queues, or causing unsafe operations on the edge.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Device Profile** | A JSON document defining a device’s channels, their types, units, access rights, and validation rules (static limits, dynamic limit references, allowed enums). |
| **Static Limit** | A fixed numerical bound defined in the profile (e.g., `static_min: -100000`, `static_max: 100000`). These represent the absolute physical capability of the device. |
| **Dynamic Limit** | A limit that varies based on real‑time conditions, such as `MaxChargePower` reported by the BMS. The profile references another channel (e.g., `dynamic_max_channel: "MaxChargePower"`) whose current value is fetched during validation. |
| **Effective Limit** | The combination of static and dynamic limits. For a `write` channel, the effective allowed range is `max(static_min, dynamic_min)` to `min(static_max, dynamic_max)`. |
| **Sign Convention Matching** | Dynamic limit channels must use the **same sign convention** as the setpoint channel they constrain. For example, if the setpoint uses **negative values for discharge**, then the `MaxDischargePower` channel must also be negative (e.g., `-50000`). The protocol adapter is responsible for converting hardware‑positive values to the EMS convention. |
| **Staleness Threshold** | Dynamic limit telemetry older than a defined threshold (e.g., 30 seconds) is considered unreliable. Commands validated against stale data must be rejected. |
| **Enum Validation** | For string‑based channels (e.g., `RequestedMode`), the profile lists `allowed_values` (e.g., `["AUTO", "FORCE_CHARGE", "FORCE_DISCHARGE", "IDLE"]`). The command value must exactly match one of these. |
| **Access Rights** | Each channel has an `access` field: `"read"`, `"write"`, or `"read_write"`. Commands can only target channels with `write` or `read_write` access. |

---

## Validation Layers

Validation is performed in sequence; if any layer fails, the command is rejected immediately with a descriptive error.

### Cloud‑Side Validation

When a command arrives at the Cloud‑Side Dispatcher, it undergoes:

1. **Permission Check** – User must have `can_control` permission for the target edge.
2. **Device Profile Lookup** – Fetch the profile for the target device.
3. **Channel Existence and Access** – Ensure the channel exists in the profile and has `access: "write"` (or `"read_write"`).
4. **Static Limit Validation** – If numeric, check `static_min`/`static_max`. If out of bounds, reject.
5. **Enum Validation** – If channel is enum, check `allowed_values`.
6. **Dynamic Limit Validation** – If the profile defines `dynamic_min_channel` or `dynamic_max_channel`, query the latest telemetry for those channels. **Check the timestamp** of the telemetry; if older than the configured staleness threshold (e.g., 30 seconds), reject the command with `reason: "Cannot validate: Telemetry for dynamic limits is stale."` Otherwise, compute effective limits and compare.

If any check fails, the cloud returns an error to the UI and does not forward the command to the edge.

### Edge‑Side Validation

Even after cloud validation, the edge performs its own lightweight validation before writing to the request channel:

1. **Device Existence** – Does the device exist locally?
2. **Request Channel Mapping** – Is the target channel a known request channel (e.g., `ui_requests/...`)? This is based on locally cached profiles.
3. **Basic Type Check** – Ensure the value type matches expectations (numbers for numeric channels, strings for enums).

Edge validation uses the same device profiles (synced via Edge Config Sync) to avoid discrepancies. If edge validation fails, it sends a rejection acknowledgment back to the cloud (which then relays to the UI).

### Controller‑Level Validation

Once the command lands in the request channel, the Core Cycle controllers read it and may perform additional **contextual validation**. This step is essential because:

- Commands may have been queued offline, and real‑time conditions (SOC, temperature, limits) could have changed between cloud validation and execution.
- The controller has the most up‑to‑date snapshot, including any dynamic limit channels refreshed in the current cycle.

The controller may reject the command based on current conditions (e.g., SOC too low for discharge). If rejected, it writes a final status to the `ack/` channel, which the edge forwards to the cloud. This is the final opportunity to refuse execution.

---

## How Device Profiles Define Validation Rules

Below is an excerpt from a typical ESS device profile showing validation‑related fields:

```json
{
  "profile_id": "ess_generic_v1",
  "channels": {
    "RequestedActivePower": {
      "type": "numeric",
      "unit": "W",
      "access": "write",
      "validation": {
        "type": "range",
        "static_min": -100000,
        "static_max": 100000,
        "dynamic_min_channel": "MaxDischargePower",
        "dynamic_max_channel": "MaxChargePower"
      }
    },
    "RequestedMode": {
      "type": "string",
      "access": "write",
      "validation": {
        "type": "enum",
        "allowed_values": ["AUTO", "FORCE_CHARGE", "FORCE_DISCHARGE", "IDLE"]
      }
    },
    "MaxChargePower": {
      "type": "numeric",
      "unit": "W",
      "access": "read"
    },
    "MaxDischargePower": {
      "type": "numeric",
      "unit": "W",
      "access": "read"
    }
  }
}
```

- For `RequestedActivePower`, the cloud will fetch the latest `MaxChargePower` and `MaxDischargePower` values (from telemetry) to compute the effective range.
- For `RequestedMode`, the cloud ensures the value is one of the four allowed strings.

---

## Dynamic Limit Handling in Detail

Dynamic limits are essential for safety because battery capabilities change with temperature, SOC, and age. The validation process includes several critical steps:

### Sign Convention Matching
Protocol adapters must ensure that limit channels (e.g., `MaxDischargePower`) are stored in the DAL using the **same sign convention as the setpoint** they constrain. According to EMS conventions:
- Charging uses **positive** power values.
- Discharging uses **negative** power values.
- Therefore, `MaxDischargePower` should be a **negative** number (e.g., `-50000`) when written to the DAL. The protocol adapter must convert the hardware‑reported positive discharge limit (e.g., 50000) to negative. If this conversion is not done, the effective limit calculation `max(static_min, dynamic_min)` will fail – for example, `static_min = -100000` and `dynamic_min = 50000` would incorrectly yield `50000` as the minimum, making discharge impossible.

### Telemetry Retrieval and Staleness Check
1. Cloud receives command for `RequestedActivePower = 30000` (30kW charge).
2. Profile indicates `dynamic_max_channel: "MaxChargePower"`.
3. Cloud queries the latest telemetry for `MaxChargePower`. The result includes a timestamp.
4. **Staleness Check:** If the timestamp is older than the configured staleness threshold (e.g., 30 seconds), the cloud rejects the command with `reason: "Cannot validate: Telemetry for dynamic limits is stale."` This prevents the system from acting on outdated battery limits.
5. If the telemetry is fresh, the cloud computes effective max = `min(static_max, dynamic_max)`.
6. It compares the requested value and accepts or rejects accordingly.

### Effective Limit Calculation Example
- Static limits: `static_min = -100000` (max discharge), `static_max = 100000` (max charge).
- Dynamic limits: `MaxChargePower` (from BMS) = `50000` (positive, meaning maximum charge power). `MaxDischargePower` (after sign conversion) = `-30000` (meaning maximum discharge is 30kW, negative).
- Effective range: `min = max(-100000, -30000) = -30000`, `max = min(100000, 50000) = 50000`.
- Allowed values: from `-30000` to `50000`. A request for `-40000` (40kW discharge) would be rejected because it exceeds the current discharge limit.

---

## Example Validation Flows

### Example 1: Successful Power Command

- User requests `RequestedActivePower = 30000` (30kW charge).
- Profile: static limits ±100kW, dynamic max = `MaxChargePower`.
- Current `MaxChargePower` from telemetry = 50kW (timestamp < 30s old).
- Cloud computes effective max = 50kW, value 30kW is within range.
- Command passes cloud validation, is forwarded to edge.
- Edge writes to `ui_requests/battery_1_power`.
- Controller reads request, checks SOC (50%) and limits, accepts, and writes to output. Controller also writes `ack/battery_1_power` with status `executed`.
- Edge forwards ack to cloud → UI shows success.

### Example 2: Rejected Due to Dynamic Limit

- User requests `RequestedActivePower = 60000` (60kW charge).
- Current `MaxChargePower` = 50kW (fresh).
- Cloud computes effective max = 50kW, rejects with reason `"Requested power exceeds current charge limit (50000W)"`.
- UI immediately shows error.

### Example 3: Rejected Due to Stale Telemetry

- User requests `RequestedActivePower = 30000`.
- `MaxChargePower` telemetry in database is 15 minutes old.
- Cloud staleness check fails; rejects with `reason: "Cannot validate: Telemetry for dynamic limits is stale."`
- UI shows error indicating telemetry is stale.

### Example 4: Rejected by Controller (Contextual)

- User requests `RequestedMode = "FORCE_DISCHARGE"`.
- Cloud validation passes (allowed enum, dynamic limits ok).
- Edge writes to `ui_requests/battery_1_mode`.
- Controller reads request, but battery SOC is 5% (below `MinSoc`). It refuses and writes to `ack/battery_1_mode` with status `rejected` and reason `"SOC too low for discharge"`.
- Edge forwards ack → UI shows rejection reason.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Device Profiles** | Provide all validation rules. Profiles are stored in cloud and synced to edge. |
| **Telemetry Database** | Supplies current values and timestamps for dynamic limit channels (cloud‑side validation). |
| **Redis Cache** | (Optional) Caches latest telemetry for low‑latency dynamic limit checks, preserving timestamps. |
| **Cloud‑Side Dispatcher** | Performs static, dynamic, and staleness validation before forwarding commands. |
| **Edge‑Side Dispatcher** | Performs lightweight validation (device existence, channel mapping) before writing to request channels. |
| **Controllers** | Perform final contextual validation and write outcome to `ack/` channels. |
| **Device Abstraction Layer** | Holds the latest values and timestamps of dynamic limit channels (e.g., `MaxChargePower`). |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Cloud validation latency** | < 50 ms (p99) | Time from command receipt to validation decision |
| **Dynamic limit query latency** | < 100 ms (p99) | Time to fetch telemetry from cache/db |
| **Staleness enforcement** | 100% of commands with stale dynamic telemetry rejected | Unit tests with mocked old timestamps |
| **Rejection reason accuracy** | 100% of rejections include a human‑readable reason | Manual review of logs |
| **Validation coverage** | 100% of writable channels have validation rules | Audit of all device profiles |
| **False positive rate** | 0% (valid commands should never be rejected) | Regression tests |

---

## Implementation Notes

- **Cloud‑side:** The validation logic should be a reusable module called by the WebSocket handler. It returns a `(valid, reason)` tuple. For dynamic limits, use a cache (Redis) keyed by `edge_id:device_id:channel` with a short TTL (e.g., 5 seconds) and store both value and timestamp. When retrieving, check if timestamp is within staleness threshold.
- **Sign convention enforcement:** Protocol adapters must be documented to convert hardware discharge limits to negative values. The cloud validation logic can assume that limit channels already follow the EMS sign convention.
- **Staleness threshold:** Configure globally (e.g., 30 seconds) and possibly per device type. Make it configurable.
- **Edge‑side validation:** Simple checks using in‑memory map built from local profiles.
- **Controller‑side:** Controllers have access to the same device profile data. They should write to `ack/` channels with a consistent schema:
  ```json
  { "command_id": "...", "status": "executed|rejected", "reason": "optional", "executed_at": "..." }
  ```
- **Testing:** Create a suite of test profiles with various validation rules and test commands against them to ensure correct rejection/acceptance. Include tests for stale telemetry and sign convention.

---

## Summary

Command validation is a multi‑layered safety net that protects the EMS from invalid or unsafe user requests. By combining static limits, dynamic real‑time limits with freshness checks, and explicit sign convention management, we ensure that only feasible commands reach the control logic. Clear rejection messages provide immediate feedback to users, and the layered approach (cloud, edge, controller) ensures no single point of failure. This design is fundamental to the reliability and safety of the entire system.