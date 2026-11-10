# Household profiles and personas

## Short viva answer

“A household profile contains facts used to check scheme rules, such as age,
income, or state residence. In a real session, the backend gets the household
identity from the verified sign-in token and collects profile facts from the
user. The persona files are fictional test examples; they are not the source
of a real user's profile.”

## Two meanings of “persona”

It is useful to distinguish these two things:

### 1. Household profile in a real conversation

The user describes their situation. The intake agent extracts possible facts
from the conversation and asks follow-up questions if important details are
missing. The UI shows the facts for confirmation before they are used for a
final eligibility result.

The household ID is different from profile facts. For a citizen, the backend
reads the household ID from the verified Auth0 token. It does not trust a
household ID supplied by the language model or request body.

### 2. Synthetic persona files for testing

Files in `data/personas/` and `data/household_profiles/` describe fictional
test households. They help the team run repeatable examples and test whether
the system asks useful questions and handles unknown facts correctly.

[`graph/persona.py`](../graph/persona.py) can load a test persona by ID for
[`graph/run.py`](../graph/run.py) and tests. It converts some test-data field
names to rule-engine names. For example, `respondent_age` becomes `age`.
Values that are `null` are left out, so an unknown answer is not treated as a
confirmed fact.

These persona files are test inputs, not production user accounts or the
long-term memory database.

## How profile facts are used

1. Intake collects facts that may matter to the selected or suggested schemes.
2. Facts remain unconfirmed until the user sees and confirms them.
3. The rule engine checks the confirmed facts against reviewed scheme rules.
4. The result is `eligible`, `not_eligible`, or `cant_tell`. A missing fact can
   lead to `cant_tell`; the system should ask rather than guess.
5. Only confirmed facts are used to fill matching fields in an application
   draft. The person reviews the draft before any mock submission.

The language model can extract and explain information, but eligibility is
decided by code and reviewed rules.

## When the conversation is about another person

A citizen or CSC operator may ask about a family member. The workflow tracks
whose facts are being discussed and may store a short relation such as
“mother.” When it switches to a different person, it clears the earlier
assessment facts so that details from two people do not get mixed.

Operators are assigned an operator identity, not a citizen household identity.
They do not get standing permission to save household facts. A person being
helped must agree separately if an application reference is to be kept for an
operator's list.

## Files to mention

- Synthetic persona loader: [`graph/persona.py`](../graph/persona.py)
- Example fictional personas: `data/personas/`
- Example household test cases: `data/household_profiles/`
- Authenticated household identity: [`app/auth.py`](../app/auth.py)
- Profile and subject handling: [`graph/agent_workflow.py`](../graph/agent_workflow.py)
- Allowed memory fields and retention: `data/memory_policy.json`
