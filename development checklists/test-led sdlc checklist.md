# Test-Led SDLC Checklist

This checklist defines the software development lifecycle (SDLC) standards for a **Test-Led (Test-First) Development** strategy. It incorporates Test-Driven Development (TDD), Behavior-Driven Development (BDD), Contract Testing, and Shift-Left Quality Assurance to ensure that testing drives architecture, design, and implementation from day one.

---

## Core Philosophy & Testing Pyramid

* **Shift-Left Testing:** Tests are written *before* functional code at every layer.
* **Executable Specifications:** Requirements are written in actionable, testable formats (e.g., Gherkin) that serve as automated verification suites.
* **Test Pyramid Adherence:**
  * **Base Layer (70%):** Fast, isolated Unit & Domain Tests (TDD).
  * **Middle Layer (20%):** Integration, Component & Consumer Contract Tests.
  * **Top Layer (10%):** End-to-End (E2E) & User Journey Acceptance Tests.

---

## Phase 1: Requirements & Behavioral Specification (BDD)

Define expected behavior in readable, testable formats prior to any code implementation.

* [ ] **Three Amigos Collaboration**
  * [ ] Conduct Three Amigos sessions (Product Owner, Developer, QA/Tester) for every user story before estimation or backlog refinement.
* [ ] **Behavior-Driven Specification (Gherkin/Feature Files)**
  * [ ] Write acceptance criteria as executable Gherkin scenarios using `Given-When-Then` format.
  * [ ] Cover Happy Paths, Edge Cases, Boundary Conditions, and Error Scenarios.
* [ ] **Definition of Ready (DoR) Gate**
  * [ ] Ensure user story contains executable test scenarios and measurable success criteria before entering a sprint or build cycle.

---

## Phase 2: Architecture, Testability & Contract Design

Ensure systems are designed from the ground up to support fast, decoupled, and deterministic testing.

* [ ] **Design for Testability**
  * [ ] Enforce Dependency Injection (DI) and inversion of control to allow easy mocking and stubbing.
  * [ ] Decouple external infrastructure dependencies (Databases, Third-Party APIs, Queues) using repository/adapter patterns.
* [ ] **Consumer-Driven Contract Design**
  * [ ] Define API contracts (e.g., Pact, OpenAPI specs) jointly between provider and consumer teams.
  * [ ] Publish consumer contract tests *before* provider endpoint implementation begins.
* [ ] **Test Data Strategy**
  * [ ] Define automated test data generation strategy (Factories, Fixtures, Anonymized Data) without relying on live production database dumps.

---

## Phase 3: Implementation & Test-Driven Development (TDD)

Execute the strict Red-Green-Refactor loop for all internal unit and domain logic.

* [ ] **Red Phase (Write a Failing Test First)**
  * [ ] Write a small, focused unit test that defines a single behavior or specification.
  * [ ] Run the test to confirm it **fails for the correct reason** (compilation failure or assertion mismatch).
* [ ] **Green Phase (Make the Test Pass)**
  * [ ] Write the minimum amount of functional code necessary to make the failing test pass.
  * [ ] Avoid writing speculative features or unrequested functionality ("YAGNI" principle).
* [ ] **Refactor Phase (Improve Design Under Safety)**
  * [ ] Refactor both functional and test code for clarity, performance, and maintenance while keeping all tests passing.
  * [ ] Remove code duplication and ensure adherence to clean code standards.

---

## Phase 4: Integration, Component & Contract Testing

Verify interactions between decoupled components and external systems without relying on full end-to-end environments.

* [ ] **Integration & Component Testing**
  * [ ] Implement integration tests against ephemeral dependencies (e.g., Testcontainers for databases, Redis, LocalStack).
  * [ ] Test asynchronous message handling, event publishing, and queue consumers.
* [ ] **Contract Verification**
  * [ ] Run provider contract verification tests against the consumer contracts generated in Phase 2.
  * [ ] Ensure zero breaking API schema changes occur without consumer contract updates.
* [ ] **Fault Injection & Resilience Testing**
  * [ ] Implement negative testing for timeout handling, retry policies, and circuit breaker activation.

---

## Phase 5: Non-Functional & Security Testing (Shift-Left)

Automate security, performance, and accessibility checks within early testing phases.

* [ ] **Automated Security Testing**
  * [ ] Run Static Application Security Testing (SAST) and Software Bill of Materials (SBOM) dependency scans during unit/integration runs.
  * [ ] Execute dynamic API security tests (e.g., checking for OWASP API Top 10 like BOLA, authentication bypass) against local stubs.
* [ ] **Automated Performance & Load Benchmarks**
  * [ ] Run micro-benchmarks and automated performance smoke tests (e.g., k6) on critical path logic during local/CI builds.
* [ ] **Accessibility (a11y) & Cross-Browser Testing (UI/Mobile)**
  * [ ] Integrate automated accessibility assertions (e.g., axe-core) into UI component test execution.

---

## Phase 6: CI/CD Pipeline & Quality Gates

Enforce strict, automated deployment blockages based on quality and test outcomes.

* [ ] **Pipeline Execution Sequence**
  * [ ] Run fast unit/TDD tests first (sub-5 minute target execution time).
  * [ ] Execute integration, contract, and component tests in parallel.
  * [ ] Run headless E2E acceptance tests on successful integration builds.
* [ ] **Quality Gates & Code Coverage Standards**
  * [ ] Enforce automated pipeline failure on any single broken test across any tier.
  * [ ] Enforce minimum test coverage thresholds (e.g., ≥ 85% domain coverage, 100% critical path coverage).
  * [ ] Prevent merge requests if code coverage drops relative to the main branch.
* [ ] **Test Environment Ephemerality**
  * [ ] Provision ephemeral, isolated test environments (e.g., dynamic Kubernetes preview environments) per feature branch.

---

## Phase 7: Post-Deployment Verification & Observability

Extend test-led practices into production to monitor live health and mitigate deployment risks.

* [ ] **Synthetic Monitoring & Smoke Testing**
  * [ ] Run automated post-deployment synthetic tests (Canary/Smoke tests) against production/staging endpoints immediately after deployment.
* [ ] **Feature Flag & Dark Launch Verification**
  * [ ] Verify both enabled and disabled states of feature flags using automated test runs in staging/production.
* [ ] **Telemetry & Failure Feedback Loop**
  * [ ] Capture production bug reports and outages as failing automated test cases in the code repository to prevent regressions.
