# OpenEMS Communication and Bridge Modules Documentation

## Overview

OpenEMS implements a sophisticated bridge architecture that provides abstraction layers for different communication protocols and interfaces. These bridges enable seamless integration with various hardware devices including inverters, meters, batteries, and IoT sensors.

---

## 1. Modbus Bridge (`io.openems.edge.bridge.modbus`)

### Description
Modbus is the most widely adopted fieldbus communication standard in industrial automation and energy management systems. The OpenEMS Modbus bridge provides comprehensive support for both TCP/IP and serial (RTU) variants.

### Supported Protocols

#### 1.1 Modbus/TCP (`BridgeModbusTcpImpl`)
- **Transport**: TCP/IP network communication
- **Connection**: Ethernet-based field devices
- **Configuration**: IP address and port (default: 502)
- **Use Cases**: Network-connected inverters, meters, PLCs

#### 1.2 Modbus/RTU Serial (`BridgeModbusSerialImpl`)
- **Transport**: RS485 serial bus
- **Connection**: Serial port (e.g., `/dev/ttyUSB0` or `COM3`)
- **Configuration**: Port name, baudrate, data bits, stop bits, parity
- **Use Cases**: Legacy devices, daisy-chained field equipment

### Key Features

#### Task-Based Protocol Architecture
- **ModbusProtocol**: Defines communication patterns for each component
- **Task System**: Individual read/write operations covering multiple registers/coils
- **Dynamic Task Management**: Add/remove tasks at runtime
- **Priority Scheduling**: Tasks can have HIGH or LOW priority

#### Supported Modbus Function Codes
| Function Code | Task Class | Description |
|---------------|------------|-------------|
| FC1 | `FC1ReadCoilsTask` | Read Coils (0x01) |
| FC2 | `FC2ReadInputsTask` | Read Discrete Inputs (0x02) |
| FC3 | `FC3ReadRegistersTask` | Read Holding Registers (0x03) |
| FC4 | `FC4ReadInputRegistersTask` | Read Input Registers (0x04) |
| FC5 | `FC5WriteCoilTask` | Write Single Coil (0x05) |
| FC6 | `FC6WriteRegisterTask` | Write Single Register (0x06) |
| FC16 | `FC16WriteRegistersTask` | Write Multiple Registers (0x10) |

#### Execution Strategy
The `ModbusWorker` implements intelligent task scheduling:
- **Write-First Strategy**: Write tasks execute immediately after EXECUTE_WRITE event
- **Read-Last Strategy**: Read tasks execute just before BEFORE_PROCESS_IMAGE event
- **Cycle Delay Learning**: Automatically calculates optimal delay to maximize bus efficiency
- **Defective Component Handling**: Delays communication with failed devices (max 5 min)
- **Priority-Based Execution**:
  - HIGH priority: Executed every cycle
  - LOW priority: One LOW task executed per cycle across all components

#### SunSpec Protocol Support
The bridge includes comprehensive SunSpec model definitions (`DefaultSunSpecModel`):
- **Model Types**: Common, Inverter, Meter, Battery, Aggregator, Network Configuration
- **Supported Models**: S_1 (Common), S_101-113 (Inverters), S_201-213 (Meters), S_120-145 (Controls)
- **Point Types**: Value points, enum points, bit-field points, scale factors
- **Auto-Discovery**: Dynamic model detection and mapping

### Communication Patterns

```java
// Component implements ModbusComponent interface
public class MyDevice extends AbstractOpenemsModbusComponent {
    
    private ModbusProtocol modbusProtocol;
    
    @Override
    protected ModbusProtocol defineModbusProtocol() {
        return new ModbusProtocol(this,
            // High priority: read every cycle
            new FC3ReadRegistersTask(0, Priority.HIGH, 
                m(ChannelId.STATE, new UnsignedWordElement(0)),
                m(ChannelId.POWER, new SignedWordElement(1))),
            
            // Low priority: read occasionally
            new FC3ReadRegistersTask(100, Priority.LOW,
                m(ChannelId.SERIAL_NUMBER, new StringWordElement(100, 16)))
        );
    }
}
```

### Monitoring Channels
- `CYCLE_TIME_IS_TOO_SHORT`: Indicates configured cycle time is insufficient
- `CYCLE_DELAY`: Learned optimal delay time in milliseconds
- `BRIDGE_IS_STOPPED`: Communication stopped state

### Debug & Logging
Configurable verbosity levels via `LogVerbosity` parameter:
- `NONE`: No logs
- `DEBUG_LOG`: Basic logging via Controller.Debug.Log
- `READS_AND_WRITES`: All read/write requests
- `READS_AND_WRITES_VERBOSE`: Include hex/binary values
- `READS_AND_WRITES_DURATION`: Include timing information
- `READS_AND_WRITES_DURATION_TRACE_EVENTS`: Full state machine tracing

### Dependencies
- **j2mod**: Pure Java Modbus library (ghgande fork)
- **io.openems.common**: Core utilities
- **io.openems.edge.common**: Edge component framework

---

## 2. MQTT Bridge (`io.openems.edge.bridge.mqtt`)

### Description
MQTT (Message Queuing Telemetry Transport) bridge provides lightweight publish/subscribe messaging with support for multiple protocol versions. Ideal for IoT integration and cloud connectivity.

### Supported Protocols

#### MQTT Protocol Versions
| Version | Enum | Description |
|---------|------|-------------|
| MQTT 3.1 | `V3_1` | Legacy version |
| MQTT 3.1.1 | `V3_1_1` | Most widely supported |
| MQTT 5.0 | `V5` | Latest with enhanced features |

### Key Features

#### Connection Management
- **Dual Implementation**:
  - `Mqtt3ConnectionHandler`: For MQTT 3.1 and 3.1.1
  - `Mqtt5ConnectionHandler`: For MQTT 5.0
- **Client ID**: Auto-generated or user-specified
- **Auto-Reconnect**: Automatic connection recovery
- **Subscription Persistence**: Re-subscribes on reconnect
- **SSL/TLS Support**: Secure connections

#### Quality of Service (QoS)
| QoS Level | Value | Description | Use Case |
|-----------|-------|-------------|----------|
| `AT_MOST_ONCE` | 0 | Fire and forget, may lose messages | Sensor data where occasional loss is acceptable |
| `AT_LEAST_ONCE` | 1 | Guarantees delivery, may duplicate | Most common, reliable messaging |
| `EXACTLY_ONCE` | 2 | Guaranteed single delivery | Critical commands, highest overhead |

#### Message Structure
The `MqttMessage` record encapsulates:
- **Topic**: Hierarchical topic string (e.g., `openems/meter/power`)
- **Payload**: Byte array (UTF-8 string methods provided)
- **QoS**: Quality of service level
- **Retained**: Whether broker retains message for new subscribers

### Communication Patterns

#### Publishing Messages
```java
@Reference
private BridgeMqtt mqttBridge;

// Simple publish with defaults (QoS 1, not retained)
mqttBridge.publish("openems/meter/power", "1234");

// Full control
mqttBridge.publish("openems/status", "{\"state\":\"active\"}", 
                   QoS.EXACTLY_ONCE, true);

// Binary payload
byte[] data = ...;
mqttBridge.publish("openems/binary/data", data, QoS.AT_LEAST_ONCE, false);
```

#### Subscribing to Topics
```java
// Subscribe with callback
mqttBridge.subscribe("openems/commands/#", QoS.AT_LEAST_ONCE, message -> {
    String topic = message.topic();
    String payload = message.payloadAsString();
    
    // Process message
    System.out.println("Received on " + topic + ": " + payload);
});

// Topic wildcards
// + matches single level: openems/+/status
// # matches multiple levels: openems/commands/#
```

### Monitoring Channels
- `CONNECTED`: Boolean indicating broker connection status
- `CONNECTION_FAILED`: Fault state when connection fails
- `BROKER_UNREACHABLE`: Warning when broker is unreachable

### Component Integration
Components using MQTT implement `MqttComponent` interface:
- Provides `MQTT_COMMUNICATION_FAILED` channel
- Offers `retryMqttCommunication()` method for error recovery

### Configuration
- **Host**: Broker hostname/IP
- **Port**: Broker port (default: 1883, SSL: 8883)
- **Client ID**: Unique client identifier
- **MQTT Version**: V3_1, V3_1_1, or V5
- **Username/Password**: Authentication credentials
- **SSL**: Enable secure connection
- **Debug Mode**: Enable detailed logging

### Dependencies
- **Eclipse Paho**: MQTT client libraries
  - `org.eclipse.paho.client.mqttv3`: For MQTT 3.x
  - `org.eclipse.paho.mqttv5.client`: For MQTT 5.0
- **Bouncy Castle**: Cryptography for SSL (bcpkix, bcprov)

---

## 3. HTTP Bridge (`io.openems.common.bridge.http` & `io.openems.edge.bridge.http`)

### Description
HTTP bridge provides RESTful API communication capabilities with advanced features like connection pooling, retry logic, OAuth authentication, and cycle-based scheduling. Split into common (shared) and edge-specific implementations.

### Architecture

#### Common Bridge (`io.openems.common.bridge.http`)
Core HTTP communication framework used by both Edge and Backend:
- **BridgeHttp**: Main interface for HTTP operations
- **BridgeHttpFactory**: Factory for creating bridge instances
- **BridgeHttpExecutor**: Thread pool management
- **EndpointFetcher**: Actual HTTP request execution

#### Edge Cycle Bridge (`io.openems.edge.bridge.http`)
Extends common bridge with cycle-aware scheduling:
- **HttpBridgeCycleService**: Cycle-synchronized HTTP requests
- **CycleSubscriber**: Event-based cycle coordination

### Key Features

#### HTTP Methods Support
- **GET**: Retrieve data from endpoints
- **POST**: Submit data to endpoints
- **PUT**: Update resources
- **DELETE**: Remove resources
- **PATCH**: Partial updates

#### Endpoint Builder Pattern
```java
Endpoint endpoint = BridgeHttp.create("http://example.com/api/data")
    .setMethod(HttpMethod.POST)
    .setHeader("Authorization", "Bearer token123")
    .setHeader("Content-Type", "application/json")
    .setBodyJson(jsonElement)
    .build();
```

#### Response Handling
```java
// Simple GET request
CompletableFuture<HttpResponse<String>> future = 
    httpBridge.get("http://example.com/status");

// JSON response
CompletableFuture<HttpResponse<JsonElement>> jsonFuture = 
    httpBridge.getJson("http://example.com/data.json");

// Handle response
future.thenAccept(response -> {
    int statusCode = response.status();
    String body = response.data();
    // Process response
}).exceptionally(error -> {
    // Handle error
    return null;
});
```

#### Cycle-Based Scheduling
The Edge bridge extends HTTP with cycle-aware features:

```java
// Subscribe to endpoint every cycle
httpBridgeCycle.subscribeEveryCycle("http://device/status", 
    response -> {
        // Process data every cycle
    },
    error -> {
        // Handle errors
    });

// Subscribe every N cycles
httpBridgeCycle.subscribeCycle(5, "http://device/config",
    response -> {
        // Executes every 5th cycle
    });
```

**Cycle Scheduling Features**:
- **Non-Blocking**: If request takes longer than cycle, waits for completion
- **No Duplicate Execution**: Won't start new request while previous is pending
- **Automatic Retry**: Can be configured for automatic retry on failure
- **Dynamic Endpoints**: Endpoint supplier allows runtime URL changes

### Advanced Features

#### OAuth 2.0 Support
- **HttpBridgeOAuthService**: Manages OAuth token lifecycle
- **Automatic Token Refresh**: Handles token expiration
- **Multiple Grant Types**: Support for various OAuth flows

#### Connection Management
- **Connection Pooling**: Reuses connections for efficiency
- **Configurable Timeouts**:
  - Connect timeout: Default 5000ms
  - Read timeout: Default 5000ms
- **Maximum Pool Size**: Adjustable thread pool
- **Metrics**: Track request counts, durations, errors

#### Event System
```java
// Subscribe to HTTP events
httpBridge.subscribeEvent(eventDefinition, data -> {
    // Handle event
});
```

#### Time Provider Service
- **DelayTimeProvider**: Controls request timing and delays
- **DefaultDelayTimeProvider**: Standard implementation
- Useful for rate limiting and scheduled requests

### Communication Patterns

#### Simple Request-Response
```java
// One-time request
var response = httpBridge.get("http://meter/reading").get();
```

#### Continuous Polling
```java
// Poll device every 10 cycles
CycleEndpoint endpoint = new CycleEndpoint(
    10, // every 10 cycles
    () -> Endpoint.create("http://device/data").build(),
    response -> updateChannels(response),
    error -> logError(error)
);
httpBridgeCycle.subscribeCycle(endpoint);
```

#### Form-Encoded POST
```java
Map<String, String> formData = Map.of(
    "username", "admin",
    "password", "secret"
);

Endpoint endpoint = BridgeHttp.create("http://device/login")
    .setBodyFormEncoded(formData)
    .build();
```

### Debug Features
- **Debug Mode**: Enum with OFF, STANDARD, DETAILED
- **Request/Response Logging**: Configurable verbosity
- **Performance Metrics**: Track request durations

### Dependencies
- **Java HTTP Client**: Native Java 11+ HTTP client
- **GSON**: JSON serialization/deserialization
- **io.openems.common**: Core utilities

---

## 4. M-Bus Bridge (`io.openems.edge.bridge.mbus`)

### Description
M-Bus (Meter-Bus) is a European standard (EN 13757) for remote reading of utility meters and sensors. Widely used in building automation and district heating/cooling systems.

### Supported Protocols

#### M-Bus Serial
- **Transport**: RS232/RS485 serial connection
- **Standard**: EN 13757-2 (Physical and Link Layer)
- **Typical Use**: Direct connection to meter network

### Key Features

#### Connection Configuration
- **Port Name**: Serial port identifier (e.g., `/dev/ttyUSB0`)
- **Baudrate**: Configurable (common: 2400, 9600, 38400)
- **Protocol**: M-Bus frame format with checksums

#### Task-Based Communication
Similar to Modbus, uses task abstraction:
- **MbusTask**: Defines a read operation from a slave device
- **Task Manager**: Coordinates execution across multiple devices
- **ChannelDataRecordMapper**: Maps M-Bus data records to OpenEMS channels

#### Data Record Handling
- **ChannelRecord**: Represents a single M-Bus data record
- **Type Support**: Various data types (integer, float, date, string)
- **Unit Support**: Automatic unit conversion (Wh, m³, °C, etc.)

### Communication Patterns

```java
// Component implements M-Bus communication
public class MbusDevice extends AbstractOpenemsMbusComponent {
    
    @Override
    protected void defineConfiguration() {
        // Create task for primary address 1
        MbusTask task = new MbusTask(
            1, // Primary address
            this.createChannelMappers()
        );
        
        // Register with bridge
        bridge.addTask(this.id(), task);
    }
    
    private ChannelDataRecordMapper[] createChannelMappers() {
        return new ChannelDataRecordMapper[] {
            new ChannelDataRecordMapper(
                ChannelId.ENERGY,
                dataRecord -> dataRecord.getScaledDataValue()
            )
        };
    }
}
```

### Monitoring Channels
- `SLAVE_COMMUNICATION_FAILED`: Communication failure with slave device
- `CYCLE_TIME_IS_TOO_SHORT`: Cycle time insufficient for all tasks
- `EXECUTION_DURATION`: Time taken to execute all tasks

### Native Library Requirements

#### jMBus Library
- **Provider**: OpenMUC (www.openmuc.org)
- **Version**: 3.3.0
- **License**: LGPL

#### RxTx Serial Library
- **Purpose**: Native serial port access via JNI
- **Linux Setup**: 
  - Install RxTx: `apt-get install librxtx-java`
  - May need to move `.so` files to `/usr/lib`
- **Windows Setup**: 
  - Include `rxtxSerial.dll` in library path

### Configuration Considerations
- **Baud Rate**: Must match M-Bus network configuration
- **Primary Addressing**: Each device has unique address (0-250)
- **Secondary Addressing**: Alternative addressing via manufacturer ID
- **Cycle Time**: Must account for multiple devices on bus

### Dependencies
- **org.openmuc.jmbus**: M-Bus protocol implementation
- **org.openmuc.jrxtx**: Serial port communication

---

## 5. OneWire Bridge (`io.openems.edge.bridge.onewire`)

### Description
OneWire (1-Wire) is a device communications bus system designed by Dallas Semiconductor/Maxim Integrated. Provides low-speed data, signaling, and power over a single conductor.

### Supported Protocols

#### OneWire Protocol
- **Transport**: Single-wire bus (plus ground)
- **Devices**: Temperature sensors, humidity sensors, switches, EEPROMs
- **Topology**: Multi-drop network, up to 100+ devices per bus

### Key Features

#### Hardware Support
- **Bus Masters**: Direct support for:
  - DS9490R USB adapter (most common)
  - DS1481 (RS232 adapter)
  - Other USB/serial OneWire adapters

#### Native Access
- **Direct Communication**: Talks directly to busmaster without OWFS
- **JNI Integration**: Uses Maxim Integrated's native C library via Java JNI
- **Exclusive Access**: Requires exclusive busmaster access (blocks Linux kernel driver)

#### Supported Container Types
Extensive device family support:
- **Temperature**: DS18B20, DS18S20, DS1820, DS1822
- **Battery Monitor**: DS2438 (voltage, current, temperature)
- **Memory**: DS2502, DS2506, DS28E04 (EEPROM)
- **Switches**: DS2405, DS2406, DS2408
- **ID Only**: DS2401, DS1990A

### Device Discovery

The bridge provides JSON-RPC API for device enumeration:

#### Request Format
```json
{
  "method": "componentJsonApi",
  "params": {
    "componentId": "onewire0",
    "payload": {
      "method": "getDevices",
      "params": {}
    }
  }
}
```

#### Response Example
```json
{
  "devices": [
    {
      "address": "4D0000094xxxxxxx",
      "name": "DS18B20",
      "alternateName": "DS1820B, DS18B20X",
      "description": "Digital thermometer measures temperatures...",
      "details": {
        "type": "TemperatureContainer",
        "temperature": 17.5625
      }
    },
    {
      "address": "6F0000022xxxxxxx",
      "name": "DS2438",
      "alternateName": "Smart Battery Monitor",
      "description": "1-Wire device that integrates...",
      "details": {
        "type": "TemperatureContainer",
        "temperature": 19.3125
      }
    }
  ]
}
```

### Setup & Installation

#### Linux Setup
```bash
# 1. Download OneWireViewer-Linux.zip from Maxim Integrated
unzip OneWireViewer-Linux.zip

# 2. Install dependencies
sudo apt-get install libusb-dev build-essential openjdk-8-jdk

# 3. Compile native library
cd OneWireViewer-Linux/PDKAdapterUSB
make
sudo make install

# 4. Disable kernel driver (conflicts with direct access)
sudo rmmod ds2490
echo "blacklist ds2490" | sudo tee /etc/modprobe.d/ds2490.conf
```

### Communication Patterns

```java
// Access OneWire device
public class OneWireTempSensor extends AbstractOneWireDevice {
    
    @Override
    protected void readValues() {
        try {
            // Get device by address
            OneWireContainer device = adapter.getDeviceContainer(deviceAddress);
            
            if (device instanceof TemperatureContainer) {
                TemperatureContainer tempDevice = (TemperatureContainer) device;
                
                // Read temperature
                byte[] state = tempDevice.readDevice();
                tempDevice.doTemperatureConvert(state);
                
                double temperature = tempDevice.getTemperature(state);
                
                // Update channel
                this.channel(ChannelId.TEMPERATURE).setNextValue(temperature);
            }
        } catch (OneWireException e) {
            logError("OneWire read failed", e);
        }
    }
}
```

### Device Addressing
- **64-bit ROM Address**: Unique identifier for each device
- **Format**: 8-byte hex string (e.g., `4D0000094ABCDEF0`)
- **Family Code**: First byte identifies device type
- **Serial Number**: 6 bytes unique serial
- **CRC**: Last byte for error detection

### Container Types in Bridge

| Container Interface | Device Types | Capabilities |
|---------------------|--------------|--------------|
| `TemperatureContainer` | DS18B20, DS1822, DS2438 | Temperature measurement |
| `ADContainer` | DS2450, DS2438 | Analog-to-digital conversion |
| `ClockContainer` | DS1904, DS2404 | Real-time clock |
| `SwitchContainer` | DS2405, DS2408 | Digital I/O control |
| `MemoryBankContainer` | DS2502, DS28E04 | EEPROM storage |

### Key Implementation Classes

#### Core OneWire Stack
- **OneWireAccessProvider**: Main access point for OneWire network
- **OneWireContainer**: Base class for all device types
- **OneWireException**: Exception handling
- **CRC8/CRC16**: Checksum utilities

#### Utilities
- **IOHelper**: I/O utility functions
- **OWPath**: Network path management
- **SHA**: Cryptographic operations for secure devices

### Configuration
- **Adapter Type**: USB, Serial, etc.
- **Port Name**: Adapter device path
- **Search Options**: Normal, alarming, conditional
- **Power Delivery**: Standard, overdrive modes

### Performance Considerations
- **Bus Speed**: Standard (15.4 kbps) or Overdrive (125 kbps)
- **Search Time**: Can be slow with many devices (seconds)
- **Temperature Conversion**: DS18B20 takes 750ms at 12-bit resolution
- **Parasitic Power**: Some devices draw power from data line

### Dependencies
- **RXTXcomm.jar**: Native serial communication
- **Native Library**: Maxim OneWire C library (compiled for platform)

---

## Communication Architecture Comparison

| Feature | Modbus | MQTT | HTTP | M-Bus | OneWire |
|---------|--------|------|------|-------|---------|
| **Type** | Request/Response | Pub/Sub | Request/Response | Request/Response | Multi-drop |
| **Speed** | Medium (9.6k-115.2k bps) | Medium-High | High (network) | Low (2.4k-38.4k bps) | Low (15.4k-125k bps) |
| **Topology** | Master/Slave | Many-to-Many | Client/Server | Master/Slave | Multi-drop |
| **Range** | Serial: 1200m, TCP: Unlimited | Unlimited (network) | Unlimited (network) | Serial: 1000m | 100m per segment |
| **Devices** | 247 (serial), Unlimited (TCP) | Unlimited | Unlimited | 250 | 100+ per bus |
| **Power** | External | External | External | External | Can be bus-powered |
| **Typical Use** | Industrial automation | IoT, Cloud | Web APIs, Modern devices | Utility metering | Temperature sensing |

---

## Common Communication Patterns

### 1. Polling Pattern
**Used by**: Modbus, M-Bus, HTTP Cycle Bridge

```
System Cycle → Trigger Read → Execute Task → Parse Response → Update Channels
     ↓                                                             ↑
     └─────────────────── Next Cycle ──────────────────────────────┘
```

### 2. Subscribe Pattern
**Used by**: MQTT

```
Component → Subscribe(topic) → Broker → Messages → Component Callback
                                 ↑
                                 └── Publishers
```

### 3. Event-Driven Pattern
**Used by**: All bridges for integration with OpenEMS event system

```
OpenEMS Event System
    ↓
TOPIC_CYCLE_EXECUTE_WRITE → Write Tasks Execute
    ↓
TOPIC_CYCLE_BEFORE_PROCESS_IMAGE → Read Tasks Execute
    ↓
TOPIC_CYCLE_AFTER_PROCESS_IMAGE → Values Available
```

### 4. Task Priority Pattern
**Used by**: Modbus, M-Bus

- **HIGH Priority**: Critical data, read every cycle
- **LOW Priority**: Non-critical data, read periodically
- **Write Priority**: Always highest, execute immediately

---

## Integration Guidelines

### Choosing the Right Bridge

#### Use **Modbus** when:
- Connecting to industrial devices (inverters, meters, PLCs)
- Standard automation equipment
- Need for reliable, deterministic communication
- Direct register access required

#### Use **MQTT** when:
- IoT device integration
- Cloud connectivity needed
- Event-driven architecture preferred
- Multiple consumers need same data
- Devices are geographically distributed

#### Use **HTTP** when:
- Modern RESTful APIs
- Web-based device interfaces
- Cloud service integration
- OAuth authentication required
- JSON data exchange

#### Use **M-Bus** when:
- Connecting to utility meters
- Building automation systems
- District heating/cooling networks
- European meter standards compliance

#### Use **OneWire** when:
- Temperature monitoring applications
- Simple sensor networks
- Low-cost temperature sensing
- Parasitic power operation needed
- Small form factor devices

### Performance Optimization

#### Modbus Optimization
- Use LOW priority for infrequently changing values
- Group sequential registers in single task
- Monitor `CYCLE_DELAY` channel for tuning
- Handle defective devices gracefully

#### MQTT Optimization
- Use appropriate QoS level (0 for high-frequency data)
- Implement topic hierarchies for filtering
- Use retained messages for current state
- Wildcard subscriptions for multiple topics

#### HTTP Optimization
- Use cycle-based scheduling to avoid overlapping requests
- Implement connection pooling
- Cache responses when appropriate
- Handle timeouts gracefully

#### M-Bus Optimization
- Minimize devices per cycle
- Use appropriate baud rate
- Consider secondary addressing for large networks

#### OneWire Optimization
- Limit devices per bus (20-30 for best performance)
- Use overdrive speed when supported
- Cache device addresses to avoid repeated searches
- Consider branch couplers for large networks

---

## Error Handling & Reliability

### Modbus Error Handling
- **Defective Component Logic**: Backs off failed devices (up to 5 min)
- **Retry Method**: Components can call `retryModbusCommunication()`
- **Channels**: `COMMUNICATION_FAILED` state channel
- **Exception Hierarchy**: ModbusException with specific error types

### MQTT Error Handling
- **Auto-Reconnect**: Automatic broker reconnection
- **Subscription Persistence**: Re-subscribes after reconnect
- **QoS Guarantees**: Message delivery guarantees based on QoS level
- **Channels**: `CONNECTION_FAILED`, `BROKER_UNREACHABLE`

### HTTP Error Handling
- **Timeout Configuration**: Separate connect and read timeouts
- **Retry Logic**: Can be implemented in cycle callbacks
- **Error Callbacks**: Separate error handlers
- **HTTP Status Codes**: Proper status code handling

### M-Bus Error Handling
- **Communication Failed Channel**: Indicates slave communication errors
- **Cycle Time Monitoring**: Detects insufficient cycle time
- **Timeout Handling**: Serial communication timeout management

### OneWire Error Handling
- **Device Search**: Automatic device discovery on errors
- **CRC Validation**: All communications CRC-checked
- **Exception Handling**: OneWireException for device errors
- **Retry Logic**: Automatic retry on CRC failures

---

## Configuration Examples

### Modbus TCP Configuration
```json
{
  "id": "modbus0",
  "alias": "PV Inverter",
  "enabled": true,
  "ip": "192.168.1.100",
  "port": 502,
  "logVerbosity": "NONE",
  "invalidateElementsAfterReadErrors": 1
}
```

### MQTT Configuration
```json
{
  "id": "mqtt0",
  "alias": "Cloud Bridge",
  "enabled": true,
  "host": "mqtt.example.com",
  "port": 8883,
  "mqttVersion": "V3_1_1",
  "clientId": "openems-edge-001",
  "username": "device",
  "password": "secret",
  "secureConnect": true,
  "debugMode": false
}
```

### HTTP Configuration (via code)
```java
BridgeHttp httpBridge = httpBridgeFactory.get();
httpBridge.setDebugMode(DebugMode.DETAILED);
httpBridge.setMaximumPoolSize(10);
```

### M-Bus Configuration
```json
{
  "id": "mbus0",
  "alias": "Heat Meter Network",
  "enabled": true,
  "portName": "/dev/ttyUSB0",
  "baudrate": 2400
}
```

### OneWire Configuration
```json
{
  "id": "onewire0",
  "alias": "Temperature Sensors",
  "enabled": true,
  "adapterName": "{DS9490}",
  "searchMode": "NORMAL"
}
```

---

## Testing & Development

### Mock Implementations
Each bridge provides test/dummy implementations:

- **DummyModbusBridge**: Mock Modbus for unit testing
- **DummyBridgeMqtt**: Mock MQTT broker connection
- **DummyBridgeHttp**: Mock HTTP responses
- Testing helpers in respective test packages

### Debug Modes
All bridges support enhanced debugging:
- Detailed request/response logging
- Timing information
- State machine tracing (Modbus)
- Event logging (MQTT)

---

## Dependencies Summary

### Modbus Bridge
```
io.openems.j2mod (Pure Java Modbus)
io.openems.common
io.openems.edge.common
```

### MQTT Bridge
```
org.eclipse.paho.client.mqttv3
org.eclipse.paho.mqttv5.client
bcpkix, bcprov (Bouncy Castle SSL)
io.openems.common
io.openems.edge.common
```

### HTTP Bridge
```
Java 11+ HTTP Client (native)
com.google.gson
io.openems.common
```

### M-Bus Bridge
```
org.openmuc.jmbus (version 3.3.0)
org.openmuc.jrxtx (version 1.0)
io.openems.common
io.openems.edge.common
```

### OneWire Bridge
```
RXTXcomm.jar (serial communication)
Maxim OneWire C Library (native, via JNI)
io.openems.common
io.openems.edge.common
```

---

## Conclusion

The OpenEMS bridge architecture provides a flexible, extensible framework for integrating diverse communication protocols. Each bridge is optimized for its specific use case while maintaining consistent patterns for error handling, monitoring, and integration with the OpenEMS component lifecycle.

Key architectural principles:
- **Abstraction**: Protocol details hidden behind clean interfaces
- **Task-Based**: Common task execution model across bridges
- **Event-Driven**: Integration with OpenEMS cycle events
- **Monitoring**: Rich channel-based status information
- **Error Handling**: Graceful degradation and recovery
- **Extensibility**: Easy to add new devices and protocols

---

**Document Version**: 1.0  
**Last Updated**: January 28, 2026  
**OpenEMS Version**: Based on current development branch
