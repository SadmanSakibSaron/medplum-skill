# Medplum Skill

A Claude Code plugin that answers Medplum and FHIR R4 questions the way the Medplum team would, citing medplum.com docs.

## Install

In Claude Code, run:

```
/plugin marketplace add SadmanSakibSaron/medplum-skill
/plugin install medplum@medplum-skill
```

You need read access to this repo first. Restart Claude Code after installing.

## Using the medplum plugin

Two ways to call it:

- **Slash command:** `/medplum <your question>`. Use this when you want it for sure.
- **Just ask:** mention Medplum, FHIR or a resource name (Task, Communication, AccessPolicy) and Claude picks the skill up on its own.

Answers name the exact FHIR resources and fields, recommend a default, flag the gotchas, and cite the medplum.com page they came from.

### Examples

**Model a feature in FHIR**

```
/medplum admin and medical should have different inboxes since they are different teams. how can we achieve that
```
```
/medplum how should we model a questionnaire the patient fills before a video consult, and turn the answers into structured data
```
```
/medplum a parent manages accounts for their children. how do we let them see and book for the kids but not other patients
```

**Access and security**

```
/medplum we have white-label partners like Binsina and NGI. how do we keep each partner's patients separate
```
```
/medplum can a receptionist see appointments but not clinical notes? what does the access policy look like
```

**Automation**

```
/medplum send a reminder 24 hours before an appointment. subscription and bot, or cron?
```
```
/medplum when a lab result comes in, create a task for the ordering doctor
```

**Migration from our Laravel backend**

```
/medplum we are moving from a custom Laravel EHR to Medplum. where do we start and what do we migrate first
```
```
/medplum how do we handle duplicate patient records during migration
```

**Billing and compliance**

```
/medplum how do we bill for async messaging under a subscription
```
```
/medplum what compliance do we inherit from hosted Medplum and what is still on us
```

**Look something up**

```
/medplum what does the Medplum doc say about read receipts
```
```
/medplum explain AccessPolicy parameters like I'm a designer
```

### Tips

- Give context: who the user is, what they do, which platform (patient app, EHR web). Better input, sharper answer.
- Ask for the output you want: "as a table", "as user stories", "just the FHIR resources".
- Ask follow-ups in the same chat. It keeps the model it already built.
- It knows Medplum's docs, not our code. For how your product works today, point it at your code too.
- The docs are a Sept 2026 snapshot. For exact API behavior, check https://www.medplum.com/docs.

## Update

```
/plugin marketplace update medplum-skill
```

## Plugins

| Plugin | What it does |
|---|---|
| `medplum` | Answers Medplum and FHIR R4 questions from the medplum.com docs and blog (493 pieces, as of Sept 2026). Cites source pages. |

## Notes

The `medplum` corpus is Medplum documentation, licensed Apache 2.0. See `plugins/medplum/skills/medplum/corpus/MEDPLUM-LICENSE.txt` and `MEDPLUM-NOTICE`.
The content is a snapshot. For exact current API behavior, check the live docs.
