# GovFlow — Dependencies & Risk Register

| | |
|---|---|
| **Status** | Phase 1 |

## 1. Technology dependencies

| Dependency | Version target | Used by | License | Notes / pinning strategy |
|---|---|---|---|---|
| Node.js | 22 LTS | frontend | MIT | `.nvmrc` + Dockerfile arg |
| Next.js | 15.x | frontend | MIT | App Router, locked via lockfile |
| React | 19.x | frontend | MIT | |
| TypeScript | 5.x (strict) | frontend | Apache-2.0 | `strict: true`, no `any` escapes |
| Tailwind CSS | 4.x | frontend | MIT | Design tokens in config |
| Python | 3.12 | backend/ai-agent/document-service/worker/mock-government | PSF | `.python-version` + images |
| FastAPI | 0.11x | all Python services | MIT | Pydantic v2 everywhere |
| SQLAlchemy | 2.x async | backend | MIT | Async sessions, typed models |
| Alembic | 1.1x | backend | MIT | Migrations only via Alembic |
| PostgreSQL | 16 | data plane | PostgreSQL License | RDS in AWS; 16 features used: none exotic |
| Redis | 7.x | queues/cache | RSAL/SSPL → **use Valkey 7.x-compatible client** | ElastiCache-compatible; Streams + consumer groups |
| Tesseract OCR | 5.x | document-service | Apache-2.0 | Sandboxed, size/time limited |
| ClamAV | 1.x | document-service (scan hook) | GPL (sidecar container, not linked) | Optional compose profile |
| LLM SDK | provider SDK, abstracted | ai-agent | varies | **Provider-agnostic adapter**; mock provider for tests/local |
| k6 | 1.x/0.5x current | tests/load | AGPL (test tool only) | Not distributed in product |
| Terraform | >= 1.10 | infrastructure | BUSL | S3 native state locking (`use_lockfile`) |
| Jenkins LTS | current LTS | CI/CD | MIT | Pipeline-as-code |
| Docker/BuildKit | current | builds | Apache-2.0 | Multi-stage, non-root |
| Kubernetes | 1.3x (EKS default) | orchestration | Apache-2.0 | Managed node groups |
| AWS Load Balancer Controller, External Secrets Operator, Metrics Server, Karpenter/Cluster Autoscaler | current | EKS | Apache-2.0 | Add-ons via Terraform |
| Prometheus / Grafana | current LTS | observability | Apache-2.0/AGPL (Grafana) | Compose profile `monitoring` |

**Key risk on Redis licensing:** ElastiCache for Redis is an AWS managed service (no license issue); local dev may use Valkey or Redis 7 community builds. Abstracted via `REDIS_URL`.

## 2. External service dependencies

| Dependency | MVP approach | Production path |
|---|---|---|
| UMANG / state government APIs | **Mock only** (`mock-government`) | Certified integration gateway, MoU-bound; never direct |
| DigiLocker | Mock behind `DocumentProvider` interface | Official DigiLocker partner APIs with consent artifacts |
| LLM provider | API key via env (dev/staging), mock provider for tests | Enterprise agreement, data-processing addendum, no-training clause, PII guards |
| SMS/Email notifications | Simulated in-app | SNS/SES with templates + DLT compliance (India) |
| Payments (tatkal fee) | Simulated | Payment aggregator with PCI scope separation |
| ASR (voice) | Browser Web Speech API | Indic-language ASR service (Bhashini-class) |

## 3. Risk register

Scale: Likelihood (L) / Impact (I) — Low/Medium/High.

| # | Risk | L | I | Mitigation | Owner |
|---|---|---|---|---|---|
| R1 | LLM output instability breaks flows | M | M | Strict schema validation, bounded repair, deterministic fallback to manual forms; mock provider in tests | AI lead |
| R2 | Prompt-injection succeeds in demo | M | H | Layered defense (screening, schema, tool gating, egress deny); rehearsed attack demos prove the negative | Security |
| R3 | Scope creep beyond 3 services | H | M | Registry architecture makes "more services" cheap but **frozen** for MVP; change control via development plan | PM |
| R4 | Redis Streams data-loss edge cases | L | H | Postgres source-of-truth + reconciler; kill-tests in Phase 11; DLQ + alerting | Backend |
| R5 | Mock APIs unrealistic → demo not credible | M | M | Fault profiles calibrated to real patterns (429 bursts, 503 windows, latency tails); k6 against mocks | Backend |
| R6 | EKS/Terraform complexity delays MVP | M | M | Local compose is always sufficient for full demo; AWS tracks behind (Phases 15–18) with clear exit criteria | DevOps |
| R7 | CI security gates block delivery (false positives) | M | L | Severity-threshold policy (fail on critical/high only), suppression review process, baseline triage in Phase 18 | DevSecOps |
| R8 | Accessibility gaps for low-literacy users | M | H | WCAG AA targets from Phase 2; voice-first UX; guided flows; user testing with sample scripts | Frontend |
| R9 | Data-minimization violations (accidental PII storage/logging) | M | H | Encrypted field design, log allowlist, CI redaction tests, synthetic-data-only seed rule | Security |
| R10 | LLM cost overrun during load tests | M | M | Mock LLM provider for load runs; per-session token budgets; caching of registry answers | AI lead |
| R11 | Jenkins/ECR/EKS permission misconfiguration | L | H | Least-privilege IRSA roles, Checkov/IAM scans, no AdministratorAccess, CloudTrail alerts on IAM changes | DevOps |
| R12 | Queue backlog grows unbounded during spike demo | M | M | Admission control → DEFERRED with consent; capacity dashboards; autoscaling on queue lag | Backend |
| R13 | Single-region staging outage during demo | L | M | Demo rehearsed on compose fallback; staging IaC reprovisionable in < 1h | DevOps |
| R14 | Hindi/Marathi NLU quality gaps | M | M | Registry-keyed intent examples per language; suggested-response chips reduce free-text dependence; fallback clarifying questions | AI lead |
| R15 | Audit-chain implementation bugs (false tamper alarms) | L | M | Property tests on hash chain; chain-verification tooling in Phase 14 | Backend |
| R16 | Timeline pressure → security shortcuts | M | H | Phase exit criteria include security tests; "no phase breaks security architecture" working agreement; DoD gates | All |

## 4. Explicit non-dependencies (by design)

- No real UMANG/government API dependency — everything behind `mock-government`.
- No real identity data — synthetic only; DigiLocker is mocked.
- No Kubernetes/AWS required for local development — `docker compose up` is self-sufficient.
- No vendor lock on LLM provider — adapter interface + mock provider.
