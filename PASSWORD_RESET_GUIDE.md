# Password reset — endpoint guide

Every way a password can be set in this system, who may do it, what to send and
what comes back.

## Read this first

Eight endpoints, all of them built. Three groups.

| Group | Endpoints | Who calls them |
|---|---|---|
| **Self-service** | request a code, reset with it, change your own password | Anybody |
| **Administrative** | check eligibility, reset someone's password | Administrators over their own hierarchy |
| **Support** | look a request up, step up, read a code out | Super admin and national support only |

Every path below is mounted under the API host. The administrative and support
endpoints all sit under `/v1/password-resets`, on their own router.

---

## Contents

- [Who can do what](#who-can-do-what)
- [Request a reset code](#request-a-reset-code)
- [Reset with the code](#reset-with-the-code)
- [Change your own password](#change-your-own-password)
- [Force someone to change their password](#force-someone-to-change-their-password)
- [Check whether you may reset someone](#check-whether-you-may-reset-someone)
- [Reset someone's password](#reset-someones-password)
- [Find a reset request](#find-a-reset-request)
- [Read the code out to a caller](#read-the-code-out-to-a-caller)
- [Rules that decide every reset](#rules-that-decide-every-reset)
- [All error codes](#all-error-codes)
- [Frontend guidance](#frontend-guidance)
- [What this does not do](#what-this-does-not-do)

---

## Who can do what

Reading down: can the person on the left reset the password of the person along the top?

| Actor | Member in their unit | Admin one level below | Peer admin | Admin above | Sensitive role | Themselves |
|---|---|---|---|---|---|---|
| Super Admin | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| National Support | ✅ | ✅ | ✅ | ✅ | ❌ | ❌ |
| Continent Admin | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Sub-Continent Admin | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Regional Admin | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Province Admin | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ |
| Parish Pastor (`pic-parish`) | ✅ own parish only | — | ❌ | ❌ | ❌ | ❌ |
| Area Admin | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Zone Admin | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |

Four things this table is saying.

**A parish pastor reaches their own parish and nothing else.** Every member
whose `parish` code equals the pastor's own. Not the next parish, not the area,
not the zone. If a member has no parish code on their record, the pastor cannot
reach them.

**Area and zone administrators cannot reset anyone.** That was a decision, not
an oversight. They can still raise it with the province.

**Nobody resets a peer.** A province admin cannot reset another province admin,
even one inside their own region. That takes a regional admin, national support
or a super admin.

**National Support is bound by the sensitive rule.** They can reset a regional
admin, but not the National Treasurer. Only a super admin can. See
[Rules that decide every reset](#rules-that-decide-every-reset).

Nobody resets their own password through an administrative endpoint. Use
[change your own password](#change-your-own-password) instead.

---

## Request a reset code

Someone has forgotten their password and wants a code sent to them. No login
required.

```
POST /auth/request-password-reset
```

**Who** Anyone. No token.

### Request

Send **either** `email` **or** `phone`, not both.

```json
{
  "email": "pastor.adeyemi@rccg.org"
}
```

```json
{
  "phone": "08031234567"
}
```

> **`phone` must be the person's login username, not any phone number on their
> profile.** The lookup is `username = phone`. If they log in with an email
> address, sending their mobile number here returns "does not exist".

### Response — 200

```json
{
  "success": true,
  "message": "OTP sent to pastor.adeyemi@rccg.org",
  "reference": "PR-4K7MQ-2XB9T"
}
```

**Show the reference to the user.** It is what support asks for when the code
does not arrive, and it authenticates nothing on its own, so it is safe to
display, print and read aloud. It is absent only if the support record could not
be written, which never blocks the reset itself.

Requesting by phone also emails the code if the account carries an email
address, and the message says so:

```json
{
  "success": true,
  "message": "OTP sent to 08031234567, and email (if available on your portal profile)."
}
```

### Response — 400

```json
{ "success": false, "message": "Email does not exist" }
```

| Message | Cause |
|---|---|
| `Email does not exist` | No account with that email |
| `Phone number does not exist as a login username` | The number is not their username |
| `Email or phone is required` | Neither field sent |
| `Too many failed attempts. Try again after 27 minute(s).` | Locked out from five wrong codes |
| `We could not send your code right now. Please try again shortly.` | No delivery channel succeeded |

That last one is important. It means the code was created but **no email or SMS
went out**. Do not tell the user to check their inbox. Ask them to try again.

### What happens behind it

- A six-digit code is generated with a cryptographic random source.
- It is valid for **15 minutes**.
- **Every earlier unused code for that identifier is retired.** Only the newest
  one works, so a person who clicks "resend" three times has one live code, not
  three.
- The failed-attempt count **carries across** a new request. Asking for a fresh
  code does not clear a lockout.

### Use case

> A pastor cannot sign in. The app shows "Forgot password", they type their
> email, and this is called. They get a six-digit code by email and have fifteen
> minutes to use it.

---

## Reset with the code

```
POST /auth/reset-password
```

**Who** Anyone holding a valid code. No token.

### Request

Identify the same way as the request step, and send the code.

```json
{
  "email": "pastor.adeyemi@rccg.org",
  "otp": "418205",
  "newPassword": "Harvest2026!",
  "confirmPassword": "Harvest2026!"
}
```

| Field | Rule |
|---|---|
| `email` or `phone` | One of them, matching the request step |
| `otp` | The six digits. Leading zeros are real — send `"048120"`, never `48120` |
| `newPassword` | At least **8 characters** |
| `confirmPassword` | Must equal `newPassword` |

### Response — 200

```json
{
  "success": true,
  "message": "Password reset successful — 3 other session(s) signed out."
}
```

The count appears only when there were sessions to end.

### Response — 400

```json
{ "success": false, "message": "Invalid or expired OTP" }
```

| Message | Cause |
|---|---|
| `Passwords do not match` | The two password fields differ |
| `New password must be at least 8 characters long` | Too short |
| `Invalid or expired OTP` | Wrong code, expired code, or already used |
| `Too many failed attempts. Try again after 22 minute(s).` | Five wrong codes; locked 30 minutes |
| `User not found` | The identifier matches no account |

`Invalid or expired OTP` is deliberately one message for four different causes.
Do not try to tell the user which one it was.

### What happens on success

- The password is replaced.
- Any outstanding "must change password" requirement is cleared.
- **Every session on the account is signed out**, including any an attacker held.
  The person must sign in again with the new password.

### Use case

> The pastor types the code from their email and a new password. All their
> devices are signed out and they sign in fresh.

---

## Change your own password

```
POST /auth/change-password
```

**Who** Any signed-in user, for their own account only.

### Request

```json
{
  "currentPassword": "OldHarvest2025",
  "newPassword": "Harvest2026!",
  "confirmPassword": "Harvest2026!"
}
```

Bearer token in the `Authorization` header. The account is taken from the token,
so there is no user id to send.

### Response — 200

```json
{
  "message": "Password changed successfully",
  "otherSessionsRevoked": 2
}
```

**The caller stays signed in.** Their other devices do not. That is the
difference from a reset, where everything is signed out.

### Response — 400

| Message | Cause |
|---|---|
| `currentPassword, newPassword and confirmPassword are all required` | A field is missing |
| `New passwords do not match` | The two differ |
| `New password must be at least 8 characters long` | Too short |
| `New password must be different from the current password` | They sent the same one |
| `Current password is incorrect` | Verification failed |

### Also live at a second address

```
POST /v1/users/{id}/change-password
```

The same operation, and `{id}` **must be your own user id**. Another id returns:

```json
{ "message": "You may only change your own password on this endpoint." }
```

Prefer `/auth/change-password`. It needs no id and cannot be called wrongly.

### Use case

> A finance officer wants a stronger password. They enter the old one and a new
> one, stay signed in on the machine they are using, and their forgotten session
> on a shared office computer is signed out.

---

## Force someone to change their password

This does **not** set a password. It marks the account so the person must choose
a new one at their next sign-in.

```
PATCH /v1/users/{id}/force-password-change
```

**Who** Super admin only.

### Request

```json
{
  "required": true,
  "reason": "Shared credentials reported by the province office"
}
```

| Field | Default | Meaning |
|---|---|---|
| `required` | `true` | `true` sets the requirement, `false` clears it |
| `reason` | — | Optional, recorded in the audit log |

### Response — 200

```json
{
  "message": "User must set a new password before continuing",
  "userId": "64b7f0c2f1a2b3c4d5e6f701"
}
```

### Use case

> Two people are sharing one login. A super admin flags the account. The holder
> is made to set a new password next time they sign in, and nobody has to learn
> a temporary one over the phone.

---

## Check whether you may reset someone

Answers "am I allowed?" without doing anything. Always returns 200, whether the
answer is yes or no, so asking is never treated as an attack.

```
GET /v1/password-resets/users/{id}/eligibility
```

**Who** Super admin, national support, continent, sub-continent, regional or
province admin, or a parish pastor.

### Response — 200, allowed

```json
{
  "allowed": true,
  "via": "province:PR0042",
  "code": "",
  "message": "",
  "target": {
    "id": "64b7f0c2f1a2b3c4d5e6f701",
    "username": "grace.okonkwo",
    "fullName": "Grace Okonkwo",
    "email": "grace.okonkwo@rccg.org",
    "parish": "PA015520",
    "area": "AR0301",
    "zone": "ZN0140",
    "province": "PR0042",
    "region": "R11",
    "roles": ["rpms-member"],
    "status": "1",
    "userStatus": "ACTIVE"
  }
}
```

`via` names the unit the permission came through, and is the same string that
goes into the audit log. A super admin gets `"super-admin"`, national support
gets `"elevated:nat-support"`.

### Response — 200, refused

```json
{
  "allowed": false,
  "via": "",
  "code": "TARGET_OUTSIDE_UNIT",
  "message": "This person is not inside any unit you administer.",
  "target": {
    "id": "64b7f0c2f1a2b3c4d5e6f702",
    "username": "daniel.eze",
    "fullName": "Daniel Eze"
  }
}
```

### Use case

> A province admin opens a member's profile. The app calls this first and shows
> or hides the "Reset password" button. No refusal is ever shown as an error.

---

## Reset someone's password

```
POST /v1/password-resets/users/{id}
```

**Who** The same list as eligibility. The full rules are in
[Rules that decide every reset](#rules-that-decide-every-reset).

### Request

```json
{
  "reason": "Member called the province office, cannot access their email"
}
```

| Field | Required | Rule |
|---|---|---|
| `reason` | **yes** | At least 5 characters. Recorded in the audit log |
| `newPassword` | no | Omit it and the server generates one. If you send it, at least 8 characters |

Omitting `newPassword` is the recommended path. A generated password is 16
characters from an alphabet with no look-alikes, so nobody confuses a zero for
an O while reading it out.

### Response — 200

```json
{
  "success": true,
  "temporaryPassword": "Kpna-7Rtq4Vbx2Wm",
  "mustChangePassword": true,
  "sessionsRevoked": 2,
  "via": "province:PR0042",
  "target": {
    "id": "64b7f0c2f1a2b3c4d5e6f701",
    "username": "grace.okonkwo",
    "fullName": "Grace Okonkwo"
  }
}
```

> **`temporaryPassword` is returned once and never again.** It is not stored in
> readable form and never appears in the audit log. If it is lost, reset again.

The person must change it at next sign-in, and every session they had is signed
out.

### Response — 403

```json
{
  "success": false,
  "code": "TARGET_ROLE_SENSITIVE",
  "message": "This person holds a role that only a super administrator may reset."
}
```

### Use case

> A member in Province PR0042 has lost access to the email on their account, so
> the code cannot reach them. The province admin resets it, reads the temporary
> password to them on the phone, and the member is made to set their own at the
> next sign-in. The audit log records who did it, to whom, and why.

> A parish pastor does the same for a member of their own parish. A member of
> the parish next door returns `TARGET_OUTSIDE_UNIT`.

---

## Find a reset request

Lets support see the state of somebody's reset request. **It does not show the
code.** That is the next endpoint.

```
GET /v1/password-resets/lookup?phone=08031234567
GET /v1/password-resets/lookup?email=pastor.adeyemi@rccg.org
GET /v1/password-resets/lookup?reference=PR-4K7MQ-2XB9T
```

**Who** Super admin and national support only. Deliberately not the
administrators who may reset a password: resetting leaves a trail and ends every
session, while reading out a live code is a larger power that stays with the
national desk.

Give one identifier. A reference wins if you send more than one, then phone,
then email. Returns up to ten records, newest first. Phone numbers are folded
before matching, so `08031234567`, `+2348031234567` and `2348031234567` all
find the same person.

### Response — 200

```json
{
  "found": true,
  "records": [
    {
      "reference": "PR-4K7MQ-2XB9T",
      "requestedAt": "2026-09-17T09:14:02.000Z",
      "channel": "email",
      "maskedEmail": "pas•••••@rccg.org",
      "maskedPhone": "0803•••4567",
      "user": {
        "fullName": "Grace Okonkwo",
        "username": "08031234567",
        "parish": "PA015520",
        "province": "PR0042"
      },
      "status": "PENDING",
      "otpExpiresAt": "2026-09-17T09:29:02.000Z",
      "expired": false,
      "attempts": 1,
      "lockedUntil": null,
      "disclosureCount": 0,
      "disclosable": true
    }
  ]
}
```

Never returned: the code, any hash of it, the encrypted form, the unmasked phone
or email, the caller's IP address.

| `status` | Meaning |
|---|---|
| `PENDING` | Issued, unused, still valid |
| `VERIFIED` | Code accepted, password not yet set |
| `COMPLETED` | Password was reset |
| `EXPIRED` | The 15 minutes ran out |
| `SUPERSEDED` | A newer request replaced it |
| `LOCKED` | Five wrong attempts |

### Response — 200, nothing found

```json
{ "found": false, "records": [] }
```

### Use case

> Someone rings the national desk saying "I asked for a code twice and nothing
> came". Support looks them up and sees two records, the first `SUPERSEDED` and
> the second `PENDING`, sent by email to an address the member no longer uses.
> Support now knows the real problem is the stale email address, not delivery.

---

## Read the code out to a caller

The most sensitive endpoint in this document. Hands a live code to an operator so they can read it to the person on the phone.
Two steps, deliberately.

### Step one, prove it is really you

```
POST /v1/password-resets/step-up
```

```json
{ "password": "<the operator's own password>" }
```

```json
{
  "stepUpToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "expiresIn": 300
}
```

Valid for **five minutes**, usable **once**, and tied to the session that asked
for it. It cannot be passed to a colleague, and it cannot be reused after a
successful disclosure.

Note this is the **operator's** own password, not the caller's.

### Step two, disclose

```
POST /v1/password-resets/{reference}/disclose
```

Header `X-Step-Up-Token: <the token>`.

```json
{
  "reason": "Caller verified by date of birth and parish; email undeliverable"
}
```

### Response — 200

```json
{
  "reference": "PR-4K7MQ-2XB9T",
  "otp": "418205",
  "otpExpiresAt": "2026-09-17T09:29:02.000Z",
  "minutesRemaining": 11,
  "disclosureCount": 1,
  "disclosuresRemaining": 1
}
```

### Refusals

| Status | Code | Meaning |
|---|---|---|
| 401 | `STEP_UP_REQUIRED` | No token, expired, already used, or another session's |
| 403 | `DISCLOSURE_LIMIT` | Twice for this request, or ten this hour for this operator |
| 409 | `DISCLOSURE_UNAVAILABLE` | Encryption key missing; the code cannot be recovered |
| 410 | `OTP_EXPIRED` | The code has expired. Ask the caller to request a new one |

Every call is audited whether it succeeds or fails, with the operator, the
reference, the reason and the time.

### Use case

> A pastor in a parish with no reliable email or network rings the national
> desk. Support confirms who they are, looks up the request, re-enters their own
> password, discloses the code and reads the six digits aloud. The pastor
> completes the reset themselves and chooses their own password. Support never
> learns it.

---

## Rules that decide every reset

Applied in this order. The first one that fails is the answer.

| # | Check | Code | Status |
|---|---|---|---|
| 1 | The target exists | `TARGET_NOT_FOUND` | 404 |
| 2 | You are not the target | `SELF_RESET_NOT_PERMITTED` | 403 |
| 3 | Every role they hold is a known role | `TARGET_ROLE_UNRESOLVED` | 409 |
| 4 | They are not a super admin or national support | `TARGET_UNBOUNDED` | 403 |
| 5 | **Super admin stops here — allowed** | — | 200 |
| 6 | None of their roles is sensitive | `TARGET_ROLE_SENSITIVE` | 403 |
| 7 | **National support stops here — allowed** | — | 200 |
| 8 | You hold an administrative unit | `NO_RESET_STANDING` | 403 |
| 9 | They are inside a unit you administer | `TARGET_OUTSIDE_UNIT` | 403 |
| 10 | You outrank every role they hold | `TARGET_NOT_JUNIOR` | 403 |

**Why super admin is checked at 5 and national support at 7.** Between them sits
the sensitive check. That single position is what makes national support able to
reset a regional admin but not the National Treasurer.

### What "inside a unit you administer" means

Every user record carries all seven hierarchy codes. You contain them if **any**
level you administer matches their code at that same level.

| Level | Rank | The code compared |
|---|---|---|
| Continent | 1 | `continent` |
| Sub-continent | 2 | `subContinent` |
| Region | 3 | `region` |
| Province | 4 | `province` |
| Zone | 5 | `zone` |
| Area | 6 | `area` |
| Parish | 7 | `parish` |

A regional admin for R11 reaches everyone whose `region` is R11, however many
provinces and parishes sit under it. A parish pastor reaches everyone whose
`parish` matches theirs. **A blank code on either side never matches**, so a
member with no parish recorded cannot be reset by any parish pastor.

### What "you outrank them" means

The target's most senior role decides. A province admin is rank 4, so they can
reset ranks 5, 6 and 7, and cannot reset rank 4 or above. Equal rank is refused,
which is why no one resets a peer.

Roles with no geography — national, legal and similar — have no rank. They are
protected by the sensitive flag instead.

### Which roles are sensitive

Held in the `sensitive` column on the roles collection, so it changes without a
deploy. Currently proposed: every national role, accountants at region level and
above, and the sub-continental ICT role. Super admin and national support are
protected in code as well, so editing the database cannot expose them.

Review the live list at `GET /v1/roles/sensitivity-audit`.

---

## All error codes

| Code | Status | Endpoint | Meaning |
|---|---|---|---|
| `TARGET_NOT_FOUND` | 404 | reset, eligibility | No such user |
| `SELF_RESET_NOT_PERMITTED` | 403 | reset | Use change-password instead |
| `TARGET_ROLE_UNRESOLVED` | 409 | reset, eligibility | They hold a role not in the roles collection |
| `TARGET_UNBOUNDED` | 403 | reset | They are a super admin or national support |
| `TARGET_ROLE_SENSITIVE` | 403 | reset | Super admin only |
| `NO_RESET_STANDING` | 403 | reset | You administer no unit |
| `TARGET_OUTSIDE_UNIT` | 403 | reset | Not in your hierarchy |
| `TARGET_NOT_JUNIOR` | 403 | reset | Same rank or more senior |
| `REASON_REQUIRED` | 400 | reset, disclose | Missing or under 5 characters |
| `WEAK_PASSWORD` | 400 | reset | Supplied password under 8 characters |
| `STEP_UP_REQUIRED` | 401 | disclose | Step-up token missing or invalid |
| `DISCLOSURE_LIMIT` | 403 | disclose | Per-request or per-operator cap reached |
| `DISCLOSURE_UNAVAILABLE` | 409 | disclose | Encryption key absent |
| `OTP_EXPIRED` | 410 | disclose | Code has expired |
| `TARGET_DELETED` | 400 | reset | The account is deleted; restore it first |
| `INVALID_REQUEST` | 400 | reset | The body is malformed in some other way |
| `IDENTIFIER_REQUIRED` | 400 | lookup | No phone, email or reference given |
| `REFERENCE_NOT_FOUND` | 404 | disclose | No request with that reference |
| `PASSWORD_REQUIRED` | 400 | step-up | No password sent |

---

## Frontend guidance

**Ask before you show.** Call eligibility and use it to show or hide the reset
button. Never show the button and let the server refuse; a refusal after the
click looks like a fault.

**Show the temporary password once, and say so.** Put it on screen with a copy
button and a line reading "This will not be shown again." There is no way to
retrieve it.

**Make `reason` a real field.** It appears in the audit log and is what makes
the reset defensible later. A free-text box with a five-character minimum.

**Treat these as the same failure.** Wrong code, expired code and already-used
code all return `Invalid or expired OTP`. Show that message and offer to send a
new code. Do not guess which it was.

**Handle the delivery failure separately.** `We could not send your code right
now` means nothing was sent. Do not say "check your inbox".

**Send the OTP as a string.** `"048120"` is a valid code. Sending it as a number
drops the leading zero and the reset fails.

**A reset signs the person out everywhere. A change does not sign the caller
out.** After a reset, send them to the sign-in screen. After a change, keep them
where they are.

**Nothing here changes how the existing endpoints behaved.** Request, reset and
change-password keep their shapes; request-password-reset only gains a
`reference` field alongside what it already returned.

---

## What this does not do

Stated plainly, so nobody plans around a protection that is not there.

**There is no rate limiting.** The reset endpoints are protected by the
per-identifier lockout — five wrong codes, thirty minutes — and by the
database-counted disclosure caps. Nothing limits how often a caller may ask for
a code in the first place.

**The check-username, check-email and check-phone endpoints still confirm
whether an account exists.** Making the reset messages vague would not hide
anything while those remain open.

**An area or zone administrator cannot reset anyone.** If that turns out to be
wrong for the way the organisation works, it is one entry in
`PASSWORD_RESET_ROLES`, not a code change.

**Disclosure is reversible encryption, by decision.** A compromised support
account that also passes the step-up can read a live code. The compensating
controls are the session binding, the required reason, the caps counted in the
database and the audit row on every attempt.
