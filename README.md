# Housecall Claude Plugins

Shared Claude Code plugins for the Housecall team.

## Install

In Claude Code, run:

```
/plugin marketplace add <this-repo-url-or-local-path>
/plugin install medplum@housecall-plugins
```

Restart Claude Code after installing. Then use `/medplum <question>`, or just ask a Medplum or FHIR question.

## Update

```
/plugin marketplace update housecall-plugins
```

## Plugins

| Plugin | What it does |
|---|---|
| `medplum` | Answers Medplum and FHIR R4 questions from the medplum.com docs and blog (493 pieces, as of Sept 2026). Cites source pages. |

## Notes

The `medplum` corpus is Medplum documentation, licensed Apache 2.0. See `plugins/medplum/skills/medplum/corpus/MEDPLUM-LICENSE.txt` and `MEDPLUM-NOTICE`.
The content is a snapshot. For exact current API behavior, check the live docs.
