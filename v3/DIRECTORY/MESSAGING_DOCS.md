# Messaging — API reference

SMS and email for the auth and directory service: country routing for SMS, a
switch between Resend and Mandrill for email, contact validation, the
suppression list, provider webhooks and test endpoints.

Setup, provider accounts and the rollout checklist are in
[MESSAGING_INTEGRATION_GUIDE.md](./MESSAGING_INTEGRATION_GUIDE.md). Every
variable named here is listed with its default in
[CONFIGURATION.md](./CONFIGURATION.md#messaging-password-reset-rate-limit-and-birthday-reports).

## Contents

- [How sends work](#how-sends-work)
- [The per-channel result](#the-per-channel-result)
- [SMS routing](#sms-routing)
- [Email provider](#email-provider)
- [Validation and suppression](#validation-and-suppression)
- [Suppression endpoints](#suppression-endpoints)
- [Bounce CSV import](#bounce-csv-import)
- [Webhooks](#webhooks)
- [Test endpoints](#test-endpoints)
- [Error codes](#error-codes)
- [Collections](#collections)

## How sends work

All outgoing SMS and email go through `src/components/Messaging/`:

| Caller | Channel | Purpose tag |
|---|---|---|
| Password reset (`POST /auth/request-password-reset`) | SMS and email | `PASSWORD_RESET` |
| Birthday greetings (`src/crons/birthday.ts`) | SMS and email | `BIRTHDAY_GREETING` / `BIRTHDAY` |
| Birthday reports (`src/config/cron/birthdayReports.ts`) | email | `BIRTHDAY_REPORT` |
| Test endpoints below | SMS / email | `TEST` |

**Default configuration changes nothing.** With no new variables set:

- every SMS goes to Termii, with the `N-Alert` sender, the `dnd` channel and the
  same request body as before;
- email goes to Resend. The client is built lazily, so a missing
  `RESEND_API_KEY` is reported on each send and does not stop the service
  starting;
- no address is checked before sending.

**The one deliberate fix.** The password-reset email used to send from
`"undefined <undefined>"` whenever `MANDRILL_FROM_NAME`/`MANDRILL_FROM_ADDRESS`
were unset. The sender now resolves as `EMAIL_FROM_*`, then `MANDRILL_FROM_*`,
then `RCCG NOTIFICATIONS <auth.notifications@rccg.org>`. That is the default
the birthday emails already used.

The OTP is generated, stored and checked by this service as before
(`src/utils/otpSecret.ts`, the `PasswordReset` collection). No provider
generates or verifies codes. Message bodies are never logged.

## The per-channel result

Every send returns one of these. Nothing in the API reports a message as
"delivered". `ACCEPTED` means the provider took it, and only a webhook can say
what happened after that.

```json
{
  "channel": "sms",
  "status": "ACCEPTED",
  "provider": "msg91",
  "reason": "",
  "providerMessageId": "3561686c5a33343537363234",
  "to": "447400123456"
}
```

| `status` | Meaning |
|---|---|
| `ACCEPTED` | The provider accepted the message. |
| `FAILED` | A provider was called and refused or errored. See `reason` and `detail`. |
| `SKIPPED` | No provider was called. Causes include no address, an invalid number, or a suppressed address. |

`reason` values: `NOT_CONFIGURED`, `PROVIDER_REJECTED`, `PROVIDER_ERROR`,
`MISSING_PHONE`, `INVALID_PHONE`, `MISSING_EMAIL`, `INVALID_EMAIL`,
`PLACEHOLDER_EMAIL`, `NO_MX`, `SUPPRESSED`.

Sending to one person on several channels (`notify()` in
`Messaging/notify.ts`) gives a result per channel. A failure on one channel
never stops the other. The result also carries `actionRequired`:
`UPDATE_EMAIL` when the address is missing, invalid, a placeholder or
suppressed, and `UPDATE_PHONE` when the number is missing or invalid. Bulk
sends add up these results with `countResults()`.

## SMS routing

1. **Parsing.** `src/utils/phonePolicy.ts` normalises local forms (`0803…`,
   `803…`, `234 0803…`) to international digits. `libphonenumber-js` then
   parses the result to E.164 and works out the country. The profile `country`
   only supplies the dialling code for a number stored without one. It may be
   an alpha-2 code, a full name or a dialling code. A number that carries its
   own `+44` keeps it, whatever the profile says.
2. **Routing.**

| `SMS_COUNTRY_ROUTING_ENABLED` | Nigeria (`NG`) | Everywhere else | Number with no determinable country |
|---|---|---|---|
| `false` (default) | Termii | Termii | Still sent to Termii, formatted the old way (`0…` → dialling code + number), as before |
| `true` | `SMS_PROVIDER_NIGERIA` (default `termii`) | `SMS_PROVIDER_INTERNATIONAL` (default `msg91`) | `SKIPPED`, `INVALID_PHONE` |

Both providers receive international digits without the `+`, for example
`2348031234567`. Each provider call times out after
`SMS_PROVIDER_TIMEOUT_MS` (default 15 s).

**MSG91 only delivers our text.** Its OTP API is not used. See the
integration guide for the template and endpoint assumptions.

## Email provider

`EMAIL_PROVIDER` is `resend` (default) or `mandrill`.

- **Unset:** Resend, with the old lazy behaviour.
- **Set explicitly:** startup fails if the value is unknown, or if that
  provider's key is missing (`RESEND_API_KEY` / `MANDRILL_API_KEY`). The
  service will not boot on a mail configuration that cannot work.
- Mandrill is called over HTTP (`POST {MANDRILL_API_ENDPOINT}/messages/send.json`),
  with no SDK. Mandrill statuses `sent`, `queued` and `scheduled` count as
  `ACCEPTED`. `rejected` and `invalid` count as `FAILED`, and the detail
  carries the `reject_reason`.
- Each provider call times out after `EMAIL_PROVIDER_TIMEOUT_MS` (default 15 s).

## Validation and suppression

With `EMAIL_SUPPRESSION_ENABLED=true`, every email is checked before sending,
in this order. The first refusal ends the check, and the send is `SKIPPED`
with that reason:

| Check | `reason` |
|---|---|
| Empty | `MISSING_EMAIL` |
| An RFC 2606 `.invalid` placeholder | `PLACEHOLDER_EMAIL` |
| `src/utils/emailPolicy.ts` (syntax, and disposable domains when blocking those is on) | `INVALID_EMAIL` |
| MX lookup, only with `EMAIL_DNS_VALIDATION_ENABLED=true`. Cached for 6 h. A resolver timeout does not count against the address. | `NO_MX` |
| On the suppression list with status `ACTIVE` | `SUPPRESSED` |

If the suppression lookup itself fails, the email is sent anyway and a warning
is logged.

The checks run as a chain, so another verifier (for example Mailgun's) can be
added with `registerEmailCheck()` without touching anything else. No such
verifier is built yet.

**What suppresses an address.** A hard bounce, a spam complaint, a Mandrill
`reject`, a CSV import, or a manual block. A **soft bounce never suppresses.**
It increments `softBounceCount`, and if the address has no row yet it gets
one with status `WATCH`.

**Changing an address does not clear the old one.** If a user replaces a
bounced address, the old address stays suppressed.

**Nothing is deleted.** Lifting a suppression sets `RELEASED` and records who
lifted it in `history`.

## Suppression endpoints

All are under `/v1/messaging` and need a bearer token. The guard is
`requireDbElevated`: super-admin, `nat-support` and `nat-ict` (the configured
`ELEVATED_ROLES`).

### `GET /v1/messaging/suppressions`

Query parameters: `search` (substring of the address), `channel`
(`email`/`sms`), `status` (`ACTIVE`/`RELEASED`/`WATCH`), `reason`, `pageNo`,
`pageSize` (max 200).

```json
{
  "totalCount": 1,
  "pageNo": 1,
  "pageSize": 20,
  "records": [{
    "_id": "6650…",
    "channel": "email",
    "normalized": "someone@example.org",
    "value": "someone@example.org",
    "status": "ACTIVE",
    "reason": "HARD_BOUNCE",
    "source": "mandrill:webhook",
    "softBounceCount": 0,
    "firstDetected": "2026-11-01T09:12:03.000Z",
    "lastDetected": "2026-11-01T09:12:03.000Z",
    "createdBy": "system:webhook",
    "history": [{ "at": "…", "action": "SUPPRESSED", "reason": "HARD_BOUNCE", "source": "mandrill:webhook", "by": "system:webhook", "note": "" }]
  }]
}
```

### `POST /v1/messaging/suppressions`

```json
{ "channel": "email", "value": "someone@example.org", "reason": "MANUAL_BLOCK", "note": "asked to stop" }
```

`reason` is optional (default `MANUAL_BLOCK`). It must be one of
`HARD_BOUNCE`, `INVALID_ADDRESS`, `SPAM_COMPLAINT`, `MANUAL_BLOCK` or
`PROVIDER_SUPPRESSION`. Returns 201 with the row. If the address already has
a row, that row is reactivated rather than duplicated.

### `PATCH /v1/messaging/suppressions/:id`

```json
{ "status": "RELEASED", "note": "mailbox fixed, confirmed by phone" }
```

`status` is `ACTIVE` or `RELEASED`. `reason` may also be changed. Returns
200 with the row.

## Bounce CSV import

The same preview-then-confirm shape as the parish import.

### `POST /v1/messaging/suppressions/import/preview`

Send the file as multipart `file`, or send the CSV text as `{ "csv": "…" }`.

The import reads an `email` column, or the first column if there is no
header named `email`. It also reads an optional `reason` column (default
`HARD_BOUNCE`). Other columns are ignored. The limit is 20 000 rows.

Preview writes nothing to the suppression list.

```json
{
  "batchId": "SUP-20261108-1a2b3c4d",
  "status": "PREVIEWED",
  "counts": { "total": 6, "valid": 2, "rejected": 2, "duplicate": 1, "alreadySuppressed": 1 },
  "sample": [{ "rowNumber": 2, "email": "a@example.org", "normalized": "a@example.org", "reason": "HARD_BOUNCE", "status": "VALID", "error": "" }],
  "rejectedDownload": "/v1/messaging/suppressions/import/SUP-20261108-1a2b3c4d/rejected"
}
```

Each row gets one of these statuses: `VALID`, `REJECTED` (a bad address or
an unknown reason), `DUPLICATE` (repeats an earlier row in the file), or
`ALREADY_SUPPRESSED`.

### `POST /v1/messaging/suppressions/import/confirm`

`{ "batchId": "SUP-…" }` applies the `VALID` rows and returns
`{ batchId, status: "APPLIED", applied, counts }`. A batch applies only once.
A second confirm gets **409** `ALREADY_APPLIED`.

### `GET /v1/messaging/suppressions/import/:batchId/rejected`

Returns a CSV of the `REJECTED` and `DUPLICATE` rows, with the columns
`rowNumber,email,reason,status,error`.

## Webhooks

`/v1/webhooks` is **unauthenticated**, because providers cannot send a token.
Instead, every request must carry a valid signature, or it gets **401**
`INVALID_SIGNATURE` and nothing is read.

Events are de-duplicated in `messagingEvents` by provider and event id, so a
retried webhook is applied only once.

### `POST /v1/webhooks/resend`

- Signed with Svix headers (`svix-id`, `svix-timestamp`, `svix-signature`),
  using `RESEND_WEBHOOK_SECRET` (`whsec_…`).
- A timestamp more than 5 minutes from the server clock is refused.
- The signature covers the raw body. The JSON and urlencoded parsers keep a
  copy of the raw body for `/v1/webhooks/` paths only.
- `email.bounced` with `bounce.type` `Permanent` → `HARD_BOUNCE`.
  `Transient` or `Undetermined` → soft bounce, which is counted only.
- `email.complained` → `SPAM_COMPLAINT`.
- Other event types are recorded and otherwise ignored.
- Response: `200 { "received": true, "duplicate": false }`.

### `POST /v1/webhooks/mandrill` (and `HEAD`/`GET` for Mandrill's URL check)

- Mandrill posts form-encoded `mandrill_events` (a JSON array) with an
  `X-Mandrill-Signature` header. The signature is base64 HMAC-SHA1 over the
  webhook URL followed by every POST key and value, sorted by key.
- It is checked against `MANDRILL_WEBHOOK_KEY` and the exact public URL in
  `MANDRILL_WEBHOOK_URL`. The URL comes from configuration, not from the
  request, because behind the load balancer the request's own host and scheme
  are not the public ones.
- Event mapping: `hard_bounce` → `HARD_BOUNCE`; `soft_bounce` → counted only;
  `spam` → `SPAM_COMPLAINT`; `reject` → `PROVIDER_SUPPRESSION`.
- The event id is `<_id>:<event>:<ts>`.
- Response: `200 { "received": 2, "applied": 2, "duplicates": 0 }`.

## Test endpoints

These are **super-admin only** (`requireDbSuperAdmin`).

### `POST /v1/messaging/sms/route-test`

Works out routing only. **Nothing is sent.**

```json
{ "phone": "07400123456", "country": "GB" }
```

```json
{ "ok": true, "routingEnabled": true, "provider": "msg91", "to": "447400123456", "e164": "+447400123456", "country": "GB", "reason": "", "message": "" }
```

### `POST /v1/messaging/sms/send-test`

`{ "phone": "...", "country": "NG", "message": "optional text" }`

Sends a real SMS, but only to a number listed in `SMS_TEST_NUMBERS`
(comma-separated, compared as international digits). Any other number gets
**403** `NOT_A_TEST_NUMBER`. Returns the per-channel result.

### `POST /v1/messaging/email/send-test`

`{ "to": "you@example.org", "subject": "optional" }`

Sends a short test email through the current provider. Suppression checks
apply when they are switched on. Returns the per-channel result.

## Error codes

| Code | HTTP | Where |
|---|---|---|
| `INVALID_SIGNATURE` | 401 | webhooks |
| `NOT_A_TEST_NUMBER` | 403 | `sms/send-test` |
| `ALREADY_APPLIED` | 409 | import confirm |
| `EMPTY_FILE`, `TOO_MANY_ROWS`, `INVALID_EMAIL`, `VALIDATION_FAILED` | 400 | suppressions, import |
| `RATE_LIMITED` | 429 | password reset (see [PASSWORD_RESET_DOCS.md](./PASSWORD_RESET_DOCS.md#rate-limits)) |
| `SUPPRESSED`, `INVALID_PHONE`, … | — | the `reason` of a per-channel result, not an HTTP error |

## Collections

| Collection | Unique index | Purpose |
|---|---|---|
| `contactSuppressions` | `{channel, normalized}` | The suppression list. |
| `messagingEvents` | `{provider, eventId}` | Webhook de-duplication and an audit of what each provider reported. |
| `suppressionImportBatches` | `{batchId}` | Bounce CSV imports, held between preview and confirm. |
| `rateLimitHits` | `{key, windowStart}`, plus a TTL index on `expiresAt` | Password-reset rate limit counters. |

Production runs `MONGO_AUTO_INDEX=false`, so create the indexes with
`scripts/createMessagingIndexes.ts` (see the integration guide) **before**
switching anything on.
