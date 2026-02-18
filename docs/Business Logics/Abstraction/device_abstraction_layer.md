
# Device Abstraction Layer (DAL)

## Purpose

The **Device Abstraction Layer (DAL)** is the central nervous system of the EMS Edge node. Its job is to **hide the complexity of physical devices** and provide a **unified, consistent view of all data** to the rest of the system – controllers, cloud, and UI.

Think of it as a **real‑time dictionary** that holds the latest value of every measurable or controllable property of your energy system. Whether a value comes from a Modbus inverter, a CAN battery, or a simulated device, it appears in the DAL with a **standard name, unit, and timestamp**.

### Why It Matters
- **Protocol agnosticism:** Controllers don't need to know if a battery speaks Modbus or CAN – they just read `battery_1/Soc`.
- **Decoupling:** Protocol adapters can be developed, tested, and replaced independently.
- **Data quality:** All channels carry timestamps, so downstream components know how fresh the data is.
- **Extensibility:** New device types can be added by defining new channel names – no changes to the core cycle or controllers.

---

## Role in the EMS Architecture

The DAL sits at the heart of the edge software stack:

```
┌─────────────────────────────────────────────┐
│           Cloud / Dashboard                  │
│  (UI, telemetry database, alarms)            │
└─────────────────────┬───────────────────────┘
                      │ (WebSocket / API)
┌─────────────────────▼───────────────────────┐
│           Edge Core Cycle & Scheduler        │
│  (runs controllers, dispatches commands)     │
└─────────────────────┬───────────────────────┘
                      │ (reads snapshot, writes setpoints)
┌─────────────────────▼───────────────────────┐
│         DEVICE ABSTRACTION LAYER (DAL)       │ ← You are here
│  • Holds all normalized channels             │
│  • Provides atomic snapshots                  │
│  • Thread‑safe read/write                     │
└─────────────────────┬───────────────────────┘
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Modbus   │ │   CAN    │ │SunSpec   │
   │ Adapter  │ │ Adapter  │ │Engine    │
   └──────────┘ └──────────┘ └──────────┘
          │           │           │
          ▼           ▼           ▼
      [Physical Devices]  [Simulated Models]
```

- **Protocol adapters** (Modbus, CAN, SunSpec, simulators) write incoming telemetry **into** the DAL.
- The **Edge Core Cycle** takes a **snapshot** of all input channels **from** the DAL at the start of each cycle.
- **Controllers** read this snapshot (they never read live channels directly) and may write setpoints **back to** the DAL (as output channels).
- The **Command Dispatcher** reads output channels from the DAL and sends them to the protocol adapters for execution.
- **Cloud telemetry** and the **Alarm Engine** also consume channels from the DAL (via streaming or polling).

---

## Key Concepts

### Channel
A **channel** is a named variable representing a single piece of data. Every channel has:
- **Name:** e.g., `battery_1/Soc`
- **Value:** e.g., `67.5`
- **Unit:** e.g., `%`
- **Timestamp:** when the value was measured or last updated (microsecond precision).
- **Quality (optional):** e.g., `GOOD`, `STALE`, `ERROR`

### Naming Convention
We use a hierarchical naming scheme:  
`{device_type}_{instance_id}/{channel_name}`

Examples:
- `battery_1/Soc` – State of charge of the first battery.
- `grid/ActivePower` – Grid import/export power (positive = import).
- `pv_2/EnergyTotal` – Total energy produced by the second PV inverter.
- `load/ActivePower` – Site consumption (if directly metered).
- `virtual/NetLoad` – Computed net load (grid + battery).

### Channel Categories

| Category | Direction | Description |
|----------|-----------|-------------|
| **Input channels** | Read‑only | Measurements from devices (telemetry). Written by protocol adapters. |
| **Output channels** | Write‑only | Setpoints for devices (commands). Written by controllers, read by command dispatcher. |
| **Derived channels** | Read‑only | Computed values (e.g., virtual meters). Written by separate services (e.g., virtual meter service). |

### Snapshot
A **snapshot** is a consistent, read‑only copy of all input channels taken at a single instant. The Edge Core Cycle uses snapshots to ensure all controllers see the same data, eliminating race conditions.

### Timestamping
Every channel update must include a timestamp. This is typically provided by the protocol adapter (the time the data was requested or received). Timestamps are critical for:
- Detecting stale data.
- Aligning data from different sources.
- Historical analysis.

### Quality Flags
- `GOOD` – Value is fresh and reliable.
- `STALE` – Value is older than a threshold (e.g., no update for 5 cycles).
- `ERROR` – Device communication failed.

---

## Functional Requirements

- Provide thread‑safe read and write access to channels for multiple concurrent producers and consumers.
- Support dynamic addition and removal of channels (e.g., when a device is discovered or disconnected).
- Allow a consumer to take an atomic snapshot of all input channels (bulk read) with consistent timestamps.
- Maintain the latest value and its timestamp for every channel.
- Expose channels via a simple API: `get_channel(name)`, `set_channel(name, value, timestamp)`, `snapshot()`.
- Support optional quality flags per channel.
- Provide a subscription mechanism for real‑time updates (used by cloud telemetry, alarm engine).
- Enforce naming conventions and unit consistency (e.g., all power channels in watts).

---

## Relationship with Other EMS Features

| Feature | Interaction with DAL |
|---------|----------------------|
| **Edge Core Cycle** | Takes a snapshot at cycle start; writes final setpoints back. |
| **Controllers** | Read snapshot (never live channels); write setpoints to output channels. |
| **Modbus/CAN/SunSpec Adapters** | Write incoming telemetry to input channels. |
| **Command Dispatcher** | Reads output channels and sends commands to devices. |
| **Cloud Telemetry** | Subscribes to channel updates and streams them to cloud. |
| **Alarm Engine** | Monitors input channels for threshold violations. |
| **Virtual Meter Service** | Reads input channels, computes derived values, writes them back as new input channels. |
| **Advanced SOC Estimation (Kalman/ML)** | Reads raw battery channels (voltage, current, temperature), writes enhanced `Soc` to DAL. |

**Important:** The DAL itself **does not perform calculations** like SOC estimation or virtual metering. It is just a storage and retrieval mechanism. All intelligence lives in separate services that read from and write to the DAL.

---

## Advanced SOC Estimation

**Kalman filter** and **Machine Learning SOC** are **not part of the DAL**. They are separate components that:

1. Subscribe to or periodically read raw battery channels from DAL (e.g., `battery_1/Voltage`, `battery_1/Current`, `battery_1/Temperature`).
2. Run their algorithms to compute an improved SOC estimate.
3. Write the result back to a DAL channel, e.g., `battery_1/SocEnhanced` or even overwrite `battery_1/Soc` (if configured).

---

## Virtual Meter Service

Virtual meters are computed channels that aggregate or transform existing channels. They are implemented as a **separate service** (which could be a controller or a standalone microservice) that:

- Runs periodically (e.g., every cycle or every few seconds).
- Reads relevant input channels from DAL.
- Computes derived values.
- Writes them back to DAL as new input channels (e.g., `virtual/NetLoad`).

These derived channels then become available to controllers, UI, and cloud just like any other channel.

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Read latency** | < 1 ms | Time to retrieve a single channel. |
| **Write latency** | < 1 ms | Time to store a channel update. |
| **Snapshot time** | < 10 ms for 1000 channels | Time to copy all input channels. |
| **Concurrency** | No data corruption under 100 simultaneous writes/reads | Stress test with multiple threads. |
| **Timestamp accuracy** | Timestamps reflect actual measurement time | Compare with source. |

---
## Glossary

- **Channel:** A named variable with a value, unit, timestamp, and optional quality.
- **Input channel:** Read‑only, represents telemetry from devices.
- **Output channel:** Writable, represents commands to devices.
- **Derived channel:** Computed from other channels, appears as input.
- **Snapshot:** A consistent, atomic copy of all input channels at a moment in time.
- **Virtual meter:** A service that computes derived channels.

---