# Experiment 7: UML Sequence and Activity Diagram Modeling

## Dynamic Behavioral & Workflow Interaction Modeling

**Design UML Sequence Diagrams illustrating chronological message exchange patterns, and construct UML Activity Diagrams modeling complex business workflows and concurrency for the EV Charging Station Locator system.**

---

### Aim
To model the dynamic, time-ordered behavioral interactions between system components using UML Sequence Diagrams, and to represent end-to-end operational workflows, decision branching, and concurrency using UML Activity Diagrams for the EV Charging Station Locator.

---

### Objectives
* Identify lifelines, synchronous/asynchronous messages, activation blocks, and return flows in key user scenarios.
* Construct UML Sequence Diagrams for primary business transactions (Geospatial Search, Slot Reservation & Payment, Hardware Telemetry Broadcasting).
* Model business workflows with start/end states, decision diamonds, merge nodes, forks, and joins using UML Activity Diagrams.
* Represent concurrent and distributed asynchronous operations (distributed locking, payment webhooks, real-time push).
* Validate behavioral specifications against the IEEE SRS requirements.

---

### Introduction
While structural diagrams (such as Class Diagrams) illustrate the static architecture of software, dynamic UML diagrams capture how components collaborate over time to fulfill use cases.
* **Sequence Diagrams** emphasize the chronological sequence of messages passed between object lifelines. They are invaluable for designing REST/WebSocket interactions, identifying distributed race conditions, and planning API contracts.
* **Activity Diagrams** depict step-by-step control flows, decision logic, and parallel execution paths within a business process, serving as the bridge between user journey maps and backend transaction boundaries.

---

### Sequence Diagram 1: Geospatial Station Search & Map Rendering

This sequence captures the interaction when an EV driver opens the locator map, queries nearby stations, and filters by connector standard.

```mermaid
sequenceDiagram
    autonumber
    actor Driver as 👤 EV Driver
    participant UI as 🖥️ Web / Mobile Client
    participant GW as 🚪 API Gateway
    participant GeoSvc as 📍 Station / Geo Service
    participant Cache as ⚡ Redis Cache
    participant DB as 🗄️ PostgreSQL (PostGIS)
    participant MapAPI as 🌐 Mapbox Maps API

    Driver->>UI: Opens app & allows Location
    UI->>UI: Acquire coordinates (lat, lng, accuracy)
    UI->>GW: GET /api/v1/stations/nearby?lat=28.61&lng=77.20&radius=5km&connector=CCS2
    GW->>GW: Validate request parameters & rate limit
    GW->>GeoSvc: forward query(lat, lng, radius, filter)
    
    GeoSvc->>Cache: GET geo_query_hash
    alt Cache Hit (Within 60 seconds)
        Cache-->>GeoSvc: Return cached station summaries
    else Cache Miss
        GeoSvc->>DB: SELECT * FROM stations WHERE ST_DWithin(geom, ST_MakePoint(lng,lat), 5000) AND connector='CCS2'
        DB-->>GeoSvc: Return matching station records & active ports
        GeoSvc->>Cache: SETEX geo_query_hash 60 station_json
    end

    GeoSvc-->>GW: HTTP 200 OK (GeoJSON payload)
    GW-->>UI: Deliver station pins with live availability
    UI->>MapAPI: Batch request map vector tiles
    MapAPI-->>UI: Deliver tile imagery
    UI-->>Driver: Display interactive map with green/amber/red station markers
```

---

### Sequence Diagram 2: Slot Reservation & Secure Payment Flow

This sequence models the atomic reservation protocol, ensuring distributed slot locking, third-party payment settlement, and dynamic QR pass generation.

```mermaid
sequenceDiagram
    autonumber
    actor Driver as 👤 EV Driver
    participant UI as 🖥️ Client UI
    participant BookSvc as 🎫 Booking Service
    participant LockSvc as 🔒 Redis Distributed Lock
    participant PayGW as 💳 Razorpay / Stripe
    participant DB as 🗄️ PostgreSQL Database
    participant Notif as 🔔 Notification Service

    Driver->>UI: Selects Station -> Chooses Slot #3 -> Clicks "Reserve"
    UI->>BookSvc: POST /api/v1/bookings/reserve {slotId: "S3", vehicleId: "V101"}
    
    BookSvc->>LockSvc: SET slot:lock:S3 NX PX 900000 (15 min TTL)
    alt Slot Already Locked by another user
        LockSvc-->>BookSvc: Lock Failed (Key exists)
        BookSvc-->>UI: HTTP 409 Conflict ("Slot is currently reserved")
        UI-->>Driver: Alert: "Slot taken. Select another port"
    else Lock Acquired Successfully
        LockSvc-->>BookSvc: Lock OK
        BookSvc->>DB: INSERT INTO bookings (status='PENDING', expires_at=NOW()+15m)
        DB-->>BookSvc: booking_id: "B-9024"
        BookSvc->>PayGW: Create Payment Order (₹50 reservation fee)
        PayGW-->>BookSvc: order_id: "order_k9x2m1"
        BookSvc-->>UI: HTTP 201 Created (order_id, booking_id, 15-min countdown)
        
        UI->>PayGW: Opens Checkout Sheet (UPI / Card Modal)
        Driver->>PayGW: Authorizes UPI / Card PIN
        PayGW-->>UI: Payment Authorized
        
        PayGW->>BookSvc: Webhook: payment.captured (signature verified)
        BookSvc->>DB: UPDATE bookings SET status='CONFIRMED', payment_id='PAY-441'
        BookSvc->>BookSvc: Generate Cryptographic QR Token
        BookSvc->>Notif: Send Booking Confirmation (Email/SMS + QR)
        Notif-->>Driver: Deliver SMS & Email confirmation
        BookSvc-->>UI: WebSocket Push: Booking Confirmed (Display QR Pass)
        UI-->>Driver: Show dynamic QR Pass & "Start Navigation" button
    end
```

---

### Sequence Diagram 3: Real-Time Hardware Telemetry Broadcasting

Illustrates the real-time IoT integration where physical dispensers update availability via MQTT and WebSockets.

```mermaid
sequenceDiagram
    autonumber
    participant Kiosk as ⚡ Charging Kiosk (OCPP)
    participant MQTT as 📡 MQTT / IoT Broker
    participant Ingestion as ⚙️ Telemetry Ingestion Worker
    participant DB as 🗄️ Database
    participant PubSub as 🔄 Redis Pub/Sub
    participant WS as 🌐 WebSocket Gateway
    actor Clients as 👥 Connected Drivers

    Kiosk->>MQTT: Publish topic: stations/ST-101/slots/S3/status {"status": "CHARGING"}
    MQTT->>Ingestion: Forward MQTT message
    Ingestion->>DB: UPDATE charging_slots SET status='CHARGING' WHERE slot_id='S3'
    Ingestion->>PubSub: PUBLISH station_updates {"stationId": "ST-101", "slot": "S3", "status": "OCCUPIED"}
    PubSub->>WS: Distribute event to active socket cluster
    WS->>Clients: Broadcast JSON: slot_status_change
    Clients-->>Clients: Update map pin color from Green (Available) to Amber/Red (In Use)
```

---

### UML Activity Diagram 1: Complete End-to-End User Booking Workflow

The activity diagram models user decisions, edge cases (payment failure, reservation timeout), and parallel actions (sending receipts while launching navigation).

```mermaid
flowchart TD
    Start([● Start: User Opens Locator App]) --> CheckGPS{GPS Location Allowed?}
    
    CheckGPS -- No --> ManualInput[Enter City or Landmark manually] --> FetchCoords[Geocode Coordinates]
    CheckGPS -- Yes --> AutoCoords[Acquire Lat/Lng from Device]
    
    ManualInput --> FetchCoords
    AutoCoords --> QueryNearby[Query Stations within Radius]
    FetchCoords --> QueryNearby
    
    QueryNearby --> RenderMap[Render Interactive Map & List]
    RenderMap --> FilterReq{Apply Filters?}
    
    FilterReq -- Yes --> SetFilter[Select Connector CCS2/Speed/Tariff] --> ApplyFilter[Dynamically Filter Stations] --> InspectStation[Select Station Card]
    FilterReq -- No --> InspectStation
    
    InspectStation --> CheckAvailability{Available Slots > 0?}
    
    CheckAvailability -- No --> NotifyWaitlist[Suggest Nearby Alternative Stations] --> RenderMap
    CheckAvailability -- Yes --> SelectSlot[Select Specific Port & Time Slot]
    
    SelectSlot --> CheckAuth{User Logged In?}
    CheckAuth -- No --> AuthModal[Prompt Login / Quick OTP Signup] --> CheckAuth
    CheckAuth -- Yes --> AcquireLock[Request 15-Minute Slot Lock]
    
    AcquireLock --> LockDecision{Lock Acquired?}
    LockDecision -- No --> RaceAlert[Port reserved just now by peer] --> InspectStation
    LockDecision -- Yes --> Checkout[Calculate Tariff & Open Payment Gateway]
    
    Checkout --> ProcessPay{Payment Status}
    
    ProcessPay -- Failed / Cancelled --> RetryPay{Retry Payment within 2 mins?}
    RetryPay -- Yes --> Checkout
    RetryPay -- No --> ReleaseSlot[Release Redis Lock to Public Inventory] --> EndFail([✖ Reservation Aborted])
    
    ProcessPay -- Success --> ForkSuccess[Fork Concurrent Confirmation Actions]
    
    ForkSuccess --> TaskA[Generate Encrypted Dynamic QR Pass]
    ForkSuccess --> TaskB[Send SMS/Email Digital Tax Invoice]
    ForkSuccess --> TaskC[Broadcast Status Change via WebSocket]
    
    TaskA --> JoinConfirm[Join Confirmation Tasks]
    TaskB --> JoinConfirm
    TaskC --> JoinConfirm
    
    JoinConfirm --> DriveToStation[Driver Navigates to Physical Charger]
    DriveToStation --> ArrivalDecision{Driver Arrives within 15 mins?}
    
    ArrivalDecision -- No (Timeout) --> AutoCancel[Expire Reservation & Assess No-Show Fee] --> FreePort[Release Slot Back to Available] --> EndTimeout([✖ Booking Expired])
    ArrivalDecision -- Yes --> ScanQR[Scan QR Code at Station Kiosk]
    
    ScanQR --> PlugIn[Plug Connector into Vehicle Socket]
    PlugIn --> ChargingLoop[Initiate Battery Charge Session]
    ChargingLoop --> SessionComplete[Battery Full / Stop Requested]
    SessionComplete --> Unplug[Unplug Cable & Final Settlement]
    Unplug --> Feedback[Prompt User for Review & Rating]
    Feedback --> EndSuccess([✔ End: Charging Session Completed])
```

---

### Activity Diagram 2: Station Operator Maintenance Workflow

```mermaid
flowchart TD
    OpStart([● Start: Operator Discovers Fault]) --> LoginOp[Authenticate with Operator Credentials]
    LoginOp --> OpenStationPanel[Open Station Management Console]
    OpenStationPanel --> PickPort[Select Defective Port ID]
    PickPort --> SetStatus[Change Status: AVAILABLE to UNDER_MAINTENANCE]
    SetStatus --> SystemCheck{Active Reservation Exists?}
    
    SystemCheck -- Yes --> Reassign[Auto-reassign Driver to Alternate Port + Send Apology SMS]
    SystemCheck -- No --> DirectUpdate[Commit Port Status Update]
    
    Reassign --> DirectUpdate
    DirectUpdate --> InvalidateCache[Purge Spatial Cache & Push Alert to Drivers]
    InvalidateCache --> DispatchTech[Log Maintenance Work Order for Field Technician]
    DispatchTech --> OpEnd([✔ End: Port Safely Isolated])
```

---

### Observation
Modeling dynamic workflows with sequence and activity diagrams revealed critical operational nuances:
1. Distributed concurrency: using Redis distributed locks with explicit 15-minute TTL guarantees prevents overbooking.
2. Graceful fault handling: separating payment timeout branches ensures locked inventory is systematically returned to public circulation.

---

### Result
UML Sequence Diagrams for Geospatial Search, Slot Reservation & Payment, and Telemetry Broadcasting were created. Comprehensive Activity Diagrams detailing end-to-end user reservation and maintenance workflows were successfully designed and verified.
