# Components Reference

Detailed reference for each EMS component.

## Core Components

### EMSContext (`ems/context.py`)

Thread-safe state container for shared data.

**Attributes:**

| Attribute | Type | Description | Updated By |
|-----------|------|-------------|------------|
| `state` | EMSState | Current operating state | Controller |
| `meter_power` | float | Grid power (W) | Modbus Worker |
| `inverter_power` | float | ESS power (W) | Modbus Worker |
| `bms_soc` | float | Battery SOC (%) | Modbus/CAN Worker |
| `power_setpoint` | float | Power command (W) | Controller |
| `modbus_ok` | bool | Modbus health | Modbus Worker |
| `can_ok` | bool | CAN health | CAN Worker |
| `lock` | Lock | Thread synchronization | All |

**Usage:**

```python
with ctx.lock:
    power = ctx.meter_power
    soc = ctx.bms_soc
```

### EMSController (`ems/controller.py`)

Main control loop coordinator.

**Methods:**

- `start()`: Initialize and run main loop
- `update()`: Execute one control cycle

**Control Logic:**

```python
def update(self):
    self.sm.update()  # Update state machine
    
    # Read state
    with self.ctx.lock:
        state = self.ctx.state
        meter = self.ctx.meter_power
    
    # Calculate setpoint based on state
    if state == EMSState.SELF_CONSUMPTION:
        setpoint = -meter
    elif state == EMSState.EXTERNAL_CONTROL:
        setpoint = self.sm.external_power_cmd
    else:
        setpoint = 0
    
    # Write setpoint
    with self.ctx.lock:
        self.ctx.power_setpoint = setpoint
```

### EMSStateMachine (`ems/state_machine.py`)

State and mode management.

**Attributes:**

- `run_mode`: Current run mode (IDLE, SELF_CONSUMPTION, etc.)
- `external_power_cmd`: External power command (W)

**Methods:**

- `update()`: Update run mode based on state and conditions
- `set_external_command(power)`: Set external power command
- `_handle_self_consumption_mode(soc)`: Self-consumption logic
- `_handle_external_control_mode(soc)`: External control logic

**State Handlers:**

```python
def _handle_self_consumption_mode(self, soc):
    """No SOC restrictions - BMS handles protection"""
    self.run_mode = RunMode.SELF_CONSUMPTION

def _handle_external_control_mode(self, soc):
    """No SOC restrictions - BMS handles protection"""
    self.run_mode = RunMode.EXTERNAL
```

## Worker Threads

### ModbusWorker (`interfaces/modbus_worker.py`)

Modbus TCP interface for ESS and meter.

**Connections:**

- ESS: `127.0.0.1:1503` (unit 100)
- Meter: `127.0.0.1:1502` (unit 1)

**Methods:**

```python
def read_meter(self):
    """Read grid power from meter (HR 32790-32791)"""
    rr = self.meter.read_holding_registers(32790, 2, unit=1)
    # Parse IEEE-754 float
    lo, hi = rr.registers
    raw = struct.pack(">HH", hi, lo)
    power = struct.unpack(">f", raw)[0]
    
    with self.ctx.lock:
        self.ctx.meter_power = power

def read_ess(self):
    """Read ESS power and SOC"""
    # Active power (HR 552, scale 0.01 kW)
    rr = self.ess.read_holding_registers(552, 1, unit=100)
    power = rr.registers[0] * 10.0
    
    # SOC (HR 5122, %)
    rr = self.ess.read_holding_registers(5122, 1, unit=100)
    soc = rr.registers[0]
    
    with self.ctx.lock:
        self.ctx.inverter_power = power
        self.ctx.bms_soc = soc

def write_setpoint(self):
    """Write power setpoint to ESS (HR 1281)"""
    with self.ctx.lock:
        watts = self.ctx.power_setpoint
    
    raw = int(watts / 10)  # Convert to 0.01 kW
    self.ess.write_register(1281, raw, unit=100)
```

**Polling:** 1 second

### CANWorker (`interfaces/can_worker.py`)

CAN bus interface for BMS.

**Configuration:**

- Channel: `can0`
- Bitrate: 500 kbps
- Bustype: `socketcan`

**Methods:**

```python
def run(self):
    """Listen for CAN messages from BMS"""
    while True:
        msg = self.bus.recv(timeout=1.0)
        
        if msg is None:
            # Timeout - set CAN not OK
            with self.ctx.lock:
                self.ctx.can_ok = False
        else:
            # Process message
            self.process_message(msg)
            
            with self.ctx.lock:
                self.ctx.can_ok = True

def process_message(self, msg):
    """Parse BMS CAN message"""
    # Parse SOC, voltage, current, etc.
    # Update context
```

**Timeout:** 1 second

## Enumerations

### EMSState (`ems/state.py`)

```python
class EMSState(Enum):
    STOP = 0              # System off
    STANDBY = 1           # Ready but idle
    SELF_CONSUMPTION = 2  # Active self-consumption
    EXTERNAL_CONTROL = 3  # Following external commands
    FAULT = 4             # Error condition
```

### RunMode (`ems/state_machine.py`)

```python
class RunMode(Enum):
    IDLE = auto()              # No control
    SELF_CONSUMPTION = auto()  # Grid balancing
    CHARGE_ONLY = auto()       # Force charge (override)
    DISCHARGE_ONLY = auto()    # Force discharge (override)
    EXTERNAL = auto()          # External command
```

## Data Structures

### Register Mapping

**ESS (Modbus Unit 100):**

| Register | Name | Type | Scale | Access |
|----------|------|------|-------|--------|
| 552 | Active Power | INT16 | 0.01 kW | R |
| 1280 | Work State | UINT16 | - | R/W |
| 1281 | Set Active Power | INT16 | 0.01 kW | W |
| 5122 | SOC | UINT16 | % | R |

**Meter (Modbus Unit 1):**

| Register | Name | Type | Format | Access |
|----------|------|------|--------|--------|
| 32790 | Active Power (Lo) | UINT16 | IEEE-754 | R |
| 32791 | Active Power (Hi) | UINT16 | Float BE | R |

### CAN Message Format

Example BMS message structure:

| Byte | Field | Description |
|------|-------|-------------|
| 0-1 | SOC | State of charge (0.01%) |
| 2-3 | Voltage | Pack voltage (0.1V) |
| 4-5 | Current | Pack current (0.1A) |
| 6-7 | Temperature | Pack temperature (0.1°C) |

## Thread Synchronization

### Lock Usage Pattern

```python
# Reading
with self.ctx.lock:
    value = self.ctx.some_value

# Writing
with self.ctx.lock:
    self.ctx.some_value = new_value

# Multiple operations
with self.ctx.lock:
    val1 = self.ctx.value1
    val2 = self.ctx.value2
    self.ctx.value3 = val1 + val2
```

### Thread Safety Rules

1. **Always use lock** when accessing context
2. **Keep critical sections short** (minimize time holding lock)
3. **Never call blocking I/O** while holding lock
4. **Read-modify-write** as atomic operation

## Error Handling

### Communication Errors

```python
try:
    self.read_meter()
    self.read_ess()
    self.write_setpoint()
    
    with self.ctx.lock:
        self.ctx.modbus_ok = True
        
except Exception as e:
    log.error(f"Modbus error: {e}")
    with self.ctx.lock:
        self.ctx.modbus_ok = False
```

### Fault State

```python
if not modbus_ok or not can_ok:
    with self.ctx.lock:
        self.ctx.state = EMSState.FAULT
    self.run_mode = RunMode.IDLE
```

## Extension Points

### Adding New States

1. Add to `EMSState` enum
2. Add case in `update()` control logic
3. Add state transition logic in state machine

### Adding New Sensors

1. Add attribute to `EMSContext`
2. Update worker to read sensor
3. Use value in control logic

### Custom Control Modes

1. Add to `RunMode` enum
2. Add handler method in state machine
3. Implement control logic in controller

## Next Steps

- Review [State Machine](state-machine.md)
- Understand [Architecture](overview.md)
- See [Development Guide](../development/emulators.md)
