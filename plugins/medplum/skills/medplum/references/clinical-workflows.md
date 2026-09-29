# Clinical workflows

How Medplum models the work of care. Tasks are the backbone; Questionnaires carry data in; PlanDefinitions and CarePlans hold protocols. Also covers charting (SOAP, signing), intake, referrals, consents, labs and imaging, medications, and the Beta scheduling API. The Medplum Provider app is the reference implementation.

## Tasks: workflow is a queue

- **Workflow apps are queues of Tasks.** Task is "a workhorse resource defining all clinical work items". Tasks and ServiceRequests are the most common async resources. (`/blog/task-management-apps`, `/docs/careplans`)
- **Field discipline:**
  - `status` is the coarse lifecycle (most apps use `ready`); `businessStatus` is your ops funnel; `statusReason` is orthogonal.
  - `for` is the beneficiary. `owner` is the performer; find unassigned tasks with `owner:missing`. Route to roles with `performerType` (SNOMED).
  - **Always populate `focus`**: it drives UI and metrics.
  - `restriction.period.end` is the due date.
  - Use the standard priority codes even when they feel awkward.
  
  (`/docs/careplans/tasks`)
- **Metrics come from timestamps:** turnaround = `executionPeriod.end − authoredOn`; work time = `executionPeriod.end − start`. Support permalinks to task searches. (`/blog/task-management-apps`)
- **Subtasks via `partOf`,** but start single-level. Queues route by specialty, credential and availability, which serves the top-of-license and fifty-state goals. (`/docs/careplans/tasks`)

## Protocols and care plans

- **Two modes.** PlanDefinition is the training manual; `$apply` produces a CarePlan + RequestGroup, the checklist on the chart. Goals are measurable targets. (`/docs/careplans`, `/docs/api/fhir/operations/plandefinition-apply`)
- **`$apply` executes a focused subset:** sequential Tasks and ActivityDefinition resolution. `kind` decides the output (ServiceRequest → Task + draft SR; Questionnaire → Task with form input). Conditional logic is "under active development". (`/docs/careplans/protocols`)
- **Longitudinal cases:** EpisodeOfCare is a lightweight grouping; CarePlan is the clinical layer. They are linked by `supportingInfo` and a shared Condition. "An Encounter… can't represent the full arc of care." (`/docs/careplans/longitudinal-patient-case-tracking`)
- **Timeline over episodes:** Everself built one chronological stream per patient because episodic EHRs make providers hop tabs. (`/blog/everself-case-study`)
- **CDS per ONC (a)(9)/HTI-1:** predictive (LLMs, risk; Insights reports required), linked referential (Infobutton), evidence-based (DoseSpot via SMART). Medplum supplies the pieces but is not (a)(9) certified. (`/docs/careplans/clinical-decision-support`)

## Charting

- **Lean structured, respect muscle memory.** Default: structured S, O and P; narrative Assessment. "If a section is faster as narrative, force-fitting a form is a step backward." "Charting changes have high adoption risk." (`/docs/charting/designing-charting`, `/docs/decision-guides/charting`)
- **Visit template = one PlanDefinition per visit type.** Subjective vs Objective is distinguished by `performer`. The Assessment lives on ClinicalImpression. **Sign** by completing the ClinicalImpression and adding a Provenance on the Encounter. Co-signing adds another Provenance; amendments are addendum DocumentReferences. (`/docs/charting/visit-templates`, `/docs/decision-guides/charting`)
- **Always parse forms into resources;** raw QuestionnaireResponses aren't queryable. (`/docs/charting/visit-templates`, `/docs/decision-guides/charting`)
- **Chart model rules:**
  - Condition category separates encounter diagnosis from problem list; a recurrence is a new Condition.
  - A single fever spike is an Observation. Allergies are never Conditions.
  - Record NKDA only when asserted.
  - Medplum "is not optimized for massive wearable ingestion", so store summaries.
  
  (`/docs/charting/chart-data-model`)

## Forms, intake, consent

- **"Is it even a healthcare app without tons of forms?"** Questionnaire has the most core extensions of any resource, and `subjectType` decides where a form appears. Distinguish `source`, `subject` and `author`. (`/blog/understanding-fhir-questionnaires`, `/docs/questionnaires`, `/docs/questionnaires/questionnaires-and-responses`)
- **Two ways to extract:** SDC `$extract` (templates in the form, no deploy, suits simple, often-edited forms) or a Bot (scoring, conditionals, external calls). Mixing both is normal. `templateExtractContext` is needed for optional or repeating items. (`/docs/questionnaires/parsing-questionnaire-responses`, `/docs/api/fhir/operations/extract`)
- **The linkId contract:** use semantic linkIds (`allergy-substance`), because renaming them later means migrating history, a one-way door. (`/docs/intake/intake-questionnaires`, `/docs/decision-guides/intake`)
- **Intake fans out into Patient, Coverage, Consent, RelatedPerson and clinical history,** upserted by natural keys so resubmission never duplicates. The Coverage `child` ↔ RelatedPerson `PRN` inversion is "one of the most common intake bugs". (`/docs/intake/intake-data-model`)
- **Intake automation:** a create-only Subscription scoped by `questionnaire`, so the Bot linking `subject` doesn't re-trigger. Failures become Tasks and are never silently dropped. Prefill shows and confirms, never silently overwrites. (`/docs/intake/post-intake-automation`, `/docs/decision-guides/intake`)
- **Consent:** one Consent per agreement. "A declined agreement is a rejected Consent, not a missing one." Supersede rather than edit, so audits can answer "what had they agreed to on date X". Signature pads work only on single-page forms. (`/docs/consent`)
- **SMART Health Links** ("Kill the Clipboard"): test against independent implementations, and resolve identity with `$match` before importing. (`/docs/intake/smart-health-links`)

## Orders: referrals, labs, meds

- **Referral = one ServiceRequest** as the source of truth across every channel. It is captured by Questionnaire, sent as a Communication (PDF or C-CDA for non-FHIR recipients), tracked by a Task with `businessStatus`, and closed by results that link `basedOn` it. Free-text recipients are a one-way door. (`/docs/careplans/referrals`, `/docs/decision-guides/referrals`, `/docs/careplans/referrals/transmition-and-tracking`)
- **Labs follow request-and-report:** ServiceRequest → Specimen → DiagnosticReport → Observations. Configuration (catalog, ranges) is separate from workflow. Amend orders in place; cancel and `replaces` when retransmitting. Always include the `LAB` category. (`/docs/labs-imaging`, `/docs/labs-imaging/ordering-labs-imaging`, `/docs/labs-imaging/results-and-review`)
- **Diagnostic catalog** (Order Catalog IG): ObservationDefinition, SpecimenDefinition, PlanDefinition (orderable panel) and ActivityDefinition (procedure). Ranges live in `qualifiedInterval`: reference, critical and absolute. (`/docs/careplans/diagnostic-catalog`, `/docs/careplans/reference-ranges`)
- **Meds:** RxNorm is what doctors prescribe; NDC is what pharmacies dispense. MedicationRequest carries no drug detail, which lives in MedicationKnowledge (the formulary, PDex). `numberOfRepeatsAllowed` excludes the first fill. eRx goes through DoseSpot. (`/docs/medications/medication-codes`, `/docs/medications/formulary`, `/docs/medications/representing-prescriptions-and-medication-orders`, `/docs/medications/e-prescibe`)

## Scheduling (Beta)

- **Four steps:** define the service (HealthcareService + SchedulingParameters) → availability (one actor per Schedule, with a timezone) → `$find` → `$book` or `$hold`/`$confirm`, and `$cancel`. (`/docs/scheduling`, `/blog/scheduling-beta`)
- **Implicit availability:** "Recurring availability does not require pre-generated slots". Only busy and blocked Slots persist, and `$find` results are virtual. (`/docs/scheduling`, `/docs/scheduling/appointment-find`)
- **`$book` is serializable,** so two requests can't both take the last slot. Multi-resource bookings (surgeon + OR) use contained Slots. Holds consume capacity. (`/docs/scheduling/appointment-book`)
- **Gotchas:**
  - Schedule overrides replace fields rather than merge.
  - End of day is `00:00:00`.
  - `alignmentTimezone` survives DST.
  - Use one HealthcareService per location.
  
  (`/docs/scheduling/defining-availability`)
- **Licensure isn't enforced on purpose:** "Bake that business logic into which Schedules you pass into $find". (`/docs/scheduling/state-by-state-licensure`)
- **Store dateTimes as submitted;** recurring rules need local time + timezone. (`/docs/scheduling/timezones`)

## Medplum Provider

- **An open-source EHR you can adopt, fork a start from, or read as a reference.** A Visit unifies Appointment + Encounter, generates the ClinicalImpression and Care Template tasks, and creates ChargeItems. Care Templates are no-code definitional resources. (`/docs/provider`, `/docs/provider/visits`)
- **Case studies:** Ro coordinated at-home diagnostics across labs and logistics (`/blog/ro-case-study`). Profile used an LLM intake grounded in Questionnaires and kept "the LLM… out of the clinical decision" (`/blog/profile-case-study`).

## Key source articles
`/docs/careplans/tasks` · `/blog/task-management-apps` · `/docs/decision-guides/charting` · `/docs/charting/visit-templates` · `/docs/decision-guides/intake` · `/docs/questionnaires/parsing-questionnaire-responses` · `/docs/consent` · `/docs/decision-guides/referrals` · `/docs/scheduling/defining-availability` · `/docs/careplans/protocols`
