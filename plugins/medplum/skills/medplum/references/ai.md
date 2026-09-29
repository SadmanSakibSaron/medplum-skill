# AI, LLMs, agents

Medplum's AI thesis: **healthcare AI is an infrastructure problem, not a model problem.** A headless FHIR platform with no private internal pathways is exactly what agents need. LLMs already know FHIR, and agents should be governed like clinicians: "can suggest, but not act".

## The thesis

- **"The barrier to production isn't the AI model; it's the lack of a secure, auditable foundation for healthcare data."** Self-built infrastructure carries a "maintenance tax". (`/docs/ai`)
- **Four requirements:** programmatic data access, standardised data models, explicit guardrails, and built-in interoperability. "AI can only automate what it can access", so the API has "no separate internal pathways". (`/docs/ai`)
- **LLMs already know FHIR,** and FHIR is a clean target for extracting unstructured notes and faxes. Proprietary shortcuts such as hard-coded enums become liabilities. Because the code is open source, it is in LLM training data. (`/docs/ai`)
- **"Interoperability is the difference between a demo and production."** Build a foundation, not a bet on one model. (`/docs/ai`)
- **AI-native beats bolt-on.** MediMind: "Within a few years, practicing medicine without AI assistance will look like malpractice." Build the thing AI runs on, "inside the flow instead of beside it". "A patient handed to you in fragments can only be reasoned about in fragments." (`/blog/medimind-case-study`)
- **FHIR as the operational model, not an edge interop layer.** MediMind wanted "to spend [two years] writing the hospital", not a FHIR server. Each new department became "a vocabulary, not an architecture". (`/blog/medimind-case-study`)

## Guardrails

- **"Can suggest, but not act."** Agents live under the same AccessPolicies as humans, and every action is an AuditEvent. (`/docs/ai`)
- **`$ai` returns suggested tool calls but never executes them.** The app validates, checks permissions and executes. The key stays a project secret; `LLM_BASE_URL` can point at LiteLLM. Streaming disables tool calls. (`/docs/ai/ai-operation`)
- **Spaces:** "The Provider UI, not the bots, executes the FHIR requests", so the assistant can never exceed the user's access. It ships without canonical prompts on purpose. Scope the `ai` feature away from high-risk write roles. Keep the "always use `fhir_request`" instruction or the model invents results. Each iteration is a model call (5–10 for multi-hop prompts). (`/docs/provider/spaces`)
- **AI output stays marked as AI:** preliminary and AI-generated, never merged into the clinical conclusion (MediMind). LLMs stay "out of the clinical decision", with risk scoring grounded in curated data (Profile). (`/blog/medimind-case-study`, `/blog/profile-case-study`)
- **A recommendation someone can accept or reject gets adopted faster** than one that acts on its own. Write model reasoning where reviewers read it: DetectedIssue, Task, Communication. (`/docs/analytics`)
- **PHI requires a BAA with the LLM vendor,** or an access policy limited to non-PHI (definitional) resources. (`/docs/building-with-ai-coding-assistants`)

## MCP

- **One low-level `fhir-request` tool instead of dozens of narrow ones,** because everything is FHIR resources and operations. "We don't have to teach the model healthcare." `search` and `fetch` exist only for client compatibility (ChatGPT). Endpoint: `api.medplum.com/mcp/stream` with OAuth. (`/docs/ai/mcp`, `/blog/unlocking-healthcare-ai-medplum-support-mcp`)
- **"The model already knows how to ask good questions of a FHIR server; the MCP simply lets it ask."** Reach extends through Bots (`$deploy`/`$execute`) and the Agent (`$push` HL7 on-prem). The launch preview was synthetic data only, not for PHI. (`/docs/ai/mcp`, `/blog/unlocking-healthcare-ai-medplum-support-mcp`)

## Building with AI coding assistants

- **Ground the agent in Medplum's own code and docs:** clone and symlink the repo. "FHIR is a standard, not an implementation." Agents "hallucinate fields, mix FHIR versions, and produce plausible-but-wrong code". (`/blog/building-on-medplum-with-ai`, `/docs/building-with-ai-coding-assistants`)
- **Plan first, one task per thread, reset when output turns generic.** Adapt the closest existing implementation (Medplum Provider). (`/docs/building-with-ai-coding-assistants`)
- **Verify mechanically.** Strict TypeScript types turn hallucinated fields into compile errors. Pin R4. `$validate`/`$validate-code` catch invented LOINC/SNOMED codes. (`/docs/building-with-ai-coding-assistants`)
- **Humans review access control:** "a wrong access policy does not throw an error — it quietly exposes data." Encode conventions in CLAUDE.md or AGENTS.md. (`/blog/building-on-medplum-with-ai`)

## Patterns in the field

- **System of action:** the FHIR store triggers and tracks agent work. A Subscription on an Appointment confirmation Bundle starts a voice agent, which verifies DOB and updates Task and Appointment. RelatedPerson lets one call to a parent cover two children. (`/blog/scheduling-agents-unity-ai`)
- **Distributed EHR:** specialised vertical apps on a shared headless FHIR layer, replacing monoliths. Keep app config in FHIR too, rather than a side SQL DB. (`/blog/scheduling-agents-unity-ai`)
- **Meet the fax where it is:** Titan normalises referral documents into FHIR with LLMs, predicts HCC/Elixhauser codes, and runs an EMPI before syncing to Epic, Cerner and NextGen. (`/blog/titan-case-study`)
- **OCR pipeline:** `$aws-textract` (+ Comprehend Medical) writes outputs as new Binary + Media linked to the source and leaves the original untouched. (`/docs/ai/aws`)
- **Conversational intake grounded in Questionnaires/ValueSets** replaced a 26-page form (Profile). Everself wanted "a data structure we own that we can integrate more AI into". (`/blog/profile-case-study`, `/blog/everself-case-study`)

## Key source articles
`/docs/ai` · `/docs/ai/mcp` · `/docs/ai/ai-operation` · `/docs/provider/spaces` · `/docs/building-with-ai-coding-assistants` · `/blog/medimind-case-study` · `/blog/scheduling-agents-unity-ai` · `/blog/unlocking-healthcare-ai-medplum-support-mcp` · `/blog/titan-case-study`
