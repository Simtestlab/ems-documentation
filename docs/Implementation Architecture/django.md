# Cloud Logic: History & Management using Django

## Role of Django

While FastAPI handles the high-speed "firehose" of raw data, Django acts as the secure **Management Layer**. It is responsible for two critical things:

1. Who are you? (Authentication & Authorization)

2. What can you see? (Data Retrieval & Aggregation)

## Implementation Strategy

Django is configured to talk to two different databases simultaneously, but strictly seperates their purpose:

### Database A (SQLite - Default)
- Stores: User accounts, passwords (hashed), profiles and device ownership mapping.
- SQLite is lighweight and perfect for relational data that doesn't change 100 times a second (For Telemetry data we use TimescaleDB.

**Note**: Refer [here](./fastAPI.md) for FastAPI and how it handles data every second).

### Database B (TimescaleDB - telemetry)
- Stores: The raw energy reading inserted by FastAPI.

- Read-Only Mode: Django treat this database as "Read-Only". It never writes to it; it only querires it to generate reports.

## 2. History Retrieval Strategy

A common mistake in IoT dashboards is trying to load every single data point (e.g., 86,400 points for a single day) into the browser. This crashes the user's browser.

To resolve it efficiently, we implement a Smart Downsampling Strategy in Django:

1. Raw View (Zoomed In):
    - **Request**: "Last 10 minutes"
    - **Action**: Django fetches raw rows from TimescaleDB.
    - **Resolution**: 1-second intervals.

2. Hourly View (Normal):
    - **Request**: "Last 24 hours."
    - **Action**: Django uses TimescaleDB's *time_bucket('5 minutes')* function.
    - **Resolution**: It averages every 300 data points into 1 point.

3. Monthly view (Zoomed Out):
    - **Request**: "Last 30 days".
    - **Action**: Django uses *time_bucket('1 hour')*.
    - **Resolution**: Returns hourly averages.

**Result**: No matter the time range, the Frontend always receives ~500 clean data points, keeping the dashboard fast.

## 3. Securing & Permissions ("Row-level" Security)

Since the *telemetry* database holds data for all users mixed together, we cannot just "select all".

1. The Interceptor:
- When a user requests */api/history/device_001*, Django first checks the SQLite database: "Does the current logged-in user own device_001?"

2. The Filter:
- If yes, it constructs a specific SQL query for TimescaleDB: *SELECT * FROM energy_data WHERE device_id = 'device_001'*

- If no, it returns *403 Forbidden* immediately, never touching the metrics database.

## 4. Architecture Summary

- FastAPI writes the data (Write-Heavy)

- Django reads and aggregates the data (Read-Heavy).

- TimescaleDB sits in the middle, optimized to handle both loads simultaneously.