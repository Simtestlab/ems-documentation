# Energy Management System Architecture

## Unified Design for **myems + openems**

This document defines the **combined architecture, data structures, and parameters** of two Energy Management platforms:

* **myems** → microservice ETL + analytics + dashboard system
* **openems** → real-time device/channel control framework

It also proposes a **unified architecture** suitable for:

* dashboards
* analytics
* real-time monitoring
* edge control
* scalable cloud deployment

---

# 1. Architecture Overview

## myems – ETL & Analytics Pipeline

![Image](https://static.wixstatic.com/media/904900_03cec6a515434918ad8db97814d98a5c~mv2.png/v1/fill/w_1000%2Ch_510%2Cal_c%2Cq_90%2Cusm_0.66_1.00_0.01/904900_03cec6a515434918ad8db97814d98a5c~mv2.png)

![Image](https://images.openai.com/static-rsc-3/AJxCH_7sFzg4zIlSqrVj5mI8e9gHkx7eB6VxLJHp8YyClExBEEwj_4QQAacD_USevke9kZd1MVp3os5H19bNxxdvwvjPfRUyrDLOwyYTLPA?purpose=fullsize)

![Image](https://images.ctfassets.net/fevtq3bap7tj/4Z3xdca3bymwimUoa408Ck/8c3bf8a8d2335a9613131d03c650131b/Energy_Dashboard_2x.jpg.png)

![Image](https://cdn.dribbble.com/userupload/26440454/file/original-e0a6103cbddd6aaf17cb38abb38135fd.png?resize=400x0)

### Flow

```
Meters → Modbus → Cleaning → Normalization → Aggregation → DB → API → UI
```

### Design Traits

* Database first
* Historical analytics
* Batch processing
* Microservices
* Reporting/Billing focused

---

## openems – Real-Time Component Framework

![Image](https://www.researchgate.net/publication/337984796/figure/fig5/AS%3A840330547048449%401577361823575/Edge-computing-based-IoT-architecture.png)

![Image](https://www.researchgate.net/publication/262528253/figure/fig9/AS%3A667914940203013%401536254739291/Block-diagram-of-smart-power-grid-composed-of-an-energy-source-users-with-smart-meter.jpg)

![Image](https://docs.oracle.com/en/database/oracle/oracle-database/26/dshdg/img/telemetry_streaming-fig-2-architecture.png)

![Image](https://www.researchgate.net/publication/322583817/figure/fig3/AS%3A713475991019527%401547117340941/Telemetry-service-architecture-for-fully-disaggregated-FD-white-boxes.png)

### Flow

```
Devices → Components → Channels → Controllers → Edge → Backend
```

### Design Traits

* Object oriented
* Real-time telemetry
* Device abstraction
* Edge computing
* Control & automation focused

---

---

# 2. Folder Structure Comparison

## myems

```
database/
myems-modbus-tcp/
myems-cleaning/
myems-normalization/
myems-aggregation/
myems-api/
myems-web/
myems-admin/
```

| Module        | Purpose           |
| ------------- | ----------------- |
| modbus-tcp    | device polling    |
| cleaning      | bad data removal  |
| normalization | unit conversion   |
| aggregation   | analytics         |
| api           | REST gateway      |
| web           | user dashboard    |
| admin         | configuration     |
| database      | schema/migrations |

---

## openems

```
edge/
  component/
  controller/
  meter/
  battery/
  inverter/
backend/
ui/
```

| Module     | Purpose             |
| ---------- | ------------------- |
| component  | device abstraction  |
| channel    | measurement signals |
| controller | logic/control       |
| edge       | real-time execution |
| backend    | sync/storage        |
| ui         | monitoring          |

---

---

# 3. Core Data Structures

---

## 3.1 myems – Time-Series Model

### RawReading

```ts
RawReading {
  meter_id: number
  timestamp: datetime
  register: number
  value: float
  quality: string
}
```

### CleanReading

```ts
CleanReading {
  meter_id
  timestamp
  value
  status
}
```

### NormalizedReading

```ts
NormalizedReading {
  meter_id
  timestamp
  value_kwh
  unit
}
```

### AggregatedEnergy

```ts
AggregatedEnergy {
  meter_id
  period
  energy_kwh
  demand_kw
  peak_kw
  cost
  co2_kg
}
```

### Pattern

```
(meter_id, timestamp, value)
```

---

## 3.2 openems – Component/Channel Model

### Component

```ts
Component {
  id
  alias
  enabled
  channels[]
}
```

### Channel

```ts
Channel {
  id
  type
  unit
  value
  timestamp
}
```

### Pattern

```
component.channel.value
```

---

---

# 4. Energy Component Parameters (Complete)

---

## Electrical Measurements

| Parameter     | Unit | Meaning       |
| ------------- | ---- | ------------- |
| Voltage       | V    | line voltage  |
| Current       | A    | phase current |
| ActivePower   | kW   | real power    |
| ReactivePower | kVAR | reactive      |
| ApparentPower | kVA  | total         |
| Frequency     | Hz   | grid freq     |
| PowerFactor   | –    | efficiency    |

---

## Energy Metrics

| Parameter    | Unit     |
| ------------ | -------- |
| EnergyImport | kWh      |
| EnergyExport | kWh      |
| TotalEnergy  | kWh      |
| Demand       | kW       |
| PeakDemand   | kW       |
| Cost         | currency |
| CO2          | kg       |

---

## Battery Metrics

| Parameter      | Unit |
| -------------- | ---- |
| SOC            | %    |
| ChargePower    | kW   |
| DischargePower | kW   |
| Voltage        | V    |
| Current        | A    |
| Temperature    | °C   |
| Capacity       | kWh  |

---

## Inverter Metrics

| Parameter  | Unit    |
| ---------- | ------- |
| AC_Power   | kW      |
| DC_Power   | kW      |
| Efficiency | %       |
| State      | enum    |
| Fault      | boolean |

---

---

# 5. Conceptual Comparison

| Feature    | myems      | openems     |
| ---------- | ---------- | ----------- |
| Core model | DB rows    | objects     |
| Data style | batch      | real-time   |
| Storage    | SQL        | memory/edge |
| Analytics  | strong     | moderate    |
| Control    | weak       | strong      |
| Best for   | dashboards | automation  |

---

---

# 6. Mapping Between Systems

| openems       | myems equivalent |
| ------------- | ---------------- |
| Component.id  | meter_id         |
| Channel.name  | register         |
| Channel.value | value            |
| timestamp     | timestamp        |
| ActivePower   | demand_kw        |
| EnergyImport  | energy_kwh       |
| SOC           | battery_soc      |

---

---

# 7. Unified Architecture (Recommended)

## Hybrid Flow

```
Devices
   ↓
OpenEMS Edge (channels)
   ↓
Timeseries DB (myems style)
   ↓
API
   ↓
React Dashboard
```

---

## Unified Data Model (Recommended)

### Device

```ts
Device {
  id
  type
  location
  metadata
}
```

### Channel

```ts
Channel {
  device_id
  name
  unit
}
```

### Reading

```ts
Reading {
  device_id
  channel
  timestamp
  value
}
```

### Aggregate

```ts
Aggregate {
  device_id
  period
  energy
  demand
  cost
}
```

---

---

# 8. Recommended API Format

```json
{
  "deviceId": "meter-01",
  "channel": "ActivePower",
  "value": 3200,
  "unit": "W",
  "timestamp": 1734200000
}
```

### Suitable for

* charts
* React Flow nodes
* live dashboards
* alerts
* analytics

---

---

# 9. Recommended Tech Stack

### Backend

* FastAPI / Node
* TimescaleDB / PostgreSQL
* Redis
* MQTT (optional)

### Frontend

* React
* MUI / Chakra
* Recharts / ECharts
* Zustand

### Edge

* OpenEMS components

---

---

# 10. When to Use What

### Use myems if:

* reports
* billing
* analytics
* historical queries

### Use openems if:

* device control
* live telemetry
* edge logic


#  Energy System Components → Parameters Mapping Guide

Think physically first, not software first:

```
Grid → Meter → Inverter → Battery → Load → Solar
```

Each device exposes **different electrical/energy parameters**.

---

---

# 1️. Energy Meter (Grid / Feeder / Load Meter)

![Image](https://image.made-in-china.com/2f0j00medWOIvBwhRH/DIN-Rail-Panel-Three-3-Phase-Smart-Meter-Energy-Meter-for-Electricity-Power-Qualtiy-Analyzer.webp)

![Image](https://www.hioki.com/system/files/image/2021-06/PQ3198EN_1_.png)

![Image](https://image.made-in-china.com/365f3j00lDSWioqKZubU/CT-Operate-Meter-Box-Metering-Unit-200A-250A-300A-500A-Customized.webp)

## Purpose

Measures **electrical quantities + energy consumption**

## Parameters

### Electrical (instantaneous)

| Parameter        | Unit | Meaning        | Needed for      |
| ---------------- | ---- | -------------- | --------------- |
| Voltage_L1/L2/L3 | V    | phase voltages | power calc      |
| Current_L1/L2/L3 | A    | phase currents | load monitoring |
| Frequency        | Hz   | grid frequency | stability       |
| PowerFactor      | –    | efficiency     | billing/quality |

---

### Power

| Parameter     | Unit | Meaning           |
| ------------- | ---- | ----------------- |
| ActivePower   | kW   | real usable power |
| ReactivePower | kVAR | non-working power |
| ApparentPower | kVA  | total demand      |

---

### Energy

| Parameter    | Unit | Meaning         |
| ------------ | ---- | --------------- |
| EnergyImport | kWh  | consumed energy |
| EnergyExport | kWh  | exported energy |
| TotalEnergy  | kWh  | cumulative      |

---

## System Mapping

### openems

```
meter0.ActivePower
meter0.EnergyImport
```

### myems

```
meter_id, timestamp, energy_kwh
```

---

---

# 2️. Battery Storage System (BESS)

![Image](https://www.tls-containers.com/uploads/1/1/3/0/11305885/bess-container-battery-energy-storage-system-container-tls-offshore-3-orig-orig.jpeg)

![Image](https://static.wixstatic.com/media/a5ba95_c8a1517e5be14b168861711096743cc4~mv2.jpg/v1/fill/w_560%2Ch_504%2Cal_c%2Cq_80%2Cusm_0.66_1.00_0.01%2Cenc_avif%2Cquality_auto/a5ba95_c8a1517e5be14b168861711096743cc4~mv2.jpg)

![Image](https://images.openai.com/static-rsc-3/YjnhG6NrO66CDT0tIQkD0ApESMaaE_QligpICC38iU_B9YsFwHqerX3b40hTZQ28O7SJeij56n14iKg-3Rg__h6iYoFeyzBHJVQoro6t3sY?purpose=fullsize)

![Image](https://eu-images.contentstack.com/v3/assets/blt0bbd1b20253587c0/bltf530092eabeaf67b/650c1cd3b3ac2181b098c830/AES_20Alamitos_20Battery_20Energy_20Storage_20System.jpg)

## Purpose

Stores and supplies energy

## Parameters

### State

| Parameter | Unit | Meaning         |
| --------- | ---- | --------------- |
| SOC       | %    | state of charge |
| SOH       | %    | state of health |
| Capacity  | kWh  | usable capacity |

---

### Electrical

| Parameter | Unit |
| --------- | ---- |
| Voltage   | V    |
| Current   | A    |
| Power     | kW   |

---

### Flow

| Parameter        | Unit | Meaning     |
| ---------------- | ---- | ----------- |
| ChargePower      | kW   | charging    |
| DischargePower   | kW   | discharging |
| EnergyCharged    | kWh  | stored      |
| EnergyDischarged | kWh  | used        |

---

### Safety

| Parameter   | Unit |
| ----------- | ---- |
| Temperature | °C   |
| Alarm/Fault | bool |

---

## System Mapping

### openems

```
battery0.SOC
battery0.ChargePower
```

### myems

```
battery_soc
battery_energy_kwh
```

---

---

# 3️. Solar / PV Inverter

![Image](https://www.everexceed.com/uploadfile/202509/25/37ad586ab4c17cee5edebfc6c4a59419.jpg)

![Image](https://media-01.imu.nl/storage/sinovoltaics.com/2234/wp/2018/12/Sunny_Boy_3000.jpg)

![Image](https://image.made-in-china.com/365f3j00CghiokFckyWV/RS485-WiFi-High-Output-Power-LED-Display-Hybrid-Inverter-Panel-Solar-Energy-System.webp)

![Image](https://m.media-amazon.com/images/I/71QEzAe3znL.jpg)

## Purpose

Converts DC → AC

## Parameters

### DC side

| Parameter  | Unit |
| ---------- | ---- |
| DC_Voltage | V    |
| DC_Current | A    |
| DC_Power   | kW   |

---

### AC side

| Parameter  | Unit |
| ---------- | ---- |
| AC_Voltage | V    |
| AC_Current | A    |
| AC_Power   | kW   |

---

### Performance

| Parameter       | Unit |
| --------------- | ---- |
| Efficiency      | %    |
| EnergyGenerated | kWh  |
| Status          | enum |
| Fault           | bool |

---

## System Mapping

### openems

```
inverter0.AC_Power
inverter0.Efficiency
```

### myems

```
generation_kwh
```

---

---

# 4️. Load / Equipment

![Image](https://www.pumpsandsystems.com/sites/default/files/0419/image-1-mm-schneider.jpg)

![Image](https://image.made-in-china.com/202f0j00OVafsUnrniqF/1u-Rack-Mount-6-Ways-Universal-PDU-with-V-a-Meter-for-Data-Center-Server-Rack.webp)

![Image](https://images.openai.com/static-rsc-3/MjEs34cykezxbzvBMeT2x66Bzc_13vTcZ7FaUysVw6XowsSgL5FcN4RkcWBaxJPmc0DfdQgW597ubkSJnzKKkiRTzziKJfH9dcyVFiw7WN4?purpose=fullsize)

![Image](https://transform.octanecdn.com/crop/1600x900/https%3A//octanecdn.com/estesaircom/estesaircom_743227702.JPG)

## Purpose

Consumes energy

## Parameters

| Parameter      | Unit   |
| -------------- | ------ |
| ActivePower    | kW     |
| EnergyConsumed | kWh    |
| Runtime        | hrs    |
| Demand         | kW     |
| PeakDemand     | kW     |
| Status         | on/off |

---

## Mapping

Same as **meter**, but logically tagged:

```
type = load
```

---

---

# 5️. Grid / Utility Connection

![Image](https://images.openai.com/static-rsc-3/F5pxdYq_kQAgzT9BcAyIaIWb-pbbhfer6QeAANSy7dsP0VBzoFotLIaJVCxce2jEfyu0ilxT9kMM5VXi9wC5f1ZhhSNYlnV39i_5v3bPfjo?purpose=fullsize)

![Image](https://eepower.com/uploads/thumbnails/featured-image-substation-switchyard-design-considerations-size-load-cost.jpg)

![Image](https://tees.tamu.edu/news/2022/05/_news-images/ecen-news-Davis-DOE-CESER-26May2022.jpg)

## Purpose

Tracks import/export with utility

## Parameters

| Parameter    | Unit         |
| ------------ | ------------ |
| ImportPower  | kW           |
| ExportPower  | kW           |
| ImportEnergy | kWh          |
| ExportEnergy | kWh          |
| Tariff       | currency/kWh |
| Cost         | currency     |

---

---

# Complete Parameter → Component Table

| Parameter       | Meter | Battery | Inverter | Load | Grid |
| --------------- | ----- | ------- | -------- | ---- | ---- |
| Voltage         | ✅     | ✅       | ✅        | –    | –    |
| Current         | ✅     | ✅       | ✅        | –    | –    |
| ActivePower     | ✅     | ✅      | ✅        | ✅    | ✅    |
| ReactivePower   | ✅     | –       | –        | –    | –    |
| EnergyImport    | ✅     | –       | –        | –    | ✅    |
| EnergyExport    | ✅     | –       | –        | –    | ✅    |
| SOC             | –     | ✅       | –        | –    | –    |
| ChargePower     | –     | ✅       | –        | –    | –    |
| DischargePower  | –     | ✅       | –        | –    | –    |
| EnergyGenerated | –     | –       | ✅        | –    | –    |
| Temperature     | –     | ✅       | –        | –    | –    |
| Demand/Peak     | ✅     | –       | –        | ✅    | –    |
| Cost            | –     | –       | –        | –    | ✅    |

---

---

# Final Practical Rule

### If it measures electricity → Meter parameters

### If it stores energy → Battery parameters

### If it converts energy → Inverter parameters

### If it consumes energy → Load parameters

### If it bills energy → Grid parameters

---

---

# For Your React Dashboard

Use this **universal format**:

```ts
{
  deviceId: "battery-1",
  component: "battery",
  parameter: "SOC",
  value: 76,
  unit: "%"
}
```

This directly maps to:

* charts
* React Flow nodes
* alerts
* analytics
* live monitoring

---

