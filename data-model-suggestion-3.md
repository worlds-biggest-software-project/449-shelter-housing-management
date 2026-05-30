# Data Model Suggestion 3: Hybrid Relational + Document (PostgreSQL JSONB)

> Project: Shelter & Housing Management (449)
> Model Type: Hybrid Relational + Document Store (PostgreSQL with JSONB)
> Database: PostgreSQL 16+ with JSONB columns
> Generated: 2026-05-25

---

## Overview

This model takes a pragmatic hybrid approach: stable, frequently-queried, and compliance-critical data is stored in normalized relational tables, while flexible, variable, and rapidly-evolving data lives in JSONB columns within those same tables. This design directly addresses the central tension in HMIS software: HUD data standards change biennially, assessment instruments vary across CoCs, service types differ by programme, and custom fields are constantly requested by agencies -- yet the core relational structure (clients, enrollments, projects, beds) remains fundamentally stable.

The key insight for shelter/housing management is that different parts of the data have different rates of change:

- **Stable structure**: Client demographics (name, DOB, SSN), enrollment dates, bed inventory, organizational hierarchy, referral workflows. These benefit from relational constraints, foreign keys, and type-safe columns.
- **Variable structure**: Assessment questions and responses (instruments change, new ones are adopted, scoring algorithms evolve), HUD programme-specific data elements (differ by project type and funder), custom agency fields, service catalogues (each agency tracks different services), intake form configurations.
- **Evolving structure**: HUD data standard revisions (new fields, changed enumerations, new CSV files every two years), AI/ML feature vectors for deduplication, external data integrations.

By using JSONB for the variable and evolving data while maintaining relational integrity for the stable core, we get the compliance benefits of a relational database with the flexibility of a document store.

---

## Schema Definition

### Organization and Project Layer (Fully Relational)

```sql
-- These entities are stable and benefit from full normalization
CREATE TABLE core.continuum_of_care (
    coc_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_code            VARCHAR(6) NOT NULL UNIQUE,
    coc_name            VARCHAR(255) NOT NULL,
    geographic_area     TEXT,
    settings            JSONB NOT NULL DEFAULT '{}',  -- CoC-level configuration
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE core.organization (
    organization_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    organization_name   VARCHAR(255) NOT NULL,
    victim_service_provider BOOLEAN NOT NULL DEFAULT FALSE,
    -- Stable fields as columns
    federal_ein         VARCHAR(11),
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state               VARCHAR(2),
    zip_code            VARCHAR(10),
    phone               VARCHAR(20),
    email               VARCHAR(255),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    -- Variable/custom fields as JSONB
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"website": "...", "contactPerson": "...", "languages": ["en","es"],
    --        "operatingHours": {"mon": "8:00-17:00", ...}}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_org_coc ON core.organization(coc_id);
CREATE INDEX idx_org_custom ON core.organization USING GIN (custom_fields);

CREATE TABLE core.project (
    project_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES core.organization(organization_id),
    project_name        VARCHAR(255) NOT NULL,
    -- HUD Project Descriptor Data Elements as typed columns
    project_type        SMALLINT NOT NULL,
    tracking_method     SMALLINT,
    operating_start_date DATE NOT NULL,
    operating_end_date  DATE,
    continuum_project   BOOLEAN NOT NULL DEFAULT TRUE,
    residential_affiliation BOOLEAN,
    housing_type        SMALLINT,
    rrh_sub_type        SMALLINT,
    target_population   SMALLINT,
    -- HUD participation flags
    hmis_participating  BOOLEAN NOT NULL DEFAULT TRUE,
    -- Funder information as JSONB array (changes frequently, multiple funders)
    funders             JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"funderType": 1, "grantId": "XX-1234", "startDate": "2025-01-01", "endDate": null}]
    -- CoC mapping as JSONB array
    coc_mappings        JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"cocCode": "CA-500", "geocode": "123456", "geographyType": 1}]
    -- CE Participation details (new in FY2024, likely to evolve)
    ce_participation    JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"accessPoint": true, "crisisAssessment": true, "housingAssessment": false, ...}
    -- Programme-specific configuration
    programme_config    JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"maxStayDays": 90, "intakeFormId": "uuid", "requiredServices": [...]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_project_type CHECK (project_type IN (0,1,2,3,4,6,7,8,9,10,11,12,13,14))
);

CREATE INDEX idx_project_org ON core.project(organization_id);
CREATE INDEX idx_project_type ON core.project(project_type);
CREATE INDEX idx_project_funders ON core.project USING GIN (funders);
CREATE INDEX idx_project_ce ON core.project USING GIN (ce_participation);
```

### Client Layer (Relational Core + JSONB Extensions)

```sql
CREATE TABLE client.client (
    personal_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    -- HUD Universal Data Elements 3.01-3.07 as typed columns
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    name_suffix         VARCHAR(25),
    name_data_quality   SMALLINT NOT NULL DEFAULT 99,
    ssn_encrypted       BYTEA,
    ssn_last_four_hash  VARCHAR(64),
    ssn_data_quality    SMALLINT NOT NULL DEFAULT 99,
    date_of_birth       DATE,
    dob_data_quality    SMALLINT NOT NULL DEFAULT 99,
    veteran_status      SMALLINT,
    -- Multi-select HUD fields stored as JSONB arrays (cleaner than junction tables)
    races               JSONB NOT NULL DEFAULT '[]',    -- e.g., [1, 5] for AmIndAKNative + White
    genders             JSONB NOT NULL DEFAULT '[]',    -- e.g., [0] for Female
    -- Deduplication metadata
    aliases             JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"firstName": "John", "lastName": "Doe", "dob": "1985-03-15", "source": "OrgA"}]
    merge_history       JSONB NOT NULL DEFAULT '[]',
    -- e.g., [{"mergedId": "uuid", "method": "probabilistic", "confidence": 0.95, "date": "..."}]
    -- AI/ML features for probabilistic matching (evolving rapidly)
    matching_features   JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"nameEmbedding": [...], "dobDistance": 0, "locationProximity": 0.85,
    --        "phonetix": {"first": "JN", "last": "SM0"}}
    -- Photo and preferences
    photo_url           VARCHAR(500),
    preferred_language  VARCHAR(50),
    -- Contact information (variable structure)
    contact_info        JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"phone": "555-1234", "email": "...", "emergencyContact": {"name": "...", "phone": "..."}}
    -- Agency-specific custom fields
    custom_fields       JSONB NOT NULL DEFAULT '{}',
    -- Source tracking
    source_organization_id UUID REFERENCES core.organization(organization_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID,
    CONSTRAINT chk_name_dq CHECK (name_data_quality IN (1,2,8,9,99)),
    CONSTRAINT chk_ssn_dq CHECK (ssn_data_quality IN (1,2,8,9,99)),
    CONSTRAINT chk_dob_dq CHECK (dob_data_quality IN (1,2,8,9,99))
);

CREATE INDEX idx_client_name ON client.client(last_name, first_name);
CREATE INDEX idx_client_dob ON client.client(date_of_birth);
CREATE INDEX idx_client_ssn_hash ON client.client(ssn_last_four_hash);
CREATE INDEX idx_client_races ON client.client USING GIN (races);
CREATE INDEX idx_client_genders ON client.client USING GIN (genders);
CREATE INDEX idx_client_matching ON client.client USING GIN (matching_features);

-- Consent records (relational because they involve joins to agreements)
CREATE TABLE client.client_consent (
    consent_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    agreement_id        UUID NOT NULL,
    consent_given       BOOLEAN NOT NULL,
    consent_date        DATE NOT NULL,
    expiration_date     DATE,
    revocation_date     DATE,
    details             JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"witnessName": "...", "signatureUrl": "...", "scopeOfConsent": [...]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consent_client ON client.client_consent(personal_id);
```

### Enrollment Layer (Relational Structure + JSONB Data Elements)

```sql
CREATE TABLE enrollment.enrollment (
    enrollment_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    -- Relational keys for joins
    household_id        UUID NOT NULL,
    relationship_to_hoh SMALLINT NOT NULL,
    -- Core dates as typed columns (used in every query)
    entry_date          DATE NOT NULL,
    exit_date           DATE,
    move_in_date        DATE,
    -- Destination at exit (needed for HUD reporting)
    destination         SMALLINT,
    enrollment_coc      VARCHAR(6),
    disabling_condition SMALLINT,
    -- Living situation at entry (HUD 3.917) as JSONB
    -- This field group changes with HUD revisions and has many sub-fields
    living_situation_at_entry JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"livingSituation": 16, "lengthOfStay": 2, "losUnderThreshold": 1,
    --        "previousStreetESSH": 1, "dateToStreetESSH": "2024-06-15",
    --        "timesHomelessPastThreeYears": 4, "monthsHomelessPastThreeYears": 112}
    -- Custom intake data (varies by project and agency)
    intake_data         JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"referralSource": "211", "presentingIssues": ["housing", "employment"],
    --        "immediateSafetyNeeds": false, "petInfo": {"type": "dog", "name": "Rex"}}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID,
    CONSTRAINT chk_rel_hoh CHECK (relationship_to_hoh IN (1,2,3,4,5,99))
);

CREATE INDEX idx_enrollment_client ON enrollment.enrollment(personal_id);
CREATE INDEX idx_enrollment_project ON enrollment.enrollment(project_id);
CREATE INDEX idx_enrollment_household ON enrollment.enrollment(household_id);
CREATE INDEX idx_enrollment_entry ON enrollment.enrollment(entry_date);
CREATE INDEX idx_enrollment_active ON enrollment.enrollment(project_id)
    WHERE exit_date IS NULL;
CREATE INDEX idx_enrollment_living ON enrollment.enrollment USING GIN (living_situation_at_entry);

-- HUD Data Collection Records
-- Instead of separate tables for IncomeBenefits, HealthAndDV, EmploymentEducation, Disabilities,
-- we use a single table with a JSONB payload. The data_element_type distinguishes the content.
-- This dramatically simplifies HUD standard version changes.
CREATE TABLE enrollment.data_collection (
    data_collection_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    -- Relational fields used for all queries
    data_element_type   VARCHAR(50) NOT NULL,
    -- 'income_benefits', 'health_and_dv', 'employment_education',
    -- 'disability', 'current_living_situation'
    data_collection_stage SMALLINT NOT NULL,  -- 1=Entry, 2=Update, 3=Exit, 5=Annual
    information_date    DATE NOT NULL,
    hud_standard_version VARCHAR(10) NOT NULL DEFAULT '2024',
    -- The actual data as JSONB, schema varies by data_element_type
    data                JSONB NOT NULL,
    /*
    For data_element_type = 'income_benefits':
    {
        "incomeFromAnySource": 1,
        "totalMonthlyIncome": 1250.00,
        "earnedIncome": 800.00,
        "ssi": 0, "ssdi": 450.00, "tanf": 0,
        "benefitsFromAnySource": 1,
        "snap": 1, "wic": 0,
        "insuranceFromAnySource": 1,
        "medicaid": 1, "medicare": 0
    }
    
    For data_element_type = 'health_and_dv':
    {
        "domesticViolenceSurvivor": 0,
        "generalHealthStatus": 3,
        "dentalHealthStatus": 2,
        "mentalHealthStatus": 4,
        "pregnancyStatus": 0
    }
    
    For data_element_type = 'disability':
    {
        "disabilityType": 9,
        "disabilityResponse": 1,
        "indefiniteAndImpairs": 1
    }
    
    For data_element_type = 'employment_education':
    {
        "lastGradeCompleted": 12,
        "schoolStatus": 1,
        "employed": 1,
        "employmentType": 2,
        "notEmployedReason": null
    }
    
    For data_element_type = 'current_living_situation':
    {
        "currentLivingSituation": 16,
        "verifiedBy": "Case Manager",
        "leaveSituation14Days": 0,
        "subsequentResidence": 1,
        "resourcesToObtain": 0
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_dc_enrollment ON enrollment.data_collection(enrollment_id);
CREATE INDEX idx_dc_type_stage ON enrollment.data_collection(data_element_type, data_collection_stage);
CREATE INDEX idx_dc_date ON enrollment.data_collection(information_date);
CREATE INDEX idx_dc_data ON enrollment.data_collection USING GIN (data);
-- Specific path indexes for common queries
CREATE INDEX idx_dc_income ON enrollment.data_collection((data->>'totalMonthlyIncome'))
    WHERE data_element_type = 'income_benefits';
CREATE INDEX idx_dc_disability_type ON enrollment.data_collection((data->>'disabilityType'))
    WHERE data_element_type = 'disability';

-- Services provided (single table with JSONB for service-type-specific details)
CREATE TABLE enrollment.service (
    service_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    date_provided       DATE NOT NULL,
    -- HUD service categorisation as relational columns
    record_type         SMALLINT NOT NULL,
    type_provided       SMALLINT NOT NULL,
    sub_type_provided   SMALLINT,
    -- Financial assistance
    fa_amount           DECIMAL(10,2),
    referral_outcome    SMALLINT,
    -- Service-specific details as JSONB (varies by service type)
    details             JSONB NOT NULL DEFAULT '{}',
    -- For meals: {"mealType": "lunch", "dietaryRestrictions": ["vegetarian"]}
    -- For employment: {"workshopName": "Resume Writing", "hours": 2, "provider": "WorkForce One"}
    -- For health: {"referredTo": "County Mental Health", "appointmentDate": "2026-03-15"}
    -- For benefits: {"benefitType": "SNAP", "applicationDate": "...", "approvedAmount": 250}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_service_enrollment ON enrollment.service(enrollment_id);
CREATE INDEX idx_service_date ON enrollment.service(date_provided);
CREATE INDEX idx_service_type ON enrollment.service(record_type, type_provided);
CREATE INDEX idx_service_details ON enrollment.service USING GIN (details);

-- Case notes
CREATE TABLE enrollment.case_note (
    case_note_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    note_date           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    note_type           VARCHAR(50) NOT NULL DEFAULT 'general',
    title               VARCHAR(255),
    note_text           TEXT NOT NULL,
    is_confidential     BOOLEAN NOT NULL DEFAULT FALSE,
    -- AI-generated metadata stored alongside the note
    ai_metadata         JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"autoTags": ["housing", "employment"], "sentiment": "neutral",
    --        "linkedGoals": ["uuid1", "uuid2"], "suggestedActions": [...]}
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_note_enrollment ON enrollment.case_note(enrollment_id);
CREATE INDEX idx_note_date ON enrollment.case_note(note_date);
CREATE INDEX idx_note_ai ON enrollment.case_note USING GIN (ai_metadata);

-- Service plans with JSONB for flexible goal structures
CREATE TABLE enrollment.service_plan (
    plan_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    plan_type           VARCHAR(50) NOT NULL,
    plan_start_date     DATE NOT NULL,
    plan_end_date       DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Goals as JSONB array (flexible structure, frequently modified)
    goals               JSONB NOT NULL DEFAULT '[]',
    /*
    [
        {
            "goalId": "uuid",
            "description": "Obtain stable employment",
            "targetDate": "2026-06-30",
            "status": "in_progress",
            "milestones": [
                {"description": "Complete resume", "completed": true, "completedDate": "2026-03-01"},
                {"description": "Apply to 5 jobs", "completed": false}
            ],
            "services": ["employment_counseling", "resume_workshop"],
            "outcome": null,
            "completedDate": null
        }
    ]
    */
    created_by          UUID NOT NULL,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_plan_enrollment ON enrollment.service_plan(enrollment_id);
CREATE INDEX idx_plan_status ON enrollment.service_plan(status);
CREATE INDEX idx_plan_goals ON enrollment.service_plan USING GIN (goals);
```

### Assessment Layer (JSONB-Heavy for Instrument Flexibility)

```sql
-- Assessment instruments stored as JSONB documents
-- This is the area that benefits most from document storage
CREATE TABLE enrollment.assessment_instrument (
    instrument_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_name     VARCHAR(200) NOT NULL,
    instrument_version  VARCHAR(50) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    -- The entire instrument definition as JSONB
    instrument_definition JSONB NOT NULL,
    /*
    {
        "name": "VI-SPDAT 2.0 - Single Adults",
        "version": "2.0",
        "targetPopulation": "single_adults",
        "maxScore": 20,
        "sections": [
            {
                "sectionId": "A",
                "title": "History of Housing and Homelessness",
                "questions": [
                    {
                        "questionId": "A1",
                        "text": "Where do you sleep most frequently?",
                        "type": "single_select",
                        "options": [
                            {"value": "sheltered", "label": "Shelters", "score": 0},
                            {"value": "unsheltered", "label": "Outdoors/Street", "score": 1}
                        ],
                        "required": true
                    },
                    {
                        "questionId": "A2",
                        "text": "How long has it been since you lived in permanent stable housing?",
                        "type": "single_select",
                        "options": [...],
                        "conditionalOn": null
                    }
                ]
            },
            {
                "sectionId": "B",
                "title": "Risks",
                "questions": [...]
            }
        ],
        "scoringAlgorithm": {
            "type": "sum",
            "thresholds": [
                {"min": 0, "max": 3, "recommendation": "no_intervention"},
                {"min": 4, "max": 7, "recommendation": "rapid_rehousing"},
                {"min": 8, "max": 20, "recommendation": "permanent_supportive_housing"}
            ]
        }
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (instrument_name, instrument_version)
);

CREATE INDEX idx_instrument_active ON enrollment.assessment_instrument(is_active)
    WHERE is_active = TRUE;

-- Assessment records with responses embedded as JSONB
CREATE TABLE enrollment.assessment (
    assessment_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    instrument_id       UUID NOT NULL REFERENCES enrollment.assessment_instrument(instrument_id),
    -- Relational fields for common queries
    assessment_date     DATE NOT NULL,
    assessment_type     SMALLINT,  -- HUD: 1=Phone, 2=Virtual, 3=In-Person
    assessment_level    SMALLINT,  -- HUD: 1=Crisis, 2=Housing
    prioritization_status SMALLINT,
    total_score         DECIMAL(6,2),
    -- All responses stored as a JSONB document
    responses           JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "A1": {"value": "unsheltered", "score": 1},
        "A2": {"value": "more_than_1_year", "score": 1},
        "B1": {"value": "yes", "score": 1, "notes": "Client reported..."},
        ...
    }
    */
    -- Section-level scores for quick access
    section_scores      JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"A": 5, "B": 3, "C": 2, "D": 4}
    -- AI-enhanced scoring metadata
    ai_scoring          JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"predictedPlacementSuccess": 0.72, "riskFactors": ["chronic_health", "substance_use"],
    --        "recommendedIntervention": "psh", "confidence": 0.85}
    conducted_by        UUID,
    location            VARCHAR(255),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_assessment_client ON enrollment.assessment(personal_id);
CREATE INDEX idx_assessment_date ON enrollment.assessment(assessment_date);
CREATE INDEX idx_assessment_score ON enrollment.assessment(total_score DESC);
CREATE INDEX idx_assessment_responses ON enrollment.assessment USING GIN (responses);
CREATE INDEX idx_assessment_ai ON enrollment.assessment USING GIN (ai_scoring);
```

### Bed Management Layer (Relational Core + JSONB Attributes)

```sql
CREATE TABLE bed.unit (
    unit_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    -- Relational columns for filtering and dashboard queries
    unit_name           VARCHAR(100) NOT NULL,
    unit_type           VARCHAR(50) NOT NULL,
    max_occupants       INTEGER NOT NULL DEFAULT 1,
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    gender_restriction  VARCHAR(20),
    age_restriction     VARCHAR(20),
    -- Variable attributes as JSONB (differ by shelter)
    attributes          JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"floor": "2", "building": "Main", "isAccessible": true,
    --        "isPetFriendly": false, "hasMedicalSupport": false,
    --        "qrCode": "BED-2A-001", "barcode": "123456789",
    --        "amenities": ["locker", "charging_station"],
    --        "specialFeatures": ["medical_bed", "crib_available"]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_unit_project ON bed.unit(project_id);
CREATE INDEX idx_unit_status ON bed.unit(project_id, status);
CREATE INDEX idx_unit_type ON bed.unit(unit_type, gender_restriction, status);
CREATE INDEX idx_unit_attrs ON bed.unit USING GIN (attributes);

-- HUD Inventory (JSONB for the many category-specific bed counts)
CREATE TABLE bed.inventory (
    inventory_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    coc_code            VARCHAR(6) NOT NULL,
    household_type      SMALLINT NOT NULL,
    availability        SMALLINT NOT NULL,
    inventory_start_date DATE NOT NULL,
    inventory_end_date  DATE,
    -- Bed counts as JSONB (HUD adds new categories periodically)
    bed_counts          JSONB NOT NULL,
    /*
    {
        "unitInventory": 25,
        "bedInventory": 50,
        "chVetBedInventory": 5,
        "youthVetBedInventory": 0,
        "vetBedInventory": 8,
        "chYouthBedInventory": 3,
        "youthBedInventory": 10,
        "chBedInventory": 12,
        "otherBedInventory": 12
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_inventory_project ON bed.inventory(project_id);
CREATE INDEX idx_inventory_counts ON bed.inventory USING GIN (bed_counts);

-- Occupancy records (fully relational -- hot path for real-time operations)
CREATE TABLE bed.occupancy (
    occupancy_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id             UUID NOT NULL REFERENCES bed.unit(unit_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    check_in_time       TIMESTAMPTZ NOT NULL,
    check_out_time      TIMESTAMPTZ,
    check_in_method     VARCHAR(20) NOT NULL DEFAULT 'manual',
    check_out_method    VARCHAR(20),
    checked_in_by       UUID,
    checked_out_by      UUID,
    -- Extra context as JSONB
    context             JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"notes": "Arrived late", "belongingsStored": true, "mealProvided": true}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_occupancy_unit ON bed.occupancy(unit_id);
CREATE INDEX idx_occupancy_active ON bed.occupancy(unit_id) WHERE check_out_time IS NULL;
CREATE INDEX idx_occupancy_client ON bed.occupancy(personal_id);
CREATE INDEX idx_occupancy_time ON bed.occupancy(check_in_time);

-- Reservations
CREATE TABLE bed.reservation (
    reservation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id             UUID REFERENCES bed.unit(unit_id),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    reservation_date    DATE NOT NULL,
    expected_arrival    TIMESTAMPTZ,
    expiration_time     TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    waitlist_position   INTEGER,
    priority_score      DECIMAL(6,2),
    -- Requirements as JSONB (flexible matching criteria)
    requirements        JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"unitType": "family", "gender": "female", "accessible": true,
    --        "petFriendly": true, "medicalNeeds": ["oxygen"]}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reservation_project ON bed.reservation(project_id, status);
CREATE INDEX idx_reservation_client ON bed.reservation(personal_id);
CREATE INDEX idx_reservation_waitlist ON bed.reservation(project_id, waitlist_position)
    WHERE status = 'pending';
CREATE INDEX idx_reservation_reqs ON bed.reservation USING GIN (requirements);

-- Nightly census (relational for time-series queries)
CREATE TABLE bed.nightly_census (
    census_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    census_date         DATE NOT NULL,
    total_beds          INTEGER NOT NULL,
    occupied_beds       INTEGER NOT NULL,
    available_beds      INTEGER NOT NULL,
    reserved_beds       INTEGER NOT NULL DEFAULT 0,
    turnaway_count      INTEGER NOT NULL DEFAULT 0,
    -- Breakdown details as JSONB
    breakdown           JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"byUnitType": {"single": {"total": 20, "occupied": 18}, "family": {...}},
    --        "byGender": {"male": {"total": 30, "occupied": 25}, ...},
    --        "maintenanceBeds": 3, "outOfServiceBeds": 1}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, census_date)
);

CREATE INDEX idx_census_date ON bed.nightly_census(census_date);
```

### Coordinated Entry and Referrals (Relational + JSONB)

```sql
CREATE TABLE coordination.prioritization_list (
    list_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    list_name           VARCHAR(200) NOT NULL,
    list_type           VARCHAR(50) NOT NULL,
    -- Scoring criteria as JSONB (configurable per CoC)
    scoring_config      JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "weights": {
            "assessmentScore": 0.4,
            "timeHomeless": 0.3,
            "vulnerabilityFactors": 0.2,
            "chronicity": 0.1
        },
        "bonusPoints": {
            "veteran": 5,
            "chronicallyHomeless": 10,
            "fleeing_dv": 3
        },
        "tiebreakers": ["timeOnList", "assessmentDate"]
    }
    */
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE coordination.prioritization_entry (
    entry_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    list_id             UUID NOT NULL REFERENCES coordination.prioritization_list(list_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    assessment_id       UUID REFERENCES enrollment.assessment(assessment_id),
    priority_score      DECIMAL(8,2) NOT NULL,
    rank_position       INTEGER,
    date_added          DATE NOT NULL,
    date_removed        DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- Detailed scoring breakdown as JSONB
    score_breakdown     JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"assessmentScore": 14, "timeHomelessDays": 365, "bonusVeteran": 5, ...}
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_pe_list ON coordination.prioritization_entry(list_id, rank_position)
    WHERE status = 'active';
CREATE INDEX idx_pe_client ON coordination.prioritization_entry(personal_id);

-- Referrals (relational core for workflow, JSONB for details)
CREATE TABLE coordination.referral (
    referral_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    referring_org_id    UUID NOT NULL REFERENCES core.organization(organization_id),
    receiving_org_id    UUID NOT NULL REFERENCES core.organization(organization_id),
    referral_type       VARCHAR(50) NOT NULL,
    referral_date       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    target_project_id   UUID REFERENCES core.project(project_id),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    status_date         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    priority_score      DECIMAL(8,2),
    -- Workflow history and details as JSONB
    status_history      JSONB NOT NULL DEFAULT '[]',
    /*
    [
        {"status": "pending", "date": "2026-03-01T10:00:00Z", "user": "uuid", "notes": ""},
        {"status": "accepted", "date": "2026-03-02T14:30:00Z", "user": "uuid", "notes": "Bed available"},
        {"status": "completed", "date": "2026-03-05T09:00:00Z", "user": "uuid", "notes": "Client housed"}
    ]
    */
    referral_details    JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"housingUnitId": "uuid", "declineReason": null, "outcome": "housed",
    --        "outcomeDate": "2026-03-05", "matchCriteria": {...}}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_referral_client ON coordination.referral(personal_id);
CREATE INDEX idx_referral_status ON coordination.referral(status);
CREATE INDEX idx_referral_receiving ON coordination.referral(receiving_org_id, status);
CREATE INDEX idx_referral_history ON coordination.referral USING GIN (status_history);

-- Housing placements
CREATE TABLE coordination.housing_placement (
    placement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    referral_id         UUID REFERENCES coordination.referral(referral_id),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    -- Core placement fields as typed columns
    placement_date      DATE NOT NULL,
    move_in_date        DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'placed',
    return_to_homelessness BOOLEAN DEFAULT FALSE,
    -- Housing and financial details as JSONB (vary by housing type)
    housing_details     JSONB NOT NULL DEFAULT '{}',
    /*
    {
        "unitAddress": "123 Main St, Apt 4B",
        "landlordName": "ABC Properties",
        "landlordPhone": "555-0100",
        "bedroomCount": 2,
        "monthlyRent": 1200.00,
        "clientRentPortion": 400.00,
        "subsidyType": "Section 8",
        "leaseStartDate": "2026-04-01",
        "leaseEndDate": "2027-03-31",
        "utilitiesIncluded": ["water", "trash"],
        "moveInCosts": {"deposit": 1200, "firstMonth": 1200, "movingAssistance": 500}
    }
    */
    -- Stability tracking as JSONB
    stability_checks    JSONB NOT NULL DEFAULT '[]',
    /*
    [
        {"date": "2026-05-01", "status": "stabilizing", "notes": "Settling in well"},
        {"date": "2026-06-01", "status": "stable", "notes": "Employed, paying rent on time"},
        {"date": "2026-07-01", "status": "at_risk", "notes": "Lost job, referral to employment services"}
    ]
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_placement_client ON coordination.housing_placement(personal_id);
CREATE INDEX idx_placement_status ON coordination.housing_placement(status);
CREATE INDEX idx_placement_housing ON coordination.housing_placement USING GIN (housing_details);
```

### Configurable Forms Engine (JSONB-Native)

```sql
-- Form definitions for custom intake forms, surveys, and data collection
CREATE TABLE core.form_definition (
    form_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID REFERENCES core.organization(organization_id),
    form_name           VARCHAR(200) NOT NULL,
    form_type           VARCHAR(50) NOT NULL,
    -- 'intake', 'assessment', 'service_log', 'exit', 'survey', 'custom'
    form_version        INTEGER NOT NULL DEFAULT 1,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    -- Complete form definition as JSONB
    definition          JSONB NOT NULL,
    /*
    {
        "title": "Emergency Shelter Intake Form",
        "sections": [
            {
                "title": "Basic Information",
                "fields": [
                    {
                        "fieldId": "presenting_issues",
                        "label": "Presenting Issues",
                        "type": "multi_select",
                        "options": ["housing", "employment", "health", "safety", "other"],
                        "required": true,
                        "hudMapping": null
                    },
                    {
                        "fieldId": "referral_source",
                        "label": "How did you hear about us?",
                        "type": "single_select",
                        "options": ["211", "outreach", "self", "hospital", "police", "other"],
                        "required": false,
                        "hudMapping": null
                    }
                ]
            }
        ],
        "validationRules": [...],
        "conditionalLogic": [...]
    }
    */
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Submitted form responses
CREATE TABLE core.form_submission (
    submission_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    form_id             UUID NOT NULL REFERENCES core.form_definition(form_id),
    form_version        INTEGER NOT NULL,
    personal_id         UUID REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    submitted_by        UUID NOT NULL,
    submitted_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    -- All responses as JSONB
    responses           JSONB NOT NULL,
    -- e.g., {"presenting_issues": ["housing", "employment"], "referral_source": "211"}
    is_complete         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_submission_form ON core.form_submission(form_id);
CREATE INDEX idx_submission_client ON core.form_submission(personal_id);
CREATE INDEX idx_submission_responses ON core.form_submission USING GIN (responses);
```

### Audit Log (Relational + JSONB)

```sql
CREATE TABLE audit.audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    event_time          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id             UUID,
    organization_id     UUID,
    action              VARCHAR(20) NOT NULL,
    table_schema        VARCHAR(50) NOT NULL,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID,
    -- Changes as JSONB for flexible diff storage
    changes             JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"old": {"status": "pending"}, "new": {"status": "accepted"}}
    -- For JSONB column changes, stores JSON path diffs
    request_context     JSONB NOT NULL DEFAULT '{}',
    -- e.g., {"ipAddress": "10.0.0.1", "userAgent": "...", "sessionId": "..."}
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Partition by month
CREATE INDEX idx_audit_time ON audit.audit_log(event_time);
CREATE INDEX idx_audit_user ON audit.audit_log(user_id);
CREATE INDEX idx_audit_table ON audit.audit_log(table_schema, table_name);
CREATE INDEX idx_audit_changes ON audit.audit_log USING GIN (changes);
```

---

## JSONB Validation Strategy

Since JSONB columns do not enforce schema by default, validation is handled at multiple layers:

### Database-Level CHECK Constraints

```sql
-- Validate that income_benefits data has required fields
ALTER TABLE enrollment.data_collection ADD CONSTRAINT chk_income_required
    CHECK (
        data_element_type != 'income_benefits'
        OR (
            data ? 'incomeFromAnySource'
            AND data ? 'benefitsFromAnySource'
            AND data ? 'insuranceFromAnySource'
        )
    );

-- Validate that assessment responses are objects with value keys
ALTER TABLE enrollment.assessment ADD CONSTRAINT chk_responses_format
    CHECK (jsonb_typeof(responses) = 'object');

-- Validate bed counts contain required HUD fields
ALTER TABLE bed.inventory ADD CONSTRAINT chk_bed_counts_required
    CHECK (
        bed_counts ? 'unitInventory'
        AND bed_counts ? 'bedInventory'
    );
```

### Application-Level JSON Schema Validation

```sql
-- Store JSON schemas for each data_element_type
CREATE TABLE core.json_schema_registry (
    schema_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    schema_name         VARCHAR(100) NOT NULL,
    schema_version      VARCHAR(10) NOT NULL,
    hud_standard_version VARCHAR(10),
    json_schema         JSONB NOT NULL,  -- JSON Schema 2020-12 document
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (schema_name, schema_version)
);
```

---

## Pros and Cons

### Pros

1. **HUD standard evolution is painless.** When HUD adds new data elements (as in FY 2024 with HMISParticipation and CEParticipation), they can be added as new keys in existing JSONB columns without database migrations. Existing data retains its structure untouched. The `hud_standard_version` column on data_collection records enables version-aware processing.

2. **Assessment instrument flexibility.** New assessment instruments (replacements for VI-SPDAT, locally-developed tools) are simply new JSON documents in the `assessment_instrument` table. No schema changes needed. Questions, scoring algorithms, and response formats are all defined in JSON.

3. **Agency customization without schema changes.** Each agency can define custom intake fields, service tracking details, and reporting metrics using JSONB `custom_fields` columns and the form engine. A rural shelter tracking "pet type" and an urban shelter tracking "storage locker assigned" coexist without separate columns.

4. **Simpler query patterns for common operations.** Multi-select HUD fields (Race, Gender) stored as JSONB arrays (`races JSONB DEFAULT '[]'`) are simpler to query than junction tables: `WHERE races @> '[1]'` vs. a JOIN to a `client_race` table. GIN indexes make these queries fast.

5. **Single-database simplicity.** Unlike Event Sourcing (Suggestion 2), this model uses a single PostgreSQL instance with no separate read model stores, event buses, or projection processors. Operational complexity is low.

6. **Strong relational guarantees where they matter.** Foreign keys enforce that every enrollment belongs to a real client and a real project. The bed occupancy hot path is fully relational with proper constraints. Only the variable/flexible parts of the data use JSONB.

7. **Natural fit for offline sync payloads.** JSONB data collected offline can be synced as complete JSON documents and merged into the appropriate JSONB columns without complex conflict resolution on individual relational columns.

### Cons

1. **JSONB validation gap.** PostgreSQL does not enforce JSONB document structure at the database level. A typo in a JSON key (`"totalMonthyIncome"` instead of `"totalMonthlyIncome"`) is silently accepted. Application-level validation and JSON Schema enforcement are required, adding a validation layer that relational CHECK constraints handle natively.

2. **Inconsistent query syntax.** Developers must mix SQL column queries with JSONB path queries (`data->>'totalMonthlyIncome'`), JSONB containment operators (`@>`), and JSONB path expressions (`data @? '$.disabilityType ? (@ == 9)'`). This creates a higher learning curve and more opportunities for bugs.

3. **GIN index overhead.** JSONB GIN indexes consume significant storage (typically 2-3x the data size for large JSONB documents) and slow down writes. Over-indexing JSONB columns can degrade insert performance on the data_collection table during bulk intake operations.

4. **Reporting challenges.** HUD reports (APR, LSA, CAPER) require tabular output with specific columns. Extracting structured report data from JSONB fields requires extensive JSON path expressions in SQL, making report queries harder to write, review, and maintain than pure relational queries.

5. **Type safety loss.** JSONB stores all values as JSON types (string, number, boolean, null, array, object). The database cannot enforce that `totalMonthlyIncome` is a number or that `informationDate` is a valid date. Type validation must happen in the application.

6. **Migration complexity for JSONB evolution.** While adding new keys to JSONB is easy, renaming keys, changing value types, or restructuring nested objects requires UPDATE statements with JSONB manipulation functions (`jsonb_set`, `jsonb_strip_nulls`), which are more complex than `ALTER TABLE ... RENAME COLUMN`.

7. **ORM friction.** Most ORMs (ActiveRecord, Prisma, Sequelize) have limited support for JSONB querying and indexing. Complex JSONB queries often require raw SQL, negating some ORM benefits.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pgcrypto, pg_trgm, PostGIS extensions |
| JSONB validation | JSON Schema validation in application layer (ajv for Node.js, jsonschema for Python) |
| JSONB indexing | GIN indexes for containment queries; expression indexes for frequently-accessed paths |
| Schema registry | Application-maintained JSON Schema registry per HUD standard version |
| ORM | Prisma (TypeScript) or SQLAlchemy (Python) -- both have decent JSONB support |
| Migrations | Flyway or Liquibase with JSONB-aware migration scripts |
| Search | pg_trgm for fuzzy name matching; consider pg_vector for ML-based dedup features |
| Caching | Redis for bed availability dashboard (computed from relational occupancy table) |

---

## Migration and Scaling Considerations

### Adding New HUD Data Elements

When HUD releases new data standards (e.g., FY 2028):

1. **New fields in existing JSONB columns**: Add new keys to `data_collection.data`, `enrollment.living_situation_at_entry`, etc. No DDL migration needed.
2. **Update JSON Schema registry**: Register new schema versions for validation.
3. **Update application validation**: New fields validated against new schema version.
4. **Backfill**: Existing records retain their original structure; the `hud_standard_version` column distinguishes old and new records.
5. **New CSV file types**: If HUD introduces a new CSV file (as with HMISParticipation in FY 2024), add a new `data_element_type` value to the `data_collection` table. No new table needed.

### Scaling

1. **Table partitioning**: Partition `enrollment.data_collection` by `information_date` (range) and `data_element_type` (list) for large CoCs.
2. **Read replicas**: Route reporting queries (HUD reports, analytics dashboards) to streaming replicas.
3. **JSONB column splitting**: If a specific JSONB column grows too large (>1KB average), consider promoting frequently-queried paths to dedicated relational columns while keeping the rest in JSONB.
4. **Partial GIN indexes**: Create GIN indexes only on specific JSONB paths rather than entire columns to reduce index size.

### HMIS CSV Import/Export

Importing HMIS CSV files:
- Map each CSV file to the appropriate table and `data_element_type`.
- CSV column headers map to JSONB keys (using the same PascalCase naming convention).
- The CSV→JSONB conversion is straightforward: each CSV row becomes a JSONB object.

Exporting HMIS CSV:
- Query the relational columns for IDs, dates, and keys.
- Extract JSONB values with `->>'fieldName'` notation.
- Format output per HUD CSV Format Specifications.
