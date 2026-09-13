<p align="center">
  <img src="../img/MitM_Data_Aggregator_transparent.png" width="400" alt="MitM-3 Data Aggregator Logo"/>
</p>

# Architecture Concept: MitM-3 Aggregator (Agentic SDD)

## 1. Goal and Characteristics

- **Goal:** Develop an asynchronous, reliable middleware (`mitm-3`) that gathers heterogeneous on-premise application data (CSV, Oracle, PostgreSQL, Kafka, Medical Devices), normalizes it, and sends it to external SaaS platforms.
- **Latency Tolerance:** Fully asynchronous. Delay times up to 24 hours are acceptable.
- **Data Privacy (PII):** Highly sensitive employee data. Must be strictly encrypted using "Envelope Encryption" (Master Key / DEK).
- **Throughput:** Scalable to handle millions of fragments per day, aggregated into manageable JSON packages.
- **Availability:** Focus on resilience and retry capabilities rather than 99.999% uptime, as the daily window allows sufficient time for automatic retries.

## 2. Methodology & Governance

- **SpecDD & GitHub Spec Kit:** The project utilizes a "Flow-Forward" workflow. Architecture intent is defined in `.sdd` files, while iterative feature changes use GitHub Spec Kit in isolated `specs/` directories.
- **Governance:** Strict enforcement of AES-GCM Envelope Encryption and separation of Authentication/Authorization (AuthN/AuthZ) in the Core Layer.

## 3. Bounded Contexts and Components

- **Core-Layer (Rust 2024):**
  - **mitm_http-server:** Permanent, domain-driven REST API entry point with AuthN/AuthZ middleware.
  - **mitm_scheduler-server:** Controls and triggers the collection, transformation, and delivery processes.
  - **mitm_iam-server:** Permanent Identity and Access Management with OIDC/OAuth2, RBAC, and ABAC for PII data.
- **Admin Frontend (Qt6/C++23):** Located in `admin-frontend` (branch `mitm-3_v2.xx`). Native UI for orchestration, mapping rules, and monitoring logs.
- **Collector-Layer (Go):** Connects to heterogeneous sources (CSV, Oracle, PostgreSQL, Kafka, Medical Devices) via isolated Go binaries. Retrieves raw data from these sources.
- **Transformation-Layer (Rust):** Located in `transformation-layer` (branch `mitm-3_v2.xx`). Transforms raw data, validates, and encrypts (via DEK) before persisting fragments.
- **Delivery-Layer (Rust 2024):** Separated into `mitm_apigee` and `mitm_cority`. Builds JSON packages and sends them to SaaS platforms (handles rate limits, retries, DLQ).
- **Maintenance-Layer (Rust 2024):** Includes `mitm_cleanup` for scheduled database cleanup batch jobs.
- **State (Storage):** Persistently stores progress (cursors) and asynchronously buffered fragments (PostgreSQL database).
- **Security:** Cross-cutting component for encryption (Envelope Encryption), key management (IPC Unix Sockets), and ABAC-based access controls.

## 4. Interfaces

- **Adapter Interface:** Definition of how pollers read data and layers communicate.
- **REST Contract to SaaS:** Transmission via POST request including authentication and idempotency keys.
- **DB Schema:** State management via PostgreSQL.
- **DLQ & Audit:** Tables for undeliverable payloads (DLQ) and a write-only audit log for security-relevant events.

## 5. Data Flow and Workflow

1. **Trigger:** `mitm_scheduler-server` (Rust) triggers the respective collector in the **Collector-Layer** (Go).
2. **Transformation & Validation:** Raw data passes to the **Transformation-Layer** (Rust) for validation, normalization, DEK-based encryption, and persistence in the `fragments` table.
3. **Packaging & Delivery:** The **Delivery-Layer** (`mitm_apigee` or `mitm_cority` in Rust) aggregates pending fragments, persists them in the `packages` table, and sends them to the external SaaS REST API.
   - **Success:** Package status is set to `delivered`.
   - **Temporary Error:** Exponential backoff & retry.
   - **Permanent Error:** Moved to the DLQ for manual review/replay.
4. **IAM Control:** `mitm_iam-server` (Rust) handles access checks (RBAC/ABAC) when administrators interact with the `mitm_http-server` API.

## 6. Deployment and Operation

- **Container Image:** Rust and Go components compiled as distroless or Alpine images (minimal footprint).
- **Runtime:** Docker containers on the Admin Host (AWS EC2). Kubernetes ready.
- **Backup Strategy:** Encrypted backups of PostgreSQL database in AWS S3.
- **Monitoring & Logging:** `prometheus` metrics (`/metrics`). Structured JSON logging.
- **Health Checks:** `/healthz` (app status) and `/readyz` (DB connection, IPC key availability).

## 7. Security and Key Management

- **Envelope Encryption:** Each fragment gets a generated DEK. The DEK is encrypted with the Master Key (KEK).
- **MasterKey Handling:** **The KEK must never be persisted to disk**. Memory locking (`mlock` via secrecy crates) and zeroization MUST be applied to prevent swap leakage. The root Scheduler retrieves it into RAM and securely injects it and database credentials to sub-processes (Go collectors, Rust modules) via a bidirectional Unix Domain Socket (IPC). Child processes never inherit environment variables.
- **AuthN/AuthZ:** External IdP or internal OAuth2 API via `mitm_iam-server`. Passwords MUST be hashed using Argon2. MFA enforced. Access tokens (JWT) used for internal API communication.
- **TLS:** All external connections enforce TLS 1.2+.
- **Audit:** An append-only audit log records key rotations, app starts, and failed authentication attempts. It uses cryptographic hash-chaining to guarantee tamper-evidence for Medical Device regulatory compliance.

## 8. Toolchain and Libraries

- **Languages:** Rust 2024 (Core/Transformation/Delivery/Maintenance), Go (Collectors), C++23 (Frontend).
- **DB Driver:** Rust `sqlx` or `tokio-postgres`, Go `pgx/v5`.
- **Encryption:** Rust `aes-gcm` crate / Go `crypto/aes` + `crypto/cipher` (AES-GCM).
- **API/Web:** Rust `axum` or `actix-web` for HTTP, `casbin` for dynamic AuthZ policies.

## 9. Risks and Mitigation

| Risk | Mitigation |
| :--- | :--- |
| **Data Privacy / Key Leakage** | Envelope Encryption; KEK in RAM via IPC only; no PII in logs; strict ABAC. |
| **Schema Drift of Sources** | Versioned Go collectors; robust fallbacks; DLQ on parsing errors. |
| **SaaS Rate Limits** | Politeness delays and exponential backoff in Rust Delivery components. |
| **Architecture Drift** | Flow-Forward workflow with SpecDD linting and GitHub Spec Kit. |
| **PostgreSQL Concurrency** | Use a connection pool (e.g., pgxpool, sqlx pool), configure appropriate connection limits, and enforce `SELECT ... FOR UPDATE SKIP LOCKED` for fragment queue processing. |

---

### Security Flow Diagram

1. **Start:** `mitm_scheduler-server` starts, retrieves `MASTER_KEY` (KEK) into RAM, and opens a secure IPC Unix socket.
2. **Collect:** Collector-Layer instances boot, fetch KEK and DB credentials via IPC, and retrieve data.
3. **Transform & Validate:** Transformation-Layer validates and normalizes.
4. **DEK Gen:** App generates a random 32-byte DEK (Data Encryption Key) for the fragment.
5. **Encrypt Payload:** Data encrypted via AES-GCM (`payload_encrypted`).
6. **Encrypt DEK:** DEK encrypted via AES-GCM and `MASTER_KEY` (`encrypted_dek`).
7. **Store:** `payload_encrypted` and `encrypted_dek` are stored in PostgreSQL. DEK is deleted from RAM.
8. **Read & Send:** Delivery-Layer (`mitm_apigee`/`mitm_cority`) reads `encrypted_dek`, decrypts it with `MASTER_KEY` to retrieve the DEK, decrypts payload, packages it, sends, and discards the DEK.
9. **IAM API Check:** UI/Admin tasks pass via `mitm_http-server`, which validates JWTs against `mitm_iam-server` before modifying policies or exposing logs.

---

## Acceptance Criteria (Checklist)

- [ ] Structure covers all requested areas of the hybrid Rust/Go MitM-3 architecture.
- [ ] PII data protection through envelope encryption and ABAC is conceptually anchored.
- [ ] Architecture considers asynchronous retry management and Dead Letter Queues for robust SaaS delivery.
- [ ] The chosen tech stack (Rust, Go, PostgreSQL, OSS libraries) consists 100% of open, license-free components.
- [ ] All required artifacts (ERD, DDL, SpecDD, CI/CD, prototype) are integrated.
- [ ] Key security (AuthN/AuthZ separation, IPC key sharing) and operational decisions are highlighted in the text.
