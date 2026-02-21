
# Cloud‑Side Dispatcher Service

## Purpose

The **Cloud‑Side Dispatcher Service** is the entry point for all user‑initiated commands in the EMS. It runs in the cloud (typically as a FastAPI service) and is responsible for:

- Accepting commands from authenticated UI clients via WebSocket.

- Validating commands against device profiles (access rights, static limits, dynamic limits).

- Routing commands to the appropriate edge device if online.
- Queuing commands with time‑to‑live (TTL) for offline edges, using Redis sorted sets.
- Providing immediate feedback to the UI on command acceptance or rejection.
- Broadcasting final command status (executed, rejected, failed, timeout) back to the UI.
- Monitoring in‑flight commands with a mandatory watchdog to prevent UI hangs.

This service ensures that every command is validated before it ever reaches the edge, reducing network traffic and protecting edge devices from invalid or dangerous requests.

---

## Role in the Command Dispatcher Architecture

The Cloud‑Side Dispatcher sits between the UI and the edge, acting as the first line of validation and routing.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
│  (Next.js, mobile app, etc.)                 │
└───────────────┬─────────────────────────────┘
                │ WebSocket / WSS
┌───────────────▼─────────────────────────────┐
│     CLOUD‑SIDE DISPATCHER SERVICE           │ ← You are here
│  • WebSocket server for UI connections       │
│  • Authentication & permission checks        │
│  • Device profile validation                  │
│  • Static & dynamic limit checks              │
│  • Edge connection registry                   │
│  • Offline queue (Redis Sorted Set) with TTL │
│  • In‑flight watchdog (mandatory)             │
└───────────────┬─────────────────────────────┘
                │ Persistent WebSocket (edge)
                │ or Redis queue (offline)
┌───────────────▼─────────────────────────────┐
│           Edge Gateway                       │
│  (Edge‑Side Receiver)                        │
└─────────────────────────────────────────────┘
```

---

## Key Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **WebSocket Server** | Maintains persistent WebSocket connections to authenticated UI clients. Authentication is handled via query string token or first‑message authentication (since browsers do not support custom headers). |
| **Authentication** | Validates user identity (JWT) and permissions (`can_control` for target edge). |
| **Device Profile Validation** | Fetches the device profile for the target device and checks: • Channel exists and is writable. • Value within static limits (`static_min`/`static_max`). • Value is in allowed enum list (if applicable). |
| **Dynamic Limit Validation** | For channels with `dynamic_min_channel` or `dynamic_max_channel`, queries the latest telemetry values (from TimescaleDB or a Redis cache) and computes effective limits. Rejects command if value exceeds current hardware limits (e.g., `MaxChargePower`). |
| **Edge Connection Registry** | Maintains a real‑time registry of online edges (mapping `edge_id` → active WebSocket connection). Used for immediate routing. |
| **Offline Queue (Redis Sorted Set)** | Stores commands for offline edges in a Redis sorted set, with score set to absolute expiration timestamp. Delivers only non‑expired commands when edge reconnects; expired commands are automatically discarded. |
| **In‑Flight Watchdog** | After forwarding a command to an online edge, places the `command_id` in a temporary Redis hash with a 5‑second TTL. A background process monitors this hash; if no final acknowledgment is received within TTL, it sends a `failed: edge_timeout` to the UI. |
| **Acknowledgment Relay** | Forwards immediate and final acknowledgments from the edge back to the UI. |
| **Timeout Handling** | Monitors queued commands for expiry; when TTL exceeded, discards command and sends `failed: timeout` to UI. |
| **Rate Limiting** | Enforces rate limits per user/edge to prevent abuse (e.g., max 10 commands per second). |
| **Audit Logging** | Logs all commands (who, what, when, result) for traceability and debugging. |

---

## Functional Requirements

- Maintain persistent WebSocket server for UI connections (WSS, TLS 1.3).
- Authenticate each UI connection. Because browser WebSocket API does not support custom headers, authentication must be performed via one of: 
	- Query string parameter: `wss://api.ems.example/v1/commands/ws?token=<jwt>` 
	- Or a first‑message authentication: client must send `{"action": "authenticate", "token": "<jwt>"}` as the first message.
- For each incoming command, validate that the user has `can_control` permission for the target edge/site (via Django auth).
- Fetch device profile for target device from configuration repository; reject if device or profile not found.
- Validate command against device profile: 
	- Channel must exist and have `access: "write"`. 
	- If numeric, value must satisfy `static_min`/`static_max`. 
	- If enum, value must be in `allowed_values`. 
	- Reject with clear error message if any check fails.

- For channels with dynamic limits (`dynamic_min_channel`/`dynamic_max_channel`), query latest telemetry values for those channels from database or cache; compute effective limit as `max(static_min, dynamic_min)` and `min(static_max, dynamic_max)`; reject if value outside effective range.

- Maintain a registry of online edges (keyed by `edge_id`) with their active WebSocket connections.
- If edge is online, forward command immediately via its WebSocket connection.

- If edge is offline, store command in a Redis sorted set (`queue:edge_id`) with score = absolute expiration timestamp (current time + TTL). TTL based on command type: 
	- Power setpoints/manual overrides: TTL ≤ 60 seconds. 
	- Configuration updates/schedules: TTL ≤ 24 hours. 
	- System commands (restart, sync): immediate discard (offline = fail).
- When edge reconnects, query sorted set for non‑expired commands: `ZRANGEBYSCORE queue:edge_id <current_timestamp> +inf`. Deliver them in score order (FIFO). After delivery, delete delivered items with `ZREMRANGEBYSCORE`. Expired commands (score < current_timestamp) are automatically removed by a background process.
- Implement an **in‑flight watchdog**:
	- After forwarding a command to an online edge, store `command_id` in a Redis hash `in_flight:<command_id>` with TTL = 5 seconds. If final acknowledgment is received before TTL, delete the hash. If TTL expires, broadcast `{"command_id": "...", "status": "failed", "reason": "edge_timeout"}` to the UI.
- Relay acknowledgments from edge back to UI: 
	- Immediate `received` acknowledgment.
	- Final `executed`/`rejected`/`failed` acknowledgment.
- Enforce rate limiting: max 10 commands per second per user or per edge; return `429 Too Many Requests` when exceeded.
- Log every command with full payload, `command_id`, user, target, timestamp, and final status. Logs must be queryable by `command_id` for traceability.

---

## Command Flow

### Connection Establishment
- UI client connects to `wss://api.ems.example/v1/commands/ws?token=<jwt>` (or sends authentication message as first frame).
- Server validates JWT; if invalid, closes connection with close code 4001.
- Server maintains the connection, listening for incoming command messages.

### Command Reception
- UI sends a JSON payload (see [Topic 10: Command Payload Schemas](./command-payload-schemas.md) for exact format).
- Example:
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

### Validation Steps
1. **Authentication & Permission** – Verify user has `can_control` for `site1_edge`.
2. **Device Profile Lookup** – Fetch profile for `battery_1`. If missing, reject.
3. **Channel Validation** – Check that `RequestedActivePower` exists and is writable.
4. **Static Limit Validation** – If numeric, check `static_min`/`static_max`. If enum, check `allowed_values`.
5. **Dynamic Limit Validation** – If dynamic limits defined, query latest telemetry for `MaxChargePower` and `MaxDischargePower`. Compute effective max charge = min(static_max, current `MaxChargePower`). Reject if 50kW exceeds effective limit.

### Routing Decision
- Look up `site1_edge` in connection registry.
- **If online:** Forward command to edge via its WebSocket connection. Then place `command_id` in the in‑flight watchdog hash with a 5‑second TTL.
- **If offline:** Push command to Redis sorted set `queue:site1_edge` with score = current_time + `expiry_sec`. Return immediate acknowledgment to UI: `{"command_id": "cmd_12345", "status": "queued"}`.

### Offline Queue Processing (Redis Sorted Set)
- When edge reconnects, the Edge‑Side Receiver sends an `edge_online` signal.
- Cloud dispatcher queries sorted set: `ZRANGEBYSCORE queue:site1_edge <current_timestamp> +inf` to get all non‑expired commands.
- For each command (in score order, i.e., FIFO), forward to edge.
- After successful forwarding, delete the command with `ZREMRANGEBYSCORE queue:site1_edge score score` (or remove all delivered in batch).
- Expired commands (score < current_timestamp) are left in the set; a periodic background task can clean them using `ZREMRANGEBYSCORE queue:site1_edge -inf <current_timestamp>`.

### In‑Flight Watchdog
- When a command is forwarded to an online edge, the dispatcher stores `command_id` in a Redis hash `in_flight:<command_id>` with a TTL of 5 seconds.
- A background worker (or Redis keyspace notifications) monitors these hashes.
- If a final acknowledgment is received before TTL expiry, the hash is deleted.
- If the TTL expires without acknowledgment, the dispatcher sends a final status to the UI:
  ```json
  {
    "command_id": "cmd_12345",
    "status": "failed",
    "reason": "edge_timeout"
  }
  ```

### Acknowledgment Handling
- Edge sends back two types of acknowledgments:
  - **Immediate:** `{"command_id": "cmd_12345", "status": "received"}` – relay to UI.
  - **Final:** `{"command_id": "cmd_12345", "status": "executed", "executed_at": "..."}` – relay to UI and delete the in‑flight hash if present.
- If edge sends `rejected` with a reason, relay that to UI and delete in‑flight hash.

### Timeout Handling
- A background process periodically scans offline queues and removes expired commands (using `ZREMRANGEBYSCORE`). For each expired command, send a `failed: timeout` status to the UI if the user who issued it is still connected; otherwise log and discard.

---

## API Contract (WebSocket)

### Connection Endpoint
```
wss://api.ems.example/v1/commands/ws
```

### Authentication
Since browser WebSocket API does not support custom HTTP headers, one of the following methods must be used:

- **Query String Parameter:** Connect with token in URL:  
  `wss://api.ems.example/v1/commands/ws?token=<jwt>`
- **First‑Message Authentication:** Connect without token, then immediately send a JSON message:
  ```json
  { "action": "authenticate", "token": "<jwt>" }
  ```
  The server will close the connection if authentication fails.

Connections without valid authentication are closed with close code 4001.

### Message Format (Client → Server)
All messages from UI to cloud must be valid JSON objects with at least:
- `command_id` (string, UUID recommended)
- `type` (string) – one of: `"setpoint"`, `"mode_change"`, `"config_override"`, `"system"`, `"schedule_update"`
- `target` (object) – contains `edge_id`, `device_id`, `channel` (for setpoints)
- `value` (optional) – depends on type
- `expiry_sec` (number) – TTL in seconds (max allowed enforced by server)
- `timestamp` (string) – ISO 8601 timestamp

### Message Format (Server → Client)
Server sends two types of messages:

**Immediate Acknowledgment:**
```json
{
  "command_id": "cmd_12345",
  "status": "received" | "queued" | "rejected",
  "reason": "optional rejection reason"
}
```

**Final Acknowledgment:**
```json
{
  "command_id": "cmd_12345",
  "status": "executed" | "rejected" | "failed",
  "reason": "optional reason for rejection/failure",
  "executed_at": "2026-02-20T10:30:01.050Z"
}
```

**Timeout / Failure Notifications:**
```json
{
  "command_id": "cmd_12345",
  "status": "failed",
  "reason": "timeout" | "edge_timeout"
}
```

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Authentication Service (Django)** | Provides user validation and permission checks. Cloud dispatcher queries Django (or validates JWT directly) to confirm `can_control` permission. |
| **Device Profile Repository** | Stores device profiles (JSON). Cloud dispatcher fetches profiles by `device_id` and caches them (with TTL) for performance. |
| **Telemetry Database (TimescaleDB)** | Provides latest values for dynamic limit channels (e.g., `MaxChargePower`). Cloud dispatcher queries this before forwarding commands with dynamic limits. May use a Redis cache for low‑latency access. |
| **Edge Connection Registry** | In‑memory registry (backed by Redis for multi‑instance support) mapping `edge_id` to active WebSocket connection objects. Updated when edges connect/disconnect. |
| **Redis** | Used for: • Offline queues (sorted sets) • In‑flight watchdog (hashes with TTL) • Rate limiting counters • Connection registry backing (pub/sub) |
| **UI / Dashboard** | The ultimate consumer of acknowledgments and timeouts. UI must handle all statuses appropriately. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Command ingestion rate** | ≥ 1000 commands/second | Load test with synthetic traffic |
| **Validation latency** | < 50 ms (p99) | Time from receipt to validation completion |
| **Dynamic limit query latency** | < 100 ms (p99) | Time to fetch latest telemetry from cache/db |
| **Offline queue capacity** | ≥ 10,000 commands per edge | Redis sorted set limits |
| **TTL enforcement** | 100% of expired commands discarded and notified | Test with long offline period |
| **In‑flight watchdog** | 100% of commands without final ack generate timeout within 5 seconds | Simulated edge failure |
| **Acknowledgment relay latency** | < 50 ms (p99) | Time from edge ack to UI delivery |
| **Availability** | 99.9% uptime | Cloud monitoring |

---

## Implementation Notes

- **WebSocket Server:** FastAPI with `WebSocket` endpoint; use `redis.pub/sub` to broadcast acknowledgments in multi‑instance deployments.
- **Authentication:** Validate JWT using `python-jose` and public key from Django. For query‑string token, parse from URL.
- **Device Profile Cache:** Redis cache with 5‑minute TTL to avoid hitting database every time.
- **Dynamic Limits:** For MVP, query latest telemetry from TimescaleDB directly; later, implement a Redis cache of latest values updated by telemetry stream.
- **Edge Connection Registry:** Use Redis `SET` with `edge_id` and connection ID; for multi‑instance, use Redis pub/sub to route messages to the correct instance holding the WebSocket.
- **Offline Queue:** Use Redis **Sorted Set** per edge (`queue:site1_edge`). Store command JSON as the member, and score = expiration timestamp (Unix time). Use `ZADD` to insert, `ZRANGEBYSCORE` to retrieve non‑expired, and `ZREMRANGEBYSCORE` to clean expired.
- **In‑Flight Watchdog:** Store `command_id` in a Redis hash `in_flight:<command_id>` with `EXPIRE` 5 seconds. Use a background task (or Redis keyspace notifications) to detect expiry and trigger timeout.
- **Rate Limiting:** Token bucket algorithm per `(user_id, edge_id)`; Redis counters with expiry.
- **Logging:** Structured JSON logs with `command_id` as correlation key; ship to ELK or Datadog for traceability.

---

## Summary

The Cloud‑Side Dispatcher Service is the first line of defense in the command pipeline. By validating every command against device profiles (static and dynamic limits), routing efficiently to online edges, and safely queuing commands for offline edges using Redis sorted sets with per‑command TTL, it ensures that only safe, valid commands ever reach the edge. Its mandatory in‑flight watchdog guarantees that the UI never hangs indefinitely, and the robust acknowledgment system gives clients clear, timely feedback. Comprehensive logging enables end‑to‑end traceability for debugging and audit purposes.