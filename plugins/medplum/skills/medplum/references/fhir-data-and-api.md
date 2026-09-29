# FHIR data modeling and the API

How Medplum stores data and how to model it. It is FHIR-native on R4, so data is stored as FHIR, not translated from an internal schema. Covers systems and identifiers, extensions and profiles, workflow patterns, CRUD, history, batches, search, GraphQL, terminology, binaries, analytics exports, and the React layer.

## Modeling fundamentals

- **Store FHIR itself; don't `create table patient`.** Medplum is "FHIR Native": data lives in FHIR format, plus custom resource types for users, auth and access. The spec "anticipates many of the complexities" that otherwise force costly backend rewrites. (`/docs/api/fhir`, `/docs/fhir-basics`)
- **System strings are the #1 early confusion.** Identifier rule: one system string per source system (`https://hospitalname.org/patientId`), and search by `identifier=system|value` because MRN 123 means different patients at different hospitals. For codes, use standard systems where they exist; a local code's URL should show how widely it is agreed (`company.org/productLine` vs `company.org/messaging/productLine`). (`/blog/demystifying-fhir-systems`)
- **Classify every resource as Definition, Request or Event.** Definitions describe how things should work (PlanDefinition, ActivityDefinition, Questionnaire). Requests say "please do this" (ServiceRequest, the "Swiss Army knife of clinical orders", plus Task and CarePlan). Events say "this happened" (Observation, Procedure, Communication). Link them with `basedOn`, `instantiates`, `partOf` and `replaces`. Example: insurance-rejection Communications point at the Task with `basedOn`. (`/blog/fhir-workflow-patterns-to-simplify-your-life`)
- **No field? Add an extension:** one top-level extension at your institution URL with sub-extensions for each value. Extensions are versioned and audited like any field. (`/blog/fhir-extensions-intro`)
- **Profiles are "subclasses" enforced on write.** Upload the StructureDefinition to the project, declare it in `meta.profile`, author in FSH/SUSHI, and use Data Absent Reason for required-but-unknown values. Changing a profile is "more similar to a database migration": bump the version and revalidate yourself. (`/docs/fhir-datastore/profiles`)
- **Family models, simplest first:** Patient.contact, then RelatedPerson, then shared members factored into their own Patient via `link`, and Group when the family is the subject. Use one EpisodeOfCare per pregnancy. Avoid mixing in Person. (`/docs/fhir-datastore/family-relationships`)
- **Provider directory follows Da Vinci PDEX Plan Net.** Use one Practitioner per person and one PractitionerRole per organisation or network. Licences go in `Practitioner.qualification` with a `whereValid` state; specialties use NUCC codes. Put PractitionerRole, not Practitioner, on CareTeams. (`/docs/administration/provider-directory/provider-organizations`, `/docs/administration/provider-directory/provider-credentials`, `/docs/administration/provider-directory/provider-networks`)
- **Canonical lab write:** conditional-create the Patient by MRN ("a duplicate… would be incorrect (and confusing)"), then write the ServiceRequest, then Observations and a DiagnosticReport linked to both. (`/docs/fhir-datastore/working-with-fhir`)
- **USCDI compliance = FHIR modeling + correct code systems + required fields.** US Core 6.1.0 / USCDI v3. (`/docs/fhir-datastore/understanding-uscdi-dataclasses`)

## CRUD, history, batches

- **Soft delete by default.** A delete leaves a tombstone (410 Gone), and history stays readable. `$expunge` hard-deletes, is admin-only, and is irreversible, including a whole Project. "Referential integrity is not supported for deletes." (`/docs/fhir-datastore/deleting-data`, `/docs/api/fhir/operations/expunge`)
- **Prevent lost updates:** use `If-Match` (412 on mismatch) and a patch `test` on `/meta/versionId`, which is "strongly recommended… on all patch operations". Upsert means a conditional PUT: update the single match, create if there is none, error if several match. (`/docs/fhir-datastore/updating-data`)
- **History is never rewritten;** to revert, write an old version as a new one. There is no creation timestamp field, so read the first version's `lastUpdated`. (`/docs/fhir-datastore/resource-history`)
- **Transaction gotcha:** without the `transaction-bundles` project flag, a transaction "silently runs as a batch". Limits: ≤50 updates; conditional ops force serializable isolation with ≤8 entries; 8MB/60s synchronous. `Prefer: respond-async` raises the limit to 50MB and isn't counted against quota. (`/docs/fhir-datastore/fhir-batch-requests`, `/docs/fhir-datastore/processing-async-bundles`)
- **Link bundle entries with `urn:uuid` or conditional references;** use `ifNoneExist` for idempotency. Autobatching only helps with `Promise.all`. Use `$clone` to copy projects. (`/docs/fhir-datastore/fhir-batch-requests`)

## Search and retrieval

- **"FHIR doesn't allow Resources to be queried by arbitrary elements".** Only defined search params work, so add a custom SearchParameter or an extension-based one. `string` does a prefix match (`eve` finds Evelyn, not Steve); `token` is exact. (`/docs/search/basic-search`, `/docs/fhir-basics`)
- **Chaining filters, `_include` returns.** Prefer chaining when paginating, since includes make pages unpredictable. Chains and `_has` are REST-only. `_filter` handles cross-parameter ORs. (`/docs/search/chained-search`, `/docs/search/includes`, `/docs/search/filter-search-parameter`)
- **Pagination:** `_offset` goes up to 10k; `_cursor` needs `_sort=_lastUpdated` and handles millions. Always sort. `_count` defaults to 20 (max 1000). `_total=accurate` silently falls back to an estimate above ~1M. (`/docs/search/paginated-search`, `/docs/search/advanced-search-parameters`)
- **GraphQL vs REST: use both.** GraphQL for nested linked reads and exact fields (mobile). REST for complex search, PATCH, batch and history, since it is "the de-facto standard". GraphQL has no modifiers, chaining or history; introspection is off by default. (`/blog/graphql-vs-rest`, `/docs/graphql`)
- **"Speed isn't a property of the data standard."** A 131,000-resource chart reached the browser in about 5s. Use recent-first loading: last 30 days first, hydrate the rest. (`/blog/large-patient-charts`)
- **`Patient/$everything` is the EHI export format;** `$graph` walks a GraphDefinition (limit 1,000 resources, depth 5). (`/docs/api/fhir/operations/patient-everything`, `/docs/api/fhir/operations/resource-graph`)

## Terminology

- **Code shapes:** bare `code` only when the system is implied, otherwise Coding or CodeableConcept. Standard systems: LOINC for labs, RxNorm for meds, ICD-10-CM for billing diagnoses, SNOMED for granular problems and allergies, CPT/HCPCS for procedures, CVX for vaccines. (`/docs/terminology`, `/docs/terminology/common-terminologies`)
- **Don't bind to all of SNOMED or RxNorm; scope a ValueSet you own.** `descendent-of` excludes the parent. Use RxNorm TTY SCD/SBD for prescribable drugs and ICD `tty=PT` for billable leaves. Filters within one include are ANDed; separate includes are ORed. `content: fragment` imports silently return nothing for missing codes. (`/docs/terminology/filtering-large-code-systems`)
- **Validate at entry** (`$validate-code`) because invalid codes cascade into analytics, CDS and interop. `$translate` handles SNOMED→ICD using grouped, priority-ordered maps. `$subsumes` lets rules target categories. (`/docs/api/fhir/operations/codesystem-validate-code`, `/blog/terminology-2026`, `/docs/api/fhir/operations/codesystem-subsumes`)
- **v5 terminology:** synonym search ("Wheal" vs "hives"), `$expand` 95% faster (~450ms to <50ms), ConceptMap lookup tables at 10–50ms, and opt-in `validate-terminology` for required bindings. (`/blog/v5-terminology`)
- **Multilingual:** use the `translation` extension on `_field` for strings (not searchable); use CodeSystem designations + `$expand displayLanguage` for codes. (`/docs/fhir-datastore/multilingual-support`)

## Files, analytics, UI

- **Files go in Binary, referenced from an Attachment.** Never put bytes in `Attachment.data`. Reads rewrite Attachment URLs to 60-min presigned URLs so `<img>`/`<video>` work without auth headers and with range requests. Presigned URLs are unauthenticated, so limit where they go. (`/docs/fhir-datastore/binary-data`, `/docs/api/fhir/operations/binary-presigned-url`)
- **FHIR search serves per-patient lookups, not population aggregation.** Enterprise syncs history to Iceberg, which is shared to Snowflake or Redshift as `<type>_history` JSON rows. "Don't flatten at export"; "changing your mind costs a view definition instead of a re-export". The warehouse is a full PHI copy that access policies don't govern. (`/docs/analytics/redshift`, `/docs/analytics/snowflake`)
- **Enforce coding at write time with Bots so analytics work;** keep CDS event-driven. "A recommendation someone can accept or reject gets adopted faster than one that acts on its own." (`/docs/analytics`)
- **The React components power the Medplum App itself** (Mantine 7, MedplumProvider). The App is an admin console; most customers build custom UIs. `useSubscription` replaces polling over one shared WebSocket. (`/docs/react`, `/docs/tutorials/importing-sample-data`, `/docs/react/use-subscription`)
- **Learn FHIR by watching the App's network tab;** more than half of implementations use synthetic Synthea data. (`/blog/2021/12/06/learning-fhir-quickly`, `/blog/2021/12/13/synthea`)

## Key source articles
`/docs/fhir-basics` · `/blog/demystifying-fhir-systems` · `/blog/fhir-workflow-patterns-to-simplify-your-life` · `/docs/fhir-datastore/fhir-batch-requests` · `/docs/search/paginated-search` · `/blog/graphql-vs-rest` · `/blog/large-patient-charts` · `/docs/terminology/filtering-large-code-systems` · `/docs/fhir-datastore/binary-data` · `/docs/analytics/redshift`
