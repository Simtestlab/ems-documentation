# SunSpec Compliance – Technical Documentation

## Purpose

The SunSpec Compliance feature enables the EMS to **automatically discover and communicate with SunSpec‑compliant inverters** (PV, battery, and other DERs) over Modbus. SunSpec is an industry standard that defines common register maps for distributed energy resources, allowing plug‑and‑play integration without manual configuration.

By implementing SunSpec support, the EMS can:

- Auto‑detect connected SunSpec devices on a Modbus network.
- Parse standard SunSpec data models to extract real‑time telemetry (power, voltage, current, energy, status).
- Map these values to normalized EMS channels for use in control algorithms and monitoring.
- Reduce manual configuration effort and eliminate register‑map guesswork.

This feature extends the **Modbus Protocol Adapter** by adding a discovery engine and model parser that work alongside the existing HAL and scan engine.

---

## Key Concepts

| Concept | Description |
|---------|-------------|
| **SunSpec Alliance** | An industry trade group that develops open standards for DER communication. |
| **SunSpec Model** | A well‑defined set of registers that describe a particular device type or function (e.g., Model 1 – Common, Model 101 – Inverter, Model 160 – Meter). Each model has a unique ID. |
| **Model ID** | A 16‑bit identifier stored in a fixed register (typically at offset 0) that indicates which SunSpec model is implemented starting at that address. |
| **SunSpec Header** | The first block of every SunSpec device, containing the SunSpec identifier (`0x53756E53` = "SunS") and the model count. |
| **Discovery** | The process of reading the SunSpec header and then sequentially reading model IDs to build a complete map of the device's capabilities. |
| **Mandatory Models** | All SunSpec devices must implement at least the Common model (Model 1) and an appropriate device‑specific model (e.g., Model 101 for single‑phase inverters). |
| **SunSpec‑compliant Device** | Any device that implements SunSpec models as defined by the SunSpec Alliance specifications. |

---

## Architectural Overview

SunSpec support is implemented as an **extension to the Modbus Protocol Adapter**. It adds a **discovery engine** and a **model parser** that sit alongside the existing HAL and scan engine.

- **Physical Layer:** SunSpec‑compliant inverters connected via Modbus TCP/RTU.
- **Hardware Interfaces:** Ethernet/RS‑485 ports on the edge gateway.
- **Modbus Protocol Adapter:** The existing adapter now includes a SunSpec Engine.
- **Device Abstraction Layer:** Normalized channels from SunSpec devices appear identically to those from manually configured devices.
- **Core EMS & Cloud:** Higher‑level functions consume SunSpec data without knowing its origin.

### How It Fits

| Existing Component | SunSpec Integration |
|-------------------|---------------------|
| **Hardware Abstraction Layer** | Enhanced to recognize SunSpec devices and trigger discovery. |
| **Scan Engine** | Uses dynamically generated profiles from discovery. |
| **Device Profiles** | Auto‑generated from discovered models, cached locally. |
| **Edge Configuration Sync** | Can store auto‑generated profiles for faster startup. |

---

## Functional Requirements

- The system shall be able to discover SunSpec‑compliant devices on any Modbus TCP/RTU network by reading the SunSpec header starting at a configurable base address (default 40000)
- The system shall parse the SunSpec header to verify the presence of the SunSpec identifier (`0x53756E53`) and determine the number of models implemented.
- The system shall sequentially read model IDs from the device and construct a complete model map.
- The system shall support the most common SunSpec models (at minimum: Model 1 – Common, Model 101 – Inverter, Model 102 – Inverter (three‑phase), Model 103 – Inverter (single‑phase), Model 160 – Meter).
- For each supported model, the system shall map its registers to normalized EMS channels (e.g., `ActivePower`, `Voltage`, `Current`, `Frequency`, `EnergyTotal`) with correct scaling and units as defined by the SunSpec specification.
- The system shall allow manual override of SunSpec discovery via a configured device profile (if the user prefers a fixed register map).
- The system shall log the discovered models and register maps for diagnostic purposes.
- The system shall support caching of discovered SunSpec profiles locally (in the Edge Configuration Sync cache) to accelerate reconnection.
- The system shall raise an alarm if a SunSpec device is detected but fails to respond to model reads (partial discovery).
---

## Discovery Process (Data Flow)

```
[Start Discovery]
       │
       ▼
┌─────────────────────────────┐
│ Read SunSpec header at      │
│ base address (e.g., 40000)  │
└──────────────┬──────────────┘
       │
       ▼
┌─────────────────────────────┐
│ Verify "SunS" signature     │
│ (0x53756E53)                │
└──────────────┬──────────────┘
       │
       ▼
┌─────────────────────────────┐
│ If signature valid:         │
│ Read model count N          │
│ from header                 │
└──────────────┬──────────────┘
       │
       ▼
┌─────────────────────────────┐
│ For i = 1 to N:             │
│   Read model ID at          │
│   current address           │
│   If ID == 0xFFFF → stop    │
│   Look up model definition  │
│   Add to model map          │
│   Advance address by        │
│   model length              │
└──────────────┬──────────────┘
       │
       ▼
┌─────────────────────────────┐
│ Generate dynamic profile    │
│ with all discovered models  │
└─────────────────────────────┘
```

Once the dynamic profile is generated, the scan engine uses it to poll the device registers according to the model definitions (fast/slow groups can be assigned based on register type).

---

## Integration with Modbus Protocol Adapter

| Aspect | Description |
|--------|-------------|
| **Discovery Trigger** | Discovery can be triggered automatically when a new device is added (via Edge Configuration Sync) with `protocol: "sunspec"` or manually via API. |
| **Profile Generation** | The discovered model map is converted into an internal JSON profile similar to the static device profiles, but with `auto_generated: true`. |
| **Channel Naming** | Channels are named following the EMS convention, e.g., `sunspec_inv_0/ActivePower`, `sunspec_inv_0/TotalEnergy`. The device ID is assigned based on the Modbus unit ID and bus. |
| **Polling Groups** | The scan engine automatically assigns registers from fast‑changing models (e.g., Model 101 AC measurements) to the `FAST` group and energy accumulators to the `SLOW` group. |
| **Error Handling** | If a device fails SunSpec discovery, the adapter falls back to the configured static profile (if any) or marks the device as unsupported. |

---

## Example: Model 101 (Single‑Phase Inverter)

SunSpec Model 101 defines registers for basic inverter measurements. A partial mapping:

| Register Offset | Length | Type | Scale | Unit | EMS Channel |
|-----------------|--------|------|-------|------|-------------|
| 0 | 1 | uint16 | – | – | Model ID (101) |
| 2 | 1 | uint16 | – | – | Inverter status |
| 4 | 2 | uint32 | 1 | W | `ActivePower` |
| 8 | 2 | uint32 | 0.001 | kWh | `TotalEnergy` |
| 12 | 2 | int32 | 0.1 | V | `Voltage` |
| 16 | 2 | int32 | 0.1 | A | `Current` |
| 20 | 2 | int16 | 0.01 | Hz | `Frequency` |

The SunSpec engine reads these offsets, applies scaling, and publishes channels like `sunspec_inv_0/ActivePower`.

---

## Device Profile Example (Auto‑Generated)

```json
{
  "profile_id": "sunspec_auto_101_001",
  "protocol": "Modbus_TCP",
  "sunspec": true,
  "discovered_models": [1, 101, 120],
  "connection": {
    "port": 502,
    "unit_id": 1
  },
  "optimization": {
    "fast_poll_ms": 500,
    "slow_poll_ms": 10000
  },
  "channels": [
    {
      "tag": "ActivePower",
      "model": 101,
      "offset": 4,
      "length": 2,
      "type": "UINT32",
      "scale": 1.0,
      "unit": "W",
      "poll_group": "fast"
    },
    {
      "tag": "TotalEnergy",
      "model": 101,
      "offset": 8,
      "length": 2,
      "type": "UINT32",
      "scale": 0.001,
      "unit": "kWh",
      "poll_group": "slow"
    },
    {
      "tag": "Voltage",
      "model": 101,
      "offset": 12,
      "length": 2,
      "type": "INT32",
      "scale": 0.1,
      "unit": "V",
      "poll_group": "fast"
    }
  ]
}
```

This profile is stored in the local cache and used by the scan engine exactly like a manually configured profile.

---

## Real‑World Example: SunSpec on a TRUMPF Inverter

While the TRUMPF TruConvert AC 3025 documented earlier is not SunSpec‑compliant, a SunSpec‑compliant inverter (e.g., from SMA, Fronius, or SolarEdge) would be discovered as follows:

1. **Connection:** Inverter connected via Modbus TCP to the edge gateway.
2. **Discovery:** EMS reads the SunSpec header at address 40000, finds `0x53756E53`, and discovers models (e.g., 1, 101, 120).
3. **Profile Generation:** An auto‑generated profile like the one above is created.
4. **Channel Creation:** The inverter appears in the EMS as `sunspec_inv_0` with channels for power, energy, voltage, etc.
5. **Control:** If the inverter supports SunSpec writable registers (e.g., power curtailment), the EMS can send commands via the same auto‑generated profile.

This contrasts with the manual profile approach used for the TRUMPF inverter, where an engineer must create the profile from the manual.

---

## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Discovery time** | < 5 seconds for a typical device (5 models) | From initiation to profile generation |
| **Model parsing accuracy** | 100% of known models map to correct channels | Unit tests against SunSpec simulator |
| **Auto‑detection rate** | ≥ 99% of SunSpec‑compliant devices detected | Field trial with multiple device types |
| **Polling correctness** | Registers read according to model lengths | Verification with Modbus sniffer |
| **Fallback to manual profile** | Successful when discovery fails | Integration test |

---

## Dependencies on Other EMS Features

| Feature | Dependency Description |
|---------|------------------------|
| **Modbus Protocol Adapter** | Provides the underlying Modbus communication and scan engine. SunSpec is an extension of this adapter. |
| **Device Abstraction Layer** | Consumes the normalized channels produced from SunSpec models. |
| **Edge Configuration Sync** | May store auto‑generated profiles locally for faster startup and offline use. |
| **Multi‑Tenancy** | Ensures discovered devices are correctly associated with the right site/organization. |

---

## Use Cases

### Commissioning a New PV Inverter
- Installer connects a new SunSpec‑compliant inverter to the Modbus network.
- EMS automatically discovers it during the next scan, creates a profile, and begins populating channels.
- The installer verifies the data in the dashboard and configures control algorithms (e.g., curtailment) without any register mapping.

### Replacing a Failed Inverter
- An old inverter is replaced with a newer SunSpec model.
- EMS detects a new device on the same Modbus unit ID, runs discovery, and updates the profile automatically.
- Historical data continuity is maintained via the same device ID in the abstraction layer.

### Mixed Fleet with Non‑SunSpec Devices
- A site has some legacy inverters with custom Modbus maps (configured via static profiles) and new SunSpec inverters.
- The Modbus adapter handles both seamlessly: static profiles are used for legacy devices, dynamic discovery for SunSpec devices.

---