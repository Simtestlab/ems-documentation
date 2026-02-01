# Communication Interface

## Overview

The communication between the **Edge Device (Raspberry Pi)** and the Cloud Server (FastAPI) occurs over a persistent **WebSocket (WSS)** connection. This ensures low-latency, bidirectional control.

- **Protocol**: WebSocket Secure (WSS)
- **Endpoint**: *wss://api.domain.com/ws/device/{device_id}*
- **Frequency**: Telemetry is pushed upstream every **1.0 second**.

---
## Upstream Payload (Telemetry)

![Telemetry Payload](./telemetry-payload.png)

**Direction**: Edge (Pi) --> Cloud --> Dashboard

**Purpose**: To visualize the live status of the system.

The Raspberry Pi sends this JSON packet every second.

```json
{
  "device_id": "ems_001",
  "timestamp": "2026-01-31T12:00:01Z",
  "system_status": {
    "state": "CHARGING",       // Must match State Machine definitions
    "mode": "AUTO",            // AUTO or MANUAL
    "error_flags": [],         // List of active errors (e.g. ["OVERHEAT"])
    "wifi_strength": -45
  },
  "battery": {
    "voltage": 53.2,           // Volts
    "current": 10.5,           // Amps (+ = Charge, - = Discharge)
    "power": 558.6,            // Watts
    "soc": 85                  // State of Charge (%)
  },
  "grid": {
    "voltage": 230.1,
    "power": -200,             // Negative = Selling to Grid
    "frequency": 50.0
  }
}
```

### Key Field Definitions:

- **system_status.state**: The current active state from the FSM (e.g., **IDLE**, **FULL**, **ERROR**)

- **battery.current**: Follows the polarity standart (Postive=Charging).

## Downstream Payload (Control Commands)

![Downstream payload diagram](./downstream-payload.png)

**Direction**: Dashboard --> Cloud --> Edge (Pi)

**Purpose**: To change the system's operating mode or reset errors.

The Raspberry Pi listerns for these JSON packets asynchronously.

#### A. Set Mode Command

Used to force the system into a specific strategy or manual override.

```json
{
  "command_id": "cmd_8293",    // Unique ID for tracking
  "action": "SET_MODE",
  "payload": {
    "target_mode": "MANUAL_DISCHARGE",
    "target_power": 500        // Watts (Optional, specific to mode)
  }
}
```

#### B. Reset Error Command

Used to clear the **ERROR** state after a manual inspection.

```json
{
  "command_id": "cmd_8294",
  "action": "RESET_ERROR",
  "payload": {}
}
```

## Data storage

![Data storage flow diagram](./storage-API.png)

The FastAPI server receives the live telemetry stream and immediately inserts it into TimescaleDB for efficient, long-term historical storage.