---
generated: '2026-09-19'
method: generated
name: Report a data gap or result problem back to CharitySense
description: Send structured agent feedback when a result is missing, wrong, or needs a different data signal - the one write most agents will legitimately make, and a consequential POST that needs the user's explicit confirmation.
api: openapi/charitysense-com-openapi.yml
operations: [getApiUsage, submitAgentFeedback]
tier: advanced
source: >-
  Grounded in openapi/charitysense-com-openapi.yml (AgentFeedbackRequest schema; operationIds verified verbatim) and
  the "Access, consent, and side effects" section of https://data.charitysense.com/INSTRUCTIONS_FOR_AGENTS.md.
---

# Report a data gap or result problem back to CharitySense

`submitAgentFeedback` is the provider's designated channel for agents: "Agents may POST to
/api/v2/agent-feedback when results are missing, confusing, incomplete, or need a different data signal." It is a
paid Advanced operation and the provider classes every POST as **consequential**.

## Auth
- Paid key as `Authorization: Bearer <token>` or `X-CharitySense-API-Key: <token>`. Anonymous POSTs return `403
  EntitlementRequired` (observed 2026-09-19).

## Consent - do this before the call
- Provider rule: "Explain the exact action and obtain user confirmation immediately before calling one. Never
  infer consent from an earlier research request." Show the user the `Message` you are about to send and get a
  yes.

## Steps
1. **Check the allowance** - `getApiUsage` (`GET /api/v2/usage`); the POST consumes one Advanced request.
2. **Compose the feedback** - body for `submitAgentFeedback` (`POST /api/v2/agent-feedback`,
   `application/json`), schema `AgentFeedbackRequest`:
   - `Message` (required, max 8,000 chars) - what was missing or wrong, and what signal would have helped.
   - `FeedbackType` (optional string), `EIN` (optional, the organization concerned), `Email` (optional, for a
     reply), `RecoverySessionID` (optional, max 128), `Metadata` (optional object).
   Optionally identify yourself with the voluntary `X-Agent-Name` / `X-Agent-Version` / `X-Agent-Owner` /
   `X-Agent-Contact` / `X-Agent-Purpose` headers the provider lists in `ai-profile.json`.
3. **Send once** - the response is `FeedbackResult`. Do not resend on a timeout without asking again.

## Errors
- `422` if `Message` is missing; `401 ApiKeyRequired` / `403 EntitlementRequired` without a valid paid key; `429`
  when the Advanced allowance is spent. See `errors/charitysense-com-problem-types.yml`.

## Idempotency and reversibility
- **None.** There is no `Idempotency-Key`, no de-duplication and no delete/withdraw operation
  (`conventions/charitysense-com-conventions.yml`, `idempotency.coverage: none`, `reversibility.status: none`).
  A retried POST files a second feedback record - which is why the confirmation step is not optional.
