# 449 – Shelter & Housing Management — Feature & Functionality Survey

> Candidate #449 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| Clarity Human Services (Bitfocus) | Commercial SaaS | Proprietary | https://www.bitfocus.com/ |
| ClientTrack (Bowman Systems) | Commercial SaaS | Proprietary | https://bowmansystems.com/ |
| ResiDex | Commercial SaaS | Proprietary | https://mission-tracker.com/ |
| CaseWorthy | Commercial SaaS | Proprietary | https://www.caseworthy.com/ |
| CharityTracker | Commercial SaaS | Proprietary | https://www.charitytracker.com/ |
| PlanStreet | Commercial SaaS | Proprietary | https://www.planstreet.com/ |
| Open Path HMIS | Open Source | Open Source (Boston CAS) | https://greenriver.com/openpath/ |
| WellSky HMIS | Commercial SaaS | Proprietary | https://wellsky.com/hmis-software/ |

## Feature Analysis by Solution

### Clarity Human Services (Bitfocus)

**Core features**
- Client intake and rapid assessment workflows
- VI-SPDAT vulnerability assessment tools (with version tracking)
- Real-time bed management and occupancy tracking
- Coordinated Entry integration with prioritization and matching
- Case management with longitudinal client records
- Service tracking and outcome documentation
- Housing referral management
- HUD HMIS compliance (2026 Data Standards implemented October 2025)
- Customizable assessment tools and scoring processors
- Geospatial mapping and analytics for outreach reach
- Mobile-friendly coordinated entry workflows (November 2025 update)
- Comprehensive reporting dashboards

**Differentiating features**
- Assessment Level Analysis Dashboard (June 2025) distinguishing enrollment-linked vs. non-linked assessments
- Customizable scoring processors for need-based prioritization
- Real-time referral tracking to reduce bottlenecks
- Geospatial analytics for outreach effectiveness
- Seamless coordinated entry integration as primary workflow (not add-on)

**UX patterns**
- Coordinated entry-first: CE workflow is core design, not peripheral
- Mobile-progressive: responsive interfaces for field and office
- Assessment transparency: clear visibility into assessment-enrollment linkage
- Data-driven prioritization: scores surface automatically

**Integration points**
- HUD HMIS data standards (versioned for 2026 updates)
- Geospatial tools for mapping
- Assessment version tracking (VI-SPDAT, etc.)
- Real-time referral API

**Known gaps**
- Offline mobile capability not documented
- Barcode/scanning integration for intake not mentioned
- Multi-tenant federated architecture (CoC-level data sharing) not explicitly documented
- Volunteer/staffing management not mentioned

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### ClientTrack (Bowman Systems)

**Core features**
- Client intake and assessment workflows
- HMIS data collection aligned with HUD requirements
- Case management with service history
- Bed tracking and availability management
- Housing placement workflows
- Resource coordination
- Real-time data sharing across agencies
- Mobile accessibility for field staff
- HUD compliance reporting
- Robust reporting tools

**Differentiating features**
- Highly rated for ease of use (recognized as "overall top pick")
- Integrated bed tracking with multi-provider visibility
- Mobile-first field access
- Real-time inter-agency data sharing

**UX patterns**
- Field-staff optimized: mobile-first with ease-of-use emphasis
- Real-time collaboration: agencies see updates across system
- Intake-to-placement linearity: clear client journey

**Integration points**
- Multi-provider real-time sharing
- HMIS data standard compliance
- Mobile apps for field staff
- HUD reporting export

**Known gaps**
- Coordinated Entry support not explicitly documented
- Vulnerability assessment tools not mentioned
- Offline capability not documented
- Analytics and trending beyond basic reporting unclear

**Licence / IP notes**
- Proprietary commercial SaaS (Bowman Systems); no IP concerns

---

### ResiDex

**Core features**
- Client intake and demographics tracking
- Interactive BedBoard for real-time occupancy management
- Case notes and progress tracking
- Service documentation
- Customizable reporting
- HUD HMIS compliance and integration
- Outcome tracking
- Mobile access for staff
- Program outcomes monitoring

**Differentiating features**
- Interactive BedBoard: visual, real-time bed management interface
- Outcome-focused: programme outcomes tracking as primary feature
- Mid-market positioning: focused on mid-sized shelters

**UX patterns**
- Visual-first bed management: graphical occupancy interface
- Outcome-centric: results and resident stability emphasized
- Streamlined: focused on essential features vs. comprehensive system

**Integration points**
- HMIS integration for compliance reporting
- Standard HUD data export

**Known gaps**
- Coordinated Entry support not documented
- Vulnerability assessments not mentioned
- Multi-agency data sharing limited to HMIS integration
- Offline mobile capability not mentioned
- Analytics and reporting appear basic

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### CaseWorthy

**Core features**
- Client intake with customizable forms
- HMIS-specific data collection (native forms and fields)
- Real-time bed management across multiple programme types
- Case management with configurable workflows
- Referral management with closed-loop tracking
- Coordinated Entry prioritization and eligibility engine
- Customizable data fields, forms, and workflows
- HUD reporting (LSA, CAPER, HUD CSV)
- HIPAA-compliant data handling
- Role-based access control with granular permissions
- Real-time dashboards and aggregate reporting

**Differentiating features**
- Out-of-the-box HUD compliance (meets full HUD Vendor Checklist)
- Highly customizable configuration (non-code form/workflow designer)
- Closed-loop referral management
- Prioritization engine specifically for coordinated entry
- Granular role and permission management
- Aggregate community-level reporting

**UX patterns**
- Customization-first: staff can design workflows without code
- HUD-centric: compliance is embedded, not bolted-on
- Workflow-driven: closed-loop referrals with tracking
- Permission-centric: data access controlled at field level

**Integration points**
- HUD reporting standards (LSA, CAPER, HUD CSV)
- HIPAA compliance
- Role-based access control
- Multi-provider aggregation

**Known gaps**
- Vulnerability assessment tools not mentioned
- Coordinated entry modules present but depth unclear
- Offline mobile not documented
- Geospatial analytics not mentioned
- Integration with external assessment tools underspecified

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### CharityTracker

**Core features**
- Bed management with real-time availability tracking
- Client check-in/check-out with barcode scanning
- Case management
- Assessment tools
- Service coordination for housing and shelters
- HMIS and non-HMIS agency support
- Outcomes and analytics dashboards
- HUD compliance reporting
- Multi-site network operation
- Duplicate detection (reduces duplication by average 20%)

**Differentiating features**
- Barcode scanning integration for check-in/check-out
- Hybrid HMIS/non-HMIS support (non-HMIS agencies can participate with shared data)
- Scale: used in 2,000+ cities, tracks 3.5 million cases, 10+ million assistance records
- Outcomes emphasis: real-time dashboards quantifying client progress and ROI
- Duplication reduction: 20% average reduction through system-wide deduplication

**UX patterns**
- Barcode-first: scanning reduces manual data entry
- Outcomes-centric: ROI and progress dashboards primary interface
- Community-level: designed for multi-agency networks from inception
- Practical: addresses real operational pain (duplication, availability)

**Integration points**
- Barcode scanning hardware
- HMIS and non-HMIS agency data sharing
- Real-time availability across multi-agency network
- Outcomes reporting

**Known gaps**
- Vulnerability assessments not mentioned
- Coordinated entry workflows not explicitly documented
- Offline mobile capability not mentioned
- Geospatial analytics not documented
- Mobile access documented as present but light on detail

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

### PlanStreet

**Core features**
- HMIS-compliant case management
- Client intake with customizable forms
- Case management with service tracking
- HUD reporting (APR, CAPER) with document automation
- Transitional housing, sober living, supportive housing support
- Real-time data access for decision-making
- Collaboration tools (shared assessments, files, notes)
- Data export and customizable reports
- Multi-provider network support
- FedRAMP compliance (government-grade)

**Differentiating features**
- FedRAMP compliance: enables government agency partnerships
- Automated HUD report generation (APR, CAPER in clicks)
- Customizable reporting format flexibility
- Transitional housing and sober living focus (vs. pure shelter/emergency focus)
- Network-wide collaboration with shared documents

**UX patterns**
- Compliance-driven: HUD requirements embedded
- Collaboration-first: shared assessments and documentation
- Report-ready: automation reduces manual compliance labour
- Multi-programme: single platform for emergency shelter, transitional, supportive

**Integration points**
- HUD HMIS data standards
- FedRAMP government security framework
- Export and reporting APIs
- Inter-agency collaboration tools

**Known gaps**
- Vulnerability assessments not mentioned
- Bed management capabilities underspecified
- Coordinated entry support not explicitly documented
- Real-time bed availability across network unclear
- Offline mobile not documented
- Geospatial analytics not mentioned

**Licence / IP notes**
- Proprietary commercial SaaS with government certifications; no IP concerns

---

### Open Path HMIS

**Core features**
- HUD-compliant HMIS data collection
- Coordinated Entry system connecting outreach, shelter, housing
- Client-level data management
- Housing and service tracking
- CoC-level reporting (LSA, APR, HIC, PIT, CAPER)
- Role-based access control
- Data encryption in transit and at rest
- Shared data governance aligned with HUD standards
- Prioritization lists and housing match management
- Open-source codebase with community contributions

**Differentiating features**
- Open-source: free to self-host; code available for customization
- Coordinated entry as native feature (not add-on)
- Green River stewardship: professional support available (optional)
- Transparent, community-driven development
- Full HUD compliance without vendor lock-in

**UX patterns**
- Developer-friendly: source code available, customization possible
- CoC-first: designed for community-wide coordination
- Interoperable: built to work across multiple organisations
- Privacy-first: granular access control and data governance

**Integration points**
- Open-source code repository (customization via code)
- HUD HMIS data standards
- REST APIs (if present; not extensively documented)
- Standard HMIS data exports

**Known gaps**
- Vulnerability assessment tools not documented
- Mobile app capabilities underspecified
- Offline mobile not mentioned
- Real-time bed availability features unclear
- User experience design compared to commercial platforms unclear
- Community activity and maintenance status: active but smaller than commercial vendors

**Licence / IP notes**
- Open source (Boston CAS / Green River model); free for government use
- Allows local customization and community contributions
- Source code available on GitHub-like platforms
- No licensing restrictions for government or nonprofit use

---

### WellSky HMIS

**Core features**
- HMIS data collection and management
- Case management for housing and homelessness services
- Service tracking
- HUD compliance and reporting
- Real-time data capabilities
- Housing and shelter operations support

**Differentiating features**
- Enterprise-scale (part of larger WellSky suite)
- Likely integrated with broader WellSky portfolio

**UX patterns**
- Enterprise-oriented: part of platform ecosystem

**Integration points**
- WellSky ecosystem integration
- HUD reporting

**Known gaps**
- Feature documentation is very sparse in public domain
- Coordinated entry capabilities unclear
- Vulnerability assessment tools not mentioned
- Bed management specifics unknown
- Offline mobile not documented
- Competitive differentiation unclear

**Licence / IP notes**
- Proprietary commercial SaaS; no IP concerns

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

These capabilities are present in nearly every solution and are essential:

- Client intake with HUD-required data elements
- Case management with longitudinal client records
- Bed/occupancy management (real-time or near-real-time)
- Service tracking and documentation
- HUD HMIS compliance and reporting (LSA, APR, CAPER, HIC, PIT)
- Role-based access control
- Data encryption and security
- Multi-programme/multi-agency support
- Export/reporting capabilities for audits and funders

### Differentiating Features

Capabilities present in some solutions that provide competitive advantage:

- **Barcode scanning for check-in** (CharityTracker) – Eliminates manual entry, improves accuracy, speeds throughput at high-volume sites
- **Coordinated Entry as native core** (Clarity, Open Path) – Not a module but fundamental system design, enabling real-time prioritization and referral tracking
- **Vulnerability assessment integration** (Clarity with VI-SPDAT) – Built-in assessment tools with version tracking, enabling automated prioritization
- **Geospatial analytics** (Clarity) – Mapping of outreach reach and service gaps by geography
- **Closed-loop referral management** (CaseWorthy) – Referrals tracked end-to-end with closure confirmation
- **Customizable workflow designer** (CaseWorthy) – Non-code form/workflow configuration enabling organisation-specific processes
- **Outcomes and ROI dashboards** (CharityTracker) – Quantified programme impact vs. transactional reporting
- **Automated HUD report generation** (PlanStreet) – Document templates that auto-populate from case data
- **FedRAMP compliance** (PlanStreet) – Enables direct government agency partnerships
- **Hybrid HMIS/non-HMIS support** (CharityTracker) – Non-HMIS agencies can participate with shared data without full HMIS compliance burden

### Underserved Areas / Opportunities

Gaps that multiple solutions share, representing genuine opportunities:

- **True offline-first mobile** – Most systems support mobile access, but offline capability (collecting intake/service data without connectivity, syncing on reconnection) is not standard. Opportunity: field-staff-first design with full offline capability and automatic sync, critical for outreach and mobile shelter contexts.
- **Probabilistic client deduplication** – Most systems rely on exact-match rules or manual review. Opportunity: probabilistic matching using name embeddings, date-of-birth distance, location proximity, and historical aliases to auto-flag duplicates while minimizing false positives (critical for homeless populations with multiple names/identities).
- **Real-time bed availability across community** – While some systems track occupancy at single sites, true real-time network-wide visibility (knowing vacant beds across all shelters in a CoC instantly) is not standard. Opportunity: event-driven architecture enabling sub-second updates to shared occupancy state.
- **Vulnerability assessment interoperability** – VI-SPDAT is being phased out, but no standard replacement assessment tool is built into platforms. Opportunity: support multiple assessment tools (VI-SPDAT, SPDAT, newer tools) with configurable scoring and auto-routing based on scores.
- **Housing placement prediction** – No platform uses historical outcomes (who was placed, who returned, what interventions worked) to predict success for current clients. Opportunity: ML models to suggest most-likely-successful placements based on cohort history.
- **Contextual data integration** – Most systems don't link to external data (weather, shelter capacity, local housing market, labour demand). Opportunity: integrate external data feeds to surface risk signals (e.g., forecast of cold weather → alert to prepare emergency shelter capacity).
- **Peer support and lived experience validation** – Most systems are staff-centric; no platform channels peer support expertise or validates data through lived-experience voices. Opportunity: workflows for peer counsellors, lived-experience advisors to validate case plans and outcomes.
- **Privacy-preserving data sharing** – CoC-level reporting requires sharing sensitive client data; most systems use role-based access but not cryptographic privacy. Opportunity: differential privacy or homomorphic encryption to enable aggregate reporting without individual-level data access.

### AI-Augmentation Candidates

Features currently implemented with manual/rule-based approaches where AI could provide measurably better results:

- **Probabilistic client matching** – Current: exact-name or manual matching. AI: embeddings + temporal/geographic proximity to identify duplicates with confidence scoring.
- **Automated vulnerability assessment** – Current: VI-SPDAT questionnaire. AI: conversational intake that adapts questions based on responses, collects richer context, produces more accurate prioritization scores.
- **Housing placement prediction** – Current: staff judgment + HMIS history. AI: predict placement success (will client accept offer? stay housed?) based on cohort outcomes, client history, market conditions.
- **Service recommendation** – Current: static case plans. AI: recommend next services based on client history, outcome patterns in cohort, and market availability (e.g., if employment pathway succeeded for similar clients, prioritize job training).
- **Anomaly detection** – Current: none. AI: flag unusual patterns (client disappeared then re-entered, sudden service demand spike, bed availability mismatch across network) to alert staff.
- **Natural language case notes** – Current: structured forms or freeform text. AI: conversational case note entry that auto-tags, links to client problems/goals, and suggests relevant prior patterns.
- **Outreach targeting** – Current: geographic zones and outreach worker judgment. AI: predict highest-need micro-neighborhoods (blocks) and optimal outreach timing based on historical data.
- **Workflow optimization** – Current: static intake forms and referral processes. AI: adapt form fields and routing based on client responses, predict likely next steps, suggest process improvements based on bottleneck analysis.
- **Return-to-homelessness prediction** – Current: ex-post reporting. AI: predict churn risk after placement based on early warning signals, trigger prevention interventions.

---

## Legal & IP Summary

All commercial platforms reviewed are proprietary SaaS offerings with standard commercial licensing. Open Path HMIS is open-source software released under a community-friendly model by Boston/Green River, allowing government and nonprofit free deployment and customization.

No copyright, patent, or licensing conflicts identified. All feature descriptions paraphrased from public documentation; no proprietary code consulted. VI-SPDAT itself is a public-domain assessment tool developed by Community Solutions; platform implementations track versions appropriately to avoid drift from original methodology. HUD HMIS Data Standards are federal requirements, not proprietary intellectual property.

---

## Recommended Feature Scope

Based on the above analysis, a new shelter/housing management platform should prioritise:

**Must-have (MVP)**
- Client intake with HUD-required data elements (demographics, income, needs)
- Case management with longitudinal records and service history
- Real-time bed/occupancy tracking with availability by programme type
- Multi-site network operation (visibility across agencies)
- HUD HMIS compliance (LSA, APR, CAPER reporting)
- Role-based access control and data encryption
- Coordinated Entry workflow with prioritization and referral tracking
- Service documentation and outcome tracking
- Mobile access for field staff

**Should-have (v1.1)**
- VI-SPDAT or equivalent vulnerability assessment with auto-scoring
- Barcode/QR code check-in for intake and occupancy tracking
- Offline-capable mobile app with automatic sync on reconnection
- Geospatial analytics and outreach reach mapping
- Customizable workflow and form designer (non-code)
- Closed-loop referral management with confirmation tracking
- Probabilistic client deduplication across multi-site network
- Outcomes and ROI dashboards
- Automated HUD report generation from case data
- Federated data sharing (multiple agencies with consent-based access)

**Nice-to-have (backlog)**
- Housing placement prediction models based on cohort history
- Service recommendation engine based on outcome patterns
- Anomaly detection for unusual patterns (client churn, capacity mismatches)
- Natural language case note interface with auto-tagging
- Integration with external data (weather, housing market, labour demand)
- Peer support and lived-experience validation workflows
- Return-to-homelessness prediction and prevention triggers
- Privacy-preserving aggregate reporting (differential privacy or homomorphic encryption)
- Contextual outreach targeting and optimal timing recommendations
- Workflow optimization suggestions based on bottleneck analysis

---

## Sources

- [Clarity Human Services | Bitfocus](https://www.bitfocus.com/)
- [Coordinated Entry System & HMIS Software | Clarity](https://www.bitfocus.com/coordinated-entry-products)
- [HMIS Software: Homeless Management Information System | Bitfocus](https://www.bitfocus.com/hmis-software)
- [2026 HMIS Data Standards: DIT Impacts | Bitfocus Help](https://help.bitfocus.com/2026-hmis-data-standards-dit-impacts)
- [Clarity Human Services: June 2025 Feature Updates | Bitfocus](https://help.bitfocus.com/clarity-human-services-june-2025-feature-updates)
- [Coordinated Entry, Unlocked: Your November 2025 Clarity Update | Bitfocus](https://www.bitfocus.com/blog/coordinated-entry-unlocked-your-november-2025-clarity-update)
- [ClientTrack HMIS User Manual | Georgia DCA](https://dca.georgia.gov/document/manuals/ga-hmis-clienttrack-user-manual/download)
- [ResiDex Homeless Shelter Case Management | MissionTracker](https://mission-tracker.com/resident-tracker/why-resident-tracker/)
- [CaseWorthy HMIS Software](https://www.caseworthy.com/who-we-serve/hmis-software-and-housing-services/)
- [HMIS+ Case Management | CaseWorthy](https://www.caseworthy.com/solutions/hmis-case-management/)
- [Top HMIS Features That Will Improve Your Coordinated Entry | CaseWorthy](https://www.caseworthy.com/articles/top-hmis-features-that-will-improve-your-coordinated-entry-system/)
- [CharityTracker Case Management Software for Housing & Shelters](https://www.charitytracker.com/who-we-serve/hmis)
- [PlanStreet HMIS Software for Housing and Homeless Services](https://www.planstreet.com/hmis-software/)
- [Streamline HUD-Compliant HMIS Reporting with PlanStreet](https://www.planstreet.com/how-planstreet-simplifies-hud-compliant-hmis-reporting)
- [Open Path HMIS | Green River](https://greenriver.com/openpath/)
- [User-focused HMIS Built in Partnership with Community Orgs | Open Path HMIS](https://greenriver.com/openpath/hmis/)
- [Coordinated Entry Software for HMIS | Open Path](https://greenriver.com/openpath/hmis/coordinated-entry/)
- [WellSky HMIS: Housing and Homelessness Information Management System](https://wellsky.com/hmis-software/)
- [Service Prioritization Decision Assistance Tool (VI-SPDAT) | Everyone Home](https://everyonehome.org/wp-content/uploads/2016/02/VI-SPDAT-2.0-Single-Adults.pdf)
- [Selecting the Best Vulnerability Assessment Tool for Nonprofits | CaseWorthy](https://www.caseworthy.com/articles/the-best-vulnerability-assessment-tool/)
- [Top 10 Best Homeless Shelter Management Software of 2026 | Gitnux](https://gitnux.org/best/homeless-shelter-management-software/)
