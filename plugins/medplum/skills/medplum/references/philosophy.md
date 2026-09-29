# Philosophy and strategy

Why Medplum exists and how it thinks about building healthcare software: the "terrible choice" facing builders, the headless / API-first answer, open source as a trust mechanism, standards as the route to composability, and operations as the real game in digital health. Its roadmaps, monthly updates and case studies show these ideas applied.

## The problem: the terrible choice

- **Healthcare apps are "complex, rigid and hideous" because of the domain, not lazy developers.** Care is complicated and regulated, and "any app built for healthcare by default serves many stakeholders". (`/blog/2021/11/04/introducing-medplum`, `/blog/medplum-in-yc`)
- **The terrible choice.** Either build a tailored experience and roll all your own infrastructure (certification, interop, workflow), or fight a rigid off-the-shelf product and end up "stuffing data where it's not meant to be". Fintech and insurtech have a better story. (`/blog/medplum-mitre-talk`, `/blog/yc-oss-faq`)
- **Minimal apps "spiral in complexity when the app makes contact with the healthcare establishment."** Other pain points: integration and workflow are harder than in fintech, poor data quality is rampant, and healthcare-savvy engineers are scarce. (`/blog/yc-oss-faq`, `/blog/medplum-mitre-talk`)
- **Monolithic EHRs aren't badly designed; needs expanded.** EHR limits are "an acknowledgment of today's expanded healthcare needs". The next phase is a platform (UDHP), but one-size-fits-all platforms fail the "last mile" of tailoring. (`/blog/ehr-vs-udhp`)

## The answer: a headless EHR where the API is the product

- **"Medplum is a headless EHR."** Data, auth, compliance and automation, with no fixed UI; builders supply the experience. (`/docs`, `/blog/ehr-vs-udhp`)
- **"The API is the product."** The UIs look bare-bones on purpose. A SaaS app with an API is not the same as a headless, devtools-centric product; full-stack SaaS is "brittle and slow" with interop as an afterthought. (`/blog/medplum-mitre-talk`)
- **"Please just use FHIR" instead of `create table patient`.** Native FHIR storage plus open source prevents data lock-in, "the rot that is so common in the industry". (`/blog/medplum-mitre-talk`, `/blog/2021/11/04/introducing-medplum`)
- **"Integrations are the product."** Bots behave like lambdas (e.g. sync a new patient to a legacy EHR) with no DevOps. A typical app is a static JS site embedding the SDK, with no backend. (`/blog/medplum-mitre-talk`)
- **Composability through standards, open source and API-first design.** OpenID, SCIM, FHIR, UMLS and OpenAPI let a stack evolve; partial APIs force humans to copy data between tools ("some groups where it is a person's job to move data from one tool to another"). (`/blog/composability-medplum`)

## Open source: for trust, not growth

- **Open source is about trust.** Healthcare devs are jaded by undocumented black boxes. At their previous company, MedXT, customers kept asking for source in escrow. Customers trust Medplum with their primary health datastore. (`/blog/yc-oss-faq`, `/blog/podcast-appearances`)
- **"Electronic Health Records have no 'hobbyist' use case."** Mindshare comes from customers in production early (before 20 stars) and from certifications (SOC2, HIPAA, ONC, CLIA/CAP), not launch weeks or memes. (`/blog/yc-oss-faq`)
- **Apache 2.0 because auditors understand it;** "there are code scanners everywhere in an audit". (`/blog/yc-oss-faq`)
- **Business model = hosting,** GitLab-style. Developers build first, then the organisation chooses self-host or cloud. Unusually, Medplum started hosted multi-tenant, and its first paid customer was hosted. (`/blog/medplum-mitre-talk`, `/blog/yc-oss-faq`)
- **Don't fork: "technical debt in disguise."** Contribute upstream, add a thin BFF/extension layer, or sponsor roadmap work for "90% of the control with 10% of the cost". Forkers "merge back within months". Fork only for abandonment, licence retreat, or an irreconcilable legal clash. (`/blog/so-youre-thinking-about-forking`)
- **Tests, CI/CD and docs are treated as products;** customers search the repo for examples. (`/blog/medplum-mitre-talk`)

## Digital health is an operations game

- **Four foundations:** a defined service menu, top-of-license care, fifty-state workflows, and async/hybrid care. Traditional EHRs were built "within the four walls of a single site". (`/blog/digital-health-operations`)
- **The service menu is your codes** (ICD-10/CPT, LOINC, RxNorm). A narrow scope enables purpose-built UIs, which means less data entry and less burnout. (`/blog/digital-health-operations`)
- **Top of license needs one Task system** with queues, credential routing and escalation; fifty-state work needs licences in the data model. Visit-centric EHRs fail async care: "What is considered a 'visit' in the age of day-long text chains?" (`/blog/digital-health-operations`)
- **Operational store ≠ analytical store.** Building an operational product on an analytical foundation leads to "a cache and a second database". Awell deleted its cached FHIR copy (three sync paths gone) and kept only a projection, scaling from 2 to 180 hospitals in nine months. (`/blog/awell-panels-case-study`)
- **Forward-deployed engineering, no PMs.** Code got cheaper with AI, so the bottleneck moved to integration and adoption, which is "irreducibly human". FDEs own scoping through support and set the roadmap; values pair empathy with expertise, drive, and rigor. (`/blog/fde`)

## Standards alignment over novelty

- **Follow US regulation, not the newest spec.** In 2023 the plan was to support R5 alongside R4 (`/blog/fhir-r5`). **Later (Jan 2024):** US Core was set to jump to R6, so R5 work paused and Medplum stays on R4 with US Core/USCDI. The 2025 version policy confirms R4 through 5.x and R6 planned for 6.x (`/docs/compliance/versions`).

## Trajectory (roadmaps and updates)

- **2022 trust** (open source, SOC2/HIPAA, CLIA/CAP) → **2023 connected** (ONC, Labcorp/Epic, Agent, 99.999% uptime) → **2024 community** (47 new contributors) → **2025 v5**, MCP, custom operations, PlumCon → **2026** HITRUST, HTI-4/prior auth, sharding, scheduling, DICOM, RCM. (`/blog/2022-year-in-review-medplum`, `/blog/2023-year-in-review-medplum`, `/blog/2024-year-in-review-medplum`, `/blog/2025-year-in-review-medplum`, `/blog/2026-roadmap`)
- **Case studies show the range:** a 2-engineer seed team on Retool + Medplum (`/blog/chamber-cardio-case-study`), a pediatric EHR with a project per customer (`/blog/develo-case-study`), and a Raspberry Pi 4 running the whole stack at ~1–1.5W (`/blog/medplum-for-nasa-and-trish`).

## Key source articles
`/blog/medplum-mitre-talk` · `/blog/yc-oss-faq` · `/blog/composability-medplum` · `/blog/digital-health-operations` · `/blog/ehr-vs-udhp` · `/blog/so-youre-thinking-about-forking` · `/blog/awell-panels-case-study` · `/blog/fde` · `/blog/2021/11/04/introducing-medplum` · `/blog/fhir-r5`
