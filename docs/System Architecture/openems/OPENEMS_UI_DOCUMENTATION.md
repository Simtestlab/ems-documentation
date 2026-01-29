# OpenEMS UI - Comprehensive Documentation

## Overview

OpenEMS UI is a modern, feature-rich Angular 20-based web application with mobile app capabilities built using Ionic and Capacitor. It provides comprehensive monitoring, control, and configuration interfaces for the OpenEMS Energy Management System.

## Technology Stack

### Core Technologies
- **Framework**: Angular 20.3.16
- **Mobile Framework**: Ionic 8.7.17
- **Mobile Native**: Capacitor 7.4.5 (Android & iOS)
- **Language**: TypeScript 5.8.3
- **PWA Support**: Service Workers enabled

### Key Libraries & Tools

#### Visualization & Charts
- **Chart.js**: 4.5.1 - Primary charting library
- **ng2-charts**: 8.0.0 - Angular wrapper for Chart.js
- **chartjs-adapter-date-fns**: 3.0.0 - Date handling for charts
- **chartjs-plugin-annotation**: 3.1.0 - Chart annotations
- **chartjs-plugin-datalabels**: 2.2.0 - Data labels on charts
- **chartjs-plugin-zoom**: 2.2.0 - Zoom functionality
- **d3**: 7.9.0 - Advanced data visualizations

#### UI Components & Forms
- **@ngx-formly/core**: 7.0.1 - Dynamic forms
- **@ngx-formly/ionic**: 7.0.1 - Ionic integration for forms
- **@angular-mydatepicker**: 20.0.0 - Date picker component

#### Internationalization & Localization
- **@ngx-translate/core**: 17.0.0 - Translation framework
- **date-fns**: 4.1.0 - Date manipulation and formatting
- **@date-fns/tz**: 1.4.1 - Timezone support

#### State Management & Data
- **rxjs**: 7.8.2 - Reactive programming
- **ngx-cookie-service**: 20.1.1 - Cookie management
- **ngx-device-detector**: 10.1.0 - Device detection

#### Native Capabilities (Capacitor Plugins)
- **@capacitor/app**: 7.1.1
- **@capacitor/filesystem**: 7.1.6
- **@capacitor/splash-screen**: 7.0.4
- **capacitor-blob-writer**: 1.1.19
- **capacitor-secure-storage-plugin**: 0.12.0
- **capacitor-ios-autofill-save-password**: 5.0.0

---

## Application Architecture

### 1. Main Application Structure

#### Routing Structure (`app-routing.module.ts`)
```
/
├── index (LoadingScreen)
├── login (Authentication)
├── overview (Dashboard)
├── device/:edgeId (Edge Device View)
│   ├── live (Real-time monitoring)
│   ├── history (Historical data & charts)
│   └── settings (Configuration)
├── user (User profile & settings)
└── changelog (Release notes)
```

### 2. Core Application Components

#### App Module (`app.module.ts`)
The main application module includes:
- Angular core modules (BrowserModule, BrowserAnimationsModule)
- HTTP Client for API communication
- Translation services with German locale support
- Chart.js with default registerables
- Ionic routing strategy
- Edge, Index, User, and Settings modules

#### Platform Service (`platform.service.ts`)
- Detects platform (web, Android, iOS)
- Handles device information
- Manages Capacitor integration

---

## Live View Features & Widgets

### Energy Monitor (`energymonitor.component.ts`)
The central energy flow visualization component displaying real-time energy data.

**Monitored Channels:**
- ESS (Energy Storage System): SoC, Active Power, Min/Max Discharge Power
- Grid: Active Power, Min/Max Active Power, Grid Mode
- Production: AC/DC Active Power, Max Active Power
- Consumption: Active Power, Max Active Power

**Features:**
- Real-time data streaming via WebSocket
- Animated energy flow visualization
- Three-column responsive layout
- Refresh drag-down functionality
- Offline detection and handling

### Available Widget Classes

#### Core Widgets (Always Visible)
1. **Energymonitor** - Central energy flow diagram
2. **Common_Autarchy** - Self-sufficiency metrics
3. **Common_Selfconsumption** - Self-consumption statistics
4. **Storage** - Battery/storage information
5. **Grid** - Grid connection metrics
6. **Common_Production** - Solar/production data
7. **Consumption** - Consumption breakdown

#### Controller Widgets (Configuration-Dependent)
8. **EVCS (Electric Vehicle Charging Station)**
   - Single charging stations
   - Cluster management (Peak Shaving & Self Consumption)
   - Schedule management
   - Phase switching
   - Energy limits
   - Charge mode configuration
   - Forecast capabilities

9. **Peak Shaving Controllers**
   - Symmetric Peak Shaving
   - Asymmetric Peak Shaving
   - Time-slot Peak Shaving

10. **ESS Controllers**
    - Fix Active Power
    - Grid Optimized Charge
    - Time-Of-Use-Tariff (ToU)
    - Delayed Sell To Grid

11. **Heat Management**
    - Heat Pump (SG Ready)
    - Heating Element
    - Heating Room Controller
    - Heat MyPV

12. **IO Controllers**
    - Digital Input/Output
    - Fix Digital Output
    - Channel Single Threshold
    - Channel Threshold

13. **Additional Controllers**
    - Modbus TCP API
    - CHP SoC (Combined Heat and Power)
    - Enerix Control (Clever-PV)
    - Weather (OpenMeteo integration)

### Widget Architecture

#### Widget Types Enum
```typescript
enum WidgetClass {
    "Energymonitor",
    "Common_Autarchy",
    "Common_Selfconsumption",
    "Storage",
    "Grid",
    "Common_Production",
    "Consumption",
    "Controller_ChannelThreshold",
    "Controller_Io_Digital_Outputs"
}

enum WidgetNature {
    "io.openems.edge.evcs.api.Evcs",
    "io.openems.edge.heat.api.ManagedHeatElement",
    "io.openems.impl.controller.channelthreshold.ChannelThresholdController",
    "io.openems.edge.io.api.DigitalInput"
}

enum WidgetFactory {
    "Evse.Controller.Single",
    "Evse.Controller.Cluster",
    "Controller.Api.ModbusTcp.ReadWrite",
    "Controller.Asymmetric.PeakShaving",
    "Controller.ChannelThreshold",
    "Controller.CHP.SoC",
    "Controller.Clever-PV",
    "Controller.Ess.DelayedSellToGrid",
    "Controller.Ess.FixActivePower",
    "Controller.Ess.GridOptimizedCharge",
    "Controller.Ess.Time-Of-Use-Tariff.Discharge",
    "Controller.Ess.Time-Of-Use-Tariff",
    "Controller.IO.ChannelSingleThreshold",
    "Controller.Io.FixDigitalOutput",
    "Controller.IO.HeatingElement",
    "Controller.IO.Heating.Room",
    "Controller.Io.HeatPump.SgReady",
    "Controller.Heat.Heatingelement",
    "Controller.Symmetric.PeakShaving",
    "Controller.TimeslotPeakshaving",
    "Evcs.Cluster.PeakShaving",
    "Evcs.Cluster.SelfConsumption",
    "Weather.OpenMeteo"
}
```

#### Widget Rendering
Widgets are dynamically rendered based on:
- Edge configuration
- Component factory IDs
- Implemented natures
- User permissions
- Component enabled status

### Live Component Features
- **Pull-to-Refresh**: Manual data refresh capability
- **Auto-Refresh Check**: Monitors data freshness (20-second threshold)
- **Responsive Layout**: 3-column grid (Energymonitor, Widgets, Weather)
- **Online/Offline Detection**: Displays offline message when edge is unavailable
- **Progress Indicator**: Shows when channels are being subscribed

---

## History View & Data Visualization

### History Component (`history.component.ts`)
Provides historical data visualization and analysis capabilities.

**Key Features:**
- Timedata availability check
- Dynamic widget loading based on configuration
- Responsive chart sizing
- Multi-period data queries (Day, Week, Month, Year)
- Energy and power charts

### Historical Chart Types

#### Common History Charts
1. **Autarchy Chart** - Historical self-sufficiency
2. **Consumption Chart** - Historical consumption with phase-accurate details
3. **Grid Chart** - Grid import/export history
   - External limitation details
   - Phase-accurate breakdown
4. **Production Chart** - Solar/generator production history
5. **Self-consumption Chart** - Self-consumption trends
6. **Storage Chart** - Battery charge/discharge history

#### Controller-Specific History Charts
7. **Asymmetric Peak Shaving Chart**
8. **Symmetric Peak Shaving Chart**
9. **Time-slot Peak Shaving Chart**
10. **Delayed Sell To Grid Chart**
11. **Grid Optimized Charge Chart**
12. **Time-Of-Use-Tariff (Schedule) Chart**
13. **Heating Element Chart**
14. **Heat Pump Chart**
15. **Heat MyPV Chart**
16. **Enerix Control Chart**
17. **Modbus TCP API Chart**
18. **Channel Threshold Chart**
19. **Digital Output Chart**
20. **Current & Voltage Chart** (Phase-accurate meter data)

### Chart Architecture

#### AbstractHistoryChart (`abstracthistorychart.ts`)
Base class for all historical charts with 1353 lines of functionality.

**Core Features:**
- Chart.js integration with plugins (annotation, datalabels, zoom)
- Automatic resolution calculation based on time period
- Energy and power data queries
- Dynamic dataset creation
- Legend management
- Tooltip customization
- Responsive sizing
- Data caching and optimization

**Chart Types Supported:**
- Line charts (default)
- Bar charts
- Combined line/bar charts

**Plugins:**
- **chartjs-adapter-date-fns**: Time series handling
- **chartjs-plugin-annotation**: Add markers and regions
- **chartjs-plugin-datalabels**: Show values on data points
- **chartjs-plugin-zoom**: Pan and zoom functionality

#### Chart Constants (`chart.constants.ts`)
Defines standard configurations, colors, and utilities for charts.

**Key Constants:**
- `NUMBER_OF_Y_AXIS_TICKS`: 7
- `MAX_LENGTH_OF_Y_AXIS_TITLE`: 6
- `REQUEST_TIMEOUT`: 500ms

**Utility Classes:**
- `Plugins.Legend`: Legend customization
- `Plugins.Datasets`: Dataset styling
- `Plugins.ToolTips`: Tooltip formatting
- Phase colors: `["rgb(255,127,80)", "rgb(91, 92, 214)", "rgb(128,128,0)"]`

#### Chart Component (`chart.ts`)
Wrapper component providing:
- Chart header with title
- Period selection
- Phase/Total toggle
- Chart options popover
- Navigation integration

### History Data Service (`historydataservice.ts`)
Manages historical data queries and caching.

**Responsibilities:**
- Query historic timeseries data
- Query historic energy data
- Query energy per period
- Handle response caching
- Manage refresh requests

---

## Settings & Configuration UI

### Settings Component Structure

#### Main Settings View (`settings.component.ts`)
Role-based configuration menu with the following sections:

**Available for All Users:**
1. **System Profile** - View edge information and components
2. **Changelog** - View UI version history

**Installer Role & Above:**
3. **Network Configuration** - Network settings management

**Owner Role & Above:**
4. **Live Log** - Real-time system logs
5. **System** - System updates and maintenance
6. **Edge Apps** - Application installation and management
7. **Alerting** - Alert configuration

**Admin Role & Above:**
8. **Channels** - View all available channels
9. **Add Components** - Install new components
10. **Adjust Components** - Configure existing components
11. **System Execute** - Execute system commands
12. **Power Assistant** - Power configuration wizard

### Component Configuration

#### Component Management
- **Install**: Add new components with factory-based selection
- **Update**: Modify existing component configuration
- **Formly Integration**: Dynamic forms based on component schema

#### Network Configuration (`network.component.ts`)
- Get network information
- Configure network settings
- DHCP/Static IP configuration
- DNS settings

#### System Management

##### System Component (`system.component.ts`)
- System updates
- Update state monitoring
- Maintenance mode
- System information display

##### System Log (`systemlog.component.ts`)
- Live log streaming
- Log level filtering
- Real-time updates via WebSocket

##### System Execute (`systemexecute.component.ts`)
- Execute system commands
- View execution results

#### App Management
- **App Index**: Browse available apps
- **App Install**: Install new apps with key management
- **App Update**: Update existing apps
- **App Single View**: Detailed app information
- **OAuth Configuration**: OAuth provider setup

#### Alerting Module
- Configure alert rules
- Notification settings
- Alert history

#### Profile Management
- View component aliases
- Update component names
- Export channel information (XLSX)
- Export Modbus protocol information (XLSX)

---

## Edge Management UI

### Edge Component (`edge.component.ts`)
Container component for all edge-specific views.

**Child Routes:**
- `/device/:edgeId/live` - Live monitoring
- `/device/:edgeId/history` - Historical data
- `/device/:edgeId/settings` - Configuration

### Edge Features

#### Edge Data Service
Abstract service providing:
- WebSocket subscription management
- Channel data retrieval
- Real-time data updates
- Refresh handling

#### Edge Permission System
Role-based access control:
- **Guest**: View-only access
- **Installer**: Basic configuration
- **Owner**: Full configuration and app management
- **Admin**: Complete system access

---

## User Authentication UI

### Login Component (`login.component.ts`)
Comprehensive authentication interface.

**Features:**
- Username/password authentication
- OAuth 2.0 support (callback handling)
- Theme selection (Light, Dark, System)
- Credential preprocessing
- Session management
- Cookie-based state preservation
- iOS/Android specific autofill support
- Password visibility toggle

**Authentication Flow:**
1. User enters credentials
2. WebSocket authentication request
3. Session token storage (secure storage on mobile)
4. Edge list retrieval
5. Redirect to overview or saved state

### OAuth Integration (`oauth.service.ts`)
- OAuth 2.0 flow handling
- Deep link support for mobile apps
- State parameter validation
- Token exchange

---

## User Profile & Settings

### User Component (`user.component.ts`)
Comprehensive user management interface (348 lines).

**Features:**

#### Theme Management
- Light mode
- Dark mode
- System theme (follows OS settings)
- Theme preview images

#### Language Selection
Supported languages:
- German (de)
- English (en)
- Czech (cz)
- Spanish (es)
- French (fr)
- Japanese (ja)
- Dutch (nl)

#### User Information
- Firstname, Lastname
- Email, Phone
- Address (Street, ZIP, City, Country)
- Company name (for company users)

#### UI Preferences
- New navigation toggle
- Navigation position (left sidebar vs. new navigation)

#### Version Information
- UI version display
- Changelog access
- Environment information

---

## Widget Components & Shared UI

### Flat Widget Components
Located in `shared/components/flat/`:

1. **AbstractFlatWidget**: Base class for live view widgets
2. **FlatWidgetButton**: Action buttons within widgets
3. **FlatWidgetHorizontalLine**: Separator lines
4. **FlatWidgetLine**: Information display lines
5. **FlatWidgetLineDivider**: Section dividers
6. **FlatWidgetPercentagebar**: Progress bars for percentages

### Modal Components
Located in `shared/components/modal/`:

1. **AbstractModal**: Base class for modal dialogs
2. **ModalButton**: Action buttons in modals
3. **ModalInfoLine**: Information display in modals
4. **ModalLine**: Generic content lines
5. **ModalPhases**: Phase-specific data display
6. **ModalValueLine**: Value display lines
7. **ModalHorizontalLine**: Separators
8. **HelpButton**: Contextual help buttons
9. **HelpLink**: Help documentation links

### Navigation Components

#### NavigationService (`navigation.service.ts`)
- Widget filtering and sorting
- Navigation tree generation
- View management
- Position tracking (left sidebar vs. new navigation)

#### NavigationRoutingModule
Defines routes for the new navigation system:
- EVSE charge point pages
- Common widgets (Autarchy, Grid, Consumption, etc.)
- History views
- IoT device pages

#### Footer Navigation (`footerNavigation`)
- Period selection (Day, Week, Month, Year)
- Date picker
- Quick navigation buttons

### Shared Components

#### Chart Components
- **ChartComponent**: Main chart wrapper
- **ChartLegend**: Custom legend rendering
- **AbstractHistoryChartOverview**: History chart container

#### Edge Components
- **Meter Components**: Current/voltage display
- **Status Components**: Connection status indicators
- **Edge Info**: Edge device information

#### Form Components (Formly)
- Dynamic form generation
- Validation
- Custom field types
- Ionic integration

#### Utility Components
- **OeCheckbox**: Custom checkbox
- **OeImg**: Image component with error handling
- **Percentagebar**: Progress visualization
- **PickDateTimeRange**: Date range picker
- **Pickdate**: Date picker
- **PullToRefresh**: Pull-to-refresh gesture

---

## Mobile App Capabilities (Capacitor)

### Platform Support
- **Android**: Full support via Capacitor 7.4.5
- **iOS**: Full support via Capacitor 7.4.5
- **Web**: PWA with service worker

### Native Features

#### App Lifecycle
- Splash screen with configurable duration
- Background/foreground detection
- Deep linking support

#### Storage & Security
- **Secure Storage**: Encrypted storage for sensitive data
- **Filesystem Access**: Read/write files
- **Cookie Management**: Persistent cookies across sessions

#### iOS-Specific Features
- Autofill save password support
- iOS keychain integration

#### Android-Specific Features
- Edge-to-edge margin adjustment
- Custom splash screen configuration

### Mobile Build Configuration

#### Capacitor Config (`capacitor.config.ts`)
```typescript
{
  webDir: 'target',
  server: {
    androidScheme: 'https',
    iosScheme: 'https'
  },
  android: {
    adjustMarginsForEdgeToEdge: 'auto'
  },
  plugins: {
    SplashScreen: {
      launchShowDuration: 1000,
      launchAutoHide: false,
      backgroundColor: "#ffffffff"
    },
    CapacitorCookies: {
      enabled: true
    }
  }
}
```

#### Build Process
1. Build Angular app: `ng build`
2. Copy to Capacitor: `ionic cap sync`
3. Build native apps: `ionic cap build android/ios`
4. Asset generation: `npx @capacitor/assets generate`

### Mobile-Specific Optimizations
- Touch-optimized UI components
- Responsive breakpoints (sm, md, lg, xl)
- Mobile-friendly gestures (swipe, pull-to-refresh)
- Native navigation patterns

---

## Charts & Visualization Libraries

### Chart.js Integration

#### Version & Configuration
- **Chart.js**: 4.5.1
- **ng2-charts**: 8.0.0 (Angular wrapper)
- Default registerables included

#### Chart Types Used
1. **Line Charts**: Time series data (power, energy)
2. **Bar Charts**: Aggregated data (daily, monthly totals)
3. **Combined Charts**: Mixed line and bar visualizations

#### Chart.js Plugins

##### 1. chartjs-adapter-date-fns (3.0.0)
- Date/time axis formatting
- Timezone support via @date-fns/tz
- Automatic time unit selection

##### 2. chartjs-plugin-annotation (3.1.0)
- Add lines, boxes, and labels to charts
- Highlight specific time periods
- Mark thresholds and limits

##### 3. chartjs-plugin-datalabels (2.2.0)
- Display values on data points
- Customizable label formatting
- Conditional visibility based on resolution

##### 4. chartjs-plugin-zoom (2.2.0)
- Pan and zoom functionality
- Mouse wheel zoom
- Touch gesture support
- Reset zoom button

### D3.js Integration (7.9.0)
Used for advanced visualizations:
- Custom energy flow diagrams
- Complex data transformations
- SVG-based graphics

### Chart Features

#### Time Series Charts
- Automatic resolution calculation (5min, 15min, 1hour, 1day, 1month)
- Dynamic data aggregation
- Gap filling for missing data
- Tooltip with timestamp and values

#### Energy Charts
- Cumulative energy display
- Period-based aggregation
- Unit conversion (Wh, kWh, MWh)

#### Multi-Axis Charts
- Left and right Y-axes
- Different units on same chart
- Color-coded datasets

#### Responsive Design
- Dynamic height calculation
- Breakpoint-based sizing
- Mobile-optimized touch controls

### Visualization Utilities

#### ChartAxis Enum
```typescript
enum ChartAxis {
    LEFT = "left",
    RIGHT = "right"
}
```

#### Color Utilities (`color.utils.ts`)
- Phase color assignment
- RGB/Hex conversion
- Opacity manipulation
- Color palette generation

#### Chart Data Utilities (`utils.ts`)
- Empty dataset creation
- Dataset conversion
- Value formatting
- Axis configuration

---

## Internationalization (i18n)

### Translation Framework
- **@ngx-translate/core**: 17.0.0
- Runtime language switching
- Lazy-loaded translation files

### Supported Languages
1. **German (de)**: Primary language
2. **English (en)**: International default
3. **Czech (cz)**
4. **Spanish (es)**
5. **French (fr)**
6. **Japanese (ja)**
7. **Dutch (nl)**

### Translation Structure
Located in `assets/i18n/*.json`:
- Hierarchical JSON structure
- Component-specific translations
- Common translations shared across views
- Controller and widget-specific translations

### Usage Examples

#### In HTML Templates
```html
<p translate>General.storageSystem</p>
```

#### In TypeScript
```typescript
this.translate.instant('General.storageSystem')
```

#### Translation Keys Structure
```
General.*
MENU.*
EDGE.INDEX.WIDGETS.*
EDGE.HISTORY.*
EDGE.SETTINGS.*
CONTROLLER.*
```

---

## Data Services & State Management

### Service Architecture

#### Service (`service.ts`)
Core service providing:
- Current edge management (signals)
- Configuration retrieval
- Period management
- Metadata handling
- WebSocket integration

#### WebSocket Service (`websocket.ts`)
- Connection management
- Authentication
- Channel subscriptions
- Real-time data streaming
- Automatic reconnection
- iOS autofill password saving integration

#### LiveDataService (`livedataservice.ts`)
Extends DataService for live view:
- Current value queries
- Channel subscriptions
- Real-time updates

#### HistoryDataService (`historydataservice.ts`)
Extends DataService for history view:
- Historical data queries
- Time-range selection
- Data caching
- Energy calculations

### State Management

#### RxJS-based State
- Subject/BehaviorSubject for reactive data
- Signals for Angular 20 reactivity
- Observable streams for async operations

#### UserService (`user.service.ts`)
- Current user management (signals)
- Role checking
- Theme preferences
- Language preferences
- Navigation mode

#### AppStateTracker (`states.ts`)
WebSocket connection states:
- INITIAL
- CONNECTING
- CONNECTED
- AUTHENTICATED
- AUTHENTICATION_WITH_CREDENTIALS
- FAILED
- CLOSED

---

## Advanced Features

### Formly Dynamic Forms
- Component configuration forms
- Dynamic validation
- Custom field types
- Ionic styling integration

### Progressive Web App (PWA)
- Service worker registration
- Offline support
- Update notifications via `CheckForUpdateService`
- Manifest configuration

### Performance Optimizations
- Lazy loading of modules
- Component-level code splitting
- Change detection optimization
- Virtual scrolling for large lists
- Image lazy loading

### Error Handling
- Custom error handler (`MyErrorHandler`)
- Error reporting
- User-friendly error messages
- Fallback UI states

---

## Navigation Patterns

### Old Navigation (Classic)
- Tab-based navigation
- Flat widget display
- Modal-based details

### New Navigation (Modern)
- Hierarchical routing
- Dedicated pages for widgets
- Breadcrumb navigation
- Back button support
- Deep linking

### Navigation Toggle
Users can switch between navigation modes in user settings.

---

## Testing

### Test Configuration
- **Jasmine**: Test framework
- **Karma**: Test runner
- **Karma Chrome Launcher**: Browser testing

### Test Scripts
```bash
ng test                 # Standard testing
ng test -c "local"      # Testing with Karma UI
```

---

## Build Configurations

### Development Builds

#### OpenEMS Edge (Local)
```bash
ng serve -c openems-edge-dev
```
Connects to Edge websocket on port 8085.

#### OpenEMS Backend (Remote)
```bash
ng serve -c openems-backend-dev
```
Connects to Backend websocket on port 8082.

### Production Builds

#### Edge Production
```bash
ng build -c "openems,openems-edge-prod,prod"
```

#### Backend Production
```bash
ng build -c "openems,openems-backend-prod,prod"
```

### Mobile App Builds

#### Android
```bash
NODE_ENV=$theme ionic cap build android -c "$theme,$theme-backend-deploy-app"
THEME="{$theme}" gradlew bundleThemeRelease
```

#### iOS
```bash
NODE_ENV=$theme ionic cap build ios -c "$theme,$theme-backend-deploy-app"
```

---

## Key Widgets Detailed

### EVSE (Electric Vehicle Charging)

#### Pages & Features
1. **Home**: Current status, charge control
2. **History**: Charging session history with charts
3. **Schedule**: Time-based charging schedules
4. **Forecast**: Prediction-based charging
5. **Energy Limit**: Set energy limits
6. **Phase Switching**: 1-phase/3-phase switching
7. **Charge Mode**: Mode selection
8. **Car Config**: Vehicle-specific settings

#### Components
- `EvseSingleComponent`: Single charge point
- `EvseClusterComponent`: Multiple charge points
- `ScheduleComponent`: Schedule management
- `TaskFormComponent`: Schedule task creation
- Charts: Power chart, Status chart, Schedule chart

### Storage Management

#### Features
- Real-time SoC (State of Charge)
- Charge/discharge power
- Capacity information
- Admin modal for advanced settings
- Owner/installer/guest-specific views

### Weather Widget (OpenMeteo)

#### Features
- Current weather conditions
- Hourly forecast
- Daily forecast (7 days)
- Weather icons and descriptions
- Temperature, humidity, wind
- Integration with PV production predictions

---

## Summary of UI Capabilities

### Live View Capabilities
✅ Real-time energy flow visualization
✅ 40+ dynamic widgets based on configuration
✅ EV charging management (EVSE)
✅ Battery/storage monitoring
✅ Peak shaving controls
✅ Heat pump and heating controls
✅ Weather integration
✅ Digital I/O monitoring
✅ Grid management
✅ Production and consumption breakdown
✅ Auto-refresh with staleness detection
✅ Pull-to-refresh gesture support
✅ Offline detection and messaging

### History View Capabilities
✅ 20+ historical chart types
✅ Multiple time periods (Day, Week, Month, Year, Custom)
✅ Phase-accurate data visualization
✅ Energy and power charts
✅ Zooming and panning
✅ Data export capabilities
✅ Controller-specific history views
✅ Current and voltage analysis
✅ Production forecasting
✅ Consumption patterns

### Settings & Configuration
✅ Role-based access control (Guest/Installer/Owner/Admin)
✅ Component installation and configuration
✅ Network configuration
✅ System updates
✅ Live system logs
✅ App marketplace integration
✅ Alert configuration
✅ Profile management
✅ Channel export (XLSX)
✅ Modbus protocol export

### Mobile App Features
✅ Native Android app
✅ Native iOS app
✅ Secure storage
✅ Biometric authentication support (iOS)
✅ Push notifications capability
✅ Offline mode with PWA
✅ Deep linking
✅ Native file operations

### User Experience
✅ 7 language support
✅ Light/Dark/System themes
✅ Responsive design (mobile, tablet, desktop)
✅ Touch-optimized controls
✅ Keyboard navigation support
✅ Accessibility features
✅ Progressive Web App (installable)
✅ Real-time updates via WebSocket

### Developer Experience
✅ Angular 20 with TypeScript
✅ Modular architecture
✅ Lazy loading
✅ Type-safe code
✅ Comprehensive error handling
✅ Testing infrastructure
✅ Hot reload development
✅ Multi-environment builds
✅ Theme system for white-labeling

---

## File Structure Overview

```
ui/
├── src/
│   ├── app/
│   │   ├── app.component.ts          # Root component
│   │   ├── app.module.ts             # Main module
│   │   ├── app-routing.module.ts    # Root routing
│   │   ├── edge/                    # Edge device views
│   │   │   ├── live/               # Live monitoring
│   │   │   │   ├── energymonitor/  # Energy flow diagram
│   │   │   │   ├── common/         # Common widgets (grid, storage, etc.)
│   │   │   │   ├── Controller/     # Controller widgets
│   │   │   │   ├── Io/             # I/O widgets
│   │   │   │   └── Multiple/       # Cluster widgets
│   │   │   ├── history/            # Historical data views
│   │   │   │   ├── common/         # Common history charts
│   │   │   │   └── Controller/     # Controller history charts
│   │   │   └── settings/           # Configuration UI
│   │   │       ├── component/      # Component management
│   │   │       ├── network/        # Network settings
│   │   │       ├── system/         # System management
│   │   │       ├── app/            # App marketplace
│   │   │       └── alerting/       # Alert config
│   │   ├── index/                  # Login & overview
│   │   ├── user/                   # User profile
│   │   ├── shared/                 # Shared components
│   │   │   ├── components/         # Reusable components
│   │   │   │   ├── chart/          # Chart components
│   │   │   │   ├── flat/           # Flat widget components
│   │   │   │   ├── modal/          # Modal components
│   │   │   │   └── navigation/     # Navigation components
│   │   │   ├── service/            # Services
│   │   │   ├── type/               # Type definitions
│   │   │   └── utils/              # Utility functions
│   │   └── changelog/              # Release notes
│   ├── assets/
│   │   ├── i18n/                   # Translation files
│   │   └── img/                    # Images
│   └── themes/                     # Theme configurations
├── android/                        # Android app
├── ios/                           # iOS app
├── capacitor.config.ts            # Capacitor configuration
├── angular.json                   # Angular CLI configuration
├── ionic.config.json              # Ionic configuration
└── package.json                   # Dependencies
```

---

## Conclusion

OpenEMS UI is a comprehensive, production-ready energy management interface that successfully combines:

- **Modern Web Technologies**: Angular 20, TypeScript, Ionic
- **Mobile-First Design**: Native iOS/Android apps via Capacitor
- **Rich Visualizations**: Chart.js with multiple plugins, D3.js
- **Extensive Widget System**: 40+ widgets for various energy management scenarios
- **Role-Based Access**: Secure, multi-level permission system
- **Internationalization**: 7 languages supported
- **Real-Time Updates**: WebSocket-based live data streaming
- **Historical Analysis**: Comprehensive charting and data export
- **Configurability**: Dynamic forms, component management
- **User Experience**: Themes, responsive design, offline support

The application architecture is well-organized, maintainable, and scalable, supporting both edge and backend deployments, making it suitable for residential, commercial, and industrial energy management scenarios.
