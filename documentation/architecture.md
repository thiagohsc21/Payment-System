## Architecture Components Description

<img width="1648" height="696" alt="image" src="https://github.com/user-attachments/assets/9732169f-f3e6-4995-9709-5cc90196452e" />

This documentation describes the responsibilities and interactions of each main component of the Payment Orchestrator Platform. The system is designed as an event-driven microservices architecture.

### 1. Load Balancer (LB) & API Gateway

* **Primary Responsibility:** To serve as the system's single entry point, managing traffic distribution and routing.
* **Inputs:** HTTP/HTTPS requests from external clients (Web/Mobile).
* **Processing:**
    1.  **SSL Termination:** Decrypts incoming traffic.
    2.  **Load Balancing:** Distributes requests among available instances of the `Payment Service` (Round-robin or Least Connections).
    3.  **Routing:** Directs traffic to the appropriate service endpoints based on the URL path.
* **Outputs:** Forwarded HTTP requests to the `Payment Service`.

### 2. Payment Service (API)

* **Primary Responsibility:** The synchronous interface of the system. It handles request validation, idempotency checks, and the initial persistence of the transaction state.
* **Inputs:** JSON payloads representing payment intentions.
* **Processing:**
    1.  **Validation:** Verifies payload structure, mandatory fields, and business rules (e.g., amount > 0).
    2.  **Idempotency Check:** Queries the `Transaction Database` to ensure the `idempotency_key` has not been processed yet.
    3.  **ACID Persistence:** Saves the transaction with status `PENDING` to the `Transaction Database` (PostgreSQL Master).
    4.  **Event Publication:** Upon successful persistence, publishes a `PaymentCreatedEvent` to the `Message Broker`.
* **Outputs:** A `PaymentCreatedEvent` to Kafka and an HTTP 201 Created response to the client.

### 3. Transaction Database (PostgreSQL)

* **Primary Responsibility:** To act as the Source of Truth for the financial state of all transactions. It guarantees strong consistency (ACID).
* **Inputs:** SQL INSERTs/UPDATEs from `Payment Service` and `Payment Process Worker`.
* **Structure:**
    * **Master Node:** Handles all WRITE operations to ensure consistency.
    * **Slave Nodes:** Replicates data from Master and handles READ operations (e.g., Status Query) to offload the Master.
* **Outputs:** Result sets for queries and confirmation of writes.

### 4. Message Broker (Apache Kafka)

* **Primary Responsibility:** To decouple the synchronous API from the asynchronous workers, ensuring high throughput and acting as a buffer during traffic spikes.
* **Inputs:** Events from the `Payment Service` and `Payment Process Worker`.
* **Processing:** Durable storage of messages in partitioned Topics.
* **Outputs:** Delivers messages to consumer groups (`Payment Process Worker`, `Notification Worker`).

### 5. Payment Process Worker

* **Primary Responsibility:** The core asynchronous engine. It orchestrates the integration with external gateways and reconciles the final state.
* **Inputs:** `PaymentCreatedEvent` messages from Kafka.
* **Processing:**
    1.  **Consumption:** Reads the pending payment event.
    2.  **Gateway Integration:** Formats and sends a request to the `External Payment Gateway` (Stripe).
    3.  **State Update:** Based on the gateway response, updates the `Transaction Database` status to `APPROVED` or `DENIED`.
    4.  **Result Publication:** Publishes a `PaymentResultEvent` back to Kafka for downstream consumers.
    5.  **Auditing:** Asynchronously pushes the raw request/response logs to the `Audit Database`.
* **Outputs:** SQL Updates to Postgres, Events to Kafka, and Logs to MongoDB.

### 6. External Payment Gateway (Stripe Integration)

* **Primary Responsibility:** To execute the actual financial charge against the credit card network.
* **Inputs:** API calls from the `Payment Process Worker`.
* **Processing:** External financial logic (Black box).
* **Outputs:** A JSON response indicating success (`Authorized`) or failure (`Insufficient Funds`, `Fraud`).

### 7. Retry Queue (Topic)

* **Primary Responsibility:** To handle transient errors (e.g., temporary network timeouts, gateway unavailable) by holding failed messages for a specific backoff period before re-delivery.
* **Inputs:** Failed messages from `Payment Process Worker` or `Notification Worker`.
* **Processing:** Implements a "Wait and Retry" mechanism (e.g., Exponential Backoff). It ensures that temporary failures do not block the processing of new, healthy messages in the main topic.
* **Outputs:** Re-delivers the message to the respective Worker after the delay expires.

### 8. Dead Letter Queue (DLQ) 

* **Primary Responsibility:** To act as a safety net for messages that cannot be processed after all retry attempts have been exhausted (Max Retries Reached) or for malformed messages (Poison Pills).
* **Inputs:** Failed messages from Workers after the `Retry Queue` cycle ends.
* **Processing:** Durable storage of failed contexts for manual inspection and replay.
* **Outputs:** Alerts to the operations team.

### 9. Notification Worker

* **Primary Responsibility:** To handle user communication, ensuring the customer is informed about the transaction outcome.
* **Inputs:** `PaymentResultEvent` messages from Kafka.
* **Processing:**
    1.  **Templating:** Selects the appropriate email template based on the transaction status.
    2.  **Dispatch:** Sends the email via the `Email Provider` (MailHog).
* **Outputs:** SMTP commands to the email server.

### 10. Email Provider (MailHog)

* **Primary Responsibility:** To simulate an SMTP server for development and testing, capturing sent emails without delivering them to real addresses.
* **Inputs:** SMTP traffic from the `Notification Worker`.
* **Processing:** Stores emails in memory and provides a Web UI for inspection.
* **Outputs:** Visual representation of sent emails via a web interface.

### 11. Audit Database (MongoDB)

* **Primary Responsibility:** To store unstructured, voluminous data for debugging, auditing, and traceability logs.
* **Inputs:** Raw JSON objects (Requests/Responses) from the `Payment Process Worker`.
* **Processing:** High-speed ingestion of documents.

* **Outputs:** Detailed historical logs for troubleshooting.
