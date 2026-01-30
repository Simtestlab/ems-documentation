# EMS State Machine - Operating Modes

## Overview

The EMS implements a comprehensive state machine with multiple operating modes to handle different use cases and control scenarios.

## State Diagram

```
┌──────────┐
│   STOP   │◄──────────────────────────┐
└────┬─────┘                           │
     │ Start                           │
     ▼                                 │ Fault/Stop
┌──────────┐                           │
│ STANDBY  │                           │
└────┬─────┘                           │
     │ Activate                        │
     ▼                                 │
┌─────────────────────────────────────┴┐
│   ┌────────────────────┐             │
│   │ SELF_CONSUMPTION   │             │
│   └────────────────────┘             │
│   ┌────────────────────┐             │
│   │  LOCAL_CONTROL     │             │
│   └────────────────────┘             │
│   ┌────────────────────┐             │
│   │ EXTERNAL_CONTROL   │             │
│   └────────────────────┘             │
│   ┌────────────────────┐             │
│   │  TIME_OF_USAGE     │             │
│   └────────────────────┘             │
│                                      │
│  Active Control States               │
└──────────────────────────────────────┘
```

## Operating States

### 1. STOP
**Purpose**: System completely stopped, no operation

**Behavior**:
- All control outputs set to 0
- No communication with devices
- System initialization state

**Transitions**:
- To STANDBY: Manual start command

---

### 2. STANDBY
**Purpose**: System ready but not actively controlling

**Behavior**:
- Communication active with ESS and Meter
- Monitoring only, no control actions
- Power setpoint = 0W
- Health checks running

**Transitions**:
- To Active States: Mode selection
- To FAULT: Communication failure
- To STOP: Shutdown command

---

### 3. SELF_CONSUMPTION ⚡
**Purpose**: Minimize grid consumption using PV and battery

**Goal**: 0W taken from the grid

**Strategy**:
- Use PV power to supply loads directly
- Excess PV charges battery
- Battery discharges when PV insufficient
- Grid import minimized to zero

**Control Logic**:
```python
setpoint = -meter_power
```
- If meter importing (meter > 0) → discharge battery (setpoint < 0)
- If meter exporting (meter < 0) → charge battery (setpoint > 0)

**SOC Protection**:
- Below 10% SOC + importing → IDLE (prevent deep discharge)
- Above 95% SOC + exporting → IDLE (prevent overcharge)

**Use Case**: Maximize self-consumption of PV energy, reduce electricity bills

---

### 4. LOCAL_CONTROL 🔌
**Purpose**: Allow PLC to control inverter via Modbus

**Goal**: External device (PLC) controls active power setpoint

**Control Flow**:
```
PLC → Modbus Write → EMS → Inverter
```

**Register Map** (to be defined):
- HR xxxx: Local Control Enable (0=disabled, 1=enabled)
- HR xxxx: Local Power Setpoint (W, signed int16)

**Behavior**:
- EMS reads local_setpoint from context
- If local_control_active == True → use local setpoint
- If local_control_active == False → IDLE

**Use Case**: Integration with building automation systems, PLCs

---

### 5. EXTERNAL_CONTROL 🌐
**Purpose**: Remote API control of inverter power

**Goal**: Remote user controls active power setpoint via REST API

**Control Flow**:
```
API Call → EMS HTTP Server → State Machine → Inverter
```

**API Endpoint** (example):
```
POST /api/v1/control/setpoint
{
  "power_watts": 2000,  // positive = charge, negative = discharge
  "duration_s": 3600    // optional timeout
}
```

**Behavior**:
- External command stored in `sm.external_power_cmd`
- Command followed without SOC restrictions (BMS protects)
- Timeout can be implemented for safety

**Use Case**: Remote monitoring systems, demand response programs

---

### 6. TIME_OF_USAGE 💰
**Purpose**: Optimize charging/discharging based on electricity prices

**Goal**: Minimize electricity costs by smart battery scheduling

**Strategy**:
The battery fills the gap when electricity prices are highest. This is achieved by:
1. Charging battery with PV (always preferred)
2. If PV insufficient, buy from grid when prices are low
3. Discharge battery when prices are high to avoid expensive grid power

**Requirements**:
- 24-hour forecast for PV production
- 24-hour forecast for load consumption  
- 24-hour forecast for electricity pricing

**Price Thresholds**:
```python
price_threshold_low = 0.10 €/kWh   # Cheap: charge from grid
price_threshold_high = 0.30 €/kWh  # Expensive: discharge battery
```

**Decision Logic**:

| Condition | Action | Setpoint |
|-----------|--------|----------|
| Price ≥ High & SOC > 20% | Discharge battery | -5000W (max) |
| Price ≤ Low & SOC < 90% | Charge from grid | +5000W (max) |
| Price Medium | Self-consumption | -meter_power |
| SOC out of range | IDLE | 0W |

**Optimization Algorithm** (future enhancement):
```python
# Advanced: Use forecasts to pre-charge before high-price periods
def optimize_schedule(price_forecast, pv_forecast, load_forecast):
    # Dynamic programming or MPC to find optimal charge/discharge schedule
    # Maximize: ∫(avoided_cost) - battery_degradation
    pass
```

**Use Case**: Time-of-use tariffs, dynamic pricing, cost optimization

---

### 7. FAULT ⚠️
**Purpose**: Safe state when system errors detected

**Triggers**:
- Modbus communication failure
- CAN communication failure
- Watchdog timeout
- Critical sensor error

**Behavior**:
- All control outputs set to 0W
- System remains in FAULT until manually cleared
- Logs error conditions
- Health monitoring continues

**Recovery**:
- Manual acknowledgment required
- Check and resolve fault condition
- Transition to STANDBY after clear

---

## Run Modes (Internal Logic)

The state machine uses internal `RunMode` to implement state behaviors:

| RunMode | Description | Setpoint Calculation |
|---------|-------------|---------------------|
| IDLE | No control action | 0W |
| SELF_CONSUMPTION | Balance grid to zero | -meter_power |
| CHARGE_ONLY | Force charging | +3000W to +5000W |
| DISCHARGE_ONLY | Force discharging | -3000W to -5000W |
| EXTERNAL | Follow API command | external_power_cmd |
| LOCAL | Follow PLC command | local_setpoint |
| TIME_OPTIMIZED | Price-based logic | Varies by price |

---

## Power Flow Calculations

### PV Power Estimation
```python
# Method 1: Energy balance
pv_power = -meter_power - inverter_power

# If:
# - Meter = -1000W (exporting to grid)
# - Inverter = +500W (charging battery)
# Then PV = 1000 + 500 = 1500W

# Method 2: Direct measurement (if available)
pv_power = pv_sensor_power
```

### Load Calculation
```python
# Total load = PV production - grid export + battery discharge
load_power = pv_power + meter_power - inverter_power
```

### Sign Conventions
- **Meter Power**:
  - Positive (+) = Importing from grid
  - Negative (-) = Exporting to grid
  
- **Inverter Power**:
  - Positive (+) = Charging battery
  - Negative (-) = Discharging battery
  
- **PV Power**:
  - Always positive (generation)
  - Zero at night

---

## Configuration

### Context Variables
```python
ctx.meter_power = 0.0          # W (from meter)
ctx.inverter_power = 0.0       # W (from ESS)
ctx.bms_soc = 50.0             # %
ctx.pv_power = 0.0             # W (calculated or measured)
ctx.power_setpoint = 0.0       # W (to inverter)
ctx.local_setpoint = 0.0       # W (from PLC)
ctx.local_control_active = False
ctx.current_price = 0.0        # €/kWh
ctx.price_forecast = []        # 24h price data
ctx.load_forecast = []         # 24h load data
ctx.pv_forecast = []           # 24h PV data
```

### State Machine Parameters
```python
sm.price_threshold_high = 0.30  # €/kWh
sm.price_threshold_low = 0.10   # €/kWh
sm.tou_target_soc_high = 90.0   # %
sm.tou_target_soc_low = 20.0    # %
sm.external_power_cmd = 0.0     # W
```

---

## API Usage Examples

### Switch to Self-Consumption Mode
```python
ctx.state = EMSState.SELF_CONSUMPTION
```

### Enable External Control
```python
ctx.state = EMSState.EXTERNAL_CONTROL
sm.set_external_command(2000)  # Charge at 2kW
```

### Enable Time of Usage with Custom Thresholds
```python
ctx.state = EMSState.TIME_OF_USAGE
sm.set_price_thresholds(low=0.08, high=0.35)
ctx.current_price = 0.12  # Update price periodically
```

### Enable Local Control (PLC)
```python
ctx.state = EMSState.LOCAL_CONTROL
ctx.local_control_active = True
ctx.local_setpoint = -1500  # PLC wants to discharge at 1.5kW
```

---

## Safety Features

1. **SOC Protection**:
   - Soft limits in state machine (10%-95%)
   - Hard limits in BMS (hardware protection)

2. **Communication Watchdog**:
   - Automatic FAULT state on comm loss
   - Prevents uncontrolled operation

3. **Rate Limiting** (future):
   - Maximum power change per second
   - Prevents sudden power spikes

4. **Timeout Protection** (future):
   - External commands auto-expire
   - Fallback to safe mode

---

## Testing Scenarios

### Scenario 1: Sunny Day with Self-Consumption
```
Time: 12:00, PV = 4000W, Load = 2000W
Expected: Excess 2000W charges battery
Setpoint: +2000W (charge)
```

### Scenario 2: Evening with Self-Consumption
```
Time: 20:00, PV = 0W, Load = 1500W
Expected: Battery discharges to cover load
Setpoint: -1500W (discharge)
```

### Scenario 3: Low Price with ToU
```
Price: 0.05 €/kWh, SOC = 40%
Expected: Charge battery from grid
Setpoint: +5000W (max charge)
```

### Scenario 4: High Price with ToU
```
Price: 0.40 €/kWh, SOC = 80%
Expected: Discharge battery to grid
Setpoint: -5000W (max discharge)
```

---

## Future Enhancements

1. **Predictive Control**: Use ML to predict PV and load
2. **Multi-objective Optimization**: Balance cost, comfort, battery life
3. **Grid Services**: Frequency regulation, demand response
4. **Peak Shaving**: Limit maximum grid import
5. **Load Shifting**: Time-shift flexible loads
6. **Battery Health**: SOC management for longevity
7. **Islanding Mode**: Off-grid operation capability

---

## Implementation Status

| Feature | Status | Notes |
|---------|--------|-------|
| STOP State | ✅ Complete | Basic implementation |
| STANDBY State | ✅ Complete | Basic implementation |
| SELF_CONSUMPTION | ✅ Complete | With SOC protection |
| LOCAL_CONTROL | 🟡 Framework | Needs Modbus register mapping |
| EXTERNAL_CONTROL | 🟡 Framework | Needs REST API implementation |
| TIME_OF_USAGE | ✅ Complete | Basic price-based logic |
| FAULT State | ✅ Complete | Communication fault handling |
| PV Calculation | ✅ Complete | Energy balance method |
| Price Forecasting | ⏳ Planned | Integration needed |
| Load Forecasting | ⏳ Planned | Integration needed |
| Advanced Optimization | ⏳ Planned | Future work |

Legend: ✅ Complete | 🟡 Partial | ⏳ Planned
