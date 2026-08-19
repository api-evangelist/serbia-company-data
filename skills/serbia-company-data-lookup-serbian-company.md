---
name: Look up a Serbian company by registration number
description: Resolve one official Serbian company profile — registry status, legal form, activity code, municipality and the latest public financial summary — from its 8-digit registration number, paying per call with x402 on Base.
api: openapi/_original/serbia-company-data-openapi.json
operations:
  - getSerbianCompany
generated: '2026-08-09'
method: generated
source: openapi/_original/serbia-company-data-openapi.json + live probes 2026-08-09
---

# Look up a Serbian company by registration number

Use this when you already hold an 8-digit Serbian registration number (maticni broj / MB). If you only
have a name, run the search skill first and take `registrationNumber` from the result.

## Before you call

There is no account, no signup and no API key. Access is economic, not identity-based: the call costs
**$0.01 USDC on Base** and is settled with x402. You need an x402-compatible buyer client with a funded
Base wallet — see the [buyer quickstart](https://docs.x402.org/getting-started/quickstart-for-buyers).

If you have not paid before and just want to see the shape of a response, call the free
`GET /api/sample` route instead. It costs nothing and returns a complete worked example.

## Steps

1. **Validate the input yourself.** `mb` must match `^\d{8}$`. Do this locally — the payment gate fires
   *before* input validation, so `?mb=abc` returns `402`, not `400`. A malformed request will cost you
   money to discover.
2. **Call the operation.** `getSerbianCompany` — `GET /api/company?mb=07044275` on
   `https://serbia-company-x402.vercel.app`.
3. **Expect a 402 on the first attempt.** The response body is `{}`; the contract is in the base64
   `PAYMENT-REQUIRED` header. Decode it and read `accepts[0]`: `scheme` `exact`, `network`
   `eip155:8453` (Base), `amount` in USDC base units (`10000` = $0.01), `asset`, `payTo`,
   `maxTimeoutSeconds` 300.
4. **Re-send with payment.** Put the signed payload in the `PAYMENT-SIGNATURE` header. Settlement comes
   back in `PAYMENT-RESPONSE`.
5. **Read the profile.** The 200 body is `{ "company": { ... } }` with `registrationNumber`,
   `businessName`, `status`, `incorporationDate`, `legalForm`, `activityCode`,
   `latestFinancialStatement`, and a `municipality` object that the declared schema does not mention.

## Rules that will bite you

- **Retries are not free.** There is no idempotency key. Every retry is a new billable payment. Cache
  the result against the registration number; the underlying data is a monthly snapshot, so a cached
  profile stays current for weeks.
- **All money is `thousand_RSD`.** Every value under `latestFinancialStatement.amounts` is in
  *thousands* of Serbian dinars and carries its own `unit` plus the APR `aop` line code. Do not treat
  `totalRevenue.value` as dinars.
- **Text comes back in Serbian Cyrillic.** `status` (e.g. `Активан`), `legalForm` and
  `municipality.name` are as APR publishes them. Do not string-match against Latin transliterations.
- **Absence is not proof.** The dataset is a point-in-time snapshot (registry as of 2026-06-30,
  133,802 companies) and excludes sole proprietors. A miss means "not in this snapshot", not "does not
  exist".
- **Undeclared failure modes.** Only `200` and `402` are documented. Handle `404` (`{"error":"not_found"}`)
  and transient `5xx` defensively; errors are a bare JSON object, not RFC 9457 problem+json.

## Related artifacts

- `authentication/serbia-company-data-authentication.yml` — the full x402 profile
- `conventions/serbia-company-data-conventions.yml` — units, envelopes, caching
- `errors/serbia-company-data-problem-types.yml` — what can go wrong
- `examples/serbia-company-data-402-payment-required.json` — the real decoded challenge
