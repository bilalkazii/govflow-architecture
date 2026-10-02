# GovFlow — Database Design (PostgreSQL)

| | |
|---|---|
| **Status** | Approved (Phase 1) |
| **Engine** | PostgreSQL 16 · UUID primary keys · SQLAlchemy 2 async · Alembic migrations |
| **Principle** | **Data minimization** — GovFlow stores the minimum required for the MVP. It is *not* a centralized citizen-record store. |

## 1. Entity-Relationship Diagram

```mermaid
erDiagram
    USER ||--o{ SESSION : "authenticates via"
    USER ||--o{ APPLICATION : "creates"
    USER ||--o{ NOTIFICATION : "receives"
    USER ||--o{ DOCUMENT : "owns"
    SERVICE ||--o{ SERVICE_VERSION : "versioned as"
    SERVICE_VERSION ||--o{ APPLICATION : "instantiates"
    APPLICATION ||--o{ APPLICATION_FIELD : "has"
    APPLICATION ||--o{ DOCUMENT : "attaches"
    APPLICATION ||--o{ QUEUE_ITEM : "queued as"
    APPLICATION ||--o{ WORKFLOW_EXECUTION : "governed by"
    APPLICATION ||--o{ POLICY_DECISION : "gated by"
    APPLICATION ||--o{ AUDIT_EVENT : "audited"
    DOCUMENT ||--o{ DOCUMENT_EXTRACTION : "yields"
    INFRASTRUCTURE_NODE ||--o{ SECURITY_EVENT : "context (nullable)"

    USER {
        uuid id PK
        string email UK
        string phone
        string full_name
        string password_hash
        string locale
        string role
        bool mfa_enabled
        datetime last_login_at
        datetime created_at
    }
    SESSION {
        uuid id PK
        uuid user_id FK
        string token_hash
        string csrf_token_hash
        string ip_hash
        string user_agent
        string auth_strength
        datetime expires_at
        datetime revoked_at
        datetime created_at
    }
    SERVICE {
        uuid id PK
        string service_key UK
        string name
        string category
        text description
        bool active
        datetime created_at
    }
    SERVICE_VERSION {
        uuid id PK
        uuid service_id FK
        string version
        string jurisdiction
        jsonb eligibility_rules
        jsonb required_documents
        jsonb required_fields
        jsonb fee
        bool tatkal_supported
        int tatkal_daily_cap
        jsonb tatkal_fee
        string processing_class
        jsonb sla
        string workflow_key
        string mock_endpoint
        string status
        datetime effective_from
    }
    APPLICATION {
        uuid id PK
        string application_number UK
        uuid user_id FK
        uuid service_version_id FK
        string state
        string purpose
        string jurisdiction
        jsonb form_data_encrypted
        string processing_class
        decimal fee_amount
        string idempotency_key UK
        string confirmation_token_hash
        datetime confirmed_at
        datetime submitted_at
        datetime completed_at
        text rejection_reason
        string correlation_id
        datetime created_at
        datetime updated_at
    }
    APPLICATION_FIELD {
        uuid id PK
        uuid application_id FK
        string field_key
        text value_encrypted
        string source
        decimal confidence
        bool validated
        datetime updated_at
    }
    DOCUMENT {
        uuid id PK
        uuid application_id FK
        uuid user_id FK
        string doc_type
        string source
        string filename
        string mime
        int size_bytes
        string s3_key
        string checksum_sha256
        string scan_status
        bool quarantined
        string status
        datetime created_at
    }
    DOCUMENT_EXTRACTION {
        uuid id PK
        uuid document_id FK
        string field_key
        text value_encrypted
        decimal confidence
        jsonb bbox
        string validated_by
        datetime created_at
    }
    QUEUE_ITEM {
        uuid id PK
        uuid application_id FK
        string queue
        string state
        int priority
        int attempts
        int max_attempts
        datetime available_at
        string locked_by
        datetime locked_at
        string idempotency_key UK
        text last_error
        datetime scheduled_window
        datetime created_at
        datetime updated_at
    }
    WORKFLOW_EXECUTION {
        uuid id PK
        uuid application_id FK
        string workflow_key
        string current_state
        jsonb context
        int definition_version
        datetime started_at
        datetime updated_at
        datetime completed_at
    }
    POLICY_DECISION {
        uuid id PK
        uuid application_id FK
        uuid actor_user_id FK
        string operation
        string decision
        text reason
        jsonb input_snapshot
        jsonb rule_hits
        float risk_score
        datetime created_at
    }
    AUDIT_EVENT {
        uuid id PK
        datetime occurred_at
        uuid actor_id
        string actor_type
        string action
        string resource_type
        string resource_id
        string service_key
        uuid application_id
        string result
        string correlation_id
        string ip_hash
        jsonb metadata
        string prev_hash
        string entry_hash
    }
    NOTIFICATION {
        uuid id PK
        uuid user_id FK
        string channel
        string template
        jsonb payload
        string status
        datetime sent_at
        datetime created_at
    }
    INFRASTRUCTURE_NODE {
        uuid id PK
        string node_key UK
        string region
        int capacity_rps
        float cpu
        float memory
        float latency_ms
        float error_rate
        string health
        string state
        datetime updated_at
    }
    SECURITY_EVENT {
        uuid id PK
        string event_type
        string severity
        string source_ip_hash
        uuid user_id FK
        text description
        jsonb indicators
        string action_taken
        string correlation_id
        datetime created_at
    }
```

## 2. Design notes

### 2.1 Data minimization (enforced)
- **No Aadhaar/PAN/real identity numbers.** Mock documents use synthetic placeholders (`DEMO-XXXX`); the schema has no column for government identity numbers by design.
- `form_data_encrypted` / `value_encrypted` — application-layer AES-GCM encryption for sensitive values (production: KMS-wrapped data key; dev: key from env).
- `ip_hash` — IPs stored as salted hashes, never raw.
- Retention (MVP defaults): documents 30 days after completion; extractions 30 days; sessions 7 days post-expiry; audit events retained (hash-chained, append-only).

### 2.2 Key constraints & indexes
| Table | Constraint / Index | Purpose |
|---|---|---|
| `applications` | `UNIQUE(idempotency_key)` | Duplicate-submission defense |
| `applications` | `UNIQUE(application_number)` | Human-readable ID `GF-MH-847291` |
| `applications` | `INDEX(user_id, created_at DESC)` | My Applications screen |
| `service_versions` | `UNIQUE(service_id, jurisdiction, version)` | Versioned registry |
| `application_fields` | `UNIQUE(application_id, field_key)` | One value per field |
| `queue_items` | `UNIQUE(idempotency_key)`, `INDEX(state, queue, available_at)` | Claim queries |
| `audit_events` | `prev_hash → entry_hash` chain | Tamper evidence |
| `sessions` | `INDEX(expires_at)` | Cleanup job |

### 2.3 Enumerations (enforced via SQLAlchemy Enum + DB CHECK)
- `role`: `citizen | admin | auditor`
- `auth_strength`: `password | otp_sim | step_up`
- `state` (application/queue): `RECEIVED | VALIDATING | QUEUED | PROCESSING | SUBMITTED | COMPLETED | FAILED | HUMAN_REVIEW`
- `source` (field/document): `ai_extracted | user_entered | verified | digilocker | upload`
- `decision`: `ALLOW | DENY | REQUIRE_CONFIRMATION | REQUIRE_STEP_UP_AUTH | QUEUE`
- `queue`: `realtime | standard | priority | deferred | tatkal`
- `health`: `GREEN | YELLOW | RED`; node `state`: `healthy | slow | overloaded | failed | recovering`
- `action_taken` (security): `logged | blocked | quarantined | isolated | alerted`

### 2.4 Audit hash chain
`entry_hash = SHA-256(prev_hash || canonical_json(event_without_hash_fields))`. A verification job walks the chain; any mismatch raises a critical SecurityEvent. Writes are insert-only (no UPDATE/DELETE grants for the app role).

### 2.5 Migration policy
- All schema changes via Alembic; autogenerate reviewed by hand; no destructive migration without a two-step (deprecate → drop) path.
- Seed data (`scripts/seed.py`): demo user, 3 service versions, infrastructure nodes, sample applications for dashboards.
