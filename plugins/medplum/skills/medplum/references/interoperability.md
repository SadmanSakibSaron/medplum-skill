# Integrations and interoperability

"Integrations are the product." Medplum connects outward through standards (FHIR (g)(10), SMART, Bulk, C-CDA, CDS Hooks, FHIRcast, DICOMweb, TEFCA). It connects to legacy systems through the on-prem Agent (HL7v2/MLLP, ASTM, DICOM DIMSE), and to vendors through first-party integrations (Health Gorilla, DoseSpot, ScriptSure). Custom work runs as Bots you own.

## Integration philosophy

- **"Any system that exposes an API (REST/FHIR), HL7, or SFTP interface can be connected to Medplum,"** so you are not limited to a catalogue. Custom integrations run as Bots in your own project, which means you own them and can move them. (`/docs/integration`)
- **Standard APIs beat integrators.** The old path was integrators plus HL7 over VPN, "painful, brittle and costly". (g)(10) FHIR APIs let Codex connect to Epic and Cerner with no setup fee. Medplum is an open-source (g)(10) implementation, so it doubles as a writable test EHR. (`/blog/codex-and-the-power-of-g10`)
- **Pull in batches, write in real time.** EHRs throttle or crash under load. (`/blog/codex-and-the-power-of-g10`)
- **"FHIR is the delivery format, not the work."** Clinical-grade data comes from a pipeline: multi-coded LOINC, `derivedFrom`, device and algorithm version, Provenance, validation against US Core 6.1.0 (because it matches (g)(10)), and idempotent upserts. "'it works in the demo' is not the same as 'I'd run this in production'." (`/blog/ble-to-fhir-anybio-medplum`)
- **Payer APIs are "extremely variable".** Flexpa normalises 200+ of them with self-hosted Medplum as a consent-scoped cache. (`/blog/flexpa-case-study`)
- **Ecosystems over monoliths.** Radiology is "a bellwether". "Proprietary notification systems are a walled garden", while an open-source FHIRcast hub is a community asset. (`/blog/ihe-ira-radiology-reporting`, `/blog/radai-case-study`)

## HL7v2 and the Agent

- **"HL7 should only be used when necessary".** It is unencrypted, "cannot be sent over the open internet in a compliant manner", and ~95% of US institutions use it, each differently. Expect per-organisation customisation. Consume only the events you need (e.g. ADT A04/A08). (`/docs/integration/hl7-interfacing`, `/docs/integration/hl7-interfacing/orders-and-results`, `/docs/integration/hl7-interfacing/adt`)
- **The Agent is a thin on-prem bridge** (Windows service, Linux or Docker). It connects over outbound WebSockets, so no VPN and no inbound firewall rule. Setup is Endpoint + Bot + Agent + a dedicated ClientApplication with a policy. `Agent/$push` sends orders from the cloud to devices. It replaces Mirth, whose logic ran on site in Java/Rhino. (`/docs/agent`, `/docs/agent/push`, `/blog/medplum-for-mirth-users`)
- **"A 'slow' HL7 interface is almost never slow because of the network."**
  - In original ACK mode, throughput ≈ 1/(latency + RTT), about 3–4 msg/s.
  - The fix, in order: enhanced/Fast ACK or AA mode; a durable queue ("Backpressure is not durability"); then logical channels keyed per patient, which give 300–400+ msg/s.
  - "Choosing a key is a clinical-safety decision": watch for merges (A18/A40).
  
  (`/docs/agent/high-throughput-hl7`, `/docs/agent/acknowledgement-modes`)
- **Operate fleets remotely:** `$status`, `$stats` (queue depth and RTT), `$fetch-logs` (the main log is PHI-free; the channel log may contain PHI), `$reload-config`, `$upgrade`. Names and tags "are conventions, not guarantees", so target by ID. Upgrade server and agent together. Clock skew causes "Token expired". (`/docs/agent/features`, `/docs/agent/configuration`, `/docs/agent/using-search-parameters`, `/docs/agent/troubleshooting`)
- **ASTM analyzers:** frame on ENQ/EOT, never STX/ETX. ACK exactly ENQ, ETX and ETB. Set `ignoreResponse=true`. Validate checksums in the Bot. (`/docs/agent/astm-channels`)

## Standards surface

- **SMART App Launch 2.0:** `launchUri` is "the most critical field", and `launchIdentifierSystems` returns external IDs. Test with Inferno. (`/docs/integration/smart-app-launch`, `/docs/api/fhir/operations/clientapplication-smart-launch`)
- **Bulk Data 2.0:** group or system `$export` to NDJSON; the policy must allow reading AsyncJob. The CLI can also pull from external servers (BCDA, Epic) through profiles, and working flows can then move into Bots. (`/docs/api/fhir/operations/bulk-fhir`, `/docs/cli/external-fhir-servers`)
- **C-CDA via IPS as the bridge:** `$everything` → `$summary` → C-CDA. Transitions of care are sent as a Communication over Direct. (`/docs/integration/c-cda`, `/docs/api/fhir/operations/ccda-export`, `/docs/api/fhir/operations/patient-summary`)
- **CDS Hooks:** a Bot becomes a CDS service through configuration (`cdsService`), not a separate server. (`/docs/integration/cds-hooks`)
- **FHIRcast STU3 hub,** with IHE IRA Hub certification (among the first). Radiologists lose 1–2 minutes per patient syncing apps by hand; `versionId`/`priorVersionId` keep events ordered. (`/docs/fhircast`, `/blog/ihe-ira-radiology-reporting`)
- **TEFCA** is a document-first "network of networks" of QHINs. Medplum's roadmap to become a FHIR responding node: UDAP, X.509, `$match`, Provenance. (`/blog/technical-guide-to-tefca`)
- **Log streaming:** request IDs are always generated by the server, trace IDs propagate (W3C `traceparent`), and `X-Medplum-Log-Tag` identifies end users, but never carries PHI and is never used for authorisation. (`/docs/integration/log-streaming`)

## Imaging (DICOM Beta)

- **Imaging was "the part of the patient record that lives somewhere else".** Now DICOMweb (`/dicomweb`), Agent DIMSE C-STORE and CLI STOW ship together "because none of them is useful alone". (`/blog/dicom-beta`, `/docs/dicom`)
- **Stored natively, not as ImagingStudy:** "Translate at ingest and you throw away exactly the attributes a viewer needs." DicomStudy/Series/Instance plus the original .dcm, with frames extracted asynchronously. (`/blog/dicom-beta`, `/docs/dicom/data-model`)
- **No second authorisation model:** "There is no separate DICOM credential to manage", and OHIF works with no gateway. `DicomStudy.patientId` is a string, so reconcile it to the chart in a Bot. Always quote `**` globs in the CLI. (`/docs/dicom/dicomweb-api`, `/docs/dicom/ohif-viewer`, `/docs/dicom/cli`)

## Labs: Health Gorilla

- **Unified Quest/Labcorp ordering.** An order is a parent ServiceRequest (with ICD-10 diagnoses) plus child tests (compendium code, AOE answers). Validate the patient profile at registration, not at order time. Once `active`, an order is immutable at the lab. (`/docs/integration/health-gorilla`, `/docs/integration/health-gorilla/sending-orders`)
- **Results** are matched via basedOn → accession → placer → filler. Unsolicited or unknown-patient results become DetectedIssues. Backfill with `syncOnlyMissing`. Migrate in phases: receive-only → placeholder orders → ordering. (`/docs/integration/health-gorilla/receiving-results`, `/docs/integration/health-gorilla/sync-resources-from-health-gorilla`)
- **Hardening lessons:** switching PUT to PATCH protected async identifiers, and dropping practitioner email sync blocked reset attacks. The iframe URL holds a token, so never log it. (`/docs/integration/health-gorilla/hg-changelog`, `/docs/integration/health-gorilla/iframe`)

## E-prescribing

- **eRx is integration and enrollment decisions, not a data model.** Choose iframe (DoseSpot: fastest, fixed UX) or integrated API (ScriptSure: you own the workflow except the required send widget). "EPCS is a one-way gate on go-live", so start proofing early. Bad Practitioner NPI, phone or address data is the top cause of enrollment errors. (`/docs/decision-guides/e-prescribe`, `/docs/integration/dosespot/enroll-user`)
- **Coverage and cost come from the benefit network,** not from Medplum's insurance data. Replacing a vendor requires a SureScripts Change of Vendor. MedicationRequest `intent` records provenance: plan = self-reported, order = prescribed, original-order = external history. (`/docs/decision-guides/e-prescribe`, `/docs/integration/dosespot/getting-started`)
- **Vendor-neutral hooks** (`useMedicationOrder`, `$drug-search`, `$order-medication`) route to vendor bots through OperationDefinitions. A failed cart read throws rather than returning empty. Medication history is gated by Consent. (`/docs/integration/scriptsure/order-medication`, `/docs/integration/scriptsure/medication-cart`, `/docs/integration/scriptsure/sync-patient`)

## Key source articles
`/docs/integration` · `/blog/codex-and-the-power-of-g10` · `/docs/agent/high-throughput-hl7` · `/docs/integration/hl7-interfacing` · `/blog/medplum-for-mirth-users` · `/blog/dicom-beta` · `/blog/ble-to-fhir-anybio-medplum` · `/docs/decision-guides/e-prescribe` · `/docs/integration/health-gorilla/sending-orders` · `/blog/ihe-ira-radiology-reporting`
