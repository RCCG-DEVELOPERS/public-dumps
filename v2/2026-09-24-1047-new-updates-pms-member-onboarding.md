# 2026-09-24 10:47 — PMS member onboarding, and one namespace for logins

**Branch:** `dev` · **Verified:** 1355 passing, `npm run build` clean

Reference: [PMS_MEMBER_DOCS.md](PMS_MEMBER_DOCS.md)

---

## What changed and why

An external church PMS needs to onboard members. The only existing way in was
`POST /v1/users`, which accepts around fifty fields and passes the whole request
body to `UsersModel.create` — so whatever the PMS sent, it could write: `roles`,
`userStatus`, the hierarchy columns, anything. That route is also mounted a
second time, **unauthenticated**, at `/v1/usersTemp`.

Pointing a third party at that was not an option, so the PMS gets its own mount
with its own key, a short allowlist, and a role pinned in code.

**`POST /v1/users` and `/v1/usersTemp` are not modified by this work.** Nothing
about them behaves differently.

### Three things the codebase already had, which changed the design

The original brief described creating "User + Profile + Member + Role
atomically" and assigning "the PMS role". Reading the code first was worth it:

| Assumed | Actually |
|---|---|
| A Member model to write | **There is none.** No `Member` model, no `members` collection, no member component. A member *is* a user whose `roles` string is `["rpms-member"]`. So the four-document atomic write is one document, and no rollback machinery was needed. |
| A PMS role to create | **`rpms-member` already exists**, pinned by `provisioningGuard.ts` and held by 268 accounts that hold nothing else. Reused; no new role, no change to the `roles` collection. |
| A delete rule to build | **Already half-encoded.** `DELETABLE_GLOBAL_ROLES` and `mayDeleteProvisionedAccount` already require every slug on a target to be `rpms-member` before that account can be removed. |

---

## Behaviour changes on deploy

| Change | Who it affects |
|---|---|
| New mount `POST /v1/pms/members/availability`, API-key only | the external PMS. Nothing existing routes through it. |
| New OpenAPI security scheme `apiKeyAuth` | anyone reading `/api-docs`. It was *referenced* by routes but never declared, so it rendered as an unknown scheme. |

Nothing else changes. No flag, no migration, no backfill.

**A key must be issued before the endpoint is usable** — see Deployment steps.

---

## Endpoints

### `POST /v1/pms/members/availability`

Can this person be onboarded? **Guard:** `x-api-key` only.

The full reference, including the namespace table and every error body, is in
[PMS_MEMBER_DOCS.md](PMS_MEMBER_DOCS.md). The essentials:

```http
POST /v1/pms/members/availability
x-api-key: rccg_…
Content-Type: application/json

{ "email": "ade.okafor@example.com", "phone": "08031234567", "username": "ade.okafor" }
```

At least one of the three. Any other key is a `400` — the schema is a strict
allowlist, so a typo is visible rather than silently dropped.

**200, free:**

```json
{ "available": true, "conflicts": [], "code": "", "message": "" }
```

**200, taken** — still 200. Read `available`, not the status.

```json
{
  "available": false,
  "conflicts": [{ "field": "email", "matchedOn": "email" }],
  "code": "MEMBER_ALREADY_EXISTS",
  "message": "An account already exists with this email address. A new account will not be created. If this person needs access to the RPMS module, ask the Pastor in charge of their parish, or an administrator, to grant them the RPMS role on their existing account."
}
```

**400:**

```json
{ "status": 400, "name": "HttpError", "message": "Supply at least one of email, phone or username to check.", "code": "VALIDATION_FAILED" }
```

```json
{ "status": 400, "name": "HttpError", "message": "\"roles\" is not allowed", "code": "VALIDATION_FAILED" }
```

### The finding that made this harder than it looks

**Logins, emails and phone numbers are one namespace here, not three.** Password
reset resolves a phone number with `{ username: phone }`, so a mobile number
*is* a login name; and many accounts use their email address as their username.

A check that looked only at the obvious column would report a value free that
cannot actually be registered. So:

| Supplied | Searched |
|---|---|
| `email` | `email`, `username` |
| `phone` | `phone`, `username` |
| `username` | `username`, `email`, `phone` |

Two more rules fell out of the stored data rather than out of design:

- **Case is ignored for email and username.** `Users/service.ts` writes the raw
  request body, so the normalisation `emailRule` performs is discarded before
  the insert and stored values are in whatever case their writer used. An exact
  match alone missed `Ade@Example.com` against `ade@example.com` — which is
  exactly the duplicate this endpoint exists to prevent. The lookup runs an
  indexed exact match first and falls back to a case-insensitive scan **only
  when it is about to answer "available"**, the one answer that can create a
  duplicate.
- **Every spelling of a phone is tried.** `phone` was never normalised on write
  and has no unique index, so one number is stored several ways. A new
  `src/utils/phonePolicy.ts` decides what "the same number" means.

### What the response withholds, on purpose

No id, no username, no name, no parish, no roles. An external caller that could
learn *whose* address a value is would have an account-enumeration oracle
against every live user. The body carries a verdict and a remedy — and the
remedy is already phrased for a person to read, so show it to them.

---

## Configuration

One new variable, optional:

| Variable | Default | What it does |
|---|---|---|
| `DEFAULT_PHONE_DIALLING_CODE` | `234` | The country assumed for a phone number written nationally (a leading `0`). A number sent in international form ignores it. |

The default is correct for the overwhelming majority of these accounts, so
**this does not need setting** — but it is the reason a bare `08031234567` is
read as Nigerian, and that assumption is worth knowing about in an organisation
that spans continents.

> If it is ever set, it must be registered in `deployment-scripts/lib.sh` or it
> will not reach ECS. Unset is the correct state today.

---

## Deployment steps

1. Deploy as usual. No migration, no backfill, no index.
2. **Issue the key.** Nothing can call the endpoint until this runs:
   ```bash
   node scripts/seed-pms-member-key.js --ips <pms-egress-ip>,<pms-egress-ip-2>
   ```
   The plaintext is printed **once** and never stored. Hand it to the PMS team
   through whatever channel you use for secrets; if it is lost it can only be
   rotated, not recovered.
3. Rotation later, when needed — the outgoing key keeps working for 7 days:
   ```bash
   node scripts/seed-pms-member-key.js --rotate
   ```

---

## Known gaps

- **`POST /v1/pms/members` — the create itself — is not built.** It is waiting
  on the agreed field list. Everything else about it is decided and recorded in
  [PMS_MEMBER_DOCS.md](PMS_MEMBER_DOCS.md#what-is-not-built); nothing has been
  guessed at.
- **No rate limiting.** There is none anywhere in this application, and this is
  the first endpoint exposed to a third party that creates data. The key's IP
  allowlist is the only control — use it.
- **No privileged-audit row for an API-key caller.** That trail keys off a JWT
  identity and a key request has none. Every call is written to the **activity
  log** instead, with the key's name and prefix in `details`.
- **The underlying index weaknesses are untouched.** `phone` still has no unique
  index and the `email`/`username` uniques are still case-sensitive; schema-
  declared indexes on `users` silently fail to build, which is why the live ones
  were made out of band. This endpoint compensates in its own checks. Repairing
  53,000 existing rows is a separate job with its own duplicate-resolution
  problem.
- **`rpms-member` now means two things** — legacy provisioning accounts and PMS
  accounts. They are told apart by `createdBy`, not by role.
