# Messaging — integration guide

This guide covers setting up MSG91, Mandrill and Resend webhooks, switching
providers, and checking the result. The endpoint reference is in
[MESSAGING_DOCS.md](./MESSAGING_DOCS.md).

Values go into the **ECS task definition** (production) or the runner's
`.env.template` (dev/staging). Never commit them.

## 0. Before anything: create the indexes

Production runs `MONGO_AUTO_INDEX=false`. Run the index script against the
target database. Do a dry run first:

```bash
npx ts-node -r dotenv/config scripts/createMessagingIndexes.ts --dry-run
npx ts-node -r dotenv/config scripts/createMessagingIndexes.ts
```

The script creates the indexes for `rateLimitHits`, `contactSuppressions`,
`messagingEvents`, `suppressionImportBatches` and `birthdayReportDeliveries`.
All of these collections are new, so no unique index can fail on existing
duplicates. Nothing is dropped.

> The password-reset rate limit is **on by default**, and it still works
> before the indexes exist. Without them, though, two simultaneous first
> requests can create two counters, and expired windows are never cleaned up.
> Run the script as part of this deploy.

## 1. MSG91 (SMS outside Nigeria)

MSG91 is used **only to deliver text this service has already written**. The
service generates, stores and checks the OTP itself, exactly as before. Do
**not** set up MSG91's OTP product.

1. **Create the account** at msg91.com and complete KYC. International
   delivery may need to be enabled on the account. Ask MSG91 support if test
   sends to non-Indian numbers are refused.
2. **Get the auth key.** In the MSG91 dashboard, open *Authkey* and create
   one. Restrict it to the egress IPs of the ECS tasks if your network has
   fixed egress. Set:
   - `MSG91_AUTH_KEY=<key>`
3. **Register a sender ID** for the countries you send to. Several countries
   (the UK, the US and others) ignore or replace alphanumeric senders, and
   that is a carrier rule we cannot change. Set:
   - `MSG91_SENDER_ID=<approved sender>`
4. **Create a template (recommended).** In *SMS → Templates*, create a
   transactional template whose whole body is one variable, for example
   `##message##`. Once it is approved, set:
   - `MSG91_TEMPLATE_ID=<template id>`
   - `MSG91_TEMPLATE_VARIABLE=message` (the variable name, without the `#`s)

   With a template, the service calls the v5 flow API:

   ```
   POST https://control.msg91.com/api/v5/flow
   headers: authkey: <MSG91_AUTH_KEY>, Content-Type: application/json
   { "template_id": "<id>", "short_url": "0", "sender": "<MSG91_SENDER_ID>",
     "recipients": [ { "mobiles": "447400123456", "message": "<our text>" } ] }
   ```

   It treats `{ "type": "success" }` as `ACCEPTED` and anything else as
   `FAILED`.
5. **Without a template**, the service uses the plain send endpoint:
   `GET https://control.msg91.com/api/sendhttp.php?authkey&mobiles&message&sender&route=<MSG91_ROUTE>&country=0`.
   `MSG91_ROUTE` defaults to `4` (transactional). `country=0` because the
   number already includes its dialling code.
6. `MSG91_API_ENDPOINT` (default `https://control.msg91.com`) only needs
   changing if MSG91 gives you a different host.

**Assumptions to confirm with MSG91 before go-live.** These are written from
MSG91's public API documentation, which is inconsistent: some pages call the
template field `flow_id` and others `template_id`.

- The v5 flow endpoint accepts `template_id`, with per-recipient variables
  placed next to `mobiles`. If your account still expects `flow_id`, tell the
  developer. It is a one-line change in `sms/providers.ts`.
- A single-variable template is allowed for transactional messages to the
  destination countries. India's DLT rules do not allow this, but Nigeria is
  not routed to MSG91.
- The success response is `{ "type": "success", "message": "<request id>" }`.
  Delivery reports are not consumed yet.

## 2. Mandrill (alternative email provider)

1. In Mailchimp Transactional, **add and verify the sending domain**. Publish
   the DKIM and SPF records it gives you, then wait until both show as
   verified. Mail from an unverified domain is rejected.
2. **Create an API key** (*Settings → SMTP & API Info*). Set:
   - `MANDRILL_API_KEY=<key>`. The older name `MANDRILL_SECRET_KEY` is also
     accepted.
3. **Sender.** Set `EMAIL_FROM_ADDRESS` and `EMAIL_FROM_NAME`, or keep the
   existing `MANDRILL_FROM_ADDRESS` / `MANDRILL_FROM_NAME`. The address must
   be on the verified domain.
4. **Webhook.** In *Settings → Webhooks*, add:
   - URL: `https://<public api host>/v1/webhooks/mandrill`
   - Events: *Message Is Bounced*, *Message Is Soft-Bounced*, *Message Is
     Marked As Spam*, *Message Is Rejected*.

   Mandrill sends a `HEAD` request to the URL before saving it. The service
   answers it without a signature check. Then copy the webhook's key and set:
   - `MANDRILL_WEBHOOK_KEY=<webhook key>`
   - `MANDRILL_WEBHOOK_URL=https://<public api host>/v1/webhooks/mandrill`.
     This must match, character for character, the URL entered in Mandrill.
     It is part of what Mandrill signs.

## 3. Resend webhooks

1. In the Resend dashboard, open *Webhooks → Add endpoint*:
   - URL: `https://<public api host>/v1/webhooks/resend`
   - Events: `email.bounced`, `email.complained`. Other events are accepted
     and recorded, but nothing acts on them.
2. Copy the signing secret (`whsec_…`) and set
   `RESEND_WEBHOOK_SECRET=whsec_…`.
3. The service refuses signatures with timestamps more than 5 minutes off its
   clock. ECS tasks keep their time in sync, so this only matters when
   replaying captured requests by hand.

## 4. Switching the email provider

1. Set the new provider's key (`MANDRILL_API_KEY` or `RESEND_API_KEY`).
2. Set `EMAIL_PROVIDER=mandrill` (or `resend`).
3. Deploy. **If the key is missing or the name is misspelt, the task will not
   start.** The error names the problem, and ECS keeps the previous task
   running.
4. Send a test with `POST /v1/messaging/email/send-test` (step 6).

To switch back, set `EMAIL_PROVIDER=resend` and deploy. Leaving
`EMAIL_PROVIDER` unset keeps the old behaviour: Resend, with a missing key
reported per send rather than at startup.

## 5. Turning on country routing

1. Complete section 1, then set `MSG91_AUTH_KEY` and the sender or template.
2. Optionally set `SMS_TEST_NUMBERS=447400123456,2348031234567`, using
   international digits.
3. Check the routing **without sending anything**, using
   `POST /v1/messaging/sms/route-test`, for an NG, a GB, a US and a GH
   number.
4. Set `SMS_COUNTRY_ROUTING_ENABLED=true` and deploy. At startup the log warns
   if routing is on but a routed provider has no key.
5. Send one real test per provider with `POST /v1/messaging/sms/send-test`,
   to numbers in `SMS_TEST_NUMBERS`.

`SMS_PROVIDER_NIGERIA` (default `termii`) and `SMS_PROVIDER_INTERNATIONAL`
(default `msg91`) each accept `termii` or `msg91`, for example to send
everything through one provider during an outage.

## 6. Test checklist

Run these as a super-admin with a bearer token. Steps 1–4 send nothing
expensive.

| # | Check | Expect |
|---|---|---|
| 1 | `POST /v1/messaging/sms/route-test {"phone":"08031234567"}` | `provider: termii`, `country: NG` |
| 2 | Same with `"+447400123456"`, `"+14155552671"`, `"+233241234567"` | with routing on: `msg91`, `GB`/`US`/`GH`. With routing off: `termii` |
| 3 | Same with `"+999123"` | with routing on: `ok: false`, `reason: INVALID_PHONE` |
| 4 | `GET /v1/messaging/suppressions?pageSize=5` | 200, with `records` |
| 5 | `POST /v1/messaging/email/send-test {"to":"<your inbox>"}` | `status: ACCEPTED`, `provider` as configured. Check the From line is not `undefined` |
| 6 | `POST /v1/messaging/sms/send-test {"phone":"<a test number>"}` | `ACCEPTED`, and the SMS arrives |
| 7 | A number not in `SMS_TEST_NUMBERS` | 403 `NOT_A_TEST_NUMBER` |
| 8 | Mandrill/Resend dashboard: send a test webhook | 200. A row appears in `messagingEvents`. A test bounce suppresses the address |
| 9 | Request a password reset for a test account | Code arrives. The `PasswordReset` row has `deliveries` with the provider and status for each channel |
| 10 | Request a reset 6 times in 15 minutes for one email | the 6th gets 429 `RATE_LIMITED` with `Retry-After` |

## 7. Suppression at send time

Leave `EMAIL_SUPPRESSION_ENABLED` off until the webhooks have run for a while
and the bounce CSV export from the old provider has been imported (preview,
then confirm). When the list looks right, turn it on. From then on, sends to
suppressed and placeholder addresses are `SKIPPED` instead of being sent.

`EMAIL_DNS_VALIDATION_ENABLED=true` adds an MX lookup. This needs outbound DNS
from the tasks. Turn it on only after checking that a resolver timeout does
not slow password resets.

## Rate limiting and the load balancer

The password-reset IP limit keys on the client address. The app does **not**
set Express `trust proxy`, so `req.ip` is the ALB. Instead, the limiter reads
`X-Forwarded-For` from the right: the ALB appends the address it saw, so the
right-most entry is the one a client cannot forge. The left-most entry, which
other logs in this service record, is whatever the client sent.

`RATE_LIMIT_TRUSTED_PROXY_HOPS` (default `1`) is the number of proxies in
front of the app:

- Set it to `2` if CloudFront or another proxy sits in front of the ALB.
- Set it to `0` for a box with nothing in front of it. Otherwise a client
  could choose its own key.
