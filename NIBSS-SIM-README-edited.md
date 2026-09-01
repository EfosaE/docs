# NIBSS Simulator (Java / Spring Boot)

A production-shaped simulation of the **NIBSS Instant Payment (NIP)** interbank switch, built so that [`credora-backend`](https://github.com/EfosaE/credora-backend) can implement and exercise an **external interbank transfer** flow — something no fintech can legally do against the real NIBSS network without a CBN operating license or a licensed aggregator relationship (Monnify, Paystack, Flutterwave, Mono, etc.).

This service does not talk to NIBSS. It **is** a stand-in NIBSS: a separate Java system that owns bank routing, name enquiry, transfer processing, status tracking and callbacks — the same responsibilities the real switch owns — so that Credora is forced to solve the same class of distributed-systems problems it would face integrating with a real payment rail.

> **Naming disclaimer:** every institution, bank code, account number and account name in this README — including "Ridgeway Bank" and "Solace Microfinance Bank" below — is fictional. None of them correspond to real CBN-licensed institutions, and none of the codes are real NIBSS institution codes. See [Simulated bank codes](#simulated-bank-codes) for why that matters.

---

## Table of contents

- [Why this exists](#why-this-exists)
- [How this maps to the real NIBSS](#how-this-maps-to-the-real-nibss)
- [System architecture](#system-architecture)
- [What `simulated-bank-a` and `simulated-bank-b` actually are](#what-simulated-bank-a-and-simulated-bank-b-actually-are)
- [How Credora integrates](#how-credora-integrates)
- [Domain model](#domain-model)
- [Domain object examples](#domain-object-examples)
- [Simulated bank codes](#simulated-bank-codes)
- [API reference](#api-reference)
- [End-to-end worked example](#end-to-end-worked-example)
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

NIBSS (Nigeria Inter-Bank Settlement System) operates **NIP (NIBSS Instant Payment)**, the real-time interbank funds transfer rail used across Nigerian banks, microfinance banks and licensed mobile money operators as direct participants. Per NIBSS's own service description, NIP exposes a small set of core operations that this simulator deliberately mirrors:

| Real NIP capability | Simulated here |
|---|---|
| Name Enquiry — validate a beneficiary account and return the account name before a transfer is initiated | ✅ `POST /name-enquiry` |
| Funds Transfer (Direct Credit) — move funds into a beneficiary account | ✅ `POST /transfers` |
| Transaction Status Query (TSQ) — query the status of a previously sent transaction | ✅ `GET /transfers/{reference}` |
| Balance Enquiry | Out of scope for V1 (Credora doesn't need it for outbound transfers) |
| Funds Transfer (Direct Debit) / standing orders | Out of scope |

NIP is a real-time account-number-based interbank transfer service. For this learning project, keep two ideas separate: **transaction processing** (the real-time transfer flow) and **inter-institution settlement/reconciliation** (the financial settlement obligations between participants). NIBSS-SIM models the transaction flow; it does not attempt to reproduce NIBSS's private settlement infrastructure. That real-time-availability-ahead-of-settlement property is exactly why reconciliation, idempotency, and status-query machinery matter — a bank can't just "wait for settlement" to know whether a transfer succeeded, and neither can Credora.

In the real world, access to NIP is not self-service. A financial institution or PSSP must go through NIBSS certification — a formal letter of intent, application and device certification, multi-week code audits and penetration testing, and typically a dedicated VPN/leased-line connection to NIBSS — before going live. Smaller fintechs almost always integrate indirectly through a licensed aggregator instead. That's the exact reason this project exists as a **simulator**: it lets Credora behave like it holds that access without requiring a license, a leased line, or a certification cycle.

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
              │   ("the switch")        │
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
              │Ridgeway │  │ Solace  │   ...simulated participant banks,
              │  Bank   │  │  MFB    │   each with its own latency/
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

## What `simulated-bank-a` and `simulated-bank-b` actually are

This is the piece that's easiest to gloss over, so it's worth spelling out.

In the real NIP network, **NIBSS is the switch, not a bank**. When a transfer targets an account at, say, GTBank, NIBSS doesn't hold that account or know its balance — it forwards the request over the network to GTBank's own core banking system, which owns the account, does the actual credit/debit, and sends a response back through NIBSS. NIBSS's job is purely routing, messaging, and settlement bookkeeping between institutions; each *participant institution* runs its own independent system behind its own connection to the switch.

`simulated-bank-a` and `simulated-bank-b` exist to reproduce that shape, not to be an implementation detail of the simulator. They are **two separate, independently-deployed Spring Boot services** — not modules inside `nibss-simulator` — standing in for two fictional participant institutions:

| Folder | Represents | Role |
|---|---|---|
| `nibss-simulator/` | NIBSS itself | The switch: bank directory, routing, name enquiry orchestration, transfer lifecycle, callbacks. **Owns no customer accounts.** |
| `simulated-bank-a/` | "Ridgeway Bank" (fictional commercial bank) | A participant institution's core banking stand-in: owns its own `BankAccount` records and balances, decides whether a name enquiry or debit/credit succeeds, and applies its own configured latency/failure profile. |
| `simulated-bank-b/` | "Solace Microfinance Bank" (fictional MFB) | Same role as Ridgeway, but modeled as a smaller/slower participant — MFBs and other non-bank NIP participants are typically less reliable and higher-latency than tier-1 commercial banks, which is worth exercising separately. |

Concretely, this means:

- The **Transfer Router** inside `nibss-simulator` doesn't have direct database access to accounts at Ridgeway or Solace. It makes an HTTP call (`client/RidgewayBankClient`, `client/SolaceMfbClient`) to that bank's own service, the same way the real switch forwards a message over the network rather than reading another bank's database.
- Each bank service has its own Postgres schema (or its own database entirely) with its own `accounts` table, seeded independently — `nibss-simulator`'s own DB stores the **institution directory and switch transaction state** (`Bank`, `Transfer`, `CallbackAttempt`). It does not own participant customer-account balances.
- Each bank service can be started, stopped, or reconfigured independently (`docker compose stop bank-b` to simulate an entire participant institution going offline is a realistic failure mode NIBSS itself has to handle).
- Adding a third participant later (e.g. a fictional mobile money operator) means adding a third sibling service and a `Bank` directory row pointing at it — `nibss-simulator` itself shouldn't need code changes beyond configuration.

This is also why the project structure calls them out as siblings of `nibss-simulator/` rather than packages under `src/main/java/com/example/nibss/` — they are deliberately a Go-service-caller ↔ Java-switch ↔ Java-participant-bank chain of three independently-owned systems, which is what actually forces you to build idempotent, retry-safe, partial-failure-tolerant integration code instead of a single well-behaved monolith.

## How Credora integrates

End-to-end flow for a ₦10,000 outbound transfer from a Credora customer to an account at Ridgeway Bank:

1. Credora validates the request and calls `POST /name-enquiry` against the simulator to confirm the destination account and get the account name for the user to confirm.
2. Credora debits the sender inside a DB transaction and writes a `TRANSFER_INITIATED` outbox event.
3. A background worker (Asynq) picks up the event and calls `POST /transfers` against the simulator with an idempotent `requestId`.
4. The simulator's Transfer Router resolves the destination bank from the bank code (`990001` → Ridgeway Bank) and forwards the request to Ridgeway's own service, applying Ridgeway's configured latency/failure profile.
5. The simulator responds synchronously with `PROCESSING` (or, depending on configuration, an immediate terminal state), and later pushes the terminal state to Credora's webhook endpoint.
6. Credora's webhook handler verifies the signature, checks idempotency on the `reference`, and updates the ledger and transfer status exactly once — the same pattern Credora already uses for inbound Monnify webhooks.
7. If Credora never receives a callback (simulated network partition, dropped response), it falls back to polling `GET /transfers/{reference}` and reconciling.

## Domain model

```
Bank
 ├── id
 ├── code            // simulator-issued institution code, e.g. "990001"
 ├── name
 ├── active
 └── simulationProfile (latencyMs, failureRate, availability)

# BankAccount is NOT owned by NIBSS-SIM.
# It belongs to the participant-bank service (e.g. Ridgeway).
# NIBSS-SIM only knows the institution and routes messages to it.

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

## Domain object examples

Concrete, realistic values for each domain object — useful as fixtures for seed data, Postgres inserts, or JUnit test builders.

### `Bank`

```json
{
  "id": "b7e6a1f0-2c34-4e11-9f2a-1a2b3c4d5e6f",
  "code": "990001",
  "name": "Ridgeway Bank",
  "active": true,
  "simulationProfile": {
    "availability": 0.99,
    "averageLatencyMs": 300,
    "failureRate": 0.01
  }
}
```

```json
{
  "id": "c1d2e3f4-5678-49ab-8cde-f0123456789a",
  "code": "990002",
  "name": "Solace Microfinance Bank",
  "active": true,
  "simulationProfile": {
    "availability": 0.90,
    "averageLatencyMs": 2200,
    "failureRate": 0.15
  }
}
```

An inactive bank, used to exercise response code `03` (invalid receiving institution):

```json
{
  "id": "9f8e7d6c-5b4a-4321-9876-abcdef012345",
  "code": "990099",
  "name": "Northgate Digital Bank (decommissioned)",
  "active": false,
  "simulationProfile": {
    "availability": 0.0,
    "averageLatencyMs": 0,
    "failureRate": 1.0
  }
}
```

### `BankAccount` (owned by `simulated-bank-a` / Ridgeway Bank's own DB, not by `nibss-simulator`)

```json
{
  "id": "1a2b3c4d-5e6f-4708-9012-3456789abcde",
  "bankCode": "990001",
  "accountNumber": "1234567890",
  "accountName": "JOHN DOE",
  "balance": 850000000
}
```

> `balance` is in kobo, same convention as `Transfer.amount` — ₦8,500,000.00.

### `Transfer` — full lifecycle of one record

```json
{
  "id": "6d5c4b3a-2918-4776-a5f4-e3d2c1b0a9f8",
  "requestId": "credora-8f73c1e2-4a5b-4c6d-9e0f-1a2b3c4d5e6f",
  "reference": "NIBSS-20260821-000928381",
  "sessionId": "990001260821103000001234567890",
  "senderBankCode": "999999",
  "senderAccount": "1000000001",
  "destinationBankCode": "990001",
  "destinationAccount": "1234567890",
  "amount": 5000000,
  "currency": "NGN",
  "narration": "Payment for invoice #4521",
  "status": "SUCCESS",
  "responseCode": "00",
  "createdAt": "2026-08-21T10:29:58Z",
  "updatedAt": "2026-08-21T10:30:01Z"
}
```

> `senderBankCode` `999999` is Credora's own simulator-registered institution code — the code the simulator was given when Credora "onboarded" as a participant, distinct from the destination bank codes above.

### `CallbackAttempt`

```json
{
  "transferId": "6d5c4b3a-2918-4776-a5f4-e3d2c1b0a9f8",
  "attemptNumber": 1,
  "deliveredAt": null,
  "httpStatus": 504
}
```
```json
{
  "transferId": "6d5c4b3a-2918-4776-a5f4-e3d2c1b0a9f8",
  "attemptNumber": 2,
  "deliveredAt": "2026-08-21T10:30:07Z",
  "httpStatus": 200
}
```

Two rows for the same `transferId` model exactly the "clean timeout then successful retry" callback-delivery scenario described under [Failure & latency simulation](#failure--latency-simulation).

## Simulated bank codes

Real NIBSS institution/bank codes are 3-digit CBN-assigned identifiers (GTBank is `058`, for example) used across NIP, USSD (`*737#`-style routing) and banking apps. This simulator deliberately does **not** reuse real codes for its fictional banks — doing so would make seeded test data look like it belongs to an actual institution, which is exactly the kind of confusion the project's own disclaimer exists to avoid.

Instead, simulated institutions use a `99xxxx` range that cannot collide with any real CBN-assigned code:

| Code | Institution | Type |
|---|---|---|
| `990001` | Ridgeway Bank | Fictional commercial bank |
| `990002` | Solace Microfinance Bank | Fictional MFB |
| `990099` | Northgate Digital Bank | Fictional, inactive — for negative-path tests |
| `999999` | Credora (sending institution) | Credora's own simulator-registered code |

## API message format

The simulator supports **JSON for convenient application integration** and a simple XML representation for learning the message-oriented style used in financial systems.

These API XML payloads use the simulator-owned namespace:

```xml
xmlns="urn:nibss-sim:nip:api:v1"
```

This is intentionally **not** presented as an official NIBSS namespace. The separate ISO 20022 learning messages use message-specific simulator namespaces such as:

```text
urn:nibss-sim:iso20022:pain.001.001.13
urn:nibss-sim:iso20022:pacs.008.001.08
urn:nibss-sim:iso20022:pacs.002.001.10
urn:nibss-sim:iso20022:camt.053.001.08
```

The XML is therefore "NIBSS-SIM shaped": it uses Nigerian interbank terminology and identifiers while remaining explicitly a learning/simulation contract.

## API reference

### `GET /banks`

Bank/institution directory — Credora needs this to know where to route a transfer.

```json
[
  { "code": "990001", "name": "Ridgeway Bank", "active": true },
  { "code": "990002", "name": "Solace Microfinance Bank", "active": true }
]
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<BankDirectory xmlns="urn:nibss-sim:nip:api:v1">
  <Institution>
    <InstnCode>990001</InstnCode>
    <Nm>Ridgeway Bank</Nm>
    <Active>true</Active>
  </Institution>
  <Institution>
    <InstnCode>990002</InstnCode>
    <Nm>Solace Microfinance Bank</Nm>
    <Active>true</Active>
  </Institution>
</BankDirectory>
```

### `POST /name-enquiry`

Validate a beneficiary account before money moves. Mirrors NIP's real Name Enquiry service, whose job is to let the source institution confirm the beneficiary's details from the account number alone.

Request:
```json
{ "bankCode": "990001", "accountNumber": "1234567890" }
```

Response:
```json
{
  "responseCode": "00",
  "accountNumber": "1234567890",
  "accountName": "JOHN DOE",
  "bankCode": "990001",
  "bankName": "Ridgeway Bank",
  "sessionId": "990001260821103000001234567890"
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<NameEnquiryResponse xmlns="urn:nibss-sim:nip:api:v1">
  <RspCd>00</RspCd>
  <AcctNo>1234567890</AcctNo>
  <AcctNm>JOHN DOE</AcctNm>
  <InstnCode>990001</InstnCode>
  <InstnNm>Ridgeway Bank</InstnNm>
  <SessionId>990001260821103000001234567890</SessionId>
</NameEnquiryResponse>
```

Name enquiry against an account that doesn't exist at Ridgeway (`07` — invalid account):

Request:
```json
{ "bankCode": "990001", "accountNumber": "0000000000" }
```

Response:
```json
{
  "responseCode": "07",
  "accountNumber": "0000000000",
  "accountName": null,
  "bankCode": "990001",
  "bankName": "Ridgeway Bank",
  "sessionId": "990001260821103000000000000000"
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<NameEnquiryResponse xmlns="urn:nibss-sim:nip:api:v1">
  <RspCd>07</RspCd>
  <AcctNo>0000000000</AcctNo>
  <AcctNm/>
  <InstnCode>990001</InstnCode>
  <InstnNm>Ridgeway Bank</InstnNm>
  <SessionId>990001260821103000000000000000</SessionId>
</NameEnquiryResponse>
```

### `POST /transfers`

The core operation. Acts as a switch: resolves `destinationBankCode`, routes to the matching simulated bank service, and returns either a terminal result or a `PROCESSING` state that resolves later via callback.

Request:
```json
{
  "requestId": "credora-8f73c1e2-4a5b-4c6d-9e0f-1a2b3c4d5e6f",
  "senderBankCode": "999999",
  "senderAccount": "1000000001",
  "destinationBankCode": "990001",
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
  "sessionId": "990001260821103000001234567890",
  "status": "PROCESSING",
  "responseCode": "00"
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferResponse xmlns="urn:nibss-sim:nip:api:v1">
  <RspCd>00</RspCd>
  <RspMsg>Transfer accepted for processing</RspMsg>
  <Reference>NIBSS-20260821-000928381</Reference>
  <SessionId>990001260821103000001234567890</SessionId>
  <TxSts>PROCESSING</TxSts>
</TransferResponse>
```

A same-`requestId` retry against a transfer that already succeeded (response code `94` — duplicate transaction, original result returned rather than double-processed):

Response:
```json
{
  "reference": "NIBSS-20260821-000928381",
  "sessionId": "990001260821103000001234567890",
  "status": "SUCCESS",
  "responseCode": "94"
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferResponse xmlns="urn:nibss-sim:nip:api:v1">
  <RspCd>94</RspCd>
  <RspMsg>Duplicate transaction</RspMsg>
  <Reference>NIBSS-20260821-000928381</Reference>
  <SessionId>990001260821103000001234567890</SessionId>
  <TxSts>SUCCESS</TxSts>
</TransferResponse>
```

A transfer routed to Solace Microfinance Bank that fails on insufficient funds:

Request:
```json
{
  "requestId": "credora-3b2a1c9d-8e7f-4a6b-9c5d-2e1f0a9b8c7d",
  "senderBankCode": "999999",
  "senderAccount": "1000000001",
  "destinationBankCode": "990002",
  "destinationAccount": "5551239876",
  "amount": 12000000000,
  "currency": "NGN",
  "narration": "Rent payment"
}
```

Response:
```json
{
  "reference": "NIBSS-20260821-000928402",
  "sessionId": "990002260821104512000005551239876",
  "status": "FAILED",
  "responseCode": "51"
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferResponse xmlns="urn:nibss-sim:nip:api:v1">
  <RspCd>51</RspCd>
  <RspMsg>Insufficient funds</RspMsg>
  <Reference>NIBSS-20260821-000928402</Reference>
  <SessionId>990002260821104512000005551239876</SessionId>
  <TxSts>FAILED</TxSts>
</TransferResponse>
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

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferStatusResponse xmlns="urn:nibss-sim:nip:api:v1">
  <Reference>NIBSS-20260821-000928381</Reference>
  <TxSts>SUCCESS</TxSts>
  <RspCd>00</RspCd>
  <Amt Ccy="NGN">50000.00</Amt>
  <CdtrAcctNm>JOHN DOE</CdtrAcctNm>
</TransferStatusResponse>
```

Status query on a reference the simulator has no record of (`25` — unable to locate record):

```json
{
  "reference": "NIBSS-20260821-999999999",
  "status": "UNKNOWN",
  "responseCode": "25",
  "amount": null,
  "destinationAccountName": null
}
```

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferStatusResponse xmlns="urn:nibss-sim:nip:api:v1">
  <Reference>NIBSS-20260821-999999999</Reference>
  <TxSts>UNKNOWN</TxSts>
  <RspCd>25</RspCd>
  <Amt Ccy="NGN"/>
  <CdtrAcctNm/>
</TransferStatusResponse>
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

XML equivalent:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<TransferNotification xmlns="urn:nibss-sim:nip:api:v1">
  <Reference>NIBSS-20260821-000928381</Reference>
  <TxSts>SUCCESS</TxSts>
  <Amt Ccy="NGN">50000.00</Amt>
  <CreDtTm>2026-08-21T10:30:00Z</CreDtTm>
</TransferNotification>
```

Headers sent alongside the body:

```
X-NIBSS-Signature: 8f1a2e...   (hex-encoded HMAC-SHA256 of the raw body)
X-NIBSS-Timestamp: 2026-08-21T10:30:00Z
Content-Type: application/xml  # for the XML representation
# application/json may be used for the JSON representation
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

This is what turns the project from "a fake API" into an actual chaos-testable rail: you can tell Solace Microfinance Bank "fail 30% of requests, average 2s latency" and watch how Credora's retry, timeout and reconciliation logic actually behaves under it.

## End-to-end worked example

Putting the pieces above together — a single ₦50,000 transfer from Credora's customer to John Doe's account at Ridgeway Bank, traced request by request:

1. **Credora → simulator**, `POST /name-enquiry` with `{"bankCode": "990001", "accountNumber": "1234567890"}` → simulator forwards to `simulated-bank-a`, gets back `JOHN DOE`, responds `responseCode: "00"`.
2. **Credora → simulator**, `POST /transfers` with `requestId: "credora-8f73c1e2-..."`, `destinationBankCode: "990001"`, `amount: 5000000` → simulator persists a `Transfer` row with `status: INITIATED`, generates `reference: "NIBSS-20260821-000928381"` and `sessionId`, responds `202`-style with `status: "PROCESSING"`.
3. **Simulator → `simulated-bank-a`** (internal, not visible to Credora): the Transfer Router calls Ridgeway's own transfer endpoint, waits out Ridgeway's configured `averageLatencyMs: 300`, gets back a debit/credit confirmation.
4. **Simulator updates its own `Transfer` row** to `status: SUCCESS`, `responseCode: "00"`.
5. **Simulator → Credora**, `POST /webhooks/nibss/transfer` with the signed callback body from above. Credora's webhook handler checks `reference` against its own store, finds it already applied → no-op (idempotent).
6. If step 5's HTTP call had instead timed out, Credora would eventually call **`GET /transfers/NIBSS-20260821-000928381`**, see `status: "SUCCESS"` from the simulator's persisted state, and reconcile without ever having received the callback.

## ISO 20022 learning layer

The four ISO 20022 messages are useful here, but they should not be treated as four endpoints that NIBSS-SIM must expose in V1.

For this simulator, use them as a **message vocabulary** around the transfer lifecycle:

| Message | Simple meaning | Where it fits in NIBSS-SIM |
|---|---|---|
| `pain.001` | Customer Credit Transfer Initiation | Originator/customer-facing instruction to a bank or payment gateway |
| `pacs.008` | FI-to-FI Customer Credit Transfer | Interbank payment message sent by the sending institution into the switch |
| `pacs.002` | FI-to-FI Payment Status Report | Status response/report referring to the original interbank payment |
| `camt.053` | Bank-to-Customer Statement | Account statement containing booked entries for reconciliation |

### Important simplification

Do **not** make the simulator pretend that `pain.001 → pacs.008 → pacs.002 → camt.053` is a mandatory one-to-one production NIBSS sequence.

For the learning project:

```text
Credora
   │
   │  REST / JSON or simulator XML
   ▼
NIBSS-SIM
   │
   │  pacs.008-shaped interbank message
   ▼
Ridgeway Bank
   │
   │  processing result
   ▼
NIBSS-SIM
   │
   ├── pacs.002-shaped status
   └── API callback / status response
```

`pain.001` is useful if you later model the **customer-to-bank/gateway instruction** boundary. `pacs.008` is the most useful ISO message to study for the **bank-to-switch interbank leg**. `pacs.002` is useful for payment status. `camt.053` is useful for later reconciliation/statement work.

The XSDs in the companion ISO reference are therefore **learning schemas**, not official NIBSS schemas. Keep the simulator namespace explicit so nobody mistakes them for certified production messages.

## Response codes

The simulator uses a small set of two-digit response codes inspired by common interbank/ISO 8583 response-code conventions rather than free-text errors. The simulator reuses this convention so Credora's error-handling logic is built against the same shape of response it would see in production:

| Code | Meaning | Simulated trigger |
|---|---|---|
| `00` | Approved / successful | Happy path |
| `03` | Invalid sender/receiving institution | Unknown or inactive bank code (e.g. `990099`) |
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

The simulator models a NIP-style **Session ID** so the transaction can be correlated across the sending institution, NIBSS-SIM and receiving institution. Its exact format is a simulator convention, not a claim about the production NIP format. The simulator follows the same shape so Credora's logging/reconciliation code is exercised against a realistic identifier rather than a random UUID:

```
sessionId = {institutionCode:6}{YYMMDD:6}{HHmmss:6}{sequence:15}

example: 990001 260821 103000 001234567890 → "990001260821103000001234567890"
          Ridgeway   21 Aug    10:30:00      sequence
          Bank       2026
```

The simulator additionally issues its own `reference` (`NIBSS-{yyyyMMdd}-{sequence}`). Credora can store all three identifiers: its `requestId`, the simulator's NIP-style `sessionId`, and the simulator `reference`. This gives you a useful correlation model for payment-rail integrations without claiming these exact formats are production NIBSS formats.

## Failure & latency simulation

Each simulated bank carries a configurable profile:

```yaml
banks:
  "990001": { name: "Ridgeway Bank",              availability: 0.99, avgLatencyMs: 300,  failureRate: 0.01 }
  "990002": { name: "Solace Microfinance Bank",    availability: 0.90, avgLatencyMs: 2200, failureRate: 0.15 }
  "990099": { name: "Northgate Digital Bank",      availability: 0.00, avgLatencyMs: 0,    failureRate: 1.00 }
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
├── src/main/java/com/nibsssim/
│   ├── NibssSimApplication.java
│   │
│   ├── controller/
│   │   ├── BankController.java
│   │   ├── NameEnquiryController.java
│   │   ├── TransferController.java
│   │   ├── StatusController.java
│   │   └── SimulationController.java
│   │
│   ├── dto/
│   │   ├── nameenquiry/       # NameEnquiryRequest/Response
│   │   ├── transfer/          # TransferRequest/Response
│   │   └── status/            # TransferStatusResponse
│   │
│   ├── domain/
│   │   ├── Bank.java           # participant/institution directory
│   │   ├── Transfer.java       # NIP transaction lifecycle
│   │   └── CallbackAttempt.java
│   │
│   ├── repository/
│   │   ├── BankRepository.java
│   │   ├── TransferRepository.java
│   │   └── CallbackAttemptRepository.java
│   │
│   ├── service/
│   │   ├── BankDirectoryService.java
│   │   ├── NameEnquiryService.java
│   │   ├── TransferService.java
│   │   ├── TransferRoutingService.java
│   │   ├── CallbackDispatchService.java
│   │   └── SimulationProfileService.java
│   │
│   ├── client/
│   │   ├── ParticipantBankClient.java
│   │   ├── RidgewayBankClient.java
│   │   └── SolaceMfbClient.java
│   │
│   ├── mapper/                 # DTO ↔ domain ↔ XML mapping
│   ├── xml/                    # XML/XSD validation + JAXB mapping
│   ├── security/               # API key, HMAC, replay protection
│   ├── config/                 # HTTP clients, Jackson/JAXB, async/retry
│   └── exception/              # domain errors → response codes
│
├── src/main/resources/
│   ├── xsd/
│   │   ├── pain.001.simplified.xsd
│   │   ├── pacs.008.simplified.xsd
│   │   ├── pacs.002.simplified.xsd
│   │   └── camt.053.simplified.xsd
│   ├── db/migration/
│   └── application.yml
│
├── simulated-bank-a/           # separate participant service: Ridgeway
├── simulated-bank-b/           # separate participant service: Solace MFB
└── docker-compose.yml
```

`simulated-bank-a` and `simulated-bank-b` are siblings of `nibss-simulator/`, not packages inside it — see [What `simulated-bank-a` and `simulated-bank-b` actually are](#what-simulated-bank-a-and-simulated-bank-b-actually-are) for why that separation is deliberate. Each has its own minimal internal structure mirroring a tiny core-banking stand-in:

```
simulated-bank-a/
├── src/main/java/com/example/ridgeway/
│   ├── controller/         # AccountController (internal name-enquiry/debit-credit endpoints)
│   ├── domain/              # Account
│   └── repository/
├── src/main/resources/
│   ├── db/migration/
│   └── application.yml       # its own latency/failure profile, independent of nibss-simulator's copy
```

## Build order (V1 → V3)

**V1 — core switch**
`GET /banks`, `POST /name-enquiry`, `POST /transfers`, `GET /transfers/{reference}`; synchronous responses only; Bank, BankAccount, Transfer, Transaction domain models.

**V2 — realism**
Asynchronous callbacks/webhooks, `PROCESSING` as a real intermediate state, retries, timeout handling, duplicate-request and duplicate-callback handling, reconciliation, an outbox pattern on the simulator side, dead-letter handling for undeliverable callbacks, circuit breakers on the bank-service clients.

**V3 — chaos control plane**
The `/simulation/...` endpoints for live-adjustable failure rate, latency, and targeted chaos (drop callback, duplicate callback, force timeout) per simulated bank, so specific failure scenarios can be triggered on demand instead of waiting for them to occur randomly.

## Test scenarios

These are the scenarios the simulator should be able to produce on demand, and that Credora's integration should be tested against:

1. `Credora → NIBSS → Ridgeway Bank → SUCCESS` (happy path)
2. `Credora → NIBSS → Ridgeway Bank → TIMEOUT` (no response at all)
3. `Credora → NIBSS → Ridgeway Bank → SUCCESS`, then the callback is lost — Credora only learns the true state via status query/reconciliation
4. Credora retries after a perceived timeout → the simulator must detect the duplicate `requestId` and return the original result, not process it twice
5. The simulator dispatches the same callback twice → Credora's webhook handler must be idempotent
6. Simulator processes a transfer, then Credora crashes before receiving the callback; on restart Credora reconciles anything stuck in `PROCESSING`
7. Transfer to Solace Microfinance Bank fails with `51` (insufficient funds) — exercises the "expected business failure, not an infrastructure failure" path Credora must distinguish from timeouts

## Running locally

**Requirements:** Java 21, Docker (for Postgres + the simulated bank services), Gradle

```bash
# start Postgres + simulated Ridgeway Bank / Solace MFB
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