# Data Aggregation Workflow – Full End-to-End Explanation (Crystal Clear)

This document explains the EMS data flow completely from start to finish,
including how many rows are used, when jobs run, and why.

This is written to remove all confusion.

---

# 1️⃣ Data Source – Where Everything Starts

Source: Raspberry Pi

Every second the device sends telemetry data:

Example:

09:00:00 → Battery: 52.5V  
09:00:01 → Battery: 52.6V  
09:00:02 → Battery: 52.4V  

One second = One record.

---

# 2️⃣ Raw Data Storage

Table: telemetry_raw  
Location: Hot Database

Every second:
- 1 new row inserted
- No calculations
- Just stored as-is

After:

1 minute → 60 rows  
5 minutes → 300 rows  
1 hour → 3600 rows  
1 day → 86400 rows  

Raw data is kept for 7 days only in Hot DB.

After 7 days:
- Copied to Cold Storage (archive)
- Deleted from Hot DB
- Never permanently lost

Hot DB always keeps only last 7 days.

---

# 3️⃣ 1-Minute Aggregation (Important Clarification)

## When does it run?

Every 5 minutes.

Example:
At 09:05:00

## What exists at 09:05?

From 09:00:00 → 09:04:59

There are:

5 minutes × 60 seconds = 300 raw rows

## Common Doubt

"If there are 300 rows, why does it only take 60 rows?"

Answer:

It does NOT take only 60 rows.

It processes all 300 rows.

But it groups them minute by minute.

---

## What Actually Happens At 09:05

System reads all rows between:

09:00:00 → 09:04:59

It then divides them into 5 time buckets:

Bucket 1:
09:00:00 → 09:00:59 (60 rows)

Bucket 2:
09:01:00 → 09:01:59 (60 rows)

Bucket 3:
09:02:00 → 09:02:59 (60 rows)

Bucket 4:
09:03:00 → 09:03:59 (60 rows)

Bucket 5:
09:04:00 → 09:04:59 (60 rows)

Each bucket produces one average.

So:

300 raw rows → 5 one-minute records

---

## Where is it stored?

Table: telemetry_1min

Example records created at 09:05:

09:00 → 52.51V  
09:01 → 52.49V  
09:02 → 52.50V  
09:03 → 52.48V  
09:04 → 52.52V  

---

## Why does the job run every 5 minutes instead of every minute?

Because:

- It ensures the full minute is complete
- Avoids partial minute calculations
- Reduces load on database
- Safer for distributed systems

It waits until 5 full minutes are complete,
then calculates all missing 1-minute buckets together.

Retention:
Kept 30 days → then deleted.

---

# 4️⃣ 5-Minute Aggregation

## When does it run?

Every 30 minutes.

Example:
At 09:30:00

## What does it read?

telemetry_1min table.

It reads:

09:00  
09:01  
09:02  
09:03  
09:04  

Then:

09:05–09:09  
09:10–09:14  
...  
Until 09:25–09:29  

Total:
30 minutes ÷ 5 = 6 five-minute buckets.

---

## What does it do?

Each group of 5 one-minute records
produces 1 five-minute average.

Example:

5 one-minute rows → 1 five-minute row

---

## Where stored?

Table: telemetry_5min

Retention:
Kept 90 days → then deleted.

Old records are never modified.

---

# 5️⃣ Hourly Aggregation

Runs every hour (XX:00).

Reads:
12 five-minute records.

Creates:
1 hourly record.

Stored in:
telemetry_1hour

Retention:
Kept 1 year → then deleted.

---

# 6️⃣ Daily Aggregation

Runs daily at 00:01 AM.

Reads:
24 hourly records.

Creates:
1 daily record.

Stored in:
telemetry_daily

Retention:
Forever.

Never deleted.

---

# 7️⃣ Weekly & Monthly

Weekly:
- Runs Monday 00:05
- Reads 7 daily records
- Stored in telemetry_weekly
- Kept forever

Monthly:
- Runs 1st of month 00:10
- Reads 28–31 daily records
- Stored in telemetry_monthly
- Kept forever

---

# 8️⃣ Cleanup Process

Every day automatic cleanup runs.

02:00 AM:
Raw rows older than 7 days → moved to Cold Storage → deleted from Hot DB.

03:00 AM:
Delete telemetry_1min older than 30 days.

03:30 AM:
Delete telemetry_5min older than 90 days.

04:00 AM:
Delete telemetry_1hour older than 1 year.

Daily / Weekly / Monthly are never deleted.

---

# 9️⃣ Complete Lifecycle Of One Data Point

Example:

Feb 10 – 09:00:00 → 52.5V

Immediately:
Stored in telemetry_raw.

09:05:
Used in 1-min average (09:00 bucket).

09:30:
Used in 5-min average (09:00–09:04 bucket).

10:00:
Used in hourly average.

00:01 next day:
Used in daily average.

After 7 days:
Raw version archived.

After 30 days:
1-min version deleted.

After 90 days:
5-min version deleted.

After 1 year:
Hourly version deleted.

Daily / Weekly / Monthly remain forever.

---

# 🔟 Full Data Flow Summary

Device
→ telemetry_raw
→ telemetry_1min
→ telemetry_5min
→ telemetry_1hour
→ telemetry_daily
→ telemetry_weekly
→ telemetry_monthly

Cleanup Flow:

telemetry_raw → Cold Storage  
telemetry_1min → Deleted after 30 days  
telemetry_5min → Deleted after 90 days  
telemetry_1hour → Deleted after 1 year  

---

# Final Understanding

300 raw rows in 5 minutes  
→ grouped into 5 separate 1-minute buckets  
→ not just first 60 rows  
→ all 300 rows are processed  

The system works using time buckets, not "last X rows".

This architecture ensures:

- Fast queries
- Controlled database size
- Long-term trend storage
- No data corruption
- No overlapping windows
- Safe archival of raw data
