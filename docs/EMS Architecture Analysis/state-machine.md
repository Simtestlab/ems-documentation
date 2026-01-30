# State Machine

The EMS uses a hierarchical state machine with two levels:

1. **EMS States**: High-level operating states
2. **Run Modes**: Internal control modes

## EMS States

```
    ┌──────┐
    │ STOP │
    └───┬──┘
        │ start()
        ▼
   ┌─────────┐
   │STANDBY  │◄───────────┐
   └────┬────┘            │
        │                 │ state change
        │ auto transition │
        ▼                 │
┌──────────────────┐      │
│SELF_CONSUMPTION  │──────┤
└──────────────────┘      │
                          │
┌──────────────────┐      │
│EXTERNAL_CONTROL  │──────┤
└──────────────────┘      │
                          │
   ┌───────┐              │
   │ FAULT │──────────────┘
   └───────┘
    ▲
    │ comm loss
    │
```

### State Descriptions

#### STOP
- **Purpose**: System completely stopped
- **Run Mode**: `IDLE`
- **Setpoint**: 0W
- **Transitions**: → STANDBY (via `start()`)

#### STANDBY
- **Purpose**: Ready but not actively controlling
- **Run Mode**: `IDLE`
- **Setpoint**: 0W
- **Transitions**: → SELF_CONSUMPTION (auto), EXTERNAL_CONTROL (API)

#### SELF_CONSUMPTION
- **Purpose**: Minimize grid import/export
- **Run Mode**: `SELF_CONSUMPTION` (or overrides)
- **Setpoint**: Calculated to zero grid power
- **Transitions**: → EXTERNAL_CONTROL (API), FAULT (comm loss)

**Control Logic:**
```python
if mode == RunMode.SELF_CONSUMPTION:
    setpoint = -meter_power  # Cancel out grid power
```

**No SOC restrictions** - BMS handles protection

#### EXTERNAL_CONTROL
- **Purpose**: Follow external power commands
- **Run Mode**: `EXTERNAL` (or overrides)
- **Setpoint**: External command value
- **Transitions**: → SELF_CONSUMPTION (API), FAULT (comm loss)

**No SOC restrictions** - BMS handles protection

#### FAULT
- **Purpose**: Safe state during communication failures
- **Run Mode**: `IDLE`
- **Setpoint**: 0W
- **Triggers**: `modbus_ok = False` OR `can_ok = False`
- **Recovery**: Automatic when communications restored

## Run Modes

Run modes determine the actual control behavior within each state.

### IDLE
- No active control
- Setpoint = 0W
- Used in: STOP, STANDBY, FAULT

### SELF_CONSUMPTION
- Calculate setpoint to zero grid power
- Setpoint = `-meter_power`
- Used in: SELF_CONSUMPTION state

### CHARGE_ONLY
- Force charging at fixed power
- Setpoint = +3000W (example)
- Override mode (currently unused)

### DISCHARGE_ONLY
- Force discharging at fixed power
- Setpoint = -3000W (example)
- Override mode (currently unused)

### EXTERNAL
- Follow external power command
- Setpoint = `external_power_cmd`
- Used in: EXTERNAL_CONTROL state

## State Machine Logic

Located in `ems/state_machine.py`:

```python
class EMSStateMachine:
    def update(self):
        # Fault handling (highest priority)
        if not modbus_ok or not can_ok:
            self.ctx.state = EMSState.FAULT
            self.run_mode = RunMode.IDLE
            return
        
        # State-specific logic
        if state == EMSState.SELF_CONSUMPTION:
            self._handle_self_consumption_mode(soc)
        elif state == EMSState.EXTERNAL_CONTROL:
            self._handle_external_control_mode(soc)
```

### Self-Consumption Handler

```python
def _handle_self_consumption_mode(self, soc):
    """Keep grid meter at 0W"""
    # No SOC restrictions - BMS handles protection
    self.run_mode = RunMode.SELF_CONSUMPTION
```

### External Control Handler

```python
def _handle_external_control_mode(self, soc):
    """Follow external commands"""
    # No SOC restrictions - BMS handles protection
    self.run_mode = RunMode.EXTERNAL
```

## State Transitions

### Via API

Change state:
```bash
curl -X POST http://localhost:8000/state \
  -H "Content-Type: application/json" \
  -d '{"state": "EXTERNAL_CONTROL"}'
```

Send external command:
```bash
curl -X POST http://localhost:8000/external-control \
  -H "Content-Type: application/json" \
  -d '{"power_watts": 3000}'
```

### Automatic Transitions

1. **Startup**: STOP → STANDBY → SELF_CONSUMPTION
2. **Fault Detection**: ANY → FAULT (on comm loss)
3. **Fault Recovery**: FAULT → Previous state (when comm restored)

## Decision Flow

```
┌─────────────────────┐
│   Read Context      │
│ - state             │
│ - mode              │
│ - meter_power       │
│ - soc               │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Check Communications│
│ modbus_ok?          │
│ can_ok?             │
└──────────┬──────────┘
           │
      ┌────┴────┐
      │  OK?    │
      └────┬────┘
      NO   │   YES
       ┌───▼───┐
       │ FAULT │
       └───┬───┘
           │
           ▼
    ┌─────────────┐
    │ Set mode =  │
    │    IDLE     │
    └─────────────┘
           │
           ▼
    State-specific
    control logic
           │
           ▼
┌─────────────────────┐
│ Calculate Setpoint  │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Update Context      │
│ power_setpoint      │
└─────────────────────┘
```

## Next Steps

- Learn about [Components](components.md)
- Understand [Operating Modes](../user-guide/operating-modes.md)
- Explore the [Control Logic](../user-guide/self-consumption.md)
