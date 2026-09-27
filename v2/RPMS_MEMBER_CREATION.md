# Member creation

There are three ways a member comes into existence. All paths are under `/api/v1/backend/members`
and need a bearer token (`Authorization: Bearer <jwt>`), except the invitation-completion calls
(section 3), which the invitee makes without one. Their full guide is
[INVITATION_COMPLETION.md](INVITATION_COMPLETION.md).

| Variation | Endpoint | Who calls it | Auth account created |
|---|---|---|---|
| **Regular member** | `POST members/save` | Parish admin | Immediately, by this API |
| **Invitation** | `POST members/addTempLiterateMember` (one) · `POST members/bulkaddTempLiterateMember` (many) | Parish admin | Not yet |
| **Invitation completion** | `GET members/getTempMemberDetails/{parish_code}/{code}` then `POST members/saveTempMemberDetails` | The invited person | On completion, by this API |

**Passwords are never returned.** No response from this API contains `password` or
`temp_password`, on any route; passwords reach members only by email.

Every member has a login account in the central auth service. This API creates that account
itself, using the configured `AUTH_SERVICE_URL` and `AUTH_SERVICE_API_KEY`. The client never calls
the auth service.

---

## How creation works

```
check email + phone (here and in the auth service)
  → create the auth account
    → create the member here, linked by user_code = auth account id
      → if that fails: delete the auth account, return the original error
```

- Nothing is created if the email or phone is already in use.
- Nothing is created locally if the auth account cannot be created.
- Two identical requests at the same time: one succeeds, the other gets `409`.
- If the rollback delete also fails, the request answers `500` with a reference id and the failure
  is logged for manual reconciliation.

**Transition.** Sending `user_id` (an auth account id the client created itself) still works and
skips all of the above. It is deprecated: omit it.

---

## 1. Regular member — `POST members/save`

A body **without `id`** creates. With `id` it updates an existing member (unchanged, not covered
here).

### Request

```json
{
  "title": "Sis",
  "first_name": "Ngozi",
  "last_name": "Okonkwo",
  "gender": "Female",
  "marital_status": "Single",
  "date_of_birth": "1990-05-01",
  "email": "ngozi@example.org",
  "phone": "07012345678",
  "country_code": "44",
  "church": "RCCG Victory House",
  "country": "United Kingdom",
  "state": "Greater London",
  "city": "Croydon",
  "address": "14 Sumner Road",
  "parish_code": "211414",
  "area_code": "2114",
  "zone_code": "211",
  "prov_code": "21",
  "region_code": "2",
  "subcont_code": "UK",
  "cont_code": "EU",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House"
}
```

| Field | Notes |
|---|---|
| All fields above | Required. `date_of_birth` is `YYYY-MM-DD`; `phone` is digits with an optional leading `+`. |
| `email` | Required on create. Stored lower-cased; it is also the login username. |
| `password` | Optional. Generated server-side (12 characters) when omitted. Mailed to the member either way. |
| `user_id` | Deprecated. Omit it. |
| `frontendUrl` | Optional. See [Which frontend the links point at](#which-frontend-the-links-point-at). |
| Optional profile fields | `passport`, `whatsapp_phone`, `country_code_whatsapp`, `postcode`, `county`, `landmark`, `date_of_marriage`, `parish_designation`, `ord_status`, `dept`, `facebook`, `instagram`, `twitter`, and the other optional columns. |

The hierarchy codes must sit inside the caller's own scope, or the request is refused with `403`.

### Response — `201 Created`

```json
{
  "message": {
    "success": "Member Added Successfully",
    "user_code": "64f1c2a9e4b0a1b2c3d4e5f6",
    "fullname": "Ngozi Okonkwo",
    "rccg_code": "RCCG1234567890"
  }
}
```

`user_code` is the auth account id. The member then receives a welcome email (login URL, username,
password, membership code) and an SMS. A failed email or SMS does not undo the creation.

---

## 2. Invitation — `POST members/addTempLiterateMember`

Creates a pending invitation and emails the person a registration link. No account exists yet.

### Request

```json
{
  "email": "invitee@example.org",
  "first_name": "Ngozi",
  "last_name": "Okonkwo",
  "phone": "07012345678",
  "phone_code": "44",
  "parish_code": "211414",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House"
}
```

Required: `email`, `first_name`, `last_name`, `phone`, `parishPastorName`, `parishName`. The other
hierarchy codes and `*Name` fields are optional.

### Response — `201 Created`

```json
{
  "message": {
    "success": "Member Registration Link has been sent successfully",
    "passcode": "9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f",
    "fullname": "Ngozi Okonkwo",
    "email": "invitee@example.org"
  }
}
```

The emailed link is
`{frontend}/complete-membership-registration/{parish_code}/{passcode}`, where `{frontend}` is the
calling frontend (see below). The code is random and
single-use. Links issued by this API expire after `MEMBER_INVITE_TTL_DAYS` (default 30).

### Inviting someone who already has a pending invitation

If the email already has an invitation that has not been completed, nothing new is created. The
**same link** is emailed again and the answer is `200`:

```json
{
  "message": {
    "success": "A registration link was previously sent to this email and is still pending. The same link has been sent again.",
    "resent": true,
    "passcode": "9f1c1d2e-3b4a-4c5d-8e6f-7a8b9c0d1e2f",
    "fullname": "Ngozi Okonkwo",
    "email": "invitee@example.org"
  }
}
```

- The link's validity window restarts from the re-send.
- Check `resent` (or the `200` status) to tell a re-send from a new invitation (`201`).
- A pending invitation from **another parish** is not re-sent: `422`.
- An invitee who is completing their registration at that moment: `409`.
- An email whose invitation was already used, or that belongs to a member: `422` as before.

### Bulk — `POST members/bulkaddTempLiterateMember`

```json
{
  "action": "check",
  "members": [
    { "email": "a@example.org", "first_name": "Ada", "last_name": "Obi", "phone": "07000000001" },
    { "email": "b@example.org", "first_name": "Bayo", "last_name": "Ade", "phone": "07000000002" }
  ],
  "parish_code": "211414",
  "parishPastorName": "Pastor Ade Balogun",
  "parishName": "RCCG Victory House"
}
```

- `action: "check"` grades every email and writes nothing. It answers `200`.
- `action: "submit"` sends an invitation for each available email. A row whose email has a
  pending invitation is graded `available` and re-sent with its original link; its result carries
  `"resent": true`. It answers `201` when every row
  was sent, or `207` when any row was skipped or failed.

Each row in `results` carries `status` (`available`, `not_available`, `success`, `skipped`,
`failed`) and a `reason` when not available: `missing_email`, `invalid_email`,
`duplicate_in_upload`, `email_exists_in_members`, `email_exists_in_temp_members`, or
`email_exists_in_auth`.

---

## 3. Invitation completion

The invitee has no session, so none of these calls takes a token; the invitation link is the
credential. The email is locked to the invited address, and the parish and hierarchy come from the
invitation, not the body. Page-level guide, including the frontend changes:
[INVITATION_COMPLETION.md](INVITATION_COMPLETION.md).

### Step 1 — load the invitation: `GET members/getTempMemberDetails/{parish_code}/{code}`

`200` returns the invitation details (name, email, phone, hierarchy) to prefill the form, with
`"message": "Welcome, Proceed with your registration"`.

**No token needed.** The person opening the link has no account yet, so this route is public. The
parish code and the invitation code must both match, and the code is random, single-use and
expires, so together they act as the credential. Lookups are limited to 20 per minute per IP
(`429` beyond that). Other answers: `404` no pending invitation for that parish and code, `410`
link already used or expired, `403` a member already exists with that email.

### Step 2 — complete it: `POST members/saveTempMemberDetails`

The same body as a regular member (section 1), plus:

| Field | Notes |
|---|---|
| `temp_member_id` | Required. The `code` from the link. |
| `consent_code` | Required. The data-consent reference. |
| `email` | Required. Must be the invited address, or `422`. |
| `password` | Ignored when this API creates the account: it always generates the password and emails it. Used only on the `user_id` path (below). |

### Response — `201 Created`

```json
{
  "message": {
    "success": "Your Membership Profile was created successfully, check your email for your login details",
    "user_code": "64f1c2a9e4b0a1b2c3d4e5f6",
    "fullname": "Ngozi Okonkwo",
    "rccg_code": "RCCG1234567890"
  }
}
```

The invitation is marked completed, the member is stored with `registration_mode:
"INVITATION_LINK"`, and the welcome email and SMS are sent. The link cannot be used again.

The welcome email carries the login details: username (the email), the password and the
membership code. This holds on both paths. When this API creates the auth account, the password
is always generated by the server. When the page has already created the account and sends
`user_id`, the email uses the `password` in the body (the one the page set on that account), so
the page must send it. Without it no
welcome email goes out, rather than one with a blank password.

---

## 4. Resending the welcome email

`POST members/sendWelcomeEmail` (token required) sends a member their welcome email again: the
username (their email), their login password and membership code, with a login link. Use it when
the first email never arrived.

```json
{
  "user_code": "64f1c2a9e4b0a1b2c3d4e5f6",
  "parishName": "RCCG Victory House",
  "parishPastorName": "Pastor Adeyemi"
}
```

Only `user_code` is required. The login link follows the same rules as the other emails (see
below).

The password sent is **the one stored when the member was created**. No new password is
generated, because this API cannot change the password on the member's login account, so a new
one would not work.

| Status | Meaning |
|---|---|
| `200` | `{ "message": "Welcome email sent to ngozi@example.org" }` |
| `404` | No member with that `user_code` in the caller's scope |
| `422` | The member has no email address (`errors.email`), or no password is on file (`errors.password`); for the second, the member should use "Forgot password" |
| `503` | Mail is not configured, or delivery failed |

---

## Which frontend the links point at

Several frontends share this API, so the welcome email's login link and the invitation's
registration link point at the frontend that sent the request. The first of these that is an
**allowed origin** wins:

1. `frontendUrl` in the request body (may include a base path, e.g. `https://uk.portal.org/app`)
2. the `Origin` request header, which browsers send automatically
3. the origin of the `Referer` request header
4. the server's `FRONTEND_URL` setting

Allowed origins are `ALLOWED_FRONTEND_ORIGINS`, or `CORS_ORIGINS` when that is empty. A value that
is not allowed is skipped, not rejected. A browser frontend usually needs to send nothing: its
`Origin` header identifies it.

An invitation that cannot be linked to any frontend is refused with `422` and nothing is written.

---

## Error responses

Every error uses the same body:

```json
{
  "statusCode": 422,
  "message": "Member email already exists.",
  "error": "Unprocessable Entity",
  "errors": { "email": ["Member email already exists."] },
  "path": "/api/v1/backend/members/save",
  "timestamp": "2026-09-24T10:15:00.000Z"
}
```

`errors` appears only on field-level failures.

| Status | When | `message` |
|---|---|---|
| `401` | Missing or invalid token (on invitation calls: no token and no invitation) | `Unauthorized` |
| `403` | Hierarchy code outside the caller's scope | `parish_code … is outside your scope (…)` |
| `404` | Unknown invitation code | `Invalid Registration Link, Make sure you are using a valid registration link` |
| `409` | Same first-11-characters name already in the parish | name-prefix message |
| `409` | The same email is being created by another request | `A member with this email is already being created. Please wait and refresh.` |
| `409` | A previous attempt for this email needs reconciliation | `A previous attempt to create this member did not complete and is awaiting reconciliation. …` |
| `409` | The same invitation is being completed | `This registration is already being completed. Please wait a moment and log in.` |
| `410` | Invitation already used | `This registration link has already been used. Please log in.` |
| `410` | Invitation expired | `This registration link has expired. Please ask your parish for a new invitation.` |
| `410` | Invitation has no parish (some legacy rows) | `This registration link is incomplete. Please ask your parish to send a new invitation.` |
| `422` | Completion or email check with an address other than the invited one | `The email must be the one this invitation was sent to.` (`errors.email`) |
| `429` | Too many invitation calls from one IP (20/min on lookup and email check, 10/min on completion) | rate-limit message |
| `422` | Invalid fields | per-field `errors` |
| `422` | Email already in use | `Member email already exists.` (`errors.email`) |
| `422` | Phone already in use | `Member phone number already exists.` (`errors.phone`) |
| `422` | Invitation email already used or registered | `The email has already been taken.` (`errors.email`) |
| `422` | Pending invitation belongs to another parish | `This email already has a pending invitation from another parish.` (`errors.email`) |
| `409` | Re-invite while the invitee is completing registration | `This person is completing their registration right now. …` |
| `422` | Invitation with no determinable frontend | `Cannot determine which frontend to link to. …` (`errors.frontendUrl`) |
| `500` | Creation failed and the auth account could not be removed | `Member creation failed and could not be fully reversed. Support has been notified. Reference: <request id>` |
| `502` | The auth service refused the details | `The authentication service rejected the member details: <reason>` |
| `503` | The auth service is down | `The authentication service is unavailable. Please try again shortly.` |
| `503` | No auth service is configured on the server | `Member provisioning is not configured on this server.` |
| `504` | The auth service timed out | `The authentication service did not respond in time. Please try again.` |

For `5xx` answers, the `x-request-id` response header identifies the request in the server logs.
