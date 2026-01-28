# SimTestLab EMS - System Architecture

## Table of Contents

1. [Overview](#overview)
2. [Technology Stack](#technology-stack)
3. [Architecture Diagram](#architecture-diagram)
4. [Service Responsibilities](#service-responsibilities)
   - [Django Backend](#1-django-backend-user-management)
   - [FastAPI WebSocket Service](#2-fastapi-websocket-service-real-time)
   - [Next.js Frontend](#3-nextjs-frontend-visualization)
   - [Edge Controllers](#4-edge-controllers-future-development)
5. [User Roles & Access Control](#user-roles--access-control)
   - [User Types](#user-types)
   - [Permission Matrix](#permission-matrix)
   - [Multi-Tenant Architecture](#multi-tenant-architecture)
6. [Data Flow](#data-flow)
   - [Real-time Telemetry Flow](#real-time-telemetry-flow-critical-path)
   - [Permission Update Flow](#permission-update-flow-event-driven)
   - [Historical Data Query Flow](#historical-data-query-flow)
   - [Edge Controller Connection Flow](#edge-controller-connection-flow)
7. [Deployment Strategy](#deployment-strategy)
   - [Production Deployment](#production-deployment-kubernetes)
8. [Pros & Cons Analysis](#pros--cons-analysis)
   - [Pros of This Architecture](#pros-of-this-architecture)
9. [Technology Decisions](#technology-decisions)
   - [Why Django?](#why-django)
   - [Why FastAPI?](#why-fastapi-instead-of-django-channels)
   - [Why Next.js?](#why-nextjs)
   - [Why Redis?](#why-redis)
   - [Why PostgreSQL?](#why-postgresql)
---

## Overview

SimTestLab EMS uses a **microservices architecture** with clear separation of concerns:

- **Django**: User management, authentication, RBAC, and admin operations
- **Next.js**: Data visualization, dashboards, and user-facing application
- **FastAPI**: Real-time WebSocket connections and telemetry streaming
- **Edge Controllers**: Device control logic and telemetry collection

**Key Design Principles:** Separation of concerns, admin-first design, real-time performance, independent scalability, and Docker-based development

---

## Technology Stack

### Backend Services

| Service | Technology | Port | Purpose |
|---------|-----------|------|---------|
| **User Management** | Django 6.0 + DRF | 8000 | Authentication, users, organizations, permissions |
| **WebSocket Service** | FastAPI + uvicorn | 8001 | Real-time data streaming, WebSocket connections |
| **Edge Controller** | TBD (Python) | TBD | Device control, telemetry collection |

### Frontend

| Component | Technology | Port | Purpose |
|-----------|-----------|------|---------|
| **Visualization App** | Next.js 16 + React 19 | 3000 | Dashboards, charts, user interface |

### Infrastructure

| Service | Technology | Port | Purpose |
|---------|-----------|------|---------|
| **Database** | PostgreSQL 16 | 5432 | User data, organizations, sites, metadata |
| **Cache/PubSub** | Redis 7 | 6379 | Session storage, caching, real-time message broker |
| **Time-Series DB** | TimescaleDB (optional) | 5433 | Historical telemetry data (future) |

### Development Tools

- **Docker Compose**: Local development orchestration
- **TypeScript**: Type safety in frontend
- **OpenAPI**: API documentation and client generation
- **SWR**: Frontend data fetching and caching

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         Docker Compose Network                       │
│                                                                      │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐   │
│  │    Next.js      │    │     Django      │    │    FastAPI      │   │
│  │   Frontend      │    │    Backend      │    │   WebSocket     │   │
│  │   Port: 3000    │    │   Port: 8000    │    │   Port: 8001    │   │
│  │                 │    │                 │    │                 │   │
│  │ • Dashboards    │    │ • User Auth     │    │ • WebSockets    │   │
│  │ • Data Viz      │◄──►│ • RBAC          │◄───│ • Streaming     │   │
│  │ • Charts        │    │ • Admin Panel   │    │ • Real-time     │   │
│  │ • User Profile  │    │ • JWT Tokens    │    │ • PubSub        │   │
│  │                 │    │ • Organizations │    │                 │   │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘   │
│           │                      │                      │            │
│           │   REST API           │  JWT Validation      │            │
│           └──────────────────────┘  (on WS connect)     │            │
│                      │                                  │            │
│                      │                                  │            │
│           ┌──────────▼──────────┐          ┌────────────▼────────┐   │
│           │    PostgreSQL       │          │       Redis         │   │
│           │    Port: 5432       │          │     Port: 6379      │   │
│           │                     │          │                     │   │
│           │ • Users             │          │ • Permission Cache  │   │
│           │ • Organizations     │          │ • PubSub Channels   │   │
│           │ • Sites             │          │ • WebSocket State   │   │
│           │ • Roles/Permissions │          │ • Telemetry Stream  │   │
│           └─────────────────────┘          └─────────────────────┘   │
│                                                       ▲              │
└───────────────────────────────────────────────────────┼──────────────┘
                                                        │
                                         Publishes telemetry to Redis
                                                        │
                                           ┌────────────┴────────────┐
                                           │   Edge Controllers      │
                                           │   (Future Development)  │
                                           │                         │
                                           │ • Device Control        │
                                           │ • Telemetry Collection  │
                                           │ • Local Processing      │
                                           └─────────────────────────┘
```

---

## Service Responsibilities

### 1. Django Backend (User Management)

**Primary Responsibility:** User authentication, authorization, and administrative operations

**Core Features:**
- User authentication (JWT-based)
- Role-based access control (RBAC)
- Multi-tenant organization management
- Site hierarchy and access control
- Django Admin panel for operations
- Audit logging and security policies

**Key Endpoints:** Authentication, user management, organizations, sites, permissions

**Data Models:** User, Organization, Site, Role, SiteAccess, AuditLog

**Communication with FastAPI:**

Django provides authentication services to FastAPI but does NOT handle telemetry streaming:

1. **JWT Validation Endpoint:**
   - FastAPI calls Django's `/api/validate-token` when WebSocket connects
   - Django validates JWT signature, checks expiry, verifies user status
   - Returns user permissions and accessible sites list
   - This is the **only** synchronous communication between the services

2. **Permission Change Events:**
   - When admin modifies user roles/permissions in Django Admin
   - Django publishes event to Redis: `PUBLISH user:456:permission_changed`
   - FastAPI listens to these events and invalidates its permission cache
   - Asynchronous, event-driven (no direct HTTP call)

**What Django Does NOT Do:**
- ❌ Handle WebSocket connections (that's FastAPI's job)
- ❌ Stream telemetry data (that flows through Redis → FastAPI)
- ❌ Validate permissions for every message (FastAPI uses cached permissions)
- ❌ Store telemetry data (that goes to TimescaleDB in future, or stays in Redis)

**Django's Focused Role:** User/org/site management, authentication, and admin operations only

---

### 2. FastAPI WebSocket Service (Real-time)

**Primary Responsibility:** Real-time data streaming and WebSocket connections

**Core Features:**
- WebSocket connections for live telemetry
- Redis PubSub subscription and broadcasting
- JWT validation and permission caching
- Low-latency data delivery

**Communication with Django:**

FastAPI communicates with Django **only during initial WebSocket connection**, not during data streaming:

1. **Initial Connection (ONE-TIME Django Call):**
   - Client connects to FastAPI WebSocket with JWT token
   - FastAPI makes HTTP GET request to Django: `/api/validate-token`
   - Django validates JWT signature, expiry, and returns user permissions
   - FastAPI caches `{user_id: accessible_sites, role, permissions}` in Redis (5-min TTL)
   - Connection accepted/rejected based on site access

2. **Ongoing Streaming (NO Django):**
   - Edge Controller → Redis: `PUBLISH telemetry:site_123 {data}`
   - FastAPI receives via Redis PubSub subscription
   - FastAPI checks cached permissions from Redis (not Django)
   - FastAPI broadcasts to authorized WebSocket clients
   - **Django is NOT in this critical path**

3. **Permission Updates (EVENT-DRIVEN):**
   - Admin changes user role in Django Admin
   - Django → Redis: `PUBLISH user:456:permission_changed`
   - FastAPI receives event, invalidates cache, notifies client
   - Client reconnects (goes back to step 1)

**Why This Separation?**
- **Performance**: No HTTP overhead during streaming (10,000+ msg/sec possible)
- **Scalability**: FastAPI handles WebSockets independently without Django bottleneck
- **Resilience**: Django can restart without dropping WebSocket connections
- **Industry Standard**: Pattern used by OpenEMS, MyEMS, AWS IoT Core

**Key Principle:** Event-driven communication via Redis, not request-driven. FastAPI validates once, then streams independently based on cached permissions

---

### 3. Next.js Frontend (Visualization)

**Primary Responsibility:** User interface, data visualization, and user experience

**Core Features:**
- Real-time dashboards with Recharts
- Multi-site hierarchy navigation
- Alert management and historical data analysis
- Clean Architecture (Domain/Application/Infrastructure/Presentation layers)

**Integration:**
- REST API calls to Django (authentication, user data, historical queries)
- WebSocket connections to FastAPI (real-time telemetry)
- JWT token management with automatic refresh

---

### 4. Edge Controllers (Future Development)

**Primary Responsibility:** Device control and telemetry collection at the edge

**Core Features:** Device communication (Modbus, BACnet, MQTT), local control logic, telemetry aggregation, offline buffering, Redis publishing

**Architecture Decisions:** Separate from Django for performance, independent deployment (on-site or cloud), autonomous operation during cloud outages, periodic metadata sync with Django

---

## User Roles & Access Control

### User Types

#### 1. Application Users
**Purpose:** End users who consume the EMS application

**Roles:**

**a) Viewer**
- View dashboards and reports
- See sites they have access to
- Cannot modify settings
- Read-only access

**b) Operator**
- All Viewer permissions
- Acknowledge alerts
- Export reports
- Control devices (future)
- Cannot manage users

**c) Site Manager**
- All Operator permissions
- Manage site-specific settings
- Add/remove operators for their sites
- Configure site parameters
- Cannot access other sites

**d) Organization Admin**
- All Site Manager permissions
- Access all sites in their organization
- Manage organization users
- View organization-wide analytics
- Cannot access other organizations

---

#### 2. Admin/Developer Users
**Purpose:** Maintain and manage the application itself

**Roles:**

**a) Platform Admin (Django Superuser)**
- Full access to Django Admin
- Create/delete organizations
- Assign any role to any user
- View all audit logs
- System-wide configuration
- Database access (via admin panel)

**b) Developer**
- Access to development tools
- API documentation
- Test data generation
- Staging environment access
- Cannot access production user data (unless also Platform Admin)

---

### Permission Matrix

| Action | Viewer | Operator | Site Mgr | Org Admin | Platform Admin |
|--------|--------|----------|----------|-----------|----------------|
| View dashboards | ✅ | ✅ | ✅ | ✅ | ✅ |
| Export reports | ❌ | ✅ | ✅ | ✅ | ✅ |
| Acknowledge alerts | ❌ | ✅ | ✅ | ✅ | ✅ |
| Control devices | ❌ | ❌* | ✅ | ✅ | ✅ |
| Manage site settings | ❌ | ❌ | ✅ | ✅ | ✅ |
| Add site users | ❌ | ❌ | ✅ | ✅ | ✅ |
| Manage org users | ❌ | ❌ | ❌ | ✅ | ✅ |
| Access other orgs | ❌ | ❌ | ❌ | ❌ | ✅ |
| Django Admin access | ❌ | ❌ | ❌ | ❌ | ✅ |

_*Operators may have limited control permissions based on site configuration_

---

### Multi-Tenant Architecture

**Organization Hierarchy:**
```
Platform (SimTestLab EMS)
└── Organization (e.g., "Acme Corp")
    ├── Site 1 (e.g., "Manufacturing Plant A")
    │   ├── Building 1
    │   │   ├── HVAC System
    │   │   └── Lighting
    │   └── Building 2
    └── Site 2 (e.g., "Warehouse B")
        └── Solar Array
```

**Access Control:**
- Users belong to one or more organizations
- Site access is explicitly granted (not inherited)
- Organization admins can only manage their org's users
- Platform admins can see everything (via Django Admin)

**Data Isolation:**
- Database-level: `WHERE organization_id = user.organization_id`
- API-level: Django middleware enforces organization boundaries
- WebSocket-level: FastAPI validates site access before streaming

---

### Real-time Telemetry Flow (Critical Path)

**Edge → Redis → FastAPI → Frontend**

1. **Edge Controller:** Collects sensor data (Modbus, BACnet, MQTT) → Publishes to Redis channel `telemetry:site_123`
2. **FastAPI Background Worker:** Subscribes to Redis channels (`telemetry:*`, `alerts:*`, `user:*:events`) → Receives message → Checks cached permissions → Broadcasts to authorized WebSocket clients
3. **Next.js Frontend:** Opens WebSocket with JWT → FastAPI validates (one-time Django call) → Caches permissions in Redis (5-min TTL) → Streams data to client → Updates charts in real-time

**Key Advantage:** Django NOT in critical path after initial auth. Edge data flows directly through Redis → FastAPI → Client without Django overhead

---

### Permission Update Flow (Event-Driven)

1. **Django Admin:** Admin updates user role → Django saves to PostgreSQL → Invalidates Redis cache → Publishes event to Redis `user:456:permission_changed`
2. **FastAPI:** Receives Redis event → Finds active WebSocket connections for user → Sends control message to client
3. **Next.js:** Receives control message → Refetches permissions or reconnects WebSocket

**Advantage:** Instant updates (sub-second), no polling overhead, graceful user experience

---

### Historical Data Query Flow

1. **Next.js:** User requests historical data → GET `/api/telemetry/historical` with JWT
2. **Django:** Validates JWT → Checks site access permissions → Queries TimescaleDB/PostgreSQL → Returns aggregated data
3. **Next.js:** Caches response (SWR/React Query) → Renders charts

**Why REST API (not WebSocket):** One-time bulk fetch, HTTP caching, simpler for static historical data

---

### Edge Controller Connection Flow

1. **Startup:** Edge loads config → Authenticates with Django using API key → Caches site_id
2. **Data Collection:** Reads devices (Modbus, BACnet, MQTT) every 5s → Publishes to Redis `telemetry:site_123` → Optionally writes to TimescaleDB for historical storage
3. **Error Handling:** Buffers locally on Redis failure, continues on Django failure (autonomous operation)

**Deployment Models:**
- **Cloud-Hosted:** Gateway → VPN → Cloud Edge Controller → Redis (easier setup)
- **On-Premises:** Edge on Raspberry Pi/Industrial PC on-site → Local Redis → Cloud Redis replication (more resilient, offline operation)

---

### Production Deployment (Kubernetes)

**Architecture:** Load Balancer → Next.js (3 replicas) + Django (2 replicas) → FastAPI WebSocket (2 replicas) → PostgreSQL (primary + replicas) + Redis Cluster

**Scaling:** Django scales on API load, FastAPI scales on WebSocket connections, Next.js scales on user traffic

---

### Pros of This Architecture

#### 1. **Leverages Django's Strengths**
- **Django Admin**: Out-of-the-box powerful admin panel for user/org management
- **Permissions Framework**: Built-in RBAC with groups and permissions
- **Battle-tested Auth**: Industry-standard authentication and security
- **Audit Trail**: Easy to add logging for compliance (who changed what, when)

#### 2. **Real-time Performance**
- **FastAPI WebSockets**: Native async support, lower latency than Django Channels
- **Redis PubSub**: Instant message broadcasting to all connected clients
- **No Polling**: Eliminates constant HTTP requests, reduces server load
- **Sub-second Updates**: Telemetry appears on dashboards immediately

#### 3. **Clear Separation of Concerns**
- **Django**: User management only (not burdened with telemetry streaming)
- **FastAPI**: Real-time only (not burdened with user CRUD)
- **Next.js**: Visualization only (not burdened with business logic)
- **Clean Boundaries**: Teams can work independently on each service

#### 4. **Scalability**
- **Independent Scaling**: Scale WebSocket service separately from API service
- **Service Isolation**: Django crash doesn't affect WebSocket connections
- **Database Optimization**: PostgreSQL for users, TimescaleDB for telemetry (future)
- **Horizontal Scaling**: Add more containers/pods as needed

#### 5. **Developer Experience**
- **Docker Compose**: `docker-compose up` starts entire stack
- **Same Environment**: Eliminates "works on my machine" issues
- **Hot Reload**: All services support live code updates during development
- **OpenAPI Docs**: Auto-generated API documentation (FastAPI + DRF-spectacular)

#### 6. **Modern Tech Stack**
- **TypeScript**: Type safety reduces bugs in complex EMS logic
- **React Ecosystem**: Rich component libraries (Recharts, Radix UI)
- **Clean Architecture**: Domain-driven design in frontend
- **Industry Standard**: Docker/Kubernetes deployment path

#### 7. **Enterprise Features**
- **Multi-tenancy**: Organization/site hierarchy with data isolation
- **RBAC**: Granular permissions for different user roles
- **Audit Logging**: Track all changes for compliance
- **Security**: JWT tokens, CORS, SQL injection protection

---

## Technology Decisions

### Why Django?
- **Admin Panel**: Best-in-class admin interface for free
- **ORM**: Powerful database abstraction
- **Security**: Built-in protection against common vulnerabilities
- **Ecosystem**: Mature, well-documented, huge community
- **Stability**: Production-ready, used by Instagram, Pinterest, NASA

### Why FastAPI (instead of Django Channels)?
- **Performance**: 3-5x faster than Django for async workloads
- **Simplicity**: Native WebSocket support, no extra layers
- **Documentation**: Auto-generated OpenAPI/Swagger docs
- **Modern**: Built on Starlette and Pydantic (type-safe)
- **Learning Curve**: Easier than Django Channels for WebSocket-only use case

### Why Next.js?
- **React Ecosystem**: Best component libraries and tools
- **Performance**: Server-side rendering, static generation
- **Developer Experience**: Hot reload, TypeScript support
- **API Routes**: Can handle some backend logic if needed
- **Production Ready**: Used by Netflix, TikTok, Twitch

### Why Redis?
- **Speed**: In-memory data structure store (microsecond latency)
- **PubSub**: Built-in message broker for real-time updates
- **Caching**: Reduce database load for frequently accessed data
- **Session Store**: Fast session storage for Django
- **Simple**: Easy to set up and maintain

### Why PostgreSQL?
- **ACID Compliance**: Data integrity for critical user/org data
- **JSON Support**: Can store semi-structured data if needed
- **Extensions**: PostGIS for location data, TimescaleDB for time-series
- **Reliability**: Industry standard, battle-tested
- **Django Support**: First-class ORM support

---