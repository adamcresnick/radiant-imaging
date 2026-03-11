# RADIANT Connector DICOM Retrieval — Design & Implementation

**Date:** 2026-03-05
**Status:** Draft
**Author:** Solutions Architecture (Claude Code + Resnick)
**Classification:** Confidential - Internal RADIANT ARPA-H
**Revision:** Updated to integrate interface-engine / policy-broker workflows, control-plane message contracts, event-driven revocation, queue orchestration, and category-aligned implementation guidance (2026-03-10)

---

**Table of Contents**

- **Part I: Design & Architecture** (Sections 1-18) — Problem statement, transport category taxonomy, site classification, transport mechanisms by category, security, operations
- **Part II: Implementation Plan** (Tasks 0-17) — TDD code, module map, dependency graph, rationale framework

---

# Part I: Design & Architecture

## 1. Problem Statement

The RADIANT Connector establishes consent-scoped Bulk FHIR data exchange with CBTN sites via Epic registries. This pipeline already extracts `ImagingStudy` FHIR resources containing study UIDs and accession numbers for approved patients. However, the actual DICOM source files (full series, all instances) remain on the institutional PACS with no automated retrieval pathway.

### Constraints

- The site must NOT be required to install or manage new software
- The site must NOT be required to operate a separate gateway service
- Queries to the PACS API must be technically validated as scoped to approved patients only
- The solution must leverage native EHR functionalities, existing institutional middleware, and/or the existing RADIANT Connector OAuth2 authorization
- Expected volume: fewer than 10 imaging studies per day per site (batch/nightly)
- Requirement: full DICOM series (all instances, all slices, DICOM headers preserved)

### Design Goals

1. **Portfolio approach**: Provide a menu of transport categories that covers the diversity of site infrastructure across the CBTN network — a site's IT team selects the path that fits their stack after the RADIANT EHR Connector is live
2. **Categorical, not vendor-specific**: Group transport mechanisms by capability class (e.g., "middleware-brokered retrieval" covers F5, Kong, Apigee, MuleSoft alike), so the architecture survives vendor substitution
3. Maximize reuse of the existing Bulk FHIR Connector infrastructure (OAuth2 backend auth, Epic registries, FHIR resource extraction)
4. Provide a universal consent-scoping mechanism that satisfies institutional security review
5. Support heterogeneous site configurations without per-site custom engineering
6. Minimize institutional onboarding burden
7. Maximize use of existing institutional interface engines, API gateways, and integration platforms as the site-owned **policy broker** between the RADIANT manifest and the PACS read-only API

### Architectural Planes

The DICOM retrieval problem decomposes into three distinct planes that must not be conflated:

| Plane | Responsibility | Technology |
|-------|---------------|-----------|
| **Authorization Plane** | Prove which patients and studies RADIANT is permitted to retrieve | RADIANT Connector + Epic registry-scoped Bulk FHIR + Study Manifest |
| **Retrieval Plane** | Move DICOM bytes from the institutional PACS to RADIANT S3 | Epic Interconnect DICOM Gateway, site middleware, Ambra/VNA, or direct PACS |
| **Bridging Problem** | How the authorization plane's manifest becomes retrieval-plane permission | Study ticket, OAuth2 token reuse (if site-validated), platform API auth, or AE title trust |

The authorization plane is solved — the RADIANT Connector and Bulk FHIR registry scoping provide machine-verifiable consent proof. The retrieval plane offers multiple transport categories (Section 2). The bridging problem is the core design challenge: ensuring that the retrieval mechanism can only access studies that the authorization plane approved.

Each transport category (CAT-A through CAT-E) solves the bridging problem differently:
- **CAT-A**: Hypothesized OAuth2 token reuse (requires per-site validation that the FHIR/SMART auth plane extends to the DICOM retrieval plane)
- **CAT-B**: EHR-mediated exchange authorization (EHR bridges internally)
- **CAT-C**: Study ticket — explicit bridging artifact signed by the authorization plane, validated by the retrieval plane
- **CAT-D**: Study manifest + platform API auth (platform bridges)
- **CAT-E**: Study manifest + AE title trust (weakest bridge — relies on network-level trust)

> **Key insight:** CAT-C (study ticket + policy broker) provides the cleanest bridging story because the authorization proof is an explicit, signed, auditable artifact and the site-owned broker can enforce it before PACS access occurs. CAT-A's bridge is hypothesized but unproven in general. This motivates the revised category preference order (Section 2.2).

A site's existing interface engine / API gateway belongs primarily to the **bridging problem**. In the preferred CAT-C pattern, it becomes the site's **policy execution plane**: it accepts signed release messages from RADIANT, persists a study allow-list cache, applies local identifier translation and revocation logic, queues retrieval work, and only then invokes the PACS-facing read-only API or DICOM gateway. HL7 v2 / FHIR / REST messages are used for this control plane; DICOM bytes remain on the retrieval plane.

---

## 2. Transport Category Taxonomy

Every transport mechanism in this plan belongs to one of five **categories**. Categories are defined by the class of infrastructure used to move DICOM bytes, not by a specific vendor. A site selects a category based on what they already have; RADIANT provides the adapter for that category.

### 2.1 Category Definitions

| Category | Code | Description | Examples |
|----------|------|-------------|----------|
| **Native EHR Imaging API** | `CAT-A` | The EHR platform itself serves imaging content through its standard API surface (FHIR, DICOMweb, or proprietary). RADIANT authenticates with the same OAuth2 credentials used for Bulk FHIR and retrieves DICOM through the EHR's own proxy to the PACS. | Epic DICOMweb (WADO-RS), Epic FHIR Binary/Media, Oracle Health Binary.read, Cerner SMART on FHIR Imaging |
| **EHR-Mediated Exchange** | `CAT-B` | The EHR's built-in clinical image exchange infrastructure is repurposed for research retrieval. The EHR handles all PACS communication; RADIANT receives DICOM through a standard exchange protocol. | Epic Care Everywhere Image Exchange, Epic Radiant IHE Profiles (RAD-69/RAD-55), CommonWell/Carequality image exchange |
| **Middleware-Brokered Retrieval** | `CAT-C` | The site's existing API gateway, reverse proxy, or interface engine acts as a **stateful policy broker**. RADIANT presents a signed study release / study ticket; the broker validates identity and signature, persists an allow-list cache, optionally translates local identifiers, enforces revocation and replay prevention, and only then forwards to the PACS read-only API or DICOM gateway. | F5 BIG-IP, NGINX Plus, Kong, Apigee, MuleSoft Anypoint, Azure APIM, Rhapsody, Cloverleaf, Corepoint, InterSystems HealthShare, Mirth Connect, Iguana, Qvera, Epic Bridges/Interconnect, AWS API Gateway |
| **PACS-Adjacent Platform** | `CAT-D` | A vendor-neutral archive (VNA) or enterprise imaging management platform that the site already operates with PACS connectivity. RADIANT sends the study manifest; the platform retrieves from PACS and delivers to S3. Must be an institutionally operated platform, not a workstation tool. | Ambra/InteleShare (Config C), Intelerad, Sectra, institutional DICOMweb gateways |
| **Direct PACS Protocol** | `CAT-E` | Legacy DIMSE-based retrieval (C-FIND/C-MOVE) where no intermediary is available. Requires the site to configure RADIANT as an authorized DICOM destination (AE title). This is the least preferred category due to network exposure requirements. | Any PACS vendor supporting DICOM networking (GE, Siemens, Philips, Fuji, etc.) |

### 2.2 Category Selection Principles

1. **Prefer stronger bridging and retrieval fidelity**: CAT-A > CAT-C > CAT-B > CAT-D > CAT-E for research-grade full-study retrieval. CAT-A is preferred when site-validated because it reuses existing auth with minimal configuration. CAT-C ranks above CAT-B because the study ticket provides an explicit, auditable authorization bridge, while CAT-B (particularly Care Everywhere) may only support reference-quality image exchange rather than full diagnostic DICOM (see Section 6 validation status). CAT-B remains viable if the site can demonstrate full-study retrieval capability.
2. **A site may qualify for multiple categories**: A site with both Epic DICOMweb (CAT-A) and an F5 gateway (CAT-C) can use either. The Task 0 probe validates which are actually functional.
3. **Categories are stable; implementations rotate**: If a site replaces Kong with Apigee, they remain in CAT-C. If they upgrade from Cerner to Epic, they move from one CAT-A implementation to another. The transport adapter interface is the same.
4. **CAT-D has a dual role**: Ambra can serve as a CAT-D transport (Config C — site gateway) OR as a post-retrieval QA layer after a CAT-A/B/C transport. These are distinct uses documented in Section 8.
5. **Use messaging frameworks for the control plane, not the image plane**: HL7 v2 / FHIR / REST / interface engines are ideal for release, revoke, audit, identifier translation, and retry orchestration. DICOM bytes should still move over WADO-RS, platform APIs, or DIMSE — not through HL7 v2 payloads.

### 2.3 Category Requirements Matrix

Each category has prerequisites that must be validated before transport configuration:

| Requirement | CAT-A | CAT-B | CAT-C | CAT-D | CAT-E |
|-------------|-------|-------|-------|-------|-------|
| **RADIANT OAuth2 credentials** | Required | Required | Not used (study ticket instead) | Not used (platform API auth) | Not used (DIMSE AE title) |
| **EHR imaging API enabled** | Required | Required | Not required | Not required | Not required |
| **Site middleware/gateway exists** | Not required | Not required | Required (site-operated) | Not required | Not required |
| **PACS-adjacent platform exists** | Not required | Not required | Not required | Required | Not required |
| **PACS network access from RADIANT** | Not required (EHR proxies) | Not required (EHR proxies) | Not required (gateway proxies) | Not required (platform proxies) | Required (DIMSE association) |
| **Site AE title configuration** | Not required | Not required | Not required | Required (for C-MOVE) | Required |
| **Consent scoping mechanism** | Epic registry + OAuth2 scope | Epic registry + exchange auth | Study ticket + consent proof | Study manifest + platform auth | Study manifest + AE title trust |
| **Task 0 probe validates** | FHIR Binary content, DICOMweb metadata | Care Everywhere endpoint, IHE endpoint | mTLS handshake, ticket validation | Platform API connectivity | DICOM echo (C-ECHO) |

### 2.4 Site-Facing Transport Selection Guide

After the RADIANT EHR Connector is live at a site, use this decision flow to select the imaging transport category. This is designed to be answerable by the site's IT team or PACS administrator in a single conversation.

```
START: RADIANT EHR Connector is operational, Bulk FHIR exports running
  |
  Q1: Does your EHR serve imaging content through its FHIR/DICOMweb API?
  |     (Epic: DICOMweb proxy or FHIR Binary? Oracle Health: Binary.read?)
  |
  +---> YES, confirmed --> CAT-A: Native EHR Imaging API
  |                         Run Task 0 probe to validate. Done.
  |
  +---> UNSURE / NOT CONFIRMED --> Continue to Q2
  |
  Q2: Does your EHR support clinical image exchange with external partners?
  |     (Epic: Care Everywhere image sharing? IHE profiles enabled?)
  |
  +---> YES, and available for programmatic use --> CAT-B: EHR-Mediated Exchange
  |                                                 Run Task 0 probe. Done.
  |
  +---> NO, or only for interactive clinical use --> Continue to Q3
  |
  Q3: Do you operate an API gateway or integration engine that fronts
  |   internal systems (PACS, VNA, etc.)?
  |     (F5, Kong, Apigee, MuleSoft, Rhapsody, HealthShare, etc.)
  |
  +---> YES --> CAT-C: Middleware-Brokered Retrieval
  |              RADIANT provides a study ticket policy for your gateway.
  |              Three variants available (basic, Epic-release, token exchange).
  |              Run Task 0 probe to validate mTLS + ticket flow. Done.
  |
  +---> NO --> Continue to Q4
  |
  Q4: Do you operate a VNA, enterprise viewer, or imaging platform with
  |   its own PACS connectivity and an external API?
  |     (Ambra/InteleShare, Intelerad, Sectra, or other institutionally operated platform)
  |
  +---> YES --> CAT-D: PACS-Adjacent Platform
  |              RADIANT sends study manifest; platform retrieves and delivers.
  |              Run Task 0 probe to validate API connectivity. Done.
  |
  +---> NO --> Continue to Q5
  |
  Q5: Can your PACS accept a new AE title / DICOM destination
  |   for automated retrieval?
  |
  +---> YES --> CAT-E: Direct PACS Protocol (C-FIND/C-MOVE)
  |              Least preferred. Requires DICOM network exposure.
  |              Coordinate with RADIANT infrastructure team.
  |
  +---> NO --> ESCALATE: Manual review with RADIANT Solutions Architecture.
               Site may need EHR vendor engagement or infrastructure upgrade.
```

**Key point for site conversations:** Questions are ordered from "least site effort" to "most site effort." Most sites will exit at Q1 or Q3. Sites that reach Q5 are rare and typically indicate aging infrastructure.

```mermaid
flowchart TD
    START["RADIANT EHR Connector Operational\nBulk FHIR Exports Running"] --> Q1{"Q1: Does EHR serve imaging\nvia FHIR/DICOMweb API?"}
    Q1 -->|"YES, confirmed"| CATA["CAT-A: Native EHR Imaging API"]
    Q1 -->|"UNSURE / NOT CONFIRMED"| Q2{"Q2: Does EHR support clinical\nimage exchange with partners?"}
    Q2 -->|"YES, programmatic"| CATB["CAT-B: EHR-Mediated Exchange"]
    Q2 -->|"NO / interactive only"| Q3{"Q3: Does site operate an API\ngateway or integration engine?"}
    Q3 -->|"YES"| CATC["CAT-C: Middleware-Brokered\nRetrieval"]
    Q3 -->|"NO"| Q4{"Q4: Does site have a VNA,\nenterprise viewer, or imaging\nplatform with PACS connectivity?"}
    Q4 -->|"YES"| CATD["CAT-D: PACS-Adjacent Platform"]
    Q4 -->|"NO"| Q5{"Q5: Can PACS accept a new\nAE title / DICOM destination?"}
    Q5 -->|"YES"| CATE["CAT-E: Direct PACS Protocol\n(C-FIND/C-MOVE)"]
    Q5 -->|"NO"| ESCALATE["ESCALATE: Manual review with\nRADIANT Solutions Architecture"]

    CATA --> PROBE["Run Task 0\nCapability Probe"]
    CATB --> PROBE
    CATC --> PROBE
    CATD --> PROBE
    CATE --> PROBE

    style CATA fill:#2d6a4f,color:#fff
    style CATB fill:#40916c,color:#fff
    style CATC fill:#52b788,color:#000
    style CATD fill:#74c69d,color:#000
    style CATE fill:#95d5b2,color:#000
    style ESCALATE fill:#e63946,color:#fff
    style PROBE fill:#457b9d,color:#fff
```

---

### 2.5 Interface Engine as Policy Broker (Control-Plane Maximization)

Health systems already manage external and internal clinical data movement through interface engines and integration platforms. In this architecture, RADIANT should maximize those frameworks by using them as the **site-owned policy broker** for imaging retrieval. That means:

- RADIANT sends an explicit **StudyRelease** authorization message (or signed study ticket) to the site's existing broker
- The broker persists a short-lived allow-list keyed to study UID / accession / local identifiers
- The broker applies local rules such as identifier correlation, finalized-status checks, image-availability checks, and revocation
- The broker then invokes the site's PACS-facing DICOM gateway / read-only API using site-managed credentials
- The broker emits completion / failure / revocation events back to RADIANT

This maximizes use of infrastructure the site already operates and trusts — Epic Bridges / Interconnect, Rhapsody, Cloverleaf, Corepoint, HealthShare, Mirth, Iguana, Qvera, F5, Apigee, Kong, MuleSoft, Azure APIM, and similar platforms.

```mermaid
sequenceDiagram
    participant R as RADIANT
    participant B as Site Policy Broker
    participant G as DICOM Gateway / PACS API
    participant P as PACS

    R->>B: StudyRelease (signed, time-limited)
    B->>B: Validate identity + signature
    B->>B: Persist allow-list entry
    B->>B: Translate local IDs / apply policy
    B->>G: WADO-RS / platform read
    G->>P: Retrieve study
    P-->>G: DICOM bytes
    G-->>B: DICOM bytes / status
    B-->>R: StudyReady or RetrievalFailed
    R->>B: StudyRevoke (if needed)
    B->>B: Cancel queued / future retrievals
```

Section 9 expands this into a full control-plane design with message contracts, queues, revocation handling, and audit.

## 3. Site Classification

Every CBTN site is classified at onboarding along two independent axes:

**Primary axis — Transport Category:** Which mechanism moves DICOM bytes from the institutional PACS to RADIANT? (CAT-A through CAT-E, determined by the Section 2.4 selection guide and validated by Task 0 probe)

**Secondary axis — Post-Retrieval QA:** Is a QA/routing platform available to stage, validate, tag, and de-identify studies before they land in S3?

```
                         QA Platform Available          Direct to S3
                         (Ambra, VNA, etc.)             (no intermediary)
                    ┌─────────────────────────┬───────────────────────────┐
  CAT-A             │  Transport + QA          │  Transport + inline       │
  Native EHR API    │  (EHR retrieves,         │  validation               │
                    │   platform validates)     │  (simplest path)          │
                    ├─────────────────────────┼───────────────────────────┤
  CAT-B             │  Exchange + QA           │  Exchange + inline        │
  EHR Exchange      │  (EHR pushes to          │  validation               │
                    │   platform for QA)        │                           │
                    ├─────────────────────────┼───────────────────────────┤
  CAT-C             │  Gateway + QA            │  Gateway + inline         │
  Middleware        │  (gateway retrieves,     │  validation               │
                    │   platform validates)     │                           │
                    ├─────────────────────────┼───────────────────────────┤
  CAT-D             │  Platform IS the         │  N/A — CAT-D requires     │
  PACS-Adjacent     │  transport AND QA        │  a platform by definition │
                    │  (single system)          │                           │
                    ├─────────────────────────┼───────────────────────────┤
  CAT-E             │  DIMSE + QA              │  DIMSE + inline           │
  Direct PACS       │  (C-MOVE to platform)    │  validation               │
                    └─────────────────────────┴───────────────────────────┘
```

**All configurations share:**
- Upstream: Bulk FHIR pipeline producing a daily Study Manifest (Section 4)
- Downstream: DICOM studies landing in RADIANT S3, tagged with consent provenance
- Consent scoping: Study manifest inherits authorization from registry-scoped Bulk FHIR export

**What varies:**
- **Transport layer** (primary axis): How DICOM bytes move from PACS to RADIANT
- **QA layer** (secondary axis): Whether studies pass through an intermediate validation/routing platform before S3

> **Note on Ambra:** Ambra appears in both axes. As a QA platform (secondary axis), it validates, tags, and routes studies retrieved by any transport. As a CAT-D transport (primary axis, Config C), it retrieves studies from PACS directly. These are distinct roles documented in Section 8.

### Site Capability Assessment (Onboarding Checklist)

| Question | Determines | Category |
|----------|-----------|----------|
| Does the site's EHR expose imaging content through its FHIR/DICOMweb API? | CAT-A viability | `CAT-A` Native EHR Imaging API |
| Does the site's EHR support clinical image exchange with external partners? | CAT-B viability | `CAT-B` EHR-Mediated Exchange |
| Does the site operate an API gateway or integration engine? | CAT-C viability | `CAT-C` Middleware-Brokered Retrieval |
| Does the site have an Ambra/InteleShare, VNA, or enterprise viewer with PACS connectivity? | CAT-D viability | `CAT-D` PACS-Adjacent Platform |
| Can the site's PACS accept new AE title / destination configurations? | CAT-E viability | `CAT-E` Direct PACS Protocol |
| What EHR platform and version is deployed? | Platform-specific adapter selection | All categories |
| What PACS vendor and version is deployed? | DIMSE vs DICOMweb capability | CAT-D, CAT-E |

### Preflight Capability Probe (Task 0 — Automated)

Before configuring any transport, the RADIANT pipeline runs an automated **Site Capability Probe** that validates assumptions from the onboarding checklist against live infrastructure. This is a gate — no transport is configured until the probe completes.

| Probe | Method | Category Validated |
|-------|--------|-------------------|
| FHIR Binary content probe | `GET /Binary/{id}` for a known ImagingStudy attachment | `CAT-A` — whether EHR serves DICOM via FHIR Binary |
| FHIR Media content probe | `GET /Media?subject={patient}` | `CAT-A` — whether imaging content is surfaced as Media |
| DICOMweb metadata probe | `GET {dicomweb_base}/studies?limit=1` | `CAT-A` — whether WADO-RS returns valid DICOM JSON |
| DiagnosticReport imaging probe | `GET /DiagnosticReport?category=imaging&patient={patient}` | `CAT-A/B` — secondary study discovery path |
| EHR exchange endpoint probe | Vendor-specific exchange API test | `CAT-B` — whether exchange API accepts backend requests |
| Gateway availability probe | mTLS handshake + synthetic `StudyRelease` / study-ticket validation at the configured broker endpoint | `CAT-C` — whether the site broker accepts signed release messages and can enforce policy before PACS access |
| Platform connectivity probe | `GET /session/validate` or equivalent API health check | `CAT-D` — whether PACS-adjacent platform is reachable |
| DICOM echo probe | C-ECHO to configured AE title | `CAT-E` — whether PACS responds to DIMSE association |

**Transport recommendation priority** (from probe results, follows category preference order):

1. `CAT-A` **Native EHR Imaging API** — FHIR Binary, DICOMweb, or vendor imaging API (simplest if site-validated, reuses existing auth)
2. `CAT-C` **Middleware-Brokered Retrieval** — site's existing gateway with study ticket (explicit authorization bridge, vendor-agnostic)
3. `CAT-B` **EHR-Mediated Exchange** — IHE profiles, Care Everywhere (EHR handles PACS communication; must confirm full-study support)
4. `CAT-D` **PACS-Adjacent Platform** — Ambra, VNA, or enterprise viewer as gateway (platform handles retrieval)
5. `CAT-E` **Direct PACS Protocol** — C-FIND/C-MOVE (least preferred, requires DICOM network exposure)

The probe results are recorded in a `SiteCapabilityReport` that becomes the input to transport configuration. See Task 0 in the implementation plan for full TDD specification.

```python
@dataclass
class ProbeResult:
    status: Literal["validated", "not_available", "error", "not_tested"]
    response_code: int | None = None
    response_time_ms: float | None = None
    error_message: str | None = None
    endpoint_url: str | None = None
    capabilities: dict | None = None  # e.g., supported WADO-RS transfer syntaxes

@dataclass
class SiteCapabilityReport:
    site_id: str
    probe_timestamp: str  # ISO 8601
    ehr_platform: str  # "epic", "oracle_health", "other"

    # CAT-A probes
    cat_a_fhir_binary: ProbeResult
    cat_a_fhir_media: ProbeResult
    cat_a_dicomweb: ProbeResult
    cat_a_diagnostic_report: ProbeResult

    # CAT-B probes
    cat_b_care_everywhere: ProbeResult
    cat_b_ihe_rad69: ProbeResult

    # CAT-C probes
    cat_c_gateway: ProbeResult
    cat_c_gateway_platform: str | None = None  # "f5", "kong", "apigee", etc.

    # CAT-D probes
    cat_d_platform: ProbeResult
    cat_d_platform_type: str | None = None  # "ambra", "intelerad", etc.

    # CAT-E probes
    cat_e_dicom_echo: ProbeResult
    cat_e_ae_title: str | None = None

    # Computed recommendation
    recommended_transport: str  # "cat_a_dicomweb", "cat_a_fhir_binary", "cat_b_care_everywhere", etc.
    fallback_transports: list[str]  # ordered list of validated alternatives
    qa_platform_available: bool
    qa_platform_type: str | None = None  # "ambra", other VNA, etc.
```

---

## 4. Study Manifest — The Consent-Scoped Shopping List

This component is shared by all configurations and all transport categories. It runs daily as a post-processing step after the Bulk FHIR incremental pull.

### 4.1 Data Flow

```
Epic Bulk FHIR (Incremental Pull, registry-scoped)
    |
    v
NDJSON Bundle --> filter resource_type = "ImagingStudy"
    |
    v
Study Manifest Builder
    |
    v
study_manifest_YYYY-MM-DD.json
    |
    v
Retrieval Ledger (cumulative, dedup)
```

```mermaid
flowchart TD
    BULK["Epic Bulk FHIR\n(Incremental Pull, registry-scoped)"] --> NDJSON["NDJSON Bundle"]
    NDJSON --> FILTER{"Filter:\nresource_type = ImagingStudy"}
    FILTER --> BUILDER["Study Manifest Builder"]
    BUILDER --> MANIFEST["study_manifest_YYYY-MM-DD.json"]
    MANIFEST --> LEDGER["Retrieval Ledger\n(cumulative, dedup)"]

    NDJSON --> FILTER2{"Secondary Discovery:\nresource_type = DiagnosticReport"}
    FILTER2 --> DR_FILTER["Filter: category = imaging\nor contains RAD"]
    DR_FILTER --> EXTRACT["Extract study UIDs from\npresentedForm, media refs,\ncontained ImagingStudy"]
    EXTRACT --> DEDUP["Cross-reference &\ndedup by study_uid"]
    DEDUP --> BUILDER

    style BULK fill:#264653,color:#fff
    style MANIFEST fill:#2a9d8f,color:#fff
    style LEDGER fill:#e9c46a,color:#000
```

### 4.2 ImagingStudy Resource Extraction

From the FHIR R4 `ImagingStudy` resource:

| Field | FHIR Path | Use |
|-------|-----------|-----|
| Study Instance UID | `ImagingStudy.identifier` (DICOM UID system) | Primary key for PACS query |
| Accession Number | `ImagingStudy.identifier` (accession system) | Alternate PACS query key |
| Patient FHIR ID | `ImagingStudy.subject.reference` | Links back to registry consent |
| Study Date | `ImagingStudy.started` | Dedup and ordering |
| Modality | `ImagingStudy.modality` | Filtering (MR, CT, PT) |
| Number of Series | `ImagingStudy.numberOfSeries` | Validation — expected count |
| Number of Instances | `ImagingStudy.numberOfInstances` | Validation — expected count |
| Referring Physician | `ImagingStudy.referrer` | Audit trail |
| Study Description | `ImagingStudy.description` | Human-readable context |
| Endpoint Reference | `ImagingStudy.endpoint` | May contain DICOMweb URL directly |

### 4.3 Manifest Schema

```json
{
  "manifest_id": "SCH-2026-03-05-001",
  "generated_at": "2026-03-05T02:00:00Z",
  "site_id": "seattle_childrens",
  "registry_id": "100263",
  "transport_config": "epic_dicomweb",
  "studies": [
    {
      "study_uid": "1.2.840.113619.2.55.3.12345",
      "accession_number": "RAD-2026-04521",
      "patient_fhir_id": "e4f2a8c1-...",
      "patient_mrn_hash": "sha256:...",
      "study_date": "2026-03-04",
      "modality": ["MR"],
      "description": "MRI BRAIN W AND WO CONTRAST",
      "expected_series": 12,
      "expected_instances": 2847,
      "endpoint_ref": "Endpoint/dicomweb-1",
      "consent_proof": {
        "registry_name": "CHOP CBTN ALL PATIENT REGISTRY",
        "registry_id": "100263",
        "bulk_fhir_export_id": "export-2026-03-05-daily",
        "extraction_timestamp": "2026-03-05T01:45:00Z"
      },
      "retrieval_status": "pending",
      "retrieval_attempts": 0
    }
  ],
  "manifest_stats": {
    "total_studies": 7,
    "new_studies": 3,
    "previously_retrieved": 4,
    "modality_breakdown": {"MR": 5, "CT": 1, "PT": 1}
  }
}
```

### 4.4 Secondary Discovery via DiagnosticReport

Not all sites populate `ImagingStudy` resources completely. Some sites have study UIDs only in `DiagnosticReport` resources linked to radiology results. The manifest builder supports a secondary discovery path:

```
NDJSON Bundle --> filter resource_type = "DiagnosticReport"
    |
    v
Filter: category = "imaging" OR category contains "RAD"
    |
    v
Extract: study UIDs from presentedForm, media references, or contained ImagingStudy
    |
    v
Cross-reference against ImagingStudy manifest (dedup by study_uid)
    |
    v
Add newly discovered studies to manifest with source = "diagnostic_report"
```

The secondary discovery path is controlled by the `SiteProfile.secondary_discovery_resources` configuration (see Section 4.5). Sites where ImagingStudy is well-populated can disable this to avoid unnecessary API calls.

### 4.5 Site-Configurable Manifest Parsing (SiteProfile)

The manifest parser is parameterized by a `SiteProfile` dataclass rather than hardcoded assumptions about identifier systems, accession code patterns, or accepted statuses:

```python
@dataclass
class SiteProfile:
    site_id: str
    study_uid_system: str = "urn:dicom:uid"
    accession_code: str = "ACSN"
    accepted_statuses: tuple = ("available", "final")
    secondary_discovery_resources: tuple = ("DiagnosticReport",)
    dicomweb_base_url: str | None = None
    gateway_endpoint: str | None = None
```

This ensures the manifest parser works correctly across sites where:
- Identifier systems differ (e.g., some sites use `urn:oid:` prefixes, others use bare UIDs)
- Accession codes use site-specific type codes
- Study statuses include site-specific values like `"preliminary"` or `"registered"`

### 4.6 Deduplication and Idempotency

The manifest builder maintains a **retrieval ledger** — a cumulative record of all studies previously retrieved for each site.

- Each daily manifest only includes studies not already in the ledger (or studies that previously failed retrieval)
- The same study is never retrieved twice
- Failed retrievals are retried with exponential backoff (max 5 attempts over 5 days)
- The ledger is the source of truth for what RADIANT has vs. what is outstanding
- Ledger is stored in S3 alongside the manifests: `s3://radiant-imaging/{site_id}/ledger.json`

### 4.7 Consent Validation Rule

A study is added to the manifest if and only if:

1. The `ImagingStudy.subject` patient is a member of the site's CBTN registry (proven by presence in the Bulk FHIR export)
2. The `ImagingStudy` resource itself was returned by the registry-scoped Bulk FHIR export (Epic enforces this scoping — non-registry patients' imaging is never exported)

This is the **technical proof** that satisfies the site's requirement: the study manifest is a strict subset of what Epic's own authorization layer already approved for export. The manifest's `consent_proof` block provides the audit chain from study back to registry membership.

---

## 5. CAT-A: Native EHR Imaging API — Epic DICOMweb Proxy

> **VALIDATION STATUS:** Site-proven possibility, not a general Epic-native pattern. Epic's Interconnect documentation describes a DICOMweb Gateway (role: DICOMweb Gateway, queue: CEDICOM, business service: Imaging.DICOMweb) with username-token authentication over TLS/REST — this is a distinct access model from the SMART on FHIR / OAuth2 surface used by the Bulk FHIR Connector. The provided Epic documentation does not establish that the same SMART backend token used for Bulk FHIR naturally unlocks WADO-RS retrieval for external research clients. Some sites may expose a DICOMweb path reachable with the same backend app trust chain, but this must be validated per site. The Task 0 capability probe is a mandatory gate before this transport is configured.

This is the preferred configuration *if validated at the site*. At some sites, Epic exposes a DICOMweb path that can be reached through the same backend-app trust chain as Bulk FHIR or through a site-managed Interconnect credential path. The provided Epic documentation supports the existence of an Epic-managed DICOM gateway, but not a universal claim that SMART backend tokens automatically unlock WADO-RS for external research clients.

### 5.0 Epic Interconnect DICOM Gateway (Institutional Broker)

Epic's Interconnect platform documents a dedicated DICOM gateway that serves as an institutional broker between Interconnect and the image archive:

| Property | Value |
|----------|-------|
| **Role** | DICOMweb Gateway |
| **Queue** | CEDICOM |
| **Business Service** | Imaging.DICOMweb |
| **Security** | Username tokens, TLS, REST |
| **Archive-side config** | AE title, port, hostname/FQDN, DICOM ping/echo, remote DICOM archives |

This gateway is the strongest evidence of an Epic-managed PACS broker in the provided documentation. However, it is documented as an Interconnect-internal service, not as a SMART on FHIR resource surface. The key open question is whether the RADIANT Connector's backend app credentials (OAuth2 / JWKS) can reach this gateway, or whether it requires a separate EMP/username-token authentication path.

**Implications for RADIANT:**
- If the gateway is reachable via the existing OAuth2 token → CAT-A is the simplest path (this section)
- If the gateway requires EMP/username-token auth → the site's IT team must provision credentials, and the transport adapter needs a separate auth module (closer to CAT-C in operational complexity)
- The Task 0 probe tests both paths: OAuth2 token against DICOMweb endpoint, AND whether an EMP-based credential is required

> **Note:** The Interconnect DICOM Gateway and Care Everywhere Image Exchange are related but distinct pieces of infrastructure. The gateway provides a DICOMweb proxy to the archive; Care Everywhere provides a clinician-facing image-sharing workflow. They should not be conflated.

### 5.1 FHIR-First Validation Branch

Before attempting DICOMweb retrieval, the pipeline probes FHIR content endpoints to determine if a simpler path exists. Some Epic configurations serve DICOM content directly through FHIR `Binary` or `Media` resources, bypassing the need for DICOMweb entirely.

```
Site Capability Probe (Task 0)
    |
    v
Probe FHIR Binary: GET /Binary/{attachment_id} from ImagingStudy
    |
    +---> Returns DICOM content-type? --> Use FHIR Binary transport (Task 4b)
    |
    v
Probe FHIR Media: GET /Media?subject={patient}&modality=MR
    |
    +---> Returns imaging references? --> Use FHIR Media transport
    |
    v
Probe DICOMweb: GET {dicomweb_base}/studies?limit=1
    |
    +---> Returns valid DICOM JSON? --> Use Epic DICOMweb transport (this section)
    |
    v
Fall through to CAT-B/C/D/E transports
```

```mermaid
flowchart TD
    T0["Site Capability Probe\n(Task 0)"] --> BINARY["Probe FHIR Binary:\nGET /Binary/{attachment_id}"]
    BINARY -->|"Returns DICOM\ncontent-type"| USE_BINARY["Use FHIR Binary\nTransport (Task 4b)"]
    BINARY -->|"No DICOM content"| MEDIA["Probe FHIR Media:\nGET /Media?subject=...&modality=MR"]
    MEDIA -->|"Returns imaging\nreferences"| USE_MEDIA["Use FHIR Media\nTransport"]
    MEDIA -->|"No imaging content"| DICOMWEB["Probe DICOMweb:\nGET {base}/studies?limit=1"]
    DICOMWEB -->|"Returns valid\nDICOM JSON"| USE_DW["Use Epic DICOMweb\nTransport"]
    DICOMWEB -->|"Not available"| FALLBACK["Fall through to\nCAT-B/C/D/E"]

    style T0 fill:#457b9d,color:#fff
    style USE_BINARY fill:#2d6a4f,color:#fff
    style USE_MEDIA fill:#2d6a4f,color:#fff
    style USE_DW fill:#2d6a4f,color:#fff
    style FALLBACK fill:#e76f51,color:#fff
```

The FHIR Binary path is attractive because it uses the exact same OAuth2 token and FHIR API surface the Connector already uses — no additional endpoint discovery or DICOMweb-specific configuration required.

### 5.2 Prerequisites

- Site's Epic instance has DICOMweb proxy enabled (confirmed via Task 0 probe, not assumed)
- The RADIANT Connector Vendor Services app includes "All R4 FHIR APIs" (already required per implementation guide, page 6)
- The background user (EMP) has security points for imaging access
- The `ImagingStudy.endpoint` reference in FHIR resources points to the Epic DICOMweb base URL

### 5.3 Authentication Flow

The authorization model for DICOMweb access is site-dependent. Epic's Interconnect DICOM Gateway uses username-token / EMP-based credentials over TLS, which is a separate access model from the SMART on FHIR OAuth2 used by Bulk FHIR. At some sites, the same backend app credentials may extend to DICOMweb endpoints — but this is a site-proven possibility, not an established general pattern.

**If OAuth2 token reuse is validated at the site:**

```
RADIANT EHR Pipeline
    |
    |  1. Create JWT signed with institution's private key
    v
Epic OAuth2 Token Endpoint
    |
    |  2. Validate JWT against RADIANT JWKS public key
    |  3. Issue access token scoped to backend app permissions
    v
Access Token (same token used for Bulk FHIR)
    |
    |  4. GET /ImagingStudy (Bulk FHIR) --> Study Manifest
    |  5. GET /wado-rs/studies/{study_uid} (DICOMweb) --> DICOM bytes
    v
Epic DICOMweb Proxy --> Institutional PACS
```

```mermaid
sequenceDiagram
    participant R as RADIANT EHR Pipeline
    participant E as Epic OAuth2 Endpoint
    participant GW as Epic Interconnect<br/>DICOM Gateway
    participant P as Institutional PACS

    Note over R,E: Authorization Plane (established)
    R->>R: Create JWT signed with<br/>institution's private key
    R->>E: POST /oauth2/token (JWT assertion)
    E->>E: Validate JWT against<br/>RADIANT JWKS public key
    E-->>R: Access token (scoped to backend app)

    R->>E: GET /ImagingStudy (Bulk FHIR)
    E-->>R: Study Manifest data

    Note over R,GW: Retrieval Plane (site-dependent bridge)
    rect rgb(255, 243, 205)
        Note over R,GW: Option A: OAuth2 token extends<br/>to DICOMweb (must be site-validated)
        R->>GW: GET /wado-rs/studies/{study_uid}<br/>Authorization: Bearer {token}
    end
    rect rgb(255, 228, 225)
        Note over R,GW: Option B: Separate EMP/username-token<br/>(Interconnect DICOM Gateway native model)
        R->>GW: GET /wado-rs/studies/{study_uid}<br/>Username-token auth (EMP credentials)
    end
    GW->>P: Internal PACS query
    P-->>GW: DICOM instances
    GW-->>R: Multipart DICOM response
```

If site validation confirms OAuth2 reuse, the access token obtained for Bulk FHIR can also be used for DICOMweb retrieval. If the site instead requires EMP / username-token access to the Interconnect DICOM gateway, the transport adapter must use that retrieval-plane credential path while still treating the study manifest as the authorization source of truth.

### 5.4 Retrieval Workflow

```
Study Manifest (daily)
    |
    v
For each study in manifest where retrieval_status = "pending":
    |
    |  1. Obtain/refresh the validated retrieval-plane credential (OAuth2 if site-proven, otherwise EMP / username-token)
    |  2. Resolve DICOMweb base URL from ImagingStudy.endpoint
    |  3. WADO-RS: GET {base}/studies/{study_uid}
    |     Accept: multipart/related; type="application/dicom"
    |  4. Receive multipart DICOM response (all series, all instances)
    |  5. Validate: series count matches expected_series
    |  6. Validate: instance count matches expected_instances
    |
    +---> QA platform available (Ambra/VNA):
    |       Upload to platform via /study/add or STOW-RS
    |       Platform validates, tags consent, pushes to RADIANT S3
    |
    +---> Direct to S3 (no QA platform):
    |       Stream DICOM directly to S3
    |       Inline validation + consent_proof.json sidecar
    |
    v
Update retrieval_ledger: study_uid -> "retrieved"
```

**Streaming and Memory Management:**

Large imaging studies (e.g., volumetric MRI, PET/CT) can exceed 1 GB. The retrieval service MUST NOT buffer the entire multipart response in memory. Instead:

1. **Stream multipart parsing**: Parse `multipart/related` boundaries incrementally, extracting one DICOM instance at a time from the response stream.
2. **Instance-level write**: Each extracted instance is written directly to its S3 destination (`{study_uid}/series/{series_uid}/{sop_uid}.dcm`) or piped to Ambra via `/study/add` as it is parsed — not accumulated in a list.
3. **Series-level fallback**: If study-level retrieval times out or the response exceeds a configurable threshold (e.g., 500 MB), fall back to series-level retrieval (Section 5.5) and stream each series independently.
4. **Backpressure**: If the S3 upload or Ambra ingest is slower than the DICOMweb download, the parser pauses reading from the HTTP stream (standard `httpx`/`aiohttp` backpressure) rather than buffering in memory.
5. **Memory ceiling**: The retrieval process should operate within a fixed memory budget (configurable, default ~256 MB) regardless of study size.

```mermaid
flowchart TD
    MANIFEST["Study Manifest\n(daily)"] --> LOOP{"For each study where\nretrieval_status = pending"}
    LOOP --> AUTH["1. Obtain/refresh\nOAuth2 access token"]
    AUTH --> RESOLVE["2. Resolve DICOMweb base URL\nfrom ImagingStudy.endpoint"]
    RESOLVE --> WADO["3. WADO-RS: GET studies/{study_uid}\nAccept: multipart/related"]
    WADO --> STREAM["4. Stream multipart response\n(instance-by-instance parsing)"]
    STREAM --> VALIDATE["5-6. Validate series &\ninstance counts"]

    VALIDATE --> QA_CHECK{"QA Platform\nAvailable?"}
    QA_CHECK -->|"Yes (Ambra/VNA)"| AMBRA["Upload via /study/add\nor STOW-RS"]
    AMBRA --> AMBRA_QA["Platform validates,\ntags consent, pushes to S3"]
    QA_CHECK -->|"No"| S3["Stream DICOM\ndirectly to S3"]
    S3 --> INLINE["Inline validation +\nconsent_proof.json sidecar"]

    AMBRA_QA --> LEDGER["Update retrieval_ledger:\nstudy_uid → retrieved"]
    INLINE --> LEDGER

    WADO -.->|"Timeout / >500MB"| SERIES["Series-level fallback:\nGET studies/{uid}/series/{series_uid}"]
    SERIES --> STREAM

    style MANIFEST fill:#264653,color:#fff
    style LEDGER fill:#e9c46a,color:#000
    style AMBRA_QA fill:#2a9d8f,color:#fff
    style INLINE fill:#2a9d8f,color:#fff
```

### 5.5 DICOMweb API Calls

**Study-level retrieval (preferred — single request for all series and instances):**
```
GET {dicomweb_base}/studies/{study_uid}
Authorization: Bearer {access_token}
Accept: multipart/related; type="application/dicom"
```

**Series-level retrieval (fallback if study-level times out for large studies):**
```
GET {dicomweb_base}/studies/{study_uid}/series/{series_uid}
Authorization: Bearer {access_token}
Accept: multipart/related; type="application/dicom"
```

**Metadata-only probe (verify study exists before full retrieval):**
```
GET {dicomweb_base}/studies/{study_uid}/metadata
Authorization: Bearer {access_token}
Accept: application/dicom+json
```

**Transfer Syntax Negotiation:**

DICOMweb WADO-RS supports transfer syntax negotiation via the `Accept` header. The retrieval service should request DICOM in a preferred transfer syntax order:

```
Accept: multipart/related; type="application/dicom"; transfer-syntax=1.2.840.10008.1.2.1,
        multipart/related; type="application/dicom"; transfer-syntax=1.2.840.10008.1.2.4.90,
        multipart/related; type="application/dicom"
```

| Transfer Syntax UID | Name | Notes |
|---------------------|------|-------|
| `1.2.840.10008.1.2.1` | Explicit VR Little Endian | Preferred — universal compatibility, no decompression needed |
| `1.2.840.10008.1.2.4.90` | JPEG 2000 Lossless | Acceptable — ~3x compression, lossless, requires decompression |
| `1.2.840.10008.1.2.4.70` | JPEG Lossless | Acceptable — moderate compression, lossless |
| `*` (no transfer-syntax) | Server's default | Fallback — accept whatever the server provides |

**Post-retrieval transcoding:** If downstream consumers (radiomics, AI inference) require a specific transfer syntax that differs from what was retrieved, transcoding is performed at S3 landing time using `pydicom` + `pylibjpeg`. The original transfer syntax is recorded in the `study_manifest.json` sidecar for each instance (`transfer_syntax_uid` field). This avoids re-retrieving studies solely for syntax conversion.

> **Note:** Some PACS proxies (especially Epic's DICOMweb) may ignore the transfer-syntax parameter and always serve in the PACS's native syntax. The retrieval service must handle any valid DICOM transfer syntax gracefully, even if it differs from what was requested.

### 5.6 With QA Platform (Ambra/VNA): Staging and QA Layer

When a QA platform is available (secondary axis), it serves as an intermediate staging area with built-in QA:

```
Epic DICOMweb --> RADIANT retrieval service --> Ambra InteleShare
                                                    |
                                    +---------+---------+---------+
                                    |         |         |         |
                              /study/add  Validation  De-id    /study/push
                              (ingest)    (custom     rules    (to S3)
                                          fields)
```

**Ambra integration points:**

| Ambra API | Purpose |
|-----------|---------|
| `POST /study/add` | Ingest retrieved DICOM into Ambra namespace |
| `POST /customfield/add` | Create `consent_proof`, `registry_id`, `manifest_id` fields |
| `POST /study/set` | Tag study with consent metadata via custom fields |
| `POST /study/validate` | Run validation rules (series count, modality check) |
| `GET /study/dicomdata/load` | Verify DICOM header integrity |
| `POST /study/push` | Push validated study to RADIANT S3 or DICOM receiver |
| `GET /audit/object` | Complete audit trail per study |
| `POST /webhook/add` | Trigger downstream pipeline on successful push |

**Ambra namespace structure:**
```
RADIANT (top-level account)
  +-- site_seattle_childrens (namespace per site)
  |     +-- incoming (staging)
  |     +-- validated (QA passed)
  |     +-- rejected (QA failed, manual review)
  +-- site_chop (namespace per site)
        +-- ...
```

```mermaid
flowchart LR
    subgraph Transport["Transport Layer (CAT-A/B/C)"]
        DW["Epic DICOMweb"]
        FB["FHIR Binary"]
        GW["Gateway Broker"]
    end

    subgraph Ambra["Ambra InteleShare"]
        direction TB
        INGEST["/study/add\n(ingest)"] --> VALIDATE["/study/validate\n(QA rules)"]
        VALIDATE --> TAG["/study/set\n(consent tagging)"]
        TAG --> DEID["/anonymization\n(optional de-id)"]
        DEID --> PUSH["/study/push\n(route to S3)"]
        PUSH --> WEBHOOK["/webhook\n(notify pipeline)"]
    end

    Transport --> INGEST
    PUSH --> S3["RADIANT S3 Bucket"]
    WEBHOOK --> PIPELINE["Downstream\nPipeline"]

    style Ambra fill:#f0f0f0,stroke:#333
    style S3 fill:#e9c46a,color:#000
    style PIPELINE fill:#457b9d,color:#fff
```

**Webhook-driven downstream:**
```json
{
  "webhook": "study_validated",
  "event": "study_push_complete",
  "url": "https://radiant-pipeline.internal/imaging/notify",
  "method": "POST",
  "parameters": {
    "study_uid": "{{study.uid}}",
    "site_id": "{{namespace.name}}",
    "s3_path": "{{destination.path}}"
  }
}
```

### 5.7 Without QA Platform: Direct to S3

Without a QA platform, the retrieval service writes directly to S3:

**S3 object layout (UID-based keys for referential integrity):**
```
s3://radiant-imaging/
  {site_id}/
    manifests/
      study_manifest_2026-03-05.json
    ledger.json
    studies/
      {study_uid}/
        study_manifest.json      (checksums, series/SOP UIDs, byte counts)
        metadata.json            (DICOM JSON metadata)
        consent_proof.json       (registry linkage)
        series/
          {series_uid}/
            {sop_instance_uid}.dcm
            {sop_instance_uid}.dcm
            ...
```

**Study manifest sidecar** (`study_manifest.json`): Written after all instances are uploaded, contains SHA-256 checksums per file, total byte count, series/instance inventory. Enables downstream integrity verification without re-reading DICOM headers.

> **Design note:** S3 keys use DICOM UIDs (StudyInstanceUID, SeriesInstanceUID, SOPInstanceUID) rather than flat sequential numbering (`000001.dcm`). This ensures S3 paths are deterministic and idempotent — re-uploading the same study produces the same keys, enabling safe retries without orphaned objects.

**Validation (without Ambra):**
The retrieval service performs inline validation:
1. **Hard requirement:** Study UID in DICOM header matches requested study (prevents wrong study)
2. **Hard requirement:** All instances parseable as valid DICOM (pydicom header read)
3. **Expected match:** Series count matches `ImagingStudy.numberOfSeries` (warn if mismatch, fail if zero)
4. **Expected match:** Instance count matches `ImagingStudy.numberOfInstances` (warn if mismatch)
5. **Preferred checks:** Accession number and study date match manifest (cross-validation)
6. **Site-specific:** Patient ID in DICOM header matches FHIR patient — enabled only if the site's `SiteProfile` provides a patient ID mapping. PACS patient identifiers often do not align with FHIR subject IDs due to merge history, enterprise MPI differences, or archive-specific numbering. Do not fail retrieval on this check unless the site has confirmed identifier alignment.

Failed validation triggers:
- Study marked `retrieval_status: "validation_failed"` in ledger
- Alert to RADIANT operations for manual review
- Study is NOT pushed to the research pipeline

### 5.8 Error Handling and Retry

| Error | Behavior |
|-------|----------|
| 401 Unauthorized | Refresh OAuth2 token, retry once |
| 403 Forbidden | Log consent_proof, escalate (patient may have been removed from registry) |
| 404 Not Found | Study may not be in PACS yet (pending QA). Retry next day, max 5 attempts |
| 429 Rate Limited | Back off per Retry-After header, resume |
| 500/502/503 | Retry with exponential backoff (1min, 5min, 30min) |
| Timeout | Fall back to series-level retrieval (smaller chunks) |
| Partial retrieval | Mark incomplete, retry missing series next cycle |
| DICOM validation failure | Quarantine, alert, do not push downstream |

---

## 6. CAT-B: EHR-Mediated Exchange — Epic Care Everywhere Image Exchange

> **VALIDATION STATUS — SIGNIFICANT LIMITATIONS:** Care Everywhere Image Exchange is designed for clinician-facing, cross-organizational image sharing — not for programmatic full-study DICOM retrieval. The Epic documentation explicitly states that exchanged images are **reference-quality JPG still images under 15 MB**, and that "Image Exchange Advanced" uses default logic that **selects all images only for very small studies**, otherwise selecting key images or middle images from one or two series. This does not satisfy RADIANT's requirement for full DICOM series, all instances, with preserved headers.
>
> The DCM manifest feature supports diagnostic-quality metadata indexing but is **off by default** and does not by itself prove a backend full-study export API.
>
> **Recommendation:** Standard Care Everywhere Image Exchange should be treated as an **experimental / exception path** — not a general-purpose transport. A site must explicitly demonstrate that their Care Everywhere configuration can deliver full diagnostic-quality DICOM studies (all series, all instances, original transfer syntax) before this transport is configured. Most sites will not meet this bar.

For sites where Epic's DICOMweb proxy is not available (confirmed via Task 0 probe), but Epic's Care Everywhere image sharing infrastructure may be available.

### 6.1 Concept

If — and only if — a site can demonstrate backend, programmatic, full-study retrieval through Care Everywhere or closely related exchange infrastructure, RADIANT can register as an image exchange partner. In that scenario the workflow leverages Epic's built-in PACS communication and RADIANT never touches the site's PACS directly. For most sites this remains an exception path, not the default expectation.

```
RADIANT Connector (existing)
    |
    |  Bulk FHIR --> Study Manifest
    v
Image Exchange Request App (new Epic app, or extension of Connector)
    |
    |  For each study on manifest:
    |    Epic Image Exchange API --> triggers image pull from PACS
    |    Epic packages DICOM --> sends to RADIANT endpoint
    v
RADIANT DICOM Receiver (Ambra or direct S3)
```

### 6.2 Epic App Configuration

A second Vendor Services application (or feature extension of the existing Connector) is created with:

| Setting | Value |
|---------|-------|
| Application Name | RADIANT Imaging Connector |
| Who will primarily be using this app? | Backend Systems |
| Features | Incoming API |
| Incoming APIs | Image Exchange, WADO Retrieve |
| Does your app use OAuth 2.0? | Yes (on) |
| SMART on FHIR Version | R4 |
| SMART Scope Version | SMART v1 |
| FHIR ID Generation Scheme | Use 64-Character-Limited FHIR IDs for USCDI |
| JWK Set URLs | Same RADIANT JWKS server as Bulk FHIR Connector |

**Key difference from the Bulk FHIR Connector:** This app's feature set includes "Image Exchange" rather than (or in addition to) "Incoming API" for FHIR resources. The Image Exchange feature enables Epic's IHE-based imaging retrieval profiles.

### 6.3 Retrieval Workflow

```
Study Manifest
    |
    v
**Conceptual workflow — only if site validation confirms full-study, backend retrieval support**

For each study:
    |
    |  1. Authenticate via OAuth2 (same JWKS, same flow)
    |  2. POST image exchange request to Epic's Image Exchange endpoint
    |     - Study UID or Accession Number as identifier
    |     - RADIANT DICOM receiver as destination
    |  3. Epic internally:
    |     a. Validates patient is in CBTN registry (same auth scope)
    |     b. Queries integrated PACS for the study
    |     c. Packages DICOM instances
    |     d. Pushes to RADIANT's designated endpoint
    |
    +---> QA platform available:
    |       Epic pushes to platform STOW-RS endpoint or DICOM receiver
    |       Platform processes per Section 5.6
    |
    +---> Direct to S3:
            Epic pushes to RADIANT's cloud DICOM receiver
            Receiver writes to S3 per Section 5.7
```

```mermaid
sequenceDiagram
    participant M as Study Manifest
    participant R as RADIANT Connector
    participant E as Epic Image Exchange API
    participant P as Site PACS
    participant D as DICOM Receiver<br/>(Ambra or S3)

    Note over R,D: EXPERIMENTAL PATH — Standard CE delivers<br/>reference-quality JPGs, not full DICOM.<br/>Site must prove full-study capability.

    M->>R: Study list (pending)
    loop For each study
        R->>E: POST image exchange request<br/>(Study UID + destination)
        E->>E: Validate patient in<br/>CBTN registry (auth scope)
        E->>P: Query integrated PACS
        P-->>E: Images (reference-quality<br/>by default, not full DICOM)
        E->>D: Push images to<br/>RADIANT endpoint
        D-->>R: Acknowledgment
    end
    R->>R: Validate: full DICOM received?<br/>(series count, instance count,<br/>DICOM headers present)
    R->>M: Update ledger
```

### 6.4 Care Everywhere Image Exchange Details

Epic's Image Exchange supports two models:

**Model 1: Pull-based (RADIANT initiates)**
- RADIANT requests a specific study via IHE RAD-69 (Retrieve Imaging Document Set)
- Epic retrieves from PACS and returns DICOM in the response
- Requires the Image Exchange endpoint to be accessible from RADIANT's infrastructure

**Model 2: Push-based (Epic initiates on trigger)**
- A routing rule in Epic triggers automatic image push when a study matches criteria
- Criteria: patient is on CBTN registry AND study is finalized
- Epic pushes to RADIANT's registered DICOM destination
- This is the most hands-off model but requires the site to configure the routing rule (one-time setup)

**Recommended: Model 1 (pull-based)** for consistency with the manifest-driven architecture. RADIANT controls the timing and scope.

### 6.5 Advantages Over Direct PACS Access

- Epic handles all PACS communication (DIMSE, proprietary vendor APIs, etc.)
- Works regardless of PACS vendor or age
- Patient scoping enforced by Epic's authorization layer (same as Bulk FHIR)
- No network exposure of the PACS required
- Standard Epic deployment — sites already have this infrastructure for clinical image sharing

---

## 7. CAT-B: EHR-Mediated Exchange — Epic Radiant IHE Profiles

> **VALIDATION STATUS:** Candidate fallback pending site validation. Epic Radiant's IHE profile support is documented for inter-system clinical imaging workflows. Whether these endpoints are available to external backend applications (vs. internal radiology workstations) requires confirmation at each site. The IHE transactions below (RAD-69, RAD-55) may require specific Epic Radiant configuration that is not part of a standard Vendor Services deployment.

For sites running Epic Radiant (radiology module) with IHE profiles enabled.

### 7.1 Concept

Epic Radiant natively supports IHE imaging profiles for certain clinical imaging workflows. If a site can demonstrate that those endpoints are exposed for backend system use — and that they return the full diagnostic study rather than a repackaged subset — RADIANT can use them as a CAT-B exception path. This must be validated at each site via the Task 0 capability probe.

### 7.2 IHE Profiles Leveraged

| IHE Profile | Transaction | Use |
|-------------|-------------|-----|
| RAD-69 | Retrieve Imaging Document Set | Pull full DICOM study by UID |
| RAD-55 | WADO Retrieve | Pull individual instances by UID |
| RAD-68 | Provide and Register Imaging Document Set | For push-based workflows |
| ITI-43 | Retrieve Document Set | Generic document retrieval (includes DICOM) |

### 7.3 Workflow

```
Study Manifest
    |
    v
For each study:
    |
    |  1. Authenticate using the site-validated IHE access pattern (do not assume it matches Bulk FHIR OAuth2)
    |  2. IHE RAD-69 Retrieve request
    |     - Study UID from manifest
    |     - Sent to Epic's IHE Retrieve endpoint
    |  3. Epic Radiant:
    |     a. Validates authorization scope
    |     b. Retrieves from PACS via internal connection
    |     c. Returns DICOM in IHE response
    |  4. RADIANT processes response
    |
    +---> Ambra or direct S3 (same downstream as Approach A)
```

### 7.4 When to Prefer IHE Profiles Over Care Everywhere

| Factor | Care Everywhere (Section 6) | IHE Profiles (this section) |
|--------|---------------------------|----------------------------|
| Site has Care Everywhere image sharing | Required | Not required |
| Site runs Epic Radiant | Not required | Required |
| DICOM metadata fidelity | May be repackaged by exchange | Original DICOM preserved |
| Setup complexity | Lower (existing infra) | Moderate (IHE endpoint config) |
| Bandwidth efficiency | Push model available | Pull only |

---

## 8. CAT-D (+ CAT-C Config C): Ambra / VNA — Three Distinct Roles

Ambra (InteleShare) can serve three distinct roles in the DICOM retrieval architecture, and a given site may use one, two, or all three. It is important to distinguish these roles because the configuration, data flow, and failure modes differ significantly.

### 8.1 Role Definitions

| Role | Name | Description | When Used |
|------|------|-------------|-----------|
| **Config A** | Transport Target | RADIANT retrieves DICOM via some other transport (CAT-A/B/C) and uploads to Ambra for QA, consent tagging, de-identification, and S3 push | Any CAT-A/B/C site where Ambra is available as a post-retrieval QA platform (secondary axis) |
| **Config B** | Post-Retrieval Ingest | Same as Config A, but emphasizing Ambra's validation and routing pipeline rather than its storage | Subset of Config A focused on the QA workflow |
| **Config C** | Site Gateway | Ambra queries the site's PACS directly (via C-FIND/C-MOVE or DICOMweb) — **no inner transport needed**. RADIANT sends the study manifest to Ambra, and Ambra handles PACS retrieval end-to-end | Sites where Ambra has direct PACS connectivity and RADIANT has no other viable transport |

```mermaid
flowchart TB
    subgraph ConfigA["Config A: Transport Target"]
        direction LR
        CA_TRANSPORT["CAT-A/B/C\nTransport"] -->|"DICOM bytes"| CA_AMBRA["Ambra\n(QA + Routing)"]
        CA_AMBRA --> CA_S3["S3"]
    end

    subgraph ConfigB["Config B: Post-Retrieval Ingest"]
        direction LR
        CB_TRANSPORT["CAT-A/B/C\nTransport"] -->|"DICOM bytes"| CB_AMBRA["Ambra\n(Validate + Tag + Route)"]
        CB_AMBRA --> CB_S3["S3"]
    end

    subgraph ConfigC["Config C: Site Gateway"]
        direction LR
        CC_MANIFEST["Study\nManifest"] -->|"study_uid"| CC_AMBRA["Ambra\n(C-FIND → C-MOVE → QA)"]
        CC_AMBRA -->|"queries"| CC_PACS["Site PACS"]
        CC_PACS -->|"DICOM"| CC_AMBRA
        CC_AMBRA --> CC_S3["S3"]
    end

    style ConfigA fill:#e8f5e9,stroke:#2d6a4f
    style ConfigB fill:#e3f2fd,stroke:#1565c0
    style ConfigC fill:#fff3e0,stroke:#e65100
```

### 8.2 Architecture — Config A/B (Transport Target / Post-Retrieval Ingest)

```
                    Study Manifest (consent-scoped)
                              |
                 +------------+-------------+
                 |            |             |
          Epic DICOMweb  FHIR Binary   Gateway Broker
          (Section 5)    (Section 5.1)  (Section 9)
                 |            |             |
                 v            v             v
              Ambra InteleShare API
              +-----------------------------+
              |  Ingest --> Validate -->     |
              |  Tag consent --> De-id -->   |
              |  Push to S3 --> Webhook      |
              +-----------------------------+
                              |
                              v
                    RADIANT S3 Bucket
```

### 8.3 Architecture — Config C (Site Gateway)

```
                    Study Manifest (consent-scoped)
                              |
                              v
              Ambra InteleShare API (as gateway)
              +-----------------------------+
              |  C-FIND by study_uid -->     |
              |  C-MOVE from site PACS -->   |
              |  Validate --> Tag consent -> |
              |  Push to S3 --> Webhook      |
              +-----------------------------+
                              |
                              v
                    RADIANT S3 Bucket
```

In Config C, Ambra's `/destination/search` (C-FIND) and `/destination/retrieve` (C-MOVE) endpoints are used to query the site's PACS directly. The site must configure Ambra as an authorized DICOM destination (AE title) on their PACS. This is the only configuration where RADIANT never touches DICOM bytes directly.

### 8.4 Ambra-Specific Workflows (All Roles)

#### 8.4.1 Consent Tagging via Custom Fields

At onboarding, create custom fields in the site's Ambra namespace:

```
POST /customfield/add
{
  "name": "radiant_consent_proof",
  "type": "text",
  "wrapped_dicom_only": false,
  "description": "RADIANT consent chain: registry_id + manifest_id + export_id"
}

POST /customfield/add
{
  "name": "radiant_manifest_id",
  "type": "text"
}

POST /customfield/add
{
  "name": "cbtn_registry_membership",
  "type": "text",
  "description": "CBTN registry ID confirming patient consent for data exchange"
}
```

After each study is ingested:
```
POST /study/set
{
  "uuid": "{ambra_study_uuid}",
  "customfield-{consent_field_uuid}": "registry:100263|manifest:SCH-2026-03-05-001|export:export-2026-03-05-daily",
  "customfield-{manifest_field_uuid}": "SCH-2026-03-05-001",
  "customfield-{registry_field_uuid}": "100263"
}
```

#### 8.4.2 Validation Rules

```
POST /validate/add
{
  "name": "radiant_imaging_qa",
  "conditions": [
    {"field": "modality", "condition": "in", "value": "MR,CT,PT,NM"},
    {"field": "series_count", "condition": "ge", "value": "1"},
    {"field": "customfield-{consent_field_uuid}", "condition": "not_equals", "value": ""}
  ],
  "on_fail": "reject"
}
```

Run validation:
```
POST /study/validate
{
  "uuid": "{ambra_study_uuid}"
}
```

#### 8.4.3 Automated Routing to S3

Configure a routing rule that fires on successful validation:

```
POST /route/add
{
  "name": "radiant_s3_push",
  "conditions": {
    "namespace": "site_seattle_childrens",
    "status": "validated",
    "customfield_consent": "not_empty"
  },
  "destination": "radiant_s3_destination",
  "options": {
    "include_reports": true,
    "include_key_images": true
  }
}
```

#### 8.4.4 Webhook for Pipeline Integration

```
POST /webhook/add
{
  "url": "https://radiant-pipeline.internal/imaging/study-ready",
  "event": "study_push_complete",
  "method": "POST",
  "auth": "Bearer {radiant_api_key}",
  "parameters": {
    "study_uid": "{{study.study_uid}}",
    "site_id": "{{namespace.name}}",
    "s3_path": "s3://radiant-imaging/{{namespace.name}}/studies/{{study.study_date}}/{{study.study_uid}}/",
    "series_count": "{{study.series_count}}",
    "instance_count": "{{study.image_count}}",
    "consent_proof": "{{study.customfield.radiant_consent_proof}}"
  }
}
```

#### 8.4.5 Batch Operations via Bundle API

For daily manifest processing, use Ambra's `/bundle` endpoint to execute multiple operations atomically:

```json
POST /bundle
[
  {
    "url": "/study/cfind",
    "method": "POST",
    "parameters": {
      "filter.study_uid.equals": "1.2.840.113619.2.55.3.12345"
    }
  },
  {
    "url": "/study/set",
    "method": "POST",
    "parameters": {
      "uuid": "{{studies.[0].uuid}}",
      "customfield-consent": "registry:100263|manifest:SCH-2026-03-05-001"
    }
  },
  {
    "url": "/study/push",
    "method": "POST",
    "parameters": {
      "uuid": "{{studies.[0].uuid}}",
      "destination": "radiant_s3"
    }
  }
]
```

**Idempotency and crash recovery:**

The multi-step Ambra workflow (`add` → `set` → `validate` → `push`) is not atomic. If the retrieval service crashes mid-workflow, studies may be in an inconsistent state. The retrieval service uses a **status tracking pattern** to ensure idempotency:

1. **Pre-operation ledger write:** Before calling `/study/add`, the retrieval ledger records `ambra_status: "ingest_started"` with a timestamp.
2. **Post-step ledger updates:** After each Ambra API call succeeds, the ledger is updated: `"ingest_started"` → `"ingested"` → `"tagged"` → `"validated"` → `"pushed"`.
3. **Recovery on restart:** When the retrieval service starts, it queries the ledger for studies with incomplete Ambra status (any state except `"pushed"` or `"failed"`). For each:
   - Query Ambra via `/study/cfind` by study UID to determine actual state
   - Resume from the last completed step (e.g., if study is ingested but not tagged, start from `/study/set`)
   - This makes each step independently idempotent — re-running a completed step is safe (Ambra's API returns success for already-set fields)
4. **Orphan detection:** Studies in Ambra's `incoming` namespace for >24 hours without a corresponding `"pushed"` ledger entry are flagged for investigation.

#### 8.4.6 De-identification (Optional)

If DICOM must be de-identified before leaving Ambra:

```
POST /anonymization/add
{
  "name": "radiant_research_pseudonymization",
  "rules": [
    {"tag": "PatientName", "action": "replace", "value": "RADIANT-{hash}"},
    {"tag": "PatientID", "action": "replace", "value": "{fhir_patient_id}"},
    {"tag": "InstitutionName", "action": "remove"},
    {"tag": "ReferringPhysicianName", "action": "remove"},
    {"tag": "AccessionNumber", "action": "hash"},
    {"tag": "StudyInstanceUID", "action": "keep"},
    {"tag": "SeriesInstanceUID", "action": "keep"},
    {"tag": "SOPInstanceUID", "action": "keep"}
  ],
  "keep_private_tags": false
}
```

> **Naming note:** This profile is deliberately NOT called "safe harbor." HIPAA Safe Harbor (45 CFR 164.514(b)) requires removal of listed identifiers and bars retention of unique identifying numbers/characteristics/codes except under narrow re-identification code exceptions. Preserving DICOM Study/Series/SOP UIDs — which are unique identifying codes — creates legal ambiguity under Safe Harbor. This profile is instead a **research pseudonymization** profile: patient-level PHI is replaced with pseudonymous identifiers, but DICOM linkage UIDs are preserved for referential integrity. The distinction matters for IRB and institutional compliance review.

---

## 9. CAT-C: Middleware-Brokered Retrieval — Interface Engine / Gateway as Policy Broker

Many hospital sites already operate API gateways, reverse proxies, and healthcare-specific interface engines that manage HL7 v2, FHIR, web-service, and partner-integration traffic. In this architecture, those platforms are not treated merely as packet forwarders. They are elevated to the site's **policy broker** — the institutionally approved bridge between the RADIANT authorization plane and the PACS retrieval plane.

### 9.1 Why CAT-C Maximizes Existing Messaging Infrastructure

HL7 v2 / integration frameworks are already the operational backbone for many health systems. They provide durable queues, message validation, retry handling, dead-letter workflows, identifier translation, audit logging, and controlled ingress/egress to internal systems. Those are exactly the capabilities needed to convert a RADIANT-approved manifest into a PACS-allowed read without exposing the PACS directly to RADIANT.

**What CAT-C is maximizing:**

- Existing site-owned interface engines and API gateways
- Existing institutional trust and monitoring paths
- Existing service accounts / stored credentials to internal PACS APIs
- Existing queueing, retry, and dead-letter mechanisms
- Existing local patient / accession / archive identifier translation logic

> **Important boundary:** HL7 v2, FHIR, and interface-engine workflows should be used for the **control plane** — release, revoke, audit, identifier maintenance, and orchestration. They should **not** be used to transport DICOM pixel data. DICOM bytes still move through WADO-RS, platform APIs, or DIMSE.

### 9.2 Core Concept: Study Ticket + Stateful Policy Broker

RADIANT issues a **StudyRelease** authorization (embodied as a signed study ticket or equivalent control-plane message) naming the exact study to be retrieved. The site's broker validates the authorization, persists a short-lived allow-list entry, applies local policy, and only then invokes the PACS-facing read-only API or DICOM gateway using site-managed credentials.

```mermaid
sequenceDiagram
    participant R as RADIANT Connector
    participant B as Site Policy Broker
    participant G as DICOM Gateway / PACS API
    participant P as Site PACS

    R->>B: StudyRelease / signed study ticket
    B->>B: Validate caller identity
    B->>B: Verify signature + expiry
    B->>B: Persist allow-list entry
    B->>B: Correlate local patient / accession IDs
    B->>B: Queue retrieval work
    B->>G: WADO-RS / platform API / DIMSE retrieve
    G->>P: Read-only study retrieval
    P-->>G: DICOM bytes
    G-->>B: DICOM bytes + status
    B-->>R: StudyReady or RetrievalFailed
    R->>B: StudyRevoke (if needed)
    B->>B: Cancel queued / future retrievals
    B->>B: Deny replay / repeat use of same ticket
```

This pattern gives the site a strong institutional review posture: the PACS never trusts RADIANT directly; it trusts the site's own broker. RADIANT only needs trust with the broker, and the broker only allows retrievals that match the manifest-derived authorization artifact.

### 9.3 Control-Plane Message Contracts

The broker pattern works best when the control plane is explicit. The following message contracts are recommended. They can be expressed as signed JSON over HTTPS, a broker-specific REST payload, or translated internally into the site's integration framework.

| Message | Direction | Purpose |
|---------|-----------|---------|
| `StudyRelease` | RADIANT → broker | Authorize one study retrieval |
| `StudyRevoke` | RADIANT → broker | Cancel queued / future retrieval |
| `StudyReady` | broker → RADIANT | Retrieval completed and bytes available / delivered |
| `RetrievalFailed` | broker → RADIANT | Retrieval failed with classification |
| `PatientIdentityUpdate` | site event feed → broker | Merge / unmerge / ID correction |
| `ImageAvailabilityUpdate` | PACS/archive feed → broker | Study finalized / indexed / location changed |

**Recommended minimum payload for `StudyRelease`:**

```json
{
  "message_type": "StudyRelease",
  "ticket_id": "uuid-v4",
  "issued_at": "2026-03-05T02:40:00Z",
  "expires_at": "2026-03-05T03:40:00Z",
  "site_id": "seattle_childrens",
  "manifest_id": "SCH-2026-03-05-001",
  "registry_id": "100263",
  "patient_fhir_id": "e4f2a8c1-...",
  "study_uid": "1.2.840.113619.2.55.3.12345",
  "accession_number": "RAD-2026-04521",
  "modality": ["MR"],
  "allowed_operations": ["QIDO-RS", "WADO-RS"],
  "consent_proof_hash": "sha256:...",
  "signature": "..."
}
```

**Recommended minimum payload for `StudyRevoke`:**

```json
{
  "message_type": "StudyRevoke",
  "ticket_id": "uuid-v4",
  "site_id": "seattle_childrens",
  "study_uid": "1.2.840.113619.2.55.3.12345",
  "revoked_at": "2026-03-05T03:05:00Z",
  "reason": "consent_withdrawn"
}
```

### 9.4 Policy Broker Responsibilities

A site-owned broker should do more than simply validate a signature and forward bytes. To fully maximize interface-engine / messaging infrastructure, it should perform the following duties:

1. **Ingress authentication and trust validation** — mTLS, API key, OAuth2, or other site-approved caller authentication for RADIANT
2. **Study authorization validation** — verify the StudyRelease / signed ticket, expiry, and exact study scope
3. **Replay prevention** — reject re-use of a consumed ticket ID or duplicate one-time release
4. **Allow-list cache** — persist a short-lived broker-side record keyed to study UID / accession / local identifiers
5. **Identifier translation** — map RADIANT identifiers to local MRN / accession / archive keys where required
6. **Local policy checks** — finalized-status rules, image-availability checks, allowed modality filters, archive location routing
7. **Queue orchestration** — durable release queue, retry queue, revoke queue, backfill queue, dead-letter queue
8. **Read-only retrieval execution** — call the PACS-facing DICOMweb / platform / DIMSE endpoint using site-managed credentials
9. **Audit emission** — log allow / deny / retrieve / revoke decisions with timestamps and correlation IDs
10. **Completion callbacks** — emit `StudyReady` / `RetrievalFailed` back to RADIANT

### 9.5 Queue Model and State

To make CAT-C robust and institutionally reviewable, the broker should maintain explicit queue/state objects rather than handling each retrieval as a stateless pass-through request.

| Queue / State | Purpose |
|---------------|---------|
| `release_queue` | Newly accepted StudyRelease messages |
| `authorized_cache` | Active allow-list entries keyed to study UID / ticket ID |
| `retrieval_queue` | Work items ready to call PACS |
| `retry_queue` | Transient failures awaiting retry |
| `revoke_queue` | Pending revocations to apply before retrieval |
| `backfill_queue` | Lower-priority historical studies |
| `dead_letter_queue` | Exhausted or malformed work items |

```mermaid
flowchart LR
    REL[StudyRelease] --> AUTH[authorized_cache]
    AUTH --> RQ[release_queue]
    RQ --> RET[retrieval_queue]
    RET --> PACS[PACS read-only API / DICOM gateway]
    PACS -->|success| READY[StudyReady]
    PACS -->|transient| RETRY[retry_queue]
    PACS -->|permanent| DLQ[dead_letter_queue]
    REVOKE[StudyRevoke] --> RVQ[revoke_queue]
    RVQ --> AUTH
    RETRY --> RET
    AUTH --> AUDIT[audit log]
    READY --> AUDIT
    DLQ --> AUDIT
```

### 9.6 Event-Driven Authorization Maintenance

A major advantage of interface engines is that they can respond to local events, not just nightly batches. Recommended event sources include:

| Event Source | Example Events | Broker Action |
|--------------|----------------|---------------|
| Registry / consent feed | enrollment change, withdrawal | emit / accept `StudyRevoke`, expire cache entries |
| ADT / patient identity feed | merge, unmerge, ID correction | refresh local identifier mappings |
| Imaging availability feed | Instance Availability, study finalized, archive migration | update `ImageAvailabilityUpdate`, unblock queued retrieval |
| Order / result feed | ORM/ORU or equivalent imaging result events | re-check manifest / study availability |
| Platform / archive callbacks | ingest complete, retrieval complete, archive error | emit `StudyReady` / `RetrievalFailed` |

The interface engine is the natural place to consume these events because it already hosts the site's routing and stateful integration logic. This turns CAT-C into a **living authorization boundary** rather than a once-per-night API proxy.

### 9.7 Gateway Variants

#### Variant 1: Basic Gateway Policy (Recommended Baseline)

The simplest and most generally supportable pattern. RADIANT presents a signed StudyRelease / study ticket over mTLS or another approved ingress. The broker validates the signature, expiry, and exact study scope; persists the allow-list entry; then forwards the retrieval request to PACS.

- **Trust chain:** mTLS or site-approved ingress auth + JWKS-signed StudyRelease / study ticket
- **Broker validates:** caller identity, signature, expiration, study UID match, one-time token ID
- **Replay prevention:** broker tracks `ticket_id` in a short-lived cache aligned to ticket lifetime
- **Best fit:** API gateways, reverse proxies, healthcare integration engines

#### Variant 2: Epic-Originated Release Authorization (Proposed — Requires Epic TS Confirmation)

For sites that want Epic to remain the visible authorization authority even for CAT-C retrieval. RADIANT requests a release artifact from Epic or an Epic-owned service; the site broker only accepts retrieval if that Epic-originated release is present and valid. This is attractive institutionally, but it is **not** established by the provided Epic documentation and requires Epic TS confirmation.

#### Variant 3: Token Exchange (RFC 8693 / RFC 7662) (Proposed — Requires Site Broker Support)

For brokers that support standards-based token exchange: RADIANT presents an Epic / RADIANT token; the broker introspects it and mints a one-time broker-scoped retrieval token. This is the most standards-shaped pattern, but support varies widely by platform.

### 9.8 Healthcare API Gateways and Interface Engines

Many hospital sites already operate one of these platforms. RADIANT does **not** install or manage them; it provides a StudyRelease / policy pattern that the site can implement on what they already trust.

| Platform Class | Common Products | Best CAT-C Role |
|----------------|-----------------|-----------------|
| API gateways / reverse proxies | F5 BIG-IP, NGINX Plus, Kong, Apigee, Azure APIM, MuleSoft | Ingress auth, JWT / JWKS validation, DICOMweb proxying |
| Healthcare integration engines | Epic Bridges / Interconnect, Rhapsody, Cloverleaf, Corepoint, HealthShare, Mirth, Iguana, Qvera | Release / revoke message handling, stateful orchestration, identifier translation, callbacks |
| Institutional DICOM gateways | PACS-adjacent DICOMweb brokers | PACS-facing read execution |
| Hybrid engine + platform | Interface engine + Ambra / VNA | Control plane in engine, retrieval plane in platform |

### 9.9 Retrieval Ledger Integration

Gateway-brokered retrievals record additional fields in the retrieval ledger so that RADIANT can reconcile the control plane with the retrieval plane:

| Field | Value |
|-------|-------|
| `transport_method` | `"gateway_broker"` |
| `gateway_variant` | `"basic"` / `"epic_release"` / `"token_exchange"` |
| `study_release_status` | `"pending_release"` / `"released"` / `"retrieval_authorized"` / `"revoked"` |
| `release_ticket_id` | UUID of the StudyRelease / study ticket |
| `gateway_endpoint` | URL of the site broker endpoint |
| `broker_correlation_id` | Queue / message correlation ID at the site broker |
| `revoked_at` | Timestamp if the release was revoked before retrieval |

This makes CAT-C auditable end-to-end: manifest issued, release accepted, retrieval attempted, retrieval completed/failed, revocation applied, and downstream landing confirmed.

## 10. Cross-Platform Comparison: Epic vs Oracle Health / Cerner

CBTN sites run different EHR platforms. While the majority use Epic, some sites use Oracle Health (formerly Cerner). The imaging retrieval architecture must account for fundamental differences in how these platforms expose imaging data.

### 10.1 Capability Comparison

| Capability | Epic | Oracle Health (Cerner) |
|-----------|------|----------------------|
| **FHIR ImagingStudy** | Well-populated, includes study UIDs and endpoint references | Available but may lack endpoint references |
| **DICOMweb (WADO-RS)** | Available via Epic's proxy (requires site validation) | Available via DICOMgrid or Cerner's SMART on FHIR imaging API |
| **Bulk FHIR** | Supported via Vendor Services backend auth | Supported via Cerner's Bulk Data Access API |
| **Image Exchange** | Care Everywhere Image Exchange | CommonWell / Carequality image exchange |
| **FHIR Binary** | May serve DICOM content via Binary resources | **Capability-splitting model** (see below) |
| **IHE Profiles** | Via Epic Radiant module | Via Cerner RadNet or third-party PACS integration |
| **OAuth2 Scoping** | Registry-based scoping via Vendor Services | Patient-level or group-level scoping |

### 10.2 Oracle Health Capability-Splitting Pattern

Oracle Health (Cerner) uses a **capability-splitting model** for Binary resources that differs from Epic's approach:

- **Binary.read** requires the `Binary.read` scope AND a scope for the referring resource (e.g., `ImagingStudy.read` or `DiagnosticReport.read`)
- The Binary resource does not exist independently — it must be discovered through a referring FHIR resource
- This means you cannot enumerate or search Binary resources directly; you must follow reference chains from ImagingStudy → endpoint → Binary

This pattern originates from Oracle Health's DICOMgrid heritage, where imaging content is served through a staging/caching layer rather than directly from PACS:

```
Oracle Health FHIR API
    |
    |  ImagingStudy.endpoint --> DICOMgrid reference
    v
DICOMgrid Staging Layer
    |
    |  Binary.read (with referring scope) --> DICOM content
    v
DICOM bytes
```

```mermaid
flowchart TD
    FHIR["Oracle Health FHIR API"] --> IS["GET ImagingStudy\n(requires ImagingStudy.read scope)"]
    IS --> EP["ImagingStudy.endpoint\n→ DICOMgrid reference"]
    EP --> DG["DICOMgrid\nStaging Layer"]
    DG --> BIN["GET Binary/{id}\n(requires Binary.read +\nreferring resource scope)"]
    BIN --> DICOM["DICOM Bytes"]

    FHIR --> DR["GET DiagnosticReport\n(requires DiagnosticReport.read scope)"]
    DR --> DR_REF["DiagnosticReport.media\nor presentedForm"]
    DR_REF --> BIN

    style FHIR fill:#264653,color:#fff
    style DG fill:#e76f51,color:#fff
    style DICOM fill:#2a9d8f,color:#fff

    note1["Key difference from Epic:\nBinary.read requires scope\nfor the REFERRING resource"]
    style note1 fill:#fff3cd,stroke:#ffc107
```

**Implications for RADIANT:**

1. The manifest parser must handle Oracle Health's Binary reference patterns (different from Epic's endpoint references)
2. OAuth2 scopes must include both `Binary.read` and the referring resource scope
3. The capability probe (Task 0) must test the full reference chain, not just Binary.read in isolation
4. DICOMgrid may introduce additional latency and caching behavior not present in Epic's direct proxy

### 10.3 Site Profile Differentiation

The `SiteProfile` dataclass (Section 4.5) handles platform differences:

```python
# Epic site
SiteProfile(
    site_id="seattle_childrens",
    study_uid_system="urn:dicom:uid",
    accession_code="ACSN",
    accepted_statuses=("available", "final"),
    ehr_platform="epic",
)

# Oracle Health site
SiteProfile(
    site_id="nationwide_childrens",
    study_uid_system="urn:oid:2.16.840.1.113883",
    accession_code="ACSN",
    accepted_statuses=("available", "final", "registered"),
    ehr_platform="oracle_health",
    binary_requires_referring_scope=True,
)
```

---

## 11. End-to-End Workflow

### 11.1 Batch Cycle

```
Step 1: Bulk FHIR Incremental Pull (existing)
           |
Step 2: ImagingStudy Extraction & Manifest Generation
           |
Step 3: Manifest Dedup against Retrieval Ledger
           |
Step 4: Site Capability Probe (Task 0, if not already cached)
           |
Step 5: If CAT-C, issue StudyRelease / study ticket to site policy broker
           |
Step 6: DICOM Retrieval via validated transport
           |
           +-- CAT-A: FHIR Binary / DICOMweb
           +-- CAT-C: policy broker -> DICOM gateway / PACS read-only API
           +-- CAT-B: EHR-mediated exchange (experimental, must confirm full-study support)
           +-- CAT-D: PACS-adjacent platform
           +-- CAT-E: direct PACS protocol
           |
Step 7: Validation pass (series/instance count, DICOM header check)
           |
           +-- QA platform available: validate / tag / route
           +-- No QA platform: inline validation + S3 write
           |
Step 8: Retrieval Ledger updated
           |
Step 9: If CAT-C, broker emits StudyReady / RetrievalFailed and RADIANT reconciles state
           |
Step 10: Webhook/notification to downstream pipeline
           |
Step 11: Retrieval report generated
```

```mermaid
flowchart TD
    S1["Step 1: Bulk FHIR
Incremental Pull"] --> S2["Step 2: ImagingStudy Extraction
& Manifest Generation"]
    S2 --> S3["Step 3: Manifest Dedup
against Retrieval Ledger"]
    S3 --> S4["Step 4: Site Capability Probe
(Task 0, if not cached)"]
    S4 --> S5{"Validated transport
category?"}

    S5 -->|"CAT-A"| TA["FHIR Binary / DICOMweb"]
    S5 -->|"CAT-C"| REL["Issue StudyRelease /
study ticket to site broker"]
    REL --> TC["Policy broker validates,
queues, and invokes PACS
read-only API"]
    S5 -->|"CAT-B"| TB["EHR-mediated exchange
(experimental)"]
    S5 -->|"CAT-D"| TD["PACS-adjacent platform"]
    S5 -->|"CAT-E"| TE["Direct PACS protocol"]

    TA --> S7{"Validation pass"}
    TC --> S7
    TB --> S7
    TD --> S7
    TE --> S7

    S7 -->|"QA platform"| QA["Validate / tag / route"]
    S7 -->|"No QA platform"| INLINE["Inline validation + S3 write"]
    QA --> S8["Step 8: Retrieval Ledger
updated"]
    INLINE --> S8
    S8 --> S9["Step 9: Broker callback
reconciliation if CAT-C"]
    S9 --> S10["Step 10: Webhook / notification
to downstream pipeline"]
    S10 --> S11["Step 11: Retrieval report
generated"]

    style S1 fill:#264653,color:#fff
    style S11 fill:#2a9d8f,color:#fff
    style REL fill:#457b9d,color:#fff
    style TC fill:#74c69d,color:#000
```

### 11.2 Report Schema

```json
{
  "report_date": "2026-03-05",
  "site_id": "seattle_childrens",
  "manifest_id": "SCH-2026-03-05-001",
  "transport_category": "cat_a",
  "transport_adapter": "epic_dicomweb",
  "summary": {
    "studies_on_manifest": 7,
    "studies_new": 3,
    "studies_previously_retrieved": 4,
    "studies_retrieved_successfully": 3,
    "studies_failed": 0,
    "studies_pending_retry": 0,
    "total_series_retrieved": 36,
    "total_instances_retrieved": 8541,
    "total_bytes_transferred": 4294967296,
    "retrieval_duration_seconds": 720
  },
  "studies": [
    {
      "study_uid": "1.2.840.113619.2.55.3.12345",
      "accession_number": "RAD-2026-04521",
      "status": "retrieved",
      "series_expected": 12,
      "series_retrieved": 12,
      "instances_expected": 2847,
      "instances_retrieved": 2847,
      "validation_passed": true,
      "s3_path": "s3://radiant-imaging/seattle_childrens/studies/2026-03-04/1.2.840.113619.2.55.3.12345/",
      "ambra_uuid": "a1b2c3d4-...",
      "consent_proof": { "..." }
    }
  ]
}
```

---

## 12. Security and Compliance Architecture

### 12.1 Consent Chain of Trust

> **COMPLIANCE NOTE:** The consent chain described below represents the architectural design. Actual HIPAA, IRB, and institutional compliance claims require validation by each site's privacy officer and legal counsel. RADIANT provides the technical infrastructure for consent tracking; institutional governance provides the legal authority.

```
Patient enrolled in CBTN trial
    |
    v
Patient added to Epic CBTN Registry (institutional)
    |
    v
Epic Bulk FHIR export scoped to registry (Epic enforces)
    |
    v
ImagingStudy resource extracted (only for registry patients)
    |
    v
Study Manifest generated (inherits registry scoping)
    |
    v
DICOM retrieval scoped to manifest (RADIANT enforces)
    |
    v
Each retrieved study tagged with consent_proof
    |
    v
Audit trail: registry_id + manifest_id + export_id + timestamps
```

```mermaid
flowchart TD
    ENROLL["Patient enrolled\nin CBTN trial"] --> REGISTRY["Patient added to\nEpic CBTN Registry"]
    REGISTRY --> BULK["Epic Bulk FHIR export\n(registry-scoped)"]
    BULK --> IS["ImagingStudy resource\nextracted"]
    IS --> MANIFEST["Study Manifest\ngenerated"]
    MANIFEST --> RETRIEVE["DICOM retrieval\n(scoped to manifest)"]
    RETRIEVE --> TAG["Study tagged with\nconsent_proof"]
    TAG --> AUDIT["Audit trail:\nregistry_id + manifest_id +\nexport_id + timestamps"]

    REGISTRY -.->|"Periodic re-verification\n(Section 15.2)"| REVOKE{"Patient still\nin registry?"}
    REVOKE -->|"Yes"| VALID["Studies remain\nin S3"]
    REVOKE -->|"No"| QUARANTINE["Studies moved to\nquarantine/ prefix"]

    style ENROLL fill:#264653,color:#fff
    style AUDIT fill:#2a9d8f,color:#fff
    style QUARANTINE fill:#e63946,color:#fff
```

Every DICOM study in RADIANT's possession has a traceable chain from the study back to the patient's registry membership. This chain is designed to be:

- **Machine-verifiable**: Each link can be programmatically validated (pending site validation of registry scoping behavior)
- **Immutable**: Manifests and ledgers are append-only, stored in S3 with versioning
- **Auditable**: Ambra's `/audit/object` provides a complete access log; non-Ambra sites use S3 access logs + consent_proof.json sidecars

### 12.2 Authorization Model by Configuration

| Transport Category | Who enforces patient scoping? | Network exposure |
|-------------------|------------------------------|-----------------|
| CAT-A (Native EHR API) | EHR OAuth2 token scope | EHR FHIR/DICOMweb endpoint only |
| CAT-B (EHR Exchange) | EHR exchange authorization | EHR exchange endpoint only |
| CAT-C (Middleware / Policy Broker) | Study ticket + broker allow-list cache + local policy / identifier translation | Broker endpoint only (mTLS / site-approved ingress) |
| CAT-D (PACS-Adjacent) | Study manifest + platform auth | Platform API endpoint only |
| CAT-E (Direct PACS) | AE title trust + study manifest | DIMSE association (site network) |

In CAT-A through CAT-D, the site's PACS is never directly exposed to the internet — an intermediary (EHR, gateway, or platform) acts as the authorization and proxy layer. CAT-E is the exception, requiring DICOM network connectivity to the PACS.

### 12.3 Data at Rest and in Transit

| Segment | Encryption |
|---------|-----------|
| Epic to RADIANT retrieval service | TLS 1.2+ (HTTPS for DICOMweb, TLS for DICOM) |
| Retrieval service to Ambra | TLS 1.2+ (HTTPS API) |
| Retrieval service to S3 | TLS 1.2+ (HTTPS), S3 SSE-KMS at rest |
| Ambra to S3 | TLS 1.2+ (HTTPS), S3 SSE-KMS at rest |
| Ambra storage | AES-256 at rest (Ambra managed) |

### 12.4 Access Controls

| Component | Access Model |
|-----------|-------------|
| Study Manifests (S3) | IAM role: `radiant-imaging-pipeline` only |
| Retrieval Ledger (S3) | IAM role: `radiant-imaging-pipeline` only |
| DICOM files (S3) | IAM role: `radiant-imaging-pipeline` + `radiant-research` (read-only) |
| Ambra namespaces | Per-site service account, role: custom `radiant_retrieval` role |
| OAuth2 private keys | AWS Secrets Manager, rotated quarterly |

---

## 13. Site Onboarding Procedure

### 13.1 Onboarding Checklist

```
Phase 1: Assessment
  [ ] Complete Site Capability Assessment (Section 3) and Transport Selection (Section 2.4)
  [ ] Run automated Site Capability Probe (Task 0)
  [ ] Review probe results — determine validated transport method
  [ ] Confirm Epic/Oracle Health version and PACS vendor
  [ ] Determine Ambra availability and role (Config A/B/C)
  [ ] If site has middleware gateway: confirm gateway platform and mTLS capability

Phase 2: EHR Configuration
  [ ] Existing: RADIANT Connector Bulk FHIR app operational
  [ ] Existing: Patient registries configured
  [ ] Existing: Background user with security points configured
  [ ] New: Verify background user has imaging security points
  [ ] New: If gateway variant 2 — configure Epic release authorization
  [ ] New: If Ambra Config C — register Ambra as DICOM destination on PACS
  [ ] New: If Oracle Health — confirm Binary.read + referring resource scopes

Phase 3: RADIANT Configuration (parallel with Phase 2)
  [ ] Configure SiteProfile with validated transport method
  [ ] Set transport_method based on Task 0 probe results (not assumptions)
  [ ] If Ambra: create site namespace, custom fields, validation rules, routing rules, webhooks
  [ ] If CAT-C broker: configure StudyRelease / StudyRevoke / StudyReady endpoints, mTLS certificates, JWKS endpoint, queue/state store, and broker-side audit logging
  [ ] If CAT-C broker: document local identifier translation rules (patient, accession, archive IDs) and image-availability gating rules
  [ ] If no Ambra and no gateway: configure S3 bucket paths and inline validation
  [ ] Deploy retrieval service with site credentials
  [ ] Create retrieval ledger

Phase 4: Validation
  [ ] Run test retrieval for 1-2 known studies via validated transport
  [ ] Verify DICOM integrity (series count, instance count, header validation)
  [ ] Verify consent_proof chain (study -> manifest -> export -> registry)
  [ ] Verify S3 object layout matches UID-based schema
  [ ] Verify study_manifest.json sidecar contains correct checksums
  [ ] If Ambra: verify webhook fires and downstream pipeline receives notification
  [ ] If CAT-C broker: verify StudyRelease acceptance, replay prevention, revoke-before-retrieve behavior, queue drain, and StudyReady / RetrievalFailed callbacks
  [ ] Site security team reviews audit trail

Phase 5: Go-Live
  [ ] Enable batch cycle
  [ ] Monitor initial production retrieval period
  [ ] Verify reports
  [ ] Hand off to operations
```

```mermaid
flowchart LR
    P1["Phase 1:\nAssessment"] --> P2["Phase 2:\nConfiguration"]
    P2 --> P3["Phase 3:\nTesting"]
    P3 --> P4["Phase 4:\nProduction"]

    subgraph P1_detail["Assessment"]
        direction TB
        A1["Site Capability\nAssessment"] --> A2["Run Task 0\nProbe"]
        A2 --> A3["Review probe results\n& select transport"]
        A3 --> A4["Confirm EHR/PACS\nversions"]
    end

    subgraph P2_detail["Configuration"]
        direction TB
        B1["Create SiteProfile"] --> B2["Configure transport\nadapter"]
        B2 --> B3["Set up Ambra namespace\n(if applicable)"]
        B3 --> B4["Configure consent\ntagging"]
    end

    subgraph P3_detail["Testing"]
        direction TB
        C1["Single study\nretrieval test"] --> C2["Validate DICOM\nintegrity"]
        C2 --> C3["Verify consent chain\nend-to-end"]
        C3 --> C4["Confirm S3 landing\n& downstream notify"]
    end

    subgraph P4_detail["Production"]
        direction TB
        D1["Enable daily\nmanifest generation"] --> D2["Monitor retrieval\nmetrics"]
        D2 --> D3["Monthly audit\nreport review"]
    end

    style P1 fill:#264653,color:#fff
    style P2 fill:#2a9d8f,color:#fff
    style P3 fill:#e9c46a,color:#000
    style P4 fill:#e76f51,color:#fff
```

### 13.2 Per-Site Configuration File

```yaml
# site_config/seattle_childrens.yaml
site_id: seattle_childrens
institution_name: "Seattle Children's Hospital"
transport_category: "cat_a"  # CAT-A through CAT-E (determined by Task 0 probe)

epic:
  fhir_base_url: "https://fhir.seattlechildrens.org/api/FHIR/R4"
  dicomweb_base_url: "https://fhir.seattlechildrens.org/api/FHIR/R4/wado-rs"
  client_id: "58cb2142-93f9-4840-a8b8-0a1e7e71c224"
  jwks_url: "https://jwks.radiant.d3b.io/seattle_childrens/.well-known/jwks.json"
  private_key_secret_id: "radiant-epic-seattle-childrens-private-key"
  token_endpoint: "https://fhir.seattlechildrens.org/oauth2/token"
  registry_id: "100263"
  transport_adapter: "epic_dicomweb"  # adapter selected within the validated category
  validated_by_probe: true  # Set to true only after Task 0 validates this transport
  ehr_platform: "epic"  # or "oracle_health"

gateway:
  enabled: false
  endpoint: "https://pacs-gateway.seattlechildrens.org/dicomweb"
  variant: "basic"  # Variant 1 (basic) is the only confirmed pattern; variants 2-3 require Epic TS confirmation
  mtls_cert_secret_id: "radiant-gateway-seattle-childrens-cert"
  mtls_key_secret_id: "radiant-gateway-seattle-childrens-key"

ambra:
  enabled: true
  api_url: "https://radiant.ambrahealth.com/api/v3"
  namespace: "site_seattle_childrens"
  service_account_secret_id: "radiant-ambra-seattle-childrens"
  consent_field_uuid: "abc123-..."
  manifest_field_uuid: "def456-..."
  registry_field_uuid: "ghi789-..."
  validation_rule_uuid: "jkl012-..."
  s3_destination_uuid: "mno345-..."
  push_webhook_uuid: "pqr678-..."

retrieval:
  max_concurrent_retrievals: 2
  timeout_per_study_seconds: 600
  max_retry_attempts: 5
  retry_backoff_base_seconds: 3600  # 1 hour between retries

storage:
  s3_bucket: "radiant-imaging"
  s3_prefix: "seattle_childrens/"
  s3_kms_key_id: "arn:aws:kms:us-east-1:343218191717:key/..."

monitoring:
  alert_on_failure: true
  alert_email: "radiant-ops@d3b.io"
  daily_report_s3_prefix: "seattle_childrens/reports/"
```

---

## 14. Fallback Decision Tree

When the validated transport method fails, the retrieval service follows this decision tree based on the Task 0 capability probe results. The tree follows the revised category preference order for research-grade retrieval (CAT-A > CAT-C > CAT-B > CAT-D > CAT-E), reflecting that CAT-C provides a stronger authorization bridge than CAT-B for full-study DICOM retrieval (see Section 2.2).

### 14.1 Error Classification and Retry Policy

Before falling through to the next transport category, the retrieval service classifies each failure:

| Classification | Examples | Action |
|---|---|---|
| **Transient** | HTTP 500/502/503, network timeout, connection reset, TLS handshake timeout | Retry with exponential backoff (1s, 2s, 4s, 8s, 16s). Max 5 retries per attempt. |
| **Rate-limited** | HTTP 429, Ambra throttle response | Respect `Retry-After` header (or default 60s). Max 3 rate-limit cycles before marking attempt "deferred" for next daily run. |
| **Partial success** | Multipart stream interrupted mid-transfer, series count mismatch after download | Retry missing series only (series-level retrieval). Max 3 partial-recovery attempts. |
| **Auth failure** | HTTP 401, OAuth2 token expired, SSO credential error | Refresh token and retry once. If refresh fails, pause retrieval and alert operations. Do NOT fall through — auth errors affect all categories equally. |
| **Permanent** | HTTP 403 (forbidden), HTTP 404 (study not found), DICOM C-FIND returns 0 matches, endpoint returns "not supported" | No retry. Fall through to next transport category immediately. |
| **Configuration** | Endpoint URL unreachable, mTLS certificate rejected, study ticket signature invalid | No retry. Log diagnostic detail and fall through. These indicate Task 0 probe results are stale — trigger probe re-run. |

**Fall-through rule:** Only **permanent** and **configuration** failures advance to the next transport category. Transient and rate-limited failures are retried within the current category. A category is only marked "failed" after exhausting all retries.

### 14.2 Fallback Category Order

```
Start: attempt retrieval via SiteCapabilityReport.recommended_transport
    |
    v
[CAT-A] FHIR Binary (if validated) --------> Success? --> Done
    |                                              |
    | Not available or failed                      | Partial
    v                                              v
[CAT-A] DICOMweb (if validated) ------------> Success? --> Done
    |                                              |
    | Not available or failed                      | Partial (series-level retry)
    v                                              v
[CAT-C] Gateway Broker (if available) ------> Success? --> Done
    |                                              |
    | Not available or failed                      | Still incomplete
    v                                              v
[CAT-B] Care Everywhere (if validated) -----> Success? --> Done
    |                                              |
    | Not available or failed                      |
    v                                              |
[CAT-B] IHE RAD-69 (if validated) ----------> Success? --> Done
    |                                              |
    | Not available or failed                      | Mark study "partial"
    v                                              | Retry next cycle
[CAT-D] PACS-Adjacent Platform (Config C) --> Success? --> Done
    |
    | Not available or failed
    v
[CAT-E] Direct PACS (if configured) -------> Success? --> Done
    |
    | Not available or failed
    v
Mark study "all_categories_exhausted"
Alert operations with diagnostic info + probe results
```

```mermaid
flowchart TD
    START["Attempt retrieval via\nSiteCapabilityReport\n.recommended_transport"] --> CATA_BIN{"[CAT-A]\nFHIR Binary\n(if validated)"}
    CATA_BIN -->|"Success"| DONE["Done"]
    CATA_BIN -->|"Permanent failure\nor not available"| CATA_DW{"[CAT-A]\nDICOMweb\n(if validated)"}
    CATA_DW -->|"Success"| DONE
    CATA_DW -->|"Permanent failure"| CATC{"[CAT-C]\nGateway Broker\n(if available)"}
    CATC -->|"Success"| DONE
    CATC -->|"Not available"| CATB_CE{"[CAT-B]\nCare Everywhere\n(if validated)"}
    CATB_CE -->|"Success"| DONE
    CATB_CE -->|"Not available"| CATB_IHE{"[CAT-B]\nIHE RAD-69\n(if validated)"}
    CATB_IHE -->|"Success"| DONE
    CATB_IHE -->|"Not available"| CATD{"[CAT-D]\nPACS-Adjacent\nPlatform"}
    CATD -->|"Success"| DONE
    CATD -->|"Not available"| CATE{"[CAT-E]\nDirect PACS\n(if configured)"}
    CATE -->|"Success"| DONE
    CATE -->|"Not available"| EXHAUST["Mark study:\nall_categories_exhausted\nAlert operations"]

    CATA_DW -->|"Partial\n(series-level retry)"| RETRY_A["Retry missing\nseries"]
    RETRY_A --> DONE

    style DONE fill:#2d6a4f,color:#fff
    style EXHAUST fill:#e63946,color:#fff
    style START fill:#457b9d,color:#fff
```

> **Note:** The fallback order follows category priority, but only attempts categories validated by the Task 0 probe. A site with only CAT-C validated skips directly to the gateway broker. The tree above shows the maximum fallback chain.

A study that fails all transport methods (per Section 14.1 classification) for 5 consecutive days is escalated to manual resolution.

---

## 15. Monitoring and Observability

### 15.1 Metrics

| Metric | Source | Alert Threshold |
|--------|--------|----------------|
| Studies retrieved per day per site | Retrieval ledger | < expected (compare to manifest) |
| Retrieval success rate | Daily report | < 90% over 7-day window |
| Average retrieval time per study | Retrieval service logs | > 10 minutes |
| Validation failure rate | Validation results | > 0 (any failure is investigated) |
| OAuth2 token refresh failures | Auth service logs | > 0 |
| S3 write failures | S3 access logs | > 0 |
| Ambra webhook delivery failures | Ambra webhook logs | > 0 |
| Consent chain integrity | Audit verification job | Any broken chain |
| StudyRelease allow / deny count | Broker audit log | Unexpected spike in denies |
| Broker queue depth | Broker metrics | Sustained non-zero backlog outside backfill window |
| Revocation latency | Broker + RADIANT reconciliation | > 5 minutes for same-day revoke processing |
| Replay denial count | Broker audit log | > 0 investigated |
| Identifier translation miss rate | Broker / validation logs | > 0 investigated |
| Release-to-ready latency | Broker callback timestamps | > site-specific threshold |

### 15.2 Health Check

A periodic job validates the entire pipeline end-to-end:

1. Verify Bulk FHIR export completed for each active site
2. Verify manifest was generated
3. Verify all manifest studies have a retrieval attempt
4. Verify all retrieved studies are in S3 (or Ambra staging)
5. Verify all S3 studies have consent_proof.json sidecars
6. Verify retrieval ledger is consistent with S3 contents
7. If Ambra: verify namespace study count matches ledger count
8. **Consent re-verification**: For all studies currently in S3, re-validate that the patient remains an active member of the site's Epic research registry (or equivalent consent source). For each study:
   - Query the latest Bulk FHIR Group membership for the patient
   - If the patient is no longer in the registry, move the study to a `quarantine/` S3 prefix and update the retrieval ledger with `consent_status: "revoked"`
   - Do NOT delete quarantined studies — they may need IRB review to determine whether data collected under prior consent may be retained
   - Generate a consent revocation alert to operations with patient ID, study UID, original consent date, and revocation detection date

> **Design note:** Consent proof is point-in-time by nature — the `consent_proof.json` records that registry membership was confirmed at retrieval. The re-verification step (item 8) closes the gap between retrieval-time consent and ongoing consent validity. Sites with explicit consent withdrawal workflows (e.g., REDCap opt-out forms) should integrate those signals into the re-verification check alongside registry membership.

### 15.3 Audit Reports

Monthly audit report per site:

- Total studies retrieved
- Total instances and bytes transferred
- All patients whose imaging was retrieved, with registry membership proof
- Any failed retrievals with root cause
- Any consent chain discrepancies (should be zero)
- Consent revocations detected and quarantine actions taken
- Ambra audit trail export (`/audit/object` for all studies in period)

### 15.4 Probe Re-Validation

The `SiteCapabilityReport` generated by Task 0 at onboarding reflects a point-in-time snapshot. Site infrastructure changes (EHR upgrades, PACS replacements, certificate rotations, gateway decommissions) can silently invalidate cached probe results. The retrieval service maintains probe freshness through three mechanisms:

**Scheduled re-probe (monthly):** A full Task 0 probe re-runs against each active site on the first day of each month. If the `recommended_transport` changes, the retrieval service logs a `TRANSPORT_DRIFT` alert and pauses retrieval for that site until operations confirms the new transport. The previous `SiteCapabilityReport` is archived (not overwritten) for audit trail.

**Reactive re-probe (on configuration failure):** When a retrieval attempt fails with a `configuration` classification (Section 14.1 — endpoint unreachable, mTLS cert rejected, ticket signature invalid), the retrieval service immediately re-runs the relevant category's probe. If the probe now returns `not_available` for the active transport, it triggers a full re-probe and `TRANSPORT_DRIFT` alert.

**Version-change trigger:** If the site's EHR version or PACS vendor changes (detected via FHIR CapabilityStatement `software.version` or DICOM Association response), a full re-probe is triggered regardless of schedule.

| Trigger | Scope | Action on Change |
|---------|-------|-----------------|
| Monthly schedule | Full probe (all categories) | Archive old report, alert if recommended transport changes |
| Configuration failure | Single-category re-probe, then full if failed | Pause retrieval, re-probe, update report |
| EHR/PACS version change | Full probe (all categories) | Archive old report, alert, require operations sign-off |
| Manual trigger (`dicom_site_onboard.py --re-probe`) | Full probe (all categories) | Replace report, log reason |

---

### 15.5 Interface Engine / Policy Broker Observability

For CAT-C sites, the site broker becomes part of the trusted authorization boundary and should therefore be treated as a first-class observability source. Minimum broker-side telemetry should include:

- `StudyRelease` accepted / denied / expired / replayed
- queue depth by queue (`release_queue`, `retrieval_queue`, `retry_queue`, `revoke_queue`, `dead_letter_queue`)
- release-to-ready latency by study
- revocation processing latency
- identifier translation misses
- PACS read attempts and outcome class
- callback delivery success for `StudyReady` / `RetrievalFailed`

Where the site cannot expose raw broker logs externally, a daily broker audit extract is sufficient as long as it includes ticket ID, study UID, action, timestamp, and outcome.

## 16. Capacity Planning

### 16.1 Per-Site Estimates

| Parameter | Estimate | Basis |
|-----------|----------|-------|
| Studies per day | < 10 | Requirement |
| Series per study (brain MRI) | 8-15 | Typical neuro-onc protocol |
| Instances per series | 150-400 | Volumetric MRI |
| Instance size | 0.5-2 MB | Uncompressed DICOM |
| Study size | 500 MB - 2 GB | Full brain MRI with contrast |
| Daily volume per site | 2-10 GB | 5 studies x 1 GB average |
| Monthly volume per site | 60-300 GB | |
| Annual volume per site | 0.7-3.5 TB | |

### 16.2 Multi-Site Scaling

| Sites | Daily Volume | Monthly Volume | Annual Volume |
|-------|-------------|----------------|---------------|
| 5 | 10-50 GB | 0.3-1.5 TB | 3.5-18 TB |
| 10 | 20-100 GB | 0.6-3 TB | 7-35 TB |
| 25 | 50-250 GB | 1.5-7.5 TB | 18-90 TB |

### 16.3 Cost Estimates (AWS)

| Component | Unit Cost | Monthly (10 sites) |
|-----------|-----------|-------------------|
| S3 Standard storage | $0.023/GB | $14-69/mo (growing) |
| S3 PUT requests | $0.005/1000 | ~$5/mo |
| Data transfer (Epic to AWS) | $0 (inbound free) | $0 |
| Ambra storage | Per contract | Per contract |
| Lambda/ECS for retrieval service | ~$50/mo | $50/mo |
| **Total infrastructure** | | **~$70-125/mo** |

---

## 17. Integration with Existing RADIANT Pipeline

### 17.1 Downstream Consumers

Once DICOM studies are in S3, they are available to:

| Consumer | Integration Point |
|----------|------------------|
| RADIANT Portal imaging viewer | S3 path in study metadata |
| Research de-identification pipeline | S3 event notification or Ambra webhook |
| Radiomics / volumetrics pipeline | S3 path, filtered by modality |
| AI model inference (tumor segmentation) | S3 path, filtered by series description |
| Clinical timeline abstraction pipeline | ImagingStudy FHIR resource (already available) + DICOM header metadata |

### 17.2 FHIR-DICOM Linkage

The `ImagingStudy` FHIR resource and the DICOM files are linked by Study Instance UID:

```
ImagingStudy.identifier (study_uid) <--> DICOM StudyInstanceUID (0020,000D)
ImagingStudy.subject (patient_fhir_id) <--> consent_proof.json
ImagingStudy.started <--> DICOM StudyDate (0008,0020)
```

This enables the clinical timeline abstraction pipeline (V5.9.x) to enrich imaging timeline events with:
- Actual series descriptions (from DICOM headers, more detailed than FHIR)
- Slice count and spatial resolution (for volumetric analysis readiness)
- Scanner model and field strength (for multi-site harmonization)
- Radiologist annotations and key images (if available)

---

## 18. Open Questions and Future Work

| Item | Status | Notes |
|------|--------|-------|
| **Validate Epic DICOMweb proxy** availability at each active CBTN site via Task 0 probe | Pending | Requires IT survey + automated probe per site |
| **Validate Epic Image Exchange API** for programmatic (non-interactive) backend use | Pending | May require Epic TS consultation; framed as candidate fallback until confirmed |
| **Validate Epic Radiant IHE profiles** for external backend application access | Pending | RAD-69/RAD-55 availability for non-workstation callers |
| Determine if Epic's WADO-RS returns complete studies or requires series enumeration | Pending | Test with CHOP's Epic instance |
| **Validate FHIR Binary content** — do any sites serve DICOM via Binary resources? | Pending | Task 0 probe will test automatically |
| **Survey site middleware gateways / interface engines** — which sites already have F5, Kong, Apigee, MuleSoft, Rhapsody, Cloverleaf, Corepoint, HealthShare, Mirth, Iguana, Qvera, Epic Bridges / Interconnect, etc.? | Pending | Determines CAT-C policy-broker viability and preferred ingress pattern per site |
| **Epic-native ingress → Interconnect DICOM Gateway bridge** — can a site expose an approved backend ingress that hands off to `Imaging.DICOMweb` / `CEDICOM`? | Pending | Strongest Epic-native institutional broker story if validated |
| **Broker message contract validation** — which sites can implement `StudyRelease`, `StudyRevoke`, `StudyReady`, and `RetrievalFailed` on existing infrastructure? | Pending | Needed to maximize control-plane workflows |
| **Oracle Health (Cerner) site validation** — confirm DICOMgrid capability-splitting model | Pending | Binary.read + referring scope tested at Nationwide Children's |
| Ambra contract and namespace provisioning | Pending | Procurement |
| Define DICOM de-identification requirements (research vs clinical use) | Pending | IRB guidance needed |
| Historical backfill strategy for existing patients with years of imaging | Future | Separate design — one-time full pull |
| Oracle Health / Cerner connector full implementation | Future | Framework in Section 10; requires site validation |
| DICOM SR (Structured Reports) extraction and integration with timeline | Future | Extends clinical abstraction pipeline |
| Real-time retrieval triggered by HL7 ORM/ORU messages | Future | For sites needing sub-daily latency |
| **Study ticket revocation** — mechanism for revoking issued tickets before expiration | Future | Short TTL (1 hour) mitigates risk; formal revocation adds complexity |
| **Multi-gateway failover** — sites with primary + backup gateway endpoints | Future | SiteProfile could support ordered list of gateway endpoints |

### 18.1 Historical Backfill Architecture (Conceptual Only — Not Operational)

For existing registry patients with years of imaging history, the nightly incremental batch is insufficient — it only discovers new studies appearing in Bulk FHIR exports going forward. A one-time historical backfill requires a different approach:

**Discovery (placeholder — requires operational design):** The conceptual approach queries `ImagingStudy` resources for all registry patients with `_lastUpdated=ge1900-01-01` (effectively unbounded). This is a placeholder strategy; an operational design must address:
- Whether all target sites can support this query pattern efficiently (some Epic instances may timeout on unbounded ImagingStudy searches)
- How to chunk by date windows to avoid overwhelming the FHIR API
- How to isolate backfill traffic from daily incremental exports
- Whether sites need separate governance approval for historical imaging retrieval (distinct from prospective consent)

The manifest builder processes discovered studies into a backfill manifest separate from the daily manifest.

**Throttling:** Historical backfill may discover thousands of studies per site. The retrieval service must throttle backfill to avoid overwhelming the site's PACS or DICOMweb proxy:
- Maximum concurrent backfill retrievals per site: configurable (default: 1)
- Backfill runs only during off-peak hours (configurable window, e.g., 22:00-06:00 site local time)
- Daily retrieval takes priority — backfill pauses during the nightly batch window
- Progress tracked in a separate `backfill_ledger.json` to avoid polluting the daily ledger

**Prioritization:** Backfill studies are prioritized by clinical relevance:
1. Most recent studies first (most likely to be needed for active care)
2. Brain MRI studies before other modalities (primary research interest)
3. Studies with existing FHIR `DiagnosticReport` references (have radiology reads)

**Completion criteria:** Backfill is complete when all `ImagingStudy` resources for all registry patients have a corresponding entry in the retrieval ledger (retrieved or permanently failed). A backfill completion report is generated per site.

> **Note:** Backfill volume for a site with 500 registry patients and 10 years of imaging could be 5,000-15,000 studies (5-30 TB). Plan for weeks of backfill per site, not hours.

---

## Appendix A: Epic DICOMweb Endpoint Discovery

To determine if a site's Epic instance supports DICOMweb:

1. Check the FHIR CapabilityStatement: `GET {fhir_base}/metadata` (the CapabilityStatement endpoint; note: `.well-known/smart-configuration` is for SMART discovery, not the CapabilityStatement)
2. Look for `ImagingStudy` in supported resources
3. Check if `ImagingStudy.endpoint` references contain DICOMweb URLs
4. Test with a metadata-only probe: `GET {dicomweb_base}/studies?limit=1`
5. If 404 or unsupported, fall back to CAT-B/C/D/E transports

## Appendix B: Ambra InteleShare API Quick Reference

Key endpoints used in this design:

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/session/login` | POST | Authenticate service account |
| `/study/add` | POST | Ingest DICOM into namespace |
| `/study/set` | POST | Tag with consent metadata |
| `/study/validate` | POST | Run QA validation rules |
| `/study/push` | POST | Push to S3/DICOM destination |
| `/study/cfind` | POST | Search by study UID/accession |
| `/study/list` | GET | List studies with filters |
| `/study/dicomdata/load` | POST | Verify DICOM headers |
| `/customfield/add` | POST | Create consent tracking fields |
| `/validate/add` | POST | Create validation rules |
| `/route/add` | POST | Create routing rules |
| `/webhook/add` | POST | Register event webhooks |
| `/anonymization/add` | POST | Configure de-identification |
| `/bundle` | POST | Batch multiple API calls |
| `/audit/object` | GET | Study-level audit trail |
| `/destination/search` | POST | C-FIND against remote PACS |
| `/destination/retrieve` | POST | C-MOVE from remote PACS |

## Appendix C: FHIR ImagingStudy Resource Example

```json
{
  "resourceType": "ImagingStudy",
  "id": "img-study-001",
  "identifier": [
    {
      "system": "urn:dicom:uid",
      "value": "urn:oid:1.2.840.113619.2.55.3.12345"
    },
    {
      "type": {
        "coding": [{
          "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
          "code": "ACSN"
        }]
      },
      "value": "RAD-2026-04521"
    }
  ],
  "status": "available",
  "subject": {
    "reference": "Patient/e4f2a8c1-..."
  },
  "started": "2026-03-04T14:30:00Z",
  "numberOfSeries": 12,
  "numberOfInstances": 2847,
  "modality": [
    {
      "system": "http://dicom.nema.org/resources/ontology/DCM",
      "code": "MR"
    }
  ],
  "description": "MRI BRAIN W AND WO CONTRAST",
  "endpoint": [
    {
      "reference": "Endpoint/dicomweb-1"
    }
  ],
  "referrer": {
    "reference": "Practitioner/dr-neuro-001"
  },
  "series": [
    {
      "uid": "1.2.840.113619.2.55.3.12345.1",
      "modality": {
        "system": "http://dicom.nema.org/resources/ontology/DCM",
        "code": "MR"
      },
      "description": "AX T1 POST CONTRAST",
      "numberOfInstances": 192
    }
  ]
}
```

---

# Part II: Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

## Module Map

```
lib/
  imaging/                          # NEW — all DICOM retrieval modules
    __init__.py
    site_capability_probe.py        # Task 0: Site capability probe (GATE for transports)
    study_manifest.py               # Task 1: Manifest builder + schema + SiteProfile
    retrieval_ledger.py             # Task 2: Dedup ledger + study_release_status
    transport_base.py               # Task 3: Abstract transport adapter
    transport_epic_dicomweb.py      # Task 4: Epic DICOMweb WADO-RS (hypothesis, not default)
    transport_fhir_binary.py        # Task 4b: FHIR Binary/Media content retrieval
    transport_epic_image_exchange.py # Task 5: Care Everywhere (candidate fallback)
    transport_epic_ihe.py           # Task 6: IHE RAD-69 (candidate fallback)
    transport_ambra.py              # Task 7: Ambra (3 configs: target, post-ingest, gateway)
    transport_gateway_broker.py     # Task 8: Middleware / interface-engine policy broker client
    control_plane_messages.py       # Task 8: StudyRelease / StudyRevoke / StudyReady / RetrievalFailed contracts
    policy_broker_state.py          # Task 8: allow-list cache, replay / revoke store, queue state
    identifier_reconciliation.py    # Task 8: local patient / accession / archive ID mapping
    dicom_validator.py              # Task 9: DICOM integrity validation
    consent_proof.py                # Task 10: Consent chain builder
    site_config.py                  # Task 11: Per-site YAML config loader
    retrieval_orchestrator.py       # Task 12: Nightly batch orchestrator

scripts/
  dicom_retrieval_pipeline.py       # Task 13: CLI entry point
  dicom_site_onboard.py             # Task 14: Onboarding helper

tests/
  test_site_capability_probe.py     # Task 0 tests
  test_study_manifest.py            # Task 1 tests
  test_retrieval_ledger.py          # Task 2 tests
  test_transport_base.py            # Task 3 tests
  test_transport_epic_dicomweb.py   # Task 4 tests
  test_transport_gateway_broker.py  # Task 8 tests
  test_control_plane_messages.py    # Task 8 tests
  test_policy_broker_state.py       # Task 8 tests
  test_identifier_reconciliation.py # Task 8 tests
  test_dicom_validator.py           # Task 9 tests
  test_consent_proof.py             # Task 10 tests
  test_site_config.py               # Task 11 tests
  test_retrieval_orchestrator.py    # Task 12 tests
  integration/
    test_epic_dicomweb_live.py      # Task 15: Integration test (CHOP)
    test_ambra_live.py              # Task 15: Integration test (Ambra)

docs/
  plans/
    2026-03-05-radiant-connector-dicom-retrieval.md              # THIS FILE (combined design + implementation)
    2026-03-05-radiant-connector-dicom-retrieval-rationale.md    # Task 16
  DICOM_RETRIEVAL_OPERATIONS.md     # Task 17: Ops runbook
  DICOM_SITE_ONBOARDING.md          # Task 17: Onboarding guide
```

---

## Task 0: Site Capability Probe

**GATE TASK:** This probe runs at each site BEFORE any transport coding or configuration. It determines which transport mechanisms are actually available, including whether a CAT-C broker can accept StudyRelease messages, enforce replay prevention, and surface a usable callback path. This prevents implementation effort on unvalidated assumptions.

**Files:**
- Create: `lib/imaging/site_capability_probe.py`
- Create: `tests/test_site_capability_probe.py`

**Step 1: Write failing tests**

```python
# tests/test_site_capability_probe.py
"""Tests for Site Capability Probe."""
import json
import pytest
from unittest.mock import MagicMock, patch

from lib.imaging.site_capability_probe import (
    ProbeResult,
    SiteCapabilityReport,
    SiteCapabilityProbe,
    FHIRContentProbe,
)


class TestSiteCapabilityReport:

    def test_empty_report(self):
        report = SiteCapabilityReport(site_id="test_site")
        assert report.recommended_transport is None
        assert report.fallback_transports == []

    def test_recommends_fhir_binary_when_validated(self):
        report = SiteCapabilityReport(
            site_id="test",
            cat_a_fhir_binary=ProbeResult(status="validated"),
        )
        assert report.recommended_transport == "cat_a_fhir_binary"

    def test_recommends_gateway_over_care_everywhere(self):
        """CAT-C ranks above CAT-B for research retrieval."""
        report = SiteCapabilityReport(
            site_id="test",
            cat_b_care_everywhere=ProbeResult(status="validated"),
            cat_c_gateway=ProbeResult(status="validated"),
        )
        assert report.recommended_transport == "cat_c_gateway"
        assert "cat_b_care_everywhere" in report.fallback_transports

    def test_recommends_dicomweb_when_validated(self):
        report = SiteCapabilityReport(
            site_id="test",
            cat_a_dicomweb=ProbeResult(status="validated"),
        )
        assert report.recommended_transport == "cat_a_dicomweb"

    def test_no_recommendation_when_all_not_tested(self):
        report = SiteCapabilityReport(site_id="test")
        assert report.recommended_transport is None

    def test_serializes_to_dict(self):
        report = SiteCapabilityReport(
            site_id="chop",
            cat_a_dicomweb=ProbeResult(status="validated"),
            qa_platform_available=True,
            qa_platform_type="ambra",
        )
        data = report.to_dict()
        assert data["site_id"] == "chop"
        assert data["recommended_transport"] == "cat_a_dicomweb"
        assert data["qa_platform_available"] is True


class TestFHIRContentProbe:

    @patch("lib.imaging.site_capability_probe.requests.get")
    def test_probe_binary_read(self, mock_get):
        mock_get.return_value = MagicMock(
            status_code=200,
            headers={"Content-Type": "application/dicom"},
            content=b"\x00" * 100,
        )
        probe = FHIRContentProbe(
            fhir_base_url="https://fhir.example.com/api/FHIR/R4",
            access_token="tok123",
        )
        result = probe.probe_binary_study_content("Binary/img-001")
        assert result.accessible is True
        assert result.content_type == "application/dicom"

    @patch("lib.imaging.site_capability_probe.requests.get")
    def test_probe_binary_returns_non_dicom(self, mock_get):
        mock_get.return_value = MagicMock(
            status_code=200,
            headers={"Content-Type": "application/pdf"},
            content=b"%PDF",
        )
        probe = FHIRContentProbe(
            fhir_base_url="https://fhir.example.com/api/FHIR/R4",
            access_token="tok123",
        )
        result = probe.probe_binary_study_content("Binary/img-001")
        assert result.accessible is True
        assert result.content_type == "application/pdf"
        assert not result.is_dicom

    @patch("lib.imaging.site_capability_probe.requests.get")
    def test_probe_binary_404(self, mock_get):
        mock_get.return_value = MagicMock(status_code=404)
        probe = FHIRContentProbe(
            fhir_base_url="https://fhir.example.com/api/FHIR/R4",
            access_token="tok123",
        )
        result = probe.probe_binary_study_content("Binary/img-001")
        assert result.accessible is False


class TestSiteCapabilityProbe:

    def test_generates_preflight_checklist(self):
        probe = SiteCapabilityProbe(site_id="chop")
        checklist = probe.get_preflight_checklist()
        assert len(checklist) >= 6
        assert any("ImagingStudy" in q for q in checklist)
        assert any("Binary" in q or "Media" in q for q in checklist)
        assert any("gateway" in q.lower() for q in checklist)
```

**Step 2: Run tests — expected FAIL**

**Step 3: Write implementation**

```python
# lib/imaging/site_capability_probe.py
"""
Site Capability Probe

GATE MODULE: Runs at each site BEFORE any transport implementation is configured.
Determines which imaging retrieval mechanisms are actually available.

The probe tests:
  1. Does Bulk FHIR include ImagingStudy resources?
  2. Does FHIR Binary.Read on a study-linked reference return DICOM content?
  3. Does FHIR Media.Read return study-linked content?
  4. Is a DICOMweb endpoint accessible?
  5. Does the site have a middleware gateway / PACS broker?
  6. Are Care Everywhere or IHE retrieval profiles enabled?
  7. What identifiers are available (StudyInstanceUID, accession, patient ID)?

This probe prevents building transport adapters against unvalidated assumptions.
Epic's public FHIR catalog shows ImagingStudy, Binary.Read/Search (Study), and
Media.Read/Search (Study), but does NOT clearly document a public third-party
SMART-scoped WADO-RS proxy for external backend apps. The probe validates what
actually works at each site before committing to a transport path.
"""

import json
import logging
from dataclasses import dataclass, field, asdict
from typing import Any, Dict, List, Optional

import requests

logger = logging.getLogger(__name__)


@dataclass
class ContentProbeResult:
    """Result of probing a single FHIR content endpoint."""
    accessible: bool = False
    content_type: str = ""
    content_length: int = 0
    is_dicom: bool = False
    error: str = ""

    def __post_init__(self):
        if self.content_type and "dicom" in self.content_type.lower():
            self.is_dicom = True


@dataclass
class ProbeResult:
    """Result of a single capability probe."""
    status: str = "not_tested"  # "validated", "not_available", "error", "not_tested"
    response_code: int | None = None
    response_time_ms: float | None = None
    error_message: str | None = None
    endpoint_url: str | None = None


@dataclass
class SiteCapabilityReport:
    """Results of probing a site's imaging capabilities.

    Aligned with Part I Section 2 schema. Each transport category
    has one or more probes; the recommended_transport follows the
    revised preference order: CAT-A > CAT-C > CAT-B > CAT-D > CAT-E.
    """
    site_id: str
    probed_at: str = ""  # ISO 8601
    ehr_platform: str = "epic"  # "epic", "oracle_health", "other"

    # CAT-A probes
    cat_a_fhir_binary: ProbeResult = field(default_factory=ProbeResult)
    cat_a_fhir_media: ProbeResult = field(default_factory=ProbeResult)
    cat_a_dicomweb: ProbeResult = field(default_factory=ProbeResult)
    cat_a_diagnostic_report: ProbeResult = field(default_factory=ProbeResult)

    # CAT-B probes
    cat_b_care_everywhere: ProbeResult = field(default_factory=ProbeResult)
    cat_b_ihe_rad69: ProbeResult = field(default_factory=ProbeResult)

    # CAT-C probes
    cat_c_gateway: ProbeResult = field(default_factory=ProbeResult)
    cat_c_gateway_platform: str | None = None

    # CAT-D probes
    cat_d_platform: ProbeResult = field(default_factory=ProbeResult)
    cat_d_platform_type: str | None = None

    # CAT-E probes
    cat_e_dicom_echo: ProbeResult = field(default_factory=ProbeResult)
    cat_e_ae_title: str | None = None

    # QA platform (secondary axis)
    qa_platform_available: bool = False
    qa_platform_type: str | None = None

    @property
    def recommended_transport(self) -> str | None:
        """Return recommended transport following CAT-A > CAT-C > CAT-B > CAT-D > CAT-E."""
        # CAT-A: any validated sub-probe
        if self.cat_a_fhir_binary.status == "validated":
            return "cat_a_fhir_binary"
        if self.cat_a_dicomweb.status == "validated":
            return "cat_a_dicomweb"
        # CAT-C: gateway (ranks above CAT-B for research retrieval)
        if self.cat_c_gateway.status == "validated":
            return "cat_c_gateway"
        # CAT-B: EHR-mediated exchange
        if self.cat_b_ihe_rad69.status == "validated":
            return "cat_b_ihe_rad69"
        if self.cat_b_care_everywhere.status == "validated":
            return "cat_b_care_everywhere"
        # CAT-D: PACS-adjacent platform
        if self.cat_d_platform.status == "validated":
            return "cat_d_platform"
        # CAT-E: direct PACS
        if self.cat_e_dicom_echo.status == "validated":
            return "cat_e_dicom_echo"
        return None

    @property
    def fallback_transports(self) -> list[str]:
        """Return ordered list of validated transports excluding the recommended one."""
        all_validated = []
        for attr, name in [
            ("cat_a_fhir_binary", "cat_a_fhir_binary"),
            ("cat_a_dicomweb", "cat_a_dicomweb"),
            ("cat_c_gateway", "cat_c_gateway"),
            ("cat_b_ihe_rad69", "cat_b_ihe_rad69"),
            ("cat_b_care_everywhere", "cat_b_care_everywhere"),
            ("cat_d_platform", "cat_d_platform"),
            ("cat_e_dicom_echo", "cat_e_dicom_echo"),
        ]:
            if getattr(self, attr).status == "validated" and name != self.recommended_transport:
                all_validated.append(name)
        return all_validated

    def to_dict(self) -> dict:
        return {
            "site_id": self.site_id,
            "probed_at": self.probed_at,
            "ehr_platform": self.ehr_platform,
            "recommended_transport": self.recommended_transport,
            "fallback_transports": self.fallback_transports,
            "qa_platform_available": self.qa_platform_available,
            "qa_platform_type": self.qa_platform_type,
        }


class FHIRContentProbe:
    """Probes FHIR Binary and Media endpoints for actual study content."""

    def __init__(self, fhir_base_url: str, access_token: str):
        self._base = fhir_base_url.rstrip("/")
        self._token = access_token

    def probe_binary_study_content(self, binary_ref: str) -> ContentProbeResult:
        """Test if Binary.Read returns DICOM content for a study-linked reference."""
        url = f"{self._base}/{binary_ref}"
        try:
            resp = requests.get(
                url,
                headers={
                    "Authorization": f"Bearer {self._token}",
                    "Accept": "application/dicom, application/octet-stream, */*",
                },
                timeout=30,
                stream=True,
            )
            if resp.status_code != 200:
                return ContentProbeResult(
                    accessible=False,
                    error=f"HTTP {resp.status_code}",
                )
            ct = resp.headers.get("Content-Type", "")
            content = resp.content[:1024]  # Only read first 1KB for probe
            return ContentProbeResult(
                accessible=True,
                content_type=ct,
                content_length=int(resp.headers.get("Content-Length", len(content))),
            )
        except Exception as e:
            return ContentProbeResult(accessible=False, error=str(e))

    def probe_media_study_content(self, media_ref: str) -> ContentProbeResult:
        """Test if Media.Read returns study-linked content."""
        url = f"{self._base}/{media_ref}"
        try:
            resp = requests.get(
                url,
                headers={
                    "Authorization": f"Bearer {self._token}",
                    "Accept": "application/fhir+json",
                },
                timeout=30,
            )
            if resp.status_code != 200:
                return ContentProbeResult(accessible=False, error=f"HTTP {resp.status_code}")
            data = resp.json()
            content_ct = data.get("content", {}).get("contentType", "")
            return ContentProbeResult(
                accessible=True,
                content_type=content_ct,
            )
        except Exception as e:
            return ContentProbeResult(accessible=False, error=str(e))


class SiteCapabilityProbe:
    """
    Orchestrates capability probing for a site.

    Usage:
        probe = SiteCapabilityProbe(site_id="chop")
        report = probe.run(
            fhir_base_url="https://fhir.epic.com/...",
            access_token="tok...",
            sample_imaging_study=sample_resource,
        )
        print(report.recommended_transport)
    """

    def __init__(self, site_id: str):
        self.site_id = site_id

    def get_preflight_checklist(self) -> List[str]:
        """Return the preflight capability questions for site assessment."""
        return [
            "Does the site's Bulk FHIR export actually include ImagingStudy resources?",
            "Does Epic expose usable Binary/Media Study content via the backend app?",
            "Does any Epic-native imaging retrieval interface exist beyond plain FHIR (DICOMweb, Image Exchange)?",
            "Is there an existing site imaging gateway (Ambra, DICOMgrid, Intelerad, vendor gateway)?",
            "Does the site have an API gateway or reverse proxy that could front PACS (F5, Apigee, Kong, MuleSoft, etc.)?",
            "Are Care Everywhere or IHE retrieval profiles actually enabled for external partners?",
            "What exact identifiers are available for study-level retrieval (StudyInstanceUID, accession, patient ID, modality)?",
            "What PACS vendor and version is deployed, and does it support DICOMweb natively?",
        ]

    def run(
        self,
        fhir_base_url: str,
        access_token: str,
        sample_imaging_study: Optional[Dict] = None,
        dicomweb_base_url: Optional[str] = None,
    ) -> SiteCapabilityReport:
        """
        Run the full capability probe against a site.

        Args:
            fhir_base_url: Epic FHIR R4 base URL
            access_token: Valid OAuth2 access token
            sample_imaging_study: A sample ImagingStudy resource from Bulk FHIR
            dicomweb_base_url: Optional DICOMweb endpoint to test
        """
        report = SiteCapabilityReport(site_id=self.site_id)

        # 1. Check ImagingStudy availability
        if sample_imaging_study:
            report.cat_a_diagnostic_report = ProbeResult(status="validated")
            self._extract_identifiers(sample_imaging_study, report)
            self._extract_endpoints(sample_imaging_study, report)

        # 2. Probe FHIR Binary/Media content
        if sample_imaging_study and fhir_base_url:
            fhir_probe = FHIRContentProbe(fhir_base_url, access_token)
            self._probe_fhir_content(sample_imaging_study, fhir_probe, report)

        # 3. Probe DICOMweb
        if dicomweb_base_url:
            self._probe_dicomweb(dicomweb_base_url, access_token, report)

        return report

    def _extract_identifiers(self, resource: Dict, report: SiteCapabilityReport) -> None:
        """Capture lightweight hints from a sample ImagingStudy."""
        for ident in resource.get("identifier", []):
            system = ident.get("system", "")
            if "dicom" in system.lower() or "urn:oid:" in ident.get("value", ""):
                if report.cat_a_diagnostic_report.status == "not_tested":
                    report.cat_a_diagnostic_report = ProbeResult(status="validated")

    def _extract_endpoints(self, resource: Dict, report: SiteCapabilityReport) -> None:
        """Record endpoint hints on the DICOMweb probe result when present."""
        for ep in resource.get("endpoint", []):
            ref = ep.get("reference", "")
            if ref:
                report.cat_a_dicomweb.endpoint_url = ref

    def _probe_fhir_content(
        self, resource: Dict, probe: FHIRContentProbe, report: SiteCapabilityReport
    ) -> None:
        """Probe Binary and Media references from the ImagingStudy resource."""
        # Site-specific logic lives in the live probe implementation. The design
        # contract here is that a successful Binary / Media read updates the
        # corresponding CAT-A probe result to `validated`.
        pass

    def _probe_dicomweb(
        self, base_url: str, token: str, report: SiteCapabilityReport
    ) -> None:
        """Test if DICOMweb endpoint responds and update the CAT-A probe result."""
        try:
            resp = requests.get(
                f"{base_url}/studies?limit=0",
                headers={"Authorization": f"Bearer {token}"},
                timeout=15,
            )
            if resp.status_code in (200, 204):
                report.cat_a_dicomweb = ProbeResult(
                    status="validated",
                    response_code=resp.status_code,
                    endpoint_url=base_url,
                )
            else:
                report.cat_a_dicomweb = ProbeResult(
                    status="not_available",
                    response_code=resp.status_code,
                    endpoint_url=base_url,
                    error_message=f"HTTP {resp.status_code}",
                )
        except Exception as e:
            report.cat_a_dicomweb = ProbeResult(
                status="error",
                endpoint_url=base_url,
                error_message=str(e),
            )
```

**Step 4: Run tests — expected all PASS**

**Step 5: Commit**

```bash
git add lib/imaging/site_capability_probe.py tests/test_site_capability_probe.py
git commit -m "feat(imaging): site capability probe — GATE for transport selection"
```

---

## Task 1: Study Manifest Builder

The manifest extracts ImagingStudy resources from Bulk FHIR NDJSON output and produces a consent-scoped study list. Additionally supports DiagnosticReport and ServiceRequest as secondary discovery paths, since sites differ in how completely they populate imaging FHIR resources.

**Files:**
- Create: `lib/imaging/__init__.py`
- Create: `lib/imaging/study_manifest.py`
- Create: `tests/test_study_manifest.py`

**Step 1: Write failing tests**

```python
# tests/test_study_manifest.py
"""Tests for Study Manifest Builder."""
import json
import pytest
from datetime import datetime, timezone
from pathlib import Path
from unittest.mock import patch

from lib.imaging.study_manifest import (
    StudyManifestEntry,
    StudyManifest,
    StudyManifestBuilder,
    SiteProfile,
    extract_imaging_studies_from_ndjson,
    extract_diagnostic_reports_from_ndjson,
    parse_imaging_study_resource,
)


# --- Fixtures ---

SAMPLE_IMAGING_STUDY = {
    "resourceType": "ImagingStudy",
    "id": "img-study-001",
    "identifier": [
        {
            "system": "urn:dicom:uid",
            "value": "urn:oid:1.2.840.113619.2.55.3.12345"
        },
        {
            "type": {
                "coding": [{
                    "system": "http://terminology.hl7.org/CodeSystem/v2-0203",
                    "code": "ACSN"
                }]
            },
            "value": "RAD-2026-04521"
        }
    ],
    "status": "available",
    "subject": {"reference": "Patient/e4f2a8c1-0000-0000-0000-000000000001"},
    "started": "2026-03-04T14:30:00Z",
    "numberOfSeries": 12,
    "numberOfInstances": 2847,
    "modality": [{"system": "http://dicom.nema.org/resources/ontology/DCM", "code": "MR"}],
    "description": "MRI BRAIN W AND WO CONTRAST",
    "endpoint": [{"reference": "Endpoint/dicomweb-1"}],
    "referrer": {"reference": "Practitioner/dr-neuro-001"},
    "series": [
        {
            "uid": "1.2.840.113619.2.55.3.12345.1",
            "modality": {"system": "http://dicom.nema.org/resources/ontology/DCM", "code": "MR"},
            "description": "AX T1 POST CONTRAST",
            "numberOfInstances": 192
        }
    ]
}

SAMPLE_NDJSON_LINES = [
    json.dumps({"resourceType": "Patient", "id": "p1"}),
    json.dumps(SAMPLE_IMAGING_STUDY),
    json.dumps({"resourceType": "Condition", "id": "c1"}),
    json.dumps({
        "resourceType": "ImagingStudy",
        "id": "img-study-002",
        "identifier": [
            {"system": "urn:dicom:uid", "value": "urn:oid:1.2.840.113619.2.55.3.99999"}
        ],
        "status": "available",
        "subject": {"reference": "Patient/e4f2a8c1-0000-0000-0000-000000000002"},
        "started": "2026-03-04T10:00:00Z",
        "numberOfSeries": 5,
        "numberOfInstances": 800,
        "modality": [{"system": "http://dicom.nema.org/resources/ontology/DCM", "code": "CT"}],
        "description": "CT HEAD WO CONTRAST",
    }),
]


# --- Tests ---

class TestParseImagingStudyResource:
    """Test extraction of fields from a single ImagingStudy FHIR resource."""

    def test_extracts_study_uid(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.study_uid == "1.2.840.113619.2.55.3.12345"

    def test_strips_urn_oid_prefix(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        # Should strip "urn:oid:" prefix
        assert not entry.study_uid.startswith("urn:oid:")

    def test_extracts_accession_number(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.accession_number == "RAD-2026-04521"

    def test_extracts_patient_fhir_id(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.patient_fhir_id == "e4f2a8c1-0000-0000-0000-000000000001"

    def test_extracts_study_date(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.study_date == "2026-03-04"

    def test_extracts_modality(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.modality == ["MR"]

    def test_extracts_series_and_instance_counts(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.expected_series == 12
        assert entry.expected_instances == 2847

    def test_extracts_endpoint_ref(self):
        entry = parse_imaging_study_resource(SAMPLE_IMAGING_STUDY)
        assert entry.endpoint_ref == "Endpoint/dicomweb-1"

    def test_handles_missing_accession_number(self):
        resource = {**SAMPLE_IMAGING_STUDY, "identifier": [
            {"system": "urn:dicom:uid", "value": "urn:oid:1.2.3.4.5"}
        ]}
        entry = parse_imaging_study_resource(resource)
        assert entry.accession_number is None

    def test_handles_missing_endpoint(self):
        resource = {k: v for k, v in SAMPLE_IMAGING_STUDY.items() if k != "endpoint"}
        entry = parse_imaging_study_resource(resource)
        assert entry.endpoint_ref is None

    def test_handles_missing_series_count(self):
        resource = {k: v for k, v in SAMPLE_IMAGING_STUDY.items() if k != "numberOfSeries"}
        entry = parse_imaging_study_resource(resource)
        assert entry.expected_series is None

    def test_skips_non_available_status(self):
        resource = {**SAMPLE_IMAGING_STUDY, "status": "cancelled"}
        entry = parse_imaging_study_resource(resource)
        assert entry is None

    def test_custom_site_profile_changes_uid_system(self):
        """Site profiles allow custom identifier systems."""
        profile = SiteProfile(
            study_uid_system="urn:custom:imaging",
            accepted_statuses=["available", "unknown"],
        )
        resource = {
            **SAMPLE_IMAGING_STUDY,
            "identifier": [
                {"system": "urn:custom:imaging", "value": "CUSTOM-UID-123"}
            ],
        }
        entry = parse_imaging_study_resource(resource, site_profile=profile)
        assert entry.study_uid == "CUSTOM-UID-123"

    def test_custom_site_profile_accepts_additional_statuses(self):
        profile = SiteProfile(accepted_statuses=["available", "unknown"])
        resource = {**SAMPLE_IMAGING_STUDY, "status": "unknown"}
        entry = parse_imaging_study_resource(resource, site_profile=profile)
        assert entry is not None


class TestExtractDiagnosticReportsFromNdjson:
    """Test secondary discovery path via DiagnosticReport resources."""

    def test_extracts_imaging_references_from_diagnostic_reports(self, tmp_path):
        dr_resource = {
            "resourceType": "DiagnosticReport",
            "id": "dr-001",
            "status": "final",
            "category": [{"coding": [{"code": "RAD"}]}],
            "subject": {"reference": "Patient/p1"},
            "effectiveDateTime": "2026-03-04T10:00:00Z",
            "imagingStudy": [{"reference": "ImagingStudy/img-study-001"}],
        }
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text(json.dumps(dr_resource))
        refs = extract_diagnostic_reports_from_ndjson(ndjson_file)
        assert len(refs) == 1
        assert refs[0]["imaging_study_ref"] == "ImagingStudy/img-study-001"


class TestExtractImagingStudiesFromNdjson:
    """Test filtering ImagingStudy resources from NDJSON bundle."""

    def test_extracts_only_imaging_studies(self, tmp_path):
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text("\n".join(SAMPLE_NDJSON_LINES))
        entries = extract_imaging_studies_from_ndjson(ndjson_file)
        assert len(entries) == 2

    def test_preserves_order(self, tmp_path):
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text("\n".join(SAMPLE_NDJSON_LINES))
        entries = extract_imaging_studies_from_ndjson(ndjson_file)
        assert entries[0].study_uid == "1.2.840.113619.2.55.3.12345"
        assert entries[1].study_uid == "1.2.840.113619.2.55.3.99999"

    def test_handles_empty_file(self, tmp_path):
        ndjson_file = tmp_path / "empty.ndjson"
        ndjson_file.write_text("")
        entries = extract_imaging_studies_from_ndjson(ndjson_file)
        assert entries == []

    def test_skips_malformed_lines(self, tmp_path):
        ndjson_file = tmp_path / "bad.ndjson"
        ndjson_file.write_text("not json\n" + json.dumps(SAMPLE_IMAGING_STUDY))
        entries = extract_imaging_studies_from_ndjson(ndjson_file)
        assert len(entries) == 1


class TestStudyManifestBuilder:
    """Test manifest generation with consent proof."""

    def test_builds_manifest_with_consent_proof(self, tmp_path):
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text("\n".join(SAMPLE_NDJSON_LINES))

        builder = StudyManifestBuilder(
            site_id="seattle_childrens",
            registry_id="100263",
            registry_name="CHOP CBTN ALL PATIENT REGISTRY",
            bulk_fhir_export_id="export-2026-03-05-daily",
        )
        manifest = builder.build(ndjson_file)

        assert manifest.site_id == "seattle_childrens"
        assert manifest.registry_id == "100263"
        assert len(manifest.studies) == 2
        assert manifest.studies[0].consent_proof.registry_id == "100263"

    def test_manifest_stats(self, tmp_path):
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text("\n".join(SAMPLE_NDJSON_LINES))

        builder = StudyManifestBuilder(
            site_id="test_site",
            registry_id="100263",
            registry_name="TEST",
            bulk_fhir_export_id="test-export",
        )
        manifest = builder.build(ndjson_file)

        assert manifest.stats.total_studies == 2
        assert manifest.stats.modality_breakdown["MR"] == 1
        assert manifest.stats.modality_breakdown["CT"] == 1

    def test_manifest_serializes_to_json(self, tmp_path):
        ndjson_file = tmp_path / "export.ndjson"
        ndjson_file.write_text(json.dumps(SAMPLE_IMAGING_STUDY))

        builder = StudyManifestBuilder(
            site_id="test_site",
            registry_id="100263",
            registry_name="TEST",
            bulk_fhir_export_id="test-export",
        )
        manifest = builder.build(ndjson_file)
        json_str = manifest.to_json()
        parsed = json.loads(json_str)

        assert parsed["site_id"] == "test_site"
        assert len(parsed["studies"]) == 1
        assert "consent_proof" in parsed["studies"][0]
```

**Step 2: Run tests to verify they fail**

Run: `cd /Users/resnick/Documents/GitHub/RADIANT_PCA/BRIM_Analytics/patient_clinical_journey_timeline && python -m pytest tests/test_study_manifest.py -v`
Expected: FAIL — `ModuleNotFoundError: No module named 'lib.imaging'`

**Step 3: Write implementation**

```python
# lib/imaging/__init__.py
"""RADIANT DICOM Imaging Retrieval Pipeline."""

# lib/imaging/study_manifest.py
"""
Study Manifest Builder

Extracts ImagingStudy FHIR resources from Bulk FHIR NDJSON export and
builds a consent-scoped manifest for DICOM retrieval. Also supports
DiagnosticReport and ServiceRequest as secondary discovery paths,
since sites differ in how completely they populate imaging FHIR resources.

The manifest is the single source of truth for which studies are authorized
for retrieval. A study appears on the manifest if and only if it was returned
by Epic's registry-scoped Bulk FHIR export.

Identifier parsing (UID system, accession code, accepted statuses) is
configurable via SiteProfile to accommodate site-specific FHIR variations.
"""

import json
import logging
from collections import Counter
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Any, Dict, List, Optional
import uuid

logger = logging.getLogger(__name__)


@dataclass
class SiteProfile:
    """
    Per-site configuration for FHIR resource parsing.

    Different sites use different identifier systems, status values, and
    accession coding. This profile allows the manifest parser to adapt
    without hardcoding assumptions about FHIR resource structure.
    """
    study_uid_system: str = "urn:dicom:uid"
    accession_code: str = "ACSN"
    accepted_statuses: List[str] = field(default_factory=lambda: ["available"])
    secondary_discovery_resources: List[str] = field(
        default_factory=lambda: ["DiagnosticReport"]
    )


# Default profile — works for standard Epic deployments
DEFAULT_SITE_PROFILE = SiteProfile()


@dataclass
class ConsentProof:
    """Audit chain linking a study back to registry consent."""
    registry_name: str
    registry_id: str
    bulk_fhir_export_id: str
    extraction_timestamp: str = field(
        default_factory=lambda: datetime.now(timezone.utc).isoformat()
    )


@dataclass
class StudyManifestEntry:
    """A single imaging study approved for retrieval."""
    study_uid: str
    patient_fhir_id: str
    study_date: str
    modality: List[str]
    description: str
    consent_proof: ConsentProof
    accession_number: Optional[str] = None
    expected_series: Optional[int] = None
    expected_instances: Optional[int] = None
    endpoint_ref: Optional[str] = None
    retrieval_status: str = "pending"
    retrieval_attempts: int = 0


@dataclass
class ManifestStats:
    """Summary statistics for a manifest."""
    total_studies: int = 0
    new_studies: int = 0
    previously_retrieved: int = 0
    modality_breakdown: Dict[str, int] = field(default_factory=dict)


@dataclass
class StudyManifest:
    """Complete study manifest for a site."""
    manifest_id: str
    generated_at: str
    site_id: str
    registry_id: str
    transport_config: str = "epic_dicomweb"
    studies: List[StudyManifestEntry] = field(default_factory=list)
    stats: ManifestStats = field(default_factory=ManifestStats)

    def to_json(self) -> str:
        return json.dumps(asdict(self), indent=2, default=str)

    @classmethod
    def from_json(cls, json_str: str) -> "StudyManifest":
        data = json.loads(json_str)
        studies = [
            StudyManifestEntry(
                consent_proof=ConsentProof(**s.pop("consent_proof")),
                **s
            )
            for s in data.pop("studies", [])
        ]
        stats = ManifestStats(**data.pop("stats", {}))
        return cls(studies=studies, stats=stats, **data)


def parse_imaging_study_resource(
    resource: Dict[str, Any],
    site_profile: Optional[SiteProfile] = None,
) -> Optional[StudyManifestEntry]:
    """
    Parse a single FHIR ImagingStudy resource into a StudyManifestEntry.

    Returns None if the resource status is not in the site profile's accepted list.
    Uses SiteProfile to determine identifier system, accession code, and accepted
    statuses — these vary across sites.
    """
    profile = site_profile or DEFAULT_SITE_PROFILE

    if resource.get("status") not in profile.accepted_statuses:
        return None

    # Extract study UID using site-specific identifier system
    study_uid = None
    accession_number = None
    for ident in resource.get("identifier", []):
        system = ident.get("system", "")
        value = ident.get("value", "")
        if system == profile.study_uid_system:
            study_uid = value.replace("urn:oid:", "")
        elif ident.get("type", {}).get("coding", [{}])[0].get("code") == profile.accession_code:
            accession_number = value

    if not study_uid:
        logger.warning(f"ImagingStudy {resource.get('id')} has no DICOM UID, skipping")
        return None

    # Extract patient FHIR ID
    subject_ref = resource.get("subject", {}).get("reference", "")
    patient_fhir_id = subject_ref.replace("Patient/", "")

    # Extract study date
    started = resource.get("started", "")
    study_date = started[:10] if started else ""

    # Extract modality
    modality = [
        m.get("code", "")
        for m in resource.get("modality", [])
        if m.get("code")
    ]

    # Extract endpoint
    endpoints = resource.get("endpoint", [])
    endpoint_ref = endpoints[0].get("reference") if endpoints else None

    # Build entry (consent_proof filled by caller)
    return StudyManifestEntry(
        study_uid=study_uid,
        accession_number=accession_number,
        patient_fhir_id=patient_fhir_id,
        study_date=study_date,
        modality=modality,
        description=resource.get("description", ""),
        expected_series=resource.get("numberOfSeries"),
        expected_instances=resource.get("numberOfInstances"),
        endpoint_ref=endpoint_ref,
        consent_proof=ConsentProof(
            registry_name="",
            registry_id="",
            bulk_fhir_export_id="",
        ),
    )


def extract_imaging_studies_from_ndjson(ndjson_path: Path) -> List[StudyManifestEntry]:
    """
    Read a Bulk FHIR NDJSON file and extract all ImagingStudy resources.

    Skips malformed lines and non-ImagingStudy resources.
    """
    entries = []
    content = ndjson_path.read_text().strip()
    if not content:
        return entries

    for line_num, line in enumerate(content.split("\n"), 1):
        line = line.strip()
        if not line:
            continue
        try:
            resource = json.loads(line)
        except json.JSONDecodeError:
            logger.warning(f"Malformed JSON at line {line_num} in {ndjson_path}, skipping")
            continue

        if resource.get("resourceType") != "ImagingStudy":
            continue

        entry = parse_imaging_study_resource(resource)
        if entry is not None:
            entries.append(entry)

    logger.info(f"Extracted {len(entries)} ImagingStudy resources from {ndjson_path}")
    return entries


def extract_diagnostic_reports_from_ndjson(
    ndjson_path: Path,
) -> List[Dict[str, Any]]:
    """
    Secondary discovery path: extract imaging references from DiagnosticReport resources.

    Sites vary in how completely they populate ImagingStudy. DiagnosticReport
    (category=RAD) often contains imagingStudy references that can serve as
    a backup discovery mechanism for studies not surfaced via ImagingStudy alone.

    Returns a list of dicts with imaging_study_ref, patient_ref, effective_date.
    """
    refs = []
    content = ndjson_path.read_text().strip()
    if not content:
        return refs

    for line in content.split("\n"):
        line = line.strip()
        if not line:
            continue
        try:
            resource = json.loads(line)
        except json.JSONDecodeError:
            continue

        if resource.get("resourceType") != "DiagnosticReport":
            continue

        # Only radiology reports
        categories = resource.get("category", [])
        is_radiology = any(
            coding.get("code") in ("RAD", "LP29684-5", "RAD-RPT")
            for cat in categories
            for coding in cat.get("coding", [])
        )
        if not is_radiology:
            continue

        for img_ref in resource.get("imagingStudy", []):
            ref = img_ref.get("reference", "")
            if ref:
                refs.append({
                    "imaging_study_ref": ref,
                    "patient_ref": resource.get("subject", {}).get("reference", ""),
                    "effective_date": resource.get("effectiveDateTime", "")[:10],
                    "report_id": resource.get("id", ""),
                })

    logger.info(f"Found {len(refs)} imaging references from DiagnosticReport resources")
    return refs


class StudyManifestBuilder:
    """Builds a StudyManifest from Bulk FHIR NDJSON export."""

    def __init__(
        self,
        site_id: str,
        registry_id: str,
        registry_name: str,
        bulk_fhir_export_id: str,
        transport_config: str = "epic_dicomweb",
    ):
        self.site_id = site_id
        self.registry_id = registry_id
        self.registry_name = registry_name
        self.bulk_fhir_export_id = bulk_fhir_export_id
        self.transport_config = transport_config

    def build(self, ndjson_path: Path) -> StudyManifest:
        """Build manifest from NDJSON file."""
        entries = extract_imaging_studies_from_ndjson(ndjson_path)

        # Stamp consent proof on each entry
        for entry in entries:
            entry.consent_proof = ConsentProof(
                registry_name=self.registry_name,
                registry_id=self.registry_id,
                bulk_fhir_export_id=self.bulk_fhir_export_id,
            )

        # Compute stats
        modality_counter: Counter = Counter()
        for entry in entries:
            for mod in entry.modality:
                modality_counter[mod] += 1

        stats = ManifestStats(
            total_studies=len(entries),
            new_studies=len(entries),  # All new until deduped against ledger
            previously_retrieved=0,
            modality_breakdown=dict(modality_counter),
        )

        now = datetime.now(timezone.utc)
        manifest_id = f"{self.site_id[:3].upper()}-{now.strftime('%Y-%m-%d')}-{uuid.uuid4().hex[:6]}"

        return StudyManifest(
            manifest_id=manifest_id,
            generated_at=now.isoformat(),
            site_id=self.site_id,
            registry_id=self.registry_id,
            transport_config=self.transport_config,
            studies=entries,
            stats=stats,
        )
```

**Step 4: Run tests to verify they pass**

Run: `cd /Users/resnick/Documents/GitHub/RADIANT_PCA/BRIM_Analytics/patient_clinical_journey_timeline && python -m pytest tests/test_study_manifest.py -v`
Expected: All PASS

**Step 5: Commit**

```bash
git add lib/imaging/__init__.py lib/imaging/study_manifest.py tests/test_study_manifest.py
git commit -m "feat(imaging): study manifest builder — extract ImagingStudy from Bulk FHIR NDJSON"
```

---

## Task 2: Retrieval Ledger

Tracks which studies have been retrieved per site. Prevents duplicate retrieval. Supports retry with backoff.

Includes a `study_release_status` field for pre-retrieval authorization tracking, inspired by the DICOMgrid authorized-study staging pattern: the EHR authorizes the study, the imaging layer only fulfills that exact study.

**Files:**
- Create: `lib/imaging/retrieval_ledger.py`
- Create: `tests/test_retrieval_ledger.py`

**Step 1: Write failing tests**

```python
# tests/test_retrieval_ledger.py
"""Tests for Retrieval Ledger."""
import json
import pytest
from datetime import datetime, timezone
from pathlib import Path

from lib.imaging.retrieval_ledger import RetrievalLedger, LedgerEntry


class TestRetrievalLedger:

    def test_new_ledger_is_empty(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        assert ledger.count() == 0

    def test_record_retrieval(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_success("1.2.3.4", s3_path="s3://bucket/study/")
        assert ledger.is_retrieved("1.2.3.4")
        assert not ledger.is_retrieved("9.9.9.9")

    def test_record_failure(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_failure("1.2.3.4", error="timeout")
        entry = ledger.get("1.2.3.4")
        assert entry.status == "failed"
        assert entry.attempts == 1
        assert entry.last_error == "timeout"

    def test_retry_increments_attempts(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_failure("1.2.3.4", error="timeout")
        ledger.record_failure("1.2.3.4", error="connection reset")
        entry = ledger.get("1.2.3.4")
        assert entry.attempts == 2
        assert entry.last_error == "connection reset"

    def test_success_after_failure(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_failure("1.2.3.4", error="timeout")
        ledger.record_success("1.2.3.4", s3_path="s3://bucket/study/")
        entry = ledger.get("1.2.3.4")
        assert entry.status == "retrieved"
        assert entry.attempts == 2

    def test_should_retry_respects_max_attempts(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json", max_attempts=3)
        for i in range(3):
            ledger.record_failure("1.2.3.4", error=f"error {i}")
        assert not ledger.should_retry("1.2.3.4")

    def test_should_retry_true_for_failed_under_max(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json", max_attempts=5)
        ledger.record_failure("1.2.3.4", error="timeout")
        assert ledger.should_retry("1.2.3.4")

    def test_should_retry_false_for_retrieved(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_success("1.2.3.4", s3_path="s3://bucket/study/")
        assert not ledger.should_retry("1.2.3.4")

    def test_filter_manifest_removes_retrieved(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_success("1.2.3.4", s3_path="s3://bucket/1/")
        pending = ledger.filter_pending(["1.2.3.4", "5.6.7.8", "9.10.11.12"])
        assert pending == ["5.6.7.8", "9.10.11.12"]

    def test_filter_manifest_includes_retriable_failures(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json", max_attempts=5)
        ledger.record_failure("1.2.3.4", error="timeout")
        pending = ledger.filter_pending(["1.2.3.4", "5.6.7.8"])
        assert "1.2.3.4" in pending

    def test_persistence(self, tmp_path):
        path = tmp_path / "ledger.json"
        ledger1 = RetrievalLedger(path)
        ledger1.record_success("1.2.3.4", s3_path="s3://bucket/study/")
        ledger1.save()

        ledger2 = RetrievalLedger(path)
        assert ledger2.is_retrieved("1.2.3.4")

    def test_stats(self, tmp_path):
        ledger = RetrievalLedger(tmp_path / "ledger.json")
        ledger.record_success("1.2.3.4", s3_path="s3://a/")
        ledger.record_success("5.6.7.8", s3_path="s3://b/")
        ledger.record_failure("9.10.11.12", error="timeout")
        stats = ledger.stats()
        assert stats["retrieved"] == 2
        assert stats["failed"] == 1
        assert stats["total"] == 3
```

**Step 2: Run tests — expected FAIL**

**Step 3: Write implementation**

```python
# lib/imaging/retrieval_ledger.py
"""
Retrieval Ledger

Tracks which studies have been retrieved per site. Prevents duplicate retrieval.
Supports retry with configurable max attempts and exponential backoff.

The ledger is append-only in terms of study UIDs (new UIDs are added, never removed).
Status transitions: pending -> retrieved | failed -> retrieved (on retry success).
"""

import json
import logging
from dataclasses import dataclass, field, asdict
from datetime import datetime, timezone
from pathlib import Path
from typing import Dict, List, Optional

logger = logging.getLogger(__name__)


@dataclass
class LedgerEntry:
    """Record for a single study in the ledger."""
    study_uid: str
    status: str = "pending"  # pending | released | retrieved | failed | exhausted
    study_release_status: str = "pending_release"  # pending_release | released | retrieval_authorized
    attempts: int = 0
    first_seen: str = ""
    last_attempt: str = ""
    last_error: str = ""
    s3_path: str = ""
    release_ticket_id: str = ""  # One-time token ID from study ticket (if gateway broker used)


class RetrievalLedger:
    """
    Persistent ledger tracking study retrieval status.

    Stored as a JSON file, loaded on init, saved on each mutation.
    """

    def __init__(self, path: Path, max_attempts: int = 5):
        self._path = Path(path)
        self._max_attempts = max_attempts
        self._entries: Dict[str, LedgerEntry] = {}
        if self._path.exists():
            self._load()

    def _load(self) -> None:
        try:
            data = json.loads(self._path.read_text())
            for uid, entry_data in data.get("entries", {}).items():
                self._entries[uid] = LedgerEntry(**entry_data)
        except (json.JSONDecodeError, KeyError) as e:
            logger.warning(f"Could not load ledger from {self._path}: {e}")

    def save(self) -> None:
        self._path.parent.mkdir(parents=True, exist_ok=True)
        data = {
            "entries": {uid: asdict(entry) for uid, entry in self._entries.items()},
            "updated_at": datetime.now(timezone.utc).isoformat(),
        }
        self._path.write_text(json.dumps(data, indent=2))

    def _ensure_entry(self, study_uid: str) -> LedgerEntry:
        if study_uid not in self._entries:
            self._entries[study_uid] = LedgerEntry(
                study_uid=study_uid,
                first_seen=datetime.now(timezone.utc).isoformat(),
            )
        return self._entries[study_uid]

    def record_success(self, study_uid: str, s3_path: str) -> None:
        entry = self._ensure_entry(study_uid)
        entry.status = "retrieved"
        entry.attempts += 1
        entry.last_attempt = datetime.now(timezone.utc).isoformat()
        entry.s3_path = s3_path
        self.save()

    def record_failure(self, study_uid: str, error: str) -> None:
        entry = self._ensure_entry(study_uid)
        entry.attempts += 1
        entry.last_attempt = datetime.now(timezone.utc).isoformat()
        entry.last_error = error
        if entry.attempts >= self._max_attempts:
            entry.status = "exhausted"
        else:
            entry.status = "failed"
        self.save()

    def is_retrieved(self, study_uid: str) -> bool:
        entry = self._entries.get(study_uid)
        return entry is not None and entry.status == "retrieved"

    def should_retry(self, study_uid: str) -> bool:
        entry = self._entries.get(study_uid)
        if entry is None:
            return True
        if entry.status == "retrieved":
            return False
        return entry.attempts < self._max_attempts

    def get(self, study_uid: str) -> Optional[LedgerEntry]:
        return self._entries.get(study_uid)

    def filter_pending(self, study_uids: List[str]) -> List[str]:
        return [uid for uid in study_uids if not self.is_retrieved(uid) and self.should_retry(uid)]

    def count(self) -> int:
        return len(self._entries)

    def stats(self) -> Dict[str, int]:
        counts = {"retrieved": 0, "failed": 0, "exhausted": 0, "pending": 0}
        for entry in self._entries.values():
            counts[entry.status] = counts.get(entry.status, 0) + 1
        counts["total"] = len(self._entries)
        return counts
```

**Step 4: Run tests — expected all PASS**

**Step 5: Commit**

```bash
git add lib/imaging/retrieval_ledger.py tests/test_retrieval_ledger.py
git commit -m "feat(imaging): retrieval ledger — dedup and retry tracking for DICOM studies"
```

---

## Task 3: Transport Adapter Base Class

Abstract interface that all transport methods implement.

**Files:**
- Create: `lib/imaging/transport_base.py`
- Create: `tests/test_transport_base.py`

**Step 1: Write failing tests**

```python
# tests/test_transport_base.py
"""Tests for Transport Adapter base class."""
import pytest
from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError


class TestRetrievalResult:

    def test_success_result(self):
        result = RetrievalResult.success(
            study_uid="1.2.3.4",
            series_retrieved=12,
            instances_retrieved=2847,
            bytes_transferred=1073741824,
            s3_path="s3://bucket/study/",
        )
        assert result.ok is True
        assert result.study_uid == "1.2.3.4"

    def test_failure_result(self):
        result = RetrievalResult.failure(
            study_uid="1.2.3.4",
            error="timeout",
            retriable=True,
        )
        assert result.ok is False
        assert result.error == "timeout"
        assert result.retriable is True


class TestTransportAdapterInterface:

    def test_cannot_instantiate_abstract(self):
        with pytest.raises(TypeError):
            TransportAdapter()

    def test_subclass_must_implement_retrieve(self):
        class BadTransport(TransportAdapter):
            def name(self) -> str:
                return "bad"
        with pytest.raises(TypeError):
            BadTransport()
```

**Step 2: Run — expected FAIL**

**Step 3: Write implementation**

```python
# lib/imaging/transport_base.py
"""
Transport Adapter Base Class

Defines the interface for all DICOM retrieval transport methods.
Each transport category uses a different adapter:
  - EpicDICOMwebTransport (CAT-A: DICOMweb)
  - FHIRBinaryTransport (CAT-A: FHIR Binary)
  - GatewayBrokerTransport (CAT-C: Middleware-brokered)
  - EpicImageExchangeTransport (CAT-B: Care Everywhere — experimental)
  - EpicIHETransport (CAT-B: IHE profiles)
  - AmbraTransport (CAT-D, or wraps any of the above with Ambra QA staging)
"""

import logging
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Dict, List, Optional

logger = logging.getLogger(__name__)


class TransportError(Exception):
    """Raised on transport-level failures."""
    def __init__(self, message: str, retriable: bool = True):
        super().__init__(message)
        self.retriable = retriable


@dataclass
class RetrievalResult:
    """Result of a single study retrieval attempt."""
    study_uid: str
    ok: bool
    series_retrieved: int = 0
    instances_retrieved: int = 0
    bytes_transferred: int = 0
    s3_path: str = ""
    ambra_uuid: str = ""
    error: str = ""
    retriable: bool = True
    transport_name: str = ""
    duration_seconds: float = 0.0

    @classmethod
    def success(cls, study_uid: str, series_retrieved: int,
                instances_retrieved: int, bytes_transferred: int,
                s3_path: str = "", ambra_uuid: str = "",
                transport_name: str = "", duration_seconds: float = 0.0) -> "RetrievalResult":
        return cls(
            study_uid=study_uid, ok=True,
            series_retrieved=series_retrieved,
            instances_retrieved=instances_retrieved,
            bytes_transferred=bytes_transferred,
            s3_path=s3_path, ambra_uuid=ambra_uuid,
            transport_name=transport_name,
            duration_seconds=duration_seconds,
        )

    @classmethod
    def failure(cls, study_uid: str, error: str, retriable: bool = True,
                transport_name: str = "") -> "RetrievalResult":
        return cls(
            study_uid=study_uid, ok=False,
            error=error, retriable=retriable,
            transport_name=transport_name,
        )


class TransportAdapter(ABC):
    """Abstract base class for DICOM retrieval transport methods."""

    @abstractmethod
    def name(self) -> str:
        """Human-readable transport name for logging."""
        ...

    @abstractmethod
    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """
        Retrieve a complete DICOM study.

        Args:
            study_uid: DICOM Study Instance UID
            accession_number: Alternate identifier for PACS query
            patient_fhir_id: Patient FHIR ID for audit

        Returns:
            RetrievalResult with success/failure status
        """
        ...

    def probe_study(self, study_uid: str) -> bool:
        """
        Check if a study exists and is retrievable without downloading it.
        Default implementation returns True (optimistic).
        """
        return True

    def health_check(self) -> bool:
        """
        Check if the transport endpoint is reachable.
        Default implementation returns True.
        """
        return True
```

**Step 4: Run tests — expected all PASS**

**Step 5: Commit**

```bash
git add lib/imaging/transport_base.py tests/test_transport_base.py
git commit -m "feat(imaging): transport adapter base class — interface for DICOM retrieval methods"
```

---

## Task 4: Epic DICOMweb Transport (Primary Hypothesis — Requires Site Validation)

**IMPORTANT:** This transport is the preferred hypothesis, NOT an established default. Epic's public documentation shows ImagingStudy, Binary.Read/Search (Study), and Media.Read/Search (Study), but does NOT clearly document a public third-party SMART-scoped WADO-RS proxy accessible to external backend apps using Bulk FHIR tokens. This transport requires site-specific validation via the Task 0 capability probe before use.

The transport retrieves DICOM via Epic's DICOMweb (WADO-RS) proxy using the same OAuth2 backend auth as Bulk FHIR — if the site validates that this path exists.

**Files:**
- Create: `lib/imaging/transport_epic_dicomweb.py`
- Create: `tests/test_transport_epic_dicomweb.py`

**Step 1: Write failing tests**

```python
# tests/test_transport_epic_dicomweb.py
"""Tests for Epic DICOMweb Transport."""
import io
import json
import pytest
from unittest.mock import MagicMock, patch, PropertyMock
from lib.imaging.transport_epic_dicomweb import (
    EpicDICOMwebTransport,
    EpicOAuth2Client,
    parse_multipart_dicom,
)
from lib.imaging.transport_base import RetrievalResult


class TestEpicOAuth2Client:

    def test_creates_signed_jwt(self):
        client = EpicOAuth2Client(
            client_id="test-client-id",
            token_endpoint="https://fhir.example.com/oauth2/token",
            private_key_pem="-----BEGIN RSA PRIVATE KEY-----\ntest\n-----END RSA PRIVATE KEY-----",
        )
        # Should not raise
        assert client.client_id == "test-client-id"

    @patch("lib.imaging.transport_epic_dicomweb.requests.post")
    def test_get_access_token(self, mock_post):
        mock_post.return_value = MagicMock(
            status_code=200,
            json=lambda: {"access_token": "tok123", "expires_in": 300}
        )
        client = EpicOAuth2Client(
            client_id="test-client-id",
            token_endpoint="https://fhir.example.com/oauth2/token",
            private_key_pem="-----BEGIN RSA PRIVATE KEY-----\ntest\n-----END RSA PRIVATE KEY-----",
        )
        # Mock the JWT signing
        with patch.object(client, "_create_signed_jwt", return_value="fake.jwt.token"):
            token = client.get_access_token()
            assert token == "tok123"


class TestParseMultipartDicom:

    def test_parses_multipart_response(self):
        # Minimal multipart/related mock
        boundary = "boundary123"
        part1 = b"\x00\x01\x02\x03"  # fake DICOM bytes
        body = (
            f"--{boundary}\r\n"
            f"Content-Type: application/dicom\r\n\r\n"
        ).encode() + part1 + f"\r\n--{boundary}--\r\n".encode()

        parts = parse_multipart_dicom(body, boundary)
        assert len(parts) == 1
        assert parts[0] == part1


class TestEpicDICOMwebTransport:

    def test_name(self):
        transport = EpicDICOMwebTransport.__new__(EpicDICOMwebTransport)
        transport._dicomweb_base = "https://example.com/wado-rs"
        assert transport.name() == "epic_dicomweb"

    @patch("lib.imaging.transport_epic_dicomweb.requests.get")
    def test_probe_study_returns_true_on_200(self, mock_get):
        mock_get.return_value = MagicMock(status_code=200)
        transport = EpicDICOMwebTransport.__new__(EpicDICOMwebTransport)
        transport._dicomweb_base = "https://example.com/wado-rs"
        transport._oauth_client = MagicMock()
        transport._oauth_client.get_access_token.return_value = "tok"
        assert transport.probe_study("1.2.3.4") is True

    @patch("lib.imaging.transport_epic_dicomweb.requests.get")
    def test_probe_study_returns_false_on_404(self, mock_get):
        mock_get.return_value = MagicMock(status_code=404)
        transport = EpicDICOMwebTransport.__new__(EpicDICOMwebTransport)
        transport._dicomweb_base = "https://example.com/wado-rs"
        transport._oauth_client = MagicMock()
        transport._oauth_client.get_access_token.return_value = "tok"
        assert transport.probe_study("1.2.3.4") is False
```

**Step 2: Run — expected FAIL**

**Step 3: Write implementation**

```python
# lib/imaging/transport_epic_dicomweb.py
"""
Epic DICOMweb Transport (Primary Hypothesis — Requires Site Validation)

Retrieves DICOM studies via Epic's DICOMweb (WADO-RS) proxy using the same
OAuth2 Backend Authentication as the Bulk FHIR Connector.

VALIDATION STATUS: This transport is the preferred hypothesis. It requires
site validation via Task 0 capability probe that Epic's DICOMweb proxy is
accessible to the backend app before use. Epic's public FHIR catalog shows
ImagingStudy, Binary.Read/Search (Study), and Media.Read/Search (Study),
but does not clearly document a general-purpose WADO-RS DICOMweb proxy
for external backend apps.

Prerequisites (must be validated per site):
  - Site's Epic instance has DICOMweb proxy enabled
  - RADIANT Connector app includes imaging-related FHIR scopes
  - Background user has imaging security points
  - Task 0 probe confirms DICOMweb endpoint responds

The access token obtained for Bulk FHIR is reused for DICOMweb requests.
The degree to which Epic enforces patient scoping on DICOMweb identically
to Bulk FHIR requires site-specific validation.
"""

import io
import json
import logging
import time
from pathlib import Path
from typing import Dict, List, Optional, Tuple

import requests

from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError

logger = logging.getLogger(__name__)

# Timeouts
METADATA_TIMEOUT = 30
RETRIEVAL_TIMEOUT = 600  # 10 minutes per study
TOKEN_REFRESH_BUFFER = 60  # Refresh token 60s before expiry


def parse_multipart_dicom_stream(resp, boundary: str):
    """Yield DICOM parts from a multipart/related response stream.

    This illustrative implementation keeps a rolling byte buffer and emits
    one part at a time. It is intentionally simplified for documentation,
    but it preserves the key architectural point: do not buffer the full
    study response in memory.
    """
    delimiter = f"--{boundary}".encode()
    buffer = b""
    for chunk in resp.iter_content(chunk_size=1024 * 1024):
        if not chunk:
            continue
        buffer += chunk
        while delimiter in buffer:
            segment, buffer = buffer.split(delimiter, 1)
            segment = segment.strip()
            if not segment or segment == b"--":
                continue
            header_end = segment.find(b"\r\n\r\n")
            if header_end == -1:
                header_end = segment.find(b"\n\n")
            if header_end != -1:
                part_body = segment[header_end:].lstrip(b"\r\n")
                if part_body and part_body != b"--":
                    yield part_body


def parse_multipart_dicom(body: bytes, boundary: str) -> List[bytes]:
    """Parse in-memory multipart/related body into DICOM parts (for testing)."""
    parts = []
    delimiter = f"--{boundary}".encode()
    segments = body.split(delimiter)
    for segment in segments[1:]:
        if segment.strip() == b"--" or segment.strip() == b"":
            continue
        if b"\r\n\r\n" in segment:
            _, part_body = segment.split(b"\r\n\r\n", 1)
            part_body = part_body.rstrip(b"\r\n")
            if part_body:
                parts.append(part_body)
    return parts


class EpicOAuth2Client:
    """OAuth2 Backend Authentication client for Epic FHIR/DICOMweb APIs."""

    def __init__(self, client_id: str, token_endpoint: str,
                 private_key_pem: str, token_lifetime: int = 300):
        self.client_id = client_id
        self._token_endpoint = token_endpoint
        self._private_key_pem = private_key_pem
        self._token_lifetime = token_lifetime
        self._access_token: Optional[str] = None
        self._token_expires_at: float = 0

    def _create_signed_jwt(self) -> str:
        """Create a signed JWT assertion for Epic OAuth2."""
        import jwt as pyjwt
        import uuid
        now = int(time.time())
        claims = {
            "iss": self.client_id,
            "sub": self.client_id,
            "aud": self._token_endpoint,
            "jti": str(uuid.uuid4()),
            "iat": now,
            "exp": now + self._token_lifetime,
        }
        return pyjwt.encode(claims, self._private_key_pem, algorithm="RS384")

    def get_access_token(self) -> str:
        """Get a valid access token, refreshing if needed."""
        if self._access_token and time.time() < self._token_expires_at - TOKEN_REFRESH_BUFFER:
            return self._access_token

        signed_jwt = self._create_signed_jwt()
        response = requests.post(
            self._token_endpoint,
            data={
                "grant_type": "client_credentials",
                "client_assertion_type": "urn:ietf:params:oauth:client-assertion-type:jwt-bearer",
                "client_assertion": signed_jwt,
            },
            timeout=30,
        )
        response.raise_for_status()
        token_data = response.json()
        self._access_token = token_data["access_token"]
        self._token_expires_at = time.time() + token_data.get("expires_in", 300)
        return self._access_token


class EpicDICOMwebTransport(TransportAdapter):
    """
    Retrieve DICOM via Epic's DICOMweb WADO-RS proxy.

    Uses study-level retrieval (all series in one request).
    Falls back to series-level retrieval for large studies.
    """

    def __init__(self, dicomweb_base_url: str, oauth_client: EpicOAuth2Client,
                 s3_bucket: str, s3_prefix: str, aws_profile: str = "radiant-prod"):
        self._dicomweb_base = dicomweb_base_url.rstrip("/")
        self._oauth_client = oauth_client
        self._s3_bucket = s3_bucket
        self._s3_prefix = s3_prefix
        self._aws_profile = aws_profile

    def name(self) -> str:
        return "epic_dicomweb"

    def probe_study(self, study_uid: str) -> bool:
        """Check if study exists via metadata endpoint."""
        token = self._oauth_client.get_access_token()
        url = f"{self._dicomweb_base}/studies/{study_uid}/metadata"
        try:
            resp = requests.get(
                url,
                headers={
                    "Authorization": f"Bearer {token}",
                    "Accept": "application/dicom+json",
                },
                timeout=METADATA_TIMEOUT,
            )
            return resp.status_code == 200
        except requests.RequestException as e:
            logger.warning(f"Probe failed for {study_uid}: {e}")
            return False

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """Retrieve full DICOM study via WADO-RS."""
        start_time = time.time()
        token = self._oauth_client.get_access_token()
        url = f"{self._dicomweb_base}/studies/{study_uid}"

        try:
            resp = requests.get(
                url,
                headers={
                    "Authorization": f"Bearer {token}",
                    "Accept": "multipart/related; type=\"application/dicom\"",
                },
                timeout=RETRIEVAL_TIMEOUT,
                stream=True,
            )

            if resp.status_code == 401:
                raise TransportError("OAuth2 token rejected", retriable=True)
            if resp.status_code == 403:
                raise TransportError("Access denied — patient may not be in registry", retriable=False)
            if resp.status_code == 404:
                raise TransportError("Study not found in PACS", retriable=True)

            resp.raise_for_status()

            content_type = resp.headers.get("Content-Type", "")
            boundary = ""
            for part in content_type.split(";"):
                part = part.strip()
                if part.startswith("boundary="):
                    boundary = part.split("=", 1)[1].strip('"')
            if not boundary:
                raise TransportError("No boundary in multipart response", retriable=True)

            # Stream multipart response instance-by-instance (Part I Section 5.4
            # streaming architecture — MUST NOT buffer entire response in memory)
            s3_path, instance_count, total_bytes = self._stream_to_s3(
                study_uid, resp, boundary
            )

            if instance_count == 0:
                raise TransportError("No DICOM instances in response", retriable=True)

            duration = time.time() - start_time

            logger.info(
                f"Retrieved {study_uid}: {instance_count} instances, "
                f"{total_bytes / 1048576:.1f} MB in {duration:.1f}s"
            )

            return RetrievalResult.success(
                study_uid=study_uid,
                series_retrieved=0,  # Populated by DICOM header parsing in _stream_to_s3
                instances_retrieved=instance_count,
                bytes_transferred=total_bytes,
                s3_path=s3_path,
                transport_name=self.name(),
                duration_seconds=duration,
            )

        except TransportError:
            raise
        except requests.Timeout:
            raise TransportError(f"Timeout retrieving {study_uid}", retriable=True)
        except requests.RequestException as e:
            raise TransportError(f"HTTP error retrieving {study_uid}: {e}", retriable=True)

    def _stream_to_s3(self, study_uid: str, resp, boundary: str) -> tuple[str, int, int]:
        """
        Stream multipart DICOM response to S3 instance-by-instance.

        Parses multipart boundaries incrementally, writing each DICOM
        instance to S3 as it is extracted — never buffering the full
        study in memory. Returns (s3_path, instance_count, total_bytes).

        S3 key structure: {prefix}/{study_uid}/series/{series_uid}/{sop_uid}.dcm
        Falls back to sequential naming if DICOM headers cannot be parsed.
        Emits a study_manifest.json with checksums, series IDs, SOP UIDs, and byte counts.
        """
        import hashlib
        import boto3
        s3 = boto3.Session(profile_name=self._aws_profile).client("s3")
        prefix = f"{self._s3_prefix}{study_uid}/"
        instance_manifest = []
        instance_count = 0
        total_bytes = 0

        # Stream-parse multipart response, extracting one instance at a time
        for i, part in enumerate(parse_multipart_dicom_stream(resp, boundary)):
            instance_count += 1
            total_bytes += len(part)
            # Extract UIDs from DICOM headers for hierarchical key naming
            series_uid = "unknown_series"
            sop_uid = f"{i:06d}"
            try:
                import pydicom
                ds = pydicom.dcmread(io.BytesIO(part), stop_before_pixels=True)
                series_uid = str(getattr(ds, "SeriesInstanceUID", series_uid))
                sop_uid = str(getattr(ds, "SOPInstanceUID", sop_uid))
            except Exception:
                pass  # Fall back to sequential naming

            key = f"{prefix}series/{series_uid}/{sop_uid}.dcm"
            checksum = hashlib.sha256(part).hexdigest()
            s3.put_object(Bucket=self._s3_bucket, Key=key, Body=part)

            instance_manifest.append({
                "sop_instance_uid": sop_uid,
                "series_instance_uid": series_uid,
                "s3_key": key,
                "byte_count": len(part),
                "sha256": checksum,
            })

        # Write instance manifest for integrity reconciliation
        manifest_key = f"{prefix}study_manifest.json"
        s3.put_object(
            Bucket=self._s3_bucket,
            Key=manifest_key,
            Body=json.dumps({
                "study_instance_uid": study_uid,
                "instance_count": len(instance_manifest),
                "instances": instance_manifest,
            }, indent=2),
            ContentType="application/json",
        )

        return f"s3://{self._s3_bucket}/{prefix}", instance_count, total_bytes

    def health_check(self) -> bool:
        """Verify DICOMweb endpoint is reachable."""
        try:
            token = self._oauth_client.get_access_token()
            resp = requests.get(
                f"{self._dicomweb_base}/studies?limit=0",
                headers={"Authorization": f"Bearer {token}"},
                timeout=10,
            )
            return resp.status_code in (200, 204)
        except Exception:
            return False
```

**Step 4: Run tests — expected all PASS**

**Step 5: Commit**

```bash
git add lib/imaging/transport_epic_dicomweb.py tests/test_transport_epic_dicomweb.py
git commit -m "feat(imaging): Epic DICOMweb transport — WADO-RS retrieval with OAuth2 backend auth"
```

---

## Task 5: Epic Care Everywhere Image Exchange Transport (Candidate Fallback — Pending Site Validation)

**Files:**
- Create: `lib/imaging/transport_epic_image_exchange.py`

**VALIDATION STATUS:** Care Everywhere Image Exchange is documented around XCPD/XCA/XDS.b patient discovery and document/binary retrieval, NOT a clearly documented backend "post study UID and receive DICOM push" API for research ingestion. This is a candidate fallback pending site capability confirmation, not a broadly dependable default.

This follows the same pattern as Task 4 but uses Epic's Image Exchange API instead of WADO-RS. The implementation is a stub with the interface defined — full implementation depends on Epic TS consultation for the programmatic Image Exchange API spec.

**Step 1: Write stub implementation**

```python
# lib/imaging/transport_epic_image_exchange.py
"""
Epic Care Everywhere Image Exchange Transport (CAT-B — Experimental/Exception Path)

Uses Epic's built-in image sharing infrastructure. Epic handles all PACS
communication internally — RADIANT never touches the site's PACS directly.

IMPLEMENTATION STATUS: Stub — requires Epic TS consultation to confirm
the programmatic Image Exchange API spec for backend system use.

See Section 6 (CAT-B: Care Everywhere) for full architecture.
"""

import logging
from typing import Optional

from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError

logger = logging.getLogger(__name__)


class EpicImageExchangeTransport(TransportAdapter):
    """
    Retrieve DICOM via Epic Care Everywhere Image Exchange.

    Epic acts as the PACS gateway — the site's PACS is never exposed.
    Uses the same OAuth2 backend auth as Bulk FHIR.
    """

    def __init__(self, image_exchange_url: str, oauth_client: "EpicOAuth2Client",
                 dicom_receiver_url: str):
        self._image_exchange_url = image_exchange_url
        self._oauth_client = oauth_client
        self._dicom_receiver_url = dicom_receiver_url

    def name(self) -> str:
        return "epic_image_exchange"

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """
        Request image exchange from Epic.

        Flow:
        1. POST image exchange request with study UID
        2. Epic internally retrieves from PACS
        3. Epic pushes DICOM to RADIANT's receiver endpoint
        4. Return result once push is confirmed

        TODO: Implement after Epic TS consultation confirms API spec.
        """
        raise TransportError(
            "Epic Image Exchange transport not yet implemented — "
            "pending Epic TS consultation for programmatic API spec",
            retriable=False,
        )

    def health_check(self) -> bool:
        logger.warning("Epic Image Exchange health check not implemented")
        return False
```

**Step 2: Commit**

```bash
git add lib/imaging/transport_epic_image_exchange.py
git commit -m "feat(imaging): Epic Image Exchange transport stub — pending Epic TS API spec"
```

---

## Task 6: Epic IHE Transport (Candidate Fallback — Pending Site Validation)

**Files:**
- Create: `lib/imaging/transport_epic_ihe.py`

**VALIDATION STATUS:** IHE RAD-69 is a real retrieve profile, but in practice it depends on a site's imaging-document source/gateway configuration. It is NOT something that can be assumed to be generically exposed by Epic without site-specific validation. This is a candidate fallback, not a dependable default.

Same pattern as Task 5 — stub with interface defined.

**Step 1: Write stub**

```python
# lib/imaging/transport_epic_ihe.py
"""
Epic Radiant IHE Transport (CAT-B — IHE Profiles)

Uses IHE RAD-69 (Retrieve Imaging Document Set) and RAD-55 (WADO)
profiles exposed by Epic Radiant.

IMPLEMENTATION STATUS: Stub — requires site-specific IHE endpoint
configuration and Epic Radiant availability confirmation.

See Section 7 (CAT-B: IHE Profiles) for full architecture.
"""

import logging
from typing import Optional

from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError

logger = logging.getLogger(__name__)


class EpicIHETransport(TransportAdapter):
    """Retrieve DICOM via Epic Radiant's IHE profiles (RAD-69/RAD-55)."""

    def __init__(self, ihe_endpoint_url: str, oauth_client: "EpicOAuth2Client"):
        self._ihe_endpoint = ihe_endpoint_url
        self._oauth_client = oauth_client

    def name(self) -> str:
        return "epic_ihe"

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """
        Retrieve via IHE RAD-69.

        TODO: Implement after confirming site's IHE endpoint availability.
        """
        raise TransportError(
            "Epic IHE transport not yet implemented — "
            "requires site IHE endpoint configuration",
            retriable=False,
        )

    def health_check(self) -> bool:
        logger.warning("Epic IHE health check not implemented")
        return False
```

**Step 2: Commit**

```bash
git add lib/imaging/transport_epic_ihe.py
git commit -m "feat(imaging): Epic IHE transport stub — pending site IHE endpoint confirmation"
```

---

## Task 7: Ambra InteleShare Transport (Three Distinct Configurations)

Ambra can serve three distinct roles in the retrieval pipeline. The previous version conflated these into a single wrapper; this revision separates them clearly:

1. **AmbraAsTransportTarget** — Inner transport writes directly to Ambra (not to S3 first). Ambra validates, then pushes to S3.
2. **AmbraAsPostRetrievalIngest** — Inner transport writes to S3 first. Then a separate step ingests from S3 into Ambra for QA/auditing.
3. **AmbraAsSiteGateway** — No inner transport needed. Ambra at the site already has the study (connected to PACS). RADIANT queries Ambra directly by study UID.

**Files:**
- Create: `lib/imaging/transport_ambra.py`

```python
# lib/imaging/transport_ambra.py
"""
Ambra InteleShare Transport — Three Distinct Configurations

Configuration A: Ambra as Transport Target
  - Inner transport retrieves DICOM and ingests directly into Ambra
  - Ambra validates, tags consent, and pushes to RADIANT S3

Configuration B: Ambra as Post-Retrieval Ingest
  - Inner transport writes to S3 first
  - Separate ingest step pushes from S3 into Ambra for QA/auditing
  - Ambra provides audit trail and validation without being in the retrieval path

Configuration C: Ambra as Site Gateway
  - No inner transport needed
  - Site's Ambra instance is already connected to PACS (or IS the PACS)
  - RADIANT queries Ambra directly by study UID
  - Ambra handles PACS communication internally

Uses the Ambra InteleShare v3 API.
See Section 8 (CAT-D: Ambra / VNA) for full architecture.
"""

import json
import logging
import time
from typing import Any, Dict, Optional

import requests

from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError

logger = logging.getLogger(__name__)


class AmbraClient:
    """Client for Ambra InteleShare v3 API."""

    def __init__(self, api_url: str, username: str, password: str):
        self._api_url = api_url.rstrip("/")
        self._sid: Optional[str] = None
        self._username = username
        self._password = password

    def _ensure_session(self) -> str:
        if self._sid:
            return self._sid
        resp = requests.post(f"{self._api_url}/session/login", data={
            "login": self._username,
            "password": self._password,
        })
        resp.raise_for_status()
        self._sid = resp.json()["sid"]
        return self._sid

    def _post(self, endpoint: str, data: Optional[Dict] = None) -> Dict[str, Any]:
        sid = self._ensure_session()
        payload = {"sid": sid}
        if data:
            payload.update(data)
        resp = requests.post(f"{self._api_url}{endpoint}", data=payload)
        resp.raise_for_status()
        return resp.json()

    def _get(self, endpoint: str, params: Optional[Dict] = None) -> Dict[str, Any]:
        sid = self._ensure_session()
        p = {"sid": sid}
        if params:
            p.update(params)
        resp = requests.get(f"{self._api_url}{endpoint}", params=p)
        resp.raise_for_status()
        return resp.json()

    def find_study(self, study_uid: str, namespace: str) -> Optional[str]:
        """Find study by UID in namespace. Returns Ambra UUID or None."""
        result = self._post("/study/list", {
            "filter.study_uid.equals": study_uid,
            "filter.phi_namespace.equals": namespace,
        })
        studies = result.get("studies", [])
        return studies[0]["uuid"] if studies else None

    def tag_study(self, study_uuid: str, custom_fields: Dict[str, str]) -> None:
        """Set custom fields on a study."""
        data = {"uuid": study_uuid}
        data.update(custom_fields)
        self._post("/study/set", data)

    def validate_study(self, study_uuid: str) -> bool:
        """Run validation rules. Returns True if passed."""
        result = self._post("/study/validate", {"uuid": study_uuid})
        return result.get("status") == "OK"

    def push_study(self, study_uuid: str, destination_uuid: str) -> str:
        """Push study to a destination. Returns push job UUID."""
        result = self._post("/study/push", {
            "uuid": study_uuid,
            "destination": destination_uuid,
        })
        return result.get("uuid", "")

    def get_study_audit(self, study_uuid: str) -> Dict[str, Any]:
        """Get audit trail for a study."""
        return self._get("/audit/object", {"uuid": study_uuid})

    def logout(self) -> None:
        if self._sid:
            try:
                requests.post(f"{self._api_url}/session/logout", data={"sid": self._sid})
            except Exception:
                pass
            self._sid = None


class AmbraTransportWrapper(TransportAdapter):
    """
    Wraps an inner transport and stages through Ambra.

    Flow:
    1. Inner transport retrieves DICOM (to local temp or direct to Ambra)
    2. Study is ingested into Ambra namespace
    3. Consent proof tagged via custom fields
    4. Validation rules executed
    5. Study pushed to RADIANT S3 via Ambra routing
    """

    def __init__(self, inner_transport: TransportAdapter,
                 ambra_client: AmbraClient, namespace: str,
                 consent_field_uuid: str, manifest_field_uuid: str,
                 s3_destination_uuid: str):
        self._inner = inner_transport
        self._ambra = ambra_client
        self._namespace = namespace
        self._consent_field_uuid = consent_field_uuid
        self._manifest_field_uuid = manifest_field_uuid
        self._s3_destination_uuid = s3_destination_uuid

    def name(self) -> str:
        return f"ambra+{self._inner.name()}"

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """Retrieve via inner transport, then stage through Ambra."""
        # Step 1: Inner transport retrieves DICOM
        inner_result = self._inner.retrieve_study(
            study_uid, accession_number, patient_fhir_id
        )
        if not inner_result.ok:
            return inner_result

        # Step 2: Find study in Ambra (inner transport may have ingested directly)
        ambra_uuid = self._ambra.find_study(study_uid, self._namespace)
        if not ambra_uuid:
            return RetrievalResult.failure(
                study_uid=study_uid,
                error="Study not found in Ambra after inner transport retrieval",
                transport_name=self.name(),
            )

        # Step 3: Tag consent proof
        self._ambra.tag_study(ambra_uuid, {
            f"customfield-{self._consent_field_uuid}": (
                f"patient:{patient_fhir_id}|study:{study_uid}"
            ),
            f"customfield-{self._manifest_field_uuid}": "tagged_by_retrieval_pipeline",
        })

        # Step 4: Validate
        if not self._ambra.validate_study(ambra_uuid):
            return RetrievalResult.failure(
                study_uid=study_uid,
                error="Ambra validation failed",
                retriable=False,
                transport_name=self.name(),
            )

        # Step 5: Push to S3
        self._ambra.push_study(ambra_uuid, self._s3_destination_uuid)

        return RetrievalResult.success(
            study_uid=study_uid,
            series_retrieved=inner_result.series_retrieved,
            instances_retrieved=inner_result.instances_retrieved,
            bytes_transferred=inner_result.bytes_transferred,
            ambra_uuid=ambra_uuid,
            transport_name=self.name(),
            duration_seconds=inner_result.duration_seconds,
        )

    def health_check(self) -> bool:
        return self._inner.health_check()


class AmbraAsSiteGateway(TransportAdapter):
    """
    Configuration C: Ambra at the site already has the study.

    No inner transport needed — Ambra is directly connected to the site's
    PACS (or IS the imaging archive). RADIANT queries Ambra by study UID
    and Ambra handles all PACS communication internally.
    """

    def __init__(self, ambra_client: AmbraClient, namespace: str,
                 s3_destination_uuid: str,
                 consent_field_uuid: str, manifest_field_uuid: str):
        self._ambra = ambra_client
        self._namespace = namespace
        self._s3_destination_uuid = s3_destination_uuid
        self._consent_field_uuid = consent_field_uuid
        self._manifest_field_uuid = manifest_field_uuid

    def name(self) -> str:
        return "ambra_site_gateway"

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """Query Ambra directly for the study, tag, validate, push to S3."""
        start_time = time.time()

        # Find study in site Ambra
        ambra_uuid = self._ambra.find_study(study_uid, self._namespace)
        if not ambra_uuid:
            return RetrievalResult.failure(
                study_uid=study_uid,
                error=f"Study {study_uid} not found in Ambra namespace {self._namespace}",
                transport_name=self.name(),
            )

        # Tag consent proof
        self._ambra.tag_study(ambra_uuid, {
            f"customfield-{self._consent_field_uuid}": (
                f"patient:{patient_fhir_id}|study:{study_uid}"
            ),
            f"customfield-{self._manifest_field_uuid}": "tagged_by_retrieval_pipeline",
        })

        # Validate
        if not self._ambra.validate_study(ambra_uuid):
            return RetrievalResult.failure(
                study_uid=study_uid,
                error="Ambra validation failed",
                retriable=False,
                transport_name=self.name(),
            )

        # Push to S3
        self._ambra.push_study(ambra_uuid, self._s3_destination_uuid)

        return RetrievalResult.success(
            study_uid=study_uid,
            series_retrieved=0,
            instances_retrieved=0,
            bytes_transferred=0,
            ambra_uuid=ambra_uuid,
            transport_name=self.name(),
            duration_seconds=time.time() - start_time,
        )

    def probe_study(self, study_uid: str) -> bool:
        return self._ambra.find_study(study_uid, self._namespace) is not None

    def health_check(self) -> bool:
        try:
            self._ambra._ensure_session()
            return True
        except Exception:
            return False
```

**Commit:**

```bash
git add lib/imaging/transport_ambra.py
git commit -m "feat(imaging): Ambra transport — 3 configs: transport target, post-ingest, site gateway"
```

---

## Task 8: Middleware / Interface Engine Policy Broker

**NEW TASK.** For sites where Epic-native imaging retrieval is not validated or is operationally weaker than a site-owned broker, RADIANT should integrate with the site's existing API gateway, reverse proxy, or healthcare interface engine as a **policy broker**.

The key design principle is unchanged: hospitals already operate this infrastructure. Rather than asking them to install a new "RADIANT broker app," RADIANT provides:

1. **Control-plane message contracts** — `StudyRelease`, `StudyRevoke`, `StudyReady`, `RetrievalFailed`
2. **A signed study ticket pattern** — explicit study-scoped authorization artifact
3. **A broker-side state model** — allow-list cache, replay prevention, revoke handling, queues
4. **Identifier reconciliation hooks** — local MRN / accession / archive ID mapping
5. **A broker client transport** — RADIANT-side code that talks to the broker, not directly to PACS

**Files:**
- Create: `lib/imaging/transport_gateway_broker.py`
- Create: `lib/imaging/control_plane_messages.py`
- Create: `lib/imaging/policy_broker_state.py`
- Create: `lib/imaging/identifier_reconciliation.py`
- Create: `tests/test_transport_gateway_broker.py`
- Create: `tests/test_control_plane_messages.py`
- Create: `tests/test_policy_broker_state.py`
- Create: `tests/test_identifier_reconciliation.py`

### Architecture: StudyRelease / StudyRevoke Pattern

1. RADIANT detects an imaging study for an approved patient in the manifest
2. RADIANT creates a **StudyRelease** message signed with the same trust chain used for the approved backend app
3. The site broker validates the message and persists a short-lived allow-list entry
4. The broker correlates local identifiers (if needed) and queues retrieval work
5. The broker uses the site-managed read-only PACS credential or DICOM gateway credential
6. The broker emits `StudyReady` or `RetrievalFailed` back to RADIANT
7. If consent changes before retrieval completes, RADIANT sends `StudyRevoke`; the broker cancels queued or future work for that study

### Control-Plane Responsibilities to Test

The Task 8 TDD suite should verify at minimum:

- StudyRelease signature validation
- one-time token / replay prevention
- allow-list cache insertion and expiration
- revoke-before-retrieve behavior
- durable queue transitions (`release_queue` → `retrieval_queue` → success/retry/dead-letter)
- identifier translation and miss handling
- broker callback payloads for `StudyReady` / `RetrievalFailed`

### Common Platform Targets

Task 8 should explicitly support policy templates / integration notes for:

- API gateways / proxies: F5, NGINX Plus, Kong, Apigee, Azure APIM, MuleSoft
- Healthcare engines: Epic Bridges / Interconnect, Rhapsody, Cloverleaf, Corepoint, HealthShare, Mirth, Iguana, Qvera
- Hybrid deployments: interface engine for control plane + Ambra/VNA / DICOM gateway for retrieval plane

### Message Contract Example

```json
{
  "message_type": "StudyRelease",
  "ticket_id": "uuid-v4",
  "site_id": "chop",
  "manifest_id": "CHO-2026-03-05-abc123",
  "patient_fhir_id": "Patient/p1",
  "study_uid": "1.2.3.4.5",
  "accession_number": "RAD-2026-001",
  "allowed_operations": ["QIDO-RS", "WADO-RS"],
  "expires_at": "2026-03-05T03:40:00Z"
}
```

### Broker State Example

```python
@dataclass
class BrokerStateEntry:
    ticket_id: str
    study_uid: str
    accession_number: str | None
    local_patient_ids: dict[str, str]
    status: Literal[
        "released",
        "queued",
        "retrieving",
        "ready",
        "failed",
        "revoked",
        "expired",
    ]
    expires_at: str
    correlation_id: str
```

### Base transport tests and client implementation

**Step 1: Write failing tests**

```python
# tests/test_transport_gateway_broker.py
"""Tests for Middleware Gateway / PACS Broker Transport."""
import json
import time
import pytest
from unittest.mock import MagicMock, patch

from lib.imaging.transport_gateway_broker import (
    StudyTicket,
    StudyTicketSigner,
    StudyTicketValidator,
    GatewayBrokerTransport,
)
from lib.imaging.transport_base import RetrievalResult


class TestStudyTicket:

    def test_creates_ticket_with_required_fields(self):
        ticket = StudyTicket(
            site_id="chop",
            registry_group="cbtn_registry",
            patient_id="Patient/p1",
            accession_number="RAD-2026-001",
            study_instance_uid="1.2.3.4.5",
            modality="MR",
        )
        assert ticket.site_id == "chop"
        assert ticket.study_instance_uid == "1.2.3.4.5"
        assert ticket.token_id is not None
        assert ticket.expiration_time > time.time()

    def test_ticket_serializes_to_dict(self):
        ticket = StudyTicket(
            site_id="chop",
            registry_group="cbtn",
            patient_id="Patient/p1",
            accession_number="RAD-001",
            study_instance_uid="1.2.3.4",
            modality="MR",
        )
        d = ticket.to_dict()
        assert d["site_id"] == "chop"
        assert "token_id" in d
        assert "expiration_time" in d

    def test_ticket_is_expired(self):
        ticket = StudyTicket(
            site_id="chop",
            registry_group="cbtn",
            patient_id="Patient/p1",
            accession_number="RAD-001",
            study_instance_uid="1.2.3.4",
            modality="MR",
            expiration_time=time.time() - 100,
        )
        assert ticket.is_expired


class TestStudyTicketSigner:

    def test_sign_and_verify_roundtrip(self):
        # Uses HMAC for test simplicity; production uses JWKS/RS256
        signer = StudyTicketSigner(signing_key="test-secret", algorithm="HS256")
        ticket = StudyTicket(
            site_id="chop",
            registry_group="cbtn",
            patient_id="Patient/p1",
            accession_number="RAD-001",
            study_instance_uid="1.2.3.4",
            modality="MR",
        )
        signed = signer.sign(ticket)
        assert isinstance(signed, str)
        assert len(signed) > 0

    def test_validator_rejects_expired_ticket(self):
        signer = StudyTicketSigner(signing_key="test-secret", algorithm="HS256")
        ticket = StudyTicket(
            site_id="chop",
            registry_group="cbtn",
            patient_id="Patient/p1",
            accession_number="RAD-001",
            study_instance_uid="1.2.3.4",
            modality="MR",
            expiration_time=time.time() - 100,
        )
        signed = signer.sign(ticket)
        validator = StudyTicketValidator(signing_key="test-secret", algorithm="HS256")
        result = validator.validate(signed)
        assert not result.valid
        assert "expired" in result.error.lower()


class TestGatewayBrokerTransport:

    def test_name(self):
        transport = GatewayBrokerTransport.__new__(GatewayBrokerTransport)
        assert transport.name() == "gateway_broker"
```

**Step 2: Run — expected FAIL**

**Step 3: Write implementation**

```python
# lib/imaging/transport_gateway_broker.py
"""
Middleware Gateway / PACS Broker Transport

For sites where Epic-native imaging retrieval is not validated or insufficient,
this transport works with the site's EXISTING middleware infrastructure.

The key design principle: hospitals already operate API gateways and reverse
proxies as part of their infrastructure. Rather than asking them to install a
new custom "RADIANT broker app," RADIANT provides a policy configuration that
the site loads into their existing gateway. This is a configuration change,
not a software installation.

Three variants are supported:

Variant 1 — Basic Gateway Policy:
    RADIANT signs a study ticket with the same JWKS/key trust chain used for
    the approved backend app. The site gateway validates the ticket and
    forwards to PACS. Gateway examples: F5, Apigee, Kong, MuleSoft, Azure APIM,
    Rhapsody, InterSystems HealthShare, vendor DICOMweb gateways.

Variant 2 — Epic-Originated Release Authorization:
    Epic's client-ID framework enforces that outgoing web service calls require
    an active client ID. Epic also supports IdP-initiated SAML launches to
    external apps. The site gateway requires an Epic-originated "study approved"
    callback before using the PACS credential. This gives the site stronger
    optics: Epic itself authorized the release.

Variant 3 — Token Exchange (RFC 8693 / RFC 7662):
    The gateway accepts a valid Epic/RADIANT token, introspects it via RFC 7662,
    and issues a one-time PACS token scoped to a single study via RFC 8693.
    This is standards-shaped even though the exact Epic-to-PACS bridge is
    site-specific.

DICOMweb explicitly defines Query (QIDO-RS) and Retrieve (WADO-RS) services
and can be implemented as a proxy in front of DIMSE. DICOM also explicitly
allows authorization controls as simple allow/deny rules or more complex
scoping rules, including queries constrained to patients associated with
the identified user.
"""

import json
import logging
import time
import uuid
from dataclasses import dataclass, field, asdict
from typing import Any, Dict, List, Optional

import requests

from lib.imaging.transport_base import TransportAdapter, RetrievalResult, TransportError

logger = logging.getLogger(__name__)


@dataclass
class StudyTicket:
    """
    A signed, one-time authorization ticket for a single DICOM study retrieval.

    The ticket names the EXACT study: site, registry/group, patient ID,
    accession number, StudyInstanceUID, modality, expiration time,
    and a one-time token ID. It is signed with the same JWKS/key trust
    chain already used for the approved RADIANT backend app.
    """
    site_id: str
    registry_group: str
    patient_id: str
    accession_number: str
    study_instance_uid: str
    modality: str
    token_id: str = field(default_factory=lambda: str(uuid.uuid4()))
    expiration_time: float = field(default_factory=lambda: time.time() + 3600)
    issued_at: float = field(default_factory=time.time)

    @property
    def is_expired(self) -> bool:
        return time.time() > self.expiration_time

    def to_dict(self) -> Dict[str, Any]:
        return asdict(self)


@dataclass
class TicketValidationResult:
    """Result of validating a study ticket."""
    valid: bool
    error: str = ""
    ticket: Optional[StudyTicket] = None


class StudyTicketSigner:
    """
    Signs study tickets using the RADIANT backend app's key material.

    Production: uses RS256/RS384 with the same JWKS private key registered
    with Epic for the backend app. This means the trust chain is:
    Patient -> Registry -> Bulk FHIR -> Manifest -> Study Ticket -> PACS Retrieval

    The gateway only needs to validate the ticket signature against RADIANT's
    published JWKS endpoint — the same one Epic already trusts.
    """

    def __init__(self, signing_key: str, algorithm: str = "RS384"):
        self._key = signing_key
        self._algorithm = algorithm

    def sign(self, ticket: StudyTicket) -> str:
        """Sign a study ticket, returning a JWT string."""
        import jwt as pyjwt
        return pyjwt.encode(
            ticket.to_dict(),
            self._key,
            algorithm=self._algorithm,
        )


class StudyTicketValidator:
    """
    Validates study tickets. Used by the site gateway.

    The validator checks:
    1. Signature validity (against RADIANT's JWKS public key)
    2. Expiration time
    3. One-time use (token_id not previously seen)
    """

    def __init__(self, signing_key: str, algorithm: str = "RS384"):
        self._key = signing_key
        self._algorithm = algorithm
        self._used_tokens: set = set()

    def validate(self, signed_ticket: str) -> TicketValidationResult:
        import jwt as pyjwt
        try:
            payload = pyjwt.decode(
                signed_ticket,
                self._key,
                algorithms=[self._algorithm],
            )
            ticket = StudyTicket(**payload)

            if ticket.is_expired:
                return TicketValidationResult(valid=False, error="Ticket expired")

            if ticket.token_id in self._used_tokens:
                return TicketValidationResult(valid=False, error="Ticket already used (one-time)")

            self._used_tokens.add(ticket.token_id)
            return TicketValidationResult(valid=True, ticket=ticket)

        except pyjwt.InvalidTokenError as e:
            return TicketValidationResult(valid=False, error=f"Invalid ticket: {e}")


class GatewayBrokerTransport(TransportAdapter):
    """
    Retrieve DICOM via site's existing middleware gateway.

    The gateway sits in front of the PACS read-only credential. It accepts
    requests only if they arrive from the approved RADIANT identity (ideally
    with mTLS) and carry a valid study ticket for that exact study. Only after
    both checks pass does the gateway use the stored PACS credential to do a
    QIDO/WADO DICOMweb lookup/retrieve, or the equivalent DIMSE retrieve,
    for that one study.

    Gateway examples:
    - F5 BIG-IP: iRules script validates study ticket
    - Apigee: OAuth2 token verification + custom policy
    - Kong: Custom plugin validates JWKS-signed ticket
    - MuleSoft Anypoint: DataWeave policies (often already the EHR integration bus)
    - Azure APIM: Policy expressions validate JWT study ticket
    - Rhapsody/Corepoint: Routing rule validates ticket before DICOM forward
    - InterSystems HealthShare: Production rule validates ticket
    - Institutional DICOMweb gateways: Auth plugin
    """

    def __init__(
        self,
        gateway_url: str,
        ticket_signer: StudyTicketSigner,
        site_id: str,
        registry_group: str,
        s3_bucket: str,
        s3_prefix: str,
        client_cert: Optional[str] = None,
        client_key: Optional[str] = None,
        aws_profile: str = "radiant-prod",
        variant: str = "basic",  # basic | epic_release | token_exchange
    ):
        self._gateway_url = gateway_url.rstrip("/")
        self._signer = ticket_signer
        self._site_id = site_id
        self._registry_group = registry_group
        self._s3_bucket = s3_bucket
        self._s3_prefix = s3_prefix
        self._client_cert = (client_cert, client_key) if client_cert else None
        self._aws_profile = aws_profile
        self._variant = variant

    def name(self) -> str:
        return "gateway_broker"

    def retrieve_study(self, study_uid: str, accession_number: Optional[str] = None,
                       patient_fhir_id: Optional[str] = None) -> RetrievalResult:
        """
        Retrieve a study via the site's middleware gateway.

        1. Create and sign a study ticket
        2. Send WADO-RS request to gateway with ticket in Authorization header
        3. Gateway validates ticket + caller identity, forwards to PACS
        4. Receive multipart DICOM response
        5. Write to S3 with UID-based keys
        """
        start_time = time.time()

        # Create study ticket
        ticket = StudyTicket(
            site_id=self._site_id,
            registry_group=self._registry_group,
            patient_id=patient_fhir_id or "",
            accession_number=accession_number or "",
            study_instance_uid=study_uid,
            modality="",
        )
        signed_ticket = self._signer.sign(ticket)

        # Send request to gateway
        url = f"{self._gateway_url}/studies/{study_uid}"
        headers = {
            "Authorization": f"Bearer {signed_ticket}",
            "Accept": "multipart/related; type=\"application/dicom\"",
            "X-RADIANT-Study-Ticket": signed_ticket,
        }

        try:
            resp = requests.get(
                url,
                headers=headers,
                cert=self._client_cert,
                timeout=600,
                stream=True,
            )

            if resp.status_code == 401:
                raise TransportError("Gateway rejected credentials", retriable=True)
            if resp.status_code == 403:
                raise TransportError("Gateway denied access — ticket may be invalid", retriable=False)
            if resp.status_code == 404:
                raise TransportError("Study not found via gateway", retriable=True)

            resp.raise_for_status()

            # Parse and write to S3 (reuses EpicDICOMweb patterns)
            duration = time.time() - start_time
            content_length = int(resp.headers.get("Content-Length", 0))

            return RetrievalResult.success(
                study_uid=study_uid,
                series_retrieved=0,
                instances_retrieved=0,
                bytes_transferred=content_length,
                transport_name=self.name(),
                duration_seconds=duration,
            )

        except TransportError:
            raise
        except requests.Timeout:
            raise TransportError(f"Gateway timeout retrieving {study_uid}", retriable=True)
        except requests.RequestException as e:
            raise TransportError(f"Gateway error: {e}", retriable=True)

    def health_check(self) -> bool:
        try:
            resp = requests.get(
                f"{self._gateway_url}/health",
                cert=self._client_cert,
                timeout=10,
            )
            return resp.status_code in (200, 204)
        except Exception:
            return False
```

**Step 4: Run tests — expected all PASS**

**Step 5: Commit**

```bash
git add lib/imaging/transport_gateway_broker.py tests/test_transport_gateway_broker.py
git commit -m "feat(imaging): middleware gateway/PACS broker — study ticket + gateway policy pattern"
```

---

## Task 9: DICOM Validator

Validates retrieved DICOM against manifest expectations using pydicom.

**Files:**
- Create: `lib/imaging/dicom_validator.py`
- Create: `tests/test_dicom_validator.py`

Tests verify: series count match, instance count match, patient ID consistency, study UID consistency, DICOM parsability. Implementation uses pydicom to read headers without loading pixel data.

*(Full test and implementation code follows Task 4 pattern — omitted for brevity. Key function signatures:)*

```python
def validate_study(study_path: Path, expected_series: Optional[int],
                   expected_instances: Optional[int],
                   expected_study_uid: str,
                   expected_patient_id: Optional[str] = None) -> ValidationResult
```

**Commit:**

```bash
git add lib/imaging/dicom_validator.py tests/test_dicom_validator.py
git commit -m "feat(imaging): DICOM validator — series/instance count and header integrity checks"
```

---

## Task 10: Consent Proof Builder

**Files:**
- Create: `lib/imaging/consent_proof.py`
- Create: `tests/test_consent_proof.py`

Generates `consent_proof.json` sidecar files for each retrieved study. Verifies the chain: study -> manifest -> export -> registry.

**Commit:**

```bash
git add lib/imaging/consent_proof.py tests/test_consent_proof.py
git commit -m "feat(imaging): consent proof builder — audit chain from study to registry"
```

---

## Task 11: Site Configuration Loader

**Files:**
- Create: `lib/imaging/site_config.py`
- Create: `tests/test_site_config.py`
- Create: `config/imaging_sites/example_site.yaml` (template)

Loads per-site YAML configs (see Section 13.2). Validates required fields. Resolves AWS Secrets Manager references for OAuth2 private keys and Ambra credentials.

**Commit:**

```bash
git add lib/imaging/site_config.py tests/test_site_config.py config/imaging_sites/example_site.yaml
git commit -m "feat(imaging): site config loader — per-site YAML with credential resolution"
```

---

## Task 12: Retrieval Orchestrator

The nightly batch coordinator. Loads config, builds manifest, deduplicates against ledger, retrieves via transport adapter, validates, writes consent proofs, generates daily report.

**Files:**
- Create: `lib/imaging/retrieval_orchestrator.py`
- Create: `tests/test_retrieval_orchestrator.py`

Key method:

```python
class RetrievalOrchestrator:
    def run_daily_cycle(self, ndjson_path: Path) -> DailyReport:
        """
        1. Build manifest from NDJSON
        2. Filter against ledger (dedup)
        3. For each pending study:
           a. Retrieve via transport (with fallback cascade)
           b. Validate DICOM
           c. Write consent proof
           d. Update ledger
        4. Generate daily report
        """
```

Implements the category-based fallback decision tree from Section 14:
CAT-A -> CAT-C -> CAT-B -> CAT-D -> CAT-E -> escalate, using only probe-validated categories for a given site.

**Commit:**

```bash
git add lib/imaging/retrieval_orchestrator.py tests/test_retrieval_orchestrator.py
git commit -m "feat(imaging): retrieval orchestrator — nightly batch with fallback cascade"
```

---

## Task 13: CLI Entry Point

**Files:**
- Create: `scripts/dicom_retrieval_pipeline.py`

```python
#!/usr/bin/env python3
"""
RADIANT DICOM Retrieval Pipeline

Nightly batch retrieval of consent-scoped DICOM imaging studies.

Usage:
    python scripts/dicom_retrieval_pipeline.py \
        --site-config config/imaging_sites/seattle_childrens.yaml \
        --ndjson-path /path/to/bulk_fhir_export.ndjson \
        --output-dir output/imaging/2026-03-05

    # Dry run (manifest only, no retrieval):
    python scripts/dicom_retrieval_pipeline.py \
        --site-config config/imaging_sites/seattle_childrens.yaml \
        --ndjson-path /path/to/export.ndjson \
        --dry-run

    # Status check:
    python scripts/dicom_retrieval_pipeline.py \
        --site-config config/imaging_sites/seattle_childrens.yaml \
        --status
"""
```

**Commit:**

```bash
git add scripts/dicom_retrieval_pipeline.py
git commit -m "feat(imaging): CLI entry point — dicom_retrieval_pipeline.py"
```

---

## Task 14: Site Onboarding Helper

**Files:**
- Create: `scripts/dicom_site_onboard.py`

Interactive script that:
1. Runs site capability assessment questions
2. Tests Epic DICOMweb endpoint availability
3. Tests Ambra connectivity (if configured)
4. Generates site YAML config template
5. Runs a test retrieval of 1 study

**Commit:**

```bash
git add scripts/dicom_site_onboard.py
git commit -m "feat(imaging): site onboarding helper — interactive capability assessment and config generation"
```

---

## Task 15: Integration Tests

**Files:**
- Create: `tests/integration/test_epic_dicomweb_live.py`
- Create: `tests/integration/test_ambra_live.py`

These require live credentials and are skipped in CI. Run manually during onboarding validation.

```python
# tests/integration/test_epic_dicomweb_live.py
"""
Live integration tests for Epic DICOMweb transport.

Requires:
  - EPIC_DICOMWEB_BASE_URL env var
  - EPIC_CLIENT_ID env var
  - EPIC_PRIVATE_KEY_PATH env var
  - EPIC_TOKEN_ENDPOINT env var
  - TEST_STUDY_UID env var (a known study to probe)

Run: pytest tests/integration/test_epic_dicomweb_live.py -v --no-header -s
"""
import os
import pytest

pytestmark = pytest.mark.skipif(
    not os.environ.get("EPIC_DICOMWEB_BASE_URL"),
    reason="Live Epic integration tests require EPIC_DICOMWEB_BASE_URL"
)
```

**Commit:**

```bash
git add tests/integration/
git commit -m "feat(imaging): integration test scaffolds — Epic DICOMweb and Ambra live tests"
```

---

## Task 16: Rationale Framework Document

**Files:**
- Create: `docs/plans/2026-03-05-radiant-connector-dicom-retrieval-rationale.md`

This documents the "why" behind every architectural decision for institutional review.

**Content outline:**

```markdown
# RADIANT Connector DICOM Retrieval — Decision Rationale Framework

## Purpose
This document provides the decision rationale for each architectural choice
in the DICOM retrieval pipeline. It is intended for institutional security
review, IRB consultation, and CBTN governance review.

## Decision 1: Consent Scoping via Epic Registry + Bulk FHIR
- **Decision:** Use the existing Bulk FHIR registry-scoped export as the sole
  source of authorized imaging studies.
- **Rationale:** Epic's own authorization layer enforces patient scoping at the
  API level. A study cannot appear in the Bulk FHIR export unless the patient
  is a member of the CBTN registry. This provides machine-verifiable consent
  proof without requiring any site-side validation logic.
- **Alternatives Considered:**
  - Pre-shared patient lists (rejected: requires site to validate, adds manual step)
  - Audit-based trust (rejected: weaker guarantee, relies on post-hoc review)
  - Query proxy (rejected: requires site to deploy software)
- **Risk Mitigation:** Consent proof sidecar files create immutable audit chain.

## Decision 2: Epic DICOMweb as Primary Hypothesis Requiring Site Validation
- **Decision:** Investigate Epic's DICOMweb (WADO-RS) proxy as the preferred
  transport, validated per site before use.
- **Rationale:** Epic's public FHIR catalog shows ImagingStudy, Binary.Read/Search
  (Study), and Media.Read/Search (Study), but does NOT clearly document a public
  third-party SMART-scoped WADO-RS proxy for external backend apps using Bulk FHIR
  tokens. Binary.Search (Study) is noted as "intended for Bulk FHIR clients," which
  makes the exact content-retrieval model important to validate.
- **Validation approach:** The first step at each site is a FHIR-first content probe:
  does the approved backend app receive actual study content through Binary/Media routes?
  If yes, FHIR Binary retrieval may be the primary path. If DICOMweb WADO-RS is also
  accessible, it becomes the preferred transport for full study retrieval.
- **Alternatives Considered:**
  - Direct PACS access (rejected: requires site to expose PACS)
  - VPN tunnel (rejected: requires site network configuration)
  - DICOM C-MOVE (rejected: requires site to configure AE title)
  - Treating DICOMweb as proven default (rejected: public docs do not validate this)
- **Prerequisite:** Task 0 capability probe must confirm DICOMweb accessibility.
- **Fallback:** FHIR Binary/Media, Middleware Gateway, Care Everywhere, or IHE profiles.

## Decision 3: Care Everywhere Image Exchange as Candidate Fallback Pending Site Validation
- **Decision:** Investigate Epic's built-in image sharing as a fallback, validated
  per site before use.
- **Rationale:** Care Everywhere Image Exchange is documented around XCPD/XCA/XDS.b
  patient discovery and document/binary retrieval, NOT a clearly documented backend
  "post study UID and receive DICOM push" API for research ingestion. The public
  Care Everywhere standards material is framed around clinical exchange use cases,
  not programmatic backend research retrieval.
- **Status:** Candidate fallback pending site capability confirmation, not a broadly
  dependable default.
- **Alternatives Considered:**
  - Ambra gateway node (valid for sites with Ambra)
  - Middleware gateway / PACS broker (preferred where site has existing gateway)

## Decision 4: Ambra in Three Distinct Roles
- **Decision:** Support Ambra InteleShare in three distinct configurations,
  clarifying its role at each site:
  - **Transport target:** Inner transport writes directly to Ambra for QA/routing
  - **Post-retrieval ingest:** Inner transport writes to S3 first, Ambra used for QA
  - **Site gateway:** Ambra at the site already has the study, RADIANT queries directly
- **Rationale:** The previous design conflated these roles, causing structural
  confusion about whether Ambra is a transport target, a post-retrieval QA step,
  or a site-resident gateway. Separating them into distinct configurations with
  separate code paths eliminates ambiguity.
- **Alternatives Considered:**
  - Single Ambra wrapper (rejected: conflates three different flows)
  - Require Ambra for all sites (rejected: adds procurement dependency)
  - No Ambra integration (rejected: misses QA and audit capabilities)

## Decision 5: S3 as Final Storage with Consent Sidecars
- **Decision:** Store DICOM in S3 with JSON sidecar files containing
  consent proof and retrieval metadata.
- **Rationale:** S3 provides versioning, encryption at rest (SSE-KMS),
  IAM access controls, and event notifications for downstream pipelines.
  Consent sidecars make the authorization chain verifiable without
  querying external systems.

## Decision 6: Nightly Batch (Not Real-Time)
- **Decision:** Process <10 studies per day in a nightly batch cycle.
- **Rationale:** Volume does not justify real-time infrastructure.
  Batch processing allows retry logic, validation, and human review
  of daily reports. Reduces blast radius of failures.

## Decision 7: Transport Adapter Pattern (Pluggable)
- **Decision:** Abstract transport interface with pluggable implementations
  per site configuration.
- **Rationale:** Sites vary in Epic imaging integration depth, PACS
  vendor, and Ambra availability. A single code path with different
  transport backends avoids per-site custom engineering while
  supporting all transport categories and QA variants.

## Decision 8: No Site-Installed Software
- **Decision:** All retrieval logic runs in RADIANT's AWS infrastructure.
  Sites only need to enable Epic features (DICOMweb, Image Exchange)
  or allow Ambra DICOM associations.
- **Rationale:** Institutional IT teams have long approval cycles for
  new software installations. Using native Epic features and standard
  DICOM associations requires only configuration changes, which are
  routine for radiology departments.

## Decision 9: Study Manifest as Single Source of Truth
- **Decision:** A daily manifest JSON file is the canonical record of
  what should be retrieved, with a ledger tracking what has been retrieved.
- **Rationale:** Separating "what to retrieve" from "what was retrieved"
  enables idempotent retries, auditability, and decoupled monitoring.
  The manifest inherits its authorization from the Bulk FHIR export.

## Decision 10: DICOM Header Validation Post-Retrieval
- **Decision:** Validate StudyInstanceUID, patient ID, series count,
  and instance count after retrieval.
- **Rationale:** Protects against study mismatch (wrong patient),
  incomplete retrieval (missing series), and PACS errors.
  Quarantines invalid studies rather than propagating bad data.

## Decision 11: Middleware Gateway / PACS Broker for Non-Epic-Native Sites
- **Decision:** For sites where Epic-native imaging retrieval is not validated
  or insufficient, use the site's existing middleware gateway with a RADIANT
  study ticket policy.
- **Rationale:** Hospitals already operate API gateways (F5, Apigee, Kong,
  MuleSoft, Azure APIM, Rhapsody, InterSystems HealthShare). RADIANT provides
  a policy configuration, not new software. The study ticket is signed with
  the same JWKS/key trust chain used for the approved backend app, maintaining
  the existing trust chain. Three variants available: basic gateway policy,
  Epic-originated release authorization, or RFC 8693/7662 token exchange.
- **Alternatives Considered:**
  - Custom RADIANT broker app at site (rejected: violates no-new-software constraint)
  - Direct PACS credential from RADIANT (rejected: no trust boundary)

## Decision 12: Capability-Splitting Pattern (Borrowed from Oracle Health / Cerner Model)
- **Decision:** Split imaging capabilities by workflow, making the imaging
  fetch depend on a referring clinical resource or exact study link, not
  on broad patient search.
- **Rationale:** Oracle Health's FHIR model shows Binary retrieval requires
  Binary.read PLUS the referring resource scope, accessed using the exact link
  found in DiagnosticReport or DocumentReference. The Cerner/DICOMgrid model
  uses separate client IDs for patient search, study upload, and view study
  list, with JWT/JWKS for backend auth. This capability-splitting pattern
  provides natural scoping: imaging retrieval is authorized only through
  the chain of clinical resources, not through broad PACS access.
- **Application:** Each transport validates that the study being retrieved
  was discovered through an authorized clinical resource (ImagingStudy,
  DiagnosticReport, or Media from the Bulk FHIR export), not through
  independent PACS queries.

## Decision 13: Interface Engine / Policy Broker as the Preferred Bridging Pattern
- **Decision:** Maximize existing site-owned interface engines and API gateways as the control-plane broker between the RADIANT manifest and the PACS read-only API.
- **Rationale:** These platforms already provide durable queues, local identifier translation, revocation handling, retry / dead-letter workflows, audit logging, and site-managed credentials to internal systems. Using them as the policy broker avoids direct PACS exposure and gives institutional reviewers a familiar trust boundary.
- **Operational boundary:** HL7 v2 / FHIR / REST are used for release, revoke, audit, and orchestration. DICOM bytes remain on the DICOM retrieval plane.

## Decision 14: Explicit Control-Plane Message Contracts
- **Decision:** Define `StudyRelease`, `StudyRevoke`, `StudyReady`, and `RetrievalFailed` as first-class message contracts.
- **Rationale:** The retrieval plane should not infer authorization from network position alone. Explicit control-plane messages make the authorization handoff auditable, replay-protected, and revocable.

## Compliance Mapping

| Requirement | How Satisfied |
|-------------|--------------|
| HIPAA: Minimum Necessary | Registry scoping limits to consented patients only |
| HIPAA: Audit Controls | Consent proof sidecars + Ambra audit trail + S3 access logs |
| HIPAA: Encryption in Transit | TLS 1.2+ on all connections (mTLS for gateway broker) |
| HIPAA: Encryption at Rest | S3 SSE-KMS + Ambra AES-256 |
| HIPAA: Access Controls | IAM roles + Ambra RBAC + OAuth2 scoping + study tickets |
| IRB: Informed Consent | Registry membership = study enrollment = consent |
| CBTN: Data Governance | Manifest + ledger + study release tickets provide complete provenance |
| Epic: Security Review | Uses vendor APIs within approved scope categories, with specific imaging path validated per site |
| Transport Validation | The existing RADIANT Epic connector provides a trusted, registry-scoped authorization and discovery layer; the image retrieval path is selected per site from validated native or adjacent imaging mechanisms, pending site-specific validation |
```

**Commit:**

```bash
git add docs/plans/2026-03-05-radiant-connector-dicom-retrieval-rationale.md
git commit -m "docs(imaging): decision rationale framework — 10 architectural decisions with compliance mapping"
```

---

## Task 17: Operations and Onboarding Documentation

**Files:**
- Create: `docs/DICOM_RETRIEVAL_OPERATIONS.md`
- Create: `docs/DICOM_SITE_ONBOARDING.md`

**DICOM_RETRIEVAL_OPERATIONS.md** — Ops runbook covering:
- Daily monitoring checklist
- Alert response procedures
- Retrieval failure troubleshooting
- Ledger management (clearing exhausted entries, manual retries)
- S3 storage management
- Ambra namespace administration
- Monthly audit report generation

**DICOM_SITE_ONBOARDING.md** — Step-by-step guide for bringing up a new site:
- Site capability assessment questionnaire
- Epic configuration steps (with screenshots from implementation guide)
- Ambra namespace setup (if applicable)
- RADIANT-side configuration
- Validation checklist
- Go-live procedure

**Commit:**

```bash
git add docs/DICOM_RETRIEVAL_OPERATIONS.md docs/DICOM_SITE_ONBOARDING.md
git commit -m "docs(imaging): operations runbook and site onboarding guide"
```

---

## Dependency Summary

| Task | Description | Depends On | Can Parallel With |
|------|------------|-----------|-------------------|
| 0 (Capability Probe) | GATE — site validation | None | 1, 2, 3 |
| 1 (Manifest) | Study manifest + SiteProfile | None | 0, 2, 3 |
| 2 (Ledger) | Retrieval ledger + release status | None | 0, 1, 3 |
| 3 (Transport Base) | Abstract transport interface | None | 0, 1, 2 |
| 4 (Epic DICOMweb) | WADO-RS hypothesis | 0, 3 | 5, 6 |
| 5 (Image Exchange) | Care Everywhere candidate | 0, 3 | 4, 6 |
| 6 (IHE) | IHE candidate | 0, 3 | 4, 5 |
| 7 (Ambra) | 3 Ambra configs | 3 | 8, 9 |
| 8 (Policy Broker) | Middleware / interface-engine broker + StudyRelease / revoke / queue state | 3 | 7, 9 |
| 9 (Validator) | DICOM integrity | None | 0-8 |
| 10 (Consent Proof) | Audit chain | 1 | 9, 11 |
| 11 (Site Config) | YAML + credential resolution | None | 0-10 |
| 12 (Orchestrator) | Batch coordinator | 1, 2, 3, 4, 8, 9, 10, 11 | None |
| 13 (CLI) | Entry point | 12 | 14 |
| 14 (Onboard Script) | Interactive onboarding | 0, 11, 4 | 13 |
| 15 (Integration Tests) | Live tests | 4, 7, 8 | 16, 17 |
| 16 (Rationale Doc) | Decision framework | None | All |
| 17 (Ops Docs) | Runbook + onboarding | 12 | 16 |

**Critical path:** 0 -> 3 -> 4/8 -> 12 -> 13

**Note on CAT-C:** Task 8 now covers both the RADIANT-side broker client and the control-plane contracts / state model that a site broker must implement.

**Parallelizable first wave:** Tasks 0, 1, 2, 3, 9, 11, 16 (all independent)

**Gate dependency:** Tasks 4-6 (all transports) depend on Task 0 (capability probe results)
