---
name: Curate a custom provider directory
description: Correct and extend a customer-scoped H1 provider directory — edit provider and location fields, attach specialties, insurances and organizations, and register custom or boost filters so the new data becomes searchable.
api: openapi/h1-insights-openapi-original.json
base_url: https://api.ribbonhealth.com/v1
operations:
  - putCustomProvider
  - putCustomProviderLocations
  - putCustomProviderLocation
  - putCustomProviderSpecialties
  - putCustomProviderLocationInsurances
  - postCustomLocations
  - putCustomLocation
  - getCustomProviderFilters
  - createCustomProviderFilter
  - editCustomProviderFilter
  - deleteCustomProviderFilter
  - getCustomProviders
generated: '2026-08-04'
method: generated
source: openapi/h1-insights-openapi-original.json + https://ribbon.readme.io/docs/create-a-custom-filter
---

# Curate a custom provider directory

Write access is scoped to the **customer's own** directory — it never edits H1's base data and never
affects another customer. It requires the Upgraded API package; Base and trial accounts get
`403 permission_denied`.

## The two-step that people miss

Adding a custom field to providers does **not** make it searchable. You must then register a filter for
it. Data first, filter second, search third.

## Steps

1. **Edit the provider record.** `putCustomProvider` (`PUT /custom/providers/{npi}`) edits every field
   that is *not* `specialties`, `locations` or `insurances`, and can add new custom fields or remove
   existing ones. Those three collections have dedicated sub-resource endpoints instead:
   - `putCustomProviderLocations` (`PUT /custom/providers/{npi}/locations`) — add/remove locations by
     H1 location UUID
   - `putCustomProviderSpecialties` (`PUT /custom/providers/{npi}/specialties`) — add/remove by
     specialty UUID
   - `putCustomProviderPrimarySpecialties`
     (`PUT /custom/providers/{npi}/specialties/{specialty_uuid}`) — mark one as primary
   - `putCustomProviderProcedures`, `putCustomProviderClinicalAreas` — same shape
2. **Get insurance participation right.** Insurance is attached to a provider **at a location**, not to
   the provider: `putCustomProviderLocationInsurances`
   (`PUT /custom/providers/{npi}/locations/{location_uuid}/insurances`). Attaching an insurance to the
   provider globally is not a thing this API models. `putCustomProviderLocation`
   (`PUT /custom/providers/{npi}/locations/{location_uuid}`) edits that provider's view of the location
   without affecting other providers who practise there.
3. **Add a location that does not exist yet.** `postCustomLocations` (`POST /custom/locations`), then
   `putCustomLocation` for non-geographic fields. `address` + `name` is the natural unique key: a
   duplicate returns `409 conflict`, which is the API's substitute for an idempotency key. On `409`,
   search for the existing location and use it rather than retrying the create.
4. **Register the filter.** `createCustomProviderFilter` (`POST /custom/providers/filters`) turns a field
   — H1's or one you added — into a query parameter on `getCustomProviders`. A **custom filter**
   restricts the result set; a **boost filter** re-ranks it without excluding anyone. Choose deliberately:
   boosting is usually right for preference, filtering for hard requirements. `getCustomProviderFilters`
   lists what already exists, `editCustomProviderFilter` changes one, `deleteCustomProviderFilter`
   removes one. `409 conflict` means a filter with that parameter name already exists — read it before
   creating another.
5. **Verify.** Call `getCustomProviders` with the new parameter and confirm it filters or ranks as
   intended.

The same pattern exists for locations: `getCustomLocationFilters`, `createCustomLocationFilter`,
`editCustomLocationFilter`, `deleteCustomLocationFilter`.

## Write safety

- There is **no** `Idempotency-Key` header on this API. Uniqueness is enforced server-side with
  `409 conflict` on natural keys (location `address`+`name`, filter `parameter`, type `display_name`).
  **Never blind-retry a `POST` after a timeout** — re-read the collection first and only create if absent.
- `PUT` and `DELETE` here address a specific resource and are idempotent by HTTP method; retrying them is
  safe.
- For bulk edits, send `async=true` to apply edits asynchronously (documented under Latency). The call
  returns before the edit is visible — poll the read endpoint rather than assuming immediate consistency.
- H1-managed reference resources cannot be modified or deleted: `403 permission_denied` with
  "This resource is managed by Ribbon". Only customer-created resources are writable.

## Errors

Vendor envelope `{"error":{"status","code","message"}}`.
`400` — attempted to update fields that cannot be customised, used an unsupported field, sent an invalid
value type, or passed something that is not a valid UUID.
`404` — unknown NPI, location UUID, filter UUID, or an NPI that is not in the custom directory.
`409` — the resource already exists on its natural key.
