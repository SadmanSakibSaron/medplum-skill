# Bots, subscriptions, automation

Medplum's answer to "integrations are the product". A Bot is a single-file, Lambda-like function. It is triggered by Subscriptions (FHIR-search webhooks), `$execute`, cron, or custom `$operations`. Bots replace the custom servers, credentials and glue code a digital health back end usually needs.

## What a Bot is

- **"Bots drive many of the major integrations that you see in Medplum."** They are enabled per project (`features: bots`, paid on hosted). (`/docs/bots`)
- **One file, `handler(medplum, event)`.** It runs sandboxed with a MedplumClient, versioned with history like any resource, with no separate HTTP server or credentials. `event` carries input, contentType, secrets, traceId, requester and headers. (`/docs/bots/bot-basics`)
- **Typical jobs:** default values, custom validation, welcome messages, lab notifications, expanding a QuestionnaireResponse into Patient + ServiceRequest, generating PDFs (pdfmake → Binary → attachment), and pushing files over multipart or SFTP. (`/docs/bots/bot-basics`, `/docs/bots/bot-for-questionnaire-response`, `/docs/bots/creating-a-pdf`, `/docs/bots/file-uploads`)
- **Questionnaire + Bot is the fully controlled workflow pattern.** Supply semantic `linkId`s yourself and read them with `getQuestionnaireAnswers`. (`/docs/bots/bot-for-questionnaire-response`)
- **Bots are principals.** By default a Bot can read and write everything, so **apply an AccessPolicy**. `runAsUser: true` makes it use the caller's policy and attributes history to that user. (`/docs/bots/bot-basics`, `/docs/bots/bot-run-as-user`, `/docs/api/project-admin/bot`)
- **Secrets live per project** (`event.secrets`), so the same code uses different keys in staging and prod. (`/docs/bots/bot-secrets`)

## Triggers

- **Subscriptions are webhooks keyed on a FHIR search filter.** They replace the fax-back lab loop: subscribe to ServiceRequest and follow it through to the DiagnosticReport. Test the endpoint first (e.g. Pipedream). (`/docs/subscriptions/publish-and-subscribe`)
- **Never subscribe to AuditEvent.** Each notification creates an AuditEvent, so it spirals. (`/docs/bots/bot-basics`, `/docs/subscriptions/publish-and-subscribe`)
- **Retries are opt-in and matter.** The default of 4 attempts survives only ~2 minutes of outage; 12 ≈ 11h, 18 ≈ 2.5 days, then the event is dropped. **Bot-endpoint subscriptions execute once and don't retry.** Delivery order isn't guaranteed. Recover with `$resend`. (`/docs/subscriptions/subscription-extensions`, `/docs/bots/bot-for-questionnaire-response`, `/docs/api/fhir/operations/resend`)
- **Extensions:** interaction filter (create/update/delete), HMAC signing (`x-signature`), custom success codes, and FHIRPath `%previous`/`%current` criteria. On create `%previous` is empty, so write `%previous.exists() implies …`. Deletes arrive as an empty body plus `X-Medplum-Deleted-Resource`. (`/docs/subscriptions/subscription-extensions`)
- **`$execute`** by id, or by identifier across projects. Content-Type shapes `event.input` (JSON, text, form, FHIR, HL7 `x-application/hl7-v2+er7`). `Prefer: respond-async` returns an AsyncJob. (`/docs/api/fhir/operations/bot-execute`)
- **Cron:** a cron string on the Bot, or Cron resources carrying `onBehalfOf` identity and parameters. "A Bot is code, not authority": a shared Bot from a linked project runs with the customer's policy, data and audit. Invalid schedules are rejected on write "rather than as a job that silently never fires". (`/docs/bots/bot-cron-job`)
- **Server-scoped subscriptions (self-hosted)** fire across every project, giving project-per-tenant deployments one central flow. (`/docs/subscriptions/server-scoped-subscriptions`)

## Custom operations

- **`MedicationRequest/$calculate-dose` beats `Bot/dose-calculator/$execute`,** because the endpoint says what it does. The recipe is Bot + OperationDefinition + the `operationDefinition-implementation` extension. (`/blog/custom-fhir-operations`, `/docs/bots/custom-fhir-operations`)
- **"Syntactic sugar on the existing Bot execute framework",** so they inherit its security: callers must be able to read the Bot. Postel's Law: take liberal inputs (the raw body) and return strict outputs per the OperationDefinition. Return OperationOutcome for controlled errors. (`/blog/custom-fhir-operations`, `/docs/bots/custom-fhir-operations`)

## Webhooks and HL7 in

- **Consume SaaS webhooks (Stripe, Okta) through `$execute`** with a minimally privileged ClientApplication (read-only Bot access). Verify signatures with a secret stored in bot secrets. Unauthenticated `/webhook/…` needs `publicWebhook: true` plus a policy: "Do not trust external input". (`/docs/bots/consuming-webhooks`)
- **HL7v2 over HTTPS to a Bot:** convert the message, upsert the Patient by MRN, return an ACK, and let Subscriptions fan out downstream. Real HL7 logic grows complex fast, so use the typed toolkit. (`/docs/bots/hl7-into-fhir`)
- **Awell pattern:** a Subscription starts a care flow, which reads FHIR and writes actions back, so clinical teams own flows that used to live in Google Docs. (`/blog/awell-health-medplum`)

## Bots as software

- **Source control, unit tests, CI deploys, and separate staging/prod projects built from the same source.** "Your `.env` file should never be checked into source control." Use `npx medplum bot deploy '*'` driven by `medplum.config.json`. Saving is not deploying. (`/docs/bots/bots-in-production`, `/docs/bots/bot-basics`)
- **One self-contained JS file per bot, by design.** Use thin handlers in `src/bots` and shared code in `src/common`, bundled with esbuild. Mark as external only packages in the Lambda layer, otherwise `require()` fails at runtime. Lambda bots are capped at 50MB compressed. (`/docs/bots/bot-code-organization`, `/docs/bots/bot-lambda-layer`)
- **Test with `MockClient`:** a fresh client per test, seed, invoke, assert. Index the full schema, and test idempotency by running twice and asserting no writes. Known gaps: chained search, includes, most $operations, and terminology. (`/docs/bots/unit-testing-bots`)
- **Monitor via AuditEvents** (outcome 0/4/8/12). Tune `auditEventTrigger` (always, on-error, on-output) and set `auditEventDestination: log` for high volume. Missing events usually means log-only is set. (`/docs/bots/bots-in-production`, `/docs/bots/monitoring-bots`)

## Runtimes (self-hosted)

- **Lambda by default.** VM Context runs in the server process: fast but "`node:vm` is not a security mechanism", so use it only for trusted code. Alternatives are Fission on Kubernetes, or externally managed Lambdas (`botCustomFunctionsEnabled`). Self-hosters must publish bot-layer updates themselves. (`/docs/bots/running-bots-locally`, `/docs/bots/running-bots-on-fission`, `/docs/bots/external-function`, `/docs/bots/bot-lambda-layer`)
- **CLI** (Node 22+): REST verbs, bot save/deploy, bulk export/import, and `get --as-transaction` for fixtures. (`/docs/cli`)

## Key source articles
`/docs/bots/bot-basics` · `/docs/bots/bots-in-production` · `/docs/subscriptions/subscription-extensions` · `/blog/custom-fhir-operations` · `/docs/bots/bot-cron-job` · `/docs/bots/consuming-webhooks` · `/docs/bots/unit-testing-bots` · `/docs/bots/bot-code-organization` · `/docs/subscriptions/publish-and-subscribe`
