# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Open Banking Platform · Created: 2026-05-20

## Philosophy

This model layers a property graph structure on top of relational operational tables. The relational layer handles CRUD operations, transactional integrity, and regulatory data storage. The graph layer — implemented either as PostgreSQL tables with recursive CTEs or as a dedicated graph database (Neo4j, Amazon Neptune) — captures the rich web of relationships between entities: which TPPs access which accounts, which PSUs share accounts, which institutions connect to which payment networks, and which transactions flow between which parties.

Open banking is fundamentally a relationship-heavy domain. A single payment flows through a chain: PSU grants consent to TPP, TPP initiates payment via ASPSP, ASPSP routes through payment scheme, funds arrive at creditor's institution. Fraud detection requires traversing relationship graphs: does this TPP have an unusual pattern of consent requests across PSU clusters? Do these accounts share counterparties in a pattern suggesting money laundering? Are there circular payment flows? These questions are natural graph queries that are expensive or impossible in pure relational models.

Financial regulators increasingly require relationship analysis for AML (Anti-Money Laundering) and fraud prevention. Graph databases are the standard tool in financial crime investigation at major banks. By embedding graph capabilities into the open banking platform from the start, the platform gains a structural advantage for AI-powered fraud detection, consent pattern analysis, and network visualisation.

**Best for:** Teams where fraud detection, AML compliance, consent network analysis, or complex relationship queries are primary requirements.

**Trade-offs:**
- (+) Natural modelling of payment flows, consent chains, and entity relationships
- (+) Efficient traversal queries — "find all accounts linked to this TPP within 3 hops"
- (+) Excellent foundation for ML-powered fraud detection and anomaly analysis
- (+) Supports relationship-based access control patterns
- (+) Graph visualisation tools provide intuitive compliance dashboards
- (-) Dual-write to relational and graph layers increases operational complexity
- (-) Graph databases add infrastructure cost and operational expertise requirements
- (-) Transactional consistency between relational and graph stores requires saga patterns or CDC
- (-) Smaller talent pool familiar with graph query languages (Cypher, SPARQL, Gremlin)
- (-) PostgreSQL-only graph (via recursive CTEs / ltree) has performance limits at scale

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Berlin Group NextGenPSD2 | Consent and payment entities stored relationally; TPP-PSU-ASPSP relationships modelled as graph edges |
| ISO 20022 | Payment chain participants (debtor, creditor, intermediary agents) modelled as graph nodes with payment edges |
| FAPI 2.0 | Trust relationships between OAuth clients, authorization servers, and trust anchors modelled in the graph |
| FATF Recommendations | Graph structure enables the "travel rule" and beneficial ownership chain analysis required by FATF |
| ISO 3166 / ISO 4217 | Jurisdiction and currency as node properties for geographic filtering |
| PSD2 RTS on SCA | SCA sessions linked to consent and payment nodes via edges |

---

## Relational Layer (Operational CRUD)

```sql
-- =============================================================
-- TENANTS & USERS (standard relational)
-- =============================================================

CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    slug            VARCHAR(100) NOT NULL UNIQUE,
    tenant_type     VARCHAR(50) NOT NULL,
    jurisdiction    CHAR(2) NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    config          JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           VARCHAR(255) NOT NULL,
    password_hash   VARCHAR(255),
    full_name       VARCHAR(255),
    is_active       BOOLEAN DEFAULT TRUE,
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
    permissions     JSONB NOT NULL DEFAULT '[]',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE user_roles (
    user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id         UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    PRIMARY KEY (user_id, role_id)
);

-- =============================================================
-- OAUTH CLIENTS
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
    regulatory_id       VARCHAR(100),
    fapi_metadata       JSONB DEFAULT '{}',
    is_active           BOOLEAN DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE oauth_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id       UUID NOT NULL REFERENCES oauth_clients(id),
    user_id         UUID REFERENCES users(id),
    token_type      VARCHAR(20) NOT NULL,
    token_hash      VARCHAR(255) NOT NULL UNIQUE,
    scopes          TEXT[] NOT NULL,
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked_at      TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- INSTITUTIONS
-- =============================================================

CREATE TABLE institutions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            VARCHAR(255) NOT NULL,
    bic             VARCHAR(11),
    country_code    CHAR(2) NOT NULL,
    api_standard    VARCHAR(30) NOT NULL,
    supports_ais    BOOLEAN DEFAULT TRUE,
    supports_pis    BOOLEAN DEFAULT FALSE,
    supports_vrp    BOOLEAN DEFAULT FALSE,
    connection_config JSONB DEFAULT '{}',
    health_status   VARCHAR(20) DEFAULT 'healthy',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- CONSENTS
-- =============================================================

CREATE TABLE consents (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    tpp_client_id   UUID NOT NULL REFERENCES oauth_clients(id),
    psu_id          UUID REFERENCES users(id),
    consent_type    VARCHAR(30) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'received',
    api_standard    VARCHAR(30) NOT NULL,
    recurring_indicator BOOLEAN DEFAULT FALSE,
    valid_until     DATE,
    frequency_per_day INTEGER DEFAULT 4,
    standard_data   JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consents_tenant ON consents (tenant_id);
CREATE INDEX idx_consents_status ON consents (status);

CREATE TABLE consent_audit (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    consent_id      UUID NOT NULL REFERENCES consents(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    changed_by      VARCHAR(50) NOT NULL,
    reason          TEXT,
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- ACCOUNTS
-- =============================================================

CREATE TABLE accounts (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    institution_id  UUID NOT NULL REFERENCES institutions(id),
    psu_id          UUID NOT NULL REFERENCES users(id),
    consent_id      UUID REFERENCES consents(id),
    iban            VARCHAR(34),
    account_type    VARCHAR(30) NOT NULL,
    currency        CHAR(3) NOT NULL,
    display_name    VARCHAR(255),
    owner_name      VARCHAR(255),
    status          VARCHAR(20) DEFAULT 'enabled',
    raw_data        JSONB DEFAULT '{}',
    last_synced_at  TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_accounts_tenant ON accounts (tenant_id);
CREATE INDEX idx_accounts_psu ON accounts (psu_id);
CREATE INDEX idx_accounts_iban ON accounts (iban);

CREATE TABLE balances (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id      UUID NOT NULL REFERENCES accounts(id),
    balance_type    VARCHAR(30) NOT NULL,
    amount          DECIMAL(18, 4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    reference_date  DATE,
    fetched_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- TRANSACTIONS
-- =============================================================

CREATE TABLE transactions (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id          UUID NOT NULL REFERENCES accounts(id),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    amount              DECIMAL(18, 4) NOT NULL,
    currency            CHAR(3) NOT NULL,
    booking_date        DATE,
    value_date          DATE,
    creditor_name       VARCHAR(255),
    creditor_iban       VARCHAR(34),
    debtor_name         VARCHAR(255),
    debtor_iban         VARCHAR(34),
    remittance_info     TEXT,
    transaction_status  VARCHAR(20) NOT NULL DEFAULT 'booked',
    enrichment          JSONB DEFAULT '{}',
    raw_data            JSONB DEFAULT '{}',
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_tx_account_date ON transactions (account_id, booking_date);
CREATE INDEX idx_tx_tenant ON transactions (tenant_id);

-- =============================================================
-- PAYMENTS
-- =============================================================

CREATE TABLE payments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    consent_id      UUID NOT NULL REFERENCES consents(id),
    payment_type    VARCHAR(30) NOT NULL,
    payment_product VARCHAR(50),
    creditor_name   VARCHAR(255) NOT NULL,
    creditor_iban   VARCHAR(34),
    amount          DECIMAL(18, 4) NOT NULL,
    currency        CHAR(3) NOT NULL,
    status          VARCHAR(30) NOT NULL DEFAULT 'received',
    standard_data   JSONB DEFAULT '{}',
    aspsp_payment_id VARCHAR(100),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_payments_tenant ON payments (tenant_id);
CREATE INDEX idx_payments_status ON payments (status);

CREATE TABLE payment_status_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    payment_id      UUID NOT NULL REFERENCES payments(id),
    old_status      VARCHAR(30),
    new_status      VARCHAR(30) NOT NULL,
    status_detail   JSONB DEFAULT '{}',
    changed_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## Graph Layer (PostgreSQL Implementation)

```sql
-- =============================================================
-- PROPERTY GRAPH — PostgreSQL native implementation
-- =============================================================
-- This graph layer can be implemented in PostgreSQL (as shown here)
-- or in a dedicated graph database (Neo4j, Amazon Neptune).
-- The PostgreSQL implementation uses two tables: nodes and edges.
-- For production at scale, consider a dedicated graph DB with CDC
-- from the relational tables.

CREATE TABLE graph_nodes (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_type       VARCHAR(50) NOT NULL,
    -- Node types: 'psu', 'tpp', 'aspsp', 'account', 'payment',
    --             'consent', 'institution', 'jurisdiction', 'payment_scheme'
    entity_id       UUID NOT NULL,                               -- FK to the relational entity
    label           VARCHAR(255),                                -- display label
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (psu node):
    -- {
    --   "email": "user@example.com",
    --   "jurisdiction": "DE",
    --   "kyc_status": "verified",
    --   "risk_score": 0.12
    -- }
    --
    -- Example (institution node):
    -- {
    --   "bic": "DEUTDEFF",
    --   "country": "DE",
    --   "api_standard": "nextgenpsd2",
    --   "tier": "tier1"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (node_type, entity_id)
);

CREATE INDEX idx_graph_nodes_tenant ON graph_nodes (tenant_id);
CREATE INDEX idx_graph_nodes_type ON graph_nodes (node_type);
CREATE INDEX idx_graph_nodes_entity ON graph_nodes (entity_id);
CREATE INDEX idx_graph_nodes_props ON graph_nodes USING GIN (properties);

CREATE TABLE graph_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    source_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    target_node_id  UUID NOT NULL REFERENCES graph_nodes(id),
    edge_type       VARCHAR(50) NOT NULL,
    -- Edge types:
    --   'GRANTED_CONSENT'     — psu -> consent
    --   'HOLDS_CONSENT'       — tpp -> consent
    --   'COVERS_ACCOUNT'      — consent -> account
    --   'OWNS_ACCOUNT'        — psu -> account
    --   'HOSTED_BY'           — account -> institution
    --   'INITIATED_PAYMENT'   — tpp -> payment
    --   'PAID_FROM'           — payment -> account (debtor)
    --   'PAID_TO'             — payment -> account (creditor)
    --   'ROUTED_THROUGH'      — payment -> institution (intermediary)
    --   'TRANSACTED_WITH'     — account -> account (inferred from transactions)
    --   'LOCATED_IN'          — institution -> jurisdiction
    --   'MEMBER_OF'           — institution -> payment_scheme
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Example (GRANTED_CONSENT):
    -- {
    --   "consent_type": "ais",
    --   "granted_at": "2026-05-20T10:00:00Z",
    --   "valid_until": "2026-11-20",
    --   "status": "valid"
    -- }
    --
    -- Example (TRANSACTED_WITH):
    -- {
    --   "transaction_count": 47,
    --   "total_amount": 12500.00,
    --   "currency": "EUR",
    --   "first_seen": "2026-01-15",
    --   "last_seen": "2026-05-19",
    --   "avg_amount": 265.96
    -- }
    weight          DECIMAL(10, 4) DEFAULT 1.0,                  -- edge weight for graph algorithms
    valid_from      TIMESTAMPTZ DEFAULT NOW(),
    valid_to        TIMESTAMPTZ,                                 -- NULL = currently active
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_graph_edges_source ON graph_edges (source_node_id);
CREATE INDEX idx_graph_edges_target ON graph_edges (target_node_id);
CREATE INDEX idx_graph_edges_type ON graph_edges (edge_type);
CREATE INDEX idx_graph_edges_tenant ON graph_edges (tenant_id);
CREATE INDEX idx_graph_edges_props ON graph_edges USING GIN (properties);
CREATE INDEX idx_graph_edges_valid ON graph_edges (valid_from, valid_to);
```

## Graph Query Examples

```sql
-- =============================================================
-- GRAPH QUERIES (PostgreSQL recursive CTEs)
-- =============================================================

-- 1. Find all accounts accessible to a specific TPP (via active consents)
SELECT DISTINCT
    a_node.entity_id AS account_id,
    a_node.label AS account_label,
    a_node.properties->>'iban' AS iban
FROM graph_nodes tpp_node
JOIN graph_edges e1 ON e1.source_node_id = tpp_node.id AND e1.edge_type = 'HOLDS_CONSENT'
JOIN graph_edges e2 ON e2.source_node_id = e1.target_node_id AND e2.edge_type = 'COVERS_ACCOUNT'
JOIN graph_nodes a_node ON a_node.id = e2.target_node_id
WHERE tpp_node.node_type = 'tpp'
  AND tpp_node.entity_id = 'tpp-uuid-here'
  AND (e1.properties->>'status') = 'valid'
  AND (e1.valid_to IS NULL OR e1.valid_to > NOW());

-- 2. Fraud signal: find PSUs who share counterparties with flagged accounts
--    (2-hop traversal: flagged_account -> counterparty -> other_accounts -> owners)
WITH flagged_counterparties AS (
    SELECT target_node_id AS counterparty_id
    FROM graph_edges
    WHERE source_node_id = (SELECT id FROM graph_nodes WHERE entity_id = 'flagged-account-uuid')
      AND edge_type = 'TRANSACTED_WITH'
),
linked_accounts AS (
    SELECT DISTINCT e.source_node_id AS account_node_id
    FROM graph_edges e
    JOIN flagged_counterparties fc ON fc.counterparty_id = e.target_node_id
    WHERE e.edge_type = 'TRANSACTED_WITH'
      AND e.source_node_id != (SELECT id FROM graph_nodes WHERE entity_id = 'flagged-account-uuid')
)
SELECT
    psu_node.entity_id AS psu_id,
    psu_node.label AS psu_name,
    COUNT(DISTINCT la.account_node_id) AS shared_counterparty_accounts
FROM linked_accounts la
JOIN graph_edges own_edge ON own_edge.target_node_id = la.account_node_id AND own_edge.edge_type = 'OWNS_ACCOUNT'
JOIN graph_nodes psu_node ON psu_node.id = own_edge.source_node_id
GROUP BY psu_node.entity_id, psu_node.label
ORDER BY shared_counterparty_accounts DESC;

-- 3. Payment flow trace: full chain from debtor to creditor including intermediaries
WITH RECURSIVE payment_chain AS (
    -- Start: payment node
    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        n.label AS target_label,
        n.node_type AS target_type,
        1 AS depth,
        ARRAY[e.source_node_id] AS path
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.target_node_id
    WHERE e.source_node_id = (SELECT id FROM graph_nodes WHERE entity_id = 'payment-uuid-here' AND node_type = 'payment')
      AND e.edge_type IN ('PAID_FROM', 'PAID_TO', 'ROUTED_THROUGH')

    UNION ALL

    SELECT
        e.source_node_id,
        e.target_node_id,
        e.edge_type,
        n.label,
        n.node_type,
        pc.depth + 1,
        pc.path || e.source_node_id
    FROM graph_edges e
    JOIN graph_nodes n ON n.id = e.target_node_id
    JOIN payment_chain pc ON pc.target_node_id = e.source_node_id
    WHERE pc.depth < 5
      AND NOT e.source_node_id = ANY(pc.path)
)
SELECT * FROM payment_chain ORDER BY depth;

-- 4. Consent network density: how many PSUs does each TPP access?
SELECT
    tpp_node.entity_id AS tpp_id,
    tpp_node.label AS tpp_name,
    COUNT(DISTINCT psu_edge.source_node_id) AS unique_psus,
    COUNT(DISTINCT consent_edge.target_node_id) AS active_consents
FROM graph_nodes tpp_node
JOIN graph_edges consent_edge ON consent_edge.source_node_id = tpp_node.id AND consent_edge.edge_type = 'HOLDS_CONSENT'
JOIN graph_edges psu_edge ON psu_edge.target_node_id = consent_edge.target_node_id AND psu_edge.edge_type = 'GRANTED_CONSENT'
WHERE tpp_node.node_type = 'tpp'
  AND (consent_edge.properties->>'status') = 'valid'
GROUP BY tpp_node.entity_id, tpp_node.label
ORDER BY unique_psus DESC;

-- 5. Circular payment detection (potential money laundering)
WITH RECURSIVE circular AS (
    SELECT
        e.source_node_id,
        e.target_node_id,
        ARRAY[e.source_node_id, e.target_node_id] AS path,
        (e.properties->>'total_amount')::DECIMAL AS total_flow,
        1 AS depth
    FROM graph_edges e
    WHERE e.edge_type = 'TRANSACTED_WITH'
      AND e.tenant_id = 'tenant-uuid-here'

    UNION ALL

    SELECT
        c.source_node_id,
        e.target_node_id,
        c.path || e.target_node_id,
        c.total_flow + (e.properties->>'total_amount')::DECIMAL,
        c.depth + 1
    FROM circular c
    JOIN graph_edges e ON e.source_node_id = c.target_node_id AND e.edge_type = 'TRANSACTED_WITH'
    WHERE c.depth < 6
      AND NOT e.target_node_id = ANY(c.path[2:])  -- allow cycle back to start
)
SELECT path, total_flow, depth
FROM circular
WHERE target_node_id = source_node_id  -- found a cycle
  AND depth >= 3
ORDER BY total_flow DESC;
```

## Webhooks, Monitoring & Audit

```sql
-- =============================================================
-- WEBHOOKS (same as relational model)
-- =============================================================

CREATE TABLE webhook_subscriptions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    url             TEXT NOT NULL,
    secret_hash     VARCHAR(255) NOT NULL,
    event_types     TEXT[] NOT NULL,
    is_active       BOOLEAN DEFAULT TRUE,
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
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- =============================================================
-- AUDIT LOG
-- =============================================================

CREATE TABLE audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    actor_type      VARCHAR(30) NOT NULL,
    actor_id        UUID,
    action          VARCHAR(100) NOT NULL,
    resource_type   VARCHAR(50) NOT NULL,
    resource_id     UUID,
    details         JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_tenant ON audit_log (tenant_id, created_at);
CREATE INDEX idx_audit_resource ON audit_log (resource_type, resource_id);

-- =============================================================
-- FRAUD DETECTION — Graph-powered risk scoring
-- =============================================================

CREATE TABLE risk_assessments (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    entity_type     VARCHAR(50) NOT NULL,                        -- 'psu', 'account', 'tpp', 'payment'
    entity_id       UUID NOT NULL,
    risk_score      DECIMAL(5, 4) NOT NULL,                      -- 0.0000 to 1.0000
    risk_factors    JSONB NOT NULL,
    -- Example:
    -- {
    --   "circular_payment_detected": false,
    --   "shared_counterparty_with_flagged": 2,
    --   "consent_velocity_anomaly": true,
    --   "unusual_jurisdiction_mix": false,
    --   "graph_centrality_score": 0.73,
    --   "community_detection_cluster": "cluster-42"
    -- }
    model_version   VARCHAR(50) NOT NULL,
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_risk_entity ON risk_assessments (entity_type, entity_id);
CREATE INDEX idx_risk_score ON risk_assessments (risk_score DESC);
CREATE INDEX idx_risk_tenant ON risk_assessments (tenant_id);

-- =============================================================
-- GRAPH ANALYTICS CACHE — Pre-computed graph metrics
-- =============================================================

CREATE TABLE graph_metrics (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    node_id         UUID NOT NULL REFERENCES graph_nodes(id),
    metric_type     VARCHAR(50) NOT NULL,
    -- Metric types: 'degree_centrality', 'betweenness_centrality',
    --               'pagerank', 'community_id', 'clustering_coefficient'
    metric_value    DECIMAL(10, 6) NOT NULL,
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (node_id, metric_type, computed_at)
);

CREATE INDEX idx_graph_metrics_node ON graph_metrics (node_id);
CREATE INDEX idx_graph_metrics_type ON graph_metrics (metric_type);
```

## Neo4j Alternative Schema

```cypher
// =============================================================
// NEO4J CYPHER SCHEMA — Alternative to PostgreSQL graph tables
// =============================================================
// If using Neo4j instead of PostgreSQL graph tables, the following
// Cypher statements create the equivalent graph structure.
// Data is synced from PostgreSQL via CDC (Change Data Capture).

// Node constraints
CREATE CONSTRAINT psu_id IF NOT EXISTS FOR (p:PSU) REQUIRE p.entity_id IS UNIQUE;
CREATE CONSTRAINT tpp_id IF NOT EXISTS FOR (t:TPP) REQUIRE t.entity_id IS UNIQUE;
CREATE CONSTRAINT account_id IF NOT EXISTS FOR (a:Account) REQUIRE a.entity_id IS UNIQUE;
CREATE CONSTRAINT institution_id IF NOT EXISTS FOR (i:Institution) REQUIRE i.entity_id IS UNIQUE;
CREATE CONSTRAINT consent_id IF NOT EXISTS FOR (c:Consent) REQUIRE c.entity_id IS UNIQUE;
CREATE CONSTRAINT payment_id IF NOT EXISTS FOR (p:Payment) REQUIRE p.entity_id IS UNIQUE;

// Example relationship creation (from CDC events)
// MATCH (psu:PSU {entity_id: $psu_id})
// MATCH (consent:Consent {entity_id: $consent_id})
// CREATE (psu)-[:GRANTED_CONSENT {
//   consent_type: 'ais',
//   granted_at: datetime(),
//   valid_until: date('2026-11-20'),
//   status: 'valid'
// }]->(consent);

// Fraud query: circular payments in Cypher (much simpler than SQL CTE)
// MATCH path = (a:Account)-[:TRANSACTED_WITH*3..6]->(a)
// WHERE a.tenant_id = $tenant_id
// RETURN path, reduce(total = 0, r in relationships(path) | total + r.total_amount) AS total_flow
// ORDER BY total_flow DESC
// LIMIT 20;

// Community detection (built-in algorithm)
// CALL gds.louvain.stream('transactionGraph')
// YIELD nodeId, communityId
// RETURN gds.util.asNode(nodeId).entity_id AS account_id, communityId
// ORDER BY communityId;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenant & Identity | 4 | tenants, users, roles, user_roles |
| OAuth & Security | 2 | oauth_clients, oauth_tokens |
| Consent Management | 2 | consents, consent_audit |
| Institutions | 1 | institutions |
| Accounts & Balances | 2 | accounts, balances |
| Transactions | 1 | transactions |
| Payments | 2 | payments, payment_status_log |
| Graph Layer | 2 | graph_nodes, graph_edges |
| Webhooks | 2 | webhook_subscriptions, webhook_deliveries |
| Audit | 1 | audit_log |
| Fraud & Analytics | 2 | risk_assessments, graph_metrics |
| **Total** | **21** | 19 relational + 2 graph |

---

## Key Design Decisions

1. **Two-table graph model (nodes + edges)** — rather than encoding relationships only as foreign keys in relational tables, the explicit graph layer enables arbitrary relationship traversal, temporal relationship tracking (via `valid_from`/`valid_to`), and edge properties (transaction counts, amounts, risk weights).

2. **Entity ID back-references** — every graph node has an `entity_id` linking to the corresponding relational table. This enables the application to query the graph for relationship patterns and then hydrate full entity details from relational tables — combining graph traversal speed with relational data richness.

3. **Edge weight for graph algorithms** — the `weight` column on edges enables weighted graph algorithms (shortest path, PageRank, community detection) without additional computation. Transaction edges use total amount as weight; consent edges use frequency.

4. **Temporal edges** — `valid_from` and `valid_to` on edges enable point-in-time graph queries: "what did the consent network look like on March 15th?" This is critical for regulatory investigations that reconstruct historical relationship states.

5. **`TRANSACTED_WITH` edges as aggregated relationships** — rather than creating one edge per transaction (which would create billions of edges), the `TRANSACTED_WITH` edge between two accounts aggregates transaction statistics (count, total amount, date range). These edges are maintained by a background process that periodically aggregates transaction data.

6. **Risk assessments as first-class entities** — graph-computed risk scores and their contributing factors are stored in a dedicated table, enabling trend analysis ("how has this account's risk score changed over time?") and explainability ("why was this flagged?").

7. **Graph metrics cache** — expensive graph computations (centrality, PageRank, community detection) are pre-computed and cached in `graph_metrics`. These are refreshed on a schedule (e.g., nightly) rather than computed on every query.

8. **Neo4j as optional upgrade path** — the PostgreSQL graph implementation works for moderate scale. The schema is designed so that a migration to Neo4j (via CDC from the relational layer) is straightforward: node types map to Neo4j labels, edge types map to relationship types, and properties map directly.

9. **Circular payment detection** — the recursive CTE for circular payments is a core fraud detection query. The graph structure makes this query possible; in a pure relational model, detecting cycles across 3-6 hops through transaction chains would require complex self-joins that are impractical at scale.

10. **Dual-write strategy** — the relational layer is the system of record. Graph nodes and edges are created/updated via application-level event handlers or a CDC pipeline (e.g., Debezium) that listens to changes on the relational tables. This avoids distributed transactions while keeping the graph eventually consistent (typically within seconds).
