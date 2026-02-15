# Command Dispatch 

## 1. What is Command Dispatch?

Command Dispatch is the **bidirectional control channel** that enables:

- **Users** (via dashboard) to send real‑time commands to any edge device.
- **Automated cloud services** (schedulers, optimizers) to issue setpoints.
- **Edges** to acknowledge execution and report status.

It transforms the EMS from a **passive monitoring system** into an **active control system**.

---

## Command Flow (Step‑by‑Step)

![Command flow diagram](./edge-command-flow-diagram.png)

### Step 1: User Initiates Command
- Operator clicks "Set Battery Charge 50kW" in Next.js dashboard.
- Dashboard validates input (range, syntax).
- Sends JSON payload over **WebSocket**:

```json
{
  "command_id": "cmd_12345",
  "command_type": "setpoint",
  "target": {
    "edge_id": "site1_edge_prod",
    "device_id": "bms_main",
    "channel": "SetActivePower"
  },
  "value": 50000,
  "unit": "W",
  "expiry_sec": 300,
  "source": "user_456",
  "timestamp": "2026-02-15T14:30:00Z"
}
```

### Step 2: Cloud Authentication & Authorization
- FastAPI WebSocket server receives payload.
- Validates:
  - **Authentication:** User session/JWT valid.
  - **Authorization:** User has `can_control` permission for this edge.
  - **Rate limiting:** Not exceeding commands/sec.
- If invalid → error response to dashboard.

### Step 3: Command Routing
- FastAPI looks up edge's active WebSocket connection (from connection registry).
- If edge online → forward command immediately.
- If edge offline:
  - Store in **Redis command queue** (persistent).
  - Set expiry (configurable, default 24h).
  - Edge will fetch on reconnect.

### Step 4: Edge Receives & Executes
- Edge agent receives command via its persistent WebSocket.
- **Immediate validation:**
  - Device exists? Channel writable? Value within limits?
- If invalid → reject with reason.
- If valid:
  - Place command into **Core Cycle execution queue**.
  - Core Cycle will apply at next cycle (or immediately if flagged `priority: emergency`).

### Step 5: Execution & Acknowledgment
- Core Cycle executes command (writes setpoint to Modbus device).
- Result (success/failure) captured.
- Edge sends acknowledgment back to cloud:

```json
{
  "command_id": "cmd_12345",
  "status": "executed",
  "executed_at": "2026-02-15T14:30:01.050Z",
  "error": null
}
```

### Step 6: Cloud Notifies Dashboard
- FastAPI receives acknowledgment.
- Broadcasts to dashboard (if still connected) or stores in history.
- Dashboard shows "Command Executed" with timestamp.

---

## Command Types (MVP)

| Type | Description | Example |
|------|-------------|---------|
| **Setpoint** | Write a value to a device channel | `SetActivePower = 50kW` |
| **Mode change** | Switch controller mode | `Enable Peak Shaving` |
| **Configuration override** | Temporary parameter change | `Override MinSoc = 15% for 1 hour` |
| **Scheduler control** | Start/stop controllers | `Disable Balancing Controller` |
| **System command** | Edge lifecycle | `Restart`, `Sync config now` |

---

## Technical Requirements (MVP)

### Cloud Side (FastAPI)

| S No | Requirement |
|----|-------------|
| **01** | Maintain persistent WebSocket connections to all online edges. |
| **02** | Registry mapping `edge_id` → active WebSocket connection. |
| **03** | Authenticate all incoming commands (user JWT). |
| **04** | Authorize per edge (check user permissions). |
| **05** | Queue commands for offline edges (Redis). |
| **06** | Rate limiting per user/edge. |
| **07** | Full audit log: who sent what command, when, result. |
| **08** | Timeout handling: if no acknowledgment within 10s, mark as failed. |

### Edge Side (Python Agent)

| S No | Requirement |
|----|-------------|
| **01** | Maintain persistent WebSocket connection to cloud. |
| **02** | Auto‑reconnect with exponential backoff. |
| **03** | Validate incoming commands against device capabilities. |
| **04** | Queue commands if Core Cycle busy (bounded queue). |
| **05** | Execute commands in Core Cycle order (preserving priority). |
| **06** | Send acknowledgment (success/failure) back to cloud. |
| **07** | Log all commands locally (for audit when cloud unreachable). |

---

## Integration with Existing Components

| Component | Role in Command Dispatch |
|-----------|--------------------------|
| **Next.js Dashboard** | Command UI; real‑time feedback. |
| **Django Auth** | User authentication & permissions. |
| **FastAPI WebSocket** | Primary command router. |
| **Redis** | Offline queue & rate limiting store. |
| **Edge Core Cycle** | Actual command execution (writes setpoints). |
| **Modbus Adapter** | Physical device communication. |
| **TimescaleDB** | Command audit log storage. |
| **Edge Config Sync** | Provides device capability definitions. |

---