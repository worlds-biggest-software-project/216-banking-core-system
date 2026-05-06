# Standards & API Reference

> Project: Banking Core System · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 20022 — Financial Services Messaging**
- URL: https://www.iso20022.org/iso-20022
- The next-generation global standard for financial messaging, replacing legacy SWIFT MT formats. Adopted by the US Federal Reserve for Fedwire (mandated by 2025). From November 2026, SWIFT CBPR+ rejects fully unstructured address fields; structured or hybrid postal addresses become mandatory. Core banking systems must generate and consume ISO 20022 XML message sets for wire, ACH, and cross-border payments. 2026 is the inflection point where end-to-end structured data enables automated reconciliation and real-time payment visibility.
- Reference: https://thepaypers.com/payments/expert-views/the-banking-view-iso-20022-why-2026-may-be-the-real-inflection-point-hsbc

**ISO 8583:2023 — Financial Transaction Card Originated Messages**
- URL: https://www.iso.org/standard/79451.html
- The de-facto standard for card payment interchange messaging between acquirers and issuers. Defines message structure, data elements, and code values for purchase, withdrawal, deposit, refund, reversal, balance inquiry, and interbank transfers. Required for any core banking system supporting ATM, point-of-sale, or card processing integrations.
- Reference: https://en.wikipedia.org/wiki/ISO_8583

**ISO 9564-1:2017 — PIN Management and Security**
- URL: https://www.iso.org/standard/68669.html
- Specifies minimum security principles and requirements for cardholder PIN management in card-based retail banking systems including ATMs and POS terminals. Applicable to any core banking system that issues or services debit or ATM cards.

**ISO/IEC 27001 — Information Security Management**
- URL: https://www.iso.org/standard/27001
- The global standard for information security management systems (ISMS). Required as part of SOC 2 Type II certification preparation and FFIEC examination readiness. Core banking platforms handling PII and financial data are expected to operate within an ISO 27001-aligned security framework.

**ISO 9001 / NIST SP 800-53 — Quality and Security Controls**
- URL: https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final
- NIST SP 800-53 Rev 5 provides a comprehensive catalogue of security and privacy controls for federal information systems, widely adopted by banking regulators. Relevant for any core system seeking FedRAMP, OCC, or FDIC operational approval.

---

### W3C & IETF Standards

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation framework for secure API access delegation. Mandatory for all open banking API implementations. Banking-grade deployments extend OAuth 2.0 with FAPI (Financial-grade API) profiles, requiring Authorization Code Flow with PKCE, mTLS client authentication, and cryptographically secured request objects.

**OpenID Connect Core 1.0 (OIDC)**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- Identity layer on top of OAuth 2.0 used for consumer and staff authentication in digital banking. FAPI 1.0 (now required for FDX certification) and FAPI 2.0 (migration underway in 2026) extend OIDC with stricter binding and authentication requirements.

**Financial-grade API (FAPI) 1.0 / 2.0**
- URL: https://openid.net/wg/fapi/
- OpenID Foundation specification adding high-security constraints to OAuth 2.0/OIDC specifically for financial services APIs. FAPI 1.0 became mandatory for FDX API certification in 2026. FAPI 2.0 is the current migration target. Core banking systems exposing open banking APIs must implement FAPI-compliant authorisation servers.

**RFC 8705 — OAuth 2.0 Mutual-TLS Client Authentication**
- URL: https://datatracker.ietf.org/doc/html/rfc8705
- Specifies mutual TLS (mTLS) as a client authentication mechanism for OAuth 2.0. Required by FAPI 1.0 and 2.0 for high-value financial API calls. Core banking API servers must support mTLS alongside private_key_jwt for client credential verification.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- JWT is the standard token format used in OAuth 2.0/OIDC flows for bearer tokens, ID tokens, and signed authorisation requests. Core banking systems must issue, validate, and revoke JWTs as part of their API security layer.

---

### Data Model & API Specifications

**Financial Data Exchange (FDX) API v6.5**
- URL: https://financialdataexchange.org/
- The leading US open banking API specification, recognised by the CFPB in January 2025. Currently at v6.5 (released late 2025). As of early 2026, over 100 million consumer accounts have transitioned to FDX-compliant APIs. FDX v6.x extends into "Open Finance" covering payroll, tax, insurance, and investment data in addition to core banking. Over 600 defined data elements. Core banking systems should expose FDX-compliant consumer data-sharing endpoints to enable fintech integrations and CFPB Section 1033 readiness.
- Reference: https://financialdataexchange.org/

**OpenAPI Specification 3.1**
- URL: https://spec.openapis.org/oas/v3.1.0
- The de-facto standard for documenting REST APIs. All major core banking vendors (Mambu, Temenos, Finxact) provide OpenAPI 3.x specifications for client SDK generation, mock servers, and interactive documentation. A new AI-native core should publish a complete OpenAPI 3.1 spec.

**JSON:API 1.1**
- URL: https://jsonapi.org/
- Specification for structuring JSON API request and response payloads. Widely adopted in banking API ecosystems for consistent envelope formatting, error structure, pagination, and relationship linking.

**NACHA Operating Rules 2026**
- URL: https://www.nacha.org/
- The governing rules for the ACH Network in the United States. The 2026 rule package (effective March and June 2026) introduces mandatory proactive fraud monitoring for all ACH originating and receiving institutions, standardised PAYROLL and PURCHASE company entry descriptors, and expanded return rate requirements. Core banking systems must enforce a 0.5% unauthorised return rate threshold and support risk-based ACH fraud controls.
- Reference: https://www.nacha.org/newrules

**SWIFT MX (ISO 20022 messages)**
- URL: https://www.swift.com/standards/iso-20022
- SWIFT's implementation of ISO 20022 messages for cross-border correspondent banking (CBPR+). Structured address mandate takes effect November 2026. Core banking wire processing modules must generate compliant MX messages for Fedwire and SWIFT correspondent payments.

---

### Security & Authentication Standards

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/
- The OWASP API Security Top 10 defines the most critical API security risks. Directly applicable to core banking REST APIs: broken object-level authorisation, authentication weaknesses, excessive data exposure, and injection. Core banking APIs should be tested against this list as part of security assurance.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- The primary cybersecurity risk management framework used by US financial institutions and referenced by FFIEC examiners. Organised around six functions: Govern, Identify, Protect, Detect, Respond, Recover. Core banking software must support institutions in implementing CSF 2.0 controls, particularly around asset management, access control, anomaly detection, and incident response.

**FFIEC IT Examination Handbook**
- URL: https://ithandbook.ffiec.gov/
- The Federal Financial Institutions Examination Council handbook covers 11 booklets spanning audit, business continuity, information security, outsourced technology, and wholesale payments. Updated in 2026 to shift FDIC IT exams from URSIT to a single overall IT rating with emphasis on governance, cybersecurity, BCP, vendor management, and audit. Core banking platforms marketed to US community banks must help institutions satisfy FFIEC examination requirements.

**BSA/AML — FinCEN NPRM (April 2026)**
- URL: https://www.fincen.gov/resources/statutes-and-regulations/bank-secrecy-act
- FinCEN, OCC, FDIC, and NCUA published a joint Notice of Proposed Rulemaking on April 7, 2026 — the most significant proposed BSA/AML reform in two decades. The proposal moves from a rule-following model to a risk-identification and mitigation model, requiring all AML programs to include countering the financing of terrorism (CFT). Community banks must demonstrate risk-based AML/CFT programs; comment period closed June 9, 2026. Core banking systems must support flexible, risk-based suspicious activity monitoring and SAR/CTR filing workflows.
- Reference: https://compliancehub.wiki/aml-bsa-joint-nprm-april-2026-financial-institution-compliance/

**PCI DSS v4.0**
- URL: https://www.pcisecuritystandards.org/
- Payment Card Industry Data Security Standard. Required for any core banking system processing, storing, or transmitting cardholder data. PCI DSS v4.0 (effective March 2025 for new requirements) introduces stronger authentication, targeted risk analysis, and expanded scope for e-commerce and digital banking environments.

**SOC 2 Type II**
- URL: https://www.aicpa-cima.com/resources/landing/system-and-organization-controls-soc-suite-of-services
- AICPA audit framework covering security, availability, processing integrity, confidentiality, and privacy. Required by most US community banks and credit unions before onboarding a SaaS core banking vendor. An AI-native core banking system offered as SaaS must complete SOC 2 Type II attestation.

---

### MCP Server Specifications

**Model Context Protocol (MCP)**
- URL: https://modelcontextprotocol.io/
- Anthropic's open protocol for connecting AI assistants to external data sources and action servers. Nymbus launched the first production MCP Server purpose-built for core banking in April 2026, providing 19 tools for AI-driven front-office banking actions (customer lookup, account management, money movement, debit card controls) with token-based authentication, RBAC, PII masking, and audit logging. An AI-native core banking system should expose an MCP Server as a first-class integration layer for AI agent orchestration.
- Reference: https://www.prnewswire.com/news-releases/nymbus-launches-industry-leading-secure-mcp-server-for-ai-driven-core-banking-actions-302737795.html

---

## Similar Products — Developer Documentation & APIs

### Jack Henry Banking
- **Description:** Dominant core banking and digital banking platform for US community banks and credit unions. SilverLake (banks), Symitar (credit unions), Banno (digital banking).
- **API Documentation:** https://jackhenry.dev/
- **SDKs/Libraries:** REST APIs; jXchange XML middleware for legacy integrations; Payments API at https://api.payments.jackhenry.com/developer/api-cards/
- **Developer Guide:** https://jackhenry.dev/ (self-service API key registration — unique in the industry)
- **Standards:** REST/JSON; OpenAPI; NACHA ACH; Zelle/RTP via JHA PayCenter
- **Authentication:** OAuth 2.0; developer portal self-registration

### Mambu
- **Description:** Cloud-native composable banking engine serving neobanks, fintechs, and digital-first banks globally. API-first and headless.
- **API Documentation:** https://docs.mambu.com/api/
- **SDKs/Libraries:** Java, .NET, Python, Ruby SDKs; OpenAPI specs downloadable for all endpoints
- **Developer Guide:** https://support.mambu.com/docs/developer-overview
- **Standards:** REST/JSON; OpenAPI 3.x; SEPA; Faster Payments; SWIFT ISO 20022
- **Authentication:** OAuth 2.0; API key; mTLS for enterprise

### Temenos Transact
- **Description:** #1 core banking platform globally by number of banks; retail, corporate, treasury, wealth, and payments on a single cloud-native platform.
- **API Documentation:** https://developer.temenos.com/transact-apis
- **SDKs/Libraries:** OpenAPI 3.0 specs available; Transact Microservices APIs at https://developer.temenos.com/transact-microservice-apis
- **Developer Guide:** https://developer.temenos.com/
- **Standards:** REST/JSON; OpenAPI; ISO 20022; SWIFT; SEPA; FPS
- **Authentication:** OAuth 2.0; FAPI-aligned for open banking exposures

### Finxact (Fiserv)
- **Description:** API-first, cloud-native core banking platform acquired by Fiserv (2022). Microservices architecture; 30M+ accounts live. Named Best SaaS Platform at 2026 FinTech Awards.
- **API Documentation:** https://www.finxact.com/
- **SDKs/Libraries:** REST APIs; available on AWS Marketplace and Azure Marketplace
- **Developer Guide:** https://finxact.com/about/
- **Standards:** REST/JSON; OpenAPI; event-driven (Kafka-compatible); ISO 20022
- **Authentication:** OAuth 2.0; mTLS

### Nymbus
- **Description:** Cloud-native unified core, digital banking, and onboarding platform for US community banks and credit unions ($1B–$10B assets). First core vendor with production MCP Server.
- **API Documentation:** https://www.nymbus.com/solutions/core/
- **SDKs/Libraries:** REST APIs; MCP Server (19 tools for AI agent integration)
- **Developer Guide:** https://www.nymbus.com/
- **Standards:** REST/JSON; NACHA ACH; FAPI-aligned; MCP (Model Context Protocol)
- **Authentication:** Token-based auth; RBAC; PII masking; full audit logging in MCP layer

### Thought Machine Vault Core
- **Description:** Cloud-native core banking with smart contract product definitions. Leader in 2025 Gartner Magic Quadrant for Retail Core Banking Systems.
- **API Documentation:** https://www.thoughtmachine.net/vault-core
- **SDKs/Libraries:** Smart Contract API v4; event streaming APIs; cloud-provider-native deployment
- **Developer Guide:** https://www.thoughtmachine.net/
- **Standards:** REST/JSON; Smart Contract API; event-driven architecture; ISO 20022
- **Authentication:** OAuth 2.0; enterprise SSO; cloud-native IAM integration

### Apache Fineract
- **Description:** Open-source (Apache 2.0) core banking platform used by 400+ institutions in 80+ countries. Java/Spring Boot; MySQL/MariaDB backend. Primary platform for microfinance and emerging-market community banks.
- **API Documentation:** https://fineract.apache.org/docs/current/
- **SDKs/Libraries:** REST/JSON API; interactive docs at https://demo.mifos.io/api-docs/apiLive.htm
- **Developer Guide:** https://github.com/apache/fineract
- **Standards:** REST/JSON; multi-tenant; OpenAPI (partial); Apache 2.0 licence
- **Authentication:** Basic auth; OAuth 2.0 support in development

### FDX Reference Implementation
- **Description:** The Financial Data Exchange consortium provides the US open banking API standard (FDX API v6.5), now covering banking, payroll, tax, insurance, and investment data. CFPB-recognised standard-setting body.
- **API Documentation:** https://financialdataexchange.org/
- **SDKs/Libraries:** Mastercard Developer Hub for FDX APIs: https://developer.mastercard.com/fdx-dev-hub/documentation
- **Developer Guide:** https://www.openbankingtracker.com/standards/fdx
- **Standards:** FDX API v6.5; FAPI 1.0 (certification requirement); REST/JSON; OpenAPI
- **Authentication:** OAuth 2.0 / FAPI 1.0 with FAPI 2.0 migration underway

---

## Notes

**Evolving area — AI and MCP in banking:** The Model Context Protocol is emerging as the integration layer of choice for connecting AI agents to core banking systems. Nymbus's April 2026 MCP Server launch is the first production deployment in core banking. An AI-native core should build its MCP Server as a first-class component rather than a future add-on.

**Regulatory uncertainty — CFPB Section 1033:** The CFPB's open banking rule (implementing Section 1033 of the Dodd-Frank Act) is under a judicial stay as of 2026. The FDX API standard is the de-facto technical specification regardless of enforcement status; community banks should implement FDX-compliant data sharing endpoints proactively.

**BSA/AML reform in progress:** The April 2026 joint NPRM from FinCEN and banking regulators represents the most significant proposed AML reform in two decades. Final rules are expected in late 2026 or 2027. AI-native core systems should be designed to support risk-based, configurable AML programs rather than rigid threshold-based rule sets.

**ISO 20022 structured address deadline:** November 2026 is the SWIFT CBPR+ deadline for structured postal addresses. Community banks relying on third-party core vendors for wire processing should confirm vendor readiness. A new core system should build ISO 20022 structured address support from day one.
