# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Open Banking Platform · Created: 2026-05-20

## Philosophy

This model keeps core relational structure for entities that are stable across all jurisdictions and bank integrations (tenants, users, consents, payments), but uses PostgreSQL JSONB columns for fields that vary by jurisdiction, bank API standard, or financial product type. The relational skeleton provides integrity and queryability; the JSONB pockets absorb variation without schema migrations.

Open banking is inherently multi-standard: the Berlin Group NextGenPSD2, UK OBIE, and FDX each define slightly different account fields, consent structures, and payment attributes. A purely normalized model would require either union-of-all-fields tables (mostly NULL columns) or per-standard tables (duplicated logic). The hybrid approach uses typed JSONB columns with documented schemas, validated at the application layer, to hold standard-specific extensions while keeping the common denominator relational.

This is the approach taken by platforms like Plaid (which normalizes a common account/transaction model across 12,000+ institutions but stores institution-specific raw data alongside) and is well-suited for rapid iteration during the MVP phase. As patterns stabilize, frequently-queried JSONB fields can be promoted to proper columns via migrations.

**Best for:** Teams building a multi-jurisdiction platform that must integrate with NextGenPSD2, OBIE, and FDX simultaneously, or teams prioritising rapid MVP delivery with schema flexibility.

**Trade-offs:**
- (+) Handles multi-standard variation without schema migrations
- (+) Rapid feature iteration — new fields added without ALTER TABLE
- (+) Preserves raw bank API responses for debugging and compliance
- (+) GIN indexes on JSONB enable efficient querying of dynamic fields
- (+) Fewer tables than full normalization — simpler joins
- (-) JSONB fields lack database-level type constraints — validation shifts to application code
- (-) Complex JSONB queries can be slower than indexed relational columns
- (-) Schema documentation must be maintained outside the database (JSON Schema files)
- (-) Risk of schema drift between tenants if JSONB validation is not enforced
- (-) ORM support for JSONB varies — some frameworks handle it poorly

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Berlin Group NextGenPSD2 | Common consent/account/payment fields are relational; NextGenPSD2-specific extensions stored in `standard_data` JSONB columns |
| UK OBIE | OBIE-specific fields (e.g., `SchemeName`, `Identification` for UK sort code/account number) stored in `standard_data` |
| FDX v6.5 | FDX-specific attributes (e.g., `fdxAccountId`, investment/insurance data clusters) stored in `standard_data` |
| ISO 20022 | Structured remittance and address data stored in `iso20022_data` JSONB for payment messages |
| ISO 3166 / ISO 4217 | Jurisdiction and currency remain relational columns with CHECK constraints |
| FAPI 2.0 | OAuth client metadata stored in `fapi_metadata` JSONB to accommodate FAPI 1.0 vs 2.0 differences |
| GDPR | Data retention and erasure metadata tracked in `gdpr_metadata` JSONB per user |

---

## Tenant & Identity

```sql
-- =============================================================
-- TENANTS — Core relational + JSONB settings
-- =============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tenant_type     VARCHAR(50) NOT NULL,
    jurisdiction    CHAR(2) NOT NULL,                            -- ISO 3166-1
    is_active       BOOLEAN DEFAULT TRUE,
    -- Flexible configuration per tenant
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example config:
    -- {
    --   "supported_standards": ["nextgenpsd2", "obie"],
    --   "regulatory_status": "aisp_pisp_licensed",
    --   "regulatory_authority": "BaFin",
    --   "registration_number": "DE-123456",
    --   "default_consent_ttl_days": 90,
    --   "rate_limits": {"accounts": 100, "payments": 50},
    --   "branding": {"logo_url": "...", "primary_color": "#1a2b3c"},
    --   "psd3_ready": false
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tenants_slug ON tenants (slug);
CREATE INDEX idx_tenants_jurisdiction ON tenants (jurisdiction);
CREATE INDEX idx_tenants_config ON tenants USING GIN (config);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
    mfa_enabled     BOOLEAN DEFAULT FALSE,
    -- User-level flexible data
    profile         JSONB NOT NULL DEFAULT '{}',
    -- Example profile:
    -- {
    --   "phone": "+44...",
    --   "preferred_language": "de",
    --   "gdpr": {
    --     "lawful_basis": "consent",
    --     "data_retention_until": "2027-05-20",
    --     "erasure_requested": false
    --   },
    --   "kyc_status": "verified",
    --   "kyc_verified_at": "2026-01-15T10:30:00Z"
    -- }
    last_login_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, email)
);

CREATE INDEX idx_users_tenant ON users (tenant_id);
CREATE INDEX idx_users_profile ON users USING GIN (profile);

-- =============================================================
-- RBAC — Relational (stable across all jurisdictions)
-- =============================================================

CREATE TABLE roles (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            VARCHAR(100) NOT NULL,
    permissions     JSONB NOT NULL DEFAULT '[]',
    -- Example permissions:
    -- [
    --   {"resource": "accounts", "actions": ["read"]},
    --   {"resource": "payments", "actions": ["read", "write", "approve"]},
    --   {"resource": "consents", "actions": ["read", "write", "revoke"]}
    -- ]
    is_system_role  BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    granted_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id)
);
```

## OAuth & Security

```sql
-- =============================================================
-- OAUTH CLIENTS — Relational core + JSONB for standard-specific metadata
-- =============================================================

CREATE TABLE oauth_clients (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    client_id           VARCHAR(255) NOT NULL UNIQUE,
    client_secret_hash  VARCHAR(255),
    client_name         VARCHAR(255) NOT NULL,
    redirect_uris       TEXT[] NOT NULL,
    allowed_scopes      TEXT[] NOT NULL,
    tpp_role            VARCHAR(50),
    is_active           BOOLEAN DEFAULT TRUE,
    -- Standard-specific OAuth/FAPI metadata
    fapi_metadata       JSONB NOT NULL DEFAULT '{}',
    -- Example fapi_metadata:
    -- {
    --   "fapi_version": "2.0",
    --   "token_endpoint_auth_method": "private_key_jwt",
    --   "jwks_uri": "https://tpp.example.com/.well-known/jwks.json",
    --   "software_statement": "eyJ...",
    --   "regulatory_id": "PSDDE-BAFIN-123456",
    --   "request_object_signing_alg": "PS256",
    --   "id_token_signed_response_alg": "PS256",
    --   "tls_client_certificate_bound_access_tokens": true
    -- }
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_oauth_clients_tenant ON oauth_clients (tenant_id);
CREATE INDEX idx_oauth_clients_fapi ON oauth_clients USING GIN (fapi_metadata);

CREATE TABLE oauth_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES oauth_clients(id),
    user_id         UUID REFERENCES users(id),
    token_type      VARCHAR(20) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    -- Token-bound metadata (FAPI 2.0 sender-constrained tokens)
    token_metadata  JSONB DEFAULT '{}',
    -- Example: {"cnf": {"x5t#S256": "..."}, "dpop_thumbprint": "..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tokens_expires ON oauth_tokens (expires_at);
```

## Consent Management

```sql
-- =============================================================
-- CONSENTS — Relational core + JSONB for standard-specific fields
-- =============================================================

CREATE TABLE consents (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    tpp_client_id       UUID NOT NULL REFERENCES oauth_clients(id),
    psu_id              UUID REFERENCES users(id),
    api_standard        VARCHAR(30) NOT NULL,                    -- 'nextgenpsd2', 'obie', 'fdx'
    consent_type        VARCHAR(30) NOT NULL,                    -- 'ais', 'pis', 'pis_recurring', 'funds_confirmation', 'combined'
    status              VARCHAR(30) NOT NULL DEFAULT 'received',
    -- Common fields (cross-standard)
    recurring_indicator BOOLEAN DEFAULT FALSE,
    valid_until         DATE,
    frequency_per_day   INTEGER DEFAULT 4,
    -- Standard-specific consent data (varies by api_standard)
    standard_data       JSONB NOT NULL DEFAULT '{}',
    -- NextGenPSD2 example:
    -- {
    --   "access": {
    --     "accounts": [{"iban": "DE89...", "currency": "EUR"}],
    --     "balances": [{"iban": "DE89..."}],
    --     "transactions": [{"iban": "DE89..."}]
    --   },
    --   "combinedServiceIndicator": false,
    --   "psd2ConsentId": "CONSENT-12345"
    -- }
    --
    -- OBIE example:
    -- {
    --   "permissions": ["ReadAccountsDetail", "ReadBalances", "ReadTransactionsCredits"],
    --   "expirationDateTime": "2026-11-20T00:00:00Z",
    --   "transactionFromDateTime": "2026-01-01T00:00:00Z",
    --   "transactionToDateTime": "2026-12-31T23:59:59Z",
    --   "consentId": "OB-CONSENT-67890"
    -- }
    --
    -- FDX example:
    -- {
    --   "durationType": "oneTime",
    --   "resources": [
    --     {"resourceType": "ACCOUNT", "dataClusters": ["ACCOUNT_BASIC", "TRANSACTIONS"]}
    --   ],
    --   "lookbackPeriod": 90,
    --   "consentShareDurationUnit": "DAYS",
    --   "consentShareDuration": 365
    -- }
    sca_completed       BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_tenant ON consents (tenant_id);
CREATE INDEX idx_consents_tpp ON consents (tpp_client_id);
CREATE INDEX idx_consents_psu ON consents (psu_id);
CREATE INDEX idx_consents_status ON consents (status);
CREATE INDEX idx_consents_standard ON consents (api_standard);
CREATE INDEX idx_consents_standard_data ON consents USING GIN (standard_data);

CREATE TABLE consent_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consent_id      UUID NOT NULL REFERENCES consents(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    changed_by      VARCHAR(50) NOT NULL,
    reason          TEXT,
    event_data      JSONB DEFAULT '{}',                          -- additional context per event
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consent_audit_consent ON consent_audit (consent_id, changed_at);
```

## Institutions & Accounts

```sql
-- =============================================================
-- INSTITUTIONS — Relational core + JSONB for API-specific config
-- =============================================================

CREATE TABLE institutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    bic             VARCHAR(11),
    country_code    CHAR(2) NOT NULL,
    api_standard    VARCHAR(30) NOT NULL,
    -- Common capabilities
    supports_ais    BOOLEAN DEFAULT TRUE,
    supports_pis    BOOLEAN DEFAULT FALSE,
    supports_vrp    BOOLEAN DEFAULT FALSE,
    health_status   VARCHAR(20) DEFAULT 'healthy',
    -- API-specific connection configuration
    connection_config JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "api_base_url": "https://xs2a.bank.de/v1",
    --   "sandbox_url": "https://xs2a-sandbox.bank.de/v1",
    --   "sca_methods": ["redirect", "decoupled"],
    --   "auth_url": "https://auth.bank.de/oauth2/authorize",
    --   "token_url": "https://auth.bank.de/oauth2/token",
    --   "rate_limit": {"requests_per_minute": 60},
    --   "custom_headers": {"X-Request-ID-Format": "uuid"},
    --   "quirks": {
    --     "requires_psu_ip_address": true,
    --     "consent_status_polling_interval_ms": 2000,
    --     "date_format": "yyyy-MM-dd"
    --   }
    -- }
    last_health_check TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_institutions_country ON institutions (country_code);
CREATE INDEX idx_institutions_standard ON institutions (api_standard);

-- =============================================================
-- ACCOUNTS — Relational core + JSONB for raw bank data
-- =============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    psu_id          UUID NOT NULL REFERENCES users(id),
    consent_id      UUID REFERENCES consents(id),
    -- Common fields (normalized across standards)
    iban            VARCHAR(34),
    account_type    VARCHAR(30) NOT NULL,
    currency        CHAR(3) NOT NULL,                            -- ISO 4217
    display_name    VARCHAR(255),
    owner_name      VARCHAR(255),
    status          VARCHAR(20) DEFAULT 'enabled',
    -- Raw bank-specific data preserved for debugging & compliance
    raw_data        JSONB NOT NULL DEFAULT '{}',
    -- NextGenPSD2 example:
    -- {
    --   "resourceId": "3dc3d5b3-7023-4848-9853-f5400a64e80f",
    --   "bban": "1234567890",
    --   "msisdn": "+49170...",
    --   "product": "Girokonto Premium",
    --   "cashAccountType": "CACC",
    --   "usage": "PRIV",
    --   "_links": {"balances": {"href": "/v1/accounts/.../balances"}}
    -- }
    --
    -- OBIE example:
    -- {
    --   "accountId": "22289",
    --   "schemeName": "UK.OBIE.SortCodeAccountNumber",
    --   "identification": "80200110203345",
    --   "secondaryIdentification": "00021",
    --   "accountSubType": "CurrentAccount"
    -- }
    --
    -- FDX example:
    -- {
    --   "depositAccount": {
    --     "accountId": "fdx-12345",
    --     "accountCategory": "DEPOSIT_ACCOUNT",
    --     "accountType": "CHECKING",
    --     "interestRate": 0.05,
    --     "annualPercentageYield": 0.051
    --   }
    -- }
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);
CREATE INDEX idx_accounts_psu ON accounts (psu_id);
CREATE INDEX idx_accounts_iban ON accounts (iban);
CREATE INDEX idx_accounts_raw ON accounts USING GIN (raw_data);

CREATE TABLE balances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    balance_type    VARCHAR(30) NOT NULL,
    amount          DECIMAL(18, 4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    reference_date  DATE,
    -- Standard-specific balance metadata
    raw_data        JSONB DEFAULT '{}',
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_balances_account ON balances (account_id);
```

## Transactions

```sql
-- =============================================================
-- TRANSACTIONS — Normalized common fields + JSONB for enrichment & raw data
-- =============================================================

CREATE TABLE transactions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id              UUID NOT NULL REFERENCES accounts(id),
    tenant_id               UUID NOT NULL REFERENCES tenants(id),
    -- Common fields (normalized across standards)
    amount                  DECIMAL(18, 4) NOT NULL,
    currency                CHAR(3) NOT NULL,
    booking_date            DATE,
    value_date              DATE,
    creditor_name           VARCHAR(255),
    debtor_name             VARCHAR(255),
    remittance_info         TEXT,
    transaction_status      VARCHAR(20) NOT NULL DEFAULT 'booked',
    -- ML enrichment results
    enrichment              JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "category": "groceries",
    --   "subcategory": "supermarket",
    --   "merchant": {
    --     "name": "Tesco",
    --     "mcc": "5411",
    --     "logo_url": "https://..."
    --   },
    --   "confidence": 0.94,
    --   "model_version": "enrichment-v3.2",
    --   "enriched_at": "2026-05-20T10:30:00Z",
    --   "tags": ["recurring", "essential"],
    --   "carbon_estimate_kg": 2.3
    -- }
    -- Raw bank-specific data
    raw_data                JSONB NOT NULL DEFAULT '{}',
    -- Example (NextGenPSD2):
    -- {
    --   "transactionId": "TXN-12345",
    --   "entryReference": "ENT-67890",
    --   "endToEndId": "E2E-ABC",
    --   "mandateId": "MANDATE-001",
    --   "bankTransactionCode": "PMNT-RCDT-ESCT",
    --   "proprietaryBankTransactionCode": "TRF",
    --   "creditorAccount": {"iban": "DE89..."},
    --   "remittanceInformationStructured": "RF18539007547034"
    -- }
    fetched_at              TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_at              TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tx_account_date ON transactions (account_id, booking_date);
CREATE INDEX idx_tx_tenant ON transactions (tenant_id);
CREATE INDEX idx_tx_enrichment ON transactions USING GIN (enrichment);
CREATE INDEX idx_tx_raw ON transactions USING GIN (raw_data);

-- Query example: find all transactions categorised as "groceries" with high confidence
-- SELECT id, amount, currency, booking_date, enrichment->>'merchant' AS merchant
-- FROM transactions
-- WHERE tenant_id = 'uuid-...'
--   AND enrichment->>'category' = 'groceries'
--   AND (enrichment->>'confidence')::DECIMAL > 0.8
-- ORDER BY booking_date DESC;

-- Query example: find transactions by bank-specific endToEndId
-- SELECT * FROM transactions
-- WHERE raw_data->>'endToEndId' = 'E2E-ABC';
```

## Payments

```sql
-- =============================================================
-- PAYMENT INITIATION — Relational core + JSONB for standard-specific fields
-- =============================================================

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    consent_id      UUID NOT NULL REFERENCES consents(id),
    api_standard    VARCHAR(30) NOT NULL,
    -- Common fields
    payment_type    VARCHAR(30) NOT NULL,
    creditor_name   VARCHAR(255) NOT NULL,
    creditor_iban   VARCHAR(34),
    amount          DECIMAL(18, 4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'received',
    -- Standard-specific payment data
    standard_data   JSONB NOT NULL DEFAULT '{}',
    -- NextGenPSD2 example:
    -- {
    --   "paymentProduct": "sepa-credit-transfers",
    --   "debtorAccount": {"iban": "DE89...", "currency": "EUR"},
    --   "creditorAddress": {
    --     "street": "Hauptstraße 1",
    --     "buildingNumber": "1",
    --     "city": "Berlin",
    --     "postalCode": "10115",
    --     "country": "DE"
    --   },
    --   "endToEndIdentification": "E2E-12345",
    --   "remittanceInformationStructured": "RF18539007547034",
    --   "requestedExecutionDate": "2026-05-25"
    -- }
    --
    -- OBIE example:
    -- {
    --   "domesticPaymentId": "58923",
    --   "instructionIdentification": "ACME412",
    --   "endToEndIdentification": "FRESCO.21302.GFX.20",
    --   "localInstrument": "UK.OBIE.FPS",
    --   "creditorAccount": {
    --     "schemeName": "UK.OBIE.SortCodeAccountNumber",
    --     "identification": "08080021325698"
    --   }
    -- }
    -- ISO 20022 structured data for cross-border payments
    iso20022_data   JSONB DEFAULT '{}',
    -- Example:
    -- {
    --   "msgId": "MSG-20260520-001",
    --   "pmtInfId": "PMTINF-001",
    --   "dbtr": {"nm": "Acme Corp", "pstlAdr": {"strtNm": "High St", "twnNm": "London", "ctry": "GB"}},
    --   "cdtr": {"nm": "Supplier GmbH", "pstlAdr": {"strtNm": "Hauptstr", "twnNm": "Berlin", "ctry": "DE"}},
    --   "rmtInf": {"strd": [{"cdtrRefInf": {"ref": "RF18539007547034"}}]}
    -- }
    aspsp_payment_id VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_tenant ON payments (tenant_id);
CREATE INDEX idx_payments_status ON payments (status);
CREATE INDEX idx_payments_standard ON payments (api_standard);
CREATE INDEX idx_payments_standard_data ON payments USING GIN (standard_data);

CREATE TABLE payment_status_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    status_detail   JSONB DEFAULT '{}',
    -- Example: {"iso20022Code": "ACSC", "reasonCode": "AM04", "reasonText": "Insufficient funds"}
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payment_status_payment ON payment_status_log (payment_id, changed_at);
```

## Webhooks, Monitoring & Audit

```sql
-- =============================================================
-- WEBHOOKS
-- =============================================================

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    event_types     TEXT[] NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    config          JSONB DEFAULT '{}',                          -- retry policy, headers, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE webhook_deliveries (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    subscription_id UUID NOT NULL REFERENCES webhook_subscriptions(id),
    event_type      VARCHAR(100) NOT NULL,
    payload         JSONB NOT NULL,
    http_status     INTEGER,
    attempt_number  SMALLINT DEFAULT 1,
    delivered_at    TIMESTAMPTZ,
    next_retry_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- INSTITUTION HEALTH
-- =============================================================

CREATE TABLE institution_health_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    status          VARCHAR(20) NOT NULL,
    response_time_ms INTEGER,
    error_detail    JSONB DEFAULT '{}',
    checked_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_health_inst ON institution_health_log (institution_id, checked_at);

-- =============================================================
-- AUDIT LOG — Append-only with JSONB details
-- =============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_type      VARCHAR(30) NOT NULL,                       -- 'user', 'tpp', 'system', 'ai_agent'
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    details         JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "ip_address": "192.168.1.1",
    --   "user_agent": "TPPApp/2.0",
    --   "consent_type": "ais",
    --   "accounts_accessed": ["uuid-1", "uuid-2"],
    --   "api_standard": "nextgenpsd2"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);
CREATE INDEX idx_audit_details ON audit_log USING GIN (details);

-- =============================================================
-- AI / ML MODEL REGISTRY
-- =============================================================

CREATE TABLE ml_models (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    model_name      VARCHAR(100) NOT NULL,
    model_version   VARCHAR(50) NOT NULL,
    model_type      VARCHAR(50) NOT NULL,                       -- 'transaction_enrichment', 'fraud_detection', 'anomaly_detection'
    config          JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "algorithm": "gradient_boost",
    --   "training_data_range": "2025-01-01 to 2026-04-30",
    --   "accuracy": 0.94,
    --   "f1_score": 0.91,
    --   "artifact_url": "s3://models/enrichment-v3.2.tar.gz"
    -- }
    is_active       BOOLEAN DEFAULT FALSE,
    deployed_at     TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (model_name, model_version)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 2 | tenants, users |
| RBAC | 2 | roles, user_roles |
| OAuth & Security | 2 | oauth_clients, oauth_tokens |
| Consent Management | 2 | consents, consent_audit |
| Institutions | 1 | institutions |
| Accounts & Balances | 2 | accounts, balances |
| Transactions | 1 | transactions (enrichment in JSONB) |
| Payments | 2 | payments, payment_status_log |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| Monitoring | 1 | institution_health_log |
| Audit | 1 | audit_log |
| AI/ML | 1 | ml_models |
| **Total** | **19** | Fewer tables than normalized; flexibility via JSONB |

---

## Key Design Decisions

1. **`api_standard` discriminator on consents and payments** — rather than separate tables for NextGenPSD2, OBIE, and FDX consent/payment models, a single table with a `standard_data` JSONB column holds standard-specific fields. The `api_standard` column tells the application which JSON Schema to apply for validation.

2. **`raw_data` on accounts and transactions** — the raw response from the bank's API is preserved verbatim in JSONB. This serves three purposes: debugging integration issues, regulatory compliance (proving what data was received), and enabling future enrichment of fields not initially mapped.

3. **GIN indexes on all JSONB columns** — PostgreSQL GIN indexes support containment queries (`@>`) and key-existence checks (`?`) on JSONB, enabling efficient filtering without promoting fields to columns. As specific JSONB paths become performance bottlenecks, expression indexes can be added (e.g., `CREATE INDEX ON transactions ((enrichment->>'category'))`).

4. **Permissions as JSONB arrays in roles** — rather than a separate permissions table and junction table, permissions are stored as a JSONB array in the roles table. For a platform with ~20 permission types, the JSONB approach eliminates two tables and simplifies permission checking to a single row fetch.

5. **`enrichment` JSONB on transactions** — ML-powered enrichment (category, merchant, confidence) is stored as JSONB rather than separate columns because enrichment schemas evolve rapidly (new fields like `carbon_estimate_kg`, `recurring` flags) and vary by ML model version.

6. **`iso20022_data` JSONB on payments** — ISO 20022 pacs.008 messages have deep, nested structures (debtor postal address, structured remittance with creditor reference). Storing these as JSONB preserves the full ISO 20022 structure for SWIFT-connected payments without flattening into dozens of nullable columns.

7. **`connection_config` JSONB on institutions** — each bank has different API quirks (date formats, required headers, polling intervals, fallback URLs). JSONB absorbs this per-institution variation cleanly.

8. **ML model registry** — the platform's AI features (enrichment, fraud detection, anomaly detection) require tracking which model versions are deployed, their accuracy metrics, and configuration. A simple registry table with JSONB config avoids over-engineering this for the MVP.

9. **No separate transaction_categories table** — categories are stored in the `enrichment` JSONB. This avoids maintaining a category taxonomy table that would need to be synced with ML model outputs. Categories are treated as ML-generated labels, not as a reference data entity.

10. **Tenant config as JSONB** — rate limits, branding, regulatory status, and PSD3 readiness flags all live in the tenant's `config` JSONB. This enables per-tenant feature flags and settings without schema changes for each new configuration option.
