# PMS member onboarding — `/v1/pms/members`

The external church PMS's door into this system. Living reference; update it
when the endpoints change.

**Status:** the availability check is live. `POST /v1/pms/members` (the create
itself) is **not built yet** — see [What is not built](#what-is-not-built).

---

## Why this mount exists at all

The PMS could have been pointed at `POST /v1/users`. It was not, deliberately.

That endpoint accepts around fifty fields and passes the entire request body to
`UsersModel.create`. Whatever the PMS sent, it could write — `roles`,
`userStatus`, `verified`, the hierarchy columns, anything. It is also mounted a
second time, **unauthenticated**, at `/v1/usersTemp`.

So the PMS gets its own mount, with a short allowlist, a role pinned in code,
and a key that can reach nothing else. `/v1/users` and `/v1/usersTemp` are
untouched by this work and behave exactly as they did.

---

## Authentication

One header. No bearer token path — the caller is a machine.

```http
x-api-key: rccg_…
```

The key is issued by `scripts/seed-pms-member-key.js` under the name
**"PMS Member Onboarding"**, scoped to `POST` on `/v1/pms/members` and nothing
else. It cannot read users, cannot delete, cannot touch any other module. A call
to `/v1/users` with it is a `403`.

| Failure | Status |
|---|---|
| no `x-api-key` header | `401` "API key required" |
| malformed key | `401` "Invalid API key format" |
| unknown or wrong key | `401` "Invalid API key" |
| key disabled | `403` "API key is disabled" |
| wrong method, or a path the key is not scoped to | `403` |
| source IP not in the key's allowlist, when one is set | `403` |

Rotate with `--rotate`: a new key is issued and the outgoing one keeps working
for **7 days**, so the PMS can be redeployed without a window of failure.

```bash
node scripts/seed-pms-member-key.js                      # first issue
node scripts/seed-pms-member-key.js --rotate             # reissue, 7-day grace
node scripts/seed-pms-member-key.js --ips 10.0.0.4,10.0.0.5
```

The plaintext is printed **once** and never stored — only its SHA-256 hash is
kept. If it is lost it cannot be recovered, only rotated.

> Use `--ips` once the PMS egress addresses are known. There is no rate limiting
> anywhere in this application, so the IP allowlist is the only thing bounding
> who can call this.

---

## `POST /v1/pms/members/availability`

Can this person be onboarded? Ask before you create.

### Request

At least one of the three. Send all three if you have them — one call answers
for all of them.

```http
POST /v1/pms/members/availability
x-api-key: rccg_…
Content-Type: application/json

{
  "email": "ade.okafor@example.com",
  "phone": "08031234567",
  "username": "ade.okafor"
}
```

**Any other key is a `400`.** The schema is a strict allowlist — an unrecognised
field is refused, not ignored, so a typo is visible rather than silent.

### What it actually checks, and why it is wider than it looks

Login names, email addresses and phone numbers are **one namespace** in this
system, not three. Password reset resolves a phone number with
`{ username: phone }`, so a mobile number *is* a login name; and plenty of
accounts use their email address as their username.

So each identifier is checked against every column it could collide with:

| You send | Checked against |
|---|---|
| `email` | `email`, `username` |
| `phone` | `phone`, `username` |
| `username` | `username`, `email`, `phone` |

Two further rules:

- **Case is ignored** for `email` and `username`. Stored values are in whatever
  case their writer used, so an exact match alone would miss
  `Ade.Okafor@Example.com` when you send `ade.okafor@example.com`.
- **Every spelling of a phone number is tried.** `+2348031234567`,
  `2348031234567`, `08031234567` and `8031234567` are one number. The column was
  never normalised on write, so the stored form is unpredictable and the lookup
  has to offer all of them.

### Response — available

`200`. This is the only response that means "go ahead and create".

```json
{
  "available": true,
  "conflicts": [],
  "code": "",
  "message": ""
}
```

### Response — taken

Still `200`. "Not available" is a correct answer to a valid question, not an
error. **Read `available`, not the status code.** (The create route is the one
that refuses with a `409`.)

```json
{
  "available": false,
  "conflicts": [{ "field": "email", "matchedOn": "email" }],
  "code": "MEMBER_ALREADY_EXISTS",
  "message": "An account already exists with this email address. A new account will not be created. If this person needs access to the RPMS module, ask the Pastor in charge of their parish, or an administrator, to grant them the RPMS role on their existing account."
}
```

`conflicts[].field` is the identifier **you sent**. `conflicts[].matchedOn` is
the stored column it **collided with**. They differ more often than you would
expect — this is somebody whose login name is the address you asked about:

```json
{ "field": "email", "matchedOn": "username" }
```

More than one conflict is reported when more than one identifier clashes:

```json
{
  "available": false,
  "conflicts": [
    { "field": "email", "matchedOn": "email" },
    { "field": "phone", "matchedOn": "phone" }
  ],
  "code": "MEMBER_ALREADY_EXISTS",
  "message": "An account already exists with this email address and this phone number. A new account will not be created. If this person needs access to the RPMS module, ask the Pastor in charge of their parish, or an administrator, to grant them the RPMS role on their existing account."
}
```

### What the response deliberately does not contain

No user id, no username, no name, no parish, no roles. Nothing that identifies
the account it found.

That is not an oversight. An external caller that could learn *whose* address a
value is would have an account-enumeration oracle against every live user — the
same shape of mistake as the unauthenticated password-verification route that
`restrictProvisioningRole` was written to close. The body carries a verdict and
a remedy; the remedy is the useful part.

> **The remedy, in the PMS's own words to the user:** the account already
> exists, so a new one will not be created. To get RPMS access on the existing
> account, ask the Pastor in charge of the parish, or an administrator, to grant
> the role.

The message is already written for a human to read. Show it.

### Errors

All `400`, all with `code: "VALIDATION_FAILED"`.

```json
{
  "status": 400,
  "name": "HttpError",
  "message": "Supply at least one of email, phone or username to check.",
  "code": "VALIDATION_FAILED"
}
```

```json
{
  "status": 400,
  "name": "HttpError",
  "message": "\"roles\" is not allowed",
  "code": "VALIDATION_FAILED"
}
```

```json
{
  "status": 400,
  "name": "HttpError",
  "message": "\"not-an-email\" is not a valid email address",
  "code": "VALIDATION_FAILED"
}
```

An empty request is refused rather than answered `available: true`, because a
green light is exactly what the caller would read it as.

### It only ever reads

No call to this endpoint creates, updates or deletes anything. It is safe to
call as often as you like — subject to the note about rate limiting above.

---

## What is not built

**`POST /v1/pms/members` — creating the member — does not exist yet.** It is
waiting on the agreed list of fields the PMS may send. Nothing has been guessed
at in the meantime.

What is already decided for it, and will not change:

| Decision | |
|---|---|
| Role assigned | `rpms-member`, pinned in code. `roles` is not an accepted request field, so there is no path by which the PMS can ask for a different one. |
| What is written | The **user document only**. No profile is created — use `POST /v1/userProfiles` afterwards if one is needed. |
| Password | The PMS must send one. It is strength-checked and bcrypt-hashed. It is never logged. |
| Duplicates | Identical rule and identical message to the availability check, but as a `409` with `code: "MEMBER_ALREADY_EXISTS"`. **Never** a second account, never a role added to the existing one, never a merge. |
| Hierarchy | Derived from `parishDirectory` by resolving the parish code. Whatever the request asserts about area, zone, province and above is ignored. An unknown parish code is a `400`. |
| Field handling | Strict allowlist, and the write object is rebuilt field by field — nothing is spread from the request. |

---

## Known gaps

These are real and deliberate. They are recorded here rather than hidden.

1. **No rate limiting.** There is none anywhere in this application. An external
   caller can loop either endpoint. The IP allowlist on the key is the only
   control; use it.
2. **No privileged-audit row for this caller.** The privileged audit trail keys
   off a JWT identity, and an API-key request has none, so `captureReason`
   declines to capture. Every call is still written to the **activity log**,
   with the key's name and prefix in `details` — that is the audit trail for
   this integration.
3. **`phone` has no unique index, and the unique indexes on `email` and
   `username` are case-sensitive.** Schema-declared indexes on the `users`
   collection silently fail to build, so the live ones were made out of band.
   This endpoint's own checks close the gap for accounts *it* creates; they do
   not repair the 53,000 rows already stored.
4. **Conflicts describe one account, not all of them.** If a value sits on one
   account's `email` and a different account's `username`, only the first found
   is listed. The verdict is unaffected — unavailable is unavailable — and
   `conflicts` is an explanation, not an inventory.
5. **`rpms-member` is shared with the legacy `/v1/usersTemp` provisioning
   caller.** Accounts from the two sources are told apart by `createdBy`, not by
   role.

---

## Where the code is

| Thing | File |
|---|---|
| Controllers | `src/components/Pmsmembers/index.ts` |
| Lookup and refusal logic | `src/components/Pmsmembers/service.ts` |
| Request allowlist | `src/components/Pmsmembers/validation.ts` |
| Routes and OpenAPI | `src/routes/PmsmembersRouter.ts` |
| Mount | `src/routes/index.ts` |
| Phone equality | `src/utils/phonePolicy.ts` |
| Email normalisation | `src/utils/emailPolicy.ts` |
| Key issuing | `scripts/seed-pms-member-key.js` |
| Tests | `test/pms-member-availability-db.js`, `test/phone-policy.js` |
