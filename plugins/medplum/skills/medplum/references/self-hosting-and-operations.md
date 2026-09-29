# Self-hosting, operations, upgrades, performance

What it takes to run Medplum yourself, and how Medplum runs its own fleet. The honest position: **hosted is cheaper and certified; self-host only when you must.** If you do, treat the database as sacred, pin versions, upgrade through every minor, and make observability non-negotiable.

## Should you self-host?

- **Self-hosting means owning upgrades** (a week to a month per release), Postgres/Redis sizing, and 24/7 on-call "without the builders' institutional knowledge". Medplum amortises its runbooks and incident history across the whole fleet. (`/docs/self-hosting/considerations`)
- **Certifications attach to Medplum's hosted operation, not the open-source code.** Self-host only for hard regulatory or contractual constraints, low-connectivity edge sites, or education. (`/docs/self-hosting/considerations`)
- **The trade in practice:** MediMind self-hosted for data sovereignty and accepted owning upgrades, patching and backups, taking releases only after a soak period. (`/blog/medimind-case-study`)
- **Targets:** AWS CDK is recommended (the hosted service's own deployment; Lambda runs bots and CloudFront signs presigned URLs). Alternatives: the Helm chart (cloud-agnostic), GCP/Azure via Terraform + Helm, the Ubuntu apt package, or Docker compose for local dev. From-scratch installs are for learning only. (`/docs/self-hosting`, `/docs/self-hosting/aws-advantages`, `/docs/self-hosting/install-on-kubernetes`, `/docs/self-hosting/install-on-ubuntu`)
- **Verify after install:** change the default super-admin password first, then run an invite (tests email) and a bot execution (tests Lambda), and check for AuditEvents. Bots are off by default. (`/blog/post-install-verification`)

## Versions and upgrades

- **Lockstep semver:** Docker, npm, Helm and Terraform carry identical tags. Majors ship yearly in Q4 (1 year Active + 1 year Maintenance), minors 2–3× a year, patches weekly. Security SLA: Critical ≤7d, High ≤14d. "ONC certification is only valid for 'Active' versions." (`/docs/compliance/versions`)
- **"You cannot skip minor versions."** Pre-deploy migrations take seconds and block startup; post-deploy migrations run hours or days in the background. A server that won't start means a failed pre-deploy migration; a slow one means post-deploy work is running. (`/docs/self-hosting/upgrading-server`)
- **Pin exact versions, never `:latest`.** The v4 upgrade loop hit auto-pullers whose old data needed a 3.3.0 stop first. (`/blog/v4-upgrade`, `/docs/self-hosting/enabling-rollbacks`)
- **Rollbacks:** set `disablePostDeployMigrations`, validate, then migrate. The rollback window closes when post-deploy migrations start. "A rollback you have never rehearsed is not a rollback plan." (`/docs/self-hosting/enabling-rollbacks`)
- **Majors track EOL dependencies.** v5 (Oct 2025): Node 22/24, Postgres 14–18, Redis 7, React 19, Express 5, ESM by default, Biome. v4 was "largely symbolic" with the API unchanged: "We prioritize stability and backwards compatibility." (`/blog/preparing-for-v5`, `/blog/v5-release`, `/blog/v4`)
- **Maturity labels:** Alpha may change or vanish (`@experimental`); Beta has a stable core and suits non-critical production; no label means GA. (`/docs/compliance/alpha-beta`)
- **Stay current on dependencies:** weekly Monday upgrade bot, Dependabot/CodeQL/SonarCloud/Snyk. `npm audit` is "sadly quite flawed". Never `--force` installs. (`/blog/dependency-warnings`, `/docs/contributing/package-json`)

## Running it well

- **"Treat the Database as Sacred":** manual schema or data changes break upgrades. Postgres majors take days of focused effort. (`/docs/self-hosting/best-practices`)
- **"Don't Run Hot":** target 40–50% CPU. "A system running hot turns small spikes into cascading failures." Test restores; snapshots alone aren't enough. (`/docs/self-hosting/best-practices`)
- **"Observability is non-negotiable."**
  - Node is single-threaded, so alert at `1/cores` CPU.
  - Keep the Postgres writer under 60–70%.
  - Alert on LB 504s.
  - Subscription backlog tracks DB load.
  - OTel is configured via env vars only.
  
  (`/docs/self-hosting/monitoring`, `/docs/self-hosting/opentelemetry`)
- **Sizing:** tasks at 4096 CPU/8192 MiB, and scale out rather than up, because write validation causes GC pressure. `rdsInstances: 2` for failover. Estimate DB size from per-patient annual resource counts: 1M patients ≈ 308 GB/year. (`/docs/self-hosting/aws-cdk-settings`, `/blog/estimating-rds`)
- **Zero-downtime DB work:** add a reader and RDS Proxy before parameter changes; resize readers first, then the writer. Hosted Postgres went 12→16 with ~1s of impact via self-managed blue/green: logical replication, temporary PgBouncer, a scripted switch run from a jumpbox in tmux, and `ANALYZE` afterwards. (`/docs/self-hosting/rds-parameters`, `/docs/self-hosting/upgrade-rds-database`, `/blog/zero-downtime-postgres-major-version-upgrade`)
- **DR:** because the app tier is stateless, recovery is a database problem: restore, re-provision via IaC, repoint DNS (~45–60 min). Drill it. (`/docs/self-hosting/disaster-recovery`)
- **Config gotchas:**
  - `MEDPLUM_` env vars override the file, and underscores matter (`MAX_CONNECTIONS`).
  - Presigned URLs need a real shared signing key off AWS.
  - Losing an SSE-C key means losing the data.
  - Audit events are off by default.
  
  (`/docs/self-hosting/setting-configuration`, `/docs/self-hosting/presigned-urls`, `/docs/self-hosting/server-config`)
- **GIN `fastupdate` causes 40001 serialization errors under bulk upserts.** Disable it via `$db-configure-indexes`; retries only mask the problem. (`/docs/self-hosting/gin-index-performance`)
- **Super admin "can cause unrepairable damage":** rebuild, reindex, purge, `$clone` (≤1000 per type). Project `$expunge` "is the equivalent of sudo rm -rf". (`/docs/self-hosting/super-admin-guide`, `/docs/self-hosting/super-admin-cli`)

## Limits and performance

- **Two rate-limit layers:** per-IP limits (login 5/min), and a weighted per-user FHIR quota (read 1, search 20, history 10, write 100) with a project total of 10× the user quota. Read the `RateLimit` header. Hitting the auth limit usually means a client authenticates on every request. (`/docs/rate-limits`, `/docs/self-hosting/server-config`, `/docs/api/fhir/operations/project-rate-limits`)
- **2026 benchmark (15 Fargate tasks, r6gd.4xlarge):**
  - Authenticated FHIR reads: 20.8k req/s.
  - Create + read + search: 6.7k req/s (p95 403ms).
  - Zero HTTP failures.
  
  The database is usually the eventual bottleneck. Watch p95/p99. (`/blog/medplum-performance-test-2026`, `/docs/self-hosting/load-testing`)
- **Search internals:** one table per type with typed columns per search parameter. Security filters are injected into SQL, never post-filtered, so paging stays correct. Terminology uses recursive CTEs and trigram indexes. (`/docs/contributing/search-architecture`, `/docs/contributing/terminology-architecture`)

## Contributing

- **Link a PR to an `open-to-community` issue** or it is auto-closed. Every commit needs a DCO sign-off (`git commit -s`), which is lighter than a CLA. "Medplum strongly believes in the importance of testing." Hosted production deploys on every merge to main. (`/docs/contributing`, `/docs/contributing/dco`, `/docs/contributing/testing`, `/docs/contributing/publishing-npm-packages`)

## Key source articles
`/docs/self-hosting/considerations` · `/docs/self-hosting/best-practices` · `/docs/compliance/versions` · `/docs/self-hosting/upgrading-server` · `/docs/self-hosting/enabling-rollbacks` · `/docs/self-hosting/monitoring` · `/blog/zero-downtime-postgres-major-version-upgrade` · `/docs/rate-limits` · `/blog/medplum-performance-test-2026`
