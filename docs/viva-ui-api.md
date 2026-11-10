# UI and API: how they work together

## Short viva answer

“The user interacts with Scheme Saathi through a Streamlit web interface. The
interface sends the user's message to a FastAPI backend over HTTP. FastAPI
checks the user's sign-in token, passes the request to the agent workflow, and
returns the answer and workflow data to Streamlit. Streamlit then displays the
answer, eligibility result, or next question.”

## Main parts

- **Streamlit UI** — [`app/app_new.py`](../app/app_new.py). It displays the
  sign-in page, chat, eligibility results, application steps, and memory
  settings.
- **FastAPI API** — [`app/api.py`](../app/api.py). It receives requests,
  checks access, calls the workflow, and returns responses.
- **Agent workflow** — [`graph/agent_workflow.py`](../graph/agent_workflow.py).
  It decides which agent or rule tool should handle the request.

## What happens when someone sends a chat message?

1. The user signs in with Auth0. The UI uses the Authorization Code flow with
   PKCE; the token is kept in Streamlit's session state.
2. The UI sends a `POST /api/chat` request with the message and, for an ongoing
   conversation, its thread ID. It sends the access token in the HTTP
   `Authorization` header.
3. FastAPI validates the token. It gets the citizen's household identity from
   the trusted token claim, not from an ID supplied in the message.
4. FastAPI passes the message, thread, and trusted identity to the workflow.
5. The workflow routes the request and runs the required steps, such as asking
   for facts, checking rules, or answering a scheme question.
6. FastAPI returns a JSON response with the assistant's text, thread ID, next
   step, and any result data.
7. Streamlit adds the response to the chat and displays any eligibility
   results or confirmation controls.

The UI can use a separate API address from `SCHEME_SAATHI_API_URL`. If it is
not set, the Streamlit process starts a local FastAPI server on
`127.0.0.1:8766`.

## Application and document requests

The UI also uses other FastAPI routes. In the guided application flow, it sends
`POST /api/analyze` and receives progress and the final result as
server-sent events (SSE). It can upload a file to
`POST /api/threads/{thread_id}/documents/upload`, and sends the person's
approval or edit to a decision endpoint.

These are related but distinct workflows: `/api/chat` uses the agent
conversation graph. The guided application flow uses the demo service and its
application graph. The demo portal is not a real government submission
service.

## What should I say about security?

- FastAPI checks the Auth0 token before protected work.
- Thread and memory ownership use the identity verified by the backend.
- The model does not choose a household ID.
- A user must review and approve an application before the mock submission
  step.
- Streamlit browser session state is for current UI state; workflow
  checkpoints and consented long-term memory are separate storage systems.

## Main files to mention

- UI: [`app/app_new.py`](../app/app_new.py)
- API routes: [`app/api.py`](../app/api.py)
- Auth0 checks: [`app/auth.py`](../app/auth.py)
- Agent graph: [`graph/agent_workflow.py`](../graph/agent_workflow.py)
- Guided application logic: [`app/business_logic.py`](../app/business_logic.py)
- Workflow tools: [`graph/tools.py`](../graph/tools.py)
