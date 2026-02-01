# Role of FastAPI

The FastAPI server acts as the **high-concurrency gateway** for the entire system.

Unlike a traditional web server that waits for a request to send a response (HTTP), this server maintains **persistent**, **open connections** (WebSockets) with every active device and user.

## Implementation Strategy
To manage these connections effectively, the server implements a **Connection Manager**.

1. **Device Registry**
    - The server keeps an *in-memory* dictionary mapping *device_id* to *websocket_connection*.

    - Why? When a user sends a command to "Device-001", the server instantly knows which open socket belongs to that device.

2. **Broadcast Loop**
    - The server does not just store data, it acts as a mirror.

    - When Telemetry arrives from the Raspberry Pi, the server immediately "broadcasts" a copy of that JSON packet to any frontend clients (React) currently viewing that dashboard.

    - **Result**: The user sees the voltage change on their screen in <100ms, often before the data is even written to the cloud.

3. **Concurrency (AsyncIO)**
    - The entire server runs on an **Asynchronous Event Loop**.

    - This is critical: Waiting for a slow database write must never block the "Heartbeat" of the WebSocket.

## Telemetry Storage (TimescaleDB Integration)

While FastAPI handles the now, TimescaleDB handles the history. We do not use a standard SQL table structure instead, we use **Hypertables**.

## Schema Design

Standard databases (like standard PostgresSQL or MySQL) get slower as they grow to millions of rows. TimescaleDB solves this by partitioning data by time.

- **Concept**: To the application, it looks like one giant table called *energy_readings*.

- **Reality**: Under the hood, TimescaleDB automatically splits this into hundreds of smaller "chunks" (e.g,, one chunk per week).

- **Benefit**: Queries like "Get last 24 hours" only open the specific "Today" chunk, making them instant even after years of logging.

### Ingestion Pipeline
To prevent the high-speed data stream from crashing the database during peak loads, we implement a **Buffered Write Strategy**.

1. Validation
    - Incoming JSON is validated against a strict Schema (Pydantic Model) to ensure no corrupt data enters the DB.

2. Insert
    - We should use *asyncpg* (a high-perfomance database driver) to perform non-blocking inserts.

    - Optimization: For production scaling, we can use "Batch Inserts" (grouping 10 seconds of data into one write).

## Data Flow Summary (Life of a packet)

1. **Arrival**: Packet *{"voltage": 53.2}* arrives at *wss://api...*

2. **FastAPI (Hub)**
    - **Action A (Hot path)**: Immediately forwards the packet to the React Frontend.

    - **Action B (Cold path)**: Validates the data and spawns a background task to save it.

3. **TimescaleDB (Storage)**:
    - Receives the data.

    - Routes it to the correct "Time Chunk" (Hypertable partition).

    - Updates any continious aggregates (e.g., auto-calculating "Daily Total").