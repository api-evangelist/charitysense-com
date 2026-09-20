---
generated: '2026-09-19'
method: generated
name: Research a U.S. charity from a name or EIN
description: Resolve a nonprofit to its EIN, load its bounded profile, and fetch only the sections the profile advertises - all without an API key.
api: openapi/charitysense-com-openapi.yml
operations: [searchOrganizations, getCharityPage, getCharitySection, getCharityFilings]
tier: public
source: >-
  Grounded in openapi/charitysense-com-openapi.yml (operationIds verified verbatim) and the provider's
  "Reliable research workflow" in https://data.charitysense.com/INSTRUCTIONS_FOR_AGENTS.md.
---

# Research a U.S. charity from a name or EIN

Base URL `https://data.charitysense.com`. All four operations are public research reads: no credential, shared
allowance of 1,000 requests per UTC day per client (`RateLimit-Limit` / `RateLimit-Remaining` / `RateLimit-Reset`
on every response). Fields are UpperCamelCase and documents are sparse - an absent field means "not present in the
filing", never false or zero.

## Auth
- None for this flow. See `authentication/charitysense-com-authentication.yml` for the paid tier.

## Steps
1. **Resolve the organization** - `searchOrganizations` (`GET /api/v2/search?Query=<name or EIN>&Mode=Identity`).
   `Query` is required. Use `Mode=Identity` for names, EINs, aliases and acronyms; `Mode=Discovery` for causes,
   missions, geography or donor intent (add `EvidenceTier`, `State`, `Cause`, `MinRevenue`, `MaxRevenue`, `Sort`).
   `Cursor` is a one-based page number and `Limit` is 1-50. Take `Hits[].EIN` (nine digits, no punctuation) and
   keep `Hits[].Match` to explain why it matched.
2. **Load the bounded profile** - `getCharityPage` (`GET /api/v2/charity/{ein}/page`). Read
   `Context.AvailableForms` before choosing a `Form` (`990`, `990EZ`, `990N`, `990PF`, `990T`) and `Year`; the
   defaults are the primary form and its latest year. `Hero` carries identity and `EvidenceRefs`; `Verdict` is the
   live rating; `Sections` is the only valid inventory of what can be fetched next.
3. **Fetch only advertised sections** - `getCharitySection` (`GET /api/v2/charity/{ein}/sections/{SectionId}`)
   with an `Id` taken from `Sections[]`. Pass the same `Form`/`Year`. Paginated sections return `NextCursor`; pass
   it back as `Cursor` (opaque, route-specific) until it is absent. Pass `EvidenceRef` only when an earlier
   response supplied that exact value. The five synthesized sections (`WhatTheyDo`, `People`, `GrantReadiness`,
   `MoneyNetwork`, `RelatedOrganizations`) may be requested even when not advertised; a `404 SectionNotAdvertised`
   there is an ordinary absence.
4. **List filing history when needed** - `getCharityFilings` (`GET /api/v2/charity/{ein}/filings?Form=`), opaque
   `Cursor`, `Limit` 1-100. Items are filing metadata, not section content.
5. **Cite** - preserve `EvidenceRefs`, `SourcePath`, form and filing year, and cite
   `https://data.charitysense.com/charity/{ein}` with the filing year for any financial figure.

## Errors
- `422` (validation array in `detail`) for a missing `Query` or bad enum; `404 ProfileNotFound` for an unknown EIN.
  Never retry `400/401/404/422` unchanged; back off on `429` and `503`. Catalog: `errors/charitysense-com-problem-types.yml`.

## Notes
- Figures are historical IRS Form 990 data and can lag current operations; a `990T` context covers unrelated
  business income only, and a `990N` context usually has too little detail to rate. Say so in the answer.
- No idempotency or reversibility concerns: the flow is read-only (`conventions/charitysense-com-conventions.yml`).
