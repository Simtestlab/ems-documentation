# OpenEMS Energy Management Modules Documentation

## Table of Contents
1. [Overview](#overview)
2. [ESS (Energy Storage System) Modules](#ess-energy-storage-system-modules)
3. [Energy API and Scheduler](#energy-api-and-scheduler)
4. [Controller Scheduler Modules](#controller-scheduler-modules)
5. [Predictor Modules](#predictor-modules)
6. [Time-of-Use Tariff Modules](#time-of-use-tariff-modules)
7. [Integration and Workflow](#integration-and-workflow)

---

## Overview

OpenEMS provides a comprehensive energy management system built on modular, OSGi-based architecture. The system enables intelligent energy optimization through prediction, scheduling, and tariff-aware control strategies.

### Core Components
- **ESS Modules**: Energy storage system interfaces and implementations
- **Energy API**: Global optimization and scheduling framework
- **Schedulers**: Controller execution ordering and timing
- **Predictors**: Consumption and production forecasting algorithms
- **Tariff Modules**: Dynamic electricity pricing integrations

---

## ESS (Energy Storage System) Modules

### Location
- `io.openems.edge.ess.api` - Core ESS interfaces
- `io.openems.edge.ess.core` - Core ESS implementation
- `io.openems.edge.ess.generic` - Generic ESS wrapper
- `io.openems.edge.ess.cluster` - Multi-ESS clustering
- Various vendor-specific implementations (FENECON, Samsung, etc.)

### Core ESS Natures

#### 1. SymmetricEss
**Base interface for symmetric energy storage systems**

**Key Channels:**
- `SOC` - State of Charge (%, 0-100)
- `CAPACITY` - Battery capacity (Wh)
- `GRID_MODE` - Current grid mode (On-Grid/Off-Grid)
- `ACTIVE_POWER` - Active power (W, negative=charge, positive=discharge)
- `REACTIVE_POWER` - Reactive power (var)
- `MAX_APPARENT_POWER` - Maximum inverter power (VA)
- `ACTIVE_CHARGE_ENERGY` - Cumulative charge energy (Wh)
- `ACTIVE_DISCHARGE_ENERGY` - Cumulative discharge energy (Wh)
- `MIN_CELL_VOLTAGE` / `MAX_CELL_VOLTAGE` - Battery cell voltages (mV)
- `MIN_CELL_TEMPERATURE` / `MAX_CELL_TEMPERATURE` - Battery temperatures (°C)

**Purpose:** Provides balanced power across all three phases

#### 2. ManagedSymmetricEss
**Controllable symmetric ESS extending SymmetricEss**

**Key Channels:**
- `ALLOWED_CHARGE_POWER` - Maximum allowed charge power (W, negative)
- `ALLOWED_DISCHARGE_POWER` - Maximum allowed discharge power (W, positive)
- `SET_ACTIVE_POWER_EQUALS` - Write command for fixed power
- `SET_ACTIVE_POWER_LESS_OR_EQUALS` - Maximum discharge power limit
- `SET_ACTIVE_POWER_GREATER_OR_EQUALS` - Maximum charge power limit
- `SET_REACTIVE_POWER_EQUALS` - Fixed reactive power command
- `SET_ACTIVE_POWER_EQUALS_WITH_PID` - Power control with PID filter

**Purpose:** Most commonly used interface for controllable ESS implementations

#### 3. AsymmetricEss
**Three-phase asymmetric ESS extending SymmetricEss**

**Key Channels:**
- `ACTIVE_POWER_L1`, `ACTIVE_POWER_L2`, `ACTIVE_POWER_L3` - Per-phase active power
- `REACTIVE_POWER_L1`, `REACTIVE_POWER_L2`, `REACTIVE_POWER_L3` - Per-phase reactive power

**Purpose:** Provides separate power measurements for each phase

#### 4. ManagedAsymmetricEss
**Controllable asymmetric ESS**

**Additional Channels:**
- Per-phase active/reactive power write commands (L1, L2, L3)

**Purpose:** Enables independent control of each phase

#### 5. SinglePhaseEss / ManagedSinglePhaseEss
**Single-phase ESS connected to L1, L2, or L3**

**Key Methods:**
- `getPhase()` - Returns connected phase identifier

### Specialized ESS Natures

#### HybridEss
**ESS with DC-coupled PV chargers**

**Key Channels:**
- `DC_DISCHARGE_POWER` - Actual battery discharge power (W)
  - Represents: `ACTIVE_POWER` minus DC charger production
- `DC_CHARGE_ENERGY` - Cumulative battery charge energy (Wh)
- `DC_DISCHARGE_ENERGY` - Cumulative battery discharge energy (Wh)

**Purpose:** For hybrid inverters where AC power includes DC-PV excess

**Key Method:**
- `getSurplusPower()` - Returns surplus DC power when battery is full

#### MetaEss
**Virtual ESS wrapping multiple physical ESS**

**Key Methods:**
- `getEssIds()` - Returns Component-IDs of physical ESS components

**Purpose:** ESS clustering and aggregation

**Use Case:** Combines multiple ESS into single logical unit

#### OffGridEss
**ESS with off-grid/island mode capabilities**

**Key Methods:**
- `isOffGridPossible()` - Returns true if ESS can form micro-grid

**Purpose:** Systems that can maintain AC grid when disconnected from utility

### ESS Power Management

#### Power API
**Location:** `io.openems.edge.ess.api/src/io/openems/edge/ess/power/api/`

**Core Interfaces:**

1. **Power Interface**
   - Constraint-based power management
   - Methods:
     - `addConstraint(Constraint)` - Add power constraint
     - `addConstraintAndValidate(Constraint)` - Add with validation
     - `createSimpleConstraint()` - Create basic constraint
     - `removeConstraint(Constraint)` - Remove constraint
     - `getMaxPower()` / `getMinPower()` - Get power limits
     - `fitValueIntoMinMaxPower()` - Adjust value to fit constraints

2. **Constraint Class**
   - Linear coefficients for power equations
   - Relationships: EQUALS, LESS_OR_EQUALS, GREATER_OR_EQUALS
   - Value can be enabled/disabled dynamically

3. **SolverStrategy Enum**
   - `ALL_CONSTRAINTS` - Satisfy all constraints
   - `OPTIMIZE_BY_MOVING_TOWARDS_TARGET` - Optimize towards target
   - `OPTIMIZE_BY_KEEPING_TARGET_DIRECTION_AND_MAXIMIZING_IN_ORDER` - Keep direction, maximize
   - `OPTIMIZE_BY_KEEPING_ALL_EQUAL` - Equal distribution
   - `OPTIMIZE_BY_KEEPING_ALL_NEAR_EQUAL` - Near-equal distribution

**Purpose:** Manages complex power distribution across multiple ESS with competing constraints

### ESS Implementations

#### Generic ESS (`io.openems.edge.ess.generic`)
- Wraps battery and battery-inverter components
- Flexible configuration for various hardware combinations

#### ESS Cluster (`io.openems.edge.ess.cluster`)
- Combines multiple ESS into single logical unit
- Power distribution via Power-Class
- Enables controllers to work with multiple ESS

#### Vendor-Specific Implementations
- FENECON Commercial 40
- Samsung SDI ESS
- Adstec Storaxe
- Others

---

## Energy API and Scheduler

### Location
- `io.openems.edge.energy.api` - Energy scheduling API
- `io.openems.edge.energy` - Energy scheduler implementation

### Energy Scheduler Architecture

#### EnergyScheduler Service
**Singleton Component:** `_energy`

**Purpose:** Global energy schedule optimizer using genetic algorithms (Jenetics framework)

**Key Channels:**
- `SIMULATIONS_PER_QUARTER` - Number of simulations per 15-minute period
- `GENERATIONS_PER_QUARTER` - Number of genetic algorithm generations

#### Core Concepts

##### 1. Mode
- Predefined operation mode of a Controller
- Represented as a `Gene` in Jenetics
- Example: CHARGE_FROM_GRID, DELAY_DISCHARGE, BALANCING

##### 2. Period
- Time slice holding one Mode per Controller
- Represented as index in `Genotype` of `Chromosomes`
- Duration: 15 minutes (quarter) or 1 hour

##### 3. Schedule
- Set of multiple Periods defining operation over time
- Represented as `Genotype` in Jenetics
- Optimized through genetic algorithm evolution

##### 4. Optimization
- Multiple Schedules simulated and evolved
- Represented as `Population` in Jenetics
- Runs every 15 minutes with configurable execution time

#### Context Hierarchy

##### GlobalOptimizationContext (goc)
**Scope:** Entire optimization run (recreated every 15 minutes)

**Contains:**
- Clock and start timestamp
- Risk level configuration
- List of all Energy Schedule Handlers
- Grid limits and constraints
- Production and consumption predictions
- Time-of-use prices
- ESS information (capacity, current SOC)

##### GlobalScheduleContext (gsc)
**Scope:** Single Schedule simulation (recreated multiple times per second)

**Contains:**
- Initial ESS energy at start of Period
- Accumulated state during simulation

##### ControllerOptimizationContext (coc)
**Scope:** Entire optimization for one Controller/EnergyScheduleHandler

**Contains:**
- Controller-specific configuration
- Mode definitions and constraints

##### ControllerScheduleContext (csc)
**Scope:** Single Schedule simulation for one Controller

**Contains:**
- Controller state during simulation
- Mode-specific parameters

### Energy Schedule Handlers

#### EnergyScheduleHandler Interface
**Purpose:** Controllers implement this to participate in energy optimization

**Key Methods:**
- `getModes()` - Returns available operation modes
- `buildContext()` - Creates optimization context
- `applySchedule()` - Applies computed schedule

#### EnergySchedulable Interface
**Purpose:** Marker interface for components that can be energy-scheduled

```java
public interface EnergySchedulable extends OpenemsComponent {
    EnergyScheduleHandler getEnergyScheduleHandler();
}
```

### Optimization Algorithm

#### Optimizer Class
**Location:** `io.openems.edge.energy/src/io/openems/edge/energy/optimizer/`

**Process:**

1. **Quick Optimization**
   - Runs immediately on trigger
   - Single generation for fast initial result
   - Provides baseline schedule

2. **Regular Optimization**
   - Runs continuously every 15 minutes
   - Multiple generations for better results
   - Execution time calculated dynamically
   - Uses previous result as initial population

**Genetic Algorithm Configuration:**
- Codec: `EshCodec` - Encodes/decodes schedules
- Fitness Function: Minimizes cost based on:
  - Electricity prices
  - Grid constraints
  - Battery wear
  - Risk level preferences
- Selection: Tournament selection
- Evolution: Single-point crossover, mutation

**Triggering Rescheduling:**
- Configuration changes
- New ESS/Controller added/removed
- Tariff provider changes
- Manual trigger

### Energy Flow Simulation

#### Simulator Class
**Purpose:** Simulates energy flows for schedule evaluation

**Simulates:**
- Production (PV, etc.)
- Consumption (unmanaged load)
- ESS charge/discharge
- Grid import/export
- Cost calculation based on prices

**Inputs:**
- Schedule (sequence of modes)
- Predictions (production, consumption)
- Tariff prices
- ESS constraints

**Outputs:**
- Total cost
- Energy flows per period
- Constraint violations
- Fitness score

---

## Controller Scheduler Modules

### Location
- `io.openems.edge.scheduler.api` - Scheduler interface
- `io.openems.edge.scheduler.fixedorder` - Static order
- `io.openems.edge.scheduler.daily` - Time-based scheduling
- `io.openems.edge.scheduler.allalphabetically` - Alphabetical order
- `io.openems.edge.scheduler.jscalendar` - Calendar-based

### Scheduler API

#### Scheduler Interface
**Purpose:** Determines execution order of Controllers

**Key Methods:**
- `getControllers()` - Returns ordered set of Controller Component-IDs

**Execution:** Called once per cycle (every second)

**Channel:**
- `CONTROLLER_IS_MISSING` - Info state when configured Controller is missing

### Scheduler Implementations

#### 1. Fixed Order Scheduler
**Module:** `io.openems.edge.scheduler.fixedorder`

**Strategy:** Static list of Controller IDs

**Configuration:**
- List of Component-IDs in desired execution order

**Use Case:** Simple, predictable execution order

**Example:**
```
["ctrlEssTimeOfUseTariff0", "ctrlBalancing0", "ctrlEssSurplusGrid0"]
```

#### 2. Daily Scheduler
**Module:** `io.openems.edge.scheduler.daily`

**Strategy:** Time-of-day based Controller execution

**Configuration:**
- `Always Run Before` - Controllers executed first
- `Daily Schedule` - Time-based Controller lists
  ```json
  [{
    "time": "08:00:00",
    "controllers": ["ctrlFixActivePower0"]
  }, {
    "time": "13:45:00",
    "controllers": ["ctrlBalancing0"]
  }]
  ```
- `Always Run After` - Controllers executed last

**Use Case:** Different control strategies at different times

#### 3. All Alphabetically Scheduler
**Module:** `io.openems.edge.scheduler.allalphabetically`

**Strategy:** Alphabetical sorting of all Controllers

**Use Case:** Default/testing scenarios

#### 4. JS Calendar Scheduler
**Module:** `io.openems.edge.scheduler.jscalendar`

**Strategy:** Calendar-based scheduling with complex rules

**Use Case:** Advanced time-based scheduling with holidays, weekends, etc.

### Scheduler Integration

**Execution Flow:**
1. Scheduler returns ordered Controller list
2. Each Controller's `run()` method is called in order
3. Controllers set power constraints on ESS
4. Power component resolves constraints and writes to ESS

---

## Predictor Modules

### Location
- `io.openems.edge.predictor.api` - Predictor interfaces
- `io.openems.edge.predictor.lstm` - LSTM neural network
- `io.openems.edge.predictor.similardaymodel` - Similar day technique
- `io.openems.edge.predictor.persistencemodel` - Persistence model
- `io.openems.edge.predictor.profileclusteringmodel` - Profile clustering
- `io.openems.edge.predictor.production.linearmodel` - Linear regression

### Predictor API

#### Predictor Interface
**Purpose:** Provides future predictions for channels

```java
public interface Predictor extends OpenemsComponent {
    ChannelAddress[] getChannelAddresses();
    Prediction getPrediction(ChannelAddress channelAddress);
}
```

**Key Methods:**
- `getChannelAddresses()` - Supported channel addresses (can include wildcards)
- `getPrediction(ChannelAddress)` - Returns prediction for channel

#### Prediction Class
**Purpose:** Holds quarterly predictions (one value per 15 minutes)

**Features:**
- Immutable sorted map of timestamp → value
- Value unit matches source channel (e.g., Watt for power)
- Static factory methods for construction
- Value range validation (min/max)

**Static Methods:**
- `from(Instant time, Integer... values)` - Create from array
- `from(ImmutableSortedMap)` - Create from map
- `sum(Prediction...)` - Sum multiple predictions

**Typical Channels Predicted:**
- `_sum/ProductionActivePower` - PV production
- `_sum/ConsumptionActivePower` - Total consumption
- `_sum/UnmanagedConsumptionActivePower` - Uncontrolled consumption

### Prediction Algorithms

#### 1. LSTM (Long Short-Term Memory) Predictor
**Module:** `io.openems.edge.predictor.lstm`

**Algorithm:** Recurrent Neural Network (RNN) with LSTM cells

**Capabilities:**
- Captures time-series dependencies and patterns
- Learns daily and seasonal variations
- Incorporates external factors (weather, day of week, holidays)

**Training:**
- Input: Historical consumption/production data
- Pre-processing: Scaling and normalization
- Training: Backpropagation Through Time (BPTT)
- Schedule: Every 45 days (30 days training, 15 days validation)

**Data Storage:**
- Folder: `lstm` in OpenEMS data directory
- Initially uses generic model
- Automatically updated with trained models

**Use Cases:**
- Complex consumption patterns
- Long-term dependencies
- Weather-dependent production

**Advantages:**
- High accuracy for complex patterns
- Self-learning and adaptive

**Considerations:**
- Requires historical data
- Training computational overhead
- 45-day update cycle

#### 2. Similar Day Model Predictor
**Module:** `io.openems.edge.predictor.similardaymodel`

**Algorithm:** Averages same weekday from previous weeks

**Configuration:**
- Number of past weeks (n)
- Channel address to predict

**Method:**
- Monday prediction = average of last n Mondays
- Tuesday prediction = average of last n Tuesdays
- etc.

**Features:**
- Handles daylight saving time (1-hour offset)
- Scientifically verified (EMSIG project)

**Use Cases:**
- Regular weekly patterns
- Consumption prediction
- Moderate accuracy requirements

**Advantages:**
- Simple and interpretable
- No training required
- Works immediately with data

**Considerations:**
- Requires n weeks of history
- Less accurate for irregular patterns

#### 3. Persistence Model Predictor
**Module:** `io.openems.edge.predictor.persistencemodel`

**Algorithm:** "Same as last day"

**Method:**
- Predicts same values as 24 hours ago

**Use Cases:**
- Fallback predictor
- Very stable patterns
- Testing/baseline

**Advantages:**
- Extremely simple
- No configuration
- Always available

**Considerations:**
- Low accuracy for varying patterns
- Day-of-week variations ignored

#### 4. Profile Clustering Model Predictor
**Module:** `io.openems.edge.predictor.profileclusteringmodel`

**Algorithm:** Clustering of daily load profiles

**Method:**
1. Cluster historical daily profiles
2. Identify patterns (work day, weekend, holiday, etc.)
3. Predict next day's profile based on:
   - Upcoming day type
   - Previous profile
   - Profile transition probabilities

**Use Cases:**
- Household-specific patterns
- Consumption with distinct profile types
- Improved accuracy over simple averaging

**Advantages:**
- Household-specific learning
- Captures profile variations

**Considerations:**
- Requires sufficient historical data
- Clustering overhead

#### 5. Linear Model Production Predictor
**Module:** `io.openems.edge.predictor.production.linearmodel`

**Algorithm:** Linear regression model

**Use Cases:**
- PV production prediction
- Weather-based forecasting

**Method:**
- Regression on historical production data
- Factor in irradiance, temperature, etc.

### Predictor Manager

**Purpose:** Central service managing all Predictors

**Responsibilities:**
- Register/unregister Predictors
- Query predictions by channel address
- Aggregate multiple predictor results

**Used By:**
- Energy Scheduler
- Controllers requiring forecasts

---

## Time-of-Use Tariff Modules

### Location
- `io.openems.edge.timeofusetariff.api` - Tariff API
- Provider implementations:
  - `io.openems.edge.timeofusetariff.awattar` - aWATTar
  - `io.openems.edge.timeofusetariff.corrently` - Corrently by STROMDAO
  - `io.openems.edge.timeofusetariff.tibber` - Tibber
  - `io.openems.edge.timeofusetariff.entsoe` - ENTSO-E
  - `io.openems.edge.timeofusetariff.manual` - Manual configuration
  - `io.openems.edge.timeofusetariff.ancillarycosts` - Ancillary costs wrapper
  - Others (EWS, Hassfurt, LUOX, Groupe-E, Swisspower, rabot.charge)

### Time-of-Use Tariff API

#### TimeOfUseTariff Interface
**Purpose:** Provides quarterly electricity prices

```java
public interface TimeOfUseTariff {
    TimeOfUsePrices getPrices();
}
```

**Key Method:**
- `getPrices()` - Returns prices for next periods (15-minute intervals)

#### TimeOfUsePrices Class
**Purpose:** Holds quarterly price data

**Features:**
- Unit: Currency/MWh (matches `_meta/Currency`)
- One value per 15 minutes
- Immutable sorted map
- Starting from current quarter

**Static Methods:**
- `from(Instant time, Double... values)` - Create from array
- `from(ImmutableSortedMap)` - Create from map
- `from(Instant, TimeOfUsePrices)` - Update prices to current time

**Example:**
If called at 10:05:
- First value: 10:00-10:15
- Second value: 10:15-10:30
- etc.

### Tariff Provider Implementations

#### 1. aWATTar (`io.openems.edge.timeofusetariff.awattar`)
**Provider:** aWATTar (Austria/Germany)

**Update Schedule:** Daily at 14:00

**Method:**
- Retrieves hourly prices from aWATTar API
- Converts to quarterly prices (15-minute intervals)
- Stores locally for reliability

**Regions:** Austria, Germany

#### 2. Corrently (`io.openems.edge.timeofusetariff.corrently`)
**Provider:** STROMDAO Corrently

**Features:**
- Regional electricity prices
- GrünstromIndex integration

**Regions:** Germany (postal code based)

#### 3. Tibber (`io.openems.edge.timeofusetariff.tibber`)
**Provider:** Tibber

**Features:**
- Real-time pricing
- Nordic electricity markets

**Regions:** Norway, Sweden, Germany, Netherlands

#### 4. ENTSO-E (`io.openems.edge.timeofusetariff.entsoe`)
**Provider:** European Network of Transmission System Operators

**Features:**
- Day-ahead market prices
- Multiple European countries

**Regions:** Most European countries

#### 5. Manual (`io.openems.edge.timeofusetariff.manual`)
**Provider:** User-configured

**Configuration:**
- Manually defined price periods
- Static or recurring schedules

**Use Cases:**
- Testing
- Fixed tariff structures
- Custom pricing scenarios

#### 6. Ancillary Costs (`io.openems.edge.timeofusetariff.ancillarycosts`)
**Provider:** Wrapper for other providers

**Purpose:** Add fixed costs to base tariff prices

**Configuration:**
- Base tariff provider
- Additional costs per kWh

**Use Cases:**
- Grid fees
- Taxes
- Other fixed charges

### Tariff Integration

#### Controller: Time-of-Use Tariff Controller
**Module:** `io.openems.edge.controller.ess.timeofusetariff`

**Purpose:** Controls ESS based on electricity prices

**Channels:**
- `STATE_MACHINE` - Current control state
- `QUARTERLY_PRICES` - Current period price
- `CHARGED_TIME` - Seconds in forced charge mode
- `DELAYED_TIME` - Seconds with delayed discharge

**Modes:**
1. **CHARGE_FROM_GRID** - Force charge when prices are low
2. **DELAY_DISCHARGE** - Block discharge when prices are low
3. **BALANCING** - Normal operation, balance with grid

**State Machine:**
- Analyzes price profile
- Determines optimal charge/discharge windows
- Considers ESS SOC and capacity
- Applies mode based on schedule

**Integration with Energy Scheduler:**
- Implements `EnergySchedulable` interface
- Provides `EnergyScheduleHandler`
- Participates in global optimization

---

## Integration and Workflow

### Complete Energy Management Flow

#### 1. Prediction Phase
```
PredictorManager
  ├─ LSTM Predictor → Production forecast
  ├─ Similar Day Predictor → Consumption forecast
  └─ Aggregated predictions ready
```

#### 2. Price Data Phase
```
TimeOfUseTariff Provider (e.g., aWATTar)
  ├─ Fetch prices from API
  ├─ Convert to 15-minute intervals
  └─ Prices available for optimization
```

#### 3. Optimization Phase (Every 15 Minutes)
```
EnergyScheduler (Optimizer)
  ├─ GlobalOptimizationContext creation
  │   ├─ Production predictions
  │   ├─ Consumption predictions
  │   ├─ Price data
  │   ├─ ESS state (SOC, capacity)
  │   ├─ Grid constraints
  │   └─ EnergyScheduleHandlers
  │
  ├─ Quick Optimization (1 generation)
  │   └─ Immediate schedule available
  │
  └─ Regular Optimization (multiple generations)
      ├─ Genetic algorithm evolution
      ├─ Fitness evaluation (cost minimization)
      └─ Best schedule selected
```

#### 4. Schedule Application
```
Best Schedule
  └─ For each Controller/EnergyScheduleHandler
      ├─ applySchedule() called
      └─ Mode set for current period
```

#### 5. Controller Execution Phase (Every Second)
```
Scheduler (e.g., FixedOrder)
  └─ Returns ordered Controller list
      │
      └─ For each Controller (in order)
          ├─ Controller.run()
          └─ Sets power constraints on ESS
              │
              └─ Power Component
                  ├─ Resolves all constraints
                  ├─ Applies solver strategy
                  └─ Writes final power to ESS
```

#### 6. ESS Execution
```
ManagedSymmetricEss
  ├─ Receives power setpoints
  ├─ Hardware controller execution
  └─ Actual power measurement feedback
```

### Example Scenario: Time-of-Use Tariff Optimization

**Goal:** Minimize electricity costs using price-aware charging

**Setup:**
- ESS: 10 kWh capacity, 5 kW max power
- Tariff: aWATTar with variable hourly prices
- Predictors: LSTM for production, Similar Day for consumption

**Execution:**

1. **14:05** - aWATTar updates prices for next day
   - Low prices: 02:00-06:00 (€0.15/kWh)
   - High prices: 17:00-21:00 (€0.35/kWh)

2. **15:00** - Energy Scheduler optimizes
   - Predicts production: 5 kW average (12:00-16:00)
   - Predicts consumption: 2 kW base load
   - Creates schedule:
     - 02:00-06:00: CHARGE_FROM_GRID (utilize low prices)
     - 12:00-16:00: Allow PV charging
     - 17:00-21:00: Discharge to grid (utilize high prices)
     - Other times: BALANCING (self-consumption)

3. **Every Second** - Controller execution
   - 02:30: TimeOfUseTariff Controller runs
     - Current mode: CHARGE_FROM_GRID
     - Sets: `SET_ACTIVE_POWER_EQUALS = -5000` (5 kW charge)
   
   - 17:30: TimeOfUseTariff Controller runs
     - Current mode: Discharge allowed
     - Other controllers (Balancing) set power
     - ESS discharges to grid during high price period

4. **Result**
   - Charged 20 kWh at €0.15/kWh = €3.00
   - Discharged 18 kWh at €0.35/kWh = €6.30
   - Net savings vs. no optimization: ~€2.50/day

### Key Design Principles

#### 1. Modularity
- Clear interface separation (API modules)
- Pluggable implementations
- OSGi dynamic service loading

#### 2. Prediction-Based
- All optimization uses forecasts
- Multiple predictor algorithms available
- Continuous learning (LSTM auto-training)

#### 3. Constraint-Based Power Management
- Linear constraint system
- Multiple solver strategies
- Guaranteed constraint satisfaction

#### 4. Genetic Algorithm Optimization
- Multi-objective fitness function
- Balance between cost, wear, risk
- Configurable execution time

#### 5. Real-Time Adaptation
- 15-minute optimization cycles
- Quick optimization for immediate response
- Rescheduling on configuration changes

### Configuration Best Practices

#### 1. Predictor Selection
- **LSTM**: For complex, varying patterns (requires 45 days)
- **Similar Day**: For regular weekly patterns (requires n weeks)
- **Persistence**: For very stable patterns or fallback

#### 2. Risk Level Configuration
- **LOW**: Conservative, prioritize self-consumption
- **MEDIUM**: Balanced optimization
- **HIGH**: Aggressive, maximize cost savings

#### 3. Tariff Provider Selection
- Use provider matching your electricity contract
- Consider ancillary costs wrapper for grid fees
- Manual tariff for testing

#### 4. Scheduler Strategy
- **Fixed Order**: For simple, static priorities
- **Daily**: For time-varying control strategies
- Consider Controller dependencies

### Monitoring and Debugging

#### Key Channels to Monitor

1. **ESS:**
   - `SOC` - Battery state
   - `ACTIVE_POWER` - Current power flow
   - `ALLOWED_CHARGE/DISCHARGE_POWER` - Battery limits

2. **Energy Scheduler:**
   - `SIMULATIONS_PER_QUARTER` - Optimization activity
   - `GENERATIONS_PER_QUARTER` - Algorithm progress

3. **Controllers:**
   - `STATE_MACHINE` - Current operation mode
   - `QUARTERLY_PRICES` - Active electricity price

4. **Predictors:**
   - Check predictions vs. actual values
   - Monitor prediction quality

#### Log Verbosity
- Configure `logVerbosity` in Energy Scheduler
- Levels: NONE, INFO, DEBUG, TRACE
- Trace shows detailed simulation results

### Performance Considerations

#### Optimization Execution Time
- Auto-calculated based on schedule
- Targets: ~10 seconds per quarter
- More time = better optimization quality

#### Predictor Performance
- LSTM: Requires training time every 45 days
- Similar Day: Lightweight, instant predictions
- Profile Clustering: Moderate computational cost

#### Memory Usage
- Energy schedules held in memory
- Prediction data cached
- Historical data for training

---

## Conclusion

OpenEMS provides a sophisticated, modular energy management system with:

1. **Flexible ESS Integration**: Multiple interface levels from basic symmetric to complex hybrid systems

2. **Intelligent Optimization**: Genetic algorithm-based scheduling with prediction and tariff awareness

3. **Multiple Prediction Algorithms**: From simple persistence to advanced LSTM neural networks

4. **Dynamic Pricing Integration**: Support for numerous European tariff providers

5. **Real-Time Adaptation**: 15-minute optimization cycles with immediate rescheduling capability

The architecture enables sophisticated energy management strategies while maintaining modularity and extensibility for future enhancements.

### References

- **Source Code**: https://github.com/OpenEMS/openems
- **Jenetics Documentation**: https://jenetics.io/
- **EMSIG Project**: https://openems.io/research/emsig/
- **LSTM Information**: https://en.wikipedia.org/wiki/Long_short-term_memory

### Related Documentation
- [BACKEND_MODULES_DOCUMENTATION.md](BACKEND_MODULES_DOCUMENTATION.md)
- [BRIDGE_MODULES_DOCUMENTATION.md](BRIDGE_MODULES_DOCUMENTATION.md)
- [openems_CORE_ARCHITECTURE.md](openems_CORE_ARCHITECTURE.md)
- [openems_README_COMPREHENSIVE.md](openems_README_COMPREHENSIVE.md)
