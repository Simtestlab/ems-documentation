# Forecast Models — Developer Guide

This section documents the AI/ML models used in the Energy Management System (EMS) for energy forecasting. The forecasting module contains two distinct prediction systems, each built with the most suitable algorithm for its specific use case.

---

## Forecast Models Overview

| Model | Algorithm | Target | Frequency | Accuracy |
|-------|-----------|--------|-----------|----------|
| **Load Demand Forecasting** | LSTM (Deep Learning) | Total grid load demand (kW) | 15-minute intervals | ~89% |
| **Solar Generation Forecasting** | XGBoost (Gradient Boosting) | Solar power output (kW) | Hourly | ~60% |
| **Wind Generation Forecasting** | XGBoost (Gradient Boosting) | Wind power output (kW) | Hourly | ~78% |

---

## Why Different Algorithms?

The choice of algorithm is driven by the **nature of the data**:

- **Load Demand** is a **time-series problem** — past values directly influence future values. Electricity consumption follows temporal patterns (daily peaks, overnight lows, weekend dips). LSTM networks are specifically designed to learn these sequential dependencies.

- **Solar/Wind Generation** is a **feature-driven regression problem** — output depends on weather conditions (temperature, irradiance, wind speed, humidity) and seasonal factors. XGBoost excels at learning complex non-linear relationships between independent input features and a target variable.

> [!IMPORTANT]
> Using LSTM for solar/wind prediction (or XGBoost for load demand) would produce **inferior results** because the algorithm would not match the data's fundamental structure.

---

## Documentation

| Document | Description |
|----------|-------------|
| [Load Demand Forecasting](load-demand-forecasting.md) | LSTM architecture, training pipeline, feature engineering, and multi-step prediction |
| [Solar & Wind Generation Forecasting](solar-wind-generation-forecasting.md) | XGBoost model design, feature engineering, dual-target training, and weather simulation |
| [Algorithm Comparison & Justification](algorithm-comparison.md) | Deep technical comparison of why each algorithm was chosen |

---

## Directory Structure

```
forecast/
├── load_forecast/
│   ├── dataset/          ← Load demand dataset (15-min intervals)
│   ├── training/         ← stlf.py (train), predict.py (CLI predict)
│   └── models/           ← LSTM model + scalers + encoders
│
├── solar_wind_forecast/
│   ├── dataset/          ← Seasonal energy dataset (hourly)
│   ├── training/         ← train.py (train), predict.py (CLI predict)
│   └── models/           ← XGBoost models + scaler + encoder
│
└── webapp/
    ├── backend/          ← Flask API serving both models
    └── frontend/         ← React dashboard with tabbed UI
```
