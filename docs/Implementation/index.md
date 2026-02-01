# Initial Setup and Phase

![Phase-1-setup-Diagram](./phase-1-dashboard.png)

Our Energy Management System (EMS) utilizes a three-tier IoT architecture designed to decouple high-frequency sensor data from user Management.

The System Architecture is organized into three distince zones:

- **The Edge Layer (Ingestion)**: A Raspberry Pi polls the Inverter/Meter via Modbus RTU and pushes [telemetry](https://www.ibm.com/think/topics/telemetry) to the cloud every second via a persistent WebSocket connection.

- **The Cloud Layer (Processing)**: We utilize a 'Polyglot Persistence' Strategy. FastAPI acts as the high-perfomance hub, routing live data directly to **[TimescaleDB](https://medium.com/@pratiyush1/timescaledb-the-essential-guide-start-with-time-series-data-ce6423ff70c3)** (Time-series) for efficient storage. In parallel, [Django](https://docs.djangoproject.com/en/6.0/) manages user identifies in **SQLite** and queries TimescaleDB to generate historical performance reports.

- **The User Layer (Visualization)**: A Next.js frontend connects to both **backends** receiving live updates from **FastAPI** via **WebSockets** and fetching secure historical data from Django via REST APIs.

## Edge Layer

1. [State Machine Logic](./state-machine.md)

2. [Communication Interface](./communication-interface.md)