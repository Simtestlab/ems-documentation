# MyEMS Core Features & Business Logic

This document provides an in-depth explanation of the core business logic, features, and technical implementation of the MyEMS Energy Management System.

## Table of Contents

- [System Core Concepts](#system-core-concepts)
- [Energy Management Features](#energy-management-features)
- [Billing & Cost Management](#billing--cost-management)
- [Carbon Emissions Tracking](#carbon-emissions-tracking)
- [Data Processing Pipeline](#data-processing-pipeline)
- [Advanced Features](#advanced-features)
- [Technical Implementation](#technical-implementation)
- [API Architecture](#api-architecture)
- [Database Design Patterns](#database-design-patterns)

---

## System Core Concepts

### Hierarchical Organization Structure

MyEMS uses a **hierarchical multi-level architecture** to represent organizational structures:

```
Enterprise
    └── Site/Campus
        └── Building
            └── Floor
                └── Space/Room
                    └── Equipment/Systems
                        └── Meters
```

**Key Components**:

- **Sites**: Top-level organizational units (campuses, facilities)
- **Buildings**: Individual structures within a site
- **Floors**: Levels within buildings
- **Spaces**: Individual rooms or areas (offices, conference rooms, storage)
- **Equipment**: Energy-consuming systems (HVAC, lighting, motors, compressors)
- **Meters**: Physical measurement devices (electricity meters, water meters, gas meters)

**Benefits**:
- Enables energy tracking at any organizational level
- Supports cost allocation and chargeback
- Allows drill-down analysis from building to equipment level
- Facilitates energy savings tracking and benchmarking

### Multi-Tenancy Support

MyEMS supports multiple independent organizations using the same system:

- **Tenants**: Independent organizational entities
- **Cost Centers**: Cost allocation units within tenants
- **Independent Billing**: Each tenant maintains separate billing records
- **Data Isolation**: Tenant data is logically separated while sharing infrastructure
- **Permission Control**: Granular permissions per tenant

### Energy Category System

Energy types are managed through configurable categories:

| Category | Examples | Unit | Purpose |
|----------|----------|------|---------|
| **Electricity** | Grid power, solar generation, backup power | kWh | Primary energy tracking |
| **Water** | Municipal supply, well water | m³ (cubic meters) | Resource consumption |
| **Gas** | Natural gas, biogas | m³ or kg | Fuel consumption |
| **Cooling** | Chilled water, district cooling | kWh equivalent | HVAC tracking |
| **Heating** | Hot water, district heating, steam | kWh equivalent | Thermal energy tracking |
| **Other** | Fuel, compressed air, etc. | Unit-specific | Specialized tracking |

**Dynamic Configuration**:
- Custom energy categories can be added
- Each category has configurable units
- Emission factors per category for carbon calculations
- Tariffs applied per category and cost center

---

## Energy Management Features

### 1. Meter Management & Data Collection

**Meter Types Supported**:

1. **Physical Meters**
   - Connected via Modbus TCP protocol
   - Real-time data acquisition at configurable intervals
   - Multiple register configuration (energy, power, voltage, current, power factor, etc.)
   - Automatic data buffering and error handling
   - Disconnection resilience with retry logic

2. **Offline Meters**
   - Manual entry or bulk import via Excel
   - For historical data or non-connected devices
   - Supports date-range data entry
   - Audit trail for manual entries

3. **Virtual Meters**
   - Software-calculated aggregations of physical meters
   - Formula-based calculations (additions, subtractions, weighted averages)
   - Useful for:
     - Sub-meter aggregation (e.g., HVAC = heating + cooling)
     - Energy flows (e.g., gross consumption - generation)
     - Normalized comparisons
   - Real-time calculation from underlying meters

**Meter Configuration**:

```
Physical Meter Properties:
├── Identification
│   ├── Meter ID (unique identifier)
│   ├── Name (display name)
│   ├── Description
│   └── Serial Number
├── Connection
│   ├── Protocol (Modbus TCP, MQTT, HTTP)
│   ├── Gateway/Data Source
│   └── Register Mapping
├── Energy Properties
│   ├── Energy Category (electricity, water, gas, etc.)
│   ├── Energy Item (kWh, kW, reactive, etc.)
│   ├── Unit
│   └── Calibration/Multiplier
└── Business Properties
    ├── Associated Equipment
    ├── Location/Space
    ├── Cost Center (for billing)
    └── Installation Date
```

**Data Acquisition Flow**:

```
Physical Device
    ↓
Modbus TCP Service (myems-modbus-tcp)
    ├─ Read registers at interval (e.g., every 15 minutes)
    ├─ Apply multipliers (e.g., 0.01 kWh per register unit)
    ├─ Buffer in memory for batching
    └─ Write to historical database
    ↓
Raw Data Storage
    ├─ Table: meter_instantaneous_data (latest values)
    └─ Table: meter_hourly_data (aggregated hourly)
```

### 2. Data Normalization

**Purpose**: Convert raw meter readings into standardized, comparable energy metrics.

**Normalization Process**:

```
Raw Meter Data (Register Values)
    ↓
Step 1: Apply Calibration Factors
    ├─ Multiply by meter calibration factor (e.g., 0.01)
    ├─ Account for meter-specific characteristics
    └─ Example: Raw value 1234 × 0.01 = 12.34 kWh
    ↓
Step 2: Unit Conversion
    ├─ Convert to standard units (kWh, m³, etc.)
    ├─ Handle unit transformations (Wh → kWh, cm³ → m³)
    └─ Maintain precision for billing
    ↓
Step 3: Meter Aggregation
    ├─ Combine related meters (e.g., 3-phase electricity)
    ├─ Apply weighting factors if needed
    └─ Validate totals for consistency
    ↓
Step 4: Virtual Meter Calculation
    ├─ Apply formulas (addition, subtraction, division)
    ├─ Example: Total = Meter1 + Meter2 - Meter3
    └─ Calculate derived metrics
    ↓
Step 5: Data Repair
    ├─ Fill gaps in missing data via interpolation
    ├─ Detect and fix anomalies
    ├─ Validate data monotonicity (no negative consumption)
    └─ Handle meter resets and rollovers
    ↓
Normalized Energy Data (Historical Database)
```

**Key Calculations**:

```python
# Basic normalization
normalized_value = raw_value × meter_calibration × unit_conversion_factor

# Virtual meter with formula
virtual_meter_value = (meter1 + meter2) - meter3

# Handle meter reset (e.g., 99999 → 0)
if current < previous:
    actual_consumption = current + (meter_max - previous)
else:
    actual_consumption = current - previous
```

### 3. Real-Time Monitoring

**Live Dashboard Features**:

- **Instantaneous View**: Real-time energy consumption displayed as active power (kW)
- **Status Indicators**: Green (normal), yellow (warning), red (critical)
- **Trend Indicators**: Up/down arrows showing consumption increase/decrease
- **Comparisons**: 
  - vs. same time yesterday
  - vs. average of past week/month
  - vs. target/baseline
- **Automatic Refresh**: Updates at configurable intervals (typically 5-15 minutes)

**Real-Time Data Pipeline**:

```
Meter → Acquisition Service → Instantaneous Table → API Cache → Web UI
                                    ↓
                              Hourly Aggregation
                                    ↓
                              Historical Table
```

**Performance Optimization**:

- Redis caching of frequently accessed metrics
- Materialized hourly/daily/monthly aggregations
- Pre-calculated statistics to avoid runtime computation
- Lazy loading of detailed data

### 4. Historical Analysis & Trends

**Time-Series Analysis Features**:

- **Hourly Data**: Detailed consumption every hour
- **Daily Aggregation**: Sum of hourly values
- **Monthly Aggregation**: Sum of daily values
- **Peak/Off-Peak Analysis**: Time-of-use consumption patterns
- **Seasonal Comparison**: Year-over-year trends
- **Baseline Deviation**: Comparison to expected consumption

**Trend Calculations**:

```
Daily Pattern Analysis:
- Peak hours (e.g., 9 AM - 5 PM)
- Off-peak hours (e.g., 8 PM - 7 AM)
- Weekend vs weekday differences
- Holiday impact on consumption

Comparison Metrics:
- Same-day-last-year comparison
- Same-period average
- Trend slope (increasing/decreasing)
- Volatility (standard deviation)

Anomaly Detection:
- Spike detection (consumption > 1.5 × average)
- Anomalous patterns (e.g., 24/7 operation)
- Sudden changes (> 20% increase)
```

**Report Types**:

1. **Consumption Report**: Total usage by time period
2. **Comparison Report**: Current vs baseline/target
3. **Trend Report**: Historical patterns and forecasting
4. **Peak Load Report**: Maximum demand analysis
5. **Efficiency Report**: Usage per unit (e.g., kWh per square meter)

---

## Billing & Cost Management

### Multi-Tariff Billing System

MyEMS supports complex billing scenarios with multiple tariff structures:

**Tariff Types**:

```
1. Time-of-Use (TOU) Tariff
   ├─ Peak hours (e.g., 9 AM - 9 PM): $0.15/kWh
   ├─ Off-peak hours (9 PM - 9 AM): $0.08/kWh
   └─ Applied based on consumption timing

2. Tiered/Stepped Tariff
   ├─ First 1000 kWh: $0.10/kWh
   ├─ Next 1000 kWh: $0.12/kWh
   ├─ Above 2000 kWh: $0.15/kWh
   └─ Applied based on total consumption

3. Demand Charge Tariff
   ├─ Energy charge: $0.10/kWh
   ├─ Demand charge: $10/kW of peak power
   └─ Applies to peak usage during period

4. Fixed + Variable Tariff
   ├─ Fixed monthly charge: $50
   ├─ Variable charge: $0.12/kWh
   └─ Total = Fixed + (consumption × variable rate)

5. Power Factor Tariff
   ├─ Base rate: $0.10/kWh
   ├─ Power factor < 0.95: +5% surcharge
   ├─ Power factor > 0.98: -3% discount
   └─ Encourages efficient reactive load management

6. Seasonal Tariff
   ├─ Summer rate: $0.15/kWh (Jun-Aug)
   ├─ Winter rate: $0.18/kWh (Dec-Feb)
   ├─ Spring/Fall rate: $0.12/kWh (Mar-May, Sep-Nov)
   └─ Reflects seasonal demand differences
```

### Billing Calculation Process

**Step-by-Step Billing Algorithm**:

```
For Each Meter in System:
    Step 1: Retrieve Meter Configuration
        ├─ Get meter ID, energy category, cost center
        ├─ Get associated tariff(s)
        └─ Get applicable dates for tariff changes
    
    Step 2: Determine Billing Period
        ├─ Get latest processed billing timestamp
        ├─ Calculate time gap since last processing
        └─ Fetch energy data for unprocessed period
    
    Step 3: Aggregate Energy Consumption
        ├─ Sum hourly energy values by tariff period
        │  ├─ If TOU: separate peak and off-peak hours
        │  ├─ If tiered: sum total consumption
        │  └─ If seasonal: apply seasonal rates
        └─ Handle overlapping rates during transitions
    
    Step 4: Apply Tariff Rules
        ├─ For each energy block:
        │  ├─ Determine applicable tariff
        │  ├─ Apply tariff formula
        │  └─ Calculate block cost
        └─ Sum all blocks for total period cost
    
    Step 5: Calculate Final Billing
        ├─ Subtotal = sum of all blocks
        ├─ Apply adjustments:
        │  ├─ Surcharges (e.g., grid service charges)
        │  ├─ Discounts (e.g., volume discounts)
        │  └─ Taxes (VAT, excise tax, etc.)
        └─ Final billing amount = Subtotal + adjustments - credits
    
    Step 6: Store Billing Record
        ├─ Create billing_ledger entry
        ├─ Record consumption and cost
        ├─ Mark processed period
        └─ Enable future incremental processing
```

**Pseudo-Code Example**:

```python
def calculate_meter_billing(meter_id, start_time, end_time):
    """Calculate billing for a meter during a period"""
    
    # Get meter and tariff info
    meter = get_meter(meter_id)
    tariff = get_tariff(meter.cost_center_id)
    
    # Aggregate consumption by tariff period
    consumption_data = {
        'peak': query_energy(meter_id, 'peak', start_time, end_time),
        'off_peak': query_energy(meter_id, 'off_peak', start_time, end_time)
    }
    
    # Apply tariff rates
    billing = 0.0
    
    if tariff.type == 'TOU':
        # Time-of-use tariff
        billing = (consumption_data['peak'] * tariff.peak_rate +
                   consumption_data['off_peak'] * tariff.off_peak_rate)
    
    elif tariff.type == 'TIERED':
        # Tiered/stepped tariff
        total_consumption = sum(consumption_data.values())
        cumulative = 0
        
        for tier in tariff.tiers:
            tier_end = tier.threshold
            tier_start = tariff.tiers[tier_idx - 1].threshold if tier_idx > 0 else 0
            
            tier_consumption = min(tier_end, total_consumption) - cumulative
            if tier_consumption > 0:
                billing += tier_consumption * tier.rate
            
            cumulative = tier_end
            if total_consumption <= tier_end:
                break
    
    # Apply adjustments
    subtotal = billing
    tax = subtotal * tariff.tax_rate
    surcharge = tariff.surcharge_amount
    
    total_billing = subtotal + tax + surcharge
    
    return {
        'consumption': sum(consumption_data.values()),
        'cost': total_billing,
        'details': {
            'subtotal': subtotal,
            'tax': tax,
            'surcharge': surcharge
        }
    }
```

### Cost Allocation & Chargeback

**Hierarchical Cost Distribution**:

```
Building Total Cost: $10,000
    ├─ Allocated to 3 Tenants proportional to energy use
    │  ├─ Tenant A: 40% usage → $4,000
    │  ├─ Tenant B: 35% usage → $3,500
    │  └─ Tenant C: 25% usage → $2,500
    │
    └─ For Tenant A:
        ├─ Cost Centers within tenant
        │  ├─ Operations: 60% → $2,400
        │  └─ Support: 40% → $1,600
        │
        └─ For Operations Cost Center:
            ├─ Departments
            │  ├─ Department 1: 45% → $1,080
            │  └─ Department 2: 55% → $1,320
```

---

## Carbon Emissions Tracking

### Emission Factor Management

**Carbon Calculation Basis**:

```
Carbon Emissions = Energy Consumption × Emission Factor

Where Emission Factor depends on:
1. Energy Type (electricity, natural gas, etc.)
2. Geographic Region (grid mix, fuel sources)
3. Time Period (seasonal grid variations)
4. Calculation Standard (GHG Protocol, ISO 14064, etc.)
```

**Emission Factors by Energy Type**:

| Energy Type | Typical Factor | Source | Note |
|-------------|---|---|---|
| **Electricity** | 0.45 kg CO₂/kWh | Grid average | Varies by region/season |
| **Natural Gas** | 1.89 kg CO₂/m³ | Standard | Higher for raw gas |
| **Water** | 0.28 kg CO₂/m³ | Treatment & distribution | Pumping + treatment |
| **District Cooling** | 0.12 kg CO₂/kWh | Heat pump efficiency | Lower than direct air conditioning |
| **District Heating** | 0.08 kg CO₂/kWh | Fossil fuel boiler | Varies by fuel mix |

### Carbon Calculation Process

**Step-by-Step Algorithm**:

```
For Each Meter in System:
    Step 1: Retrieve Meter Configuration
        ├─ Get meter ID and energy category
        ├─ Get associated emission factor
        └─ Get applicable dates for factor changes
    
    Step 2: Determine Reporting Period
        ├─ Get latest processed carbon timestamp
        ├─ Calculate time gap since last processing
        └─ Fetch energy data for unprocessed period
    
    Step 3: Aggregate Energy Consumption
        ├─ Sum hourly values by emission factor period
        ├─ Handle overlapping factors during transitions
        └─ Separate by energy subcategory if needed
    
    Step 4: Apply Emission Factors
        ├─ For each energy block:
        │  ├─ Apply appropriate emission factor
        │  └─ Calculate CO₂ equivalent
        └─ Sum all blocks for total period carbon
    
    Step 5: Store Carbon Records
        ├─ Create carbon_ledger entry
        ├─ Record consumption and CO₂ equivalent
        ├─ Mark processed period
        └─ Enable future incremental processing
```

**Carbon Metrics**:

```
Primary Metric:
- Scope 1: Direct emissions (on-site fuel combustion)
- Scope 2: Indirect emissions (purchased electricity)
- Scope 3: Other indirect emissions (supply chain, commuting)

Reporting Units:
- kg CO₂e (kilograms CO₂ equivalent)
- Tons CO₂e (metric tons)
- kg CO₂e per square meter per year
- kg CO₂e per USD of revenue

Tracking:
- Absolute emissions (total)
- Intensity emissions (per unit)
- Year-over-year trend
- Target progress
```

### Environmental Impact Reporting

**Carbon Offset Equivalents**:

```
1 ton CO₂e equivalent to:
├─ 1.2 acres of forest grown for 10 years
├─ 143 gallons of gasoline burned
├─ 1,025 miles driven in average car
├─ 470 pounds of coal burned
├─ 2,471 kWh electricity consumption
└─ 3.6 barrels of oil consumed
```

**Carbon Reduction Initiatives**:

- Track baseline emissions (e.g., first 12 months)
- Compare current vs baseline
- Calculate savings from efficiency projects
- Monitor progress toward carbon neutrality goals
- Generate sustainability reports

---

## Data Processing Pipeline

### 1. Data Acquisition Service (myems-modbus-tcp)

**Architecture**:

```
Modbus TCP Service Architecture:

┌─────────────────────────────────────────────────┐
│      Modbus TCP Acquisition Service             │
└─────────────────────────────────────────────────┘
         ↓
    Main Event Loop
    ├─ Read configuration from system database
    ├─ Connect to Modbus gateways
    ├─ Poll registers at interval (e.g., every 15 min)
    ├─ Parse Modbus responses
    └─ Buffer data in memory
         ↓
    Data Buffering & Batching
    ├─ Accumulate readings in memory
    ├─ Batch writes to database (e.g., 100 records at once)
    └─ Optimize for throughput
         ↓
    Historical Database
    ├─ meter_instantaneous_data (latest value)
    ├─ meter_hourly_data (aggregated)
    └─ Retention: Full precision for 1-2 years
```

**Error Handling**:

- **Connection Failures**: Retry with exponential backoff
- **Timeout**: Skip reading, try next iteration
- **Invalid Data**: Log and skip, don't corrupt database
- **Database Errors**: Backoff and retry
- **Meter Reset**: Detect via value decrease, calculate actual consumption

### 2. Data Normalization Service (myems-normalization)

**Workflow**:

```
Schedule: Run hourly or on-demand

For Each Meter:
    1. Check if normalization needed (new raw data)
    2. Retrieve raw values from meter_instantaneous_data
    3. Apply normalization formula:
       - Raw × calibration_factor = normalized value
    4. Check for meter resets (current < previous)
    5. Calculate incremental consumption:
       - If reset: current + (max_value - previous)
       - Otherwise: current - previous
    6. Store in meter_hourly_data table
    7. Update last processed timestamp
    
For Virtual Meters:
    1. Apply formula (e.g., sum of related meters)
    2. Calculate based on normalized values
    3. Store calculated result
    
For Data Repair:
    1. Detect gaps in time series
    2. Use interpolation to fill gaps
    3. Validate monotonicity
    4. Flag suspicious data
```

### 3. Data Cleaning Service (myems-cleaning)

**Cleaning Operations**:

```
Schedule: Run daily (typically after peak hours)

Analog Values (e.g., temperature, humidity):
    1. Remove outliers (> 3 std dev)
    2. Apply smoothing (moving average)
    3. Flag suspicious readings
    4. Log anomalies for review

Digital Values (binary events):
    1. Validate state transitions
    2. Remove duplicates
    3. Check for impossible sequences
    4. Verify timestamp continuity

Energy Values:
    1. Ensure monotonicity (never decreases)
    2. Detect and flag sudden jumps
    3. Remove negative values
    4. Handle meter resets properly
    5. Check for unrealistic peaks

Duplicate Detection:
    1. Identify readings from same meter, same timestamp
    2. Keep latest reading
    3. Log duplicates for auditing

Historical Data Cleanup:
    1. Archive data older than retention period
    2. Prune unnecessary detail
    3. Generate summary statistics
    4. Free up database space
```

### 4. Data Aggregation Service (myems-aggregation)

**Parallel Processing Architecture**:

```
myems-aggregation Service:
    ├─ Process 1: Combined Equipment Energy
    ├─ Process 2: Combined Equipment Billing
    ├─ Process 3: Combined Equipment Carbon
    ├─ Process 4: Equipment Energy
    ├─ Process 5: Equipment Billing
    ├─ Process 6: Equipment Carbon
    ├─ Process 7: Meter Billing (all meters)
    ├─ Process 8: Meter Carbon (all meters)
    ├─ Process 9: Offline Meter Billing
    ├─ Process 10: Offline Meter Carbon
    ├─ Process 11: Virtual Meter Billing
    ├─ Process 12: Virtual Meter Carbon
    ├─ Process 13: Space Energy
    ├─ Process 14: Space Billing
    ├─ Process 15: Space Carbon
    ├─ Process 16: Store Energy/Billing/Carbon
    ├─ Process 17: Tenant Energy/Billing/Carbon
    └─ Process 18: Shopfloor Energy/Billing/Carbon

Each Process:
    1. Runs independently in separate OS process
    2. Loops continuously
    3. Queries system database for configuration
    4. Processes data incrementally
    5. Stores results in respective database
    6. Handles errors gracefully with retry logic
    7. Logs activity for monitoring
```

**Meter Billing Aggregation Example**:

```python
def meter_billing_aggregation():
    while True:
        try:
            # Step 1: Get all meters
            meters = query("SELECT * FROM tbl_meters")
            
            for meter in meters:
                # Step 2: Get last processed timestamp
                last_processed = query(
                    f"SELECT MAX(end_datetime_utc) FROM "
                    f"tbl_meter_hourly_billing WHERE meter_id = {meter.id}"
                )
                
                # Step 3: Get energy data since last processed
                energy_data = query(
                    f"SELECT * FROM tbl_meter_hourly_data "
                    f"WHERE meter_id = {meter.id} "
                    f"AND datetime_utc > {last_processed}"
                )
                
                # Step 4: Get applicable tariff
                tariff = get_tariff(meter.cost_center_id)
                
                # Step 5: Calculate billing
                for hour_data in energy_data:
                    cost = calculate_cost(hour_data.consumption, tariff)
                    
                    # Step 6: Store billing record
                    insert_billing(
                        meter_id=meter.id,
                        datetime=hour_data.datetime,
                        consumption=hour_data.consumption,
                        cost=cost
                    )
            
            # Sleep until next cycle (e.g., 1 hour)
            time.sleep(3600)
        
        except Exception as e:
            logger.error(f"Error in billing aggregation: {e}")
            time.sleep(300)  # Retry after 5 minutes
```

---

## Advanced Features

### 1. Equipment Control & Automation

**Supported Equipment Types**:

- HVAC systems (heating, cooling, ventilation)
- Lighting systems (on/off, dimming, color)
- Water pumps and valves
- Power switching and load control
- Demand response management

**Control Modes**:

```
1. Manual Control
   - Operator sends command via UI
   - Equipment responds immediately
   - Logged for audit trail

2. Scheduled Control
   - Predefined schedules (e.g., lights on at 6 AM)
   - Recurring patterns (weekdays different from weekends)
   - Holiday exceptions

3. Conditional Control (Rules Engine)
   - Trigger: If temperature > 28°C
   - Action: Turn on air conditioning
   - Additional conditions: AND time between 9 AM - 5 PM

4. Demand Response
   - Participate in utility DR programs
   - Automatically reduce load during peak periods
   - Maintain comfort while reducing costs

5. Optimization Control (AI)
   - Predictive algorithm adjusts setpoints
   - Minimizes energy while maintaining comfort
   - Learns usage patterns
```

### 2. Predictive Analytics & Machine Learning

**Prediction Capabilities**:

- **Energy Consumption Forecast**: Predict next day/week/month consumption
- **Peak Load Prediction**: Anticipate maximum demand
- **Anomaly Detection**: Identify unusual energy patterns
- **Fault Prediction**: Detect equipment degradation before failure
- **Optimization Recommendations**: Suggest energy-saving actions

**ML Algorithms Used**:

- Time-series forecasting (ARIMA, Prophet)
- Neural networks (LSTM for sequential data)
- Regression models for consumption prediction
- Clustering for pattern identification
- Anomaly detection algorithms

**Example Use Case**:

```
Input: Historical consumption data (past 12 months)
       Temperature, humidity, occupancy sensors
       Historical events (HVAC maintenance)

Model: Trained LSTM neural network

Output: Next 7 days hourly consumption prediction
        95% confidence interval
        Risk of peak demand > threshold
```

### 3. Virtual Power Plants & Microgrids

**Virtual Power Plant Capabilities**:

- Aggregate distributed energy resources (solar, wind, storage)
- Participate in electricity markets
- Provide grid services (frequency regulation, voltage support)
- Optimize dispatch based on pricing signals
- Manage battery charging/discharging

**Microgrid Management**:

- Island mode operation (disconnect from main grid)
- Load balancing among local resources
- Forecasting and scheduling
- Economic optimization
- Resilience management

### 4. Equipment Fault Diagnosis

**Fault Detection Methods**:

```
1. Threshold-Based
   - Power consumption > threshold = malfunction
   - COP (Coefficient of Performance) < expected = degradation

2. Pattern-Based
   - Unusual startup/shutdown patterns
   - Irregular runtime vs outdoor temperature
   - Cycle frequency anomalies

3. Data-Driven
   - Neural network trained on normal operation
   - Identifies deviations from learned patterns
   - Provides confidence score for fault

4. Physics-Based
   - Energy balance checks
   - Thermodynamic violation detection
   - System efficiency metrics
```

**Example - HVAC Fault Detection**:

```
Fault: Refrigerant Leak
    Symptoms:
    - COP drops from 3.5 to 2.8
    - Superheat increases
    - Compressor runtime increases 20%
    - Discharge temperature rises
    
Detection:
    - Daily COP monitoring
    - Alert when 20% below baseline
    - Recommend technician visit
    - Prevent further degradation

Impact:
    - Early detection avoids equipment failure
    - Improves efficiency
    - Reduces emergency maintenance costs
```

### 5. Work Order Management

**Workflow**:

```
1. Issue Detection
   ├─ Automatic via fault diagnosis
   ├─ Manual report by operator
   └─ Scheduled maintenance

2. Work Order Generation
   ├─ Create ticket
   ├─ Assign priority and severity
   ├─ Estimate impact on operations
   └─ Route to maintenance team

3. Tracking
   ├─ Monitor work progress
   ├─ Track resource usage
   ├─ Record downtime
   └─ Verify completion

4. Verification
   ├─ Confirm equipment restored
   ├─ Measure energy impact
   ├─ Rate technician performance
   └─ Archive for trending

5. Analytics
   ├─ MTBF (Mean Time Between Failures)
   ├─ MTTR (Mean Time To Repair)
   ├─ Cost per incident
   ├─ Technician productivity
   └─ Equipment reliability trends
```

---

## Technical Implementation

### 1. API Architecture

**Falcon Framework Benefits**:

```
Lightweight & Fast:
- Minimal overhead compared to Django/Flask
- Built for microservices
- Excellent performance for data-heavy APIs

Resource-Oriented Design:
- RESTful principles
- Resources = Python classes
- HTTP methods = class methods (on_get, on_post, on_put, on_delete)

Example Resource:

class MeterCollection:
    def on_get(self, req, resp):
        # GET /v1/meters
        # List all meters
        
    def on_post(self, req, resp):
        # POST /v1/meters
        # Create new meter

class MeterItem:
    def on_get(self, req, resp, meter_id):
        # GET /v1/meters/{meter_id}
        # Retrieve specific meter
        
    def on_put(self, req, resp, meter_id):
        # PUT /v1/meters/{meter_id}
        # Update meter
        
    def on_delete(self, req, resp, meter_id):
        # DELETE /v1/meters/{meter_id}
        # Delete meter
```

**API Endpoint Categories**:

```
1. Configuration APIs
   /v1/tenants, /v1/buildings, /v1/spaces, /v1/equipment, /v1/meters

2. Data APIs
   /v1/meters/{meter_id}/energy/hourly
   /v1/spaces/{space_id}/energy/daily
   /v1/equipment/{equipment_id}/billing/monthly

3. Report APIs
   /v1/reports/meterenergy
   /v1/reports/spacebilling
   /v1/reports/tenantcarbon

4. Control APIs
   /v1/equipment/{equipment_id}/command
   /v1/devices/{device_id}/control

5. Analysis APIs
   /v1/analytics/prediction/consumption
   /v1/analytics/fault-detection

6. Administration APIs
   /v1/users, /v1/permissions, /v1/roles
```

### 2. Caching Strategy

**Multi-Level Caching**:

```
Level 1: Browser Cache
    - Static assets (CSS, JS, images)
    - Cache duration: 1-7 days
    - Reduces network traffic

Level 2: CDN Cache (optional)
    - API responses for public data
    - Cache duration: 5-30 minutes
    - Distributes load globally

Level 3: Redis Application Cache
    - Frequently accessed data (meters, buildings)
    - List queries with filters
    - Cache duration: 5-60 minutes
    
    Example:
    meter:list:tenant_id_1 → [list of meters for tenant 1]
    meter:item:42 → {meter 42 details}
    space:children:10 → [child spaces of space 10]

Level 4: Database Query Optimization
    - Proper indexing
    - Query optimization
    - Materialized views for aggregations
```

**Cache Invalidation**:

```
Event-Based Invalidation:
    - When meter updated → clear meter:item:* and meter:list:*
    - When building updated → clear related caches
    - When tariff updated → clear billing-related caches

Time-Based Expiration:
    - Configuration data: 1 hour TTL
    - Real-time data: 5-15 minutes TTL
    - Historical aggregations: 24+ hours TTL
```

### 3. Database Performance Optimization

**Indexing Strategy**:

```
Historical Data Table (Very Large):
    Primary Key: (meter_id, datetime_utc)
    Indexes:
    - (meter_id, datetime_utc) - for range queries
    - (datetime_utc) - for time-based filtering
    - (energy_category_id, datetime_utc) - for category reports

System Configuration Tables (Medium):
    Primary Key: (id)
    Indexes:
    - (tenant_id) - for tenant data isolation
    - (space_id) - for space hierarchy
    - (equipment_id) - for equipment relationships

Aggregation Tables (Written Frequently):
    Primary Key: (meter_id, start_datetime_utc)
    Indexes:
    - (meter_id, start_datetime_utc) - for incremental processing
    - (cost_center_id, start_datetime_utc) - for cost allocation
```

**Query Optimization**:

```
Inefficient: SELECT * FROM tbl_meter_hourly_data 
             WHERE MONTH(datetime_utc) = 1 AND YEAR(datetime_utc) = 2024
Problem: Function on column prevents index usage

Optimized: SELECT * FROM tbl_meter_hourly_data 
           WHERE datetime_utc >= '2024-01-01' AND datetime_utc < '2024-02-01'
Benefit: Uses datetime index for fast range scan

Rule: Avoid functions on indexed columns, use range conditions
```

**Partitioning for Time-Series Data**:

```
Partition by Month:
    - tbl_meter_hourly_data_2024_01
    - tbl_meter_hourly_data_2024_02
    - tbl_meter_hourly_data_2024_03
    
Benefits:
    - Faster queries (smaller partition scanned)
    - Easier archival (old partitions moved to storage)
    - Parallel processing across partitions
```

---

## API Architecture

### Core API Resources

**1. Energy Data API**

```
GET /v1/meters/{meter_id}/energy/hourly
    Parameters:
    - start_datetime_local: ISO format timestamp
    - end_datetime_local: ISO format timestamp
    - language: en/zh_CN/etc.
    
    Response:
    {
      "meter_id": 42,
      "meter_name": "Building A - Electricity Meter",
      "energy_category": "Electricity",
      "energy_item": "Consumption (kWh)",
      "values": [
        {
          "datetime": "2024-01-15T00:00:00+08:00",
          "value": 123.45,
          "previous_value": 120.12
        },
        ...
      ]
    }
```

**2. Billing API**

```
GET /v1/meters/{meter_id}/billing/monthly
    Parameters:
    - month: YYYY-MM format
    - language: en/zh_CN/etc.
    
    Response:
    {
      "meter_id": 42,
      "month": "2024-01",
      "consumption": 1500.25,
      "unit": "kWh",
      "billing_amount": 234.50,
      "currency": "USD",
      "tariff": "Time-of-Use",
      "breakdown": {
        "peak_consumption": 800.0,
        "peak_cost": 150.00,
        "off_peak_consumption": 700.25,
        "off_peak_cost": 84.50
      }
    }
```

**3. Report API**

```
POST /v1/reports/equipment-batch
    Body:
    {
      "equipment_ids": [1, 2, 3],
      "start_datetime": "2024-01-01T00:00:00Z",
      "end_datetime": "2024-01-31T23:59:59Z",
      "report_type": "consumption"
    }
    
    Response:
    {
      "report_id": "abc123",
      "status": "generating",
      "progress": 45,
      "estimated_time": 120,
      "download_url": "/v1/reports/abc123/download"
    }
```

**4. Control API**

```
POST /v1/equipment/{equipment_id}/command
    Body:
    {
      "command_type": "on/off",
      "command_value": "on",
      "reason": "Manual control",
      "scheduled_time": null
    }
    
    Response:
    {
      "command_id": "cmd_456",
      "status": "sent",
      "equipment_id": 10,
      "timestamp": "2024-01-15T10:30:00Z",
      "acknowledgment_time": null
    }
```

---

## Database Design Patterns

### Time-Series Data Storage

**Challenge**: Store billions of energy readings efficiently

**Solution Pattern**:

```
1. Raw Data Table (High Frequency)
    tbl_meter_instantaneous_data
    ├─ Columns: meter_id, datetime_utc, value
    ├─ Retention: Latest 48 hours only
    ├─ Purpose: Real-time displays
    └─ Size: ~100 MB

2. Hourly Aggregation Table (Used for Most Reports)
    tbl_meter_hourly_data
    ├─ Columns: meter_id, datetime_utc, value
    ├─ Retention: 2-3 years
    ├─ Purpose: Analysis and reporting
    ├─ Calculation: SUM/AVG of minutely readings
    └─ Size: ~10 GB (much smaller than raw data)

3. Daily Aggregation Table (Trending)
    tbl_meter_daily_data
    ├─ Columns: meter_id, date, value
    ├─ Retention: 5-10 years
    ├─ Purpose: Long-term trends
    ├─ Calculation: SUM of hourly values
    └─ Size: ~100 MB

4. Monthly Aggregation Table (Billing & Reports)
    tbl_meter_monthly_data
    ├─ Columns: meter_id, year, month, value
    ├─ Retention: Permanent (archive)
    ├─ Purpose: Annual reports, audit
    ├─ Calculation: SUM of daily values
    └─ Size: ~1 MB
```

**Data Lifecycle Management**:

```
Day 1: Minutely data collected (downsampled to hourly)
    ├─ Hourly: Stored in hourly table
    └─ Raw: Kept for 48 hours for detail/verification

Day 2-90: Hourly data used for reports
    ├─ Available in hourly table
    └─ Daily aggregations calculated

Day 91+: Daily data used for trends
    ├─ Hourly data archived (optional)
    └─ Daily table is primary source

Year 2+: Monthly data for archives
    ├─ Hourly/daily data moved to cold storage
    └─ Monthly table in active database
```

### Incremental Aggregation Pattern

**Problem**: Recalculating aggregations for all historical data is expensive

**Solution**:

```
Store "Last Processed Timestamp":

tbl_meter_hourly_billing
├─ Columns:
│  ├─ meter_id
│  ├─ start_datetime_utc
│  ├─ end_datetime_utc  ← Last processed time
│  ├─ consumption
│  ├─ billing_cost
│  └─ created_datetime_utc
└─ Index: (meter_id, end_datetime_utc DESC)

Processing Algorithm:

1. Query: SELECT MAX(end_datetime_utc) FROM tbl_meter_hourly_billing 
          WHERE meter_id = 42
   Result: 2024-01-14T23:00:00Z (last processed)

2. Query: SELECT * FROM tbl_meter_hourly_data 
          WHERE meter_id = 42 AND datetime_utc > 2024-01-14T23:00:00Z
   Result: New data since last processing

3. Calculate billing only for new data
4. Insert new records
5. On next cycle, repeat from step 1 (picks up new last_processed time)

Benefit:
- Only processes new data
- O(hours_since_last_process) instead of O(total_hours)
- Scales to any data volume
```

### Hierarchical Data Storage

**Challenge**: Represent multi-level hierarchy (building → floor → space → equipment)

**Solution - Adjacency List with Materialized Path**:

```
Table: tbl_spaces

Standard Adjacency List:
├─ id: 1, name: "Building A", parent_id: NULL
├─ id: 2, name: "Floor 1", parent_id: 1
├─ id: 3, name: "Room 101", parent_id: 2
└─ id: 4, name: "Room 102", parent_id: 2

Problem: Finding all descendants requires recursive query (slow)

Solution - Add Materialized Path:
├─ id: 1, name: "Building A", parent_id: NULL, path: "/1/"
├─ id: 2, name: "Floor 1", parent_id: 1, path: "/1/2/"
├─ id: 3, name: "Room 101", parent_id: 2, path: "/1/2/3/"
└─ id: 4, name: "Room 102", parent_id: 2, path: "/1/2/4/"

Benefits:
- Finding descendants: WHERE path LIKE '/1/2/%' (single query)
- Finding ancestors: path parts between / (single query)
- Building tree: ORDER BY path (natural order)
- Depth = path.split('/').length - 2
```

---

## Summary: Core Business Logic Flow

```
Raw Physical Data
    ↓
Modbus TCP Acquisition (myems-modbus-tcp)
    ├─ Read meters every 15 minutes
    └─ Store in meter_instantaneous_data
    ↓
Data Normalization (myems-normalization)
    ├─ Apply calibration factors
    ├─ Handle meter resets
    ├─ Calculate incremental consumption
    └─ Store in meter_hourly_data
    ↓
Data Cleaning (myems-cleaning)
    ├─ Remove outliers
    ├─ Fill gaps
    ├─ Validate monotonicity
    └─ Clean historical database
    ↓
Data Aggregation (myems-aggregation) - 18 parallel processes
    ├─ Calculate energy by space/equipment/tenant
    ├─ Calculate billing with tariff rules
    ├─ Calculate carbon with emission factors
    └─ Store in respective aggregation tables
    ↓
API Service (myems-api) - Handles requests
    ├─ Query aggregation tables
    ├─ Cache results in Redis
    ├─ Generate reports
    └─ Serve to Web/Admin UIs
    ↓
User Interfaces
    ├─ Web UI (React) - Real-time dashboards
    ├─ Admin UI (AngularJS) - Configuration
    └─ Mobile Apps - On-the-go monitoring
```

---

## Contact & Support

For questions about core logic and advanced features:

- **Email**: zny@myems.org
- **WeChat**: +86 13011132526
- **QQ Group**: 792528967
- **GitHub Issues**: [MyEMS/myems/issues](https://github.com/MyEMS/myems/issues)

---

**Document Version**: 1.0  
**Last Updated**: January 2025  
**MyEMS Version**: v5.12.0
