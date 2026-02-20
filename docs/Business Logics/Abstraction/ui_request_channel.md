# UI Request Channels

## Purpose
UI Request Channels are a special category of channels in the Device Abstraction Layer (DAL) that hold user‑initiated commands. They are written by the edge command receiver (upon receiving a command from the cloud) and read by controllers during each core cycle. This separation ensures that user commands are validated and integrated safely with the active control logic.

## Naming Convention
All UI request channels follow the pattern:
```
ui_requests/{device_id}_{purpose}
```
where:
- `{device_id}` is the logical device identifier (e.g., `battery_1`, `pv_2`).
- `{purpose}` describes the intended action (e.g., `power`, `mode`, `threshold`).

Examples:
- `ui_requests/battery_1_power`
- `ui_requests/peak_shaving_enable`
- `ui_requests/grid_import_limit`

## Standard Request Channels

### Battery / ESS
| Channel Name | Type | Description | Used By |
|--------------|------|-------------|---------|
| `ui_requests/{battery_id}_power` | numeric (W) | Desired charge/discharge power (+ = charge, – = discharge). | Balancing controller, Peak shaving controller, Manual override controller. |
| `ui_requests/{battery_id}_mode` | enum | Override mode: `"AUTO"`, `"FORCE_CHARGE"`, `"FORCE_DISCHARGE"`, `"IDLE"`. | Manual override controller. |
| `ui_requests/{battery_id}_min_soc` | numeric (%) | Temporary override of minimum SOC limit. | (Future) |
| `ui_requests/{battery_id}_max_soc` | numeric (%) | Temporary override of maximum SOC limit. | (Future) |

### Grid
| Channel Name | Type | Description | Used By |
|--------------|------|-------------|---------|
| `ui_requests/grid_import_limit` | numeric (W) | Temporary peak shaving threshold override. | Peak shaving controller. |
| `ui_requests/grid_export_limit` | numeric (W) | Temporary export limit (zero‑export target). | Zero‑export controller. |

### PV Inverter
| Channel Name | Type | Description | Used By |
|--------------|------|-------------|---------|
| `ui_requests/{pv_id}_curtailment` | numeric (W or %) | Desired curtailment limit. | Curtailment controller. |
| `ui_requests/{pv_id}_enable` | boolean | Enable/disable inverter (if supported). | (Future) |

### Controllers
| Channel Name | Type | Description | Used By |
|--------------|------|-------------|---------|
| `ui_requests/peak_shaving_enable` | boolean | Enable/disable peak shaving controller. | Scheduler / controller manager. |
| `ui_requests/balancing_enable` | boolean | Enable/disable self‑consumption (balancing) controller. | Scheduler. |
| `ui_requests/manual_override_enable` | boolean | Enable manual override mode (disables automatic controllers). | Scheduler. |

### System
| Channel Name | Type | Description | Used By |
|--------------|------|-------------|---------|
| `ui_requests/edge_restart` | boolean | Request edge restart (triggers on rising edge). | System controller. |
| `ui_requests/config_sync` | boolean | Force immediate configuration sync from cloud. | Edge config sync service. |

## Relationship with Device Profiles

Each request channel corresponds to a **writeable channel** defined in a device profile. For example, the ESS profile contains:

```json
{
  "RequestedActivePower": {
    "type": "numeric",
    "unit": "W",
    "access": "write",
    "validation": { ... }
  }
}
```

When a device instance `battery_1` is created using this profile, the DAL automatically creates the channel `ui_requests/battery_1_power` (the exact mapping may be configurable). The validation rules from the profile are used by the cloud dispatcher to pre‑validate commands.

## Expiry and Safety

Request channels are **not persistent** – they hold the latest value written by the command receiver. If no new command is received, the channel retains its last value indefinitely, but controllers may implement their own timeouts (e.g., ignore requests older than X seconds) to prevent stale overrides. This is complementary to the command‑level TTL enforced by the cloud dispatcher.

## Acknowledgment Channels

For richer feedback, you may introduce **acknowledgment channels** (e.g., `ui_requests/battery_1_power_ack`) where controllers can write back the status of the last request (accepted/rejected with reason). These can be read by the command dispatcher to send final acknowledgments to the cloud.


## Cross‑Reference in Existing Documents

- In the **ESS Abstraction** document, add a note that the `RequestedActivePower` channel in the profile maps to `ui_requests/battery_1_power` in the DAL.
- In the **Command Dispatcher** document, reference the new UI Request Channels document as the authoritative list of channels that can be targeted.
- In the **Device Profile Schema** document, explain how writeable channels in profiles become request channels in the DAL.

---