# EMS Backend API Documentation

## Base URL

```
http://localhost:8000/api
```

---

# Authentication Service

## Health Check

**Endpoint**

```
GET /api/auth/health
```

**Response**

```json
{
  "status": "healthy",
  "service": "Authentication Service",
  "version": "1.0.0"
}
```

---

## Login

**Endpoint**

```
POST /api/auth/login
```

**Request Body**

```json
{
  "email": "user@example.com",
  "password": "password123"
}
```

**Response**

```json
{
  "access": "JWT_ACCESS_TOKEN",
  "refresh": "JWT_REFRESH_TOKEN"
}
```

---

## Refresh Token

**Endpoint**

```
POST /api/auth/refresh
```

**Request Body**

```json
{
  "refresh": "JWT_REFRESH_TOKEN"
}
```

**Response**

```json
{
  "access": "NEW_ACCESS_TOKEN"
}
```

---

## Verify Token

**Endpoint**

```
POST /api/auth/verify
```

**Request Body**

```json
{
  "token": "JWT_ACCESS_TOKEN"
}
```

**Response**

```json
{
  "valid": true
}
```

---

## Current User Profile

**Endpoint**

```
GET /api/auth/me
```

**Headers**

```
Authorization: Bearer ACCESS_TOKEN
```

**Response**

```json
{
  "id": 1,
  "email": "user@example.com",
  "username": "user"
}
```

---

## Logout

**Endpoint**

```
POST /api/auth/logout
```

**Headers**

```
Authorization: Bearer ACCESS_TOKEN
```

**Request Body**

```json
{
  "refresh": "JWT_REFRESH_TOKEN"
}
```

**Response**

```json
{
  "message": "Successfully logged out"
}
```

---

# EMS APIs

##  Health Check

**Endpoint**

```
GET /api/health
```

**Response**

```json
{
  "status": "healthy",
  "service": "EMS API"
}
```

---

## Live Telemetry

**Endpoint**

```
GET /api/ems/live
```

**Response**

```json
{
  "timestamp": "2026-01-29T10:21:33Z",
  "total_generation": 420.5,
  "total_consumption": 380.2,
  "battery_soc": 78.4
}
```

---

## KPIs

**Endpoint**

```
GET /api/ems/kpis
```

**Response**

```json
{
  "energy_saved": 120.4,
  "co2_reduction": 34.8,
  "cost_savings": 215.7
}
```

---

## Charts Data

**Endpoint**

```
GET /api/ems/charts
```

**Response**

```json
{
  "power_generation": [120, 130, 150],
  "consumption": [100, 110, 115],
  "timestamps": ["10:00", "10:05", "10:10"]
}
```

---

## Analytics

**Endpoint**

```
GET /api/ems/analytics
```

**Response**

```json
{
  "efficiency_score": 88.6,
  "system_health": "optimal",
  "forecast_accuracy": 92.4
}
```

---

## Alerts

**Endpoint**

```
GET /api/ems/alerts
```

**Response**

```json
[
  {
    "id": 1,
    "level": "warning",
    "message": "Battery SOC low",
    "timestamp": "2026-01-29T09:44:00Z"
  }
]
```

---

## Tariff

**Endpoint**

```
GET /api/ems/tariff
```

**Response**

```json
{
  "current_rate": 0.14,
  "daily_cost": 24.5,
  "monthly_estimate": 720.3
}
```

---

## Weather

**Endpoint**

```
GET /api/ems/weather
```

**Response**

```json
{
  "temperature": 32.4,
  "humidity": 61,
  "solar_irradiance": 720
}
```

---

## Sites List

**Endpoint**

```
GET /api/ems/sites
```

**Response**

```json
[
  {
    "id": "site_1",
    "name": "Solar Plant A",
    "status": "online"
  }
]
```

---

## Site Charts

**Endpoint**

```
GET /api/ems/sites/{site_id}/charts
```

**Example**

```
GET /api/ems/sites/site_1/charts
```

**Response**

```json
{
  "power_generation": [],
  "timestamps": []
}
```

---

# Authentication Usage Example

## Login

```bash
curl -X POST http://localhost:8000/api/auth/login \
-H "Content-Type: application/json" \
-d '{"email":"admin@test.com","password":"123456"}'
```

---

## Use Access Token

```bash
curl http://localhost:8000/api/ems/live \
-H "Authorization: Bearer ACCESS_TOKEN"
```
