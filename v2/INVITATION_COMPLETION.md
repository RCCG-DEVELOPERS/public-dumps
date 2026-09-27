# Completing an invitation (the invitee's registration page)

This is for the page at `/complete-membership-registration/:parishCode/:code`, where an invited
person finishes their own registration.

The invitee has **no account and no session**, so the page sends **no token**. The invitation link
is the credential. Its parish code and its invitation code (`code`) together prove the person was
invited, much like a password-reset link.

All paths are under `https://rpms-api-01.rccgportal.org/api/v1/backend`.

---

## The flow

```
Invitation email  ──►  /complete-membership-registration/{parish_code}/{code}
                               │
   1. GET  members/getTempMemberDetails/{parish_code}/{code}   → prefill the form
   2. POST members/checkDuplicateMail   (optional)             → confirm the email is free
   3. POST members/saveTempMemberDetails                       → create the member + login
                               │
          Welcome email (username, password, membership code) + SMS
```

None of these three calls needs an `Authorization` header.

---

## Rules the API enforces

The code is the only proof of identity, so the invitation decides who registers and where. The
request body does not.

| Rule | Effect |
|---|---|
| **The email is locked** | It must be the address the invitation was sent to; letter case and surrounding spaces don't matter. Any other address gets `422` and nothing is created. Show the email read-only. |
| **The parish comes from the invitation** | `parish_code` in the body is ignored. The member joins the invitation's parish. |
| **The hierarchy comes from the invitation** | `area_code`, `zone_code`, `prov_code`, `region_code`, `subcont_code` and `cont_code` come from the invitation. For older invitations that hold only the parish, they come from an existing member of that parish. The body is used only for a level neither source has. |
| **Single use** | A completed link answers `410`. Two submissions at the same moment: one wins, the other gets `409`. |
| **Expiry** | Links expire `MEMBER_INVITE_TTL_DAYS` days (default 14) after they were sent or last re-sent, then answer `410`. Invitations migrated from the old system do not expire. |
| **Rate limits (per IP)** | 20 per minute on steps 1 and 2, 10 per minute on step 3. Over the limit: `429`. |

---

## Step 1: load the invitation

```
GET members/getTempMemberDetails/{parish_code}/{code}
```

Take both values from the page URL.

**`200`**

```json
{
  "data": {
    "code": "ddb95939-b64b-4d9c-a04b-720d668cd39f",
    "email": "ngozi@example.org",
    "first_name": "Ngozi",
    "last_name": "Okonkwo",
    "phone_code": "44",
    "phone": "07012345678",
    "parish_code": "632101",
    "parishName": "RCCG Victory House",
    "parishPastorName": "Pastor Adeyemi",
    "area_code": "AR444077283",
    "zone_code": "ZN44400171",
    "province_code": "UKR03PR02",
    "region_code": "UKREG03",
    "subcontinent_code": "EC09SC01",
    "continent_code": "EC0009"
  },
  "message": "Welcome, Proceed with your registration"
}
```

Prefill the form from `data`, and show `email` read-only.

| Status | Meaning | What to show |
|---|---|---|
| `404` | No invitation for that parish and code | "This registration link is not valid." |
| `410` | Already used, or expired | The `message` (it says which, and what to do) |
| `403` | A member already exists with that email | "You already have an account. Please log in." |
| `429` | Too many attempts | "Please wait a minute and try again." |

---

## Step 2 (optional): check the email

```
POST members/checkDuplicateMail
```

```json
{
  "email": "ngozi@example.org",
  "parish_code": "632101",
  "temp_member_id": "ddb95939-b64b-4d9c-a04b-720d668cd39f"
}
```

Without a token, `parish_code` and `temp_member_id` are **required**, and only the invitation's
own email can be checked. This stops the route from being used to test which addresses are
registered. Signed-in admin screens keep calling it with a token and just `{ "email" }`, as
before.

| Status | Meaning |
|---|---|
| `200` | `{ "message": "Email does not exist, You can proceed" }` |
| `422` | The email is already registered, or it is not the invitation's email (`errors.email`) |
| `401` | No token and no invitation in the body |
| `404` / `410` | As in step 1 |

Step 1 already answers `403` when the invited email belongs to a member, and step 3 checks again.
The page can therefore drop this call entirely.

---

## Step 3: complete the registration

```
POST members/saveTempMemberDetails
```

Send the same fields as a regular member (see `MEMBER_CREATION.md`, section 1), plus:

| Field | Required | Notes |
|---|---|---|
| `temp_member_id` | Yes | The `code` from the link |
| `consent_code` | Yes | The data-consent reference |
| `email` | Yes | Must be the invitation's email (see the rules above) |
| `password` | No | Leave it out. The server always generates a random 12-character password and emails it to the invitee; a value sent here is ignored. |

Leave out `user_id` and do **not** call the auth service. This API creates the login account
itself with a password it generates. See "Changes to the current page" below.

**`201`**

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

Then:

- the login account exists in the auth service;
- the member is stored in the invitation's parish with `registration_mode: "INVITATION_LINK"`;
- the invitation is marked completed, so the link stops working;
- the **welcome email** goes to the invited address with the username (the email), the generated
  password and the membership code, plus a welcome SMS. The page never sees the password.

If any step fails, the login account is removed again and the link stays usable, so the invitee
can simply retry.

| Status | Meaning | What to show |
|---|---|---|
| `422` | A field is invalid, the email isn't the invited one, or the email/phone is already in use | The `message`, and `errors.<field>` next to the field |
| `404` | Unknown invitation code | "This registration link is not valid." |
| `409` | Name clash in the parish, or this link is being completed right now | The `message` |
| `410` | Link used, expired, or incomplete (no parish on it) | The `message` |
| `429` | Too many attempts | "Please wait a minute and try again." |
| `502` / `503` / `504` | The auth service refused, is down or timed out | "Something went wrong, please try again shortly." |

Every error body has the same shape (`statusCode`, `message`, `error`, optional `errors`, `path`,
`timestamp`). For `5xx` answers, the `x-request-id` response header identifies the request in the
server logs.

---

## Changes to the current page

`src/content/common/rpms/MembersTemp/PageHeader.js` in the frontend.

**Required now.** Without this the page's email check fails with `401`:

1. `checkDuplicateEmail`: add `parish_code: parishCode` and `temp_member_id: code` to the body. Or
   remove the call; see step 2.
2. Make the email field read-only, filled from step 1.

**Recommended.** Move to server-side account creation:

3. Stop calling the auth service from this page: `POST /users/query`, `POST /usersTemp`, and the
   rollback `DELETE /usersTemp/:id`.
4. Send `saveTempMemberDetails` **without** `user_id` and without `password`, remove the page's
   own `generatePassword(8)`, and drop the browser rollback.
5. Stop sending `frontendUrl`; the API works it out from the request's `Origin`.

Until step 4 ships, the page's current behaviour keeps working. It generates a password, creates
the account itself and sends `user_id` with that `password`. The API still binds the email and
parish to the invitation and emails that password, the only one that works for an account the
page created. After the switch, the auth service's anonymous `/usersTemp` routes
can be locked to this API's key.

---

## Why a link is safe to trust

- The invitation code is a random UUID (122 bits), generated server-side and never listed
  anywhere.
- The parish code in the URL must match the code's own parish.
- A link works once (it is claimed atomically) and expires.
- Guessing is rate-limited per IP.
- What a link can do is fixed by the invitation: one person, one email, one parish. Someone who
  obtains a forwarded link can only register as the invited person, into the invited parish. The
  welcome email with the password always goes to the invited address.
