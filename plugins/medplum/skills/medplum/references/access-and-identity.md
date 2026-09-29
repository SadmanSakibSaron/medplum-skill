# Identity, auth, access control, multi-tenancy

How Medplum decides who you are and what you may touch. The model is deliberately simple: Project → ProjectMembership → AccessPolicy. It still covers tenants, fields, state transitions and emergency access. On the auth side it covers standard OAuth2/OIDC, SMART, external IdPs, MFA, and back-end patterns such as on-behalf-of.

## The core model

- **"Security and access controls are notoriously difficult", so the model stays simple.** Everything lives in a Project. A user holds a ProjectMembership (profile, accessPolicy, admin flag) per project. The profile is Patient, Practitioner or RelatedPerson for people, and ClientApplication or Bot for programs. (`/docs/access/access-policies`, `/docs/user-management`)
- **Projects are the hard boundary.** No cross-project references. The common pattern is separate dev, staging and prod projects. Project linking gives read-only sharing (terminology, profiles, Bots) but **always set `exportedResourceType`**, or every type is exposed, including future ones. (`/docs/access/projects`, `/docs/tutorials/register`)
- **Server-scoped vs project-scoped users.** Server scope (default for Practitioners) spans projects and suits developers and admins. Project scope (default for Patients) suits real clinicians and patients, and is required for custom email flows. Scope decides who owns the User record, not access. (`/docs/user-management/project-vs-server-scoped-users`, `/docs/api/fhir/operations/user-rescope`)
- **Deactivate, don't delete.** Set `active: false` on the membership: it is immediate, auditable and reversible. "Never delete profile resources or hand-edit memberships." Change emails with `$update-email`, because `User.email` is the login and profile telecom is only contact info. (`/docs/user-management`, `/docs/api/fhir/operations/user-update-email`)
- **Admin ≠ unlimited.** The admin flag covers Project, ProjectMembership and User; admins still need a policy for clinical data. Super Admin bypasses validation and "can cause irreparable data changes". (`/docs/access/admin`, `/docs/decision-guides/access-control`)

## Access policies

- **Rules cover type, criteria, interaction, field and write constraint.** Criteria use FHIR search syntax (only the `:not`/`:missing` modifiers, no chaining). Interactions (`create`…`vread`) generalise readonly; upserts need search+create+update. (`/docs/access/access-policies`)
- **`hiddenFields`, `readonlyFields`, and `writeConstraint` FHIRPath with `%before`/`%after`,** e.g. no status change after `final`. `hiddenFields` only masks output; for decluttering, change the UI. Keep FHIRPath simple and put logic in Bots. (`/docs/access/access-policies`, `/docs/decision-guides/access-control`)
- **Parameterised policies are templates:** `%profile`, `%organization`, and custom variables set on `ProjectMembership.access.parameter` (names must match). Uses: parents of pediatric patients, clinicians limited by state licence. (`/docs/access/access-policies`, `/docs/access/multi-tenant-access-policy`)
- **"Stacking is additive only."** Policies combine by union and can never narrow. Test: *"If this token leaked, should the attacker reach both scopes?"* If not, use separate logins. (`/docs/decision-guides/access-control`)
- **Effective access = SMART scopes ∩ AccessPolicy.** IP rules are IPv4 prefix matches evaluated in order and must end with a `*` block. (`/docs/decision-guides/access-control`, `/docs/access/ip-access-rules`)
- **Binaries aren't searchable, so they use `securityContext`.** Without it, anyone with Binary permission who learns the UUID can read the file, and UUIDs leak via logs and URLs. Use `createMedia()`/AttachmentButton, which set it automatically. "When in doubt, err on the side of more restrictive access controls." (`/docs/access/binary-security-context`)
- **Break glass (ONC d6):** narrow, time-bound, audited. E.g. temporarily add the practitioner as `generalPractitioner`, then remove them and review the AuditEvents. (`/docs/access/access-policies`)
- **Drive role-aware UI from `AccessPolicy.basedOn` via `/auth/me`** so the UI never drifts from API enforcement. Open patient registration requires a restrictive default patient policy. (`/docs/decision-guides/access-control`, `/docs/user-management/open-patient-registration`)

## Multi-tenancy

- **Three steps:** model tenants as a resource (Organization for clinics/MSO, HealthcareService for service lines, CareTeam for per-patient teams); label data with `$set-accounts` into `meta.compartment`; grant tenants through parameterised policies on the membership. (`/docs/access/multi-tenant-access-policy`)
- **Only Patient propagates** (`propagate: true`) to its compartment; call it once and new resources inherit it. `$set-accounts` needs explicit operation permission, and should run async. (`/docs/access/multi-tenant-access-policy`, `/docs/api/fhir/operations/set-accounts`)
- **"Push complexity into enrollment Bots, not policies."** Tag at the lowest hierarchy level, and add rather than replace a compartment to share. A classic misconfiguration: forgetting unfiltered read on terminology, which leaves dropdowns empty. (`/docs/decision-guides/access-control`)
- **MSO pattern:** one Project with an Organization per clinic, so tenants can still coordinate. Consent in the compartment can gate access, and `%profile` enables per-practitioner assignment. (`/blog/multi-tenant-mso`)
- **Allowed tenants ≠ active tenant.** Hard PHI isolation needs separate memberships and logins; tenant-scoped screens need one membership plus a UI filter via `_compartment`. Validate any URL tenant parameter. (`/docs/access/tenant-selector`)
- **Policies can replace a gateway.** EnSage exposed the FHIR API directly to partner physicians "without the need to encapsulate it behind a gateway / proxy", with Google for staff and Auth0 for referrers. (`/blog/ensage-case-study`)

## Authentication

- **Decide identifiers, IdP and token flow up front.** Email is simple but changes (marriage, rebrands); external IDs are stable but lock you to one IdP. Enterprise setups go hybrid, routing by email domain to partner IdPs. (`/blog/identity-management`)
- **Default: three-legged OAuth.** The browser ends up with only a Medplum token: "Clean, simple, secure." Use token exchange (RFC 8693) only when the app also needs other services, and remember that "you own the identity-to-membership mapping". Before building custom auth middleware, ask what it solves and what it will cost to maintain. (`/blog/identity-management`, `/docs/auth/token-exchange`)
- **IdP options:** Medplum as IdP (fastest); external IdP routed by app (ClientApplication) or by email domain (DomainConfiguration, enterprise SSO including the App); direct external JWTs (self-hosted, matched by `fhirUser` or `externalId`). (`/docs/auth`, `/docs/auth/external-identity-providers`, `/docs/auth/domain-level-identity-providers`, `/docs/auth/direct-external-auth`)
- **Patients can't log in to the Medplum App,** which is an admin tool, so build a portal. Never ship a long-lived `client_secret` in a mobile binary. (`/docs/auth`)
- **Back-end patterns:** client credentials; client assertion (JWT, required by SMART Backend Services); mTLS (Da Vinci PAS/HTI-4, hosted at `mtls.api.medplum.com`); on-behalf-of (`X-Medplum-On-Behalf-Of`: the user's policy applies and `meta.onBehalfOf` is recorded); pre-authorized code for magic links and QR codes. (`/docs/auth/client-assertion`, `/docs/auth/mtls`, `/docs/auth/on-behalf-of`, `/docs/auth/pre-authorized-code`)
- **Tokens:** access tokens last 1h and refresh tokens (offline scope only) 2 weeks. The grace-period refresh doesn't run in the background, so idle mobile apps must refresh on resume. Rotate client secrets with no downtime via `$rotate-secret`. (`/docs/auth/session-management`, `/docs/api/fhir/operations/rotate-client-secret`)
- **MFA:** TOTP by default, email codes opt-in. Project and user requirements add together. MFA applies only to password logins, so configure external-IdP MFA at the IdP. (`/docs/auth/mfa`, `/docs/auth/how-mfa-works`)
- **SMART App Launch 2.0 scopes.** Hide scope selection only for provider-facing or first-party apps, because (g)(10) requires granular patient choice. (`/docs/access/smart-scopes`)
- **Provisioning:** the invite API (scope, membership overrides, `mfaRequired`), SCIM 2.0, and `Project/$init` for automated tenant onboarding. Project SMTP fails closed. (`/docs/api/project-admin/invite`, `/docs/api/scim/overview`, `/docs/api/fhir/operations/project-init`, `/docs/user-management/project-smtp`)

## Key source articles
`/docs/access/access-policies` · `/docs/decision-guides/access-control` · `/docs/access/multi-tenant-access-policy` · `/blog/identity-management` · `/blog/multi-tenant-mso` · `/docs/auth` · `/docs/access/tenant-selector` · `/docs/access/binary-security-context` · `/docs/user-management` · `/docs/auth/on-behalf-of`
