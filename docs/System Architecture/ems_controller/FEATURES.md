# Energy Management System (EMS) Controller - Features Documentation

## Table of Contents
1. [Authentication & Provisioning](#authentication--provisioning)
2. [Cloud Integration](#cloud-integration)
3. [Battery Management System (BMS)](#battery-management-system-bms)
4. [Inverter Control](#inverter-control)
5. [Power Control Modes](#power-control-modes)
6. [Cluster Management](#cluster-management)
7. [Grid Support Services](#grid-support-services)
8. [State Machine & Monitoring](#state-machine--monitoring)
9. [Communication Protocols](#communication-protocols)
10. [Climate Control](#climate-control)
11. [Safety & Error Handling](#safety--error-handling)
12. [User Interface](#user-interface)
13. [Firmware & Updates](#firmware--updates)
14. [Data Storage & Telemetry](#data-storage--telemetry)
15. [Hardware Integration](#hardware-integration)

---

## Authentication & Provisioning

### Device Bootstrapping
- **Location**: `bootstrapper/main.py`
- **Features**:
  - Automatic device provisioning and identity creation
  - Serial number extraction from Raspberry Pi CPU info
  - Internet connectivity verification
  - TeamViewer remote access configuration
  - Time synchronization with UTC servers
  - SSDP (Simple Service Discovery Protocol) for device discovery
  - State persistence across reboots
  - Automatic cleanup after successful provisioning

### Cloud Authentication
- **Location**: `sitecontroller/cloud/iothub_client_class.py`
- **Features**:
  - Azure IoT Hub connection via connection string
  - WebSocket-based secure communication
  - Automatic reconnection with exponential backoff
  - Certificate-based authentication
  - Device twin synchronization
  - Method request handling
  - Message queuing for offline operation

---

## Cloud Integration

### Azure IoT Hub Integration
- **Location**: `sitecontroller/cloud/cloudController.py`
- **Features**:
  - Device-to-cloud telemetry (10-second intervals)
  - Cloud-to-device command handling
  - Device twin updates and synchronization
  - Site parameter management from cloud
  - Local parameter caching for offline operation
  - Reported properties publishing
  - State change notifications
  - Cluster state reporting
  - Error message transmission

### Command Handler
- **Location**: `sitecontroller/cloud/commandHandler.py`
- **Supported Commands**:
  - `GoToState_Idling` - Transition to sleep state
  - `GoToState_Running` - Transition to run state
  - `Restart_Device` - Reboot the controller
  - `Update_Device` - Firmware update initiation
  - `Receive_Artifact` - Download and install artifacts
  - `Setup_Device` - Device configuration
  - `GoToMode_SelfConsumption` - Enable self-consumption mode
  - `GoToMode_LoadMatching` - Enable load matching mode
  - `GoToMode_ExternalControl` - Enable external control mode
  - `GoToMode_CostSaving` - Enable cost-saving mode
  - `GoToMode_ChargeOnly` - Enable charge-only mode
  - `GoToMode_GridForming` - Enable grid-forming/backup mode

### Blob Storage
- **Features**:
  - Log file uploads to Azure Blob Storage
  - Configuration file downloads
  - BMS configuration management
  - Firmware artifact storage

---

## Battery Management System (BMS)

### BMS Controller
- **Location**: `sitecontroller/deviceControllers/controllers/bmsController.py`
- **Features**:
  - Multi-rack battery monitoring (up to multiple racks)
  - Cell voltage monitoring (min/max/average)
  - Cell temperature monitoring with NTC compensation
  - State of Charge (SoC) tracking with delta compensation
  - State of Health (SoH) calculation
  - Pack current and voltage monitoring
  - Pack power calculation
  - Cell balancing control
  - Sleep mode management
  - SOC degradation detection and correction
  - Temperature-based voltage compensation
  - Fully charged/discharged detection
  - BMS error detection and reporting

### BMS Sleep Controller
- **Location**: `sitecontroller/deviceControllers/bmsSleepController.py`
- **Features**:
  - Modbus relay-based sleep control
  - Strato IO-based sleep control
  - Automatic sleep initiation during idle periods
  - Wake-up on demand
  - Sleep timer management

### CAN Bus Controller
- **Location**: `sitecontroller/IOSystem/controller/canBusController.py`
- **Features**:
  - CAN bus message harvesting
  - Signal decoding based on DBC files
  - Multi-rack communication
  - Error detection and recovery
  - Message filtering and parsing
  - Support for various BMS types (CBMS, NBMS)

---

## Inverter Control

### Inverter Factory
- **Location**: `sitecontroller/deviceControllers/inverterFactory.py`
- **Supported Inverters**:
  - ABB ESI
  - ABB PQstorl
  - LuxPowerTek
  - Trumpf AC3025

### Inverter Features
- **Active and reactive power control**
- **Power factor control**
- **Frequency and voltage monitoring**
- **Grid code compliance**
- **Ramp rate limiting**
- **Overcurrent protection**
- **Anti-islanding protection**
- **Grid-forming capability**
- **Three-phase monitoring (voltage, current)**
- **DC bus monitoring**
- **Temperature monitoring**
- **State management (standby, ready, running, fault)**

---

## Power Control Modes

### 1. Max Self-Consumption
- **Location**: `masterController/control/maxselfControl.py`
- **Description**: Maximizes self-consumption of solar energy by minimizing grid import/export
- **Features**:
  - PID-based grid power control
  - Zero-grid setpoint tracking
  - SOC reservation for backup
  - Automatic power distribution across cluster nodes
  - Sleep mode allowed for nodes

### 2. Peak Shaving
- **Location**: `masterController/control/peakShavingControl.py`
- **Description**: Limits grid import/export to stay within configured thresholds
- **Features**:
  - Configurable import/export limits
  - PID-based threshold control
  - Reduces demand charges
  - Grid overload prevention

### 3. External Control
- **Location**: `masterController/control/externalControl.py`
- **Description**: Allows external systems to control power flow via schedules
- **Features**:
  - Schedule-based power control
  - Active and reactive power setpoints
  - Integration with energy management systems
  - Modbus/TCP control interface

### 4. Charge Only
- **Location**: `masterController/control/chargeonlyControl.py`
- **Description**: Charges battery from grid with configurable power levels
- **Features**:
  - SOC target management
  - Charge rate limiting
  - Time-based charging schedules
  - Grid import control

### 5. Backup / Grid Forming
- **Location**: `masterController/control/backupControl.py`
- **Description**: Provides backup power during grid outages
- **Features**:
  - Automatic grid loss detection
  - Seamless transition to island mode
  - Load prioritization
  - Black start capability
  - Grid reconnection logic
  - Test mode for backup validation

### 6. Local Control
- **Location**: `masterController/control/localControl.py`
- **Description**: Local power setpoint control without cloud connectivity
- **Features**:
  - Heartbeat monitoring
  - Manual power setpoint entry
  - Local Modbus control
  - Fallback operation mode

### 7. Passive Mode
- **Location**: `masterController/control/passiveControl.py`
- **Description**: No active power control, monitoring only
- **Features**:
  - Telemetry collection
  - Status monitoring
  - Ready for activation

---

## Cluster Management

### Master Controller
- **Location**: `masterController/masterController.py`
- **Features**:
  - Multi-node coordination
  - Leader election via Raft consensus
  - Distributed power control
  - SOC averaging across cluster
  - Health monitoring of all nodes
  - Automatic failover
  - Power distribution algorithm
  - Cluster telemetry aggregation

### Raft Consensus
- **Location**: `masterController/utils/raft.py`
- **Features**:
  - Leader election
  - Heartbeat management
  - Term tracking
  - Vote request handling
  - Automatic step-down on leader failure
  - Configurable election timeout
  - Split-brain prevention

### Membership Management
- **Location**: `masterController/membership.py`
- **Features**:
  - Node discovery and registration
  - Node state tracking
  - Offline node detection
  - Data synchronization
  - Subscribe/publish mechanism
  - Master node tracking

### Node Controller
- **Location**: `masterController/nodeController.py`
- **Features**:
  - Node-level power control
  - State machine synchronization
  - Master/follower role management
  - Local command execution
  - Event handling

---

## Grid Support Services

### Grid Support Control
- **Location**: `masterController/control/gridSupport.py`
- **Features**:
  - Frequency Containment Reserve (FCR-D) services
  - Fast Frequency Response (FFR) services
  - Network Energy Management (NEM)
  - Automatic Emergency Management (AEM)
  - Fast power response
  - Droop-based frequency control
  - Grid frequency simulation for testing
  - Prequalification testing support
  - Telemetry logging for grid operators
  - SOC limit management during grid services

### Droop Control
- **Features**:
  - Configurable frequency thresholds (f1, f2, f3, f4)
  - Over-frequency and under-frequency response
  - Power proportional to frequency deviation
  - 5-minute moving average for AEM
  - Activation/deactivation logic

---

## State Machine & Monitoring

### State Machine
- **Location**: `sitecontroller/stateControllers/stateMachine.py`
- **States**:
  - **Init** - System initialization
  - **Run** - Normal operation
  - **Sleep/Idle** - Low power state
  - **Maintenance** - Service mode
  - **Error** - Fault condition
  - **Reboot** - System restart
  - **Restart** - Application restart

### Monitor
- **Location**: `sitecontroller/controllers/monitor.py`
- **Monitoring Features**:
  - Cell balance monitoring
  - BMS state tracking
  - Critical low voltage detection
  - SOC degradation detection
  - Raspberry Pi temperature monitoring
  - Humidity monitoring
  - Auto-charge detection
  - Voltage level monitoring
  - Inactive BMS timeout detection

### Event Handler
- **Location**: `sitecontroller/controllers/eventHandler.py`
- **Event Types**:
  - State change events
  - Mode change events
  - Power events
  - Meter controller events
  - Climate events
  - Setup device events
  - GFM (Grid Forming Mode) events

---

## Communication Protocols

### Modbus TCP/RTU
- **Server Implementations**:
  - **Modbus Control Server** - External system control interface
  - **Meter Gateway** - Energy meter data aggregation
  - **Base Server** - Generic Modbus functionality

- **Features**:
  - Real-time data exposure (SOC, power, limits, etc.)
  - Cluster state monitoring
  - Mode control via Modbus
  - Register mapping for standard SCADA systems
  - Multi-device support

### Modbus Client
- **Location**: `sitecontroller/modbusClient/modbusClientController.py`
- **Features**:
  - Energy meter data collection
  - External device communication
  - Carlo Gavazzi meter support
  - Register read/write operations

### ZeroMQ (ZMQ)
- **Location**: `masterController/transport.py`
- **Features**:
  - Publish/Subscribe messaging
  - Request/Reply pattern
  - Multicast communication for cluster
  - Event-based architecture
  - High-performance messaging

### Web PubSub
- **Location**: `masterController/utils/xoltaPubSub.py`
- **Features**:
  - Azure Web PubSub integration
  - Real-time event broadcasting
  - Cluster-wide event distribution

---

## Climate Control

### Climate Controllers
- **Location**: `sitecontroller/deviceControllers/climate/controllers.py`
- **Supported Devices**:
  - Fans (variable speed)
  - Heaters
  - Heater boards

### Climate Control Features
- **Temperature-based automatic control**
- **Manual force on/off**
- **Speed control (0-100%)**
- **Ramp rate limiting**
- **Hysteresis control**
- **Multi-zone support**
- **BAT5/10/80 specific algorithms**
- **Force control with timeout**
- **Thread-pool based execution**

---

## Safety & Error Handling

### Error Handler
- **Location**: `sitecontroller/controllers/errorHandler.py`
- **Error Severity Levels**:
  - **Warning** - Informational alerts
  - **Error** - Non-critical issues
  - **Critical** - Emergency stop conditions

### Error Management Features
- **Error ID generation**
- **Cloud error reporting**
- **GUI error display**
- **Active error tracking**
- **Conditional error handling**
- **Error persistence to disk**
- **Emergency stop triggering**
- **Source-based error categorization**

### Monitored Error Sources
- BMS errors
- Inverter errors
- CAN bus errors
- Communication errors
- Sensor errors
- Button/emergency stop
- Climate control errors
- Modbus errors
- Raspberry Pi errors
- Motherboard errors

### Safety Features
- **Emergency stop button monitoring**
- **Critical low voltage protection**
- **Overtemperature protection**
- **Overcurrent protection**
- **Grid loss detection**
- **Under-frequency protection**
- **Cell voltage imbalance detection**
- **Communication timeout protection**
- **Watchdog timer**

---

## User Interface

### GUI Features
- **Location**: `sitecontroller/gui/Gui.py`
- **Display Elements**:
  - System state and mode
  - Real-time power flow (kW)
  - Battery SOC and voltage
  - Cell temperatures (min/max)
  - Cell voltages (min/max)
  - Inverter status (3-phase V, I, P, Q)
  - Meter readings
  - Error messages
  - Cluster status
  - Charge/discharge limits
  - Grid frequency
  - Event log

### GUI Features
- **Auto-dimming after inactivity**
- **Touch screen support**
- **Multi-tab navigation**
- **Real-time updates (100ms)**
- **Color-coded status indicators**
- **Manual power setpoint control**
- **Test mode interface**

---

## Firmware & Updates

### OTA Update System
- **Location**: `sitecontroller/cloud/commandHandler.py`
- **Features**:
  - Cloud-initiated firmware updates
  - Artifact download from Azure Blob Storage
  - Version validation
  - Automatic update detection
  - Rollback capability
  - Update status reporting
  - Configuration file updates
  - BMS configuration updates

### Supported Artifact Types
- **Software/Firmware** - Main application updates
- **BMS Configuration** - DBC files
- **Device Configuration** - JSON configuration files
- **Meter Setup** - Carlo Gavazzi configurations

---

## Data Storage & Telemetry

### Data Store
- **Location**: `sitecontroller/utils/dataStore.py`
- **Stored Data**:
  - BMS data (voltage, current, temperature, SOC)
  - Inverter data (power, voltage, current, frequency)
  - Meter data (grid, PV, load)
  - Sensor data (temperature, humidity)
  - Climate data (fan/heater status)
  - Button states
  - Limits (charge/discharge)

### Telemetry Features
- **10-second telemetry intervals**
- **Batch message transmission**
- **Offline queue management (up to 1 week)**
- **Automatic queue persistence**
- **Compression and optimization**
- **State change events**
- **Error message transmission**
- **Cluster telemetry aggregation**

### Telemetry Objects
- `SC_Spot_10s` - Regular telemetry
- `SC_state_event` - State changes
- `SC_cluster_state_event` - Cluster state changes
- Error messages
- DSO setpoint changes
- AEM alarm events

---

## Hardware Integration

### Supported Hardware
- **Raspberry Pi** (GPIO, I2C, CAN)
- **CAN Bus interfaces**
- **Modbus RTU/TCP devices**
- **Digital I/O modules**
- **Strato IO modules**
- **Relay modules**
- **Temperature sensors (NTC, I2C)**
- **Humidity sensors**
- **Push buttons**
- **LED indicators**

### I/O System
- **Location**: `sitecontroller/IOSystem/`
- **Features**:
  - GPIO control
  - I2C communication
  - CAN bus messaging
  - Modbus RTU
  - Device abstraction layer
  - Signal mapping from configuration

### Sensor Factory
- **Location**: `sitecontroller/deviceControllers/sensorFactory.py`
- **Supported Sensors**:
  - BME680 (temperature, humidity, pressure, gas)
  - Raspberry Pi CPU temperature
  - Custom I2C sensors
  - Analog sensors via ADC

### Watchdog
- **Location**: `sitecontroller/deviceControllers/watchdogFactory.py`
- **Features**:
  - System health monitoring
  - Automatic reboot on hang
  - Configurable timeout
  - Heartbeat management

---

## Advanced Features

### PID Controller
- **Location**: `masterController/utils/pid.py`
- **Features**:
  - Proportional-Integral-Derivative control
  - Anti-windup protection
  - Saturation handling
  - Configurable gains (Kp, Ki, Kd)
  - Output limiting
  - Reset functionality

### Charge Limiter
- **Location**: `sitecontroller/stateControllers/chargeLimiter.py`
- **Features**:
  - Hardware-based limits (BMS)
  - Software-based limits (grid codes)
  - SOC-based limiting
  - Temperature-based limiting
  - Voltage-based limiting
  - Current-based limiting
  - Dynamic ramp rate adjustment

### Charge Schedule
- **Features**:
  - Time-based power schedules
  - Active and reactive power control
  - Multiple schedule entries
  - Cloud-based schedule updates
  - Local schedule fallback

### System Information
- **Location**: `sitecontroller/utils/sysInfo.py`
- **Tracked Information**:
  - Firmware version
  - Last boot time
  - Total runtime
  - Raft term
  - Last mode change
  - SOC delta tracking
  - SOH tracking
  - Last fully charged timestamp
  - BMS sleep time
  - Cell balance history

---

## Configuration Management

### Site Parameters
- **Location**: `sitecontroller/utils/siteParameters.py`
- **Configurable Parameters**:
  - Device identity (whoAmI, siteId)
  - BMS configuration
  - Inverter configuration
  - Meter configuration
  - Cluster configuration
  - Backup settings
  - Grid code settings
  - PID gains
  - Limits (SOC, power, voltage, temperature)
  - Feature flags
  - Telemetry batch size
  - Commissioning state

### Feature Flags
- **CLUSTER** - Enable cluster mode
- **BACKUP** - Enable backup/grid-forming mode
- **CELL_BALANCE** - Enable automatic cell balancing
- Additional hardware-specific flags

---

## Logging

### Log Levels
- DEBUG
- INFO
- WARNING
- ERROR
- CRITICAL

### Log Destinations
- **System log** (`/home/pi/Desktop/BESS_logs/syslog.log`)
- **Bootstrapper log** (`/home/pi/Desktop/BESS_logs/bootstrapper.log`)
- **LUX update log** (inverter-specific)
- **Azure IoT device log**
- **Grid support telemetry log** (CSV)

### Log Rotation
- Automatic log file rotation
- Cloud upload of log files
- Local storage management

---

## Testing Features

### Test Controller
- **Location**: `sitecontroller/controllers/testController.py`
- **Features**:
  - Manual power setpoint control
  - State transition testing
  - Mode change testing
  - Keyboard input for manual control
  - Test mode activation

### Grid Frequency Simulator
- **Location**: `masterController/utils/gridFrequencySimulator.py`
- **Features**:
  - FCR-D frequency simulation
  - FFR frequency simulation
  - SOC limit calculation during tests
  - Prequalification test support

---

## End Goal

The EMS Controller provides a **complete, production-ready energy management solution** for battery energy storage systems (BESS) with the following objectives:

### Primary Goals
1. **Maximize renewable energy utilization** through intelligent power management
2. **Provide grid stability services** (FCR-D, FFR, peak shaving)
3. **Ensure reliable backup power** during grid outages
4. **Optimize battery lifetime** through intelligent charge/discharge management
5. **Enable revenue generation** through grid services participation
6. **Provide real-time monitoring and control** via cloud and local interfaces

### Deployment Scenarios
- **Residential** - Home energy storage systems (5-10 kWh)
- **Commercial** - Small to medium business installations (30-80 kWh)
- **Industrial** - Large-scale grid-connected storage (multi-MWh clusters)
- **Off-grid** - Remote installations with backup capability

### Integration Capabilities
- Azure IoT Hub for cloud management
- SCADA systems via Modbus TCP/RTU
- Energy management systems via external control
- Grid operators via grid support protocols
- Local control via GUI and physical buttons

---

## System Requirements

### Hardware
- Raspberry Pi 3B+ or 4
- CAN bus interface (MCP2515 or similar)
- Supported inverter (ABB, LuxPowerTek, Trumpf)
- Supported BMS (CBMS, NBMS 2.3-2.5.x)
- Network connectivity (Ethernet/WiFi)
- Optional: Touch screen display

### Software Dependencies
- Python 3.7+
- Azure IoT Device SDK
- PyModbus
- Python-CAN
- ZeroMQ
- NumPy
- Tkinter (for GUI)
- See `requirement.txt` for complete list

---

## Version Information
- **System**: Energy Management System Controller
- **Target Platform**: Raspberry Pi with Linux
- **Python Version**: 3.7+
- **License**: Proprietary (Xolta Energy)

---

*This documentation covers the complete feature set from authentication and provisioning through all operational modes to the end goals of the EMS Controller system.*
