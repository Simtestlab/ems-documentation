# WebSocket Client Integration Guide

## Summary
1. Pi Developer: Write a Python script that runs *ws.connect()* and loops *ws.send()* every second.

2. Web Developer: Uses the telemetry data in the Live Tab component to plot the data every second if any change is broadcasted.

## Overview

To exchange real-time telemetry, both the Edge Device (Pi) and the Frontend Dashboard must act as Websocket Clients. They intiate the connection to the FastAPI Cloud Server.

- **Protocol**: wss:// (Secure WebSocket)

- **Base Endpoint**: wss://api.domain.com/ws/device/{device_id}

- **Authentication**: Handled via Headers/Tokens

## Edge Implementation (Pi)

**Goal**: Open a persistent connection, send JSON telemetry every 1 second (update) and listen for commands.

Implementation Pattern

The Pi acts as the **Publisher**. It must handle network instability (e.g., WiFi dropouts) by implementing an "Infinite Reconnect Loop".

```py
import asyncio
import json
import websockets
from dataclasses import asdict

# Configuration
DEVICE_ID = "ems_001"
URI = f"wss://api.your-domain.com/ws/device/{DEVICE_ID}"

async def send_telemetry():
    """
    Main Loop: Connects to Cloud and streams data.
    """
    while True:
        try:
            print(f"🔌 Connecting to {URI}...")
            async with websockets.connect(URI) as websocket:
                print("✅ Connected!")
                
                while True:
                    # 1. READ SENSORS (Mocking data here)
                    telemetry_payload = {
                        "device_id": DEVICE_ID,
                        "timestamp": "2026-02-03T12:00:00Z",
                        "battery": {"voltage": 53.2, "soc": 85},
                        # ... other fields per communication-interface.md
                    }

                    # 2. SEND TELEMETRY (The "Update" Request)
                    await websocket.send(json.dumps(telemetry_payload))
                    print("📤 Sent Telemetry")

                    # 3. LISTEN FOR COMMANDS (Non-blocking check)
                    # Note: meaningful logic requires asyncio.gather() or similar
                    try:
                        msg = await asyncio.wait_for(websocket.recv(), timeout=0.1)
                        print(f"📥 Received Command: {msg}")
                    except asyncio.TimeoutError:
                        pass # No command received, continue

                    # 4. WAIT (1Hz Frequency)
                    await asyncio.sleep(1)

        except Exception as e:
            print(f"⚠️ Connection Lost: {e}. Retrying in 5s...")
            await asyncio.sleep(5)

# Entry Point
if __name__ == "__main__":
    asyncio.run(send_telemetry())
```

## Dashboard Implementation
Our goal is to connect to the specific device stream and update the UI whenever a message arrives (Get).

Dashboard acts as the **Subscriber**. It must manage the component lifecycle (connect on mount, disconnect on unmount) to prevent memory leaks.

```py
// hooks/useTelemetry.ts
import { useState, useEffect, useRef } from 'react';

// Use the interface defined in your documentation
import { TelemetryPayload } from '@/types/telemetry'; 

export const useTelemetry = (deviceId: string) => {
  const [data, setData] = useState<TelemetryPayload | null>(null);
  const [status, setStatus] = useState<"CONNECTING" | "OPEN" | "CLOSED">("CLOSED");
  const ws = useRef<WebSocket | null>(null);

  useEffect(() => {
    // 1. RAISE REQUEST (Connect)
    const socketUrl = `wss://api.your-domain.com/ws/device/${deviceId}`;
    ws.current = new WebSocket(socketUrl);
    setStatus("CONNECTING");

    // 2. HANDLE STREAM (Get Telemetry)
    ws.current.onmessage = (event) => {
      try {
        const parsedData: TelemetryPayload = JSON.parse(event.data);
        setData(parsedData); // Triggers React Re-render
      } catch (err) {
        console.error("Failed to parse telemetry:", err);
      }
    };

    ws.current.onopen = () => {
      setStatus("OPEN");
      console.log(`✅ Connected to stream: ${deviceId}`);
    };

    ws.current.onclose = () => setStatus("CLOSED");

    // 3. CLEANUP (Disconnect on unmount)
    return () => {
      if (ws.current) {
        ws.current.close();
      }
    };
  }, [deviceId]);

  return { data, status };
};
```

## Identification & Connection Management

Each Raspberry Pi should have a unique identity. 

### Handshake Process

1. **Pi Boots up**: Reads *DEVICE_ID="ems_001"* from its config.

2. **Pi Connects**: Initiates WebSocket to *wss://api.domain.com/ws/device/ems_001*.

3. **Server Checks**:
    - Checks the URL parameter (ems_001) and if the ID is valid and doesn't conflict with existing IDs then becomes true.

    - If the ID is valid, then it checks the API Key/Token in the header (Security).

    - Once connection accepted, the ID will be added to a Connection Manager Dictionary for future validation.

## Websocket Security & Authentication Logic

### Bearer Tokens

Every physical device is assigned a unique, high-entopy secret string (API token) during the provisioning phase. This token acts as the device's password.

- **Rule 1**: The Token is never sent in the URL.

- **Rule 2**: The Token is send in the *Authorization* HTTP header during the Websocket Handshake.

- **Rule 3**: The server stores a *Hash* of the token (like a password), never the plain text.

### Authentication Flow
1. Pi Boot: Reads *API_SECRET* from its local *.env* file.

2. Handshake: Pi sends connection request:

    - GET /ws/device/ems_001
    - Authorization: Bearer sk_live_83928

3. Server Verification:
    - Extracts the Token from the Header.
    - Hashes the Token.
    - Compares it to the Hashed Secret stored in *devices* table for *ems_001*.

4. Decision:
    - Match: Connection Accepted (101 Switching Protocols)
    - Mismatch: Connection Rejected (403 Forbidden).

## Implementation: Edge Side (Pi)
The Python websockets library supports sending custom headers. This is how the Edge Developer must implement the connection.

```py
import asyncio
import websockets
import os

# 1. Load Secrets securely (never hardcode in script)
DEVICE_ID = os.getenv("DEVICE_ID")    # e.g., "ems_001"
API_TOKEN = os.getenv("API_TOKEN")    # e.g., "sk_live_..."
URI = f"wss://api.your-domain.com/ws/device/{DEVICE_ID}"

async def secure_connect():
    # 2. Create the Auth Header
    auth_headers = {
        "Authorization": f"Bearer {API_TOKEN}"
    }

    # 3. Connect with Headers
    try:
        async with websockets.connect(URI, extra_headers=auth_headers) as websocket:
            print(f"🔐 Authenticated & Connected as {DEVICE_ID}")
            # ... start sending telemetry ...
            
    except websockets.exceptions.InvalidStatusCode as e:
        if e.status_code == 403:
            print("⛔ Authentication Failed! Check your API Token.")
        else:
            print(f"⚠️ Connection Error: {e}")

if __name__ == "__main__":
    asyncio.run(secure_connect())
```

## Implementation: Cloud Side (FastAPI)
FastAPI allows us to intercept the connection before it opens to check the header.

```py
from fastapi import FastAPI, WebSocket, Header, status

app = FastAPI()

# MOCK DATABASE (In real life, this is your SQLite/Postgres)
# We store the HASH, not the real token.
VALID_DEVICE_TOKENS = {
    "ems_001": "hashed_secret_value_123",
    "ems_002": "hashed_secret_value_456"
}

@app.websocket("/ws/device/{device_id}")
async def websocket_endpoint(
    websocket: WebSocket, 
    device_id: str, 
    # FastAPI automatically extracts the 'Authorization' header
    authorization: str = Header(None) 
):
    # --- SECURITY CHECK ---
    
    # 1. Check if Header exists
    if authorization is None:
        print("⛔ No Token Provided")
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return

    # 2. Extract Token (Remove "Bearer " prefix)
    scheme, _, token = authorization.partition(" ")
    if scheme.lower() != "bearer":
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return

    # 3. Validate Credentials
    # (In production: verify_password_hash(token, stored_hash))
    is_valid = verify_token_in_db(device_id, token)

    if not is_valid:
        print(f"⛔ Intruder Alert! Invalid Token for {device_id}")
        await websocket.close(code=status.WS_1008_POLICY_VIOLATION)
        return

    # --- IF PASS, PROCEED ---
    await websocket.accept()
    print(f"✅ {device_id} Authenticated.")
    
    # ... Add to ConnectionManager and start loop ...
```