# Data migration and patient identity (EMPI)

Two linked disciplines: moving live operations onto Medplum without interruption, and keeping one patient as one record (deduplication/EMPI). The stance in both is the same. The hard problems are decisions about meaning, ownership and identity, not format conversion. Humans stay in the loop wherever a wrong call harms a patient.

## Migration: decisions before code

- **"The hardest problems are usually decisions about meaning, ownership, and cutover rather than converting file formats."** (`/docs/migration`)
- **"'All data' is not an actionable scope."** Each domain needs a boundary, an owner, a target and acceptance criteria. Size scope with eligibility queries on representative data, not the size of the legacy DB. (`/docs/decision-guides/data-migration`, `/docs/migration/migration-planning`)
- **One write authority per domain per phase.** "Most cutover problems are ownership problems rather than copy problems." Without it, reconciliation can't reach a conclusion. (`/docs/migration/migration-planning`, `/docs/decision-guides/data-migration`)
- **One-way door:** loading clinical data before resolving identity spreads references across duplicate patients. Resolve identity first. (`/docs/decision-guides/data-migration`)
- **Strategies:** big bang has the highest blast radius; parallel run has the lowest rollback risk but the highest conflict risk. Don't start production while any identity, meaning, authority or acceptance decision is unowned. (`/docs/migration/migration-planning`, `/docs/decision-guides/data-migration`)

## Mapping and conversion

- **Treat mappings as reviewed, versioned specifications** (a mapping register), not code internals. Never modify an approved release in place. "A source schema describes what a field can contain. Source profiling describes what it actually contains." (`/docs/migration/mapping-governance`)
- **"A FHIR resource can be structurally valid while still misrepresenting the source meaning."** `$validate` checks structure, not meaning. (`/docs/migration/convert-to-fhir`, `/docs/migration/mapping-governance`)
- **Preserve source primary keys as identifiers, with a distinct system URI per source.** "A source identifier answers 'which source record is this?'", not which person. (`/docs/decision-guides/data-migration`, `/docs/migration/convert-to-fhir`)
- **Don't invent codes:** migrate local codes now and enrich with verified standard codes later. Never infer a code from display text. A legacy `is_active=false` has no single FHIR equivalent, so route it for review. (`/docs/migration/convert-to-fhir`)
- **"Do Not Use Extensions as a Catch-All."** Missing stays missing: don't invent defaults, and quarantine invalid records rather than coercing them. (`/docs/migration/mapping-governance`)
- **Load order:** orgs/locations → practitioners → patients/coverage → encounters → current clinical state → history → documents. "Do not drop the reference merely to make the load succeed." (`/docs/migration/migration-sequence`)

## Pipelines

- **Conditional updates for idempotency, async batches for volume, transactions only for small all-or-nothing groups** (with the project flag set). Everything must be rerunnable from a manifest. Conditional-update reruns are safe only while the source stays authoritative. (`/docs/migration/migration-pipelines`, `/docs/decision-guides/data-migration`)
- **Use a dedicated ClientApplication with Basic auth,** so there are no expiring tokens during long backfills. `medplum bulk import` assigns new IDs and isn't idempotent, so use it only for simple loads. Patient count isn't a throughput estimate; benchmark the real resource mix. (`/docs/migration/migration-pipelines`)
- **Backfills fire Subscriptions and Bots.** Decide per workflow whether to keep, narrow or disable-and-replay them; disabled subscriptions don't catch up. (`/docs/migration/adoption-strategy`)

## Validation, acceptance, cutover

- **"HTTP success is not migration success."** Inspect every `entry.response`, and remember a 202 means only queued. Every in-scope unit needs an explained outcome: eligible = successful + rejected + skipped + quarantined. (`/docs/migration/validation-and-reconciliation`)
- **"An identifier match is not proof of identity."** Keep absent, zero and unknown distinct. Preserve precision and negation, and don't make history look current. (`/docs/migration/validation-and-reconciliation`)
- **Review whole charts;** field checks miss cross-resource errors. Open the documents. Test with production roles, not migration credentials. "Accepting an exception is a risk decision, not a way to close a ticket." (`/docs/migration/testing-and-acceptance`)
- **Phased adoption:** dual write → backfill → reads behind flags → front-end writes → deprecate. Avoid bidirectional sync ("cannot be resolved safely with last-write-wins"). "A rollback plan that discards post-cutover clinical writes is not safe." Retire legacy on completion, not on a deadline. (`/docs/migration/adoption-strategy`)
- **"A small pilot does not prove production throughput. A full-volume load does not prove clinical meaning."** And: "Technically correct data is not a successful migration if users cannot safely perform their work after go-live." (`/docs/decision-guides/data-migration`)
- **Proof it works:** MediMind moved ~1.1M patients and cut over 15 analyzer lines in three weeks, with analyzers dual-reporting and rollback in under a minute. (`/blog/medimind-case-study`)

## Patient identity / EMPI

- **Pipeline: ingestion → matching → merging,** always with audit of why records matched, why they merged, and who merged them. (`/docs/fhir-datastore/patient-deduplication`)
- **Choose the architecture by how easily a wrong merge can be undone.** Duplication can be rare, frequent or bimodal. Merges in patient-login apps expose data to a user, so they differ from merges for population health. (`/docs/fhir-datastore/patient-deduplication/architecture-overview`)
- **Start with batch** (easier to iterate, and 1M patients fit in <10GB), **then add incremental Bots** on Patient create. Keep identifiers from every source (payers, DoseSpot, even Stripe). Dedup rules are "policy as code": tested and source-controlled, because "a false merge can cause treatment errors". (`/docs/fhir-datastore/patient-deduplication/ingestion`, `/blog/patient-deduplication`)
- **`Patient/$match` implements the CMS framework's 26 combinations.** Discovery mode returns graded candidates; disclosure mode releases only on exactly one `certain` match (the uniqueness gate). The score is "intentionally not a probability". Deliberately excluded, "so behavior is transparent and reproducible": nickname tables, placeholder suppression, Soundex, and gender as a factor. (`/docs/api/fhir/operations/patient-match`)
- **Candidate = RiskAssessment** (method = rule, subject = source, basis = target) plus a review Task. Do-not-match lists are Lists. 1970-01-01 is a common placeholder DOB. (`/blog/empi-implementation`, `/docs/fhir-datastore/patient-deduplication/matching`)
- **Merge:** only the master stays active, with `link` replaces/replaced-by in both directions. Use independent masters (easy unmerge) or record promotion. Rewrite clinical references to the master for care apps; keep them on the source record for HIE provenance. "Every dedup pipeline needs a human review process." Set Run-as-user on merge Bots. (`/docs/fhir-datastore/patient-deduplication/merging`, `/blog/empi-implementation`)
- **Why it matters for integrations:** hospital IT fears "that an integration will introduce duplicates into their system" (Titan). (`/blog/titan-case-study`)

## Key source articles
`/docs/decision-guides/data-migration` · `/docs/migration/mapping-governance` · `/docs/migration/adoption-strategy` · `/docs/migration/validation-and-reconciliation` · `/docs/migration/testing-and-acceptance` · `/docs/api/fhir/operations/patient-match` · `/docs/fhir-datastore/patient-deduplication/merging` · `/blog/empi-implementation` · `/blog/patient-deduplication`
