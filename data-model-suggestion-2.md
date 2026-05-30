# Data Model Suggestion 2: Event-Sourced / CQRS Model

> Project: Shelter & Housing Management (449)
> Model Type: Event Sourcing with Command Query Responsibility Segregation (CQRS)
> Database: PostgreSQL (event store) + PostgreSQL/Redis (read models)
> Generated: 2026-05-25

---

## Overview

This model applies Event Sourcing and CQRS to the shelter/housing management domain. Every state change in the system -- a client intake, a bed check-in, a referral decision, an assessment score update -- is captured as an immutable event in an append-only event store. The current state of any entity is derived by replaying its event stream. Read-optimised projections (read models) are maintained asynchronously from the event stream, each tailored to a specific query pattern: real-time bed dashboards, HUD compliance reports, coordinated entry prioritisation lists, and case management timelines.

This architecture is particularly compelling for shelter/housing management because:

1. **Audit trail is inherent.** Every action is permanently recorded, satisfying HUD privacy and security audit requirements and HIPAA compliance without a separate audit log.
2. **Temporal queries are natural.** "What was this client's living situation on October 1st?" is answered by replaying events to that point, which is essential for point-in-time HUD reporting (APR, LSA).
3. **Multi-agency coordination.** Events can be selectively shared between agencies via event streams, enabling federated data sharing with explicit consent controls at the event level.
4. **Offline-first sync.** Each mobile device generates events locally and syncs them when connectivity returns; the event store handles out-of-order event processing with vector clocks or lamport timestamps.

---

## Event Store Schema

### Core Event Store (PostgreSQL)

```sql
-- The event store is the single source of truth
CREATE TABLE event_store.events (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,          -- Aggregate root ID (e.g., client ID, enrollment ID)
    stream_type         VARCHAR(100) NOT NULL,   -- Aggregate type (e.g., 'Client', 'Enrollment', 'BedUnit')
    event_type          VARCHAR(200) NOT NULL,   -- e.g., 'ClientIntakeCompleted', 'BedCheckedIn'
    event_version       INTEGER NOT NULL,        -- Sequence number within stream (for ordering)
    event_data          JSONB NOT NULL,           -- The event payload
    metadata            JSONB NOT NULL DEFAULT '{}', -- Correlation IDs, causation IDs, user context
    event_timestamp     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by_user_id  UUID,                    -- User who caused the event
    organization_id     UUID NOT NULL,            -- Tenant isolation
    coc_id              UUID,                     -- CoC context for cross-agency events
    schema_version      SMALLINT NOT NULL DEFAULT 1, -- For event schema evolution
    is_encrypted        BOOLEAN NOT NULL DEFAULT FALSE, -- PII events are encrypted at rest
    
    -- Optimistic concurrency: no two events with same stream_id + version
    UNIQUE (stream_id, event_version)
);

-- Primary access pattern: replay events for a specific aggregate
CREATE INDEX idx_events_stream ON event_store.events(stream_id, event_version);

-- Access pattern: all events of a type across all aggregates (for projections)
CREATE INDEX idx_events_type ON event_store.events(event_type, event_timestamp);

-- Access pattern: all events for an organization (tenant scoping)
CREATE INDEX idx_events_org ON event_store.events(organization_id, event_timestamp);

-- Access pattern: global ordering for catch-up subscriptions
CREATE INDEX idx_events_timestamp ON event_store.events(event_timestamp);

-- Access pattern: events within a CoC for coordinated entry projections
CREATE INDEX idx_events_coc ON event_store.events(coc_id, event_type, event_timestamp);

-- Partition by month for performance at scale
-- CREATE TABLE event_store.events_2026_01 PARTITION OF event_store.events
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

-- Snapshots for aggregates with long event streams
CREATE TABLE event_store.snapshots (
    snapshot_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(100) NOT NULL,
    snapshot_version    INTEGER NOT NULL,        -- Event version at snapshot time
    snapshot_data       JSONB NOT NULL,           -- Serialised aggregate state
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (stream_id, snapshot_version)
);

CREATE INDEX idx_snapshot_stream ON event_store.snapshots(stream_id, snapshot_version DESC);

-- Subscription checkpoints for projection processors
CREATE TABLE event_store.subscription_checkpoints (
    subscription_id     VARCHAR(100) PRIMARY KEY,
    last_event_id       UUID,
    last_event_timestamp TIMESTAMPTZ,
    last_processed_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Dead letter queue for events that failed projection processing
CREATE TABLE event_store.dead_letter_events (
    dead_letter_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id            UUID NOT NULL REFERENCES event_store.events(event_id),
    subscription_id     VARCHAR(100) NOT NULL,
    error_message       TEXT NOT NULL,
    retry_count         INTEGER NOT NULL DEFAULT 0,
    max_retries         INTEGER NOT NULL DEFAULT 5,
    next_retry_at       TIMESTAMPTZ,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Command Handling Layer

Commands are validated and produce events. Below are the aggregate roots and their commands:

```sql
-- Command log (optional, for debugging and replay)
CREATE TABLE event_store.command_log (
    command_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    command_type        VARCHAR(200) NOT NULL,
    command_data        JSONB NOT NULL,
    target_stream_id    UUID,
    target_stream_type  VARCHAR(100),
    user_id             UUID,
    organization_id     UUID NOT NULL,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'accepted', 'rejected', 'failed'
    rejection_reason    TEXT,
    produced_event_ids  UUID[],
    received_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    processed_at        TIMESTAMPTZ
);

CREATE INDEX idx_command_type ON event_store.command_log(command_type, received_at);
CREATE INDEX idx_command_stream ON event_store.command_log(target_stream_id);
```

---

## Event Type Catalogue

### Client Aggregate Events

```
ClientRegistered
  { personalId, firstName, lastName, middleName, nameSuffix, nameDataQuality,
    dateOfBirth, dobDataQuality, veteranStatus, races[], genders[],
    sourceOrganizationId }

ClientDemographicsUpdated
  { personalId, changedFields: { firstName?, lastName?, dateOfBirth?, ... } }

ClientSSNRecorded
  { personalId, ssnEncrypted, ssnDataQuality }

ClientRaceUpdated
  { personalId, races[] }

ClientGenderUpdated
  { personalId, genders[] }

ClientConsentGranted
  { personalId, agreementId, consentDate, expirationDate, witnessName }

ClientConsentRevoked
  { personalId, agreementId, revocationDate, reason }

ClientMerged
  { survivingId, mergedId, mergeMethod, mergeConfidence }

ClientMergeUndone
  { survivingId, mergedId, reason }

ClientAliasAdded
  { personalId, aliasFirstName, aliasLastName, aliasDob, source }

ClientPhotoUpdated
  { personalId, photoUrl }
```

### Enrollment Aggregate Events

```
EnrollmentCreated
  { enrollmentId, personalId, projectId, householdId, relationshipToHoH,
    entryDate, enrollmentCoC, livingSituation, lengthOfStay,
    previousStreetESSH, dateToStreetESSH, timesHomelessPastThreeYears,
    monthsHomelessPastThreeYears, disablingCondition }

EnrollmentExited
  { enrollmentId, exitDate, destination, reason }

EnrollmentMoveInDateRecorded
  { enrollmentId, moveInDate }

EnrollmentUpdated
  { enrollmentId, changedFields: { ... } }

IncomeBenefitsRecorded
  { enrollmentId, personalId, dataCollectionStage, informationDate,
    incomeFromAnySource, totalMonthlyIncome, earnedIncome, ssi, ssdi, ...
    benefitsFromAnySource, snap, wic, ...
    insuranceFromAnySource, medicaid, medicare, ... }

HealthAndDVRecorded
  { enrollmentId, personalId, dataCollectionStage, informationDate,
    domesticViolenceSurvivor, whenDVOccurred, currentlyFleeing,
    generalHealthStatus, mentalHealthStatus, pregnancyStatus, dueDate }

EmploymentEducationRecorded
  { enrollmentId, personalId, dataCollectionStage, informationDate,
    lastGradeCompleted, schoolStatus, employed, employmentType }

DisabilityRecorded
  { enrollmentId, personalId, dataCollectionStage, informationDate,
    disabilityType, disabilityResponse, indefiniteAndImpairs }

CurrentLivingSituationRecorded
  { enrollmentId, personalId, informationDate, currentLivingSituation,
    verifiedBy, leaveSituation14Days }

ServiceProvided
  { enrollmentId, personalId, dateProvided, recordType, typeProvided,
    subTypeProvided, faAmount, referralOutcome }

CaseNoteAdded
  { enrollmentId, personalId, noteDate, noteType, title, noteText,
    isConfidential, createdBy }

ServicePlanCreated
  { planId, enrollmentId, personalId, planType, planStartDate, createdBy }

ServicePlanGoalAdded
  { goalId, planId, goalDescription, targetDate }

ServicePlanGoalCompleted
  { goalId, completedDate, outcome }
```

### Assessment Aggregate Events

```
AssessmentStarted
  { assessmentId, personalId, enrollmentId, instrumentId, instrumentName,
    instrumentVersion, assessmentDate, assessmentType, assessmentLevel }

AssessmentResponseRecorded
  { assessmentId, questionId, questionNumber, responseValue, score }

AssessmentCompleted
  { assessmentId, totalScore, prioritizationStatus, conductedBy }

AssessmentVoided
  { assessmentId, reason, voidedBy }
```

### Bed Management Aggregate Events

```
BedUnitCreated
  { unitId, projectId, unitName, floor, building, unitType, maxOccupants,
    isAccessible, isPetFriendly, genderRestriction, ageRestriction }

BedUnitUpdated
  { unitId, changedFields: { ... } }

BedUnitDecommissioned
  { unitId, reason, decommissionDate }

BedCheckedIn
  { occupancyId, unitId, personalId, enrollmentId, checkInTime,
    checkInMethod }

BedCheckedOut
  { occupancyId, unitId, personalId, checkOutTime, checkOutMethod }

BedReserved
  { reservationId, unitId, projectId, personalId, reservationDate,
    expectedArrival, expirationTime, priorityScore }

BedReservationCancelled
  { reservationId, reason }

BedReservationExpired
  { reservationId }

BedReservationFulfilled
  { reservationId, occupancyId }

NightlyCensusRecorded
  { projectId, censusDate, totalBeds, occupiedBeds, availableBeds,
    reservedBeds, maintenanceBeds, turnawayCount }
```

### Coordinated Entry Aggregate Events

```
PrioritizationListCreated
  { listId, cocId, listName, listType, scoringCriteria }

ClientAddedToPrioritizationList
  { entryId, listId, personalId, enrollmentId, assessmentId,
    priorityScore, dateAdded }

ClientPriorityScoreUpdated
  { entryId, listId, personalId, oldScore, newScore, reason }

ClientRemovedFromPrioritizationList
  { entryId, listId, personalId, dateRemoved, removalReason }

ReferralCreated
  { referralId, personalId, enrollmentId, referringOrgId, receivingOrgId,
    referralType, referralDate, targetProjectId, housingUnitId, priorityScore }

ReferralAccepted
  { referralId, acceptedBy, acceptedDate }

ReferralDeclined
  { referralId, declinedBy, declineReason, declinedDate }

ReferralCompleted
  { referralId, outcome, outcomeDate }

ReferralExpired
  { referralId, expirationDate }

HousingPlacementCreated
  { placementId, personalId, enrollmentId, referralId, housingUnitId,
    projectId, placementDate }

HousingMoveInRecorded
  { placementId, moveInDate, leaseStartDate, leaseEndDate, monthlyRent,
    clientRentPortion, subsidyType }

HousingStabilityUpdated
  { placementId, newStatus, reason }

ReturnToHomelessnessRecorded
  { placementId, returnDate, reason }

CaseConferenceHeld
  { conferenceId, cocId, conferenceDate, conferenceType, facilitatorId }

ClientDiscussedAtConference
  { conferenceId, personalId, discussionNotes, actionItems, outcome }
```

### Organisation and Project Events

```
OrganizationRegistered
  { organizationId, cocId, organizationName, victimServiceProvider }

ProjectCreated
  { projectId, organizationId, projectName, projectType, operatingStartDate,
    trackingMethod, housingType }

ProjectUpdated
  { projectId, changedFields: { ... } }

ProjectClosed
  { projectId, operatingEndDate, reason }

FunderAssigned
  { funderId, projectId, funderType, grantId, startDate }

InventoryUpdated
  { inventoryId, projectId, householdType, bedType, unitInventory,
    bedInventory, inventoryStartDate }

DataSharingAgreementCreated
  { agreementId, agreementName, effectiveDate, participatingOrgs[] }

DataSharingAgreementExpired
  { agreementId, expirationDate }
```

---

## Read Model Projections

Each projection is a separate database table (or Redis data structure) optimised for specific query patterns.

### Projection 1: Client Profile (Current State)

```sql
-- Materialised from Client aggregate events
CREATE TABLE read_models.client_profile (
    personal_id         UUID PRIMARY KEY,
    first_name          VARCHAR(100),
    last_name           VARCHAR(100),
    middle_name         VARCHAR(100),
    name_suffix         VARCHAR(25),
    name_data_quality   SMALLINT,
    date_of_birth       DATE,
    dob_data_quality    SMALLINT,
    veteran_status      SMALLINT,
    races               SMALLINT[],         -- Array of race codes
    genders             SMALLINT[],         -- Array of gender codes
    aliases             JSONB DEFAULT '[]', -- [{firstName, lastName, dob}]
    source_organization_id UUID,
    has_active_consent  BOOLEAN DEFAULT FALSE,
    consent_org_ids     UUID[] DEFAULT '{}',
    photo_url           VARCHAR(500),
    preferred_language  VARCHAR(50),
    merge_status        VARCHAR(20) DEFAULT 'active', -- 'active', 'merged'
    merged_into_id      UUID,
    last_event_version  INTEGER NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_cp_name ON read_models.client_profile(last_name, first_name);
CREATE INDEX idx_cp_dob ON read_models.client_profile(date_of_birth);
CREATE INDEX idx_cp_source ON read_models.client_profile(source_organization_id);
```

### Projection 2: Active Enrollment View

```sql
-- Materialised from Enrollment aggregate events
CREATE TABLE read_models.active_enrollment (
    enrollment_id       UUID PRIMARY KEY,
    personal_id         UUID NOT NULL,
    project_id          UUID NOT NULL,
    project_name        VARCHAR(255),
    project_type        SMALLINT,
    organization_id     UUID,
    organization_name   VARCHAR(255),
    household_id        UUID,
    relationship_to_hoh SMALLINT,
    entry_date          DATE NOT NULL,
    exit_date           DATE,
    destination         SMALLINT,
    move_in_date        DATE,
    enrollment_coc      VARCHAR(6),
    living_situation    SMALLINT,
    disabling_condition SMALLINT,
    length_of_stay_days INTEGER,
    -- Denormalised latest income
    latest_total_monthly_income DECIMAL(10,2),
    latest_income_date  DATE,
    -- Denormalised latest assessment
    latest_assessment_score DECIMAL(6,2),
    latest_assessment_date DATE,
    latest_assessment_instrument VARCHAR(200),
    -- Service counts
    total_services_count INTEGER DEFAULT 0,
    total_case_notes_count INTEGER DEFAULT 0,
    -- Status
    is_active           BOOLEAN GENERATED ALWAYS AS (exit_date IS NULL) STORED,
    last_event_version  INTEGER NOT NULL,
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_ae_client ON read_models.active_enrollment(personal_id);
CREATE INDEX idx_ae_project ON read_models.active_enrollment(project_id) WHERE exit_date IS NULL;
CREATE INDEX idx_ae_active ON read_models.active_enrollment(is_active, project_id);
```

### Projection 3: Real-Time Bed Availability

```sql
-- Materialised from BedUnit events; refreshed in near-real-time
-- This is the hot path -- consider Redis for sub-second updates
CREATE TABLE read_models.bed_availability (
    unit_id             UUID PRIMARY KEY,
    project_id          UUID NOT NULL,
    project_name        VARCHAR(255),
    organization_id     UUID,
    organization_name   VARCHAR(255),
    coc_code            VARCHAR(6),
    unit_name           VARCHAR(100),
    floor               VARCHAR(20),
    building            VARCHAR(100),
    unit_type           VARCHAR(50),
    max_occupants       INTEGER,
    current_occupants   INTEGER DEFAULT 0,
    is_accessible       BOOLEAN,
    is_pet_friendly     BOOLEAN,
    gender_restriction  VARCHAR(20),
    age_restriction     VARCHAR(20),
    has_medical_support BOOLEAN,
    status              VARCHAR(20) NOT NULL,
    current_occupant_id UUID,
    current_occupant_name VARCHAR(200),
    check_in_time       TIMESTAMPTZ,
    reservation_id      UUID,
    reserved_for_id     UUID,
    reserved_until      TIMESTAMPTZ,
    qr_code             VARCHAR(100),
    barcode             VARCHAR(100),
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_ba_project_status ON read_models.bed_availability(project_id, status);
CREATE INDEX idx_ba_coc ON read_models.bed_availability(coc_code, status);
CREATE INDEX idx_ba_type ON read_models.bed_availability(unit_type, gender_restriction, status);
```

### Projection 4: Coordinated Entry Queue

```sql
-- Community-wide prioritisation queue, materialised from CE events
CREATE TABLE read_models.ce_priority_queue (
    entry_id            UUID PRIMARY KEY,
    list_id             UUID NOT NULL,
    list_name           VARCHAR(200),
    list_type           VARCHAR(50),
    coc_code            VARCHAR(6),
    personal_id         UUID NOT NULL,
    client_name         VARCHAR(200),
    date_of_birth       DATE,
    veteran_status      SMALLINT,
    enrollment_id       UUID,
    assessment_id       UUID,
    assessment_instrument VARCHAR(200),
    assessment_date     DATE,
    priority_score      DECIMAL(8,2) NOT NULL,
    rank_position       INTEGER,
    date_added          DATE NOT NULL,
    days_on_list        INTEGER GENERATED ALWAYS AS (CURRENT_DATE - date_added) STORED,
    status              VARCHAR(20) NOT NULL,
    -- Denormalised chronicity flags
    is_chronically_homeless BOOLEAN DEFAULT FALSE,
    disabling_condition BOOLEAN DEFAULT FALSE,
    -- Referral status
    active_referral_id  UUID,
    active_referral_status VARCHAR(20),
    active_referral_org VARCHAR(255),
    last_updated_at     TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_cepq_list ON read_models.ce_priority_queue(list_id, rank_position) WHERE status = 'active';
CREATE INDEX idx_cepq_client ON read_models.ce_priority_queue(personal_id);
CREATE INDEX idx_cepq_score ON read_models.ce_priority_queue(list_id, priority_score DESC) WHERE status = 'active';
```

### Projection 5: Client Timeline

```sql
-- Chronological view of all events for a client across all enrollments
CREATE TABLE read_models.client_timeline (
    timeline_entry_id   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL,
    event_timestamp     TIMESTAMPTZ NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    event_category      VARCHAR(50) NOT NULL,
    -- 'intake', 'enrollment', 'service', 'assessment', 'referral',
    -- 'housing', 'bed', 'case_note', 'consent', 'admin'
    title               VARCHAR(500) NOT NULL,
    description         TEXT,
    enrollment_id       UUID,
    project_name        VARCHAR(255),
    organization_name   VARCHAR(255),
    related_entity_id   UUID,  -- referral_id, assessment_id, etc.
    created_by_name     VARCHAR(200),
    is_confidential     BOOLEAN DEFAULT FALSE,
    metadata            JSONB DEFAULT '{}'
);

CREATE INDEX idx_ct_client ON read_models.client_timeline(personal_id, event_timestamp DESC);
CREATE INDEX idx_ct_category ON read_models.client_timeline(personal_id, event_category, event_timestamp DESC);
CREATE INDEX idx_ct_enrollment ON read_models.client_timeline(enrollment_id, event_timestamp DESC);
```

### Projection 6: HUD Reporting Aggregates

```sql
-- Pre-aggregated data for APR, LSA, CAPER reporting
CREATE TABLE read_models.hud_enrollment_snapshot (
    snapshot_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL,
    personal_id         UUID NOT NULL,
    project_id          UUID NOT NULL,
    project_type        SMALLINT,
    report_period_start DATE NOT NULL,
    report_period_end   DATE NOT NULL,
    -- Point-in-time state at report_period_end
    entry_date          DATE,
    exit_date           DATE,
    destination         SMALLINT,
    move_in_date        DATE,
    living_situation_at_entry SMALLINT,
    disabling_condition SMALLINT,
    -- Demographics at time of enrollment
    age_at_entry        INTEGER,
    races               SMALLINT[],
    genders             SMALLINT[],
    veteran_status      SMALLINT,
    relationship_to_hoh SMALLINT,
    household_id        UUID,
    -- Income at entry/exit
    income_at_entry     DECIMAL(10,2),
    income_at_exit      DECIMAL(10,2),
    income_change       DECIMAL(10,2),
    -- Disability flags
    has_physical_disability BOOLEAN,
    has_mental_health    BOOLEAN,
    has_substance_use    BOOLEAN,
    has_chronic_health   BOOLEAN,
    has_hiv_aids         BOOLEAN,
    has_developmental    BOOLEAN,
    -- Computed metrics
    length_of_stay_days INTEGER,
    chronically_homeless BOOLEAN,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_hud_snap_project ON read_models.hud_enrollment_snapshot(project_id, report_period_start, report_period_end);
CREATE INDEX idx_hud_snap_client ON read_models.hud_enrollment_snapshot(personal_id);
```

### Projection 7: Network-Wide Dashboard

```sql
-- CoC-level aggregated dashboard metrics
CREATE TABLE read_models.coc_dashboard (
    dashboard_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_code            VARCHAR(6) NOT NULL,
    snapshot_date       DATE NOT NULL,
    snapshot_time       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- Bed metrics
    total_beds          INTEGER NOT NULL DEFAULT 0,
    occupied_beds       INTEGER NOT NULL DEFAULT 0,
    available_beds      INTEGER NOT NULL DEFAULT 0,
    reserved_beds       INTEGER NOT NULL DEFAULT 0,
    occupancy_rate_pct  DECIMAL(5,2),
    -- Client metrics
    total_active_clients INTEGER NOT NULL DEFAULT 0,
    new_intakes_today   INTEGER NOT NULL DEFAULT 0,
    exits_today         INTEGER NOT NULL DEFAULT 0,
    -- CE metrics
    clients_on_priority_list INTEGER NOT NULL DEFAULT 0,
    pending_referrals   INTEGER NOT NULL DEFAULT 0,
    active_referrals    INTEGER NOT NULL DEFAULT 0,
    placements_this_month INTEGER NOT NULL DEFAULT 0,
    -- Outcome metrics
    returns_to_homelessness_this_month INTEGER NOT NULL DEFAULT 0,
    avg_length_of_stay_days DECIMAL(8,1),
    UNIQUE (coc_code, snapshot_date)
);

CREATE INDEX idx_coc_dash ON read_models.coc_dashboard(coc_code, snapshot_date DESC);
```

---

## Event Processing Architecture

```
                    ┌─────────────────┐
                    │   Mobile App    │
                    │  (Offline-first)│
                    └────────┬────────┘
                             │ Sync events on reconnect
                    ┌────────▼────────┐
                    │  API Gateway    │
                    │  (Commands)     │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Command Handler │
                    │  - Validate     │
                    │  - Load agg     │
                    │  - Apply event  │
                    │  - Persist      │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │  Event Store    │
                    │  (PostgreSQL)   │
                    │  append-only    │
                    └────────┬────────┘
                             │ LISTEN/NOTIFY or polling
              ┌──────────────┼──────────────────┐
              │              │                  │
     ┌────────▼──────┐ ┌────▼──────────┐ ┌─────▼────────┐
     │ Projection:   │ │ Projection:   │ │ Projection:  │
     │ Client Profile│ │ Bed Avail     │ │ HUD Reports  │
     │ (PostgreSQL)  │ │ (Redis+PG)    │ │ (PostgreSQL) │
     └───────────────┘ └───────────────┘ └──────────────┘
              │              │                  │
     ┌────────▼──────┐ ┌────▼──────────┐ ┌─────▼────────┐
     │ Query API:    │ │ Query API:    │ │ Query API:   │
     │ Client Search │ │ Bed Dashboard │ │ APR / CAPER  │
     └───────────────┘ └───────────────┘ └──────────────┘
```

### Event Publishing and Subscription

```sql
-- Use PostgreSQL LISTEN/NOTIFY for low-latency event distribution
CREATE OR REPLACE FUNCTION event_store.notify_new_event()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify(
        'new_event',
        json_build_object(
            'event_id', NEW.event_id,
            'stream_type', NEW.stream_type,
            'event_type', NEW.event_type,
            'organization_id', NEW.organization_id
        )::text
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_notify_event
    AFTER INSERT ON event_store.events
    FOR EACH ROW
    EXECUTE FUNCTION event_store.notify_new_event();
```

### Snapshot Strategy

For aggregates with many events (e.g., a client with 500+ enrollment events over years), snapshots prevent slow replay:

```sql
-- Snapshot every 100 events
-- When loading an aggregate:
-- 1. Load latest snapshot for stream_id
-- 2. Replay events with version > snapshot_version
-- 3. If (current_version - snapshot_version) > 100, create new snapshot

CREATE OR REPLACE FUNCTION event_store.should_create_snapshot(
    p_stream_id UUID,
    p_current_version INTEGER
) RETURNS BOOLEAN AS $$
DECLARE
    v_last_snapshot_version INTEGER;
BEGIN
    SELECT COALESCE(MAX(snapshot_version), 0)
    INTO v_last_snapshot_version
    FROM event_store.snapshots
    WHERE stream_id = p_stream_id;
    
    RETURN (p_current_version - v_last_snapshot_version) >= 100;
END;
$$ LANGUAGE plpgsql;
```

---

## Offline-First Event Synchronisation

The event-sourced model naturally supports offline-first mobile:

```sql
-- Mobile devices generate events with local timestamps and device IDs
-- On sync, events are merged into the central store with conflict detection

CREATE TABLE event_store.pending_sync_events (
    sync_event_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    device_id           UUID NOT NULL,
    device_event_id     UUID NOT NULL,     -- ID assigned on device
    device_timestamp    TIMESTAMPTZ NOT NULL, -- When created on device
    stream_id           UUID NOT NULL,
    stream_type         VARCHAR(100) NOT NULL,
    event_type          VARCHAR(200) NOT NULL,
    event_data          JSONB NOT NULL,
    metadata            JSONB NOT NULL DEFAULT '{}',
    user_id             UUID,
    organization_id     UUID NOT NULL,
    sync_status         VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'applied', 'conflict', 'rejected'
    conflict_resolution JSONB,
    synced_at           TIMESTAMPTZ,
    server_event_id     UUID REFERENCES event_store.events(event_id),
    UNIQUE (device_id, device_event_id)
);

CREATE INDEX idx_pending_device ON event_store.pending_sync_events(device_id, sync_status);
CREATE INDEX idx_pending_status ON event_store.pending_sync_events(sync_status);
```

---

## Pros and Cons

### Pros

1. **Complete audit trail by design.** Every state change is permanently recorded as an immutable event. HUD privacy and security audit requirements, HIPAA compliance logging, and internal accountability are satisfied without a separate audit subsystem. There is no way for data to be changed without a recorded event.

2. **Point-in-time reconstruction.** HUD reports like the APR and LSA require data "as of" a specific date. Event sourcing enables replaying events to any point in time, producing exact snapshots of client status, income, and housing situation as they were on that date. This eliminates the need for snapshot tables or temporal columns.

3. **Natural offline support.** Events generated on mobile devices during offline outreach are simply appended to the event stream on reconnect. The event-sourced model handles out-of-order events gracefully because each event is independent and idempotent when applied to an aggregate.

4. **Independent read model scaling.** Each projection (bed availability, CE queue, HUD reports) can be scaled independently. The bed availability projection can use Redis for sub-second updates, while the HUD reporting projection can run on a separate PostgreSQL replica optimised for analytical queries.

5. **Multi-agency event sharing.** Events can be selectively published to other agencies in the CoC through event stream subscriptions with consent-based filtering. An agency only receives events for clients who have granted data sharing consent, enforced at the event publication layer.

6. **Schema evolution without migration.** New event types can be introduced without altering existing tables. Event upcasting (transforming old event formats to new formats during replay) handles schema evolution. When HUD changes data standards, new event types are added while old events retain their original structure.

7. **Debugging and incident response.** When something goes wrong (e.g., a client was placed in the wrong housing unit), the complete event history shows exactly what happened, who did it, and when. Events can be compensated (not deleted) by publishing corrective events.

### Cons

1. **Implementation complexity.** Event sourcing and CQRS require significantly more infrastructure and developer expertise than a traditional CRUD application. The team needs to understand aggregate design, event versioning, projection management, and eventual consistency patterns.

2. **Eventual consistency for read models.** Projections are updated asynchronously, meaning a bed checked in via the command side may not appear in the bed availability dashboard for a brief period (typically milliseconds to seconds, but potentially longer under load). For real-time bed management, this latency must be managed carefully.

3. **Projection rebuild cost.** If a projection's schema changes or a bug is discovered in projection logic, the entire projection must be rebuilt by replaying all events. For a system with millions of events, this can take hours. Snapshots mitigate this but add complexity.

4. **Storage growth.** Events are never deleted, so the event store grows indefinitely. A large CoC with 50,000 clients and 5 years of history could accumulate hundreds of millions of events. Table partitioning and archival strategies are essential.

5. **Query complexity for ad-hoc reporting.** Ad-hoc queries against the event store (e.g., "how many clients were enrolled in emergency shelters in Q3 2025?") require replaying events or querying projections. If no projection exists for a specific query pattern, one must be built.

6. **Testing difficulty.** Testing event-sourced systems requires verifying both event production (commands produce correct events) and event consumption (projections produce correct read models). The testing surface area is roughly double that of a CRUD system.

7. **Operational overhead.** Monitoring subscription lag, managing dead letter queues, rebuilding projections, and maintaining snapshot schedules require operational investment that a small shelter IT team may struggle to sustain.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Event store | PostgreSQL 16+ with JSONB for event payloads; partitioned by month |
| Event bus | PostgreSQL LISTEN/NOTIFY for single-node; Apache Kafka or NATS for multi-node CoC deployments |
| Read model store | PostgreSQL for durable projections; Redis for hot-path projections (bed availability) |
| Aggregate framework | Marten (C#/.NET), Axon Framework (Java), or EventStoreDB client libraries |
| Serialisation | JSON with schema registry for event schema evolution |
| Snapshot store | PostgreSQL (same instance as event store) |
| Offline sync | CRDTs or vector clocks for conflict-free merging of mobile events |
| Monitoring | Track projection lag (last_processed - last_event), dead letter queue depth, event store partition sizes |

---

## Migration and Scaling Considerations

### Migrating from Existing HMIS

1. **Import as events.** When migrating from a traditional HMIS (e.g., importing HMIS CSV files), each record is converted to an event: a Client.csv row becomes a `ClientRegistered` event, an Enrollment.csv row becomes an `EnrollmentCreated` event, etc.

2. **Backdated timestamps.** Imported events use the original data dates (e.g., `EntryDate` becomes the event timestamp), preserving the temporal history.

3. **Bulk projection build.** After import, all projections are built from scratch by replaying the imported events. This is a one-time operation.

### Scaling to Multi-CoC Deployments

1. **Partitioned event store.** Events partitioned by `(organization_id, event_timestamp)` enable efficient tenant-scoped queries and independent data lifecycle management per agency.

2. **Event store per agency.** For full data sovereignty, each agency can run its own event store instance. Cross-agency coordination events (referrals, shared assessments) are published to a shared event bus (Kafka) with consent-based access control.

3. **Projection per consumer.** Each agency maintains its own projections from the events it has access to. CoC-level projections aggregate events from all participating agencies.

4. **Event archival.** Events older than the retention period (e.g., 7 years per HUD requirements) can be archived to cold storage (S3/GCS) while maintaining the projection data for continued querying. Archived events can be replayed if needed for audits or legal proceedings.

### Performance Targets

| Metric | Target |
|--------|--------|
| Event write latency | < 10ms (single event append) |
| Projection update latency | < 500ms (event to read model) |
| Bed availability query | < 50ms (from Redis projection) |
| Client profile query | < 100ms (from PostgreSQL projection) |
| HUD report generation | < 60s for full APR (from pre-aggregated projection) |
| Event replay throughput | > 10,000 events/second (for projection rebuilds) |
| Event store growth | ~500 bytes/event average; ~50GB/year for a large CoC |
