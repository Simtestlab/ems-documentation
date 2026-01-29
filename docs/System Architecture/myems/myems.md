# MyEMS

> **Industry-Leading Open Source Energy Management System**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Version](https://img.shields.io/badge/version-5.12.0-blue)](https://github.com/MyEMS/myems)
[![Python](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![React](https://img.shields.io/badge/react-16.13.1+-blue.svg)](https://reactjs.org/)

MyEMS is a comprehensive open-source Energy Management System designed for buildings, factories, shopping malls, hospitals, and industrial campuses. Built following the ISO 50001 energy management standard (GB/T 23331-2020), it provides complete solutions for energy data collection, analysis, carbon emissions tracking, and reporting.

---

## Table of Contents

- [Overview](#overview)
- [Features](#features)
- [Technologies Used](#technologies-used)
- [Architecture](#architecture)
- [Project Breakdown](#project-breakdown)
- [Getting Started](#getting-started)
  - [Prerequisites](#prerequisites)
  - [Installation](#installation)
  - [Usage](#usage)
- [Contributing](#contributing)
- [License](#license)
- [Contact](#contact)

---

## Overview

**MyEMS** (My Energy Management System) is a comprehensive platform that enables organizations to monitor, analyze, and optimize their energy consumption across multiple facilities and resources. The system provides real-time data acquisition from various energy meters (electricity, water, gas, cooling, heating), advanced analytics, billing calculations, carbon emissions tracking, and customizable reporting.

### Primary Goals

- **Energy Monitoring**: Real-time tracking of energy consumption across organizational hierarchies
- **Cost Management**: Automated billing calculations with support for complex tariff structures
- **Carbon Emissions**: Track and report carbon footprints for sustainability initiatives
- **Data-Driven Insights**: Advanced analytics, forecasting, and benchmarking capabilities
- **Multi-Tenancy**: Support for multiple organizations with data isolation
- **Scalability**: Microservices architecture designed for enterprise-level deployments

### Target Audience

- **Facility Managers**: Monitor building energy performance and identify optimization opportunities
- **Energy Managers**: Analyze consumption patterns and implement energy-saving strategies
- **Financial Controllers**: Track energy costs and allocate expenses to cost centers
- **Sustainability Officers**: Monitor carbon emissions and generate compliance reports
- **System Integrators**: Integrate with existing BMS/SCADA systems via open APIs
- **Developers**: Extend functionality through well-documented RESTful APIs

---

## Features

MyEMS provides a comprehensive set of features for complete energy management:

### Core Features

- **📊 Multi-Level Energy Tracking**
  - Hierarchical organization structure (Enterprise → Campus → Building → Floor → Space → Equipment → Meters)
  - Support for electricity, water, gas, cooling, heating, and custom energy categories
  - Real-time and historical data collection at configurable intervals

- **💰 Advanced Billing & Cost Management**
  - Complex tariff support: Time-of-Use (TOU), tiered/stepped, block rate, power factor, seasonal
  - Automated cost allocation by cost centers and tenants
  - Multi-currency support and tax calculations
  - Billing reports with drill-down capabilities

- **🌱 Carbon Emissions Tracking**
  - Configurable emission factors by energy category
  - Automatic carbon footprint calculation
  - Scope 1, 2, and 3 emissions reporting
  - Sustainability KPI tracking and benchmarking

- **📈 Data Processing & Analytics**
  - Virtual meters with formula-based calculations
  - Data normalization, cleaning, and repair
  - Gap filling and anomaly detection
  - Trend analysis and forecasting
  - Energy baseline and savings tracking

- **🔌 Flexible Data Acquisition**
  - **Physical Meters**: Modbus TCP, MQTT, HTTP protocols
  - **Offline Meters**: Manual entry or Excel bulk import
  - **Virtual Meters**: Calculated from multiple meter aggregations
  - Automatic retry and connection resilience

- **⚡ Equipment Management**
  - Individual equipment tracking (HVAC, lighting, motors, compressors, etc.)
  - Combined equipment groups
  - Equipment efficiency analysis
  - Maintenance scheduling and fault detection (FDD)

- **🏢 Multi-Tenancy & Spaces**
  - Support for multiple independent organizations
  - Space-based tracking (offices, stores, warehouses, production floors)
  - Tenant billing and cost allocation
  - Data isolation and permission management

- **🔋 Advanced Energy Systems**
  - Photovoltaic (PV) power stations
  - Energy storage containers and power stations
  - Microgrids with generation, storage, and load management
  - Wind farms and renewable energy integration
  - EV charging station management

- **📱 Rich Reporting & Visualization**
  - 100+ pre-built reports (energy, cost, carbon, efficiency, comparison, etc.)
  - Interactive dashboards with real-time updates
  - Energy flow diagrams and Sankey charts
  - Export to Excel, PDF, and image formats
  - Custom report builder

- **🔐 Security & Administration**
  - Role-based access control (RBAC)
  - API key management
  - User activity logging and audit trails
  - Email and WeChat message notifications
  - Multi-language support (English, Chinese, German, etc.)

---

## Technologies Used

MyEMS leverages modern, proven technologies across its microservices architecture:

### Backend Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **Python** | 3.10+ | Primary backend language for all services |
| **Falcon** | Latest | Lightweight WSGI framework for RESTful API |
| **Gunicorn** | Latest | WSGI HTTP server for production deployments |
| **MySQL** | 8.0+ | Primary relational database (13 separate databases) |
| **MySQL Connector Python** | Latest | Database connectivity and ORM operations |
| **Modbus-tk** | Latest | Modbus TCP protocol implementation |
| **MQTT (Paho)** | 2.1.0 | MQTT protocol for IoT device communication |
| **Redis** | Latest | Caching and session management |
| **SymPy** | Latest | Symbolic mathematics for virtual meter calculations |
| **OpenPyXL** | Latest | Excel file generation for reports |
| **Plotly** | 5.24.0 | Server-side chart generation |
| **Kaleido** | 0.2.1 | Static image export for charts |

### Frontend Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| **React** | 16.13.1+ | User-facing web interface framework |
| **AngularJS** | Latest | Admin panel framework (legacy, being migrated) |
| **Bootstrap** | 4.5.3+ | UI component library and grid system |
| **ECharts** | 5.2.2+ | Interactive data visualization |
| **Chart.js** | 3.9.1+ | Chart rendering library |
| **Leaflet** | 1.7.1+ | Map visualization for facility locations |
| **Mapbox GL** | 2.15.0+ | Advanced map rendering |
| **React Router** | 5.2.0+ | Client-side routing |
| **i18next** | Latest | Internationalization framework |
| **Moment.js** | 2.28.0+ | Date/time manipulation |
| **React Bootstrap Table** | Latest | Advanced table components with sorting/filtering |

### Infrastructure & DevOps

| Technology | Purpose |
|------------|---------|
| **Docker** | Containerization of all services |
| **Nginx** | Reverse proxy and static file serving |
| **Apache** | Alternative web server support |
| **Git** | Version control |
| **Systemd** | Linux service management |

### Data Processing & Analytics

| Technology | Purpose |
|------------|---------|
| **Schedule (Python)** | Job scheduling for periodic tasks |
| **Telnetlib3** | Gateway connectivity testing |
| **Anytree** | Hierarchical data structure management |
| **SimplejSON** | Fast JSON parsing and serialization |

### Supported Protocols

- **Modbus TCP**: Industrial automation standard
- **MQTT**: IoT messaging protocol
- **HTTP/HTTPS**: RESTful API communication
- **WebSockets**: Real-time data streaming

---

## Architecture

MyEMS follows a **microservices architecture** with clear separation of concerns, enabling scalability, maintainability, and independent deployment of services.

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                           Client Layer                               │
├─────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐           ┌───────────────────┐              │
│  │   myems-web      │           │   myems-admin     │              │
│  │  (React SPA)     │           │  (AngularJS SPA)  │              │
│  │  User Interface  │           │  Admin Interface  │              │
│  └────────┬─────────┘           └────────┬──────────┘              │
└───────────┼──────────────────────────────┼──────────────────────────┘
            │                               │
            │        HTTPS/REST API         │
            └───────────────┬───────────────┘
                            │
┌───────────────────────────▼───────────────────────────────────────┐
│                      API Gateway Layer                             │
├────────────────────────────────────────────────────────────────────┤
│                      ┌──────────────────┐                          │
│                      │   myems-api      │                          │
│                      │ (Falcon/Gunicorn)│                          │
│                      │   RESTful API    │                          │
│                      │   Port 8000      │                          │
│                      └────────┬─────────┘                          │
└─────────────────────────────────┼──────────────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        │                         │                         │
┌───────▼──────────┐   ┌──────────▼──────────┐   ┌────────▼─────────┐
│  Database Layer  │   │  Processing Layer   │   │ Acquisition Layer│
├──────────────────┤   ├─────────────────────┤   ├──────────────────┤
│                  │   │ myems-normalization │   │ myems-modbus-tcp │
│ MySQL Databases  │◄──┤  Data Normalizer    │◄──┤  Modbus Service  │
│  (13 DBs)        │   │  Virtual Meters     │   │  Data Collector  │
│                  │   │  Data Repair        │   └──────────────────┘
│ • system_db      │   └─────────────────────┘            │
│ • historical_db  │            │                         │
│ • energy_db      │   ┌────────▼─────────┐              │
│ • billing_db     │   │ myems-cleaning   │              │
│ • carbon_db      │◄──┤  Data Cleaner    │              │
│ • energy_plan_db │   │  Anomaly Removal │              │
│ • fdd_db         │   └──────────────────┘              │
│ • user_db        │            │                         │
│ • reporting_db   │   ┌────────▼─────────┐              │
│ • production_db  │   │ myems-aggregation│              │
│ • ...            │◄──┤  Energy Rollups  │              │
│                  │   │  Billing Calc    │              │
│                  │   │  Carbon Calc     │              │
│                  │   └──────────────────┘              │
└──────────────────┘                                     │
                                                          │
                                         ┌────────────────▼──────┐
                                         │   Physical Devices    │
                                         ├───────────────────────┤
                                         │ • Energy Meters       │
                                         │ • HVAC Systems        │
                                         │ • PV Power Stations   │
                                         │ • Battery Storage     │
                                         │ • Charging Stations   │
                                         └───────────────────────┘
```

### Data Flow Architecture

MyEMS implements a **unidirectional data flow** from acquisition to presentation:

```
┌─────────────────────────────────────────────────────────────────┐
│ Stage 1: DATA ACQUISITION (Every 5-15 minutes)                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  Physical Devices (Modbus TCP)                                  │
│         ↓                                                        │
│  myems-modbus-tcp (Data Collector)                              │
│    • Read meter registers                                       │
│    • Apply calibration factors                                  │
│    • Buffer data in memory                                      │
│    • Batch write to database                                    │
│         ↓                                                        │
│  myems_historical_db.tbl_energy_value                           │
│    • Raw timestamped readings                                   │
│    • Quality flags (is_bad, is_published)                       │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ Stage 2: DATA NORMALIZATION (Every 1 hour)                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  myems-normalization                                             │
│    • Read raw meter data                                        │
│    • Apply unit conversions                                     │
│    • Calculate virtual meters (formulas)                        │
│    • Aggregate offline meter imports                            │
│    • Handle data repairs                                        │
│         ↓                                                        │
│  myems_historical_db (Normalized tables)                        │
│    • Standardized energy values                                 │
│    • Calculated virtual meter values                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ Stage 3: DATA CLEANING (Every 1 hour)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  myems-cleaning                                                  │
│    • Detect anomalies and outliers                              │
│    • Remove duplicate entries                                   │
│    • Fill data gaps via interpolation                           │
│    • Validate data consistency                                  │
│         ↓                                                        │
│  myems_historical_db (Clean data marked)                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ Stage 4: DATA AGGREGATION (Every 1 hour)                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  myems-aggregation                                               │
│    • Aggregate by time periods (hourly → daily → monthly)       │
│    • Calculate billing (apply tariffs)                          │
│    • Calculate carbon emissions (apply factors)                 │
│    • Roll up by spaces, equipment, tenants                      │
│    • Apply tariff schedules                                     │
│         ↓                                                        │
│  myems_energy_db      - Hourly/daily/monthly energy stats       │
│  myems_billing_db     - Cost calculations by entity             │
│  myems_carbon_db      - CO2 emissions by entity                 │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────────────┐
│ Stage 5: API & PRESENTATION (Real-time on demand)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  myems-api (RESTful endpoints)                                   │
│    • Query aggregated data                                      │
│    • Generate reports                                           │
│    • Export to Excel/PDF                                        │
│    • Real-time dashboard data                                   │
│         ↓                                                        │
│  myems-web / myems-admin (Frontend)                              │
│    • Visualize energy data                                      │
│    • Display dashboards                                         │
│    • Generate reports                                           │
│    • User interaction                                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Database Architecture

MyEMS uses a **multi-database separation strategy** for optimal performance and scalability:

| Database | Purpose | Data Volume | Growth Rate |
|----------|---------|-------------|-------------|
| `myems_system_db` | System configuration, equipment, meters, users | Medium | Low |
| `myems_historical_db` | Raw time-series data, files | **Very Large** | **High** |
| `myems_energy_db` | Aggregated energy statistics | **Large** | **Medium** |
| `myems_billing_db` | Billing and cost data | **Large** | **Medium** |
| `myems_carbon_db` | Carbon emissions data | **Large** | **Medium** |
| `myems_energy_baseline_db` | Energy baselines for savings analysis | Medium | Low |
| `myems_energy_model_db` | 8760-hour annual energy models | Medium | Low |
| `myems_energy_plan_db` | Energy plans and targets | Medium | Low |
| `myems_energy_prediction_db` | Energy forecasting data | Medium | Medium |
| `myems_fdd_db` | Fault detection and diagnosis | Medium | Medium |
| `myems_user_db` | User authentication, API keys | Small | Low |
| `myems_reporting_db` | Generated reports and attachments | Small | Low |
| `myems_production_db` | Production/manufacturing data | Small | Low |

**Benefits of Database Separation**:
- **Performance**: Isolate heavy write operations (historical data) from read operations (reporting)
- **Scalability**: Scale individual databases independently based on load
- **Maintenance**: Backup and maintain large databases separately
- **Security**: Apply different access controls per database type

### Design Patterns

MyEMS implements several key design patterns:

1. **Microservices Pattern**: Each service has a single responsibility and can be deployed independently
2. **Repository Pattern**: Data access logic is abstracted from business logic
3. **Pipeline Pattern**: Data flows through distinct processing stages
4. **Observer Pattern**: Real-time notifications via webhooks and message queues
5. **Strategy Pattern**: Pluggable tariff calculations and meter formulas
6. **Factory Pattern**: Dynamic creation of meter and equipment instances
7. **Singleton Pattern**: Database connection pooling and configuration management

---

## Project Breakdown

MyEMS consists of **7 main microservices** plus supporting infrastructure:

### 1. myems-api

**Purpose**: RESTful API service providing all backend functionality

**Technology Stack**:
- Python 3.10+ with Falcon framework
- Gunicorn WSGI server (4 workers, 600s timeout)
- MySQL Connector for database access
- OpenPyXL for Excel report generation
- Plotly & Kaleido for chart generation

**Key Responsibilities**:
- 100+ RESTful API endpoints for all system functions
- User authentication and authorization (JWT tokens)
- Report generation and export (Excel, PDF, images)
- Data validation and business logic enforcement
- File upload handling (offline meter data, cost files)
- Real-time data queries and aggregations

**API Categories**:
- Core configuration APIs (spaces, equipment, meters, cost centers, tariffs)
- Report APIs (energy, billing, carbon, efficiency, comparison)
- Dashboard APIs (real-time KPIs, energy flow diagrams)
- Advanced systems APIs (PV stations, energy storage, microgrids)
- User management APIs
- Control and command APIs

**Docker Support**: ✅ Dockerfile provided  
**Port**: 8000 (HTTP)  
**Performance**: Handles 1000+ concurrent requests

**Key Files**:
- [app.py](myems-api/app.py) - Main application entry point with all route mappings
- [requirements.txt](myems-api/requirements.txt) - Python dependencies
- [config.py](myems-api/config.py) - Database and system configuration

---

### 2. myems-web

**Purpose**: User-facing web interface for energy data visualization and analysis

**Technology Stack**:
- React 16.13.1+ (Hooks and functional components)
- React Router for navigation
- ECharts 5.2+ and Chart.js for visualizations
- Bootstrap 4.5+ and Reactstrap for UI components
- i18next for multi-language support
- Moment.js for date/time operations
- Leaflet and Mapbox GL for map visualization

**Key Features**:
- **Dashboards**: Real-time energy consumption overview
- **Space Reports**: Energy analysis by buildings, floors, rooms
- **Equipment Reports**: Equipment-level energy tracking and efficiency
- **Meter Reports**: Individual meter analysis, comparisons, trends
- **Tenant/Store Reports**: Multi-tenant billing and energy allocation
- **Advanced Systems**: PV stations, energy storage, microgrids, EV charging
- **Carbon Reports**: Emissions tracking and sustainability KPIs
- **Comparison Reports**: Multi-entity comparisons (spaces, equipment, meters)
- **Export Functions**: Excel, PDF, image export for all reports

**Responsive Design**: Fully responsive, works on desktop, tablet, and mobile devices

**Docker Support**: ✅ Dockerfile provided  
**Build Tool**: React Scripts (Webpack)  
**Port**: 3000 (development), 80/443 (production with Nginx)

**Key Files**:
- [package.json](myems-web/package.json) - Dependencies and build scripts
- `src/` - React source code (components, pages, services)

---

### 3. myems-admin

**Purpose**: Administrative interface for system configuration and management

**Technology Stack**:
- AngularJS (legacy, planned migration to React)
- Bootstrap 3
- jQuery and jQuery UI
- Highcharts for data visualization
- jsTree for hierarchical data display

**Key Features**:
- **Space Management**: Configure enterprise hierarchy (sites, buildings, floors, spaces)
- **Equipment Management**: Add/edit equipment, associate with meters
- **Meter Management**: Configure physical, offline, and virtual meters
- **Data Source Configuration**: Setup Modbus gateways and protocols
- **User Management**: Create users, assign roles and permissions
- **Cost Center Management**: Configure billing entities
- **Tariff Configuration**: Setup complex tariff structures (TOU, tiered, etc.)
- **Energy Category Setup**: Configure energy types and emission factors
- **Advanced Systems**: Configure PV stations, energy storage, microgrids
- **Knowledge Base**: Upload documentation and manuals

**Access Control**: Role-based permissions (Admin, Engineer, Viewer)

**Docker Support**: ✅ Dockerfile provided  
**Port**: 8001 (default)

**Key Files**:
- [index.html](myems-admin/index.html) - Main HTML entry point
- `app/` - AngularJS application modules and controllers

---

### 4. myems-modbus-tcp

**Purpose**: Data acquisition service for Modbus TCP devices

**Technology Stack**:
- Python 3.10+ with modbus-tk library
- Schedule for periodic task execution
- Telnetlib3 for gateway connectivity testing

**Key Responsibilities**:
- Connect to Modbus TCP gateways and devices
- Read configured meter registers at intervals (5-15 minutes typical)
- Apply calibration/multiplier factors to raw register values
- Buffer data in memory for efficient batch writes
- Handle connection failures with automatic retry logic
- Write timestamped data to `myems_historical_db`

**Supported Modbus Features**:
- Read Holding Registers (function code 03)
- Read Input Registers (function code 04)
- Configurable byte swapping (big-endian/little-endian)
- Multiple data types (INT16, INT32, FLOAT32, etc.)

**Performance**: Can handle 1000+ meters with 5-minute intervals

**Docker Support**: ✅ Dockerfile provided

**Key Files**:
- [acquisition.py](myems-modbus-tcp/acquisition.py) - Main data acquisition logic
- [gateway.py](myems-modbus-tcp/gateway.py) - Gateway connectivity management
- [byte_swap.py](myems-modbus-tcp/byte_swap.py) - Byte order handling

---

### 5. myems-normalization

**Purpose**: Data normalization and virtual meter calculation service

**Technology Stack**:
- Python 3.10+
- MySQL Connector
- SymPy for mathematical formula evaluation
- OpenPyXL for offline meter data import

**Key Responsibilities**:
- **Meter Data Normalization**: Convert raw readings to standard units
- **Virtual Meter Calculation**: Execute formulas (addition, subtraction, multiplication, division, etc.)
- **Offline Meter Processing**: Import manually entered data from Excel files
- **Data Repair**: Process data repair files to fix historical gaps or errors
- **Unit Conversion**: Handle unit transformations (Wh→kWh, cm³→m³, etc.)
- **Validation**: Ensure data consistency and monotonicity

**Virtual Meter Formula Example**:
```python
# Total HVAC = Chiller + Cooling Tower + AHU
expression = "x + y + z"
variables = {"x": meter_1_value, "y": meter_2_value, "z": meter_3_value}
result = evaluate_expression(expression, variables)
```

**Execution Frequency**: Every 1 hour (configurable)

**Docker Support**: ✅ Dockerfile provided

**Key Files**:
- [meter.py](myems-normalization/meter.py) - Physical meter normalization
- [virtualmeter.py](myems-normalization/virtualmeter.py) - Virtual meter calculations
- [offlinemeter.py](myems-normalization/offlinemeter.py) - Offline meter processing

---

### 6. myems-cleaning

**Purpose**: Data quality assurance and anomaly removal service

**Technology Stack**:
- Python 3.10+
- MySQL Connector
- Schedule for periodic execution

**Key Responsibilities**:
- **Anomaly Detection**: Identify outliers and unrealistic values
- **Duplicate Removal**: Remove duplicate time-series entries
- **Gap Detection**: Identify missing data points
- **Data Smoothing**: Apply statistical methods to smooth noisy data
- **Quality Marking**: Flag bad data for exclusion from reports

**Cleaning Strategies**:
- Statistical outlier detection (Z-score, IQR methods)
- Time-series continuity checks
- Range validation (min/max bounds)
- Spike detection and removal

**Execution Frequency**: Every 1 hour (configurable)

**Docker Support**: ✅ Dockerfile provided

**Key Files**:
- [clean_analog_value.py](myems-cleaning/clean_analog_value.py) - Clean analog sensor data
- [clean_digital_value.py](myems-cleaning/clean_digital_value.py) - Clean digital sensor data
- [clean_energy_value.py](myems-cleaning/clean_energy_value.py) - Clean energy meter data

---

### 7. myems-aggregation

**Purpose**: Data aggregation, billing calculation, and carbon emissions calculation service

**Technology Stack**:
- Python 3.10+
- MySQL Connector

**Key Responsibilities**:
- **Energy Aggregation**: Roll up hourly data to daily, monthly, yearly
- **Billing Calculation**: Apply tariff structures to consumption data
- **Carbon Emissions Calculation**: Apply emission factors to energy consumption
- **Cost Allocation**: Distribute costs to spaces, equipment, tenants
- **Hierarchical Rollup**: Aggregate data up organizational hierarchy
- **Tariff Processing**: Handle TOU, tiered, block rate, and custom tariffs

**Supported Aggregation Dimensions**:
- **By Time**: Hourly → Daily → Monthly → Yearly
- **By Space**: Room → Floor → Building → Site → Enterprise
- **By Equipment**: Individual → Combined → Department
- **By Tenant**: Tenant → Cost Center → Organization
- **By Category**: Energy type (electricity, water, gas, etc.)
- **By Item**: Energy sub-items (lighting, HVAC, power, etc.)

**Tariff Types Supported**:
- Simple flat rate
- Time-of-Use (TOU) with peak/off-peak/shoulder periods
- Tiered/stepped pricing
- Block rate pricing
- Demand charges
- Power factor penalties/rewards
- Seasonal rates

**Execution Frequency**: Every 1 hour (configurable)

**Docker Support**: ✅ Dockerfile provided

**Key Files**:
- [space_energy_input_category.py](myems-aggregation/space_energy_input_category.py) - Space energy aggregation
- [equipment_billing_input_category.py](myems-aggregation/equipment_billing_input_category.py) - Equipment billing
- [meter_billing.py](myems-aggregation/meter_billing.py) - Meter billing calculation
- [meter_carbon.py](myems-aggregation/meter_carbon.py) - Carbon emissions calculation
- [tariff.py](myems-aggregation/tariff.py) - Tariff calculation logic

---

### Supporting Infrastructure

#### Database (13 MySQL Databases)

All databases use **MySQL 8.0+** with utf8mb4 character set.

**Installation Scripts**:
- [database/install/](database/install/) - Initial database schema creation
- [database/upgrade/](database/upgrade/) - Migration scripts for version upgrades
- [database/demo-en/](database/demo-en/) - English demo data
- [database/demo-cn/](database/demo-cn/) - Chinese demo data

**Database Documentation**:
- [database/README.md](database/README.md) - Complete database design documentation (Chinese)

#### Documentation

- [CORE_FEATURES_AND_LOGIC.md](CORE_FEATURES_AND_LOGIC.md) - In-depth business logic and technical implementation guide
- [docs/images/](docs/images/) - Architecture diagrams and screenshots
- Individual service READMEs in each service directory

#### Others

- [others/](others/) - Additional tools, scripts, and utilities

---

## Getting Started

### Prerequisites

Before installing MyEMS, ensure you have the following software installed:

#### Required Software

| Software | Minimum Version | Purpose |
|----------|----------------|---------|
| **MySQL Server** | 8.0 | Database server |
| **Python** | 3.10 | Backend services runtime |
| **Node.js** | 17.0 | Frontend build tool |
| **Nginx** | 1.18.0 | Web server and reverse proxy |
| **Docker** (Optional) | 20.10+ | Containerized deployment |
| **Docker Compose** (Optional) | 1.29+ | Multi-container orchestration |

#### System Requirements

**Minimum (Development/Testing)**:
- CPU: 2 cores
- RAM: 4 GB
- Storage: 50 GB
- OS: Ubuntu 20.04+, Debian 10+, CentOS 7+, Windows 10+ (for development)

**Recommended (Production)**:
- CPU: 8 cores
- RAM: 16 GB
- Storage: 500 GB SSD (database growth: ~1 GB/month per 100 meters)
- OS: Ubuntu 22.04 LTS or Debian 11

#### Network Requirements

- Port 3306: MySQL database
- Port 8000: MyEMS API
- Port 80/443: Web interfaces (myems-web, myems-admin)
- Port 502: Modbus TCP devices (if directly connected)

---

