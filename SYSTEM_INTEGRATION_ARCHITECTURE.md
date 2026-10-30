# System Integration and Architecture (SIA)
## Final Project Technical Specification & Architecture Manual
**System Title:** `ExplorePH: Ormoc Tourism Access & GPS Smart Navigation Portal`  
**Target Organization:** City Government of Ormoc / Tourism & Investment Promotions Office  
**Security & Architecture Standard:** IAS-1 Compliant / Cloud-Distributed Microservice Architecture

---

## 1. Use Case Diagram (System Users & Functionalities)

### Actors Identified:
1. **Tourist / Traveler (Primary User)**: Discovers destinations, views road route navigation, checks GPS distances, bookmarks spots, submits reviews, tracks visited locations.
2. **Tourism Staff / Content Manager (Authorized Contributor)**: Proposes new tourist spots, pins GPS coordinates, updates hours/fees/media.
3. **System Administrator (Governance & Security)**: Approves/declines destination proposals, manages user roles and locks, audits system security log trails, performs database snapshot backups & restores.
4. **Guest Traveler (Unauthenticated Actor)**: Browses public spot listings, searches destinations, views maps and GPS distances.
5. **External Routing & Geospatial Services (Secondary Actors)**: OpenStreetMap OSRM Routing Engine, Nominatim Geocoder, Leaflet Maps, Supabase BaaS.

```mermaid
flowchart LR
    subgraph Actors["System Actors"]
        Guest["👤 Guest Traveler"]
        Tourist["👤 Registered Tourist"]
        Staff["👤 Tourism Staff"]
        Admin["👤 System Administrator"]
        OSRM["🌐 OSRM / OpenStreetMap"]
        Supabase["☁️ Supabase Cloud DB"]
    end

    subgraph Portal["ExplorePH: Ormoc Tourism Access Portal"]
        UC1["Explore Spots & Filter Categories"]
        UC2["Live GPS Locate & Haversine Distance Calc"]
        UC3["Turn-by-Turn Road Route Tracing"]
        UC4["Authenticate (Sign In / Register / OTP Reset)"]
        UC5["Manage Watchlist (Bookmarks & Visited Spots)"]
        UC6["Submit Reviews & Ratings"]
        UC7["Propose New Destination Pin"]
        UC8["Approve / Decline Spot Proposals"]
        UC9["User Account Management & Role Assignment"]
        UC10["Audit Log Trail Monitoring"]
        UC11["Database Snapshot Backup & Restore"]
    end

    Guest --> UC1
    Guest --> UC2

    Tourist --> UC1
    Tourist --> UC2
    Tourist --> UC3
    Tourist --> UC4
    Tourist --> UC5
    Tourist --> UC6

    Staff --> UC4
    Staff --> UC7
    Staff --> UC1
    Staff --> UC3

    Admin --> UC4
    Admin --> UC8
    Admin --> UC9
    Admin --> UC10
    Admin --> UC11

    UC3 -.->|External API Call| OSRM
    UC4 -.->|Auth & RLS| Supabase
    UC10 -.->|Record Events| Supabase
    UC11 -.->|Snapshot Data| Supabase
```

---

## 2. System Flow Diagram (Swimlanes and Pools)

Illustrates the end-to-end operational handoff across participants: **Tourist**, **Tourism Staff (LGU)**, **System Admin**, and the **Automated Cloud Engine**.

```mermaid
sequenceDiagram
    autonumber
    actor Tourist as 👤 Tourist / User
    actor Staff as 🏛️ Tourism Staff (LGU)
    actor Admin as 🛡️ System Administrator
    participant System as 📱 ExplorePH Web Engine
    participant OSRM as 🛣️ OSRM Road Router
    participant DB as 🗄️ Database & Security Log

    %% Tourism Content Lifecycle
    rect rgb(24, 24, 24)
    note right of Staff: Phase 1: Destination Content Proposal
    Staff->>System: Pin GPS point, input spot details, hours, fee schedule
    System->>DB: INSERT INTO destinations (status='pending')
    DB-->>System: Stored as pending proposal
    System->>DB: INSERT INTO log_trails (action='SUBMIT_DESTINATION')
    end

    rect rgb(28, 20, 20)
    note right of Admin: Phase 2: Administrative Review & Governance
    Admin->>System: Review pending tourism proposals list
    alt Proposal Approved
        Admin->>System: Click "Approve Spot"
        System->>DB: UPDATE destinations SET status='approved'
        System->>DB: INSERT INTO log_trails (action='REVIEW_DESTINATION')
    else Proposal Declined
        Admin->>System: Click "Decline Spot"
        System->>DB: UPDATE destinations SET status='declined'
    end
    end

    rect rgb(20, 28, 24)
    note right of Tourist: Phase 3: Discovery, GPS Proximity & Road Tracing
    Tourist->>System: Open Portal, request live GPS position ("Locate Me")
    System->>System: Calculate Haversine Distances (km) to all Ormoc spots
    Tourist->>System: Click "Trace on Map" for Lake Danao
    System->>OSRM: GET /driving/{user_lng},{user_lat};{spot_lng},{spot_lat}
    OSRM-->>System: Return GeoJSON Road Network Coordinates & Duration
    System->>Tourist: Render high-fidelity road polyline on Leaflet canvas
    Tourist->>System: Mark spot as "Visited" or "Bookmark"
    System->>DB: UPSERT INTO spot_visits / user_bookmarks
    System->>DB: INSERT INTO log_trails (action='MARK_VISITED')
    end
```

---

## 3. Entity Relationship Diagram (ERD)

Defines all database entities, keys, field types, and relational cardinalities supporting **IAS-1 Security** and **Ormoc Tourism Data**:

```mermaid
erDiagram
    USERS ||--o{ DESTINATIONS : "submits (1:N)"
    USERS ||--o{ LOG_TRAILS : "generates (1:N)"
    USERS ||--o{ REVIEWS : "writes (1:N)"
    USERS ||--o{ USER_BOOKMARKS : "saves (1:N)"
    USERS ||--o{ SPOT_VISITS : "records (1:N)"
    DESTINATIONS ||--o{ REVIEWS : "receives (1:N)"
    DESTINATIONS ||--o{ USER_BOOKMARKS : "bookmarked_in (1:N)"
    DESTINATIONS ||--o{ SPOT_VISITS : "visited_in (1:N)"
    DESTINATIONS ||--o| DESTINATION_ANALYTICS : "tracks (1:1)"

    USERS {
        bigint id PK
        varchar full_name
        varchar username UK
        varchar email UK
        varchar password
        varchar role "admin | staff | user"
        boolean is_active
        boolean is_locked
        int failed_attempts
        timestamptz lockout_until
        varchar otp_code
        timestamptz otp_expires_at
        timestamptz created_at
        timestamptz updated_at
    }

    DESTINATIONS {
        bigint id PK
        bigint submitted_by FK
        varchar name
        varchar category "Nature | Adventure | Leisure | Heritage"
        text location_address
        text description
        double_precision latitude
        double_precision longitude
        numeric rating
        varchar status "pending | approved | declined"
        jsonb images
        timestamptz created_at
    }

    LOG_TRAILS {
        bigint id PK
        bigint user_id FK
        varchar username_used
        varchar action
        text details
        varchar ip_address
        timestamptz created_at
    }

    REVIEWS {
        bigint id PK
        bigint destination_id FK
        bigint user_id FK
        varchar username
        int rating "1 to 5"
        text comment
        timestamptz created_at
    }

    USER_BOOKMARKS {
        bigint id PK
        bigint user_id FK
        bigint destination_id FK
        timestamptz created_at
    }

    SPOT_VISITS {
        bigint id PK
        bigint user_id FK
        bigint destination_id FK
        timestamptz created_at
    }

    DESTINATION_ANALYTICS {
        bigint id PK
        bigint destination_id FK
        int total_visitors
        int repeat_visitors
        numeric return_rate_pct
        timestamptz updated_at
    }
```

---

## 4. System Flowchart (Core Logic & Decision Flow)

Represents system inputs, validation conditions, progressive lockouts, road tracing logic, and data storage workflows.

```mermaid
flowchart TD
    Start(["Start / Access Portal"]) --> CheckSession{"Has active session?"}

    %% Authentication Decision Branch
    CheckSession -- No --> AuthChoice{"Action Selected?"}
    AuthChoice -- "Continue as Guest" --> GuestMode["Set Guest Mode (Limited Tools)"] --> BrowseSpots["Load Approved Ormoc Spots"]
    AuthChoice -- "Sign In" --> EnterCreds[/"Input Username & Password"/]
    EnterCreds --> CheckLockout{"Is account locked?"}
    
    CheckLockout -- Yes --> ShowTimer[/"Display Lockout Timer (1m/5m/15m)"/] --> StopAuth(["Halt Attempt"])
    CheckLockout -- No --> CheckCaptchaReq{"Failed >= 3 attempts?"}
    
    CheckCaptchaReq -- Yes --> ValidateCaptcha{"Verify CAPTCHA code"}
    ValidateCaptcha -- Failed --> NewCaptcha["Regenerate CAPTCHA"] --> EnterCreds
    ValidateCaptcha -- Passed --> VerifyPass
    CheckCaptchaReq -- No --> VerifyPass{"Validate Password Hash"}

    VerifyPass -- Incorrect --> IncFail["failed_attempts += 1"]
    IncFail --> CheckAttempts{"failed_attempts >= 3?"}
    CheckAttempts -- Yes --> TriggerLock["Set lockout_until (1m/5m/15m) & Enable CAPTCHA"]
    TriggerLock --> AuditFail["Log ACCOUNT_LOCKOUT to log_trails"] --> StopAuth
    CheckAttempts -- No --> AuditFailLogin["Log FAILED_LOGIN to log_trails"] --> EnterCreds

    VerifyPass -- Correct --> ResetFails["failed_attempts = 0, lockout = null"]
    ResetFails --> InitSession["Store Session & Start 15-min Inactivity Watchdog"]
    InitSession --> RoleCheck{"Check User Role"}

    %% Role-based Dashboards
    RoleCheck -- "Admin" --> AdminDash["Display Admin Console (Approvals, Logs, Backups)"]
    RoleCheck -- "Staff" --> StaffDash["Display Staff Console (Propose Spots, Pinned Map)"]
    RoleCheck -- "Tourist" --> TouristDash["Display Tourist Dashboard (Watchlist, Reviews)"]
    CheckSession -- Yes --> TouristDash

    %% GPS & Navigation Subsystem
    TouristDash --> NavAction{"Navigation Action?"}
    NavAction -- "Locate Me" --> QueryGPS[/"Query Geolocation API"/]
    QueryGPS --> Haversine["Compute Haversine Distances (km) to Spots"]
    Haversine --> SortList["Sort Drawer: Nearest to Farthest Spot"]
    SortList --> UpdatePins["Render Dynamic Distance Badges on Pins"]

    NavAction -- "Trace on Map" --> CallOSRM["Query OSRM Road Router API with origin & spot coords"]
    CallOSRM --> RoadCheck{"OSRM Status OK?"}
    RoadCheck -- Yes --> DrawRoad["Render actual curved road geometry polyline + ETA"]
    RoadCheck -- No (Offline) --> FallbackLine["Render direct dashed line trace"]
    DrawRoad --> FitBounds["Fit Map Camera Bounds"]
    FallbackLine --> FitBounds

    %% Session Expiration / Inactivity
    TouristDash --> InactivityCheck{"Inactivity >= 15 mins?"}
    InactivityCheck -- Yes --> AutoLogout["Clear Session, Broadcast Cross-Tab Logout, Log AUTO_LOGOUT"]
    AutoLogout --> Start
```

---

## 5. Network Diagram (Physical & Logical Infrastructure Flow)

Illustrates communication channels, encrypted protocol transport (`HTTPS/TLS 1.3`, `WSS`), firewalls, caching layers, application runtime, and external services.

```mermaid
flowchart TD
    subgraph ClientTier["Client Device Layer (Edge)"]
        Mobile["📱 Mobile Smartphone (GPS Enabled)"]
        Workstation["💻 Desktop / Admin Workstation"]
    end

    subgraph Perimeter["Perimeter Security & CDN"]
        DNS["🌐 DNS / Cloudflare Edge CDN (DDoS Protection)"]
        WAF["🛡️ Web Application Firewall & SSL Termination"]
    end

    subgraph WebServerTier["Application Hosting Tier (Local / Cloud Host)"]
        Apache["🖥️ Web Server (XAMPP Apache / Nginx Engine)"]
        StaticEngine["📦 Single Page Application (ExplorePH HTML5, Tailwind, Leaflet.js)"]
    end

    subgraph ExternalServices["External Geospatial Cloud Engines"]
        OSRM["🛣️ OpenStreetMap OSRM Driving Engine (router.project-osrm.org)"]
        OSMTiles["🗺️ OpenStreetMap Tile Servers ({s}.tile.openstreetmap.org)"]
        Nominatim["📍 Nominatim Reverse Geocoder API"]
    end

    subgraph DataTier["Managed Cloud Database Tier (Supabase)"]
        API_GW["🚪 PostgREST / GoTrue Auth Gateway (Port 443 HTTPS)"]
        PG_DB[("🐘 PostgreSQL Distributed Database (Tables, Keys, RLS)")]
        Storage[("🗄️ Supabase Object Storage (Destination Photos Bucket)")]
    end

    %% Network Connections
    Mobile -->|HTTPS / TLS 1.3 (Port 443)| DNS
    Workstation -->|HTTPS / TLS 1.3 (Port 443)| DNS
    DNS --> WAF
    WAF --> Apache
    Apache --> StaticEngine

    StaticEngine -->|HTTPS Tile Requests (Port 443)| OSMTiles
    StaticEngine -->|Turn-by-Turn Road Route Requests| OSRM
    StaticEngine -->|Reverse Coordinate Geocoding| Nominatim

    StaticEngine -->|RESTful CRUD & Auth (JWT Bearer Token)| API_GW
    API_GW -->|Connection Pool (PgBouncer)| PG_DB
    API_GW -->|Public CDN URL Fetch| Storage
```
