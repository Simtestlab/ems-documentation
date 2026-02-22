# Maintenance Prediction Model – Complete Documentation Guide

---

## 1. Introduction
A Battery Energy Storage System (BESS) maintenance prediction model estimates:
*   🔧 **Future maintenance cost**
*   ⚠️ **Failure probability**
*   📉 **Remaining Useful Life (RUL)**
*   🛠️ **Preventive maintenance schedule**

This model helps in:
*   Reducing unexpected downtime
*   Optimizing O&M cost
*   Improving battery lifecycle
*   Supporting EMS decision-making

---

## 2. Architecture Flow

1.  **Data Collection Layer**
    *   BMS (Battery Management System)
    *   SCADA
    *   IoT Sensors
    *   Weather API
    *   EMS logs
2.  **Data Processing Layer**
    *   Cleaning
    *   Feature engineering
    *   Aggregation (1 min / 5 min / 1 hr)
3.  **Model Layer**
    *   Failure prediction model
    *   RUL regression model
    *   Cost estimation model
4.  **Application Layer**
    *   Dashboard
    *   Alerts
    *   Maintenance scheduling API

---

## 3. Input Data Requirements

### 🔹 A. Battery Parameters
*   Voltage (cell & pack)
*   Current
*   State of Charge (SOC)
*   State of Health (SOH)
*   Temperature
*   Depth of Discharge (DoD)
*   Charge/Discharge cycles

### 🔹 B. Environmental Data
*   Ambient temperature
*   Humidity
*   Location (Latitude/Longitude)
*   Grid load
*   Dust conditions (optional)

### 🔹 C. Operational Data
*   Runtime hours
*   Fault history
*   Alarm logs
*   Maintenance history
*   Installation date

---

## 4. Output Predictions

| Output | Type | Description |
| :--- | :--- | :--- |
| **Failure Probability** | Classification | Probability of failure in next X days |
| **Maintenance Cost** | Regression | Estimated repair cost (₹ / $) |
| **Remaining Useful Life** | Regression | Days/months before critical degradation |
| **Maintenance Type** | Classification | Minor / Major / Replacement |

---

## 5. Model Types Used

### 🔹 1. Classification (Failure Prediction)
*   XGBoost
*   Random Forest
*   LightGBM
*   Logistic Regression (baseline)

### 🔹 2. Regression (Cost & RUL)
*   XGBoost Regressor
*   LSTM (for time-series degradation)
*   Gradient Boosting
*   Survival Analysis models

### 🔹 3. Deep Learning (Advanced)
*   LSTM
*   GRU
*   CNN-LSTM Hybrid
*   Transformer Time-Series models

---

## 6. Feature Engineering

Important derived features:
*   Moving average temperature
*   Voltage deviation
*   SOC fluctuation rate
*   Cycle stress index
*   Thermal stress factor
*   Aging index
*   Monthly degradation slope

**Example:**
`Degradation Rate = (SOH_t - SOH_t-30days) / 30`

---

## 7. Model Training Pipeline

`Raw Data` → `Cleaning` → `Feature Engineering` → `Train/Test Split` → `Scaling` → `Model Training` → `Cross Validation` → `Hyperparameter Tuning` → `Model Evaluation` → `Deployment`

### Evaluation Metrics

#### Classification
*   Accuracy
*   Precision / Recall
*   ROC-AUC
*   F1 Score

#### Regression
*   MAE
*   RMSE
*   R²

---

## 8. Sample Use Case Flow (Location-based Prediction)

**User enters:**
*   Location
*   BESS capacity (MWh)
*   Battery type (Li-ion / LFP / NMC)
*   Installation year
*   Temperature parameters/ weather
*   Operating temperature range

**System:**
1.  Calculate stress factors
2.  Run trained ML model
3.  **Returns:**
    *   Maintenance probability: **24%**
    *   Estimated cost: **₹3.2 Lakhs**
    *   RUL: **2.3 years**

---

## 9. Data Input and Output Results

### JSON Input Schema (Single Record)
```json
{
  "timestamp": "2026-01-01T00:00:00Z",
  "site_id": "SITE_01",
  "battery_id": "BATT_001",
  "battery_type": "LFP",
  "capacity_mwh": 5.0,
  "installation_year": 2022,
  "avg_voltage": 710.5,
  "avg_current": 120.4,
  "avg_power_kw": 850.3,
  "soc_avg": 76.5,
  "soc_std": 4.3,
  "soh_current": 94.2,
  "internal_resistance_mohm": 2.3,
  "cycle_count": 542,
  "daily_charge_kwh": 4120,
  "daily_discharge_kwh": 3980,
  "avg_battery_temp": 32.4,
  "max_battery_temp": 39.2,
  "temp_variance": 3.1,
  "ambient_temp": 34.0,
  "humidity": 65.0,
  "thermal_stress_index": 17.2,
  "soh_drop_30d": -0.3,
  "voltage_variance_7d": 5.6,
  "fault_count_30d": 2,
  "avg_fault_severity": 2.0,
  "load_stress_factor": 0.78,
  "aging_index": 0.42,
  "downtime_hours_30d": 1.5
}
```

### JSON Array Format (Training Dataset)
```json
[
  {
    "timestamp": "2026-01-01T00:00:00Z",
    "site_id": "SITE_01",
    "battery_id": "BATT_001",
    "battery_type": "LFP",
    "capacity_mwh": 5.0,
    "installation_year": 2022,
    "avg_voltage": 710.5,
    "avg_current": 120.4,
    "avg_power_kw": 850.3,
    "soc_avg": 76.5,
    "soc_std": 4.3,
    "soh_current": 94.2,
    "internal_resistance_mohm": 2.3,
    "cycle_count": 542,
    "daily_charge_kwh": 4120,
    "daily_discharge_kwh": 3980,
    "avg_battery_temp": 32.4,
    "max_battery_temp": 39.2,
    "temp_variance": 3.1,
    "ambient_temp": 34.0,
    "humidity": 65.0,
    "thermal_stress_index": 17.2,
    "soh_drop_30d": -0.3,
    "voltage_variance_7d": 5.6,
    "fault_count_30d": 2,
    "avg_fault_severity": 2.0,
    "load_stress_factor": 0.78,
    "aging_index": 0.42,
    "downtime_hours_30d": 1.5
  }
]
```

### Output JSON
```json
{
  "failure_prediction": {
    "probability_30_days": 0.27,
    "risk_level": "LOW"
  },
  "rul_prediction": {
    "remaining_days": 895,
    "confidence_interval": [840, 950]
  },
  "maintenance_prediction": {
    "expected_cost": 52000,
    "downtime_hours": 6.5,
    "recommended_action": "Preventive Inspection"
  }
}
```

---

## 10. Existing Free Source Tools For BESS ROI Calculation

*   [BX Energy Systems ROI Calculator](https://bxenergysystems.com/battery-energy-storage-systems-roi-calculator/)
*   [Grid-Synk BESS ROI Calculator](https://grid-synk.com/bess-roi-calculator/)
