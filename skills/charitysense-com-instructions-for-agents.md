# CharitySense agent instructions

CharitySense provides canonical nonprofit research resources over HTTP. The
API base is `https://data.charitysense.com/api/v2`; the authoritative OpenAPI
3.1 contract is available as
[`/openapi.yaml`](https://data.charitysense.com/openapi.yaml) and
[`/openapi.json`](https://data.charitysense.com/openapi.json).

Responses use UpperCamelCase for CharitySense-owned fields and return the
documented resource directly. Documents are sparse: if a fact, block, or
section is absent, do not interpret it as false, zero, or an empty result.

## Access, consent, and side effects

Public research GET operations need no API token and are limited to 1,000
requests per UTC day per client. External Advanced calls require either:

```http
Authorization: Bearer <token>
```

or:

```http
X-CharitySense-API-Key: <token>
```

Paid keys have per-key daily Data and Advanced limits; the normal paid defaults
are 100,000 data requests and 250 Advanced requests. Request access at
`https://data.charitysense.com/contact?interest=api-access`.

Use `GET /api/v2/usage` with a paid key to retrieve that key's current UTC-day
Data and Advanced `Limit`, `Used`, `Remaining`, and `ResetAt` values. This query does
not consume either allowance and never reveals usage for another key.

AI POST operations are consequential. Explain the exact action and obtain user
confirmation immediately before calling one. Never infer consent from an
earlier research request.

## Reliable research workflow

1. Resolve the organization with
   `GET /api/v2/search?Query=...&Mode=Identity`.
2. Normalize the selected EIN to nine digits without punctuation.
3. Fetch `GET /api/v2/charity/{ein}/page`.
4. Read `Context.AvailableForms` before choosing a form. All of `990`, `990EZ`,
   `990N`, `990PF`, and `990T` are first-class contexts.
5. Read the page's `Sections` manifest. Fetch a section only if its exact `Id`
   is advertised.
6. Follow `NextCursor` until it is absent when the task needs complete results.
   Treat every returned cursor as opaque and use it only with the route that
   produced it.
7. Preserve `EvidenceRefs`, `SourcePath`, form, filing year, and source document
   in downstream analysis.
8. Cite the canonical public profile:
   `https://data.charitysense.com/charity/{ein}`.

Use `EvidenceRef` on a section request only when an earlier page or section
response supplied that exact value. It selects an evidence-targeted historical
filing or schedule record; do not construct it yourself.

## Complete operation inventory

### Stable public research

```text
GET /api/v2/health
GET /api/v2/usage
GET /api/v2/search
GET /api/v2/charity/{ein}/page
GET /api/v2/charity/{ein}/sections/{SectionId}
GET /api/v2/charity/{ein}/filings
```

### Paid Advanced

The derived reads below are GETs, but they consume the Advanced allowance
rather than the 1,000/day research allowance.

```text
GET  /api/v2/stats
GET  /api/v2/top-lists
GET  /api/v2/charity/{ein}/brand-icon
GET  /api/v2/charity/{ein}/money-network
GET  /api/v2/charity/{ein}/funders
GET  /api/v2/charity/{ein}/grantees
GET  /api/v2/charity/{ein}/awards
GET  /api/v2/charity/{ein}/subawards
GET  /api/v2/funders/for-cohort
GET  /api/v2/charity/{ein}/discovery
GET  /api/v2/charity/diligence-summary
POST /api/v2/charity-question
POST /api/v2/agent-feedback
GET  /api/v2/assistant/capabilities
POST /api/v2/assistant/chat/stream
```

Assistant chat is server-sent events and terminates with a `[DONE]` data frame.

The exact parameters, bodies, content types, status codes, and schemas for all
operations are in OpenAPI. Use `operationId` values when generating tools.

## Endpoint behavior

### Search

Required parameter: `Query`.

- `Mode=Identity` is the default and resolves EINs, names, historical names,
  aliases, and acronyms.
- `Mode=Discovery` searches cause, mission, geography, and donor intent.
- Optional filters: `State`, `Cause`, and `MinRevenue`.
- `Sort` is `Relevance`, `Revenue`, `Assets`, or `Recent`.
- `Cursor` is a one-based result page; `Limit` is 1–50.

An Identity result includes `Match`. A related-organization discovery result
may instead include `Explanation` and `Similarity`. Do not require a `Match`
object outside the search response.

### Charity page and sections

The page response is the bounded entry point. `Context` declares the selected
form/year and every available form. `Sections` is the only valid section
inventory. Some dynamic sections, including MoneyNetwork, do not have a `Ref`;
use their advertised `Id`.

A few sections are synthesized at request time rather than compiled into the
document: `WhatTheyDo`, `People`, `GrantReadiness`, `MoneyNetwork`, and
`RelatedOrganizations`. You may request one directly even when the page did not
advertise it. If that organization has nothing to serve there, the response is
`404 SectionNotAdvertised` — an ordinary absence, not an error to retry. A
`500 AdvertisedSectionEmpty` is different: it means the page advertised a
section we then failed to serve, and it is a defect on our side.

Section parameters are `Form`, `Year`, `Cursor`, `Limit`, and `EvidenceRef`.
Paginated section responses include `NextCursor`. Evidence references identify
the canonical collection, form, tax year, field path, source path, and source
document when available.

### Filings

`Form` selects an available form context. `Cursor` is opaque, and `Limit` is
1–100. Each item is filing metadata, not a substitute for the complete
section-level filed content.

### Money network

The only supported scope is `Scope=Latest`. `Direction` may be `Incoming` or
`Outgoing` relative to the requested EIN. `Cursor` is opaque, and `Limit` is
1–200. Preserve resolution confidence and filing evidence. Do not treat an edge
as misconduct or infer an unreported reciprocal filing.

### Related discovery

`Intent` is optional. If omitted, CharitySense derives an intent from the
profile's canonical cause, mission, or region. The source EIN is omitted from
results.

### Dataset endpoints

`/stats` provides current published corpus totals, form counts, and freshness;
use it instead of hard-coding counts. `/top-lists` returns six current
cause-based lists. Both consume the Advanced allowance and need a paid key. `/health` is a liveness and published-overlay integrity
probe. `/usage` is an authenticated, non-metered view of the calling paid key's
current UTC-day Data and Advanced allowance.

## Errors and retry policy

Non-2xx JSON responses contain `detail`. Depending on the failure, `detail` is:

- an object with a stable `ErrorCode` and optional `Message`;
- a validation-error array; or
- a plain message.

Do not retry `400`, `401`, `404`, or `422` unchanged. For `429`, wait before
retrying. Treat `503` as temporary only when a later retry is useful to the
user; use bounded exponential backoff.

## Evidence and interpretation

Treat every claim as a statement about a filed return or an identified overlay
source. Financial figures are historical and may lag current operations.
Ratings are computed live from canonical facts and reference inputs; no
persisted final rating should be treated as authoritative. A `990T` context
covers unrelated business income rather than the whole organization, and a
`990N` context often has too little financial detail to rate. Preserve those
limitations in the answer.

Never fabricate a missing amount, counterparty, relationship, source path,
filing year, or evidence reference.

## Machine discovery

- `/openapi.yaml` and `/openapi.json`: identical callable contract
- `/docs` and `/redoc`: human renderings of that same contract
- `/.well-known/api-catalog`: RFC 9727 linkset
- `/.well-known/agent.json`, `/agent.json`, `/agents.json`: generic agent card
- `/.well-known/agent-card.json`: OpenAPI compatibility card; no A2A task endpoint
- `/.well-known/ai-plugin.json`: OpenAPI import manifest
- `/ai-profile.json`, `/ai.txt`, `/llms.txt`, `/llms-full.txt`: discovery guides
- `/mcp.json`: compatibility notice; no MCP transport is exposed

If an integration can import OpenAPI, prefer the `operationId`-based contract
over scraping this document.
