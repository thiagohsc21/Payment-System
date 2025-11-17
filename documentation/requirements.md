# System Requirements

This document outlines the functional and non-functional requirements for the Payment Orchestrator Platform. This system prioritizes **Data Consistency**, **High Availability**, and **Scalability**.

## Functional Requirements (FR)

### FR01: Payment Ingestion and Validation

The system must provide a secure entry point to receive payment requests via HTTP (REST).

The **Payment API** is the component responsible for this function.
* **Validation:** Upon receiving a JSON payload, the API must validate mandatory fields (Amount, Currency, Card Token, Merchant ID).
* **Idempotency Check:** To prevent double-spending, the system must verify the `idempotency_key`. If a request with the same key has already been processed, the system must return the stored result immediately without re-processing.
* **Output:** If valid and unique, the system generates a unique `Transaction ID`.

### FR02: Transaction Persistence (ACID)

Before any processing occurs, the system must persist the intention to pay. This ensures no data is lost in case of a server crash.

* The **Payment API** writes to the **PostgreSQL** database.
* **Atomicity:** The insertion of the `Transaction` (Status: PENDING) and the `Idempotency Key` must happen in a single database transaction.

### FR03: Asynchronous Handoff

The system must decouple the reception of the request from its processing to ensure high throughput.

* Once the transaction is safely stored in the database, the **Payment API** must publish a `PaymentCreatedEvent` to the **Message Broker (Kafka)**.
* This allows the API to respond immediately to the client with a `201 Created` (Status: PENDING), without waiting for the external gateway.

### FR04: Payment Processing (Gateway Integration)

The system must consume pending transactions and execute the actual charge against an external provider.

* The **Payment Worker** consumes messages from the **Kafka** topic.
* It translates the internal event into a request for the external **Payment Gateway** (e.g., Stripe Mock).
* **Resilience:** If the Gateway is down, the Worker must implement a retry mechanism (Exponential Backoff) or a Circuit Breaker.

### FR05: State Reconciliation (Eventual Consistency)

The system must update the internal state of the transaction based on the external Gateway's response.

* Upon receiving a response (Approved/Denied) from the Gateway, the **Payment Worker** updates the `Transaction` status in **PostgreSQL**.
* It then publishes a final event (e.g., `PaymentApprovedEvent`) back to Kafka to trigger side effects (notifications, shipping, etc.).

### FR06: Audit Logging

The system must maintain a raw, immutable log of every step of the process for auditing and debugging purposes.

* The system must store the full JSON request and response payloads from the external Gateway.
* **Storage:** These logs are unstructured and voluminous, so they must be stored in **MongoDB** (NoSQL), separate from the transactional data.

### FR07: Status Query

Clients must be able to check the current status of a transaction since processing is asynchronous.

* The **Payment API** must provide a `GET /payments/{id}` endpoint.
* It retrieves the current status (PENDING, APPROVED, DENIED) directly from **PostgreSQL**.

---

## Non-Functional Requirements (NFR)

### NFR01: Data Consistency
* The system must ensure **Strong Consistency** for the financial state stored in PostgreSQL.
* The system accepts **Eventual Consistency** between the PostgreSQL state and the downstream consumers (Notifications/Logs).

### NFR02: Horizontal Scalability
* The **Payment API** must be stateless, allowing it to scale horizontally (adding more instances) behind a Load Balancer to handle increased traffic.

### NFR03: Reliability & Fault Tolerance
* **No Data Loss:** Once a generic HTTP 200/201 is returned to the client, the transaction must be persisted.
* **Dead Letter Queue (DLQ):** Events that fail processing repeatedly (poison pills) must be moved to a DLQ for manual inspection, preventing the consumers from getting stuck.

### NFR04: Idempotency
* The system must guarantee that processing the same Payment Request multiple times results in the same outcome and only one financial charge.

### NFR05: Observability
* **Distributed Tracing:** Every request must be tagged with a `TraceID` that propagates from the API to Kafka to the Worker, allowing full visibility of the flow in logs.