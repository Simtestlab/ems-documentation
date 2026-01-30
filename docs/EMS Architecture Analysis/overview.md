# Architecture Overview

The In-House EMS follows a modular, thread-based architecture for real-time energy management.

## System Components

```
┌────────────────────────────────────────────────────────────┐
│                     Main Controller                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              EMS Controller (main.py)                 │  │
│  │  - Initializes all workers                            │  │
│  │  - Manages state transitions                          │  │
│  │  - Executes control logic                             │  │
│  └────────┬─────────────────┬──────────────┬─────────────┘  │
│           │                 │              │                │
│  ┌────────▼────────┐ ┌──────▼──────┐ ┌────▼────────┐      │
│  │ State Machine   │ │   Context   │ │  Workers    │      │
│  │ (state flow)    │ │  (shared)   │ │ (threads)   │      │
│  └─────────────────┘ └─────────────┘ └─────────────┘      │
└────────────────────────────────────────────────────────────┘
         │                    │                    │
┌────────▼─────┐    ┌─────────▼────────┐   ┌──────▼────────┐
│ Modbus Worker│    │   CAN Worker     │   │  API Server   │
│ (ESS+Meter)  │    │   (BMS)          │   │  (REST)       │
└────────┬─────┘    └─────────┬────────┘   └───────────────┘
         │                    │
┌────────▼─────┐    ┌─────────▼────────┐
│ ESS Inverter │    │  BMS (CAN Bus)   │
│ Grid Meter   │    │                  │
└──────────────┘    └──────────────────┘
```

## Core Components

### 1. EMS Controller (`ems/controller.py`)

The main control loop that:

- Initializes all workers (Modbus, CAN)
- Transitions between states (STANDBY → SELF_CONSUMPTION)
- Executes control logic based on current state and mode
- Logs system status every second

**Key Methods:**
- `start()`: Initialize and run the main control loop
- `update()`: Execute state machine and control logic

### 2. State Machine (`ems/state_machine.py`)

Manages operational modes and handles state transitions:

**States:**
- `STOP`: System off
- `STANDBY`: Ready but not controlling
- `SELF_CONSUMPTION`: Active self-consumption
- `EXTERNAL_CONTROL`: Following external commands
- `FAULT`: Error condition (comm loss)

**Run Modes:**
- `IDLE`: No active control
- `SELF_CONSUMPTION`: Minimize grid import/export
- `CHARGE_ONLY`: Force charging (override mode)
- `DISCHARGE_ONLY`: Force discharging (override mode)
- `EXTERNAL`: Follow external power command

### 3. Context (`ems/context.py`)

Shared state container with thread-safe access:

```python
class EMSContext:
    def __init__(self):
        self.lock = threading.Lock()
        
        # State
        self.state = EMSState.STOP
        
        # Measurements (updated by workers)
        self.meter_power = 0.0       # W
        self.inverter_power = 0.0    # W
        self.bms_soc = 50.0          # %
        
        # Control (updated by controller)
        self.power_setpoint = 0.0    # W
        
        # Health (updated by workers)
        self.modbus_ok = False
        self.can_ok = True
```

### 4. Modbus Worker (`interfaces/modbus_worker.py`)

Background thread that:

- Connects to ESS and Meter via Modbus TCP
- Reads meter power, inverter power, and SOC
- Writes power setpoint to ESS
- Updates `modbus_ok` health flag

**Polling:** 1 second interval

### 5. CAN Worker (`interfaces/can_worker.py`)

Background thread that:

- Listens on CAN bus for BMS messages
- Updates SOC and battery status
- Monitors CAN communication health
- Updates `can_ok` health flag

**Timeout:** 1 second

## Data Flow

### Read Path (Monitoring)

```
Hardware → Workers → Context → Controller → Logs/API
```

1. Workers read from hardware (Modbus, CAN)
2. Workers update shared Context (thread-safe)
3. Controller reads Context for control logic
4. Status logged and exposed via API

### Write Path (Control)

```
Controller → Context → Workers → Hardware
```

1. Controller calculates power setpoint
2. Controller writes to Context
3. Modbus Worker reads from Context
4. Modbus Worker sends command to ESS

## Threading Model

All workers run as daemon threads:

- **Main Thread**: EMS Controller (control loop)
- **Modbus Thread**: Hardware I/O (read/write)
- **CAN Thread**: BMS monitoring
- **API Thread**: REST server (if enabled)

**Thread Safety:** All shared state access uses `context.lock`

## Fault Handling

The system automatically enters `FAULT` state when:

- Modbus communication fails (`modbus_ok = False`)
- CAN communication fails (`can_ok = False`)

In `FAULT` state:

- Power setpoint set to 0W (safe state)
- Run mode set to `IDLE`
- System stays in FAULT until communications recover

## Next Steps

- Understand the [State Machine](state-machine.md) in detail
- Learn about [Components](components.md)
- See [Operating Modes](../user-guide/operating-modes.md)
