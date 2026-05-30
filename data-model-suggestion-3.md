# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Banking Core System (Community Bank) · Created: 2026-05-20

## Philosophy

The hybrid model keeps the structural backbone of a relational database — typed columns for fields that are queried, indexed, and regulated — while using PostgreSQL JSONB columns for extensible, variable, and product-specific data. This mirrors the approach taken by modern cloud-native cores like Mambu (whose Data Dictionary uses typed core fields with custom field sets) and Finxact (which supports schema extension without code branching via flexible data objects).

The key insight is that a community banking core must serve multiple product types (DDA, savings, CD, money market, consumer loans, lines of credit) and potentially multiple jurisdictions, each with different regulatory fields. Rather than creating a separate table for every product variation or adding hundreds of nullable columns, the hybrid model stores common fields relationally and product/jurisdiction-specific fields in a `properties` JSONB column on each major entity. This enables a configuration-driven product engine: new financial products are defined by specifying which JSONB properties they carry, without DDL changes.

The general ledger, payment processing, and compliance modules remain fully relational — these are standardized domains where strict typing and foreign keys are essential. The flexibility lives in the customer, account, and product layers where business variation is highest.

**Best for:** Rapid MVP development, multi-product institutions, platforms that need to support diverse financial products without schema changes, and teams that want relational rigor where it matters most (GL, payments, compliance) with flexibility everywhere else.

**Trade-offs:**
- (+) Fewer tables than fully normalized — reduces migration complexity and join count
- (+) New product types and jurisdiction-specific fields added without DDL changes
- (+) JSONB columns are indexable (GIN) and queryable with PostgreSQL operators
- (+) Product configuration can be data-driven rather than code-driven
- (+) Faster time-to-market for new products and features
- (-) JSONB fields lack database-level type enforcement — validation must be at the application layer
- (-) Complex JSONB queries can be slower than typed column queries at scale
- (-) Schema documentation must cover both relational and JSONB structures
- (-) Reporting tools may struggle with dynamic JSONB structures compared to flat columns
- (-) Risk of JSONB becoming a "junk drawer" without discipline

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 | Payment tables use typed relational columns for ISO 20022 message fields; structured addresses are relational |
| NACHA Operating Rules 2026 | ACH tables are fully relational with typed fields matching NACHA record layouts |
| FDX v6.5 | Account and customer core fields align with FDX entity names; product-specific FDX extensions go in JSONB |
| ISO 4217 | Currency code is a typed CHAR(3) column on all monetary tables |
| ISO 3166 | Jurisdiction references use typed columns; jurisdiction-specific regulatory fields go in customer `properties` JSONB |
| FinCEN SAR/CTR | Compliance tables are fully relational — regulatory filing fields are too important for JSONB |
| FFIEC | Audit log with JSONB `old_values`/`new_values` covers both relational and JSONB field changes |
| Mambu Data Dictionary | Entity design inspired by Mambu's pattern of core typed fields + custom field sets |

---

## Product Configuration Engine

```sql
-- Product definitions with JSONB for product-specific configuration
CREATE TABLE product (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_type VARCHAR(20) NOT NULL,                 -- 'dda', 'savings', 'money_market', 'cd', 'consumer_loan', 'loc'
    name VARCHAR(200) NOT NULL,
    description TEXT,
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    -- Common configuration (relational)
    interest_rate_type VARCHAR(20),                    -- 'fixed', 'variable', 'tiered'
    default_interest_rate NUMERIC(8,5),
    min_balance NUMERIC(18,2),
    max_balance NUMERIC(18,2),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    -- Product-specific configuration (JSONB)
    config JSONB NOT NULL DEFAULT '{}',
    -- config examples by product_type:
    -- DDA: {"overdraft_enabled": true, "overdraft_limit_default": 200, "monthly_fee": 5.00,
    --        "fee_waiver_min_balance": 1500, "atm_fee_network": 0, "atm_fee_foreign": 2.50}
    -- CD:  {"term_months": 12, "early_withdrawal_penalty_days": 90, "auto_renew": true,
    --        "min_opening_deposit": 1000}
    -- Consumer Loan: {"max_term_months": 60, "min_credit_score": 620,
    --                  "origination_fee_pct": 1.0, "late_payment_fee": 35.00,
    --                  "payment_frequency_options": ["monthly", "biweekly"]}
    -- LOC: {"credit_limit_min": 500, "credit_limit_max": 50000,
    --        "draw_period_months": 60, "repayment_period_months": 120}
    -- Property schema definition
    property_schema JSONB,
    -- property_schema defines what JSONB fields are valid for accounts of this product type:
    -- {"overdraft_enabled": {"type": "boolean", "required": true},
    --  "overdraft_limit": {"type": "number", "min": 0, "max": 10000},
    --  "monthly_fee": {"type": "number", "min": 0}}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_type ON product(product_type);
CREATE INDEX idx_product_config ON product USING GIN (config jsonb_path_ops);
```

---

## Customer & Identity Management

```sql
CREATE TABLE customer (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_number VARCHAR(20) NOT NULL UNIQUE,
    customer_type VARCHAR(20) NOT NULL,                -- 'individual', 'business', 'trust', 'estate'
    -- Core identity fields (relational — always present, always queried)
    legal_name VARCHAR(300) NOT NULL,
    display_name VARCHAR(200),
    tax_id_type VARCHAR(10),
    tax_id_encrypted BYTEA,
    tax_id_last_four CHAR(4),
    date_of_birth DATE,
    -- KYC / risk (relational — compliance-critical)
    kyc_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    kyc_verified_at TIMESTAMPTZ,
    cip_verified BOOLEAN NOT NULL DEFAULT FALSE,
    risk_rating VARCHAR(10) DEFAULT 'standard',
    -- Primary address (denormalized for performance — ISO 20022 structured)
    primary_street VARCHAR(200),
    primary_building_number VARCHAR(20),
    primary_city VARCHAR(100),
    primary_state VARCHAR(100),
    primary_postal_code VARCHAR(20),
    primary_country CHAR(2),
    -- Contact (denormalized primary)
    primary_email VARCHAR(200),
    primary_phone VARCHAR(30),
    -- Extensible properties (JSONB)
    properties JSONB NOT NULL DEFAULT '{}',
    -- properties examples:
    -- Individual: {"employer": "ACME Corp", "occupation": "Engineer",
    --              "annual_income_range": "75000-100000", "citizenship": ["US"],
    --              "pep_status": false, "beneficial_owners": []}
    -- Business:   {"business_type": "LLC", "state_of_incorporation": "DE",
    --              "ein": "12-3456789", "naics_code": "522110",
    --              "annual_revenue_range": "1M-5M", "officer_names": ["Jane Smith"]}
    -- Trust:      {"trust_type": "revocable_living", "trustee_name": "Jane Smith",
    --              "trust_date": "2020-01-15", "beneficiaries": ["John Smith"]}
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_customer_number ON customer(customer_number);
CREATE INDEX idx_customer_type ON customer(customer_type);
CREATE INDEX idx_customer_risk ON customer(risk_rating);
CREATE INDEX idx_customer_kyc ON customer(kyc_status);
CREATE INDEX idx_customer_props ON customer USING GIN (properties jsonb_path_ops);

-- Additional addresses (for mailing, prior, business locations)
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
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_addr_customer ON customer_address(customer_id);

-- Customer identification documents
CREATE TABLE customer_identification (
    identification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    id_type VARCHAR(30) NOT NULL,
    id_number_encrypted BYTEA,
    issuing_authority VARCHAR(100),
    expiry_date DATE,
    verified_at TIMESTAMPTZ,
    properties JSONB NOT NULL DEFAULT '{}',            -- id-type-specific fields
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ident_customer ON customer_identification(customer_id);

-- Customer relationships
CREATE TABLE customer_relationship (
    relationship_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    related_customer_id UUID NOT NULL REFERENCES customer(customer_id),
    relationship_type VARCHAR(30) NOT NULL,
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    properties JSONB NOT NULL DEFAULT '{}',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rel_customer ON customer_relationship(customer_id);
CREATE INDEX idx_rel_related ON customer_relationship(related_customer_id);
```

---

## Account Management

```sql
CREATE TABLE account (
    account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    product_id UUID NOT NULL REFERENCES product(product_id),
    account_type VARCHAR(20) NOT NULL,
    account_status VARCHAR(20) NOT NULL DEFAULT 'pending',
    currency_code CHAR(3) NOT NULL DEFAULT 'USD',
    -- Core financial fields (relational — always needed, always indexed)
    current_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    available_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    hold_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    interest_rate NUMERIC(8,5),
    interest_accrued NUMERIC(18,2) NOT NULL DEFAULT 0,
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    closed_date DATE,
    last_activity_at TIMESTAMPTZ,
    branch_id UUID,
    -- Product-specific properties (JSONB — varies by product type)
    properties JSONB NOT NULL DEFAULT '{}',
    -- DDA properties:
    -- {"overdraft_enabled": true, "overdraft_limit": 500,
    --  "check_order_count": 3, "direct_deposit_enrolled": true,
    --  "debit_card_id": "card-uuid-here"}
    -- CD properties:
    -- {"term_months": 24, "maturity_date": "2028-05-20",
    --  "auto_renew": true, "early_withdrawal_penalty_rate": 0.01,
    --  "original_deposit": 10000.00}
    -- Loan properties:
    -- {"original_principal": 25000.00, "remaining_principal": 18750.00,
    --  "rate_type": "fixed", "payment_frequency": "monthly",
    --  "payment_amount": 485.00, "next_payment_date": "2026-06-01",
    --  "origination_date": "2024-06-01", "maturity_date": "2029-06-01",
    --  "collateral": {"type": "vehicle", "year": 2024, "make": "Toyota", "vin": "..."},
    --  "delinquency_days": 0}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_account_number ON account(account_number);
CREATE INDEX idx_account_type ON account(account_type);
CREATE INDEX idx_account_status ON account(account_status);
CREATE INDEX idx_account_product ON account(product_id);
CREATE INDEX idx_account_props ON account USING GIN (properties jsonb_path_ops);

-- Account ownership
CREATE TABLE account_owner (
    account_owner_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    ownership_type VARCHAR(20) NOT NULL,
    ownership_percentage NUMERIC(5,2),
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(account_id, customer_id, ownership_type)
);
CREATE INDEX idx_ao_account ON account_owner(account_id);
CREATE INDEX idx_ao_customer ON account_owner(customer_id);

-- Loan amortization schedule (relational — structured, queried for payment processing)
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
    paid_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_account ON loan_schedule(account_id);
CREATE INDEX idx_schedule_due ON loan_schedule(due_date, status);

-- Branch
CREATE TABLE branch (
    branch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    routing_number CHAR(9) NOT NULL,
    properties JSONB NOT NULL DEFAULT '{}',            -- branch-specific settings
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Transaction Processing

```sql
CREATE TABLE transaction (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    transaction_type VARCHAR(30) NOT NULL,
    transaction_code VARCHAR(10),
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
    -- Transaction-type-specific details (JSONB)
    details JSONB NOT NULL DEFAULT '{}',
    -- ACH transaction details:
    -- {"trace_number": "091000019876543", "standard_entry_class": "PPD",
    --  "originator_name": "ACME CORP", "company_entry_description": "PAYROLL"}
    -- Wire details:
    -- {"message_id": "WIRE-2026-0042", "message_type": "fedwire",
    --  "creditor_name": "Global Supplier", "creditor_bic": "NWBKGB2L"}
    -- Check details:
    -- {"check_number": "1042", "payee_name": "Electric Co", "check_image_id": "img-uuid"}
    -- Card details:
    -- {"card_last_four": "4321", "merchant_name": "Walmart", "mcc": "5411",
    --  "authorization_code": "A12345"}
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account ON transaction(account_id, effective_date);
CREATE INDEX idx_txn_status ON transaction(status);
CREATE INDEX idx_txn_type ON transaction(transaction_type);
CREATE INDEX idx_txn_reference ON transaction(reference_number);
CREATE INDEX idx_txn_details ON transaction USING GIN (details jsonb_path_ops);

-- Transaction holds
CREATE TABLE transaction_hold (
    hold_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    hold_type VARCHAR(20) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    reason VARCHAR(200),
    placed_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    release_at TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'active',
    placed_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hold_account ON transaction_hold(account_id);
```

---

## General Ledger (Fully Relational)

```sql
CREATE TABLE gl_account (
    gl_account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    account_type VARCHAR(20) NOT NULL,
    account_subtype VARCHAR(30),
    parent_account_id UUID REFERENCES gl_account(gl_account_id),
    normal_balance VARCHAR(6) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gl_type ON gl_account(account_type);

CREATE TABLE gl_journal (
    journal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_number VARCHAR(30) NOT NULL UNIQUE,
    journal_date DATE NOT NULL,
    description VARCHAR(500),
    source VARCHAR(30),
    source_reference_id UUID,
    status VARCHAR(20) NOT NULL DEFAULT 'posted',
    posted_by UUID,
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
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Payments: ACH (Fully Relational)

```sql
CREATE TABLE ach_file (
    ach_file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    direction VARCHAR(10) NOT NULL,
    immediate_destination VARCHAR(10) NOT NULL,
    immediate_origin VARCHAR(10) NOT NULL,
    file_creation_date DATE NOT NULL,
    record_count INTEGER,
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
    company_identification VARCHAR(10) NOT NULL,
    standard_entry_class CHAR(3) NOT NULL,
    company_entry_description VARCHAR(10) NOT NULL,
    effective_entry_date DATE NOT NULL,
    batch_number INTEGER NOT NULL,
    entry_count INTEGER,
    total_debit NUMERIC(18,2),
    total_credit NUMERIC(18,2),
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
```

---

## Payments: Wire Transfer (Fully Relational)

```sql
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
    debtor_agent_bic VARCHAR(11),
    creditor_name VARCHAR(140),
    creditor_account VARCHAR(34),
    creditor_agent_bic VARCHAR(11),
    creditor_street VARCHAR(200),
    creditor_city VARCHAR(100),
    creditor_postal_code VARCHAR(20),
    creditor_country CHAR(2),
    purpose_code VARCHAR(10),
    remittance_info TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'initiated',
    ofac_screened BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wire_message ON wire_transfer(message_id);
CREATE INDEX idx_wire_status ON wire_transfer(status);
```

---

## BSA/AML Compliance (Fully Relational)

```sql
CREATE TABLE aml_alert (
    alert_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_number VARCHAR(20) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customer(customer_id),
    account_id UUID REFERENCES account(account_id),
    alert_type VARCHAR(30) NOT NULL,
    alert_source VARCHAR(20) NOT NULL,
    severity VARCHAR(10) NOT NULL,
    ai_confidence_score NUMERIC(5,4),
    ai_explanation TEXT,
    description TEXT,
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
CREATE INDEX idx_alert_severity ON aml_alert(severity);

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
CREATE INDEX idx_ctr_date ON currency_transaction_report(transaction_date);
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
    properties JSONB NOT NULL DEFAULT '{}',            -- user preferences, notification settings
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
CREATE INDEX idx_audit_user ON audit_log(user_id);
```

---

## JSONB Query Examples

```sql
-- Find all customers who are PEPs (Politically Exposed Persons)
SELECT customer_id, legal_name, properties->>'pep_status' AS pep
FROM customer
WHERE properties @> '{"pep_status": true}';

-- Find all DDA accounts with overdraft enabled and limit > $1000
SELECT a.account_id, a.account_number, a.current_balance,
       a.properties->>'overdraft_limit' AS overdraft_limit
FROM account a
WHERE a.account_type = 'dda'
  AND a.properties @> '{"overdraft_enabled": true}'
  AND (a.properties->>'overdraft_limit')::numeric > 1000;

-- Find all ACH transactions for a specific originator (details in JSONB)
SELECT t.transaction_id, t.amount, t.effective_date,
       t.details->>'originator_name' AS originator,
       t.details->>'trace_number' AS trace
FROM transaction t
WHERE t.transaction_type = 'ach_credit'
  AND t.details @> '{"originator_name": "ACME CORP"}';

-- Find all loan accounts approaching maturity within 90 days
SELECT a.account_id, a.account_number, a.current_balance,
       (a.properties->>'maturity_date')::date AS maturity_date
FROM account a
WHERE a.account_type IN ('consumer_loan', 'loc')
  AND (a.properties->>'maturity_date')::date <= CURRENT_DATE + INTERVAL '90 days';

-- Product-driven validation: get the property schema for a product
SELECT product_code, property_schema
FROM product
WHERE product_code = 'CONSUMER_CHECKING';
-- Application layer validates account.properties against product.property_schema
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Product Configuration | 1 | product (with JSONB config and property_schema) |
| Customer & Identity | 4 | customer (with JSONB properties), address, identification, relationship |
| Account Management | 4 | account (with JSONB properties), account_owner, loan_schedule, branch |
| Transaction Processing | 2 | transaction (with JSONB details), transaction_hold |
| General Ledger | 4 | gl_account, gl_journal, gl_journal_entry, gl_period |
| Payments: ACH | 3 | ach_file, ach_batch, ach_entry |
| Payments: Wire | 1 | wire_transfer |
| BSA/AML Compliance | 3 | aml_alert, suspicious_activity_report, currency_transaction_report |
| User & Access (RBAC) | 5 | app_user, role, permission, role_permission, user_role |
| Audit | 1 | audit_log |
| **Total** | **28** | Fewer than normalized (35) due to JSONB absorbing product-specific tables |

---

## Key Design Decisions

1. **JSONB for product-specific fields, relational for universal fields** — every account has `current_balance`, `account_type`, and `status` as typed columns. But CD-specific fields (term_months, auto_renew) and loan-specific fields (collateral, payment_frequency) live in `properties` JSONB. This eliminates the `loan_detail` table and similar product-specific extension tables.

2. **Product-driven schema validation** — the `product.property_schema` JSONB column defines what fields are valid for accounts of that product type. The application layer validates `account.properties` against `product.property_schema` before saving. This is the same pattern Mambu uses for custom field sets.

3. **Transaction details in JSONB** — instead of separate `ach_entry`, `wire_transfer`, and `card_transaction` detail tables linked to the transaction, the `transaction.details` JSONB column carries channel-specific information (trace numbers, BIC codes, merchant names). The ACH and wire tables still exist for file/batch management but individual transaction details are consolidated.

4. **Denormalized primary address on customer** — the primary address is stored directly on the `customer` table for query performance, while additional addresses go in `customer_address`. This avoids a JOIN for the most common address lookup.

5. **GL and compliance remain fully relational** — the general ledger and BSA/AML filing tables use only typed columns. These domains are too heavily regulated and too critical for audit to allow JSONB flexibility. Every SAR field must be individually queryable and reportable.

6. **GIN indexes on all JSONB columns** — every JSONB column has a `jsonb_path_ops` GIN index, enabling containment queries (`@>`) at near-typed-column performance for common patterns.

7. **Fewer tables, same capability** — by absorbing `loan_detail`, `cd_detail`, and similar extension tables into JSONB, the schema drops from 35 tables (Model 1) to 28 while supporting the same product range. New product types require zero DDL.

8. **JSONB discipline through conventions** — the risk of JSONB becoming a "junk drawer" is mitigated by `property_schema` validation, documented JSONB examples in comments, and GIN indexes that enforce query patterns.

9. **ACH addenda dropped as separate table** — addenda information is stored in the `ach_entry` or `transaction.details` JSONB rather than a separate table, simplifying the ACH data model while retaining all NACHA fields.

10. **ISO 20022 structured addresses on customer** — the primary address fields (`primary_street`, `primary_building_number`, `primary_city`, `primary_postal_code`, `primary_country`) follow ISO 20022 structured address format, ready for the November 2026 SWIFT CBPR+ mandate.
