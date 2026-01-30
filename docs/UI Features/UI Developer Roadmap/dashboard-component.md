# UI Component Analysis 

## Admin Dashboard Templates

This document analyzes UI component structures and reusable patterns from the following React admin templates:

* Horizon UI (Chakra UI)
* Uko React (Material UI)
* Mantis React Admin (Material UI)

The goal is to identify **common UI components, layout systems, and reusable building blocks** that can be adopted in your application architecture.

---

# 1. Horizon UI – Chakra UI

**Source:** [https://github.com/horizon-ui/horizon-ui-chakra](https://github.com/horizon-ui/horizon-ui-chakra)

## Overview

Horizon UI is built entirely on **Chakra UI** and provides a modern dashboard design with heavy use of theme customization and compositional components.

---

## Common Components

### Core UI

* Buttons
* Inputs
* Icons
* Chakra-based custom variants

### Navigation

* Navbars
* Sidebars
* Persistent navigation layout components

### Content Containers

* Cards
* Panels
* Sectioned dashboard blocks for metrics

### Feedback

* Alerts
* Toasts (Chakra Toast system)

### Typography

* Headings
* Text
* Captions
* Chakra theme-driven styles

---

## Page / Feature Components

* Dashboard pages (metric cards + charts)
* Profile pages
* User forms
* Authentication (Login/Register)
* NFT / showcase/demo pages

---

# 2. Uko React – Material UI

**Source:** [https://github.com/uilibrary/uko-react](https://github.com/uilibrary/uko-react)

## Overview

Uko is built on **Material UI (MUI)** and provides a production-style admin system with many reusable enterprise components.

---

## Core MUI Components Used

### Navigation

* App Bar
* Sidebar (Drawer)
* Bottom navigation (optional)

### Layout

* Grid system
* Box
* Container wrappers

---

## Reusable UI Components

### Buttons & Icons

* Button variants
* IconButton

### Forms

* TextField
* Select
* Checkbox
* Radio
* Switch
* Auth forms (Login/Signup)

### Data Display

* Tables
* Cards
* Lists

### Feedback

* Alerts
* Snackbars
* Modals

### Charts

* Dashboard chart widgets
* Chart library integrations

---

## Feature Pages

* User lists
* Profile pages
* Dashboard analytics
* Authentication screens

---

# 3. Mantis React Admin – Material UI

**Source:** [https://mui.com/store/previews/mantis-free-react-admin-dashboard-template/](https://mui.com/store/previews/mantis-free-react-admin-dashboard-template/)

## Overview

Mantis is a clean, lightweight **MUI-based admin template** focusing on structured dashboards and responsive layouts.

---

# Core UI Component Categories

---

## Layout Components

### Sidebar Navigation

* Collapsible
* Expandable
* Multi-level menus

### Header (Top Navbar)

* Search
* Profile menu
* Notifications

### Footer

* Simple footer links

### Responsive Layout

* Desktop
* Tablet
* Mobile

---

## Dashboard Widgets & Cards

* Metric cards (users, revenue, stats)
* Icon + number + label pattern
* Progress indicators
* Chart placeholders

---

## Data Visualization

Typically integrates:

* Line charts
* Bar charts
* Pie charts
* Summary graphs

Usually built using:

* ApexCharts or similar

---

## Tables / Lists

* Basic data tables
* Row actions
* Pagination
* MUI Table / DataGrid

---

## Forms / Inputs

* TextField
* Select
* Checkbox / Radio
* Buttons
* Validation (Formik or similar)

---

## Authentication Screens

* Login
* Register
* Forgot password

Built using:

* Paper
* FormControl
* TextField
* Button

---

## Theme & Utility Components

* Light/Dark switch
* Breadcrumbs
* Typography pages
* Color palette pages

---
