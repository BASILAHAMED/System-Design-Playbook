# Uber / Ride-Sharing System Design

## 1. Introduction

Design a ride-sharing platform like Uber or Lyft that connects riders with drivers in real-time. The system must handle:
- Real-time location tracking
- Matching riders with nearby drivers
- Fare calculation
- Trip management
- Payments
- Ratings and reviews

## 2. Requirements

### Functional Requirements
- Rider app: request ride, track driver, view fare, payment
- Driver app: accept/reject rides, navigation, status updates
- Dispatch system: match riders with drivers
- Real-time location sharing
- Fare estimation
- Multiple ride types (economy, premium, pool)
- Ride history and receipts
- Ratings for both rider and driver
- In-app chat/call
- Cancellation with penalties

### Non-Functional Requirements
- Real-time response (< 1s for matching)
- High availability (99.99%)
- Low latency location updates (1-5s)
- Scalability to millions of users globally
- Consistency in fare and availability
- Security and fraud detection

## 3. Capacity Estimation

**Assumptions:**
- 20 million daily active users (10M riders, 2M drivers)
- 50 million rides per month (~1.6M/day)
- Peak: 100,000 rides/minute (~1,667/sec)
- Location updates: 1 update every 5 seconds per active user
- Active drivers: 1M
- Active riders: 5M

**Daily operations:**
- Location updates: (1M + 5M) × (86400/5) = ~1.04B updates/day
- Ride requests: 1.6M/day
- Messaging: ~10M messages/day

**Storage (5 years):**
- Rides: 1.6M/day × 365 × 5 = 2.9B records
- Location history: 1.04B/day × 365 × 5 = 1.9T records
- ~100TB raw data

**Bandwidth:**
- Location updates: 1.04B × 100 bytes = ~104GB/day (small)
- Main bandwidth: Maps, app data, photos

## 4. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        Client Layer                         │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐      │
│  │ Rider   │  │ Driver  │  │  Admin  │  │   Web   │      │
│  │  App    │  │  App    │  │  Panel  │  │ Portal  │      │
│  └────┬────┘  └────┬────┘  └────┬────┘  └────┬────┘      │
│       │           │           │           │              │
│       └───────────┴───────────┴───────────┘              │
└───────────────────────────┬───────────────────────────────┘
                            │
┌───────────────────────────▼───────────────────────────────┐
│                    API Gateway Layer                       │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  Load Balancer + Auth (JWT/OAuth)                   │  │
│  └────────────────────────────┬────────────────────────┘  │
└───────────────────────────────┼───────────────────────────┘
                                │
          ┌─────────────────────┼─────────────────────┐
          │                     │                     │
┌─────────▼─────────┐ ┌────────▼─────────┐ ┌────────▼─────────┐
│   Rider Service   │ │  Driver Service  │ │  Trip Service   │
│                   │ │                  │ │                 │
│ • Request ride    │ │ • Accept/reject  │ │ • Start/end     │
│ • Fare estimate   │ │ • Update status  │ │ • Track ETA     │
│ • Payment         │ │ • Navigation     │ │ • Route calc    │
└─────────┬─────────┘ └────────┬─────────┘ └────────┬─────────┘
          │                     │                     │
          └─────────────────────┼─────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Real-Time Services Layer                    │
│  ┌─────────────────────────────────────────────────────┐  │
│  │  WebSocket Servers (Socket.io/WebRTC)              │  │
│  │  • Location streaming (driver positions)           │  │
│  │  • Real-time notifications                         │  │
│  │  • In-app messaging                                │  │
│  └────────────────────────────┬────────────────────────┘  │
│                               │                          │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Location  │ │   Matching │ │ Notification│         │
│  │   Service   │ │   Engine   │ │  Service    │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
└───────────────────────────────┬───────────────────────────┘
                                │
┌───────────────────────────────▼───────────────────────────┐
│               Processing & Storage Layer                  │
│  ┌─────────────┐ ┌────────────┐ ┌────────────┐         │
│  │   Message   │ │Geospatial │ │   Payment  │         │
│  │    Queue    │ │Database   │ │  Service   │         │
│  │ (Kafka)     │ │(PostGIS)  │ │(Stripe/    │         │
│  │             │ │Redis Geo  │ │ Braintree) │         │
│  └─────────────┘ └────────────┘ └────────────┘         │
│                                                           │
│  ┌─────────────────────────────────────────────────────┐ │
│  │           Main Database (PostgreSQL)               │ │
│  │  • Users, rides, payments, reviews, metadata      │ │
│  └─────────────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────────────┘
```

## 5. Component Deep Dive

### 5.1 Location Tracking
**Why needed:** Real-time driver positions for matching and ETA calculation.

**Implementation:**
- Driver app sends GPS coordinates every 5 seconds via WebSocket
- Store in Redis with geospatial indexing (GEOADD, GEORADIUS)
- Keep last known location per driver
- Also persist to main DB for history/analytics (batch write)

**Storage format (Redis):**
```
key: driver:locations
member: driver_id
lon,lat: coordinates
```

**Expiry:** Auto-expire after 30 seconds (if driver goes offline)

### 5.2 Matching Engine
**Goal:** Find nearest available driver for a rider request.

**Process:**
1. Rider requests ride with pickup location (lat, lon)
2. Query Redis: `GEORADIUS driver:locations pickup 5km` (5km radius)
3. Filter by availability (online, not on trip)
4. Filter by vehicle type (economy, premium)
5. Sort by distance (and driver rating)
6. Send ride offer to top 5 drivers via push notification/WebSocket
7. First to accept gets the ride
8. If no response in 15 seconds, expand radius

**Optimization:**
- Use geohash for efficient range queries
- Pre-filter by driver status (online/offline)
- Consider driver's current destination (heading towards pickup?)
- ETA calculation using real-time traffic (Google Maps API)

**Scalability:** Partition by city/region if needed

### 5.3 Fare Estimation
**Factors:**
- Distance (pickup to destination)
- Time (duration)
- Demand/supply (surge pricing)
- Vehicle type
- Promotions/discounts
- Base fare

**Algorithm:**
```
base_fare + (distance * per_km_rate) + (time * per_min_rate)
× surge_multiplier (if demand > supply)
```

**Implementation:**
- Pre-calculate route distance/time using Maps API
- Cache frequent routes (airport → downtown)
- Surge pricing: dynamic based on area density

### 5.4 Trip Management
**States:**
```
REQUESTED → DRIVER_ASSIGNED → DRIVER_ARRIVED → 
TRIP_STARTED → TRIP_ENDED → PAYMENT_PENDING → COMPLETED
```

**Transitions:**
- Rider requests → state: REQUESTED
- Driver accepts → DRIVER_ASSIGNED
- Driver arrives at pickup → DRIVER_ARRIVED
- Trip starts (rider enters) → TRIP_STARTED
- Trip ends (destination) → TRIP_ENDED
- Payment processed → PAYMENT_PENDING → COMPLETED

**Cancellation:**
- Rider cancels before assignment: free
- Rider cancels after assignment: cancellation fee
- Driver cancels after assignment: penalty to driver

### 5.5 Database Design

**PostgreSQL schema:**

```sql
-- Users (riders and drivers share same table, role distinguishes)
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email VARCHAR(255) UNIQUE,
    phone VARCHAR(20) UNIQUE,
    name VARCHAR(200),
    role VARCHAR(20) CHECK (role IN ('rider', 'driver', 'both')),
    rating DECIMAL(3,2) DEFAULT 5.0,
    created_at TIMESTAMP
);

-- Driver-specific data
CREATE TABLE drivers (
    id UUID PRIMARY KEY REFERENCES users(id),
    license_number VARCHAR(100) UNIQUE,
    vehicle_id UUID REFERENCES vehicles(id),
    status VARCHAR(20) DEFAULT 'offline', -- online, offline, on_trip
    is_verified BOOLEAN DEFAULT FALSE,
    total_trips INT DEFAULT 0,
    earnings DECIMAL(10,2) DEFAULT 0
);

-- Vehicles
CREATE TABLE vehicles (
    id UUID PRIMARY KEY,
    driver_id UUID REFERENCES drivers(id),
    make VARCHAR(100),
    model VARCHAR(100),
    year INT,
    color VARCHAR(50),
    license_plate VARCHAR(20),
    vehicle_type VARCHAR(50), -- economy, premium, suv, van
    capacity INT DEFAULT 4
);

-- Rides
CREATE TABLE rides (
    id UUID PRIMARY KEY,
    rider_id UUID REFERENCES users(id),
    driver_id UUID REFERENCES drivers(id),
    vehicle_id UUID REFERENCES vehicles(id),
    status VARCHAR(30),
    pickup_lat DECIMAL(10,8),
    pickup_lng DECIMAL(11,8),
    pickup_address TEXT,
    dropoff_lat DECIMAL(10,8),
    dropoff_lng DECIMAL(11,8),
    dropoff_address TEXT,
    distance_km DECIMAL(6,2),
    duration_min INT,
    fare_estimate DECIMAL(10,2),
    fare_final DECIMAL(10,2),
    surge_multiplier DECIMAL(3,2) DEFAULT 1.0,
    payment_method VARCHAR(50),
    payment_status VARCHAR(20) DEFAULT 'pending',
    requested_at TIMESTAMP,
    accepted_at TIMESTAMP,
    arrived_at TIMESTAMP,
    trip_started_at TIMESTAMP,
    trip_ended_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason TEXT
);

-- Ride waypoints (for tracking route)
CREATE TABLE ride_waypoints (
    id BIGSERIAL PRIMARY KEY,
    ride_id UUID REFERENCES rides(id),
    lat DECIMAL(10,8),
    lng DECIMAL(11,8),
    timestamp TIMESTAMP
);

-- Reviews
CREATE TABLE reviews (
    id BIGSERIAL PRIMARY KEY,
    ride_id UUID REFERENCES rides(id),
    reviewer_id UUID REFERENCES users(id), -- who wrote review
    reviewee_id UUID REFERENCES users(id), -- who is reviewed
    rating INT CHECK (rating >= 1 AND rating <= 5),
    comment TEXT,
    created_at TIMESTAMP
);

-- Payments
CREATE TABLE payments (
    id UUID PRIMARY KEY,
    ride_id UUID REFERENCES rides(id) UNIQUE,
    amount DECIMAL(10,2),
    method VARCHAR(50), -- card, cash, wallet
    status VARCHAR(20), -- pending, completed, failed, refunded
    transaction_id VARCHAR(200),
    created_at TIMESTAMP
);
```

**Indexes:**
- `rides(rider_id, requested_at)` for rider history
- `rides(driver_id, status)` for driver active trips
- `rides(pickup_lat, pickup_lng)` for geospatial queries (PostGIS extension)
- `users(phone)`, `users(email)` for login

### 5.6 Payment System
- **Integration:** Stripe/Braintree/PayPal for card payments
- **Pre-authorization:** Hold estimated fare amount before trip
- **Final charge:** Adjust based on actual fare + tips
- **Payouts:** Weekly/daily to driver bank account
- **Wallet:** Optional stored balance

**Flow:**
1. Rider adds payment method → tokenized card stored with payment provider
2. Ride request → pre-authorize estimated fare
3. Trip ends → calculate final fare
4. Charge final amount (capture pre-authorization)
5. Process payout to driver (minus commission)
6. Send receipt

### 5.7 Notification Service
- **Push notifications:** Firebase Cloud Messaging (FCM) / Apple Push Notification Service (APNs)
- **SMS fallback:** Twilio for critical notifications (OTP, ride status)
- **In-app:** WebSocket for real-time updates

**Events:**
- Ride request → driver notification
- Driver accepted → rider notified
- Driver arrived → rider notified
- Trip started/ended → both parties
- Payment received

### 5.8 Rating System
- After trip completion, both rider and driver can rate each other (1-5 stars)
- Average rating stored in users table
- Low-rated users/drivers flagged for review
- One rating per ride (prevent duplicate)

## 6. Scalability Considerations

### 6.1 Database Scaling
- **Read replicas:** For rider/driver profiles, ride history queries
- **Sharding:** By city or user_id prefix for massive scale
- **Caching:** Redis for active rides, driver locations, popular routes
- **Time-series data:** Ride waypoints can go to TimescaleDB or Cassandra

### 6.2 Real-time Scaling
- **WebSocket servers:** Stateless, behind load balancer
- **Connection routing:** Use consistent hashing to map user → server
- **Redis Pub/Sub:** For broadcasting messages across server instances
- **Socket affinity:** Sticky sessions for WebSocket connections

### 6.3 Microservices Breakdown
- Rider Service
- Driver Service
- Trip Service
- Matching Service
- Location Service
- Payment Service
- Notification Service
- Review Service
- Analytics Service

**Communication:** REST APIs + Message Queue (Kafka) for async events

## 7. Advanced Features

### 7.1 Surge Pricing
- Dynamic pricing based on demand/supply ratio
- Algorithm: `surge_multiplier = f(demand, supply, area)`
- Show surge multiplier to rider before request
- Cap maximum surge (e.g., 3x)

**Implementation:**
- Track real-time demand (ride requests per area)
- Track supply (available drivers per area)
- Calculate ratio → lookup table or ML model
- Store surge map in Redis (city → zones → multiplier)

### 7.2 Pool / Shared Rides
- Multiple riders going same direction
- Algorithm: Match riders with overlapping routes
- Fare split among riders
- Dynamic routing (pickup A → pickup B → dropoff A → dropoff B)

**Challenges:**
- Increased complexity in matching
- Longer ETA for some riders
- Need to optimize for total cost/ETA

### 7.3 Scheduling
- Book ride for future time
- Reserve driver in advance
- Reminder notifications before scheduled pickup

### 7.4 In-App Navigation
- Integrated Google Maps/Apple Maps
- Show route to driver for pickup
- Real-time traffic updates
- ETA calculation

### 7.5 Safety Features
- Share ride status with emergency contacts
- Panic button (connect to authorities)
- Ride tracking for trusted contacts
- Driver background checks (verification service)

## 8. Fault Tolerance & Reliability

- **Multi-region deployment:** Active-active for critical services
- **Database replication:** Primary-replica with failover
- **Message queue durability:** Kafka with replication factor 3
- **Graceful degradation:** If matching fails, show "no drivers available"
- **Retry logic:** Exponential backoff for payment calls
- **Idempotency:** Prevent duplicate charges/payments

## 9. Monitoring & Alerting

- **Metrics:** Active drivers, ride requests, match rate, avg wait time, avg trip duration
- **Alerts:** Payment failures, high cancellation rate, low driver availability
- **Dashboards:** Real-time city-level demand/supply heatmaps
- **Logging:** Structured logs for ride lifecycle, payment transactions

## 10. Security Considerations

- **Authentication:** JWT tokens with short expiry, refresh tokens
- **Authorization:** Role-based access (rider vs driver)
- **Rate limiting:** Prevent abuse of API endpoints
- **Data encryption:** TLS everywhere, encrypt sensitive data at rest
- **Fraud detection:** Suspicious patterns (fake rides, payment fraud)
- **GDPR/Privacy:** User data deletion, consent management

## 11. Trade-offs

| Decision | Option A | Option B | Trade-off |
|----------|----------|----------|-----------|
| Matching | Centralized service | Distributed (by city) | Consistency vs Latency |
| Location storage | Redis only | Redis + DB | Speed vs History |
| Payment | Pre-authorization | Post-charge | Guarantee vs Flexibility |
| Database | PostgreSQL | Cassandra | Consistency vs Scale |
| Real-time | WebSocket | Server-Sent Events | Bi-directional vs Simplicity |

## 12. Future Enhancements

- Autonomous vehicles integration
- Drone delivery for packages
- Public transit integration
- Carbon footprint tracking
- Subscription plans (monthly ride credits)
- Voice assistant booking (Alexa/Google Home)
- AR navigation for drivers
- Predictive demand forecasting (ML)

## 13. References

- [Uber Engineering Blog](https://eng.uber.com/)
- [Designing Uber's Architecture](https://www.youtube.com/watch?v=um0Gv4oF4vA)
- [Geospatial Queries in Redis](https://redis.io/commands#geospatial)
- [PostGIS Documentation](https://postgis.net/documentation/)

---

**Difficulty:** Hard 🔴
**Time to design:** 45-60 minutes
**Key focus areas:** Real-time systems, geospatial queries, matching algorithms, scalability