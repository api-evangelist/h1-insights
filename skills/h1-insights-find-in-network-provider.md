---
name: Find an in-network provider
description: Search the H1 provider directory for clinicians who match a patient's geography, insurance network, specialty and quality bar, then pull the full record for the chosen provider.
api: openapi/h1-insights-openapi-original.json
base_url: https://api.ribbonhealth.com/v1
operations:
  - getInsurances
  - getSpecialties
  - getCustomProviders
  - getCustomProvider
generated: '2026-08-04'
method: generated
source: openapi/h1-insights-openapi-original.json + https://ribbon.readme.io/docs/provider-search
---

# Find an in-network provider

The H1 (Ribbon Health) directory is keyed on the CMS **NPI**. Everything else — insurance, specialty,
location, organization — is an H1-issued UUID you must resolve first. Never guess a UUID.

## Before you start

- Authenticate every request with `Authorization: Bearer {customer_token}`. There are no scopes; the
  account's package decides which endpoints answer. A `403 permission_denied` means the package does not
  include that surface (trial accounts have no access to custom directories at all) — it is not a
  retryable error.
- Read `conventions/h1-insights-conventions.yml` for pagination and field-filtering rules before you
  build a request.

## Steps

1. **Resolve the insurance network.** Call `getInsurances` (`GET /custom/insurances`) with a search term
   for the member's carrier and plan. Take the `uuid`. Insurance participation in this API is recorded
   per **provider-at-a-location**, not per provider, so this UUID is what makes "in-network" meaningful.
2. **Resolve the specialty.** Call `getSpecialties` (`GET /custom/specialties`) for the clinical
   specialty. Take the `uuid`. If you only know a broad category, use the `provider_types` grouping on the
   search instead (Doctor, Nursing, Dental Providers, Optometry, Chiropractic Providers).
3. **Search.** Call `getCustomProviders` (`GET /custom/providers`) combining:
   - geography — `address` (free text or ZIP) or `location`, plus `distance`
   - network — the insurance UUID from step 1
   - clinical — the specialty UUID from step 2, or `clinical_areas` / `conditions` / `treatments`
   - demographics — `gender`, `min_age`/`max_age`, languages
   - every positive filter has a negative twin prefixed `_excl_` (e.g. `_excl_provider_types`)
4. **Keep the response small.** Provider objects can carry tens of thousands of rows across `locations`
   and `insurances`. Always send `fields` (include list) or `_excl_fields` (exclude list) — never both.
   The docs name this as the single biggest latency lever.
5. **Page correctly.** Use `page` and `page_size` (default 25, generally capped at 200). `max_locations`
   caps the locations attached to each provider (default 5). The hard constraint is
   `max_locations * page_size <= 1000` — violating it returns `400`, so compute it before sending.
6. **Read the confidence score before you act.** Each provider's contact information carries a 1–5
   confidence score: 5 = manually verified within 90 days, 4 ≈ 90% accurate, 3 ≈ 70%, 2 ≈ 50%
   (industry average), 1 ≈ 20%. Do not surface a phone number or address scored 1–2 to a patient as
   though it were verified; prefer a higher-scored location or say the contact detail is unconfirmed.
7. **Fetch the full record.** Call `getCustomProvider` (`GET /custom/providers/{npi}`) with the chosen
   NPI for the complete profile: all locations, contact details, education, specialties, insurance
   participation and patient-satisfaction data.

## Constraints and gotchas

- `npis` (the exact-lookup parameter on `getCustomProviders`) accepts at most **100** NPIs and cannot be
  combined with most other parameters. `fields`/`_excl_fields` are allowed; `location`/`address` are
  accepted but only populate `distance` — they do **not** filter. Every other parameter is silently
  ignored. If you need filtering, do not use `npis`.
- Reference endpoints accept **one** search dimension per request; combining them returns
  `400 bad_request` with `"Each request can only perform one search"`.

## Errors

Errors are a vendor envelope, not RFC 9457:
`{"error": {"status": 404, "code": "not_found", "message": "resource not found"}}`

- `400 bad_request` / `invalid_query_params` — bad parameter combination, wrong type, malformed UUID, or
  the `max_locations * page_size` constraint breached. Fix the request; do not retry unchanged.
- `403 permission_denied` — outside the account's package. Stop and report; retrying will not help.
- `404 not_found` — unknown NPI or UUID.

There is **no** `Idempotency-Key` on this API and **no** documented rate limit or `429`. All operations in
this skill are `GET` and therefore safe to retry on transport errors.
