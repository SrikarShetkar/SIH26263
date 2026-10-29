# APAAR Sentinel
 
**Validate Student Records. Detect Anomalies. Enable Trusted Verification.**
 
Smart India Hackathon 2026 · Problem Statement **SIH26263** · Team **ShatVyuha** (Team ID 197434)
Theme: Smart Automation · Category: Software
 
> Development of an APAAR ID-Based Student Data Management and Verification Platform for AICTE-Approved Institutions.
 
---
 
## Overview
 
AICTE receives student information from approved institutions in different forms. PGDM institutions submit detailed data annually, while most others report only aggregate enrolment during approval. This leaves AICTE with limited student-level visibility.
 
**APAAR Sentinel** is a scalable, interoperable, and user-friendly rule-based, analytics-assisted platform that uses the **APAAR ID** as the primary key for student records. It helps AICTE:
 
- validate student-level records before and after submission,
- monitor admissions against approved intake,
- flag unusual enrolment patterns for **human review**,
- enable consent-based verification of student records.
> **Scope:** Sentinel checks the consistency of submitted records and flags potential discrepancies for human review. It does not independently prove attendance, and it does not label any student or institution as fraudulent. A flag is a lead, not proof.
 
---

## Table of Contents

- [Overview](#overview)
- [Key Features](#key-features)
- [User Flow](#user-flow)
- [Architecture](#architecture)
- [Security & Privacy](#security--privacy)
- [Validation Rule Catalogue](#validation-rule-catalogue)
- [Duplicate & Anomaly Detection](#duplicate--anomaly-detection)
- [Data Model](#data-model)
- [API Overview](#api-overview)
- [Evaluation (Synthetic Data)](#evaluation-synthetic-data)
- [Scale & Performance](#scale--performance)
- [Privacy & Governance](#privacy--governance)
- [Scope & Limitations](#scope--limitations)
- [Users & Roles](#users--roles)
- [Roadmap](#roadmap)
- [References](#references)
- [Team](#team)
 
---
 
## Key Features
 
| Area | What it does |
|---|---|
| **Upload & dry-run** | Institutions upload via CSV or API. A dry-run catches missing APAAR IDs, bad formats and impossible DOBs before submission. A what-if mode shows the effect of additional admissions on seat utilisation. |
| **APAAR validation** | Syntax check, then registry check through an adapter (reference mock in the prototype; the authorised API is swapped in at deployment without changing core logic). Unconfirmable IDs are held as **pending**, not rejected. |
| **Seat ledger** | Compares admitted students with approved intake. Supernumerary, NRI, OCI and Foreign National categories are tracked separately. Versioned per academic year and programme. |
| **Duplicate detection** | APAAR ID is the primary key. Fuzzy matching (RapidFuzz, Fellegi-Sunter-weighted) is a labelled fallback for missing or wrong IDs, used to find potential cross-institution duplicates. |
| **Cluster & case engine** | Builds a shared-contact graph (NetworkX) and groups related signals into one investigation case with evidence and the rule that fired. |
| **Anomaly detection** | Rule checks plus Isolation Forest (scikit-learn) as a statistical anomaly signal. |
| **Review & disputes** | Officer queue, institution response, correct-and-resubmit flow. Every step is logged in a tamper-evident audit log. |
| **Verification with consent** | Students create purpose-specific, time-bound, revocable QR codes signed with ECDSA P-256. Two-tier model: claim-only verification for employers (student-consent-bound), authoritative lookups for regulators (AICTE-issued credentials). |
| **Certificate API** | Certificates carry APAAR ID, student ID and a signed verification QR. |
| **Reports** | Policy, accreditation and regulatory reports — intake utilisation by state/programme, flagged-institution summaries, duplicate and pending-ID trends, aggregate-vs-record reconciliation gaps. |
| **Security & audit** | JWT + RBAC (gateway + service), TLS 1.3, field-level encryption of APAAR IDs, KMS-managed signing keys, consent ledger, HMAC-chained tamper-evident audit log (Haber-Stornetta style). |
 
---
 
## User Flow
 
```
Institution          System              System            AICTE Officer       AICTE             Verifier
Upload & dry-run --> APAAR validation --> Verify       -->  Review & resolve -> Monitor      -->  Verify with consent
(CSV / API)          (syntax+registry)    (seats,           (case, decision,    (dashboards,       (signed QR,
                                           duplicates,       dispute)            anomaly signals)   live status)
                                           clusters)
        ^------------------------ Correct & resubmit -------------------------|
```
 
---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│  CLIENTS (React · Chart.js)                                              │
│   • Institution Portal  • AICTE Dashboard  • Verifier Console            │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────────────┐
│  GATEWAY & SECURITY                                                       │
│   REST · OpenAPI · JWT + RBAC · Consent check · Rate limiting · TLS 1.3   │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────────────┐
│  CORE SERVICES                                                            │
│   • Ingestion & Validation      (Node/Express)   — upsert by APAAR ID    │
│   • Validation & Matching       (Python)         — duplicate candidates   │
│   • Cluster & Case Engine       (Python)         — graph + Isolation Forest│
│   • Review & Disputes            (Node)           — officer queue + audit  │
│   • Verify & Reports             (Node)           — signed QR + reports    │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────────────┐
│  DATA & AUDIT                                                             │
│   MySQL · Encrypted raw uploads · HMAC-chained audit log                  │
│   Consent ledger · Job queue (planned, async)                             │
└─────────────────────────────────────────────────────────────────────────┘
                                  │
┌─────────────────────────────────────────────────────────────────────────┐
│  EXTERNAL SYSTEMS (via Adapters)                                          │
│   • APAAR registry (adapter — reference mock)                            │
│   • AICTE portal   (adapter — stub contract)                             │
│   • Institution ERPs (CSV / API)                                          │
│   • DigiLocker / ABC (adapter — future)                                   │
└─────────────────────────────────────────────────────────────────────────┘
```

| Layer | Components |
|---|---|
| **Clients** (React, Chart.js) | Institution Portal (dry-run, own dashboard, disputes) · AICTE Dashboard (seats, risk, cases, reports) · Verifier Console (consent-based claims) |
| **Gateway & Security** | REST / OpenAPI · JWT + RBAC · Consent check · Rate limiting · TLS 1.3 |
| **Core services** | Ingestion & Validation (Node) · Validation & Matching (Python) · Cluster & Case Engine (Python) · Review & Disputes (Node) · Verify & Reports (Node) |
| **Data & Audit** | MySQL · Encrypted raw uploads · Job queue (planned, async) · HMAC-chained audit log · Consent ledger |
| **External systems** | APAAR registry (adapter — reference mock) · AICTE portal (adapter — stub contract) · Institution ERPs (CSV / API) · DigiLocker / ABC (adapter — future) |

### Why two runtimes

- **Node/Express** handles the transactional API and workflow, where ecosystem and developer speed matter.
- **Python** handles analytics, where Pandas, RapidFuzz, NetworkX and scikit-learn are the natural tools.
- They communicate through well-defined service interfaces, so each can be scaled and replaced independently.

### Adapter pattern

All external systems sit behind adapters. The prototype uses a reference mock registry and an AICTE API stub. In production, the authorised APIs are swapped in without changing core logic. The adapter contract is production-ready; only the concrete call to the authorised APAAR registry is swapped at deployment.
 
### Tech stack
 
- **Frontend:** React.js, Chart.js
- **Backend & data:** Node.js, Express, MySQL
- **Analytics (Python service):** Pandas, RapidFuzz, NetworkX, scikit-learn, Bloom-filter PPRL *(planned)*
- **Security:** JWT, RBAC, TLS 1.3, ECDSA P-256 (QR signatures), HMAC-SHA256 (audit chain, keyed matching), field-level encryption (APAAR IDs), KMS-managed keys
- **DevOps & QA:** Git + GitHub, pytest, Jest, Docker *(planned)*
---

## Security & Privacy

| Layer | Mechanism |
|-------|-----------|
| **Authentication** | JWT |
| **Authorization** | RBAC enforced at gateway + again in each service (one compromised layer does not expose everything) |
| **Transport** | TLS 1.3 |
| **Signatures (QR)** | ECDSA P-256 (asymmetric — third-party verifiable without holding a secret) |
| **Audit chain / keyed matching** | HMAC-SHA256 (symmetric — platform-only verification) |
| **APAAR ID storage** | Field-level encryption; matching uses HMAC-derived digest, not plaintext ID |
| **Key management** | KMS-managed signing keys (never in application config) |
| **Rate limiting** | At gateway |
| **Audit trail** | HMAC-chained, tamper-evident (Haber-Stornetta style). Editing an old entry breaks the chain and is detectable. |
| **Consent** | Purpose + expiry + revocable; gateway checks consent before any claim is served |
| **Privacy-preserving matching** | HMAC-keyed digests for exact cross-institution matching; Bloom-filter PPRL (planned) for fuzzy matching outside the trust boundary |

### DPDP Act, 2023 Alignment

| DPDP Principle | Sentinel Implementation |
|----------------|------------------------|
| **Consent** | Consent ledger with purpose, expiry, revocation |
| **Purpose limitation** | Purpose-bound tokens; verifiers receive only the authorised claim |
| **Data minimisation** | Each role sees only what it needs; HMAC-keyed matching for cross-institution dedup |
| **Retention** | Defined retention policy; raw uploads and stale data not kept indefinitely |

---

## Validation Rule Catalogue

### Format and completeness
- APAAR ID present, correct length and character pattern
- Mandatory fields present and non-empty
- Date of birth parseable and plausible for the programme (flags impossible DOBs)
- Category flags consistent (e.g., NRI and Foreign National combinations)

### Within-file checks
- Duplicate APAAR ID inside the same file
- Category counts consistent with row-level flags

### Registry checks (through the adapter)
- APAAR ID exists and is active; where the registry returns identity attributes, name and DOB are checked for broad consistency. Unconfirmable IDs are held as **pending** (not rejected).

### Seat and intake checks
- Admitted count versus approved intake, overall and by category
- Aggregate declared versus aggregate derived from records (aggregate-to-record reconciliation)

### Cross-institution checks
- Same APAAR ID active at more than one institution
- Tier 2 fallback candidates for records without a usable ID

### Pattern checks
- Isolation Forest anomaly score on institution-level features (utilisation ratio, share of pending IDs, duplicate rate, contact-sharing density, category mix, late-admission patterns)
- Shared-contact clusters spanning 2 or more institutions

> Each rule has an identifier and a plain-language explanation, so every flag can display *"what fired and why"*.

---

## Duplicate & Anomaly Detection

### Layered matching (tiered)

- **Tier 1** — Exact APAAR ID match across institutions. Primary and most reliable signal.
- **Tier 2** — Runs only for records whose ID is missing, invalid, or pending. Compares normalised name, date of birth and guardian/contact fields using weighted fuzzy scoring (Fellegi-Sunter-style weighting, implemented with RapidFuzz). Output is always labelled *"candidate"* and the reviewer sees which fields matched and how strongly.

> *Why match names at all when APAAR is unique?* Because APAAR IDs can be missing, mistyped, still pending issuance, or wrongly assigned. A platform that only works when every ID is perfect would fail in the first month of real data.

### Layered detection

| Layer | Purpose | Explainability |
|-------|---------|----------------|
| **Layer 1 — Rules** | Transparent checks (utilisation > 100%, duplicate APAAR IDs, impossible dates, missing mandatory fields, category count mismatches) | Fully explainable |
| **Layer 2 — Isolation Forest** | Statistical anomaly signal at institution level — catches patterns a fixed rule would miss | Score + contributing features |
| **Layer 3 — Graph clustering** | Shared-contact graph (NetworkX) — links records sharing contact or guardian details; escalated only if spanning 2+ institutions | Cluster members + shared fields |

> Rules come first because they are explainable. The ML layer supplements them, it does not replace them. **No layer ever outputs a "fraud" label.**

### One case, not dozens of alerts

Sentinel groups related signals into a single investigation case per institution. An officer opens one case and sees the whole picture, ranked by risk priority, instead of triaging hundreds of separate warnings.

---

## Data Model

Core entities (MySQL):

| Entity | Key fields |
|--------|------------|
| **Institution** | id, name, approval details, programmes, status |
| **Programme/intake** | institution, programme, academic year, approved intake, supernumerary allowance |
| **Student record** | APAAR ID (primary key for matching; stored field-level encrypted; matching uses HMAC-derived digest), name, DOB, programme, institution, admission year, category (NRI/OCI/FN/supernumerary flags), contact and guardian details, record status (verified/pending/flagged/cleared) |
| **Submission** | batch id, institution, file reference, dry-run or committed, row-level results |
| **Seat ledger** | institution, programme, year, approved intake, admitted count by category, utilisation, version |
| **Case** | institution, signals, evidence, risk priority, status, assigned officer, decision |
| **Dispute/response** | case reference, institution response, attachments, correction submissions |
| **Consent** | student, purpose, verifier, expiry, status (active or revoked) |
| **Audit log** | sequence number, actor, action, object, timestamp, previous-entry HMAC, entry HMAC |

Raw uploads are stored encrypted and separately from the working tables.

**Mandatory fields** (checked in dry-run): APAAR ID, student name, date of birth, programme, institution, admission year, category. Optional fields cover contact and guardian details used for cluster analysis.

---

## API Overview

The platform exposes a REST API documented with OpenAPI. High-level endpoint groups:

| Group | Method | Path | Description |
|-------|--------|------|-------------|
| Auth | POST | `/auth/login` | Login (institution admin / AICTE officer / student / verifier) |
| Institution | POST | `/upload` | Upload student records CSV |
| Institution | POST | `/dry-run` | Non-destructive validation + what-if |
| Institution | POST | `/commit` | Commit a validated batch |
| Institution | GET | `/dashboard` | Institution's own dashboard |
| AICTE | GET | `/cases` | List of investigation cases |
| AICTE | POST | `/cases/:id/decision` | Officer decision (clear / request correction / escalate) |
| Institution | POST | `/cases/:id/respond` | Institution response / correction |
| Student | POST | `/consent` | Create purpose-specific signed QR |
| Student | DELETE | `/consent/:id` | Revoke consent |
| Verifier | GET | `/verify/:qr` | Verify a student-issued QR (returns claim only) |
| Reports | GET | `/reports/utilisation` | Intake utilisation report |
| Reports | GET | `/reports/accreditation` | Accreditation-ready report (NBA/NAAC format) |
| Certificate | POST | `/certificate/issue` | Issue certificate with APAAR ID + signed QR |

---
 
## Prototype Status
 
| Built | Mocked (behind adapters) | Planned |
|---|---|---|
| Upload + dry-run | APAAR registry (reference mock) | Bloom-filter PPRL |
| Seat ledger + what-if | AICTE portal API (stub contract) | Background job queue |
| Duplicate candidates | Institution data (synthetic) | Official APAAR API |
| Clusters + case review | | ERP + DigiLocker links |
| Audit log + signed QR | | Docker packaging |
| Isolation Forest anomaly signal | | |
 
**Dependency fallbacks**
 
- Official APAAR API unavailable → adapter with mock registry; records held as "pending".
- AICTE portal link unavailable → standalone API + CSV export.
- Limited institution buy-in → CSV template, PGDM pilot first.
 
---
 
## Evaluation (Synthetic Data)
 
All results below come from **synthetic data with anomalies planted by the team** (known 5% anomaly rate). They demonstrate the pipeline works as designed and are **not claims about real-world performance**.
 
| Check | Result |
|---|---|
| Duplicate detection | 91.5% recall, 99.9% precision (500 planted pairs) |
| Cluster detection | 20 / 20 planted clusters found; precision 80% with the 2+ institution rule |
| Isolation Forest | ~75% precision and recall, vs 37% recall for a simple utilisation-rule baseline |
| Latency | 1M APAAR syntax checks in ~0.5 s (1 core, excl. DB). Measures in-memory format/syntax validation only; DB writes, registry calls and fuzzy matching are separate costs. |
 
Metrics tracked going forward: precision, recall, F1, validation latency, data completeness, flag confirmation rate, dispute turnaround time.
 
---

## Scale & Performance

**Sizing assumption:** ~3 million student records per year (working assumption, not an AICTE figure; should be re-sized with real numbers during pilot).

- Cheap checks (format, completeness, within-file duplicates) run synchronously at upload — feedback in seconds
- Registry lookups go through the adapter with caching and the pending state — a slow registry never blocks an upload
- Fuzzy matching and graph analysis run in near-real-time batches with a nightly full re-run
- A background job queue (planned) will make batch execution asynchronous and retryable
- Database indexing on APAAR ID, institution and academic year keeps upserts and lookups fast; analytics reads can move to a replica
- Services are stateless behind the gateway — they scale horizontally

---
 
## Privacy & Governance
 
- Designed to align with the **Digital Personal Data Protection Act, 2023**: data minimisation, encryption, RBAC, retention policy.
- Third-party verification is **consent-based, purpose-specific, expiring and revocable**.
- Every flag goes through institution response → officer decision → audit log. Mistaken flags are cleared.
- No automated fraud labelling.

---

## Scope & Limitations

We state these upfront because honest scoping is part of the design.

- Sentinel checks the **consistency** of submitted records. It **cannot prove attendance**, and it cannot detect a fabricated record that is internally consistent and consistent with the registry.
- Quality of matching and validation depends on access to the official APAAR API.
- Flags can be wrong. That is why the process always includes institution response, officer decision and audit, and why mistaken flags are cleared on the record.
- Results so far are on synthetic data and need validation on real pilot data.
- Isolation Forest scores are statistical signals; we present them as supporting evidence, not as findings.
- Several integrations (official APAAR API, AICTE portal, ERP, DigiLocker/ABC) are mocked or planned in the prototype.

> Sentinel never labels a student or institution as fraudulent on its own. A flag is a lead, not proof.

---

## Users & Roles

| Role | Capabilities |
|------|--------------|
| **Institution administrator** | Uploads and updates student-level data, runs dry-runs and what-if checks, views own dashboard, responds to flags, resubmits corrections |
| **AICTE officer** | Reviews investigation cases, sees evidence and the rule that fired, requests clarification, records decisions (clear / request correction / escalate). Cannot edit student records directly. |
| **AICTE analyst / policy user** | Sees aggregated dashboards and reports. No access to raw personal data unless separately authorised. |
| **Student** | Views verified record, creates/lists/revokes consent-based verification QR codes |
| **Authorised verifier (employer)** | Scans student-issued QR, receives only the authorised claim with live status. Never sees the full record. |
| **Authorised verifier (regulator)** | Authoritative lookups with AICTE-issued credentials (two-tier model) |
| **System administrator / auditor** | Manages access, keys, retention; verifies audit-chain integrity. Access is itself logged. |

> RBAC is enforced at the gateway and again in each service, so one compromised layer does not expose everything.

---
 
## Roadmap
 
1. **Ingest & validate** – upload, dry-run, APAAR checks, upsert
2. **Verify & match** – seat ledger, duplicates, cross-institution matching
3. **Cases & proof** – case review, disputes, signed QR, certificate API
4. **Pilot & scale** – PGDM pilot, official APAAR / AICTE integration, job queue, Docker, Bloom-filter PPRL, threshold tuning from officer feedback
 
---
 
## References
 
**Policy & domain**
- National Education Policy (NEP) 2020, Ministry of Education, Government of India
- APAAR ID, One Nation One Student ID initiative
- Academic Bank of Credits (ABC)
- AICTE Approval Process Handbook
- Digital Personal Data Protection Act, 2023

**Methods**
- Fellegi & Sunter (1969), *A Theory for Record Linkage*, JASA — *used for fallback record linkage*
- Schnell, Bachteler & Reiher (2009), *Privacy-preserving record linkage using Bloom filters*, BMC Medical Informatics and Decision Making — *planned for privacy-aware fuzzy matching*
- Liu, Ting & Zhou (2008), *Isolation Forest*, IEEE ICDM — *used for statistical anomaly signal*
- Haber & Stornetta (1991), *How to Time-Stamp a Digital Document* — *used as basis for audit-log chain*
 
---
 
## Team
 
**ShatVyuha** · Smart India Hackathon 2026
 
| Name | 
|---|
| Srikar Shetkar |
| Manaswitha Garine | 
| Mahendra U | 
| Vamshi Krishna Chelimella | 
| Latika Kunchala | 
| Kevna Kokkonda | 

---

## Links

- **Repository:** https://github.com/SrikarShetkar/SIH26263.git
- **Problem Statement ID:** SIH26263
- **Theme:** Smart Automation
- **Category:** Software

---

> **APAAR Sentinel** — *from approval-time numbers to verifiable student records.*
