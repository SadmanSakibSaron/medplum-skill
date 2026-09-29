# Billing, RCM, prior authorization

Medplum's billing stance: **Medplum owns the front half of the revenue cycle** (coverage, charge capture, claim assembly) as FHIR financial resources. The back half (scrubbing, submission, remittance, denials) goes either to a clearinghouse you drive (Stedi) or to an RCM partner (Candid). One Claim stays the source of truth, so the back end remains swappable.

## Strategy

- **"A clearinghouse moves your claims; an RCM platform runs your revenue cycle."** With a clearinghouse you get full control but build the back office yourself. An RCM partner brings a payer-rules engine and billers, at a recurring fee, with AR on their platform. Other options: documents only (superbills) or an external PM system. (`/docs/decision-guides/rcm-billing`)
- **One-way door: the choice of submission back end** (837P, partner API or CMS-1500). It changes formats, the response pipeline and payer identifiers. (`/docs/decision-guides/rcm-billing`)
- **Stedi = "you own the cycle" clearinghouse path; Candid = RCM partner path.** (`/docs/integration/stedi`, `/docs/integration/candid`)
- **The general pattern:** create financial resources, code them (CPT, LOINC, SNOMED), and sync to billing with Subscriptions and Bots; e.g. a finalised DiagnosticReport auto-sends to billing. Stripe and Candid are the sample integrations. (`/docs/billing`)

## Coverage and eligibility

- **Coverage is the critical resource.** Model subscriber vs beneficiary with a relationship, a member ID (required by US Core), SOPT `type`, `payor` as an Organization (so you can query patients by payer), class codes, and `order` for the coverage stack. **Create self-pay Coverage too.** The interim recommendation for copay cards is an extra coverage with a negative `costToBeneficiary`. (`/docs/billing/patient-insurance`)
- **Eligibility answers three layered questions:** is the policy active, does it cover general visits (X12 service type 30), and does it cover a specific service type. Ask about the service type, not the specific service. (`/docs/billing/insurance-eligibility-checks`)
- **`outcome` is about processing, not eligibility;** read `insurance.inforce` and `insurance.item` benefits. Mark the checked coverage with `insurance.focal`. Clearinghouses speak X12 270/271, not FHIR, so Bots convert. (`/docs/billing/insurance-eligibility-checks`)
- **Service type codes are a request, not a guarantee:** payers often answer with general benefits. The raw 271 is kept as a DocumentReference. (`/docs/integration/stedi/insurance-eligibility/eligibility-checks`)
- **Check before the encounter,** while you can still fix lapsed coverage or collect the copay. Payer identifiers differ between eligibility (Stedi network ID) and claims (Candid UUID / CMS / CHC). (`/docs/integration/candid/eligibility-check`, `/docs/integration/candid/claim-submission`)

## Charges and claims

- **Coding is "the largest source of denials", so treat it as first-class.** Link each ChargeItem to its Encounter and Account at creation. The Account anchors ChargeItem, Claim, PaymentReconciliation and Invoice; skip it and you lose in-Medplum AR. (`/docs/decision-guides/rcm-billing`)
- **Pricing = ChargeItemDefinition + `$apply`:** the first applicable base price, then surcharges and discounts, with payer-specific definitions per contract. In Provider, an ActivityDefinition's `applicable-charge-definition` makes visits generate ChargeItems automatically. (`/docs/api/fhir/operations/chargeitemdefinition-apply`, `/docs/provider/visits`)
- **Charting quality decides claim quality.** CMS-1500 data comes from Patient, Coverage, Claim and Encounter; `Claim/$export` renders the PDF. Superbills cover out-of-network or cash-pay patients. (`/docs/billing/creating-cms1500`, `/docs/billing/creating-superbills`)
- **Stedi 837P:** the billing provider goes in `Claim.provider` and clinical roles in `careTeam`, so the orgs need not match. Most rejections are malformed FHIR (invalid NPI checksum, bad NANP phone, missing EIN). On failure the Claim gets `status=error` with no ClaimResponse, so a corrected resubmit works. (`/docs/integration/stedi/claim-submission/professional-claims`)
- **Candid submit is idempotent** (an existing active ClaimResponse is returned). Self-pay means `Coverage.payor` points at the Patient. Request and response are stored as DocumentReferences for debugging. Keep all payer identifier systems. (`/docs/integration/candid/claim-submission`, `/docs/integration/candid/payer-directory`)

## Remittance and denials

- **Be idempotent on the payer's transaction id, not the webhook delivery id,** or you double-post payments. Match 835s by patient control number from the Claim id. (`/docs/decision-guides/rcm-billing`, `/docs/integration/stedi/claim-submission/claim-responses`)
- **Stedi 277CA/835 are stored verbatim as DocumentReferences** (minimal translation), fed by a webhook plus a checkpointed catch-up poller. Stedi requires a response within 5s. (`/docs/integration/stedi/claim-submission/claim-responses`)
- **Denials become ClaimResponse + a Task queue.** Corrected claims link via `Claim.related`; appeals are Tasks with deadlines. (`/docs/decision-guides/rcm-billing`)
- **Value-based care splits into measurement** (Measure/`$evaluate-measure`) **and money** (PaymentReconciliation for PMPM). Still send encounter data for risk adjustment. (`/docs/decision-guides/rcm-billing`)

## Prior authorization

- **CMS-0057-F/HTI-4 make FHIR ePA mandatory from January 2027.** Medplum is pursuing all three criteria (CRD, DTR, PAS) and proving them by multi-party FHIRplace testing, "not a reference implementation". (`/blog/fhirplace-participant-2026`)
- **Today's model:** a CoverageEligibilityRequest with purpose `auth-requirements`, then Claim `use: preauthorization`. Gate scheduling on Task `businessStatus`. (`/docs/decision-guides/rcm-billing`)
- **ePA staging (alpha) runs over CDS Hooks.** Discover via `GET /cds-services` and call by `service.id`, because "the CDS Hooks hook name is NOT the same as the service endpoint path". Start with a client secret before adding JWKS or mTLS. (`/docs/integration/electronic-prior-auth`, `/docs/auth/mtls`)

## Key source articles
`/docs/decision-guides/rcm-billing` · `/docs/billing/patient-insurance` · `/docs/billing/insurance-eligibility-checks` · `/docs/integration/stedi/claim-submission/professional-claims` · `/docs/integration/candid/claim-submission` · `/docs/integration/stedi/claim-submission/claim-responses` · `/blog/fhirplace-participant-2026` · `/docs/integration/electronic-prior-auth`
