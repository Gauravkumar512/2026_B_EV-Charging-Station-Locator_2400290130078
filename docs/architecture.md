# 🏗️ System Architecture — EV Charging Station Locator

> Complete High-Level and Component-Level Architecture Blueprint for the EV Charging Station Locator Platform.

---

## 📋 Architecture Index & Document Map
This system architecture document is part of the comprehensive Software Engineering Lab suite for **CS311L – Software Engineering and Design Principles**:

| Lab Session | Deliverable Document | Focus Area |
| :--- | :--- | :--- |
| **Session 1** | [Experiment 1: Tools & Version Control](exp1.md) | Collaborative Development, Git/GitHub, Agile SDLC |
| **Session 2** | [Experiment 2: Problem Identification & Feasibility](exp2.md) | Problem Statement, Scope & Feasibility Analysis |
| **Session 3** | [Experiment 3: Requirements Elicitation](exp3.md) | Interviews, Questionnaires, Personas, User Stories |
| **Session 4** | [Experiment 4: IEEE SRS Documentation](exp4.md) | Formal Software Requirement Specification |
| **Session 5** | [Experiment 5: UML Use Case Modeling](exp5.md) | Use Case Diagrams & Operational Narratives |
| **Session 6** | [Experiment 6: UML Class Diagram Design](exp6.md) | Object-Oriented Analysis, Classes & Invariants |
| **Session 7** | [Experiment 7: Dynamic Behavioral Modeling](exp7.md) | Sequence & Activity Workflow Diagrams |
| **Session 8** | [Experiment 8: Software Design Principles](exp8.md) | Modularization, Coupling & Cohesion Analysis |
| **Session 9** | [Experiment 9: Architecture & Design Patterns](exp9.md) | Layered, Microservices & GoF Design Patterns |

---

## 📐 Architecture Overview

The application follows a **multi-tier cloud-native architecture**:

1. **Presentation Layer (Client SPA / PWA)** — Responsive React/TypeScript interface featuring Mapbox GL interactive vector rendering, search filters, and booking sheets.
2. **Gateway & Security Layer** — Reverse proxy, SSL termination, JWT authentication, and API rate limiting.
3. **Application Layer (Business Logic Services)** — Node.js / Express and Python services managing geospatial queries, distributed 15-minute slot locks, and payment settlements.
4. **Data Persistence & Cache Layer** — PostgreSQL 15 with PostGIS spatial extension for sub-second location bounding box queries, Redis 7 for distributed locks and pub-sub caching, and MQTT for hardware telemetry.

---

## 🔗 High-Level System Architecture Diagram

```mermaid
graph TB
    subgraph Client["🖥️ Presentation Layer (Client Tier)"]
        UI["Web App / Mobile PWA (React)"]
        MAP["Mapbox GL Vector Canvas"]
        AUTH_UI["Auth & Vehicle Profile Views"]
        BOOK_UI["Slot Booking & Dynamic QR Pass"]
    end

    subgraph Gateway["🚪 Gateway & Edge Tier"]
        PROXY["Nginx Reverse Proxy & Load Balancer"]
        WAF["Rate Limiting & CORS Filter"]
        JWT["JWT Security Filter"]
    end

    subgraph Server["⚙️ Application Layer (Service Tier)"]
        AUTH["Auth & Identity Service"]
        GEO["Geospatial & Station Service"]
        BOOK["Booking & Lock Service"]
        PAY["Payment Adapter Service"]
        TELEM["OCPP Telemetry Ingestion"]
    end

    subgraph Data["🗄️ Persistence Layer (Data Tier)"]
        DB[("PostgreSQL 15 + PostGIS<br/>(Spatial R-Tree Index)")]
        REDIS[("Redis 7 In-Memory Cache<br/>& Distributed Locks")]
        MQ[("MQTT / AMQP Message Broker")]
    end

    subgraph External["🌐 External Cloud Services"]
        MAPS_API["Mapbox / Google Maps Geocoding API"]
        PAY_API["Razorpay / Stripe Payment Gateway"]
        SMS_API["Twilio / SendGrid Notifications"]
        IOT_KIOSK["Physical EV Charging Kiosks (OCPP)"]
    end

    Client -->|HTTPS REST & WSS| PROXY
    PROXY --> WAF --> JWT
    JWT --> AUTH
    JWT --> GEO
    JWT --> BOOK
    JWT --> PAY
    
    GEO --> DB
    GEO --> REDIS
    GEO --> MAPS_API
    
    BOOK --> DB
    BOOK --> REDIS
    BOOK --> SMS_API
    
    PAY --> PAY_API
    PAY --> DB
    
    IOT_KIOSK --> MQ
    MQ --> TELEM
    TELEM --> DB
    TELEM --> REDIS
    REDIS -.->|WebSocket Broadcast| Client
```

---

## 📦 Component Breakdown & Service Boundaries

### 1. Presentation Layer (Frontend)

| Component | Technology | Responsibility |
| :--- | :--- | :--- |
| **Interactive Map View** | Mapbox GL JS / Leaflet | Renders GPU-accelerated vector map, station clusters, and custom status pins |
| **Filter & Search Drawer** | React, Tailwind CSS | Multi-criteria filtering by connector type (CCS2, Type 2), power (kW), and pricing |
| **Booking & Checkout Modal** | React, Stripe/Razorpay SDK | Manages 15-minute countdown reservation hold and payment authorization |
| **QR Pass Generator** | QR Canvas / Web Crypto | Generates offline-verifiable, dynamic cryptographically signed QR code pass |
| **Operator Console** | React Dashboard | Port health toggle, tariff configuration, and real-time revenue analytics |

### 2. Application Layer (Backend Microservices / Modules)

| Service Module | Endpoints | Responsibility |
| :--- | :--- | :--- |
| **Auth & Profile Service** | `/api/v1/auth/*` | User registration, bcrypt/Argon2 password hashing, JWT token rotation |
| **Station & Geo Service** | `/api/v1/stations/*` | PostGIS spatial queries (`ST_DWithin`), connector filtering, rating aggregates |
| **Booking & Lock Service** | `/api/v1/bookings/*` | Redis-backed distributed slot reservation, concurrency conflict resolution |
| **Payment Service** | `/api/v1/payments/*` | Checkout intent creation, webhook signature verification, digital invoices |
| **Telemetry Ingestion Worker** | MQTT subscriber | Ingests live charger telemetry (voltage, current, socket status) via OCPP |

### 3. Data Persistence Layer

| Datastore | Engine | Primary Usage & Storage Schema |
| :--- | :--- | :--- |
| **Relational & Spatial DB** | PostgreSQL 15 + PostGIS | Station geospatial coordinates, user accounts, vehicle profiles, immutable payment ledgers |
| **In-Memory Cache & Lock** | Redis 7 | 15-minute slot reservation locks (`SET NX PX`), hot station query cache, WebSocket pub-sub |
| **IoT Message Broker** | Eclipse Mosquitto / RabbitMQ | Ingestion queue for high-frequency OCPP hardware status telemetry |

---

## 🔄 End-to-End Dynamic Interaction Flows

### A. Geospatial Discovery Sequence
```mermaid
sequenceDiagram
    actor Driver as 👤 EV Driver
    participant UI as 🖥️ Client App
    participant API as ⚙️ Station Service
    participant Cache as ⚡ Redis Cache
    participant DB as 🗄️ PostgreSQL (PostGIS)

    Driver->>UI: Enables location & requests nearby chargers
    UI->>API: GET /stations/nearby?lat=xx&lng=yy&radius=5km&connector=CCS2
    API->>Cache: Check cached spatial query hash
    alt Cache Hit
        Cache-->>API: Return cached station summaries
    else Cache Miss
        API->>DB: Query ST_DWithin(geom, ST_MakePoint(lng,lat), 5000)
        DB-->>API: Return stations with live port counts
        API->>Cache: Cache results with 60-second TTL
    end
    API-->>UI: Deliver station pins with color-coded availability
    UI-->>Driver: Render interactive map with clickable markers
```

### B. Slot Reservation & Atomic Settlement Sequence
```mermaid
sequenceDiagram
    actor Driver as 👤 EV Driver
    participant UI as 🖥️ Client App
    participant BookSvc as ⚙️ Booking Service
    participant Redis as 🔒 Redis Redlock
    participant PayGW as 💳 Payment Gateway
    participant DB as 🗄️ Database

    Driver->>UI: Selects Station Port -> Clicks "Reserve (15 Mins)"
    UI->>BookSvc: POST /bookings/reserve {slotId, vehicleId}
    BookSvc->>Redis: Acquire distributed slot lock (TTL: 900s)
    alt Lock Failed (Port already locked)
        Redis-->>BookSvc: Lock collision
        BookSvc-->>UI: 409 Conflict ("Slot reserved by another user")
    else Lock Succeeded
        Redis-->>BookSvc: Lock OK
        BookSvc->>DB: Record reservation (status='PENDING')
        BookSvc->>PayGW: Create payment order (₹50 deposit)
        PayGW-->>BookSvc: Order details & payment token
        BookSvc-->>UI: 201 Created (Order token + 15m timer)
        UI->>PayGW: Driver completes checkout
        PayGW->>BookSvc: Webhook: payment.successful
        BookSvc->>DB: Update reservation (status='CONFIRMED')
        BookSvc-->>UI: Deliver encrypted QR entry pass
    end
```

---

## 🧩 Architectural Design Patterns Applied

| Pattern | Category | Context & Project Implementation |
| :--- | :--- | :--- |
| **Singleton Pattern** | Creational | Shared connection pooling for PostgreSQL and Redis clients to eliminate socket overhead |
| **Factory Method Pattern** | Creational | `PaymentGatewayFactory` instantiating regional payment adapters (`StripeAdapter`, `RazorpayAdapter`) |
| **Adapter Pattern** | Structural | Standardized `IMapProvider` abstracting Mapbox, Google Maps, and OpenStreetMap implementations |
| **Facade Pattern** | Structural | `BookingFacade` orchestrating slot validation, payment hold, and notification delivery |
| **Observer Pattern** | Behavioral | Real-time WebSocket pub-sub pushing station availability mutations to client maps |
| **Strategy Pattern** | Behavioral | Dynamic tariff algorithm calculating peak/off-peak rates and commercial fleet subsidies |

---

## 🔐 Security & Non-Functional Architecture Safeguards
* **Zero Trust Transport:** Mandatory TLS 1.3 encryption across all public interfaces with strict HSTS policies.
* **Stateless Authorization:** Cryptographically signed JWT access tokens with short TTLs (15 mins) and HTTP-only refresh cookies.
* **PCI-DSS Level 1 Compliance:** Direct browser-to-gateway tokenization; zero raw credit card or banking secrets stored on platform servers.
* **High-Availability Fallbacks:** Redis cache fallback to read-replica database in the event of primary database failover.

---

## 👥 Student & Course Identification
* **Course Code:** CS311L – Software Engineering and Design Principles
* **Project Name:** EV Charging Station Locator
* **Repository Identifier:** `EV-Charging-Station-Locator`
