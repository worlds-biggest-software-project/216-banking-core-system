# Banking Core System (Community Bank)

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source core banking platform for credit unions and community banks underserved by the legacy Big Three.

The Banking Core System is a candidate project to build a modern, API-first core banking platform purpose-built for US community banks and credit unions in the USD 500 million to USD 10 billion asset range. It targets institutions struggling with legacy green-screen cores, per-module fee structures, and multi-year implementation cycles, while embedding AI capabilities directly into the transaction data layer rather than bolting them on as third-party add-ons.

---

## Why a New Community Bank Core?

- The Big Three (Jack Henry, Fiserv, FIS) collectively serve over 70% of US banks, yet user feedback consistently cites outdated UIs, "nickel-and-dime" per-module pricing, and 12–18 month onboarding timelines.
- Cloud-native challengers (Mambu, Thought Machine, Finxact) are designed greenfield-first or for Tier 1/2 banks, leaving institutions under USD 500 million in assets without affordable, low-engineering-overhead options.
- Apache Fineract is the only production-grade open-source core, but lacks US regulatory modules (BSA/AML, FFIEC, NACHA ACH) without significant customisation.
- All commercial vendors rely on third-party integrations for AML and fraud monitoring, creating data latency gaps that an AI-native core operating on the same event stream can eliminate.
- Approximately 170 credit union mergers are projected in 2025, the highest rate since 2016, driving demand for scalable platforms that absorb acquired institutions quickly.

---

## Key Features

### Core Banking Operations

- Deposit account lifecycle for DDA, savings, money market, and CDs
- Loan origination and servicing for consumer instalment and lines of credit
- General ledger with double-entry accounting and period-close
- Customer information management (CIF) with KYC data model
- Teller and branch operations UI

### Payments & Messaging

- ACH origination and receipt aligned with NACHA
- Wire transfer processing for Fedwire and SWIFT
- ISO 20022 native message support for payment rails
- Real-time event streaming for downstream consumers

### Compliance & Risk

- BSA/AML transaction monitoring with AI-driven alert scoring
- FFIEC-aligned audit trail and access control logging
- Automated regulatory change monitoring across FFIEC, FinCEN, and CFPB updates
- AI-driven fraud detection on ACH and debit card transactions

### Open APIs & Integration

- REST API with OpenAPI 3.x specifications
- FDX-aligned data schema for consumer-permissioned data sharing
- Digital banking API layer targeting FDX v6.x
- MCP server for AI agent integration on front-office banking actions

---

## AI-Native Advantage

Where incumbents bolt AI onto static rule-based engines via third-party integrations, this project embeds AI directly into the core's event stream. That enables explainable BSA/AML alerts that reduce false-positive burden without compromising detection, predictive overdraft and credit limit recommendations driven by real-time cash flow analysis, NLP-based monitoring of FFIEC, CFPB, and state regulatory updates with impact mapping to core configuration, and a conversational banking assistant for member self-service. Fraud detection runs on the same transaction graph as the ledger, eliminating the reconciliation delays inherent in batch-analytics architectures.

---

## Tech Stack & Deployment

The platform is designed cloud-native and API-first, with REST APIs documented via OpenAPI 3.x and an event-driven architecture suitable for streaming integrations. Data and messaging schemas align with FDX (open banking), NACHA (ACH), Fedwire/SWIFT (wires), and ISO 20022. Deployment targets include SaaS, customer-managed cloud, and integration with the broader fintech ecosystem through a developer portal and sandbox. An MCP server is included for AI agent orchestration with token-based auth, RBAC, PII masking, and audit logging.

---

## Market Context

The US core banking software market is projected to grow from USD 6.09 billion in 2025 to USD 16.81 billion by 2032 at a 15.6% CAGR (Fortune Business Insights, 2025). Core banking contracts for community institutions typically range from USD 500,000 to several million per year, structured per-account, per-transaction, or flat-fee. Primary buyers are CEOs, COOs, CIOs, and IT directors at community banks and credit unions managing core replacements, alongside boards overseeing digital transformation.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
