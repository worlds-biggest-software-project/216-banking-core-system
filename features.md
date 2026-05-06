# Banking Core System — Feature & Functionality Survey

> Candidate #216 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Jack Henry SilverLake + Banno | Commercial SaaS / hosted | Proprietary, custom enterprise | https://www.jackhenry.com |
| Fiserv DNA | Commercial SaaS / on-premise | Proprietary, custom enterprise | https://www.fiserv.com |
| Temenos Transact | Commercial SaaS / cloud | Proprietary, custom enterprise | https://www.temenos.com |
| Mambu | Commercial SaaS | Proprietary, consumption-based | https://mambu.com |
| Thought Machine Vault Core | Commercial SaaS | Proprietary, custom enterprise | https://www.thoughtmachine.net |
| Nymbus | Commercial SaaS | Proprietary, custom enterprise | https://www.nymbus.com |
| Finxact (Fiserv) | Commercial SaaS | Proprietary, custom enterprise | https://www.finxact.com |
| Apache Fineract / Mifos | Open source | Apache 2.0 | https://fineract.apache.org |

---

## Feature Analysis by Solution

### Jack Henry SilverLake + Banno

**Core features**
- SilverLake core processing for community banks: deposit accounts, loan servicing, general ledger, teller operations
- Banno Digital Platform: mobile and online banking for consumers and business users
- Integrated payments: ACH, wire, RTP (via JHA PayCenter), Zelle
- Open banking APIs: self-service developer registration, fintech integration network (FIN) with hundreds of certified partners
- Compliance modules: BSA/AML monitoring, FFIEC audit trail, regulatory reporting

**Differentiating features**
- Only core vendor offering fully self-service API key registration without sales engagement
- Banno's "embedded fintech" framework allows fintechs to surface within the bank's digital app natively
- JHA FIN (Fintech Integration Network) gives access to 300+ pre-certified integrations
- Native passkey / FIDO2 authentication in Banno Mobile (2026)
- Digital asset integration via Stablecore partnership (2026)

**UX patterns**
- Bank-staff-facing SilverLake UI is character-based legacy (green screen) with modern web overlay
- Banno consumer UI is modern, responsive; progressive feature rollout via versioned app releases
- Onboarding new institutions takes 12–18 months on average

**Integration points**
- REST APIs via jackhenry.dev developer portal
- jXchange XML-based middleware for legacy integrations
- Payments API for RTP and Zelle
- FIN marketplace for third-party fintech connections

**Known gaps**
- SilverLake core UI is widely criticised as outdated; modernisation in progress but incomplete
- No native AI-driven fraud or AML scoring built into the core
- Complex multi-system architecture (core + digital + payments are separate products)
- Long implementation timelines frustrate community banks needing rapid deployment

**Licence / IP notes**
- Proprietary; no open-source components in core path. Integrators must sign JHA developer agreement.

---

### Fiserv DNA

**Core features**
- Full core banking: deposits, loans, general ledger, member/customer management
- Real-time transaction processing
- Workflow automation and customisable modules
- Third-party integration via open APIs
- Risk management and compliance reporting tools

**Differentiating features**
- DNA's object-oriented data model allows deep customisation of customer records
- Designed for both banks and credit unions on a single platform
- Available as on-premise, hosted, or SaaS deployment

**UX patterns**
- Web-based UI, but described by users as having a steep learning curve
- Wizard-driven account and customer creation flows
- No mobile-first staff interface

**Integration points**
- REST/SOAP APIs; custom integration services via Fiserv professional services
- Marketplace of Fiserv-owned ancillary products (Finxact, Card Services, etc.)

**Known gaps**
- Per-module pricing model draws heavy user complaints ("nickle-and-dime" fee structure)
- Insufficient built-in data visualisation and business intelligence; requires third-party BI tools
- Internal communication gaps between Fiserv divisions create support delays
- Legacy architecture limits real-time AI use cases requiring low-latency data access
- Uptime incidents reported by users; limited SLA transparency

**Licence / IP notes**
- Proprietary. Fiserv acquired DNA with the Open Solutions acquisition; no open-source licensing.

---

### Temenos Transact

**Core features**
- Retail, corporate, treasury, wealth, and payments in a single platform
- Cloud-native and cloud-agnostic; deployable SaaS, cloud, or on-premise
- Account lifecycle management, secured and unsecured lending, collateral management
- Payments processing: SWIFT, SEPA, ACH, ISO 20022 native
- AI-native analytics layer (Temenos Financial Crime Mitigation)

**Differentiating features**
- Widest breadth of banking product coverage globally (retail through treasury)
- API-first architecture: all capabilities exposed as REST/microservice APIs
- Developer portal at developer.temenos.com with Transact Microservices APIs
- Built-in financial crime and compliance modules; real-time transaction screening

**UX patterns**
- Temenos Infinity front-end for digital banking; headless option via APIs
- Long implementation cycles; typically deployed with a large SI partner
- Configurability is a double-edged sword: powerful but complex to govern

**Integration points**
- developer.temenos.com: full API catalogue for Transact, Payments, Lending, Wealth
- OpenAPI 3.0 specifications available
- Event-driven messaging for real-time integration

**Known gaps**
- Weaker penetration in US community bank segment; implementation expertise concentrated in large SIs
- High total cost of ownership; less suitable for institutions under $500M in assets
- Complex upgrade path; customers report difficulty staying current on version releases

**Licence / IP notes**
- Proprietary. Developer programme registration required. No open-source components in core path.

---

### Mambu

**Core features**
- Cloud-native composable banking engine: deposit products, lending products, savings, current accounts
- General ledger service integrated with product engine
- Payments API for domestic and cross-border payments (SEPA, Faster Payments, SWIFT)
- Streaming API for event-driven integrations
- Multi-tenant SaaS architecture; configurable per-institution

**Differentiating features**
- "Composable banking" model: institutions select only the modules they need
- OpenAPI specifications downloadable for client SDK generation
- Strongest developer experience among commercial vendors: REST, JSON, Swagger docs, webhooks
- Product Library with 200+ preconfigured financial products
- Pricing: consumption-based model; attractive for greenfield launches

**UX patterns**
- Headless by design; no native front-end — institutions pair with digital banking vendors
- Admin console (Mambu Hub) for configuration; aimed at product owners, not tellers
- Fast implementation (weeks to months); strongly suited to neobank and BaaS models

**Integration points**
- docs.mambu.com: full REST API v2 documentation with OpenAPI specs
- Webhook / streaming API for event consumers
- SDKs available for Java, .NET, Python, Ruby
- Sandbox environment for developer testing

**Known gaps**
- Not designed for direct branch-banking operations (no teller module)
- Legacy migration tooling is limited; greenfield-first design philosophy
- Compliance modules (AML, BSA) must be integrated from third parties
- Less suited to institutions requiring on-premise or air-gapped deployment

**Licence / IP notes**
- Proprietary SaaS. Developer access requires a Mambu customer or partner agreement.

---

### Thought Machine Vault Core

**Core features**
- Cloud-native core: account management, transaction processing, product configuration
- Smart contract engine: product behaviour defined in developer-written Python-like code
- Universal Product Engine: 200+ preconfigured products in the Product Library
- Financial product simulation before live launch
- Event-driven architecture; real-time continuous processing

**Differentiating features**
- Only vendor where every financial product rule is encoded in auditable smart contracts — institutions own the logic
- Product simulation capability: forecast product performance across any time horizon before launch
- Named Leader in 2025 Gartner Magic Quadrant for Retail Core Banking
- Strong institutional backing (Lloyds Banking Group, JPMorgan as investors and early clients)

**UX patterns**
- Developer-first platform; product teams write smart contracts, not configuration screens
- Headless core; Vault Core pairs with Vault Payments and third-party digital layers
- Steep engineering ramp-up; requires banking engineers comfortable with code-as-configuration

**Integration points**
- thoughtmachine.net/vault-core: API documentation for Vault Core
- Smart Contract API v4 (latest); event streaming for downstream consumers
- Cloud-provider-native deployment (AWS, GCP, Azure)

**Known gaps**
- Limited US community bank client base; sales and implementation focus on Tier 1/2 banks
- High engineering overhead; community banks without dedicated engineering teams struggle to adopt
- Smart contract model requires version control discipline; complex to govern at small institutions

**Licence / IP notes**
- Proprietary. Smart contracts written by customers are customer IP. Vault Core platform is Thought Machine IP.

---

### Nymbus

**Core features**
- Unified cloud-native stack: core processing, digital banking, and onboarding on one platform
- Real-time processing; no middleware layer
- Deposit, loan, and credit card onboarding modules
- Banking-as-a-Service (BaaS) model: institutions can launch a digital brand without a core conversion
- MCP Server (Model Context Protocol): 19 tools for AI-driven front-office banking actions (launched April 2026)

**Differentiating features**
- First core banking vendor to launch a production-grade MCP Server for AI agent integration
- Single platform eliminates the need to reconcile separate core, digital, and onboarding systems
- BaaS launch model allows a credit union to create a separate digital brand on its existing charter
- Focused exclusively on the $1B–$10B asset community bank / credit union segment
- Target: "feature parity plus innovation" — replicates legacy features while adding capabilities not available on legacy platforms

**UX patterns**
- Modern web-based staff console; mobile-first consumer experience
- Modular activation: institutions turn features on/off per their readiness
- Implementation timeline shorter than legacy vendors (months, not years)

**Integration points**
- Open API architecture; REST APIs for core data access
- MCP Server: token-based auth, RBAC, PII masking, audit logging — designed for AI agent orchestration
- nymbus.com/solutions/core: product documentation

**Known gaps**
- Relatively young platform; fewer certified third-party integrations than Jack Henry FIN
- Limited visibility of compliance tooling depth (BSA/AML) in public documentation
- Smaller client base limits peer-benchmarking; limited Gartner / analyst coverage

**Licence / IP notes**
- Proprietary. API access via customer agreement. MCP Server architecture is novel IP.

---

### Finxact (Fiserv)

**Core features**
- API-first, cloud-native core: real-time system of record for accounts and transactions
- Microservices architecture; event-driven processing
- Flexible account model: supports multiple asset classes and non-financial positions
- Open APIs: full data and transaction access programmatically
- Available on AWS Marketplace and Azure Marketplace

**Differentiating features**
- Acquired by Fiserv (2022); positioned as Fiserv's modern core alongside legacy DNA and Signature
- 30+ million accounts live; proven at scale beyond neobank segments
- Schema extension without code branching: add new data objects without upgrade risk
- Named Best SaaS Platform at 2026 FinTech Awards

**UX patterns**
- Headless core; front-end agnostic
- Fintech-first developer experience; REST APIs with full programmatic access
- Fiserv SI and partner network for implementation

**Integration points**
- finxact.com: API documentation and developer guides
- Fiserv marketplace for ancillary products
- AWS and Azure native deployment options

**Known gaps**
- Community bank-facing documentation and case studies limited; sales focus appears weighted toward larger institutions and fintechs
- Compliance and teller modules rely on Fiserv's broader product catalogue
- Roadmap and pricing opaque compared to Mambu

**Licence / IP notes**
- Proprietary. Fiserv customer or partner agreement required for API access.

---

### Apache Fineract / Mifos

**Core features**
- Open-source core banking: client management, loan portfolio management, deposit/savings management
- General ledger with double-entry accounting
- Reporting and analytics
- RESTful API over HTTPS; JSON request/response
- Multi-tenant architecture

**Differentiating features**
- Only fully open-source production-grade core banking platform (Apache 2.0 licence)
- 400+ institutions, 20+ million customers in 80+ countries
- Mifos X provides a reference UI (web and mobile) on top of Fineract
- Interactive API docs at demo.mifos.io/api-docs/apiLive.htm
- Strong in microfinance, credit unions, SACCOs, and emerging-market community banks

**UX patterns**
- Reference UI is functional but basic; most deployments build custom front-ends on the API
- Staff console designed for loan officers and savings managers; not optimised for US community bank teller workflows

**Integration points**
- github.com/apache/fineract: source code and REST API
- fineract.apache.org/docs/current: official documentation
- Spring Boot / Java; MySQL / MariaDB backend

**Known gaps**
- Limited US community bank compliance features (BSA/AML, FFIEC, NACHA ACH) without significant customisation
- No native digital banking consumer front-end; pairs with third-party solutions
- Engineering-intensive to deploy and maintain; requires Java development capability
- Relatively small contributor base; enterprise support via Mifos Initiative or third-party vendors

**Licence / IP notes**
- Apache 2.0. Freely deployable and forkable. No patent claims in the codebase. Commercial support available from Mifos Initiative ecosystem partners.

---

## Cross-Cutting Feature Themes

### Table-Stakes Features
- Deposit account management (DDA, savings, money market, CDs)
- Loan origination and servicing (consumer, mortgage, commercial)
- General ledger and double-entry accounting
- ACH and wire payment processing (NACHA, Fedwire, SWIFT)
- Customer / member information management (CIF/CRM)
- Teller and branch operations support
- Regulatory reporting (call reports, BSA/AML SAR/CTR filing)
- Online and mobile digital banking interface
- Card management (debit, credit, ATM)
- Audit logging and access controls

### Differentiating Features
- Composable / modular architecture (Mambu, Nymbus) vs. monolithic integrated suite (Jack Henry, Fiserv)
- Smart contract product definitions (Thought Machine) vs. configuration-screen-driven setup
- Native MCP / AI agent integration layer (Nymbus) vs. third-party add-on only
- Self-service open API developer registration (Jack Henry)
- Real-time event streaming for downstream AI and analytics consumers (Mambu, Finxact)
- BaaS / digital brand launch mode without full core conversion (Nymbus)
- Financial product simulation before launch (Thought Machine)

### Underserved Areas / Opportunities
- Native AI-driven BSA/AML transaction monitoring built into the core data layer (all vendors rely on third-party integrations, creating data latency gaps)
- Integrated compliance change management — automated FFIEC / FinCEN regulatory update detection and impact mapping
- Real-time overdraft and credit limit recommendations based on cash flow prediction rather than static rules
- AI-native fraud detection operating on the same event stream as the core, eliminating reconciliation delays
- Affordable, low-engineering-overhead core banking for institutions under $500M in assets
- Open-source community bank-grade core with US regulatory compliance modules (Fineract lacks these; no open-source alternative serves this segment)
- Natural-language reporting and analytics for community bank staff without SQL skills
- Automated regulatory exam preparation and evidence packaging

### AI-Augmentation Candidates
- BSA/AML alert generation and case triage (rule-based threshold → AI pattern detection)
- Fraud scoring on card and ACH transactions in real time (batch analytics → event-driven models)
- Loan credit decisioning and pricing (scorecard rules → dynamic ML models)
- Regulatory change impact analysis (manual review → NLP-based change monitoring)
- Customer service resolution (branch/call centre → AI-assisted self-service)
- Overdraft and credit limit management (static thresholds → predictive cash flow models)

---

## Legal & IP Summary

All commercial platforms (Jack Henry, Fiserv, Temenos, Mambu, Thought Machine, Nymbus, Finxact) are proprietary, closed-source products. Their APIs require customer or partner agreements before access is granted. No patent claims were identified in public sources for any specific feature area that would preclude an independent implementation. Apache Fineract is the sole production-grade open-source option and is licensed under Apache 2.0, which is permissive and compatible with commercial use and modification. Smart contracts written by Thought Machine customers remain customer IP. There are no identified copyright or licensing concerns for an AI-native open-source community bank core that independently implements standard banking operations and complies with NACHA, ISO 20022, FDX, and FFIEC requirements.

---

## Recommended Feature Scope

**Must-have (MVP)**
- Deposit account lifecycle (open, transact, close: DDA, savings, CD)
- Loan origination and servicing (consumer instalment, line of credit)
- General ledger with double-entry accounting and period-close
- ACH payment processing (NACHA-compliant origination and receipt)
- Customer information management (CIF) with KYC data model
- REST API with OpenAPI 3.x specification and FDX-aligned data schema
- BSA/AML transaction monitoring with AI-driven alert scoring
- FFIEC-aligned audit trail and access control logging

**Should-have (v1.1)**
- Wire transfer (Fedwire, SWIFT) processing
- ISO 20022 message support for payment rails
- Digital banking API layer (FDX v6.x consumer-permissioned data sharing)
- AI-driven fraud detection on ACH and debit card transactions
- Automated regulatory change monitoring (FFIEC, FinCEN, CFPB)
- MCP server for AI agent integration (front-office banking actions)
- Teller and branch operations UI

**Nice-to-have (backlog)**
- Commercial loan origination (CRE, C&I)
- Treasury management and sweep accounts
- Card issuance and management (debit, credit)
- Financial product simulation engine (Thought Machine-style pre-launch modelling)
- Natural-language analytics and reporting for bank staff
- BaaS / digital brand launch mode for charter holders
