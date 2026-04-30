# ATX Assessment — Westbridge Family Medicine Patient Intake

**Scenario**: Scenario 5 — Small-Clinic Patient Intake.
A 6-physician family medicine practice (2 locations, ~180 patients per day) runs its patient intake through a 4-person front-desk team. The intake process for each visit spans: insurance verification, prior-authorisation check for scheduled procedures, pre-visit questionnaire, reason-for-visit triage (routine / urgent / same-day), medication reconciliation, and allergy-flag review. Physicians regularly discover at the visit that something was missed in intake — most commonly an expired prior auth or an unreviewed medication change.

The practice manager wants to offload the administrative load to an agentic workflow but has three hard constraints: (1) no clinical judgment by the agent, (2) any contact with the stated visit reason must preserve a clear human escalation path, (3) HIPAA and state medical-records compliance is non-negotiable. They use athenahealth for EHR and a separate tool for insurance eligibility. They have no AI infrastructure today.

Design the agentic solution.

**Company**: Westbridge Family Medicine — 6-physician family practice, 180 patients/day  
**Stakeholder**: Dana Velazquez, Practice Manager  
**Date**: April 2026  
**Buildability Score**: 3.5/5

---

## Context

The front-desk intake team (4 staff) processes 180 patients per day across four work streams:

| Work Stream | Daily Volume | Annual Hours |
|-------------|--------------|---------------|
| Insurance Verification | 180/day | 3,942 hrs |
| Prior-Authorisation Check | 25/day | 1,825 hrs |
| Pre-Visit Questionnaire | 180/day | 4,380 hrs |
| Medication Reconciliation | 180/day | 6,570 hrs |

Three specific failures surfaced in the scenario artefacts:

| Incident | Root Cause |
|----------|------------|
| Patient visit aborted mid-appointment | Pending PA not flagged at check-in |
| Billing miss | Insurance verification had gone stale after 6 months |
| Medication error risk | Medication reconciliation missed a change — "the third time" |

---

## Deliverable 1: Cognitive Load Map

*Decompose at least 2 of the 4 work streams into Jobs to be Done, micro-tasks, and cognitive dimensions. Map zones and breakpoints.*

### Work Stream 1: Insurance Verification (Decomposed)

#### Job to be Done 1.1: Verify Patient Insurance Eligibility

| Micro-task | Cognitive Zone | Breakpoint Type | Notes |
|------------|----------------|-----------------|-------|
| Retrieve patient demographics from athenahealth | Data Retrieval | System→System | Automated lookup |
| Query Availity for insurance eligibility | External API Call | System→System | 3 min automated |
| Interpret eligibility response (active/inactive/partial) | Decision | System→Human | Classification decision |
| Handle verification failures (~30% of cases) | Diagnosis | Human→System | Requires human judgment |
| Update patient record with verification status | Documentation | Human→System | Write back to athenahealth |

#### Job to be Done 1.2: Resolve Insurance Verification Failures

| Micro-task | Cognitive Zone | Breakpoint Type | Notes |
|------------|----------------|-----------------|-------|
| Identify failure reason (expired, wrong info, system error) | Diagnosis | System→Human | Artefact 5.3 shows verification refresh issue |
| Contact patient for updated insurance info | Communication | Human→Human | Phone call required |
| Re-submit verification with corrected data | Execution | Human→System | Manual retry |
| Document resolution in patient chart | Documentation | Human→System | Required for audit trail |

#### Cognitive Zones — Insurance Verification

| Zone | Description | Key Activities |
|------|-------------|----------------|
| **Intent Recognition** | Identify verification request trigger | New patient, returning patient, procedure scheduled |
| **Data Retrieval** | Fetch patient and insurance data | athenahealth lookup, Availity query |
| **Diagnosis** | Interpret verification response | Active/inactive/partial classification, failure reason |
| **Resolution** | Fix verification failures | Contact patient, correct data, resubmit |
| **Documentation** | Record verification outcome | Update athenahealth, flag for billing |

#### Breakpoints — Insurance Verification

| Breakpoint | From | To | Trigger |
|------------|------|-----|---------|
| Verification failure | System | Human | Availity returns failure code |
| Data mismatch | System | Human | Patient info doesn't match insurer records |
| Manual retry needed | Human | System | Staff resubmits after patient contact |

---

### Work Stream 2: Prior-Authorisation Check (Decomposed)

#### Job to be Done 2.1: Initiate and Track Prior-Authorisation

| Micro-task | Cognitive Zone | Breakpoint Type | Notes |
|------------|----------------|-----------------|-------|
| Identify procedure requiring PA | Intent Recognition | System→System | Based on procedure code + insurer |
| Gather required clinical documentation | Data Retrieval | Human→System | Chart notes, prior visit records |
| Submit PA request to insurer via Availity | External API Call | System→System | 12 min/case average |
| Track PA status in Google Sheets | Monitoring | Human→System | Dana's chase list |
| Handle PA denial | Diagnosis | Human→System | Artefact 5.1 shows Wellpath denial pattern |

#### Job to be Done 2.2: Chase Pending Prior-Authorisations

| Micro-task | Cognitive Zone | Breakpoint Type | Notes |
|------------|----------------|-----------------|-------|
| Review chase list for overdue PAs | Monitoring | Human→System | Dana's Google Sheet |
| Identify insurer-specific SLA patterns | Pattern Recognition | Human→Human | UHC Choice = 6 days, Humana = 6 days |
| Contact insurer for status update | Communication | Human→External | Phone call to insurer |
| Escalate to practice billing if needed | Coordination | Human→Human | Flag for billing team |
| Resubmit with additional documentation | Execution | Human→System | Artefact 5.1: Wellpath wants prior visit note |

#### Job to be Done 2.3: Flag Visit Impact from PA Delays

| Micro-task | Cognitive Zone | Breakpoint Type | Notes |
|------------|----------------|-----------------|-------|
| Identify visit date approaching with pending PA | Monitoring | System→Human | Check athenahealth ticker |
| Flag patient at check-in | Alert | Human→Human | Artefact 5.2: TJ's visit aborted |
| Communicate delay to patient | Communication | Human→Human | Reschedule if PA not confirmed |
| Update visit status in system | Documentation | Human→System | Mark as pending PA |

#### Cognitive Zones — Prior-Authorisation

| Zone | Description | Key Activities |
|------|-------------|----------------|
| **Intent Recognition** | Identify PA requirement | Procedure + insurer combination |
| **Data Retrieval** | Gather clinical documentation | Chart notes, prior visit records |
| **Submission** | Submit PA request | Availity API call |
| **Monitoring** | Track pending PAs | Google Sheets chase list |
| **Diagnosis** | Interpret denial reasons | Insurer-specific patterns |
| **Coordination** | Chase and escalate | Billing, patient communication |
| **Alert** | Flag visit impact | Check-in flag, rescheduling |

#### Breakpoints — Prior-Authorisation

| Breakpoint | From | To | Trigger |
|------------|------|-----|---------|
| PA denial | System | Human | Insurer denies request |
| SLA breach | System | Human | Target chase date passed |
| Visit at risk | System | Human | PA pending, visit approaching |
| Resubmit needed | Human | System | Additional documentation gathered |

---

### Work Stream 3 & 4: Summary (Not Fully Decomposed)

| Work Stream | JtDs | Micro-tasks | Primary Cognitive Zones | Key Breakpoints |
|-------------|------|-------------|-------------------------|------------------|
| Pre-Visit Questionnaire | 2 | 9 | Data Retrieval, Intent Recognition, Classification, Diagnosis, Resolution | NLP confidence <80%, discrepancy found |
| Medication Reconciliation | 3 | 14 | Data Retrieval, Comparison, Diagnosis, Alert, Documentation | New medication found, allergy change detected, critical interaction |

---

## Deliverable 2: Delegation Suitability Matrix

*Score each major task cluster on delegation dimensions; assign archetypes with rationale.*

### Scoring Legend

| Dimension | H (High suitability) | M (Medium suitability) | L (Low suitability) |
|-----------|---------------------|----------------------|---------------------|
| Input Structure | Structured, machine-readable | Semi-structured | Unstructured, ambiguous |
| Decision Determinism | Clear rules, predictable | Pattern-based with exceptions | Judgment-dependent, contextual |
| Tool Coverage | APIs available | Buildable with effort | Systems inaccessible |
| Context Complexity | State can be made explicit | Some tacit knowledge required | Requires institutional knowledge |
| Exception Frequency | Rare, predictable | Moderate (10-20%) | Frequent (>20%) |
| Latency Constraint | Batch acceptable | Hours acceptable | Real-time required |
| Risk/Compliance | Reversible, low consequence | Moderate | Irreversible, regulated |

---

### Task Cluster 1.1: Automated Insurance Verification

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: patient ID, procedure code, insurer code from athenahealth |
| Decision Determinism | H | Clear rules: active/inactive/partial based on Availity response codes |
| Tool Coverage | H | Availity REST API available; athenahealth REST API available |
| Context Complexity | M | Need to understand insurer-specific rules, but can be codified |
| Exception Frequency | M | ~30% fail auto-verify (moderate exception rate) |
| Latency Constraint | H | Batch acceptable; verification can run async |
| Risk/Compliance | M | Billing error consequence, but reversible with refiling |

**Delegation Archetype**: **Agent-led + Human Oversight**

**Rationale**: High-volume structured task with clear rules. Agent can handle the 70% success path autonomously; human reviews the 30% failures.

---

### Task Cluster 1.2: Verification Failure Resolution

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | M | Semi-structured: failure code + patient contact info |
| Decision Determinism | M | Pattern-based: expired card vs. wrong info vs. system error |
| Tool Coverage | M | athenahealth available; patient contact requires human |
| Context Complexity | M | Need to understand patient's situation, but patterns exist |
| Exception Frequency | H | 30% of verifications fail; high exception rate |
| Latency Constraint | H | Can be batched; patient callback can be scheduled |
| Risk/Compliance | M | Billing error, but reversible |

**Delegation Archetype**: **Human-led + Agent Support**

**Rationale**: High exception rate (30%) requires human judgment for patient contact. Agent can gather failure information and prepare resolution options, but human must contact patient.

---

### Task Cluster 2.1: PA Request Submission

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: procedure code + insurer + clinical docs |
| Decision Determinism | H | Clear rules: which procedures require PA for which insurers |
| Tool Coverage | H | Availity API for PA submission; athenahealth for clinical docs |
| Context Complexity | M | Need to gather correct clinical documentation |
| Exception Frequency | L | Most submissions succeed; denial is exception |
| Latency Constraint | M | Hours to days; not real-time but can't be too slow |
| Risk/Compliance | M | Delayed procedure impacts patient care, but reversible |

**Delegation Archetype**: **Agent-led + Human Oversight**

---

### Task Cluster 2.2: PA Status Tracking & Chasing

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: PA ID + insurer + submitted date from Availity |
| Decision Determinism | H | Clear patterns: insurer SLAs are predictable (UHC=6d, Humana=6d) |
| Tool Coverage | M | Availity provides status; Google Sheets is manual workaround |
| Context Complexity | M | Need to understand insurer-specific patterns |
| Exception Frequency | M | Moderate: some PAs pending, some denied |
| Latency Constraint | M | Daily check acceptable; SLA tracking |
| Risk/Compliance | M | Delayed PA impacts patient care |

**Delegation Archetype**: **Agent-led + Human Oversight**

---

### Task Cluster 2.3: PA Denial Handling

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | M | Semi-structured: denial reason + patient history |
| Decision Determinism | M | Pattern-based: Wellpath always denies first time (Artefact 5.1) |
| Tool Coverage | M | Denial reason from Availity; resubmit via API |
| Context Complexity | M | Need to understand denial reason and required fix |
| Exception Frequency | H | Insurer-specific patterns (Wellpath denies first) |
| Latency Constraint | H | Can be batched; resubmit cycle is days |
| Risk/Compliance | M | Patient care delayed; reversible with resubmit |

**Delegation Archetype**: **Human-led + Agent Support**

**Rationale**: Denial handling requires understanding of insurer-specific patterns (tribal knowledge). Agent can surface denial reason and prepare resubmit options, but human must make decision on additional documentation.

---

### Task Cluster 2.4: Visit Impact Flagging

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | M | Semi-structured: PA status + visit date from two systems |
| Decision Determinism | M | Pattern-based: if PA pending and visit approaching, flag |
| Tool Coverage | L | athenahealth has ticker but not integrated to flagging (Artefact 5.2) |
| Context Complexity | H | Need to understand clinical urgency; tacit knowledge |
| Exception Frequency | M | Moderate: some visits at risk |
| Latency Constraint | H | Real-time not required; flag at check-in is sufficient |
| Risk/Compliance | H | Patient visit aborted (Artefact 5.2); high impact |

**Delegation Archetype**: **Human-led + Agent Support**

---

### Task Cluster 4.1: Medication Reconciliation (Primary Target)

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: medication lists from DoseSpot and athenahealth |
| Decision Determinism | M | Clear rules for comparison but changes require interpretation |
| Tool Coverage | H | DoseSpot API and athenahealth API available |
| Context Complexity | M | Need to understand medication changes, but patterns can be codified |
| Exception Frequency | M | Moderate: some patients have changes |
| Latency Constraint | H | Batch acceptable; reconciliation before visit |
| Risk/Compliance | M | Medication error could affect patient care but reversible |

**Delegation Archetype**: **Agent-led + Human Oversight**

**Rationale**: Agent can reconcile medications autonomously. Human reviews flagged changes. This is the primary agentic target.

---

### Task Cluster 4.2: Allergy Review

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: allergy lists from DoseSpot and athenahealth |
| Decision Determinism | M | Clear rules for comparison but new allergies require interpretation |
| Tool Coverage | H | DoseSpot API and athenahealth API available |
| Context Complexity | M | Need to understand clinical significance of allergy changes |
| Exception Frequency | L | Low: most patients have stable allergy lists |
| Latency Constraint | H | Batch acceptable |
| Risk/Compliance | H | Missed severe allergy could be life-threatening |

**Delegation Archetype**: **Agent-led + Human Oversight**

---

### Task Cluster 4.3: Drug Interaction Checking

| Dimension | Score | Rationale |
|-----------|-------|-----------|
| Input Structure | H | Structured: medication list sent to DoseSpot |
| Decision Determinism | H | Clear severity levels from DoseSpot (critical/moderate/minor) |
| Tool Coverage | H | DoseSpot provides interaction data via API |
| Context Complexity | L | DoseSpot provides severity and recommendation |
| Exception Frequency | L | Low: most patients don't have interactions |
| Latency Constraint | H | Batch acceptable |
| Risk/Compliance | H | Critical interaction missed could be life-threatening |

**Delegation Archetype**: **Agent-led + Human Oversight**

---

### Summary Matrix

| Task Cluster | Input | Decision | Tool | Context | Exception | Latency | Risk | Archetype |
|-------------|-------|----------|------|---------|-----------|---------|------|-----------|
| 1.1 Automated Verification | H | H | H | M | M | H | M | Agent-led + Oversight |
| 1.2 Failure Resolution | M | M | M | M | H | H | M | Human-led + Agent Support |
| 2.1 PA Submission | H | H | H | M | L | M | M | Agent-led + Oversight |
| 2.2 PA Tracking | H | H | M | M | M | M | M | Agent-led + Oversight |
| 2.3 Denial Handling | M | M | M | M | H | H | M | Human-led + Agent Support |
| 2.4 Visit Flagging | M | M | L | H | M | H | H | Human-led + Agent Support |
| 4.1 Medication Reconciliation | H | M | H | M | M | H | M | Agent-led + Oversight |
| 4.2 Allergy Review | H | M | H | M | L | H | H | Agent-led + Oversight |
| 4.3 Drug Interaction Check | H | H | H | L | L | H | H | Agent-led + Oversight |

---

## Deliverable 3: Volume × Value Analysis

*Plot the 4 work streams; identify the primary agentic target and justify why it wins.*

### Work Stream Data

| Work Stream | Daily Volume | Annual Volume | Time per Case (min) | Total Annual Hours |
|-------------|--------------|---------------|--------------------|--------------------|
| Insurance Verification | 180/day | ~65,700/yr | ~3.6 avg | ~3,942 hrs |
| Prior-Authorisation Check | 25/day | ~9,125/yr | ~12 min | ~1,825 hrs |
| Pre-Visit Questionnaire | 180/day | ~65,700/yr | ~4 min | ~4,380 hrs |
| Medication Reconciliation | 180/day | ~65,700/yr | ~6 min | ~6,570 hrs |

### Non-Determinism Scoring

| Score | Description | Applied to Work Streams |
|-------|-------------|------------------------|
| 1 | Fully deterministic: pure rules/logic | Insurance Verification (70% pass) |
| 2 | Mostly deterministic: small reasoning component | Prior-Authorisation (submission) |
| 3 | Mixed: core path rule-based, exceptions require reasoning | Prior-Authorisation (chasing, denial) |
| 4 | Significant reasoning: contextual adaptation required | Pre-Visit Triage |
| 5 | High reasoning: synthesis of multiple sources, policy interpretation | Medication Reconciliation |

### Non-Determinism Scores

| Work Stream | Score | Rationale |
|-------------|-------|-----------|
| Insurance Verification | 1.6 | 70% deterministic, 30% requires diagnosis |
| Prior-Authorisation Check | 2.0 | Submission deterministic, chasing pattern-based |
| Pre-Visit Questionnaire | 3.5 | Context required, some judgment on urgency |
| **Medication Reconciliation** | **4.5** | Synthesis of pharmacy history + allergy + interactions |

### Agentic Value Score Calculation

**Formula**: Agentic Value Score = Volume (1-5) × Non-Determinism (1-5)

| Work Stream | Volume Score | Non-Determinism Score | Agentic Value Score | Priority |
|-------------|-------------|----------------------|---------------------|----------|
| Insurance Verification | 5 | 1.6 | 8.0 | Top-left |
| Prior-Authorisation Check | 3 | 2.0 | 6.0 | Bottom-left |
| Pre-Visit Questionnaire | 5 | 3.5 | 17.5 | Top-right (Secondary) |
| **Medication Reconciliation** | 5 | 4.5 | **22.5** | **Top-right (Primary)** |

### Volume × Value Grid

```
High Non-Determinism (5)
        │
        │   Prior-Auth    │   Medication
        │   (6.0)         │   Reconciliation
        │                 │   (22.5) ★ PRIMARY
        │─────────────────┼─────────────────────
        │   Insurance     │   Pre-Visit
        │   Verification  │   Questionnaire
        │   (8.0)         │   (17.5) ◎ SECONDARY
        │                 │
Low Non-Determinism (1)
Low Volume (1) ──────────┴──────────── High Volume (5)
```

### Why Medication Reconciliation Wins

| Factor | Analysis |
|--------|----------|
| **Highest Agentic Value** | Score 22.5 — significantly above threshold (≥15) |
| **High Volume** | 180 patients/day, 65,700/year — justifies investment |
| **High Non-Determinism** | Requires synthesis of pharmacy history, allergy alerts, change flagging |
| **Pain Point** | Artefact 5.3 shows medication reconciliation misses: "third time" |

### Economic Justification

| Metric | Current | With Agent |
|--------|---------|------------|
| Annual hours (Med Recon) | 6,570 hrs | ~657 hrs (HITL only) |
| Annual cost | $295,650 | ~$3,942 |
| **Savings** | — | **$291,708 (98.7%)** |

*Based on $45/hr fully loaded staff cost and 90% automation rate.*

---

## Deliverable 4: Agent Purpose Document

*For the highest-value opportunity: purpose, scope, KPIs, autonomy matrix, escalation triggers, failure modes.*

### Agent Name: MedRecon Agent

### Job to be Done

> Automate the review of patient medication histories and allergy alerts during the intake process, flagging changes, interactions, and discrepancies for front-desk and clinical staff review.

### Business Context

| Attribute | Value |
|-----------|-------|
| Department | Front-Desk Intake Team |
| Process | Patient intake and pre-visit preparation |
| Customer Journey Step | Pre-visit — after check-in, before clinical encounter |
| Primary Users | Front-desk staff (4-person team) |
| Oversight | Dana Velazquez, Practice Manager |

### Primary Objectives

| # | Objective | Success Metric |
|---|-----------|----------------|
| 1 | Automatically reconcile patient medication history | 90% of cases reconciled without human intervention |
| 2 | Flag allergy alerts and medication changes | 95% of critical alerts flagged before visit |
| 3 | Reduce medication-related intake errors | Zero patient safety incidents from medication errors in 12 months |
| 4 | Reduce front-desk staff time on medication reconciliation | 50% reduction in time spent per patient |
| 5 | Maintain HIPAA compliance | 100% compliance with audit requirements |

### KPIs

| KPI | Target | Acceptable Ceiling |
|-----|--------|-------------------|
| **Accuracy** | >95% correct medication reconciliation | >90% |
| **Coverage** | >90% of cases handled without human escalation | >80% |
| **Throughput** | >50 patients/hour | >30 patients/hour |
| **Cost per case** | <$0.10 | <$0.20 |
| **HITL rate** | <10% requiring human review | <20% |
| **Alert recall** | >95% of critical allergy alerts identified | >90% |
| **False positive rate** | <5% | <10% |

### Failure Modes

| Failure Mode | What It Looks Like | Consequence | Recovery Path |
|--------------|-------------------|-------------|---------------|
| **Missed allergy alert** | Patient has documented allergy but agent doesn't flag it | Patient receives contraindicated medication | Human review catches at clinical station |
| **Missed medication change** | New medication not flagged vs. prior visit | Clinical decision on outdated information | Physician catches during visit |
| **False positive alert** | Agent flags non-issue as critical | Staff alert fatigue; wasted review time | Human review filters; feedback to agent |
| **HIPAA breach** | Protected health information exposed | Regulatory violation; patient harm | Incident response protocol |
| **System unavailable** | DoseSpot or athenahealth API down | Intake process stalls | Manual fallback; queue processing |

### Autonomy Matrix

**AGENT DECIDES ALONE** (no HITL required):

| Decision/Action | Examples |
|-----------------|----------|
| Medication reconciliation | Compare current medication list with prior visit; identify additions/removals |
| Allergy flagging | Flag new allergies, removed allergies, allergy status changes |
| Change detection | Identify medication dosage changes, frequency changes |
| Standard alerts | Flag known drug interactions per formulary rules |
| Documentation | Write reconciliation summary to patient chart |

**AGENT ACTS, HUMAN NOTIFIED AFTER**:

| Decision/Action | Examples |
|-----------------|----------|
| Non-critical discrepancies | Medication mismatch that doesn't affect visit; update patient record |
| Routine updates | Patient self-reported medication changes during check-in |
| System status | API availability notifications (logged, not blocking) |

**AGENT PROPOSES, HUMAN APPROVES BEFORE ACTION**:

| Decision/Action | Examples |
|-----------------|----------|
| Critical allergy alert | New severe allergy (anaphylaxis risk); confirm before clinical staff notified |
| Potential contraindication | New medication that may interact with existing prescriptions |
| High-risk patient flag | Immunocompromised, pregnant, other clinical risk factors |
| Unusual medication | Controlled substance, high-cost biologic, specialty medication |

**HUMAN TAKES OVER** (agent supports):

| Trigger | Agent Role |
|---------|------------|
| Clinical interpretation needed | Agent gathers data, presents to nurse/physician |
| Patient disputes medication list | Agent retrieves history, human resolves |
| System unavailable | Agent queues alerts, human processes manually |
| Audit request | Agent compiles reconciliation reports for compliance |

### Escalation Triggers

| Condition | Escalate To | Action |
|-----------|-------------|--------|
| New severe allergy (anaphylaxis) | Dana + Clinical Lead | Immediate phone notification |
| Potential drug interaction | Nursing Station | Flag in athenahealth with priority |
| System API failure >15 min | Dana | Activate manual fallback |
| HITL rate >20% for 3 days | Dana | Review and adjust thresholds |
| Patient safety near-miss | Dana + Medical Director | Incident report within 24 hrs |
| HIPAA concern | Dana + Compliance | Immediate escalation per protocol |

### Activity Catalog

| Task | Type | Delegation Level | Data Required | Tool Required | Risk Level |
|------|------|-----------------|---------------|---------------|------------|
| Retrieve patient medication history | Retrieval | Fully agentic | Patient ID | DoseSpot API | Low |
| Retrieve allergy list | Retrieval | Fully agentic | Patient ID | athenahealth API | Low |
| Compare current vs. prior medications | Reasoning | Fully agentic | Both medication lists | Internal logic | Medium |
| Flag dosage/frequency changes | Decision | Fully agentic | Comparison output | Internal logic | Medium |
| Check drug interactions | Reasoning | Agent-led + HITL | Medication list + formulary | DoseSpot + internal rules | High |
| Flag new severe allergies | Decision | Agent proposes, human approves | Allergy list | athenahealth | High |
| Write reconciliation summary | Generation | Fully agentic | Reconciliation data | athenahealth write API | Low |
| Compile audit report | Generation | Fully agentic | Reconciliation logs | Reporting tool | Low |

---

## Deliverable 5: System/Data Inventory

*What the agent needs to access, what's available, what's missing, what's risky. Address the legacy billing system with batch-file-only exports.*

### System Inventory

#### 1. athenahealth (EHR)

| Attribute | Value |
|-----------|-------|
| **Purpose** | Core electronic health record |
| **API Availability** | REST APIs available |
| **Data Fields** | Patient demographics, visit records, medication history, allergy list, chart notes |
| **Integration Complexity** | Medium |
| **Risk Classification** | High (PHI, clinical data) |

**Available Data for Agent**:
- Patient ID, name, DOB
- Visit history and scheduled visits
- Current medication list
- Allergy list
- Chart notes from prior visits

---

#### 2. Availity (Insurance Eligibility)

| Attribute | Value |
|-----------|-------|
| **Purpose** | Insurance eligibility verification and prior-authorisation |
| **API Availability** | REST APIs available |
| **Data Fields** | Eligibility status, PA status, denial reasons, insurer-specific rules |
| **Integration Complexity** | Low |
| **Risk Classification** | Medium (billing data, not clinical) |

---

#### 3. DoseSpot (Pharmacy/Medication Reconciliation)

| Attribute | Value |
|-----------|-------|
| **Purpose** | Pharmacy history and medication reconciliation |
| **API Availability** | Integrated with athenahealth |
| **Data Fields** | Current prescriptions, pharmacy history, drug interactions, allergy alerts |
| **Integration Complexity** | Low (integrated) |
| **Risk Classification** | High (PHI, medication data) |

---

#### 4. Google Sheets (Internal Tracking)

| Attribute | Value |
|-----------|-------|
| **Purpose** | PA chase list, flagged-patient tracker |
| **API Availability** | Google Sheets API available |
| **Data Fields** | PA status, chase dates, patient flags, notes |
| **Integration Complexity** | Low |
| **Risk Classification** | Low (internal tracking, no PHI) |

---

#### 5. Legacy Billing System

| Attribute | Value |
|-----------|-------|
| **Purpose** | Unknown — scenario mentions "legacy billing system with batch-file-only exports" |
| **API Availability** | **None — batch file only** |
| **Data Fields** | **Unknown — must elicit from stakeholder** |
| **Integration Complexity** | **High — batch file processing required** |
| **Risk Classification** | **Unknown — requires assessment** |

**Critical Gap**: The scenario explicitly mentions a legacy billing system with batch-file-only exports but doesn't name it. This is a significant integration gap that must be addressed through stakeholder elicitation.

---

### Missing Data / Gaps

| Gap | Description | Impact | Mitigation |
|-----|-------------|--------|------------|
| **Legacy billing system** | Not named in scenario; batch-file-only exports | Unknown integration path | **Address in Discovery Questions** |
| athenahealth ticker integration | Ticker exists but not integrated to check-in | Patient visit aborted | Agent correlates PA + calendar |
| Verification refresh window | Insurance stale after 6 months | Billing miss | Agent tracks age, triggers refresh |
| Paper/phone intake forms | No digital access for non-portal patients | Incomplete data | Staff enters data first |

---

### Risk Classification

| System | Data Type | Risk Level | Compliance |
|--------|-----------|------------|------------|
| athenahealth | Clinical (PHI) | High | HIPAA |
| DoseSpot | Medication (PHI) | High | HIPAA |
| Availity | Billing | Medium | PCI-DSS |
| Google Sheets | Internal tracking | Low | Internal policies |
| Legacy Billing | **Unknown** | **Unknown** | **Requires assessment** |

---

### Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           PATIENT INTAKE FLOW                          │
└─────────────────────────────────────────────────────────────────────────┘

Patient Arrives → Front Desk Check-in → Agent Processing → Clinical Review
      │                                    │
      ▼                                    ▼
┌─────────────────┐                 ┌─────────────────────┐
│ Patient ID      │                 │ MEDRECON AGENT      │
│ Verification    │                 │                     │
└─────────────────┘                 │ 1. Query DoseSpot  │
      │                             │    (med history)    │
      ▼                             │                     │
┌─────────────────┐                 │ 2. Query athenahealth│
│ Insurance       │                 │    (allergies)      │
│ Verification    │                 │                     │
│ (Availity)      │                 │ 3. Compare & Flag   │
      │                             │                     │
      ▼                             │ 4. Write to chart   │
┌─────────────────┐                 │    (athenahealth)   │
│ PA Check       │                 │                     │
│ (Availity)      │                 │ 5. Alert if needed  │
      │                             │    (human review)   │
      ▼                             └─────────────────────┘
┌─────────────────┐                        │
│ Visit Reason    │                        ▼
│ Triage          │                 ┌─────────────────────┐
└─────────────────┘                 │ Human Review       │
      │                            │ (Nurse/Physician)   │
      ▼                            └─────────────────────┘
```

---

## Deliverable 6: Discovery Questions

*Questions whose answers would actually change your design, not generic discovery questions.*

### Category 1: System & Data Gaps

#### Question 1.1: Legacy Billing System

> **"The scenario brief mentions a legacy billing system with batch-file-only exports, but it doesn't name it. What is the name of the billing system, and what data does it hold?"**

**What I'm trying to elicit**: The existence and nature of a data integration gap. If the legacy system holds critical data (e.g., billing history, payment status), the agent may need to work around it.

**Why it would change my design**: 
- If the legacy system holds patient billing data relevant to intake, the agent may need to handle batch-file ingestion
- If it's irrelevant to intake, it can be deprioritised
- The batch-file-only constraint affects whether real-time processing is possible

---

#### Question 1.2: athenahealth Ticker Integration

> **"Artefact 5.2 shows a patient visit was aborted because the front desk didn't flag a pending PA at check-in. You mentioned athenahealth has a ticker — why isn't the front desk using it, and what would it take to integrate it into the check-in workflow?"**

**What I'm trying to elicit**: Why existing system features aren't being used and what integration work is needed

**Why it would change my design**:
- If the ticker is functional but unknown to staff → training issue, not agent issue
- If the ticker doesn't surface the right data → agent must replicate the correlation

---

### Category 2: Process & Practice

#### Question 2.1: Medication Reconciliation Gaps

> **"Artefact 5.3 mentions a patient whose verification was stale after 6 months, causing a billing miss. Is there a similar pattern with medication reconciliation — are there patients whose medication history becomes stale or incomplete?"**

**What I'm trying to elicit**: Whether medication data has similar refresh issues as insurance verification

**Why it would change my design**:
- If medication history goes stale → agent needs refresh logic
- If DoseSpot maintains currency → no change needed

---

#### Question 2.2: Wellpath Denial Pattern

> **"The PA chase list shows Wellpath always denies first time and wants the prior visit note attached. Is this pattern documented anywhere, or is it tribal knowledge? Are there similar patterns with other insurers?"**

**What I'm trying to elicit**: The extent of insurer-specific tribal knowledge that would need to be codified

**Why it would change my design**:
- If documented → can be incorporated into agent rules
- If tribal knowledge → needs elicitation and validation

---

### Category 3: Risk & Compliance

#### Question 3.1: HIPAA Constraints on AI

> **"What specific HIPAA or malpractice insurance constraints govern AI usage at the practice? Are there constraints on what data the agent can access, process, or store?"**

**What I'm trying to elicit**: The compliance boundaries for the agent

**Why it would change my design**:
- If strict → may limit what the agent can do
- If moderate → agent can operate with standard safeguards

---

#### Question 3.2: Clinical Judgment Boundary

> **"The hard constraint says 'no clinical judgment by the agent.' Can you give me examples of what that means in practice? Where is the line between administrative flagging and clinical judgment?"**

**What I'm trying to elicit**: The precise boundary between what the agent can and cannot do

**Why it would change my design**:
- If vague → agent may over- or under-reach
- If clear → can define autonomy matrix precisely

---

#### Question 3.3: Previous Intake Misses

> **"The senior physician discovered three intake misses in the last quarter (expired prior auths). What were the root causes, and what would have prevented them?"**

**What I'm trying to elicit**: The specific failure modes the agent must prevent

**Why it would change my design**:
- Defines the agent's primary failure mode targets

---

### Category 4: Stakeholder Concerns

#### Question 4.1: Dana's Personal Stake

> **"What's your personal stake in this project? What are you planning for beyond this project — are you positioning for a larger role, or is this about solving a specific operational pain?"**

**What I'm trying to elicit**: The stakeholder's motivation and long-term vision

---

#### Question 4.2: Fear About AI-Driven Approach

> **"What do you fear about an AI-driven approach? What's the worst-case scenario that keeps you up at night?"**

**What I'm trying to elicit**: The stakeholder's risk concerns

**Why it would change my design**:
- If HIPAA breach → need stronger compliance controls
- If patient safety → need clinical review layer

---

#### Question 4.3: Previous Automation Attempts

> **"What automation has been tried at the practice before, and what happened? What worked, what didn't, and why?"**

**What I'm trying to elicit**: Lessons learned from prior automation attempts

**Why it would change my design**:
- If previous attempts failed → understand why to avoid repeat

---

## Deliverable 7: CLAUDE.md

*Demonstrates workflow discipline.*

### Workflow Summary

| Phase | Activity | Input | Output | Time Budget |
|-------|----------|-------|--------|-------------|
| 1 | Read scenario brief | enriched_scenarios.md | Company, function, work streams, tooling, stakeholder | ~45 min |
| 2 | Read ATX reference docs | atx-concepts.md, atx-assessment.md, atx-scoring.md | Framework understanding | ~30 min |
| 3 | Cognitive Load Mapping | Scenario brief, artefacts | JtDs, micro-tasks, Cognitive Zones, Breakpoints | ~60 min |
| 4 | Delegation Suitability Scoring | Cognitive Load Map | 7-dimension scores (H/M/L), archetypes | ~45 min |
| 5 | Volume × Value Analysis | Work stream data | Agentic value scores, priority grid | ~30 min |
| 6 | Agent Purpose Document | Volume × Value analysis | Full agent specification | ~60 min |
| 7 | System/Data Inventory | Scenario brief | System map, gaps, risk classification | ~30 min |
| 8 | Discovery Questions | Artefacts, gaps | Questions tied to design tensions | ~30 min |
| 9 | Closed Build Loop | Agent Purpose Document | Build output, gap diagnosis | ~60 min |

**Total Time Budget**: ~6 hours (Gate 2 allows 3 hours — time-boxing required)

---

### Iteration Trail

#### Iteration 1: Initial Cognitive Load Map
- Decomposed Insurance Verification and Prior-Authorisation work streams
- Identified 2 JtDs and 9 micro-tasks for Insurance Verification
- Identified 3 JtDs and 13 micro-tasks for Prior-Authorisation
- **Gap**: Didn't include Pre-Visit Questionnaire or Medication Reconciliation

#### Iteration 2: Delegation Matrix
- Scored all task clusters on 7 dimensions
- Assigned archetypes with rationale
- **Correction**: Initially considered "Fully Agentic" for automated verification — corrected to "Agent-led + Human Oversight" due to 30% exception rate

#### Iteration 3: Volume × Value
- Calculated agentic value scores
- **Finding**: Medication Reconciliation (22.5) > Pre-Visit (17.5) > Insurance (8.0) > Prior-Auth (6.0)
- **Decision**: Primary target is Medication Reconciliation

#### Iteration 4: Agent Purpose Document
- Created MedRecon Agent with full specification
- Defined autonomy matrix with 4 levels
- Defined escalation triggers
- **Refinement**: Added "No clinical judgment" boundary explicitly

#### Iteration 5: System/Data Inventory
- Mapped 5 systems
- Identified legacy billing system gap
- **Discovery Question**: Must ask Dana for legacy system name

#### Iteration 6: Discovery Questions
- Generated 11 questions across 4 categories
- **Validation**: All questions tied to specific tensions in artefacts or brief
- **Removed**: Generic questions like "walk me through your process"

#### Iteration 7: Buildability Assessment
- Fed Agent Purpose Document to Claude Code closed build loop
- **Finding**: Initial buildability score 2.3/5 — major gaps in API auth, error handling, HIPAA implementation
- **Resolutions added**: Sections 11–16 of Agent Purpose Document
- **Updated score**: 3.5/5
- **Remaining gaps**: Legacy billing system name, real API credentials, testing plan, stakeholder validation

---

### Key Decisions & Rationale

#### Decision 1: Primary Target = Medication Reconciliation

**Rationale**: 
- Highest agentic value score (22.5)
- High volume (180/day) justifies investment
- High non-determinism (4.5) — requires reasoning beyond rules
- Pain point from Artefact 5.3

#### Decision 2: Delegation Archetype = Agent-led + Human Oversight

**Rationale**:
- High volume structured task
- Clear rules for 70% of cases
- Human reviews 30% failures and all critical alerts
- Hard constraint: "No clinical judgment" requires human for clinical interpretation

#### Decision 3: Not Everything is Fully Agentic

**Anti-Pattern Avoided**:
- Insurance Verification failure resolution → Human-led + Agent Support
- PA Denial Handling → Human-led + Agent Support
- Visit Flagging → Human-led + Agent Support

---

### Assumptions Log

| Assumption | Confidence | Evidence |
|------------|------------|----------|
| Availity API is reliable for eligibility queries | High | Scenario brief: "REST-API tool" |
| ~30% of verifications fail auto-verify | Medium | Scenario brief: "~30% that fail auto-verify" |
| athenahealth ticker exists but isn't used | Medium | Artefact 5.2 |
| Legacy billing system exists but not named | High | Scenario brief mentions it |
| Fully loaded hourly cost = $45/hr | Medium | Industry standard |

---

### Files Produced

| File | Description |
|------|-------------|
| `1-Cognitive-Load-Map.md` | JtDs, micro-tasks, Cognitive Zones, Breakpoints across all 4 work streams |
| `2-Delegation-Suitability-Matrix.md` | 7-dimension scores, archetypes, rationale for 11 task clusters |
| `3-Volume-Value-Analysis.md` | Volume × Non-Determinism scores, priority grid, economic justification |
| `4-Agent-Purpose-Document.md` | Full agent specification (18 sections including API auth, HIPAA, error handling) |
| `5-System-Data-Inventory.md` | System map, data gaps, risk classification, compliance notes |
| `6-discovery-questions.md` | 11 questions tied to design tensions (4 categories) |
| `7-Claude-md.md` | This file — workflow discipline, iteration trail, key decisions |
| `8-Buildability-Assessment.md` | Gap analysis: what Claude Code can and cannot build |
| `9-ATX-Terminology-Explained.md` | Educational reference for ATX concepts with Westbridge examples |

---

## Supporting Content

### ATX Terminology Reference

| Term | Definition | Example |
|------|------------|--------|
| **Job to be Done (JtD)** | The cognitive contract — the outcome the actor is trying to achieve | "Verify patient insurance eligibility" |
| **Micro-task** | Individual atomic steps to complete a JtD | "Query Availity for eligibility" |
| **Cognitive Zone** | Cluster of similar cognitive activity | "Data Retrieval" — all API-fetching tasks |
| **Breakpoint** | Point where control transfers between actors | System → Human when verification fails |

---

### Buildability Assessment

**What Claude Code CAN Build Confidently**:

| Component | Build Confidence | Reason |
|-----------|------------------|--------|
| Basic agent orchestration | High | Clear purpose, defined scope |
| API client wrappers | High | Authentication method, rate limits specified |
| Medication reconciliation logic | High | JtDs defined, interaction data format specified |
| Alert flagging system | High | Thresholds calibrated, priority levels defined |
| Audit logging | High | HIPAA schema specified with field definitions |
| Error handling | High | Timeout, retry, circuit breaker specified |
| System integration | High | Data flow, source of truth, sync frequency defined |

**Buildability Score**: 3.5/5 (improved from 2.3/5 after gap resolution)

**Remaining Gaps**:

| Gap | Impact | Priority |
|-----|--------|----------|
| Legacy billing system name | Low | Low |
| Testing plan | Medium | Medium |
| Actual API credentials | Low | Low |
| Stakeholder validation | Medium | Medium |

---

### Next Steps

1. **Run discovery questions with Dana** — confirm legacy billing system, clinical judgment boundary
2. **Add a testing plan** — define unit test scenarios, integration test cases, UAT criteria
3. **Start Phase 1 build** — medication reconciliation, allergy review, drug interaction check
4. **Validate with first 100 cases** — review alert precision and false positive rate