---
generated: '2026-09-19'
method: generated
name: Trace grant flows and funders for a charity or cohort
description: With a paid key, walk a charity's latest money network, list its funders and grantees, find funders shared across a cohort, and pull a bulk diligence summary - while watching the Advanced allowance.
api: openapi/charitysense-com-openapi.yml
operations: [getApiUsage, getCharityPage, getCharityMoneyNetwork, getCharityFunders, getCharityGrantees, getFundersForCohort, getDiligenceSummary]
tier: advanced
source: >-
  Grounded in openapi/charitysense-com-openapi.yml (operationIds verified verbatim) and the "Paid Advanced" and
  "Money network" sections of https://data.charitysense.com/INSTRUCTIONS_FOR_AGENTS.md.
---

# Trace grant flows and funders for a charity or cohort

Every operation here except `getCharityPage` is an **Advanced operation**: it needs a paid key and consumes that
key's daily Advanced allowance (default 250 per UTC day). Anonymous calls return `403 EntitlementRequired` with
`UpgradeURL: /contact?interest=api-access`.

## Auth
- Send the paid key as `Authorization: Bearer <token>` **or** `X-CharitySense-API-Key: <token>` - the two
  schemes are interchangeable (`authentication/charitysense-com-authentication.yml`). Keys are issued through
  https://data.charitysense.com/contact?interest=api-access; there is no self-serve signup.

## Steps
1. **Check the allowance first** - `getApiUsage` (`GET /api/v2/usage`). Unmetered. Read `Advanced.Remaining` and
   `ResetAt`; if `Remaining` is 0, stop and tell the user when it resets rather than burning `403`s.
2. **Anchor on the profile** - `getCharityPage` (`GET /api/v2/charity/{ein}/page`, public). Confirm the EIN and
   note `Context.SelectedForm` / `SelectedYear`; the money network is always the **latest** filing.
3. **Walk the money network** - `getCharityMoneyNetwork`
   (`GET /api/v2/charity/{ein}/money-network?Scope=Latest&Direction=Outgoing`), then again with
   `Direction=Incoming`. `Scope=Latest` is the only supported scope. `Limit` is 1-200; follow `NextCursor` (opaque)
   until absent. Edges are investigative evidence: preserve resolution confidence and filing evidence, and never
   infer misconduct or an unreported reciprocal filing.
4. **List funders and grantees** - `getCharityFunders` (`GET /api/v2/charity/{ein}/funders?Limit=&MinYear=`) and
   `getCharityGrantees` (`GET /api/v2/charity/{ein}/grantees?Limit=&MinYear=&MinAmount=`). Rows use the `Funder`
   schema (counterparty `EIN`, amounts, years); `AllCategories=true` widens beyond the default grant categories.
5. **Find shared funders across a shortlist** - `getFundersForCohort`
   (`GET /api/v2/funders/for-cohort?EINs=<comma-separated>`) - the question a grantmaker asks of a shortlist
   rather than of one grantee.
6. **Summarize the cohort** - `getDiligenceSummary` (`GET /api/v2/charity/diligence-summary?EINs=`), up to 25
   EINs, returning grant-readiness, financial and live-rating summaries in one call.
7. **Cite** every figure to `https://data.charitysense.com/charity/{ein}` with its filing year.

## Errors
- `401 ApiKeyRequired` (usage endpoint without a key), `403 EntitlementRequired` (no Advanced allowance), `404
  ProfileNotFound`, `429` when the daily allowance is spent - wait for `ResetAt`. See
  `errors/charitysense-com-problem-types.yml` and `rate-limits/charitysense-com-rate-limits.yml`.

## Notes
- Budget: steps 3-6 are at least six Advanced requests per charity before pagination. Plan the cohort size against
  `Advanced.Remaining` from step 1.
- Read-only flow; no idempotency or reversal concerns.
