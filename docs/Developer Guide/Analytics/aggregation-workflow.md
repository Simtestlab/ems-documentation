# Data Aggregation Workflow

## Overview

Data flows from 1-second raw data through multiple aggregation levels, with automatic archival and cleanup happening continuously every day.

---

## 📅 **When Does Archival Happen?**

### **Raw 1-Second Data Archival**

This document shows exactly when aggregation happens, what data is used, what gets created, and what happens to old data.

---

## 📋 **Complete Aggregation & Retention Schedule**

| **What** | **When It Happens** | **Created From** | **Time Range** | **How Many Records Used** | **Keeps For** | **After Retention** |
|----------|-------------------|-----------------|---------------|------------------------|------------|-------------------|
| **Raw 1s Data** | Every second | Raspberry Pi telemetry | 1 second | 1 new record | 7 days in Hot DB | Moved to Cold Storage (kept forever) + Deleted from Hot DB |
| **1-min Average** | Every 5 minutes | Raw 1s data | 60 seconds (e.g., 09:00:00-09:00:59) | 60 records | 30 days | Deleted (no archive) |
| **5-min Average** | Every 30 minutes | 1-min averages | 5 minutes (e.g., 09:00:00-09:04:59) | 5 records | 90 days | Deleted (no archive) |
| **1-hour Average** | Every hour (XX:00) | 5-min averages | 1 hour (e.g., 09:00:00-09:59:59) | 12 records | 1 year | Deleted (no archive) |
| **Daily Average** | Daily at 00:01 AM | 1-hour averages | 24 hours (previous day) | 24 records | Forever | Never deleted ✅ |
| **Weekly Average** | Monday at 00:05 AM | Daily averages | 7 days (previous week) | 7 records | Forever | Never deleted ✅ |
| **Monthly Average** | 1st of month at 00:10 AM | Daily averages | 28-31 days (previous month) | 28-31 records | Forever | Never deleted ✅ |

---

## � **Understanding "How Many Records Used"**

This column shows **how many individual data points are averaged together** to create ONE average.

### **Example 1: Creating 1-Minute Average**

**Time:** 09:05:00 (Job runs every 5 minutes)  
**Task:** Create 1-minute averages for the last 5 minutes

**What happens:**
- Takes raw 1s data from **09:00:00 to 09:00:59** (60 records) → Creates **ONE** 1-minute average called "09:00 average"
- Takes raw 1s data from **09:01:00 to 09:01:59** (60 records) → Creates **ONE** 1-minute average called "09:01 average"
- Takes raw 1s data from **09:02:00 to 09:02:59** (60 records) → Creates **ONE** 1-minute average called "09:02 average"
- Takes raw 1s data from **09:03:00 to 09:03:59** (60 records) → Creates **ONE** 1-minute average called "09:03 average"
- Takes raw 1s data from **09:04:00 to 09:04:59** (60 records) → Creates **ONE** 1-minute average called "09:04 average"

**Result:** 5 new 1-minute averages created (one for each minute)

**Concrete Example:**
```
Raw data from 09:00:00 to 09:00:59:
09:00:00 → Battery: 52.1V
09:00:01 → Battery: 52.2V
09:00:02 → Battery: 52.0V
... (57 more records)
09:00:59 → Battery: 52.3V

Average = (52.1 + 52.2 + 52.0 + ... + 52.3) / 60 = 52.15V
Stored as: "09:00:00 average = 52.15V"
```

**Answer to Question 1:** It uses **specific time windows** (09:00:00-09:00:59), NOT just "last 60 records"

**Answer to Question 2:** It does **NOT add up**. Each minute gets its own separate average. They don't combine.

---

### **Example 2: Creating 5-Minute Average**

**Time:** 09:30:00 (Job runs every 30 minutes)  
**Task:** Create 5-minute averages for the last 30 minutes

**What happens:**
- Takes 1-min averages: **09:00, 09:01, 09:02, 09:03, 09:04** (5 records) → Creates **ONE** 5-minute average called "09:00-09:04 average"
- Takes 1-min averages: **09:05, 09:06, 09:07, 09:08, 09:09** (5 records) → Creates **ONE** 5-minute average called "09:05-09:09 average"
- ...continues for all 30 minutes...
- Takes 1-min averages: **09:25, 09:26, 09:27, 09:28, 09:29** (5 records) → Creates **ONE** 5-minute average called "09:25-09:29 average"

**Result:** 6 new 5-minute averages created (30 minutes / 5 = 6 averages)

**Concrete Example:**
```
1-minute averages:
09:00 average = 52.15V
09:01 average = 52.20V
09:02 average = 52.10V
09:03 average = 52.25V
09:04 average = 52.18V

5-minute average = (52.15 + 52.20 + 52.10 + 52.25 + 52.18) / 5 = 52.18V
Stored as: "09:00-09:04 average = 52.18V"
```

**Key Point:** Uses 5 **already-calculated** 1-minute averages, NOT 300 raw records

---

### **Example 3: Creating 1-Hour Average**

**Time:** 10:00:00 (Job runs every hour)  
**Task:** Create hourly average for 09:00-09:59

**What happens:**
- Takes 5-min averages: **09:00-09:04, 09:05-09:09, 09:10-09:14, ..., 09:55-09:59** (12 records) → Creates **ONE** hourly average

**Result:** 1 new hourly average for 09:00-09:59

**Concrete Example:**
```
5-minute averages:
09:00-09:04 = 52.18V
09:05-09:09 = 52.22V
09:10-09:14 = 52.15V
... (9 more)
09:55-09:59 = 52.20V

Hourly average = (52.18 + 52.22 + 52.15 + ... + 52.20) / 12 = 52.19V
Stored as: "09:00-09:59 average = 52.19V"
```

---

## 🔄 **Key Clarifications**

### **1. Which Records Are Used?**

✅ **Specific time windows** - NOT "last X records"

Examples:
- 09:00 1-min average uses **09:00:00 to 09:00:59** (exactly those 60 seconds)
- 09:01 1-min average uses **09:01:00 to 09:01:59** (different 60 seconds)
- They don't overlap or "add up"

### **2. Do Averages Add Up?**

❌ **NO** - They do NOT add up or accumulate values

Each time window gets its **own separate average**:
- 09:00 average = 52.15V
- 09:01 average = 52.20V
- These are stored as **2 separate records**, not combined

### **3. What Does "60 Records Used" Mean?**

It means: **60 individual data points are averaged together to create 1 average**

Example:
- Raw data: 60 separate voltage readings from 09:00:00 to 09:00:59
- Process: Add all 60 values, divide by 60
- Result: 1 average value representing that entire minute

### **4. How Often Are Averages Created?**

**Every 5 minutes** the system creates **5 new 1-minute averages** (one for each of the last 5 minutes):

| **Job Run Time** | **Averages Created** |
|-----------------|---------------------|
| 09:05:00 | 09:00, 09:01, 09:02, 09:03, 09:04 |
| 09:10:00 | 09:05, 09:06, 09:07, 09:08, 09:09 |
| 09:15:00 | 09:10, 09:11, 09:12, 09:13, 09:14 |

**Total after 1 hour:** 60 1-minute averages (one for each minute)

---

## 📊 **Step-by-Step Aggregation Process**

This section shows **exactly** how many rows are processed at each aggregation level.

### **Level 1: Raw Data Collection**

| **Time** | **Action** | **Rows in Table** | **Storage** |
|----------|-----------|------------------|------------|
| 09:00:00 | 1st data point stored | 1 | telemetry_raw |
| 09:00:01 | 2nd data point stored | 2 | telemetry_raw |
| 09:00:59 | 60th data point stored | 60 | telemetry_raw |
| 09:04:59 | After 5 minutes | 300 | telemetry_raw |
| 09:59:59 | After 1 hour | 3,600 | telemetry_raw |
| 23:59:59 | After 1 day | 86,400 | telemetry_raw |

**Key Point:** Every second = 1 new row added to Hot DB

---

### **Level 2: Creating 1-Minute Averages**

| **Job Run Time** | **Reads From** | **Time Buckets Processed** | **Rows Read** | **Rows Created** | **Calculation** |
|-----------------|---------------|---------------------------|--------------|-----------------|----------------|
| 09:05:00 | telemetry_raw | 09:00:00-09:00:59<br/>09:01:00-09:01:59<br/>09:02:00-09:02:59<br/>09:03:00-09:03:59<br/>09:04:00-09:04:59 | 300 raw rows | 5 1-min averages | Bucket 1: 60 rows → 1 avg<br/>Bucket 2: 60 rows → 1 avg<br/>Bucket 3: 60 rows → 1 avg<br/>Bucket 4: 60 rows → 1 avg<br/>Bucket 5: 60 rows → 1 avg |
| 09:10:00 | telemetry_raw | 09:05:00-09:09:59 (5 buckets) | 300 raw rows | 5 1-min averages | Same pattern |
| 09:15:00 | telemetry_raw | 09:10:00-09:14:59 (5 buckets) | 300 raw rows | 5 1-min averages | Same pattern |

**Result after 1 hour:** 300 raw rows per 5-min job × 12 jobs = 3,600 raw rows processed → 60 1-min averages created

**Key Point:** It processes ALL 300 rows, grouped into 5 separate time buckets (not just first 60 rows)

---

### **Level 3: Creating 5-Minute Averages**

| **Job Run Time** | **Reads From** | **Time Buckets Processed** | **Rows Read** | **Rows Created** | **Calculation** |
|-----------------|---------------|---------------------------|--------------|-----------------|----------------|
| 09:30:00 | telemetry_1min | 09:00-09:04<br/>09:05-09:09<br/>09:10-09:14<br/>09:15-09:19<br/>09:20-09:24<br/>09:25-09:29 | 30 1-min rows | 6 5-min averages | Bucket 1: 5 1-min rows → 1 avg<br/>Bucket 2: 5 1-min rows → 1 avg<br/>...<br/>Bucket 6: 5 1-min rows → 1 avg |
| 10:00:00 | telemetry_1min | 09:30-09:59 (6 buckets) | 30 1-min rows | 6 5-min averages | Same pattern |

**Key Point:** Uses already-calculated 1-minute averages, NOT raw data

---

### **Level 4: Creating Hourly Averages**

| **Job Run Time** | **Reads From** | **Time Buckets Processed** | **Rows Read** | **Rows Created** | **Calculation** |
|-----------------|---------------|---------------------------|--------------|-----------------|----------------|
| 10:00:00 | telemetry_5min | 09:00-09:59 (split into 12 buckets) | 12 5-min rows | 1 hourly average | All 12 5-min rows → 1 hourly avg |
| 11:00:00 | telemetry_5min | 10:00-10:59 | 12 5-min rows | 1 hourly average | Same pattern |

**Key Point:** 1 hour = 12 five-minute periods

---

### **Level 5: Creating Daily Averages**

| **Job Run Time** | **Reads From** | **Time Range** | **Rows Read** | **Rows Created** |
|-----------------|---------------|---------------|--------------|-----------------|
| Feb 11, 00:01 AM | telemetry_1hour | Feb 10, 00:00-23:59 | 24 hourly rows | 1 daily average |
| Feb 12, 00:01 AM | telemetry_1hour | Feb 11, 00:00-23:59 | 24 hourly rows | 1 daily average |

**Key Point:** 1 day = 24 hourly periods

---

## 🔢 **Why "300 Rows → 5 Averages"**

**Common Confusion:** "If there are 300 rows, why only 60 records used?"

**Answer:** ALL 300 rows ARE used, but they're **grouped into 5 time buckets**.

### **Detailed Breakdown at 09:05:00**

| **Bucket** | **Time Range** | **Rows in Bucket** | **Action** | **Result** |
|-----------|---------------|-------------------|-----------|-----------|
| 1 | 09:00:00 - 09:00:59 | 60 rows | Average all 60 → One value | 09:00 avg = 52.15V |
| 2 | 09:01:00 - 09:01:59 | 60 rows | Average all 60 → One value | 09:01 avg = 52.20V |
| 3 | 09:02:00 - 09:02:59 | 60 rows | Average all 60 → One value | 09:02 avg = 52.10V |
| 4 | 09:03:00 - 09:03:59 | 60 rows | Average all 60 → One value | 09:03 avg = 52.25V |
| 5 | 09:04:00 - 09:04:59 | 60 rows | Average all 60 → One value | 09:04 avg = 52.18V |
| **TOTAL** | **5 minutes** | **300 rows** | **5 separate calculations** | **5 1-min averages** |

**Visual Example:**

```
Raw Data (300 rows):
09:00:00 → 52.1V  ┐
09:00:01 → 52.2V  │
...               ├─ Bucket 1 (60 rows) → Average = 52.15V
09:00:58 → 52.1V  │
09:00:59 → 52.2V  ┘

09:01:00 → 52.3V  ┐
09:01:01 → 52.2V  │
...               ├─ Bucket 2 (60 rows) → Average = 52.20V
09:01:58 → 52.1V  │
09:01:59 → 52.3V  ┘

... (3 more buckets)

Result: 5 one-minute averages stored in telemetry_1min
```

---

## 📅 **Complete Row Count Example: February 10**

| **Data Level** | **Rows Created on Feb 10** | **Total Rows in Table** | **Notes** |
|---------------|---------------------------|------------------------|----------|
| Raw 1s | 86,400 new rows | ~605,000 rows | 7 days × 86,400/day |
| 1-min avg | 1,440 new rows | ~43,200 rows | 30 days × 1,440/day |
| 5-min avg | 288 new rows | ~25,920 rows | 90 days × 288/day |
| 1-hour avg | 24 new rows | ~8,760 rows | 365 days × 24/day |
| Daily avg | 1 new row | 365+ rows | All days since system start |
| Weekly avg | 1 new row (Mon only) | 52+ rows | All weeks since system start |
| Monthly avg | 1 new row (1st only) | 12+ rows | All months since system start |

**After Feb 18 (8 days later):**
- Feb 10 raw data (86,400 rows) → Moved to Cold Storage
- Feb 10 raw data → Deleted from Hot DB
- All averages still present

---

---

## 🗑️ **Cleanup & Archival Schedule**

| **What Gets Cleaned** | **When** | **Age Limit** | **Action** | **Example** |
|---------------------|---------|------------|-----------|-----------|
| Raw 1s Data | Daily at 02:00 AM | 8 days old | Archive to Cold Storage → Delete from Hot DB | Feb 10 at 2 AM: Feb 2 data archived & deleted |
| 1-min Averages | Daily at 03:00 AM | 31 days old | Delete (no archive) | Feb 10 at 3 AM: Jan 10 1-min data deleted |
| 5-min Averages | Daily at 03:30 AM | 91 days old | Delete (no archive) | Feb 10 at 3:30 AM: Nov 11 5-min data deleted |
| 1-hour Averages | Daily at 04:00 AM | 1 year old | Delete (no archive) | Feb 10, 2026 at 4 AM: Feb 9, 2025 hourly data deleted |
| Daily Averages | Never | N/A | Never deleted ✅ | All daily data kept forever |
| Weekly Averages | Never | N/A | Never deleted ✅ | All weekly data kept forever |
| Monthly Averages | Never | N/A | Never deleted ✅ | All monthly data kept forever |

---

## 📅 **Example: What Happens on February 10, 2026**

### **Throughout the Day (Every Second, Minute, Hour)**

| **Time** | **What Happens** | **Details** |
|---------|-----------------|-----------|
| 00:00:00 | New raw data arrives | Battery: 52.5V, Grid: 230V (stored in Hot DB) |
| 00:00:01 | New raw data arrives | Battery: 52.6V, Grid: 229V |
| 00:01:00 | **Daily average created** | Feb 9 daily average created from 24 hourly averages (00:00-23:00) |
| 00:05:00 | **1-min averages created** | Averages for 00:00, 00:01, 00:02, 00:03, 00:04 created from raw data |
| 00:05:00 (Mon) | **Weekly average created** | Feb 3-9 weekly average created (only runs on Monday) |
| 00:10:00 | **1-min averages created** | Averages for 00:05, 00:06, 00:07, 00:08, 00:09 created |
| 00:10:00 (1st) | **Monthly average created** | January monthly average created (only runs 1st of month) |
| 00:30:00 | **5-min averages created** | Averages for 00:00-00:04, 00:05-00:09, ..., 00:25-00:29 created |
| 01:00:00 | **1-hour average created** | Average for 00:00-00:59 created from 12 five-min averages |
| 02:00:00 | **🔴 ARCHIVAL** | Feb 2 raw data (8 days old) → Moved to Cold Storage → Deleted from Hot DB |
| 03:00:00 | **Cleanup 1-min** | Jan 10 1-min averages (31 days old) deleted |
| 03:30:00 | **Cleanup 5-min** | Nov 11, 2025 5-min averages (91 days old) deleted |
| 04:00:00 | **Cleanup hourly** | Feb 9, 2025 hourly averages (1 year old) deleted |
| 09:00:00 | **1-hour average created** | Average for 08:00-08:59 created |
| 09:05:00 | **1-min averages created** | Averages for 09:00, 09:01, 09:02, 09:03, 09:04 created |
| ... | Continue all day | Aggregation and data collection continues 24/7 |

---

## � **Real-World Example: Following One Data Point**

**February 10, 09:00:00** - Battery voltage: 52.5V

| **When** | **What Happens to This Data** | **Where It Is** |
|---------|------------------------------|----------------|
| Feb 10, 09:00:00 | Raw data stored | Hot DB `telemetry_raw` table |
| Feb 10, 09:05:00 | Included in 1-min average (09:00:00-09:00:59) | Hot DB `telemetry_1min` table |
| Feb 10, 09:30:00 | Included in 5-min average (09:00:00-09:04:59) | Hot DB `telemetry_5min` table |
| Feb 10, 10:00:00 | Included in 1-hour average (09:00:00-09:59:59) | Hot DB `telemetry_1hour` table |
| Feb 11, 00:01:00 | Included in daily average (Feb 10) | Hot DB `telemetry_daily` table |
| Feb 17, 00:05:00 | Included in weekly average (Feb 10-16) | Hot DB `telemetry_weekly` table |
| Feb 18, 02:00:00 | **Raw data archived** | Moved to Cold Storage `archive_raw_1s` |
| Feb 18, 02:05:00 | **Raw data deleted from Hot DB** | Only in Cold Storage now |
| Mar 12, 03:00:00 | **1-min average deleted** | Still in 5-min, hourly, daily |
| May 11, 03:30:00 | **5-min average deleted** | Still in hourly, daily |
| Mar 1, 00:10:00 | Included in monthly average (February) | Hot DB `telemetry_monthly` table |
| Feb 10, 2027, 04:00:00 | **Hourly average deleted** | Still in daily, weekly, monthly |
| Forever | **Daily/weekly/monthly averages kept** | Hot DB forever ✅ |

---

## ✅ **Key Takeaways**

**1. Archival Happens Every Day**
- Not just once
- Daily at 2 AM
- Oldest 8-day-old data gets moved to Cold Storage
- Process continues forever automatically

**2. Averages Accumulate (Don't Replace)**
- New averages are added every interval
- Old averages stay until retention period ends
- Example: Day 1 = 1,440 1-min averages, Day 2 = 2,880 total

**3. Raw Data Never Lost**
- Kept 7 days in Hot DB (fast access)
- Then moved to Cold Storage (kept forever)
- Can always retrieve for investigation

**4. Higher-Level Averages Kept Forever**
- Daily, weekly, monthly never deleted
- Perfect for long-term trends and reports

**Result:** Fast queries + Small database + Complete data preservation! 🚀
