# Compliance, certification, security

Medplum treats certification as part of the product. Customers building on the hosted service inherit ONC, SOC 2 Type II, HIPAA, HITRUST e1 and more, and reuse the evidence. "Compliance is a shared responsibility." Open source is treated as something you have to *prove* secure, not assume.

## Certifications as a product

- **Hosted customers inherit certifications:** ONC, HITRUST e1, SOC2 Type II, HIPAA, CLIA/CAP support, 21 CFR 11, ISO 9001, HTI-4 and GMP. Use them for security questionnaires, for your own audits, and to be eligible to sell. Evidence lives in the Vanta Trust Center. (`/docs/compliance`)
- **"The most important thing to understand about the certification is that it requires FHIR API access - for patients and practitioners."** ONC pays off in CMS reimbursement and payer contracting. "Prepare for frequent requirements changes", and certify subsets over time. (`/blog/what-is-onc-certification`)
- **ONC-certified v5** (CHPL, 12/31/2025) covers a2, a5, a14, b1, b10, b11, c1, d1–d13 and g3–g10, under HTI-1 with US Core 5.0.1 via SVAP. (`/docs/compliance/onc`)
- **(b)(10) EHI Export:** Medplum is probably the only open-source implementation, and only 70 of 708 EHRs had certified it. "The key benefit of open source is composability": you add compliance progressively instead of ripping and replacing. (`/blog/onc-b10-certification-medplum`)
- **Inheriting certification in practice:** Profile connected to an HIE whose reciprocity obligations require ONC certification. Building on hosted Medplum meant no de novo certification. (`/blog/profile-case-study`)
- **Track record:** SOC 2 Type I in early 2022, now Type II with controls monitored continuously, "beyond the audit window". HITRUST e1 was certified June 1 2026. HIPAA BAA/MSA templates apply to hosted only. (`/blog/soc2-type1`, `/docs/compliance/soc2`, `/blog/hitrust-e1-certification`, `/docs/compliance/hipaa`)

## Regulation on the horizon

- **HTI-4 (effective Oct 2025)** adds eRx (SCRIPT v2023011), RTPB and FHIR prior auth (CRD g31, DTR g32, PAS g33). CMS-0057-F makes payers ready by Jan 1 2027. Estimated labour savings: ~$19B over ten years. Medplum is pursuing all three. Live-testing them also covers (j)(20)/(j)(21). (`/docs/compliance/hti-4`, `/blog/fhirplace-participant-2026`)
- **CDS under (a)(9)/HTI-1:** predictive DSIs need Insights reports. Medplum provides the building blocks but is not (a)(9) certified. (`/docs/careplans/clinical-decision-support`)

## Mapping regulations onto FHIR practice

- **CLIA/CAP (Medplum as LIS):**
  - Authentication via memberships and policies.
  - "All calculations that include reportable results must have unit tests" (Bots).
  - Specimen rejections use v2-0490 codes.
  - Critical-result call logs are CommunicationRequest + Communication.
  - Turnaround runs from collection to `DiagnosticReport.issued`.
  
  (`/docs/compliance/clia-cap`)
- **21 CFR 11:** audit trails and versioning cover §11.10. For signatures (§11.50/70), integrate a compliant service such as DocuSign. Consent signatures map `who`/`when`/`type`. (`/docs/compliance/cfr11`, `/docs/consent`)
- **GMP:** Medplum sits beside the QMS/LMS/ERP rather than being the source of truth for SOPs. Subscriptions and Bots are the validation and alerting mechanism. (`/docs/compliance/gmp`)
- **ISO 9001:** QMS clauses map to scrum ceremonies, PR review and automated verification (tests, scanners, Inferno). Open source makes the practices publicly visible. (`/docs/compliance/iso9001`)

## Availability and data durability

- **"The main line of defense is live redundancy, not backups."** Aurora readers are ~20ms behind across AZs, and a Global Database ~100ms behind in a second US region. When hardware fails: "Nothing is restored, because nothing was lost." (`/docs/compliance/backup-and-recovery`)
- **Backups exist for logical damage.** Replication faithfully copies bad migrations, so PITR (7 days) is "the only defense against that class of event". RTO/RPO <1h is the worst case (losing a region). Data never leaves the US, and retention is indefinite. (`/docs/compliance/backup-and-recovery`)
- **The SLA is 99.99% (~4.3 min/month); actual delivery has run at five to six nines.** "No component requires downtime to operate, scale, or upgrade." Self-hosted deployments aren't covered. (`/docs/uptime`)
- **AuditEvents are the queryable record; the log stream feeds your SIEM.** Soft delete by default; `$expunge` for regulatory deletion. (`/docs/compliance/backup-and-recovery`)

## Security posture

- **"Being 'open source' isn't enough; we need to actively prove our commitment to security."** OpenSSF Best Practices gold, and Scorecard 9.5+ against a research-software mean of 3.5. First results were "humbling". (`/blog/openssf`)
- **Beg bounties:** no discussion of money until there are full details and a reproduction. Pay only for novel, verifiable impact. Suboptimal headers, missing SPF/DMARC and raw scanner output are explicitly out of scope. Genuine reports are rare because of regular third-party pentests. (`/blog/security-reports`)
- **Platform defaults that matter for audits:**
  - Password-reset responses never enumerate users.
  - Passwords are checked against breach databases.
  - Secrets rotate with zero downtime.
  - mTLS and client assertion are available for payer-grade authentication.
  
  (`/docs/api/auth/resetpassword`, `/docs/api/auth/setpassword`, `/docs/api/fhir/operations/rotate-client-secret`, `/docs/auth/mtls`)
- **The warehouse is a PHI system that access policies don't govern:** treat it with a BAA, access reviews and de-identification. (`/docs/analytics/redshift`)

## Key source articles
`/docs/compliance` · `/blog/what-is-onc-certification` · `/blog/onc-b10-certification-medplum` · `/docs/compliance/onc` · `/docs/compliance/hti-4` · `/docs/compliance/backup-and-recovery` · `/docs/uptime` · `/blog/openssf` · `/blog/security-reports` · `/docs/compliance/clia-cap`
