# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Open Banking Platform · Created: 2026-05-20

## Philosophy

This model treats every state change as an immutable event. The event store is the single source of truth; all queryable state (accounts, balances, consent status) is derived by replaying or projecting events into materialised read models. The write path appends events; the read path queries denormalised projections optimised for specific access patterns.

This approach is a natural fit for open banking, where PSD2 mandates immutable audit trails for consent lifecycle changes, payment status transitions, and data access records. Event sourcing does not bolt auditing onto a mutable schema — auditing IS the schema. Every consent grant, every payment status change, every balance query is a recorded fact that can be replayed to reconstruct the exact system state at any point in time.

Financial systems like LMAX Exchange and major payment processors have proven this pattern at scale. The Berlin Group NextGenPSD2 specification itself describes consent and payment flows as sequences of state transitions — a natural mapping to event streams. Combined with CQRS (Command Query Responsibility Segregation), the write-heavy compliance path and the read-heavy API serving path can be scaled independently.

**Best for:** Teams prioritising regulatory compliance, bi-temporal querying ("what was the consent state on date X?"), and AI-powered analytics on behavioural patterns.

**Trade-offs:**
- (+) Complete, immutable audit trail — every state change is a permanent record
- (+) Bi-temporal queries: reconstruct system state at any historical point
- (+) Write and read paths scale independently (CQRS)
- (+) Natural fit for fraud detection ML — event streams are training data
- (+) No data loss from UPDATE/DELETE operations
- (-) Higher implementation complexity — requires event store, projections, and eventual consistency handling
- (-) Read model rebuilds can be slow for large event volumes
- (-) Eventual consistency between write (events) and read (projections) requires careful UX design
- (-) Debugging requires understanding event replay, not just "look at the row"
- (-) Schema evolution of event types requires versioning strategy

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Berlin Group NextGenPSD2 | Consent and payment state transitions map directly to event types; NextGenPSD2 status values become event payloads |
| PSD2 RTS on SCA | SCA authentication steps recorded as individual events with factor-type metadata |
| ISO 20022 | Payment status codes (pacs.002) map to payment status change events; structured remittance preserved in event payloads |
| FAPI 2.0 / OAuth 2.0 | Token issuance, refresh, and revocation recorded as security events |
| GDPR | Right-to-erasure implemented via crypto-shredding — PII encrypted with per-user keys that can be destroyed |
| FDX v6.5 | Account and transaction data in read projections aligns with FDX resource definitions |

---

## Event Store (Core — Write Side)

```sql
-- =============================================================
-- EVENT STORE — The single source of truth
-- =============================================================

CREATE TABLE events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id       UUID NOT NULL,                               -- aggregate ID (consent, payment, account, etc.)
    stream_type     VARCHAR(50) NOT NULL,                        -- 'consent', 'payment', 'account', 'institution', 'user'
    event_type      VARCHAR(100) NOT NULL,                       -- e.g., 'ConsentRequested', 'PaymentInitiated', 'BalanceFetched'
    event_version   INTEGER NOT NULL,                            -- schema version for this event type
    sequence_number BIGINT NOT NULL,                             -- ordering within the stream
    tenant_id       UUID NOT NULL,
    -- Event payload
    data            JSONB NOT NULL,                              -- the event payload (domain-specific)
    metadata        JSONB NOT NULL DEFAULT '{}',                 -- correlation IDs, causation, actor info
    -- Timestamps
    occurred_at     TIMESTAMPTZ NOT NULL,                        -- when the domain event actually happened
    recorded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),          -- when it was persisted
    -- Optimistic concurrency
    UNIQUE (stream_id, sequence_number)
);

-- Primary query: replay a single stream
CREATE INDEX idx_events_stream ON events (stream_id, sequence_number);
-- Category queries: all events of a type
CREATE INDEX idx_events_type ON events (stream_type, event_type, recorded_at);
-- Tenant isolation
CREATE INDEX idx_events_tenant ON events (tenant_id, recorded_at);
-- Global ordering for projections
CREATE INDEX idx_events_recorded ON events (recorded_at);

-- Partition by month for performance at scale
-- CREATE TABLE events_2026_05 PARTITION OF events
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- =============================================================
-- EVENT TYPE REGISTRY — Schema evolution tracking
-- =============================================================

CREATE TABLE event_type_registry (
    event_type      VARCHAR(100) NOT NULL,
    version         INTEGER NOT NULL,
    json_schema     JSONB NOT NULL,                              -- JSON Schema for this version
    description     TEXT,
    deprecated      BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (event_type, version)
);

-- =============================================================
-- STREAM SNAPSHOTS — Performance optimisation for long streams
-- =============================================================

CREATE TABLE stream_snapshots (
    stream_id       UUID NOT NULL,
    stream_type     VARCHAR(50) NOT NULL,
    sequence_number BIGINT NOT NULL,                             -- snapshot taken at this sequence
    state           JSONB NOT NULL,                              -- serialised aggregate state
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (stream_id, sequence_number)
);
```

### Example Event Payloads

```sql
-- ConsentRequested event
-- data: {
--   "consent_type": "ais",
--   "tpp_client_id": "uuid-...",
--   "psu_id": "uuid-...",
--   "access": {
--     "accounts": ["uuid-1", "uuid-2"],
--     "balances": ["uuid-1"],
--     "transactions": ["uuid-1", "uuid-2"]
--   },
--   "recurring_indicator": true,
--   "frequency_per_day": 4,
--   "valid_until": "2026-11-20"
-- }
-- metadata: {
--   "correlation_id": "req-uuid-...",
--   "actor_type": "tpp",
--   "actor_id": "uuid-...",
--   "ip_address": "192.168.1.1",
--   "user_agent": "TPPApp/2.0"
-- }

-- PaymentStatusChanged event
-- data: {
--   "old_status": "received",
--   "new_status": "acsc",
--   "iso20022_code": "ACSC",
--   "aspsp_payment_id": "BANK-PAY-12345",
--   "reason": null
-- }

-- SCAFactorCompleted event
-- data: {
--   "factor_type": "possession",
--   "method": "mobile_push",
--   "device_id": "device-uuid-...",
--   "success": true
-- }

-- TransactionEnriched event
-- data: {
--   "transaction_id": "uuid-...",
--   "category": "groceries",
--   "merchant_name": "Tesco",
--   "merchant_mcc": "5411",
--   "confidence": 0.94,
--   "model_version": "enrichment-v3.2"
-- }
```

## Tenant & Identity (Read Projections)

```sql
-- =============================================================
-- READ PROJECTIONS — Materialised views rebuilt from events
-- =============================================================

-- All projection tables include a checkpoint column to track
-- which event they've processed up to.

CREATE TABLE projection_checkpoints (
    projection_name VARCHAR(100) PRIMARY KEY,
    last_event_id   UUID NOT NULL,
    last_recorded_at TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- TENANT PROJECTION
-- =============================================================

CREATE TABLE proj_tenants (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tenant_type     VARCHAR(50) NOT NULL,
    jurisdiction    CHAR(2) NOT NULL,
    regulatory_status VARCHAR(50),
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- =============================================================
-- USER PROJECTION
-- =============================================================

CREATE TABLE proj_users (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    email           VARCHAR(255) NOT NULL,
    full_name       VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    mfa_enabled     BOOLEAN DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_users_tenant ON proj_users (tenant_id);
```

## Consent Projection

```sql
-- =============================================================
-- CONSENT PROJECTION (derived from consent event stream)
-- =============================================================

CREATE TABLE proj_consents (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    tpp_client_id       UUID NOT NULL,
    psu_id              UUID,
    consent_type        VARCHAR(30) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    access_accounts     UUID[],
    access_balances     UUID[],
    access_transactions UUID[],
    recurring_indicator BOOLEAN,
    frequency_per_day   INTEGER,
    valid_until         DATE,
    sca_completed       BOOLEAN DEFAULT FALSE,
    status_change_count INTEGER DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_consents_tenant ON proj_consents (tenant_id);
CREATE INDEX idx_proj_consents_tpp ON proj_consents (tpp_client_id);
CREATE INDEX idx_proj_consents_psu ON proj_consents (psu_id);
CREATE INDEX idx_proj_consents_status ON proj_consents (status);
CREATE INDEX idx_proj_consents_expiry ON proj_consents (valid_until);

-- =============================================================
-- CONSENT TIMELINE — Flattened view of consent history for dashboards
-- =============================================================

CREATE TABLE proj_consent_timeline (
    id              UUID PRIMARY KEY,
    consent_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    event_type      VARCHAR(100) NOT NULL,
    status          VARCHAR(30),
    actor_type      VARCHAR(50),
    actor_id        UUID,
    details         JSONB,
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_consent_timeline_consent ON proj_consent_timeline (consent_id, occurred_at);
CREATE INDEX idx_proj_consent_timeline_tenant ON proj_consent_timeline (tenant_id, occurred_at);
```

## Account & Transaction Projections

```sql
-- =============================================================
-- ACCOUNT PROJECTION
-- =============================================================

CREATE TABLE proj_accounts (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    institution_id  UUID NOT NULL,
    psu_id          UUID NOT NULL,
    iban            VARCHAR(34),
    account_type    VARCHAR(30) NOT NULL,
    currency        CHAR(3) NOT NULL,
    name            VARCHAR(255),
    owner_name      VARCHAR(255),
    status          VARCHAR(20) DEFAULT 'enabled',
    current_balance DECIMAL(18, 4),                              -- denormalised for fast reads
    balance_currency CHAR(3),
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_accounts_tenant ON proj_accounts (tenant_id);
CREATE INDEX idx_proj_accounts_psu ON proj_accounts (psu_id);

-- =============================================================
-- TRANSACTION PROJECTION
-- =============================================================

CREATE TABLE proj_transactions (
    id                      UUID PRIMARY KEY,
    account_id              UUID NOT NULL,
    tenant_id               UUID NOT NULL,
    amount                  DECIMAL(18, 4) NOT NULL,
    currency                CHAR(3) NOT NULL,
    booking_date            DATE,
    value_date              DATE,
    creditor_name           VARCHAR(255),
    debtor_name             VARCHAR(255),
    remittance_unstructured TEXT,
    transaction_status      VARCHAR(20) NOT NULL,
    -- Enrichment (projected from TransactionEnriched events)
    category                VARCHAR(100),
    merchant_name           VARCHAR(255),
    merchant_mcc            VARCHAR(4),
    enrichment_confidence   DECIMAL(3, 2),
    created_at              TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_tx_account ON proj_transactions (account_id, booking_date);
CREATE INDEX idx_proj_tx_tenant ON proj_transactions (tenant_id);
CREATE INDEX idx_proj_tx_category ON proj_transactions (category);
```

## Payment Projection

```sql
-- =============================================================
-- PAYMENT PROJECTION
-- =============================================================

CREATE TABLE proj_payments (
    id                  UUID PRIMARY KEY,
    tenant_id           UUID NOT NULL,
    consent_id          UUID NOT NULL,
    payment_type        VARCHAR(30) NOT NULL,
    payment_product     VARCHAR(50) NOT NULL,
    creditor_name       VARCHAR(255) NOT NULL,
    creditor_iban       VARCHAR(34),
    amount              DECIMAL(18, 4) NOT NULL,
    currency            CHAR(3) NOT NULL,
    status              VARCHAR(30) NOT NULL,
    aspsp_payment_id    VARCHAR(100),
    status_transitions  INTEGER DEFAULT 0,                      -- count of status changes
    created_at          TIMESTAMPTZ NOT NULL,
    updated_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_payments_tenant ON proj_payments (tenant_id);
CREATE INDEX idx_proj_payments_status ON proj_payments (status);
CREATE INDEX idx_proj_payments_created ON proj_payments (created_at);

-- =============================================================
-- PAYMENT STATUS TIMELINE — for status tracking dashboards
-- =============================================================

CREATE TABLE proj_payment_timeline (
    id              UUID PRIMARY KEY,
    payment_id      UUID NOT NULL,
    tenant_id       UUID NOT NULL,
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    iso20022_code   VARCHAR(4),
    reason          TEXT,
    occurred_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_proj_pay_timeline ON proj_payment_timeline (payment_id, occurred_at);
```

## Institution & Analytics Projections

```sql
-- =============================================================
-- INSTITUTION PROJECTION
-- =============================================================

CREATE TABLE proj_institutions (
    id              UUID PRIMARY KEY,
    name            VARCHAR(255) NOT NULL,
    bic             VARCHAR(11),
    country_code    CHAR(2) NOT NULL,
    api_standard    VARCHAR(30) NOT NULL,
    supports_ais    BOOLEAN DEFAULT TRUE,
    supports_pis    BOOLEAN DEFAULT FALSE,
    health_status   VARCHAR(20) DEFAULT 'healthy',
    avg_response_ms INTEGER,                                    -- rolling average
    uptime_pct      DECIMAL(5, 2),                              -- rolling 30-day uptime
    last_health_check TIMESTAMPTZ,
    updated_at      TIMESTAMPTZ NOT NULL
);

-- =============================================================
-- ANALYTICS: Daily aggregation projection for dashboards
-- =============================================================

CREATE TABLE proj_daily_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    metric_date     DATE NOT NULL,
    total_api_calls BIGINT DEFAULT 0,
    total_consents_created INTEGER DEFAULT 0,
    total_consents_revoked INTEGER DEFAULT 0,
    total_payments_initiated INTEGER DEFAULT 0,
    total_payments_completed INTEGER DEFAULT 0,
    total_payments_failed INTEGER DEFAULT 0,
    total_transactions_fetched BIGINT DEFAULT 0,
    avg_api_latency_ms DECIMAL(8, 2),
    UNIQUE (tenant_id, metric_date)
);

CREATE INDEX idx_proj_metrics_tenant_date ON proj_daily_metrics (tenant_id, metric_date);
```

## Infrastructure Tables

```sql
-- =============================================================
-- OAUTH & SECURITY (operational — not event-sourced)
-- =============================================================

CREATE TABLE oauth_clients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL,
    client_id           VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash  VARCHAR(255),
    client_name         VARCHAR(255) NOT NULL,
    redirect_uris       TEXT[] NOT NULL,
    allowed_scopes      TEXT[] NOT NULL,
    token_endpoint_auth VARCHAR(50) DEFAULT 'private_key_jwt',
    jwks_uri            TEXT,
    tpp_role            VARCHAR(50),
    regulatory_id       VARCHAR(100),
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tokens are short-lived operational data, not event-sourced
CREATE TABLE oauth_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES oauth_clients(id),
    user_id         UUID,
    token_type      VARCHAR(20) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_oauth_tokens_expires ON oauth_tokens (expires_at);

-- =============================================================
-- WEBHOOK SUBSCRIPTIONS & DELIVERY (operational)
-- =============================================================

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    event_types     TEXT[] NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_id        UUID NOT NULL,                               -- references events.id
    event_type      VARCHAR(100) NOT NULL,
    http_status     INTEGER,
    attempt_number  SMALLINT NOT NULL DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- GDPR: CRYPTO-SHREDDING KEY STORE
-- =============================================================

CREATE TABLE encryption_keys (
    subject_id      UUID PRIMARY KEY,                            -- user/PSU ID
    tenant_id       UUID NOT NULL,
    key_encrypted   BYTEA NOT NULL,                              -- AES key encrypted with master key
    key_version     INTEGER NOT NULL DEFAULT 1,
    is_shredded     BOOLEAN DEFAULT FALSE,                       -- TRUE = right-to-erasure exercised
    shredded_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Key Query Patterns

```sql
-- Replay consent stream to reconstruct current state
SELECT event_type, data, occurred_at
FROM events
WHERE stream_id = 'consent-uuid-here'
  AND stream_type = 'consent'
ORDER BY sequence_number ASC;

-- Bi-temporal query: "What was the consent status at 2026-03-15T14:00:00Z?"
SELECT data->>'new_status' AS status, occurred_at
FROM events
WHERE stream_id = 'consent-uuid-here'
  AND stream_type = 'consent'
  AND event_type IN ('ConsentRequested', 'ConsentAuthorised', 'ConsentRevoked', 'ConsentExpired')
  AND occurred_at <= '2026-03-15T14:00:00Z'
ORDER BY sequence_number DESC
LIMIT 1;

-- Fraud signal: find all PSUs who had consents revoked more than 3 times in 30 days
SELECT data->>'psu_id' AS psu_id, COUNT(*) AS revocation_count
FROM events
WHERE stream_type = 'consent'
  AND event_type = 'ConsentRevoked'
  AND occurred_at >= NOW() - INTERVAL '30 days'
GROUP BY data->>'psu_id'
HAVING COUNT(*) > 3;

-- Rebuild a projection from scratch (disaster recovery)
-- SELECT * FROM events
-- WHERE stream_type = 'payment'
-- ORDER BY recorded_at ASC;
-- Then process each event through the projection handler.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 3 | events, event_type_registry, stream_snapshots |
| Projection Infrastructure | 1 | projection_checkpoints |
| Tenant/User Projections | 2 | proj_tenants, proj_users |
| Consent Projections | 2 | proj_consents, proj_consent_timeline |
| Account/Transaction Projections | 2 | proj_accounts, proj_transactions |
| Payment Projections | 2 | proj_payments, proj_payment_timeline |
| Institution/Analytics Projections | 2 | proj_institutions, proj_daily_metrics |
| OAuth & Security (operational) | 2 | oauth_clients, oauth_tokens |
| Webhooks (operational) | 2 | webhook_subscriptions, webhook_deliveries |
| GDPR (operational) | 1 | encryption_keys |
| **Total** | **19** | 3 core + 10 projections + 6 operational |

---

## Key Design Decisions

1. **Single `events` table with `stream_type` discriminator** — rather than separate event tables per aggregate, a single table enables cross-aggregate queries (e.g., "show me all events for this tenant in the last hour") while stream-specific replay uses the `(stream_id, sequence_number)` index.

2. **`occurred_at` vs `recorded_at` separation** — `occurred_at` is the business timestamp (when the consent was actually granted); `recorded_at` is the persistence timestamp (when the event was written). This bi-temporal design supports regulatory queries about what happened when.

3. **Event versioning via `event_version` and registry** — as the platform evolves, event schemas change. The `event_type_registry` stores JSON Schema definitions per version, enabling forward-compatible event processing without breaking existing projections.

4. **Snapshots for long-lived aggregates** — consents and accounts can accumulate thousands of events over their lifetime. Snapshots store the computed state at a point, so replay starts from the snapshot rather than from event zero.

5. **Projections are disposable** — every `proj_*` table can be dropped and rebuilt from the event store. This means schema changes to read models are risk-free: deploy the new projection, rebuild it, swap traffic.

6. **Crypto-shredding for GDPR** — PII in event payloads is encrypted with per-user keys stored in `encryption_keys`. When a user exercises their right to erasure, the key is destroyed (`is_shredded = TRUE`), rendering the PII in all their events unrecoverable without deleting the events themselves.

7. **OAuth tokens are NOT event-sourced** — tokens are short-lived operational data with no audit value beyond their creation. Event-sourcing them would generate noise without regulatory benefit.

8. **Webhook deliveries reference event IDs** — each delivery links back to the source event, enabling full traceability from business event to webhook notification to delivery status.

9. **Daily metrics projection for dashboards** — rather than running expensive aggregation queries on the event store, a dedicated projection pre-computes daily metrics, enabling instant dashboard rendering.

10. **Partition-ready event store** — the events table is designed for PostgreSQL declarative partitioning by `recorded_at`, essential when the platform handles billions of API calls generating millions of events per month.
