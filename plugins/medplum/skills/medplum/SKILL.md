---
name: medplum
description: The complete Medplum philosophy and platform knowledge, distilled from all 493 pieces (398 docs + 95 blog posts) on medplum.com. Covers the headless-EHR / API-first thesis, FHIR data modeling and search, access control and multi-tenancy, Bots and Subscriptions, clinical workflows (tasks, charting, intake, scheduling), messaging, integrations (HL7/Agent, DICOM, labs, eRx), billing/RCM and prior auth, compliance (ONC, SOC2, HITRUST), self-hosting and upgrades, migration/EMPI, and AI/MCP. Use when the user asks how to build a healthcare app on Medplum or FHIR, how Medplum models or secures something, whether to self-host, how to integrate or migrate, or what Medplum / the Medplum team thinks.
---

# Medplum

This skill encodes the documentation and blog published at [medplum.com](https://www.medplum.com/docs) by **Medplum**, the open-source (Apache 2.0), FHIR-native "headless EHR" company (YC S22). It is distilled from all 493 pieces, 398 docs pages and 95 blog posts, from "Introducing Medplum" (2021) through "Awell Panels: Worklists That Update Themselves" (Sept 2026).

## What this is for

Use it to answer questions the way the Medplum team would:
- how to model a clinical or operational workflow in FHIR;
- how to secure and tenant data;
- how to automate with Bots;
- how to connect legacy systems;
- how to bill;
- how to run, upgrade or migrate onto Medplum;
- why the platform is built the way it is.

For a broad question, answer from the **core philosophy** below plus the relevant reference file. For a specific page or term, open the matching reference file and cite by path, e.g. (`/docs/bots/bot-basics`) → https://www.medplum.com/docs/bots/bot-basics. `references/article-index.md` lists all 493 pieces with one-line theses. `references/glossary.md` defines the coined terms and core platform nouns. The raw corpus is bundled at `corpus/` in this skill's base directory (`docs__…md` / `blog__…md`); grep it when a detail is missing from the references.

## The one-sentence thesis

> **Healthcare builders shouldn't face "the terrible choice" between building everything and bending a rigid EHR: give them a headless, standards-native (FHIR) platform where the API is the product, integrations are the product, and compliance comes built in, so they can spend their time on the experience and the operations that actually differentiate care.**

Everything else is a corollary of this.

## The core philosophy (the load-bearing ideas)

1. **The terrible choice.** Healthcare apps are "complex, rigid and hideous" because of the domain. Builders either roll their own infrastructure or fight an off-the-shelf EHR by "stuffing data where it's not meant to be". Medplum exists to remove that choice. (`/blog/medplum-mitre-talk`, `references/philosophy.md`)
2. **Headless; the API is the product.** There is no mandatory UI. The App is a bare-bones admin console; customers build their own experience from the SDK and React components. "Integrations are the product" too: Bots work like lambdas, with no DevOps. (`/blog/medplum-mitre-talk`, `/docs/tutorials/medplum-hello-world`)
3. **Please just use FHIR.** Store data *as* FHIR, not `create table patient`. The spec anticipates complexity that otherwise forces rewrites, prevents lock-in, and "Speed isn't a property of the data standard" (a 131k-resource chart loads in about 5s). (`/docs/fhir-basics`, `/blog/large-patient-charts`, `references/fhir-data-and-api.md`)
4. **Composability through standards + open source + API-first.** Use OpenID, SCIM, FHIR, UMLS, OpenAPI and SMART so a stack can evolve piece by piece. Partial APIs leave "a person's job to move data from one tool to another". (`/blog/composability-medplum`)
5. **Open source for trust, not growth.** "EHRs have no hobbyist use case." Apache 2.0 because auditors understand it. Security must be proven (OpenSSF gold). The business is hosting. Don't fork: forking is "technical debt in disguise". (`/blog/yc-oss-faq`, `/blog/openssf`, `/blog/so-youre-thinking-about-forking`)
6. **Simple access model, enforced in the platform.** Project → ProjectMembership → AccessPolicy. Parameterised policies act as templates, tenants are labelled with `$set-accounts`, and "stacking is additive only". Policies are strong enough to expose the FHIR API directly to partners. (`/docs/access/access-policies`, `/docs/decision-guides/access-control`, `references/access-and-identity.md`)
7. **Automation is code you own.** Subscriptions + Bots + custom `$operations` replace glue servers. Treat Bots as software: tests, CI, idempotency, staging and prod. Never subscribe to AuditEvent. Retries are opt-in. (`/docs/bots/bots-in-production`, `/docs/subscriptions/subscription-extensions`, `references/bots-and-automation.md`)
8. **Digital health is an operations game.** Four foundations: a service menu, top-of-license care, fifty-state workflows, and async care. Tasks are the backbone, with `focus` always set. The operational store is kept separate from the analytical store. (`/blog/digital-health-operations`, `/docs/careplans/tasks`, `/blog/awell-panels-case-study`)
9. **Forms in, structured data out.** Questionnaires are the schema. Always parse responses into resources (SDC `$extract` or Bots). The linkId is a contract; upsert by natural keys; a declined consent is a *rejected* Consent. (`/docs/decision-guides/intake`, `/docs/consent`, `references/clinical-workflows.md`)
10. **Meet legacy where it is, securely.** Prefer FHIR or REST, but a thin on-prem Agent bridges HL7v2, ASTM and DICOM over outbound WebSockets. "A 'slow' HL7 interface is almost never slow because of the network." (`/docs/integration/hl7-interfacing`, `/docs/agent/high-throughput-hl7`, `references/interoperability.md`)
11. **Compliance is a product you inherit.** ONC (incl. (b)(10), (g)(10)), SOC2 Type II, HIPAA and HITRUST e1 attach to hosted operation. "Compliance is a shared responsibility." Follow regulation over novelty: stay on R4/US Core, skip R5, plan for R6. (`/docs/compliance`, `/docs/compliance/versions`, `references/compliance-and-security.md`)
12. **Hosted by default; self-host deliberately.** Self-hosting means owning upgrades, on-call and Postgres. Pin versions, never skip minors, "Treat the Database as Sacred", and "A rollback you have never rehearsed is not a rollback plan." (`/docs/self-hosting/considerations`, `/docs/self-hosting/enabling-rollbacks`, `references/self-hosting-and-operations.md`)
13. **Meaning, ownership and identity before format.** Migrations fail on decisions, not conversions. Resolve identity before loading clinical data. Keep humans in the loop on merges. "HTTP success is not migration success." (`/docs/migration`, `/docs/decision-guides/data-migration`, `references/migration-and-identity.md`)
14. **AI is an infrastructure problem.** "AI can only automate what it can access." LLMs already know FHIR, so one `fhir-request` MCP tool is enough. Agents are governed like clinicians: "can suggest, but not act". (`/docs/ai`, `/docs/ai/mcp`, `references/ai.md`)

## The Medplum method, end to end

1. **Scope with a decision guide:** who, what data, which integrations, and which one-way doors. (`/docs/decision-guides`)
2. **Model in FHIR:** workflow patterns, systems and identifiers, US Core profiles, and scoped ValueSets. (`/blog/fhir-workflow-patterns-to-simplify-your-life`)
3. **Set up Projects** (dev/staging/prod), **memberships and parameterised AccessPolicies.** (`/docs/access/projects`)
4. **Capture with Questionnaires, and extract** into resources idempotently. (`/docs/questionnaires/parsing-questionnaire-responses`)
5. **Drive work with Tasks,** and automate with Subscriptions + Bots deployed through CI. (`/docs/careplans/tasks`)
6. **Build a custom UI** on the SDK and React components, or start from Medplum Provider. (`/docs/provider`)
7. **Integrate** through standards first, then the Agent or vendor bots. (`/docs/integration`)
8. **Bill** from Coverage → ChargeItem → Claim, through a clearinghouse or RCM partner. (`/docs/decision-guides/rcm-billing`)
9. **Migrate and cut over** in phases with one write authority per domain; operate on hosted, or self-host with discipline. (`/docs/migration/adoption-strategy`)

## Reference files

- **`references/philosophy.md`**: the terrible choice, headless/UDHP, open source for trust, not forking, the operations game, operational vs analytical store, FDE, R4 vs R5, the roadmap trajectory.
- **`references/fhir-data-and-api.md`**: systems/identifiers, extensions, profiles, CRUD/history, batch/transaction limits, search/pagination, GraphQL vs REST, terminology, binaries, warehouse exports.
- **`references/access-and-identity.md`**: the Project/Membership/AccessPolicy model, field/write constraints, Binary securityContext, multi-tenancy, IdPs, OAuth/SMART, MFA, on-behalf-of.
- **`references/bots-and-automation.md`**: Bots, triggers, subscription retries/extensions, custom operations, webhooks, HL7 in, cron, testing, runtimes.
- **`references/clinical-workflows.md`**: Tasks, PlanDefinition/CarePlan, charting/SOAP/signing, intake, consent, referrals, labs, meds, scheduling, Medplum Provider.
- **`references/communications.md`**: thread model, routing via Task, read receipts, drafts/edits, SMS/eFax bridges, async encounters.
- **`references/interoperability.md`**: (g)(10), Agent/HL7 throughput, ASTM, SMART, Bulk, C-CDA, CDS Hooks, FHIRcast, TEFCA, DICOM, Health Gorilla, eRx.
- **`references/billing-and-rcm.md`**: clearinghouse vs RCM partner, Coverage, eligibility, pricing, claims, remittance, denials, prior auth (HTI-4).
- **`references/compliance-and-security.md`**: inherited certifications, ONC, HTI-4, CLIA/CAP, CFR 11, redundancy vs backups, SLA, OpenSSF, security reports.
- **`references/self-hosting-and-operations.md`**: whether to self-host, version policy, upgrades/rollbacks, monitoring, sizing, Postgres upgrades, rate limits, benchmarks, contributing.
- **`references/migration-and-identity.md`**: migration scope/authority, mapping governance, pipelines, reconciliation, cutover, EMPI/`$match`/merging.
- **`references/ai.md`**: the AI-as-infrastructure thesis, guardrails, `$ai`, Spaces, MCP, coding assistants, field patterns.
- **`references/glossary.md`**: 52 coined terms and core platform nouns, each with its source.
- **`references/article-index.md`**: all 493 pieces with path and one-line thesis, grouped by theme.

## How to answer

- **Write like Medplum's docs:**
  - Direct and opinionated, e.g. "Never subscribe to AuditEvent" and "Pin exact versions, never `:latest`".
  - Name the one-way doors and the gotchas.
  - Give the FHIR resource, field and operation by exact name.
  - Recommend a default, then the trade-off.
- **Prefer the concrete:**
  - Exact limits: `_count` max 1000, transactions ≤50 updates, a quota weight of 100 per write.
  - Case-study numbers: Awell 2 → 180 hospitals, MediMind at 1.1M patients.
  - Named patterns: retract and correct, recent-first, logical channels.
- **Always cite** the source path so the user can read the original at https://www.medplum.com + path.
- If the user asks about a specific page, check `references/article-index.md` first, then the theme file, then grep the corpus.
- Present contested and changed views with dates: R5 was planned in 2023 and paused in 2024; scheduling moved from Alpha to Beta; v4 → v5 breaking changes.
- If the corpus doesn't cover the question (e.g. pricing tiers, another vendor's internals), say so. Don't invent Medplum behaviour; the docs are the source of truth.

## Scope

This skill covers only published medplum.com docs and blog content as of September 2026. It does not include the GitHub source code, issues, Discord, or API reference pages generated from code. For exact current behaviour of an API, verify against the live docs or the code. Alpha/Beta features (scheduling, DICOM, ePA, ASTM) may have changed since.
