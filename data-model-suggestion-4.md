# Data Model Suggestion 4: Graph + Relational Polyglot Architecture

> Project: Shelter & Housing Management (449)
> Model Type: Graph Database (Neo4j) + Relational (PostgreSQL) + Time-Series (TimescaleDB)
> Databases: Neo4j Aura / Community for relationship graphs, PostgreSQL for transactional data, TimescaleDB for occupancy time-series
> Generated: 2026-05-25

---

## Overview

This model uses a polyglot persistence architecture where each database technology handles the aspect of the domain it models best:

1. **Neo4j (Graph Database)** for the relationship-heavy core: client networks, referral chains, agency collaboration, coordinated entry matching, service pathways, and housing placement networks. The homelessness services domain is fundamentally a graph problem -- clients are connected to households, agencies, programmes, case workers, referrals, housing units, and each other through complex, many-to-many relationships that change over time. Graph traversal queries ("find all agencies that have served this client within 2 degrees of separation", "trace the referral chain from outreach contact to housing placement", "identify clients connected to a household where any member has been placed") are natural in a graph model and prohibitively expensive as relational JOINs.

2. **PostgreSQL** for transactional operations, HUD compliance data, user authentication, and the operational data that requires ACID guarantees: bed check-in/check-out, financial records, audit logs, and HMIS CSV import/export.

3. **TimescaleDB (PostgreSQL extension)** for time-series data: bed occupancy metrics, census snapshots, client journey milestones, service utilization trends, and real-time capacity monitoring. TimescaleDB's hypertables and continuous aggregates are purpose-built for the temporal queries that drive shelter dashboards and HUD reporting.

---

## Graph Model (Neo4j)

### Node Types

```cypher
// ===== PERSON NODES =====

// Client node -- the central entity in the graph
// Relationships to everything else radiate from here
(:Client {
    personalId: "uuid",
    firstName: "John",
    lastName: "Smith",
    middleName: "A",
    dateOfBirth: date("1985-03-15"),
    veteranStatus: 1,
    nameDataQuality: 1,
    dobDataQuality: 1,
    ssnLastFourHash: "a1b2c3...",  // Never store actual SSN in graph
    races: [1, 5],
    genders: [1],
    preferredLanguage: "en",
    photoUrl: "https://...",
    sourceOrganizationId: "uuid",
    createdAt: datetime("2026-01-15T10:30:00Z"),
    updatedAt: datetime("2026-03-22T14:15:00Z")
})

// Alias nodes for deduplication -- each alias is a separate node
// linked to the canonical client
(:ClientAlias {
    aliasId: "uuid",
    firstName: "Johnny",
    lastName: "Smith",
    dateOfBirth: date("1985-03-15"),
    source: "Agency B Intake",
    createdAt: datetime("2026-02-01T08:00:00Z")
})

// Staff / case worker node
(:StaffMember {
    userId: "uuid",
    firstName: "Maria",
    lastName: "Garcia",
    email: "mgarcia@shelter.org",
    role: "case_manager",
    isActive: true
})


// ===== ORGANIZATION NODES =====

(:ContinuumOfCare {
    cocId: "uuid",
    cocCode: "CA-500",
    cocName: "San Jose/Santa Clara City & County CoC",
    geographicArea: "Santa Clara County"
})

(:Organization {
    organizationId: "uuid",
    name: "Downtown Emergency Shelter",
    victimServiceProvider: false,
    ein: "12-3456789",
    isActive: true
})

(:Project {
    projectId: "uuid",
    name: "Emergency Shelter - Main Campus",
    projectType: 1,       // ES-NBN
    trackingMethod: 3,    // Night-by-Night
    housingType: 1,       // Site-based
    operatingStartDate: date("2020-01-01"),
    isActive: true
})

(:Funder {
    funderId: "uuid",
    funderType: 1,        // HUD:CoC
    grantId: "CA-500-2025-001",
    startDate: date("2025-10-01"),
    endDate: date("2026-09-30")
})


// ===== ENROLLMENT & SERVICE NODES =====

(:Enrollment {
    enrollmentId: "uuid",
    entryDate: date("2026-01-15"),
    exitDate: null,
    destination: null,
    moveInDate: null,
    enrollmentCoC: "CA-500",
    disablingCondition: 1,
    livingSituation: 16,
    lengthOfStay: 2,
    timesHomelessPastThreeYears: 4,
    monthsHomelessPastThreeYears: 112,
    relationshipToHoH: 1,
    isActive: true
})

(:Household {
    householdId: "uuid",
    createdAt: datetime("2026-01-15T10:30:00Z")
})

(:Service {
    serviceId: "uuid",
    dateProvided: date("2026-02-10"),
    recordType: 200,
    typeProvided: 1,
    description: "Employment workshop - resume writing",
    faAmount: null
})

(:CaseNote {
    noteId: "uuid",
    noteDate: datetime("2026-02-15T14:30:00Z"),
    noteType: "progress",
    title: "Employment progress update",
    isConfidential: false
    // Note text stored in PostgreSQL for full-text search and encryption
})

(:ServicePlan {
    planId: "uuid",
    planType: "housing",
    startDate: date("2026-01-20"),
    status: "active"
})

(:Goal {
    goalId: "uuid",
    description: "Obtain stable employment",
    targetDate: date("2026-06-30"),
    status: "in_progress",
    completedDate: null
})


// ===== ASSESSMENT NODES =====

(:AssessmentInstrument {
    instrumentId: "uuid",
    name: "VI-SPDAT 2.0 - Single Adults",
    version: "2.0",
    maxScore: 20,
    isActive: true
})

(:Assessment {
    assessmentId: "uuid",
    assessmentDate: date("2026-01-15"),
    assessmentType: 3,    // In-person
    assessmentLevel: 2,   // Housing
    totalScore: 14,
    prioritizationStatus: 2  // Pending
})


// ===== BED & HOUSING NODES =====

(:BedUnit {
    unitId: "uuid",
    unitName: "Bed 2A-001",
    unitType: "single",
    maxOccupants: 1,
    status: "available",
    genderRestriction: "male",
    isAccessible: false,
    isPetFriendly: false,
    floor: "2",
    building: "Main",
    qrCode: "BED-2A-001"
})

(:HousingUnit {
    housingUnitId: "uuid",
    address: "123 Main St, Apt 4B",
    bedroomCount: 2,
    monthlyRent: 1200.00,
    subsidyType: "Section 8",
    status: "available",
    isAccessible: true
})


// ===== COORDINATION NODES =====

(:PrioritizationList {
    listId: "uuid",
    name: "Chronic Homeless Priority List",
    listType: "chronic",
    isActive: true
})

(:Referral {
    referralId: "uuid",
    referralType: "housing",
    referralDate: datetime("2026-03-01T10:00:00Z"),
    status: "pending",
    priorityScore: 85.5
})

(:HousingPlacement {
    placementId: "uuid",
    placementDate: date("2026-03-15"),
    moveInDate: date("2026-04-01"),
    status: "stabilizing",
    monthlyRent: 1200.00,
    clientRentPortion: 400.00,
    returnToHomelessness: false
})

(:CaseConference {
    conferenceId: "uuid",
    conferenceDate: datetime("2026-02-28T09:00:00Z"),
    conferenceType: "by_name_list"
})

(:DataSharingAgreement {
    agreementId: "uuid",
    name: "Santa Clara CoC Data Sharing Agreement 2026",
    effectiveDate: date("2026-01-01"),
    expirationDate: date("2026-12-31"),
    isActive: true
})
```

### Relationship Types

```cypher
// ===== ORGANIZATIONAL HIERARCHY =====
(:Organization)-[:BELONGS_TO]->(:ContinuumOfCare)
(:Project)-[:OPERATED_BY]->(:Organization)
(:Project)-[:SERVES_COC]->(:ContinuumOfCare)
(:Project)-[:FUNDED_BY]->(:Funder)
(:StaffMember)-[:WORKS_FOR]->(:Organization)
(:StaffMember)-[:ASSIGNED_TO]->(:Project)
(:Organization)-[:SIGNED {signedDate, signedBy}]->(:DataSharingAgreement)

// ===== CLIENT IDENTITY GRAPH =====
// Core identity relationships
(:ClientAlias)-[:ALIAS_OF {confidence: 0.95, method: "probabilistic"}]->(:Client)
(:Client)-[:MERGED_INTO {mergeDate, method, confidence, mergedBy}]->(:Client)

// Household membership
(:Client)-[:MEMBER_OF {relationshipToHoH: 1}]->(:Household)

// Consent chain
(:Client)-[:CONSENTED_TO {consentDate, expirationDate, revocationDate}]->(:DataSharingAgreement)

// ===== ENROLLMENT GRAPH =====
// Client journey through the system
(:Client)-[:ENROLLED_IN {entryDate, isActive: true}]->(:Enrollment)
(:Enrollment)-[:AT_PROJECT]->(:Project)
(:Enrollment)-[:IN_HOUSEHOLD]->(:Household)

// Services and case management
(:Service)-[:PROVIDED_DURING]->(:Enrollment)
(:Service)-[:PROVIDED_BY]->(:StaffMember)
(:CaseNote)-[:RECORDED_FOR]->(:Enrollment)
(:CaseNote)-[:WRITTEN_BY]->(:StaffMember)
(:ServicePlan)-[:CREATED_FOR]->(:Enrollment)
(:ServicePlan)-[:CREATED_BY]->(:StaffMember)
(:Goal)-[:PART_OF]->(:ServicePlan)

// Assessments
(:Assessment)-[:CONDUCTED_FOR]->(:Client)
(:Assessment)-[:LINKED_TO]->(:Enrollment)
(:Assessment)-[:USES_INSTRUMENT]->(:AssessmentInstrument)
(:Assessment)-[:CONDUCTED_BY]->(:StaffMember)

// ===== BED MANAGEMENT GRAPH =====
(:BedUnit)-[:LOCATED_AT]->(:Project)
(:Client)-[:OCCUPIES {checkInTime, checkOutTime, checkInMethod}]->(:BedUnit)
(:Client)-[:RESERVED {reservationDate, expirationTime, status}]->(:BedUnit)

// ===== COORDINATED ENTRY GRAPH =====
// Priority list membership
(:Client)-[:ON_LIST {priorityScore, rank, dateAdded, status}]->(:PrioritizationList)
(:PrioritizationList)-[:MANAGED_BY]->(:ContinuumOfCare)

// Referral chain -- this is where graph shines
(:Referral)-[:REFERS]->(:Client)
(:Referral)-[:FROM_ORG]->(:Organization)
(:Referral)-[:TO_ORG]->(:Organization)
(:Referral)-[:MADE_BY]->(:StaffMember)
(:Referral)-[:TARGETS]->(:Project)
(:Referral)-[:FOR_UNIT]->(:HousingUnit)

// Housing placement chain
(:HousingPlacement)-[:PLACES]->(:Client)
(:HousingPlacement)-[:RESULTED_FROM]->(:Referral)
(:HousingPlacement)-[:AT_UNIT]->(:HousingUnit)
(:HousingPlacement)-[:UNDER_PROJECT]->(:Project)

// Case conferencing
(:CaseConference)-[:DISCUSSED]->(:Client)
(:CaseConference)-[:FACILITATED_BY]->(:StaffMember)
(:CaseConference)-[:HELD_BY]->(:ContinuumOfCare)

// ===== CROSS-AGENCY RELATIONSHIP GRAPH =====
// These relationships are unique to the graph model and nearly
// impossible to express efficiently in a relational database

// Client served by multiple agencies (implicit through enrollments)
// but also explicit inter-agency relationships:
(:Organization)-[:REFERRED_CLIENT_TO {count, lastDate}]->(:Organization)
(:Organization)-[:COLLABORATES_WITH {agreementId}]->(:Organization)
(:StaffMember)-[:CASE_MANAGED {startDate, endDate}]->(:Client)
```

### Graph Queries for Key Operations

```cypher
// 1. Client journey -- trace a client's complete path through the system
// This query is the "killer feature" of the graph model
MATCH path = (c:Client {personalId: $clientId})-[*1..5]-(connected)
WHERE connected:Enrollment OR connected:Assessment OR connected:Referral
    OR connected:HousingPlacement OR connected:Service
RETURN path
ORDER BY connected.date DESC

// 2. Find duplicate clients using the alias graph
// Traverse alias chains to find potential matches
MATCH (c1:Client)-[:ALIAS_OF|MERGED_INTO*1..3]-(c2:Client)
WHERE c1.personalId <> c2.personalId
    AND c1.dateOfBirth = c2.dateOfBirth
RETURN c1, c2, 
    apoc.text.levenshteinDistance(c1.lastName, c2.lastName) AS nameDistance

// 3. Referral chain analysis -- trace how clients move through agencies
MATCH (c:Client)<-[:REFERS]-(r:Referral)-[:FROM_ORG]->(fromOrg:Organization),
      (r)-[:TO_ORG]->(toOrg:Organization)
WHERE c.personalId = $clientId
RETURN fromOrg.name AS referringAgency, 
       toOrg.name AS receivingAgency,
       r.referralDate AS date,
       r.status AS status,
       r.priorityScore AS score
ORDER BY r.referralDate

// 4. Coordinated Entry matching -- find best housing match for a client
// considering their network of services, assessments, and preferences
MATCH (c:Client {personalId: $clientId})-[:ENROLLED_IN]->(e:Enrollment)-[:AT_PROJECT]->(p:Project)
MATCH (c)<-[:CONDUCTED_FOR]-(a:Assessment)
WHERE a.assessmentLevel = 2  // Housing assessment
WITH c, MAX(a.totalScore) AS maxScore, COLLECT(DISTINCT p.projectType) AS projectHistory
MATCH (hu:HousingUnit {status: "available"})
MATCH (hu)<-[:AT_UNIT]-(proj:Project)
WHERE NOT (c)-[:PLACED_AT]->(hu)  // Not already placed here
RETURN hu, proj,
    CASE WHEN 3 IN projectHistory THEN 10 ELSE 0 END +  // Bonus if PSH history
    maxScore AS matchScore
ORDER BY matchScore DESC
LIMIT 10

// 5. Network-wide bed availability across a CoC
MATCH (coc:ContinuumOfCare {cocCode: $cocCode})<-[:SERVES_COC]-(p:Project)<-[:LOCATED_AT]-(b:BedUnit)
WITH p, b,
     COUNT(CASE WHEN b.status = 'available' THEN 1 END) AS available,
     COUNT(CASE WHEN b.status = 'occupied' THEN 1 END) AS occupied,
     COUNT(b) AS total
RETURN p.name AS projectName, p.projectType AS type,
       available, occupied, total,
       toFloat(occupied) / total * 100 AS occupancyRate
ORDER BY available DESC

// 6. Identify clients at risk of return to homelessness
// based on their service graph and placement history
MATCH (c:Client)-[:PLACED_AT]->(hp:HousingPlacement {status: "placed"})
WHERE hp.placementDate > date() - duration({months: 6})
OPTIONAL MATCH (c)-[:ENROLLED_IN]->(e:Enrollment)-[:AT_PROJECT]->(prev:Project)
WHERE prev.projectType IN [1, 4]  // Previous ES or SO enrollment
WITH c, hp, COUNT(DISTINCT e) AS previousHomelessEpisodes
WHERE previousHomelessEpisodes >= 2
OPTIONAL MATCH (c)<-[:PROVIDED_DURING]-(s:Service)
WHERE s.dateProvided > hp.placementDate
WITH c, hp, previousHomelessEpisodes, COUNT(s) AS postPlacementServices
WHERE postPlacementServices < 3  // Low service engagement post-placement
RETURN c.firstName, c.lastName, c.personalId,
       hp.placementDate, previousHomelessEpisodes, postPlacementServices
ORDER BY previousHomelessEpisodes DESC, postPlacementServices ASC

// 7. Agency collaboration network -- which agencies work together most?
MATCH (o1:Organization)-[:REFERRED_CLIENT_TO]->(o2:Organization)
MATCH (o1)-[:BELONGS_TO]->(coc:ContinuumOfCare {cocCode: $cocCode})
RETURN o1.name AS fromAgency, o2.name AS toAgency,
       o1.REFERRED_CLIENT_TO.count AS referralCount
ORDER BY referralCount DESC

// 8. Service pathway analysis -- what service sequences lead to housing?
MATCH (c:Client)-[:ENROLLED_IN]->(e:Enrollment {isActive: false}),
      (e)-[:AT_PROJECT]->(p:Project)
WHERE e.destination IN [410, 411, 421, 422, 423]  // Permanent housing destinations
MATCH (c)<-[:PROVIDED_DURING]-(s:Service)
WHERE s.dateProvided >= e.entryDate AND s.dateProvided <= e.exitDate
WITH c, e, COLLECT(s.typeProvided) AS serviceSequence, e.exitDate - e.entryDate AS stayDuration
RETURN serviceSequence, COUNT(c) AS clientCount, AVG(stayDuration) AS avgStayDays
ORDER BY clientCount DESC
LIMIT 20
```

---

## Relational Store (PostgreSQL) -- Transactional Data

```sql
-- PostgreSQL handles ACID-critical operations, HUD compliance data,
-- user authentication, and data that needs relational integrity

-- User management and authentication (not in graph)
CREATE TABLE auth.app_user (
    user_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255) NOT NULL,
    organization_id     UUID NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret_encrypted BYTEA,
    last_login_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE auth.role (
    role_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name           VARCHAR(100) NOT NULL UNIQUE,
    permissions         JSONB NOT NULL DEFAULT '[]'
);

CREATE TABLE auth.user_role (
    user_id             UUID NOT NULL REFERENCES auth.app_user(user_id),
    role_id             UUID NOT NULL REFERENCES auth.role(role_id),
    project_id          UUID,
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (user_id, role_id, project_id)
);

-- Encrypted PII storage (SSN, sensitive notes)
-- The graph stores hashes/references; PostgreSQL stores encrypted values
CREATE TABLE secure.client_pii (
    personal_id         UUID PRIMARY KEY,
    ssn_encrypted       BYTEA,
    ssn_data_quality    SMALLINT,
    sensitive_notes     BYTEA,  -- Encrypted case notes flagged as confidential
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- HUD HMIS CSV export staging tables
-- These mirror the exact HUD CSV structure for compliance export
CREATE TABLE hmis_export.client_csv (
    personal_id         UUID NOT NULL,
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    name_suffix         VARCHAR(25),
    name_data_quality   SMALLINT,
    ssn                 VARCHAR(9),  -- Decrypted only during export, never stored in clear
    ssn_data_quality    SMALLINT,
    date_of_birth       DATE,
    dob_data_quality    SMALLINT,
    race                SMALLINT,
    ethnicity           SMALLINT,
    gender              SMALLINT,
    veteran_status      SMALLINT,
    export_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE hmis_export.enrollment_csv (
    enrollment_id       UUID NOT NULL,
    personal_id         UUID NOT NULL,
    project_id          UUID NOT NULL,
    entry_date          DATE NOT NULL,
    household_id        UUID,
    relationship_to_hoh SMALLINT,
    enrollment_coc      VARCHAR(6),
    living_situation    SMALLINT,
    length_of_stay      SMALLINT,
    los_under_threshold SMALLINT,
    previous_street_essh SMALLINT,
    date_to_street_essh DATE,
    times_homeless_past_three_years SMALLINT,
    months_homeless_past_three_years SMALLINT,
    disabling_condition SMALLINT,
    move_in_date        DATE,
    export_id           UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Additional HUD CSV staging tables: Exit.csv, IncomeBenefits.csv, 
-- HealthAndDV.csv, EmploymentEducation.csv, Disabilities.csv, etc.
-- (follow same pattern as above)

CREATE TABLE hmis_export.export_metadata (
    export_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    export_start_date   DATE NOT NULL,
    export_end_date     DATE NOT NULL,
    export_type         VARCHAR(50) NOT NULL,  -- 'full', 'incremental', 'apr', 'lsa', 'caper'
    coc_code            VARCHAR(6),
    project_ids         UUID[],
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    file_url            VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    completed_at        TIMESTAMPTZ
);

-- Audit log (relational for compliance)
CREATE TABLE audit.audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    event_time          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id             UUID,
    organization_id     UUID,
    action              VARCHAR(20) NOT NULL,
    resource_type       VARCHAR(100) NOT NULL,  -- 'Client', 'Enrollment', 'BedUnit', etc.
    resource_id         UUID,
    changes             JSONB NOT NULL DEFAULT '{}',
    ip_address          INET,
    session_id          VARCHAR(100)
);

CREATE INDEX idx_audit_time ON audit.audit_log(event_time);
CREATE INDEX idx_audit_user ON audit.audit_log(user_id);
CREATE INDEX idx_audit_resource ON audit.audit_log(resource_type, resource_id);

-- Form definitions and submissions (relational for transactional integrity)
CREATE TABLE forms.form_definition (
    form_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID,
    form_name           VARCHAR(200) NOT NULL,
    form_type           VARCHAR(50) NOT NULL,
    form_version        INTEGER NOT NULL DEFAULT 1,
    definition          JSONB NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE forms.form_submission (
    submission_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id             UUID NOT NULL REFERENCES forms.form_definition(form_id),
    personal_id         UUID,
    enrollment_id       UUID,
    responses           JSONB NOT NULL,
    submitted_by        UUID NOT NULL,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## Time-Series Store (TimescaleDB)

```sql
-- TimescaleDB extends PostgreSQL with hypertables for time-series data
-- Install: CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Bed occupancy time-series (the hottest data path)
CREATE TABLE timeseries.bed_occupancy_events (
    event_time          TIMESTAMPTZ NOT NULL,
    project_id          UUID NOT NULL,
    unit_id             UUID NOT NULL,
    event_type          VARCHAR(20) NOT NULL,  -- 'check_in', 'check_out', 'reserve', 'cancel'
    personal_id         UUID,
    method              VARCHAR(20),  -- 'manual', 'barcode', 'qr_code'
    user_id             UUID
);

-- Convert to hypertable for time-series optimisation
SELECT create_hypertable('timeseries.bed_occupancy_events', 'event_time');

-- Continuous aggregate: occupancy metrics per project per hour
CREATE MATERIALIZED VIEW timeseries.hourly_occupancy
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', event_time) AS bucket,
    project_id,
    COUNT(*) FILTER (WHERE event_type = 'check_in') AS check_ins,
    COUNT(*) FILTER (WHERE event_type = 'check_out') AS check_outs,
    COUNT(*) FILTER (WHERE event_type = 'reserve') AS reservations
FROM timeseries.bed_occupancy_events
GROUP BY bucket, project_id;

-- Continuous aggregate: daily occupancy summary
CREATE MATERIALIZED VIEW timeseries.daily_occupancy
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', event_time) AS bucket,
    project_id,
    COUNT(*) FILTER (WHERE event_type = 'check_in') AS total_check_ins,
    COUNT(*) FILTER (WHERE event_type = 'check_out') AS total_check_outs,
    COUNT(DISTINCT personal_id) FILTER (WHERE event_type = 'check_in') AS unique_clients
FROM timeseries.bed_occupancy_events
GROUP BY bucket, project_id;

-- Client service utilization time-series
CREATE TABLE timeseries.service_events (
    event_time          TIMESTAMPTZ NOT NULL,
    project_id          UUID NOT NULL,
    personal_id         UUID NOT NULL,
    service_type        SMALLINT NOT NULL,
    service_category    VARCHAR(50) NOT NULL,  -- 'meal', 'shower', 'case_management', etc.
    quantity            INTEGER DEFAULT 1,
    staff_id            UUID
);

SELECT create_hypertable('timeseries.service_events', 'event_time');

-- Continuous aggregate: service utilization by category per day
CREATE MATERIALIZED VIEW timeseries.daily_services
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', event_time) AS bucket,
    project_id,
    service_category,
    COUNT(*) AS service_count,
    COUNT(DISTINCT personal_id) AS unique_clients
FROM timeseries.service_events
GROUP BY bucket, project_id, service_category;

-- Client journey milestones (longitudinal tracking)
CREATE TABLE timeseries.client_milestones (
    event_time          TIMESTAMPTZ NOT NULL,
    personal_id         UUID NOT NULL,
    milestone_type      VARCHAR(50) NOT NULL,
    -- 'first_contact', 'intake', 'assessment', 'prioritized', 'referred',
    -- 'accepted', 'housed', 'stabilized', 'exited', 'returned'
    project_id          UUID,
    organization_id     UUID,
    score               DECIMAL(8,2),  -- Assessment score at milestone
    details             JSONB DEFAULT '{}'
);

SELECT create_hypertable('timeseries.client_milestones', 'event_time');

-- Continuous aggregate: housing outcomes by month
CREATE MATERIALIZED VIEW timeseries.monthly_outcomes
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 month', event_time) AS bucket,
    organization_id,
    milestone_type,
    COUNT(*) AS event_count,
    COUNT(DISTINCT personal_id) AS unique_clients,
    AVG(score) AS avg_score
FROM timeseries.client_milestones
GROUP BY bucket, organization_id, milestone_type;

-- Real-time capacity metrics (updated every check-in/check-out)
CREATE TABLE timeseries.capacity_snapshots (
    snapshot_time       TIMESTAMPTZ NOT NULL,
    coc_code            VARCHAR(6) NOT NULL,
    project_id          UUID NOT NULL,
    total_beds          INTEGER NOT NULL,
    occupied_beds       INTEGER NOT NULL,
    available_beds      INTEGER NOT NULL,
    reserved_beds       INTEGER NOT NULL DEFAULT 0,
    turnaway_count      INTEGER NOT NULL DEFAULT 0,
    occupancy_rate      DECIMAL(5,2)
);

SELECT create_hypertable('timeseries.capacity_snapshots', 'snapshot_time');

-- Continuous aggregate: capacity trends by day
CREATE MATERIALIZED VIEW timeseries.daily_capacity
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', snapshot_time) AS bucket,
    coc_code,
    project_id,
    AVG(occupancy_rate) AS avg_occupancy_rate,
    MAX(occupancy_rate) AS peak_occupancy_rate,
    MIN(available_beds) AS min_available_beds,
    SUM(turnaway_count) AS total_turnaways
FROM timeseries.capacity_snapshots
GROUP BY bucket, coc_code, project_id;

-- Retention policies -- compress and eventually drop old data
SELECT add_compression_policy('timeseries.bed_occupancy_events', INTERVAL '30 days');
SELECT add_compression_policy('timeseries.service_events', INTERVAL '30 days');
SELECT add_compression_policy('timeseries.capacity_snapshots', INTERVAL '7 days');

-- Keep raw data for 2 years, aggregates forever
SELECT add_retention_policy('timeseries.bed_occupancy_events', INTERVAL '2 years');
SELECT add_retention_policy('timeseries.service_events', INTERVAL '2 years');
SELECT add_retention_policy('timeseries.capacity_snapshots', INTERVAL '2 years');
```

---

## Data Synchronisation Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      Application Layer                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐  │
│  │  Write API   │  │  Read API    │  │  Real-Time API       │  │
│  │  (Commands)  │  │  (Queries)   │  │  (WebSocket/SSE)     │  │
│  └──────┬───────┘  └──────┬───────┘  └──────────┬───────────┘  │
│         │                 │                      │              │
│  ┌──────▼───────┐  ┌──────▼───────┐  ┌──────────▼───────────┐  │
│  │  Sync Layer  │  │  Query       │  │  Event Publisher     │  │
│  │  (Debezium/  │  │  Router      │  │  (Change Data        │  │
│  │   CDC)       │  │              │  │   Capture)           │  │
│  └──┬───┬───┬───┘  └──┬───┬──────┘  └──────────────────────┘  │
│     │   │   │         │   │                                    │
└─────┼───┼───┼─────────┼───┼────────────────────────────────────┘
      │   │   │         │   │
  ┌───▼─┐ │ ┌─▼──────┐ │ ┌─▼────────────┐
  │Neo4j│ │ │Timescale│ │ │PostgreSQL    │
  │     │ │ │DB      │ │ │(Transactional│
  │Graph│ │ │        │ │ │ + Auth +     │
  │Store│ │ │Time-   │ │ │ Audit +      │
  │     │ │ │Series  │ │ │ HUD Export)  │
  └─────┘ │ └────────┘ │ └──────────────┘
          │             │
     Writes go to       Reads routed to
     all stores via     best store for
     sync layer         query pattern
```

### Synchronisation Strategy

The application layer writes to PostgreSQL as the primary source of truth. Changes are captured and propagated:

1. **PostgreSQL -> Neo4j**: Change Data Capture (Debezium or custom triggers) detects writes to PostgreSQL and updates the corresponding graph nodes and relationships. For example, a new enrollment row in PostgreSQL triggers creation of an `(:Enrollment)` node and `(:Client)-[:ENROLLED_IN]->(:Enrollment)` relationship in Neo4j.

2. **PostgreSQL -> TimescaleDB**: Bed check-in/check-out events, service records, and milestone events are written to both PostgreSQL (for transactional integrity) and TimescaleDB (for time-series optimisation) within the same transaction (since TimescaleDB is a PostgreSQL extension, they share the same database instance).

3. **Neo4j -> PostgreSQL**: Graph-computed results (deduplication matches, referral chain analysis) are written back to PostgreSQL for compliance reporting and HMIS CSV export.

```sql
-- Example: PostgreSQL trigger to sync enrollment creation to Neo4j
-- (In practice, use Debezium CDC or application-level dual-write)
CREATE OR REPLACE FUNCTION sync_enrollment_to_graph()
RETURNS TRIGGER AS $$
BEGIN
    -- Queue the sync event (processed by background worker)
    INSERT INTO sync.outbox (
        event_type, entity_type, entity_id, payload, created_at
    ) VALUES (
        'CREATE', 'Enrollment', NEW.enrollment_id,
        row_to_json(NEW)::JSONB, NOW()
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Outbox table for reliable event delivery to Neo4j
CREATE TABLE sync.outbox (
    outbox_id           BIGSERIAL PRIMARY KEY,
    event_type          VARCHAR(20) NOT NULL,
    entity_type         VARCHAR(100) NOT NULL,
    entity_id           UUID NOT NULL,
    payload             JSONB NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    processed_at        TIMESTAMPTZ,
    error_message       TEXT,
    retry_count         INTEGER DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_pending ON sync.outbox(status, created_at)
    WHERE status = 'pending';
```

---

## Pros and Cons

### Pros

1. **Relationship queries are transformatively faster.** The coordinated entry matching query ("find clients on the priority list whose service history matches housing success patterns, considering household composition, agency network, and assessment trajectory") executes in milliseconds on Neo4j versus seconds or minutes with relational JOINs across 6-8 tables. For a CoC with 10,000+ active clients, this difference is operationally significant.

2. **Client deduplication becomes a graph problem.** Instead of comparing every client against every other client (O(n^2) in a relational database), the graph model enables probabilistic matching through alias chains, shared household memberships, and agency overlap. The `(:ClientAlias)-[:ALIAS_OF]->(:Client)` relationship pattern with confidence scores makes deduplication a graph traversal problem solvable in linear time.

3. **Service pathway analysis is native.** Understanding which service sequences lead to successful housing outcomes requires traversing variable-length paths through enrollments, services, and placements. This is natural in Cypher (`MATCH path = (c:Client)-[*]->(:HousingPlacement)`) but requires recursive CTEs or complex self-joins in SQL.

4. **Network visualisation for case conferencing.** During by-name-list case conferences, case workers need to see a client's complete network: household members, all agencies involved, referral chains, service history, assessment trajectory. The graph model produces this visualisation directly; the relational model requires assembling it from multiple queries.

5. **Time-series queries are orders of magnitude faster.** TimescaleDB's continuous aggregates pre-compute hourly/daily/monthly occupancy, service utilisation, and outcome metrics. Queries like "show me the occupancy trend for all shelters in CA-500 over the last 12 months" return instantly from aggregates rather than scanning millions of occupancy records.

6. **Each store is optimised for its access pattern.** PostgreSQL handles ACID transactions and compliance data; Neo4j handles relationship traversals and pattern matching; TimescaleDB handles temporal queries and trend analysis. No single database is forced to handle access patterns it was not designed for.

7. **HUD compliance through PostgreSQL export.** The relational store maintains HUD CSV-compatible structures, ensuring compliance exports are straightforward despite the polyglot architecture.

### Cons

1. **Operational complexity is the highest of all four models.** Three database systems must be deployed, monitored, backed up, scaled, and secured independently. The team needs expertise in PostgreSQL, Neo4j (Cypher query language, graph modeling), and TimescaleDB. For a small shelter IT team, this is a significant burden.

2. **Data synchronisation is the critical failure point.** If the sync layer fails or lags, the graph and time-series stores become stale. Inconsistencies between stores can lead to referral decisions based on outdated priority scores or bed availability based on stale occupancy data. The outbox pattern mitigates this but adds complexity.

3. **Transactional consistency across stores is impossible.** There is no distributed transaction across PostgreSQL, Neo4j, and TimescaleDB. A referral creation that succeeds in PostgreSQL but fails to propagate to Neo4j creates an inconsistency. Compensating transactions and eventual consistency patterns are required.

4. **Neo4j licensing and cost.** Neo4j Community Edition has limitations (single database, no role-based access, no clustering). Neo4j Enterprise or Aura (cloud) is required for production multi-tenant deployments, adding significant licensing cost. Alternatives like Apache AGE (PostgreSQL extension for graph queries) could reduce this cost but sacrifice Neo4j's optimised graph storage engine.

5. **Testing complexity.** Integration tests must verify data consistency across three stores. Unit tests for graph queries require a Neo4j test instance. The test infrastructure is substantially more complex than a single-database model.

6. **Offline sync is harder.** The mobile app must generate changes that are eventually applied to three different stores. The sync layer must handle ordering, conflict resolution, and partial failures across stores.

7. **HMIS CSV import requires multi-store writes.** Importing HMIS CSV files (the standard migration path) requires parsing CSV rows, writing to PostgreSQL, generating graph mutations for Neo4j, and inserting time-series events into TimescaleDB. The import pipeline is more complex than any single-database approach.

8. **Query routing complexity.** The application layer must know which store to query for each access pattern. A client profile page might query PostgreSQL for PII, Neo4j for the relationship network, and TimescaleDB for service utilisation trends -- three queries that must be composed in the application layer.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Graph database | Neo4j Aura (managed cloud) or Neo4j Community (self-hosted, single database) |
| Relational database | PostgreSQL 16+ with pgcrypto, pg_trgm |
| Time-series | TimescaleDB 2.x (PostgreSQL extension -- shares the PostgreSQL instance) |
| Graph alternative | Apache AGE (PostgreSQL extension) if Neo4j licensing is prohibitive |
| Data sync | Debezium for CDC from PostgreSQL; custom outbox processor for Neo4j |
| API layer | GraphQL with federated schema across all three stores |
| Caching | Redis for real-time bed availability (fed from TimescaleDB events) |
| Monitoring | Prometheus + Grafana for all three databases; custom sync lag dashboards |

---

## Migration and Scaling Considerations

### Phased Adoption Strategy

Given the complexity, a phased approach is recommended:

**Phase 1 (MVP)**: PostgreSQL + TimescaleDB only. TimescaleDB is a PostgreSQL extension, so it adds no operational overhead. Use it for bed occupancy time-series and dashboard metrics from day one. Store all data in PostgreSQL.

**Phase 2 (v1.1)**: Add Neo4j for the relationship graph. Initially populate it from PostgreSQL data. Use it for coordinated entry matching, deduplication, and case conference visualisations. PostgreSQL remains the source of truth; Neo4j is a read-optimised projection.

**Phase 3 (v2.0)**: Full polyglot with optimised query routing. Each query is routed to the optimal store. The graph becomes authoritative for relationship data; PostgreSQL for transactional data; TimescaleDB for temporal analytics.

### Scaling

1. **Neo4j**: Neo4j Aura scales automatically in the cloud. Self-hosted: read replicas for query distribution, sharding by CoC for very large deployments.

2. **PostgreSQL**: Standard scaling -- read replicas for reporting, connection pooling, table partitioning for large tables (audit logs, export staging).

3. **TimescaleDB**: Automatic chunk management handles partitioning. Compression reduces storage for historical data by 90%+. Continuous aggregates handle pre-computation. Multi-node TimescaleDB for very large deployments.

### Cost Comparison

| Component | Self-Hosted (per month) | Cloud Managed (per month) |
|-----------|------------------------|--------------------------|
| PostgreSQL + TimescaleDB | $200-500 (single server) | $300-800 (RDS + TimescaleDB Cloud) |
| Neo4j Community | $0 (bundled with app server) | N/A |
| Neo4j Aura Professional | N/A | $65-650 (depending on size) |
| Total (self-hosted) | $200-500 | - |
| Total (cloud) | - | $365-1,450 |

### Apache AGE Alternative

For organisations that cannot justify Neo4j licensing, Apache AGE provides graph query capabilities as a PostgreSQL extension:

```sql
-- Apache AGE: graph queries within PostgreSQL
-- Reduces the architecture to PostgreSQL (with AGE + TimescaleDB extensions)

-- Create a graph
SELECT create_graph('shelter_graph');

-- Create nodes
SELECT * FROM cypher('shelter_graph', $$
    CREATE (c:Client {personalId: 'uuid-here', firstName: 'John', lastName: 'Smith'})
    RETURN c
$$) AS (v agtype);

-- Create relationships
SELECT * FROM cypher('shelter_graph', $$
    MATCH (c:Client {personalId: 'client-uuid'}), (e:Enrollment {enrollmentId: 'enrollment-uuid'})
    CREATE (c)-[:ENROLLED_IN {entryDate: '2026-01-15'}]->(e)
    RETURN c, e
$$) AS (c agtype, e agtype);

-- Query: find client journey
SELECT * FROM cypher('shelter_graph', $$
    MATCH path = (c:Client {personalId: 'client-uuid'})-[*1..5]-(connected)
    RETURN path
$$) AS (path agtype);
```

Using Apache AGE reduces the architecture to a single PostgreSQL instance (with two extensions: AGE and TimescaleDB), dramatically simplifying operations while retaining graph query capabilities. The trade-off is that AGE's graph engine is less mature and less performant than Neo4j for complex traversals, but for the expected data volumes in shelter management (thousands to tens of thousands of clients per CoC), this is likely acceptable.
