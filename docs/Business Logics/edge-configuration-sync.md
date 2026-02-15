# Edge Configuration Sync

## Purpose

The Edge Configuration Sync service is the centralized configuration management backebone of the EMS.

- Synchronizes all operational settings from the cloud to every edge device.

- Ensures that all edges across multiple sites run identical (or site-specific) configurations without manual intervention.

- Provides a robust, fault-tolerant mechanism for updating devie mappings, controller parameters, and system settings remotely.

- Maintains configuration versioning and audit trails for compilance and troubleshooting.

## Core Functional Requirements

### Cloud Configuration Repository

| S No | Requirement | Description |
|----|-------------|-------------|
| **01** | **Central source of truth** | All edge configurations are stored in the cloud database (PostgreSQL), organized by organization → site → edge. |
| **02** | **Versioned configurations** | Every configuration change creates a new version; previous versions are retainable for audit and rollback. |
| **03** | **Configuration hierarchy** | Support inheritance: organization‑level defaults → site‑level overrides → edge‑specific exceptions. |
| **04** | **UI for configuration management** | Django admin / Next.js dashboard allows operators to view and edit configurations for any edge. |
| **05** | **Bulk operations** | Apply configuration changes to multiple edges simultaneously (e.g., "update peak shaving threshold for all edges in Site A"). |

### Sync API Server

| S No | Requirement | Description |
|----|-------------|-------------|
| **01** | **Edge authentication** | Each edge has a unique API key. The API validates the edge identity and checks its organization/site membership. |
| **02** | **Config fetch endpoint** | `GET /api/v1/edges/{edge_id}/config` returns the full configuration for the requesting edge. Supports `If-None-Match` with ETags for cache efficiency. |
| **03** | **Delta updates** | Optional endpoint `GET /api/v1/edges/{edge_id}/config/delta?since=<version>` returns only changes since the specified version. |
| **04** | **Change notifications** | WebSocket channel (`/ws/edges/{edge_id}/notify`) pushes real‑time alerts when configuration changes occur. Edge can then fetch updates immediately. |
| **05** | **Heartbeat & status** | Edge periodically sends heartbeat (`POST /api/v1/edges/{edge_id}/heartbeat`) with its current config version and sync status. |

### Edge Sync Agent

| S No | Requirement | Description |
|----|-------------|-------------|
| **01** | **Initial sync on boot** | On startup, edge agent fetches full configuration from cloud and stores it in local SQLite cache. |
| **02** | **Periodic polling** | Agent polls cloud every `sync_interval` (configurable, default 5 minutes) using ETags to check for changes. |
| **03** | **WebSocket listener** | Agent maintains persistent WebSocket connection to receive real‑time change notifications (falls back to polling if connection drops). |
| **04** | **Delta application** | When changes are detected, agent downloads only the changed sections (delta) and applies them atomically. |
| **05** | **Local cache** | SQLite database stores the current active configuration and at least one previous version for rollback. |
| **06** | **Validation before apply** | Agent validates new configuration against schema before applying; if invalid, rejects and alerts cloud. |
| **07** | **Controlled rollout** | Support for staged updates: download, stage, apply at next cycle start, or apply immediately (configurable per component). |
| **08** | **Rollback capability** | If a new configuration causes errors (e.g., Core Cycle crashes), agent automatically reverts to previous known‑good version and reports failure. |
| **09** | **Status reporting** | Agent reports sync status, current version, and any errors to cloud via heartbeat API. |

---

## Configuration Data Model

The configuration is structured as a JSON object that maps directly to the edge's operational parameters.

```json
{
  "version": 42,
  "timestamp": "2026-02-15T10:30:00Z",
  "edge_id": "site1_edge_prod",
  "core_cycle": {
    "period_ms": 1000,
    "supervisor": {
      "overrun_alert_threshold": 2,
      "auto_recovery": true
    }
  },
  "scheduler": {
    "controllers": [
      {
        "id": "ctrlBalancing0",
        "class": "BalancingController",
        "enabled": true,
        "execution_order": 1,
        "parameters": {
          "target_grid_power": 0,
          "priority": "self_consumption"
        }
      },
      {
        "id": "ctrlPeakShaving0",
        "class": "PeakShavingController",
        "enabled": true,
        "execution_order": 2,
        "parameters": {
          "threshold_watts": 50000,
          "recharge_enabled": true
        }
      }
    ]
  },
  "modbus_devices": [
    {
      "device_id": "bms_main",
      "profile": "tesla_powerwall_2",
      "connection": {
        "protocol": "tcp",
        "host": "192.168.1.100",
        "port": 502,
        "unit_id": 1
      },
      "poll_groups": {
        "fast": ["ActivePower", "Soc"],
        "slow": ["Temperature", "Cycles"]
      }
    },
    {
      "device_id": "inverter_pv",
      "profile": "sma_sunny_tripower",
      "connection": {
        "protocol": "tcp",
        "host": "192.168.1.101",
        "port": 502,
        "unit_id": 2
      }
    }
  ],
  "alarms": [
    {
      "id": "alarm_soc_low",
      "channel": "bms_main/Soc",
      "condition": "<",
      "threshold": 20,
      "severity": "warning",
      "notification": ["email", "dashboard"]
    }
  ]
}
```

---

## Sync Process Flow

### Normal Operation (Polling Mode)

```
Edge                          Cloud
 |                              |
 |--- GET /config (ETag: abc) -->|
 |<-- 304 Not Modified ----------| (no changes)
 |                              |
 | (wait sync_interval)         |
 |                              |
 |--- GET /config (ETag: abc) -->|
 |<-- 200 OK (new config v43) ---|
 |                              |
 |-- Validate schema -----------|
 |-- Stage new config ----------|
 |-- Apply at next cycle -------|
 |-- POST /heartbeat (v43) ----->|
```

### Real‑Time Notification Mode

```
Edge                          Cloud
 |                              |
 |--- WebSocket connect ------->|
 |<-- Connection accepted ------|
 |                              |
 | (Admin changes config via UI)|
 |                              |
 |<-- WS message: "config updated v43"
 |                              |
 |--- GET /config/delta?v=42 -->|
 |<-- 200 OK (delta only) -------|
 |                              |
 |-- Apply delta ---------------|
 |-- POST /heartbeat (v43) ----->|
```

### Failure & Rollback

```
Edge                          Cloud
 |                              |
 |--- GET /config (v43) ------->|
 |<-- 200 OK (new config) ------|
 |                              |
 |-- Validate: PASS ------------|
 |-- Stage config --------------|
 |-- Apply at cycle start ------|
 |-- Core Cycle crashes! -------|
 |-- Auto-detect failure -------|
 |-- Rollback to v42 -----------|
 |-- POST /heartbeat (v42, error: "controller init failed")
 |                              |
 | (Operator alerted)           |
```

---
## Metrics & Acceptance Criteria

| Metric | Target | How Measured |
|--------|--------|--------------|
| **Sync latency (polling)** | ≤ 5 minutes (configurable) | Cloud logs time of change → edge heartbeat with new version |
| **Sync latency (WebSocket)** | ≤ 5 seconds | Time from cloud change → edge acknowledgment |
| **Configuration freshness** | 99.9% of edges run config < 1 hour old | Heartbeat data aggregated in cloud |
| **Rollback success rate** | 100% of detected failures | Edge logs + cloud alerts |
| **Sync success rate** | ≥ 99.5% of scheduled polls | Successful polls / total attempts |
| **API availability** | ≥ 99.9% | Cloud monitoring |

---

## Integration with Other EMS Components

| Component | Interaction |
|-----------|-------------|
| **Device & Site Management (Cloud)** | Provides the source data for configurations (sites, edges, devices). |
| **Core Cycle & Scheduler** | Receives `core_cycle` and `scheduler` settings from sync agent; applies them at runtime. |
| **Modbus Protocol Adapter** | Receives `modbus_devices` configuration; reloads device profiles when changed. |
| **Controllers (Balancing, Peak Shaving)** | Receive parameters (thresholds, enable/disable) dynamically without restart. |
| **Alarm Engine** | Receives alarm rule definitions; updates in real time. |
| **Dashboard (Next.js)** | Allows operators to view and edit configurations; displays sync status per edge. |
| **Multi‑Tenancy** | Ensures configuration data is strictly isolated by organization. |

---

## Security Considerations

| Concern | Mitigation |
|---------|------------|
| **Edge authentication** | Each edge provisioned with unique API key (stored securely); rotate periodically. |
| **Man‑in‑middle** | All communication over HTTPS / WSS with TLS 1.3. |
| **Configuration tampering** | Digital signatures: cloud signs configurations; edge verifies signature before applying. |
| **Denial of service** | Rate limiting on API endpoints; edge backoff on repeated failures. |
| **Multi‑tenant isolation** | All API queries enforce `edge_id` belongs to authenticated organization. |

---