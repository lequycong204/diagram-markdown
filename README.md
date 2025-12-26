```mermaid
sequenceDiagram
    autonumber

    participant U as User App
    participant API as API Gateway
    participant TS as Trip Service
    participant DS as Driver Service
    participant PS as Payment Service
    participant MQ as RabbitMQ
    participant DB as MongoDB
    participant OSRM as OSRM API
    participant PG as Payment Gateway

    %% ================= USER REQUEST =================
    U ->> API: Request ride (HTTP / WebSocket)
    API ->> TS: gRPC CreateTrip()

    %% ================= TRIP CREATION =================
    TS ->> TS: Validate request
    TS ->> OSRM: Calculate route & ETA
    OSRM -->> TS: Route info
    TS ->> DB: Save trip
    TS ->> MQ: Publish event trip.created

    %% ================= DRIVER MATCHING =================
    MQ ->> DS: Consume trip.created
    DS ->> DB: Geo query nearest drivers
    DS ->> MQ: Publish event driver.matched
    MQ ->> API: driver.matched
    API ->> U: Notify matched driver

    %% ================= REAL-TIME TRACKING =================
    DS ->> MQ: Publish location.updated
    MQ ->> API: location.updated
    API ->> U: Push driver location (WebSocket)

    %% ================= PAYMENT =================
    TS ->> PS: Request payment
    PS ->> PG: Create charge
    PG -->> PS: Payment success
    PS ->> DB: Save payment
    PS ->> MQ: Publish payment.completed
    MQ ->> TS: payment.completed

    %% ================= TRIP COMPLETION =================
    TS ->> DB: Update trip status = completed
    TS ->> API: Trip finished
    API ->> U: Payment confirmation