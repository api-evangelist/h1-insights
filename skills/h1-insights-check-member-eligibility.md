---
name: Check member eligibility and benefits
description: Verify a member's active coverage and pull their deductible, out-of-pocket, copay and coinsurance position from H1's real-time eligibility endpoint.
api: openapi/h1-insights-openapi-original.json
base_url: https://api.ribbonhealth.com/v1
operations:
  - getEligibilityInsurancePartners
  - getEligibilityInsurancePartner
  - getEligibility
generated: '2026-08-04'
method: generated
source: openapi/h1-insights-openapi-original.json + https://ribbon.readme.io/docs/eligibility
---

# Check member eligibility and benefits

This is the most sensitive operation on the H1 API. The request carries member identifiers and the
response carries coverage and benefit detail — **PHI under HIPAA**. Treat it accordingly.

Eligibility is an **add-on** to Base API Access; an unentitled account gets `403 permission_denied`.

## Steps

1. **Confirm the payer is supported.** `getEligibilityInsurancePartners`
   (`GET /eligibility_insurance_partners`) lists every insurance partner supported by the eligibility
   feature; `getEligibilityInsurancePartner` (`GET /eligibility_insurance_partners/{insurance_partner}`)
   fetches one. Not every carrier in the directory is an eligibility partner — check first rather than
   letting the check fail.
2. **Run the check.** `getEligibility` (`POST /eligibility`) with the member's identifiers and the
   insurance partner. Note this is the one **POST** in this flow even though it reads rather than writes.
3. **Read the benefit position.** The response carries the member's progress against **deductible** and
   **out-of-pocket maximum**, plus **copay** and **coinsurance** summaries by service type.

## Rules for an agent

- Do not log, cache, echo into a transcript, or pass to another tool any member identifier or benefit
  detail from this flow unless the calling application has explicitly established a HIPAA-permitted
  purpose. Return the minimum needed to answer the question asked.
- The result is a point-in-time coverage snapshot from the payer, not a payment guarantee and not an
  adjudicated claim. Say so when you present it.
- Combine with a negotiated rate (see `h1-insights-price-a-procedure.md`) to estimate member
  out-of-pocket cost; neither number alone answers "what will this cost me".
- `POST /eligibility` has **no** `Idempotency-Key`. It is a read-shaped POST, so a retry is not
  destructive, but retry only on transport failure — never on a `4xx`.

## Errors

Vendor envelope `{"error":{"status","code","message"}}`.
`404 not_found` on `getEligibilityInsurancePartner` means the partner identifier is unknown.
`403 permission_denied` means the eligibility add-on is not on the account.

## Supply chain

H1's own status page names **pVerify** as the provider powering eligibility checks. If eligibility is
degraded while the rest of the API is healthy, check https://status.ribbonhealth.com/ before assuming a
client-side fault.
