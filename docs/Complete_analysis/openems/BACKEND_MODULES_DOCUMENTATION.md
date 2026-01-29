# OpenEMS Backend Modules Documentation

This document provides a comprehensive overview of all OpenEMS backend modules, their features, and capabilities.

## Table of Contents

1. [Authentication Mechanisms](#1-authentication-mechanisms)
2. [API Endpoints](#2-api-endpoints)
3. [Time-Series Data Storage](#3-time-series-data-storage)
4. [Metadata Management](#4-metadata-management)
5. [Alerting and Monitoring](#5-alerting-and-monitoring)
6. [Edge Manager Capabilities](#6-edge-manager-capabilities)
7. [Metrics and Prometheus Integration](#7-metrics-and-prometheus-integration)
8. [Additional Backend Modules](#8-additional-backend-modules)

---

## 1. Authentication Mechanisms

### 1.1 Authentication API (`io.openems.backend.authentication.api`)

**Module**: `io.openems.backend.authentication.api`

**Description**: Core authentication interface definitions for the OpenEMS backend.

**Key Features**:
- Defines authentication service interfaces
- Provides base models for authentication:
  - `AuthUserRegistrationService` - User registration
  - `AuthUserAuthorizationCodeFlowService` - OAuth2 authorization code flow
  - `AuthUserPasswordAuthenticationService` - Password-based authentication
- Authentication result models (`OAuthToken`, `PasswordAuthenticationResult`)
- Registration request/response models

**Interfaces**:
```java
- AuthUserRegistrationService
- AuthUserAuthorizationCodeFlowService  
- AuthUserPasswordAuthenticationService
```

---

### 1.2 OAuth2 Authentication (`io.openems.backend.authentication.oauth2`)

**Module**: `io.openems.backend.authentication.oauth2`

**Description**: OAuth2 and JWT-based authentication implementation using Keycloak.

**Key Features**:
- **OAuth2 Integration**: Full OAuth2 authorization code flow with PKCE support
- **JWT Token Verification**: Uses Auth0 JWT library for token validation
  - RSA public key verification
  - JWK provider with rate limiting
  - Token issuer validation
- **Keycloak Integration**: 
  - Configurable Keycloak realm and base URL
  - Custom token and certificate endpoints
  - Multi-OEM support with separate OAuth configurations
- **User Authentication Flow**:
  - Authorization code generation with code verifier (PKCE)
  - Token exchange and validation
  - User registration and session management
- **Concurrent Request Handling**: Configurable max concurrent requests and thread pooling

**Main Components**:
- `OAuthUserAuthenticationServiceImpl` - Main OAuth2 service implementation
- `OAuthOemService` - OEM-specific OAuth configuration
- JWT verification with RSA key provider
- HTTP bridge for external OAuth provider communication

**Configuration**:
```properties
baseKeycloakUrl: Base URL of Keycloak server
realm: Keycloak realm name
issuerUrl: Token issuer URL
loginUrl: OAuth login endpoint
tokenUrl: Token exchange endpoint
certsUrl: JWK certificates URL
maxConcurrentRequests: Maximum concurrent HTTP requests
```

**Security Features**:
- PKCE (Proof Key for Code Exchange) support
- Secure code verifier generation
- Token refresh mechanism
- Rate-limited JWK provider
- RSA signature verification

---

## 2. API Endpoints

### 2.1 Backend-to-Backend REST API (`io.openems.backend.b2brest`)

**Module**: `io.openems.backend.b2brest`

**Description**: RESTful API for backend-to-backend communication using JSON-RPC over HTTP.

**Key Features**:
- **HTTP REST Endpoint**: Jetty-based HTTP server
- **Basic Authentication**: Username/password authentication via HTTP Basic Auth
- **JSON-RPC Protocol**: Supports JSON-RPC 2.0 over HTTP POST
- **Request Handling**: Routes to internal JSON-RPC request handler

**Endpoints**:

#### `/jsonrpc` - JSON-RPC Endpoint

**Authentication**: HTTP Basic Authentication

**Supported Methods**:

1. **getEdgesStatus**
   ```json
   {
     "method": "getEdgesStatus",
     "params": {}
   }
   ```
   Returns the online/offline status of all edges.

2. **getEdgesChannelsValues**
   ```json
   {
     "method": "getEdgesChannelsValues",
     "params": {
       "ids": ["edge0"],
       "channels": ["_sum/State"]
     }
   }
   ```
   Returns current channel values for specified edges.

**Configuration**:
```properties
port: HTTP server port (default: 8075)
```

**Implementation Details**:
- Uses Jetty Server for HTTP handling
- Authenticates via `AuthUserPasswordAuthenticationService`
- Auto-adds missing JSON-RPC fields (jsonrpc, id)
- Timeout: 1 minute for authentication
- Returns JSON-RPC success/error responses

---

### 2.2 Backend-to-Backend WebSocket (`io.openems.backend.b2bwebsocket`)

**Module**: `io.openems.backend.b2bwebsocket`

**Description**: WebSocket server for real-time backend-to-backend communication.

**Key Features**:
- **WebSocket Protocol**: Full-duplex communication channel
- **Connection Management**: Handles WebSocket lifecycle (OnOpen, OnMessage, OnClose, OnError)
- **Configurable Threading**: Thread pool for concurrent connection handling
- **Event-Driven Architecture**: Integrates with OSGi EventHandler

**Configuration**:
```properties
port: WebSocket server port (default: 8081)
poolSize: Thread pool size for connection handling
```

**Main Components**:
- `Backend2BackendWebsocket` - Main WebSocket server component
- `WebsocketServer` - Server implementation
- `OnOpen`, `OnClose`, `OnMessage`, `OnError` - WebSocket event handlers

---

### 2.3 UI WebSocket (`io.openems.backend.uiwebsocket`)

**Module**: `io.openems.backend.uiwebsocket`

**Description**: WebSocket API for user interface communication with authentication and session management.

**Key Features**:
- **User Authentication**: Supports multiple authentication methods:
  - OAuth2 authorization code flow
  - Password-based authentication
  - User registration
- **Real-Time Communication**: WebSocket for live data streaming
- **Session Management**: User session tracking and validation
- **JSON-RPC Integration**: Handles UI JSON-RPC requests
- **Edge Data Access**: Routes requests to edge devices
- **Time-Series Data**: Query historical data from TimedataManager
- **Simulation Support**: Optional simulation engine integration

**Authentication Handlers**:
- `OAuthAuthenticationHandler` - OAuth2 flow handling
- Session-based user identification
- Token validation and refresh

**Main Components**:
- `UiWebsocketImpl` - Main UI WebSocket implementation
- `WebsocketServer` - WebSocket server for UI connections
- `OnRequest` - Request handler for JSON-RPC
- `OnNotification` - Notification handler
- `WsData` - WebSocket connection data

**References**:
- `AuthUserRegistrationService`
- `AuthUserAuthorizationCodeFlowService`
- `AuthUserPasswordAuthenticationService`
- `JsonRpcRequestHandler`
- `Metadata`
- `EdgeManager`
- `TimedataManager`

---

## 3. Time-Series Data Storage

### 3.1 InfluxDB Time-Series Storage (`io.openems.backend.timedata.influx`)

**Module**: `io.openems.backend.timedata.influx`

**Description**: InfluxDB v2.x integration for time-series data storage and querying.

**Key Features**:
- **InfluxDB v2.x Client**: Uses official InfluxDB Java client
- **Write Operations**: 
  - Point-based data writing with timestamps
  - Batch write support
  - Write precision configuration (WritePrecision)
- **Query Operations**:
  - Historical data queries
  - Time-range filtering
  - Channel-based data retrieval
  - Energy calculations
- **Connection Management**:
  - InfluxConnector for connection pooling
  - Configurable query language (Flux or InfluxQL)
  - API key authentication
  - Organization and bucket configuration
- **Data Handling**:
  - Field type conflict handling
  - Channel filtering
  - Edge ID parsing and formatting
  - Read-only mode support

**Configuration**:
```properties
id: Component ID
url: InfluxDB server URL
org: InfluxDB organization
apiKey: Authentication API key
bucket: Data bucket name
queryLanguage: Query language (FLUX/INFLUX_QL)
isReadOnly: Read-only mode flag
```

**Main Components**:
- `TimedataInfluxDb` - Main timedata implementation
- `InfluxConnector` - Connection management
- `FieldTypeConflictHandler` - Handles data type conflicts
- `ChannelFilter` - Filters channels for storage
- `TimeFilter` - Time-based filtering

**Data Model**:
- Points with tags (edge ID, channel)
- Fields (channel values)
- Timestamps with configurable precision
- OEM tagging support

---

### 3.2 TimescaleDB Time-Series Storage (`io.openems.backend.timedata.timescaledb`)

**Module**: `io.openems.backend.timedata.timescaledb`

**Description**: PostgreSQL with TimescaleDB extension for time-series data storage.

**Key Features**:
- **TimescaleDB Integration**: PostgreSQL-based time-series database
- **JDBC Connectivity**: Direct JDBC connection to PostgreSQL
- **Schema Management**: 
  - Dynamic schema initialization
  - Schema-aware read/write handlers
- **Write Operations**:
  - Batch write support
  - Timestamped data notifications
  - Write handler with connection pooling
- **Read Operations**:
  - Historical data queries with time ranges
  - Energy queries (cumulative and per-period)
  - Channel address-based filtering
  - Resolution-based aggregation
- **Debug Monitoring**: Periodic debug logging of write handler status

**Configuration**:
```properties
host: PostgreSQL host
port: PostgreSQL port (default: 5432)
database: Database name
user: Database user
password: Database password
isReadOnly: Read-only mode flag
```

**Main Components**:
- `TimedataTimescaleDb` - Main timedata implementation
- `TimescaledbWriteHandler` - Write operations handler
- `TimescaledbReadHandler` - Read operations handler
- Schema initialization and management

**Query Capabilities**:
- `queryHistoricData()` - Time-range data queries
- `queryHistoricEnergy()` - Cumulative energy queries
- `queryHistoricEnergyPerPeriod()` - Energy per time period

**Advantages**:
- SQL-based queries
- ACID transactions
- Better for complex queries
- PostgreSQL ecosystem compatibility
- Cost-effective for large datasets

---

### 3.3 Aggregated InfluxDB (`io.openems.backend.timedata.aggregatedinflux`)

**Module**: `io.openems.backend.timedata.aggregatedinflux`

**Description**: Specialized InfluxDB storage for aggregated time-series data.

**Key Features**:
- **Aggregated Data Handling**: Processes pre-aggregated data notifications
- **Channel Whitelisting**: Configurable allowed channels for storage
- **Multiple Data Types**:
  - Timestamped data
  - Aggregated data with averages/sums
  - Resend data notifications
- **InfluxDB Write Parameters**:
  - Configurable write precision
  - Write consistency settings
  - Batch operations
- **Channel Type Management**:
  - ENUM channel types
  - Cumulated channels
  - Regular numeric channels

**Main Components**:
- `AggregatedInflux` - Main aggregated data handler
- `AllowedChannels` - Channel whitelist configuration
- `QueryWithCurrentData` - Combines historical and current data

**Use Case**: 
Optimized for storing and querying pre-aggregated data from edge devices, reducing storage overhead and improving query performance.

---

### 3.4 Dummy Time-Series Storage (`io.openems.backend.timedata.dummy`)

**Module**: `io.openems.backend.timedata.dummy`

**Description**: Mock time-series storage for testing and development without external dependencies.

**Key Features**:
- No-op write operations
- Empty result set for queries
- Development and testing support

---

## 4. Metadata Management

### 4.1 Odoo Metadata Provider (`io.openems.backend.metadata.odoo`)

**Module**: `io.openems.backend.metadata.odoo`

**Description**: Full-featured metadata management using Odoo ERP system with PostgreSQL backend.

**Key Features**:
- **Odoo ERP Integration**:
  - XML-RPC communication with Odoo
  - User and edge management
  - Role-based access control (OdooUserRole)
  - Partner and customer management
- **PostgreSQL Direct Access**:
  - Direct database queries for performance
  - PostgresHandler for low-level operations
  - Edge cache updates
- **User Management**:
  - User authentication (implements `AuthUserPasswordAuthenticationService`)
  - Password verification
  - User registration and profile management
  - Multi-language support
  - Global and edge-specific roles
- **Edge Management**:
  - Edge registration and configuration
  - Edge online/offline status tracking
  - Edge versioning and updates
  - Configuration diff tracking
  - Edge metadata caching
- **App Center Integration**:
  - Implements `AppCenterMetadata` interfaces
  - Edge data management
  - UI data management
- **Alerting Settings**:
  - User-specific alerting configurations
  - Offline edge alerting
  - Sum state alerting
- **Email Notifications**: Implements `Mailer` interface for user notifications
- **Event Handling**: Responds to edge events (online/offline, sum state)

**Configuration**:
```properties
odooProtocol: HTTP/HTTPS
odooHost: Odoo server host
odooPort: Odoo server port
odooUid: Odoo user ID
odooPassword: Odoo login password
pgHost: PostgreSQL host
pgPort: PostgreSQL port
pgDatabase: PostgreSQL database name
pgUser: PostgreSQL user
pgPassword: PostgreSQL password
```

**Main Components**:
- `MetadataOdoo` - Main metadata implementation
- `OdooHandler` - Odoo XML-RPC communication
- `PostgresHandler` - Direct PostgreSQL access
- `OdooUserRole` - User role definitions
- Edge and User models

**Capabilities**:
- Complete user lifecycle management
- Edge device management and monitoring
- Configuration management
- Access control and permissions
- Email notifications
- Real-time status tracking

---

### 4.2 File-Based Metadata Provider (`io.openems.backend.metadata.file`)

**Module**: `io.openems.backend.metadata.file`

**Description**: Simple file-based metadata provider using JSON configuration files.

**Key Features**:
- **JSON Configuration**: Reads edge and user data from JSON file
- **Static Configuration**: File-based edge definitions
- **No External Dependencies**: No database required
- **Simple Deployment**: Easy setup for development and small installations

**File Format**:
```json
{
  "edges": {
    "edge0": {
      "apikey": "...",
      "comment": "..."
    }
  }
}
```

**Configuration**:
```properties
path: Path to JSON configuration file
```

**Main Components**:
- `MetadataFile` - File-based metadata implementation
- JSON file parsing and caching
- Static user with global role
- Edge configuration from file

**Use Cases**:
- Development and testing
- Small installations without database
- Static edge configurations
- Proof of concept deployments

**Limitations**:
- No user management
- No dynamic updates (requires file edit and reload)
- No authentication
- Single user mode

---

### 4.3 Dummy Metadata Provider (`io.openems.backend.metadata.dummy`)

**Module**: `io.openems.backend.metadata.dummy`

**Description**: Mock metadata provider for testing without external dependencies.

**Key Features**:
- No-op implementations
- Testing support
- Development convenience

---

## 5. Alerting and Monitoring

### 5.1 Alerting System (`io.openems.backend.alerting`)

**Module**: `io.openems.backend.alerting`

**Description**: Comprehensive alerting system for monitoring edge devices and sending notifications.

**Key Features**:
- **Alert Types**:
  - **Offline Edge Alerts**: Monitors edge device connectivity
  - **Sum State Alerts**: Monitors aggregated system state
- **Email Notifications**: Integrates with Mailer service for user notifications
- **Scheduled Monitoring**: 
  - Scheduler for periodic checks
  - Configurable check intervals
  - Initial delay configuration
- **Thread Pool Management**:
  - Dedicated thread pool for alert processing
  - Configurable pool size (default: 4 threads)
  - Queue monitoring with warning thresholds
- **Event-Driven**:
  - Responds to edge online/offline events
  - Responds to sum state change events
  - Metadata initialization events
- **Alert Handlers**:
  - `OfflineEdgeHandler` - Detects and alerts on offline edges
  - `SumStateHandler` - Monitors system state changes
- **Metrics**:
  - Messages sent counter
  - Message queue size monitoring

**Configuration**:
```properties
notifyOnOffline: Enable offline edge notifications
initialDelay: Delay before first check (minutes)
```

**Main Components**:
- `Alerting` - Main alerting component
- `Scheduler` - Alert scheduling engine
- `OfflineEdgeHandler` - Offline detection handler
- `SumStateHandler` - State monitoring handler
- `Handler` - Base alert handler interface

**Event Topics**:
- `Edge.Events.ON_SET_ONLINE`
- `Edge.Events.ON_SET_SUM_STATE`
- `Metadata.Events.AFTER_IS_INITIALIZED`

**Workflow**:
1. Scheduler triggers periodic checks
2. Handlers evaluate conditions
3. Mailer sends notifications to users
4. Metrics track alert volume

**User Settings Integration**:
- User-specific alerting preferences from Metadata
- Configurable alert thresholds per user
- Edge-specific alert configurations

---

## 6. Edge Manager Capabilities

### 6.1 Edge Manager (`io.openems.backend.edge.manager`)

**Module**: `io.openems.backend.edge.manager`  
**Bundle Description**: Manager for connections from Backend Edge Applications

**Description**: Central component for managing WebSocket connections from edge devices to the backend.

**Key Features**:
- **WebSocket Server**: 
  - Dedicated WebSocket server for edge connections
  - Configurable port and thread pool
  - Connection lifecycle management (OnOpen, OnMessage, OnClose, OnError)
- **Edge Connection Management**:
  - Real-time online/offline status tracking
  - Connection state monitoring
  - Edge authentication and authorization
- **Request Routing**:
  - Routes JSON-RPC requests to appropriate handlers
  - Edge-to-backend request forwarding
  - Backend-to-edge command sending
- **System Log Management**:
  - `SystemLogHandler` for remote edge logging
  - Subscribe/unsubscribe to edge system logs
  - Log forwarding to UI WebSocket
- **OAuth Integration**:
  - OAuth registry for edge authentication
  - Token validation and refresh
- **Metadata Integration**:
  - Edge data from Metadata service
  - User authorization checks
  - App Center metadata support
- **Event Handling**:
  - Publishes edge online/offline events
  - Responds to metadata initialization
  - Event-driven architecture with EventAdmin

**Configuration**:
```properties
port: WebSocket server port
poolSize: Thread pool size for connections
```

**Main Components**:
- `EdgeManagerImpl` - Main edge manager implementation
- `WebsocketServer` - WebSocket server for edge connections
- `SystemLogHandler` - Remote logging handler
- `OnOpen`, `OnClose`, `OnError`, `OnRequest`, `OnNotification` - Event handlers
- `WsData` - WebSocket connection data wrapper

**Request Handling**:
- `EdgeRpcRequest` - Route requests to specific edges
- `AuthenticatedRpcRequest` - User-authenticated requests
- `SubscribeSystemLogRequest` - System log subscription
- Generic JSON-RPC request forwarding

**Integration Points**:
- `Metadata` - Edge and user information
- `TimedataManager` - Historical data queries
- `UiWebsocket` - UI notification forwarding
- `EventAdmin` - Event publishing
- `OAuthRegistry` - OAuth token management
- `AppCenterMetadata` - App Center integration

**Edge Capabilities**:
- Send commands to edge devices
- Query edge configuration
- Monitor edge status
- Collect edge logs
- Retrieve edge data

**Connection Flow**:
1. Edge connects via WebSocket
2. Authentication/authorization
3. Connection registered in EdgeManager
4. Status updates to Metadata
5. Events published to system
6. Ready for bidirectional communication

---

### 6.2 Edge Application Proxy (`io.openems.backend.edge.application`)

**Module**: `io.openems.backend.edge.application`  
**Bundle Description**: Proxy between OpenEMS Edges and OpenEMS Backend

**Description**: Proxy layer that sits between edge devices and the backend, facilitating communication.

**Key Features**:
- Request/response proxying
- Protocol translation
- Connection pooling
- Load distribution

---

## 7. Metrics and Prometheus Integration

### 7.1 Prometheus Metrics (`io.openems.backend.metrics.prometheus`)

**Module**: `io.openems.backend.metrics.prometheus`

**Description**: Prometheus-compatible metrics endpoint for monitoring OpenEMS backend health and performance.

**Key Features**:
- **Prometheus Metrics Endpoint**: 
  - HTTP `/metrics` endpoint
  - Configurable port (default: 9400)
  - OpenMetrics format compatible
- **Bearer Token Authentication**: Optional security for metrics endpoint
- **Metric Types**:
  - **Gauge Metrics**: Current state measurements
  - **Histogram Metrics**: Request distribution and latency
  - **Info Metrics**: Version and build information
- **Comprehensive Monitoring**:
  - WebSocket connection tracking
  - Thread pool metrics
  - Request performance metrics
  - Alerting metrics
  - System version info

**Exposed Metrics**:

#### WebSocket Metrics
- `websocket_connections{component}` - Active WebSocket connections
- `websocket_requests{component, method}` - WebSocket request histogram

#### Thread Pool Metrics
- `thread_pool_queue{component}` - Queue size
- `thread_pool_active_count{component}` - Active threads
- `thread_pool_completed_tasks{component}` - Completed task count
- `thread_pool_current_size{component}` - Current pool size
- `thread_pool_max_size{component}` - Maximum pool size

#### Alerting Metrics
- `alerting_messages_sent{component}` - Total alerts sent
- `alerting_messages_queue{component}` - Alerts in queue

#### System Metrics
- `openems_version{version}` - OpenEMS version information

**Configuration**:
```properties
port: HTTP server port for metrics (default: 9400)
bearerToken: Optional authentication token
```

**Prometheus Configuration Example**:
```yaml
scrape_configs:
  - job_name: 'openems-backend'
    bearer_token: <your_token_here>
    static_configs:
      - targets: ['localhost:9400']
```

**Integration**:
- Automatically collects metrics from all components
- Thread-safe metric updates
- Label-based metric organization
- Compatible with Grafana dashboards

**Main Components**:
- `PrometheusMetrics` - Metric definitions
- HTTP server for `/metrics` endpoint
- Automatic metric registration

**Use Cases**:
- Real-time performance monitoring
- Capacity planning
- Alerting based on metrics
- Debugging performance issues
- Historical trend analysis

---

## 8. Additional Backend Modules

### 8.1 OAuth Registry (`io.openems.backend.oauthregistry`)

**Module**: `io.openems.backend.oauthregistry`

**Description**: Central registry for managing OAuth providers and tokens.

**Key Features**:
- **OAuth Provider Management**: Register and manage OAuth providers
- **Token Operations**:
  - Fetch tokens from authorization codes
  - Refresh token management
  - Token exchange
- **Init Metadata**: Provides OAuth initialization metadata (auth URL, client ID, redirect URL)
- **Multi-Provider Support**: Handle multiple OAuth providers with unique identifiers

**Interfaces**:
```java
- OAuthRegistry
  - getInitMetadata(identifier)
  - fetchTokensFromRefreshToken(identifier, refreshToken, scopes)
  - fetchTokensFromCode(identifier, code, scopes, codeVerifier)
```

**Data Models**:
- `OAuthTokens` - Access and refresh token pair
- `OAuthInitMetadata` - Authentication URL, client ID, redirect URL

---

### 8.2 Backend Core (`io.openems.backend.core`)

**Module**: `io.openems.backend.core`  
**Bundle Description**: Supportive services that are used throughout OpenEMS Backend

**Description**: Core utilities and services shared across all backend modules.

**Key Features**:
- Common utility functions
- Shared service interfaces
- Base component classes
- Debug logging support

---

### 8.3 Backend Application (`io.openems.backend.application`)

**Module**: `io.openems.backend.application`

**Description**: Main backend application bootstrap and configuration.

**Key Features**:
- Application lifecycle management
- Component initialization
- Configuration management
- OSGi framework setup

---

### 8.4 Backend Common (`io.openems.backend.common`)

**Module**: `io.openems.backend.common`

**Description**: Common interfaces, models, and utilities shared across backend modules.

**Key Features**:
- **Component Base Classes**:
  - `AbstractOpenemsBackendComponent` - Base for all backend components
- **Interface Definitions**:
  - `EdgeManager` - Edge management interface
  - `Metadata` - Metadata service interface
  - `Timedata` - Time-series data interface
  - `UiWebsocket` - UI WebSocket interface
  - `Mailer` - Email notification interface
- **Models**:
  - `Edge` - Edge device model
  - `User` - User model with roles
  - `EdgeConfig` - Edge configuration
- **JSON-RPC**:
  - `JsonRpcRequestHandler` - Request handling interface
  - JSON-RPC request/response models
- **Utilities**:
  - Event handling utilities
  - Debug logging support (`DebugLoggable`)
  - Thread pool utilities

---

### 8.5 Simulator (`io.openems.backend.simulator`)

**Module**: `io.openems.backend.simulator`

**Description**: Simulation engine for testing and development.

**Key Features**:
- Edge device simulation
- Data generation
- Test scenarios
- Development support

---

## Architecture Overview

### Component Dependencies

```
┌─────────────────────────────────────────────────────────────┐
│                    Backend Application                       │
└─────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐  ┌─────────▼────────┐
│ Authentication │  │   Metadata  │  │   Edge Manager   │
│   (OAuth2)     │  │   (Odoo)    │  │   (WebSocket)    │
└───────┬────────┘  └──────┬──────┘  └─────────┬────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐  ┌─────────▼────────┐
│  UI WebSocket  │  │  B2B REST   │  │  B2B WebSocket   │
└───────┬────────┘  └──────┬──────┘  └─────────┬────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
┌───────▼────────┐  ┌──────▼──────┐  ┌─────────▼────────┐
│    InfluxDB    │  │ TimescaleDB │  │     Alerting     │
│   Timedata     │  │  Timedata   │  │                  │
└────────────────┘  └─────────────┘  └──────────────────┘
                            │
                    ┌───────┴───────┐
                    │   Prometheus  │
                    │    Metrics    │
                    └───────────────┘
```

### Authentication Flow

```
User/System
    │
    ▼
┌───────────────────┐     ┌──────────────────┐
│  Authentication   │────▶│    Keycloak      │
│      API          │◀────│   (OAuth2/JWT)   │
└─────────┬─────────┘     └──────────────────┘
          │
          ▼
┌───────────────────┐     ┌──────────────────┐
│    UI WebSocket   │     │    B2B REST      │
│   (User Auth)     │     │  (Basic Auth)    │
└───────────────────┘     └──────────────────┘
```

### Data Flow

```
Edge Device
    │
    ▼
┌───────────────────┐
│   Edge Manager    │ (WebSocket)
│   (Authenticate)  │
└─────────┬─────────┘
          │
          ▼
┌───────────────────┐     ┌──────────────────┐
│    Timedata       │────▶│   InfluxDB or    │
│    Manager        │     │   TimescaleDB    │
└─────────┬─────────┘     └──────────────────┘
          │
          ▼
┌───────────────────┐
│   UI WebSocket    │──────▶ User Interface
│   (Broadcast)     │
└───────────────────┘
```

---

## Configuration Summary

### Required Modules for Typical Deployment

1. **Authentication**: `io.openems.backend.authentication.oauth2`
2. **Metadata**: `io.openems.backend.metadata.odoo` or `.file`
3. **Time-Series Storage**: `io.openems.backend.timedata.influx` or `.timescaledb`
4. **Edge Communication**: `io.openems.backend.edge.manager`
5. **UI Communication**: `io.openems.backend.uiwebsocket`
6. **Monitoring**: `io.openems.backend.metrics.prometheus`
7. **Alerting** (optional): `io.openems.backend.alerting`
8. **B2B API** (optional): `io.openems.backend.b2brest` and/or `.b2bwebsocket`

### Deployment Patterns

#### Pattern 1: Full Production Setup
- OAuth2 Authentication
- Odoo Metadata (with PostgreSQL)
- InfluxDB for time-series
- Edge Manager + UI WebSocket
- Alerting enabled
- Prometheus metrics
- B2B REST API

#### Pattern 2: Lightweight Development
- File-based Metadata
- Dummy Timedata
- Edge Manager + UI WebSocket
- No authentication (development mode)

#### Pattern 3: Scalable Enterprise
- OAuth2 with multiple OEM configs
- Odoo Metadata with clustering
- TimescaleDB for time-series (better for complex queries)
- Multiple Edge Managers (load balanced)
- UI WebSocket cluster
- Prometheus + Grafana monitoring
- B2B WebSocket for real-time integration

---

## Security Considerations

### Authentication Security
- OAuth2 with PKCE prevents authorization code interception
- JWT signature verification ensures token integrity
- Rate-limited JWK provider prevents DoS attacks
- Secure token storage and refresh mechanisms

### API Security
- HTTP Basic Auth for B2B REST API
- Bearer token authentication for Prometheus metrics
- User authentication for UI WebSocket
- Edge authentication for Edge Manager

### Data Security
- API keys for InfluxDB connections
- PostgreSQL password protection
- Odoo authentication with UID and password
- Read-only mode support for timedata

---

## Performance Optimization

### Concurrency
- Virtual thread support in authentication service
- Thread pool management in alerting
- Concurrent request handling in all APIs
- Connection pooling for databases

### Caching
- Edge metadata caching in Edge Manager
- User session caching in UI WebSocket
- InfluxDB query result caching

### Monitoring
- Prometheus metrics for all components
- Thread pool metrics for bottleneck detection
- WebSocket connection tracking
- Alerting queue monitoring

---

## Troubleshooting

### Common Issues

1. **Authentication Failures**
   - Check Keycloak connectivity
   - Verify OAuth2 configuration
   - Check JWT certificate URL
   - Validate token expiration

2. **Edge Connection Issues**
   - Check Edge Manager WebSocket port
   - Verify edge authentication credentials
   - Check Metadata service status
   - Review edge online/offline events

3. **Time-Series Data Issues**
   - Verify InfluxDB/TimescaleDB connectivity
   - Check database credentials
   - Review write/read handler logs
   - Validate channel configurations

4. **Alerting Not Working**
   - Check Mailer configuration
   - Verify user alerting settings in Metadata
   - Review alerting scheduler status
   - Check thread pool queue size

5. **Prometheus Metrics Not Available**
   - Verify Prometheus metrics port
   - Check bearer token configuration
   - Ensure components are registering metrics
   - Review HTTP server logs

---

## API Reference Summary

### Authentication API
- `AuthUserRegistrationService.registerUser()`
- `AuthUserAuthorizationCodeFlowService.initiateConnect()`
- `AuthUserPasswordAuthenticationService.authenticateWithPassword()`

### Metadata API
- `Metadata.getEdge()`
- `Metadata.getUser()`
- `Metadata.authenticate()`
- `Metadata.getEdgeMetadata()`

### Timedata API
- `Timedata.write()`
- `Timedata.queryHistoricData()`
- `Timedata.queryHistoricEnergy()`

### Edge Manager API
- `EdgeManager.send()`
- `EdgeManager.isOnline()`
- `EdgeManager.getEdge()`

### UI WebSocket API
- WebSocket connection with JSON-RPC
- Request/Response pattern
- Notification broadcasting

### B2B REST API
- `POST /jsonrpc` with JSON-RPC payload
- HTTP Basic Authentication

---

## Version Information

This documentation is based on the OpenEMS backend modules found in the workspace. For the most up-to-date information, please refer to:
- Source code in each module's `src/` directory
- README files in module directories
- Official OpenEMS documentation at https://github.com/OpenEMS/openems

---

## Conclusion

The OpenEMS backend provides a comprehensive, modular, and scalable architecture for managing energy management systems. The backend supports:

- **Multiple authentication methods** (OAuth2, JWT, password-based)
- **Flexible metadata management** (Odoo ERP or file-based)
- **Scalable time-series storage** (InfluxDB or TimescaleDB)
- **Real-time communication** (WebSocket for UI and edges)
- **REST and WebSocket APIs** for B2B integration
- **Comprehensive monitoring** (Prometheus metrics)
- **Intelligent alerting** (offline detection, state monitoring)

Each module is designed to be independently deployable and can be configured based on specific deployment requirements, from lightweight development setups to full-scale production environments.
