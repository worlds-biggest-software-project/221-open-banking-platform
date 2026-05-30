# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Open Banking Platform · Created: 2026-05-20

## Philosophy

This model follows classical relational database design: every domain concept gets its own table, relationships are expressed through foreign keys, and data integrity is enforced at the database level. Every field has a defined type and constraint. There is no ambiguity about what data exists or where it lives.

This approach mirrors how the Berlin Group NextGenPSD2 specification structures its own data model — with distinct entities for accounts, balances, transactions, consents, and payment initiations, each with well-defined fields and relationships. Regulatory bodies and auditors understand normalized schemas intuitively, and the schema itself serves as living documentation of the platform's data contracts.

The trade-off is rigidity: adding jurisdiction-specific fields or new financial product types requires schema migrations. But for a platform where regulatory compliance is the primary concern and data integrity failures can trigger fines, this rigidity is a feature, not a bug.

**Best for:** Teams building a compliance-first platform where data integrity, auditability, and standards alignment outweigh schema flexibility.

**Trade-offs:**
- (+) Maximum data integrity — constraints enforced at the database level
- (+) Clear, auditable schema that maps directly to regulatory specifications
- (+) Straightforward query patterns — no JSONB parsing or event replay
- (+) Excellent tooling support (ORMs, migration tools, reporting)
- (-) Schema migrations required for every new field or jurisdiction-specific variation
- (-) Higher table count increases join complexity for cross-cutting queries
- (-) Less flexible for rapid prototyping or multi-jurisdiction differences
- (-) Junction tables proliferate as many-to-many relationships grow

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Berlin Group NextGenPSD2 | Direct mapping of consent, account, transaction, and payment entities to NextGenPSD2 data model definitions |
| FDX v6.5 | Account and transaction field names aligned with FDX API resource definitions for North American compatibility |
| ISO 20022 (pacs.008) | Payment initiation fields mirror pacs.008 structured elements: debtor/creditor identification, IBAN, structured remittance |
| ISO 3166-1/2 | Jurisdiction codes use ISO 3166 alpha-2 country codes and subdivision codes |
| ISO 4217 | All currency fields use ISO 4217 three-letter currency codes |
| FAPI 2.0 / OAuth 2.0 | OAuth client registration and token tables align with RFC 6749 and FAPI 2.0 security profile requirements |
| GDPR | Data subject fields support right-to-erasure markers, lawful basis tracking, and data retention policies |
| PSD2 RTS on SCA | SCA authentication tables capture the two-factor requirement categories (knowledge, possession, inherence) |

---

## Tenant & Identity Management

```sql
-- =============================================================
-- TENANT & ORGANISATION
-- =============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tenant_type     VARCHAR(50) NOT NULL CHECK (tenant_type IN ('fintech', 'bank', 'aspsp', 'tpp', 'enterprise')),
    regulatory_status VARCHAR(50),                              -- e.g., 'aisp_licensed', 'pisp_licensed', 'both'
    jurisdiction    CHAR(2) NOT NULL,                            -- ISO 3166-1 alpha-2
    settings        JSONB DEFAULT '{}',                          -- tenant-level configuration
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_jurisdiction ON tenants (jurisdiction);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    email_verified  BOOLEAN DEFAULT FALSE,
    mfa_enabled     BOOLEAN DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    description     TEXT,
    is_system_role  BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE permissions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    resource        VARCHAR(100) NOT NULL,                      -- e.g., 'accounts', 'payments', 'consents'
    action          VARCHAR(50) NOT NULL,                       -- e.g., 'read', 'write', 'delete', 'approve'
    description     TEXT,
    UNIQUE (resource, action)
);

CREATE TABLE role_permissions (
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    permission_id   UUID NOT NULL REFERENCES permissions(id) ON DELETE CASCADE,
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by      UUID REFERENCES users(id),
    PRIMARY KEY (user_id, role_id)
);
```

## OAuth & Security

```sql
-- =============================================================
-- OAUTH CLIENTS & TOKENS (FAPI 2.0 aligned)
-- =============================================================

CREATE TABLE oauth_clients (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id),
    client_id               VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash      VARCHAR(255),
    client_name             VARCHAR(255) NOT NULL,
    client_type             VARCHAR(20) NOT NULL CHECK (client_type IN ('confidential', 'public')),
    redirect_uris           TEXT[] NOT NULL,
    allowed_scopes          TEXT[] NOT NULL,                     -- e.g., {'accounts', 'payments', 'funds-confirmation'}
    token_endpoint_auth     VARCHAR(50) DEFAULT 'private_key_jwt', -- FAPI 2.0 recommended
    jwks_uri                TEXT,
    software_statement      TEXT,                                -- signed JWT from trust directory
    tpp_role                VARCHAR(50),                         -- 'aisp', 'pisp', 'cbpii'
    regulatory_id           VARCHAR(100),                        -- national competent authority registration number
    is_active               BOOLEAN DEFAULT TRUE,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_oauth_clients_tenant ON oauth_clients (tenant_id);
CREATE INDEX idx_oauth_clients_client_id ON oauth_clients (client_id);

CREATE TABLE oauth_tokens (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id           UUID NOT NULL REFERENCES oauth_clients(id),
    user_id             UUID REFERENCES users(id),
    token_type          VARCHAR(20) NOT NULL CHECK (token_type IN ('access', 'refresh', 'id')),
    token_hash          VARCHAR(255) NOT NULL UNIQUE,
    scopes              TEXT[] NOT NULL,
    expires_at          TIMESTAMPTZ NOT NULL,
    revoked_at          TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_oauth_tokens_client ON oauth_tokens (client_id);
CREATE INDEX idx_oauth_tokens_expires ON oauth_tokens (expires_at);

CREATE TABLE sca_sessions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES users(id),
    consent_id          UUID,                                   -- linked after consent creation
    auth_method         VARCHAR(50) NOT NULL,                   -- 'redirect', 'decoupled', 'embedded'
    factor_knowledge    BOOLEAN DEFAULT FALSE,                  -- password, PIN, security question
    factor_possession   BOOLEAN DEFAULT FALSE,                  -- OTP device, mobile app, hardware token
    factor_inherence    BOOLEAN DEFAULT FALSE,                  -- biometric
    sca_status          VARCHAR(30) NOT NULL DEFAULT 'started'
                        CHECK (sca_status IN ('started', 'factor1_complete', 'authenticated', 'failed', 'expired')),
    started_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ,
    expires_at          TIMESTAMPTZ NOT NULL,
    ip_address          INET,
    user_agent          TEXT
);

CREATE INDEX idx_sca_sessions_user ON sca_sessions (user_id);
```

## Consent Management

```sql
-- =============================================================
-- CONSENT MANAGEMENT (PSD2 / NextGenPSD2 aligned)
-- =============================================================

CREATE TABLE consents (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id               UUID NOT NULL REFERENCES tenants(id),
    tpp_client_id           UUID NOT NULL REFERENCES oauth_clients(id),
    psu_id                  UUID REFERENCES users(id),          -- Payment Service User
    consent_type            VARCHAR(30) NOT NULL
                            CHECK (consent_type IN ('ais', 'pis', 'pis_recurring', 'funds_confirmation', 'combined')),
    status                  VARCHAR(30) NOT NULL DEFAULT 'received'
                            CHECK (status IN ('received', 'rejected', 'partiallyAuthorised',
                                              'valid', 'revokedByPsu', 'expired', 'terminatedByTpp')),
    -- AIS-specific fields (NextGenPSD2 consent request)
    access_accounts         UUID[],                             -- list of account IDs for account access
    access_balances         UUID[],                             -- list of account IDs for balance access
    access_transactions     UUID[],                             -- list of account IDs for transaction access
    recurring_indicator     BOOLEAN DEFAULT FALSE,
    frequency_per_day       INTEGER DEFAULT 4,                  -- max API calls per day under this consent
    valid_until             DATE,                               -- consent expiry date
    combined_service        BOOLEAN DEFAULT FALSE,              -- combined AIS+PIS consent
    -- Regulatory metadata
    last_action_date        TIMESTAMPTZ,
    sca_session_id          UUID REFERENCES sca_sessions(id),
    psd2_consent_id         VARCHAR(100),                       -- external consent ID per NextGenPSD2
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_tenant ON consents (tenant_id);
CREATE INDEX idx_consents_tpp ON consents (tpp_client_id);
CREATE INDEX idx_consents_psu ON consents (psu_id);
CREATE INDEX idx_consents_status ON consents (status);
CREATE INDEX idx_consents_valid_until ON consents (valid_until);

CREATE TABLE consent_status_history (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consent_id      UUID NOT NULL REFERENCES consents(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    changed_by      VARCHAR(50) NOT NULL,                       -- 'psu', 'tpp', 'aspsp', 'system'
    reason          TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consent_history_consent ON consent_status_history (consent_id);
CREATE INDEX idx_consent_history_changed_at ON consent_status_history (changed_at);
```

## Bank Connections & Accounts

```sql
-- =============================================================
-- BANK CONNECTIONS & INSTITUTIONS
-- =============================================================

CREATE TABLE institutions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name                VARCHAR(255) NOT NULL,
    bic                 VARCHAR(11),                            -- ISO 9362 BIC/SWIFT code
    country_code        CHAR(2) NOT NULL,                       -- ISO 3166-1
    api_standard        VARCHAR(30) NOT NULL
                        CHECK (api_standard IN ('nextgenpsd2', 'obie', 'fdx', 'proprietary')),
    api_base_url        TEXT NOT NULL,
    sandbox_url         TEXT,
    supports_ais        BOOLEAN DEFAULT TRUE,
    supports_pis        BOOLEAN DEFAULT FALSE,
    supports_vrp        BOOLEAN DEFAULT FALSE,
    supports_cof        BOOLEAN DEFAULT FALSE,                  -- confirmation of funds
    sca_methods         TEXT[],                                  -- e.g., {'redirect', 'decoupled'}
    health_status       VARCHAR(20) DEFAULT 'healthy'
                        CHECK (health_status IN ('healthy', 'degraded', 'down', 'unknown')),
    last_health_check   TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_institutions_country ON institutions (country_code);
CREATE INDEX idx_institutions_bic ON institutions (bic);

-- =============================================================
-- ACCOUNTS (Berlin Group / FDX aligned)
-- =============================================================

CREATE TABLE accounts (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    psu_id              UUID NOT NULL REFERENCES users(id),
    consent_id          UUID REFERENCES consents(id),
    -- Account identifiers
    resource_id         VARCHAR(100),                            -- ASPSP-assigned resource ID
    iban                VARCHAR(34),                             -- ISO 13616
    bban                VARCHAR(30),
    sort_code           VARCHAR(6),                              -- UK-specific
    account_number      VARCHAR(30),
    -- Account details (NextGenPSD2 accountDetails)
    account_type        VARCHAR(30) NOT NULL
                        CHECK (account_type IN ('current', 'savings', 'credit_card', 'loan', 'mortgage', 'investment')),
    currency            CHAR(3) NOT NULL,                        -- ISO 4217
    name                VARCHAR(255),                            -- account name/product name
    owner_name          VARCHAR(255),
    bic                 VARCHAR(11),                             -- ISO 9362
    status              VARCHAR(20) DEFAULT 'enabled'
                        CHECK (status IN ('enabled', 'deleted', 'blocked')),
    usage_type          VARCHAR(20) CHECK (usage_type IN ('private', 'organisation')),
    last_synced_at      TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);
CREATE INDEX idx_accounts_psu ON accounts (psu_id);
CREATE INDEX idx_accounts_institution ON accounts (institution_id);
CREATE INDEX idx_accounts_iban ON accounts (iban);

CREATE TABLE balances (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id          UUID NOT NULL REFERENCES accounts(id),
    balance_type        VARCHAR(30) NOT NULL
                        CHECK (balance_type IN ('closingBooked', 'openingBooked', 'expected',
                                                'interimAvailable', 'interimBooked', 'forwardAvailable',
                                                'nonInvoiced')),       -- NextGenPSD2 balance types
    amount              DECIMAL(18, 4) NOT NULL,
    currency            CHAR(3) NOT NULL,                        -- ISO 4217
    reference_date      DATE,
    last_change_dt      TIMESTAMPTZ,
    credit_limit_included BOOLEAN DEFAULT FALSE,
    fetched_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_balances_account ON balances (account_id);
CREATE INDEX idx_balances_type ON balances (balance_type);
```

## Transactions

```sql
-- =============================================================
-- TRANSACTIONS (Berlin Group transactionDetails aligned)
-- =============================================================

CREATE TABLE transactions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id              UUID NOT NULL REFERENCES accounts(id),
    tenant_id               UUID NOT NULL REFERENCES tenants(id),
    -- Transaction identifiers
    transaction_id_aspsp    VARCHAR(100),                        -- ASPSP-assigned ID
    entry_reference         VARCHAR(100),                        -- bank statement entry reference
    end_to_end_id           VARCHAR(35),                         -- ISO 20022 EndToEndId
    mandate_id              VARCHAR(35),                         -- for direct debits
    -- Amount & currency
    amount                  DECIMAL(18, 4) NOT NULL,
    currency                CHAR(3) NOT NULL,                    -- ISO 4217
    -- Dates
    booking_date            DATE,
    value_date              DATE,
    booking_datetime        TIMESTAMPTZ,
    -- Counterparty
    creditor_name           VARCHAR(255),
    creditor_iban           VARCHAR(34),
    creditor_bic            VARCHAR(11),
    debtor_name             VARCHAR(255),
    debtor_iban             VARCHAR(34),
    debtor_bic              VARCHAR(11),
    -- Details
    remittance_unstructured TEXT,
    remittance_structured   TEXT,                                -- ISO 20022 structured remittance
    bank_transaction_code   VARCHAR(20),                         -- ISO 20022 BankTransactionCode
    proprietary_code        VARCHAR(50),
    transaction_status      VARCHAR(20) NOT NULL DEFAULT 'booked'
                            CHECK (transaction_status IN ('booked', 'pending', 'information')),
    -- Enrichment (ML-powered)
    category_id             UUID REFERENCES transaction_categories(id),
    merchant_name           VARCHAR(255),
    merchant_mcc            VARCHAR(4),                          -- ISO 18245 Merchant Category Code
    enrichment_confidence   DECIMAL(3, 2),                       -- 0.00 to 1.00
    -- Metadata
    fetched_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_transactions_account ON transactions (account_id);
CREATE INDEX idx_transactions_tenant ON transactions (tenant_id);
CREATE INDEX idx_transactions_booking_date ON transactions (booking_date);
CREATE INDEX idx_transactions_e2e ON transactions (end_to_end_id);
CREATE INDEX idx_transactions_creditor ON transactions (creditor_name);

CREATE TABLE transaction_categories (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(id),                -- NULL = system-wide
    parent_id       UUID REFERENCES transaction_categories(id),
    name            VARCHAR(100) NOT NULL,
    code            VARCHAR(50),
    is_system       BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tx_categories_parent ON transaction_categories (parent_id);
```

## Payment Initiation

```sql
-- =============================================================
-- PAYMENT INITIATION (PIS — NextGenPSD2 / ISO 20022 aligned)
-- =============================================================

CREATE TABLE payment_initiations (
    id                          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id                   UUID NOT NULL REFERENCES tenants(id),
    consent_id                  UUID NOT NULL REFERENCES consents(id),
    tpp_client_id               UUID NOT NULL REFERENCES oauth_clients(id),
    -- Payment type
    payment_type                VARCHAR(30) NOT NULL
                                CHECK (payment_type IN ('single', 'bulk', 'periodic', 'vrp')),
    payment_product             VARCHAR(50) NOT NULL,            -- e.g., 'sepa-credit-transfers', 'instant-sepa', 'faster-payments'
    -- Debtor
    debtor_name                 VARCHAR(255),
    debtor_iban                 VARCHAR(34),
    debtor_bic                  VARCHAR(11),
    debtor_account_id           UUID REFERENCES accounts(id),
    -- Creditor (ISO 20022 pacs.008 aligned)
    creditor_name               VARCHAR(255) NOT NULL,
    creditor_iban               VARCHAR(34),
    creditor_bic                VARCHAR(11),
    creditor_address_street     VARCHAR(255),
    creditor_address_city       VARCHAR(100),
    creditor_address_postcode   VARCHAR(20),
    creditor_address_country    CHAR(2),                         -- ISO 3166-1
    -- Amount
    amount                      DECIMAL(18, 4) NOT NULL,
    currency                    CHAR(3) NOT NULL,                -- ISO 4217
    -- References
    end_to_end_id               VARCHAR(35),                     -- ISO 20022
    remittance_unstructured     TEXT,
    remittance_structured       TEXT,
    -- Scheduling (periodic payments)
    start_date                  DATE,
    end_date                    DATE,
    execution_rule              VARCHAR(20),                     -- 'following', 'preceding'
    frequency                   VARCHAR(20),                     -- 'daily', 'weekly', 'monthly', etc.
    day_of_execution            SMALLINT,
    -- VRP fields
    vrp_max_amount              DECIMAL(18, 4),
    vrp_max_frequency           VARCHAR(50),
    -- Status
    status                      VARCHAR(30) NOT NULL DEFAULT 'received'
                                CHECK (status IN ('received', 'rejected', 'pending',
                                                  'accp', 'acsc', 'acsp', 'actc', 'acwc', 'acwp',
                                                  'rjct', 'canc', 'pdng')),  -- ISO 20022 status codes
    status_reason               TEXT,
    aspsp_payment_id            VARCHAR(100),                    -- ASPSP-assigned payment ID
    created_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at                  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_tenant ON payment_initiations (tenant_id);
CREATE INDEX idx_payments_consent ON payment_initiations (consent_id);
CREATE INDEX idx_payments_status ON payment_initiations (status);
CREATE INDEX idx_payments_created ON payment_initiations (created_at);

CREATE TABLE payment_status_history (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id          UUID NOT NULL REFERENCES payment_initiations(id),
    old_status          VARCHAR(30),
    new_status          VARCHAR(30) NOT NULL,
    iso20022_code       VARCHAR(4),                              -- e.g., 'ACSC', 'RJCT'
    reason_code         VARCHAR(50),
    reason_text         TEXT,
    changed_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_history_payment ON payment_status_history (payment_id);
```

## Webhooks & Events

```sql
-- =============================================================
-- WEBHOOKS & EVENT NOTIFICATIONS
-- =============================================================

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    event_types     TEXT[] NOT NULL,                             -- e.g., {'consent.expired', 'payment.completed', 'transaction.new'}
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhooks_tenant ON webhook_subscriptions (tenant_id);

CREATE TABLE webhook_deliveries (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id     UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_type          VARCHAR(100) NOT NULL,
    payload             JSONB NOT NULL,
    http_status         INTEGER,
    response_body       TEXT,
    attempt_number      SMALLINT NOT NULL DEFAULT 1,
    delivered_at        TIMESTAMPTZ,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_webhook_deliveries_sub ON webhook_deliveries (subscription_id);
CREATE INDEX idx_webhook_deliveries_retry ON webhook_deliveries (next_retry_at) WHERE next_retry_at IS NOT NULL;
```

## API Health & Monitoring

```sql
-- =============================================================
-- INSTITUTION HEALTH MONITORING
-- =============================================================

CREATE TABLE institution_health_checks (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id      UUID NOT NULL REFERENCES institutions(id),
    check_type          VARCHAR(30) NOT NULL,                    -- 'ais_availability', 'pis_availability', 'latency'
    status              VARCHAR(20) NOT NULL,                    -- 'healthy', 'degraded', 'down'
    response_time_ms    INTEGER,
    error_message       TEXT,
    checked_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_health_institution ON institution_health_checks (institution_id);
CREATE INDEX idx_health_checked_at ON institution_health_checks (checked_at);

-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    user_id         UUID REFERENCES users(id),
    client_id       UUID REFERENCES oauth_clients(id),
    action          VARCHAR(100) NOT NULL,                       -- e.g., 'consent.created', 'payment.initiated'
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    ip_address      INET,
    user_agent      TEXT,
    details         JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id);
CREATE INDEX idx_audit_action ON audit_log (action);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);
CREATE INDEX idx_audit_created ON audit_log (created_at);

-- Partition audit_log by month for performance
-- CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

## Rate Limiting

```sql
-- =============================================================
-- API RATE LIMITING & THROTTLING
-- =============================================================

CREATE TABLE rate_limit_policies (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID REFERENCES tenants(id),                -- NULL = global
    client_id       UUID REFERENCES oauth_clients(id),          -- NULL = tenant-wide
    endpoint_pattern VARCHAR(255) NOT NULL,                     -- e.g., '/accounts/*', '/payments'
    max_requests    INTEGER NOT NULL,
    window_seconds  INTEGER NOT NULL,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_rate_limits_tenant ON rate_limit_policies (tenant_id);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 5 | tenants, users, roles, permissions, role_permissions |
| RBAC | 1 | user_roles |
| OAuth & Security | 3 | oauth_clients, oauth_tokens, sca_sessions |
| Consent Management | 2 | consents, consent_status_history |
| Bank Connections | 1 | institutions |
| Accounts & Balances | 2 | accounts, balances |
| Transactions | 2 | transactions, transaction_categories |
| Payment Initiation | 2 | payment_initiations, payment_status_history |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| Monitoring | 1 | institution_health_checks |
| Audit & Compliance | 1 | audit_log |
| Rate Limiting | 1 | rate_limit_policies |
| **Total** | **23** | |

---

## Key Design Decisions

1. **UUID primary keys everywhere** — required for multi-tenant SaaS where sequential IDs leak information and cause conflicts in distributed systems.

2. **Separate consent_status_history table** — PSD2 requires a full audit trail of consent lifecycle changes. Rather than relying on application-level logging, the schema enforces this at the data layer.

3. **ISO 20022 status codes in payment_initiations** — the `status` field uses the actual ISO 20022 four-letter status codes (ACSC, RJCT, etc.) to maintain alignment with bank responses without translation.

4. **Balance types from NextGenPSD2** — the `balance_type` enum mirrors the Berlin Group specification exactly (closingBooked, interimAvailable, etc.) rather than inventing custom terminology.

5. **Tenant-scoped everything** — every business entity includes `tenant_id` with an index, enabling PostgreSQL Row-Level Security policies for data isolation.

6. **SCA sessions as first-class entities** — Strong Customer Authentication is a regulatory requirement, not an implementation detail. Tracking factor completion (knowledge, possession, inherence) as explicit booleans makes compliance verification queryable.

7. **Transaction enrichment fields on the transaction table** — category, merchant name, and confidence score are stored alongside raw transaction data for query simplicity, rather than in a separate enrichment table.

8. **Audit log designed for time-partitioning** — the audit_log table is structured to support PostgreSQL declarative partitioning by month, critical for performance as the table grows.

9. **Array types for consent account lists** — NextGenPSD2 consents specify lists of account IDs for access; UUID arrays provide type safety while avoiding junction tables for this specific use case.

10. **Institution health as a separate monitoring table** — API health data is append-only and time-series in nature, kept separate from the operational institutions table to avoid write contention.
