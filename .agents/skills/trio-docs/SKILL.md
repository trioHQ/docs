---
name: trio-docs
description: Help customers implement the Trio Banking API V2 — auth, cash-in, cash-out, webhooks, and reconciliation in any stack.
---

# Trio Docs — Implement the Trio API (V2 only)

Use this skill when a customer asks to integrate, debug, or extend a Trio integration: creating charges, paying out, handling webhooks, reconciling documents, onboarding customers, or moving an integration from sandbox to production.

Source of truth is the V2 documentation in this repo under `v2/` and the V2 OpenAPI spec at `v2/api-reference/openapi.json` (106 paths, `Trio API v2`, version `2026-10-19.v2`). Ignore V1 completely: never use V1 paths, V1 field names, or `api-reference/` (non-`v2/`) pages. If a V2 page contains a `TODO(v2): verify` note, flag the value as unverified instead of inventing it.

## Non-negotiable V2 rules

- Every path starts with `/v2`. There are no virtual accounts in V2; the receiving or paying account is always `bank_account_id`.
- Amounts are integer cents: BRL 100.00 is `10000`. Money objects are `{ "amount": 10000, "currency": "BRL" }`.
- Timestamps are UTC ISO 8601 (`2026-10-07T20:16:02.471297Z`); plain dates are `YYYY-MM-DD`. See `v2/developers/date-and-time.mdx`.
- Branch on `error.error_code`, never on the human message. See `v2/developers/error-table.mdx`.
- `external_id` is the reconciliation key. It must be unique per operation (use a UUID), is enforced for 24 hours, and must be persisted with every identifier Trio returns.
- Webhooks are the source of truth for final status. Polling is a fallback, not the design.
- Never expose `client_secret`, `notification_secret`, or `notification_key` in front-end code, logs, or the Trio Checkout session creation. Sessions and API calls that use secrets happen in the back end only.

## Environments and base URLs

| Environment | Base URL | Notes |
| --- | --- | --- |
| Sandbox | `https://api.sandbox.trio.com.br` | Free, isolated, no real money. App: `https://app.sandbox.trio.com.br` |
| Production | `https://api.trio.com.br` | Real data. App: `https://app.trio.com.br` |

Keep the base URL in environment config, never hard-coded per call. All examples in the V2 docs use the sandbox URL; switching to production is a config change plus new production credentials. See `v2/developers/environments.mdx` and `v2/guides/sandbox_account.mdx`.

## Authentication — every request

Auth is HTTP Basic: `client_id` is the username, `client_secret` is the password. Create the pair in the Trio App (Settings → API Keys → Create Key) with Read and Write or Read-Only permission and an allowed-IP list; store the secret immediately, it is shown once. See `v2/guides/creating-your-first-api-credential.mdx` and `v2/developers/authentication.mdx`.

Every request sends:

- `Content-Type: application/json` when it has a body.
- `Trio-Version` with the current API version. A missing or outdated value is refused with `400 INVALID_TRIO_VERSION`. The V2 docs do not state the literal current value — ask the customer to confirm it from their Trio contact or dashboard instead of guessing.
- Basic credentials via `-u {client_id}:{client_secret}`.

```bash
curl https://api.sandbox.trio.com.br/v2/entities \
  -H "Content-Type: application/json" \
  -H "Trio-Version: {trio_version}" \
  -u {client_id}:{client_secret}
```

Failures: `401 INVALID_CREDENTIALS` for missing/invalid credentials; `403 FORBIDDEN` when the key may not perform the operation (for example a read-only key writing). Read `X-Request-Id` from every response and quote it with the `error_code` when contacting support (`v2/developers/request-identification.mdx`, `v2/getting_started/support.mdx`).

### Acting on another organization (distributors)

Send `Trio-Target-Org-Id: {org_id}` to operate on a customer organization with your own credentials. Access is granted per organization by Trio (typically automatically for organizations you create via onboarding); without it the request returns `403 FORBIDDEN`. Read-only access allows only `GET`/`HEAD`. See the header rules in `v2/developers/authentication.mdx` and the flow in `v2/guides/onboarding-guide.mdx`.

## Core objects

Read `v2/developers/concepts.mdx` before designing data models:

- **Company** — top-level business; holds users, API logs, and one or more entities.
- **Entity** — a Brazilian legal entity with its own CNPJ. Status flows `created → in_validation → in_review → approved` (or `rejected` / `onhold` / `canceled`). Endpoints: `GET /v2/entities`, `GET /v2/entities/{id}`.
- **Group** — organizes the bank accounts of an entity; `group_id` scopes account creation, listings, and webhooks.
- **Bank account** — holds and moves money; reports `balances`, per-type `limits`, and `parameters` (auto-block/refund rules). Category is `linked`, `owner`, `transactional`, or `operational`. Endpoints: `GET/POST /v2/bank_accounts`, `GET/PUT /v2/bank_accounts/{id}`, `GET /v2/bank_accounts/{id}/balances`, `GET /v2/bank_accounts/{id}/transactions`, `GET /v2/bank_accounts/{id}/metrics`.
- **Counterparty** — the other party (payer or receiver) with `tax_number`, name, and bank accounts; also used as split recipient. Endpoints under `/v2/counterparties`, `/v2/counterparties/{id}/bank_accounts`, `/v2/counterparties/{id}/recipients`, and `/v2/counterparty_recipients`.
- **Document** — every money movement. An **in document** is money received, an **out document** is money paid, a **transfer document** moves money between your accounts (`from_bank_account_id` / `to_bank_account_id`), and refunds are `in_document_refund` / `out_document_refund`. Read them under `/v2/documents/in`, `/v2/documents/out`, `/v2/documents/transfer` (plus `/search` and `/refunds` variants).

Institutions: `GET /v2/institutions`.

## Pagination, errors, rate limits

Pagination is cursor-based (`v2/developers/pagination.mdx`). List responses return `{ data, metadata: { before, after, has_more } }`. Pass `after` for the next page or `before` for the previous one (never both). `limit` defaults to 50, max 100. Document listings (`GET /v2/documents/in`, `/out`, refunds) additionally require `from_datetime` and `to_datetime`. Always loop while `has_more` is true before aggregating totals.

Error body (`v2/developers/error-table.mdx`):

```json
{
  "error": {
    "error_code": "INVALID_PARAMETER",
    "error_message": "Invalid parameters: /amount: Invalid integer. Got: string",
    "errors": [{ "field": "/amount", "message": "Invalid integer. Got: string" }]
  }
}
```

| Status | Meaning | Error codes |
| --- | --- | --- |
| 200 / 201 / 204 | OK / created / empty-body success (updates, cancellations) | — |
| 400 | Malformed JSON or bad `Trio-Version` | `MALFORMED_REQUEST`, `INVALID_TRIO_VERSION` |
| 401 | Bad credentials | `INVALID_CREDENTIALS` |
| 403 | Key or target-org not allowed | `FORBIDDEN` |
| 404 | Unknown ID | `NOT_FOUND_RESOURCE` |
| 422 | Validation or business-rule failure; `errors[]` lists fields for `INVALID_PARAMETER` | `INVALID_PARAMETER`, `REQUIRED_PARAMETER`, plus business codes such as `PAYMENT_ACCOUNT_WITHOUT_BALANCE`, `PIX_KEY_ALREADY_EXISTS`, `KEY_NOT_FOUND` |
| 429 | Rate limited | back off (below) |
| 503 | Dependency unavailable | `TRIO_SERVICE_UNAVAILABLE` |

Rate limits (`v2/developers/rate-limit.mdx`, kept from V1 — treat as unverified against the V2 spec): 60 req/s production, 5 req/s sandbox, per account and per endpoint. On `429`, retry with exponential backoff and spread queued requests evenly; contact support for a raise instead of bursting.

## Cash-in — receive money (`/v2/in/...`)

Every receivable is created against a `bank_account_id` and becomes an in document when paid. Start from `v2/guides/receivables/overview.mdx` and `v2/guides/receivables/cash-in.mdx`.

### Pix QR codes

| Type | Create | Use when |
| --- | --- | --- |
| Dynamic | `POST /v2/in/pix/qrcodes` | One charge, one code; automated reconciliation |
| Static | `POST /v2/in/pix/qrcodes/static` | Reusable code (counter, flyer); send `amount: 0` for open amount |
| Dynamic with due date | `POST /v2/in/pix/qrcodes/with_due` | Invoice-like charge with due date, fine, interest, rebate, discount |

Dynamic QR request (`v2/guides/pix/qr-codes.mdx`): required `bank_account_id`, `amount` (cents), `options.allow_change_the_amount_on_payment`, `options.show_qrcode_image`; optional `counterparty { tax_number, name }`, `external_id`, `description`, `expiration_datetime`, `options.fee`, `splits`. Response `201` returns `data.id` (the QR code ID, the webhook `ref_id`), `status: created`, `hash` (the BR code / Pix copia-e-cola to share), `amount`, `fee`, `counterparty.id`, `origin { id, type }`, `inserted_at` / `updated_at`. Fetch the image later with `GET /v2/in/pix/qrcodes/{id}/image` (Base64) or the hash with `GET .../hash`; update amount/expiration/fee with `PUT /v2/in/pix/qrcodes/{id}` (`204`). Static and with-due variants mirror this under `/static/{id}` and `/with_due/{id}`.

Critical: `data.id` is only the QR code. The final in-document `id` exists only after settlement and arrives in the `in_document.settled` webhook (its `data.reference_id` / `reference.id` points back to the QR code). Never reconcile cash-in on the QR ID alone. See `v2/operations/best-practices.mdx`.

### Boletos (receivables)

Create with `POST /v2/in/boletos` (`v2/guides/boleto/create-boleto.mdx`). Required: `amount`, `bank_account_id`, `invoice_type`, `wallet_type`, `expiration_date`, full `counterparty` (name, tax_number, address, city, state, district, postal_code), and `due_detail.due_date`. Optional: `description` (≤25 chars), `external_id` (≤40 chars), `notes`, `splits`, `options`, `merchant`, and `due_detail` rules. Rules convention: monetary rules send `amount` in cents; percentage rules send `value` in basis points (100 = 1%, so 500 = 5%). Discount is either `fixed_until_due` (list of `{ date, amount|value }`) or `per_day_antecipated` — never both. Manage with `GET /v2/in/boletos`, `GET/PUT /v2/in/boletos/{id}`, `POST .../cancel`, `POST .../recreate`, `PUT .../protest/add`, `PUT .../protest/remove`. Types are listed in `v2/guides/boleto/boleto-types.mdx`; practices (hybrid boleto+Pix, due date vs expiration) in `v2/guides/boleto/best-practices.mdx`.

### Pix keys, refunds, splits, fees

- Pix keys on your accounts: `POST/GET /v2/pix/keys`, `GET /v2/pix/keys/{id}`, `PUT .../confirm`, `POST .../resend_token`, `POST .../cancel`. Key types and validation regexes (email, phone `+...`, CPF/CNPJ as `tax_number`, `random_key`) are in `v2/guides/pix/using-pix-key.mdx`.
- Refund a settled receivable: `POST /v2/documents/in/{id}/refund`; read refunds under `/v2/documents/in/refunds`.
- Fees: charge `options.fee` / `fee` on top of the amount (`v2/guides/receivables/fees.mdx`). Splits: send `splits` to distribute a receivable to counterparty recipients (`v2/guides/receivables/split-payments.mdx`).

### Pix Automático (recurring Pix)

Three chained objects: recurrence (authorization) → collection (each charge) → attempt (each settlement try). Create with `POST /v2/in/pix/recurrences` (required: `bank_account_id`, `counterparty`, `description` ≤35 chars, `contract_number` ≤35 chars, `periodicity` in `week|mnth|qurt|mian|year`, `start_date`; `start_date` at least 3 days ahead). Amounts: `amounts.fixed`, `amounts.minimum`, or omit for fully variable (then the amount must be set per collection). Journeys combine `options.authorization_type` (`qrcode` vs `app`), `first_payment`, and `first_payment.recurrence_required`. Full payloads are in `v2/guides/receivables/using-pix-automatic.mdx`. Manage via `/v2/in/pix/recurrences/{id}`, `.../cancel`, `/collections`, `/collections/{id}`, `/collections/next`, `/collections/{id}/schedule`, `/collections/attempts`, and `/requests/{id}`.

## Cash-out — pay money (`/v2/out/...`)

Every payment debits `bank_account_id` and creates an out document. The response is immediate and final: `{ data: { out_document_id, approval_request_id } }`. Persist `out_document_id` and read the document with `GET /v2/documents/out/{id}`. Full walkthrough: `v2/guides/payments/cash-out.mdx`; overview: `v2/guides/payments/overview.mdx`.

### Pix — two flows, same out document

| Flow | Endpoints | When |
| --- | --- | --- |
| Asynchronous (one call) | `POST /v2/out/pix/async` with `type: key|brcode` + `reference` | Automated payouts; Trio checks the receiver while processing |
| Check, then pay | `POST /v2/out/pix/keys/check` or `POST /v2/out/pix/brcodes/check`, then `POST /v2/out/pix` with `check_id` | Confirm receiver name/tax number before paying |
| Bank account direct | `POST /v2/out/pix` with `type: bank_account` + full `counterparty` | Pay ISPB/branch/number/digit/type directly, no check |

Async body: required `bank_account_id`, `type`, `reference` (Pix key or BR code), `amount` (cents); optional `external_id`, `description`, and an expected-receiver assertion `counterparty { tax_number, bank_accounts[] }` — Trio refuses the payment if the key/BR code resolves elsewhere (closed-loop payout rules, `v2/operations/closed-loop.mdx`). Check calls take `entity_id` + `key` (or `hash` for BR codes) and return the check `id`, resolved `counterparty`, and masked `tax_number` for CPF. Key checks can be read back with `GET /v2/out/pix/keys/{id}` and BR-code checks with `GET /v2/out/pix/brcodes/{id}`.

Timeout rule: if a cash-out request times out, do **not** re-send blindly — search first with `GET /v2/documents/out/search` by `external_id`, check received webhooks, then contact support. A duplicate send is a duplicate payout. Out-document `settled` and `failed` are both terminal. See `v2/operations/best-practices.mdx`.

### Boleto, utility, tax, DARF payments

Each family has the same three shapes — check, pay (synchronous), pay async:

- Boleto: `POST /v2/out/boleto/check`, `POST /v2/out/boleto`, `POST /v2/out/boleto/async`
- Utility: `POST /v2/out/utility/check`, `POST /v2/out/utility`, `POST /v2/out/utility/async`
- Tax: `POST /v2/out/tax/check`, `POST /v2/out/tax`, `POST /v2/out/tax/async`
- DARF: `POST /v2/out/darf/check`, `POST /v2/out/darf`, `POST /v2/out/darf/async`

### Transfers, cards, pledged advances

- Transfer between your bank accounts: `POST /v2/transfer` (async; answers `200` with the future document ID). Read with `GET /v2/documents/transfer` and `GET /v2/documents/transfer/{id}`.
- Corporate cards: `POST/GET /v2/cards`, `GET/PUT /v2/cards/{id}`, `GET /v2/cards/{id}/info`, `POST .../block|unblock|cancel|topup|transfer_balance`, `POST /v2/cards/{id}/topup/qrcode`, invoices via `/v2/cards/invoices`, `/current`, `/{id}/lines`, `/{id}/export`, purchases via `/v2/cards/purchases`, `/{id}`, `/e2e/{id}`.
- Pledged advance requests: `POST/GET /v2/pledged/advance_requests`, `GET /v2/pledged/advance_requests/{id}`, `POST .../cancel`.

## Onboarding (distributors only)

Requires a Trio distribution channel — without one, do not build against these endpoints; tell the customer to contact Trio Support or their Account Manager. Flow (`v2/guides/onboarding-guide.mdx`): collect company (CNPJ, name, `business_infos { annual_revenue, average_ticket }` as money objects), `representatives[]` (CPF, email, E.164 phone, name, `is_pep`, `is_us_person`), and `consent { tax_number, ip_address, consent_at }`; `POST /v2/onboarding`; store returned `org_id`, `entity_id`, and each representative `id`; upload PDFs with `POST /v2/onboarding/documents`; follow `entity` webhook events (`in_review`, `approved`, `rejected`); after approval, operate with `Trio-Target-Org-Id: {org_id}`.

## Webhooks — design for them first

Most operations are asynchronous; webhooks replace polling. Read `v2/webhooks/introduction.mdx`, `v2/webhooks/authentication.mdx`, `v2/webhooks/content-and-structure.mdx`, and `v2/webhooks/events/overview.mdx` before writing handlers.

Register via the Trio App (`v2/guides/creating-your-first-webhook.mdx`) or `POST /v2/webhooks` with `notification_url` (HTTPS, public, accepts POST), `authentication_method` (`url_secret` with `notification_secret`, or `oauth` with `oauth_client_id/secret/url/scope`), optional `entity_id`, `group_id`, `event_categories_allowed [{ category, types }]`, and `api_version`. Manage with `GET /v2/webhooks` (single page, unpaginated), `GET/PUT/DELETE /v2/webhooks/{webhook_id}`. Replace (`PUT`) is full-replace: omitted fields are cleared.

Payload version matters: `v100` (default) uses legacy names (`collecting_document`, `virtual_account_id`, no `stages`); `v200` uses V2 names (`in_document`, `bank_account_id`, nested `counterparty`/`reference`, `stages`). New integrations must create the webhook with `api_version: "v200"`. Note the current filter caveat: `event_categories_allowed`, `entity_id`, and `group_id` only filter `v100` and `entity` events — a `v200` webhook receives all categories, so filter on `category`/`type`/`data.entity_id` in the handler. In `event_categories_allowed`, document categories still use V1 names (`collecting_document` for `in_document`, etc.); V2 names there are refused with `422`.

Envelope: `{ ref_id, category, type, timestamp, data }`. Route on `category` + `type` (for example `in_document.settled`), find your record by `ref_id` or `data.external_id`, then update from `data`. Categories: `in_document`, `in_document_refund`, `out_document`, `out_document_refund`, `transfer_document`, `recurrence`, `recurrence_request`, `recurrence_collection`, `recurrence_collection_attempt`, `boleto`, `pledged_advance_request`, `entity`.

Verify every delivery before trusting it: compare `x-webhook-secret` for `url_secret` webhooks, and always verify `x-webhook-signature` — HMAC-SHA-256 of the **raw** body bytes with `notification_key`, uppercase hex, constant-time compare. Never re-serialize parsed JSON for verification.

Delivery contract: Trio waits 1 second; `200/201/202/204` acknowledges. Other statuses, connection refusals, and TLS failures retry from 5 seconds, doubling to a 3-minute cap, up to 24 attempts or 1 hour. Timeouts are **not** retried — re-read the resource from the API when an event may be missing. Handler rules: persist the payload, acknowledge `2XX` immediately, process asynchronously, and dedupe on (`ref_id`, `category`, `type`) because retries redeliver the same event.

## Lifecycles and reconciliation

Read `v2/operations/best-practices.mdx`, `v2/operations/closed-loop.mdx`, and `v2/guides/pix/document-lifecycle.mdx` before writing state machines:

- Cash-out: `created → settled` or `created → failed`; both ends are final. The identifier (`out_document_id`) is final from the first response.
- Cash-in: the QR code settles later. `failed` on an in document means one payment attempt was refused (closed-loop rules: unidentified deposit, duplicated QR payment, amount/tax-number/bank-account mismatch, legal-entity payer, unauthorized participant) — the QR code stays payable and a later attempt can still settle. Only `settled` closes the charge. Sequences like `created → failed → settled` are normal.
- Persist every identifier: `external_id`, QR code `id`, in/out document `id`, `check_id`, `end_to_end_id`, `reconciliation_id`, `receipt_url`, and `X-Request-Id` per call.
- Expected latencies (from V2 operations docs): cash-out ~200–280 ms stable (up to 1 s on oscillation); QR creation ~80–100 ms with a 500 ms timeout threshold — on QR timeout, create a new request only with a new `external_id`.

Sandbox simulations: pay a QR code successfully (`v2/guides/receivables/simulating-deposit-success.mdx`) and force deposit failures (`v2/guides/receivables/simulating-deposit-failures.mdx`) to test both branches before production.

## Trio Checkout (SDK) and Trio MCP

- Checkout SDK (`v2/trio-sdk/introduction.mdx`, `v2/trio-sdk/using-the-sdk.mdx`): embed `https://assets.trio.com.br/checkout-js-sdk.min.js`, create the session in your **back end** (`options.session_type`: `payin`, `payout`, `guest_onboarding`, `user_onboarding`), then `Trio.create({ environment, session, onSuccess, onExit, onEvent, onLoad })` and `Trio.open()` in the front end. Flows: `v2/trio-sdk/flows/payin.mdx`, `v2/trio-sdk/flows/payout.mdx`; runnable example: `v2/trio-sdk/local-example.mdx`.
- Trio MCP (`v2/mcp/what-is-trio-mcp.mdx`, `v2/mcp/connecting.mdx`, `v2/mcp/tools.mdx`): hosted read-only server at `https://mcp.trio.com.br/mcp` with OAuth sign-in, scoped to one organization. It reads balances, documents, cards, and invoices; it never moves money. Recommend it for finance-team questions, not as a substitute for an API integration.

## How to answer and build with this skill

1. Identify the job first (cash-in, cash-out, reconciliation, webhooks, onboarding, cards, SDK/MCP) and open the matching V2 guide plus `v2/api-reference/openapi.json` operation before writing code. Endpoint MDX files under `v2/api-reference/banking-api/` are thin OpenAPI pointers — the spec and the guides carry the fields.
2. Confirm prerequisites out loud: sandbox account, API credential, `bank_account_id` (list with `GET /v2/bank_accounts`), and — for distributors — the target `org_id` and distribution channel.
3. Generate code in the customer's stack with a single API client: base URL from env, Basic auth, `Trio-Version` header, JSON everywhere, `error_code`-based error handling, cursor-pagination loops, and exponential backoff on `429`. Use integer cents and UTC timestamps; never float money.
4. Always ship the webhook handler with the first money-moving feature, not after it: raw-body signature verification, fast `2XX`, async processing, idempotent dedupe, and in-handler filtering for `v200` webhooks.
5. Enforce the safety checks: unique `external_id` per operation, no blind cash-out retries (search by `external_id` first), expected-receiver assertion on payouts where the receiver is known, secrets only in the back end.
6. Test in sandbox end-to-end (success + at least one failure simulation + one webhook redelivery), then switch base URL and credentials to production. Suggested order from `v2/getting_started/plan-your-integration.mdx`: sandbox account → API credential → webhook → receivables or payments guide.

When a request is ambiguous (for example "charge a customer" without saying QR code, boleto, or Pix Automático), ask which receivable type, amount behavior, and due-date rules apply before coding — the three QR types, boleto, and recurrence payloads are not interchangeable.

## V2 documentation map

| Need | Read |
| --- | --- |
| Start an integration | `v2/getting_started/plan-your-integration.mdx`, `v2/guides/sandbox_account.mdx`, `v2/guides/creating-your-first-api-credential.mdx` |
| Auth, errors, pagination, limits, dates | `v2/developers/authentication.mdx`, `error-table.mdx`, `pagination.mdx`, `rate-limit.mdx`, `date-and-time.mdx`, `request-identification.mdx`, `environments.mdx`, `concepts.mdx` |
| Receive money | `v2/guides/receivables/overview.mdx`, `cash-in.mdx`, `v2/guides/pix/qr-codes.mdx`, `v2/guides/boleto/create-boleto.mdx`, `v2/guides/receivables/using-pix-automatic.mdx`, `fees.mdx`, `split-payments.mdx` |
| Pay money | `v2/guides/payments/overview.mdx`, `cash-out.mdx`, `v2/guides/pix/using-pix-key.mdx` |
| Webhooks | `v2/webhooks/introduction.mdx`, `authentication.mdx`, `content-and-structure.mdx`, `events/overview.mdx`, `v2/guides/creating-your-first-webhook.mdx` |
| Reliability | `v2/operations/best-practices.mdx`, `closed-loop.mdx`, `v2/guides/pix/document-lifecycle.mdx`, `v2/guides/receivables/simulating-deposit-*.mdx` |
| Onboarding, SDK, MCP | `v2/guides/onboarding-guide.mdx`, `v2/trio-sdk/`, `v2/mcp/` |
| Exact fields for an operation | `v2/api-reference/openapi.json` (search the path) + the guide that links it |
