# MyEMS Complete Features Documentation

> **Comprehensive Feature Guide: From Authentication to End Goal**  
> Industry-Leading Open Source Energy Management System

[![Version](https://img.shields.io/badge/version-5.12.0-blue)](https://github.com/MyEMS/myems)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

---

## Table of Contents

1. [Overview](#overview)
2. [Authentication & Security](#1-authentication--security)
3. [User Management](#2-user-management)
4. [System Configuration](#3-system-configuration)
5. [Data Acquisition](#4-data-acquisition)
6. [Data Processing Pipeline](#5-data-processing-pipeline)
7. [Energy Management](#6-energy-management)
8. [Billing & Cost Management](#7-billing--cost-management)
9. [Carbon Emissions Tracking](#8-carbon-emissions-tracking)
10. [Advanced Energy Systems](#9-advanced-energy-systems)
11. [Reporting & Analytics](#10-reporting--analytics)
12. [Fault Detection & Diagnostics](#11-fault-detection--diagnostics)
13. [Integration & APIs](#12-integration--apis)
14. [Mobile & Notifications](#13-mobile--notifications)
15. [Advanced Features](#14-advanced-features)
16. [End Goals & Business Value](#15-end-goals--business-value)

---

## Overview

MyEMS is a comprehensive Energy Management System that provides end-to-end functionality for monitoring, analyzing, and optimizing energy consumption across enterprises. This document outlines every feature from initial user authentication through the complete energy management lifecycle to achieving business objectives.

**Core Purpose**: Enable organizations to achieve energy efficiency, cost reduction, carbon neutrality, and operational excellence through data-driven insights.

---

## 1. Authentication & Security

### 1.1 User Authentication

#### Login Mechanism
- **Username/Email-based Login**
  - Users can log in with username OR email address
  - Case-insensitive email matching
  - Password hashing using SHA-512 with unique salt per user
  - Failed login attempt tracking
  
- **Session Management**
  - Token-based authentication (SHA-512 tokens)
  - Configurable session expiration (default: configurable in config.py)
  - Single sign-on across web and admin interfaces
  - Session tracking in database (`tbl_sessions`)
  - Automatic session cleanup on logout

#### Password Security
- **Password Policy Enforcement**
  - Minimum/maximum length validation
  - Password expiration dates
  - Password history tracking (prevents reuse)
  - Secure password reset flow
  
- **Account Lockout Protection**
  - Configurable failed login threshold
  - Account locks after maximum failed attempts
  - Admin unlock capability
  - Automatic lockout status display

#### Account Management
- **Account Expiration**
  - Configurable account expiration dates
  - Automatic access denial for expired accounts
  - Grace period handling
  
- **Password Reset Flow**
  1. User requests password reset via email
  2. Verification code sent to registered email (6-digit code)
  3. Code valid for 1 hour (configurable)
  4. User enters code and sets new password
  5. All existing sessions invalidated

### 1.2 Authorization & Access Control

#### Role-Based Access Control (RBAC)
- **Admin Users**
  - Full system access
  - User management capabilities
  - System configuration rights
  - Read-only vs. full admin distinction
  
- **Regular Users**
  - Privilege-based access
  - Granular permissions per resource type
  - Multi-tenant data isolation
  
- **Privilege System**
  - Customizable privilege levels
  - Fine-grained permission sets
  - Association with user accounts

#### API Security
- **API Key Authentication**
  - Alternative to session tokens for integrations
  - SHA-512 hashed keys
  - Expiration date support
  - Key management (create, revoke, list)
  - Usage tracking per key
  
- **Request Validation**
  - USER-UUID header validation
  - TOKEN header validation
  - Session expiry checks on every request
  - Admin privilege verification for protected endpoints

### 1.3 Audit & Compliance

#### Activity Logging
- **User Activity Tracking**
  - All user actions logged (create, read, update, delete)
  - Timestamp and user identification
  - Resource type and ID tracking
  - Request body logging for modifications
  
- **Login Audit Trail**
  - Successful login events
  - Failed login attempts
  - Account lockout events
  - Password change history

#### Data Security
- **Data Isolation**
  - Multi-tenant logical separation
  - Cost center-based access control
  - Space hierarchy permissions
  - Equipment ownership validation

---

## 2. User Management

### 2.1 User Administration

#### User Creation & Management
- **User Profile**
  - Username (unique identifier)
  - Display name (friendly name)
  - Email address (unique, validated)
  - UUID (system-generated unique ID)
  - Admin status flag
  - Read-only admin flag
  - Privilege assignment
  
- **User Lifecycle**
  - Create new users with credentials
  - Update user information
  - Deactivate/delete users
  - Lock/unlock user accounts
  - Reset passwords (admin or self-service)

#### New User Registration
- **Self-Registration Flow**
  1. New user submits registration request
  2. Email verification code sent
  3. User confirms email with code
  4. Admin reviews and approves/rejects
  5. Account activated upon approval
  
- **Approval Workflow**
  - Pending user queue for admins
  - Review registration details
  - Approve or reject with reason
  - Automatic notification on approval

### 2.2 User Privileges

#### Privilege Configuration
- **Built-in Privileges**
  - System Administrator
  - Energy Manager
  - Facility Manager
  - Viewer
  - Custom privileges
  
- **Permission Granularity**
  - Per-module permissions
  - Resource-level access control
  - Action-based permissions (view, create, edit, delete)

### 2.3 User Notifications

#### Email Notifications
- **Email Message System**
  - Scheduled email delivery
  - Email template support
  - Status tracking (new, sent, failed)
  - Recipient management
  - Subject and body customization
  
- **Common Email Scenarios**
  - Password reset requests
  - Account approval notifications
  - System alerts and warnings
  - Report delivery
  - Energy threshold violations

#### In-App Messages
- **Web Messages**
  - Real-time notifications in web interface
  - Message status (new, read, acknowledged)
  - Priority levels
  - Batch operations (mark as read, delete)
  
- **WeChat Integration** (for Chinese users)
  - WeChat message delivery
  - Official account integration
  - Template message support

---

## 3. System Configuration

### 3.1 Organizational Hierarchy

#### Space Management
- **Hierarchical Structure**
  ```
  Enterprise
    └── Campus/Site
        └── Building
            └── Floor
                └── Space/Room
                    └── Work Area
  ```

- **Space Attributes**
  - Name and description
  - Area (square meters/feet)
  - Floor area
  - Number of occupants
  - Postal code and address
  - GPS coordinates (latitude/longitude)
  - Time zone
  - Parent-child relationships
  
- **Space Operations**
  - Create/edit/delete spaces
  - Clone spaces with configurations
  - Import/export space hierarchies
  - Associate meters and equipment
  - Define cost centers

#### Space Configuration Features
- **Child Space Management**
  - View all child spaces
  - Inherit configurations
  - Aggregate data from children
  
- **Tree Visualization**
  - Interactive space tree display
  - Expand/collapse branches
  - Quick navigation
  - Hierarchy validation

### 3.2 Equipment Management

#### Equipment Registry
- **Equipment Types**
  - HVAC systems (chillers, boilers, AHUs)
  - Lighting systems
  - Motors and drives
  - Compressors
  - Production equipment
  - Custom equipment categories
  
- **Equipment Attributes**
  - Name and UUID
  - Equipment type
  - Is input/output flag
  - Cost center assignment
  - Location (space association)
  - Capacity and rated power
  - Description and notes

#### Combined Equipment
- **Equipment Groups**
  - Combine multiple equipment into logical groups
  - Examples: "Total HVAC", "Building Lighting"
  - Aggregate energy consumption
  - Group-level reporting
  
- **Equipment Relationships**
  - Add/remove equipment from groups
  - Weighted aggregation support
  - Hierarchical equipment structures

#### Equipment Associations
- **Meter Assignments**
  - Associate physical meters
  - Assign offline meters
  - Link virtual meters
  - Multiple meters per equipment
  
- **Point Mapping**
  - Map data points to equipment
  - Real-time parameter monitoring
  - Sensor integration
  
- **Command Integration**
  - Control commands for equipment
  - On/off controls
  - Setpoint adjustments
  - Schedule-based automation

### 3.3 Meter Configuration

#### Physical Meters
- **Meter Types**
  - Electricity meters
  - Water meters
  - Gas meters
  - Thermal energy meters (heating/cooling)
  - BTU meters
  - Custom meter types
  
- **Meter Properties**
  - Name and UUID
  - Energy category (electricity, water, gas, etc.)
  - Energy item (consumption, demand, reactive power)
  - Master meter designation
  - Cost center assignment
  - Description
  
- **Submeter Hierarchy**
  - Define parent-child meter relationships
  - Validate submeter balance
  - Hierarchical energy accounting
  - Loss detection

#### Offline Meters
- **Manual Data Entry**
  - For non-connected meters
  - Bulk import via Excel
  - Historical data entry
  - Date range support
  
- **Offline Meter Features**
  - Same attributes as physical meters
  - File-based data import
  - Manual reading schedules
  - Audit trail for entries

#### Virtual Meters
- **Calculated Meters**
  - Formula-based calculations
  - Example: Total = Meter A + Meter B - Meter C
  - Mathematical expressions support (using SymPy)
  - Real-time calculation
  
- **Formula Examples**
  ```python
  # Summation
  expression = "x + y + z"
  
  # Difference
  expression = "gross - generation"
  
  # Weighted average
  expression = "(x * 0.7) + (y * 0.3)"
  
  # Complex formulas
  expression = "(x + y) / area * efficiency"
  ```

- **Virtual Meter Use Cases**
  - Aggregate consumption across multiple meters
  - Calculate net energy (consumption - generation)
  - Normalize by area or occupancy
  - Efficiency calculations

### 3.4 Data Source Configuration

#### Gateway Management
- **Modbus TCP Gateways**
  - Gateway name and token
  - IP address and port
  - Channel count
  - Last seen timestamp
  - Connection status monitoring
  
- **Gateway Operations**
  - Add/edit/delete gateways
  - Test connectivity (telnet)
  - View associated data sources
  - Clone configurations

#### Data Sources
- **Protocol Support**
  - Modbus TCP/IP
  - BACnet IP
  - OPC UA
  - MQTT
  - HTTP/REST APIs
  - Custom protocols
  
- **Data Source Attributes**
  - Name and UUID
  - Protocol type
  - Gateway assignment
  - Connection parameters
  - Polling interval (5-60 seconds typical)

#### Point Configuration
- **Point Definition**
  - Point name and UUID
  - Data source association
  - Address (register number, object ID, etc.)
  - Data type (integer, float, digital, string)
  - Unit of measurement
  - Multiplier/ratio
  - High/low limits
  
- **Point Types**
  - Analog inputs (temperature, pressure, flow)
  - Digital inputs (on/off, alarm states)
  - Energy accumulation (kWh, m³)
  - Power demand (kW, MW)
  
- **Point Association**
  - Map to meters
  - Link to equipment
  - Associate with spaces
  - Sensor connections

### 3.5 Cost Center & Tariff Management

#### Cost Centers
- **Cost Center Definition**
  - Name and UUID
  - External ID (for ERP integration)
  - Description
  - Parent cost center (hierarchy)
  
- **Tariff Assignments**
  - Associate tariffs with cost centers
  - Multiple tariffs per cost center
  - Different tariffs per energy category
  - Time-based tariff switching

#### Tariff Configuration
- **Tariff Types**
  1. **Simple Flat Rate**
     - Single price per unit
     - Example: $0.10/kWh
  
  2. **Time-of-Use (TOU)**
     - Peak hours pricing
     - Off-peak hours pricing
     - Shoulder period pricing
     - Hourly schedule definition
     - Weekday vs. weekend rates
  
  3. **Tiered/Stepped Pricing**
     - Multiple price tiers
     - Threshold-based pricing
     - Example:
       - First 1000 kWh: $0.08/kWh
       - Next 2000 kWh: $0.10/kWh
       - Above 3000 kWh: $0.12/kWh
  
  4. **Block Rate**
     - Fixed blocks of consumption
     - Declining block rates
     - Inclining block rates
  
  5. **Demand Charges**
     - Peak demand pricing
     - Ratchet clauses
     - Seasonal demand charges
  
  6. **Power Factor Penalties/Rewards**
     - Penalty for low power factor
     - Reward for high power factor
     - Threshold-based adjustments
  
  7. **Seasonal Rates**
     - Summer rates
     - Winter rates
     - Spring/Fall rates
     - Month-based activation

- **Tariff Parameters**
  - Unit price
  - Time schedule
  - Valid date ranges
  - Tax rates
  - Surcharges
  - Fixed charges

### 3.6 Energy Categories & Items

#### Energy Category Management
- **Built-in Categories**
  - Electricity
  - Water
  - Natural Gas
  - Heating
  - Cooling
  - Compressed Air
  - Fuel Oil
  - Propane
  
- **Custom Categories**
  - User-defined energy types
  - Unit specification
  - kgce (kilogram of coal equivalent) conversion
  - kgCO₂ emission factor

#### Energy Items
- **Item Types per Category**
  - **Electricity**: Consumption, Demand, Reactive Power, Apparent Power
  - **Water**: Volume, Flow Rate
  - **Gas**: Volume, Mass, Energy Content
  - **Thermal**: Energy (kWh), Power (kW), Temperature, Flow
  
- **Item Configuration**
  - Name and display name
  - Unit of measurement
  - Conversion factors
  - Aggregation rules

### 3.7 Working Calendars

#### Calendar Definition
- **Purpose**: Define working vs. non-working days for energy baseline calculations
  
- **Calendar Features**
  - Multiple calendars per organization
  - Annual schedule
  - Holiday definitions
  - Custom non-working day patterns
  
- **Non-Working Days**
  - Public holidays
  - Company holidays
  - Maintenance shutdowns
  - Special events
  
- **Calendar Association**
  - Link to spaces
  - Link to equipment
  - Link to tenants
  - Link to shopfloors

---

## 4. Data Acquisition

### 4.1 Physical Meter Data Collection

#### Modbus TCP Acquisition
- **Automatic Data Collection**
  - Service: `myems-modbus-tcp`
  - Polling intervals: 5-60 seconds (configurable per point)
  - Batch writing to database
  - In-memory buffering
  
- **Data Flow**
  ```
  Physical Meter
    ↓ (Modbus TCP)
  Gateway
    ↓ (Read registers)
  myems-modbus-tcp service
    ↓ (Apply calibration)
  Database (meter_instantaneous_data)
  ```

#### Register Reading
- **Supported Functions**
  - Function Code 03: Read Holding Registers
  - Function Code 04: Read Input Registers
  - Single and multiple register reads
  
- **Data Types**
  - INT16, UINT16
  - INT32, UINT32
  - FLOAT32
  - FLOAT64
  - Custom byte order handling (big-endian/little-endian)

#### Calibration & Processing
- **Calibration Factors**
  - Multiplier per point/meter
  - Example: Register value 12345 × 0.01 = 123.45 kWh
  - Unit conversion
  - Offset adjustments
  
- **Data Validation**
  - Range checking (high/low limits)
  - Meter reset detection (rollover handling)
  - Quality flags (is_bad indicator)

### 4.2 Offline Meter Data Import

#### Manual Data Entry
- **Input Methods**
  - Web interface form entry
  - Excel file bulk import
  - API-based data submission
  
- **Offline Meter File Format**
  - Excel template provided
  - Columns: Meter ID, Timestamp, Value
  - Date range validation
  - Bulk upload support

#### File Processing
- **Import Workflow**
  1. Upload Excel file via API/web interface
  2. System validates file format
  3. Data extracted and parsed
  4. Validation (meter exists, date ranges valid)
  5. Insert into offline meter tables
  6. Mark file as processed
  
- **Error Handling**
  - Duplicate timestamp detection
  - Invalid meter ID warnings
  - Data type validation
  - Detailed error reports

### 4.3 MQTT Integration

#### MQTT Protocol Support
- **Broker Connectivity**
  - Connect to MQTT brokers
  - Topic subscription
  - QoS level configuration
  - TLS/SSL support
  
- **Message Processing**
  - JSON payload parsing
  - Topic-to-meter mapping
  - Real-time data ingestion
  - Timestamp handling

### 4.4 Real-Time Monitoring

#### Point Real-Time Data
- **Live Value Display**
  - Current point values
  - Last update timestamp
  - Update frequency
  - Value trends (increasing/decreasing)
  
- **Set Value Commands**
  - Write setpoints to devices
  - Control outputs
  - Acknowledge alarms

#### Meter Real-Time Data
- **Instantaneous Meter View**
  - Latest meter readings
  - Current power demand
  - 48-hour data retention in real-time table
  - Automatic archival to historical tables

---

## 5. Data Processing Pipeline

### 5.1 Data Normalization (myems-normalization)

#### Service Overview
- **Execution**: Every 1 hour (configurable)
- **Purpose**: Convert raw meter data into standardized energy consumption values

#### Physical Meter Processing
- **Processing Steps**
  1. Retrieve last processed timestamp per meter
  2. Fetch raw data since last processing
  3. Calculate incremental consumption (current - previous)
  4. Handle meter resets and rollovers
  5. Apply calibration factors
  6. Store hourly aggregated values
  
- **Calculations**
  ```python
  # Incremental consumption
  consumption = current_reading - previous_reading
  
  # Handle meter reset (e.g., 99999 → 0)
  if current_reading < previous_reading:
      consumption = (meter_max - previous_reading) + current_reading
  
  # Apply calibration
  normalized_value = consumption * calibration_factor
  ```

#### Virtual Meter Calculation
- **Formula Evaluation**
  - Real-time formula execution
  - Uses underlying physical meters
  - SymPy for mathematical expressions
  - Supports: +, -, *, /, sqrt, power, etc.
  
- **Example Virtual Meter**
  ```python
  # Total HVAC = Chiller + Cooling Tower + AHU
  meters = {"x": chiller_kwh, "y": tower_kwh, "z": ahu_kwh}
  expression = "x + y + z"
  result = evaluate(expression, meters)
  ```

#### Offline Meter Processing
- **File Import Processing**
  - Scan for unprocessed offline meter files
  - Parse Excel files
  - Extract meter readings
  - Validate and insert into historical database
  - Mark files as processed

### 5.2 Data Cleaning (myems-cleaning)

#### Service Overview
- **Execution**: Every 1 hour (configurable)
- **Purpose**: Ensure data quality, remove anomalies, fill gaps

#### Anomaly Detection
- **Outlier Identification**
  - Statistical methods (Z-score, IQR)
  - Spike detection (> 150% of average)
  - Sudden drop detection
  - Unrealistic values (negative consumption)
  
- **Handling Strategies**
  - Flag as bad data (is_bad = TRUE)
  - Remove from aggregations
  - Interpolate missing values
  - Generate alerts for investigation

#### Data Gap Filling
- **Gap Detection**
  - Identify missing timestamps
  - Expected vs. actual data points
  
- **Interpolation Methods**
  - Linear interpolation
  - Previous value carry-forward
  - Weighted averages
  - Seasonal pattern matching

#### Duplicate Removal
- **Deduplication**
  - Identify duplicate timestamps
  - Keep first/last entry (configurable)
  - Log duplicate events
  - Prevent aggregation errors

### 5.3 Data Aggregation (myems-aggregation)

#### Service Overview
- **Execution**: Every 1 hour (configurable)
- **Purpose**: Roll up data for billing, carbon, energy analysis

#### Energy Aggregation
- **Aggregation Dimensions**
  - By Time: Hourly → Daily → Monthly → Yearly
  - By Space: Room → Floor → Building → Campus
  - By Equipment: Individual → Combined → Department
  - By Tenant: Tenant → Cost Center
  - By Category: Electricity, Water, Gas, etc.
  - By Item: Consumption, Demand, etc.
  
- **Aggregation Process**
  ```
  For each entity (space, equipment, tenant):
    1. Get list of associated meters
    2. Retrieve meter data for period
    3. Sum consumption by category
    4. Calculate min/max/avg power
    5. Store aggregated results
  ```

#### Billing Calculation
- **Process Flow**
  1. Retrieve meter consumption for billing period
  2. Fetch applicable tariff for cost center
  3. Apply tariff rules (TOU, tiers, demand charges)
  4. Calculate cost breakdown
  5. Store in billing database
  
- **Tariff Application Logic**
  ```python
  def calculate_tou_billing(consumption_by_hour, tariff):
      total_cost = 0
      for hour, kwh in consumption_by_hour:
          if is_peak_hour(hour, tariff.schedule):
              cost = kwh * tariff.peak_rate
          else:
              cost = kwh * tariff.offpeak_rate
          total_cost += cost
      return total_cost
  ```

#### Carbon Calculation
- **Emission Calculation**
  ```
  Carbon Emissions = Energy Consumption × Emission Factor
  ```
  
- **Emission Factors**
  - Electricity: 0.45 kg CO₂/kWh (varies by grid)
  - Natural Gas: 1.89 kg CO₂/m³
  - Water: 0.28 kg CO₂/m³ (treatment + distribution)
  - Custom factors per energy category
  
- **Aggregation Levels**
  - Space carbon footprint
  - Equipment carbon footprint
  - Tenant carbon footprint
  - Enterprise-wide emissions

#### Incremental Processing
- **Efficiency Strategy**
  - Track last processed timestamp per entity
  - Only process new data since last run
  - Avoid reprocessing historical data
  - Handle late-arriving data
  
- **Performance**
  - Processes 1000+ meters in minutes
  - Parallel processing support (18 processes)
  - Database indexing optimization
  - Redis caching for lookups

---

## 6. Energy Management

### 6.1 Hierarchical Energy Tracking

#### Multi-Level Monitoring
- **Space-Based Tracking**
  - Energy consumption by room
  - Floor-level aggregation
  - Building-level totals
  - Campus/enterprise rollup
  
- **Equipment-Based Tracking**
  - Individual equipment monitoring
  - Equipment group aggregation
  - System-level analysis (all HVAC, all lighting)
  
- **Tenant-Based Tracking**
  - Per-tenant consumption
  - Cost center allocation
  - Chargeback billing

#### Energy Flow Visualization
- **Energy Flow Diagrams**
  - Interactive Sankey diagrams
  - Source-to-end-use visualization
  - Loss identification
  - Sub-metering balance
  
- **Diagram Components**
  - Energy sources (grid, solar, etc.)
  - Distribution nodes
  - End-use categories
  - Energy losses
  - Flow quantities and percentages

### 6.2 Energy Analysis Features

#### Consumption Analysis
- **Historical Trends**
  - Hourly, daily, monthly, yearly trends
  - Year-over-year comparisons
  - Same-period-last-year analysis
  - Seasonal pattern identification
  
- **Peak Analysis**
  - Peak demand tracking
  - Peak hour identification
  - Demand limiting opportunities
  - Load factor analysis

#### Comparison Reports
- **Multi-Entity Comparison**
  - Compare spaces (buildings, floors, rooms)
  - Compare equipment (similar types)
  - Compare time periods
  - Benchmark against baselines
  
- **Performance Metrics**
  - Energy use intensity (EUI): kWh/m²/year
  - Energy per occupant
  - Energy per production unit
  - Equipment efficiency

#### Load Profiling
- **Load Curves**
  - 24-hour load profiles
  - Weekly patterns
  - Monthly patterns
  - Base load identification
  
- **Load Factor Calculation**
  ```
  Load Factor = Average Load / Peak Load × 100%
  ```
  - Higher load factor = more efficient energy use
  - Identify improvement opportunities

### 6.3 Energy Efficiency

#### Efficiency Metrics
- **Equipment Efficiency**
  - Output/Input ratios
  - COP (Coefficient of Performance) for HVAC
  - Power factor tracking
  - Efficiency degradation over time
  
- **Space Efficiency**
  - EUI benchmarking
  - Occupancy-normalized consumption
  - Weather-normalized consumption

#### Energy Savings Tracking
- **Savings Calculation**
  ```
  Savings = Baseline - Actual
  
  Where:
  - Baseline = Expected consumption without improvements
  - Actual = Measured consumption
  ```
  
- **Baseline Methods**
  - Historical average
  - Regression models (weather-normalized)
  - Engineering calculations
  - Industry standards

#### Energy Plans & Targets
- **Target Setting**
  - Annual energy reduction targets
  - Monthly savings goals
  - Departmental targets
  - Equipment-specific targets
  
- **Plan vs. Actual**
  - Compare planned vs. actual consumption
  - Variance analysis
  - Progress tracking
  - Early warning for target misses

### 6.4 Energy Production (Renewable Energy)

#### Photovoltaic (PV) Power Stations
- **PV Station Management**
  - Multiple PV stations per enterprise
  - Inverter monitoring
  - Panel group tracking
  - Grid connectivity status
  
- **PV Metrics**
  - Total generation (kWh)
  - Instantaneous power (kW)
  - Grid export vs. self-consumption
  - Performance ratio
  - Capacity factor
  - Revenue from generation
  
- **PV Reports**
  - Daily/monthly/yearly generation
  - Inverter-level analysis
  - Grid vs. load comparison
  - Revenue reporting

#### Wind Farms
- **Wind Turbine Monitoring**
  - Individual turbine tracking
  - Farm-level aggregation
  - Wind speed correlation
  - Power curve analysis
  
- **Wind Metrics**
  - Generation by turbine
  - Availability factor
  - Capacity factor
  - Curtailment tracking

---

## 7. Billing & Cost Management

### 7.1 Automated Billing Calculation

#### Multi-Tariff Support
- **Tariff Application**
  - Automatic tariff selection by cost center
  - Time-based tariff switching
  - Seasonal rate application
  - Complex tariff logic (TOU, tiered, demand)
  
- **Billing Components**
  - Energy charges (based on consumption)
  - Demand charges (based on peak power)
  - Power factor adjustments
  - Fixed charges (monthly fees)
  - Taxes and surcharges

#### Cost Allocation
- **Hierarchical Cost Distribution**
  ```
  Enterprise Total
    ├─ Building A: 40% → $X
    ├─ Building B: 35% → $Y
    └─ Building C: 25% → $Z
  
  For Building A:
    ├─ Tenant 1: 60% → $M
    ├─ Tenant 2: 25% → $N
    └─ Tenant 3: 15% → $O
  ```
  
- **Allocation Methods**
  - Proportional by consumption
  - Fixed percentage allocation
  - Area-based allocation
  - Custom allocation rules

#### Cost Center Management
- **Cost Center Hierarchy**
  - Parent-child relationships
  - Multi-level cost centers
  - Department/location-based
  - Project/activity-based
  
- **Cost Tracking**
  - Real-time cost accumulation
  - Budget vs. actual tracking
  - Variance alerts
  - Forecasting

### 7.2 Billing Reports

#### Detailed Billing Reports
- **Space Billing**
  - Cost by space (building, floor, room)
  - Breakdown by energy category
  - Time period summaries
  - Trend analysis
  
- **Equipment Billing**
  - Cost per equipment/equipment group
  - Operating cost analysis
  - Efficiency-based cost allocation
  
- **Tenant Billing**
  - Tenant invoice generation
  - Detailed consumption breakdown
  - Multi-site tenant billing
  - Export to accounting systems

#### Cost Comparison
- **Benchmarking**
  - Compare costs across spaces
  - Compare costs across time periods
  - Industry benchmark comparisons
  - Identify high-cost areas
  
- **Cost Reduction Opportunities**
  - Highlight cost anomalies
  - Suggest tariff optimizations
  - Identify inefficient equipment
  - Recommend energy savings measures

### 7.3 Budget Management

#### Budget Planning
- **Budget Setting**
  - Annual energy budgets by cost center
  - Monthly budget allocation
  - Seasonal budget adjustments
  - Multi-year planning
  
- **Budget Import**
  - Excel-based budget import
  - Bulk budget updates
  - Historical budget tracking

#### Budget Tracking
- **Real-Time Monitoring**
  - Budget vs. actual spend
  - Percentage of budget consumed
  - Remaining budget
  - Projected year-end spend
  
- **Alerts & Notifications**
  - Budget threshold alerts (80%, 90%, 100%)
  - Overspend notifications
  - Weekly/monthly budget summaries
  - Cost spike alerts

---

## 8. Carbon Emissions Tracking

### 8.1 Carbon Calculation

#### Emission Factor Management
- **Global Emission Factors**
  - Configurable per energy category
  - Time-based factors (seasonal grid mix)
  - Geographic-based factors
  - Source tracking (grid, renewable, fossil)
  
- **Standard Emission Factors**
  | Energy Type | Emission Factor | Unit | Source |
  |-------------|----------------|------|--------|
  | Grid Electricity | 0.45 | kg CO₂/kWh | National average |
  | Natural Gas | 1.89 | kg CO₂/m³ | Combustion factor |
  | Water | 0.28 | kg CO₂/m³ | Treatment + pumping |
  | District Heating | 0.08 | kg CO₂/kWh | Heat network |
  | District Cooling | 0.12 | kg CO₂/kWh | Chiller efficiency |

#### Emission Calculation
- **Calculation Formula**
  ```
  CO₂ Emissions (kg) = Energy Consumption × Emission Factor
  
  Example:
  Electricity: 1000 kWh × 0.45 kg CO₂/kWh = 450 kg CO₂
  ```
  
- **Aggregation Levels**
  - Per meter
  - Per space (room, floor, building)
  - Per equipment/equipment group
  - Per tenant
  - Enterprise-wide

### 8.2 Carbon Reporting

#### Scope-Based Reporting
- **Scope 1 Emissions**
  - Direct emissions from owned sources
  - Natural gas combustion
  - Company vehicles
  - On-site generators
  
- **Scope 2 Emissions**
  - Indirect emissions from purchased energy
  - Electricity from grid
  - District heating/cooling
  - Steam purchase
  
- **Scope 3 Emissions** (Optional)
  - Upstream/downstream emissions
  - Water consumption emissions
  - Waste disposal emissions
  - Business travel

#### Carbon Reports
- **Space Carbon Footprint**
  - Total emissions by space
  - Breakdown by energy source
  - Emissions intensity (kg CO₂/m²)
  - Trend analysis
  
- **Equipment Carbon Footprint**
  - Emissions per equipment
  - Equipment efficiency ratings
  - Carbon-intensive equipment identification
  
- **Tenant Carbon Reporting**
  - Tenant-specific emissions
  - Multi-site tenant carbon tracking
  - Carbon invoice/reporting

### 8.3 Sustainability KPIs

#### Key Performance Indicators
- **Absolute Metrics**
  - Total CO₂ emissions (kg, tonnes)
  - Year-over-year emission changes
  - Monthly emission trends
  
- **Intensity Metrics**
  - Emissions per square meter (kg CO₂/m²)
  - Emissions per occupant (kg CO₂/person)
  - Emissions per production unit
  - Emissions per revenue dollar
  
- **Reduction Tracking**
  - Emission reduction percentage
  - Carbon saved vs. baseline
  - Progress toward carbon neutrality goals

#### Carbon Neutrality Planning
- **Carbon Reduction Targets**
  - Annual reduction goals (e.g., 5% per year)
  - Net-zero target dates
  - Interim milestones
  
- **Offset Tracking**
  - Renewable energy generation credits
  - Carbon offset purchases
  - Net carbon calculation (emissions - offsets)

---

## 9. Advanced Energy Systems

### 9.1 Energy Storage Power Stations

#### Battery Energy Storage Systems (BESS)
- **Station Management**
  - Multiple stations per enterprise
  - Container-based storage units
  - Battery management system (BMS) integration
  - Power conversion system (PCS) monitoring
  
- **Key Components**
  - Battery containers (multiple per station)
  - DC/DC converters
  - PCS (Power Conversion System)
  - Grid connection points
  - Load monitoring
  - HVAC systems for containers
  - Fire control systems
  - Static transfer switches (STS)

#### BESS Monitoring
- **Real-Time Data**
  - State of charge (SOC)
  - Battery voltage, current, temperature
  - Power flow (charging/discharging)
  - Grid frequency and voltage
  - System efficiency
  
- **Operational Metrics**
  - Charge/discharge cycles
  - Round-trip efficiency
  - Battery health status
  - Temperature management
  - Alarm status

#### BESS Reports
- **Energy Reports**
  - Charging energy (kWh)
  - Discharging energy (kWh)
  - Grid import/export
  - Self-consumption tracking
  
- **Revenue Reports**
  - Arbitrage revenue (buy low, sell high)
  - Demand charge reduction savings
  - Frequency regulation revenue
  - Capacity payments
  
- **Performance Reports**
  - Efficiency trends
  - Degradation analysis
  - Cycle life tracking
  - Maintenance predictions

### 9.2 Microgrids

#### Microgrid Management
- **Microgrid Components**
  - PV generation
  - Battery storage
  - Grid connection
  - Loads
  - EV chargers
  - Generators (backup)
  - Heat pumps
  - Power conversion systems
  
- **Control Modes**
  - Grid-connected mode
  - Island mode (off-grid)
  - Peak shaving mode
  - Self-consumption mode
  - Arbitrage mode

#### Microgrid Monitoring
- **Energy Flow Tracking**
  - PV generation
  - Battery charge/discharge
  - Grid buy/sell
  - Load consumption
  - EV charging
  - Generator operation
  
- **Real-Time Dashboard**
  - System topology view
  - Power flow diagram
  - Component status indicators
  - Alarm and event log
  - Control mode display

#### Microgrid Reports
- **Energy Analysis**
  - Self-sufficiency ratio (solar + battery / total load)
  - Grid independence level
  - Renewable energy percentage
  - Energy balance by component
  
- **Financial Reports**
  - Grid cost savings
  - Energy arbitrage revenue
  - Demand charge avoidance
  - Total cost of ownership
  
- **Carbon Reports**
  - Avoided emissions (vs. grid-only)
  - Renewable energy credits
  - Carbon footprint reduction

### 9.3 EV Charging Stations

#### Charging Station Management
- **Station Configuration**
  - Multiple charging stations
  - Charger types (Level 1, 2, DC fast)
  - Power capacity per charger
  - Location mapping
  
- **Charging Session Tracking**
  - Session start/end times
  - Energy delivered per session
  - Charging power profile
  - User identification
  - Payment processing integration

#### Load Management
- **Smart Charging**
  - Load balancing across chargers
  - Peak demand limiting
  - Time-of-use optimization
  - Grid capacity management
  
- **V2G (Vehicle-to-Grid)**
  - Bi-directional charging support
  - Grid services (frequency regulation)
  - Emergency backup power
  - Battery sharing for microgrids

### 9.4 Virtual Power Plants (VPP)

#### VPP Aggregation
- **Distributed Energy Resources (DER) Aggregation**
  - Multiple microgrids
  - PV systems
  - Battery storage
  - Controllable loads
  - EV chargers
  
- **Centralized Control**
  - Aggregate capacity management
  - Coordinated dispatch
  - Grid services provision
  - Revenue optimization

#### VPP Services
- **Grid Services**
  - Frequency regulation
  - Voltage support
  - Demand response
  - Capacity reserves
  
- **Market Participation**
  - Energy market trading
  - Ancillary services
  - Capacity market participation
  - Revenue distribution to participants

---

## 10. Reporting & Analytics

### 10.1 Pre-Built Reports (100+ Reports)

#### Space Reports
- **Energy Reports**
  - Space Energy by Category (electricity, water, gas)
  - Space Energy by Item (consumption, demand)
  - Space Comparison (multiple spaces)
  - Space Output (production-related)
  
- **Cost Reports**
  - Space Cost (total cost breakdown)
  - Space Cost by Category
  - Space Cost Comparison
  
- **Carbon Reports**
  - Space Carbon Footprint
  - Space Carbon by Category
  - Space Carbon Comparison
  
- **Additional**
  - Space Load Profile
  - Space Efficiency (EUI, benchmarking)
  - Space Income (for rental properties)
  - Space Savings (vs. baseline)
  - Space Plan (target vs. actual)
  - Space Statistics (summary KPIs)
  - Space Prediction (forecasting)
  - Space Environment Monitor (temperature, humidity)

#### Equipment Reports
- **Energy Reports**
  - Equipment Energy by Category
  - Equipment Energy by Item
  - Equipment Batch (multiple equipment)
  - Equipment Comparison
  - Equipment Tracking (real-time)
  
- **Cost Reports**
  - Equipment Cost
  - Equipment Cost by Category
  
- **Carbon Reports**
  - Equipment Carbon
  - Equipment Carbon Comparison
  
- **Performance Reports**
  - Equipment Efficiency
  - Equipment Load Profile
  - Equipment Output
  - Equipment Income
  - Equipment Savings
  - Equipment Plan
  - Equipment Statistics

#### Meter Reports
- **Basic Reports**
  - Meter Energy (consumption)
  - Meter Cost
  - Meter Carbon
  - Meter Batch (multiple meters)
  - Meter Comparison
  
- **Advanced Reports**
  - Meter Real-Time (live data)
  - Meter Trend (historical trends)
  - Meter Tracking (time-series)
  - Meter Savings
  - Meter Plan
  - Meter Submeters Balance (parent-child validation)

#### Tenant Reports
- **Tenant Billing**
  - Tenant Bill (invoice generation)
  - Tenant Cost Breakdown
  - Tenant Cost by Category
  
- **Tenant Energy**
  - Tenant Energy by Category
  - Tenant Energy by Item
  - Tenant Batch
  - Tenant Comparison
  
- **Tenant Carbon**
  - Tenant Carbon Footprint
  - Tenant Carbon Comparison
  
- **Additional**
  - Tenant Load Profile
  - Tenant Savings
  - Tenant Plan
  - Tenant Statistics
  - Tenant Dashboard

#### Store Reports (for Retail/Multi-Store Operations)
- Store Energy by Category
- Store Cost
- Store Carbon
- Store Dashboard
- Store Batch
- Store Comparison
- Store Load Profile
- Store Savings
- Store Plan
- Store Statistics

#### Shopfloor Reports (for Manufacturing)
- Shopfloor Energy by Category
- Shopfloor Cost
- Shopfloor Carbon
- Shopfloor Dashboard
- Shopfloor Batch
- Shopfloor Comparison
- Shopfloor Load Profile
- Shopfloor Savings
- Shopfloor Plan
- Shopfloor Statistics

### 10.2 Dashboard & Real-Time Displays

#### Main Dashboard
- **Real-Time KPIs**
  - Current power demand (kW)
  - Today's consumption (kWh)
  - Today's cost ($)
  - Today's carbon emissions (kg CO₂)
  - Comparison to yesterday/last week/last year
  
- **Visual Components**
  - Energy flow Sankey diagram
  - Real-time trend charts
  - Top energy consumers (equipment/spaces)
  - Alerts and warnings panel
  - System status indicators

#### Specialized Dashboards
- **Energy Storage Dashboard**
  - Battery SOC gauges
  - Charge/discharge power
  - Grid interaction
  - Revenue tracking
  
- **PV Station Dashboard**
  - Current generation
  - Today's generation
  - Inverter status
  - Grid export/import
  - Revenue counter
  
- **Microgrid Dashboard**
  - System topology
  - Power flows
  - Component status
  - Control mode
  - Self-sufficiency ratio
  
- **Tenant Dashboard**
  - Tenant-specific consumption
  - Cost accumulation
  - Budget tracking
  - Comparison to other tenants

### 10.3 Report Export & Scheduling

#### Export Formats
- **Excel Export**
  - Formatted Excel workbooks
  - Multiple sheets per report
  - Charts and graphs included
  - Raw data tables
  
- **PDF Export**
  - Professional report layout
  - Company branding support
  - Charts and graphs
  - Summary tables
  
- **Image Export**
  - PNG/JPEG format
  - Chart screenshots
  - Dashboard snapshots
  - For presentations and documents

#### Report Scheduling
- **Automated Report Generation**
  - Daily, weekly, monthly schedules
  - Custom report intervals
  - Email delivery to recipients
  - Batch report generation
  
- **Report Distribution**
  - Email distribution lists
  - WeChat delivery (for Chinese users)
  - Web message notifications
  - File storage for archival

### 10.4 Advanced Analytics

#### Predictive Analytics
- **Energy Forecasting**
  - Short-term forecasting (hours/days)
  - Medium-term forecasting (weeks/months)
  - Weather-adjusted forecasts
  - Machine learning models
  
- **Anomaly Detection**
  - Automated anomaly identification
  - Pattern deviation alerts
  - Equipment failure prediction
  - Consumption spike detection

#### Benchmarking
- **Internal Benchmarking**
  - Compare spaces within enterprise
  - Compare equipment of same type
  - Compare time periods
  
- **External Benchmarking**
  - Industry standard comparisons
  - Building rating systems (LEED, BREEAM)
  - Regional averages
  - Best-in-class targets

#### Custom Reports (Advanced Report Builder)
- **Report Builder Features**
  - Drag-and-drop interface
  - Custom SQL queries
  - Multiple data sources
  - Custom calculations
  - Chart type selection
  - Layout customization
  
- **Use Cases**
  - Executive summaries
  - Regulatory compliance reports
  - Custom KPI dashboards
  - Specialized analysis

---

## 11. Fault Detection & Diagnostics

### 11.1 Fault Detection Features

#### Real-Time Monitoring
- **Continuous Monitoring**
  - All meter points monitored
  - Equipment parameters tracked
  - Sensor data analyzed
  - Control system status
  
- **Alarm Types**
  - High/low value alarms
  - Rate-of-change alarms
  - Equipment status alarms
  - Communication failure alarms

#### Fault Detection Rules
- **Rule Configuration**
  - Define fault conditions
  - Set threshold values
  - Configure alarm priorities
  - Define response actions
  
- **Rule Types**
  - Simple threshold (e.g., temperature > 80°C)
  - Range check (e.g., 0 < power factor < 1)
  - Rate of change (e.g., flow increase > 20%/hr)
  - Pattern deviation (vs. baseline)
  - Multi-condition logic (AND/OR/NOT)

#### Equipment-Specific FDD
- **HVAC Fault Detection**
  - Refrigerant leak detection (COP degradation)
  - Air filter clogging (pressure drop increase)
  - Economizer failure (incorrect operation)
  - Simultaneous heating and cooling
  - Excessive runtime
  
- **Lighting Fault Detection**
  - Bulb failures (power drop)
  - Ballast failures
  - Excessive operating hours
  
- **Motor Fault Detection**
  - Vibration anomalies
  - Current imbalance
  - Overheating
  - Bearing wear

### 11.2 Diagnostics & Analysis

#### Root Cause Analysis
- **Fault Investigation Tools**
  - Historical trend review
  - Correlation analysis
  - Event timeline
  - Related alarm grouping
  
- **Diagnostic Reports**
  - Fault summary report
  - Equipment reliability report
  - Mean time between failures (MTBF)
  - Mean time to repair (MTTR)

#### Performance Degradation Tracking
- **Efficiency Monitoring**
  - Equipment efficiency trends
  - Performance benchmarking
  - Degradation rate calculation
  - Maintenance prediction
  
- **Indicators of Degradation**
  - Increasing energy consumption
  - Decreasing output
  - Rising operating temperatures
  - Longer cycle times

### 11.3 Maintenance Management

#### Work Order Generation
- **Automatic Work Orders**
  - Triggered by fault detection
  - Scheduled preventive maintenance
  - Condition-based maintenance
  
- **Work Order Attributes**
  - Equipment/location identification
  - Fault description
  - Priority level
  - Estimated repair time
  - Assigned technician
  - Parts required

#### Maintenance Tracking
- **Work Order Lifecycle**
  1. Created (fault detected or scheduled)
  2. Assigned (to technician)
  3. In Progress (work started)
  4. Completed (work finished)
  5. Verified (performance confirmed)
  6. Closed
  
- **Maintenance Analytics**
  - Maintenance costs per equipment
  - Response time tracking
  - Completion rate
  - Recurring fault analysis
  - Equipment downtime

---

## 12. Integration & APIs

### 12.1 RESTful API

#### API Overview
- **Technology**: Falcon framework (Python)
- **Protocol**: HTTP/HTTPS
- **Format**: JSON
- **Authentication**: Token-based or API keys
- **Port**: 8000 (default)

#### API Categories
- **Configuration APIs** (100+ endpoints)
  - Spaces: `/v1/spaces`, `/v1/spaces/{id}`
  - Equipment: `/v1/equipments`, `/v1/equipments/{id}`
  - Meters: `/v1/meters`, `/v1/meters/{id}`
  - Cost Centers: `/v1/costcenters`, `/v1/costcenters/{id}`
  - Tariffs: `/v1/tariffs`, `/v1/tariffs/{id}`
  
- **Data Query APIs**
  - Energy data: `/v1/meters/{id}/energy/hourly`
  - Billing data: `/v1/meters/{id}/billing/monthly`
  - Carbon data: `/v1/meters/{id}/carbon/daily`
  - Real-time data: `/v1/meters/{id}/realtime`
  
- **Report APIs**
  - Generate reports: `/v1/reports/{report_type}`
  - Download reports: `/v1/reports/{report_id}/download`
  
- **Control APIs**
  - Equipment commands: `/v1/equipments/{id}/command`
  - Point setvalues: `/v1/points/{id}/setvalue`

#### API Features
- **CRUD Operations**
  - Create (POST)
  - Read (GET)
  - Update (PUT)
  - Delete (DELETE)
  
- **Query Parameters**
  - Filtering (e.g., `?q=search_term`)
  - Pagination (e.g., `?page=1&limit=100`)
  - Date ranges (e.g., `?startdatetime=...&enddatetime=...`)
  - Language selection (e.g., `?language=en`)

- **Response Formats**
  ```json
  {
    "status": "success",
    "data": {
      "id": 123,
      "name": "Building A Meter",
      "value": 1234.56
    },
    "message": null
  }
  ```

### 12.2 Integration Protocols

#### Modbus TCP/IP
- **Master/Client Mode**
  - Connect to Modbus devices as master
  - Read holding/input registers
  - Support for multiple slaves per gateway
  
- **Slave/Server Mode** (for integration)
  - Expose MyEMS data as Modbus slave
  - Allow SCADA/BMS systems to read data
  - Configurable register mapping

#### BACnet IP
- **BACnet Client**
  - Discover BACnet devices
  - Read object properties
  - Subscribe to COV (Change of Value)
  
- **BACnet Server** (for integration)
  - Expose meters as BACnet objects
  - Support for BACnet/IP protocol
  - Standard object types (Analog Input, Binary Input)

#### MQTT
- **MQTT Publisher**
  - Publish real-time data to topics
  - Configurable topic structure
  - QoS levels supported
  
- **MQTT Subscriber**
  - Subscribe to external MQTT topics
  - Ingest data from IoT devices
  - JSON payload parsing

#### OPC UA
- **OPC UA Client**
  - Connect to OPC UA servers
  - Browse node tree
  - Subscribe to data changes
  
- **OPC UA Server** (for integration)
  - Expose MyEMS data as OPC UA server
  - Standard information model
  - Security certificate support

### 12.3 Third-Party Integrations

#### ERP Integration
- **Data Export**
  - Energy cost data to accounting systems
  - Billing data for invoicing
  - Budget vs. actual reports
  
- **Supported ERPs**
  - SAP integration
  - Oracle ERP Cloud
  - Microsoft Dynamics
  - Custom ERP via APIs

#### BMS/SCADA Integration
- **Data Exchange**
  - Import BMS point data
  - Export energy analytics back to BMS
  - Alarm forwarding
  
- **Control Integration**
  - Send control commands to BMS
  - Receive setpoints from BMS
  - Coordinated control strategies

#### Cloud Platforms
- **Cloud Data Export**
  - AWS IoT Core
  - Azure IoT Hub
  - Google Cloud IoT
  
- **Data Analytics**
  - Stream data to cloud analytics platforms
  - Machine learning integration
  - Big data processing

---

## 13. Mobile & Notifications

### 13.1 Mobile Access

#### Web-Based Mobile Interface
- **Responsive Design**
  - Fully responsive web UI
  - Works on smartphones and tablets
  - Touch-optimized controls
  - Adaptive layouts
  
- **Mobile Features**
  - Dashboard view
  - Real-time monitoring
  - Alerts and notifications
  - Quick reports
  - Equipment status

### 13.2 Notification System

#### Email Notifications
- **Automated Emails**
  - Scheduled report delivery
  - Alarm notifications
  - Threshold violation alerts
  - System status updates
  
- **Email Configuration**
  - SMTP server settings
  - Email templates
  - Recipient lists
  - Send schedules

#### WeChat Notifications (for Chinese Users)
- **WeChat Official Account**
  - Template messages
  - Real-time alerts
  - Interactive menus
  - Report sharing
  
- **Notification Types**
  - Energy alerts
  - Cost warnings
  - Equipment alarms
  - Daily summaries

#### Web Messages
- **In-App Notifications**
  - Real-time message display
  - Notification bell icon
  - Unread count badges
  - Message history
  
- **Message Types**
  - System announcements
  - Alarm notifications
  - Task assignments
  - Report completion

#### SMS/Text Messages
- **SMS Alerts**
  - Critical alarm notifications
  - System down alerts
  - Urgent maintenance notifications
  
- **SMS Gateway Integration**
  - Configurable SMS provider
  - Multi-recipient support
  - Message templates

---

## 14. Advanced Features

### 14.1 Distribution System Monitoring

#### Electrical Distribution
- **Distribution System Components**
  - Main distribution boards
  - Sub-distribution panels
  - Distribution circuits
  - Circuit breakers
  - Transformers
  
- **Circuit Monitoring**
  - Current per circuit
  - Voltage levels
  - Power factor
  - Circuit loading (% of capacity)
  - Unbalance detection

#### Distribution Visualization
- **Single-Line Diagrams**
  - Interactive diagrams
  - Real-time data overlays
  - Color-coded status indicators
  - Alarm highlighting
  
- **Distribution Reports**
  - Load analysis by circuit
  - Loss calculations
  - Capacity utilization
  - Power quality metrics

### 14.2 Power Quality Monitoring

#### Power Quality Metrics
- **Voltage Quality**
  - Voltage sags
  - Voltage swells
  - Voltage unbalance
  - Harmonic distortion (THD)
  
- **Current Quality**
  - Current unbalance
  - Harmonic content
  - Neutral current
  
- **Frequency**
  - Frequency deviation
  - Frequency stability

#### Power Quality Reports
- **PQ Analysis**
  - Event detection and classification
  - PQ indices calculation
  - Compliance with standards (IEEE 519, IEC 61000)
  - Root cause analysis
  
- **Power Quality Improvement**
  - Identify PQ issues
  - Recommend mitigation measures
  - Track improvement over time

### 14.3 Production Integration (for Manufacturing)

#### Production Data Integration
- **Production Metrics**
  - Production output (units)
  - Shift schedules
  - Machine operating hours
  - Downtime tracking
  
- **Energy vs. Production**
  - Specific energy consumption (SEC)
  - Energy per unit produced
  - Production efficiency
  - Correlation analysis

#### Shopfloor Management
- **Shopfloor Configuration**
  - Multiple shopfloors per site
  - Equipment associations
  - Meter assignments
  - Production line definitions
  
- **Shopfloor Reports**
  - Energy by production line
  - Energy by shift
  - Production vs. energy correlation
  - Efficiency improvements

### 14.4 SVG Graphics

#### Interactive Graphics
- **Custom Visualizations**
  - Upload custom SVG files
  - Overlay real-time data
  - Clickable elements
  - Animated indicators
  
- **Use Cases**
  - Facility layouts with real-time data
  - Process flow diagrams
  - Equipment schematics
  - System topology views

### 14.5 Knowledge Management

#### Knowledge Base
- **File Management**
  - Upload documents (PDF, Word, Excel)
  - Manuals and datasheets
  - Best practices guides
  - Training materials
  
- **Categorization**
  - Organize by category
  - Tag-based organization
  - Search functionality
  - Version control

---

## 15. End Goals & Business Value

### 15.1 Energy Efficiency Achievements

#### Quantifiable Outcomes
- **Energy Reduction**
  - 10-30% energy savings typical
  - Identify low-cost/no-cost opportunities
  - Track and verify savings
  - Continuous improvement

- **Peak Demand Reduction**
  - Lower demand charges
  - Avoid capacity fees
  - Improve load factor
  - Grid flexibility

#### Operational Excellence
- **Equipment Optimization**
  - Optimal equipment operation
  - Reduce equipment runtime
  - Extend equipment lifespan
  - Predictive maintenance

- **Comfort & Productivity**
  - Maintain occupant comfort
  - Reduce temperature/humidity complaints
  - Improve indoor air quality
  - Enhance workplace productivity

### 15.2 Cost Reduction

#### Direct Cost Savings
- **Energy Bill Reduction**
  - Lower energy consumption costs
  - Reduced demand charges
  - Tariff optimization
  - Avoid power factor penalties

- **Operational Cost Savings**
  - Reduced maintenance costs
  - Fewer emergency repairs
  - Extended equipment life
  - Lower replacement costs

#### Financial Visibility
- **Cost Transparency**
  - Understand where money is spent
  - Identify cost reduction opportunities
  - Justify capital investments
  - Budget accurately

### 15.3 Carbon Neutrality & Sustainability

#### Environmental Goals
- **Carbon Reduction**
  - Track progress to carbon neutrality
  - Measure emission reductions
  - Renewable energy integration
  - Carbon offset management

- **Regulatory Compliance**
  - Meet carbon reporting requirements
  - Comply with energy regulations
  - Prepare for carbon taxes
  - Achieve green building certifications

#### Corporate Sustainability
- **ESG Reporting**
  - Environmental, Social, Governance metrics
  - Investor reporting
  - Sustainability rankings
  - Corporate responsibility

### 15.4 Data-Driven Decision Making

#### Strategic Insights
- **Informed Decisions**
  - Data-backed capital planning
  - Technology selection
  - Energy procurement strategies
  - Investment prioritization

- **Performance Management**
  - KPI tracking
  - Benchmarking
  - Target setting
  - Continuous improvement

#### Competitive Advantage
- **Market Differentiation**
  - Green building certifications
  - Sustainability leadership
  - Operational excellence
  - Cost leadership

### 15.5 Compliance & Reporting

#### Regulatory Compliance
- **Energy Regulations**
  - ISO 50001 compliance
  - GB/T 23331 (China)
  - ASHRAE standards
  - Local building codes

- **Reporting Requirements**
  - Government energy reporting
  - Carbon disclosure (CDP)
  - LEED/BREEAM reporting
  - Utility rebate applications

#### Audit & Verification
- **Data Integrity**
  - Auditable data trails
  - Measurement & Verification (M&V)
  - Third-party verification
  - Energy audit support

### 15.6 Scalability & Future-Proofing

#### Enterprise Scalability
- **System Growth**
  - Add meters and equipment easily
  - Expand to new sites
  - Integrate new technologies
  - Support 1000+ meters per installation

- **Technology Evolution**
  - Open architecture
  - Standard protocols
  - API-first design
  - Cloud-ready

#### Innovation Enablement
- **Emerging Technologies**
  - AI/ML integration ready
  - IoT device support
  - Edge computing compatibility
  - Blockchain for energy trading

---

## Appendix: Feature Summary Matrix

| Feature Category | Key Features | User Benefit | Business Value |
|------------------|--------------|--------------|----------------|
| **Authentication** | Username/email login, session management, password security, account lockout | Secure access, user accountability | Protect sensitive data, regulatory compliance |
| **Authorization** | RBAC, privilege system, API keys, data isolation | Granular access control, multi-tenancy | Secure multi-organization deployments |
| **System Config** | Spaces, equipment, meters, cost centers, tariffs | Flexible system setup, easy expansion | Adapt to any organizational structure |
| **Data Acquisition** | Modbus TCP, MQTT, offline import, real-time monitoring | Automatic data collection, minimal manual entry | Reduce labor, improve data quality |
| **Data Processing** | Normalization, cleaning, aggregation, virtual meters | Accurate data, consistent reporting | Reliable analytics, confident decisions |
| **Energy Management** | Multi-level tracking, flow visualization, load profiling | Complete visibility, identify waste | Optimize energy use, reduce consumption |
| **Billing & Cost** | Multi-tariff support, cost allocation, budget tracking | Transparent costs, budget control | Reduce costs, accurate invoicing |
| **Carbon Tracking** | Emission calculations, scope-based reporting, sustainability KPIs | Track carbon footprint, sustainability goals | Meet regulations, corporate responsibility |
| **Advanced Systems** | Energy storage, microgrids, PV, EV charging, VPP | Renewable integration, grid services | Revenue generation, energy independence |
| **Reporting** | 100+ pre-built reports, dashboards, export, scheduling | Comprehensive insights, time savings | Inform strategy, demonstrate ROI |
| **FDD** | Fault detection, diagnostics, maintenance management | Early problem detection, prevent failures | Reduce downtime, lower maintenance costs |
| **Integration** | RESTful API, Modbus, BACnet, MQTT, OPC UA | Connect existing systems, seamless data exchange | Leverage existing investments, avoid silos |
| **Notifications** | Email, WeChat, web messages, SMS | Timely alerts, stay informed | Quick response, prevent issues |
| **Advanced Features** | Distribution monitoring, power quality, production integration | Specialized capabilities | Industry-specific value |

---

## Conclusion

MyEMS provides a comprehensive, end-to-end energy management solution that addresses every aspect of energy monitoring, analysis, and optimization. From initial user authentication through data acquisition, processing, analysis, and reporting, the system delivers:

✅ **Complete Visibility**: Real-time and historical energy data across the entire organization  
✅ **Cost Control**: Automated billing, budget tracking, and cost reduction opportunities  
✅ **Sustainability**: Carbon tracking, renewable integration, and carbon neutrality goals  
✅ **Operational Excellence**: Fault detection, maintenance optimization, and equipment efficiency  
✅ **Data-Driven Decisions**: 100+ reports, dashboards, and advanced analytics  
✅ **Scalability**: Support for 1000+ meters, multi-site deployments, and future technologies  
✅ **Integration**: Open APIs and standard protocols for seamless system integration  

**End Goal Achievement**: Organizations using MyEMS typically achieve 10-30% energy savings, significant cost reductions, carbon neutrality progress, and operational excellence through data-driven energy management.

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**MyEMS Version**: v5.12.0  
**Author**: MyEMS Documentation Team

For more information:
- Email: zny@myems.org
- WeChat: +86 13011132526
- GitHub: [MyEMS/myems](https://github.com/MyEMS/myems)
