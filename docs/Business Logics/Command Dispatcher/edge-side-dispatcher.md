# Edge‑Side Dispatcher Service
---

## Purpose

The **Edge‑Side Dispatcher Service** is the component running on each edge gateway (e.g., Raspberry Pi) that receives commands from the cloud, validates them, and safely injects them into the local control system. It acts as the bridge between the cloud and the Device Abstraction Layer, ensuring that all user commands are handled reliably and without bypassing the active control logic.

Its main responsibilities are:

- Maintaining a persistent WebSocket connection to the cloud.
- Authenticating the edge via a secure first‑message handshake.
- Receiving commands and checking them against locally cached device profiles.
- Writing valid commands to **request channels** in the DAL (e.g., `ui_requests/battery_1_power`).
- Sending immediate acknowledgments, and forwarding **final execution status** from the dedicated `ack/` channels in the DAL.
- Handling connection loss and reconnection with automatic resumption of command delivery.
- Ensuring idempotency (duplicate commands are ignored via a fixed‑size cache).

This service is critical for safe operation: it never writes directly to output channels, leaving final execution to the controllers in the Core Cycle.

---

## Role

The Edge‑Side Dispatcher is the counterpart to the Cloud‑Side Dispatcher, residing on each edge gateway.

```
┌─────────────────────────────────────────────┐
│           Cloud‑Side Dispatcher             │
│  (Cloud)                                    │
└───────────────┬─────────────────────────────┘
                │ Persistent WebSocket
┌───────────────▼─────────────────────────────┐
│     EDGE‑SIDE DISPATCHER SERVICE            │ ← You are here
│  • WebSocket client (auto‑reconnect)        │
│  • Authentication (first‑message handshake) │
│  • Command reception & validation            │
│  • Idempotency cache (fixed‑size deque)     │
│  • Writes to request channels                │
│  • Subscribes to ack/ channels               │
│  • Forwards acknowledgments to cloud         │
└───────────────┬─────────────────────────────┘
                │ writes to request channels
                │ reads from ack/ channels
┌───────────────▼─────────────────────────────┐
│      Device Abstraction Layer (DAL)         │
│  • ui_requests/... channels                  │
│  • ack/... channels (written by controllers) │
│  • output channels (SetActivePower, etc.)   │
└───────────────┬─────────────────────────────┘
                │ read by controllers, written by controllers
┌───────────────▼─────────────────────────────┐
│           Edge Core Cycle                    │
│  (Controllers: Peak Shaving, Balancing,     │
│   Manual Override)                           │
└─────────────────────────────────────────────┘
```

---

## Key Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **WebSocket Client** | Maintains a persistent connection to the cloud (wss://...). Handles reconnection with exponential backoff. |
| **Authentication** | Upon connection, sends an authentication frame with `edge_id` and API key. No commands are processed until authentication succeeds. |
| **Command Reception** | Listens for incoming command JSON messages. |
| **Idempotency** | Maintains a fixed‑size rolling cache of recently processed `command_id`s to silently drop duplicates. |
| **Local Validation** | Checks that the target device exists and that the requested channel is a valid request channel (using locally cached device profiles). |
| **Request Channel Write** | Writes the command value to the corresponding request channel in DAL (e.g., `ui_requests/battery_1_power`). |
| **Acknowledgment Subscription** | Subscribes to (or polls) the `ack/` namespace in DAL to receive final execution status from controllers. |
| **Acknowledgment Forwarding** | Forwards acknowledgment messages from `ack/` channels to the cloud. |
| **Connection Monitoring** | Detects disconnection and automatically reconnects; resends any pending acknowledgments if needed. |
| **Local Logging** | Logs all commands and outcomes for audit and debugging. |

---

## Functional Requirements
- Maintain a persistent WebSocket connection to the cloud endpoint `wss://api.ems.example/v1/edges/ws`. 
- Upon connection, immediately send an authentication frame: `{"action": "authenticate", "edge_id": "<edge_id>", "token": "<api_key>"}`. The cloud will not send any commands until this frame is validated.
-  Implement automatic reconnection with exponential backoff (initial delay 1s, max 5 minutes).
- On receipt of a command, validate:
	- The target device exists in the local device registry.
	- The channel specified is a known request channel (e.g., `ui_requests/...`). Use locally cached device profiles.
- Maintain an in‑memory fixed‑size cache (e.g., a deque of size 50) of recent `command_id`s to detect and silently drop duplicate deliveries. The size automatically evicts old entries; no separate eviction timer is needed.
- If validation passes, write the value to the appropriate request channel in DAL using `DAL.set_channel()`. Include the original timestamp from the command.
-  Immediately after writing to the request channel, send an **immediate acknowledgment** to the cloud: `{"command_id": "...", "status": "received"}`.
- Subscribe to (or periodically poll) the `ack/` namespace in DAL. When a controller writes a final status to `ack/{request_channel_name}` (e.g., `ack/battery_1_power`), forward that message as the **final acknowledgment** to the cloud.
- Log every command (command_id, payload, result) locally. Logs should be structured and include timestamps.
- On disconnection, stop processing incoming commands (none arrive) but continue normal operation. On reconnection, re‑authenticate and resume; do not replay old commands.

---

## Command Flow

### Connection Establishment and Authentication
- Edge service starts and loads its configuration (edge_id, API key, cloud URL).
- Connects to `wss://api.ems.example/v1/edges/ws` (without any query parameters).
- Immediately sends an authentication frame:
  ```json
  {
    "action": "authenticate",
    "edge_id": "site1_edge",
    "token": "sk_live_abc123..."
  }
  ```
- The cloud validates the API key and responds with an authentication acknowledgment. If invalid, the cloud closes the connection.
- Once authenticated, the cloud may begin sending commands.

### Command Reception
- Edge listens for incoming WebSocket messages (JSON).
- Example command payload:
  ```json
  {
    "command_id": "cmd_12345",
    "type": "setpoint",
    "target": {
      "edge_id": "site1_edge",
      "device_id": "battery_1",
      "channel": "RequestedActivePower",
      "value": 50000,
      "unit": "W"
    },
    "expiry_sec": 60,
    "source": "user_456",
    "timestamp": "2026-02-20T10:30:00Z"
  }
  ```

### Idempotency Check
- Edge checks if `command_id` exists in the fixed‑size cache (e.g., deque of size 50).
- If found, the message is silently dropped (no acknowledgment sent).
- If not found, the `command_id` is added to the cache (pushing out the oldest if full).

### Local Validation
- Using the locally cached device profile for `battery_1` (synced via Edge Config Sync):
  - Verify that the device `battery_1` exists.
  - Verify that `RequestedActivePower` is a defined request channel (mapped to `ui_requests/battery_1_power`).
- If validation fails, send a **rejection acknowledgment** immediately:
  ```json
  { "command_id": "cmd_12345", "status": "rejected", "reason": "Device or channel not found" }
  ```
  and stop processing this command.

### Write to Request Channel
- Edge calls `DAL.set_channel("ui_requests/battery_1_power", 50000, command_timestamp)`.
- The request channel now holds the user's desired value.

### Immediate Acknowledgment
- Edge sends back to cloud:
  ```json
  { "command_id": "cmd_12345", "status": "received" }
  ```

### Core Cycle Processing and Acknowledgment Channels
- The Edge Core Cycle runs every second, taking a snapshot that includes all request channels.
- Controllers (e.g., `ManualOverrideController`) read `ui_requests/battery_1_power` and decide whether to apply it.
- After making a decision, the controller **must** write the final outcome to a corresponding **acknowledgment channel** in the DAL, e.g., `ack/battery_1_power`. The payload should include the command_id, status, and optional reason.
  ```python
  # Inside controller
  decision = {
      "command_id": "cmd_12345",
      "status": "executed",  # or "rejected"
      "reason": "Battery SOC too low",  # if rejected
      "executed_at": now_iso
  }
  DAL.set_channel("ack/battery_1_power", decision, now)
  ```

### Acknowledgment Forwarding
- The Edge‑Side Dispatcher subscribes to changes on all `ack/` channels (or polls them periodically).
- When a new value appears on `ack/battery_1_power`, it reads the decision payload and forwards it to the cloud as the **final acknowledgment**.
- Example final acknowledgment sent to cloud:
  ```json
  {
    "command_id": "cmd_12345",
    "status": "executed",
    "executed_at": "2026-02-20T10:30:01.050Z"
  }
  ```
- The dispatcher does **not** use timers or guesswork; it simply relays the controller's decision.

### Disconnection and Reconnection
- If the WebSocket connection drops, the edge attempts to reconnect with exponential backoff.
- During disconnection, incoming commands are impossible; the edge continues normal operation (controllers run, but no new user commands).
- On reconnection, the edge re‑authenticates (first message). The cloud may resend any queued commands that were not yet acknowledged.
- The edge does **not** replay old commands or acknowledgments; the cloud's in‑flight watchdog and queue system ensure reliable delivery.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Device Abstraction Layer (DAL)** | Writes command values to request channels (e.g., `ui_requests/battery_1_power`). Reads acknowledgment channels (`ack/battery_1_power`). |
| **Device Profiles (local cache)** | Used to validate that target device exists and that the requested channel is a valid request channel. Profiles are synced via Edge Config Sync. |
| **Edge Core Cycle & Controllers** | Controllers read request channels and decide whether to apply them. They must write the final decision to the corresponding `ack/` channel. |
| **Edge Configuration Sync** | Provides fresh device profiles and ensures the edge has the latest channel definitions. |
| **Local Logging** | Logs all commands for audit. Logs should be structured (JSON) and include `command_id` for correlation. |

---

## Implementation Notes (MVP)

- **WebSocket Client:** Use Python's `websockets` library with `asyncio`. Implement reconnection logic with exponential backoff (e.g., using `asyncio.sleep` and a loop).
- **Authentication:** Send the authentication frame immediately after connection. The cloud will not process any commands until this is received and validated.
- **Idempotency Cache:** Use a `collections.deque` of fixed size (e.g., 50). When a new `command_id` arrives, check if it exists; if not, append to the deque. When the deque reaches max length, the oldest entry is automatically evicted. No separate eviction timer is needed.
- **Local Validation:** Load device profiles into memory at startup. For each command, look up the device and channel. Reject if not found.
- **DAL Integration:** The DAL should provide a thread‑safe `set_channel(name, value, timestamp)` method and a mechanism to subscribe to changes on channels (e.g., via callbacks or polling). The edge dispatcher can maintain a dictionary of the last known value of each `ack/` channel to detect changes.
- **Acknowledgment Subscription:** The simplest approach is to poll the `ack/` channels every second (or each core cycle) and compare with the previous value. If changed, forward the new value to the cloud. More efficient is to use a pub/sub mechanism within the DAL.
- **Logging:** Use structured logging (e.g., `import structlog`) with fields: `command_id`, `edge_id`, `device_id`, `channel`, `value`, `status`, `timestamp`. Ship logs to a central system (e.g., via syslog, filebeat) for traceability.
- **Testing:** Simulate disconnections, duplicate commands, invalid devices to ensure robust handling. Also verify that acknowledgments from controllers are correctly forwarded.

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Connection uptime** | ≥ 99.5% of time (excluding planned maintenance) | Track connection state and disconnection events. |
| **Command processing latency** | < 100 ms (from receipt to writing to DAL) | Log timestamps at receipt and after DAL write. |
| **Idempotency effectiveness** | 100% of duplicate commands dropped | Test with retransmission of same command_id. |
| **Validation accuracy** | 100% of invalid commands rejected with clear reason | Unit tests against mock profiles. |
| **Acknowledgment forwarding** | 100% of controller decisions forwarded to cloud | Monitor that each ack channel update results in a cloud message. |
| **Reconnection time** | ≤ 30 seconds after network restore | Simulate outage and measure. |

---

## Summary

The Edge‑Side Dispatcher Service is the critical link between cloud commands and local control. By writing commands to request channels instead of directly to outputs, it ensures that all user requests are subject to the same safety checks and control logic as automated decisions. The use of dedicated `ack/` channels allows controllers to report final execution status accurately, without dangerous timers or guesswork. The fixed‑size idempotency cache prevents duplicate processing, and secure first‑message authentication protects API keys from exposure in logs. With robust reconnection and local validation, this service maintains reliability even under adverse network conditions.

For a complete understanding, refer to:
- [Cloud‑Side Dispatcher Service](./cloud-side-dispatcher.md)
- [Request Channels](./request-channels.md)
- [Acknowledgment Channels](./ack-channels.md)
- [Device Abstraction Layer (DAL)](../Abstraction/device_abstraction_layer.md)