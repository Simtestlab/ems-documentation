# WebSocket Architecture & Implementation Guide

## Overview

The WebSocket layer acts as the "Nervous System" of the EMS, facilitating real-time bidirectional communication between the **Edge (Raspberry Pi), Cloud (FastAPI)**, and **User (Dashboard)**.

- **Protocol**: *wss://* (Secure WebSocket)
- **Topology**: Pub/Sub (Publisher/Subscriber)
- **Latency Goal**: *<100ms* from Edge to Dashboard.

## System Topology
The system follows a **publish-subscribe** pattern centered around a centralized "Hub" (Cloud Server).

- **Publisher (Edge/Pi)**
Pushes data up to the cloud. It doesn not know or care who is watching.

- **Hub (FastAPI)**
Acts as the "Switchboard". It receives data, validates it, and routes it to the correct recipients.

- **Subscriber (Dashboard)**
Listens for updates. It subscribes to a specific "Topic" (Device ID) to receive real-time feeds.

## Connection Lifecycle Logic
Every connection must pass through a strict lifecycle to ensure security and stability.

### Handshake (Initiation)
1. **Request**: The Client (Pi) initiates a standard HTTP GET request with an "Upgrade" header to switch to the WebSocket protocol.

2. **Identity Assertion**: The Client must include the *device_id* in the URL path (e.g., */ws/device/ems_001*) to declare which device stream it wants to access.

3. **Credential Check**: The client must provide an **Authentication Token** in the HTTP Headers. URL parameters are strictly forbidden for credentials.

## Connection Manager (Routing)
Once connected, the server must track who is who. It uses an **In-Memory Registry**.

- **Registry**: A dynamic Dictionary mapping *Device_ID* --> *Active_Socket_Object*.

- **Concurrency Rule**: The Manager enforces a "**One-Device-One-Connection**" policy.

## Authentication & Security

We utilize **Token-Based Authentication**. Devices and Users must prove their identity via HTTP Headers.

### Golden Rules

1. No Tokens in URLs:
Never use *ws://api.com/ws/device/ems_001?token=123. This leaks credentials in server logs.

2. Headers Only:
Credentials pass via the *Authorization* Header.

3. Strict Whitelisting:
The *device_id* in the URL must match the Owner of the Token.

### Handshake Flow

1. **Client**: Sends GET */ws/device/{device_id}* + Header *Authorization: Bearer < SECRET_TOKEN >*.

2. **Server**: Intercepts request -> Hashes Token -> Checks DB for match.

3. **Result**: *101 Switching Protocols* (Success) or *403 Forbidden* (fail).
    - **Logic**: If *ems_001* tries to connect, but the Registry shows *ems_001* is already connected, the Server immediately disconnects the old connection and accpets the new one. This resolves "Ghost Connections" (stuck sockets).

### Data Flow Architecture

#### Inbound Stream (Edge --> Cloud)

1. Payload Receipt: The server receives a raw JSON packet.

2. The "Gatekeeper" (Schema Validation)
    - The Server attempts to map the raw JSON to a strict **Data Model** (e.g., checking that *voltage* is a float, not text).

    - Pass: The data proceeds to the next step.

    - Fail: The data is discarded, and an error is logged. The socket remains open.

3. Async Forking (Dual Write)

    - Path 1 (Hot Path): The validated data is immediately passed to the **Connection Manager** for broadcasting.

    - Path 2 (Cold Path): A background task is spawned to write the data to **TimescaleDB** (ensuring database latency doesn't slow down the live stream).

### Outbound Broadcast (Cloud --> UI)
1. **Lookup**: The Connection Manager checks if there are any active UI clients subscribed to this specific *device_id*.

2. **Forwarding**: If subscribers exist, the JSON packet is serialized and sent down their open sockets.

3. **Efficiency**: If no UI users are watching *ems_001*, the broadcast step is skipped (saving bandwidth), but the Database Write (Path 2) still occurs.

### Reliability & Resilence Logic

Network devices (routers, load balancers) silently kill idle connections. We implement a **Hearbeat Strategy**.

- **Server Role**: Sends a hidden *PING* frame every 20 seconds.

- **Client Role**: Must respond with a *PONG* frame automatically.

- **Failure Logic**: If the client misses 3 consecutive pings (60 seconds), the Server declares the connection "Dead", removes it from the Registry, and closes the socket resources.

### Infinite Reconnect Loop (Client Side)

The Edge device logic is stateless and persistent. It does not "give up" on errors.

- **State**: The client has two states: **CONNECTED** or **RETRYING**.

- **Logic**:
    1. Attempt Connection.
    2. If successful, enter **Stream Loop**.
    3. If **ConnectionClosed** or **NetworkError** occurs:
        - Log Error.
        - Wait for **Backoff Interval** (e.g., 5 seconds)
        - Return to Step 1.

## Security & Provisioning Logic

**Identity Management**
- **Edge Identity**: The Device ID is hardcoded in the device configuration file. It acts as the "Source of Truth".

- **Conflict Prevention**: The Server's "Concurrency Rule" (mentioned in Phase B) prevents ID cloning or duplicate streams.

**Token Architecture**
- **Generation**: Tokens are generated cryptographically (high entropy) during device creation in the Admin Panel.

- **Storage (Edge)** Stored in secured environment file.

- **Storage (Cloud)**: Only the *Hash* of the token is stored in the database. Even if the DB is compromised, the actual keys remain safe.