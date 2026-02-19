# Energy Management System (EMS) Documentation

Welcome to the comprehensive documentation for the **SIM Energy Management System**. This documentation covers system analysis, architecture design, implementation guides, and reference materials for building a real-time energy management platform.

---

##  Documentation Structure

### [Analysis](Analysis/index.md)

System analysis, requirements, and design specifications.

- **[EMS Explanation](Analysis/ems-explanation.md)** - Core concepts and system overview
- **[System Architecture](Analysis/sim-ems-system-architecture.md)** - High-level system design
- **[Energy Component Parameters](Analysis/parameter.md)** - Complete parameter definitions for Grid, Battery, Inverter, Load
- **[State Machine Modes](Analysis/state-machine-modes.md)** - Operational states (IDLE, CHARGING, DISCHARGING, etc.)
- **[Dashboard Components](Analysis/dashboard-component.md)** - UI component specifications
- **[UI Implementation](Analysis/ui-implementation.md)** - Frontend design and layout
- **[MVP Scope](Analysis/mvp.md)** - Minimum Viable Product definition
- **[Folder Structure](Analysis/folder-structure.md)** - Project organization
- **[Self-Consumption Definition](Analysis/self-consumption-definition.md)** - Energy self-sufficiency metrics
- **[Comprehensive EMS Analysis](Analysis/COMPREHENSIVE_EMS_ANALYSIS_AND_CONSOLIDATED_DESIGN.md)** - Full system analysis document
- **[SIM EMS Roadmap](Analysis/SIM_EMS_ROADMAP_README.md)** - Development roadmap and phases

---

### [Implementation Architecture](Implementation%20Architecture/index.md)

Technical architecture and implementation details.

#### Core Architecture
- **[State Machine](Implementation%20Architecture/state-machine.md)** - State machine logic and transitions
- **[Communication Interface](Implementation%20Architecture/communication-interface.md)** - MQTT/WebSocket protocols and message formats
- **[WebSocket Implementation](Implementation%20Architecture/websocket-implementation.md)** - Real-time data streaming
- **[WebSocket Details](Implementation%20Architecture/websocket.md)** - WebSocket architecture

#### Backend Services
- **[FastAPI](Implementation%20Architecture/fastAPI.md)** - REST API and WebSocket backend
- **[Django](Implementation%20Architecture/django.md)** - Data management and admin interface

#### Frontend
- **[Dashboard Integration](Implementation Architecture/dashboard.md)**
  - Next.js dashboard with complete telemetry interfaces
  - Real-time telemetry payload definitions
  - Grid, Battery, Inverter, Load, Energy metrics
  - Multi-site scalability design
  - TypeScript interfaces and data contracts

---

### [Business Logics](Business%20Logics/index.md)

Business logic implementation and edge processing.

- **[Edge Core Cycle](Business%20Logics/edge-core-cycle.md)** - Edge device control loop
- **[Protocol Adapter](Business%20Logics/protocol-adapter.md)** - Communication protocol handling

---

### Developer Guide

Developer resources and guides.

- **SiteMap** - Site structure and navigation
- **WebSocket** - WebSocket development guides

---

### References

External system references and inspiration.

- **[IOT Gateways](./IOT%20Device/index.md)**

#### [EMS Controller](References/ems-controller/)
- **[EMS Features](References/ems-controller/ems-features.md)** - Standard EMS feature set

#### [MyEMS](References/myems/)
Reference system for ETL & Analytics Pipeline.

- **[MyEMS Overview](References/myems/myems-overview.md)** - System introduction
- **[MyEMS Features](References/myems/myems-features.md)** - Feature list
- **[Components Overview](References/myems/components-overview.md)** - Architecture overview
- **[Components List](References/myems/myems-components-list.md)** - Detailed component breakdown
- **[Business Logics](References/myems/myems-business-logics.md)** - Business rule implementation
- **[UI Components](References/myems/myems-UI-components.md)** - Dashboard UI elements

#### [OpenEMS](References/openems/)
Reference system for Real-Time Component Framework.

- **[OpenEMS Overview](References/openems/openems-overview.md)** - System introduction
- **[Core Architecture](References/openems/openems-core-architecture.md)** - Architecture design
- **[Modules](References/openems/openems-modules.md)** - Module structure
- **[Backend Modules](References/openems/openems-backend-modules.md)** - Backend components
- **[Communication Bridge](References/openems/openems-communication-bridge-modules.md)** - Device communication

#### [Theme](References/theme/)
- **[Theme Configuration](References/theme/theme.md)** - UI theme customization

---

## Quick Start Guides

### For System Architects
1. Start with **[EMS Explanation](Analysis/ems-explanation.md)**
2. Review **[System Architecture](Analysis/sim-ems-system-architecture.md)**
3. Understand **[Energy Parameters](Analysis/parameter.md)**
4. Check **[State Machine Modes](Analysis/state-machine-modes.md)**

### For Backend Developers
1. Study **[Communication Interface](Implementation%20Architecture/communication-interface.md)**
2. Implement **[State Machine](Implementation%20Architecture/state-machine.md)**
3. Set up **[FastAPI](Implementation%20Architecture/fastAPI.md)** backend
4. Review **[Edge Core Cycle](Business%20Logics/edge-core-cycle.md)**

### For Frontend Developers
1. Start with **[Dashboard Integration](Implementation%20Architecture/dashboard.md)**
2. Review **[Dashboard Components](Analysis/dashboard-component.md)**
3. Study **[UI Implementation](Analysis/ui-implementation.md)**
4. Understand **[WebSocket Implementation](Implementation%20Architecture/websocket-implementation.md)**

### For Project Managers
1. Review **[MVP Scope](Analysis/mvp.md)**
2. Check **[SIM EMS Roadmap](Analysis/SIM_EMS_ROADMAP_README.md)**
3. Understand **[Folder Structure](Analysis/folder-structure.md)**
4. Read **[Comprehensive EMS Analysis](Analysis/COMPREHENSIVE_EMS_ANALYSIS_AND_CONSOLIDATED_DESIGN.md)**

---