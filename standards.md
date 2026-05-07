# Standards & API Reference

> Project: Shelter & Housing Management · Generated: 2026-05-07

---

## Industry Standards & Specifications

### HUD HMIS Data Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| FY 2026 HMIS Data Standards Manual | https://www.hudexchange.info/programs/hmis/hmis-data-standards/ | The primary federal regulatory framework governing all HMIS and comparable database software in the US. Defines Universal Data Elements (UDE), Programme Specific Data Elements (PSDE), and Project Descriptor Data Elements (PDDE) that every client record must collect. Updated biennially; FY 2026 edition effective 01 Oct 2025. |
| FY 2024 HMIS Data Dictionary | https://files.hudexchange.info/resources/documents/HMIS-Data-Dictionary-2024.pdf | Machine-readable reference for every data element in the HMIS standard, including field names, types, valid values, and collection requirements. Primary input for schema design and data validation. |
| HMIS CSV Format Specifications (FY 2024) | https://cthmis.com/wp-content/uploads/2025/03/HMIS-CSV-Format-Specifications-2024.pdf | Defines the 24-file ZIP bundle used for interoperability between HMIS installations (e.g., bulk imports, CoC-level data aggregation). Includes two new files introduced in FY 2024: `HMISParticipation.csv` and `CEParticipation.csv`. Vendors must produce compliant CSV/XML exports. |
| HMIS Data and Technical Standards — HUD Exchange | https://www.hudexchange.info/programs/hmis/hmis-data-and-technical-standards/ | Central HUD resource hub linking all standards documents, vendor checklists, reporting specifications (APR, ESG CAPER, LSA, HIC, PIT), and programming guidance. |
| HMIS Data Exchange Resources (Vendor Checklist) | https://hudhdx.info/vendorresources.aspx | HUD's official vendor compliance checklist and data exchange resources. Compliance with this checklist is required for CoC HMIS designation. |
| HUD HMIS Privacy & Security Standards (Oct 2024) | https://files.hudexchange.info/resources/documents/Privacy-and-Security-Toolkit-HMIS-Data-Uses-and-Disclosures.pdf | October 2024 updated guidance on permissible uses and disclosures of HMIS client data, including consent requirements, data sharing agreements between CoC agencies, and conditions for use by federal partners. |
| HUD HMIS Regulations and Notices | https://www.hudexchange.info/programs/hmis/hmis-regulations-and-notices/ | Collection of federal register notices and regulatory requirements governing HMIS operation, including privacy plans, security plans, and governance charter requirements. |

---

### HL7 FHIR & Social Determinants of Health (SDOH)

| Standard | URL | Relevance |
|----------|-----|-----------|
| HL7 FHIR US Core — SDOH Guidance | https://hl7.org/fhir/us/core/sdoh.html | Maps social determinants including homelessness to standard FHIR resources. The US Core Condition Profile represents SDOH conditions (e.g., homelessness); US Core ServiceRequest and Procedure Profiles represent referrals and interventions. Relevant if the platform integrates with healthcare EHR systems. |
| Gravity Project — FHIR SDOH Clinical Care IG | https://build.fhir.org/ig/HL7/fhir-sdoh-clinicalcare/ | HL7 Implementation Guide (STU) that standardises SDOH data elements for food insecurity, housing instability, and homelessness. Defines FHIR value sets, assessment instrument support, and task-based referral workflows. Critical for interoperability with healthcare partners (hospitals, FQHCs) doing bi-directional referrals with shelter systems. |
| FHIR Human Services Directory IG | https://build.fhir.org/ig/HL7/FHIR-IG-Human-Services-Directory/ | HL7 Implementation Guide enabling a FHIR-based directory of human services (shelters, food banks, etc.) aligned with Open Referral HSDS. Enables 211 systems and EHRs to query shelter availability and service eligibility. |

---

### W3C & IETF Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| RFC 7231 — HTTP/1.1 Semantics | https://tools.ietf.org/html/rfc7231 | Baseline HTTP protocol governing all REST API interactions. Defines method semantics (GET, POST, PUT, DELETE), status codes, and content negotiation. |
| RFC 6749 — OAuth 2.0 Authorization Framework | https://tools.ietf.org/html/rfc6749 | Standard authorization framework for API access control. Required for inter-agency data sharing, third-party integrations, and mobile client authentication. The Authorization Code with PKCE flow is recommended for mobile/browser clients. |
| RFC 7519 — JSON Web Tokens (JWT) | https://tools.ietf.org/html/rfc7519 | Token format used for stateless API authentication and session management, typically combined with OAuth 2.0 for short-lived, scoped access tokens. |
| OpenID Connect 1.0 | https://openid.net/specs/openid-connect-core-1_0.html | Identity layer on top of OAuth 2.0. Enables single sign-on across CoC agencies and integration with government identity providers. |
| RFC 8288 — Web Linking | https://tools.ietf.org/html/rfc8288 | Defines `Link` header semantics used in RESTful hypermedia APIs and pagination. |

---

### Data Model & API Specifications

| Standard | URL | Relevance |
|----------|-----|-----------|
| OpenAPI Specification 3.1 | https://spec.openapis.org/oas/v3.1.0.html | Industry-standard format for describing RESTful APIs in machine-readable YAML or JSON. Version 3.1 adds full JSON Schema 2020-12 compatibility. Every API surface of the platform should be described in an OpenAPI document to enable auto-generated SDKs, documentation, and testing tooling. |
| JSON Schema 2020-12 | https://json-schema.org/specification.html | Schema language for validating JSON request and response payloads. Used to enforce HMIS data element constraints (required fields, valid values, data types) at the API layer before persistence. |
| Open Referral Human Services Data Specification (HSDS) 3.0 | http://docs.openreferral.org/en/3.0/hsds/about.html | Open standard data model for describing organisations, services, locations, and accessibility details in human services directories. Widely adopted by 211 systems. A shelter/housing platform should expose an HSDS-compatible endpoint so its service inventory can be queried by 211 directories and other community resource tools. |
| Open Referral Human Services Data API (HSDA) | https://openreferral.github.io/api-specification/ | OpenAPI-described REST API for querying HSDS-formatted service directories. Endorsed by AIRS as the industry standard for resource directory data exchange. |
| AIRS/Inform USA Taxonomy of Human Services | https://www.informusa.org/standards | Hierarchical controlled vocabulary of 9,000+ human service terms used to classify shelter and housing services consistently across 211 systems and HMIS resource directories. Services should be tagged with AIRS taxonomy codes for interoperability with 211 referral systems. |

---

### Security & Compliance Standards

| Standard | URL | Relevance |
|----------|-----|-----------|
| HIPAA Security Rule (45 CFR Part 164) | https://www.hhs.gov/hipaa/for-professionals/security/index.html | US federal law requiring administrative, physical, and technical safeguards for electronic protected health information (ePHI). Shelter platforms that integrate with healthcare providers or store health-related client data must maintain a Business Associate Agreement (BAA) and implement HIPAA-compliant controls including encryption, audit logging, and access controls. The 2025 proposed NPRM would make encryption of ePHI at rest and in transit explicitly mandatory, along with MFA. |
| NIST SP 800-53 Rev. 5 — Security and Privacy Controls | https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final | NIST's comprehensive 1,196-control catalogue organized across 20 control families. Required baseline for FedRAMP-authorized cloud services. Maps directly to HIPAA Security Rule requirements. Relevant for platforms targeting government CoC contracts or direct federal agency partnerships. |
| FedRAMP (Federal Risk and Authorization Management Program) | https://www.fedramp.gov/ | US government cloud security authorization program. Mandatory for any cloud software that stores or processes federal data. Achieving FedRAMP Moderate authorization enables direct contracts with HUD and federal CoC leads. Process takes 6–18 months and requires a third-party assessment by a 3PAO against NIST 800-53 Moderate baseline. |
| GDPR — Art. 5 & Art. 9 (Processing of Special Categories) | https://gdpr-info.eu/ | EU data protection regulation. Art. 9 classifies homelessness-related social care data as a "special category" requiring explicit consent or a legal basis under national law. Relevant for any deployment of the platform in EU member states or for EU-based homeless populations. Requires a lawful basis, privacy notice, data minimisation, and right-to-erasure capabilities. |
| OWASP Application Security Top 10 | https://owasp.org/www-project-top-ten/ | The de-facto baseline checklist for web application security. Given the sensitivity of shelter client data, the platform must address all OWASP Top 10 risks including broken access control, cryptographic failures, injection, and insecure direct object references (IDOR). |
| TLS 1.2 / 1.3 (RFC 5246 / RFC 8446) | https://tools.ietf.org/html/rfc8446 | Mandatory transport encryption for all API endpoints and web interfaces handling client PII. TLS 1.3 preferred; TLS 1.2 minimum. HSTS required to prevent plaintext HTTP downgrade attacks. |

---

## Similar Products — Developer Documentation & APIs

### Eccovia ClientTrack (formerly Bowman Systems)

- **Description:** Commercial HMIS and case management platform serving CoCs, shelters, and transitional housing; highly rated for usability and HMIS compliance.
- **API Documentation:** https://apidoc.eccovia.com/
- **Developer Support:** https://eccovia.com/developer-support/
- **Integration Guide:** https://help.eccovia.com/en_US/integrations/guide-for-clienttrack-integration-tools
- **HMIS Data Dictionary (ClientTrack):** https://help.eccovia.com/clienttracks-hmis-data-dictionary-
- **Standards:** REST/JSON; CRQL (ClientTrack REST Query Language) for SQL-style querying; CTObject endpoints for CRUD on HMIS entities; HMIS CSV import/export
- **Authentication:** API key / token-based; OAuth 2.0 for third-party apps

---

### Bitfocus Clarity Human Services

- **Description:** Leading cloud HMIS and Coordinated Entry System platform, used by major US CoCs. Native CE workflows, real-time bed management, and geospatial analytics.
- **API Overview (Integrations & APIs):** https://www.bitfocus.com/integrations-apis-products
- **Operational API (Early Access):** https://help.bitfocus.com/operational-api-early-access
- **XML API Import Tool:** https://help.bitfocus.com/importing-data-using-the-xml-api-import-tool
- **Help Centre:** https://help.bitfocus.com/
- **Standards:** REST/JSON (Operational API); XML-based Data Import Tool aligned to HMIS Data Standards; HMIS CSV/XML export; versioned schema per HMIS release cycle
- **Authentication:** OAuth 2.0 for Operational API; API key for XML import; instance-scoped schemas at `https://[instance].clarityhs.com/data-import/schema`

---

### Green River Open Path HMIS (open-source)

- **Description:** Open-source HMIS and Coordinated Entry System hosted on GitHub by Green River. Used by City of Boston and other CoCs. Ingests standard HMIS CSV files, deduplicates clients across installations, and supplies data to the Boston CAS housing placement system.
- **GitHub (HMIS Warehouse):** https://github.com/greenriver/hmis-warehouse
- **Green River GitHub Org:** https://github.com/greenriver
- **Product Page:** https://greenriver.com/openpath/hmis/
- **Standards:** HMIS CSV standard import/export; REST API endpoints (Eccovia ClientTrack and ETO API integrations documented in codebase); Ruby on Rails; PostgreSQL
- **Authentication:** Role-based access control; open-source codebase allows custom auth integration
- **Licence:** Open-source; free for government/nonprofit use; community contributions welcome

---

### OpenHMIS API Server (open-source community project)

- **Description:** Vendor-neutral open RESTful API for HMIS data exchange, designed to allow any HMIS application to interoperate regardless of underlying storage. Based on HUD 2014 Data Standards; beta-testing of v3 covering full HUD 2014 standard.
- **GitHub:** https://github.com/hmis-tools/hmis-api-server
- **API Documentation:** https://github.com/hmis-tools/hmis-api-server/blob/master/docs/API.md
- **Standards:** REST/JSON (`Content-Type: application/json`, `Accept: application/json`); HUD HMIS data elements; OpenAPI-compatible design
- **Authentication:** Token-based
- **Notes:** Community-maintained; useful as a reference design for an open API layer, though it targets the 2014 data standards. An updated implementation aligned to FY 2026 standards would be a significant contribution.

---

### CaseWorthy

- **Description:** Configurable case management and HMIS platform for human services organisations. Supports custom forms/workflows, coordinated entry, closed-loop referrals, and HUD compliance.
- **Form API Overview:** https://caseworthy.com/articles/unlock-seamless-integrations-and-unmatched-flexibility-with-the-caseworthy-form-api/
- **Product/Integration Page:** https://caseworthy.com/products/
- **Standards:** REST/JSON Form API; HL7 and FHIR for health data interoperability; secure SFTP for state reporting; HUD CSV export (LSA, CAPER, HUD CSV); HIPAA-compliant
- **Authentication:** API key / token; role-based access control at field level
- **Notes:** Public API documentation is limited; detailed endpoint specs require vendor engagement. Form API allows push/pull of custom form data with external systems.

---

### CharityTracker (Simon Solutions)

- **Description:** Multi-city case management and shelter network platform covering 2,000+ cities and 3.5 million cases. Features barcode check-in, duplicate detection, and outcomes dashboards.
- **API Beta Program:** https://help.simonsolutions.com/en/articles/6066264-api-beta-program
- **API Documentation (Beta):** http://apidocs.simonsolutions.com/ *(requires API token from network admin)*
- **Standards:** REST/JSON; API token-based; designed for data warehouse integrations, mobile app intake, and cross-database deduplication
- **Authentication:** API token issued per network administrator
- **Notes:** API is in beta; token access requires network admin request. Useful reference for community-scale network integration patterns.

---

### WellSky Community Services (HMIS)

- **Description:** Enterprise-scale HMIS and human services platform; part of the broader WellSky healthcare software suite. Supports HMIS data exchange and Open Referral HSDS.
- **Integration Hub:** https://hubportal.wellsky.io/
- **Provider API (HSDS):** https://info.wellsky.com/20200520-Provider-API.html
- **FHIR API Documentation:** https://wellsky.dynamicfhir.com/wellsky/basepractice/r4/Home/ApiDocumentation
- **Standards:** REST/JSON; FHIR R4 (WellSky Personal Care); Open Referral Human Service Data Specification (HSDS); HMIS CSV data exchange
- **Authentication:** OAuth 2.0 for FHIR endpoints; API key for integration hub
- **Notes:** WellSky explicitly supports HSDS for service directory interoperability, making it the reference implementation for HSDS-compliant shelter inventory APIs.

---

### PlanStreet

- **Description:** FedRAMP-compliant HMIS and case management SaaS supporting emergency shelter, transitional, and supportive housing. Automated HUD report generation (APR, CAPER).
- **Product Page:** https://www.planstreet.com/hmis-software/
- **HUD Reporting Overview:** https://www.planstreet.com/how-planstreet-simplifies-hud-compliant-hmis-reporting
- **HMIS Specifications Reference:** https://www.planstreet.com/specifications-and-requirements-for-hmis
- **Standards:** HUD HMIS Data Standards (full vendor checklist compliance); FedRAMP Moderate; integrations with Google Suite, Microsoft 365, QuickBooks
- **Authentication:** FedRAMP-grade; OAuth 2.0 implied
- **Notes:** Public API documentation is not available. FedRAMP authorization is the key differentiator enabling direct government agency contracts.

---

### Open Referral / HSDS Reference Implementation

- **Description:** Open-source reference implementation of the Human Services Data Specification (HSDS) and HSDA API, enabling machine-readable directories of health, human, and social services.
- **HSDS Specification:** https://docs.openreferral.org/en/v2.0.1/hsds/
- **HSDA API Specification (OpenAPI):** https://openreferral.github.io/api-specification/
- **GitHub (Specification):** https://github.com/openreferral/specification
- **Standards:** OpenAPI 3.x; JSON/REST; AIRS Taxonomy integration; endorsed by Inform USA (AIRS) as the industry standard for human services resource data exchange
- **Authentication:** API key (implementation-dependent)
- **Notes:** Any new shelter/housing platform should expose an HSDS-compatible `/services`, `/locations`, and `/organizations` endpoint to enable 211 directory integration and community resource tool queries.

---

## Notes

### Emerging Standards

- **FHIR SDOH Clinical Care IG** is gaining adoption as health systems and social care providers move toward bi-directional FHIR-based referrals (e.g., hospitals referring patients to shelters post-discharge). New platforms should plan for FHIR task-based referral workflows even if not required at MVP.
- **HMIS FY 2026 Data Standards** (effective 01 Oct 2025) introduced the `ImplementationID` field in `Export.csv` to facilitate cross-CoC and cross-system analytics. Platforms should implement this from day one rather than retrofitting.
- **HUD HMIS Operational API** — as of the research date there is no single published HUD-mandated API standard beyond the CSV/XML export format. The OpenHMIS API project provides a community-maintained reference, but the ecosystem is fragmented. A modern platform with a well-designed OpenAPI-described REST API would itself become a de-facto reference.
- **Differential privacy and privacy-preserving analytics** are not yet standardised in the homelessness services domain but are an active research area. The HUD Privacy and Security Toolkit (Oct 2024) provides the current regulatory baseline; cryptographic approaches remain opportunities for innovation.

### Key Gaps in Published API Documentation

- **Bitfocus Clarity Operational API** was in Early Access as of the research date; full public documentation not yet available.
- **PlanStreet** provides no public API documentation.
- **CharityTracker** API remains in beta with access restricted to network admins.
- **OpenHMIS API** is aligned to 2014 data standards; no publicly-maintained update to FY 2026 standards has been identified.
