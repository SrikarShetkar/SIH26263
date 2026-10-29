# APAAR Sentinel
 
**Validate Student Records. Detect Anomalies. Enable Trusted Verification.**
 
Smart India Hackathon 2026 · Problem Statement **SIH26263** · Team **ShatVyuha** (Team ID 197434)
Theme: Smart Automation · Category: Software
 
> Development of an APAAR ID-Based Student Data Management and Verification Platform for AICTE-Approved Institutions.
 
---
 
## Overview
 
AICTE receives student information from approved institutions in different forms. PGDM institutions submit detailed data annually, while most others report only aggregate enrolment during approval. This leaves AICTE with limited student-level visibility.
 
**APAAR Sentinel** is a rule-based, analytics-assisted platform that uses the **APAAR ID** as the primary key for student records. It helps AICTE:
 
- validate student-level records before and after submission,
- monitor admissions against approved intake,
- flag unusual enrolment patterns for **human review**,
- enable consent-based verification of student records.
> **Scope:** Sentinel checks the consistency of submitted records and flags potential discrepancies for human review. It does not independently prove attendance, and it does not label any student or institution as fraudulent. A flag is a lead, not proof.
 
---
 
## Key Features
 
| Area | What it does |
|---|---|
| **Upload & dry-run** | Institutions upload via CSV or API. A dry-run catches missing APAAR IDs, bad formats and impossible DOBs before submission. A what-if mode shows the effect of additional admissions on seat utilisation. |
| **APAAR validation** | Syntax check, then registry check through an adapter (mock registry in the prototype). |
| **Seat ledger** | Compares admitted students with approved intake. Supernumerary, NRI, OCI and Foreign National categories are tracked separately. |
| **Duplicate detection** | APAAR ID is the primary key. Fuzzy matching (RapidFuzz) is a fallback for missing or wrong IDs, used to find potential cross-institution duplicates. |
| **Cluster & case engine** | Builds a shared-contact graph (NetworkX) and groups related signals into one investigation case with evidence and the rule that fired. |
| **Anomaly detection** | Rule checks plus Isolation Forest (scikit-learn) as a statistical anomaly signal. |
| **Review & disputes** | Officer queue, institution response, correct-and-resubmit flow. Every step is logged. |
| **Verification with consent** | Students create purpose-specific, revocable, signed QR codes. Verifiers get a claim-only answer with live status. |
| **Certificate API** | Certificates carry APAAR ID, student ID and a verification QR. |
| **Security & audit** | JWT + RBAC, TLS 1.3, encrypted raw uploads, consent ledger, HMAC-chained tamper-evident audit log. |
 
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
 
| Layer | Components |
|---|---|
| **Clients** (React, Chart.js) | Institution Portal (dry-run, own dashboard, disputes) · AICTE Dashboard (seats, risk, cases, reports) · Verifier Console (consent-based claims) |
| **Gateway & Security** | REST / OpenAPI · JWT + RBAC · Consent check · Rate limiting · TLS 1.3 |
| **Core services** | Ingestion & Validation (Node) · Validation & Matching (Python) · Cluster & Case Engine (Python) · Review & Disputes (Node) · Verify & Reports (Node) |
| **Data & Audit** | MySQL · Encrypted raw uploads · Job queue (planned, async) · HMAC-chained audit log · Consent ledger |
| **External systems** | APAAR registry (mock) · AICTE portal (API stub) · Institution ERPs (CSV / API) · DigiLocker / ABC (future) |
 
### Tech stack
 
- **Frontend:** React.js, Chart.js
- **Backend & data:** Node.js, Express, MySQL
- **Analytics (Python service):** Pandas, RapidFuzz, NetworkX, scikit-learn, Bloom-filter PPRL *(planned)*
- **Security:** JWT, RBAC, TLS 1.3, ECDSA P-256 (QR signatures), HMAC-SHA256 (audit chain, keyed matching)
- **DevOps & QA:** Git + GitHub, pytest, Jest, Docker *(planned)*
---
 
## Prototype Status
 
| Built | Mocked | Planned |
|---|---|---|
| Upload + dry-run | APAAR registry | Bloom-filter PPRL |
| Seat ledger + what-if | AICTE portal API | Background job queue |
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
 
All results below come from **synthetic data with anomalies planted by the team**. They demonstrate the pipeline works as designed and are not claims about real-world performance.
 
| Check | Result |
|---|---|
| Duplicate detection | 91.5% recall, 99.9% precision (500 planted pairs) |
| Cluster detection | 20 / 20 planted clusters found; precision 80% with the 2+ institution rule |
| Isolation Forest | ~75% precision and recall, vs 37% recall for a simple utilisation-rule baseline |
 
Metrics tracked: precision, recall, F1, validation latency, data completeness, flag confirmation rate.
 
---
 
## Privacy & Governance
 
- Designed to align with the **Digital Personal Data Protection Act, 2023**: data minimisation, encryption, RBAC, retention policy.
- Third-party verification is **consent-based, purpose-specific, expiring and revocable**.
- Every flag goes through institution response → officer decision → audit log. Mistaken flags are cleared.
- No automated fraud labelling.
---

## Roadmap
 
1. **Ingest & validate** – upload, dry-run, APAAR checks
2. **Verify & match** – seat ledger, duplicates, clusters
3. **Cases & proof** – case review, disputes, signed QR, certificate API
4. **Pilot & scale** – PGDM pilot, official APAAR / AICTE integration, job queue, Docker
---
 
## References
 
**Policy & domain**
- National Education Policy (NEP) 2020, Ministry of Education, Government of India
- APAAR ID, One Nation One Student ID initiative
- Academic Bank of Credits (ABC)
- AICTE Approval Process Handbook
- Digital Personal Data Protection Act, 2023
**Methods**
- Fellegi & Sunter (1969), *A Theory for Record Linkage*, JASA
- Schnell, Bachteler & Reiher (2009), *Privacy-preserving record linkage using Bloom filters*, BMC Medical Informatics and Decision Making
- Liu, Ting & Zhou (2008), *Isolation Forest*, IEEE ICDM
- Haber & Stornetta (1991), *How to Time-Stamp a Digital Document*
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
 

