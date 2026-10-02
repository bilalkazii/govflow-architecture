# GovFlow — Development Plan

| | |
|---|---|
| **Status** | Phase 1 complete |
| **Method** | Incremental, testable phases. No phase breaks prior functionality. |
| **Rule** | After each phase: run tests → security check → verify architecture → update docs. |

## 0. Working agreements

- Every phase ends in a green, runnable state (local compose at minimum from Phase 2 onward).
- Security architecture is never shortcut: policy engine, confirmation gate, injection screening, and audit calls are wired **when the feature that needs them lands**, not retrofitted.
- Definition of Done checklist (spec §58) is tracked at the bottom; items are claimed only when demonstrably working.

## Phase 1 — Repository + architecture ✅ (complete)

**Goal:** Foundation documents and structure.
**Deliverables:** monorepo skeleton; `docs/architecture.md`; `docs/threat-model.md`; `docs/development-plan.md`; `docs/database-erd.md`; `docs/api-contract.md`; `docs/service-workflows.md`; `docs/terraform-architecture.md`; `docs/kubernetes-architecture.md`; `docs/jenkins-cicd-architecture.md`; `docs/dependencies-and-risks.md`; `docs/demo-script.md`; README; .gitignore.
**Exit criteria:** all docs reviewed for consistency; ERD covers all 15 entities; API contract covers every screen's data needs; workflow definitions for 3 services complete.

## Phase 2 — Frontend shell

**Goal:** Premium UI shell matching the reference image.
**Tasks:** Next.js (App Router) + TS strict + Tailwind + ESLint/Prettier; design tokens (blue/white gov-tech, WCAG AA); app shell (sidebar, topbar, search, language selector, notifications, profile); screens 1–3 (Home/AI assistant, Conversation, Service identification); routing; loading/skeleton/error/empty components; i18n scaffold (en/hi/mr); mock API layer behind service interfaces.
**Exit criteria:** all three screens render responsively at 1440/768/390 widths; Lighthouse a11y ≥ 90; no duplicated UI primitives.

## Phase 3 — Backend skeleton

**Goal:** FastAPI service with strict schemas, health, structured logging, error envelope.
**Tasks:** app factory, settings via pydantic-settings; `/healthz`, `/readyz`, `/metrics`; request ID + correlation middleware; structured JSON logs with redaction filter; global exception handler (no stack traces to clients); API versioning; rate limiter; Dockerfile (multi-stage, non-root).
**Exit criteria:** pytest suite green; log redaction test passes; image runs as non-root.

## Phase 4 — Database

**Goal:** PostgreSQL + Alembic + repositories.
**Tasks:** SQLAlchemy 2 async engine; models per [database-erd.md](database-erd.md); Alembic initial migration; async repository layer with ownership filters; seed script (demo user, services); docker-compose postgres.
**Exit criteria:** migrations up/down clean; repository tests green; no raw SQL in API layer.

## Phase 5 — Service Registry

**Goal:** DB-backed, versioned registry driving all service logic.
**Tasks:** Service/ServiceVersion models + admin CRUD API; load the 3 MVP definitions from [service-workflows.md](service-workflows.md) as seed; public read endpoints (`/services`, requirements, fees, tatkal); registry cache invalidation.
**Exit criteria:** AI-facing requirement queries served entirely from registry; version pinning works; tests for jurisdiction/version resolution.

## Phase 6 — AI Agent

**Goal:** Controlled agent with 9 tools behind the Tool Gateway.
**Tasks:** ai-agent service; conversation sessions; prompt assembly (registry-derived facts); structured output validator + bounded repair; tool implementations with ownership checks; injection screening on input; output scanning; LLM provider abstraction (mock provider for tests/local); language handling (en/hi/mr); screens 2–3 wired to real backend.
**Exit criteria:** critical tests #1 (no direct submit), #4 (injection executes no tools) pass; tool-call audit events emitted; mock-provider e2e conversation works.

## Phase 7 — Document Intelligence

**Goal:** Upload → validate → scan → OCR → extract → verify pipeline.
**Tasks:** document-service; S3/MinIO-compatible storage with SSE-KMS (dev: local KMS-equivalent envelope encryption); MIME/size gates; ClamAV hook (+ stub for local profile); OCR sandbox (Tesseract); extraction with schema validation + confidence; DigiLocker mock behind `DocumentProvider` interface (consent → list → fetch); screens 4–6; verification UI with source/confidence badges.
**Exit criteria:** malicious-document test quarantines with zero tool calls (test #4 variant); extracted fields flow into form draft; screens 4–6 complete.

## Phase 8 — Workflow Engine

**Goal:** Deterministic persisted state machine controlling applications.
**Tasks:** workflow definitions per service version; `WorkflowExecution` persistence; transition guards + idempotency; screens 7–9 (auto-populated editable form with AI/user/verified badges, review & confirm, application received with `GF-XX-######` ID).
**Exit criteria:** illegal transitions rejected + tested; crash mid-transition recovers; form screens complete.

## Phase 9 — Policy Engine

**Goal:** Central decision point for sensitive operations.
**Tasks:** declarative policy rules; decisions persisted + audited; confirmation artifact (hash of reviewed payload) required for `request_submission`; step-up auth simulation; admin authorization matrix; screens wired (submit blocked without confirmation — test #3).
**Exit criteria:** policy tests green incl. unauthorized admin block (test #9); every decision auditable.

## Phase 10 — Mock Government APIs

**Goal:** Realistic mock UMANG/state backend with fault injection.
**Tasks:** mock-government service; endpoints per spec §41; fault profiles (healthy/slow/overloaded/429/500/503/timeout/maintenance/recovery) with admin controls + UI; latency/jitter simulation; request logs.
**Exit criteria:** integration-gateway consumes it; admin toggles demonstrably change behavior.

## Phase 11 — Queues

**Goal:** Durable smart queues on Redis Streams.
**Tasks:** producer/consumer framework; 5 queues + consumer groups; Postgres `QueueItem` source-of-truth + reconciler; retry/backoff/DLQ; idempotent submission handler; tracking timeline (screen 10) + queue position; tests #2, #5, #6, #8.
**Exit criteria:** kill worker mid-processing → no loss; duplicate submit → single application.

## Phase 12 — Off-Peak Scheduler

**Goal:** Load-smoothed deferred processing.
**Tasks:** capacity monitor vs simulator; window planner (gradual release curve); deferred queue releaser with jitter + backpressure; citizen-facing "flexible processing" option with ETA; demo control for off-peak view.
**Exit criteria:** release curve matches configured percentages; backpressure pauses on threshold.

## Phase 13 — Infrastructure Simulator

**Goal:** Live infra model + dashboards.
**Tasks:** node model (CENTRAL_DC/STATE_DC/AWS_CLOUD) with CPU/RAM/latency/error/health; spike scenario engine (10K→150K); breaker + load-balancer integration; SSE streams; screen 12 (traffic chart, health, AI forecast + recommendations, scaling events); demo "SIMULATE TRAFFIC SPIKE".
**Exit criteria:** spike demo shows queueing + routing away from overloaded node; no lost applications (test #6).

## Phase 14 — Security

**Goal:** Full security control set + dashboards.
**Tasks:** sessions/CSRF/Argon2id/MFA-sim/step-up; rate limiting; security event pipeline; screen 13 (threat level, blocked requests, failed logins, injection attempts, suspicious IPs, anomalies, malicious docs, live events); 5 attack simulations; screen 14 admin control center (services, users, queues, policies, config, audit log viewer); screen 15 service details; audit hash chain; security tests.
**Exit criteria:** all §54 security demos behave as specified; audit chain verification tool passes.

## Phase 15 — AWS Terraform

**Goal:** Reproducible AWS foundation.
**Tasks:** modules per [terraform-architecture.md](terraform-architecture.md); dev environment apply; remote state with locking; ECR repos; KMS keys; Secrets Manager entries.
**Exit criteria:** `terraform plan/apply` clean on dev; no hardcoded IDs/secrets; Checkov findings triaged.

## Phase 16 — Docker

**Goal:** Production-grade images for all deployables.
**Tasks:** multi-stage builds (frontend, backend, ai-agent, document-service, worker, mock-government); non-root, slim/distroless bases; `.dockerignore`; health checks; Trivy scan in CI; compose parity with K8s env vars.
**Exit criteria:** images < agreed size budgets; scans pass; compose stack green.

## Phase 17 — Kubernetes/EKS

**Goal:** Deploy to EKS staging.
**Tasks:** manifests per [kubernetes-architecture.md](kubernetes-architecture.md); External Secrets; IRSA; ingress via AWS Load Balancer Controller; deploy via pipeline; smoke tests.
**Exit criteria:** staging serves e2e citizen flow; pod kill/AZ disruption drill passes.

## Phase 18 — Jenkins CI/CD

**Goal:** Full pipeline with security gates + manual prod approval.
**Tasks:** Jenkinsfile per [jenkins-cicd-architecture.md](jenkins-cicd-architecture.md); shared library; scan gates (Semgrep, Trivy, Gitleaks, Checkov, k8s-lint); ECR push; tf plan/apply staging; staging deploy; integration tests; approval gate; prod deploy.
**Exit criteria:** pipeline green end-to-end on staging; prod deploy blocked without approval.

## Phase 19 — Observability

**Goal:** Metrics, dashboards, alerts.
**Tasks:** Prometheus metrics per architecture §15; Grafana dashboards (traffic, queues, AI, integrations, infra, security); alert rules; CloudWatch log shipping; log-redaction verification in pipeline.
**Exit criteria:** dashboards reflect live simulator; alerts fire in demo scenarios.

## Phase 20 — Load & security testing

**Goal:** Evidence the system absorbs spikes and resists abuse.
**Tasks:** k6 suites (1K/10K/50K/100K VUs) against staging mocks; reports (throughput, p95/p99, errors, queue depth); security test suite (spec §50 list); chaos drills (worker kill, Redis restart, breaker open).
**Exit criteria:** zero lost applications across all load runs; all 10 critical tests green in CI.

## Phase 21 — Final demo polish

**Goal:** Demo mode + story rehearsal.
**Tasks:** demo-mode orchestrator (10 controls); [demo-script.md](demo-script.md) rehearsed end-to-end; seed data; performance polish; docs final pass; DoD checklist sign-off.
**Exit criteria:** full demo story runs in < 15 minutes with visible dashboard reactions.

## Testing strategy (summary)

| Layer | Tooling | Focus |
|---|---|---|
| Unit | pytest, vitest/jest | Schemas, validators, policy rules, redaction |
| Integration | pytest + compose | API + DB + Redis + mocks |
| Workflow/policy/AI-tool | pytest | §50 critical tests #1–#10 |
| E2E | Playwright | Citizen journey en/hi/mr |
| Load | k6 | 1K→100K VUs, spike curve |
| Security | Semgrep, Trivy, Gitleaks, Checkov, OWASP ZAP baseline | CI gates |

## Definition of Done tracking

The §58 checklist is maintained in this file's companion — [../README.md](../README.md) status section — and reviewed at the end of every phase; an item may only be checked when a test or demo demonstrates it.
