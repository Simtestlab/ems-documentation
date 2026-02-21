# Acknowledgment Flow

## Purpose

The **Acknowledgment Flow** provides end‑to‑end visibility into the status of every user command, from the moment it leaves the UI to its final execution (or rejection) on the edge. Without a robust acknowledgment system, users would be left guessing whether their action succeeded, and operators would have no way to debug failed commands.

The flow is **two‑stage**:
- **Immediate Acknowledgment** – Confirms that the cloud has received the command, validated it, and either forwarded it to the edge or queued it for later delivery.
- **Final Acknowledgment** – Reports the ultimate outcome after the edge controllers have processed the command: `executed`, `rejected`, or `failed` (with a reason).

This design gives users clear, timely feedback and provides full traceability for debugging and audit purposes.

---

## Role in the Command Dispatcher Architecture

The acknowledgment flow spans the UI, cloud, edge, and controllers, tying together the entire command lifecycle.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
│  (displays command status)                   │
└───────────────┬─────────────────────────────┘
                │ command │ ack
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
├─────────────────────────────────────────────┤
│  • Validates command                         │
│  • Sends immediate ack                        │
│  • Forwards to edge / queues                  │
│  • Receives final ack from edge               │
│  • Relays final ack to UI                     │
│  • Manages in‑flight watchdog                 │
└───────────────┬─────────────────────────────┘
                │ forward │ final ack
┌───────────────▼─────────────────────────────┐
│     Edge‑Side Dispatcher                     │
├─────────────────────────────────────────────┤
│  • Writes to request channel                  │
│  • Sends immediate ack (received)             │
│  • Subscribes to ack/ channels                │
│  • Forwards final ack to cloud                │
└───────────────┬─────────────────────────────┘
                │ writes decision
┌───────────────▼─────────────────────────────┐
│         Device Abstraction Layer             │
│  • ui_requests/... channels                   │
│  • ack/... channels (written by controllers)  │
└───────────────┬─────────────────────────────┘
                │ reads │ writes
┌───────────────▼─────────────────────────────┐
│           Controllers                         │
│  (Peak Shaving, Balancing, etc.)              │
│  • Read request channels                      │
│  • Apply safety checks                        │
│  • Write decision to ack/ channel             │
└─────────────────────────────────────────────┘
```

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Immediate Acknowledgment** | Sent by the cloud (or edge) as soon as a command is accepted and written to a request channel (or queued). Status values: `"received"` (edge got it), `"queued"` (edge offline), or `"rejected"` (validation failed). |
| **Final Acknowledgment** | Sent after the command has been processed by the edge controllers. Status values: `"executed"`, `"rejected"`, or `"failed"`. Includes optional `reason` and `executed_at` timestamp. |
| **Acknowledgment Channel (`ack/`)** | A DAL channel where controllers write the final outcome of a command. Naming convention: `ack/{request_channel_name}` (e.g., `ack/battery_1_power`). |
| **In‑Flight Watchdog** | A cloud‑side mechanism that tracks commands forwarded to online edges. If no final acknowledgment is received within a timeout (e.g., 5 seconds), it sends a `failed: edge_timeout` to the UI. |
| **Command ID Correlation** | Every acknowledgment includes the original `command_id`, allowing the UI and logs to correlate the response with the initial request. |
| **Duplicate Acknowledgment Filter** | Both cloud and edge may ignore duplicate acknowledgments (e.g., if the controller writes the same status twice). |
| **Terminal State Lock** | Once a command reaches a terminal state (`executed`, `rejected`, or `failed`), any subsequent acknowledgments for that `command_id` are discarded. This ensures UI consistency. |

---

## Functional Requirements

-   Every command received by the Cloud-Side Dispatcher shall result in an immediate acknowledgment sent to the UI. Status shall be `"received"`, `"queued"`, or `"rejected"`.
    
-   When the edge receives a command and writes it to a request channel, it shall send an immediate acknowledgment `"received"` to the cloud.
    
-   The edge shall maintain a subscription to the `ack/` namespace in the DAL. When a controller writes a value to an acknowledgment channel, the edge shall forward that payload to the cloud as the final acknowledgment.
    
-   Final acknowledgment payloads shall include at minimum:
    
    -   `command_id`
        
    -   `status` (`"executed"`, `"rejected"`, or `"failed"`)
        
    -   `executed_at` (ISO timestamp)
        
    -   optionally a `reason` string
        
-   The cloud shall maintain an **in-flight watchdog** for every command forwarded to an online edge. If no final acknowledgment is received within a configurable timeout (default 5 seconds), the cloud shall send a `failed: edge_timeout` to the UI and log the incident.
    
-   The cloud shall relay all final acknowledgments (from edge or watchdog) to the UI via the same WebSocket connection that issued the command (if still connected).
    
-   If the UI connection is no longer available, the cloud may store the final status briefly (e.g., in Redis with TTL) for later retrieval if the user reconnects, or simply log it for audit.
    
-   Both cloud and edge shall ignore duplicate acknowledgments for the same `command_id` (e.g., if the same ack is sent twice).
    
-   Once a command has reached a terminal state (`executed`, `rejected`, or `failed`), the cloud dispatcher shall **discard** any further acknowledgments for that `command_id` and log them as `late_ack_dropped`.
    
-   All acknowledgments shall be logged with full details (`command_id`, status, reason, timestamps) for audit and traceability.

## Detailed Flow

### Command Initiation and Immediate Acknowledgment

#### Scenario A: Edge Online
1. UI sends command to Cloud‑Side Dispatcher.
2. Cloud validates command, looks up edge connection, finds it online.
3. Cloud forwards command to edge via its WebSocket.
4. Cloud places `command_id` in in‑flight watchdog with 5‑second TTL (using Redis sorted set).
5. Edge receives command, performs local validation, writes to request channel.
6. Edge sends immediate acknowledgment `{"command_id": "...", "status": "received"}` back to cloud.
7. Cloud relays this to UI.
8. UI updates command status to "Processing" (or similar).

#### Scenario B: Edge Offline
1. UI sends command to Cloud‑Side Dispatcher.
2. Cloud validates command, edge offline.
3. Cloud pushes command to queue and sends immediate acknowledgment `{"command_id": "...", "status": "queued"}`.
4. UI displays "Queued (offline)".
5. (Later, when edge reconnects, command will be delivered and follow the online path from step 3 onward.)

### Controller Processing and Final Acknowledgment

1. During the next Core Cycle, the appropriate controller reads the request channel (e.g., `ui_requests/battery_1_power`).
2. Controller applies safety checks (SOC, limits, etc.) and decides whether to execute.
3. Controller writes the final decision to the corresponding acknowledgment channel `ack/battery_1_power` with a payload like:
   ```json
   {
     "command_id": "cmd_12345",
     "status": "executed",
     "executed_at": "2026-02-20T10:30:01.050Z"
   }
   ```
   or
   ```json
   {
     "command_id": "cmd_12345",
     "status": "rejected",
     "reason": "Battery SOC too low",
     "executed_at": "2026-02-20T10:30:01.050Z"
   }
   ```

### Edge Forwards Final Acknowledgment to Cloud

1. Edge‑Side Dispatcher, which subscribes to all `ack/` channels via an event-driven callback, detects the new value on `ack/battery_1_power` immediately.
2. It forwards the exact payload (as JSON) to the cloud via its WebSocket connection.
3. Cloud receives the final acknowledgment, cancels the in‑flight watchdog for that `command_id`, and checks if the command is already in a terminal state (due to a previous watchdog timeout). If not, it proceeds.
4. Cloud relays the payload to the UI (if the UI connection is still alive).
5. UI updates the command status to "Executed", "Rejected", or "Failed" accordingly.

### In‑Flight Watchdog Timeout

1. Cloud forwarded command to edge at `t=0` and set watchdog score in Redis sorted set at `t+5s`.
2. At `t=5.1`, a background worker scans the sorted set and finds the command expired.
3. Worker marks the command as terminal, sends a final status to UI:
   ```json
   {
     "command_id": "cmd_12345",
     "status": "failed",
     "reason": "edge_timeout",
     "executed_at": "2026-02-20T10:30:05.100Z"
   }
   ```
4. Cloud logs the timeout.
5. If a final acknowledgment arrives later (e.g., delayed network), it is discarded (per ACK‑09) and logged as `late_ack_dropped`.

### Duplicate Acknowledgment Handling

- Edge may accidentally send the same final acknowledgment twice (e.g., due to a bug or race condition).
- Cloud maintains a small cache of recently processed final acknowledgment `command_id`s (e.g., last 100, with 1‑minute TTL). If an incoming final ack's `command_id` is already in the cache, it is silently dropped.
- Similarly, edge may drop duplicate acks from controllers if needed.

### Terminal State Lock

Once the cloud dispatcher has delivered a final acknowledgment (executed, rejected, or failed) to the UI, it considers that `command_id` to be in a **terminal state**. Any subsequent acknowledgment messages received for the same `command_id` are discarded and logged as `late_ack_dropped`. This ensures that the UI never receives conflicting updates (e.g., timeout followed by late success).

---

## Acknowledgment Payload Schemas

### Immediate Acknowledgment (Cloud → UI)

```json
{
  "command_id": "cmd_12345",
  "status": "received" | "queued" | "rejected",
  "reason": "optional rejection reason"  // present only if status = "rejected"
}
```

### Final Acknowledgment (Edge → Cloud → UI)

```json
{
  "command_id": "cmd_12345",
  "status": "executed" | "rejected" | "failed",
  "reason": "optional reason",  // required for rejected/failed
  "executed_at": "2026-02-20T10:30:01.050Z"  // ISO 8601 timestamp
}
```

### Watchdog Timeout (Cloud → UI)

Same as final acknowledgment with `status: "failed"` and `reason: "edge_timeout"`.

### Queue Expiry Notification (Cloud → UI)

Same as final acknowledgment with `status: "failed"` and `reason: "timeout_in_queue"`.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Cloud‑Side Dispatcher** | Sends immediate acks; receives final acks; manages in‑flight watchdog; relays all to UI. |
| **Edge‑Side Dispatcher** | Sends immediate acks; subscribes to `ack/` channels via callbacks; forwards final acks to cloud. |
| **Device Abstraction Layer** | Provides `ack/` channels where controllers write decisions; supports observer pattern for instant notifications. |
| **Controllers** | Read request channels; write final decisions to `ack/` channels. |
| **In‑Flight Watchdog** | Cloud‑side background worker using Redis sorted set; triggers on missing final ack. |
| **Redis** | Sorted set for watchdog timers; optional mapping for command‑to‑user. |
| **Audit Logging** | All acknowledgments logged with full details. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Immediate ack latency** | < 100 ms (p99) from command receipt | Log timestamps at cloud entry and ack send |
| **Final ack latency** | < 2 seconds (p99) from command receipt to final ack (for online edges) | Log timestamps across the flow |
| **Watchdog accuracy** | 100% of commands without final ack trigger timeout | Test with simulated edge failure |
| **Late ack discard** | 100% of late acks dropped and logged | Send ack after timeout, verify no UI update |
| **Duplicate ack filtering** | 100% of duplicates dropped | Send duplicate final acks and verify only one reaches UI |
| **Ack delivery rate** | 100% of final acks reach UI (when UI connected) | End‑to‑end integration tests |

---

## Implementation Notes

- **Immediate Ack in Edge:** After writing to request channel, the edge should send `"received"` immediately. No need to wait for controller.
- **Ack Channel Subscription:** Do not use polling. The DAL must implement a simple observer pattern (e.g., passing a callback function `on_ack_update(channel, payload)` or using an `asyncio.Queue`). When a Controller writes to the `ack/` namespace, the DAL instantly pushes the payload to the Edge Dispatcher, achieving near‑zero latency and simpler code.
- **Command‑to‑User Mapping:** To route final acks to the correct UI connection when the user may have multiple tabs or reconnect later, store a mapping in Redis: `cmd_user_map:{command_id}` with value = `user_id` and TTL = 24h. When a final ack arrives, look up the user and broadcast to all their active connections (or store for retrieval on reconnect).
- **Duplicate Filter:** Use a Redis set or local cache (e.g., `cachetools.TTLCache`) for recent `command_id`s.
- **Watchdog Implementation:** Do not rely on Redis TTL and keyspace notifications, as expiration events are not timely and do not carry metadata. Instead, use a Redis Sorted Set (`in_flight_watchdog`) where the `score` is the exact Unix timestamp of the timeout (e.g., `now + 5s`). The member of the set should be a JSON string containing `command_id` and the `user_id` (or routing information). A background worker continuously queries `ZRANGEBYSCORE in_flight_watchdog -inf <current_time>` to fetch all timed‑out commands. For each result, it processes the timeout (sends `failed: edge_timeout` to the UI), removes it from the set with `ZREM`, and logs accordingly.
- **Terminal State Lock:** After sending any final acknowledgment (or watchdog timeout), record the terminal state in a Redis cache (e.g., `cmd_terminal:{command_id}` with TTL 24h). Before processing any incoming final ack, check if the command already has a terminal state; if so, discard and log.
- **Logging:** Structured logs with fields: `command_id`, `status`, `reason`, `latency_ms`. Index by `command_id` for easy tracing.

---

## Summary

The two‑stage acknowledgment flow provides complete visibility into the command lifecycle:
- **Immediate acks** reassure users that their command was received and is being processed.
- **Final acks** give definitive outcome, with clear reasons for rejection or failure.
- **Watchdog timeouts** prevent hanging UIs when the edge fails to respond.
- **Queue expiry notifications** inform users when their queued command never made it.
- **Terminal state lock** ensures UI consistency and avoids conflicting updates.

Together with the event‑driven edge subscription, robust watchdog implementation, and duplicate filtering, this design ensures that every command is accounted for and that users are never left wondering what happened.

---