# 📊 Live Graph Analytics — A-Z Implementation Guide

> **Goal:** Build a dashboard where the user **selects a time range** (1H, 6H, 24H, 7D, 30D) and **instantly sees a live chart** showing energy data for that exact period.

---

## 🧠 What Does This Dashboard Do?

One simple action drives everything:

```
  User clicks  [ 1H ]  [ 6H ]  [ 24H ]  [ 7D ]  [ 30D ]
                  |
                  v
        Chart shows data for THAT range
        KPI cards update for THAT range
        Donut chart updates for THAT range
```

**The entire analytics page revolves around this one choice — the selected time range.**

```
+-----------------------------------------------------+
|  Analytics Dashboard                                |
|                                                     |
|  [1H] [6H] [24H] [7D] [30D]    <- Pick a range     |
|  [*] Live data (WebSocket)      <- Data source      |
|                                                     |
|  +---------+  +---------+  +---------+              |
|  | Solar   |  |  Peak   |  |  Cost   |  <- KPI      |
|  | 42 kWh  |  |  12 kW  |  |  R1245  |    Cards     |
|  +---------+  +---------+  +---------+              |
|                                                     |
|  +-----------------------------------------------+  |
|  |  Power Chart (for selected range)              |  |
|  |  -- Solar  -- Grid  -- Battery  -- Load        |  |
|  |  [chart shows data from THAT time range]       |  |
|  +-----------------------------------------------+  |
|                                                     |
|  +---------------+  +---------------------------+   |
|  |  Energy Mix   |  |  Consumption Breakdown    |   |
|  |  (for range)  |  |  (for range)              |   |
|  +---------------+  +---------------------------+   |
+-----------------------------------------------------+
```

---

## 🏗️ Architecture — The Complete System

### How the Whole System is Connected

```
+------------+     +----------------+     +-------------------+     +----------------+
| SENSORS    |---->| CONTROLLER     |---->|  WEBSOCKET        |---->|  FRONTEND      |
| (Pi)       |     | (Python)       |     |  SERVER           |     |  (React)       |
|            |     |                |     |  (FastAPI)        |     |                |
| Reads      |     | Packages       |     | Streams live      |     | Stores data    |
| data       |     | telemetry      |     | data to browser   |     | in memory      |
| every 1s   |     | as JSON        |     |                   |     | Draws charts   |
+------------+     +-------+--------+     +-------------------+     +-------+--------+
                           |                                                |
                           v                                                |
                   +----------------+     +-------------------+             |
                   |  DATABASE      |---->|  BACKEND API      |-------------+
                   | (TimescaleDB)  |     |  (Django REST)    |
                   |                |     |                   |
                   | Stores all     |     | Serves pre-       |
                   | historical     |     | aggregated data   |
                   | data           |     | on request        |
                   +----------------+     +-------------------+
```

### The Two Data Paths

The dashboard gets data from **two different sources** based on the selected range:

```
                          User selects a range
                                 |
                    +------------+------------+
                    v                         v
             SHORT RANGE                LONG RANGE
             (1H or 6H)               (24H, 7D, 30D)
                    |                         |
                    v                         v
           +-----------------+      +------------------+
           |   WEBSOCKET     |      |    REST API      |
           |   (live feed)   |      |  (on-demand)     |
           +--------+--------+      +--------+---------+
                    |                         |
                    v                         v
           Ring Buffer in            Historical Cache
           browser memory            in browser memory
                    |                         |
                    +-----------+-------------+
                                v
                    +--------------------+
                    |   CHART RENDERS    |
                    |   the data         |
                    +--------------------+
```

> **💡 Why two paths?**
>
> - **WebSocket** = already streaming data every second, zero extra cost
> - **REST API** = server pre-summarizes 30 days into 720 hourly points instead of sending 2.6 million raw seconds

---

## 🎯 What Happens When the User Clicks Each Range

This is the **core workflow** — understand this and you understand the whole system:

### Clicking "1H" (Last 1 Hour)

```
Step 1: User clicks [1H]
Step 2: Store sets selectedRange = "1H"
Step 3: NO API call needed (data already in live buffer)
Step 4: Chart reads the last 3,600 points from the ring buffer
Step 5: Chart draws lines for the last 60 minutes
Step 6: KPIs computed from those 3,600 points
Step 7: Chart keeps auto-updating (new point every second)
```

**Data source:** Ring buffer (filled by WebSocket, always running)
**Points shown:** ~3,600 (1 per second × 60 minutes)
**Updates:** ✅ Yes — chart moves in real-time

---

### Clicking "6H" (Last 6 Hours)

```
Step 1: User clicks [6H]
Step 2: Store sets selectedRange = "6H"
Step 3: NO API call needed (data already in live buffer)
Step 4: Chart reads ALL points in the ring buffer (max 3,600)
Step 5: For the 5 hours before the buffer, shows most recent available data
Step 6: KPIs computed from all available points
Step 7: Chart keeps auto-updating (new point every second)
```

**Data source:** Ring buffer + optionally REST API for older 5 hours
**Points shown:** ~3,600 live + up to 300 historical (1 per minute for older hours)
**Updates:** ✅ Yes — newest hour updates live, older data is static

---

### Clicking "24H" (Last 24 Hours)

```
Step 1: User clicks [24H]
Step 2: Store sets selectedRange = "24H"
Step 3: isLoading = true (show spinner)
Step 4: Browser calls: GET /api/ems/charts?range=24H&site=ems_sim_01
Step 5: Server queries database for last 24 hours
Step 6: Server returns 1,440 pre-aggregated points (1 per minute)
Step 7: Store saves response in historicalData cache
Step 8: isLoading = false (hide spinner)
Step 9: Chart draws lines from the 1,440 points
Step 10: KPIs computed from those 1,440 points
```

**Data source:** REST API → database
**Points shown:** 1,440 (1 per minute × 24 hours)
**Updates:** ❌ No — static snapshot, user refreshes to get latest

---

### Clicking "7D" (Last 7 Days)

```
Step 1: User clicks [7D]
Step 2: Store sets selectedRange = "7D"
Step 3: isLoading = true (show spinner)
Step 4: Browser calls: GET /api/ems/charts?range=7D&site=ems_sim_01
Step 5: Server queries database for last 7 days
Step 6: Server returns 2,016 pre-aggregated points (1 per 5 minutes)
Step 7: Store saves response in historicalData cache
Step 8: isLoading = false (hide spinner)
Step 9: Chart draws lines from the 2,016 points
Step 10: KPIs computed from those 2,016 points
```

**Data source:** REST API → database (5-min averages)
**Points shown:** 2,016 (1 per 5 min × 7 days)
**Updates:** ❌ No — static snapshot

---

### Clicking "30D" (Last 30 Days)

```
Step 1: User clicks [30D]
Step 2: Store sets selectedRange = "30D"
Step 3: isLoading = true (show spinner)
Step 4: Browser calls: GET /api/ems/charts?range=30D&site=ems_sim_01
Step 5: Server queries database for last 30 days
Step 6: Server returns 720 pre-aggregated points (1 per hour)
Step 7: Store saves response in historicalData cache
Step 8: isLoading = false (hide spinner)
Step 9: Chart draws lines from the 720 points
Step 10: KPIs computed from those 720 points
```

**Data source:** REST API → database (hourly averages)
**Points shown:** 720 (1 per hour × 30 days)
**Updates:** ❌ No — static snapshot

---

### Summary Table: All Ranges at a Glance

| Range | Data Source | How Many Points | Resolution | Live Updates? | API Call? |
|:-----:|:----------:|:--------------:|:----------:|:------------:|:---------:|
| **1H** | WebSocket buffer | ~3,600 | 1 per second | ✅ Yes | ❌ No |
| **6H** | WebSocket + API | ~3,600 + ~300 | 1s (recent) + 1min (older) | ✅ Partial | ⚠️ Maybe |
| **24H** | REST API | 1,440 | 1 per minute | ❌ No | ✅ Yes |
| **7D** | REST API | 2,016 | 1 per 5 minutes | ❌ No | ✅ Yes |
| **30D** | REST API | 720 | 1 per hour | ❌ No | ✅ Yes |

> **💡 Why fewer points for longer ranges?** Showing 1 point per second for 30 days = 2.6 million points. That would crash the browser. So the server **pre-calculates averages** — same visual result, 3,600× less data.

---

## 🔩 Architecture Deep Dive — Component by Component

### Component 1: The Analytics Store

**File:** `src/store/analyticsStore.ts`

This is the **brain** of the analytics page. Everything routes through here.

```
+--------------------------------------------------------+
|                 analyticsStore                          |
|                                                        |
|  +---------------------------------------------------+ |
|  |  liveBuffer: { "ems_sim_01": [point, point...] }  | |  <- WebSocket data
|  |  Max 3,600 points per site (ring buffer)          | |
|  +---------------------------------------------------+ |
|                                                        |
|  +---------------------------------------------------+ |
|  |  historicalData: { "ems_sim_01": [point, ...] }   | |  <- REST API data
|  |  Cached per site, refreshed on range change       | |
|  +---------------------------------------------------+ |
|                                                        |
|  +---------------------------------------------------+ |
|  |  selectedRange: "1H" | "6H" | "24H" | "7D" | "30D"| |
|  +---------------------------------------------------+ |
|                                                        |
|  +---------------------------------------------------+ |
|  |  summary: { totalSolar, peakDemand, savings... }  | |  <- Auto-computed
|  +---------------------------------------------------+ |
|                                                        |
|  +---------------------------------------------------+ |
|  |  isLoading: true/false                            | |  <- Show/hide spinner
|  +---------------------------------------------------+ |
+--------------------------------------------------------+
```

**What each function does:**

| Function | Triggered By | What It Does |
|----------|-------------|-------------|
| `pushTelemetry(siteId, data)` | New WebSocket message | Appends to ring buffer, drops oldest if full |
| `setTimeRange(range)` | User clicks range button | Updates `selectedRange`, triggers fetch if needed |
| `fetchHistoricalData(siteId, range)` | `setTimeRange` (for 24H/7D/30D) | Calls REST API, stores result in `historicalData` |
| `getChartData(siteId)` | Chart component render | Returns `liveBuffer` (short) or `historicalData` (long) |
| `getSummary(siteId)` | KPI cards render | Returns computed KPI values for the current range |

---

### The Ring Buffer — Why and How

The ring buffer is a **fixed-size array** that holds live data:

```
Every second, WebSocket sends a new data point:

Second 1:    [A]
Second 2:    [A] [B]
Second 3:    [A] [B] [C]
...
Second 3600: [A] [B] [C] ... [3600]    ← Buffer is FULL (1 hour of data)

Second 3601: [B] [C] ... [3600] [3601]  ← "A" dropped, new point added
Second 3602: [C] ... [3600] [3601] [3602]← "B" dropped, new point added
```

**Why?** Without it:

| Time | Without buffer | With buffer (max 3,600) |
|------|:--------------:|:----------------------:|
| 1 hour | 3,600 points | 3,600 points |
| 1 day | 86,400 points 😰 | 3,600 points ✅ |
| 1 week | 604,800 points 💥 | 3,600 points ✅ |

---

### Component 2: The Telemetry Bridge

**File:** `src/modules/analytics/hooks/useAnalyticsTelemetry.ts`

This hook **connects the existing WebSocket to the analytics store**:

```
  telemetryStore          useAnalyticsTelemetry          analyticsStore
  (already exists)              (bridge)                   (new)
       |                          |                          |
       |  latestTelemetry ------->|                          |
       |                          |  "Is this new?" ------>  |
       |                          |  "Yes, push it" ------> | pushTelemetry()
       |                          |                          |
```

**How it works:**

1. Watches `telemetryStore.sites[siteId].latestTelemetry`
2. Compares the timestamp with the last one it processed
3. If different → calls `analyticsStore.pushTelemetry(siteId, data)`
4. Uses `useRef` to track the last timestamp (avoids duplicates)

> **💡 Why a bridge?** The WebSocket is already streaming data for the Live Dashboard page. This bridge simply **taps into that same stream** and copies data to the analytics store. No duplicate WebSocket connections.

---

### Component 3: Time Range Selector

**File:** `src/modules/analytics/components/TimeRangeSelector.tsx`

```
  +------+  +------+  +------+  +------+  +------+
  |  1H  |  |  6H  |  | 24H  |  |  7D  |  | 30D  |
  | #### |  |      |  |      |  |      |  |      |
  +------+  +------+  +------+  +------+  +------+
    active

  [*] Live data (WebSocket)
```

**Click behavior:**

| Clicked | store.setTimeRange() | API Call? | Data Source |
|:-------:|:-------------------:|:---------:|:-----------:|
| 1H | `"1H"` | ❌ No | Ring buffer |
| 6H | `"6H"` | ❌ No | Ring buffer |
| 24H | `"24H"` | ✅ Yes | REST API |
| 7D | `"7D"` | ✅ Yes | REST API |
| 30D | `"30D"` | ✅ Yes | REST API |

**Status indicator changes:**

- 🟢 `● Live data (WebSocket)` — shown for 1H, 6H
- 🔵 `● Historical data (API)` — shown for 24H, 7D, 30D

---

### Component 4: Power Flow Chart

**File:** `src/modules/analytics/components/PowerFlowChart.tsx`

This is the **main chart** — a time-series line chart with 4 overlaid lines:

| Line | Color | Meaning | Example |
|------|:-----:|---------|:-------:|
| ☀️ Solar | `#facc15` (Yellow) | Solar panel power output | 8.5 kW |
| ⚡ Grid | `#60a5fa` (Blue) | Grid import (+) or export (-) | 3.2 kW |
| 🔋 Battery | `#4ade80` (Green) | Discharge (+) or charge (-) | -2.1 kW |
| 🏠 Load | `#f87171` (Red) | Building consumption | 9.6 kW |

**How the X-axis adapts to the range:**

| Range | X-axis Format | Example Labels |
|:-----:|:------------:|:--------------:|
| 1H | `HH:MM:SS` | 10:30:15, 10:30:16, 10:30:17 |
| 6H | `HH:MM` | 04:30, 05:00, 05:30 |
| 24H | `HH:MM` | 00:00, 06:00, 12:00, 18:00 |
| 7D | `DD Mon` | 03 Feb, 04 Feb, 05 Feb |
| 30D | `DD Mon` | 12 Jan, 19 Jan, 26 Jan |

**Chart library options:**

| Library | Size | Recommendation |
|---------|:----:|:--------------:|
| Recharts | ~40 KB | ⭐ Best for beginners (React-native, easy API) |
| Chart.js | ~60 KB | Good for animations |
| HTML Canvas | 0 KB | Hard but fastest performance |

**Empty state:** `⏳ Waiting for data... Points collected: 0`

---

### Component 5: KPI Summary Cards

**File:** `src/modules/analytics/components/AnalyticsSummaryCards.tsx`

6 metric cards that **always reflect the selected range**:

| Card | Formula | Example |
|------|---------|:-------:|
| ☀️ Solar Generated | Sum of `solar_kw × time_interval` in kWh | 42.5 kWh |
| ⚡ Grid Import | Sum of positive `grid_kw × time_interval` | 28.3 kWh |
| 📈 Peak Demand | Max `load_kw` in the range | 12.3 kW |
| 🍃 Self-Consumption | `(Solar Used ÷ Total Load) × 100` | 62% |
| 💰 Cost Savings | `Solar kWh × ₹8/kWh` (tariff rate) | ₹1,245 |
| ⚖️ Avg Load | Mean of all `load_kw` in the range | 8.2 kW |

> **Key point:** When the user switches from 1H → 7D, these cards recalculate using the 7-day data, not the 1-hour data.

---

### Component 6: Energy Mix Donut Chart

**File:** `src/modules/analytics/components/EnergyMixChart.tsx`

Shows **where energy came from** in the selected period:

```
  +----------+     Solar   ######## 62%
  |          |     Grid    #####   28%
  |  Donut   |     Battery ##     10%
  +----------+
```

Rendered with pure SVG (`stroke-dasharray`) — no library needed.

---

### Component 7: The Assembled Page

**File:** `src/modules/analytics/pages/AnalyticsPage.tsx`

```
+--------------------------------------------------+
|  1. HEADER:  Site Selector + [ Time Range Btns ] |
+--------------------------------------------------+
|  2. KPI CARDS:  6 metric cards (for range)       |
+--------------------------------------------------+
|  3. MAIN CHART:  Power Flow (for range)          |
|     Loading spinner if fetching historical data  |
+----------------------+---------------------------+
|  4. DONUT CHART      |  5. More charts (future)  |
+----------------------+---------------------------+
```

This page also mounts `useAnalyticsTelemetry(siteId)` to start the WebSocket → analytics bridge.

---

## 📁 File Structure

```
src/
+-- store/
|   +-- telemetryStore.ts              <- EXISTS (WebSocket connection)
|   +-- analyticsStore.ts              <- NEW: ring buffer + cache + KPIs
|
+-- modules/analytics/
    +-- hooks/
    |   +-- useAnalyticsTelemetry.ts   <- NEW: bridges WebSocket -> store
    +-- components/
    |   +-- TimeRangeSelector.tsx       <- NEW: range toggle buttons
    |   +-- PowerFlowChart.tsx          <- NEW: main line chart
    |   +-- AnalyticsSummaryCards.tsx    <- NEW: 6 KPI cards
    |   +-- EnergyMixChart.tsx          <- NEW: donut chart
    +-- pages/
        +-- AnalyticsPage.tsx           <- MODIFY: assemble everything
```

---

## 🔌 Backend API — What's Needed

### Current State

The backend currently has:

- `GET /api/ems/charts` — returns last ~100 data points (in-memory, no range filter)
- `GET /api/ems/sites/{site_id}/charts` — same data, per site

### What Needs to Change

Add a `range` query parameter:

```
GET /api/ems/charts?range=24H&site=ems_sim_01
```

**Server-side logic per range:**

| Range | Query | Resolution | Points Returned |
|:-----:|-------|:----------:|:--------------:|
| 1H | Last 60 minutes | 1 per minute | 60 |
| 6H | Last 360 minutes | 1 per minute | 360 |
| 24H | Last 24 hours | 1 per minute | 1,440 |
| 7D | Last 7 days | 1 per 5 min | 2,016 |
| 30D | Last 30 days | 1 per hour | 720 |

**Response format:**

```json
{
  "range": "7D",
  "site": "ems_sim_01",
  "data": [
    {
      "timestamp": "2026-02-05T10:00:00Z",
      "solar_kw": 8.5,
      "grid_kw": 3.2,
      "battery_kw": -2.1,
      "load_kw": 9.6
    },
    {
      "timestamp": "2026-02-05T10:05:00Z",
      "solar_kw": 8.7,
      "grid_kw": 3.0,
      "battery_kw": -2.3,
      "load_kw": 9.4
    }
  ]
}
```

**Each data point:**

| Field | Type | From Telemetry |
|-------|:----:|----------------|
| `timestamp` | ISO string | `telemetry.timestamp` |
| `solar_kw` | number | `telemetry.solar.power_ac_kw` |
| `grid_kw` | number | `telemetry.grid.power_kw` |
| `battery_kw` | number | `telemetry.battery.power_kw` |
| `load_kw` | number | Computed: `solar + grid + battery` |

---

## 🔄 End-to-End Data Flow — Both Paths

### Path 1: Live Data Flow (1H / 6H)

```
 1. Raspberry Pi reads sensor
         |
         v
 2. Controller (Python) packages it as JSON
         |
         v
 3. WebSocket Server pushes to all connected browsers
         |
         v
 4. telemetryStore.ts receives it and updates latestTelemetry
         |
         v
 5. useAnalyticsTelemetry.ts detects the new timestamp
         |
         v
 6. analyticsStore.pushTelemetry() adds to ring buffer
    (drops oldest if buffer > 3,600 points)
         |
         v
 7. analyticsStore recomputes KPIs from buffer data
         |
         v
 8. PowerFlowChart.tsx re-renders with new point on the right
    KPI cards update with new values

 Total latency: < 100ms from sensor to screen
 Frequency: every 1 second
```

### Path 2: Historical Data Flow (24H / 7D / 30D)

```
 1. User clicks [7D] button
         |
         v
 2. TimeRangeSelector calls store.setTimeRange("7D")
         |
         v
 3. Store sets isLoading = true (spinner appears on chart)
         |
         v
 4. Store calls: GET /api/ems/charts?range=7D&site=ems_sim_01
         |
         v
 5. Backend queries database:
    SELECT AVG(solar_kw), AVG(grid_kw), ...
    FROM telemetry_5min
    WHERE timestamp > NOW() - INTERVAL '7 days'
    ORDER BY timestamp
         |
         v
 6. Server returns 2,016 pre-aggregated data points
         |
         v
 7. Store saves response in historicalData["ems_sim_01"]
    Store sets isLoading = false (spinner disappears)
         |
         v
 8. Store recomputes KPIs from the 2,016 points
         |
         v
 9. PowerFlowChart.tsx renders the 7-day chart
    KPI cards show 7-day totals
    Donut chart shows 7-day energy mix %
```

---

## 🧩 Step-by-Step Implementation Workflow

Build in this order — each step depends on the previous one:

### Step 1: Create the Analytics Store

Create `src/store/analyticsStore.ts` with:

- Ring buffer (array + max size 3,600)
- Historical data cache (object keyed by site ID)
- Selected range state (`"1H"` default)
- KPI summary computation
- `pushTelemetry()`, `setTimeRange()`, `fetchHistoricalData()`, `getChartData()`, `getSummary()`

### Step 2: Create the Telemetry Bridge Hook

Create `src/modules/analytics/hooks/useAnalyticsTelemetry.ts`:

- Subscribe to `telemetryStore.sites[siteId].latestTelemetry`
- On new timestamp → call `analyticsStore.pushTelemetry()`
- Track last timestamp with `useRef` to avoid duplicates

### Step 3: Build the Time Range Selector

Create `src/modules/analytics/components/TimeRangeSelector.tsx`:

- 5 buttons: 1H, 6H, 24H, 7D, 30D
- Highlight the active button
- On click → call `store.setTimeRange(range)`
- Show data source indicator (🟢 WebSocket / 🔵 API)

### Step 4: Build the Power Flow Chart

Create `src/modules/analytics/components/PowerFlowChart.tsx`:

- Read data from `store.getChartData(siteId)`
- Draw 4 lines: Solar (yellow), Grid (blue), Battery (green), Load (red)
- Auto-scale Y-axis
- Format X-axis based on range (HH:MM:SS for 1H, DD Mon for 30D)
- Show loading spinner when `store.isLoading` is true
- Show empty state when no data

### Step 5: Build KPI Summary Cards

Create `src/modules/analytics/components/AnalyticsSummaryCards.tsx`:

- Read from `store.getSummary(siteId)`
- Display 6 cards in responsive grid (2 cols mobile, 3 tablet, 6 desktop)
- Values auto-update for live ranges

### Step 6: Build the Energy Mix Donut Chart

Create `src/modules/analytics/components/EnergyMixChart.tsx`:

- Compute percentages from summary data
- Render SVG donut with 3 segments (Solar, Grid, Battery)

### Step 7: Assemble the Analytics Page

Modify `src/modules/analytics/pages/AnalyticsPage.tsx`:

- Mount `useAnalyticsTelemetry(activeSiteId)`
- Lay out all components top-to-bottom
- Connect everything to the analytics store

### Step 8: Update Backend API

Modify `backend/ems_api/services.py` and `views.py`:

- Add `range` query parameter to the charts endpoint
- Return pre-aggregated data at the right resolution per range
- In production: query TimescaleDB continuous aggregates

---

## 🚀 Production Considerations

| Area | What to Do |
|------|-----------|
| **Database** | Use TimescaleDB or InfluxDB for time-series queries instead of in-memory storage |
| **Aggregation** | Pre-compute 1-min, 5-min, hourly, daily averages via background jobs (Celery / cron) |
| **Caching** | Cache API responses in Redis (TTL = aggregation interval) |
| **Performance** | Pause ring buffer writes when analytics tab is hidden (Page Visibility API) |
| **Export** | Add CSV / PDF export for the selected time range |
| **Multi-site** | Allow comparing data from different sites on the same chart |
