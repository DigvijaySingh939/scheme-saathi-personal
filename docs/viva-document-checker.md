# Document checker: how it works

## Short viva answer

“The document checker compares a scheme's required-document list with the
documents the family reports. It can identify checklist gaps and compare
explicitly reported details with confirmed profile facts. It does not prove a
document is genuine or legally valid, and it does not automatically read a
file using OCR.”

## Inputs and result

The checker is given:

1. The selected scheme.
2. The documents required for that scheme.
3. A list of available documents or descriptions, when supplied.
4. Confirmed household facts for simple consistency checks.

It returns:

- documents that are not clearly listed as available;
- possible inconsistencies based on explicit details;
- clarifying questions; and
- `authenticity_verified`, which remains `false`.

## How it checks the information

1. It compares document names against the required checklist. It ignores
   differences in case, spaces, and punctuation and recognizes a small set of
   common name variants.
2. A required item is reported as missing only when no listed item clearly
   matches it. This means “not listed,” not “the family does not own it.”
3. If a description explicitly includes a detail, such as a labeled name or
   age, local code can compare it with a corresponding profile fact.
4. It ignores sensitive profile fields such as Aadhaar, PAN, bank-account
   numbers, passwords, and tokens. It does not repeat those values in its
   response.
5. It asks a question when the supplied information is unclear instead of
   guessing.

The language model helps interpret descriptions and explain results. Local
code calculates the missing checklist items and performs the supported
explicit-value comparisons. The checker cannot decide scheme eligibility.

## Important limit in the current application

The application has a file-upload flow and a document-checking agent, but
uploading a PDF or image does **not** mean the checker reads the file. The
uploaded file is stored for the application; the checker works on text fields
and document descriptions passed to it. The current workflow does not
automatically extract file contents with OCR or verify authenticity.

So, in a viva, say “we check the reported checklist and explicitly supplied
details,” not “we verify the uploaded certificate.”

Uploaded supporting files are stored locally by
[`app/document_storage.py`](../app/document_storage.py), with a 30-day
retention period. The UI asks users to mask full identity numbers before
uploading. Production storage should be configured and reviewed separately.

## Where the code lives

- Agent comparison and safety limits:
  [`agents/document_checker.py`](../agents/document_checker.py)
- Read-only MCP field comparison tool:
  [`servers/doc_check/server.py`](../servers/doc_check/server.py)
- Agent workflow node:
  [`graph/agent_workflow.py`](../graph/agent_workflow.py)
- Uploaded file storage:
  [`app/document_storage.py`](../app/document_storage.py)
- Application upload API:
  [`app/api.py`](../app/api.py)

The MCP comparison tool is read-only. It does not modify a document or submit
an application.
