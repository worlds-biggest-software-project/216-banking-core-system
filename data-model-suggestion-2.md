# Data Model Suggestion 2: Event-Sourced / Audit-First (CQRS)

> Project: Banking Core System (Community Bank) · Created: 2026-05-20

## Philosophy

The event-sourced model treats every state change in the banking system as an immutable event appended to a log. Rather than storing "current balance = $5,000" and overwriting it with each transaction, the system stores the complete sequence of events — AccountOpened, DepositReceived, WithdrawalProcessed, InterestAccrued — and derives the current state by replaying them. Combined with CQRS (Command Query Responsibility Segregation), the system separates write operations (commands that produce events) from read operations (queries against materialised projections).

This pattern is used by modern core banking platforms like Thought Machine Vault Core (which stores product behaviour as auditable smart contracts on an event-driven architecture) and Mambu (which uses streaming APIs for event-driven integrations). Financial regulators increasingly require institutions to demonstrate "what was true on date X" for any entity — a capability that event sourcing provides natively through temporal replay, whereas traditional schemas require complex versioning or audit trigger systems.

The event store becomes the single source of truth. Materialised read models (projections) are rebuilt from the event stream as needed — one projection for the teller screen, another for regulatory reporting, another for the AI/ML pipeline. If a new reporting requirement emerges, a new projection is built by replaying historical events without any schema migration.

**Best for:** Institutions that need complete audit trails, temporal querying ("what was the account state on March 15?"), AI-native analytics on change patterns, and the ability to add new read models without modifying the core data layer.

**Trade-offs:**
- (+) Perfect, immutable audit trail — satisfies FFIEC examination requirements by design
- (+) Temporal queries are trivial: replay events to any point in time
- (+) New read models can be added without schema changes to the write side
- (+) AI/ML pipelines consume the same event stream as the core, eliminating data latency
- (+) Natural fit for BSA/AML: every financial event is already in a structured stream
- (-) Higher storage requirements — events are never deleted
- (-) Read model consistency is eventually consistent (not strongly consistent)
- (-) Debugging requires understanding event replay, not just reading a row
- (-) Schema evolution of event types requires careful versioning
- (-) Reporting queries hit projections, not the source of truth directly
- (-) Engineering team must understand event sourcing patterns — steeper learning curve

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 | Payment-related events carry ISO 20022 message identifiers and structured address data in event payloads |
| NACHA Operating Rules 2026 | ACH events map to NACHA record types; the ACH projection materialises files directly from the event stream |
| FDX v6.5 | Read-model projections for consumer data sharing align with FDX entity names and field structures |
| FFIEC IT Handbook | The immutable event store IS the audit trail; no separate audit log mechanism needed |
| FinCEN SAR/CTR | AML detection operates directly on the event stream; SAR/CTR filing events are first-class domain events |
| ISO 4217 | All monetary events include currency code per ISO 4217 |
| ISO 3166 | Jurisdiction references use ISO 3166 codes in event payloads and projections |

---

## Event Store (Write Side)

```sql
-- The immutable event store — append-only, never updated or deleted
CREATE TABLE event_store (
    event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,                           -- aggregate root ID (account_id, customer_id, etc.)
    stream_type VARCHAR(30) NOT NULL,                  -- 'account', 'customer', 'payment', 'compliance', 'ledger'
    event_type VARCHAR(80) NOT NULL,                   -- e.g., 'AccountOpened', 'DepositPosted', 'LoanDisbursed'
    event_version INTEGER NOT NULL,                    -- optimistic concurrency: monotonically increasing per stream
    event_data JSONB NOT NULL,                         -- the event payload (see examples below)
    metadata JSONB NOT NULL DEFAULT '{}',              -- correlation_id, causation_id, user_id, ip_address, channel
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, event_version)                   -- prevents concurrent writes to same aggregate
);

-- Partitioned by stream_type for query performance
CREATE INDEX idx_event_stream ON event_store(stream_id, event_version);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_event_created ON event_store(created_at);
CREATE INDEX idx_event_stream_type ON event_store(stream_type);

-- GIN index for querying inside event payloads
CREATE INDEX idx_event_data ON event_store USING GIN (event_data jsonb_path_ops);
CREATE INDEX idx_event_metadata ON event_store USING GIN (metadata jsonb_path_ops);
```

### Event Payload Examples

```sql
-- AccountOpened event
-- event_data:
-- {
--   "account_number": "1001234567",
--   "account_type": "dda",
--   "product_code": "CONSUMER_CHECKING",
--   "currency_code": "USD",
--   "customer_id": "550e8400-e29b-41d4-a716-446655440000",
--   "ownership_type": "primary",
--   "branch_code": "001",
--   "initial_deposit": 500.00,
--   "overdraft_limit": 200.00
-- }
-- metadata:
-- {
--   "correlation_id": "req-abc-123",
--   "user_id": "teller-001",
--   "channel": "teller",
--   "ip_address": "10.0.1.50",
--   "branch_id": "br-001"
-- }

-- DepositPosted event
-- event_data:
-- {
--   "transaction_type": "deposit",
--   "amount": 1500.00,
--   "currency_code": "USD",
--   "channel": "teller",
--   "reference_number": "DEP-2026-001234",
--   "description": "Cash deposit",
--   "running_balance": 2000.00
-- }

-- ACHEntryReceived event
-- event_data:
-- {
--   "direction": "inbound",
--   "standard_entry_class": "PPD",
--   "transaction_code": "22",
--   "amount": 3200.00,
--   "originator_name": "ACME CORP",
--   "company_entry_description": "PAYROLL",
--   "trace_number": "091000019876543",
--   "effective_entry_date": "2026-05-20"
-- }

-- WireTransferInitiated event (ISO 20022 pacs.008 fields)
-- event_data:
-- {
--   "message_type": "swift_pacs008",
--   "instruction_id": "INSTR-2026-0042",
--   "end_to_end_id": "E2E-2026-0042",
--   "amount": 50000.00,
--   "currency_code": "USD",
--   "debtor_name": "Jane Smith",
--   "debtor_account": "1001234567",
--   "creditor_name": "Global Supplier Inc",
--   "creditor_account": "GB29NWBK60161331926819",
--   "creditor_agent_bic": "NWBKGB2L",
--   "creditor_address": {
--     "street_name": "123 High Street",
--     "city": "London",
--     "postal_code": "EC2V 8AH",
--     "country": "GB"
--   },
--   "purpose_code": "SUPP",
--   "remittance_info": "Invoice INV-2026-5678"
-- }

-- AMLAlertGenerated event
-- event_data:
-- {
--   "alert_type": "structuring",
--   "severity": "high",
--   "source": "ai_model",
--   "ai_confidence_score": 0.8742,
--   "ai_explanation": "Multiple cash deposits below $10,000 threshold across 3 branches within 48 hours",
--   "triggering_event_ids": ["evt-001", "evt-002", "evt-003"],
--   "customer_id": "550e8400-e29b-41d4-a716-446655440000",
--   "total_amount": 28500.00,
--   "time_window_hours": 48
-- }
```

---

## Command Handlers (Application Layer)

Commands are validated and produce events. This is application code, not DDL, but the command table tracks in-flight commands for idempotency:

```sql
-- Command log for idempotency and debugging
CREATE TABLE command_log (
    command_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type VARCHAR(80) NOT NULL,                  -- 'OpenAccount', 'PostDeposit', 'InitiateWireTransfer'
    stream_id UUID NOT NULL,
    payload JSONB NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'received',    -- 'received', 'processed', 'rejected'
    rejection_reason TEXT,
    user_id UUID,
    correlation_id UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    processed_at TIMESTAMPTZ
);
CREATE INDEX idx_cmd_stream ON command_log(stream_id);
CREATE INDEX idx_cmd_correlation ON command_log(correlation_id);
CREATE INDEX idx_cmd_status ON command_log(status);
```

---

## Materialised Read Models (Projections)

Projections are rebuilt from the event stream. They can be dropped and recreated at any time.

### Account Projection

```sql
-- Current account state (materialised from AccountOpened, DepositPosted, WithdrawalProcessed, etc.)
CREATE TABLE proj_account (
    account_id UUID PRIMARY KEY,
    account_number VARCHAR(20) NOT NULL UNIQUE,
    account_type VARCHAR(20) NOT NULL,
    product_code VARCHAR(30) NOT NULL,
    account_status VARCHAR(20) NOT NULL,
    currency_code CHAR(3) NOT NULL,
    current_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    available_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    hold_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    interest_rate NUMERIC(8,5),
    interest_accrued NUMERIC(18,2) NOT NULL DEFAULT 0,
    overdraft_limit NUMERIC(18,2) DEFAULT 0,
    opened_date DATE,
    closed_date DATE,
    last_activity_at TIMESTAMPTZ,
    branch_code VARCHAR(10),
    last_event_version INTEGER NOT NULL,               -- tracks projection currency
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_acct_number ON proj_account(account_number);
CREATE INDEX idx_proj_acct_type ON proj_account(account_type);
CREATE INDEX idx_proj_acct_status ON proj_account(account_status);
```

### Customer Projection

```sql
CREATE TABLE proj_customer (
    customer_id UUID PRIMARY KEY,
    customer_number VARCHAR(20) NOT NULL UNIQUE,
    customer_type VARCHAR(20) NOT NULL,
    legal_name VARCHAR(300) NOT NULL,
    display_name VARCHAR(200),
    tax_id_last_four CHAR(4),
    kyc_status VARCHAR(20) NOT NULL,
    risk_rating VARCHAR(10),
    primary_address JSONB,                             -- denormalized current primary address
    primary_email VARCHAR(200),
    primary_phone VARCHAR(30),
    account_count INTEGER NOT NULL DEFAULT 0,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_event_version INTEGER NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_cust_number ON proj_customer(customer_number);
CREATE INDEX idx_proj_cust_risk ON proj_customer(risk_rating);
```

### Transaction History Projection

```sql
CREATE TABLE proj_transaction (
    transaction_id UUID PRIMARY KEY,
    account_id UUID NOT NULL,
    transaction_type VARCHAR(30) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    direction VARCHAR(6) NOT NULL,
    running_balance NUMERIC(18,2) NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(50),
    effective_date DATE NOT NULL,
    posted_date DATE,
    status VARCHAR(20) NOT NULL,
    channel VARCHAR(20),
    source_event_id UUID NOT NULL,                     -- links back to event_store
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_txn_account ON proj_transaction(account_id, effective_date);
CREATE INDEX idx_proj_txn_status ON proj_transaction(status);
CREATE INDEX idx_proj_txn_ref ON proj_transaction(reference_number);
```

### General Ledger Projection

```sql
-- GL balances materialised from LedgerEntryPosted events
CREATE TABLE proj_gl_account (
    gl_account_id UUID PRIMARY KEY,
    account_number VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    normal_balance VARCHAR(6) NOT NULL,
    current_debit_total NUMERIC(18,2) NOT NULL DEFAULT 0,
    current_credit_total NUMERIC(18,2) NOT NULL DEFAULT 0,
    current_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    period_id UUID,
    last_event_version INTEGER NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_gl_journal_entry (
    entry_id UUID PRIMARY KEY,
    journal_id UUID NOT NULL,
    journal_date DATE NOT NULL,
    gl_account_number VARCHAR(20) NOT NULL,
    debit_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    credit_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    description VARCHAR(500),
    source VARCHAR(30),
    source_event_id UUID NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_gl_journal ON proj_gl_journal_entry(journal_date);
CREATE INDEX idx_proj_gl_acct ON proj_gl_journal_entry(gl_account_number);
```

### ACH Projection

```sql
CREATE TABLE proj_ach_batch (
    ach_batch_id UUID PRIMARY KEY,
    ach_file_id UUID,
    direction VARCHAR(10) NOT NULL,
    service_class_code CHAR(3) NOT NULL,
    standard_entry_class CHAR(3) NOT NULL,
    company_name VARCHAR(16),
    company_entry_description VARCHAR(10),
    effective_entry_date DATE,
    entry_count INTEGER NOT NULL DEFAULT 0,
    total_debit NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_credit NUMERIC(18,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proj_ach_entry (
    ach_entry_id UUID PRIMARY KEY,
    ach_batch_id UUID NOT NULL,
    account_id UUID,
    transaction_code CHAR(2) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    individual_name VARCHAR(22),
    trace_number VARCHAR(15) NOT NULL,
    status VARCHAR(20) NOT NULL,
    return_reason_code CHAR(3),
    source_event_id UUID NOT NULL,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_ach_batch ON proj_ach_entry(ach_batch_id);
CREATE INDEX idx_proj_ach_trace ON proj_ach_entry(trace_number);
```

### Compliance Projection

```sql
CREATE TABLE proj_aml_alert (
    alert_id UUID PRIMARY KEY,
    alert_number VARCHAR(20) NOT NULL UNIQUE,
    customer_id UUID,
    alert_type VARCHAR(30) NOT NULL,
    alert_source VARCHAR(20) NOT NULL,
    severity VARCHAR(10) NOT NULL,
    ai_confidence_score NUMERIC(5,4),
    status VARCHAR(20) NOT NULL,
    assigned_to UUID,
    created_at TIMESTAMPTZ NOT NULL,
    resolved_at TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_alert_status ON proj_aml_alert(status);
CREATE INDEX idx_proj_alert_customer ON proj_aml_alert(customer_id);

CREATE TABLE proj_sar (
    sar_id UUID PRIMARY KEY,
    sar_number VARCHAR(30) NOT NULL UNIQUE,
    alert_id UUID,
    subject_name VARCHAR(300),
    activity_start_date DATE,
    activity_end_date DATE,
    total_amount NUMERIC(18,2),
    filing_status VARCHAR(20) NOT NULL,
    fincen_bsa_id VARCHAR(20),
    filed_at TIMESTAMPTZ,
    projected_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_proj_sar_status ON proj_sar(filing_status);
```

---

## Snapshot Table (Performance Optimization)

To avoid replaying millions of events for long-lived aggregates, periodic snapshots are stored:

```sql
CREATE TABLE event_snapshot (
    snapshot_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id UUID NOT NULL,
    stream_type VARCHAR(30) NOT NULL,
    snapshot_version INTEGER NOT NULL,                  -- event_version at snapshot time
    state_data JSONB NOT NULL,                          -- serialised aggregate state
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(stream_id, snapshot_version)
);
CREATE INDEX idx_snapshot_stream ON event_snapshot(stream_id, snapshot_version DESC);
```

---

## Temporal Query Examples

```sql
-- "What was the balance of account X on March 15, 2026?"
SELECT
    e.event_data->>'running_balance' AS balance,
    e.event_type,
    e.created_at
FROM event_store e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.stream_type = 'account'
  AND e.event_type IN ('DepositPosted', 'WithdrawalProcessed', 'FeeCharged', 'InterestAccrued', 'AccountOpened')
  AND e.created_at <= '2026-03-15 23:59:59-05'
ORDER BY e.event_version DESC
LIMIT 1;

-- "Show all events for a customer flagged by AML between January and March 2026"
SELECT
    e.event_type,
    e.event_data,
    e.created_at
FROM event_store e
WHERE e.stream_id = '550e8400-e29b-41d4-a716-446655440000'
  AND e.created_at BETWEEN '2026-01-01' AND '2026-03-31'
ORDER BY e.event_version;

-- "Replay all events to rebuild the GL projection from scratch"
SELECT
    e.event_id,
    e.stream_id,
    e.event_type,
    e.event_data,
    e.event_version
FROM event_store e
WHERE e.stream_type = 'ledger'
ORDER BY e.created_at, e.event_version;
```

---

## Event Catalog

| Stream Type | Event Type | Key Payload Fields |
|-------------|------------|--------------------|
| customer | CustomerOnboarded | customer_number, legal_name, customer_type, tax_id_encrypted, kyc_status |
| customer | KYCVerified | verification_method, verified_by, risk_rating |
| customer | CustomerUpdated | changed_fields, old_values, new_values |
| customer | RiskRatingChanged | old_rating, new_rating, reason |
| account | AccountOpened | account_number, account_type, product_code, customer_id, initial_deposit |
| account | DepositPosted | amount, channel, reference_number, running_balance |
| account | WithdrawalProcessed | amount, channel, reference_number, running_balance |
| account | TransferExecuted | from_account_id, to_account_id, amount |
| account | InterestAccrued | amount, rate, period_start, period_end |
| account | FeeCharged | fee_type, amount, description |
| account | HoldPlaced | hold_type, amount, release_date |
| account | HoldReleased | hold_id, amount |
| account | AccountClosed | reason, final_balance |
| account | OverdraftTriggered | amount, ai_recommendation, approved |
| loan | LoanOriginated | principal, rate, term_months, payment_frequency |
| loan | LoanDisbursed | amount, disbursement_method |
| loan | LoanPaymentReceived | principal_portion, interest_portion, total_amount |
| loan | LoanDelinquent | days_past_due, amount_past_due |
| payment | ACHBatchCreated | batch_id, entry_count, total_amount, standard_entry_class |
| payment | ACHEntryReceived | trace_number, amount, originator_name |
| payment | ACHEntryReturned | trace_number, return_reason_code |
| payment | WireTransferInitiated | message_id, amount, creditor_name, creditor_bic |
| payment | WireTransferSettled | message_id, settlement_date |
| payment | WireTransferRejected | message_id, rejection_reason |
| ledger | LedgerEntryPosted | journal_id, gl_account, debit_amount, credit_amount |
| ledger | PeriodClosed | period_name, closing_balances |
| compliance | AMLAlertGenerated | alert_type, severity, ai_confidence_score, customer_id |
| compliance | AMLAlertReviewed | alert_id, disposition, reviewed_by |
| compliance | SARFiled | sar_number, fincen_bsa_id, subject_name |
| compliance | CTRFiled | ctr_number, total_cash_in, total_cash_out |
| compliance | OFACScreeningCompleted | entity_name, match_result, screening_provider |

---

## Reference Data (Shared Across Write and Read Sides)

```sql
-- These tables are mutable reference data, not event-sourced
CREATE TABLE ref_product (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_type VARCHAR(20) NOT NULL,
    name VARCHAR(200) NOT NULL,
    currency_code CHAR(3) NOT NULL,
    interest_rate_type VARCHAR(20),
    default_interest_rate NUMERIC(8,5),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE ref_jurisdiction (
    jurisdiction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,
    subdivision_code VARCHAR(6),
    name VARCHAR(200) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

CREATE TABLE ref_currency (
    currency_code CHAR(3) PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    minor_units SMALLINT NOT NULL DEFAULT 2
);

CREATE TABLE ref_branch (
    branch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    routing_number CHAR(9) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
```

---

## User & Access (Same Pattern as Normalized Model)

```sql
CREATE TABLE app_user (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(200) NOT NULL UNIQUE,
    password_hash VARCHAR(200),
    full_name VARCHAR(200) NOT NULL,
    branch_id UUID REFERENCES ref_branch(branch_id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
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
    branch_id UUID REFERENCES ref_branch(branch_id),
    PRIMARY KEY (user_id, role_id)
);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store (write side) | 3 | event_store, command_log, event_snapshot |
| Read Projections: Account & Customer | 3 | proj_account, proj_customer, proj_transaction |
| Read Projections: Ledger | 2 | proj_gl_account, proj_gl_journal_entry |
| Read Projections: Payments | 2 | proj_ach_batch, proj_ach_entry |
| Read Projections: Compliance | 2 | proj_aml_alert, proj_sar |
| Reference Data | 4 | ref_product, ref_jurisdiction, ref_currency, ref_branch |
| User & Access (RBAC) | 5 | app_user, role, permission, role_permission, user_role |
| **Total** | **21** | Fewer tables than normalized; projections can be added/dropped freely |

---

## Key Design Decisions

1. **Single event_store table as source of truth** — all domain events across all aggregate types live in one append-only table, partitionable by `stream_type`. This eliminates the need for separate audit tables — the event store IS the audit trail.

2. **JSONB event payloads** — event data is stored as JSONB rather than typed columns, allowing the event schema to evolve without DDL migrations. New fields are added to events; old events retain their original structure. GIN indexes support efficient querying within payloads.

3. **Optimistic concurrency via event_version** — the `UNIQUE(stream_id, event_version)` constraint prevents concurrent writers from producing conflicting events for the same aggregate, replacing traditional row-level locks.

4. **Projections are disposable** — every `proj_*` table can be dropped and rebuilt from the event stream. This means new reporting requirements never require schema changes to the write side. A new FFIEC report? Build a new projection.

5. **Snapshot optimization** — for long-lived accounts with thousands of events, periodic snapshots avoid full replay. Replay starts from the latest snapshot rather than event zero.

6. **Event catalog as domain language** — the event types (AccountOpened, DepositPosted, AMLAlertGenerated) serve as a ubiquitous language shared by developers, compliance officers, and auditors. This is a direct benefit of domain-driven design.

7. **AI/ML pipeline integration** — the event stream is the natural feed for AI models. AML pattern detection, fraud scoring, and overdraft prediction all consume events in real time rather than polling a relational database. The `AMLAlertGenerated` event includes `ai_confidence_score` and `ai_explanation` fields.

8. **Correlation and causation tracking** — the `metadata` JSONB on each event carries `correlation_id` and `causation_id`, enabling full causal chain tracing from a customer action through to GL posting and compliance screening.

9. **Reference data is mutable** — product definitions, jurisdictions, currencies, and branches are traditional mutable tables, not event-sourced. These change infrequently and don't benefit from event history.

10. **No separate audit_log table** — unlike the normalized model, this design does not need a dedicated audit log. Every state change is an event with who, what, when, and why captured in the event payload and metadata. FFIEC examiners can query the event store directly.
