# NIBSS Simulator (Java / Spring Boot)

A production-shaped simulation of the **NIBSS Instant Payment (NIP)** interbank switch, built so that [`credora-backend`](https://github.com/EfosaE/credora-backend) can implement and exercise an **external interbank transfer** flow — something no fintech can legally do against the real NIBSS network without a CBN operating license or a licensed aggregator relationship (Monnify, Paystack, Flutterwave, Mono, etc.).

This service does not talk to NIBSS. It **is** a stand-in NIBSS: a separate Java system that owns bank routing, name enquiry, transfer processing, status tracking and callbacks — the same responsibilities the real switch owns — so that Credora is forced to solve the same class of distributed-systems problems it would face integrating with a real payment rail.

---

## Table of contents

- [Why this exists](#why-this-exists)
- [How this maps to the real NIBSS](#how-this-maps-to-the-real-nibss)
- [System architecture](#system-architecture)
- [How Credora integrates](#how-credora-integrates)
- [Domain model](#domain-model)
- [API reference](#api-reference)
- [Response codes](#response-codes)
- [Session ID / reference format](#session-id--reference-format)
- [Failure & latency simulation](#failure--latency-simulation)
- [Security model](#security-model)
- [Compliance mirroring (non-binding)](#compliance-mirroring-non-binding)
- [Tech stack](#tech-stack)
- [Project structure](#project-structure)
- [Build order (V1 → V3)](#build-order-v1--v3)
- [Test scenarios](#test-scenarios)
- [Running locally](#running-locally)

---

## Why this exists

`credora-backend` today is a real digital-banking backend (Go, Chi, PostgreSQL via `pgx`/`sqlc`, Redis + Asynq, double-entry ledger, idempotent webhook ingestion) that uses **Monnify** as a wrapper bank for virtual accounts and inbound funding. It has **no outbound external-transfer interface** — Credora cannot yet send money to an arbitrary account at another Nigerian bank.

In production, that gap would be closed by integrating with a licensed payment rail — Monnify, Paystack, or (for a licensed institution) NIBSS directly. This project simulates that rail so Credora can implement, and be tested against, a realistic **outbound transfer** path: name enquiry → debit → route → settle/fail → reconcile.

The design goal is explicitly **not** "clone every NIBSS endpoint." It's to build the minimum surface area — bank directory, name enquiry, transfer initiation, status query, and asynchronous callbacks — with **realistic failure modes** (timeouts, lost callbacks, duplicate retries, partial completion) layered on top, because that's where the actual engineering lessons live.

## How this maps to the real NIBSS

NIBSS (Nigeria Inter-Bank Settlement System) operates **NIP (NIBSS Instant Payment)**, the real-time interbank funds transfer rail used across Nigerian banks and licensed fintechs. Per NIBSS's own service description, NIP exposes a small set of core operations that this simulator deliberately mirrors:

| Real NIP capability | Simulated here |
|---|---|
| Name Enquiry — validate a beneficiary account and return the account name before a transfer is initiated | ✅ `POST /name-enquiry` |
| Funds Transfer (Direct Credit) — move funds into a beneficiary account | ✅ `POST /transfers` |
| Transaction Status Query (TSQ) — query the status of a previously sent transaction | ✅ `GET /transfers/{reference}` |
| Balance Enquiry | Out of scope for V1 (Credora doesn't need it for outbound transfers) |
| Funds Transfer (Direct Debit) / standing orders | Out of scope |

NIP is a **Deferred Net Settlement (DNS)** system: funds appear in the beneficiary's account online, in real time, before the underlying interbank settlement is finalized in one of the scheme's settlement sessions. That real-time-availability-before-settlement property is exactly why reconciliation, idempotency, and status-query machinery matter — a bank can't just "wait for settlement" to know whether a transfer succeeded, and neither can Credora.

In the real world, access to NIP is not self-service. A financial institution or PSSP must go through NIBSS certification — a formal letter of intent, application and device certification, multi-week code audits and penetration testing — before going live. Smaller fintechs almost always integrate indirectly through a licensed aggregator instead. That's the exact reason this project exists as a **simulator**: it lets Credora behave like it holds that access without requiring a license, a leased line, or a certification cycle.

## System architecture

```
                         YOUR SYSTEM
┌──────────────────────────────────────────────────────────┐
│                         CREDORA (Go)                     │
│                                                            │
│  Chi API → Transfer Service → Ledger (Postgres)          │
│                         │                                 │
│                         ▼                                 │
│                  PaymentRail interface                    │
│                         │                                 │
│                  NIBSSSimulatorClient                     │
└─────────────────────────┬──────────────────────────────────┘
                          │ REST over HTTP (+ HMAC signature)
                          ▼
              ┌─────────────────────────┐
              │   NIBSS SIMULATOR       │
              │   Java 21 / Spring Boot │
              │                         │
              │  Bank Directory         │
              │  Name Enquiry           │
              │  Transfer Router        │
              │  Transaction Processor  │
              │  Status Manager         │
              │  Callback Dispatcher    │
              │  Simulation Control     │
              └──────┬─────────┬────────┘
                     │         │
              ┌──────▼──┐  ┌───▼─────┐
              │ Bank A  │  │ Bank B  │   ...simulated participant banks,
              │ svc     │  │ svc     │   each with its own latency/
              └─────────┘  └─────────┘   failure profile
```

Credora should never know, at the code level, that it's talking to a simulator rather than a real rail. It depends on a `PaymentRail` interface; the simulator is one implementation, a future Monnify/NIBSS client is another:

```go
type PaymentRail interface {
    NameEnquiry(ctx context.Context, bankCode, accountNumber string) (*BeneficiaryDetails, error)
    InitiateTransfer(ctx context.Context, req TransferRequest) (*TransferResult, error)
    GetTransferStatus(ctx context.Context, reference string) (*TransferStatus, error)
}
```

```
              PaymentRail
                  │
        ┌─────────┴──────────┐
        ▼                    ▼
 NIBSSSimulatorClient   MonnifyClient
       (dev/test)          (prod)
```

## How Credora integrates

End-to-end flow for a ₦10,000 outbound transfer from a Credora customer to an account at a simulated Bank A:

1. Credora validates the request and calls `POST /name-enquiry` against the simulator to confirm the destination account and get the account name for the user to confirm.
2. Credora debits the sender inside a DB transaction and writes a `TRANSFER_INITIATED` outbox event.
3. A background worker (Asynq) picks up the event and calls `POST /transfers` against the simulator with an idempotent `requestId`.
4. The simulator's Transfer Router resolves the destination bank from the bank code and forwards the request to the matching simulated bank service, applying that bank's configured latency/failure profile.
5. The simulator responds synchronously with `PROCESSING` (or, depending on configuration, an immediate terminal state), and later pushes the terminal state to Credora's webhook endpoint.
6. Credora's webhook handler verifies the signature, checks idempotency on the `reference`, and updates the ledger and transfer status exactly once — the same pattern Credora already uses for inbound Monnify webhooks.
7. If Credora never receives a callback (simulated network partition, dropped response), it falls back to polling `GET /transfers/{reference}` and reconciling.

## Domain model

```
Bank
 ├── id
 ├── code            // CBN-style institution/bank code, e.g. "000013"
 ├── name
 ├── active
 └── simulationProfile (latencyMs, failureRate, availability)

BankAccount
 ├── id
 ├── bankCode
 ├── accountNumber
 ├── accountName
 └── balance

Transfer
 ├── id
 ├── requestId          // idempotency key, supplied by Credora
 ├── reference           // simulator-generated, returned to Credora
 ├── sessionId            // NIP-style session identifier
 ├── senderBankCode
 ├── senderAccount
 ├── destinationBankCode
 ├── destinationAccount
 ├── amount
 ├── currency
 ├── narration
 ├── status              // INITIATED → PROCESSING → SUCCESS | FAILED | TIMEOUT | UNKNOWN
 ├── responseCode
 ├── createdAt / updatedAt

CallbackAttempt
 ├── transferId
 ├── attemptNumber
 ├── deliveredAt
 └── httpStatus
```

## API reference

### `GET /banks`

Bank directory — Credora needs this to know where to route a transfer.

```json
[
  { "code": "000013", "name": "Bank A", "active": true },
  { "code": "000014", "name": "Bank B", "active": true }
]
```

### `POST /name-enquiry`

Validate a beneficiary account before money moves. Mirrors NIP's real Name Enquiry service, whose job is to let the source institution confirm the beneficiary's details from the account number alone.

Request:
```json
{ "bankCode": "000013", "accountNumber": "1234567890" }
```

Response:
```json
{
  "responseCode": "00",
  "accountNumber": "1234567890",
  "accountName": "JOHN DOE",
  "bankCode": "000013",
  "bankName": "Bank A",
  "sessionId": "000013260821103000001234567890"
}
```

### `POST /transfers`

The core operation. Acts as a switch: resolves `destinationBankCode`, routes to the matching simulated bank, and returns either a terminal result or a `PROCESSING` state that resolves later via callback.

Request:
```json
{
  "requestId": "credora-8f73c1e2-...",
  "senderBankCode": "999999",
  "senderAccount": "1000000001",
  "destinationBankCode": "000013",
  "destinationAccount": "1234567890",
  "amount": 5000000,
  "currency": "NGN",
  "narration": "Payment for invoice #4521"
}
```

> Amount is in kobo (integer), matching the way Credora's ledger already avoids floating point for money.

Response (immediate):
```json
{
  "reference": "NIBSS-20260821-000928381",
  "sessionId": "000013260821103000001234567890",
  "status": "PROCESSING",
  "responseCode": "00"
}
```

### `GET /transfers/{reference}`

Transaction Status Query — the mechanism Credora falls back to when it can't be sure a transfer succeeded (timeout, missed callback, or after a restart).

```json
{
  "reference": "NIBSS-20260821-000928381",
  "status": "SUCCESS",
  "responseCode": "00",
  "amount": 5000000,
  "destinationAccountName": "JOHN DOE"
}
```

### `POST` (outbound, simulator → Credora) `/webhooks/nibss/transfer`

The callback dispatcher. Signed with HMAC-SHA256 over the raw body using a shared secret, mirroring how Credora already verifies inbound Monnify webhooks.

```json
{
  "reference": "NIBSS-20260821-000928381",
  "status": "SUCCESS",
  "amount": 5000000,
  "timestamp": "2026-08-21T10:30:00Z"
}
```

### Simulation control plane (`/simulation/...`, non-production surface)

```
POST /simulation/banks/{bankCode}/profile
{
  "availability": 0.95,
  "averageLatencyMs": 2000,
  "failureRate": 0.10
}

POST /simulation/banks/{bankCode}/chaos/drop-callback   // next N callbacks are silently dropped
POST /simulation/banks/{bankCode}/chaos/duplicate-callback
POST /simulation/reset
```

This is what turns the project from "a fake API" into an actual chaos-testable rail: you can tell Bank B "fail 30% of requests, average 2s latency" and watch how Credora's retry, timeout and reconciliation logic actually behaves under it.

## Response codes

NIP, like most interbank rails, returns ISO 8583–style two-character response codes rather than free-text errors. The simulator reuses this convention so Credora's error-handling logic is built against the same shape of response it would see in production:

| Code | Meaning | Simulated trigger |
|---|---|---|
| `00` | Approved / successful | Happy path |
| `03` | Invalid sender/receiving institution | Unknown bank code |
| `07` | Invalid account | Name enquiry against a non-existent account |
| `12` | Invalid transaction | Malformed request |
| `13` | Invalid amount | Zero/negative amount |
| `14` | Invalid account number | Fails checksum/format |
| `25` | Unable to locate record | Status query on unknown reference |
| `51` | Insufficient funds | Simulated bank account below balance |
| `57` | Transaction not permitted | Account flagged/restricted in simulation |
| `61` | Exceeds transfer limit | Amount over the configured per-transaction ceiling |
| `91` | Issuer/switch inoperative, timeout | Destination bank's simulated `availability` roll fails |
| `94` | Duplicate transaction | Same `requestId` replayed |
| `96` | System malfunction | Random chaos-injected failure |

## Session ID / reference format

Real NIP transactions carry a **Session ID**, generated by the source institution, that uniquely identifies the transaction across the network — typically built from the institution code, transaction date/time, and a sequence number. The simulator follows the same shape so Credora's logging/reconciliation code is exercised against a realistic identifier rather than a random UUID:

```
sessionId = {institutionCode:6}{YYMMDD:6}{HHmmss:6}{sequence:15}
```

The simulator additionally issues its own `reference` (`NIBSS-{yyyyMMdd}-{sequence}`) that Credora stores alongside its own `requestId` — the standard three-identifier pattern (client idempotency key, network session ID, provider reference) that real payment-rail integrations end up needing.

## Failure & latency simulation

Each simulated bank carries a configurable profile:

```yaml
banks:
  "000013": { name: "Bank A", availability: 0.99, avgLatencyMs: 300,  failureRate: 0.01 }
  "000014": { name: "Bank B", availability: 0.95, avgLatencyMs: 2000, failureRate: 0.10 }
  "000015": { name: "Bank C", availability: 0.70, avgLatencyMs: 5000, failureRate: 0.30 }
```

This is what produces the failure modes worth building the rest of the system for:

- **Clean timeout** — Credora gives up waiting; the simulator never got the request.
- **Silent success** — the simulated bank actually succeeded, but the response/callback is dropped. Credora sees a timeout; the money moved. Only a status query or reconciliation catches this.
- **Retried duplicate** — Credora retries after a timeout; the simulator must recognize the repeated `requestId` and return the original result instead of double-crediting.
- **Duplicate callback** — the dispatcher redelivers the same callback twice; Credora's webhook handler must be idempotent on `reference`.
- **Crash mid-flight** — Credora restarts between sending the transfer and receiving the callback; on restart it must reconcile any transfer left in `PROCESSING`.

## Security model

The real NIBSS network runs over dedicated VPN tunnels or leased lines between NIBSS and each certified institution, layers in mandatory MFA for portal access, requires certified hardware for biometric flows, and revokes network access outright if a participant's endpoint is found non-compliant. A local simulator obviously can't (and shouldn't try to) reproduce private leased-line infrastructure, but it mirrors the parts that actually matter for the integration code:

- **Mutual authentication** — API key + shared secret per calling institution (`senderBankCode` maps to a registered credential), analogous to the certified-participant model.
- **Request signing** — HMAC-SHA256 over the request body for `/transfers` and the outbound webhook, so Credora and the simulator both verify integrity the same way Credora already verifies Monnify webhooks.
- **Replay protection** — `requestId` + timestamp window, rejecting requests outside a configurable tolerance.
- **TLS everywhere**, even in local docker-compose, via a self-signed cert — so certificate handling isn't something Credora only learns about at first real integration.

## Compliance mirroring (non-binding)

This project does **not** implement real BVN/NDPA compliance — there is no live customer PII flowing through it, and none of this is legal advice. It's included only so the API surface *shapes* Credora's code the way the real constraints would, in case a later phase adds identity verification:

- Real BVN validation flows require an explicit, documented consent step (NIBSS's Retrieval Token mechanism) before any lookup is permitted — modeled here, if added, as a mandatory `consentToken` field rather than a free lookup.
- The **zero-storage principle** — real institutions may store only verification *result flags*, never raw BVN — is worth keeping in mind if this simulator is ever extended toward identity verification: store `verified: true/false`, not the underlying identifier.
- None of this simulator's data should be treated as sensitive; it exists purely to test integration behavior.

## Tech stack

| Layer | Technology |
|---|---|
| Language | Java 21 |
| Framework | Spring Boot 3.x (Web, Validation) |
| Database | PostgreSQL |
| Migrations | Flyway |
| Messaging (V2+) | Spring Events → Kafka or RabbitMQ for callback dispatch/retry |
| Auth | API key + HMAC request signing |
| Docs | springdoc-openapi (Swagger UI) |
| Tests | JUnit 5, Testcontainers (Postgres), WireMock (simulated bank services in isolation) |
| Build | Gradle or Maven |

## Project structure

```
nibss-simulator/
├── src/main/java/com/example/nibss/
│   ├── controller/         # BankController, NameEnquiryController,
│   │                       # TransferController, StatusController, SimulationController
│   ├── service/            # TransferRoutingService, CallbackDispatchService,
│   │                       # NameEnquiryService, SimulationProfileService
│   ├── domain/              # Bank, BankAccount, Transfer, CallbackAttempt
│   ├── repository/          # Spring Data JPA repositories
│   ├── client/               # per-simulated-bank HTTP clients
│   ├── security/             # HMAC signing/verification, API key filter
│   ├── config/                # bank profile config, async/retry config
│   └── exception/              # domain exceptions → response-code mapping
├── src/main/resources/
│   ├── db/migration/           # Flyway SQL
│   └── application.yml
├── simulated-bank-a/            # separate Spring Boot service
├── simulated-bank-b/            # separate Spring Boot service
└── docker-compose.yml            # nibss-simulator + bank services + postgres
```

Bank A and Bank B are deliberately separate services rather than in-process stubs — that's what gives the project real cross-service communication to practice, on top of the Go ↔ Java boundary between Credora and the simulator.

## Build order (V1 → V3)

**V1 — core switch**
`GET /banks`, `POST /name-enquiry`, `POST /transfers`, `GET /transfers/{reference}`; synchronous responses only; Bank, BankAccount, Transfer, Transaction domain models.

**V2 — realism**
Asynchronous callbacks/webhooks, `PROCESSING` as a real intermediate state, retries, timeout handling, duplicate-request and duplicate-callback handling, reconciliation, an outbox pattern on the simulator side, dead-letter handling for undeliverable callbacks, circuit breakers on the bank-service clients.

**V3 — chaos control plane**
The `/simulation/...` endpoints for live-adjustable failure rate, latency, and targeted chaos (drop callback, duplicate callback, force timeout) per simulated bank, so specific failure scenarios can be triggered on demand instead of waiting for them to occur randomly.

## Test scenarios

These are the scenarios the simulator should be able to produce on demand, and that Credora's integration should be tested against:

1. `Credora → NIBSS → Bank A → SUCCESS` (happy path)
2. `Credora → NIBSS → Bank A → TIMEOUT` (no response at all)
3. `Credora → NIBSS → Bank A → SUCCESS`, then the callback is lost — Credora only learns the true state via status query/reconciliation
4. Credora retries after a perceived timeout → the simulator must detect the duplicate `requestId` and return the original result, not process it twice
5. The simulator dispatches the same callback twice → Credora's webhook handler must be idempotent
6. Simulator processes a transfer, then Credora crashes before receiving the callback; on restart Credora reconciles anything stuck in `PROCESSING`

## Running locally

**Requirements:** Java 21, Docker (for Postgres + the simulated bank services), Gradle

```bash
# start Postgres + simulated Bank A / Bank B
docker compose up -d postgres bank-a bank-b

# run migrations
./gradlew flywayMigrate

# start the simulator
./gradlew bootRun

# API docs
# http://localhost:8081/swagger-ui.html
```

Configure Credora's `PaymentRail` implementation to point at `http://localhost:8081` and register a webhook URL back to Credora's local server for callback delivery.

---

*This is a development/learning simulator. It is not affiliated with, endorsed by, or a replacement for NIBSS Plc., and must never be pointed at real customer data or presented as a certified NIP integration.*