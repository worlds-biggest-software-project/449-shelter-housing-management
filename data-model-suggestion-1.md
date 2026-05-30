# Data Model Suggestion 1: Normalized Relational Database (PostgreSQL)

> Project: Shelter & Housing Management (449)
> Model Type: Fully Normalized Relational (3NF+)
> Database: PostgreSQL 16+
> Generated: 2026-05-25

---

## Overview

This model implements a fully normalized relational schema in PostgreSQL, closely aligned with the HUD HMIS Data Standards (FY 2024/2026) CSV/XML file structure. Every entity is decomposed into third normal form or higher, with explicit foreign key constraints, check constraints for HUD-mandated enumerated values, and comprehensive indexing for the query patterns required by shelter operations, coordinated entry, and HUD compliance reporting.

The schema is organized into PostgreSQL schemas (namespaces) to separate concerns:

- `core` -- Organizations, projects, users, and system configuration
- `client` -- Client demographics, identifiers, and deduplication
- `enrollment` -- Enrollments, assessments, services, and HUD data elements
- `bed` -- Bed inventory, occupancy, reservations, and availability
- `coordination` -- Coordinated entry, prioritization, referrals, and housing placement
- `reporting` -- Materialized views and reporting aggregates for HUD reports
- `audit` -- Audit logging and change tracking

---

## Schema Definition

### Core Schema -- Organizations, Projects, and Users

```sql
CREATE SCHEMA IF NOT EXISTS core;

-- Continuum of Care
CREATE TABLE core.continuum_of_care (
    coc_id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_code            VARCHAR(6) NOT NULL UNIQUE,  -- e.g., 'CA-500'
    coc_name            VARCHAR(255) NOT NULL,
    geographic_area     TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Organizations within a CoC
CREATE TABLE core.organization (
    organization_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    organization_name   VARCHAR(255) NOT NULL,
    victim_service_provider BOOLEAN NOT NULL DEFAULT FALSE,
    organization_common_name VARCHAR(255),
    federal_ein         VARCHAR(11),
    address_line1       VARCHAR(255),
    address_line2       VARCHAR(255),
    city                VARCHAR(100),
    state               VARCHAR(2),
    zip_code            VARCHAR(10),
    phone               VARCHAR(20),
    email               VARCHAR(255),
    website             VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    data_sharing_agreement_id UUID,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_organization_coc ON core.organization(coc_id);
CREATE INDEX idx_organization_active ON core.organization(is_active) WHERE is_active = TRUE;

-- Projects (programs) operated by organizations
-- Aligned with HUD Project Descriptor Data Elements
CREATE TABLE core.project (
    project_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID NOT NULL REFERENCES core.organization(organization_id),
    project_name        VARCHAR(255) NOT NULL,
    project_common_name VARCHAR(255),
    continuum_project   BOOLEAN NOT NULL DEFAULT TRUE,
    project_type        SMALLINT NOT NULL,  -- HUD: 1=ES-NBN, 2=TH, 3=PH-PSH, etc.
    residential_affiliation BOOLEAN,
    tracking_method     SMALLINT,  -- 0=Entry/Exit, 3=Night-by-Night
    hmis_participating_project BOOLEAN NOT NULL DEFAULT TRUE,
    target_population   SMALLINT,
    operating_start_date DATE NOT NULL,
    operating_end_date  DATE,
    housing_type        SMALLINT,  -- 1=Site-based, 2=Tenant-based, 3=Project-based
    rrh_sub_type        SMALLINT,  -- for RRH projects
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_project_type CHECK (project_type IN (0,1,2,3,4,6,7,8,9,10,11,12,13,14)),
    CONSTRAINT chk_tracking_method CHECK (tracking_method IN (0,3))
);

CREATE INDEX idx_project_org ON core.project(organization_id);
CREATE INDEX idx_project_type ON core.project(project_type);

-- Funder information per project (HUD Funder.csv)
CREATE TABLE core.funder (
    funder_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    funder_type         SMALLINT NOT NULL,  -- HUD: 1=HUD:CoC, 2=HUD:ESG, etc.
    grant_id            VARCHAR(100),
    start_date          DATE NOT NULL,
    end_date            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_funder_project ON core.funder(project_id);

-- Project CoC mapping (a project can serve multiple CoCs)
CREATE TABLE core.project_coc (
    project_coc_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    geocode             VARCHAR(6),
    address_line1       VARCHAR(255),
    city                VARCHAR(100),
    state               VARCHAR(2),
    zip_code            VARCHAR(10),
    geography_type      SMALLINT,  -- 1=Urban, 2=Suburban, 3=Rural
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, coc_id)
);

-- HMIS Participation tracking (new in FY 2024)
CREATE TABLE core.hmis_participation (
    hmis_participation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    hmis_participation_type SMALLINT NOT NULL,  -- 1=HMIS Participating, 2=Non-HMIS
    hmis_participation_status_start_date DATE NOT NULL,
    hmis_participation_status_end_date DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- CE Participation tracking (new in FY 2024)
CREATE TABLE core.ce_participation (
    ce_participation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    access_point        SMALLINT,  -- 0=No, 1=Yes
    prevention_assessment SMALLINT,
    crisis_assessment   SMALLINT,
    housing_assessment  SMALLINT,
    direct_services     SMALLINT,
    receiving_referrals SMALLINT,
    ce_participation_status_start_date DATE NOT NULL,
    ce_participation_status_end_date DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Users and roles
CREATE TABLE core.app_user (
    user_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id     UUID REFERENCES core.organization(organization_id),
    email               VARCHAR(255) NOT NULL UNIQUE,
    password_hash       VARCHAR(255) NOT NULL,
    first_name          VARCHAR(100) NOT NULL,
    last_name           VARCHAR(100) NOT NULL,
    phone               VARCHAR(20),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    last_login_at       TIMESTAMPTZ,
    mfa_enabled         BOOLEAN NOT NULL DEFAULT FALSE,
    mfa_secret_encrypted BYTEA,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE core.role (
    role_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_name           VARCHAR(100) NOT NULL UNIQUE,
    description         TEXT,
    is_system_role      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE core.user_role (
    user_role_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id             UUID NOT NULL REFERENCES core.app_user(user_id),
    role_id             UUID NOT NULL REFERENCES core.role(role_id),
    project_id          UUID REFERENCES core.project(project_id),  -- NULL = org-wide
    granted_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    granted_by          UUID REFERENCES core.app_user(user_id),
    UNIQUE (user_id, role_id, project_id)
);

CREATE TABLE core.permission (
    permission_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    permission_name     VARCHAR(100) NOT NULL UNIQUE,
    resource_type       VARCHAR(50) NOT NULL,  -- e.g., 'client', 'enrollment', 'bed'
    action              VARCHAR(20) NOT NULL,  -- 'read', 'write', 'delete', 'admin'
    field_level         BOOLEAN NOT NULL DEFAULT FALSE,
    field_name          VARCHAR(100),  -- specific field if field_level = TRUE
    description         TEXT
);

CREATE TABLE core.role_permission (
    role_permission_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    role_id             UUID NOT NULL REFERENCES core.role(role_id),
    permission_id       UUID NOT NULL REFERENCES core.permission(permission_id),
    UNIQUE (role_id, permission_id)
);

-- Data sharing agreements between organizations
CREATE TABLE core.data_sharing_agreement (
    agreement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agreement_name      VARCHAR(255) NOT NULL,
    effective_date      DATE NOT NULL,
    expiration_date     DATE,
    agreement_text      TEXT,
    signed_document_url VARCHAR(500),
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE core.agreement_organization (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    agreement_id        UUID NOT NULL REFERENCES core.data_sharing_agreement(agreement_id),
    organization_id     UUID NOT NULL REFERENCES core.organization(organization_id),
    role_in_agreement   VARCHAR(50) NOT NULL DEFAULT 'participant',
    signed_date         DATE,
    signed_by           UUID REFERENCES core.app_user(user_id),
    UNIQUE (agreement_id, organization_id)
);
```

### Client Schema -- Demographics and Identification

```sql
CREATE SCHEMA IF NOT EXISTS client;

-- Client master record (HUD Client.csv)
-- Fields aligned with Universal Data Elements 3.01-3.07
CREATE TABLE client.client (
    personal_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    first_name          VARCHAR(100),
    middle_name         VARCHAR(100),
    last_name           VARCHAR(100),
    name_suffix         VARCHAR(25),
    name_data_quality   SMALLINT NOT NULL DEFAULT 99,
    -- SSN stored encrypted; this column holds the encrypted value
    ssn_encrypted       BYTEA,
    ssn_last_four_hash  VARCHAR(64),  -- for dedup matching without decryption
    ssn_data_quality    SMALLINT NOT NULL DEFAULT 99,
    date_of_birth       DATE,
    dob_data_quality    SMALLINT NOT NULL DEFAULT 99,
    -- HUD 3.04 Race (multi-select, so separate table)
    -- HUD 3.05 Ethnicity moved to separate table in FY2024
    -- HUD 3.06 Gender (multi-select, so separate table)
    veteran_status      SMALLINT,  -- 0=No, 1=Yes, 8=DK, 9=Refused, 99=N/A
    photo_url           VARCHAR(500),
    preferred_language  VARCHAR(50),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID REFERENCES core.app_user(user_id),
    source_organization_id UUID REFERENCES core.organization(organization_id),
    CONSTRAINT chk_name_dq CHECK (name_data_quality IN (1,2,8,9,99)),
    CONSTRAINT chk_ssn_dq CHECK (ssn_data_quality IN (1,2,8,9,99)),
    CONSTRAINT chk_dob_dq CHECK (dob_data_quality IN (1,2,8,9,99)),
    CONSTRAINT chk_veteran CHECK (veteran_status IN (0,1,8,9,99))
);

CREATE INDEX idx_client_name ON client.client(last_name, first_name);
CREATE INDEX idx_client_dob ON client.client(date_of_birth);
CREATE INDEX idx_client_ssn_hash ON client.client(ssn_last_four_hash);
CREATE INDEX idx_client_source_org ON client.client(source_organization_id);

-- Race (HUD 3.04, multi-select in FY2024+)
CREATE TABLE client.client_race (
    client_race_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id) ON DELETE CASCADE,
    race                SMALLINT NOT NULL,
    -- 1=AmIndAKNative, 2=Asian, 3=BlackAfAmerican, 4=NatHIPacific,
    -- 5=White, 6=HispanicLatinx, 8=DK, 9=Refused, 99=N/A
    UNIQUE (personal_id, race),
    CONSTRAINT chk_race CHECK (race IN (1,2,3,4,5,6,7,8,9,99))
);

-- Gender (HUD 3.06, multi-select in FY2024+)
CREATE TABLE client.client_gender (
    client_gender_id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id) ON DELETE CASCADE,
    gender              SMALLINT NOT NULL,
    -- 0=Female, 1=Male, 2=A gender that is not singularly Female or Male,
    -- 3=Transgender, 4=Questioning, 5=Non-Binary, 8=DK, 9=Refused, 99=N/A
    UNIQUE (personal_id, gender),
    CONSTRAINT chk_gender CHECK (gender IN (0,1,2,3,4,5,8,9,99))
);

-- Client aliases for probabilistic matching
CREATE TABLE client.client_alias (
    alias_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id) ON DELETE CASCADE,
    alias_first_name    VARCHAR(100),
    alias_last_name     VARCHAR(100),
    alias_dob           DATE,
    source              VARCHAR(100),  -- which agency/intake reported this
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_alias_name ON client.client_alias(alias_last_name, alias_first_name);

-- Client merge/deduplication tracking
CREATE TABLE client.client_merge (
    merge_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    surviving_id        UUID NOT NULL REFERENCES client.client(personal_id),
    merged_id           UUID NOT NULL,  -- the ID that was retired
    merge_confidence    DECIMAL(5,4),  -- 0.0000 to 1.0000
    merge_method        VARCHAR(50) NOT NULL,  -- 'manual', 'auto_probabilistic', 'exact_match'
    merged_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    merged_by           UUID REFERENCES core.app_user(user_id),
    is_undone           BOOLEAN NOT NULL DEFAULT FALSE,
    UNIQUE (merged_id)
);

-- Consent records for data sharing
CREATE TABLE client.client_consent (
    consent_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    agreement_id        UUID NOT NULL REFERENCES core.data_sharing_agreement(agreement_id),
    consent_given       BOOLEAN NOT NULL,
    consent_date        DATE NOT NULL,
    expiration_date     DATE,
    revocation_date     DATE,
    witness_name        VARCHAR(200),
    signature_url       VARCHAR(500),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_consent_client ON client.client_consent(personal_id);
```

### Enrollment Schema -- Enrollments, Assessments, and HUD Data Elements

```sql
CREATE SCHEMA IF NOT EXISTS enrollment;

-- Household grouping
CREATE TABLE enrollment.household (
    household_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Enrollment (HUD Enrollment.csv)
CREATE TABLE enrollment.enrollment (
    enrollment_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    household_id        UUID NOT NULL REFERENCES enrollment.household(household_id),
    relationship_to_hoh SMALLINT NOT NULL,  -- 1=Self, 2=Child, 3=Spouse, etc.
    entry_date          DATE NOT NULL,
    exit_date           DATE,  -- NULL = still enrolled
    destination         SMALLINT,  -- HUD 3.12 exit destination
    move_in_date        DATE,  -- for PH projects
    enrollment_coc      VARCHAR(6),  -- HUD 3.16
    -- Living situation at entry (HUD 3.917)
    living_situation     SMALLINT,
    length_of_stay      SMALLINT,
    los_under_threshold SMALLINT,
    previous_street_essh SMALLINT,
    date_to_street_essh DATE,
    times_homeless_past_three_years SMALLINT,
    months_homeless_past_three_years SMALLINT,
    -- Disabling condition (HUD 3.08)
    disabling_condition SMALLINT,
    -- Date created / updated
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    created_by          UUID REFERENCES core.app_user(user_id),
    CONSTRAINT chk_rel_hoh CHECK (relationship_to_hoh IN (1,2,3,4,5,99))
);

CREATE INDEX idx_enrollment_client ON enrollment.enrollment(personal_id);
CREATE INDEX idx_enrollment_project ON enrollment.enrollment(project_id);
CREATE INDEX idx_enrollment_household ON enrollment.enrollment(household_id);
CREATE INDEX idx_enrollment_entry ON enrollment.enrollment(entry_date);
CREATE INDEX idx_enrollment_exit ON enrollment.enrollment(exit_date) WHERE exit_date IS NOT NULL;
CREATE INDEX idx_enrollment_active ON enrollment.enrollment(project_id, personal_id)
    WHERE exit_date IS NULL;

-- Income and Benefits (HUD IncomeBenefits.csv)
CREATE TABLE enrollment.income_benefits (
    income_benefits_id  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    data_collection_stage SMALLINT NOT NULL,  -- 1=Entry, 2=Update, 3=Exit, 5=Annual
    information_date    DATE NOT NULL,
    income_from_any_source SMALLINT,
    total_monthly_income DECIMAL(10,2),
    earned_income       DECIMAL(10,2),
    unemployment_insurance DECIMAL(10,2),
    ssi                 DECIMAL(10,2),
    ssdi                DECIMAL(10,2),
    va_disability_service DECIMAL(10,2),
    va_disability_non_service DECIMAL(10,2),
    private_disability  DECIMAL(10,2),
    workers_comp        DECIMAL(10,2),
    tanf                DECIMAL(10,2),
    ga                  DECIMAL(10,2),
    socSecRetirement    DECIMAL(10,2),
    pension             DECIMAL(10,2),
    child_support       DECIMAL(10,2),
    alimony             DECIMAL(10,2),
    other_income_source DECIMAL(10,2),
    other_income_source_identify TEXT,
    -- Non-cash benefits
    benefits_from_any_source SMALLINT,
    snap                SMALLINT,
    wic                 SMALLINT,
    tanf_child_care     SMALLINT,
    tanf_transportation SMALLINT,
    other_tanf          SMALLINT,
    other_benefits_source SMALLINT,
    other_benefits_source_identify TEXT,
    -- Health insurance
    insurance_from_any_source SMALLINT,
    medicaid            SMALLINT,
    medicare            SMALLINT,
    schip               SMALLINT,
    vha_services        SMALLINT,
    employer_provided   SMALLINT,
    cobra               SMALLINT,
    private_pay         SMALLINT,
    state_health_ins    SMALLINT,
    indian_health_services SMALLINT,
    other_insurance     SMALLINT,
    other_insurance_identify TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    CONSTRAINT chk_data_stage CHECK (data_collection_stage IN (1,2,3,5))
);

CREATE INDEX idx_income_enrollment ON enrollment.income_benefits(enrollment_id);
CREATE INDEX idx_income_stage ON enrollment.income_benefits(enrollment_id, data_collection_stage);

-- Health and Domestic Violence (HUD HealthAndDV.csv)
CREATE TABLE enrollment.health_and_dv (
    health_dv_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    data_collection_stage SMALLINT NOT NULL,
    information_date    DATE NOT NULL,
    domestic_violence_survivor SMALLINT,
    when_dv_occurred    SMALLINT,
    currently_fleeing   SMALLINT,
    general_health_status SMALLINT,
    dental_health_status SMALLINT,
    mental_health_status SMALLINT,
    pregnancy_status    SMALLINT,
    due_date            DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_health_enrollment ON enrollment.health_and_dv(enrollment_id);

-- Employment and Education (HUD EmploymentEducation.csv)
CREATE TABLE enrollment.employment_education (
    employment_education_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    data_collection_stage SMALLINT NOT NULL,
    information_date    DATE NOT NULL,
    last_grade_completed SMALLINT,
    school_status       SMALLINT,
    employed            SMALLINT,
    employment_type     SMALLINT,
    not_employed_reason SMALLINT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_employment_enrollment ON enrollment.employment_education(enrollment_id);

-- Disabilities (HUD Disabilities.csv)
CREATE TABLE enrollment.disability (
    disability_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    data_collection_stage SMALLINT NOT NULL,
    information_date    DATE NOT NULL,
    disability_type     SMALLINT NOT NULL,
    -- 5=Physical, 6=Developmental, 7=Chronic Health, 8=HIV/AIDS, 9=Mental Health, 10=Substance Use
    disability_response SMALLINT,
    indefinite_and_impairs SMALLINT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_disability_enrollment ON enrollment.disability(enrollment_id);

-- Current Living Situation (HUD CurrentLivingSituation.csv)
CREATE TABLE enrollment.current_living_situation (
    current_living_situation_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    information_date    DATE NOT NULL,
    current_living_situation SMALLINT NOT NULL,
    verified_by         VARCHAR(100),
    leave_situation_14_days SMALLINT,
    subsequent_residence SMALLINT,
    resources_to_obtain SMALLINT,
    lease_own_60_day    SMALLINT,
    moved_two_or_more   SMALLINT,
    location_details    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_cls_enrollment ON enrollment.current_living_situation(enrollment_id);
CREATE INDEX idx_cls_date ON enrollment.current_living_situation(information_date);

-- Services provided (HUD Services.csv)
CREATE TABLE enrollment.service (
    service_id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    date_provided       DATE NOT NULL,
    record_type         SMALLINT NOT NULL,  -- 141=PATH, 142=RHY, 143=HOPWA, etc.
    type_provided       SMALLINT NOT NULL,
    other_type_provided TEXT,
    sub_type_provided   SMALLINT,
    fa_amount           DECIMAL(10,2),
    referral_outcome    SMALLINT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_service_enrollment ON enrollment.service(enrollment_id);
CREATE INDEX idx_service_date ON enrollment.service(date_provided);

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
    created_by          UUID NOT NULL REFERENCES core.app_user(user_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_case_note_enrollment ON enrollment.case_note(enrollment_id);
CREATE INDEX idx_case_note_date ON enrollment.case_note(note_date);

-- Service plans and goals
CREATE TABLE enrollment.service_plan (
    plan_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    plan_type           VARCHAR(50) NOT NULL,  -- 'housing', 'employment', 'health', etc.
    plan_start_date     DATE NOT NULL,
    plan_end_date       DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    created_by          UUID NOT NULL REFERENCES core.app_user(user_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE enrollment.service_plan_goal (
    goal_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    plan_id             UUID NOT NULL REFERENCES enrollment.service_plan(plan_id),
    goal_description    TEXT NOT NULL,
    target_date         DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'in_progress',
    outcome             TEXT,
    completed_date      DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Vulnerability assessments (VI-SPDAT and alternatives)
CREATE TABLE enrollment.assessment_instrument (
    instrument_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_name     VARCHAR(200) NOT NULL,  -- 'VI-SPDAT 2.0', 'VI-SPDAT 3.0', 'TAY-VI-SPDAT'
    instrument_version  VARCHAR(50) NOT NULL,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    scoring_algorithm   TEXT,  -- description or reference
    max_score           DECIMAL(6,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (instrument_name, instrument_version)
);

CREATE TABLE enrollment.assessment (
    assessment_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    instrument_id       UUID NOT NULL REFERENCES enrollment.assessment_instrument(instrument_id),
    assessment_date     DATE NOT NULL,
    assessment_location VARCHAR(255),
    assessment_type     SMALLINT,  -- HUD: 1=Phone, 2=Virtual, 3=In-Person
    assessment_level    SMALLINT,  -- HUD: 1=Crisis, 2=Housing
    prioritization_status SMALLINT,  -- HUD: 1=Placed, 2=Pending
    total_score         DECIMAL(6,2),
    score_details       TEXT,
    conducted_by        UUID REFERENCES core.app_user(user_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_assessment_client ON enrollment.assessment(personal_id);
CREATE INDEX idx_assessment_date ON enrollment.assessment(assessment_date);
CREATE INDEX idx_assessment_score ON enrollment.assessment(total_score DESC);

-- Individual assessment question responses
CREATE TABLE enrollment.assessment_question (
    question_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    instrument_id       UUID NOT NULL REFERENCES enrollment.assessment_instrument(instrument_id),
    question_number     INTEGER NOT NULL,
    question_text       TEXT NOT NULL,
    question_type       VARCHAR(20) NOT NULL,  -- 'yes_no', 'scale', 'text', 'multi_select'
    max_points          DECIMAL(4,2),
    section             VARCHAR(100),
    UNIQUE (instrument_id, question_number)
);

CREATE TABLE enrollment.assessment_response (
    response_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    assessment_id       UUID NOT NULL REFERENCES enrollment.assessment(assessment_id),
    question_id         UUID NOT NULL REFERENCES enrollment.assessment_question(question_id),
    response_value      TEXT,
    score               DECIMAL(4,2),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_response_assessment ON enrollment.assessment_response(assessment_id);

-- HUD Event.csv (Coordinated Entry events)
CREATE TABLE enrollment.event (
    event_id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    enrollment_id       UUID NOT NULL REFERENCES enrollment.enrollment(enrollment_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    event_date          DATE NOT NULL,
    event_type          SMALLINT NOT NULL,  -- HUD: 1=Referral, 2=Problem Solving, etc.
    problem_sol_div_rr_result SMALLINT,
    referral_case_manage_after SMALLINT,
    location_crisis_or_ph_housing VARCHAR(255),
    referral_result     SMALLINT,
    result_date         DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_event_enrollment ON enrollment.event(enrollment_id);
CREATE INDEX idx_event_date ON enrollment.event(event_date);
```

### Bed Management Schema

```sql
CREATE SCHEMA IF NOT EXISTS bed;

-- Inventory of beds/units per project (HUD Inventory.csv)
CREATE TABLE bed.inventory (
    inventory_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    household_type      SMALLINT NOT NULL,  -- 1=Individual, 3=Family, 4=Both
    bed_type            SMALLINT,  -- 1=Facility, 2=Voucher, 3=Other
    availability        SMALLINT,  -- 1=Year-round, 2=Seasonal, 3=Overflow
    unit_inventory      INTEGER NOT NULL DEFAULT 0,
    bed_inventory       INTEGER NOT NULL DEFAULT 0,
    ch_vet_bed_inventory INTEGER DEFAULT 0,
    youth_vet_bed_inventory INTEGER DEFAULT 0,
    vet_bed_inventory   INTEGER DEFAULT 0,
    ch_youth_bed_inventory INTEGER DEFAULT 0,
    youth_bed_inventory INTEGER DEFAULT 0,
    ch_bed_inventory    INTEGER DEFAULT 0,
    other_bed_inventory INTEGER DEFAULT 0,
    inventory_start_date DATE NOT NULL,
    inventory_end_date  DATE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_inventory_project ON bed.inventory(project_id);

-- Physical bed/unit records for real-time tracking
CREATE TABLE bed.unit (
    unit_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    inventory_id        UUID REFERENCES bed.inventory(inventory_id),
    unit_name           VARCHAR(100) NOT NULL,  -- e.g., 'Room 201', 'Unit 3A'
    floor               VARCHAR(20),
    building            VARCHAR(100),
    unit_type           VARCHAR(50) NOT NULL,  -- 'single', 'double', 'family', 'dorm'
    max_occupants       INTEGER NOT NULL DEFAULT 1,
    is_accessible       BOOLEAN NOT NULL DEFAULT FALSE,  -- ADA accessible
    is_pet_friendly     BOOLEAN NOT NULL DEFAULT FALSE,
    gender_restriction  VARCHAR(20),  -- 'male', 'female', 'any', 'family'
    age_restriction     VARCHAR(20),  -- 'adult', 'youth', 'child', 'any'
    has_medical_support BOOLEAN NOT NULL DEFAULT FALSE,
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    -- 'available', 'occupied', 'reserved', 'maintenance', 'out_of_service'
    qr_code             VARCHAR(100) UNIQUE,
    barcode             VARCHAR(100) UNIQUE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_unit_project ON bed.unit(project_id);
CREATE INDEX idx_unit_status ON bed.unit(project_id, status);
CREATE INDEX idx_unit_type ON bed.unit(unit_type, gender_restriction);

-- Bed occupancy records
CREATE TABLE bed.occupancy (
    occupancy_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id             UUID NOT NULL REFERENCES bed.unit(unit_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    check_in_time       TIMESTAMPTZ NOT NULL,
    check_out_time      TIMESTAMPTZ,
    check_in_method     VARCHAR(20) NOT NULL DEFAULT 'manual',
    -- 'manual', 'barcode', 'qr_code', 'auto'
    check_out_method    VARCHAR(20),
    checked_in_by       UUID REFERENCES core.app_user(user_id),
    checked_out_by      UUID REFERENCES core.app_user(user_id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_occupancy_unit ON bed.occupancy(unit_id);
CREATE INDEX idx_occupancy_client ON bed.occupancy(personal_id);
CREATE INDEX idx_occupancy_active ON bed.occupancy(unit_id)
    WHERE check_out_time IS NULL;
CREATE INDEX idx_occupancy_checkin ON bed.occupancy(check_in_time);

-- Reservations and waitlist
CREATE TABLE bed.reservation (
    reservation_id      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    unit_id             UUID REFERENCES bed.unit(unit_id),  -- NULL if waitlist
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    reservation_date    DATE NOT NULL,
    expected_arrival    TIMESTAMPTZ,
    expiration_time     TIMESTAMPTZ,
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'confirmed', 'checked_in', 'expired', 'cancelled'
    waitlist_position   INTEGER,
    priority_score      DECIMAL(6,2),
    unit_type_requested VARCHAR(50),
    gender_preference   VARCHAR(20),
    special_requirements TEXT,
    created_by          UUID REFERENCES core.app_user(user_id),
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_reservation_project ON bed.reservation(project_id, status);
CREATE INDEX idx_reservation_client ON bed.reservation(personal_id);
CREATE INDEX idx_reservation_waitlist ON bed.reservation(project_id, waitlist_position)
    WHERE status = 'pending';

-- Nightly census snapshots (for historical reporting)
CREATE TABLE bed.nightly_census (
    census_id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    census_date         DATE NOT NULL,
    total_beds          INTEGER NOT NULL,
    occupied_beds       INTEGER NOT NULL,
    available_beds      INTEGER NOT NULL,
    reserved_beds       INTEGER NOT NULL DEFAULT 0,
    maintenance_beds    INTEGER NOT NULL DEFAULT 0,
    turnaway_count      INTEGER NOT NULL DEFAULT 0,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (project_id, census_date)
);

CREATE INDEX idx_census_date ON bed.nightly_census(census_date);
```

### Coordinated Entry and Housing Schema

```sql
CREATE SCHEMA IF NOT EXISTS coordination;

-- Prioritization list (community queue)
CREATE TABLE coordination.prioritization_list (
    list_id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    list_name           VARCHAR(200) NOT NULL,
    list_type           VARCHAR(50) NOT NULL,  -- 'chronic', 'veteran', 'youth', 'family', 'general'
    scoring_criteria    TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Clients on the prioritization list
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
    removal_reason      VARCHAR(100),
    status              VARCHAR(20) NOT NULL DEFAULT 'active',
    -- 'active', 'referred', 'housed', 'inactive', 'removed'
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_priority_list ON coordination.prioritization_entry(list_id, status);
CREATE INDEX idx_priority_client ON coordination.prioritization_entry(personal_id);
CREATE INDEX idx_priority_rank ON coordination.prioritization_entry(list_id, rank_position)
    WHERE status = 'active';

-- Housing inventory available for placement
CREATE TABLE coordination.housing_unit (
    housing_unit_id     UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    unit_address        VARCHAR(500),
    unit_number         VARCHAR(50),
    landlord_name       VARCHAR(200),
    landlord_phone      VARCHAR(20),
    landlord_email      VARCHAR(255),
    bedroom_count       INTEGER NOT NULL,
    max_occupants       INTEGER NOT NULL,
    monthly_rent        DECIMAL(8,2),
    subsidy_type        VARCHAR(100),
    is_accessible       BOOLEAN NOT NULL DEFAULT FALSE,
    is_pet_friendly     BOOLEAN NOT NULL DEFAULT FALSE,
    availability_date   DATE,
    status              VARCHAR(20) NOT NULL DEFAULT 'available',
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_housing_unit_status ON coordination.housing_unit(status);
CREATE INDEX idx_housing_unit_project ON coordination.housing_unit(project_id);

-- Referrals between agencies
CREATE TABLE coordination.referral (
    referral_id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    referring_org_id    UUID NOT NULL REFERENCES core.organization(organization_id),
    receiving_org_id    UUID NOT NULL REFERENCES core.organization(organization_id),
    referring_user_id   UUID REFERENCES core.app_user(user_id),
    receiving_user_id   UUID REFERENCES core.app_user(user_id),
    referral_type       VARCHAR(50) NOT NULL,  -- 'housing', 'shelter', 'service', 'health'
    referral_date       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    target_project_id   UUID REFERENCES core.project(project_id),
    housing_unit_id     UUID REFERENCES coordination.housing_unit(housing_unit_id),
    priority_score      DECIMAL(8,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'pending',
    -- 'pending', 'accepted', 'declined', 'completed', 'expired', 'withdrawn'
    status_date         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    decline_reason      TEXT,
    outcome             VARCHAR(50),
    outcome_date        DATE,
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_referral_client ON coordination.referral(personal_id);
CREATE INDEX idx_referral_receiving ON coordination.referral(receiving_org_id, status);
CREATE INDEX idx_referral_referring ON coordination.referral(referring_org_id);
CREATE INDEX idx_referral_status ON coordination.referral(status);

-- Housing placements
CREATE TABLE coordination.housing_placement (
    placement_id        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    enrollment_id       UUID REFERENCES enrollment.enrollment(enrollment_id),
    referral_id         UUID REFERENCES coordination.referral(referral_id),
    housing_unit_id     UUID REFERENCES coordination.housing_unit(housing_unit_id),
    project_id          UUID NOT NULL REFERENCES core.project(project_id),
    placement_date      DATE NOT NULL,
    move_in_date        DATE,
    lease_start_date    DATE,
    lease_end_date      DATE,
    subsidy_type        VARCHAR(100),
    monthly_rent        DECIMAL(8,2),
    client_rent_portion DECIMAL(8,2),
    status              VARCHAR(20) NOT NULL DEFAULT 'placed',
    -- 'placed', 'stabilizing', 'stable', 'at_risk', 'exited'
    exit_date           DATE,
    exit_reason         VARCHAR(200),
    return_to_homelessness BOOLEAN DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_placement_client ON coordination.housing_placement(personal_id);
CREATE INDEX idx_placement_status ON coordination.housing_placement(status);
CREATE INDEX idx_placement_date ON coordination.housing_placement(placement_date);

-- Case conferencing records
CREATE TABLE coordination.case_conference (
    conference_id       UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    coc_id              UUID NOT NULL REFERENCES core.continuum_of_care(coc_id),
    conference_date     TIMESTAMPTZ NOT NULL,
    conference_type     VARCHAR(50),  -- 'ce_committee', 'by_name_list', 'case_review'
    facilitator_id      UUID REFERENCES core.app_user(user_id),
    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE coordination.case_conference_client (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    conference_id       UUID NOT NULL REFERENCES coordination.case_conference(conference_id),
    personal_id         UUID NOT NULL REFERENCES client.client(personal_id),
    discussion_notes    TEXT,
    action_items        TEXT,
    outcome             VARCHAR(100),
    UNIQUE (conference_id, personal_id)
);
```

### Audit Schema

```sql
CREATE SCHEMA IF NOT EXISTS audit;

-- Comprehensive audit log
CREATE TABLE audit.audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    event_time          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id             UUID REFERENCES core.app_user(user_id),
    organization_id     UUID,
    action              VARCHAR(20) NOT NULL,  -- 'INSERT', 'UPDATE', 'DELETE', 'READ', 'LOGIN'
    table_schema        VARCHAR(50) NOT NULL,
    table_name          VARCHAR(100) NOT NULL,
    record_id           UUID,
    old_values          JSONB,
    new_values          JSONB,
    ip_address          INET,
    user_agent          TEXT,
    session_id          VARCHAR(100)
);

CREATE INDEX idx_audit_time ON audit.audit_log(event_time);
CREATE INDEX idx_audit_user ON audit.audit_log(user_id);
CREATE INDEX idx_audit_table ON audit.audit_log(table_schema, table_name);
CREATE INDEX idx_audit_record ON audit.audit_log(record_id);

-- Table partitioning by month for audit logs
-- (Apply after creation)
-- ALTER TABLE audit.audit_log PARTITION BY RANGE (event_time);
-- CREATE TABLE audit.audit_log_2026_01 PARTITION OF audit.audit_log
--     FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');
```

### Reporting Schema -- Materialized Views

```sql
CREATE SCHEMA IF NOT EXISTS reporting;

-- Real-time bed availability (materialized view, refreshed frequently)
CREATE MATERIALIZED VIEW reporting.bed_availability AS
SELECT
    p.project_id,
    p.project_name,
    p.project_type,
    o.organization_id,
    o.organization_name,
    c.coc_code,
    u.unit_type,
    u.gender_restriction,
    COUNT(u.unit_id) AS total_beds,
    COUNT(u.unit_id) FILTER (WHERE u.status = 'available') AS available_beds,
    COUNT(u.unit_id) FILTER (WHERE u.status = 'occupied') AS occupied_beds,
    COUNT(u.unit_id) FILTER (WHERE u.status = 'reserved') AS reserved_beds,
    COUNT(u.unit_id) FILTER (WHERE u.status = 'maintenance') AS maintenance_beds,
    ROUND(
        COUNT(u.unit_id) FILTER (WHERE u.status = 'occupied')::DECIMAL /
        NULLIF(COUNT(u.unit_id) FILTER (WHERE u.status != 'out_of_service'), 0) * 100, 1
    ) AS occupancy_rate_pct
FROM bed.unit u
JOIN core.project p ON u.project_id = p.project_id
JOIN core.organization o ON p.organization_id = o.organization_id
JOIN core.continuum_of_care c ON o.coc_id = c.coc_id
GROUP BY p.project_id, p.project_name, p.project_type,
         o.organization_id, o.organization_name, c.coc_code,
         u.unit_type, u.gender_restriction;

CREATE UNIQUE INDEX idx_bed_avail ON reporting.bed_availability(project_id, unit_type, gender_restriction);

-- Active enrollment summary
CREATE MATERIALIZED VIEW reporting.active_enrollments AS
SELECT
    e.project_id,
    p.project_name,
    p.project_type,
    COUNT(DISTINCT e.personal_id) AS unique_clients,
    COUNT(e.enrollment_id) AS total_enrollments,
    COUNT(DISTINCT e.household_id) AS unique_households,
    AVG(CURRENT_DATE - e.entry_date) AS avg_length_of_stay_days,
    MIN(e.entry_date) AS earliest_entry,
    MAX(e.entry_date) AS latest_entry
FROM enrollment.enrollment e
JOIN core.project p ON e.project_id = p.project_id
WHERE e.exit_date IS NULL
GROUP BY e.project_id, p.project_name, p.project_type;

-- Housing placement outcomes
CREATE MATERIALIZED VIEW reporting.placement_outcomes AS
SELECT
    hp.project_id,
    p.project_name,
    DATE_TRUNC('month', hp.placement_date) AS placement_month,
    COUNT(hp.placement_id) AS total_placements,
    COUNT(hp.placement_id) FILTER (WHERE hp.status IN ('stable', 'stabilizing')) AS successful,
    COUNT(hp.placement_id) FILTER (WHERE hp.return_to_homelessness = TRUE) AS returns,
    ROUND(
        COUNT(hp.placement_id) FILTER (WHERE hp.return_to_homelessness = TRUE)::DECIMAL /
        NULLIF(COUNT(hp.placement_id), 0) * 100, 1
    ) AS return_rate_pct,
    AVG(hp.move_in_date - hp.placement_date) AS avg_days_to_move_in
FROM coordination.housing_placement hp
JOIN core.project p ON hp.project_id = p.project_id
GROUP BY hp.project_id, p.project_name, DATE_TRUNC('month', hp.placement_date);
```

---

## Pros and Cons

### Pros

1. **HUD HMIS alignment.** The schema directly maps to the HUD HMIS CSV/XML file structure, making compliance reporting (APR, ESG CAPER, LSA, HIC/PIT) straightforward -- each HUD CSV file corresponds to a table or a simple join across two tables.

2. **Referential integrity.** PostgreSQL enforces foreign key constraints across all relationships, preventing orphaned records. CHECK constraints enforce HUD enumerated value lists at the database level, catching invalid data before it reaches the application.

3. **Mature tooling ecosystem.** PostgreSQL has extensive support for migration tools (Flyway, Liquibase, Alembic), ORMs (SQLAlchemy, Prisma, Sequelize, ActiveRecord), reporting tools, and backup/replication infrastructure.

4. **Strong security model.** PostgreSQL schemas enable row-level security (RLS) policies per organization, ensuring tenants in a federated deployment can only access their own data. Field-level encryption for SSN and other PII is straightforward with pgcrypto.

5. **Query flexibility.** Complex queries for coordinated entry (e.g., "find all chronically homeless veterans with a VI-SPDAT score above 10 who have not been referred in the last 30 days") are natural in SQL and can be optimised with composite indexes.

6. **ACID compliance.** All operations (intake, check-in, referral creation) are fully transactional, preventing the data inconsistencies that plague systems relying on eventual consistency for critical shelter operations.

### Cons

1. **Schema rigidity.** Every change to HUD data standards (which occur biennially) requires a database migration. Adding new data elements, changing enumerated values, or restructuring collection stages demands coordinated DDL changes, application code updates, and migration scripts.

2. **Multi-select field overhead.** HUD's FY 2024 changes made Race, Gender, and other fields multi-select, requiring separate junction tables. This increases join complexity for queries that previously used simple column filters.

3. **Scaling write-heavy bed management.** Real-time bed check-in/check-out across a high-volume shelter network creates write contention on the `bed.occupancy` and `bed.unit` tables. At scale (thousands of check-ins per minute across a large CoC), row-level locking may become a bottleneck.

4. **Offline sync complexity.** The normalized structure makes offline-to-online synchronization harder. Conflict resolution when two outreach workers create enrollments for the same client offline requires application-level merge logic that the relational model does not natively support.

5. **Assessment instrument rigidity.** While the schema handles multiple assessment instruments, adding a new instrument with different question structures requires inserting rows into the question table. Dynamic question types (conditional branching, matrix questions) are awkward to represent in pure relational form.

6. **Reporting performance.** Complex HUD reports (LSA, APR) that span large date ranges and aggregate across many enrollment-related tables can be slow without careful indexing or pre-aggregation. The materialized views help but require scheduled refreshing.

---

## Technology Recommendations

| Component | Recommendation |
|-----------|---------------|
| Database | PostgreSQL 16+ with pgcrypto, pg_trgm (fuzzy matching), and PostGIS (geospatial) extensions |
| Connection pooling | PgBouncer or Supavisor for multi-tenant connection management |
| Migrations | Flyway or Liquibase for versioned, repeatable schema migrations |
| Row-level security | PostgreSQL RLS policies per organization_id for multi-tenant isolation |
| Full-text search | pg_trgm + GIN indexes for client name search; consider adding pg_vector for embedding-based deduplication |
| Encryption | pgcrypto for field-level encryption (SSN); application-level encryption for other PII |
| Replication | Streaming replication with read replicas for reporting workloads |
| Backup | pg_basebackup + WAL archiving for point-in-time recovery |
| Monitoring | pg_stat_statements, pgwatch2 or Datadog PostgreSQL integration |

---

## Migration and Scaling Considerations

### HUD Standards Versioning

The schema includes versioned data structures that can absorb annual HUD changes:
- Enumerated values are stored as SMALLINT with CHECK constraints; updating for new HUD values requires `ALTER TABLE ... DROP CONSTRAINT ... ADD CONSTRAINT`.
- New data elements can be added as nullable columns to existing tables without breaking existing queries.
- Major structural changes (e.g., new CSV files like HMISParticipation and CEParticipation in FY 2024) require new tables and migration scripts.

### Scaling Strategy

1. **Vertical scaling** is sufficient for single-site shelters up to ~50,000 clients. PostgreSQL handles this comfortably on a single server with 16GB RAM.

2. **Read replicas** for reporting: Route all HUD report generation and analytics dashboards to streaming replicas, keeping the primary database available for operational workloads.

3. **Table partitioning** for historical data:
   - `audit.audit_log` partitioned by month (range partitioning on `event_time`)
   - `enrollment.enrollment` partitioned by year on `entry_date` for large CoCs
   - `bed.occupancy` partitioned by month on `check_in_time`

4. **Connection pooling** with PgBouncer is essential for multi-tenant deployments where hundreds of concurrent users across agencies share a database.

5. **Materialized view refresh** should be scheduled:
   - `bed_availability`: every 30 seconds for real-time dashboard
   - `active_enrollments`: every 5 minutes
   - `placement_outcomes`: hourly

### Multi-Tenant Architecture

For CoC-level deployments with multiple agencies:
- **Schema-per-tenant**: Each organization gets its own PostgreSQL schema, sharing the same database instance. Cross-agency queries (coordinated entry) use schema-qualified table names.
- **Row-level security (RLS)**: Alternative to schema-per-tenant; all data in shared tables with `organization_id` filters enforced by RLS policies. Simpler to manage but requires careful policy testing.
- **Federated option**: Separate database instances per agency with a central coordination database for shared client records (with consent). Uses PostgreSQL foreign data wrappers (FDW) or application-level data federation.

### Data Migration from Existing HMIS

The schema is designed to accept standard HMIS CSV imports:
- Each HUD CSV file maps to one or two tables in the schema.
- `PersonalID` in CSV maps to `personal_id` (UUID); a migration lookup table maps legacy IDs.
- HMIS CSV export can be generated by querying the appropriate tables and formatting output per the HUD CSV Format Specifications.

---

## Entity Relationship Summary

```
continuum_of_care 1──M organization 1──M project 1──M funder
                                              │
                                              ├──M inventory
                                              ├──M unit 1──M occupancy
                                              │            └──M reservation
                                              └──M enrollment 1──M income_benefits
                                                     │        ├──M health_and_dv
                                                     │        ├──M employment_education
                                                     │        ├──M disability
                                                     │        ├──M current_living_situation
                                                     │        ├──M service
                                                     │        ├──M case_note
                                                     │        ├──M service_plan 1──M goal
                                                     │        ├──M assessment 1──M response
                                                     │        └──M event
                                                     │
client 1──M enrollment                              household
  ├──M client_race
  ├──M client_gender
  ├──M client_alias
  ├──M client_consent
  └──M client_merge

prioritization_list 1──M prioritization_entry
referral ──> organization (referring, receiving)
housing_placement ──> housing_unit, referral
case_conference 1──M case_conference_client
```
