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

### Example

```
/medplum admin and medical staff are different teams. how do we give each team its own inbox for patient messages
```

### Tips

- Give context: who the user is, what they do, which platform (patient app, clinician app). Better input, sharper answer.
- Ask for the output you want: "as a table", "as user stories", "just the FHIR resources".
- Ask follow-ups in the same chat. It keeps the model it already built.
- It knows Medplum's docs, not your code. For how your product works today, point it at your code too.
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
