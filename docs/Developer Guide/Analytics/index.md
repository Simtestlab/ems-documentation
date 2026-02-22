# EMS Analytics

## Overview

The EMS Analytics system collects real-time telemetry every second from edge devices, automatically aggregates data for clean charts, and preserves all raw data forever through intelligent archival.

---

## 📊 Quick Overview: Analytics Page Implementation

The analytics dashboard lets users **pick a time range** and instantly see energy charts, KPI cards, and energy mix data for that period.

### Dashboard Layout

```
+-----------------------------------------------------+
|  [1H]  [6H]  [24H]  [7D]  [30D]   <- Pick a range  |
|                                                     |
|  +---------+  +---------+  +---------+              |
|  | Solar   |  |  Peak   |  |  Cost   |  <- KPI      |
|  | 42 kWh  |  |  12 kW  |  |  R1245  |    Cards     |
|  +---------+  +---------+  +---------+              |
|                                                     |
|  +-----------------------------------------------+  |
|  |  Power Chart (Solar, Grid, Battery, Load)      |  |
|  +-----------------------------------------------+  |
|                                                     |
|  +---------------+  +---------------------------+   |
|  |  Energy Mix   |  |  Consumption Breakdown    |   |
|  +---------------+  +---------------------------+   |
+-----------------------------------------------------+
```

### Two Data Paths

| Path | Ranges | Source | Updates? |
|:----:|:------:|:------:|:--------:|
| **Live** | 1H, 6H | WebSocket ring buffer | ✅ Real-time |
| **Historical** | 24H, 7D, 30D | REST API → Database | ❌ Static |

### Timeframe → Data Source

| User Selects | Data Source | Points | Resolution |
|:------------:|:----------:|:------:|:----------:|
| 1H | WebSocket buffer | ~3,600 | 1/second |
| 6H | Buffer + API | ~3,900 | 1s + 1min |
| 24H | REST API | 1,440 | 1/minute |
| 7D | REST API | 2,016 | 1/5 minutes |
| 30D | REST API | 720 | 1/hour |

### Tech Stack

- **Database:** TimescaleDB
- **Backend:** FastAPI (WebSocket) + Django REST (historical API)
- **Frontend:** Next.js + React (Recharts for charts)
- **Real-time:** WebSocket → Ring Buffer (max 3,600 points)

👉 **Full details:** [Dashboard Implementation Guide](./dashboard-implementation.md)

---

## 📦 Quick Overview: Data Schema

Every second, the Raspberry Pi sends a JSON telemetry payload with these categories:

| Category | Key Fields | Example Values |
|:--------:|:----------:|:--------------:|
| 🔋 **Battery** | SOC, voltage, current, power, temperature, cycle count | 85%, 380V, 12A, 4.68kW, 28°C |
| ⚡ **Grid** | frequency, voltage, power, price, status | 50Hz, 230V, 1.96kW, connected |
| ☀️ **Solar** | AC power, DC power, irradiance, panel temp, efficiency | 6.5kW, 850 W/m², 45°C, 95.6% |
| 🔄 **Inverter** | priority, action, reason | solar_first, charging_battery |
| 🏠 **Load** | total load, critical load, peak today | 1.82kW, 0.8kW, 5.2kW |
| 🖥️ **System** | state, mode, uptime, CPU, memory | CHARGING, AUTO, 10 days |
| 💰 **Economics** | tariff, cost today, savings today | ₹0.15/kWh, ₹1.88, ₹4.28 |

**Payload size:** ~500–800 bytes per second (very efficient for 1Hz)

### Critical Fields for Production

- **Load monitoring** — essential for optimization decisions
- **Cell-level battery data** — safety and longevity tracking
- **Energy economics** — user ROI and financial visibility
- **Power quality metrics** — grid compliance and efficiency

👉 **Full schema with all fields:** [Telemetry Schema](./telemetry-schema.md)

---

## 🔄 Quick Overview: Data Aggregation

Raw 1-second data flows through multiple aggregation levels automatically:

### Aggregation Pipeline

```
Raspberry Pi (1s)
  |
  +-> telemetry_raw       [every second]
  +-> telemetry_1min      [job runs every 5 min]
  +-> telemetry_5min      [job runs every 30 min]
  +-> telemetry_1hour     [job runs every hour]
  +-> telemetry_daily     [job runs at 00:01 AM]
  +-> telemetry_weekly    [Monday 00:05 AM]
  +-> telemetry_monthly   [1st of month 00:10 AM]
```

### Retention & Cleanup

| Data Level | Kept For | After Retention |
|:----------:|:--------:|:---------------:|
| Raw 1s | 7 days | Archived to Cold Storage → deleted from Hot DB |
| 1-min avg | 30 days | Deleted |
| 5-min avg | 90 days | Deleted |
| Hourly avg | 1 year | Deleted |
| Daily avg | **Forever** | Never deleted ✅ |
| Weekly avg | **Forever** | Never deleted ✅ |
| Monthly avg | **Forever** | Never deleted ✅ |

### Daily Cleanup Schedule

| Time | Action |
|:----:|:------:|
| 02:00 AM | Raw data (>7 days) → archive + delete |
| 03:00 AM | 1-min averages (>30 days) → delete |
| 03:30 AM | 5-min averages (>90 days) → delete |
| 04:00 AM | Hourly averages (>1 year) → delete |

### Storage Savings

**Without aggregation:** 10 GB per device/year  
**With aggregation:** 1.8 GB per device/year  
**Savings:** 80%+ while keeping ALL data forever!

👉 **Detailed workflow:** [Aggregation Overview](./aggregation-overview.md) · [Aggregation Workflow](./aggregation-workflow.md)

---

## 📚 Documentation

| Document | Description |
|:--------:|:------------|
| [Telemetry Schema](./telemetry-schema.md) | Complete JSON structure for all sensor data |
| [Aggregation Overview](./aggregation-overview.md) | End-to-end aggregation explanation |
| [Aggregation Workflow](./aggregation-workflow.md) | Detailed step-by-step with row counts and examples |
| [Dashboard Implementation](./dashboard-implementation.md) | A-Z guide for building the analytics dashboard |

---

## 🎯 Key Features

✅ **Zero Data Loss** — All 1-second data archived to cold storage forever  
✅ **Clean Charts** — Auto-selects optimal data granularity for each timeframe  
✅ **Automatic** — Aggregation and cleanup runs on schedule, no manual work  
✅ **Fast Queries** — Dashboard queries pre-aggregated data for instant loading  
✅ **Cost-Effective** — 80% storage savings while preserving all data  