# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Banking Core System (Community Bank) · Created: 2026-05-20

## Philosophy

The entity-centric normalized relational model follows the traditional approach used by established core banking systems like Jack Henry SilverLake, Fiserv DNA, and Apache Fineract. Every business concept — customer, account, transaction, product, payment batch — gets its own table with strict foreign key relationships enforcing referential integrity at the database level. The schema is normalized to third normal form (3NF) or higher, eliminating data redundancy and ensuring that updates propagate consistently.

This approach mirrors how regulators think about banking data: every entity is discrete, every relationship is explicit, and every field has a defined type and constraint. The general ledger uses classical double-entry bookkeeping with separate debit and credit entries linked by a journal ID, ensuring that the books always balance. Reference data (account types, transaction codes, jurisdictions) lives in dedicated lookup tables aligned with industry standards (NACHA transaction codes, ISO 3166 country codes, ISO 20022 message types).

The model is designed for a PostgreSQL deployment and draws heavily from the Microsoft Common Data Model for Financial Services entity structure, the Apache Fineract database schema, and the FDX v6.x data element catalogue.

**Best for:** Institutions that prioritize data integrity, regulatory transparency, and compatibility with traditional banking workflows and examination processes.

**Trade-offs:**
- (+) Strong referential integrity — the database enforces business rules
- (+) Straightforward SQL queries for regulatory reporting and call reports
- (+) Well-understood by banking auditors and examiners; maps directly to FFIEC examination expectations
- (+) Mature tooling ecosystem for ORMs, migration tools, and BI platforms
- (-) High table count increases schema complexity and migration overhead
- (-) Adding jurisdiction-specific or product-specific fields requires DDL changes
- (-) Audit trail requires explicit trigger-based or application-level change capture
- (-) Many-to-many junction tables add join overhead for complex queries

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| ISO 20022 | Payment message fields (pain, pacs, camt) mapped to `payment_message` table; structured address fields on `address` table |
| NACHA Operating Rules 2026 | ACH batch/entry/addenda record types modeled as `ach_batch`, `ach_entry`, `ach_addenda` tables with standard field positions |
| FDX v6.5 | Account, transaction, and customer entities align with FDX data element names for API compatibility |
| ISO 3166 | `jurisdiction` lookup table uses ISO 3166-1 alpha-2 country codes and ISO 3166-2 subdivision codes |
| ISO 4217 | `currency` lookup table with ISO 4217 codes; all monetary columns reference a currency code |
| ISO 8583 | Card transaction interchange fields mapped to `card_transaction` table |
| FinCEN SAR/CTR | `suspicious_activity_report` and `currency_transaction_report` tables model FinCEN filing fields |
| FFIEC | Audit trail tables (`audit_log`, `access_log`) satisfy FFIEC IT Examination Handbook requirements |
| Microsoft CDM Banking | Entity naming and relationship patterns informed by Microsoft Common Data Model for Financial Services |

---

## Reference Data & Lookups

```sql
-- ISO 3166 jurisdictions
CREATE TABLE jurisdiction (
    jurisdiction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    country_code CHAR(2) NOT NULL,           -- ISO 3166-1 alpha-2
    subdivision_code VARCHAR(6),              -- ISO 3166-2 (e.g., 'US-TX')
    name VARCHAR(200) NOT NULL,
    jurisdiction_type VARCHAR(20) NOT NULL,   -- 'country', 'state', 'territory'
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_jurisdiction_country ON jurisdiction(country_code);

-- ISO 4217 currencies
CREATE TABLE currency (
    currency_code CHAR(3) PRIMARY KEY,        -- ISO 4217 (e.g., 'USD')
    name VARCHAR(100) NOT NULL,
    minor_units SMALLINT NOT NULL DEFAULT 2,
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);

-- Product catalog
CREATE TABLE product (
    product_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    product_code VARCHAR(30) NOT NULL UNIQUE,
    product_type VARCHAR(20) NOT NULL,        -- 'deposit', 'loan', 'cd', 'money_market'
    name VARCHAR(200) NOT NULL,
    description TEXT,
    currency_code CHAR(3) NOT NULL REFERENCES currency(currency_code),
    interest_rate_type VARCHAR(20),           -- 'fixed', 'variable', 'tiered'
    default_interest_rate NUMERIC(8,5),
    min_balance NUMERIC(18,2),
    max_balance NUMERIC(18,2),
    term_months INTEGER,                      -- for CDs and term loans
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_product_type ON product(product_type);
```

---

## Customer & Identity Management

```sql
-- Core customer/member record (CIF)
CREATE TABLE customer (
    customer_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_number VARCHAR(20) NOT NULL UNIQUE,  -- institution-assigned CIF number
    customer_type VARCHAR(20) NOT NULL,            -- 'individual', 'business', 'trust', 'estate'
    tax_id_type VARCHAR(10),                       -- 'SSN', 'EIN', 'ITIN'
    tax_id_encrypted BYTEA,                        -- encrypted TIN (PCI/PII protection)
    tax_id_last_four CHAR(4),                      -- last 4 digits for display
    legal_name VARCHAR(300) NOT NULL,
    display_name VARCHAR(200),
    date_of_birth DATE,
    date_of_incorporation DATE,
    kyc_status VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'verified', 'enhanced_due_diligence', 'rejected'
    kyc_verified_at TIMESTAMPTZ,
    cip_verified BOOLEAN NOT NULL DEFAULT FALSE,   -- Customer Identification Program
    risk_rating VARCHAR(10) DEFAULT 'standard',    -- 'low', 'standard', 'high', 'prohibited'
    jurisdiction_id UUID REFERENCES jurisdiction(jurisdiction_id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_customer_number ON customer(customer_number);
CREATE INDEX idx_customer_type ON customer(customer_type);
CREATE INDEX idx_customer_risk ON customer(risk_rating);

-- Customer addresses
CREATE TABLE customer_address (
    address_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    address_type VARCHAR(20) NOT NULL,              -- 'primary', 'mailing', 'business', 'prior'
    -- ISO 20022 structured address fields
    street_name VARCHAR(200),
    building_number VARCHAR(20),
    building_name VARCHAR(100),
    floor VARCHAR(20),
    suite VARCHAR(20),
    post_box VARCHAR(20),
    city VARCHAR(100) NOT NULL,
    state_province VARCHAR(100),
    postal_code VARCHAR(20),
    country_code CHAR(2) NOT NULL,                  -- ISO 3166-1
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_address_customer ON customer_address(customer_id);

-- Customer contact information
CREATE TABLE customer_contact (
    contact_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    contact_type VARCHAR(20) NOT NULL,              -- 'phone', 'email', 'mobile'
    contact_value VARCHAR(200) NOT NULL,
    is_primary BOOLEAN NOT NULL DEFAULT FALSE,
    is_verified BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_contact_customer ON customer_contact(customer_id);

-- Customer identification documents (CIP)
CREATE TABLE customer_identification (
    identification_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    id_type VARCHAR(30) NOT NULL,                   -- 'drivers_license', 'passport', 'state_id', 'military_id'
    id_number_encrypted BYTEA,
    issuing_authority VARCHAR(100),
    issue_date DATE,
    expiry_date DATE,
    verification_method VARCHAR(30),                -- 'documentary', 'non_documentary'
    verified_at TIMESTAMPTZ,
    verified_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_identification_customer ON customer_identification(customer_id);

-- Customer-to-customer relationships
CREATE TABLE customer_relationship (
    relationship_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    related_customer_id UUID NOT NULL REFERENCES customer(customer_id),
    relationship_type VARCHAR(30) NOT NULL,          -- 'joint_owner', 'authorized_signer', 'beneficiary', 'guarantor', 'poa'
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_rel_customer ON customer_relationship(customer_id);
CREATE INDEX idx_rel_related ON customer_relationship(related_customer_id);
```

---

## Account Management

```sql
-- Financial accounts (deposits and loans)
CREATE TABLE account (
    account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    product_id UUID NOT NULL REFERENCES product(product_id),
    account_type VARCHAR(20) NOT NULL,               -- 'dda', 'savings', 'money_market', 'cd', 'loan', 'loc'
    account_status VARCHAR(20) NOT NULL DEFAULT 'pending', -- 'pending', 'active', 'dormant', 'frozen', 'closed'
    currency_code CHAR(3) NOT NULL REFERENCES currency(currency_code),
    current_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    available_balance NUMERIC(18,2) NOT NULL DEFAULT 0,
    hold_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    interest_rate NUMERIC(8,5),
    interest_accrued NUMERIC(18,2) NOT NULL DEFAULT 0,
    overdraft_limit NUMERIC(18,2) DEFAULT 0,
    maturity_date DATE,                               -- for CDs and term loans
    opened_date DATE NOT NULL DEFAULT CURRENT_DATE,
    closed_date DATE,
    last_activity_date TIMESTAMPTZ,
    branch_id UUID REFERENCES branch(branch_id),
    officer_id UUID,                                   -- assigned account officer
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_account_number ON account(account_number);
CREATE INDEX idx_account_type ON account(account_type);
CREATE INDEX idx_account_status ON account(account_status);
CREATE INDEX idx_account_product ON account(product_id);

-- Account ownership (junction table: customer <-> account)
CREATE TABLE account_owner (
    account_owner_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    customer_id UUID NOT NULL REFERENCES customer(customer_id),
    ownership_type VARCHAR(20) NOT NULL,              -- 'primary', 'joint', 'beneficiary', 'custodian', 'trustee'
    ownership_percentage NUMERIC(5,2),
    effective_from DATE NOT NULL DEFAULT CURRENT_DATE,
    effective_to DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE(account_id, customer_id, ownership_type)
);
CREATE INDEX idx_ao_account ON account_owner(account_id);
CREATE INDEX idx_ao_customer ON account_owner(customer_id);

-- Branch / location
CREATE TABLE branch (
    branch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    branch_code VARCHAR(10) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    routing_number CHAR(9) NOT NULL,                  -- ABA routing transit number
    address_id UUID REFERENCES customer_address(address_id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Loan-specific details
CREATE TABLE loan_detail (
    loan_detail_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL UNIQUE REFERENCES account(account_id),
    original_principal NUMERIC(18,2) NOT NULL,
    remaining_principal NUMERIC(18,2) NOT NULL,
    interest_rate NUMERIC(8,5) NOT NULL,
    rate_type VARCHAR(10) NOT NULL,                   -- 'fixed', 'variable'
    payment_frequency VARCHAR(20) NOT NULL,           -- 'monthly', 'biweekly', 'weekly'
    payment_amount NUMERIC(18,2),
    next_payment_date DATE,
    origination_date DATE NOT NULL,
    maturity_date DATE NOT NULL,
    collateral_description TEXT,
    loan_purpose VARCHAR(50),
    delinquency_days INTEGER NOT NULL DEFAULT 0,
    charge_off_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_loan_account ON loan_detail(account_id);

-- Loan amortization schedule
CREATE TABLE loan_schedule (
    schedule_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    installment_number INTEGER NOT NULL,
    due_date DATE NOT NULL,
    principal_amount NUMERIC(18,2) NOT NULL,
    interest_amount NUMERIC(18,2) NOT NULL,
    total_amount NUMERIC(18,2) NOT NULL,
    paid_amount NUMERIC(18,2) NOT NULL DEFAULT 0,
    status VARCHAR(20) NOT NULL DEFAULT 'scheduled',  -- 'scheduled', 'paid', 'partial', 'overdue', 'waived'
    paid_date DATE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_schedule_account ON loan_schedule(account_id);
CREATE INDEX idx_schedule_due ON loan_schedule(due_date, status);
```

---

## Transaction Processing

```sql
-- Financial transactions
CREATE TABLE transaction (
    transaction_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    transaction_type VARCHAR(30) NOT NULL,             -- 'deposit', 'withdrawal', 'transfer', 'interest', 'fee', 'ach_credit', 'ach_debit', 'wire_in', 'wire_out', 'adjustment'
    transaction_code VARCHAR(10),                      -- NACHA standard entry class code or internal code
    amount NUMERIC(18,2) NOT NULL,
    direction VARCHAR(6) NOT NULL,                     -- 'debit', 'credit'
    running_balance NUMERIC(18,2) NOT NULL,
    description VARCHAR(500),
    reference_number VARCHAR(50),
    effective_date DATE NOT NULL,
    posted_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',     -- 'pending', 'posted', 'reversed', 'returned'
    channel VARCHAR(20),                               -- 'teller', 'atm', 'online', 'mobile', 'ach', 'wire', 'internal'
    branch_id UUID REFERENCES branch(branch_id),
    teller_id UUID,
    reversal_of UUID REFERENCES transaction(transaction_id),
    currency_code CHAR(3) NOT NULL REFERENCES currency(currency_code),
    memo TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_txn_account ON transaction(account_id);
CREATE INDEX idx_txn_effective ON transaction(effective_date);
CREATE INDEX idx_txn_posted ON transaction(posted_date);
CREATE INDEX idx_txn_type ON transaction(transaction_type);
CREATE INDEX idx_txn_status ON transaction(status);
CREATE INDEX idx_txn_reference ON transaction(reference_number);

-- Transaction holds (pending authorizations, check holds)
CREATE TABLE transaction_hold (
    hold_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_id UUID NOT NULL REFERENCES account(account_id),
    hold_type VARCHAR(20) NOT NULL,                    -- 'check_hold', 'authorization', 'legal', 'administrative'
    amount NUMERIC(18,2) NOT NULL,
    reason VARCHAR(200),
    placed_date TIMESTAMPTZ NOT NULL DEFAULT now(),
    release_date TIMESTAMPTZ,
    status VARCHAR(20) NOT NULL DEFAULT 'active',      -- 'active', 'released', 'expired'
    placed_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_hold_account ON transaction_hold(account_id);
CREATE INDEX idx_hold_status ON transaction_hold(status);
```

---

## General Ledger (Double-Entry)

```sql
-- Chart of accounts
CREATE TABLE gl_account (
    gl_account_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    account_number VARCHAR(20) NOT NULL UNIQUE,
    name VARCHAR(200) NOT NULL,
    account_type VARCHAR(20) NOT NULL,                 -- 'asset', 'liability', 'equity', 'revenue', 'expense'
    account_subtype VARCHAR(30),
    parent_account_id UUID REFERENCES gl_account(gl_account_id),
    normal_balance VARCHAR(6) NOT NULL,                -- 'debit', 'credit'
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_gl_type ON gl_account(account_type);

-- Journal entries (each journal must balance: sum of debits = sum of credits)
CREATE TABLE gl_journal (
    journal_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    journal_number VARCHAR(30) NOT NULL UNIQUE,
    journal_date DATE NOT NULL,
    description VARCHAR(500),
    source VARCHAR(30),                                -- 'transaction', 'interest_accrual', 'fee', 'manual', 'period_close'
    source_reference_id UUID,                          -- FK to transaction, ach_entry, etc.
    status VARCHAR(20) NOT NULL DEFAULT 'posted',      -- 'posted', 'reversed', 'pending'
    posted_by UUID,
    posted_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_journal_date ON gl_journal(journal_date);
CREATE INDEX idx_journal_source ON gl_journal(source, source_reference_id);

-- Journal entry lines (double-entry: must have at least one debit and one credit per journal)
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

-- GL period management
CREATE TABLE gl_period (
    period_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    period_name VARCHAR(30) NOT NULL,                  -- '2026-01', '2026-Q1', '2026'
    period_type VARCHAR(10) NOT NULL,                  -- 'month', 'quarter', 'year'
    start_date DATE NOT NULL,
    end_date DATE NOT NULL,
    status VARCHAR(20) NOT NULL DEFAULT 'open',        -- 'open', 'closing', 'closed'
    closed_by UUID,
    closed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Payments: ACH

```sql
-- ACH file (record type 1 / 9)
CREATE TABLE ach_file (
    ach_file_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    file_id_modifier CHAR(1) NOT NULL DEFAULT 'A',
    direction VARCHAR(10) NOT NULL,                    -- 'outbound', 'inbound'
    immediate_destination VARCHAR(10) NOT NULL,         -- routing number
    immediate_origin VARCHAR(10) NOT NULL,
    file_creation_date DATE NOT NULL,
    file_creation_time TIME,
    record_count INTEGER,
    entry_addenda_count INTEGER,
    total_debit NUMERIC(18,2),
    total_credit NUMERIC(18,2),
    status VARCHAR(20) NOT NULL DEFAULT 'created',     -- 'created', 'submitted', 'acknowledged', 'settled', 'returned'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ACH batch (record type 5 / 8)
CREATE TABLE ach_batch (
    ach_batch_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ach_file_id UUID NOT NULL REFERENCES ach_file(ach_file_id),
    service_class_code CHAR(3) NOT NULL,               -- '200' (mixed), '220' (credits), '225' (debits)
    company_name VARCHAR(16) NOT NULL,
    company_identification VARCHAR(10) NOT NULL,
    standard_entry_class CHAR(3) NOT NULL,             -- 'PPD', 'CCD', 'WEB', 'TEL', 'CTX'
    company_entry_description VARCHAR(10) NOT NULL,    -- 'PAYROLL', 'PURCHASE', etc.
    effective_entry_date DATE NOT NULL,
    originator_status_code CHAR(1) NOT NULL DEFAULT '1',
    batch_number INTEGER NOT NULL,
    entry_count INTEGER,
    total_debit NUMERIC(18,2),
    total_credit NUMERIC(18,2),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ach_batch_file ON ach_batch(ach_file_id);

-- ACH entry detail (record type 6)
CREATE TABLE ach_entry (
    ach_entry_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ach_batch_id UUID NOT NULL REFERENCES ach_batch(ach_batch_id),
    transaction_code CHAR(2) NOT NULL,                 -- NACHA: '22' checking credit, '27' checking debit, '32' savings credit, '37' savings debit
    receiving_dfi VARCHAR(9) NOT NULL,                 -- receiving bank routing number
    dfi_account_number VARCHAR(17) NOT NULL,
    amount NUMERIC(18,2) NOT NULL,
    individual_id VARCHAR(15),
    individual_name VARCHAR(22) NOT NULL,
    trace_number VARCHAR(15) NOT NULL,
    addenda_indicator CHAR(1) DEFAULT '0',
    transaction_id UUID REFERENCES transaction(transaction_id),
    return_reason_code CHAR(3),                        -- R01-R85 if returned
    return_date DATE,
    status VARCHAR(20) NOT NULL DEFAULT 'pending',     -- 'pending', 'settled', 'returned', 'dishonored'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ach_entry_batch ON ach_entry(ach_batch_id);
CREATE INDEX idx_ach_entry_trace ON ach_entry(trace_number);
CREATE INDEX idx_ach_entry_txn ON ach_entry(transaction_id);

-- ACH addenda (record type 7)
CREATE TABLE ach_addenda (
    ach_addenda_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ach_entry_id UUID NOT NULL REFERENCES ach_entry(ach_entry_id),
    addenda_type_code CHAR(2) NOT NULL DEFAULT '05',
    payment_related_info VARCHAR(80),
    addenda_sequence INTEGER NOT NULL DEFAULT 1,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ach_addenda_entry ON ach_addenda(ach_entry_id);
```

---

## Payments: Wire Transfer (ISO 20022)

```sql
-- Wire transfer messages (Fedwire / SWIFT)
CREATE TABLE wire_transfer (
    wire_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    message_type VARCHAR(20) NOT NULL,                 -- 'fedwire', 'swift_mt103', 'swift_pacs008'
    direction VARCHAR(10) NOT NULL,                    -- 'outbound', 'inbound'
    message_id VARCHAR(50) NOT NULL UNIQUE,
    -- ISO 20022 pacs.008 key fields
    instruction_id VARCHAR(35),
    end_to_end_id VARCHAR(35),
    transaction_id UUID REFERENCES transaction(transaction_id),
    amount NUMERIC(18,2) NOT NULL,
    currency_code CHAR(3) NOT NULL REFERENCES currency(currency_code),
    -- Originator (debtor)
    debtor_name VARCHAR(140),
    debtor_account VARCHAR(34),                        -- IBAN or domestic account
    debtor_agent_bic VARCHAR(11),                      -- SWIFT BIC
    debtor_agent_routing VARCHAR(9),
    -- Beneficiary (creditor)
    creditor_name VARCHAR(140),
    creditor_account VARCHAR(34),
    creditor_agent_bic VARCHAR(11),
    creditor_agent_routing VARCHAR(9),
    -- Structured address (ISO 20022 mandate Nov 2026)
    creditor_street VARCHAR(200),
    creditor_building_number VARCHAR(20),
    creditor_city VARCHAR(100),
    creditor_postal_code VARCHAR(20),
    creditor_country CHAR(2),
    -- Processing
    purpose_code VARCHAR(10),
    remittance_info TEXT,
    status VARCHAR(20) NOT NULL DEFAULT 'initiated',   -- 'initiated', 'sent', 'acknowledged', 'settled', 'rejected', 'returned'
    ofac_screened BOOLEAN NOT NULL DEFAULT FALSE,
    ofac_screened_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_wire_message ON wire_transfer(message_id);
CREATE INDEX idx_wire_txn ON wire_transfer(transaction_id);
CREATE INDEX idx_wire_status ON wire_transfer(status);
```

---

## BSA/AML Compliance

```sql
-- AML transaction monitoring alerts
CREATE TABLE aml_alert (
    alert_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    alert_number VARCHAR(20) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customer(customer_id),
    account_id UUID REFERENCES account(account_id),
    alert_type VARCHAR(30) NOT NULL,                   -- 'structuring', 'unusual_activity', 'high_risk_geography', 'rapid_movement', 'pattern_match'
    alert_source VARCHAR(20) NOT NULL,                 -- 'rule_engine', 'ai_model', 'manual'
    severity VARCHAR(10) NOT NULL,                     -- 'low', 'medium', 'high', 'critical'
    ai_confidence_score NUMERIC(5,4),                  -- 0.0000 to 1.0000 for AI-generated alerts
    ai_explanation TEXT,                                -- explainable AI rationale
    description TEXT,
    triggering_transactions UUID[],                    -- array of transaction_ids
    status VARCHAR(20) NOT NULL DEFAULT 'new',         -- 'new', 'under_review', 'escalated', 'filed', 'cleared', 'false_positive'
    assigned_to UUID,
    reviewed_at TIMESTAMPTZ,
    reviewed_by UUID,
    disposition VARCHAR(30),
    disposition_notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_alert_customer ON aml_alert(customer_id);
CREATE INDEX idx_alert_status ON aml_alert(status);
CREATE INDEX idx_alert_severity ON aml_alert(severity);
CREATE INDEX idx_alert_type ON aml_alert(alert_type);

-- Suspicious Activity Reports (FinCEN SAR)
CREATE TABLE suspicious_activity_report (
    sar_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sar_number VARCHAR(30) NOT NULL UNIQUE,
    alert_id UUID REFERENCES aml_alert(alert_id),
    -- Part I: Subject information
    subject_customer_id UUID REFERENCES customer(customer_id),
    subject_name VARCHAR(300),
    subject_tin_last_four CHAR(4),
    -- Part II: Suspicious activity
    activity_start_date DATE NOT NULL,
    activity_end_date DATE,
    total_amount NUMERIC(18,2),
    activity_categories VARCHAR(200)[],                -- FinCEN activity type codes
    -- Part III: Filing institution
    filing_institution_name VARCHAR(200) NOT NULL,
    filing_institution_ein VARCHAR(10) NOT NULL,
    -- Part V: Narrative
    narrative TEXT NOT NULL,
    -- Filing status
    fincen_bsa_id VARCHAR(20),                         -- assigned after filing
    filing_status VARCHAR(20) NOT NULL DEFAULT 'draft', -- 'draft', 'review', 'filed', 'amended'
    filed_at TIMESTAMPTZ,
    filed_by UUID,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_sar_alert ON suspicious_activity_report(alert_id);
CREATE INDEX idx_sar_status ON suspicious_activity_report(filing_status);

-- Currency Transaction Reports (FinCEN CTR)
CREATE TABLE currency_transaction_report (
    ctr_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    ctr_number VARCHAR(30) NOT NULL UNIQUE,
    customer_id UUID REFERENCES customer(customer_id),
    transaction_date DATE NOT NULL,
    total_cash_in NUMERIC(18,2) NOT NULL DEFAULT 0,
    total_cash_out NUMERIC(18,2) NOT NULL DEFAULT 0,
    transaction_ids UUID[],
    conductor_name VARCHAR(300),                        -- person conducting the transaction if different from account holder
    conductor_id_type VARCHAR(30),
    conductor_id_number_encrypted BYTEA,
    fincen_bsa_id VARCHAR(20),
    filing_status VARCHAR(20) NOT NULL DEFAULT 'draft',
    filed_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_ctr_customer ON currency_transaction_report(customer_id);
CREATE INDEX idx_ctr_date ON currency_transaction_report(transaction_date);
```

---

## User & Access Management (RBAC)

```sql
CREATE TABLE app_user (
    user_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) NOT NULL UNIQUE,
    email VARCHAR(200) NOT NULL UNIQUE,
    password_hash VARCHAR(200),
    full_name VARCHAR(200) NOT NULL,
    employee_id VARCHAR(20),
    branch_id UUID REFERENCES branch(branch_id),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at TIMESTAMPTZ,
    mfa_enabled BOOLEAN NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE role (
    role_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name VARCHAR(50) NOT NULL UNIQUE,             -- 'teller', 'loan_officer', 'compliance_officer', 'branch_manager', 'admin'
    description VARCHAR(300),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE permission (
    permission_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permission_code VARCHAR(50) NOT NULL UNIQUE,       -- 'account.read', 'account.create', 'transaction.post', 'sar.file'
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
    branch_id UUID REFERENCES branch(branch_id),        -- optional: role scoped to a branch
    PRIMARY KEY (user_id, role_id)
);
```

---

## Audit & Compliance Logging

```sql
-- Application audit log (FFIEC-aligned)
CREATE TABLE audit_log (
    audit_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type VARCHAR(50) NOT NULL,                    -- 'account.created', 'transaction.posted', 'customer.updated', 'sar.filed'
    entity_type VARCHAR(30) NOT NULL,                   -- 'customer', 'account', 'transaction', etc.
    entity_id UUID NOT NULL,
    action VARCHAR(10) NOT NULL,                        -- 'create', 'read', 'update', 'delete'
    old_values JSONB,
    new_values JSONB,
    user_id UUID REFERENCES app_user(user_id),
    ip_address INET,
    user_agent VARCHAR(500),
    session_id VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_user ON audit_log(user_id);
CREATE INDEX idx_audit_created ON audit_log(created_at);
CREATE INDEX idx_audit_event ON audit_log(event_type);

-- Access log (authentication events)
CREATE TABLE access_log (
    access_log_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES app_user(user_id),
    event_type VARCHAR(20) NOT NULL,                    -- 'login', 'logout', 'login_failed', 'mfa_challenge', 'password_reset'
    ip_address INET,
    user_agent VARCHAR(500),
    success BOOLEAN NOT NULL,
    failure_reason VARCHAR(100),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_access_user ON access_log(user_id);
CREATE INDEX idx_access_created ON access_log(created_at);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Reference Data & Lookups | 3 | jurisdiction, currency, product |
| Customer & Identity | 5 | customer, address, contact, identification, relationship |
| Account Management | 5 | account, account_owner, branch, loan_detail, loan_schedule |
| Transaction Processing | 2 | transaction, transaction_hold |
| General Ledger | 4 | gl_account, gl_journal, gl_journal_entry, gl_period |
| Payments: ACH | 4 | ach_file, ach_batch, ach_entry, ach_addenda |
| Payments: Wire | 1 | wire_transfer |
| BSA/AML Compliance | 4 | aml_alert, suspicious_activity_report, currency_transaction_report |
| User & Access (RBAC) | 5 | app_user, role, permission, role_permission, user_role |
| Audit & Logging | 2 | audit_log, access_log |
| **Total** | **35** | |

---

## Key Design Decisions

1. **Separate tables for every entity** — following 3NF principles, every business concept has its own table. This makes regulatory reporting straightforward (each report maps to a clear set of tables) but increases join complexity.

2. **Double-entry GL with journal/entry separation** — the `gl_journal` and `gl_journal_entry` pattern ensures that every financial movement is balanced at the database level via application-enforced constraints. The CHECK constraint on `gl_journal_entry` guarantees each line is either a debit or credit, never both.

3. **NACHA-faithful ACH structure** — the `ach_file` / `ach_batch` / `ach_entry` / `ach_addenda` hierarchy mirrors the NACHA file format (record types 1/5/6/7/8/9), making file generation and parsing a direct table-to-record mapping.

4. **ISO 20022 structured addresses** — the `customer_address` and `wire_transfer` tables use discrete fields for street, building number, city, postal code, and country rather than free-text address blocks, satisfying the November 2026 SWIFT CBPR+ mandate.

5. **Encrypted PII fields** — tax IDs and identification numbers are stored as `BYTEA` (encrypted at the application layer), with last-four fields for display purposes. This supports PCI DSS v4.0 and FFIEC data protection requirements.

6. **AI confidence scoring on AML alerts** — the `aml_alert` table includes `ai_confidence_score` and `ai_explanation` fields to support the project's AI-native BSA/AML monitoring capability while maintaining explainability for examiners.

7. **RBAC with branch-scoped roles** — the `user_role` junction table includes an optional `branch_id`, allowing role assignments to be scoped to specific branches (e.g., a teller role that only applies at one location).

8. **Audit log with old/new values** — the `audit_log` table captures both `old_values` and `new_values` as JSONB, providing a complete change history for FFIEC examination evidence without requiring triggers on every table.

9. **Running balance on transactions** — each transaction row stores the `running_balance` at the time of posting, enabling statement generation without aggregate queries. This is a denormalization trade-off for read performance.

10. **UUID primary keys throughout** — all tables use UUID v4 primary keys for global uniqueness across distributed deployments and to prevent sequential ID enumeration attacks.
