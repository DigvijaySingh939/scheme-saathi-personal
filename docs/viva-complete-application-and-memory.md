# Complete application workflow and memory

## Short viva answer

“Scheme Saathi collects and confirms a person's relevant details, finds
relevant schemes, and uses reviewed rules in code to check eligibility. The
language model helps understand messages and explain results, but it does not
decide eligibility. If the person wants to apply, the system prepares a draft
for review and waits for human approval. Optional long-term memory uses
ChromaDB and works only with recorded consent, allowed fields, and expiry
rules.”

## Main components

1. **Streamlit UI** — the pages and controls the user sees.
2. **FastAPI backend** — receives UI requests, validates identity, and calls
   application services.
3. **Agent workflow** — routes questions to intake, scheme information,
   eligibility explanation, document checking, and application preparation.
4. **Rule engine** — makes the eligibility decision from reviewed rules.
5. **Scheme knowledge tools** — provide scheme details, evidence, and
   document lists.
6. **Memory service** — optionally remembers consented facts and application
   references.
7. **Mock portal** — creates and records a mock application; it is not a real
   government portal.

## End-to-end flow

1. **Sign in and consent.** The user signs in. Before entering chat, the UI
   records the user's general consent to use the service. Separate optional
   choices control whether profile facts or application references may be
   remembered.
2. **Ask a question.** The user asks in English, Hindi, or Marathi. The
   supervisor identifies the likely intent and language and routes the
   request.
3. **Collect facts.** Intake extracts facts from the message and asks about
   missing information that could affect a result. Facts about another person
   are tracked separately.
4. **Find relevant schemes.** The workflow can shortlist schemes using the
   request and scheme information. Scheme data and evidence are retrieved by
   the scheme knowledge tools.
5. **Check eligibility.** The rule engine evaluates facts against reviewed
   rules. Its results are `eligible`, `not_eligible`, or `cant_tell`. Draft
   rules do not make a final decision. If an important fact is unknown, the
   workflow asks a question rather than guessing.
6. **Confirm facts.** Before final results rely on a profile, the user sees
   and confirms the details. Remembered details are shown as unconfirmed
   until the user checks them.
7. **Explain the result.** The eligibility agent explains the rule-engine
   result in plain language. It must keep the verdict and citations from the
   rule engine; it cannot change the decision.
8. **Prepare an application if requested.** The application agent fills form
   fields from confirmed facts. It does not invent applicant values. Missing
   required fields are shown.
9. **Human review and approval.** The graph pauses at an approval step. The
   person can approve, edit, or reject. Approval is tied to a hash of the
   exact form. If the form changes, the old approval is no longer valid.
10. **Mock submission and follow-up.** Only after valid approval can the
    mock portal record a submission. The UI clearly tells the user this is
    not a real government submission.

## What the language model does, and what code does

**The model helps with:**

- understanding the user's words and likely intent;
- extracting possible facts from natural language;
- choosing relevant scheme information to discuss;
- writing a clear explanation of an existing result.

**Code and reviewed data control:**

- eligibility verdicts and conditions;
- whether a rule is reviewed or only a draft;
- user and household ownership;
- required form fields and validation;
- consent and memory expiry;
- whether an approval matches the exact application being submitted.

## How long-term memory is stored

The persistent implementation is
[`agents/chromadb_memory_service.py`](../agents/chromadb_memory_service.py).
It uses ChromaDB with three collections:

- `memory_facts` — one current allowed fact per household and field;
- `memory_application_references` — references to submitted applications;
- `memory_consents` — consent decisions and withdrawals.

The memory directory is selected with `SCHEME_SAATHI_MEMORY_DIR`; without
that setting, the service uses its configured local default. Memory is
different from conversation checkpoints: checkpoints keep workflow progress,
while memory is optional reusable information and application references.

### How facts are saved

1. A citizen chooses whether to allow profile facts to be remembered.
2. The person confirms the details shown by the workflow.
3. The memory code checks that consent is active and the fact is on the
   allowlist in `data/memory_policy.json`.
4. It checks the value type and retention policy. Sensitive identifiers and
   fields marked temporary-only are not stored.
5. It records confirmation, reconfirmation, expiry, and consent information.

Facts about another person are not saved as the signed-in citizen's profile.
CSC operators do not have standing consent to save household facts. An
application reference for an operator's list needs the person's agreement for
that individual application.

### How facts are retrieved

1. The backend builds the memory scope from the verified sign-in token.
2. For a citizen checking their own eligibility, the workflow asks memory only
   for allowed fields relevant to the candidate schemes and not already
   supplied in the conversation.
3. ChromaDB retrieves records by exact IDs and metadata filters. It does not
   use similarity search for personal memory.
4. The workflow shows the returned facts to the user as unconfirmed. The user
   must confirm them before the rule engine can rely on them.

The memory policy sets field-specific reconfirmation and expiry periods.
Application references default to 90 days. Expired entries are unusable
immediately and a background cleanup task removes expired records periodically.

### How to forget data

The memory panel lets the user inspect remembered facts, withdraw consent,
delete individual items, and choose **Forget my data**. The forget endpoint
deletes that household's stored facts, application references, relevant
consents, and workflow records. It cannot delete anything already submitted
to an external government service.

## Different kinds of stored data

- **Streamlit session state:** temporary UI state in the current browser
  session.
- **Workflow checkpoints:** paused or ongoing workflow state. They can be
  persisted to SQLite when `SCHEME_SAATHI_CHECKPOINT_DB` is configured;
  otherwise they are in memory and are lost on process restart.
- **Long-term memory:** optional, consented ChromaDB records with policy
  checks and expiry.
- **Uploaded supporting documents:** local files with a 30-day retention
  period in the current implementation. They are not automatically read or
  authenticated by OCR.
- **Scheme information and rules:** project data used for search and
  deterministic eligibility checks.

These stores serve different purposes and should not be described as one
database.

## A compact flow to draw in a viva

```text
User
  -> Streamlit UI
  -> FastAPI + Auth0 identity check
  -> Supervisor / intake
  -> Scheme search + reviewed rule engine
  -> User confirms facts
  -> Explain result
  -> Optional document checklist and application draft
  -> Human approval of exact draft
  -> Mock portal

Optional memory:
  consent + confirmed allowed facts -> ChromaDB
  next session -> exact scoped lookup -> user confirms again -> rule engine
```

## Main files to mention

- UI and API: [`viva-ui-api.md`](viva-ui-api.md)
- Rule and agent workflow:
  [`graph/agent_workflow.py`](../graph/agent_workflow.py)
- Eligibility rules: `servers/rule_engine/`
- Scheme data: `data/schemes/` and `data/rules/`
- Memory policy: `data/memory_policy.json`
- Memory API: [`app/memory_api.py`](../app/memory_api.py)
- Memory service: [`agents/chromadb_memory_service.py`](../agents/chromadb_memory_service.py)
- Checkpoint configuration: [`app/checkpoints.py`](../app/checkpoints.py)
- Document checking limits: [`viva-document-checker.md`](viva-document-checker.md)
