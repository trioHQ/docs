# Proprietary Skills — Internal & Customer Documentation

Documentation for the proprietary agent skills maintained in this repository
(`/Users/petersonfs/Work/trio/docs`). Written for two audiences: customers
integrating the Trio Banking API V2 with an AI coding agent, and internal
maintainers who keep the skills in sync with the V2 documentation.

> **Scope rule for this document:** every claim below is based on files
> actually read in this repo — principally
> [`.agents/skills/trio-docs/SKILL.md`](.agents/skills/trio-docs/SKILL.md)
> (23,776 bytes, the only file in that skill directory) and the V2 spec at
> `v2/api-reference/openapi.json` (verified directly: `info.title` =
> `Trio API v2`, `info.version` = `2026-10-19.v2`, 106 paths). This skill is
> **V2-only**; V1 is never cited as a source (see "V2-only rule" below).

---

## 1. Inventory

Skills found under `.agents/skills/` (verified with `ls -R .agents/skills`
on 2026-10-26):

| Skill | Path | Files | Status in git |
| --- | --- | --- | --- |
| `trio-docs` | `.agents/skills/trio-docs/` | `SKILL.md` only | Untracked (`?? .agents/`) — new, not yet committed |

**No other proprietary skills exist under `.agents/skills/` at the time of
writing.** The inventory is exactly one skill. Searches of `README.md`,
`docs.json`, and `llms.txt` return no references to `.agents/skills`,
`SKILL.md`, or `trio-docs`, so this file is currently the only documentation
of the skill outside the skill itself.

Note for maintainers: because `.agents/` is untracked, the skill is not yet
distributed via git clone/pull. Committing it is a prerequisite for the
customer install flow in §4 to work from a fresh checkout (see §5).

---

## 2. Skill: `trio-docs` — Trio Banking API V2 implementation

### 2.1 Purpose

Frontmatter in `SKILL.md` (lines 1–4):

```yaml
---
name: trio-docs
description: Help customers implement the Trio Banking API V2 — auth, cash-in, cash-out, webhooks, and reconciliation in any stack.
---
```

In the skill's own words (line 8): it is used "when a customer asks to
integrate, debug, or extend a Trio integration: creating charges, paying
out, handling webhooks, reconciling documents, onboarding customers, or
moving an integration from sandbox to production."

The title line (line 6) states the version scope explicitly:
"Trio Docs — Implement the Trio API (V2 only)".

### 2.2 When it triggers

The skill triggers on the `description` frontmatter above. Based on the
body, the intended trigger situations are:

- A customer asks to **integrate** Trio (new integration, any stack).
- A customer asks to **debug** an existing Trio integration.
- A customer asks to **extend** an existing Trio integration.
- Concrete jobs named in the body: creating charges (cash-in), paying out
  (cash-out), handling webhooks, reconciling documents, onboarding
  customers (distributors), moving sandbox → production.

The closing sections (§ "How to answer and build with this skill", lines
191–200) reinforce the trigger model: the agent first identifies the job
(cash-in, cash-out, reconciliation, webhooks, onboarding, cards, SDK/MCP),
then opens the matching V2 guide plus the OpenAPI operation before writing
code. If the request is ambiguous (e.g. "charge a customer" without saying
QR code, boleto, or Pix Automático), the agent must ask which receivable
type, amount behaviour, and due-date rules apply before coding — the three
QR types, boleto, and recurrence payloads are not interchangeable.

### 2.3 V2-only rule (non-negotiable)

Line 10 of `SKILL.md` is the source-of-truth clause every maintainer and
customer should know:

- Source of truth is the V2 documentation under `v2/` and the V2 OpenAPI
  spec at `v2/api-reference/openapi.json` (106 paths, `Trio API v2`,
  version `2026-10-19.v2`) — all three values verified against the spec
  file itself.
- **Ignore V1 completely:** never use V1 paths, V1 field names, or
  `api-reference/` (non-`v2/`) pages.
- If a V2 page contains a `TODO(v2): verify` note, flag the value as
  unverified instead of inventing it.

Core V2 invariants the skill enforces (lines 14–20):

- Every path starts with `/v2`. There are no virtual accounts in V2; the
  receiving/paying account is always `bank_account_id`.
- Amounts are integer cents (BRL 100.00 = `10000`); money objects are
  `{ "amount": 10000, "currency": "BRL" }`. Never float money.
- Timestamps are UTC ISO 8601; plain dates are `YYYY-MM-DD`.
- Branch on `error.error_code`, never on the human-readable message.
- `external_id` is the reconciliation key: unique per operation (UUID),
  enforced for 24 hours, persisted with every identifier Trio returns.
- Webhooks are the source of truth for final status; polling is a fallback.
- Secrets (`client_secret`, `notification_secret`, `notification_key`)
  never appear in front-end code, logs, or Checkout session creation;
  secret-bearing calls happen in the back end only.

### 2.4 What V2 areas it covers

Coverage is taken section-by-section from `SKILL.md`. All paths below are
V2 (`/v2/...`); all doc references are under `v2/`.

#### Auth (Authentication — every request)

- HTTP Basic auth: `client_id` = username, `client_secret` = password.
  Create the pair in the Trio App (Settings → API Keys → Create Key) with
  Read and Write or Read-Only permission and an allowed-IP list; the
  secret is shown once, store it immediately.
- Every request sends `Content-Type: application/json` (when it has a
  body), `Trio-Version` with the current API version, and Basic
  credentials. Missing/outdated `Trio-Version` → `400 INVALID_TRIO_VERSION`.
  **Known gap (in the skill itself):** the V2 docs do not state the literal
  current `Trio-Version` value — the agent must ask the customer to confirm
  it from their Trio contact or dashboard instead of guessing.
- Failures: `401 INVALID_CREDENTIALS`, `403 FORBIDDEN` (e.g. read-only key
  writing). Quote `X-Request-Id` with the `error_code` when contacting
  support.
- Distributor variant: `Trio-Target-Org-Id: {org_id}` to act on another
  organization; access is granted per organization by Trio; read-only
  access allows only `GET`/`HEAD`.
- Environments table (line 24–27): Sandbox
  `https://api.sandbox.trio.com.br` (app `https://app.sandbox.trio.com.br`),
  Production `https://api.trio.com.br` (app `https://app.trio.com.br`).
  Base URL lives in environment config, never hard-coded per call.
- Source docs: `v2/guides/creating-your-first-api-credential.mdx`,
  `v2/developers/authentication.mdx`, `v2/developers/environments.mdx`,
  `v2/developers/request-identification.mdx`, `v2/getting_started/support.mdx`.

#### Cash-in — receive money (`/v2/in/...`)

Every receivable is created against a `bank_account_id` and becomes an
**in document** when paid. Areas covered:

- **Pix QR codes** — three types with distinct endpoints: Dynamic
  `POST /v2/in/pix/qrcodes` (one charge, one code, automated
  reconciliation), Static `POST /v2/in/pix/qrcodes/static` (reusable;
  `amount: 0` for open amount), Dynamic with due date
  `POST /v2/in/pix/qrcodes/with_due` (invoice-like: due date, fine,
  interest, rebate, discount). Includes required/optional request fields,
  the `201` response shape (`data.id`, `status: created`, `hash` = BR
  code / Pix copia-e-cola), image/hash retrieval, and `PUT` update (204).
  **Critical reconciliation warning:** `data.id` is only the QR code; the
  final in-document `id` arrives only in the `in_document.settled` webhook
  (`data.reference_id` / `reference.id` points back to the QR code). Never
  reconcile cash-in on the QR ID alone.
- **Boletos (receivables)** — `POST /v2/in/boletos` with required fields
  (amount, `bank_account_id`, `invoice_type`, `wallet_type`,
  `expiration_date`, full counterparty address block,
  `due_detail.due_date`), the cents-vs-basis-points convention (monetary
  rules send `amount` in cents; percentage rules send `value` in basis
  points, 100 = 1%), the discount rule (`fixed_until_due` vs
  `per_day_antecipated` — never both), and lifecycle endpoints (GET, PUT,
  cancel, recreate, protest add/remove).
- **Pix keys** — `POST/GET /v2/pix/keys`, confirm, resend_token, cancel;
  key types and validation regexes referenced from the V2 guide.
- **Refunds & splits & fees** — refund via
  `POST /v2/documents/in/{id}/refund`; `options.fee`/`fee` on top of the
  amount; `splits` to counterparty recipients.
- **Pix Automático (recurring Pix)** — three chained objects: recurrence
  (authorization) → collection (each charge) → attempt (each settlement
  try). `POST /v2/in/pix/recurrences` with `periodicity` in
  `week|mnth|qurt|mian|year`, `start_date` at least 3 days ahead,
  fixed/minimum/variable amount modes, `qrcode` vs `app` authorization
  journeys, plus collection/attempt management endpoints.
- Source docs: `v2/guides/receivables/overview.mdx`, `cash-in.mdx`,
  `v2/guides/pix/qr-codes.mdx`, `v2/guides/boleto/create-boleto.mdx`,
  `boleto-types.mdx`, `best-practices.mdx`,
  `v2/guides/receivables/using-pix-automatic.mdx`, `fees.mdx`,
  `split-payments.mdx`, `v2/guides/pix/using-pix-key.mdx`.

#### Cash-out — pay money (`/v2/out/...`)

Every payment debits `bank_account_id` and creates an **out document**.
The response is immediate and final:
`{ data: { out_document_id, approval_request_id } }`; persist
`out_document_id` and read the document at `GET /v2/documents/out/{id}`.

- **Pix — two flows, same out document:** asynchronous one-call
  `POST /v2/out/pix/async` (`type: key|brcode` + `reference`), and
  check-then-pay (`POST /v2/out/pix/keys/check` or
  `POST /v2/out/pix/brcodes/check`, then `POST /v2/out/pix` with
  `check_id`), plus bank-account-direct `POST /v2/out/pix` with
  `type: bank_account` + full counterparty. Includes the
  expected-receiver assertion (Trio refuses if the key/BR code resolves
  elsewhere — closed-loop payout rules).
- **Timeout rule (safety-critical):** on cash-out timeout, do **not**
  re-send blindly — search `GET /v2/documents/out/search` by
  `external_id`, check received webhooks, then contact support. A duplicate
  send is a duplicate payout. `settled` and `failed` are both terminal.
- **Boleto, utility, tax, DARF payments** — each family has check /
  pay-sync / pay-async triplets under `/v2/out/boleto`, `/v2/out/utility`,
  `/v2/out/tax`, `/v2/out/darf`.
- **Transfers, cards, pledged advances** — `POST /v2/transfer` between own
  bank accounts; corporate cards (`/v2/cards`, invoices, purchases,
  block/unblock/cancel/topup); pledged advance requests
  (`/v2/pledged/advance_requests`).
- Source docs: `v2/guides/payments/cash-out.mdx`, `overview.mdx`,
  `v2/guides/pix/using-pix-key.mdx`, `v2/operations/closed-loop.mdx`,
  `v2/operations/best-practices.mdx`.

#### Webhooks — design for them first

"Most operations are asynchronous; webhooks replace polling."

- **Registration:** via the Trio App or `POST /v2/webhooks` with
  `notification_url` (HTTPS, public, accepts POST),
  `authentication_method` (`url_secret` + `notification_secret`, or
  `oauth` + client id/secret/url/scope), optional `entity_id`, `group_id`,
  `event_categories_allowed [{ category, types }]`, and `api_version`.
  Manage with GET (single page, unpaginated), GET/PUT/DELETE by
  `{webhook_id}`; PUT is full-replace — omitted fields are cleared.
- **Payload versions:** `v100` (default) uses legacy names
  (`collecting_document`, `virtual_account_id`, no `stages`); `v200` uses
  V2 names (`in_document`, `bank_account_id`, nested
  `counterparty`/`reference`, `stages`). **New integrations must create
  the webhook with `api_version: "v200"`.** Caveats the skill flags:
  `event_categories_allowed`/`entity_id`/`group_id` filters only apply to
  `v100` and `entity` events — a `v200` webhook receives all categories,
  so filter on `category`/`type`/`data.entity_id` in the handler; and in
  `event_categories_allowed`, document categories still use V1 names
  (V2 names there are refused with `422`).
- **Envelope & routing:** `{ ref_id, category, type, timestamp, data }`;
  route on `category` + `type` (e.g. `in_document.settled`), find the
  record by `ref_id` or `data.external_id`. Categories: `in_document`,
  `in_document_refund`, `out_document`, `out_document_refund`,
  `transfer_document`, `recurrence`, `recurrence_request`,
  `recurrence_collection`, `recurrence_collection_attempt`, `boleto`,
  `pledged_advance_request`, `entity`.
- **Verification:** compare `x-webhook-secret` for `url_secret` webhooks;
  always verify `x-webhook-signature` — HMAC-SHA-256 of the **raw** body
  bytes with `notification_key`, uppercase hex, constant-time compare.
  Never re-serialize parsed JSON for verification.
- **Delivery contract:** Trio waits 1 second; `200/201/202/204`
  acknowledges. Other statuses, connection refusals, and TLS failures
  retry from 5 s, doubling to a 3-minute cap, up to 24 attempts or 1 hour.
  **Timeouts are not retried** — re-read the resource from the API when an
  event may be missing. Handler rules: persist the payload, acknowledge
  `2XX` immediately, process asynchronously, dedupe on
  (`ref_id`, `category`, `type`).
- Source docs: `v2/webhooks/introduction.mdx`, `authentication.mdx`,
  `content-and-structure.mdx`, `events/overview.mdx`,
  `v2/guides/creating-your-first-webhook.mdx`.

#### Reconciliation (lifecycles & reconciliation)

- **Cash-out lifecycle:** `created → settled` or `created → failed`; both
  ends are final; `out_document_id` is final from the first response.
- **Cash-in lifecycle:** the QR code settles later. `failed` on an in
  document means one payment attempt was refused (closed-loop reasons:
  unidentified deposit, duplicated QR payment, amount/tax-number/
  bank-account mismatch, legal-entity payer, unauthorized participant) —
  the QR code stays payable and a later attempt can still settle. Only
  `settled` closes the charge; `created → failed → settled` is normal.
- **Identifiers to persist:** `external_id`, QR code `id`, in/out document
  `id`, `check_id`, `end_to_end_id`, `reconciliation_id`, `receipt_url`,
  and `X-Request-Id` per call.
- **Expected latencies (from V2 operations docs):** cash-out ~200–280 ms
  stable (up to 1 s on oscillation); QR creation ~80–100 ms with a 500 ms
  timeout threshold — on QR timeout, create a new request only with a new
  `external_id`.
- **Pagination for aggregation:** cursor-based
  (`{ data, metadata: { before, after, has_more } }`, `limit` default 50
  max 100); document listings additionally require `from_datetime` and
  `to_datetime`; always loop while `has_more` before aggregating totals.
- **Error model:** branch on `error.error_code`; status table 200/201/204,
  400 (`MALFORMED_REQUEST`, `INVALID_TRIO_VERSION`), 401
  (`INVALID_CREDENTIALS`), 403 (`FORBIDDEN`), 404 (`NOT_FOUND_RESOURCE`),
  422 (`INVALID_PARAMETER`, `REQUIRED_PARAMETER`, business codes), 429
  (exponential backoff), 503 (`TRIO_SERVICE_UNAVAILABLE`).
- **Rate limits — flagged unverified in the skill:** 60 req/s production,
  5 req/s sandbox, per account and per endpoint, "kept from V1 — treat as
  unverified against the V2 spec".
- **Sandbox simulations:** pay a QR code successfully and force deposit
  failures (V2 guides) to test both branches before production.
- Source docs: `v2/operations/best-practices.mdx`, `closed-loop.mdx`,
  `v2/guides/pix/document-lifecycle.mdx`,
  `v2/guides/receivables/simulating-deposit-success.mdx`,
  `simulating-deposit-failures.mdx`, `v2/developers/pagination.mdx`,
  `error-table.mdx`, `rate-limit.mdx`.

#### Onboarding (distributors only)

- Requires a Trio distribution channel — without one, do not build against
  these endpoints; tell the customer to contact Trio Support or their
  Account Manager.
- Flow: collect company (CNPJ, name, `business_infos { annual_revenue,
  average_ticket }` as money objects), `representatives[]` (CPF, email,
  E.164 phone, name, `is_pep`, `is_us_person`), and
  `consent { tax_number, ip_address, consent_at }`; `POST /v2/onboarding`;
  store returned `org_id`, `entity_id`, and each representative `id`;
  upload PDFs with `POST /v2/onboarding/documents`; follow `entity`
  webhook events (`in_review`, `approved`, `rejected`); after approval,
  operate with `Trio-Target-Org-Id: {org_id}`.
- Supporting core-object context: **Entity** status flows
  `created → in_validation → in_review → approved` (or `rejected` /
  `onhold` / `canceled`); Company, Group, Bank account (category `linked`,
  `owner`, `transactional`, `operational`), Counterparty, and Document
  (in/out/transfer/refund) definitions are in the "Core objects" section.
- Source docs: `v2/guides/onboarding-guide.mdx`,
  `v2/developers/concepts.mdx`, `v2/developers/authentication.mdx`.

#### SDK / MCP (Trio Checkout & Trio MCP)

- **Trio Checkout SDK:** embed
  `https://assets.trio.com.br/checkout-js-sdk.min.js`; create the session
  in the **back end** (`options.session_type`: `payin`, `payout`,
  `guest_onboarding`, `user_onboarding`), then
  `Trio.create({ environment, session, onSuccess, onExit, onEvent, onLoad })`
  and `Trio.open()` in the front end. Runnable example referenced.
  Source docs: `v2/trio-sdk/introduction.mdx`, `using-the-sdk.mdx`,
  `flows/payin.mdx`, `flows/payout.mdx`, `local-example.mdx`.
- **Trio MCP:** hosted **read-only** server at
  `https://mcp.trio.com.br/mcp` with OAuth sign-in, scoped to one
  organization. It reads balances, documents, cards, and invoices; it
  never moves money. The skill recommends it for finance-team questions,
  not as a substitute for an API integration.
  Source docs: `v2/mcp/what-is-trio-mcp.mdx`, `connecting.mdx`,
  `tools.mdx`.

#### Also covered (beyond the seven headline areas)

- Core objects & institutions (`GET /v2/institutions`).
- Pagination, error table, rate limits, date/time, request identification.
- Integration order (from `v2/getting_started/plan-your-integration.mdx`):
  sandbox account → API credential → webhook → receivables or payments
  guide; test in sandbox end-to-end (success + ≥1 failure simulation +
  1 webhook redelivery), then switch base URL and credentials to
  production.
- A "V2 documentation map" table (lines 202–213) mapping needs (start,
  auth/errors/pagination, receive, pay, webhooks, reliability,
  onboarding/SDK/MCP, exact fields) to the exact `v2/` files to read.
  Endpoint MDX files under `v2/api-reference/banking-api/` are thin
  OpenAPI pointers — the spec and the guides carry the fields.

---

## 3. Skill file anatomy (for maintainers)

The skill is a single Markdown file with YAML frontmatter:

```
.agents/skills/trio-docs/
└── SKILL.md        # name + description frontmatter, then the body
```

- Frontmatter: `name: trio-docs`,
  `description: Help customers implement the Trio Banking API V2 — auth, cash-in, cash-out, webhooks, and reconciliation in any stack.`
  The `description` is the trigger surface — agents decide to load the
  skill from it (see §2.2).
- Body sections, in order: V2-only rule & non-negotiable rules →
  environments/base URLs → authentication (incl. distributor header) →
  core objects → pagination/errors/rate limits → cash-in → cash-out →
  onboarding → webhooks → lifecycles & reconciliation → Checkout SDK &
  MCP → "How to answer and build with this skill" (6-step working
  procedure) → V2 documentation map.
- There are no scripts, assets, or reference files bundled with the skill;
  it points into the repo's `v2/` docs and `v2/api-reference/openapi.json`
  instead of duplicating them. That makes the repo checkout part of the
  skill's runtime surface (see §4.1).

---

## 4. How a customer installs and uses it

### 4.1 Install

The skill itself contains no installer and no install instructions (true
of the file as read). Installation is the standard agent-skills layout
that this repo already follows:

1. Get this repo (clone or download) so the `v2/` docs and
   `v2/api-reference/openapi.json` are on disk — the skill references them
   by relative path (`v2/...`) and will direct the agent to read them.
   Without the `v2/` tree next to it, the skill's doc pointers dangle.
   (Today `.agents/` is untracked in git; until it is committed, obtain
   the `.agents/skills/trio-docs/` folder from a maintainer rather than a
   fresh clone — see §5.)
2. Place the `trio-docs` folder under the skills directory of the
   customer's coding agent, keeping the repo's `v2/` tree reachable from
   the workspace the agent works in. The repo-canonical location is:

   ```
   <workspace>/.agents/skills/trio-docs/SKILL.md
   ```

   Vendor-specific agents may expect the same folder under their own
   skills directory; the folder content (`SKILL.md` with `name` /
   `description` frontmatter) is unchanged in either case.
3. Start a new agent session in that workspace. The agent discovers the
   skill from the `description` frontmatter; no registration or config
   edit is needed beyond the folder placement.

Keep the skill read-only for customers: local edits will be overwritten
the next time maintainers ship an updated copy (see §5).

### 4.2 Use

Once installed, the customer does not invoke the skill by name in normal
use — they ask for the integration work in plain language, for example:

- "Integrate Trio Pix cash-in (dynamic QR codes) in my Node back end."
- "Add a webhook handler for `in_document.settled` with signature
  verification."
- "Why did my payout fail with `PAYMENT_ACCOUNT_WITHOUT_BALANCE`?"
- "Move my Trio integration from sandbox to production."

Per the skill's working procedure, the agent will then:

1. Identify the job and open the matching V2 guide plus the OpenAPI
   operation before writing code.
2. Confirm prerequisites out loud: sandbox account, API credential,
   `bank_account_id` (listed via `GET /v2/bank_accounts`), and — for
   distributors — the target `org_id` and distribution channel.
3. Generate code in the customer's stack around a single API client:
   base URL from environment config, Basic auth, `Trio-Version` header,
   JSON everywhere, `error_code`-based error handling, cursor-pagination
   loops, exponential backoff on `429`, integer cents, UTC timestamps.
4. Ship the webhook handler with the first money-moving feature: raw-body
   signature verification, fast `2XX` acknowledgement, async processing,
   idempotent dedupe, in-handler filtering for `v200` webhooks.
5. Enforce the safety checks: unique `external_id` per operation, no blind
   cash-out retries (search by `external_id` first), expected-receiver
   assertion where the receiver is known, secrets only in the back end.
6. Test in sandbox end-to-end (success + at least one failure simulation +
   one webhook redelivery), then switch base URL and credentials to
   production.

Suggested integration order (from the skill, citing
`v2/getting_started/plan-your-integration.mdx`): sandbox account → API
credential → webhook → receivables or payments guide.

Two customer-facing cautions the skill bakes in:

- **Ask, don't guess:** if the receivable type is ambiguous (QR vs boleto
  vs Pix Automático), the agent should ask before coding; the payloads are
  not interchangeable.
- **Trio-Version value:** the V2 docs do not state the literal current
  value; customers confirm it from their Trio contact or dashboard.

---

## 5. How maintainers update it

> **Ownership note:** `.agents/skills/trio-docs/SKILL.md` is owned by
> another session. Do not edit it as part of routine documentation work;
> route changes through the owning session/owner. This section describes
> the update procedure for that owner.

Update the skill whenever either source of truth changes:

- any doc under `v2/` that `SKILL.md` cites (guides, `developers/`,
  `webhooks/`, `operations/`, `trio-sdk/`, `mcp/`, `getting_started/`), or
- `v2/api-reference/openapi.json` (paths, fields, versions).

### 5.1 Checklist when `v2/api-reference/openapi.json` changes

1. Re-read the spec header: `info.title` and `info.version`, and count
   `paths`. Today: `Trio API v2`, `2026-10-19.v2`, 106 paths. If any of
   these change, update the parenthetical on line 10 of `SKILL.md`
   ("106 paths, `Trio API v2`, version `2026-10-19.v2`") to match —
   verbatim values, verified from the file, never from memory.
2. Diff the path list against every endpoint named in `SKILL.md`:
   cash-in (`/v2/in/...`, `/v2/pix/keys`, recurrences), cash-out
   (`/v2/out/...` incl. boleto/utility/tax/DARF triplets), `/v2/transfer`,
   `/v2/cards`, `/v2/pledged/advance_requests`, `/v2/bank_accounts`,
   `/v2/entities`, `/v2/counterparties`, `/v2/documents/...`,
   `/v2/institutions`, `/v2/onboarding`, `/v2/webhooks`. Update the tables
   and endpoint lists for added/removed/renamed operations.
3. Check field-level changes for the payloads the skill spells out:
   QR-code create/response fields, boleto required fields and rule
   conventions (cents vs basis points, discount modes), Pix Automático
   recurrence fields and `periodicity` enum, cash-out response
   (`out_document_id`, `approval_request_id`), webhook registration
   fields, onboarding payload (`business_infos`, `representatives[]`,
   `consent`). Endpoint MDX files under `v2/api-reference/banking-api/`
   are thin pointers — verify against the spec JSON and the guides.
4. Confirm the error table still matches `v2/developers/error-table.mdx`
   and the spec's error responses (status → `error_code` mapping).
5. Re-run a path-count sanity check after editing, e.g.:

   ```bash
   python3 -c "import json; d=json.load(open('v2/api-reference/openapi.json')); print(d['info']['title'], d['info']['version'], len(d['paths']))"
   ```

### 5.2 Checklist when `v2/` docs change

1. For every changed MDX file, find its mentions in `SKILL.md` (most
   sections end with "See `v2/...`" pointers, and the "V2 documentation
   map" table at lines 202–213 lists the canonical reading paths). Update
   pointers for moved/renamed files; fix section content for changed
   behaviour.
2. **Preserve the V2-only boundary.** Never "fix" a gap by citing V1:
   no V1 paths, no V1 field names, no `api-reference/` (non-`v2/`) pages.
   The two deliberate exceptions already documented in the skill must stay
   explicitly labelled as legacy behaviour, not silently normalised:
   webhook payload version `v100` (legacy names) and the V1 names required
   inside `event_categories_allowed` (V2 names there return `422`), and
   the rate-limit figures "kept from V1 — treat as unverified".
3. **`TODO(v2): verify` notes:** if a cited V2 page carries one, the skill
   must flag the affected value as unverified rather than stating it as
   fact. When a TODO is resolved upstream, remove the "unverified" flag
   from the skill in the same change.
4. **Known open items to re-check on every pass:**
   - `Trio-Version` literal value — currently *not stated* in the V2 docs;
     the skill tells agents to ask the customer. If the docs start stating
     it, update the Authentication section and drop the caveat.
   - Rate limits (60 req/s prod, 5 req/s sandbox) — currently marked
     unverified against the V2 spec; verify against
     `v2/developers/rate-limit.mdx` and remove the caveat when confirmed.
   - Webhook `v200` filtering caveat (`event_categories_allowed`,
     `entity_id`, `group_id` only filtering `v100`/`entity` events) —
     remove the in-handler-filtering instruction if the API starts
     honouring filters for `v200`.
   - Expected latencies (cash-out ~200–280 ms, QR ~80–100 ms / 500 ms
     threshold) — sourced from V2 operations docs; re-verify when those
     pages change.
5. Environments and credentials: if base URLs, the Trio App flow
   (Settings → API Keys → Create Key), permissions, or the
   `Trio-Target-Org-Id` distributor flow change, update the Environments
   table and Authentication section together — they are cited in several
   other sections.
6. Keep the `description` frontmatter accurate: it is the trigger surface.
   If the skill gains or loses a whole area (e.g. a new product family),
   update the description in the same change so triggering stays correct.

### 5.3 Verifying an update

1. Every `v2/...` path cited in `SKILL.md` still exists:

   ```bash
   grep -oE 'v2/[A-Za-z0-9_./*-]+\.mdx' .agents/skills/trio-docs/SKILL.md | sort -u | while read -r f; do [ -e "$f" ] || echo "MISSING: $f"; done
   ```

2. Every `/v2/...` endpoint cited in `SKILL.md` exists in the spec's
   `paths` (allowing for `{id}`-style templates).
3. The spec facts on line 10 (title, version, path count) match a fresh
   read of `v2/api-reference/openapi.json` (see §5.1, step 5).
4. No V1 leakage: search the updated skill for `api-reference/` references
   that are not under `v2/`, and for any V1-only field names; the only
   legacy mentions allowed are the labelled `v100` webhook payload and
   `event_categories_allowed` naming caveats (§5.2, step 2).
5. Frontmatter still parses (`name`, `description` present) and the file
   remains the only file the skill needs — if you add reference files,
   document them in §3 of this file in the same change.
6. Commit `.agents/` (currently untracked) so customers can obtain the
   skill from a fresh clone, and update the inventory table in §1 of this
   file if the skill set changes.

### 5.4 Adding a new proprietary skill

1. Create `.agents/skills/<skill-name>/SKILL.md` with `name` and
   `description` frontmatter; the `description` is what triggers it, so
   write it as the customer-facing capability statement (follow the
   `trio-docs` pattern).
2. Prefer pointing into canonical repo docs over duplicating content —
   `trio-docs` carries no bundled reference files and stays maintainable
   because `v2/` + `openapi.json` remain the single source of truth.
3. Add a row to the inventory in §1 and a section like §2 to this file in
   the same change.

---

## Appendix — evidence base

Files read to write this document:

- `.agents/skills/trio-docs/SKILL.md` (full file, 213 lines) — purpose,
  triggers, all coverage sections, working procedure, documentation map.
- `v2/api-reference/openapi.json` — `info.title` = `Trio API v2`,
  `info.version` = `2026-10-19.v2`, 106 paths (parsed, not assumed).
- Directory listings: `.agents/skills/` (single skill, `SKILL.md` only),
  `v2/` (`api-reference`, `developers`, `faq`, `getting_started`, `guides`,
  `mcp`, `operations`, `trio-sdk`, `webhooks`), `v2/trio-sdk/`,
  `v2/mcp/`, `v2/webhooks/`, `v2/getting_started/`, `v2/guides/`.
- `git status` — `.agents/` untracked; `git log` for the V2 docs commit.
- Negative searches: no `trio-docs` / `SKILL.md` / `.agents/skills`
  references in `README.md`, `docs.json`, or `llms.txt`.

No V1 sources were used. `.agents/skills/trio-docs/SKILL.md` was not
modified.
