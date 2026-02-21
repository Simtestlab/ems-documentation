# Offline Command Queue and TTL

## Purpose

The **Offline Command Queue** ensures that user commands (setpoints, mode changes, etc.) are not lost when the target edge device is temporarily disconnected from the cloud. Commands are stored in a persistent queue with a **time‑to‑live (TTL)** and delivered automatically when the edge reconnects. The TTL mechanism prevents stale commands (e.g., a discharge command meant for a peak event hours ago) from being executed after the relevant time window has passed.

Key objectives:
- Reliable delivery of commands despite network interruptions.
- Prevention of stale commands through per‑command expiration.
- **Strict FIFO ordering** regardless of TTL values.
- Efficient storage and retrieval using a combination of Redis list and sorted set.
- Clear user feedback when queued commands expire before delivery.

---

## Role in the Command Dispatcher Architecture

The offline queue resides in the cloud, managed by the Cloud‑Side Dispatcher. It acts as a buffer for commands addressed to edges that are currently offline.

```
┌─────────────────────────────────────────────┐
│           UI / Dashboard                     │
└───────────────┬─────────────────────────────┘
                │ command
┌───────────────▼─────────────────────────────┐
│     Cloud‑Side Dispatcher                    │
├─────────────────────────────────────────────┤
│  • Validation                                │
│  • Online → forward immediately              │
│  • Offline → push to queue                   │
└───────────────┬─────────────────────────────┘
                │ LPUSH to LIST
                │ ZADD to TTL SET
┌───────────────▼─────────────────────────────┐
│              Redis                           │
│  ┌───────────────────────────────────────┐  │
│  │ LIST per edge:   queue:edge_id         │  │
│  │   (order preserved, stores payload)    │  │
│  │ Sorted Set per edge: ttl:edge_id       │  │
│  │   score = expiration timestamp         │  │
│  │   member = command_id                   │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
                │ on edge reconnect
                │ RPOP from LIST, check TTL
┌───────────────▼─────────────────────────────┐
│           Edge Gateway                       │
│  (receives queued commands)                  │
└─────────────────────────────────────────────┘
```

When an edge reconnects, the dispatcher pops commands from the list (FIFO order) and checks each command's expiration by looking up its TTL in the sorted set (or simply comparing with the command's embedded timestamp + TTL). Expired commands are discarded and trigger a UI notification. Non‑expired commands are forwarded to the edge.

A background process periodically scans TTL sorted sets and removes expired entries; it also notifies the UI of any expired commands that were never delivered.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **Command Queue (LIST)** | A Redis list (key `queue:{edge_id}`) storing command payloads in the order they were received. This preserves strict FIFO ordering. |
| **TTL Index (Sorted Set)** | A Redis sorted set (key `ttl:{edge_id}`) where each member is a `command_id` and the score is the absolute Unix timestamp at which the command expires. Used for efficient expiration cleanup. |
| **Command Payload** | The full JSON of the command, including `command_id`, `type`, `target`, `value`, `timestamp`, and the assigned `expiry_sec`. |
| **TTL (Time‑To‑Live)** | The duration for which a command remains valid. TTL is set based on command type: • **Power setpoints / manual overrides:** ≤ 60 seconds. • **Configuration updates / schedules:** ≤ 24 hours. • **System commands:** typically not queued (fail immediately if edge offline). |
| **Expiration Check** | Before delivering a command, the dispatcher computes `now > command.timestamp + command.expiry_sec`. If true, the command is considered expired and discarded. |
| **Hard Queue Limit** | A maximum size (e.g., 10,000 commands) for each edge's queue. When the limit is reached, new commands are rejected with an error. |
| **UI Notification on Expiry** | When a command expires in the queue (either during delivery attempt or background cleanup), the system sends a final status `failed: timeout_in_queue` to the UI to prevent indefinite hanging. |

---

## Functional Requirements

-   The system shall maintain a separate **list** (`queue:{edge_id}`) and a **sorted set** (`ttl:{edge_id}`) for each edge.
    
-   When a validated command targets an offline edge, it shall be:
    
    -   Pushed to the tail of the list using `LPUSH` (or `RPUSH` depending on orientation; we'll define consistently).
        
    -   Added to the sorted set with score = `current_time + expiry_sec` and member = `command_id`.
        
-   TTL values shall be configurable per command type. Defaults:
    
    -   Power setpoints/manual overrides: 60s
        
    -   Configuration updates/schedules: 86400s (24h)
        
    -   System commands: not queued (immediate failure).
        
-   When an edge reconnects, the dispatcher shall repeatedly pop commands from the head of the list using `RPOP` (or `LPOP`) until the list is empty. For each popped command:
    
    -   Retrieve its embedded `timestamp` and `expiry_sec`.
        
    -   If `now > timestamp + expiry_sec`, discard the command and send a `failed: timeout_in_queue` notification to the UI (if the user connection is still alive).
        
    -   Otherwise, forward the command to the edge.
        
-   After processing each command (delivered or expired), the corresponding entry shall be removed from the TTL sorted set using `ZREM`.
    
-   A background process shall periodically scan all TTL sorted sets and remove entries with score < current time. For each removed entry, it shall:
    
    -   Publish a notification (via Redis pub/sub) to the Cloud Dispatcher, which will send a `failed: timeout_in_queue` to the UI.
        
-   Each edge's queue shall have a **hard limit** (e.g., 10,000 commands). If the list length exceeds this limit, the dispatcher shall reject any new incoming commands for that edge with an HTTP 429 (Too Many Requests) or WebSocket error.
    
-   The system shall log all queue operations (add, deliver, expire, overflow) with timestamps and command IDs for audit.

## Detailed Flow

###  Adding a Command to the Queue
1. Cloud‑Side Dispatcher validates a command targeting edge `site1_edge`.
2. Edge is determined to be offline (no active WebSocket connection).
3. **Check queue size:** If `LLEN queue:site1_edge` ≥ MAX_QUEUE_SIZE (e.g., 10,000), reject the command with a `429` error (Queue Full) and send status `failed: queue_full` to UI. No further processing.
4. Compute expiration timestamp:
   ```
   expiry_ts = now + TTL(command.type)
   ```
5. Add command payload to Redis list (at the tail):
   ```
   RPUSH queue:site1_edge command_json
   ```
6. Add command ID to TTL sorted set:
   ```
   ZADD ttl:site1_edge expiry_ts command_id
   ```
7. Send immediate acknowledgment to UI: `{"command_id": "...", "status": "queued"}`.

### Edge Reconnects
1. Edge establishes WebSocket connection and authenticates.
2. Cloud‑Side Dispatcher detects the edge online event.
3. Dispatcher repeatedly processes commands from the list:
   ```
   while command_json = LPOP queue:site1_edge
   ```
   (LPOP removes from the head, preserving FIFO order.)
4. For each popped command:
   - Parse JSON, extract `command_id`, `timestamp`, `expiry_sec`.
   - If `now > timestamp + expiry_sec`:
     - Command has expired.
     - Send final status to UI: `{"command_id": "...", "status": "failed", "reason": "timeout_in_queue"}`.
     - Remove from TTL set: `ZREM ttl:site1_edge command_id`.
     - Continue to next command.
   - Else:
     - Forward command to edge via its WebSocket.
     - Wait for immediate acknowledgment (`received`). If edge rejects, log and continue.
     - Remove from TTL set: `ZREM ttl:site1_edge command_id`.
     - (The command is now in the edge's request channel; final ack will come later.)
5. After processing all commands, the edge is ready for real‑time commands.

### Background Expiration Cleanup and UI Notification
- A periodic task (e.g., every minute) runs:
  ```
  now = current_unix_time()
  for each edge:
      # Find all command_ids that have expired
      expired_ids = ZRANGEBYSCORE ttl:edge_id -inf now
      for each id in expired_ids:
          # Publish notification to Redis pub/sub channel "command_expired"
          publish "command_expired" json.dumps({"command_id": id, "reason": "timeout_in_queue"})
          # Remove from TTL set
          ZREM ttl:edge_id id
  ```
- The Cloud‑Side Dispatcher subscribes to the `command_expired` channel. Upon receiving a notification, it looks up the user who issued the command (by `command_id` in a separate mapping) and sends a final WebSocket message to the UI: `{"command_id": "...", "status": "failed", "reason": "timeout_in_queue"}`.

### Handling Queue Overflow
- Before adding any command to the queue, the dispatcher checks `LLEN queue:edge_id`.
- If length ≥ MAX_QUEUE_SIZE (configurable, default 10,000), the command is rejected immediately.
- The UI receives a `failed: queue_full` error.

---

## Integration with Other Components

| Component | Interaction |
|-----------|-------------|
| **Cloud‑Side Dispatcher** | Performs queue insertion, retrieval, overflow checks, and publishes expiry notifications. |
| **Edge Connection Registry** | Provides online/offline status; triggers queue flush on reconnect. |
| **Redis** | Stores lists and sorted sets. Must be persistent (RDB/AOF) to survive restarts. Also used for pub/sub for expiry notifications. |
| **Command Validation** | Ensures only validated commands are queued. |
| **UI / Dashboard** | Receives immediate `queued` status and final `failed: timeout_in_queue` or `failed: queue_full` notifications. |

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Queue write latency** | < 10 ms (p99) | Time from decision to Redis RPUSH + ZADD |
| **Queue read latency** | < 50 ms (p99) for 1000 commands | Time to LPOP and process |
| **Expiry notification latency** | < 5 seconds from expiration to UI | Monitor from background job to UI |
| **Hard limit enforcement** | 100% of commands rejected when queue full | Load test to capacity |
| **Storage efficiency** | Support ≥ 10,000 commands per edge | Load test with Redis |
| **Delivery success rate** | 100% of non‑expired commands delivered when edge reconnects | Test with simulated disconnects |

---

## Implementation Notes

- **Data Structures:**
  - **List:** `queue:{edge_id}` – store command JSON. Use `RPUSH` to add, `LPOP` to consume (FIFO).
  - **Sorted Set:** `ttl:{edge_id}` – store `command_id` with score = expiration timestamp. Use `ZADD` to insert, `ZREM` after delivery or expiry, and `ZRANGEBYSCORE` for cleanup.
- **TTL Calculation:** Use absolute Unix timestamp (seconds). Ensure clocks are synchronized across cloud instances (NTP).
- **Command JSON:** Store the exact same JSON that would be forwarded to the edge. Include all original fields, plus the assigned `expiry_sec` (which is used to compute expiration on delivery). Do not rely solely on the TTL set for delivery check – use the command's embedded timestamp + expiry to allow per‑command variation.
- **Hard Queue Limit:** Set MAX_QUEUE_SIZE in configuration (e.g., 10,000). Use `LLEN` before each queuing operation.
- **Expiry Notification Pub/Sub:** Use a dedicated Redis pub/sub channel, e.g., `command_expired`. The Cloud Dispatcher instances subscribe to this channel and forward to the appropriate UI connection (via user‑command mapping stored in Redis or local memory). For MVP, a simple mapping of `command_id` → `user_id` and `connection_id` can be stored with a TTL (e.g., 24h) to allow routing.
- **Monitoring:** Use Redis `INFO` to track memory. Log queue sizes periodically. Alert if any queue exceeds a warning threshold (e.g., 5,000 commands) as it may indicate a stuck edge.
- **Idempotency:** The edge already has idempotency cache, so duplicate delivery after reconnect is safe.
- **Staleness of Dynamic Limits:** Already handled by cloud validation and controller revalidation; queued commands that become invalid due to dynamic limits will be rejected by the controller at execution time (via ack channel). That's acceptable.

---

## Summary

The revised Offline Command Queue design ensures:
- **Strict FIFO ordering** via Redis lists, independent of TTL values.
- **Accurate expiration** using per‑command timestamps and a TTL index for background cleanup.
- **User feedback** on expiry via pub/sub notifications, eliminating indefinitely hanging UI.
- **Protection against queue overload** with a hard limit and immediate rejection.

This design is robust, scalable, and provides a reliable foundation for command delivery even in the face of network interruptions.