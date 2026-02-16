
# CAN Bus Support for BMS Communication

## Purpose

The **CAN Bus Support** feature enables the EMS to integrate and manage **Battery Management Systems (BMS)** and other energy devices that communicate via the **Controller Area Network (CAN)** protocol. It provides a unified, scalable architecture to:

- Receive real‑time telemetry (SOC, voltage, current, temperature) from BMS units.
- Send control commands (charge/discharge limits, balancing activation, fault resets) to BMS units.
- Handle multiple BMS units, including those with identical CAN identifiers, through physical or logical separation.
- Abstract CAN‑specific details from higher‑level EMS functions, presenting normalized device channels.

This feature extends the EMS beyond traditional Modbus connectivity, allowing it to interface with modern lithium‑ion battery systems commonly found in energy storage, electric vehicle charging, and industrial applications.

---
## Architectural Overview

The CAN BMS integration follows a **layered architecture** that separates concerns and provides clear interfaces between physical hardware, operating system, EMS services, and higher‑level applications. The diagram below illustrates the main layers and data flow.

![Can Bus Support Diagram](./can-protocol-diagram.png)
---

## Layer Descriptions

### Physical Layer
- **Components:** Physical CAN buses (twisted pair) with termination resistors. Each bus connects a group of devices (e.g., one BMS per battery string).
- **Purpose:** Provides the electrical medium for CAN communication. Multiple buses allow physical separation of devices with identical CAN IDs.

### Hardware Interface Layer
- **Components:** CAN interface hardware on the edge gateway.
- **Purpose:** Converts electrical signals to/from digital frames and presents them to the operating system. Each bus requires a dedicated interface.

### CAN Stack (OS) Layer
- **Component:** `SocketCAN` (Linux) or equivalent driver.
- **Purpose:** Provides a unified API for user‑space applications to send and receive CAN frames. Handles low‑level timing, error detection, and basic filtering.

### EMS CAN Services Layer
This layer contains the core logic for CAN device integration:

| Sub‑component | Responsibility |
|---------------|----------------|
| **CAN Message Receiver** | Listens for raw CAN frames from all active interfaces, timestamps them, and forwards to the parser. |
| **Protocol Parser / Dispatcher** | Uses device profiles to decode frames into engineering values (e.g., SOC, voltage). Routes values to the Device Abstraction Layer. |
| **Command Encoder** | Formats high‑level commands (e.g., "set charge current") into CAN frames according to the device profile. |
| **Device Profile Engine** | Manages the loading and caching of device profiles. Provides parsing rules to the parser and encoding rules to the encoder. |
| **Local Profile Cache** | Stores device profiles locally (e.g., SQLite) for offline operation and fast access. Synced with the cloud configuration repository. |

### Device Abstraction Layer
- **Component:** A set of normalized channels, each with a name, value, timestamp, and unit.
- **Purpose:** Hides protocol specifics from the rest of the EMS. All controllers and cloud services consume data via these channels (e.g., `bms_0/Soc`, `bms_0/Voltage`). Commands are issued by writing to specific command channels (e.g., `bms_0/SetChargeCurrent`).

### Core EMS Integration
- **Core Cycle & Scheduler:** Reads a snapshot of all channels at the beginning of each cycle and executes control algorithms (balancing, peak shaving, etc.).
- **Control Algorithms:** Use normalized channels to compute setpoints, which are sent to the **Command Dispatcher**.
- **Command Dispatcher:** Routes setpoints (e.g., from algorithms or cloud) to the appropriate device‑specific command encoders (e.g., `Command Encoder` in CAN Services).

### Cloud Platform Integration
- **Configuration Repository:** Stores device profiles and edge configurations. Pushed to edges via the **Edge Configuration Sync** feature.
- **Telemetry Database (TimescaleDB):** Receives historical telemetry from the Device Abstraction Layer for reporting and analysis.
- **Alarm Engine:** Monitors channel values against user‑defined rules and triggers notifications.
- **Command API / UI:** Allows users to send manual commands (e.g., override charge limits) from the dashboard.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **CAN Bus** | A robust multi‑master serial bus standard (ISO 11898) widely used in automotive and industrial systems. It uses differential signalling over a twisted pair and supports data rates up to 1 Mbit/s. |
| **BMS (Battery Management System)** | An electronic system that manages a rechargeable battery by monitoring its state, protecting it from unsafe conditions, and balancing cells. BMS units typically communicate over CAN using standard or proprietary protocols. |
| **CANopen** | A higher‑layer protocol based on CAN, commonly used in battery systems (CiA 454 "EnergyBus"). It defines an object dictionary, PDOs (Process Data Objects) for cyclic data, and SDOs (Service Data Objects) for configuration. |
| **SAE J1939** | A CAN‑based protocol used in heavy‑duty vehicles and large battery packs. It uses 29‑bit identifiers and predefined parameter groups (PGNs) for battery data. |
| **Proprietary CAN Protocols** | Many BMS manufacturers implement custom CAN message formats. Integration requires a protocol specification from the vendor. |
| **CAN Identifier (ID)** | Each CAN message has a unique ID (11‑bit standard or 29‑bit extended) that determines its priority and identifies the sender. Multiple devices on the same bus must use distinct IDs to avoid collisions. |
| **CAN ID Conflict** | When two or more BMS units are configured with the same CAN ID, they cannot be distinguished on a single bus. Solutions include separate physical buses, external CAN bridges, or software remapping if the device supports ID changes. |
| **Device Profile** | A JSON configuration that defines how to parse incoming CAN messages and format outgoing commands for a specific BMS model. It includes CAN IDs, byte mappings, data types, scaling factors, and units. |
---

## Data Flow Summary

| Direction | Path | Description |
|-----------|------|-------------|
| **Upstream (Telemetry)** | Physical → Hardware → CAN Stack → CAN Receiver → Parser → Device Abstraction → Cloud (Telemetry DB, Alarm Engine) | Real‑time BMS data flows upward for monitoring and storage. |
| **Downstream (Commands)** | Cloud (Command API) → Command Dispatcher → Command Encoder → CAN Stack → Hardware → Physical | User or algorithm‑generated commands flow down to the BMS. |
| **Configuration Sync** | Cloud Configuration Repository ↔ Local Profile Cache | Device profiles are synchronised to the edge for offline use. |

---

## Functional Requirements

- The system shall support multiple independent CAN buses, each with its own hardware interface.
- The system shall provide a framework for device‑specific protocol parsers (CANopen, J1939, proprietary).
- The system shall maintain a local cache of device profiles, synchronised with the cloud.
- The system shall expose normalized channels for each BMS (Soc, Voltage, Current, Temperature, etc.) to the Device Abstraction Layer.
- The system shall support sending commands to BMS units (e.g., set charge current, enable balancing) via CAN.
- The system shall handle BMS units with identical CAN IDs through separate physical buses or external CAN bridges.
- The system shall timestamp all received CAN frames with microsecond precision for temporal coherence.
- The system shall monitor CAN bus health (error frames, bus off) and report diagnostics to the cloud.

---

## Metrics & Performance Criteria

| Metric | Target | Measurement Method |
|--------|--------|---------------------|
| **Message reception latency** | < 10 ms (from bus to channel update) | Timestamp comparison (hardware vs. snapshot) |
| **Maximum bus load** | Up to 80% without message loss | Simulated high‑load test |
| **Number of BMS units per edge** | ≥ 10 (assuming moderate data rates) | Integration test with multiple simulators |
| **Profile sync time** | < 30 seconds from cloud change to edge activation | End‑to‑end timing |
| **Command round‑trip time** | < 500 ms (user → cloud → edge → BMS → acknowledgment) | Instrumented test |

---

## Dependencies on Other EMS Features

| Feature | Dependency Description |
|---------|------------------------|
| **Edge Core Cycle** | Provides the execution context; normalized CAN channels are read by the cycle as inputs. |
| **Device Abstraction Layer** | Consumes normalized channels from CAN services and makes them available to controllers and cloud. |
| **Command Dispatch** | Forwards user/algorithm commands to the CAN command encoder. |
| **Edge Configuration Sync** | Delivers device profiles and CAN interface configuration from cloud to edge. |
| **Multi‑Tenancy** | Ensures device data is isolated per organization/site. |

---

## Use Cases

### Single BMS Integration
- A site has one lithium‑ion battery pack with a CANopen BMS.
- EMS loads the BMS profile, receives periodic PDOs, and updates `bms_0/Soc`, `bms_0/Voltage`, etc.
- Controllers use these values for peak shaving and self‑consumption.

### Multiple BMS with Identical CAN IDs
- A large storage system uses three identical BMS units, all transmitting with CAN ID `0x181`.
- The site uses three separate CAN buses (`can0`, `can1`, `can2`), each connected to one BMS.
- EMS creates three logical devices (`bms_0`, `bms_1`, `bms_2`), each associated with a distinct bus and separate profiles.

### Commissioning a New BMS
- An installer connects a new BMS and adds its profile via the cloud UI.
- The edge syncs the profile, starts listening for the specified CAN IDs, and begins populating channels.

---

## Relevant Standards and References

| Standard / Reference | Application |
|----------------------|-------------|
| **ISO 11898‑1/2** | Physical and data link layer specifications for CAN. |
| **CiA 301 (CANopen)** | Application layer and communication profile for CANopen devices. |
| **CiA 454 (EnergyBus)** | CANopen profile for energy management systems and battery interfaces. |
| **Ixxat CANbridge NT420** | Example external CAN bridge for ID remapping (reference for handling identical IDs). |

---