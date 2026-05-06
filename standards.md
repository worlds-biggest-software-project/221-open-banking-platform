# Standards & API Reference

> Project: Open Banking Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### Regulatory Frameworks

**PSD2 — EU Payment Services Directive 2 (EU 2015/2366)**
- Official URL: https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A32015L2366
- The foundational EU regulation mandating that banks open their APIs to licensed Third-Party Providers (TPPs). Defines the legal obligations for Account Information Service Providers (AISPs) and Payment Initiation Service Providers (PISPs), consent requirements, and Strong Customer Authentication (SCA). Still the operative regulatory text in most EU jurisdictions while PSD3 is finalised.

**PSD3 / Payment Services Regulation (PSR) — provisional agreement November 2025**
- Reference: https://www.payment-services-directive-3.com/
- Overview: https://www.nortonrosefulbright.com/en/knowledge/publications/cedd39c6/psd3-and-psr-from-provisional-agreement-to-2026-readiness
- PSD3 and the accompanying PSR reached provisional political agreement in November 2025. Final texts are expected to be published in the Official Journal of the EU in H1 2026, entering into force after a 21-month transition period (expected approximately late 2027). Key changes include more prescriptive API requirements, enhanced SCA obligations, mandatory IBAN-name verification for credit transfers, and expanded fraud liability rules. Any new open banking platform must be designed to accommodate PSD3/PSR compliance.

**EBA Regulatory Technical Standards (RTS) on SCA and Common and Secure Communication (CSC)**
- Official URL: https://www.eba.europa.eu/regulation-and-policy/payment-services-and-electronic-money/regulatory-technical-standards-on-strong-customer-authentication-and-secure-communication-under-psd2
- Specifies the technical requirements for Strong Customer Authentication (SCA) — requiring two independent authentication factors from the categories of knowledge, possession, and inherence — and the technical requirements for secure communications between parties under PSD2. All open banking implementations in the EU must comply with this RTS.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- Official URL: https://gdpr-info.eu/
- All open banking activities involving personal financial data are subject to GDPR. Defines lawful basis for processing, consent requirements, data minimisation, the right to erasure, data breach notification (72 hours), and penalties of up to €20 million or 4% of global turnover. The interaction between PSD2 data-sharing obligations and GDPR consent creates compliance complexity that must be managed in the consent layer.

**CFPB Section 1033 Rule (US)**
- Reference: https://www.consumerfinance.gov/
- The US Consumer Financial Protection Bureau's rule implementing Section 1033 of the Dodd-Frank Act, formally recognising FDX as a standard-setting body. As of 2026 the rule is under a judicial stay (Forbright Bank v. CFPB), but it remains the most likely basis for eventual open banking mandates in the US. Any platform targeting North America should align with the FDX standard.

---

### API Standards & Specifications

**Berlin Group NextGenPSD2 XS2A Framework**
- Official Downloads: https://www.berlin-group.org/nextgenpsd2-downloads
- Open Finance Portal: https://www.berlin-group.org/open-finance
- The dominant European XS2A (Access to Account) specification, adopted by more than 75% of European banks. Defines RESTful APIs, consent models (one-off, recurring, combined), SCA flows, and the full data model for account information and payment initiation. Structured as four artefacts: Operational Rules, Implementation Guidelines, OpenAPI specs, and test tools. As of 2026, incorporated into the Berlin Group openFinance Framework and on a published 2026 workplan for further evolution. Compliance with this specification is mandatory for any EU-facing open banking infrastructure.

**FDX API — Financial Data Exchange (v6.5, current as of late 2025)**
- Official Site: https://financialdataexchange.org/
- API Spec Overview: https://www.openbankingtracker.com/standards/fdx
- Developer Hub (Mastercard): https://developer.mastercard.com/fdx-dev-hub/documentation
- The leading industry-led open finance standard in the US and Canada. FDX v6.5 defines RESTful APIs for sharing deposit, loan, investment, insurance, tax data, and reward programs. Built on OAuth 2.0 with FAPI security profile support. The consortium has over 180 members and has transitioned more than 100 million consumer accounts. The de facto standard for any open banking platform targeting North American markets.

**Open Banking UK Read/Write API Specification (v4.0.1)**
- Official Specs: https://standards.openbanking.org.uk/api-specifications/latest/
- Standards Home: https://standards.openbanking.org.uk/
- GitHub (Read/Write API): https://openbankinguk.github.io/read-write-api-site3/
- The UK-specific open banking standard, now governed by the Joint Regulatory Oversight Committee (JROC) and maintained by Open Banking Limited (OBL). Defines five specification types: Read/Write APIs (accounts, payments, funds confirmation), Open Data APIs, Directory Specifications, Dynamic Client Registration, and MI Reporting. Currently evolving toward Commercial VRP, Premium APIs, and Smart Data interoperability. Required for any UK-facing TPP or ASPSP.

---

### Security & Authentication Standards

**FAPI 2.0 — Financial-grade API Security Profile**
- FAPI Working Group: https://openid.net/wg/fapi/
- FAPI 2.0 Security Profile: https://fapi.openid.net/
- FAPI 1.0 Part 1 (Baseline): https://openid.net/specs/openid-financial-api-part-1-1_0.html
- FAPI 1.0 Part 2 (Advanced): https://openid.net/specs/openid-financial-api-part-2-1_0.html
- Auth0 FAPI 2.0 overview: https://auth0.com/blog/fapi-2-0-the-future-of-api-security-for-high-stakes-customer-interactions/
- An OpenID Foundation security profile layered on OAuth 2.0 that provides financial-grade security for APIs. FAPI 1.0 defines baseline (read) and advanced (read-write) levels; FAPI 2.0 restructures these around threat-model-based protection levels. FAPI 2.0 is being integrated into the next iteration of the Berlin Group framework and is already mandated in several jurisdictions (e.g., Colombia's open finance regulation). Must-have for any high-security open banking platform.

**OAuth 2.0 — RFC 6749**
- IETF RFC: https://datatracker.ietf.org/doc/html/rfc6749
- RFC Editor: https://www.rfc-editor.org/rfc/rfc6749.html
- The foundational authorization framework for open banking. Defines the Authorization Code, Client Credentials, and other grant flows. Every major open banking standard (Berlin Group, FAPI, FDX, UK OB) mandates OAuth 2.0 as the base authorization layer.

**PKCE — Proof Key for Code Exchange (RFC 7636)**
- IETF RFC: https://datatracker.ietf.org/doc/html/rfc7636
- Best Security Practice (RFC 9700): https://datatracker.ietf.org/doc/rfc9700/
- Extends the OAuth 2.0 Authorization Code flow to prevent authorization code interception attacks. Mandatory in FAPI 2.0 and required by best-practice OAuth security guidance (RFC 9700).

**OpenID Connect Core 1.0**
- Official Spec: https://openid.net/specs/openid-connect-core-1_0.html
- Specs List: https://openid.net/developers/specs/
- An identity layer built on OAuth 2.0 that enables clients to verify end-user identity via an Authorization Server and retrieve basic profile claims. Underpins all FAPI-based consent and identity flows in open banking.

**FAPI CIBA — Client Initiated Backchannel Authentication Profile**
- Spec: https://openid.net/specs/openid-financial-api-ciba.html
- Overview: https://www.pingidentity.com/en/resources/blog/post/simple-open-banking-payments-decoupled-authentication.html
- A FAPI profile of OpenID Connect's CIBA specification supporting decoupled authentication flows — where authentication is initiated on one device (e.g., browser) and completed on another (e.g., mobile push notification). Increasingly important for frictionless SCA in payment initiation flows.

**OpenID Federation 1.0**
- Spec: https://openid.net/specs/openid-federation-1_0.html
- Defines how entities in multilateral federations (such as the open banking trust framework) can establish trust via Trust Anchors. Relevant for building trust registries and directory services in open banking ecosystems.

**JSON Web Token — RFC 7519 / RFC 8725**
- JWT RFC: https://datatracker.ietf.org/doc/html/rfc7519
- Best Practices (RFC 8725): https://www.rfc-editor.org/rfc/rfc8725.html
- JWT Decoder/Reference: https://www.jwt.io/
- The compact, URL-safe token format used in OAuth 2.0 and OIDC flows. JWTs serve as access tokens, ID tokens, and signed request objects (JAR) in financial-grade API implementations. RFC 8725 provides mandatory best practices for secure JWT implementation.

---

### Data & Messaging Standards

**ISO 20022 — Financial Message Standard**
- ISO 20022 Portal: https://www.iso20022.org/iso-20022
- SWIFT ISO 20022: https://www.swift.com/standards/iso-20022/iso-20022-standards
- Standard Chartered API Guide: https://www.sc.com/en/uploads/sites/66/content/docs/ISO-20022-CBPR-API-Guide.pdf
- The global standard for financial message exchange, increasingly required alongside open banking payment initiation flows. Defines structured data formats for credit transfers (pacs.008), payment status (pacs.002), and remittance information. From November 2026, SWIFT CBPR+ frameworks will reject payment messages that do not use structured address formats. Any platform with cross-border or SWIFT-connected payment initiation must align with ISO 20022 CBPR+.

**OpenAPI Specification (OAS) 3.x**
- Official Site: https://spec.openapis.org/oas/latest.html
- All major open banking platforms (Plaid, TrueLayer, Yapily, Berlin Group) publish their API contracts as OpenAPI (formerly Swagger) specifications, enabling code generation, automated testing, and documentation. Adopting OAS 3.x as the API contract format is a de facto requirement.

**JSON Schema**
- Official Site: https://json-schema.org/
- Used for request/response validation in REST APIs. The Berlin Group's NextGenPSD2 and FDX API v6.5 use JSON Schema for data model definitions.

---

### MCP Server Specifications

Model Context Protocol (MCP) is not currently a formalised standard in open banking infrastructure. However, the AI-native features described for this platform (consent lifecycle management, regulatory change monitoring, fraud detection agents) are suitable candidates for MCP server implementation, enabling LLM-driven agents to interact with open banking data via standardised tool interfaces. The MCP specification is available at: https://spec.modelcontextprotocol.io/

---

## Similar Products — Developer Documentation & APIs

### Plaid
- **Description:** The leading open banking data aggregation platform in the US, connecting to 12,000+ institutions. Offers Auth, Balance, Transactions, Identity, Investments, Liabilities, and Income APIs; FDX-aligned Core Exchange product for financial institution connectivity.
- **API Documentation:** https://plaid.com/docs/api/
- **Developer Docs Home:** https://plaid.com/docs/
- **SDKs/Libraries:** Node.js, Python, Ruby, Java, Go — https://plaid.com/docs/api/#client-libraries
- **OpenAPI Spec:** https://github.com/plaid/plaid-openapi
- **Developer Guide:** https://plaid.com/docs/ (includes sandbox, development, and production environments)
- **Standards:** REST/JSON; FDX-aligned; OpenAPI 3.0
- **Authentication:** OAuth 2.0 with access_token exchange via public_token; API key for server-side calls

### TrueLayer
- **Description:** EU/UK open banking platform supporting PSD2-compliant account aggregation (AIS) and payment initiation (PIS) including Variable Recurring Payments (VRP), with connectivity to 2,000+ European banks.
- **API Documentation:** https://docs.truelayer.com/
- **API Reference:** https://docs.truelayer.com/reference/welcome-api-reference
- **Payments API Basics:** https://docs.truelayer.com/docs/payments-api-basics
- **Data API Basics:** https://docs.truelayer.com/docs/data-api-basics
- **SDKs/Libraries:** iOS SDK (https://docs.truelayer.com/docs/ios-sdk), Android SDK, React Native SDK — https://github.com/TrueLayer/TrueLayer-iOS-SDK
- **Standards:** REST/JSON; FAPI; PSD2/PSD3-aligned; OAuth 2.0 + PKCE
- **Authentication:** OAuth 2.0 Authorization Code flow; client_id / client_secret with redirect URI registration

### Yapily
- **Description:** Open banking API infrastructure platform connecting to 2,000+ banks in 20+ countries. Provides AIS, PIS, A2A instant payments, VRP, and Data Plus transaction enrichment.
- **API Documentation:** https://docs.yapily.com/
- **API Reference:** https://docs.yapily.com/api-reference/introduction
- **Getting Started:** https://docs.yapily.com/pages/getting-started/overview/
- **SDKs/Libraries:** Python, JavaScript (Node.js), Java — https://github.com/yapily
- **OpenAPI Spec:** Available from Yapily; used for generating client libraries
- **Standards:** REST/JSON; FAPI; PSD2-aligned; OpenAPI 3.0
- **Authentication:** HTTP Basic Authentication (application_id + application_secret) for server-side; OAuth 2.0 for user consent flows

### WSO2 Open Banking Accelerator
- **Description:** Open-source accelerator framework for building PSD2-compliant open banking environments. Runs on WSO2 Identity Server and API Manager; supports Berlin Group NextGenPSD2 and UK OB standards; self-hosted deployment.
- **API Documentation:** https://ob.docs.wso2.com/en/latest/develop/developer-guide/
- **Getting Started:** https://ob.docs.wso2.com/en/latest/get-started/open-banking/
- **GitHub:** https://github.com/wso2/docs-open-banking
- **Docker Deployment:** https://github.com/wso2/docker-open-banking
- **Standards:** Berlin Group NextGenPSD2; UK OB; FAPI; OIDC; Apache 2.0 licence
- **Authentication:** OIDC / FAPI-aligned; configurable identity provider plugins

### GoCardless Bank Account Data (formerly Nordigen)
- **Description:** Free-tier open banking API aggregator connecting to 2,300+ banks across 31 European countries. Acquired by GoCardless in 2022; rebranded as Bank Account Data in 2023. Offers AIS including account, balance, and transaction retrieval.
- **API Documentation:** https://bankaccountdata.gocardless.com/api/docs
- **OpenAPI Spec:** https://bankaccountdata.gocardless.com/api/swagger.json
- **GoCardless Developer Portal:** https://developer.gocardless.com/api-reference/
- **SDKs/Libraries:** Python (https://github.com/nordigen/nordigen-python), PHP (https://github.com/nordigen/nordigen-php)
- **Standards:** REST/JSON; PSD2-aligned; OpenAPI; OAuth 2.0
- **Authentication:** SECRET_ID / SECRET_KEY; JWT-based access tokens

### Token.io
- **Description:** A2A payment infrastructure platform built on open banking rails, connecting to 4,000+ banks. Offers PIS including single payments, future-dated payments, VRP mandates, settlement accounts, and refunds. Also supports AIS.
- **API Reference:** https://reference.token.io/
- **Developer Documentation:** https://developer.token.io/
- **Payment Initiation Flow:** https://developer.token.io/bank_integration/content/h-bank_integration/pis_workflow.htm
- **Developers Portal:** https://token.io/developers
- **SDKs/Libraries:** Multiple language SDKs available via developer.token.io
- **Standards:** PSD2-aligned; FAPI; VRP
- **Authentication:** OAuth 2.0 consent flows; TPP SDK-based credential management

### Salt Edge
- **Description:** Global open banking and open finance platform connecting to 5,000+ banks in 70 countries. Provides AIS (account aggregation), PIS, transaction categorisation, and merchant identification APIs. Used by banks, fintechs, and lenders.
- **API Documentation (v6):** https://docs.saltedge.com/v6/
- **Account Information API:** https://docs.saltedge.com/account_information/v5/
- **GitHub:** https://github.com/saltedge
- **Standards:** REST/JSON; PSD2-aligned; OpenAPI
- **Authentication:** OAuth 2.0; client secret; encrypted channel communications

### Moneyhub Enterprise
- **Description:** UK-based open finance platform offering AIS, PIS, pension data aggregation, and personalised financial insights. All APIs built on OpenID Connect and exposed as RESTful/JSON services with Swagger/OpenAPI documentation.
- **API Reference:** https://www.moneyhub.com/apis/developer-support
- **Developer Docs:** https://docs.moneyhubenterprise.com/docs/introduction
- **Standards:** REST/JSON; OpenID Connect; OpenAPI (Swagger)
- **Authentication:** OpenID Connect (OIDC); OAuth 2.0 Authorization Code flow

---

## Notes

**Emerging and evolving areas:**

- **PSD3/PSR transition (2026–2027):** The most significant near-term standards change. Platforms built today must architect consent flows, SCA modules, and API interfaces to accommodate PSD3's more prescriptive requirements without a full rebuild. The 21-month transition period makes 2027 the effective compliance deadline.

- **ISO 20022 structured address deadline (November 2026):** For any platform routing cross-border payments via SWIFT, non-compliant unstructured address fields will be rejected from November 2026. Relevant for PIS features that connect to SWIFT-based clearing.

- **FAPI 2.0 adoption:** FAPI 2.0 is gaining regulatory traction globally (Colombia mandated it in 2024; expected to be folded into the next Berlin Group framework revision). Designing to FAPI 2.0 rather than FAPI 1.0 is the forward-compatible choice.

- **Open Finance scope expansion:** PSD3 and equivalent frameworks in Australia, Brazil, and the UK are extending open data mandates beyond bank accounts to include investments, pensions, insurance, and mortgage data. Any platform architecture should anticipate multi-asset-class data models beyond current payment accounts.

- **MCP for AI agents:** No open banking-specific MCP server standard exists yet. This represents a greenfield opportunity for the platform to define canonical MCP tool schemas for consent queries, transaction retrieval, and payment initiation — enabling AI agents to interact with open banking data via standardised interfaces.
