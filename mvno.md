# Telecom Platform — MVNO (Java) + MNO Simulator (Go)

A learning project for cross-service communication and distributed systems, modeled on the real-world MVNO/MNO relationship (MNO ≈ MTN equivalent).

## Core Concept

Not "MVNO pretending to own a network." Instead, two independently deployable systems:

- **Java MVNO** — the virtual operator / business + service layer (subscribers, products, orders, billing, charging, provisioning)
- **Go MNO Simulator** — the wholesale network provider exposing network capabilities to the MVNO (capacity, SIM/MSISDN, usage simulation, network events)

GSMA describes MVNOs as leasing network capacity from MNOs on a spectrum of models — a full MVNO owns much of its own BSS/core, a thin MVNO relies more on the host MNO. The exact division of network functions depends on the model, which is why this project keeps the two systems cleanly separated from day one so responsibilities can shift between them as it evolves.

## Architecture

```
Customer → Mobile App
              │
              ▼
    ┌──────────────────────┐
    │     MVNO - Java      │
    │  Subscriber          │
    │  Products            │
    │  Orders              │
    │  Billing             │
    │  Charging            │
    │  Provisioning        │
    └──────────┬───────────┘
               │ Wholesale API (HTTP / gRPC)
               ▼
    ┌──────────────────────┐
    │      MNO - Go        │
    │  Subscriber Network  │
    │  Capacity             │
    │  SIM/MSISDN           │
    │  Usage Simulation     │
    │  Network Events       │
    └──────────┬───────────┘
               │ Simulated Network
       ┌───────┼───────┐
       ▼       ▼       ▼
     Data     SMS     Voice
```

## Repo Layout

```
telecom-platform/
├── mvno/                 # Java / Spring Boot
├── mno-simulator/         # Go
├── contracts/             # OpenAPI / protobuf + event schemas
│   ├── mno-api.yaml
│   └── events/
│       ├── subscriber-provisioned.json
│       ├── data-usage.json
│       ├── sms-usage.json
│       └── voice-usage.json
├── infrastructure/
│   └── docker-compose.yml
└── README.md
```

One repository initially, but run as **two processes / two containers**, each with its **own database** (`mvno-postgres`, `mno-postgres`). No shared tables, no direct cross-database reads — communication only via HTTP or Kafka. This is the boundary the whole project exists to teach; losing it defeats the purpose.

Mental model: *one repository, two organizations, two runtimes, two databases* — even if both run on the same laptop.

## V1 Scope (keep it small)

No Kafka, Kubernetes, Raft, IMS, Diameter, or HSS yet. V1 answers one question:

> Can a customer buy mobile service from the MVNO, have the MVNO request that service from the MNO, and receive simulated network usage?

### MNO Simulator — V1

**1. Subscriber/network identity**
```
SIM          MSISDN        NetworkSubscriber
├ iccid      ├ number      ├ imsi
├ imsi       └ status      ├ msisdn
└ status                   └ serviceStatus
```
No need to reproduce a real HLR/HSS yet — just: MVNO asks "can you provision this subscriber?", MNO responds "IMSI X is now associated with MSISDN Y."

**2. Network capacity (wholesale model)**

Model it as a commercial/network resource allocation agreement, not literal disk-like capacity:

```go
WholesaleAgreement {
    id
    mvnoId
    dataCapacity
    smsCapacity
    voiceMinutes
    startDate
    endDate
    pricingModel
}
```
The MNO simulator enforces this (e.g., MVNO has 20 TB/day allowance, tracks usage against remaining).

**3. MNO API — ~5 endpoints for V1**
- `POST /wholesale/subscribers` — provision subscriber
- `POST /wholesale/subscribers/{id}/suspend`
- `POST /wholesale/subscribers/{id}/activate`
- `GET /wholesale/subscribers/{id}/usage`
- `GET /wholesale/capacity`

**4. Usage generation (the key feature)**

The MNO generates network events — this is the foundation for everything downstream:
```json
{
  "eventId": "evt_123",
  "type": "DATA_USAGE",
  "imsi": "123456789",
  "msisdn": "08012345678",
  "bytes": 52428800,
  "timestamp": "..."
}
```
Event types: `DATA_USAGE`, `SMS_USAGE`, `VOICE_USAGE`.

### MVNO — V1

Single Spring Boot app, modular (not microservices yet): `Customer`, `Subscriber`, `Product`, `Order`, `Subscription`, `Provisioning`, `Charging`, `MNO Integration`.

**1. Products** — e.g. 5GB/₦2,000/30 days, 20GB/₦5,000/30 days.

**2. Purchase flow**
```
POST /subscriptions { subscriberId, productId }
Create Order → Payment (mock: always SUCCESS) → Create Subscription
→ Ask MNO to provision → Activate subscription
```
Don't let payment integration distract from the telecom architecture.

**3. MVNO → MNO call** — REST/HTTP to start (Spring `WebClient`, Go `net/http`). This is where cross-service learning begins: two independently running programs communicating over the network.

**4. Then add async communication** — once sync flow works, introduce `MNO → Kafka → MVNO` for usage events. The point is to *feel* the difference between:
- **Request/response** (HTTP): timeouts, connection failures, retries, API contracts, auth
- **Event-driven** (Kafka): producers, consumers, partitions, offsets, consumer groups, ordering, replay, delivery semantics, backpressure

### V1 End-to-End Demo Scenario

1. Create customer: Alice, 08012345678
2. MVNO assigns SIM / IMSI / MSISDN
3. Alice purchases 5GB / 30 days
4. MVNO calls `POST /wholesale/subscribers`
5. MNO provisions her → `NetworkSubscriber ACTIVE`
6. MNO simulator generates usage: 100MB, 250MB, SMS, 500MB
7. MNO emits `DATA_USAGE`, `DATA_USAGE`, `SMS_USAGE`, `DATA_USAGE`
8. MVNO consumes events, balance decrements: 5GB → 4.9GB → 4.65GB → 4.15GB

## Roadmap

| Version | Focus |
|---|---|
| **V1** | Java MVNO ↔ Go MNO over REST, one DB each, basic provisioning + usage |
| **V2** | Wholesale agreements — MNO manages capacity/pricing per MVNO, rejects requests beyond agreement |
| **V3** | Kafka usage event pipeline (MNO → Kafka → MVNO), fully async |
| **V4** | Charging — MVNO turns usage into rating/charging/balance updates |
| **V5** | Wholesale settlement — MNO usage vs. MVNO usage → reconciliation & discrepancies |
| **V6** | Failure engineering — timeouts, retries, exponential backoff, idempotency, circuit breakers; what happens when Kafka delivery fails or events are reprocessed |
| **V7** | Scale — 100 → 10K → 1M subscribers, 1 → 100K events/sec; partitioning, batching, caching, read replicas, sharding, load balancing, rate limiting, backpressure |
| **V8** | Richer network simulation — regions/towers (Lagos, Abuja, Benin), events carry tower, technology, latency, signal strength |
| **V9** | Data science on synthetic network data — worst-performing towers, churn prediction, network quality vs. churn, plan margin analysis, promotion impact, tower-outage effects, congestion vs. speed |

## Key Design Principles

- **Separate databases, always.** Communication only through HTTP/Kafka — never direct DB reads across services.
- **Contracts directory** (`contracts/`) defines the interface boundary explicitly (OpenAPI/protobuf + JSON event schemas). Can evolve REST → gRPC → async events later without prematurely picking tech.
- **Don't over-engineer the MNO simulator early.** Its V1 job: "I'm the network provider — here's the capacity you've purchased, I provision your subscribers, and I generate the usage your customers produce."
- **The MVNO's job:** "I sell connectivity to customers and use the MNO's network to fulfill it."
- GSMA's telecom billing/charging work covers usage records, billing reports, charging documents, and reconciliation between providers — directly relevant to the V5 settlement/reconciliation phase.

## End State (rough shape)

```
Java MVNO (BSS, Billing, Charging, Subscribers)
        │ HTTP / gRPC
        ▼
Go MNO (Wholesale, Network, Simulator)
        │ Kafka
        ▼
Usage Events
        │
        ▼
Java MVNO (Mediation, Rating, Charging)
```

From there: room to grow into replication, partitioning, multi-region operation, failure injection, reconciliation, observability, capacity planning, and ML on synthetic telecom data.