# Edge Platform Backend API Interface Documentation

This document describes the backend API interfaces consumed by the Edge
UI application. The backend uses JSON-RPC and Modbus TCP based service
endpoints.

------------------------------------------------------------------------

## Base API Gateway

### JSON-RPC Endpoint

POST /jsonrpc

Content-Type: application/json

All application requests are sent via this unified JSON-RPC gateway.

------------------------------------------------------------------------

## Common JSON-RPC Request Format

``` json
{
  "jsonrpc": "2.0",
  "method": "METHOD_NAME",
  "params": {},
  "id": 1
}
```

------------------------------------------------------------------------

## Common JSON-RPC Response Format

``` json
{
  "jsonrpc": "2.0",
  "result": {},
  "id": 1
}
```

------------------------------------------------------------------------

## Live Telemetry API

### Method

getLiveData

### Description

Fetches real-time edge device telemetry values.

### Request

``` json
{
  "method": "getLiveData",
  "params": {
    "deviceId": "EDGE_DEVICE_ID"
  }
}
```

### Response

``` json
{
  "voltage": 230.4,
  "current": 12.8,
  "power": 2950,
  "frequency": 50.0
}
```

------------------------------------------------------------------------

## Historical Data API

### Method

getHistoryData

### Description

Fetch historical telemetry data for charts.

### Request

``` json
{
  "method": "getHistoryData",
  "params": {
    "deviceId": "EDGE_DEVICE_ID",
    "from": "2026-01-01T00:00:00Z",
    "to": "2026-01-02T00:00:00Z"
  }
}
```

### Response

``` json
{
  "timestamps": [],
  "values": []
}
```

------------------------------------------------------------------------

## Digital Input IO API

### Method

getDigitalInputs

### Description

Reads digital IO states from edge hardware.

### Request

``` json
{
  "method": "getDigitalInputs",
  "params": {
    "deviceId": "EDGE_DEVICE_ID"
  }
}
```

### Response

``` json
{
  "inputs": [
    {
      "channel": 1,
      "state": true
    }
  ]
}
```

------------------------------------------------------------------------

## Modbus TCP Register Read

### Method

readHoldingRegisters

### Description

Reads Modbus TCP holding registers.

### Request

``` json
{
  "method": "readHoldingRegisters",
  "params": {
    "host": "192.168.1.10",
    "port": 502,
    "startAddress": 0,
    "count": 10
  }
}
```

### Response

``` json
{
  "registers": [123, 456, 789]
}
```

------------------------------------------------------------------------

## Modbus TCP Register Write

### Method

writeHoldingRegister

### Description

Writes value to Modbus register.

### Request

``` json
{
  "method": "writeHoldingRegister",
  "params": {
    "host": "192.168.1.10",
    "port": 502,
    "address": 1,
    "value": 500
  }
}
```

### Response

``` json
{
  "status": "success"
}
```

------------------------------------------------------------------------

## Device Status API

### Method

getDeviceStatus

### Description

Fetches health and connectivity state.

### Response

``` json
{
  "online": true,
  "uptime": "48h",
  "temperature": 42.5
}
```

------------------------------------------------------------------------

## Authentication Header (If Enabled)

    Authorization: Bearer ACCESS_TOKEN

------------------------------------------------------------------------

## Error Response Format

``` json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32602,
    "message": "Invalid params"
  },
  "id": 1
}
```

