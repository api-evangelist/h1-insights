---
name: Price a procedure for a member
description: Use H1's price-transparency and cost-estimate endpoints to find who performs a procedure, what each provider's insurer-negotiated rate is, and what a member should expect to pay.
api: openapi/h1-insights-openapi-original.json
base_url: https://api.ribbonhealth.com/v1
operations:
  - getProcedures
  - getInsurances
  - getPricingProviders
  - getPricingProviderProcedures
  - getPricingProviderProcedure
  - getPricingProviderProcedureLocation
  - getPricingCarriers
  - getPricingCarrier
  - getProcedureCostEstimate
generated: '2026-08-04'
method: generated
source: openapi/h1-insights-openapi-original.json + https://ribbon.readme.io/docs/cost-quality
---

# Price a procedure for a member

Price Transparency is an **add-on**, not part of Base API Access. If the account is not entitled you get
`403 permission_denied` on every `/pricing/*` call — report that rather than retrying.

## Steps

1. **Resolve the procedure.** `getProcedures` (`GET /procedures`) to turn a plain-language procedure name
   into a `procedure_uuid`. `getProcedure` (`GET /procedures/{procedure_uuid}`) confirms one.
2. **Resolve the insurance.** `getInsurances` (`GET /custom/insurances`) — negotiated rates only exist
   relative to a specific carrier/plan.
3. **Search the market.** `getPricingProviders` (`GET /pricing/providers`) with the procedure UUID, the
   insurance and optionally an address/distance. Returns providers who perform it, ordered so the lowest
   available negotiated rate in the area surfaces first.
4. **Widen or narrow from one provider:**
   - `getPricingProviderProcedures` (`GET /pricing/providers/{npi}/procedures`) — every procedure that
     provider is likely to perform with a negotiated rate for the given insurance, minimum price each.
   - `getPricingProviderProcedure` (`GET /pricing/providers/{npi}/procedures/{procedure_uuid}`) — that
     provider's price for one procedure **across all their practice locations**. This is where site-of-care
     variation shows up: the same clinician often costs materially more at a hospital outpatient
     department than at their own office.
   - `getPricingProviderProcedureLocation`
     (`GET /pricing/providers/{npi}/procedures/{procedure_uuid}/locations/{location_uuid}`) — the single
     provider + procedure + location + plan price.
5. **Check data recency before quoting.** Negotiated-rate files go stale. `getPricingCarriers`
   (`GET /pricing/carriers`) lists the carriers H1 holds data for; `getPricingCarrier`
   (`GET /pricing/carrier/{carrier_uuid}`) returns that carrier's data recency. Always state the vintage
   alongside a price. Do **not** use `getPricingCarrierNames` or `getPricingVersionCarrier` — both are
   deprecated in the spec in favour of the two above.
6. **Estimate member cost.** `getProcedureCostEstimate` (`GET /procedure_cost_estimate`) computes an
   estimated cost for a procedure from the user's location. This is an estimate, not an adjudicated claim.

## How to present the answer

A negotiated rate is what the insurer pays the provider — it is **not** the member's out-of-pocket cost.
To get to member cost you must combine it with the member's benefit position from the eligibility flow
(see `h1-insights-check-member-eligibility.md`): deductible progress, out-of-pocket progress, copay and
coinsurance. Never present a negotiated rate to a patient as "what you will pay".

## Errors

Vendor envelope `{"error":{"status","code","message"}}`.
`400 bad_request` on this surface most often means the search combined parameters that may not be
combined, did not specify a procedure in any way, or named an insurance/procedure that could not be
found. All operations here are `GET` and safe to retry on transport errors; there is no idempotency key
and no documented rate limit.
