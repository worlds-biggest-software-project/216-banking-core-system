# Data Model Suggestion 4: Graph-Relational Hybrid

> Project: Banking Core System (Community Bank) · Created: 2026-05-20

## Philosophy

The graph-relational hybrid maintains a conventional relational core for CRUD operations, general ledger, and payment processing, but adds a property graph layer specifically for relationship-intensive queries: fraud detection across transaction networks, AML suspicious activity pattern recognition, beneficial ownership chain analysis, and conflict-of-interest detection. The graph layer operates on `graph_node` and `graph_edge` tables within PostgreSQL (using recursive CTEs and the `ltree` extension), avoiding the operational complexity of a separate graph database while enabling relationship-centric queries that would require deeply nested joins in a purely relational model.

This approach is motivated by the project's AI-native positioning. The research identifies fraud detection, AML monitoring, and relationship analysis as core differentiators over legacy cores that rely on threshold-based rules. Graph structures are the natural representation for these problems: "show me all accounts connected to this customer through shared addresses, shared phone numbers, or transfer patterns within the last 90 days" is a graph traversal, not a SQL join. Regulatory bodies (FinCEN, FFIEC) increasingly expect institutions to demonstrate relationship-aware monitoring — not just individual transaction thresholds.

The relational side handles all standard banking operations (accounts, transactions, GL, ACH, wires). The graph side provides an analytical overlay: nodes represent customers, accounts, addresses, phones, and external entities; edges represent ownership, transfers, shared attributes, and suspicious connections. The graph is maintained by event-driven projections from the relational side, ensuring it stays current without application-level dual-write complexity.

**Best for:** Institutions prioritizing AI-native fraud detection, AML network analysis, beneficial ownership tracing, and relationship-based risk scoring — where understanding the connections between entities matters as much as the entities themselves.

**Trade-offs:**
- (+) Powerful relationship traversal for fraud/AML pattern detection
- (+) Beneficial ownership chains and network analysis without recursive JOIN complexity
- (+) Graph data feeds directly into AI/ML models for link prediction and anomaly detection
- (+) Graph layer is an analytical overlay — does not complicate core CRUD operations
- (+) No separate graph database infrastructure; everything in PostgreSQL
- (-) Graph tables add storage overhead for edge data
- (-) Graph maintenance logic adds application complexity (event-driven sync)
- (-) Recursive CTEs have performance limits for very deep traversals (6+ hops)
- (-) Team must understand graph query patterns in addition to standard SQL
- (-) Graph consistency depends on reliable event-driven projection from relational tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 | Payment tables use ISO 20022 message fields; wire counterparties are projected as graph nodes |
| NACHA Operating Rules 2026 | ACH tables are fully relational; ACH originators and receivers are projected as graph edges |
| FDX v6.5 | Account and customer relational fields align with FDX entity names |
| FinCEN SAR/CTR | AML alerts enriched with graph-derived relationship context; SAR narrative can reference network analysis |
| FFIEC IT Handbook | Graph analysis supports the FFIEC expectation for "relationship-aware" BSA/AML monitoring |
| ISO 3166 / ISO 4217 | Standard codes used in relational tables and graph node properties |
| OCSF | Graph edge events align with OCSF (Open Cybersecurity Schema Framework) for structured security event logging |

---

## Graph Layer

```sql
-- Property graph: nodes
CREATE TABLE graph_node (
    node_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    node_type VARCHAR(30) NOT NULL,                    -- 'customer', 'account', 'address', 'phone', 'email', 'external_entity', 'device', 'ip_address'
    entity_id UUID,                                    -- FK to the relational table (customer_id, account_id, etc.) — NULL for external entities
    label VARCHAR(200) NOT NULL,                       -- human-readable label
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties examples by node_type:
    -- customer:        {"customer_number": "C001234", "risk_rating": "high", "kyc_status": "verified"}
    -- account:         {"account_number": "1001234567", "account_type": "dda", "balance": 5000.00}
    -- address:         {"street": "123 Main St", "city": "Springfield", "state": "IL", "postal_code": "62704"}
    -- phone:           {"number": "+12175551234", "type": "mobile"}
    -- email:           {"address": "john@example.com"}
    -- external_entity: {"name": "ACME Corp", "type": "ach_originator", "routing_number": "091000019"}
    -- device:          {"device_id": "dev-abc-123", "type": "mobile", "os": "iOS 19"}
    -- ip_address:      {"ip": "203.0.113.42", "geo_country": "US", "geo_city": "Chicago"}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gn_type ON graph_node(node_type);
CREATE INDEX idx_gn_entity ON graph_node(entity_id);
CREATE INDEX idx_gn_label ON graph_node(label);
CREATE INDEX idx_gn_props ON graph_node USING GIN (properties jsonb_path_ops);

-- Property graph: edges (directed relationships between nodes)
CREATE TABLE graph_edge (
    edge_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_node_id UUID NOT NULL REFERENCES graph_node(node_id),
    target_node_id UUID NOT NULL REFERENCES graph_node(node_id),
    edge_type VARCHAR(40) NOT NULL,                    -- relationship types (see catalog below)
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties examples by edge_type:
    -- OWNS_ACCOUNT:    {"ownership_type": "primary", "since": "2020-01-15"}
    -- TRANSFERRED_TO:  {"amount": 5000.00, "count": 12, "first_transfer": "2026-01-15", "last_transfer": "2026-05-15"}
    -- SHARES_ADDRESS:  {"address_id": "addr-uuid", "overlap_start": "2024-06-01"}
    -- SHARES_PHONE:    {"phone_number": "+12175551234"}
    -- RECEIVED_ACH:    {"total_amount": 38400.00, "count": 12, "company_entry_desc": "PAYROLL"}
    -- FLAGGED_WITH:    {"alert_id": "alert-uuid", "severity": "high", "reason": "structuring"}
    weight NUMERIC(10,4) DEFAULT 1.0,                  -- edge weight for ML/scoring algorithms
    first_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    last_seen_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ge_source ON graph_edge(source_node_id);
CREATE INDEX idx_ge_target ON graph_edge(target_node_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_props ON graph_edge USING GIN (properties jsonb_path_ops);
-- Composite index for traversal queries
CREATE INDEX idx_ge_source_type ON graph_edge(source_node_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_node_id, edge_type);
```

### Edge Type Catalog

| Edge Type | Source Node | Target Node | Meaning |
|-----------|-------------|-------------|---------|
| OWNS_ACCOUNT | customer | account | Customer owns/co-owns the account |
| AUTHORIZED_ON | customer | account | Customer is authorized signer but not owner |
| BENEFICIARY_OF | customer | account | Customer is a beneficiary |
| TRANSFERRED_TO | account | account | Funds transferred between accounts |
| SHARES_ADDRESS | customer | customer | Two customers share the same address |
| SHARES_PHONE | customer | customer | Two customers share the same phone number |
| SHARES_DEVICE | customer | customer | Two customers logged in from the same device |
| SHARES_IP | customer | customer | Two customers accessed from the same IP |
| RESIDES_AT | customer | address | Customer's registered address |
| USES_PHONE | customer | phone | Customer's registered phone |
| USES_EMAIL | customer | email | Customer's registered email |
| USES_DEVICE | customer | device | Customer has logged in from this device |
| ACCESSED_FROM | customer | ip_address | Customer accessed the platform from this IP |
| RECEIVED_ACH | account | external_entity | Account received ACH from this entity |
| SENT_ACH | account | external_entity | Account sent ACH to this entity |
| WIRE_TO | account | external_entity | Wire transfer to external entity |
| WIRE_FROM | account | external_entity | Wire transfer from external entity |
| RELATED_TO | customer | customer | Declared relationship (family, business) |
| FLAGGED_WITH | customer | aml_alert | Customer flagged by AML alert |
| GUARANTOR_FOR | customer | account | Customer guarantees a loan |

---

## Relational Core: Customer & Account

```sql
CREATE TABLE customer (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_number VARCHAR(20) NOT NULL UNIQUE,
    customer_type VARCHAR(20) NOT NULL,
    legal_name VARCHAR(300) NOT NULL,
    display_name VARCHAR(200),
    tax_id_type VARCHAR(10),
    tax_id_encrypted BYTEA,
    tax_id_last_four CHAR(4),
    date_of_birth DATE,
    kyc_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    risk_rating VARCHAR(10) DEFAULT 'standard',
    -- Graph node reference
    graph_node_id UUID,                                -- FK to graph_node for this customer
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_customer_number ON customer(customer_number);
CREATE INDEX idx_customer_risk ON customer(risk_rating);

CREATE TABLE customer_address (
    address_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    address_type VARCHAR(20) NOT NULL,
    street_name VARCHAR(200),
    building_number VARCHAR(20),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) NOT NULL,
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    graph_node_id UUID,                                -- FK to graph_node for this address
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_addr_customer ON customer_address(customer_id);

CREATE TABLE customer_contact (
    contact_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    contact_type VARCHAR(20) NOT NULL,
    contact_value VARCHAR(200) NOT NULL,
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    graph_node_id UUID,                                -- FK to graph_node for this contact point
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_customer ON customer_contact(customer_id);

CREATE TABLE account (
    account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    product_id UUID NOT NULL REFERENCES product(product_id),
    account_type VARCHAR(20) NOT NULL,
    account_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    current_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    available_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    hold_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    interest_rate NUMERIC(8,5),
    interest_accrued NUMERIC(18,2) NOT NULL DEFAULT 0,
    overdraft_limit NUMERIC(18,2) DEFAULT 0,
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    closed_date DATE,
    last_activity_at TIMESTAMPTZ,
    branch_id UUID,
    graph_node_id UUID,                                -- FK to graph_node for this account
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_account_number ON account(account_number);
CREATE INDEX idx_account_type ON account(account_type);
CREATE INDEX idx_account_status ON account(account_status);

CREATE TABLE account_owner (
    account_owner_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    ownership_type VARCHAR(20) NOT NULL,
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(account_id, customer_id, ownership_type)
);
CREATE INDEX idx_ao_account ON account_owner(account_id);
CREATE INDEX idx_ao_customer ON account_owner(customer_id);

CREATE TABLE product (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_type VARCHAR(20) NOT NULL,
    name VARCHAR(200) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    interest_rate_type VARCHAR(20),
    default_interest_rate NUMERIC(8,5),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE branch (
    branch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    routing_number CHAR(9) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE loan_detail (
    loan_detail_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL UNIQUE REFERENCES account(account_id),
    original_principal NUMERIC(18,2) NOT NULL,
    remaining_principal NUMERIC(18,2) NOT NULL,
    interest_rate NUMERIC(8,5) NOT NULL,
    rate_type VARCHAR(10) NOT NULL,
    payment_frequency VARCHAR(20) NOT NULL,
    payment_amount NUMERIC(18,2),
    next_payment_date DATE,
    maturity_date DATE NOT NULL,
    collateral_description TEXT,
    delinquency_days INTEGER NOT NULL DEFAULT 0,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE loan_schedule (
    schedule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    installment_number INTEGER NOT NULL,
    due_date DATE NOT NULL,
    principal_amount NUMERIC(18,2) NOT NULL,
    interest_amount NUMERIC(18,2) NOT NULL,
    total_amount NUMERIC(18,2) NOT NULL,
    paid_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_account ON loan_schedule(account_id);
```

---

## Transaction Processing

```sql
CREATE TABLE transaction (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    transaction_type VARCHAR(30) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    direction VARCHAR(6) NOT NULL,
    running_balance NUMERIC(18,2) NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(50),
    effective_date DATE NOT NULL,
    posted_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    channel VARCHAR(20),
    branch_id UUID,
    teller_id UUID,
    reversal_of UUID REFERENCES transaction(transaction_id),
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    -- Counterparty reference for graph edge creation
    counterparty_account_id UUID,                      -- internal transfer target
    counterparty_external_name VARCHAR(200),            -- external entity name (for graph projection)
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account ON transaction(account_id, effective_date);
CREATE INDEX idx_txn_status ON transaction(status);
CREATE INDEX idx_txn_type ON transaction(transaction_type);
CREATE INDEX idx_txn_counterparty ON transaction(counterparty_account_id);
```

---

## General Ledger (Same as Normalized Model)

```sql
CREATE TABLE gl_account (
    gl_account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    parent_account_id UUID REFERENCES gl_account(gl_account_id),
    normal_balance VARCHAR(6) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE gl_journal (
    journal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_number VARCHAR(30) NOT NULL UNIQUE,
    journal_date DATE NOT NULL,
    description VARCHAR(500),
    source VARCHAR(30),
    source_reference_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'posted',
    posted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_journal_date ON gl_journal(journal_date);

CREATE TABLE gl_journal_entry (
    entry_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_id UUID NOT NULL REFERENCES gl_journal(journal_id),
    gl_account_id UUID NOT NULL REFERENCES gl_account(gl_account_id),
    debit_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    credit_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    description VARCHAR(300),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    CONSTRAINT chk_debit_or_credit CHECK (
        (debit_amount > 0 AND credit_amount = 0) OR
        (credit_amount > 0 AND debit_amount = 0)
    )
);
CREATE INDEX idx_gle_journal ON gl_journal_entry(journal_id);
CREATE INDEX idx_gle_account ON gl_journal_entry(gl_account_id);

CREATE TABLE gl_period (
    period_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_name VARCHAR(30) NOT NULL,
    period_type VARCHAR(10) NOT NULL,
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Payments: ACH and Wire (Relational)

```sql
CREATE TABLE ach_file (
    ach_file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    direction VARCHAR(10) NOT NULL,
    immediate_destination VARCHAR(10) NOT NULL,
    immediate_origin VARCHAR(10) NOT NULL,
    file_creation_date DATE NOT NULL,
    total_debit NUMERIC(18,2),
    total_credit NUMERIC(18,2),
    status VARCHAR(20) NOT NULL DEFAULT 'created',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ach_batch (
    ach_batch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ach_file_id UUID NOT NULL REFERENCES ach_file(ach_file_id),
    service_class_code CHAR(3) NOT NULL,
    company_name VARCHAR(16) NOT NULL,
    standard_entry_class CHAR(3) NOT NULL,
    company_entry_description VARCHAR(10) NOT NULL,
    effective_entry_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ach_batch_file ON ach_batch(ach_file_id);

CREATE TABLE ach_entry (
    ach_entry_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ach_batch_id UUID NOT NULL REFERENCES ach_batch(ach_batch_id),
    transaction_code CHAR(2) NOT NULL,
    receiving_dfi VARCHAR(9) NOT NULL,
    dfi_account_number VARCHAR(17) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    individual_name VARCHAR(22) NOT NULL,
    trace_number VARCHAR(15) NOT NULL,
    transaction_id UUID REFERENCES transaction(transaction_id),
    return_reason_code CHAR(3),
    status VARCHAR(20) NOT NULL DEFAULT 'pending',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ach_entry_batch ON ach_entry(ach_batch_id);
CREATE INDEX idx_ach_entry_trace ON ach_entry(trace_number);

CREATE TABLE wire_transfer (
    wire_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_type VARCHAR(20) NOT NULL,
    direction VARCHAR(10) NOT NULL,
    message_id VARCHAR(50) NOT NULL UNIQUE,
    instruction_id VARCHAR(35),
    end_to_end_id VARCHAR(35),
    transaction_id UUID REFERENCES transaction(transaction_id),
    amount NUMERIC(18,2) NOT NULL,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    debtor_name VARCHAR(140),
    debtor_account VARCHAR(34),
    creditor_name VARCHAR(140),
    creditor_account VARCHAR(34),
    creditor_agent_bic VARCHAR(11),
    creditor_street VARCHAR(200),
    creditor_city VARCHAR(100),
    creditor_country CHAR(2),
    purpose_code VARCHAR(10),
    remittance_info TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'initiated',
    ofac_screened BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wire_message ON wire_transfer(message_id);
```

---

## BSA/AML Compliance (Graph-Enhanced)

```sql
CREATE TABLE aml_alert (
    alert_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_number VARCHAR(20) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customer(customer_id),
    account_id UUID REFERENCES account(account_id),
    alert_type VARCHAR(30) NOT NULL,
    alert_source VARCHAR(20) NOT NULL,                 -- 'rule_engine', 'ai_model', 'graph_analysis', 'manual'
    severity VARCHAR(10) NOT NULL,
    ai_confidence_score NUMERIC(5,4),
    ai_explanation TEXT,
    -- Graph-specific enrichment
    graph_context JSONB,                               -- subgraph snapshot at alert time
    -- graph_context example:
    -- {
    --   "center_node": "customer-uuid",
    --   "connected_accounts": 5,
    --   "shared_address_customers": 2,
    --   "shared_device_customers": 1,
    --   "transfer_network_size": 8,
    --   "suspicious_edges": [
    --     {"from": "acct-001", "to": "acct-007", "type": "TRANSFERRED_TO", "amount": 9800},
    --     {"from": "acct-001", "to": "acct-012", "type": "TRANSFERRED_TO", "amount": 9700}
    --   ],
    --   "network_risk_score": 0.87
    -- }
    graph_node_id UUID,                                -- FK to the graph node that's the alert subject
    triggering_transactions UUID[],
    status VARCHAR(20) NOT NULL DEFAULT 'new',
    assigned_to UUID,
    reviewed_at TIMESTAMPTZ,
    disposition VARCHAR(30),
    disposition_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_customer ON aml_alert(customer_id);
CREATE INDEX idx_alert_status ON aml_alert(status);
CREATE INDEX idx_alert_source ON aml_alert(alert_source);
CREATE INDEX idx_alert_graph ON aml_alert USING GIN (graph_context jsonb_path_ops);

CREATE TABLE suspicious_activity_report (
    sar_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sar_number VARCHAR(30) NOT NULL UNIQUE,
    alert_id UUID REFERENCES aml_alert(alert_id),
    subject_customer_id UUID REFERENCES customer(customer_id),
    subject_name VARCHAR(300),
    activity_start_date DATE NOT NULL,
    activity_end_date DATE,
    total_amount NUMERIC(18,2),
    activity_categories VARCHAR(200)[],
    filing_institution_name VARCHAR(200) NOT NULL,
    filing_institution_ein VARCHAR(10) NOT NULL,
    narrative TEXT NOT NULL,
    -- Graph analysis included in narrative evidence
    graph_analysis_summary TEXT,                       -- human-readable summary of network analysis
    fincen_bsa_id VARCHAR(20),
    filing_status VARCHAR(20) NOT NULL DEFAULT 'draft',
    filed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sar_status ON suspicious_activity_report(filing_status);

CREATE TABLE currency_transaction_report (
    ctr_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ctr_number VARCHAR(30) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customer(customer_id),
    transaction_date DATE NOT NULL,
    total_cash_in NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_cash_out NUMERIC(18,2) NOT NULL DEFAULT 0,
    transaction_ids UUID[],
    fincen_bsa_id VARCHAR(20),
    filing_status VARCHAR(20) NOT NULL DEFAULT 'draft',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## User & Access (RBAC)

```sql
CREATE TABLE app_user (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(200) NOT NULL UNIQUE,
    password_hash VARCHAR(200),
    full_name VARCHAR(200) NOT NULL,
    branch_id UUID REFERENCES branch(branch_id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(300),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    permission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permission_code VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(300),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role_permission (
    role_id UUID NOT NULL REFERENCES role(role_id),
    permission_id UUID NOT NULL REFERENCES permission(permission_id),
    PRIMARY KEY (role_id, permission_id)
);

CREATE TABLE user_role (
    user_id UUID NOT NULL REFERENCES app_user(user_id),
    role_id UUID NOT NULL REFERENCES role(role_id),
    branch_id UUID REFERENCES branch(branch_id),
    PRIMARY KEY (user_id, role_id)
);
```

---

## Audit Logging

```sql
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(50) NOT NULL,
    entity_type VARCHAR(30) NOT NULL,
    entity_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL,
    old_values JSONB,
    new_values JSONB,
    user_id UUID,
    ip_address INET,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
```

---

## Graph Query Examples

```sql
-- 1. Find all accounts owned by a customer and their direct transfer partners (2-hop traversal)
WITH customer_accounts AS (
    SELECT e.target_node_id AS account_node_id
    FROM graph_edge e
    WHERE e.source_node_id = :customer_graph_node_id
      AND e.edge_type = 'OWNS_ACCOUNT'
),
transfer_partners AS (
    SELECT DISTINCT e.target_node_id AS partner_node_id,
           e.properties->>'amount' AS total_transferred,
           e.properties->>'count' AS transfer_count
    FROM graph_edge e
    JOIN customer_accounts ca ON e.source_node_id = ca.account_node_id
    WHERE e.edge_type = 'TRANSFERRED_TO'
)
SELECT n.label, n.properties, tp.total_transferred, tp.transfer_count
FROM transfer_partners tp
JOIN graph_node n ON n.node_id = tp.partner_node_id;

-- 2. Find all customers who share an address with a flagged customer (AML investigation)
WITH RECURSIVE network AS (
    -- Start from the flagged customer
    SELECT e.target_node_id AS node_id, 1 AS depth, ARRAY[e.source_node_id] AS path
    FROM graph_edge e
    WHERE e.source_node_id = :flagged_customer_node_id
      AND e.edge_type IN ('SHARES_ADDRESS', 'SHARES_PHONE', 'SHARES_DEVICE')
    
    UNION ALL
    
    -- Expand to 2 hops
    SELECT e.target_node_id, n.depth + 1, n.path || e.source_node_id
    FROM graph_edge e
    JOIN network n ON e.source_node_id = n.node_id
    WHERE n.depth < 2
      AND e.edge_type IN ('SHARES_ADDRESS', 'SHARES_PHONE', 'SHARES_DEVICE')
      AND NOT (e.target_node_id = ANY(n.path))  -- prevent cycles
)
SELECT DISTINCT gn.node_id, gn.label, gn.properties, nw.depth
FROM network nw
JOIN graph_node gn ON gn.node_id = nw.node_id
WHERE gn.node_type = 'customer';

-- 3. Detect structuring: find accounts that received multiple cash deposits just below $10,000
--    from customers who share attributes
SELECT
    source_cust.label AS customer_name,
    source_cust.properties->>'risk_rating' AS risk,
    COUNT(DISTINCT shared.target_node_id) AS shared_attribute_count,
    array_agg(DISTINCT shared.edge_type) AS shared_types
FROM graph_edge owns
JOIN graph_node acct ON owns.target_node_id = acct.node_id AND acct.node_type = 'account'
JOIN graph_node source_cust ON owns.source_node_id = source_cust.node_id
JOIN graph_edge shared ON shared.source_node_id = source_cust.node_id
    AND shared.edge_type IN ('SHARES_ADDRESS', 'SHARES_PHONE', 'SHARES_DEVICE')
WHERE owns.edge_type = 'OWNS_ACCOUNT'
  AND acct.node_id IN (
      -- accounts with suspicious cash deposit patterns (from relational side)
      SELECT a.graph_node_id FROM account a
      JOIN transaction t ON t.account_id = a.account_id
      WHERE t.transaction_type = 'deposit'
        AND t.channel = 'teller'
        AND t.amount BETWEEN 8000 AND 9999
        AND t.effective_date >= CURRENT_DATE - INTERVAL '30 days'
      GROUP BY a.graph_node_id
      HAVING COUNT(*) >= 3
  )
GROUP BY source_cust.node_id, source_cust.label, source_cust.properties->>'risk_rating';

-- 4. Calculate network risk score for a customer (number of high-risk connections)
SELECT
    COUNT(*) FILTER (WHERE target_risk.properties->>'risk_rating' = 'high') AS high_risk_connections,
    COUNT(*) FILTER (WHERE flagged.edge_id IS NOT NULL) AS flagged_connections,
    COUNT(DISTINCT e.target_node_id) AS total_connections
FROM graph_edge e
JOIN graph_node target_risk ON e.target_node_id = target_risk.node_id
LEFT JOIN graph_edge flagged ON flagged.source_node_id = e.target_node_id
    AND flagged.edge_type = 'FLAGGED_WITH'
WHERE e.source_node_id = :customer_graph_node_id
  AND e.edge_type IN ('SHARES_ADDRESS', 'SHARES_PHONE', 'TRANSFERRED_TO', 'SHARES_DEVICE');
```

---

## Graph Maintenance (Event-Driven Projection)

The graph layer is maintained by an event-driven projection process that listens to changes in the relational tables:

```sql
-- Example: when a new account_owner row is inserted, create graph nodes and edges
-- (This would be triggered by an application-level event handler, not a DB trigger)

-- Step 1: Ensure customer node exists
INSERT INTO graph_node (node_id, node_type, entity_id, label, properties)
VALUES (:customer_graph_node_id, 'customer', :customer_id, :customer_name,
        jsonb_build_object('customer_number', :customer_number, 'risk_rating', :risk_rating))
ON CONFLICT (node_id) DO UPDATE SET
    properties = EXCLUDED.properties,
    last_seen_at = now();

-- Step 2: Ensure account node exists
INSERT INTO graph_node (node_id, node_type, entity_id, label, properties)
VALUES (:account_graph_node_id, 'account', :account_id, :account_number,
        jsonb_build_object('account_number', :account_number, 'account_type', :account_type))
ON CONFLICT (node_id) DO UPDATE SET
    properties = EXCLUDED.properties,
    last_seen_at = now();

-- Step 3: Create ownership edge
INSERT INTO graph_edge (source_node_id, target_node_id, edge_type, properties)
VALUES (:customer_graph_node_id, :account_graph_node_id, 'OWNS_ACCOUNT',
        jsonb_build_object('ownership_type', :ownership_type, 'since', CURRENT_DATE));

-- Step 4: Detect shared addresses and create SHARES_ADDRESS edges
INSERT INTO graph_edge (source_node_id, target_node_id, edge_type, properties)
SELECT :customer_graph_node_id, other_cust.graph_node_id, 'SHARES_ADDRESS',
       jsonb_build_object('address_id', :address_id)
FROM customer_address ca
JOIN customer other_cust ON other_cust.customer_id != :customer_id
JOIN customer_address other_addr ON other_addr.customer_id = other_cust.customer_id
WHERE ca.customer_id = :customer_id
  AND ca.postal_code = other_addr.postal_code
  AND ca.street_name = other_addr.street_name
  AND ca.building_number = other_addr.building_number
ON CONFLICT DO NOTHING;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer | 2 | graph_node, graph_edge |
| Customer & Identity | 4 | customer, customer_address, customer_contact, customer_relationship (not shown but same as Model 1) |
| Account Management | 6 | account, account_owner, product, branch, loan_detail, loan_schedule |
| Transaction Processing | 1 | transaction (with counterparty references for graph projection) |
| General Ledger | 4 | gl_account, gl_journal, gl_journal_entry, gl_period |
| Payments: ACH | 3 | ach_file, ach_batch, ach_entry |
| Payments: Wire | 1 | wire_transfer |
| BSA/AML Compliance | 3 | aml_alert (with graph_context), suspicious_activity_report, currency_transaction_report |
| User & Access (RBAC) | 5 | app_user, role, permission, role_permission, user_role |
| Audit | 1 | audit_log |
| **Total** | **30** | 28 relational + 2 graph tables |

---

## Key Design Decisions

1. **Graph as analytical overlay, not operational store** — the relational tables remain the system of record for all CRUD operations. The graph layer is a projection that enables relationship queries without replacing standard banking workflows. If the graph is corrupted, it can be rebuilt from relational data.

2. **Two graph tables instead of a separate database** — using `graph_node` and `graph_edge` tables in PostgreSQL avoids the operational complexity of running Neo4j or Amazon Neptune alongside the relational database. Recursive CTEs handle traversals up to 4-6 hops efficiently, which covers the vast majority of AML investigation patterns.

3. **Edge weight for ML scoring** — the `weight` column on `graph_edge` supports graph-based ML algorithms (PageRank, community detection, link prediction) that feed into the AI-native fraud and AML scoring pipeline.

4. **Graph context snapshot on AML alerts** — when an AML alert is generated, the `graph_context` JSONB captures a snapshot of the relevant subgraph at alert time. This ensures that even if the graph evolves, the examiner can see exactly what the network looked like when the alert was triggered.

5. **Counterparty references on transactions** — the `transaction` table includes `counterparty_account_id` and `counterparty_external_name` fields that simplify graph edge creation for transfer relationships without requiring a separate lookup.

6. **`graph_node_id` on relational entities** — customer, account, address, and contact tables carry a `graph_node_id` reference that links them to their graph representation, enabling efficient bidirectional lookups.

7. **Shared-attribute detection** — edges like `SHARES_ADDRESS`, `SHARES_PHONE`, and `SHARES_DEVICE` are created by the graph projection process whenever two customers are found to share an attribute. This is the foundation for synthetic identity detection and account takeover analysis.

8. **Alert source includes `graph_analysis`** — the `aml_alert.alert_source` field includes `'graph_analysis'` as a value, distinguishing alerts generated by graph pattern detection from rule-based or ML-based alerts.

9. **Temporal edge tracking** — `first_seen_at` and `last_seen_at` on both nodes and edges enable temporal analysis: "when did this connection first appear?" is a critical question for AML investigations.

10. **OCSF-aligned edge events** — graph edge creation and updates follow patterns compatible with OCSF (Open Cybersecurity Schema Framework) event structures, enabling integration with SIEM systems and security monitoring tools.
