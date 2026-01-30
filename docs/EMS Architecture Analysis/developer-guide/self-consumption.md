# Self consumption

## Overview

Self-consumption control is a fundamental energy management strategy that maximizes the use of locally generated renewable energy (primarily solar PV) by storing excess energy in battery systems and releasing it when needed. The goal is to minimize energy exchange with the grid, reducing electricity costs and improving energy independence.

### Key Objectives
- **Zero Grid Exchange**: Maintain grid power flow as close to zero as possible
- **Cost Optimization**: Reduce electricity bills by minimizing grid imports
- **Energy Independence**: Maximize utilization of local renewable energy
- **Grid Stability**: Reduce stress on the electrical grid infrastructure

### Operational Principle
```
Solar Generation > Load Consumption → Charge Battery (Excess to Battery)
Solar Generation < Load Consumption → Discharge Battery (Battery to Load)
Solar Generation = Load Consumption → No Battery Action (Zero Grid Exchange)
```

# 1. Physical and Data Acquisition Layer
Before any control logic can execute, the EMS must establish a reliable "Nervous System" to sense the environment.
This stage involves setting up the hardware interfaces and the thread-safe data structures that Local and Global controllers depend on.

### Hardware Interface Setup
The system must establish bidirectional communication with three primary conponents:

#### Grid Meter (The primary sensor):
Connected via **Modebus TCP**, the meter provides the "Ground Truth" for site power balance. The EMS polls the **active_power** register at high frequency (e.g., every 500ms).

#### Battery Management System (BMS):
Connected via **CAN Bus**, the BMS provides critical safety data, including individual cell voltages and the overall **State of Charge (SOC)**.

#### Inverters (The Actuators):
Connected via **Modbus TCP**, these receive the final power setpoints calculated by the controllers.

## 2. The Centralized Data Store (DataStore)
To ensure that the high-speed control loops do not crash when accessing data simultaneously, a thread-safe **DataStore** must be implemented.

- **Atomic Updates**: Modbus and CAN workers update the DataStore upon receiving data.
- **Historical Buffering**: Maintains a buffer of previous meter readings and timestamps to calculate the rate of power change for Derivate (*Kd*) calculations.
- **Thread Safety**: Access is protected by threading locks to prevent race conditions during simultaneous read/write operations.

## 3. Data Ingestion Logic
The ingestion helper retrieves and validates sensor info.
- **Validation**: Returns a boolean **success** flag and the numeric **value** of the meter.

- **Reliability**: If a reading is missing or stale (e.g., older than 2 seconds), the **success** flag must be set to **False** to trigger the Sensor Loss Failsafe.

# 2. Local Device Control (Level 1)
The goal of this stage is to implement the local feedback loop that stabilizes a single site or device before the Master Controller takes over global coordination.

## 1. The Local PID Controller
Local control uses the PID utility class to calculate the power requirement needed to nullify the grid.

- **Proportional Response**: The system applies a standard gain (**prop = 0.1**) to the current grid meter reading. This creates an immediate reaction to any power flow detected at the grid boundary.

- **Integral Component (Memory)**: Calculates an integral value by multiplying the previous setpoint by a decay factor (**0.995** for multi-device sites) to maintain stability. 

- **Error Adjustment**: For sites with multiple devices, the integral is multiplied by a factor to account for small synchronization differences between units.

## 2. Operational Modes Implementation
The Site Controller provides standardized entry points for different energy strategies:

- **Grid Nullification Strategy**:
The primary goal is to maintain the site's energy balance at exactly *0W watts* relative to the utility grid.
The controller continuously calculates the difference between local production and local demand to ensure no energy is imported or exported.

- **Unidirectional Charging Logic**:
A restricted operational state that only allows energy to flow into the batteries.
While still balancing the grid, the algorithm suppresses any output that would result in a discharge.

- **Emergency Recovery Sequence**:
An automated survival mode triggered by critical hardware states. When the battery hits the **Battery Safety Floor**, the system overrides all external commands to force a steady, controlled recharge from the grid until cells reach safe voltage.

## 3. Failsafes and Hardware Protection
As Local Device Control is the closest software layer to the hardware, it must enforce physical safety limits:

- **Battery Safety Floor Logic**:
This acts as a physical "Safety Buffer" to prevent battery cells from reaching a critically low voltage state. The system monitors the lowest individual cell voltage and compares it against a predefined safety margin.

- **Sensor Loss Failsafe**:
If the grid meter becomes unreachable, the controller must not stay at the last known power level. Instead, it executes a "**Graceful Degradation**" protocol, reducing the current active power command by **25% per iteration** until it reaches 0W.

- **Hardware Power Ceiling**:
All calculated power requests are passed through a "Clamping" function to ensure commands do not exceed the absolute physical capablities of the electrical system.

- **Enenrgy Depletion Guard**:
Blocks discharge attempts if the State of Charge hits the **Reserved SOC** threshold (e.g., for backup power).

# 3. Master Controller Coordination (Level 2)
The goal is to implement the global control loop that manages multiple inverters and distributes power commands across the cluster.

## Global PID Controller
Controller Coordination implements a full **Proportional-Integral-Derivative (PID)** controller for precise grid balancing.

- **Target Setpoint (SP)**:
The target is fixed at **0W**, representing the ideal state of no energy exchange with the utility grid.

- **Calculation**:
Compares the target against the meter reading to determine a site-wide **Control Variable (CV)**.

## Anti-Windup and Saturation Logic
To prevent the system from becoming unstable when the battery hits its physical limits, specific "guardrails" are implemented in the PID math.

- **Saturation Protection**:
If the battery is 100% full or completely empty, the controller is notified that it has reached "Saturation".

- **Unwind Routine**:
When saturation occurs or if a meter reading is missed, the system executes an "Unwind" operation.
This slowly decays the accumulated integral error, preventing the controller from "overshooting" once the battery is able to accept or provide power again.

## Power Distribution and SoC Balancing
Once the PID controller determines the total power needed for the site (Control Variable), the Master Controller must decide how to split that work.

- **Cluster-Wide Distribution**
Splits the site-wide command among available nodes based on individual charge/discharge limits.

- **SoC Balancing**:
To ensure all batteries age evenly, the system applies power offsets. Batteries with a higher State of Charge (SoC) are given a larger share of the discharge load, while batteries with lower SoC are prioritized for charging.
Applies power offsets when the SOC differences exceed 5% and stops when the difference is under 2%.

- **Constraint Checking**:
Before seding commands, the Master Controller verifies the total **Cluster Charge and Discharge** limits to ensure the site-wide command stays within safe electrical boundaries.

- **Grid Code Override**: 
Upon entering self-consumption mode, grid code ramps should be set to **False** to allow instantaneous reactions.

# 4. Command & State Machine Integration
The goal of this stage is to manage how the system transitions into different energy management modes and how it handles asynchronous commands from external sources.

## External Command Mapping
The system uses a dedicated **Command Interface** to translate high-level requests into internal events.

- **Method Validation**
When a command like a request for self-consumption is received, the system first validates the payload to ensure all necessary parameters are present.

- **Event Generation**
Once a command is validated, it is transformed into a standardized **Local Event** (e.g., "Mode Change Event").

- **Mode Mapping**
Requests for specific behaviours - such as prioritizing storage or balancing the grid are mapped to internal logic configurations, ensuring the controllers know which math to apply.

## Hierarchical State Machine (HSM)

The system's behavior is governed by a State Machine that manages operational transitions and ensures the cluster is in a safe state.

- **Operational Transitions**:
Manages moves between states such as **activate**, **maxself** (Grid Nullification), and **chargeonly**.

- **Entry and Exit Logic**:
Each state has specific "On Enter" and "On Exit" protocols.
For example, entering the Grid Nullification state triggers the reset of the PID controller and disables standard grid-code ramps to allow for faster local response.

- **Condition Checking**:
Transitions only occur if specifice conditions are met, such as verifying that at least one inverter is active and no-critical system-wide error are present.

## Execution Loop
The system operates on a continuous periodic cycle ("Main Loop") to maintain real-time control.

- **Iteration**:
The main loop runs every 500ms to trigger the state machine and active control functions.

- **Heartbeat Monitoring**:
The loop monitors for a "Heartbeat" signal. If external control signals are lost for a defined period, the state machine automatically triggers a transition to a safe or "Stop" state.

- **Cloud Telemetry**:
Periodically, the loop sends a cluster-wide status update to the cloud, reporting active power, State of Charge and current operational mode.

# 5. Technical Specifications & Tuning Parameters

## 1. Control System Constants
The stability of the grid nullification process depends on these specific parameters used in the PID control logic:

| Parameter                     | Value  | Description                                                                 |
|-------------------------------|--------|-----------------------------------------------------------------------------|
| Main Loop Interval            | 500ms  | The frequency at which the state machine and control logic execute.          |
| Proportional Gain ($K_p$)     | 0.1    | The local site-level response factor for immediate power correction.         |
| Integral Decay Factor         | 0.995  | Used in multi-device sites to prevent synchronization drift.                |
| Anti-Windup Constant          | 0.02   | The rate at which integral error is decayed during saturation or signal loss.|
| Grid Code Ramp Rate           | 0.00167| The standard power increment (per 0.5s) used only when grid codes are enabled.|

## 2. Critical Safety & Operational Thresholds
These logic-based values govern the system's transition into failsafe and backup modes:

- **Data Staleness Limit**: 2.0 Seconds, If the grid meter reading is older than this, the system enters the **Sensor Loss Failsafe**.

- **Sensor Loss Ramp-Down**: 25% Reduction, The active power setpoint is multiplied by 0.75 in every iteration cycle until it reaches 0W.

- **Local Heartbeat Timeout**: 20 seconds, The maximum time the system will remain in **local** mode without receiving a fresh control signal before reverting to a **stop** state.

- **Off-Grid Detection**: If phase voltages drop below this threshold (230V - 40%), the system initiates the **Backup Power Sequence**.

- **SOC Balance Start/Stop**: The cluster begins balancing node power when SOC differences exceed 5% and stops when the difference drops below 2%.

# 3. Cluster Power Logic
The Master Controller handles power distribution using the following hierarchical logic:

## 1. Read Total Capacity
Calculates the **total_inverter_capacity** based on all nodes currently in **cluster** mode.

## 2. Caclculate Global Command:
The PID loop generates a single site-wide Control Variable (CV) to hit the 0W target.

## 3. Distribution Logic:
- **Charge/Discharge Limits**:
Commands are capped by the sum of individual node limits (cluster_charge_limit / cluster_discharge_limit).

- **Grid Code Override**:
Upon entering **maxself** mode, **grid_code_ramps_enabled** is set to **False** to allow instantaneous reaction to grid loads.