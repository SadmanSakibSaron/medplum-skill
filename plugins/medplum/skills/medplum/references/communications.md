# Messaging and communications

How to build chat, SMS, email and fax on Medplum. Everything is a `Communication`, which is independent of medium. Messaging is a two-level thread hierarchy, routing is done by Tasks, and external channels are bridged by Bots. Async care becomes billable by wrapping sessions in Encounters.

## Data model

- **Communication separates content and metadata from the channel,** whether email, SMS, chat or fax. (`/docs/communications`)
- **Two levels.** The thread header has no payload and no `partOf`, holds the full participant list in `recipient`, and carries the topic as its title. Child messages carry a `payload` and `partOf`. Find headers with `part-of:missing=true`. (`/docs/communications/messaging-data-model`)
- **Put the creator in `recipient`.** FHIR search can't express `recipient OR sender`, so without this, inboxes miss threads the user started themselves. The alternative is two searches merged on the client. (`/docs/communications/messaging-data-model`, `/docs/communications/searching-and-querying-threads`)
- **`category` is the broad class; `reasonCode` is the specific reason.** Set category on both header and children. Combined category codes "explode over time". (`/docs/communications/messaging-data-model`)
- **Linear chat uses `partOf` sorted by `sent`.** Reserve `inResponseTo` for explicit replies. In production, create the header and first message in one transaction with `urn:uuid`. The Medplum App is an inspector, "not the primary authoring experience you ship to users". (`/docs/communications/creating-your-first-thread`)
- **Attachments:** mix `contentString`, `contentAttachment` (a Binary URL, not bytes) and `contentReference` → DocumentReference, which makes each file a searchable first-class clinical document (ThreadChat's default). (`/docs/communications/sending-messages-and-attachments`, `/docs/decision-guides/messaging`)
- **Internal threads may have no `subject`,** so never assume one in queries or UI. (`/docs/decision-guides/messaging`)

## Lifecycle, editing, reading

- **Header `status` means open or closed,** independently of the messages; closing a thread doesn't change its children. (`/docs/communications/thread-lifecycle-participants-access-control`)
- **Retract and correct instead of patching:** mark the original `entered-in-error` and send a new message with `inResponseTo` and a `correction` category, so edits stay searchable. Filter `status:not=entered-in-error`. (`/docs/communications/message-editing-and-drafts`)
- **Drafts** are `status: preparation`, scoped to the sender ("Other users should never see your drafts"), and purged by a cron Bot. (`/docs/communications/message-editing-and-drafts`)
- **Pick a read-receipt model by two questions: group threads, and searchable unread?**
  - A: `completed` = read (1:1 only).
  - B: a per-participant lastRead extension on the header. Update it by URL, never by patch index.
  - C: a read-receipt Task per recipient per message, which gives counts but is the most complex.
  
  (`/docs/communications/read-receipts-and-message-status`)
- **Query thread lists header-first, then each thread.** Avoid `_revinclude=Communication:part-of`, which "tends to over-fetch". Use one WebSocket `useSubscription` per open thread, and find the resource in notification bundles by `resourceType`, not by index. (`/docs/communications/searching-and-querying-threads`)

## Access and participants

- **Visibility is enforced by access policies, not app code.** Use two ORed Communication entries, one for recipient and one for sender. Adding someone as a recipient doesn't grant access if their policy doesn't match. Test with two roles before real data arrives. (`/docs/communications/thread-lifecycle-participants-access-control`)
- **Removing a header recipient isn't retroactive:** they may still read old children, so full revocation needs updates to the children or compartment policies. Edit recipients with `If-Match`. (`/docs/communications/thread-lifecycle-participants-access-control`)
- **Pediatrics:** caregivers act for child patients, which needs parameterised policies (Summer Health, built in 16 weeks). (`/blog/summer-case-study`)

## Routing and automation

- **Content lives in Communication; assignment lives in Task (`focus` = header).** "If there's ever a conflict between `Task.owner` and `Communication.recipient`, the Task is the source of truth." Only create Tasks for messages that need a response. (`/docs/communications/message-response-tracking-and-routing`)
- **Pool + claim:** route by `performerType`; on claim, set `owner` and clear performerType. Reroute the Task first, then the recipient. Record reasons in `note` or Provenance. `owner` is 0..1, so if the previous owner must keep visibility, create a new Task. (`/docs/communications/message-response-tracking-and-routing`, `/docs/decision-guides/messaging`)
- **Real-time vs recurring automation.** Real-time: a Subscription (`part-of:missing=false&status=in-progress`) triggers a Bot; e.g. if Schedule `$find` returns no free slots, the assignee is out of office and the Task returns to the pool. Recurring: a cron Bot scans for stale threads. (`/docs/communications/messaging-automations`)
- **SLA analytics go to a warehouse:** "FHIR search is designed for clinical data lookups, not aggregate analytics." Use `Task.priority` for urgency UI. (`/docs/communications/messaging-automations`)

## External channels

- **Outbound:** call the Bot synchronously (`executeBot`) after creating the message, so provider errors reach the caller instead of failing silently in a Subscription. (`/docs/communications/external-messaging-integration-patterns`)
- **Inbound:** resolve the sender with a conditional reference ("There is no partial write"). Match the thread separately: first by an external conversation id, then by the latest open thread, then a new thread. Dedupe on the provider message id with `createResourceIfNoneExist`, because providers retry. Never silently drop unknown numbers. `medium` routes replies back over the same channel. (`/docs/communications/external-messaging-integration-patterns`)
- **Twilio SMS (hosted only):** `$twilio-sms-install` (idempotent), `$send-sms-twilio` (E.164 normalisation; terminal status never regresses), and inbound SMS stored as a completed Communication with no threading. Inbound is secured by both credentials and an HMAC signature. (`/docs/integration/twilio-sms/setup`, `/docs/integration/twilio-sms/sending-sms`, `/docs/integration/twilio-sms/receiving-sms`)
- **Threading is left to you on purpose.** Use a create-only Subscription Bot keyed by a phone-pair identifier; without create-only, every status callback re-triggers it. (`/docs/integration/twilio-sms/threading`)
- **eFax:** `$send-efax` only submits, so you must cron `$sync-efax-status`. Numbers must be E.164. Track the faxed file as a DocumentReference. A passing connection test followed by 403s means the user id is wrong. (`/docs/integration/efax`)

## Async encounters (billing)

- **Define what a "session" is** (an SMS chain, a thread, or a day's messages) and model it as an Encounter with class `VR`, linked from the thread header only. (`/docs/communications/async-encounters`)
- **Multi-patient sessions get one child medical Encounter per patient** (`partOf` the session), because per-patient practitioner, diagnoses and reasonCode are "critical to billing". No billing intent means no Encounter is needed. (`/docs/communications/async-encounters`, `/docs/decision-guides/messaging`)
- **"Messaging workflows are convenient for users but need aggregation and synthesis to be actionable for providers."** (`/blog/summer-case-study`)

## Key source articles
`/docs/communications/messaging-data-model` · `/docs/decision-guides/messaging` · `/docs/communications/message-response-tracking-and-routing` · `/docs/communications/external-messaging-integration-patterns` · `/docs/communications/read-receipts-and-message-status` · `/docs/communications/async-encounters` · `/docs/communications/thread-lifecycle-participants-access-control` · `/docs/integration/twilio-sms/threading`
