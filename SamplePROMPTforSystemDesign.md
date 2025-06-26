# Sample PROMPT for system design

consider following problem context for a system design interview, and give a step wise breakdown of the solution as per RADIO framework. Create back of the envelope estimates while thinking of non-functional requirements:

**Online Sales System is a digital Loan Origination System designed to administer and process customer applications for financial & mortgage products submitted through online forms, submitted via the XYZ bank homepage/Net Bank, Mobile banking & Front-Line office. The system automates the end-to-end loan application lifecycle by applying business, credit, and fraud validation rules, while seamlessly interfacing with internal XYZ Bank systems (e.g., Credit System) and external services**

---

# RESPONSE

Of course. Here is a detailed, step-by-step breakdown for designing the Online Sales System (OSS) using the RADIO framework.

### **R - Requirements**

First, we must clarify the system's functional and non-functional requirements.

#### **1. Functional Requirements (FRs)**

These define what the system must do.

  * **Customer-Facing:**
      * **F1: Application Submission:** Customers must be able to fill out and submit loan/mortgage applications through various channels (Web, Mobile, Net Bank).
      * **F2: Document Upload:** Customers must be able to upload required supporting documents (e.g., payslips, identity proofs).
      * **F3: Application Tracking:** Customers must be able to view the real-time status of their application.
      * **F4: Offer Management:** Customers must be able to view, accept, or reject a loan offer.
      * **F5: E-Signature:** Customers should be able to electronically sign loan agreements.
  * **Bank Employee-Facing (Front-Line/Underwriter):**
      * **F6: Application Dashboard:** Employees must have a dashboard to view and manage incoming applications.
      * **F7: Manual Review:** Underwriters must be able to review application details, documents, and automated rule results.
      * **F8: Manual Override:** Authorized personnel must be able to override automated decisions.
      * **F9: Communication:** Employees should be able to request additional information from the customer via the system.
  * **System/Administrative:**
      * **F10: Rule Engine:** The system must process applications against a configurable set of business, credit, and fraud rules.
      * **F11: Integrations:** The system must integrate with internal systems (Credit System, Customer Database) and external services (Credit Bureaus, Identity Verification Services).
      * **F12: Notifications:** The system must send automated notifications (email, SMS, push) to customers and employees at key stages.
      * **F13: Audit Trail:** The system must maintain a detailed, immutable log of all actions taken on an application for compliance and security.

#### **2. Non-Functional Requirements (NFRs)**

These define the qualities and constraints of the system.

  * **N1: Availability:** As a core financial system, it must be highly available.
      * **Target:** 99.99% uptime.
      * **Calculation:** This translates to a maximum of $0.01% \\times 365.25 \\times 24 \\times 60 \\approx 52.6$ minutes of downtime per year.
  * **N2: Latency:** User interactions should be fast, while backend processing can be asynchronous.
      * **P99 Latency for synchronous APIs (UI-facing):** \< 250ms.
      * **End-to-end Application Processing (from submission to initial decision):** Can vary from a few seconds (fully automated) to hours/days (if manual review is needed). The goal for automated decisions is \< 1 minute.
  * **N3: Scalability & Throughput (Back-of-the-Envelope Estimates)**
      * **Assumptions:**
          * XYZ Bank has \~5 million digital users.
          * On a peak day (e.g., marketing campaign), 0.1% of users might initiate an application.
          * Number of applications/day: $5,000,000 * 0.001 = 5,000$ applications.
      * **Write QPS (Queries Per Second):**
          * An application submission might involve \~50 API calls (saving form pages, uploads, etc.).
          * Total requests: 5,000 * 50 req/application = 250,000 requests/day.
          * Peak activity occurs over 4 hours: 250,000 / (4 * 3600) /s = ~ 20 QPS.
          * With a 5x safety multiplier, we should design for **~100 Writes QPS**.
      * **Read QPS:**
          * Customers check status multiple times. Let's assume 10 status checks per application.
          * 5,000 * 10 / checks = 50,000 read requests.
          * Bank employees also query the system. Let's estimate total read requests are 5x write requests.
          * We should comfortably handle **~500 Read QPS**.
  * **N4: Storage (Back-of-the-Envelope Estimates)**
      * **Assumptions:**
          * Application data (structured): ~500 KB per application.
          * Documents (unstructured): ~10 MB per application (images, PDFs).
          * Audit logs: ~200 KB per application.
          * Total per application: ~ 11 MB.
      * **Daily Growth:** 5,000 applicationss/day * 11 MB/application = ~ 54 GB/day.
      * **Annual Growth:** $54 GB/day * 365 days = ~ 20 TB/year.
      * **5-Year Retention Policy:** The system should be designed to store at least $20 TB * 5 = ~ 100 TB of data.
  * **N5: Consistency:** Strong consistency is required for financial transactions. When an application is submitted or a decision is made, that state must be immediately and consistently reflected across the system for that user.
  * **N6: Durability:** The system must guarantee zero data loss for submitted applications and their associated artifacts. Data should be backed up and replicated across multiple availability zones or regions.
  * **N7: Security & Compliance:**
      * **Encryption:** End-to-end encryption (TLS 1.3+) for data in transit and encryption at rest (AES-256) for all stored data.
      * **PII:** Personally Identifiable Information must be handled with extreme care, possibly encrypted at the field level.
      * **Compliance:** Must comply with GDPR, PSD2, and other relevant financial regulations.
      * **Authentication & Authorization:** Role-Based Access Control (RBAC) for all users (customers and employees).

-----

### **A - Architecture / High-Level Design**

We will use a **Microservices Architecture** to promote scalability, separation of concerns, and independent deployment cycles. The system is event-driven to handle the asynchronous and multi-step nature of a loan application.

1.  **Clients:** Web/Mobile/Net Bank frontends (built on React/Angular/Swift/Kotlin) and the Front-Line Office portal. They communicate with the backend via a single entry point.
2.  **API Gateway:** A managed service (like AWS API Gateway or Apigee) that acts as the front door for all incoming requests. It handles SSL termination, authentication, rate limiting, and routing to the appropriate downstream microservice.
3.  **Core Microservices:**
      * **Application Service:** The primary orchestrator. It manages the state of the loan application lifecycle (e.g., `DRAFT`, `SUBMITTED`, `IN_REVIEW`). It exposes the main APIs for creating, updating, and fetching applications.
      * **Customer Service:** Manages customer profiles, authentication, and authorization.
      * **Document Service:** Handles the secure upload, storage, and retrieval of documents. It abstracts the underlying object storage.
      * **Rules Engine Service:** A dedicated service that ingests application data and executes a chain of business, credit, and fraud rules. This service is highly configurable.
      * **Decision Service:** Consumes the output from the Rules Engine and external integrations to produce a final or preliminary decision (e.g., `AUTO_APPROVED`, `REJECTED`, `MANUAL_REVIEW_REQUIRED`).
      * **Notification Service:** A centralized service for sending emails, SMS, and push notifications. It subscribes to events from other services.
4.  **Integration Layer (Anti-Corruption Layer):**
      * This is a crucial pattern. A dedicated service acts as a facade for all external and internal legacy systems. This isolates our modern OSS microservices from the complexities and potential instability of external dependencies. It handles protocol translation and data transformation for:
          * **Internal:** XYZ Bank Credit System, Core Banking System.
          * **External:** Credit Bureaus (e.g., Experian), Identity Verification (e.g., Veriff), Fraud Detection services.
5.  **Asynchronous Communication Backbone:**
      * **Message Queue (e.g., Apache Kafka, RabbitMQ):** This decouples the services and enables a resilient, event-driven workflow.
          * When an application is submitted, the **Application Service** writes its state to its database and publishes an `ApplicationSubmitted` event to a Kafka topic.
          * The **Rules Engine**, **Integration Layer**, and other services subscribe to this event, perform their tasks in parallel, and publish their results (e.g., `CreditCheckCompleted`, `FraudCheckFailed`) back to another topic.
          * The **Decision Service** listens for these result events to make a final judgment.

-----

### **D - Data Model / Database Design**

We'll use a polyglot persistence approach: the right database for the right job.

  * **Primary Database (Relational - e.g., PostgreSQL):** For transactional, structured data where ACID compliance is paramount.
  * **Object Storage (e.g., Amazon S3):** For storing unstructured binary data like documents.
  * **Cache (e.g., Redis):** For session data and caching frequently accessed, non-critical information.

**Key Database Tables (PostgreSQL Schema):**

```sql
-- Stores customer information
CREATE TABLE Customers (
    customer_id UUID PRIMARY KEY,
    first_name VARCHAR(255) NOT NULL,
    last_name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE,
    phone_number VARCHAR(50),
    XYZ_id VARCHAR(100) UNIQUE NOT NULL, -- Internal bank customer ID
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Main table for tracking the loan application lifecycle
CREATE TYPE application_status AS ENUM (
    'DRAFT', 'SUBMITTED', 'IN_REVIEW', 'ADDITIONAL_INFO_REQUIRED',
    'APPROVED', 'REJECTED', 'OFFER_ACCEPTED', 'CLOSED'
);

CREATE TABLE Applications (
    application_id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES Customers(customer_id),
    product_type VARCHAR(100) NOT NULL, -- e.g., 'MORTGAGE', 'PERSONAL_LOAN'
    status application_status NOT NULL DEFAULT 'DRAFT',
    amount_requested DECIMAL(15, 2) NOT NULL,
    loan_term_months INT NOT NULL,
    application_data JSONB, -- Flexible field for all form data
    decision_details JSONB, -- Stores output from the decision engine
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

-- Stores metadata about uploaded documents
CREATE TABLE Documents (
    document_id UUID PRIMARY KEY,
    application_id UUID NOT NULL REFERENCES Applications(application_id),
    document_type VARCHAR(100) NOT NULL, -- e.g., 'PAYSLIP', 'ID_PROOF'
    storage_path VARCHAR(1024) NOT NULL, -- e.g., S3 object key
    file_name VARCHAR(255) NOT NULL,
    mime_type VARCHAR(100),
    uploaded_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Immutable log for compliance and auditing
CREATE TABLE AuditLogs (
    log_id BIGSERIAL PRIMARY KEY,
    application_id UUID NOT NULL REFERENCES Applications(application_id),
    actor VARCHAR(255) NOT NULL, -- e.g., 'customer:uuid' or 'system:rules_engine'
    action VARCHAR(255) NOT NULL, -- e.g., 'APPLICATION_SUBMITTED', 'DOCUMENT_UPLOADED'
    details JSONB,
    timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

-----

### **I - Interfaces / APIs**

APIs will be designed using RESTful principles for synchronous communication. Asynchronous communication will be managed via events.

#### **1. Synchronous REST APIs (Example)**

  * **Application Service:**
      * `POST /api/v1/applications`: Creates a new application in `DRAFT` state.
      * `PUT /api/v1/applications/{applicationId}`: Updates application data (saves progress).
      * `GET /api/v1/applications/{applicationId}`: Retrieves the current status and details of an application.
      * `POST /api/v1/applications/{applicationId}/submit`: Finalizes and submits the application, changing its status to `SUBMITTED`.
  * **Document Service:**
      * `POST /api/v1/applications/{applicationId}/documents`: Uploads a document. The service returns a `document_id`. The client would get a pre-signed URL to upload the file directly to S3 to offload traffic from the service.
      * `GET /api/v1/documents/{documentId}/download`: Returns a secure, time-limited URL to download the document.

#### **2. Asynchronous Events (Kafka)**

  * **Topic: `application_events`**
      * **Event:** `ApplicationSubmitted`
      * **Payload:** `{ "applicationId": "uuid", "customerId": "uuid", "productType": "MORTGAGE", "submissionTimestamp": "iso8601" }`
      * **Consumers:** Rules Engine, Integration Layer (for credit check).
  * **Topic: `decision_events`**
      * **Event:** `RuleEngineResult`
      * **Payload:** `{ "applicationId": "uuid", "results": [...], "outcome": "PASS" }`
      * **Consumers:** Decision Service.
      * **Event:** `CreditBureauResult`
      * **Payload:** `{ "applicationId": "uuid", "creditScore": 750, "reportId": "xyz" }`
      * **Consumers:** Decision Service.
  * **Topic: `notification_requests`**
      * **Event:** `SendNotification`
      * **Payload:** `{ "channel": "EMAIL", "recipient": "customer:uuid", "template": "APPLICATION_APPROVED", "data": {...} }`
      * **Consumer:** Notification Service.

-----

### **O - Optimizations / Deep Dive**

#### **1. Long-Running Workflows & State Management**

A loan application is a long-running workflow with potential human-in-the-loop steps (manual underwriting). Simply managing state in the `Applications` table can become complex.

  * **Solution:** Introduce a **Workflow Orchestration Engine** like [Temporal](https://temporal.io/), [Cadence](https://cadenceworkflow.io/), or AWS Step Functions.
  * **How it works:** The `Application Service` would initiate a workflow instance when an application is submitted. The orchestrator would then manage the entire flow: calling the Rules Engine, waiting for external API calls, assigning a task to a human underwriter, handling timeouts, and ensuring the process completes reliably, even over days. This is far more robust than custom state management logic.

#### **2. The Rules Engine Design**

This is the system's "brain" and must be flexible.

  * **Decoupled:** It should be a standalone service.
  * **Configurable:** Business analysts, not engineers, should be able to update rules.
  * **Implementation:** Use a known rules engine framework like **Drools** or build a custom DSL (Domain-Specific Language) where rules are defined in a human-readable format (like YAML or JSON) and stored in a database.
  * **Example Rule (in JSON):**
    ```json
    {
      "ruleName": "CheckCreditScoreMinimum",
      "condition": {
        "fact": "creditScore",
        "operator": "lessThan",
        "value": 620
      },
      "action": {
        "type": "FLAG_FOR_REJECTION",
        "reason": "Credit score is below the minimum threshold of 620."
      }
    }
    ```

#### **3. Security & Idempotency**

  * **Idempotent APIs:** In a distributed system with retries, a service might receive the same message twice. APIs must be idempotent. For example, a `POST /submit` request should include a unique `Idempotency-Key` in the header. The server would store this key for a short period, and if the same key is seen again, it would simply return the original response without reprocessing the request.
  * **PII Data Vaulting:** For extremely sensitive PII (e.g., national ID numbers), instead of storing it directly in the main database, we can use a **data vault** (like HashiCorp Vault). The main DB would only store a token/reference to the data in the vault, which has much stricter access controls and audit capabilities.

#### **4. Fault Tolerance**

  * **Circuit Breaker:** When calling external services (like a credit bureau), wrap the call in a circuit breaker pattern (e.g., using a library like Resilience4j). If the external service is down or slow, the circuit "opens," and requests fail fast, preventing system resources from being tied up. The system can then route the application for manual review or retry later.
  * **Dead Letter Queue (DLQ):** For the message queue, if a message consistently fails processing after several retries, it should be moved to a DLQ. An operator can then inspect these failed messages to debug the issue without blocking the main processing pipeline.

---

# Arch

                  +---------------------------------+
                  |      Clients (Web, Mobile,      |
                  |      Front-Line Office)         |
                  +---------------------------------+
                              |
                              | (HTTPS/REST API Calls)
                              v
+------------------------------------------------------------------------------------------+
|                            API Gateway                                                   |
|                (Authentication, Rate Limiting, Routing)                                  |
+------------------------------------------------------------------------------------------+
     |                                      |                                      ^
     | (e.g., /applications/*)              | (e.g., /documents/*)                 | (e.g., /customers/*)
     v                                      v                                      |
+--------------------------------------[ Core OSS Backend Services (Kubernetes/Cloud Platform) ]-----------------------------------------+
|                                                                                                                                        |
|  +---------------------+   (Writes State)   +-----------------------+           +---------------------+      +---------------------+   |
|  | Application Service |------------------->|   Primary Database    |<----------|   Decision Service  |      |   Customer Service  |---|
|  | (Orchestrator)      |<-------------------|   (PostgreSQL - ACID) |<--+       +---------------------+      +---------------------+   |
|  +---------------------+   (Reads State)    +-----------------------+   |               ^                             |                |
|            |                                                            |               | (Listens for results)       | (CRUD)         |
|            | (Publishes Event: e.g., 'ApplicationSubmitted')            |               |                             v                |
|            v                                                            |               |                     +-----------------+      |
|    +--------------------------------------------------------------------+               |                     |   Cache (Redis) |      |
|    |                      Message Queue (Apache Kafka)                  |---------------+---------------------| (Sessions, etc) |      |
|    | [Topics: application_events, decision_events, notification_reqs]   |               |                     +-----------------+      |
|    +--------------------------------------------------------------------+               |                                              |
|            |                                                                            |                                              |
|            +------------------+------------------+---------------------+----------------+                                              |
|   (Subscribes to Events)      |                  |                     |                (Subscribes to Events)                         |
|            v                  v                  v                     v                                                               |
|  +-----------------+  +--------------------+  +-------------------+  +------------------------+      +--------------------------+      |
|  |   Rules Engine  |  | Integration Layer  |  | Notification Svc  |  |    Document Service    |----->|   Object Storage (S3)    |      |
|  | (Runs Checks)   |  | (Anti-Corruption)  |  | (Email, SMS)      |  | (Manages Files)        |<-----|   (PDFs, Images, etc)    |      |
|  +-----------------+  +--------------------+  +-------------------+  +------------------------+      +--------------------------+      |
|            |                  |                        |                                                                               |
|            | (Reads data)     | (Calls External APIs)  | (Calls 3rd Party Gateways)                                                    |
|            +------------------+                        v                                                                               |
|                               |               +----------------+                                                                       |
|                               |               | Email/SMS Gw   |                                                                       |
|                               v               +----------------+                                                                       |
|             +----------------------------------+-----------------------------------+                                                   |
|             |        External & Internal System Integrations                       |                                                   |
|             |                                                                      |                                                   |
|             |     +----------------+   +------------------+   +----------------+   |                                                   |
|             +---->|  Credit Bureau |-->|  ID Verification |-->| XYZ Systems |<--+                                                   |
|                   |   (Experian)   |   |     (Veriff)     |   | (Credit, Core) |                                                       |
|                   +----------------+   +------------------+   +----------------+                                                       |
|                                                                                                                                        |
+----------------------------------------------------------------------------------------------------------------------------------------+


LEGEND:
--------> : Synchronous Request / Direct DB Call (REST, gRPC, SQL)
--------- : Asynchronous Event Flow (Publish/Subscribe to Kafka)

# SYSDESIGN


+---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|                                                           AWS Organization / Landing Zone                                                                                                         |
|                                                                                                                                                                                                   |
| +------------------------------------[ OSS Application Account ]--------------------------------------+  +--------------------[ Customer Service Account ]------------------------+                |
| |                                                                                                     |  |                                                                        |                |
| | +-----------[ Web Tier VPC ]-----------+      +----------------[ App Tier VPC ]----------------+    |  | +-----------------------[ CQRS Implementation ]--------------------+ |
| | | +----------------------------------+ |      |                                                |    |  | | +----------------+ (Writes) +-----------------------------+    | |
| | | |        Amazon CloudFront         | |      |  +--------------------------+                  |    |  | | | Command API    |---------->| Amazon RDS for PostgreSQL   |    | |
| | | |          + AWS WAF               | |      |  | AWS Fargate on Amazon EKS|                  |    |  | | +----------------+           | (Write Model)               |    | |
| | | +----------------------------------+ |      |  | [OSS API Backend]        |--------->(Cross-Account API Call)-->|                            +-----------------------------+    | |
| | |                 | (HTTPS)            |      |  +--------------------------+       ^          |    |  | |                              | (DMS/Event Replication)          | |
| | |                 v                    |      |            |                      |          |    |  | |                              v                                | |
| | | +----------------------------------+ |      |            | (Publishes Events)   |          |    |  | | +-----------------------------+                              | |
| | | |      Amazon API Gateway          | |      |            v                      |(Cross-Acct |    |  | | | Amazon DynamoDB             |<-------------------------------+ |
| | | |      (Public Endpoint)           | |      |    +------------------+           | API Call) |    |  | | | (Read Model, Hierarchical)|      +---------------------+   | |
| | | +----------------------------------+ |      |    | Amazon           |           |           |    |  | | +-----------------------------+      | ElastiCache (Redis) |   | |
| | |                                      |      |    | EventBridge      |           |           |    |  | |                                      | (1hr TTL)           |   | |
| | +-----------------|--------------------+      |    | (Central Bus)    |           |           |    |  | |                                      +----------^----------+   | |
| |                   |                          |    +--------^---------+           |           |    |  | |                                                 | (Reads)    | |
| |                   v (VPC Peering / Transit Gateway)         | (Subscribes)          |           |    |  | | +----------------+                              |            | |
| |                                                |            +-----------------------+           |    |  | | | Query API      |<--------------------------------+            | |
| |                                                |                                                |    |  | | +----------------+                                           | |
| |                                                +------------------------------------------------+    |  | +--------------------------------------------------------------+ |
| +----------------------------------------------------------------------------------------------------+  +------------------------------------------------------------------------+ |
|                                                               ^      |                                                                                                           |
|          (Cross-Account Permissions via IAM Roles & Resource Policies) |      v (Event Published to Bus)                                                                           |
|                                                                        |      |                                                                                                           |
| +--------------------[ Document Management Account ]-------------------+      +-------------------------------[ Credit System Account ]--------------------------+ |
| |                                                                       |      |                                                                                  | |
| |  +---------------------------------------------------------------+    |      |  +--------------------------+       +------------------+                       | |
| |  | Document API (API GW + Fargate)                               |    |<----(Cross-Account API Call)       |  | << Sub (EventBridge Rule) |       | Event Published >> |                       | |
| |  +---------------------------------------------------------------+    |      |  +--------------------------+       +------------------+                       | |
| |                           |                                         |      |                                                                                  | |
| |                           v (Manages & serves from S3)              |      +----------------------------------------------------------------------------------+ |
| |  +---------------------------------------------------------------+    |                                                                                         |
| |  | Amazon S3 (KMS Encrypted)                                     |    |                                                                                         |
| |  +---------------------------------------------------------------+    |                                                                                         |
| |                                                                       |                                                                                         |
| +-----------------------------------------------------------------------+                                                                                         |
|                                                                                                                                                                   |
| +-----------------[ Centralized Logging & Auditing Account ]--------------------------+  +--------------------[ Digital Core Account ]---------------------------+ |
| |                                                                                     |  |                                                                       | |
| |  +-----------------------------------------------------------------------------+  |  |  (Legacy Systems, Core Banking APIs)                                   | |
| |  | Log Ingestion API (API GW + Lambda)                                         |  |  |  +-------------------------------------------------------------------+ |
| |  +-----------------------------------------------------------------------------+  |  |  |  Core Banking Service API                                         | |
| |                                  ^                                                |  |  |  (Accessed via PrivateLink / Transit Gateway)                     | |
| |                                  | (Cross-Account API Call)                       |  |  +-------------------------------------------------------------------+ |
| |  +-----------------------------------------------------------------------------+  |  |                                                                       | |
| |  | Write-Ahead-Log (WAL) Database                                              |  |  |                                                                       | |
| |  | (e.g., RDS w/ append-only tables, managed service)                          |  |  |                                                                       | |
| |  +-----------------------------------------------------------------------------+  |  |                                                                       | |
| |                                                                                     |  |                                                                       | |
| +-------------------------------------------------------------------------------------+  +-----------------------------------------------------------------------+ |
|                                                                                                                                                                   |
+--------------------------------------------------------------------------------------------------------------------------------------------------------------------+
