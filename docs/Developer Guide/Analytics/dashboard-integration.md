# Dashboard Analytics Integration

## Overview

The Next.js dashboard automatically queries the right dat a source based on the timeframe the user selects, ensuring clean charts with optimal performance.

---


## 📈 **Dashboard Page Structure**

### **Main Analytics Page**

```
┌────────────────────────────────────────────────────────┐
│                   EMS Analytics                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│  Timeframe: [10s][1m][10m][1h][6h][Today][7d][30d]...  │
│                                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │         Power Flow Chart (Line Graph)          │    │
│  │  ─── Battery │ ─── Grid │ ─── Solar │ ─── Load │    │
│  │                                                │    │
│  │  [Chart showing power over selected timeframe] │    │
│  │                                                │    │
│  └────────────────────────────────────────────────┘    │
│                                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │       Battery Status Chart (Line Graph)        │    │
│  │  ─── SOC % │ ─── Voltage │ ─── Temperature     │    │
│  │                                                │    │
│  │  [Chart showing battery metrics over time]     │    │
│  │                                                │    │
│  └────────────────────────────────────────────────┘    │
│                                                        │
│  ┌────────────────────────────────────────────────┐    │
│  │         System State Timeline                  │    │
│  │  Shows: IDLE → CHARGING → DISCHARGING pattern  │    │
│  │                                                │    │
│  │  [Timeline chart showing state changes]        │    │
│  │                                                │    │
│  └────────────────────────────────────────────────┘    │
│                                                        │
└────────────────────────────────────────────────────────┘
```

---

## 🔄 **Real-Time Updates**

### **For Live Monitor (Last 10s - 1min)**

Uses **WebSocket** connection instead of polling:

```
Dashboard Component
        ↓
Opens WebSocket Connection
        ↓
Receives new data every 1 second
        ↓
Appends to chart (keep last 60 points)
        ↓
Auto-scrolls chart showing latest
```

### **For Historical Views (> 1min)**

Uses **REST API** requests:

```
User clicks timeframe button
        ↓
Fetch data via API call
        ↓
Render complete chart
        ↓
No auto-refresh (historical data doesn't change)
```

---

## 🎨 **Chart Best Practices**

### **Data Points Per Chart**
- **< 100 points:** Perfect (every point visible)
- **100-500 points:** Good (smooth curves)
- **> 500 points:** Use coarser aggregation

### **Multiple Metrics on One Chart**
- Battery SOC (%)
- Grid Power (kW) - positive = import, negative = export
- Solar Power (kW)
- Total Load (kW)

### **Color Coding**
- 🔋 Battery: Blue
- ⚡ Grid: Green (import) / Orange (export)
- ☀️ Solar: Yellow
- 🏠 Load: Red

---

## 📱 **User Features**

### **Timeframe Selector**
Buttons for quick access:
- Real-time: 10s, 1min, 10min
- Recent: 1hr, 6hr, Today, Yesterday
- Long-term: 7d, 30d, 90d, Year

### **Chart Interactions**
- Hover tooltips showing exact values
- Zoom in/out on time range
- Toggle metric visibility (hide grid, show only solar, etc.)
- Export data as CSV

### **Live Indicator**
- Show "LIVE" badge when viewing real-time data
- Show "Historical" when viewing past data
- Display data source: "Showing 1-minute averages"

---

## ✅ **Implementation Summary**

**Backend:**
- Smart query router API endpoint
- Auto-selects database table based on timeframe
- Returns optimized data points

**Frontend:**
- Timeframe selector component
- Chart rendering with Recharts or Chart.js
- WebSocket for real-time updates
- REST API for historical queries

**Result:** Users get clean, fast charts that automatically show the right level of detail for any timeframe! 🎉
