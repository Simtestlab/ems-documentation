# Command Payload Schemas (API Contract)

## Purpose

The **Command Payload Schemas** define the exact JSON structure for every type of command that can be sent from the UI (or automated cloud service) to the EMS edge. This document serves as the authoritative API contract for all command‑related communication over WebSocket.

A consistent schema ensures:
- Clear expectations for frontend developers building the UI.
- Robust validation on the cloud side.
- Error‑free parsing and routing on the edge.

All commands share a common envelope and are distinguished by the `type` field. Each command type has its own specific required fields.

**Note for Backend:** The `type` field should be used as a **discriminator** in Pydantic to safely parse the polymorphic `value` field into the correct schema.

---

## Role in the Command Dispatcher Architecture

The command payload originates in the UI (or a cloud automation service) and is sent over WebSocket to the Cloud‑Side Dispatcher. After validation, it is forwarded to the edge (or queued). The schema must be understood by all parties.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
│  (sends command JSON via WebSocket)          │
└───────────────┬─────────────────────────────┘
                │ payload
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
│  • Validates structure against schema        │
│  • Checks device profile                      │
│  • Forwards / queues                          │
└───────────────┬─────────────────────────────┘
                │ payload (unchanged)
┌───────────────▼─────────────────────────────┐
│     Edge‑Side Dispatcher                     │
│  • Parses and writes to request channels     │
└─────────────────────────────────────────────┘
```

---

## Common Envelope Fields

Every command, regardless of type, must include these top‑level fields:

- **`command_id`** (string, required): Unique identifier for the command (UUID recommended). Used for idempotency, logging, and correlation.
- **`type`** (string, required): One of the command types listed in Section 4.
- **`target`** (object, required): Identifies the destination edge and device. Contains `edge_id` and `device_id`. For setpoint commands, also includes `channel`.
- **`timestamp`** (string, required): ISO 8601 timestamp (with milliseconds) **strictly in UTC (must end in 'Z')**. Used for replay protection and staleness checks.
- **`expiry_sec`** (number, required): Time-to-live in seconds. Determines how long the command remains valid if queued. Must respect per-type limits enforced by the cloud.
- **`source`** (string, required): Identifier of the user or service that issued the command (e.g., `user_456`). Used for audit logging.
- **`value`** (varies, optional): The payload of the command; structure depends on `type`. **Can be `null` to clear an active request and revert the channel to its default/automatic state.**

**Example common envelope:**
```json
{
  "command_id": "cmd_12345",
  "type": "setpoint",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "battery_1",
    "channel": "RequestedActivePower"
  },
  "timestamp": "2026-02-22T10:30:00.123Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": 50000
}
```

---

## Command Types and Their Value Schemas

### `setpoint` – Write a numeric value to a device channel

Used for commands like setting battery power, charge current, etc.

**Target Requirements:**
- `edge_id`: string
- `device_id`: string
- `channel`: string – must be a writable channel in the device profile (e.g., `RequestedActivePower`).

**Value:** number (integer or float) or `null`.  
- A numeric value sets the desired setpoint.  
- `null` clears any active setpoint and returns control to automatic controllers.

**Example:**
```json
{
  "command_id": "cmd_12345",
  "type": "setpoint",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "battery_1",
    "channel": "RequestedActivePower"
  },
  "timestamp": "2026-02-22T10:30:00.123Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": 50000
}
```

### `mode_change` – Change the operational mode of a device or controller

Used for enabling/disabling controllers, switching battery modes (charge/discharge/idle), etc.

**Target Requirements:**
- `edge_id`: string
- `device_id`: string (may be a controller ID or device ID)
- `channel`: string – the mode channel (e.g., `RequestedMode`).

**Value:** string, boolean, or `null`.  
- String/boolean sets the desired mode (must be one of the allowed values in the profile).  
- `null` clears the override and reverts to automatic control.

**Example (battery mode):**
```json
{
  "command_id": "cmd_12346",
  "type": "mode_change",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "battery_1",
    "channel": "RequestedMode"
  },
  "timestamp": "2026-02-22T10:31:00.123Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": "FORCE_CHARGE"
}
```

**Example (controller enable/disable):**
```json
{
  "command_id": "cmd_12347",
  "type": "mode_change",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "controller_peak_shaving",
    "channel": "Enable"
  },
  "timestamp": "2026-02-22T10:32:00.123Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": true
}
```

### `config_override` – Temporarily change a configuration parameter

Used for overriding thresholds, limits, or other configurable parameters without a full config sync.

**Target Requirements:**
- `edge_id`: string
- `device_id`: string (or `"system"` for edge‑wide settings)
- `channel`: string – the configuration parameter name.

**Value:** varies – the new value for the parameter. Type depends on the parameter. `null` clears the override and reverts to the default value.

**Example (peak shaving threshold override):**
```json
{
  "command_id": "cmd_12348",
  "type": "config_override",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "controller_peak_shaving",
    "channel": "Threshold"
  },
  "timestamp": "2026-02-22T10:33:00.123Z",
  "expiry_sec": 3600,
  "source": "user_456",
  "value": 40000
}
```

### `system` – Edge lifecycle commands

Used for restarting the edge, forcing a config sync, etc.

**Target Requirements:**
- `edge_id`: string
- `device_id` may be omitted or set to `"system"`.

**Value:** object with a required `action` field and optional parameters. `null` is not typically used.

| Sub‑field | Type | Required | Description |
|-----------|------|----------|-------------|
| `action` | string | ✅ | The system action: `"restart"`, `"sync_config"`, `"update_firmware"`. |
| `parameters` | object | ❌ | Additional parameters (e.g., firmware URL). |

**Example (restart):**
```json
{
  "command_id": "cmd_12349",
  "type": "system",
  "target": {
    "edge_id": "site1_edge"
  },
  "timestamp": "2026-02-22T10:34:00.123Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": {
    "action": "restart"
  }
}
```

### `schedule_update` – Update a scheduled tariff or forecast

Used by cloud automation to push new schedules (e.g., time‑of‑use tariffs, forecast data) to the edge.

**Target Requirements:**
- `edge_id`: string
- `device_id` may be set to `"scheduler"`.

**Value:** object containing the schedule data. Structure depends on schedule type. `null` clears the schedule and reverts to defaults.

**Example (TOU tariff update):**
```json
{
  "command_id": "cmd_12350",
  "type": "schedule_update",
  "target": {
    "edge_id": "site1_edge"
  },
  "timestamp": "2026-02-22T10:35:00.123Z",
  "expiry_sec": 86400,
  "source": "automation_service",
  "value": {
    "schedule_type": "tou_tariff",
    "data": [
      {"start": "00:00", "end": "06:00", "rate": 0.05},
      {"start": "06:00", "end": "18:00", "rate": 0.12},
      {"start": "18:00", "end": "24:00", "rate": 0.08}
    ]
  }
}
```

---

## Validation Rules Summary

The cloud dispatcher enforces the following validation rules for all commands:

1. **Required fields:** `command_id`, `type`, `target`, `timestamp`, `expiry_sec`, `source` must be present.
2. **Command ID uniqueness:** Not enforced globally, but used for replay detection (within 60‑second window).
3. **Timestamp freshness:** `abs(server_time - payload.timestamp)` must be ≤ 60 seconds; otherwise reject.
4. **Expiry limits:** Cloud may enforce maximum allowed `expiry_sec` per command type (e.g., setpoint ≤ 60s, schedule ≤ 24h).
5. **Target validation:** `edge_id` and `device_id` must exist and be accessible to the user.
6. **Channel validation (for setpoint/mode_change/config_override):** The channel must be writable according to the device profile.
7. **Value validation:** Must satisfy static/dynamic limits as defined in the device profile. `null` is always allowed for clearing requests.

If any validation fails, the cloud sends an immediate rejection acknowledgment with a `reason` field.

---

## Example: Complete Command with All Fields

```json
{
  "command_id": "a7e3f1c8-9b2d-4f6a-8e5d-3c1b9a7f2d6e",
  "type": "setpoint",
  "target": {
    "edge_id": "site1_edge",
    "device_id": "battery_1",
    "channel": "RequestedActivePower"
  },
  "timestamp": "2026-02-22T14:30:00.000Z",
  "expiry_sec": 60,
  "source": "user_456",
  "value": -25000
}
```

---

## Error Responses (Immediate Rejection)

When a command fails validation, the cloud sends an error message over the same WebSocket connection:

```json
{
  "command_id": "a7e3f1c8-9b2d-4f6a-8e5d-3c1b9a7f2d6e",
  "status": "rejected",
  "reason": "Channel 'RequestedActivePower' not writable for device 'battery_1'"
}
```

---

## Summary

This document defines the contract between the UI and the EMS cloud for all commands. By adhering to these schemas, frontend and backend teams can develop independently with confidence that their messages will be correctly interpreted and validated. For any new command type, follow the same pattern: a clear `type` identifier, a well‑defined `value` structure, and appropriate validation rules.