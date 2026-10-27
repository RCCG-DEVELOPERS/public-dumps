# First-timer QR self-service check-in

The public check-in page (`/first-timer/check-in?p=<parish_code>&…`) is opened by a visitor who
has scanned a parish's QR code. They have no account and no token. Posting that form to
`POST /api/v1/backend/firsttimers/save` returns **401**, because that route is an officer's route
and every officer route in this API requires a token.

Use the public route instead.

## Endpoint

```
POST /api/v1/backend/firsttimers/public/save
Content-Type: application/json
```

No token and no auth header of any kind. The route is rate-limited to **20 requests per minute
per IP**, so a 429 is possible under a burst; the global default for every other route is 60.

The body the page already builds (`buildFirstTimerPayload` in
`src/content/common/rpms/FirstTimers/shared.js`) is accepted as-is. The only frontend change is
the URL in `src/content/pages/FirstTimerCheckIn/index.js`:

```js
await axios.post(
    `${process.env.REACT_APP_RPMS_ENDPOINT_BASE_URL_VERSIONED}/firsttimers/public/save`,
    buildFirstTimerPayload(values, checkInContext)
);
```

## What the API reads from the body

| Field | Rule |
|---|---|
| `parish_code` | **required** — must name a parish with at least one member on record, else 404 |
| `first_name`, `last_name`, `gender` | required |
| `service_date`, `service_type` | required (`service_date` as `YYYY-MM-DD`) |
| `phone` | required, digits only |
| `email` | required, stored as typed |
| `country`, `state`, `city`, `address`, `postcode` | required |
| `landmark`, `county`, `whatsapp` | optional; `whatsapp` digits only when present; blank is fine |
| `altar_call`, `preferred_contact`, `service_feedback`, `prayer_point`, `improvement` | optional |
| `service_feedback_score` | optional integer 0–5 |
| `source` | optional; only `qr_self_service` is accepted and it is the default |

Everything else in the payload (`sn`, the camelCase duplicates such as `firstName` and
`whatsAppLine`, `parish`, `area`, `zone`, `province`, `region`, `subContinent`, `continent`,
`country_code`, `country_code_whatsapp`, `stateCounty`) is **ignored**. It is safe to keep sending
it. `first_timers` has no hierarchy columns above `parish_code`, so the hierarchy codes in the QR
URL are only used by the page itself.

Officer-side fields (`id`, `type`, `followee_code`, `follower_code`, `hf_attendance_id`, the
follow-up block) are never read on this route, whatever the body contains.

## Responses

| Status | Meaning | Body |
|---|---|---|
| 201 | Created | `{"message":"First timer Record Saved Successfully","ref":"FT…"}` |
| 200 | Already registered: same `email` (case-insensitive) on the same `service_date` in the same parish. Nothing was inserted. | same shape, with the existing `ref` |
| 404 | `parish_code` names no parish | `{"message":"Parish not found."}` |
| 422 | Validation failed | Laravel-shaped `errors` map, e.g. `{"errors":{"first_name":["The first name field is required."]}}` |
| 429 | More than 20 requests per minute from one IP | |

Treat 200 and 201 both as success. The 200 is what a double tap on the submit button produces,
so the page should not show it as an error.

## What the stored row looks like

Identical to one an officer saves through `firsttimers/save`, with `type: "Parish"` and a minted
`ref`, plus two columns only this route writes:

- `source: "qr_self_service"`
- `service_feedback_score: <0–5 or null>`

Officers see the record in the normal first-timers list for their parish.

## Smoke test

```bash
export BASE_URL=https://<api-host>
curl -i -X POST "$BASE_URL/api/v1/backend/firsttimers/public/save" \
  -H 'Content-Type: application/json' \
  -d '{"parish_code":"721487","first_name":"Wale","last_name":"Ojo","gender":"Male","service_date":"2026-10-03","service_type":"Sunday","phone":"2349099392941","email":"wale@example.com","country":"NG","state":"Abia","city":"Umuahia","address":"14 Grove Road","postcode":"34444","service_feedback_score":5}'
```

First call: 201 with a `ref`. Same call again: 200 with the same `ref`. A made-up `parish_code`:
404.
