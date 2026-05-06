# Open Banking Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source open banking platform providing PSD2/PSD3-compliant APIs, consent management, and data aggregation for fintechs, banks, and embedded finance providers.

The Open Banking Platform is infrastructure for connecting applications to bank accounts under regulatory frameworks like PSD2, the Berlin Group NextGenPSD2 standard, FDX, and FAPI. It targets fintech startups, incumbent banks building TPP gateways, and software vendors embedding financial data — replacing per-call SaaS pricing with a self-hostable alternative augmented by AI for consent, fraud, and compliance workflows.

---

## Why Open Banking Platform?

- **Per-API-call pricing scales painfully.** Incumbents like Plaid, TrueLayer, and Yapily charge $0.01–$0.10 per API call, with payment initiation fees often mirroring card interchange. High-volume fintechs face escalating infrastructure costs as usage grows.
- **Geographic coverage is fragmented.** Plaid is US-centric, TrueLayer and Yapily are EU/UK-focused, BANKSapi and finAPI are strongest in DACH. No single commercial provider offers seamless global coverage.
- **The only open-source incumbent demands heavy lifting.** WSO2 Open Banking is Apache 2.0 but requires significant internal engineering investment and offers a smaller ecosystem of pre-built connectors than commercial platforms.
- **Consolidation is reducing independence.** Visa acquired Tink; Mastercard acquired Aiia. Buyers increasingly want infrastructure that is not owned by a card network competitor.
- **AI capabilities in incumbents are immature.** Fraud detection and consent anomaly detection remain largely rule-based; regulatory change monitoring is manual or absent.

---

## Key Features

### Account Aggregation & Data Access

- Multi-bank account aggregation (AIS) with a standardised data schema
- Real-time balance and transaction history retrieval
- Account verification API to prove account ownership
- Multi-currency support
- Batch transaction processing

### Consent, Security & Compliance

- PSD2-compliant consent management and consent lifecycle tracking
- Strong Customer Authentication (SCA) orchestration
- OAuth 2.0 / OIDC / FAPI security profiles
- Encrypted data transmission and storage
- Audit trails for regulatory reporting
- Standardised error codes per open banking standards
- Rate limiting and API throttling

### Payments

- Payment Initiation Service (PIS) with instant account-to-account (A2A) routing
- Variable Recurring Payments (VRPs) for recurring transactions of varying amounts
- Webhook notifications for transaction updates and consent lifecycle events

### Enrichment & Insights

- Transaction enrichment with categorisation and merchant classification
- Balance prediction based on historical patterns
- Anomaly detection on payment flows for fraud signals
- Integration Health monitoring with error diagnostics

### Developer & Operator Experience

- REST API with comprehensive error handling
- White-label UX customisation for fintech partners
- Admin dashboard for transaction monitoring and consent audit trails

---

## AI-Native Advantage

AI is layered through the platform rather than bolted on. Intelligent consent lifecycle management automates expiry monitoring, anomaly detection on data-sharing patterns, and proactive re-consent flows. ML models trained on real-time payment streams identify account takeover, synthetic identity fraud, and authorised push payment scams with lower false-positive rates than rule-based systems. AI agents monitor third-party bank API reliability, auto-reroute to fallback endpoints, and reconcile payment status discrepancies. LLMs over aggregated account data generate personalised spend analysis and cash-flow forecasts, while regulatory-monitoring agents continuously parse PSD3/PSR updates to flag required changes to consent flows and SCA configurations.

---

## Tech Stack & Deployment

The platform targets self-hosted and cloud deployment, aligned with the open standards adopted by European and global open banking: Berlin Group NextGenPSD2 (used by approximately three-quarters of European banks), FAPI for OAuth 2.0/OIDC security, OBIE standards for the UK, FDX for North America, and ISO 20022 for payment messaging. Integration is via REST APIs with webhook event delivery, OAuth 2.0/FAPI authorisation flows, and white-label or hosted user-flow options for account linking and consent.

---

## Market Context

The global open banking market surpassed $38 billion in 2025 and is projected to exceed $115 billion by 2030, with API call volumes forecast to grow from 137 billion in 2025 to 720 billion by 2029. Incumbent infrastructure providers charge $0.01–$0.10 per API call, with payment initiation fees typically below 1% per transaction. Primary buyers are fintech startups building financial apps, incumbent banks building TPP gateways, enterprise software vendors embedding financial data, and neobanks seeking AIS/PIS connectivity.

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
