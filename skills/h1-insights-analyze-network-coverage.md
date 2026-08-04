---
name: Analyze network coverage and sites of care
description: Use H1's location, organization, focus-area and network-analysis endpoints to understand what care is available in a geography and how a provider network is composed.
api: openapi/h1-insights-openapi-original.json
base_url: https://api.ribbonhealth.com/v1
operations:
  - getCustomLocationTypes
  - getCustomLocations
  - getCustomLocation
  - getOrganizations
  - getOrganization
  - getClinicalAreas
  - getConditions
  - getTreatments
  - getCustomVirtualCarePlatforms
  - getNetworkAnalysis
  - getLanguages
generated: '2026-08-04'
method: generated
source: openapi/h1-insights-openapi-original.json + https://ribbon.readme.io/docs/network-data
---

# Analyze network coverage and sites of care

Provider search answers "who". This flow answers "where, run by whom, and how thin is the network".

## Steps

1. **Resolve the site type.** `getCustomLocationTypes` (`GET /location_types`) returns the location-type
   taxonomy (urgent care, lab, imaging centre, therapy centre, and so on). Take the UUID.
2. **Search locations.** `getCustomLocations` (`GET /custom/locations`) by geography, insurance and
   location type. `getCustomLocation` (`GET /custom/locations/{location_uuid}`) for one record. Note that
   insurance participation on a location is a property of the location, and separately a property of a
   provider *at* that location — the two are not the same question.
3. **Roll up to the operator.** `getOrganizations` (`GET /custom/organizations`) finds health systems and
   groups; `getOrganization` (`GET /custom/organizations/{organization_uuid}`) fetches one. Use this to
   map a fragmented list of sites back to the handful of systems that actually run them.
4. **Scope clinically.** Focus areas are a three-level surface:
   `getClinicalAreas` (`GET /custom/clinical_areas`) → `getConditions` (`GET /custom/conditions`) →
   `getTreatments` (`GET /custom/treatments`). Each has a singular fetch-by-UUID sibling. Use these UUIDs
   as filters on provider and location search rather than free-text specialty matching when the question
   is condition-shaped ("who treats X") rather than credential-shaped ("who is a dermatologist").
5. **Include virtual care.** `getCustomVirtualCarePlatforms` (`GET /custom/virtual_care_platforms`) and
   `getVirtualCarePlatform` cover telehealth options, which will not appear in a distance-bounded
   geographic search.
6. **Measure the network.** `getNetworkAnalysis` (`GET /network_analysis`) returns a provider network
   viewed across geographies (counties, by SSA code). This is the adequacy/expansion question: where is
   the network thin, where is it dense, how does it compare region to region.
7. **Check language access.** `getLanguages` (`GET /languages`) lists provider languages; feed the result
   back as a filter on provider search when language concordance matters.

## Constraints

- `getNetworkAnalysis` returns `400` when required parameters are missing or **too many SSA codes** are
  specified in one request. Chunk large geographies into multiple calls rather than widening one.
- Reference endpoints accept one search dimension per request — combining them returns
  `400 bad_request` with "Each request can only perform one search".
- Every operation here is a `GET` and safe to retry on transport errors. No idempotency key exists and no
  rate limit is documented, so back off politely on your own initiative.

## Errors

Vendor envelope `{"error":{"status","code","message"}}`. `403 permission_denied` on the `custom/*`
endpoints means the account's package does not include custom directories — trial accounts never do.
