# Shelter & Housing Management

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for emergency shelters, transitional housing, and supportive housing providers to manage client intake, real-time bed availability, case management, and HUD HMIS compliance across an entire Continuum of Care.

Shelter and housing providers operate in high-pressure environments where real-time bed visibility, rapid intake triage, and longitudinal case management directly affect outcomes for people in crisis. Existing HMIS platforms are widely criticised for poor usability, lack of offline capability, and slow innovation cycles, leaving frontline staff to wrestle with software that was not designed for their workflows. This project aims to deliver a modern, mobile-first platform purpose-built for shelter operations, coordinated entry, and community-level homelessness response.

---

## Why Shelter & Housing Management?

- **Frontline staff are underserved by current tools.** Platforms like Clarity Human Services and ClientTrack were designed around compliance reporting, not the intake desk or the outreach van. Staff in the field lack offline capability, and mobile interfaces are an afterthought.
- **Real-time bed availability does not exist at community scale.** Most systems track occupancy at individual sites, but no widely available solution provides sub-second, network-wide visibility into vacant beds across all shelters in a Continuum of Care.
- **Client deduplication remains primitive.** Existing platforms rely on exact-match rules or manual review. Homeless populations frequently use multiple names or lack consistent identification, making probabilistic matching essential and currently absent.
- **Vulnerability assessment tools are fragmented.** VI-SPDAT is being phased out, yet no platform supports multiple assessment instruments with configurable scoring and automated prioritisation routing.
- **Incumbent pricing excludes smaller providers.** Enterprise HMIS platforms require significant implementation budgets that mid-size CoCs and individual shelters cannot justify, despite being mandated by HUD to maintain an HMIS.

---

## Key Features

### Client Intake and Triage

- Rapid intake forms optimised for walk-in and outreach contexts
- Vulnerability assessment tools (VI-SPDAT and configurable alternatives) with version tracking
- Duplicate client detection using probabilistic matching across the CoC network
- HUD-required Universal Data Elements (UDE) and Programme Specific Data Elements (PSDE) collected at intake

### Bed Management and Occupancy

- Real-time bed inventory by programme type, gender, age group, and special requirements (family, pet-friendly, medical)
- Network-wide availability dashboard across all shelters in a CoC
- Reservation and waitlist management
- Barcode/QR code check-in and check-out for high-volume sites
- Nightly occupancy reporting

### Case Management and Service Tracking

- Longitudinal client records with goals, service plans, case notes, and referrals
- Timeline view of client journey from intake through housing placement
- Service documentation per client per visit (meals, showers, storage, employment support, benefits assistance)
- Closed-loop referral management with end-to-end tracking and confirmation
- Outcomes and ROI dashboards quantifying programme impact

### Coordinated Entry and Housing Placement

- Housing prioritisation lists with configurable scoring engines
- Referral management and inter-agency case conferencing tools
- Integration with community-level housing placement workflows
- Automated HUD report generation (APR, ESG CAPER, CoC HIC/PIT, LSA)

### Outreach and Mobile Access

- Mobile-optimised intake and service logging for street outreach workers
- Offline-first design with automatic background sync on reconnection
- Geospatial analytics and outreach reach mapping

---

## AI-Native Advantage

This platform uses AI to address problems that rule-based systems cannot solve well in the homelessness domain. Probabilistic client matching uses name embeddings, date-of-birth distance, and location proximity to identify duplicates with confidence scoring across agencies -- critical for populations who may use multiple names or lack consistent identification. Conversational intake adapts assessment questions based on responses, collecting richer context and producing more accurate prioritisation scores than static questionnaires. Housing placement prediction models, trained on historical cohort outcomes and market conditions, suggest the most likely successful placements for each client. Anomaly detection flags unusual patterns -- sudden service demand spikes, client disappearances followed by re-entry, bed availability mismatches across the network -- enabling proactive staff intervention rather than reactive reporting.

---

## Tech Stack & Deployment

- **Deployment modes:** Self-hosted, cloud, or hybrid. Designed for organisations that may require on-premises data sovereignty due to client data sensitivity.
- **Federated architecture:** Agencies maintain separate tenants but share specific client records with explicit consent under data sharing agreements, enabling CoC-level coordination without centralised data exposure.
- **Offline-first mobile:** Native mobile apps with full offline capability and background sync, essential for outreach workers operating without reliable connectivity.
- **HUD HMIS Data Standards:** Versioned standards layer that can be updated for annual HUD revisions without requiring full database migrations.
- **Security baseline:** Field-level encryption, granular role-based access control, HIPAA-compliant data handling, and comprehensive audit logging. Privacy-preserving aggregate reporting (differential privacy) planned for future releases.
- **Standards compliance:** HUD HMIS Data Standards (FY2024+), FedRAMP alignment for government agency partnerships.

---

## Market Context

The US federal government allocated over USD 4 billion for homelessness assistance in FY2024, and every CoC receiving HUD funding is mandated to maintain an HMIS -- creating a required-software market across roughly 400 CoCs and thousands of individual shelter providers. Existing platforms such as Clarity Human Services, ClientTrack, and CaseWorthy dominate but face criticism for poor usability and high implementation costs. A modern, open-source alternative would compete most effectively among mid-size CoCs that need full HUD compliance without enterprise-scale budgets.

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
