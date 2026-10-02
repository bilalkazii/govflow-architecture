# GovFlow — API Contract Specification (v1)

| | |
|---|---|
| **Status** | Approved (Phase 1) |
| **Base path** | `/api/v1` |
| **Style** | REST + SSE for live dashboards · JSON only · OpenAPI auto-generated (FastAPI) |

## 1. Conventions

### 1.1 General
- **Content type:** `application/json; charset=utf-8` (uploads: `multipart/form-data`).
- **Timestamps:** UTC, ISO 8601 (`2026-08-24T10:15:00Z`).
- **IDs:** UUID v4; human-facing application numbers `GF-<STATE>-<6 digits>` (e.g., `GF-MH-847291`).
- **Auth:** session cookie `gf_session` (HttpOnly, Secure, SameSite=Lax) + `X-CSRF-Token` header on mutations. Admin routes require role `admin` (+ step-up for sensitive ops).
- **Idempotency:** consequential POSTs accept `Idempotency-Key` header; replay returns original result with `X-Idempotent-Replay: true`.
- **Correlation:** every request gets `X-Correlation-Id` (generated if absent) echoed in responses, errors, logs, and audit events.
- **Versioning:** `/api/v1` prefix; additive changes only within v1; breaking changes → v2 with 90-day overlap policy.

### 1.2 Rate limits (per session/IP)

| Route class | Limit |
|---|---|
| Auth (login/OTP) | 5/min, lockout after 5 consecutive failures |
| AI chat | 20/min per session |
| Uploads | 10/hour, 10 MB/file |
| Standard reads | 120/min |
| Submission request/confirm | 10/hour |
| Admin mutations | 60/min + step-up |

Headers on every response: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` on 429.

### 1.3 Error envelope

```json
{
  "error": {
    "code": "POLICY_DENIED",
    "message": "Submission requires your explicit confirmation.",
    "correlation_id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "details": { "operation": "request_submission", "required": "citizen_confirmation" }
  }
}
```

Stable codes: `VALIDATION_ERROR` (422), `UNAUTHENTICATED` (401), `FORBIDDEN` (403), `NOT_FOUND` (404), `RATE_LIMITED` (429), `IDEMPOTENCY_CONFLICT` (409), `POLICY_DENIED` (403), `STEP_UP_REQUIRED` (403), `INTEGRATION_UNAVAILABLE` (503), `QUEUE_BUSY` (503), `INTERNAL_ERROR` (500 — never exposes stack traces).

## 2. Auth & session

| Method | Path | Purpose | Notes |
|---|---|---|---|
| POST | `/auth/session` | Login (demo credentials) | Sets cookie; audit `USER_LOGIN` |
| POST | `/auth/otp/simulate` | MFA simulation (demo OTP) | Returns dev OTP in response **only** when `DEMO_MODE=true` |
| POST | `/auth/step-up` | Step-up auth for sensitive ops | Raises `auth_strength` |
| POST | `/auth/logout` | Revoke session | |
| GET | `/auth/me` | Current user profile | |

## 3. Service registry

| Method | Path | Purpose |
|---|---|---|
| GET | `/services` | List active services (filters: `jurisdiction`, `category`, `q`) |
| GET | `/services/{service_key}` | Service details (latest active version per jurisdiction) |
| GET | `/services/{service_key}/versions` | Version history |
| GET | `/services/{service_key}/requirements?jurisdiction=MH&version=1.0` | Documents, fields, fees, SLA, tatkal availability + remaining tatkal capacity |

## 4. AI agent

| Method | Path | Purpose |
|---|---|---|
| POST | `/agent/chat` | Send message; returns validated agent envelope |
| GET | `/agent/sessions/{session_id}` | Conversation history (owner only) |
| POST | `/agent/voice/transcribe` | Voice → text (MVP: accepts client-transcribed text or audio stub) |

**`POST /agent/chat`** request:

```json
{
  "session_id": "uuid-or-null",
  "message": "Mujhe income certificate banana hai",
  "language": "hi",
  "attachment_document_ids": []
}
```

Response (strict server-validated envelope — LLM output never passes through raw):

```json
{
  "session_id": "…",
  "reply": "Sure. I can help you apply for an Income Certificate. Which state are you applying in?",
  "language": "hi",
  "intent": "apply_service",
  "candidate_service": { "service_key": "income_certificate", "confidence": 0.93 },
  "suggested_responses": ["Maharashtra", "Madhya Pradesh", "Delhi"],
  "ui_action": { "type": "service_identification", "payload": { "service_key": "income_certificate", "jurisdiction": "MH" } },
  "requires_confirmation": false,
  "security_flags": []
}
```

`ui_action` types: `service_identification`, `show_requirements`, `start_document_flow`, `open_form`, `open_review`, `show_status`, `none`. `security_flags` non-empty → UI shows safe error; tool trace is audited server-side (never exposed raw).

## 5. Documents

| Method | Path | Purpose |
|---|---|---|
| POST | `/documents/upload` | Multipart upload → validation pipeline |
| GET | `/documents/{id}` | Metadata + scan/extraction status (owner only) |
| POST | `/documents/{id}/analyze` | Trigger OCR + extraction |
| GET | `/documents/{id}/extractions` | Extracted fields + confidence + validation |
| POST | `/documents/digilocker/consent` | Start mock consent flow → `consent_id` |
| GET | `/documents/digilocker/documents?consent_id=` | Mock available documents |
| POST | `/documents/digilocker/fetch` | Fetch selected mock document |

Upload response: `{ "id", "scan_status": "pending|clean|infected|error", "quarantined": false, "status": "uploaded" }`. Infected/suspicious → `409 DOCUMENT_REJECTED` + SecurityEvent; content never analyzed.

## 6. Applications

| Method | Path | Purpose |
|---|---|---|
| POST | `/applications` | Create draft (`Idempotency-Key` required) |
| GET | `/applications` | List own (filters: `state`, `service_key`) |
| GET | `/applications/{id}` | Full detail incl. fields + documents |
| PATCH | `/applications/{id}` | Update editable fields (draft states only) |
| POST | `/applications/{id}/validate` | Run registry validation → issues list |
| POST | `/applications/{id}/processing-options` | Choose standard/priority/deferred/tatkal (tatkal subject to capacity) |
| POST | `/applications/{id}/submission-request` | **Policy-gated**: returns decision + confirmation token (no submission) |
| POST | `/applications/{id}/confirm` | Citizen confirms with token → enqueues |
| GET | `/applications/{id}/status` | Timeline + queue position + ETA |

**`submission-request`** response (decision paths):

```json
{ "decision": "REQUIRE_CONFIRMATION",
  "confirmation_token": "cvt_…",
  "summary": { "service": "Income Certificate", "jurisdiction": "Maharashtra",
               "fee": { "currency": "INR", "amount": 50 }, "processing_class": "standard",
               "estimated_days": 15, "documents": ["identity", "address_proof", "income_proof"] } }
```

`DENY` → `403 POLICY_DENIED` with citizen-safe reason. **There is no API path that submits without a prior `submission-request` + `confirm` carrying a valid confirmation token bound to the exact application payload hash.**

**`confirm`** response: `{ "application_number": "GF-MH-847291", "queue": "standard", "state": "QUEUED", "position": 42, "eta": "…" }`.

**`GET /applications/{id}/status`** response:

```json
{ "state": "SUBMITTED",
  "timeline": [
    { "stage": "received", "at": "…", "done": true },
    { "stage": "documents_verified", "at": "…", "done": true },
    { "stage": "application_prepared", "at": "…", "done": true },
    { "stage": "submitted", "at": "…", "done": true },
    { "stage": "government_processing", "at": null, "done": false },
    { "stage": "completed", "at": null, "done": false } ],
  "queue": { "name": "standard", "position": 12 },
  "estimated_completion": "2026-09-05" }
```

## 7. Notifications

| Method | Path | Purpose |
|---|---|---|
| GET | `/notifications` | List own |
| POST | `/notifications/{id}/read` | Mark read |

## 8. Admin (role=admin; all mutations audited)

| Method | Path | Purpose |
|---|---|---|
| GET | `/admin/overview` | System summary |
| GET/PATCH | `/admin/services/{service_key}` | Registry admin (tatkal toggle, capacity, fees) |
| GET | `/admin/queues` | Depth, age, per-queue stats |
| POST | `/admin/queues/{queue}/pause` · `/resume` · `/drain` | Queue ops |
| GET | `/admin/infrastructure` | Node metrics + health |
| POST | `/admin/infrastructure/nodes/{key}/state` | Set `healthy/slow/overloaded/failed/recovering` |
| POST | `/admin/infrastructure/simulate-spike` | `{ "target_rps": 100000, "ramp_minutes": 5 }` |
| GET | `/admin/integrations` · POST `/admin/integrations/{name}/health` | Mock gov health: `healthy/slow/overloaded/maintenance/failure/recover` |
| GET | `/admin/security/events?severity=&type=` | Security events |
| POST | `/admin/security/simulate` | `{ "scenario": "prompt_injection\|api_flood\|malicious_document\|compromised_service\|failed_auth" }` |
| GET | `/admin/audit?actor=&action=&from=&to=` | Audit log viewer |
| GET | `/admin/workflows/active` | Active workflow executions |
| POST | `/admin/demo/scenario` | Demo-mode orchestrator (see demo-script.md) |

## 9. Mock government gateway (service `mock-government`, port 8003)

| Method | Path | Purpose |
|---|---|---|
| POST | `/mock/umang/income` | Income certificate submission |
| POST | `/mock/umang/birth` | Birth certificate submission |
| POST | `/mock/umang/driving` | Driving licence submission/renewal |
| GET | `/mock/umang/applications/{id}` | Status lookup |
| GET | `/mock/umang/admin/health` · POST `/mock/umang/admin/health` | Fault-injection control |
| GET | `/mock/umang/admin/stats` | Call counts, latency profile, failure log |

Behavior driven by admin health profile: `healthy` (p50 200ms), `slow` (+2–8s), `overloaded` (429 bursts + 503 tails), `maintenance` (503 + `Retry-After`), `failure` (500s), `recover` (gradual success ramp). Responses are deterministic and schema-validated by the integration gateway.

## 10. Real-time streams (SSE)

| Path | Events |
|---|---|
| `/api/v1/streams/infrastructure` | `metrics`, `node_health`, `scaling_event`, `breaker_state`, `forecast` |
| `/api/v1/streams/security` | `security_event`, `threat_level` |
| `/api/v1/streams/applications/{id}` | `state_change`, `queue_position` |

Auth: session cookie; admin streams require admin role. Heartbeat every 15s; clients fall back to polling GETs on disconnect.

## 11. Health & metrics

`GET /healthz` (liveness) · `GET /readyz` (readiness: DB, Redis, integrations) · `GET /metrics` (Prometheus, internal-only).
