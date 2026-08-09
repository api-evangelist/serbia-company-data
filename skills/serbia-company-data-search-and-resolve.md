---
name: Search Serbian companies by name and resolve them in bulk
description: Find Serbian companies by business name in Latin or Cyrillic, then resolve the matched registration numbers into full profiles in one batched, cheaper call.
api: openapi/serbia-company-data-openapi.json
operations:
  - searchSerbianCompanies
  - batchGetSerbianCompanies
generated: '2026-08-09'
method: generated
source: openapi/serbia-company-data-openapi.json + live probes 2026-08-09
---

# Search Serbian companies by name and resolve them in bulk

The name-to-profile flow. Use it when you hold company names rather than registration numbers — for
example enriching a CRM list or reconciling counterparties against the official Serbian registry.

## Cost shape — this is why you batch

- `searchSerbianCompanies` — $0.01 USDC per query
- `batchGetSerbianCompanies` — $0.05 USDC for **up to 10** companies

Resolving ten companies one at a time with `getSerbianCompany` costs $0.10. One batch call costs $0.05.
Always chunk into tens.

## Steps

1. **Search.** `searchSerbianCompanies` — `GET /api/search?q=air%20serbia&limit=5`. `q` is 3-100
   characters and accepts Serbian Latin *or* Cyrillic. `limit` is 1-10, default 5.
2. **Pay the 402.** Same x402 flow as every priced route: decode the `PAYMENT-REQUIRED` header, sign,
   re-send with `PAYMENT-SIGNATURE`. See `authentication/serbia-company-data-authentication.yml`.
3. **Pick your matches.** The 200 body is `{ query, count, results[] }`, ranked. Each result already
   carries `registrationNumber`, `businessName`, `status`, `incorporationDate`, `legalForm` and
   `activityCode` — for many use cases you can stop here and never call the batch route at all.
4. **Chunk the registration numbers into groups of 10.**
5. **Resolve.** `batchGetSerbianCompanies` — `POST /api/company/batch` with
   `{"registrationNumbers":["07044275","20047216"]}`. Max 10 items, each matching `^\d{8}$`.
6. **Read `results[]`.** Each entry is `{ registrationNumber, company: { ... } }`.

## Rules that will bite you

- **There is no pagination.** `limit` caps at 10 and there is no cursor or offset. If a name is too
  generic you cannot page past the tenth match — narrow the query instead.
- **Validate before you pay.** `q` shorter than 3 characters still returns `402`, not `400`. Enforce
  the length and the `^\d{8}$` pattern locally.
- **No idempotency.** Every retry is a new charge. Persist results keyed on `registrationNumber`.
- **Match on the registration number, never on the name.** `businessName` is the full registered legal
  name as APR holds it (e.g. *"Akcionarsko društvo za vazdušni saobraćaj Air SERBIA Beograd"*), not a
  trading name, and may come back Cyrillic.
- **Check `status` before you act.** A company can be in the snapshot and not be active.
- **Financials are latest-year only and in `thousand_RSD`.** There is no history endpoint.

## Related artifacts

- `data-model/serbia-company-data-data-model.yml` — the entity graph and the AOP measure codes
- `conventions/serbia-company-data-conventions.yml` — limits, envelopes, units
- `sandbox/serbia-company-data-sandbox.yml` — the free `/api/sample` route and published example IDs
