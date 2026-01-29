# UI Components

This document describes the UI pages implemented in the **MyEMS Web application**, their components, and functional behavior.

---

## Login / Authentication Page

This page is responsible for validating users and initializing the application context after successful login.

### Implemented UI Components & Functionalities

- User Email Input  
- Password Input  
- Login Action  
- Error Handling  

### Functional Description

- Users enter their credentials for authentication.
- On successful validation, the user is redirected to the dashboard.
- On failure, appropriate error messages are displayed.
- This page serves as the entry point to the system.

!!! note:
    User creation and management are handled by Admin users.  
    Self-registration is not required in standard MyEMS deployments.

---

## Dashboard Pages

Dashboard pages provide a consolidated overview of system performance, energy usage metrics, and recent operational activities.

### Implemented UI Components & Functionalities

- **KPI Cards**
    - Total Energy Usage  
    - Cost  
    - Carbon Emissions  
    - Active Alarms
- **Charts**
    - Line and bar charts for trends and comparisons  
    - Tooltips, legends, and responsive resizing
- **Recent Activity / Alerts**
    - Displays recent tickets, alarms, or system events
- **Date & Period Filters**
    - Day, week, and month-based filtering

### Functional Description

- Aggregated data is visualized for quick operational insights.
- Filters dynamically update all dashboard widgets.
- Acts as the primary landing page after login.

---

## Energy Analysis

Energy Analysis pages provide detailed insights into energy consumption patterns over time.

### Implemented UI Components & Functionalities

- Time-series charts showing energy consumption over time
- Period selection (hourly, daily, monthly)
- Entity filters (meter, equipment, space)
- KPI summaries (total consumption, average usage, peak demand)

### Functional Description

- Enables trend analysis and energy performance monitoring.
- Supports comparison across time periods and entities.
- Data updates dynamically based on user-selected filters.

---

## Equipment Monitoring

Equipment Monitoring pages provide visibility into the operational status of equipment.

### Implemented UI Components & Functionalities

- Equipment list/table with name, location, and type
- Status indicators (Online, Offline, Maintenance, Alarm)
- Filters by equipment type, location, and status
- Quick navigation to equipment details

### Functional Description

- Displays real-time or near-real-time equipment health.
- Helps operators quickly identify faulty or offline equipment.
- Designed for day-to-day operational monitoring.

---

## Carbon Management

Carbon Management pages visualize carbon emissions derived from energy consumption.

### Implemented UI Components & Functionalities

- Carbon emission KPI cards
- Carbon trend charts
- Breakdowns by:
    - Equipment
    - Space
    - Energy category
- Period-over-period comparison views

### Functional Description

- Converts energy usage into carbon emission metrics.
- Supports sustainability tracking and compliance reporting.
- Read-only analytical views suitable for audits and reporting.

---

## Reports

Report pages provide structured views of historical data and exportable reports.

### Implemented UI Components & Functionalities

- Tabbed reports:
    - Energy
    - Cost
    - Carbon
    - Savings
- Advanced filters (entity, date range, category)
- Export actions (Excel/downloadable reports)
- Tabular data views

### Functional Description

- Enables formal reporting for management and regulatory audits.
- Supports exporting structured data for offline analysis.

---

## Tickets

Ticket pages support issue tracking and operational follow-ups.

### Implemented UI Components & Functionalities

- Ticket list table with:
    - Title
    - Status
    - Assigned user
    - Due date
- Status badges (Open, In Progress, Closed)
- Filters by status, priority, and assignee
- Navigation to ticket details

### Functional Description

- Used for maintenance, inspections, and operational issues.
- Supports assignment and progress tracking.
- This is the primary work-tracking feature in MyEMS.

---

## Settings Pages

Settings pages allow users to manage their personal account preferences.

### Implemented UI Components & Functionalities

- Profile settings (name, avatar, contact information)
- Security settings (change password, if enabled)
- Form validation with success/error feedback

### Functional Description

- Allows authenticated users to update personal account details.
- Access is restricted to logged-in users only.

---

## Admin Pages

Admin pages provide system-level configuration and control.

### Implemented UI Components & Functionalities

- User management (create, update, enable/disable users)
- Role and privilege views
- System configuration tables
- Administrative forms

### Functional Description

- Accessible only to Admin users.
- Used for managing users, permissions, and system configuration.
- Critical for platform governance and security.

---

## Template / Utility Pages

The following pages exist as part of the UI template and are not core MyEMS features.

### Included Pages

- E-Commerce Pages  
- Email Module  
- Event Management  
- People & Social Pages  
- Kanban Board  
- Utility Pages (Invoice, FAQ, Pricing, Landing Page)

### Functional Description

- Fully designed UI components and layouts.
- Primarily use static or mock data.
- Not integrated with MyEMS backend logic.

---

## Error & System Pages

System pages handle invalid routes and application-level errors.

### Implemented UI Components & Functionalities

- 404 – Page Not Found  
- 500 – Server Error  
- Coming Soon – Placeholder Page  

### Functional Description

- Provides safe fallback behavior during navigation or failures.
- Improves overall user experience and system robustness.

---