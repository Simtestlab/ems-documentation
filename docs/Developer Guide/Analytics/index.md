# EMS Analytics

## Overview

The EMS Analytics system collects real-time telemetry every second from edge devices, automatically aggregates data for clean charts, and preserves all raw data forever through intelligent archival.

---

## 📚 **Documentation**

### **[Data Schema](./data-schema.md)**
What data is collected every second from the Raspberry Pi:
- Battery metrics (SOC, voltage, current, temperature)
- Grid metrics (power, voltage, frequency)
- Solar metrics (power, irradiance, efficiency)
- Load monitoring and energy economics
- Critical fields to add for production

### **[Aggregation Workflow](./aggregation-strategy.md)**
How data flows through multiple aggregation levels:
- 1-second → 1-minute → 5-minute → 1-hour → daily → weekly → monthly
- When each aggregation runs
- What happens to old data (archive vs delete)
- Storage architecture (hot vs cold database)

### **[Dashboard Integration](./dashboard-integration.md)**
How the Next.js dashboard displays clean charts:
- Smart query logic (automatic data source selection)
- Timeframe to data source mapping
- Dashboard page structure
- Real-time vs historical data handling

---

## 🎯 **Key Features**

✅ **Zero Data Loss** - All 1-second data archived to cold storage forever  
✅ **Clean Charts** - Auto-selects optimal data granularity for each timeframe  
✅ **Automatic** - Aggregation and cleanup runs on schedule, no manual work  
✅ **Fast Queries** - Dashboard queries pre-aggregated data for instant loading  
✅ **Cost-Effective** - 80% storage savings while preserving all data  

---

## 🔄 **Quick Overview**

### **Data Flow**
```
Raspberry Pi (1s) → Hot Database → Cold Archive
                         ↓
                  Auto-Aggregate
                         ↓
              1min → 5min → 1hour → daily
                         ↓
                  Dashboard Charts
```

### **Storage Strategy**
- **Hot DB:** Recent data (7 days raw, 30 days 1-min, 1 year hourly, forever daily)
- **Cold Archive:** All raw 1-second data forever (compressed backup)

### **Dashboard Logic**
- **User clicks timeframe** → System auto-selects best data source → Renders clean chart
- **Real-time (< 1min):** WebSocket live updates
- **Historical (> 1min):** REST API query
  
---

## 📊 **Timeframe Quick Reference**

| **User Selects** | **Data Source** | **Points** |
|-----------------|-----------------|------------|
| Last 10 seconds | Raw 1s | 10 |
| Last Hour | 1-min avg | 60 |
| Today | 5-min avg | 288 |
| Last 7 Days | 1-hour avg | 168 |
| This Year | Daily avg | 365 |

---

## 💾 **Storage Savings**

**Without aggregation:** 10 GB per device/year  
**With aggregation:** 1.8 GB per device/year  
**Savings:** 80%+ while keeping ALL data forever!

---

## 🚀 **Implementation**

**Technology Stack:**
- **Database:** TimescaleDB (time-series optimized)
- **Aggregation:** Continuous aggregates (automatic)
- **Backend:** FastAPI (query router)
- **Frontend:** Next.js (dashboard charts)
- **Real-time:** WebSocket (live updates)

All documentation is simplified and focused on architecture and workflows - ready to implement!