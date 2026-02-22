# Audit Logging & Command Traceability

## Purpose

Audit logging and command traceability provide a complete, immutable record of every command that flows through the EMS. This capability is essential for:

- **Security investigations:** Determining who did what and when.
- **Operational debugging:** Tracing a failed command across cloud, queue, and edge.
- **Compliance:** Meeting regulatory requirements for energy grid interactions.
- **Customer support:** Answering questions like "Why didn't my battery charge at 2 PM?"
- **System forensics:** Reconstructing events leading up to an incident.

Every command must be logged at every stage of its lifecycle, with a common `command_id` serving as the correlation key across all components.

---

## Role in the Command Dispatcher Architecture

Audit logging is a cross‑cutting concern that touches every component involved in command processing. Logs are generated at key waypoints and shipped to a central system for analysis.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
│  (command initiated)                         │
└───────────────┬─────────────────────────────┘
                │ command
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
│  • Log: command_received                     │
│  • Log: validation_result                    │
│  • Log: routing_decision (online/queued)     │
└───────────────┬─────────────────────────────┘
                │ command / ack
┌───────────────▼─────────────────────────────┐
│     Redis (Queue)                            │
│  • Log: queued (if offline)                  │
│  • Log: dequeued (when delivered)            │
└───────────────┬─────────────────────────────┘
                │ command / ack
┌───────────────▼─────────────────────────────┐
│     Edge‑Side Dispatcher                     │
│  • Log: edge_received                        │
│  • Log: edge_validation                      │
│  • Log: request_channel_write                 │
│  • Log: ack_received (from controller)       │
└───────────────┬─────────────────────────────┘
                │ ack
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
│  • Log: final_ack_received                   │
│  • Log: sent_to_ui                           │
└─────────────────────────────────────────────┘
```

All logs are shipped to a central logging system (e.g., ELK stack, Datadog, Splunk) with the `command_id` as the primary correlation field.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Command ID** | A universally unique identifier (UUID) generated at command creation. It is the single most important field for tracing a command across the system. |
| **Log Entry** | A structured JSON record containing timestamp, component, event type, command ID, and relevant metadata. |
| **Correlation ID** | The `command_id` serves as the correlation ID; all log entries for a given command share the same ID. |
| **Waypoint** | A logical point in the command lifecycle where a log entry is created (e.g., received, validated, queued, forwarded, acknowledged). |
| **Log Aggregator** | A central system (e.g., Elasticsearch) that collects, indexes, and allows searching of logs. |
| **Retention Policy** | How long logs are kept (e.g., 1 year for compliance, 30 days for operational debugging). |
| **Immutability** | Logs should be append‑only and protected from tampering (e.g., using signed logs or secure storage). |
| **Store & Forward** | Edge devices must buffer logs locally and reliably upload them when connectivity is restored, preventing log loss during outages. |

---

## Functional Requirements

- Every component involved in command processing shall generate structured JSON logs for each significant event in the command lifecycle.
- All log entries shall include at minimum: `timestamp` (ISO 8601 UTC), `component` (e.g., `cloud_dispatcher`, `edge_dispatcher`, `queue`), `event_type`, `command_id`, and a human-readable `message`.
- The Cloud-Side Dispatcher shall log the following events for every command:  
  • `command_received` (with full payload)  
  • `validation_passed` / `validation_failed` (with reason)  
  • `routing_decision` (online, queued, or rejected)  
  • `final_ack_received` (with status)  
  • `sent_to_ui` (acknowledgment delivered).
- If the command is queued, the queue system (Redis) shall log:  
  • `command_queued` (with queue name and TTL)  
  • `command_dequeued` (when delivered to edge)  
  • `command_expired` (if TTL exceeded before delivery).
- The Edge-Side Dispatcher shall log:  
  • `edge_command_received`  
  • `edge_validation_passed` / `edge_validation_failed`  
  • `request_channel_written` (channel name, value)  
  • `ack_received_from_controller` (with status and reason).
- **Edge devices must implement store-and-forward logging.** Logs shall be written to a local, size-capped rotating file system. A log shipper (e.g., Filebeat, Promtail) shall reliably upload logs to the central aggregator when connectivity is available, including backfilling logs generated during offline periods.
- All logs shall be shipped to a central log aggregator in near-real-time when online.
- The log aggregator shall support searching by `command_id`, `user_id`, `edge_id`, `device_id`, and time range.
- Logs shall be retained according to a defined policy (e.g., 90 days for operational logs, 1 year for audit).
- Logs shall be protected from unauthorized access and tampering (e.g., access controls, immutable storage).
- A support dashboard or tool shall allow querying logs by `command_id` to reconstruct the full lifecycle of any command.
- Edge devices must run an NTP client to maintain time accuracy.
- **Support dashboards must account for clock drift.** When reconstructing a command's lifecycle, events should be sorted by the predefined sequence (e.g., Cloud Received → Edge Received → Controller Ack) for the same `command_id`, rather than relying exclusively on absolute timestamp ordering.

---

## Detailed Flow

### Command Lifecycle with Logging

The following sequence illustrates the log events generated for a successful command delivered to an online edge.

| Step | Component | Event Type | Logged Data |
|------|-----------|------------|-------------|
| 1 | UI | (optional) | UI may log locally, but cloud is source of truth. |
| 2 | Cloud Dispatcher | `command_received` | Full command payload, source IP (last octet masked if required), user ID. |
| 3 | Cloud Dispatcher | `validation_passed` | List of checks passed. |
| 4 | Cloud Dispatcher | `routing_decision` | Decision: `online`, target edge ID. |
| 5 | Edge Dispatcher | `edge_command_received` | Command ID, timestamp. |
| 6 | Edge Dispatcher | `edge_validation_passed` | Device and channel validated. |
| 7 | Edge Dispatcher | `request_channel_written` | Channel `ui_requests/battery_1_power` set to value. |
| 8 | Edge Dispatcher | `sent_immediate_ack` | Acknowledgment sent to cloud. |
| 9 | Cloud Dispatcher | `immediate_ack_sent` | Log that `received` ack was sent to UI. |
| 10 | Controller | `controller_decision` | (Logged by controller) – decision to execute/reject. |
| 11 | Controller | `ack_written` | Wrote to `ack/battery_1_power` with status. |
| 12 | Edge Dispatcher | `ack_received_from_controller` | Payload of ack. |
| 13 | Edge Dispatcher | `final_ack_forwarded` | Forwarded ack to cloud. |
| 14 | Cloud Dispatcher | `final_ack_received` | Payload of ack. |
| 15 | Cloud Dispatcher | `sent_to_ui` | Final ack delivered to UI. |

If the command is queued, additional events from Redis (`command_queued`, `command_dequeued`, `command_expired`) are logged.

### Log Format Example (JSON)

```json
{
  "timestamp": "2026-02-22T14:30:00.123Z",
  "component": "cloud_dispatcher",
  "event_type": "command_received",
  "command_id": "a7e3f1c8-9b2d-4f6a-8e5d-3c1b9a7f2d6e",
  "user_id": "user_456",
  "source_ip": "192.168.1.100",
  "payload": {
    "type": "setpoint",
    "target": {
      "edge_id": "site1_edge",
      "device_id": "battery_1",
      "channel": "RequestedActivePower"
    },
    "value": 50000
  },
  "message": "Command received from user user_456"
}
```

```json
{
  "timestamp": "2026-02-22T14:30:01.456Z",
  "component": "redis_queue",
  "event_type": "command_queued",
  "command_id": "a7e3f1c8-9b2d-4f6a-8e5d-3c1b9a7f2d6e",
  "queue": "queue:site1_edge",
  "ttl_seconds": 60,
  "message": "Command queued for offline edge site1_edge"
}
```

### Correlation by `command_id`

To trace a command, an operator simply searches for its `command_id` in the log aggregator. All events from all components will appear, sorted by timestamp, giving a complete picture of the command's journey. In cases of clock drift, the support dashboard should also allow sorting by the logical sequence (cloud first, then edge) for the same command ID.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Cloud‑Side Dispatcher** | Generates logs at each stage; provides command payload for logging. |
| **Edge‑Side Dispatcher** | Logs receipt, validation, request channel writes, and acknowledgments to local disk. |
| **Redis Queue** | Logs queuing, dequeuing, and expiry events. (Redis itself may not log; a wrapper or monitoring process should generate these logs.) |
| **Controllers** | Should log their decisions (executed/rejected) with reason and command ID. These logs can be written to the same aggregator via the edge dispatcher or directly. |
| **Log Shipper (Filebeat/Promtail)** | Monitors local log files on the edge and reliably uploads to central aggregator. |
| **Log Aggregator (e.g., ELK)** | Collects, indexes, and provides search UI. |
| **Authentication Service** | May log user login/logout events, which can be correlated with command logs via user ID. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Log generation overhead** | ≤ 1 ms per log | Measure time added to command processing. |
| **Log completeness** | 100% of commands generate all expected waypoint logs | Sample audit of command IDs. |
| **Edge log buffering** | No logs lost during 24‑hour offline period | Simulate outage and verify logs arrive after reconnect. |
| **Search time** | < 5 seconds to retrieve all logs for a given command ID | Query aggregator with typical load. |
| **Log retention** | Logs older than retention period are purged | Verify with automated check. |
| **Log tamper resistance** | (Optional) Logs cannot be modified after writing | Use append‑only storage with access controls. |

---

## Implementation Notes

- **Structured Logging:** Use a library like `structlog` (Python) or `pino` (Node.js) to produce JSON logs consistently.
- **Edge Log Buffering (Store & Forward):** The Edge must never stream logs directly to the cloud via HTTP in memory. All Edge logs must be written to a local, size‑capped rotating file system (e.g., using `RotatingFileHandler` in Python). A shipper like Filebeat or Promtail must monitor these files and handle the reliable uploading and backfilling of logs when internet connectivity is restored.
- **Command ID Propagation:** Ensure `command_id` is passed in all internal messages (e.g., from cloud to edge, in queue entries, in acknowledgments). Never drop it.
- **Central Schema:** Define a common JSON schema for all logs; use a linter to validate.
- **Sensitive Data:** Do not log Personal Identifiable Information (PII) beyond the `user_id`. Ensure `source_ip` logging complies with local privacy regulations (e.g., GDPR), masking the last octet if necessary. Never log full JWT tokens or API keys.
- **Log Levels:** Use INFO for normal command lifecycle events, WARN for validation failures, ERROR for unexpected errors. DEBUG for development.
- **Queue Logging:** Since Redis itself does not log, the components that interact with the queue (cloud dispatcher) should log queue operations. For example, when adding to queue, log `command_queued`; when retrieving, log `command_dequeued`. If a background cleanup process expires commands, it should log `command_expired`.
- **Controller Logging:** Controllers should have access to a logging client that can send logs with the `command_id`. They can log via the edge dispatcher or directly to the aggregator.
- **NTP Synchronization:** All edge devices must run an NTP client (e.g., `chrony` or `ntpd`) to maintain time accuracy. The logging system should monitor clock sync status and flag edges with excessive drift.
- **Support Tool:** Build a simple internal tool (or use Kibana Discover) with a pre‑built dashboard that allows support staff to enter a command ID and see the entire timeline. When displaying events, the tool should offer sorting by either timestamp or logical sequence to handle clock drift cases.

---

## Summary

Audit logging and command traceability are fundamental to operating a reliable, secure, and supportable EMS. By logging every significant event across all components, buffering logs locally on edges to prevent loss during outages, and correlating them with a common `command_id`, we enable:

- **Rapid debugging** of failed commands.
- **Security audits** to detect unauthorized access.
- **Compliance** with grid operator requirements.
- **Customer support** with precise answers.

The logs become the system's memory, providing an immutable record of every action taken. With proper aggregation, retention, and handling of edge cases like clock drift and disconnection, they are an invaluable tool for operations and development alike.