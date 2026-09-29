# API-Led SDLC Checklist

This checklist defines the software development lifecycle (SDLC) standards for designing, building, deploying, and maintaining APIs using an **API-Led Connectivity** architecture. It enforces contract-driven development, automated governance, security, and operational observability across **System**, **Process**, and **Experience** API layers.

---

## Architecture Overview: API Layers

* **System APIs:** Provide raw access to underlying Systems of Record (ERP, CRM, Databases, legacy platforms). Abstract core system complexity without business logic or orchestration.
* **Process APIs:** Orchestrate, aggregate, and transform data across multiple System APIs or domains to implement reusable core business logic.
* **Experience APIs:** Purpose-built, client-tailored data delivery layer. Format, filter, and optimize data specifically for mobile apps, web applications, or external B2B partner integrations.

---

## Phase 1: Planning & Governance

Ensure alignment with enterprise domain models, establish context boundaries, and verify existing assets before initiating design.

- [ ] **Domain & Bounded Context Mapping**
  - [ ] Define the domain boundary and classify the API layer target (**System**, **Process**, or **Experience**).
  - [ ] Assign domain owners, API Product Managers, and technical leads.
- [ ] **Consumer & Use-Case Definition**
  - [ ] Identify key target consumer personas and client environments.
  - [ ] Document functional requirements, expected latency SLAs, and peak throughput (RPS/TPS).
  - [ ] Define required access controls and security parameters (OAuth2 scopes, mTLS, API keys).
- [ ] **Service Catalog & Reuse Audit**
  - [ ] Audit the central API Portal/Registry to avoid redundant endpoints or overlapping business capabilities.
  - [ ] Identify upstream APIs/services needed for consumption (System APIs for Process APIs, etc.).

---

## Phase 2: Design & API-First Modeling

Establish the API contract using OpenAPI Specification (OAS) or AsyncAPI prior to writing implementation code. The contract serves as the single source of truth.

- [ ] **Contract Specification (OAS / AsyncAPI)**
  - [ ] Author an OpenAPI 3.x or AsyncAPI definition using standard RESTful or event-driven conventions.
  - [ ] Standardize URI patterns: kebab-case paths, plural nouns (`/v1/accounts/{accountId}/transactions`).
  - [ ] Standardize field naming conventions: `camelCase` for JSON request/response payloads.
  - [ ] Explicitly declare standard HTTP response codes (`200`, `201`, `204`, `400`, `401`, `403`, `404`, `409`, `422`, `500`).
- [ ] **Standardized Error Handling**
  - [ ] Adhere to structured error response schemas (e.g., `RFC 7807 Problem Details`).
  - [ ] Ensure public payloads omit stack traces, internal database schema details, or server infrastructure info.
- [ ] **Versioning & Lifecycle Strategy**
  - [ ] Define explicit semantic versioning strategy in URI path (`/v1/...`) or header scheme.
  - [ ] Define deprecation policy guidelines (`Sunset` and `Deprecation` HTTP headers).
- [ ] **Automated Linting & Design Governance**
  - [ ] Run automated linters (e.g., Spectral) in CI to check adherence to enterprise API style guides.
  - [ ] Conduct API Design Review with Architecture Governance Board or Lead API Architect.
- [ ] **Mocking & Consumer Validation**
  - [ ] Deploy dynamic mock endpoints (e.g., Prism, WireMock) generated directly from the spec.
  - [ ] Share mock endpoints with downstream clients for interface validation and sign-off.

---

## Phase 3: Build & Implementation

Implement integration logic while maintaining strict decoupling between API layers.

- [ ] **Code Generation & Scaffolding**
  - [ ] Generate server stubs, interfaces, and model classes directly from the API specification using code generation tools.
- [ ] **Layer Segregation Compliance**
  - [ ] **System APIs:** Verify zero orchestration or cross-domain business logic exists in this layer.
  - [ ] **Process APIs:** Verify no direct database queries or raw infrastructure connections occur; route all data calls via System APIs.
  - [ ] **Experience APIs:** Verify logic is strictly limited to response formatting, data stripping, or client-specific aggregation.
- [ ] **Configuration & Secrets Management**
  - [ ] Externalize all environment-specific configurations (endpoints, feature flags, timeouts).
  - [ ] Retrieve secrets, keys, and tokens from a central secret store (e.g., HashiCorp Vault, AWS Secrets Manager, Kubernetes Secrets); never hardcode credentials.
- [ ] **Resilience & Fault Tolerance**
  - [ ] Implement exponential backoff and randomized jitter for outbound HTTP/event retries.
  - [ ] Configure circuit breakers and connection pooling for external system dependencies.
  - [ ] Set strict read/write socket timeouts on all HTTP client calls.

---

## Phase 4: Security & Compliance

Apply zero-trust principles and enterprise security guardrails before deploying to shared environments.

- [ ] **Authentication & Access Control**
  - [ ] Enforce OAuth 2.0 / OIDC validation at the API Gateway or ingress level for Experience APIs.
  - [ ] Implement Mutual TLS (mTLS) for East-West traffic between System and Process APIs.
  - [ ] Enforce Role-Based Access Control (RBAC) or Fine-Grained Authorization (FGA) using JWT scopes and claims.
- [ ] **Input Validation & Data Sanitization**
  - [ ] Enable strict request payload schema validation against the OpenAPI contract at ingress.
  - [ ] Sanitize inputs against SQL Injection (SQLi), Cross-Site Scripting (XSS), and nested payload attacks.
- [ ] **Static & Dynamic Security Testing**
  - [ ] Run Static Application Security Testing (SAST) in automated build pipelines.
  - [ ] Generate Software Bill of Materials (SBOM) and run dependency vulnerability scans (e.g., Snyk, Trivy).
  - [ ] Perform Dynamic Application Security Testing (DAST) on exposed staging environments.

---

## Phase 5: Testing & Quality Assurance

Verify functional correctness, contract compatibility, and performance bounds using automated test suites.

- [ ] **Unit & Component Testing**
  - [ ] Maintain target code coverage (e.g., ≥ 80%) across business rules, custom transformations, and error branches.
- [ ] **Consumer-Driven Contract Testing**
  - [ ] Implement contract testing frameworks (e.g., Pact) to detect breaking downstream changes automatically.
- [ ] **API Security Testing**
  - [ ] Validate OWASP API Security Top 10 controls (e.g., BOLA, Broken Function Level Authorization, Mass Assignment).
- [ ] **Performance & Load Testing**
  - [ ] Execute stress, spike, and soak tests (e.g., k6, JMeter) against SLA throughput and latency criteria.
  - [ ] Test rate-limiting and response behavior under heavy burst traffic conditions.

---

## Phase 6: CI/CD & Deployment

Automate delivery pipelines using GitOps or declarative deployment strategies.

- [ ] **CI Pipeline Stages**
  - [ ] Validate automated pipeline sequence: `Lint Contract → Run Unit Tests → SAST/Dependency Scan → Container/Artifact Build → Publish Image`.
- [ ] **API Gateway Registration & Policy Enforcement**
  - [ ] Provision and update API Gateway routes programmatically via declarative configuration (GitOps).
  - [ ] Apply standard gateway policies: Rate Limiting, Spiking Arrest, CORS, IP Whitelisting/Blacklisting, and Logging.
- [ ] **Deployment Strategy**
  - [ ] Execute zero-downtime deployment patterns (Blue/Green or Canary).
  - [ ] Define automated health check endpoints (`/healthz`, `/livez`, `/readyz`) and configure automated rollback triggers on deployment failure.

---

## Phase 7: Observability & Operational Management

Ensure full post-deployment operational visibility and maintain API catalog lifecycle state.

- [ ] **Structured Logging & Distributed Tracing**
  - [ ] Output logs in structured JSON format containing standard fields: `trace_id`, `span_id`, `consumer_id`, and `api_version`.
  - [ ] Propagate W3C Trace Context / OpenTelemetry headers across System, Process, and Experience API boundaries.
- [ ] **Telemetry & Alerting**
  - [ ] Monitor Golden Signals: **Latency**, **Traffic (RPS)**, **Errors (4xx/5xx rates)**, and **Resource Saturation**.
  - [ ] Configure automated alerting channels (e.g., PagerDuty, Slack) for SLA thresholds or elevated error rates.
- [ ] **Developer Portal & Documentation**
  - [ ] Publish interactive API documentation (Swagger UI / Redoc) to the central Developer Portal automatically upon release.
  - [ ] Provide client SDKs, Postman collections, or sample request/response payloads for developer onboarding.
- [ ] **Lifecycle Management**
  - [ ] Track active API consumers and monitor usage metrics.
  - [ ] Enforce lifecycle state updates (`Design` → `Active` → `Deprecated` → `Retired`).
