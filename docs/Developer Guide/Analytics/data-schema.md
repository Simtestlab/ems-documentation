# Telemetry Data Schema

## Overview

Every second, the Raspberry Pi sends telemetry data to the cloud via WebSocket in JSON format.

---

## 📦 **Complete Telemetry Structure**

```json
{
    "device_id": "ems_001",
    "timestamp": "2026-02-10T14:25:30.123456",
    "battery": { ... },
    "grid": { ... },
    "solar": { ... },
    "inverter": { ... },
    "load": { ... },
    "system": { ... },
    "economics": { ... }
}
```

---

## 🔋 **1. Battery**

### Current Schema
```json
"battery": {
    "soc": 0.85,              // State of Charge (0.0 - 1.0)
    "soh": 0.98,              // State of Health (0.0 - 1.0)
    "voltage": 380.5,         // Volts
    "current": 12.3,          // Amps (+ charge, - discharge)
    "power_kw": 4.68,         // Kilowatts
    "temperature_c": 28.5,    // Celsius
    "cycle_count": 342,       // Total cycles
    "faults": [],             // Active fault codes
    "warnings": ["HIGH_TEMP"] // Active warnings
}
```

### Recommended Additions
```json
"battery": {
    // ... existing fields ...
    
    // Cell-level monitoring (safety & longevity)
    "cell_voltage_min": 3.75,      // Lowest cell voltage (detect weak cells)
    "cell_voltage_max": 3.78,      // Highest cell voltage
    "cell_voltage_delta": 0.03,    // Difference (balance indicator)
    "cell_temp_min": 27.2,         // Coldest cell temperature
    "cell_temp_max": 29.8,         // Hottest cell temperature
    
    // Capacity & availability
    "remaining_capacity_kwh": 8.5, // Available energy at current SOC
    "total_capacity_kwh": 10.0,    // Total rated capacity
    "charge_rate_kw": 5.0,         // Max safe charge rate
    "discharge_rate_kw": 5.0,      // Max safe discharge rate
    
    // User convenience
    "time_to_full_min": 45,        // Minutes until 100% (null if discharging)
    "time_to_empty_min": null,     // Minutes until 0% (null if charging)
    
    // Maintenance
    "balancing_active": false,     // Cell balancing in progress
    "last_full_charge": "2026-02-10T08:30:00Z"
}
```

---

## ⚡ **2. Grid**

### Current Schema
```json
"grid": {
    "frequency": 50.02,       // Hz
    "voltage": 230.5,         // Volts
    "power_kw": 1.96,         // kW (+ import, - export)
    "price": 0.15,            // $/kWh current rate
    "status": "connected",    // connected/disconnected/fault
    "trip_occurred": false    // Circuit breaker status
}
```

### Recommended Additions
```json
"grid": {
    // ... existing fields ...
    
    // Power quality (efficiency & compliance)
    "current": 8.5,                  // Amps
    "power_factor": 0.98,            // 0.0 - 1.0
    "reactive_power_kvar": 0.15,     // Reactive power
    "apparent_power_kva": 2.0,       // Apparent power
    "thd_voltage": 2.1,              // Total Harmonic Distortion %
    "thd_current": 3.5,              // THD current %
    
    // Energy tracking (billing & analytics)
    "energy_imported_kwh": 12.5,     // Daily cumulative import
    "energy_exported_kwh": 3.2,      // Daily cumulative export
    
    // Demand management (cost optimization)
    "peak_demand_kw": 6.8,           // Highest power today
    "peak_demand_time": "2026-02-10T07:15:00Z",
    
    // Safety & reliability
    "grid_events": [],               // Voltage sags/swells/outages
    "islanding_detected": false      // Anti-islanding protection
}
```

---

## ☀️ **3. Solar**

### Current Schema
```json
"solar": {
    "power_ac_kw": 6.5,       // AC output (after inverter)
    "power_dc_kw": 6.8,       // DC from panels
    "irradiance_w_m2": 850,   // Solar irradiance
    "panel_temp_c": 45.2,     // Panel temperature
    "efficiency": 0.956       // Conversion efficiency (0.0 - 1.0)
}
```

### Recommended Additions
```json
"solar": {
    // ... existing fields ...
    
    // DC side monitoring
    "voltage_dc": 385.0,             // DC voltage
    "current_dc": 17.7,              // DC current
    
    // String-level diagnostics (fault detection)
    "string_voltages": [192.5, 192.5],
    "string_currents": [8.85, 8.85],
    
    // Performance tracking
    "mppt_efficiency": 0.985,        // MPPT algorithm efficiency
    "energy_today_kwh": 28.5,        // Total generated today
    "energy_total_kwh": 12450.0,     // Lifetime generation
    "expected_power_kw": 6.8,        // Expected at current irradiance
    "performance_ratio": 0.956,      // Actual vs expected (health metric)
    
    // Fault detection
    "shading_detected": false,       // Shading affecting output
    "inverter_temp_c": 52.0          // Solar inverter temperature
}
```

---

## 🔄 **4. Inverter**

### Current Schema
```json
"inverter": {
    "priority": "solar_first",        // Operating priority
    "action": "charging_battery",     // Current action
    "reason": "excess_solar_available", // Why this action
    "peak_demand_threshold": 8.0      // kW limit for peak shaving
}
```

### Recommended Additions
```json
"inverter": {
    // ... existing fields ...
    
    // Operating status
    "operating_state": "grid_tie",   // standby/grid_tie/off_grid/backup
    "available_power_kw": 10.0,      // Max output capacity
    "utilization": 0.65,             // Current load vs capacity (0.0 - 1.0)
    "efficiency": 0.956,             // AC out / DC in
    
    // Maintenance & reliability
    "accumulated_runtime_hours": 8520,
    "fault_count_today": 0,
    "last_state_change": "2026-02-10T11:20:00Z",
    
    // Capabilities
    "grid_tie_enabled": true,        // Grid export allowed
    "backup_ready": true             // UPS mode ready
}
```

---

## 🏠 **5. Load** (Critical Addition!)

```json
"load": {
    "total_load_kw": 1.82,           // Total household consumption
    "critical_load_kw": 0.8,         // Essential circuits only
    "non_critical_load_kw": 1.02,    // Non-essential load
    
    // Peak tracking
    "peak_load_today_kw": 5.2,
    "peak_load_time": "2026-02-10T08:00:00Z",
    
    // Analytics
    "average_load_1h_kw": 2.1,       // Rolling 1-hour average
    "energy_consumed_today_kwh": 15.8
}
```

> **Why critical:** Load monitoring is essential for optimizing battery charge/discharge decisions and calculating self-sufficiency.

---

## 🖥️ **6. System**

```json
"system": {
    // Operating state (FSM)
    "system_state": "CHARGING",      // IDLE/CHARGING/DISCHARGING/ERROR
    "system_mode": "AUTO",           // AUTO/MANUAL/SCHEDULED
    "error_flags": [],               // Active system errors
    "warning_flags": ["BATTERY_HIGH_TEMP"],
    
    // Health monitoring
    "uptime_seconds": 864000,        // System uptime
    "wifi_rssi": -45,                // Signal strength (dBm)
    "cpu_usage": 15.2,               // CPU %
    "memory_usage": 32.5,            // Memory %
    "disk_usage": 45.0,              // SD card %
    
    // Version tracking
    "last_boot_time": "2026-02-01T00:00:00Z",
    "firmware_version": "v2.1.5"
}
```

---

## 💰 **7. Economics** (High Value Addition!)

```json
"economics": {
    // Real-time pricing
    "tariff_current": 0.15,          // $/kWh current rate
    "tariff_type": "tou",            // tou/flat/dynamic
    
    // Daily financials
    "cost_today": 1.88,              // $ spent on grid import
    "savings_today": 4.28,           // $ saved using solar/battery
    "revenue_today": 0.38,           // $ earned from grid export
    "net_cost_today": -2.78          // Total: cost - savings - revenue
}
```

> **Why high value:** Shows users immediate ROI and justifies the EMS investment.

---

## 📊 **Complete Production Payload Example**

```json
{
    "device_id": "ems_001",
    "timestamp": "2026-02-10T14:25:30.123456",
    
    "battery": {
        "soc": 0.85,
        "soh": 0.98,
        "voltage": 380.5,
        "current": 12.3,
        "power_kw": 4.68,
        "temperature_c": 28.5,
        "cycle_count": 342,
        "faults": [],
        "warnings": ["HIGH_TEMP"],
        "cell_voltage_min": 3.75,
        "cell_voltage_max": 3.78,
        "cell_voltage_delta": 0.03,
        "cell_temp_min": 27.2,
        "cell_temp_max": 29.8,
        "remaining_capacity_kwh": 8.5,
        "total_capacity_kwh": 10.0,
        "time_to_full_min": 45
    },
    
    "grid": {
        "frequency": 50.02,
        "voltage": 230.5,
        "current": 8.5,
        "power_kw": 1.96,
        "power_factor": 0.98,
        "price": 0.15,
        "status": "connected",
        "energy_imported_kwh": 12.5,
        "energy_exported_kwh": 3.2,
        "peak_demand_kw": 6.8
    },
    
    "solar": {
        "power_ac_kw": 6.5,
        "power_dc_kw": 6.8,
        "voltage_dc": 385.0,
        "current_dc": 17.7,
        "irradiance_w_m2": 850,
        "panel_temp_c": 45.2,
        "efficiency": 0.956,
        "energy_today_kwh": 28.5,
        "performance_ratio": 0.956
    },
    
    "inverter": {
        "priority": "solar_first",
        "action": "charging_battery",
        "reason": "excess_solar_available",
        "operating_state": "grid_tie",
        "utilization": 0.65,
        "efficiency": 0.956,
        "backup_ready": true
    },
    
    "load": {
        "total_load_kw": 1.82,
        "critical_load_kw": 0.8,
        "peak_load_today_kw": 5.2,
        "energy_consumed_today_kwh": 15.8
    },
    
    "system": {
        "system_state": "CHARGING",
        "system_mode": "AUTO",
        "error_flags": [],
        "warning_flags": ["BATTERY_HIGH_TEMP"],
        "uptime_seconds": 864000,
        "wifi_rssi": -45,
        "firmware_version": "v2.1.5"
    },
    
    "economics": {
        "tariff_current": 0.15,
        "tariff_type": "tou",
        "cost_today": 1.88,
        "savings_today": 4.28,
        "revenue_today": 0.38,
        "net_cost_today": -2.78
    }
}
```

---

## ✅ **Summary**

**Current Schema:** Good foundation with battery, grid, solar, and inverter basics

**Critical Additions:**
1. 🏠 **Load monitoring** - Essential for optimization decisions
2. 🔋 **Cell-level battery data** - Safety and longevity tracking
3. 💰 **Energy economics** - User ROI and financial visibility
4. ⚡ **Power quality metrics** - Grid compliance and efficiency

**Payload Size:** ~500-800 bytes per second (very efficient for 1Hz transmission)
