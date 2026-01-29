# OpenEMS Core Architecture & Logic

## Table of Contents
- [Overview](#overview)
- [Input-Process-Output (IPO) Model](#input-process-output-ipo-model)
- [Process Image Pattern](#process-image-pattern)
- [Channel System](#channel-system)
- [Nature-Based Abstraction](#nature-based-abstraction)
- [Controller Execution & Prioritization](#controller-execution--prioritization)
- [Scheduler Architecture](#scheduler-architecture)
- [Cycle Management](#cycle-management)
- [Asynchronous Communication](#asynchronous-communication)
- [Bridge Pattern](#bridge-pattern)
- [ESS Power Distribution](#ess-power-distribution)
- [Component Lifecycle](#component-lifecycle)

---

## Overview

OpenEMS Edge is built on proven industrial control principles adapted for energy management. The architecture borrows heavily from **PLC (Programmable Logic Controller)** programming patterns to ensure predictable, reliable, and real-time-capable energy management.

### Core Principles

1. **Deterministic Execution**: Fixed cycle-based execution ensures predictable timing
2. **Data Consistency**: Process Image guarantees immutable data during each cycle
3. **Prioritization**: Scheduler ensures higher-priority controls take precedence
4. **Abstraction**: Device-independent algorithms through Nature interfaces
5. **Modularity**: OSGi-based plugin architecture for easy extensibility

---

## Input-Process-Output (IPO) Model

The IPO model is the **heartbeat** of OpenEMS Edge, executing continuously in a fixed cycle (typically 1 second).

```
┌─────────────────────────────────────────────────────────────┐
│                    OpenEMS Edge Cycle                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌────────────┐      ┌────────────┐      ┌────────────┐   │
│  │   INPUT    │  ──► │  PROCESS   │  ──► │   OUTPUT   │   │
│  └────────────┘      └────────────┘      └────────────┘   │
│       │                    │                    │           │
│       │                    │                    │           │
│  Read Sensors      Execute Controllers    Write Setpoints  │
│  Create Process    (Scheduler orders       Apply to        │
│  Image            Controllers)             Devices          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Phase 1: INPUT

**Purpose**: Collect all sensor data and create an immutable snapshot

```java
// Simplified pseudo-code
void inputPhase() {
    // Read from all Modbus devices
    modbusProtocol.readAllRegisters();
    
    // Read from all MQTT topics
    mqttSubscriber.processMessages();
    
    // Read from all HTTP endpoints
    httpBridge.fetchData();
    
    // CRITICAL: Switch Process Image
    channelManager.switchProcessImage();
    // After this point, all Channel.value fields are frozen for this cycle
}
```

**Key Actions**:
- Trigger asynchronous read operations on all bridges (Modbus, MQTT, HTTP, etc.)
- Wait for read operations to complete (with timeout)
- **Switch Process Image**: Copy `nextValue` → `value` for all Channels
- Process Image is now **immutable** for the remainder of the cycle

### Phase 2: PROCESS

**Purpose**: Execute business logic using the frozen Process Image

```java
void processPhase() {
    // Get sorted list of Controllers from Scheduler
    List<Controller> controllers = scheduler.getControllers();
    
    // Execute each Controller in priority order
    for (Controller controller : controllers) {
        try {
            controller.run();
            // Controller reads from Channel.value (Process Image)
            // Controller writes to Channel.nextValue or setpoint channels
        } catch (Exception e) {
            log.error("Controller {} failed: {}", controller.id(), e);
        }
    }
}
```

**Key Actions**:
- Scheduler provides ordered list of Controllers
- Each Controller executes sequentially
- Controllers read from **frozen** Process Image (`Channel.value`)
- Controllers write setpoints to special write Channels
- Later Controllers cannot override earlier Controllers (if configured)

### Phase 3: OUTPUT

**Purpose**: Apply all setpoints to physical devices

```java
void outputPhase() {
    // Write all setpoints to Modbus devices
    modbusProtocol.writeAllRegisters();
    
    // Publish to MQTT topics
    mqttPublisher.publishAll();
    
    // Send HTTP requests
    httpBridge.sendCommands();
    
    // Digital outputs (relays, etc.)
    digitalOutputs.apply();
}
```

**Key Actions**:
- Trigger asynchronous write operations to all devices
- Non-blocking (fire and forget with error handling)
- Next cycle will verify if commands were successful

---

## Process Image Pattern

### Problem Statement

In multi-threaded systems with asynchronous communication, data can change at any moment:

```java
// WITHOUT Process Image - DANGEROUS!
int gridPower = gridMeter.getActivePower();  // Read: 1000 W
// ... 50 lines of code ...
if (gridPower > 0) {  // gridPower might now be -2000 W!
    chargeBattery();  // Wrong decision!
}
```

### Solution: Process Image

```java
// Channel implementation
public class IntegerChannel {
    private Integer value;      // Current (frozen in Process Image)
    private Integer nextValue;  // Latest from device
    
    // Called by device drivers (async threads)
    public void setNextValue(Integer newValue) {
        this.nextValue = newValue;
    }
    
    // Called once per cycle during INPUT phase
    void switchProcessImage() {
        this.value = this.nextValue;  // Atomic copy
    }
    
    // Controllers read this (guaranteed stable)
    public Integer getValue() {
        return this.value;
    }
}
```

### Benefits

1. **No Race Conditions**: Data cannot change mid-cycle
2. **Simplified Logic**: Developers don't need synchronization primitives
3. **Reproducible Behavior**: Same inputs always produce same outputs
4. **Easier Debugging**: Single snapshot to analyze

### Visual Timeline

```
Time ─────────────────────────────────────────────────────►

Cycle N-1        │  Cycle N                      │  Cycle N+1
                 │                               │
─────────────────┼───────────────────────────────┼──────────────
Device Thread    │                               │
(async):         │    nextValue = 1500 W        │
  Read Modbus ───┼───►(async, anytime)          │
                 │                               │
─────────────────┼───────────────────────────────┼──────────────
Cycle Thread:    │                               │
                 │                               │
  INPUT:         │  value = nextValue (1500 W)  │
                 │  ▲ PROCESS IMAGE CREATED     │
                 │                               │
  PROCESS:       │  Controllers read value      │
                 │  (always see 1500 W)         │
                 │                               │
  OUTPUT:        │  Write setpoints             │
                 │                               │
─────────────────┼───────────────────────────────┼──────────────
```

---

## Channel System

Channels are the **fundamental data abstraction** in OpenEMS. Every sensor value, configuration parameter, and setpoint is a Channel.

### Channel Hierarchy

```java
// Base interface
public interface Channel<T> {
    String address();           // e.g., "ess0/Soc"
    T value();                  // Current value (Process Image)
    Optional<T> getNextValue(); // Latest value
    void setNextValue(T value); // Set by device drivers
}

// Specialized types
public interface IntegerReadChannel extends Channel<Integer> { }
public interface IntegerWriteChannel extends IntegerReadChannel {
    void setNextWriteValue(Integer value);  // Set by Controllers
}
```

### Channel Categories

| Category | Read/Write | Examples | Usage |
|----------|-----------|----------|-------|
| **Status Channels** | Read-only | Battery SoC, Grid Power, Temperature | Monitoring |
| **Configuration Channels** | Read-only | Max Charge Power, Component ID | Settings |
| **Control Channels** | Write-only | Setpoint Active Power, Relay State | Actuation |
| **State Machine Channels** | Read/Write | Operating State, Mode | State Management |

### Channel Address Convention

```
<component-id>/<channel-name>

Examples:
- ess0/Soc                    → Battery State of Charge
- meter0/ActivePower          → Grid meter active power
- ctrlBalancing0/Enabled      → Controller enabled state
- _sum/GridActivePower        → Virtual channel (sum of all grid meters)
```

### Channel Implementation Example

```java
@Component
public class SimulatorEss extends AbstractOpenemsComponent implements Ess {
    
    // Channel definitions
    public final ModbusSlaveTable table = ModbusSlaveTable.builder()
        .channel(0, EssChannelId.SOC, UINT16)
        .channel(1, EssChannelId.ACTIVE_CHARGE_ENERGY, UINT32)
        .channel(3, EssChannelId.SET_ACTIVE_POWER_EQUALS, SINT16)
        .build();
    
    // Channel access
    public IntegerReadChannel getSocChannel() {
        return this.channel(EssChannelId.SOC);
    }
    
    // Process simulation
    @Override
    public void run() {
        // Update State of Charge based on power flow
        int currentSoc = this.getSoc().orElse(50);
        int power = this.getActivePower().orElse(0);
        int newSoc = simulateSocChange(currentSoc, power);
        
        // Write to nextValue (will be picked up next cycle)
        this._setSoc(newSoc);
    }
}
```

---

## Nature-Based Abstraction

**Nature** = Interface defining capabilities of a device type

### Core Concept

Instead of Controllers knowing about specific devices (e.g., "FENECON Commercial 40"), they work with **Natures** (e.g., "any ESS").

```java
// Bad: Tight coupling
public class BalancingController {
    @Reference
    private FeneconCommercial40 battery;  // Only works with one brand!
    
    public void run() {
        battery.setChargePower(1000);
    }
}

// Good: Abstraction via Nature
public class BalancingController {
    @Reference
    private ManagedSymmetricEss ess;  // Works with ANY ESS!
    
    public void run() {
        ess.setActivePowerEquals(1000);  // Standard interface
    }
}
```

### Common Natures

```java
// Energy Storage System
public interface ManagedSymmetricEss extends SymmetricEss {
    IntegerReadChannel getSoc();           // State of Charge (%)
    IntegerReadChannel getCapacity();      // Capacity (Wh)
    IntegerWriteChannel setActivePowerEquals();  // Setpoint (W)
    default void setActivePower(int power) { ... }
}

// Power Meter
public interface SymmetricMeter extends Meter {
    IntegerReadChannel getActivePower();   // Current power (W)
    IntegerReadChannel getReactivePower(); // Reactive power (var)
}

// EV Charging Station
public interface Evcs extends OpenemsComponent {
    IntegerReadChannel getChargePower();   // Current charging (W)
    IntegerWriteChannel setChargePowerLimit();  // Max power limit
    BooleanReadChannel getPlugged();       // Cable connected?
}
```

### Nature Composition

Devices can implement **multiple Natures**:

```java
@Component
public class FeneconCommercial40 
    implements 
        ManagedSymmetricEss,      // It's a controllable battery
        SymmetricMeter,           // It measures its own power
        AsymmetricEss {           // It supports per-phase control
    
    // Implements all interface methods
}
```

### Benefits

1. **Device Independence**: Write controller once, works with all compatible devices
2. **Substitutability**: Swap hardware without code changes
3. **Testability**: Mock Natures for unit testing
4. **Evolution**: Add new devices without modifying existing code

---

## Controller Execution & Prioritization

### The Priority Problem

```
Scenario: Two Controllers want to control the same battery

Controller A (Priority 1): "Charge at 5000 W"  (emergency reserve)
Controller B (Priority 2): "Discharge at 3000 W"  (self-consumption)

Question: What should the battery do?
Answer: Charge at 5000 W (higher priority wins)
```

### Scheduler Role

The Scheduler determines **execution order** and **priority**.

```java
public interface Scheduler extends OpenemsComponent {
    /**
     * Returns Controllers in execution order (high priority first)
     */
    List<Controller> getControllers();
}
```

### Example: Fixed Order Scheduler

```java
@Component
public class FixedOrderScheduler implements Scheduler {
    
    @Reference(cardinality = MULTIPLE)
    private List<Controller> controllers;
    
    @Override
    public List<Controller> getControllers() {
        // Return in order: ctrlA, ctrlB, ctrlC
        return controllers.stream()
            .sorted(Comparator.comparing(c -> c.id()))
            .collect(Collectors.toList());
    }
}
```

Configuration:
```
Scheduler ID: scheduler0
Controller IDs: 
  1. ctrlLimitDischarge    (highest priority - safety)
  2. ctrlBackend           (remote control)
  3. ctrlPeakShaving       (grid limit)
  4. ctrlBalancing         (optimization)
```

### Write-Once Semantics

Controllers that execute later **cannot override** earlier decisions (for write channels with this behavior):

```java
// Simplified ESS Power Channel implementation
public class EssPowerChannel {
    private Integer requestedPower = null;
    private boolean isLocked = false;
    
    public void setNextWriteValue(Integer power) {
        if (!isLocked) {
            this.requestedPower = power;
            this.isLocked = true;  // First write wins
        }
    }
    
    void unlockForNextCycle() {
        this.isLocked = false;
    }
}

// Execution:
ctrlLimitDischarge.run();  // Sets power to 5000 W, locks channel
ctrlBalancing.run();        // Tries to set -3000 W, IGNORED
// Result: Battery charges at 5000 W
```

---

## Scheduler Architecture

### Built-in Schedulers

#### 1. All Alphabetically Scheduler
```java
// Simplest: Execute all Controllers alphabetically by ID
List<Controller> getControllers() {
    return controllers.stream()
        .sorted((a, b) -> a.id().compareTo(b.id()))
        .collect(Collectors.toList());
}
```

#### 2. Fixed Order Scheduler
```java
// Explicit priority via configuration
List<Controller> getControllers() {
    return configuredOrder;  // e.g., [ctrlA, ctrlC, ctrlB]
}
```

#### 3. Daily Scheduler
```java
// Time-based activation
List<Controller> getControllers() {
    LocalTime now = LocalTime.now();
    return controllers.stream()
        .filter(c -> isActiveAtTime(c, now))
        .collect(Collectors.toList());
}

// Config:
// ctrlNightCharge: active 22:00-06:00
// ctrlDayOptimize: active 06:00-22:00
```

### Custom Scheduler Example

```java
@Component
public class DynamicPriorityScheduler implements Scheduler {
    
    @Reference
    private ManagedSymmetricEss ess;
    
    @Override
    public List<Controller> getControllers() {
        int soc = ess.getSoc().orElse(50);
        
        // Low battery: prioritize charging controllers
        if (soc < 20) {
            return List.of(
                ctrlEmergencyCharge,
                ctrlNormalControllers...
            );
        }
        
        // Normal priority
        return normalOrder;
    }
}
```

---

## Cycle Management

### Cycle Implementation

Located in: `io.openems.edge.core/src/io/openems/edge/core/cycle/CycleImpl.java`

```java
@Component
public class CycleImpl implements Cycle {
    
    private static final int DEFAULT_CYCLE_TIME = 1000; // ms
    
    @Reference
    private ComponentManager componentManager;
    
    @Reference
    private Scheduler scheduler;
    
    @Activate
    void activate() {
        // Start cycle worker thread
        this.worker = new CycleWorker(this);
        this.worker.start();
    }
    
    class CycleWorker extends Thread {
        @Override
        public void run() {
            while (!isInterrupted()) {
                long cycleStart = System.currentTimeMillis();
                
                try {
                    // INPUT PHASE
                    eventAdmin.postEvent(new Event(INPUT_START));
                    executeInputPhase();
                    eventAdmin.postEvent(new Event(INPUT_FINISHED));
                    
                    // PROCESS PHASE
                    eventAdmin.postEvent(new Event(PROCESS_START));
                    executeProcessPhase();
                    eventAdmin.postEvent(new Event(PROCESS_FINISHED));
                    
                    // OUTPUT PHASE
                    eventAdmin.postEvent(new Event(OUTPUT_START));
                    executeOutputPhase();
                    eventAdmin.postEvent(new Event(OUTPUT_FINISHED));
                    
                } catch (Exception e) {
                    log.error("Cycle error: {}", e.getMessage());
                }
                
                // Sleep until next cycle
                long duration = System.currentTimeMillis() - cycleStart;
                long sleep = DEFAULT_CYCLE_TIME - duration;
                if (sleep > 0) {
                    Thread.sleep(sleep);
                } else {
                    log.warn("Cycle took {}ms (target: {}ms)", 
                        duration, DEFAULT_CYCLE_TIME);
                }
            }
        }
    }
}
```

### Cycle Events

Components can listen to Cycle Events to synchronize:

```java
@Component
public class ModbusBridge {
    
    @Reference
    private EventAdmin eventAdmin;
    
    @Activate
    void activate() {
        // Subscribe to cycle events
        eventAdmin.subscribe(new EventHandler() {
            public void handleEvent(Event event) {
                String topic = event.getTopic();
                
                switch (topic) {
                    case "org/openems/edge/cycle/INPUT_START":
                        // Start asynchronous Modbus reads
                        triggerReads();
                        break;
                        
                    case "org/openems/edge/cycle/OUTPUT_START":
                        // Start asynchronous Modbus writes
                        triggerWrites();
                        break;
                }
            }
        }, "org/openems/edge/cycle/*");
    }
}
```

### Cycle Timing Analysis

```
Typical Cycle Breakdown (1000ms total):

├─ INPUT:    200ms  (20%)  │ Read all Modbus devices
├─ PROCESS:  100ms  (10%)  │ Execute 20 Controllers
├─ OUTPUT:   150ms  (15%)  │ Write all setpoints
└─ IDLE:     550ms  (55%)  │ Sleep until next cycle

If cycle takes > 1000ms: WARNING logged
If cycle regularly exceeds: Consider reducing cycle time or optimizing
```

---

## Asynchronous Communication

### Challenge

Modbus, MQTT, HTTP are **slow** and **blocking**. Reading 50 Modbus registers might take 500ms. We cannot block the cycle!

### Solution: Async Threads + Cycle Sync

```java
public class ModbusBridge {
    
    private ExecutorService executor = Executors.newCachedThreadPool();
    private Map<String, ModbusTask> tasks = new ConcurrentHashMap<>();
    
    // Called at INPUT_START event
    public void triggerReads() {
        for (ModbusTask task : tasks.values()) {
            // Submit async read
            executor.submit(() -> {
                try {
                    // This runs in separate thread
                    byte[] response = modbusClient.read(
                        task.unitId, 
                        task.startAddress, 
                        task.length
                    );
                    
                    // Update Channel.nextValue (thread-safe)
                    task.updateChannels(response);
                    
                } catch (IOException e) {
                    log.error("Modbus read failed: {}", e);
                }
            });
        }
    }
    
    // Called at INPUT_FINISHED event (or with timeout)
    public void waitForReads() {
        // Wait for all reads to complete (max 200ms)
        executor.awaitTermination(200, TimeUnit.MILLISECONDS);
    }
}
```

### Timeline Visualization

```
Cycle N:
0ms   ─── INPUT_START event fired
      ├─► Modbus: Submit 10 read tasks (async)
      ├─► MQTT: Check message queue
      └─► HTTP: Submit API call (async)
      
100ms ─── Wait for async operations...
      ├─► Modbus task 1 completes → updates nextValue
      ├─► Modbus task 2 completes → updates nextValue
      ...
      
200ms ─── INPUT_FINISHED event (timeout or all complete)
      ├─► Switch Process Image (nextValue → value)
      
300ms ─── PROCESS_START
      ├─► Execute Controllers (use frozen 'value')
      
400ms ─── PROCESS_FINISHED
      
500ms ─── OUTPUT_START
      ├─► Modbus: Submit 5 write tasks (async)
      └─► MQTT: Publish messages
      
600ms ─── OUTPUT_FINISHED
      
1000ms ─── Cycle N ends, Cycle N+1 starts
```

---

## Bridge Pattern

Bridges abstract **protocol-specific communication** from device logic.

### Architecture

```
Device Component (e.g., FeneconEss)
         │
         │ Uses
         ▼
Bridge Component (e.g., ModbusBridge)
         │
         │ Implements
         ▼
Physical Protocol (Modbus TCP)
         │
         ▼
Hardware Device
```

### Example: Modbus Bridge

```java
@Component
public interface ModbusBridge extends OpenemsComponent {
    /**
     * Register a protocol for execution
     */
    void addProtocol(String sourceId, ModbusProtocol protocol);
    
    /**
     * Remove protocol when component deactivates
     */
    void removeProtocol(String sourceId);
}

// Device uses bridge
@Component
public class SimulatorGridMeter implements SymmetricMeter {
    
    @Reference
    private ModbusBridge modbus;
    
    @Activate
    void activate(Config config) {
        // Define Modbus registers to read
        ModbusProtocol protocol = new ModbusProtocol(
            new FC3ReadRegistersTask(0, Priority.HIGH,
                m(GridMeterChannelId.ACTIVE_POWER, new SignedWordElement(0)),
                m(GridMeterChannelId.REACTIVE_POWER, new SignedWordElement(1))
            )
        );
        
        // Register with bridge
        modbus.addProtocol(this.id(), protocol);
    }
}
```

### Available Bridges

| Bridge | Protocol | Use Case | Example Devices |
|--------|----------|----------|-----------------|
| `ModbusBridge` | Modbus TCP/RTU | Industrial devices | Inverters, meters, PLCs |
| `MqttBridge` | MQTT | IoT sensors | Temperature, occupancy |
| `HttpBridge` | HTTP/REST | Web APIs | Weather, pricing, cloud |
| `MbusBridge` | M-Bus | Utility meters | Heat meters, water meters |
| `OneWireBridge` | 1-Wire | Simple sensors | DS18B20 temperature |

---

## ESS Power Distribution

### The Power Constraint Problem

```
Scenario:
- Battery: Max charge = 10 kW, Max discharge = 10 kW
- Current SoC: 10% (near empty)
- Controllers:
  1. LimitDischarge: "Force charge at least 2 kW"
  2. PeakShaving: "Discharge 8 kW to reduce grid peak"
  
Problem: Conflicting constraints!
- Must charge ≥ 2 kW
- Want to discharge 8 kW
- These are mutually exclusive

Solution: Prioritization + Constraint Solver
```

### ESS Power Component

Special component manages ESS power constraints:

```java
@Component
public class EssPower {
    
    // Constraints from sequential Controllers
    private List<Constraint> constraints = new ArrayList<>();
    
    /**
     * Add constraint from a Controller
     */
    public void addConstraint(Constraint c) {
        constraints.add(c);
    }
    
    /**
     * Solve constraints and determine feasible setpoint
     */
    public int solve() {
        // Build constraint system:
        // -10000 ≤ power ≤ 10000    (battery limits)
        // power ≤ -2000               (force charge, high priority)
        // power = 8000                (discharge request, low priority)
        
        // Solution: power = -2000 (satisfy high priority constraint)
        
        LinearProgramSolver solver = new LinearProgramSolver();
        for (Constraint c : constraints) {
            solver.addConstraint(c);
        }
        
        return solver.solve();  // Returns: -2000 W (charge)
    }
}
```

### Visual Example

```
Power Range Reduction:

Initial:     [-10000 W ............. +10000 W]  (discharge ← → charge)
                      Full battery range

After Ctrl 1: [-10000 W ........... -2000 W]    Force charge constraint
                                         ↑ 
                                    Must be here or lower

After Ctrl 2:          [-2000 W]                Final setpoint
                           ↑
                      Best effort to discharge 8kW,
                      but constrained to -2kW charge
```

---

## Component Lifecycle

### OSGi Declarative Services

OpenEMS components follow OSGi DS lifecycle:

```java
@Component(
    name = "Controller.Ess.Balancing",
    immediate = true,
    configurationPolicy = REQUIRE
)
public class BalancingController 
    extends AbstractOpenemsComponent 
    implements Controller {
    
    @Reference
    private ComponentManager componentManager;
    
    @Reference
    private Ess ess;  // Injected by OSGi
    
    @Activate
    void activate(ComponentContext context, Config config) {
        super.activate(context, config.id(), config.alias(), config.enabled());
        // Component is now active
    }
    
    @Deactivate
    void deactivate() {
        super.deactivate();
        // Cleanup resources
    }
    
    @Modified
    void modified(ComponentContext context, Config config) {
        super.modified(context, config.id(), config.alias(), config.enabled());
        // Configuration changed
    }
    
    @Override
    public void run() throws OpenemsNamedException {
        // Executed every cycle by Scheduler
    }
}
```

### Lifecycle States

```
       ┌─────────────┐
       │ UNSATISFIED │  (Missing dependencies)
       └──────┬──────┘
              │
       @Reference resolved
              │
       ┌──────▼──────┐
       │  SATISFIED  │  (Ready to activate)
       └──────┬──────┘
              │
       @Activate called
              │
       ┌──────▼──────┐
       │   ACTIVE    │  ←──┐
       └──────┬──────┘     │
              │             │
       Config changed       │
              │             │
              └─ @Modified ─┘
              │
       @Deactivate called
              │
       ┌──────▼──────┐
       │  DISPOSED   │
       └─────────────┘
```

### Dependency Injection

```java
@Component
public class BalancingController {
    
    // Static reference (1:1, required)
    @Reference
    private Ess ess;
    
    // Dynamic reference (1:n, optional)
    @Reference(
        cardinality = MULTIPLE,
        policy = DYNAMIC,
        policyOption = GREEDY
    )
    private volatile List<SymmetricMeter> meters;
    
    // Method called when meter added
    @Reference(
        cardinality = MULTIPLE,
        policy = DYNAMIC
    )
    void addMeter(SymmetricMeter meter) {
        log.info("Meter added: {}", meter.id());
    }
    
    // Method called when meter removed  
    void removeMeter(SymmetricMeter meter) {
        log.info("Meter removed: {}", meter.id());
    }
}
```

---

## Summary: Data Flow Example

Let's trace a complete cycle with a **Self-Consumption Balancing** scenario:

### Initial State
- Grid: Exporting 2000 W (PV producing more than consumption)
- Battery: 50% SoC, idle
- Goal: Store excess in battery

### Cycle Execution

#### 1. INPUT Phase (0-200ms)
```
ModbusBridge triggers reads:
  ├─► GridMeter: Read ActivePower register
  │   Result: -2000 W (negative = export)
  │   Updates: meter0/ActivePower.nextValue = -2000
  │
  ├─► ESS: Read Soc register
  │   Result: 50%
  │   Updates: ess0/Soc.nextValue = 50
  │
  └─► ESS: Read ActivePower register
      Result: 0 W (idle)
      Updates: ess0/ActivePower.nextValue = 0

At 200ms: Switch Process Image
  meter0/ActivePower.value = -2000 W  (FROZEN)
  ess0/Soc.value = 50%                (FROZEN)
  ess0/ActivePower.value = 0 W        (FROZEN)
```

#### 2. PROCESS Phase (200-300ms)
```
Scheduler returns: [ctrlLimitDischarge0, ctrlBalancing0]

Execute ctrlLimitDischarge0:
  ├─► Read: ess0/Soc.value = 50% (safe, above 10% minimum)
  └─► Action: No constraints needed
      (if SoC was <10%, would force charge)

Execute ctrlBalancing0:
  ├─► Read: meter0/ActivePower.value = -2000 W (exporting)
  ├─► Read: ess0/Soc.value = 50% (can charge)
  ├─► Calculate: Need to charge at 2000 W to zero grid export
  └─► Write: ess0/SetActivePowerEquals.nextValue = -2000 W
      (negative = charge)
```

#### 3. OUTPUT Phase (300-500ms)
```
ModbusBridge triggers writes:
  └─► ESS: Write SetActivePowerEquals register
      Value: -2000 W
      ESS begins charging at 2000 W

Result: Grid export reduced to ~0 W, battery stores excess solar
```

#### Next Cycle (1000ms)
```
INPUT will read:
  ├─► meter0/ActivePower = ~0 W (balanced!)
  ├─► ess0/ActivePower = -2000 W (charging)
  └─► ess0/Soc = 50.1% (slightly increased)

PROCESS will maintain:
  └─► ctrlBalancing0 adjusts setpoint to keep grid at 0 W
```

---

## Key Takeaways

1. **IPO Model**: Fixed cycle (INPUT → PROCESS → OUTPUT) ensures deterministic execution

2. **Process Image**: Frozen data snapshot eliminates race conditions and timing bugs

3. **Channels**: Universal data abstraction for sensors, config, and setpoints

4. **Natures**: Device-independent interfaces enable reusable control algorithms

5. **Scheduler**: Prioritization ensures critical controls (safety, grid limits) take precedence

6. **Async Communication**: Bridges handle slow protocols without blocking the cycle

7. **Constraint Solving**: ESS Power component resolves conflicting controller demands

8. **OSGi Lifecycle**: Declarative Services provides robust dependency management

This architecture enables OpenEMS to reliably control complex energy systems with **sub-second response times** while maintaining **code simplicity** and **extensibility**.
