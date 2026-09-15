# Principal office register audit

Where the register of office holders and `users.roles` disagree, and how to make
them agree.

- [The two stores](#the-two-stores)
- [GET /v1/principal-officers/register-audit](#get-v1principal-officersregister-audit)
- [POST /v1/principal-officers/admin/strip-roles](#post-v1principal-officersadminstrip-roles)
- [Working through the audit](#working-through-the-audit)
- [What is logged](#what-is-logged)
- [Outcome codes](#outcome-codes)
- [What this deliberately does not do](#what-this-deliberately-does-not-do)

---

## The two stores

A principal office is recorded in two places that have never been tied
together:

| Store | Says | Written by |
|---|---|---|
| `users.roles` | what the account **can do** — `picp` in the roles string unlocks province-level authority | `POST /v1/users`, `PATCH /v1/users/:id`, `PATCH /v1/users/:id/roles` |
| `principalOfficeHolders` | who **officially holds** the office at a unit — one active row per office | `POST /v1/principal-officers`, `/:id/end`, `/:id/transfer` |

Nothing forces them to agree, and they drift in three ways, each invisible from
inside the other:

| | What it looks like | What it means |
|---|---|---|
| **`rolesWithoutOffice`** | the account carries `picp`; the register shows nobody — or someone else — as that province's picp | the person has the **authority without the appointment**. If someone else is registered, both cannot be true |
| **`officesWithoutRole`** | the register shows them holding `picp`; the role is no longer on the account, or the account is gone | the appointment is a record of an office **nobody can exercise** |
| **`officesAtWrongUnit`** | a primary appointment at LA47; the holder's profile now says LA99 | they **moved and the office did not follow** |

`GET /v1/principal-officers/conflicts` already reported the first of these, for
the first hundred users. The audit reports all three, for every user, from one
read of the register.

---

## GET /v1/principal-officers/register-audit

Super-admin. Report only — nothing is changed.

```bash
curl -H "Authorization: Bearer $SUPER_JWT" \
  "$API_HOST/v1/principal-officers/register-audit"

# narrow it
curl -H "Authorization: Bearer $SUPER_JWT" \
  "$API_HOST/v1/principal-officers/register-audit?roleSlug=picp&levelType=province&scopeCode=LA47"
```

| Query | Effect |
|---|---|
| `roleSlug` | one office role only |
| `levelType` | one level only |
| `scopeCode` | one unit only |
| `pageSize` | records **per list**, default 200, max 1000. Every `count` is the true total regardless |

```jsonc
{
  "totals": {
    "rolesWithoutOffice": 41,
    "officesWithoutRole": 7,
    "officesAtWrongUnit": 3,
    "usersScanned": 612,
    "appointmentsScanned": 388
  },
  "rolesWithoutOffice": {
    "count": 41, "truncated": false,
    "records": [
      {
        "userId": "65a1…", "username": "jdoe", "memberName": "John Doe",
        "accountStatus": "active",
        "roleSlug": "picp", "roleName": "PASTOR IN CHARGE OF PROVINCE",
        "levelType": "province", "scopeCode": "LA47",
        "office": "held-by-other",
        "registeredHolder": "aadeyemi",
        "registeredAppointmentId": "65a1…aaa"
      }
    ],
    "fix": "Either appoint them … or remove the role … 'held-by-other' means both cannot be true."
  },
  "officesWithoutRole": {
    "count": 7, "truncated": false,
    "records": [
      {
        "appointmentId": "65a1…bbb", "userId": "65a1…", "username": "kokafor",
        "roleSlug": "prov-admin", "levelType": "province", "scopeCode": "OG12",
        "source": "primary",
        "mismatch": "ROLE_NOT_HELD",
        "rolesHeld": ["basic-user"]
      }
    ],
    "fix": "End the appointment … or give the role back …"
  },
  "officesAtWrongUnit": {
    "count": 3, "truncated": false,
    "records": [
      {
        "appointmentId": "65a1…ccc", "username": "tbello",
        "roleSlug": "pic-parish", "levelType": "parish", "scopeCode": "211549",
        "source": "primary",
        "mismatch": "UNIT_MISMATCH",
        "usersOwnUnit": "211601"
      }
    ],
    "fix": "The holder's profile names a different unit from the appointment …"
  },
  "note": "Report only — nothing is changed. …"
}
```

### Reading `office` in rolesWithoutOffice

| Value | Meaning | Likely fix |
|---|---|---|
| `vacant` | nobody is registered for that office at that unit | **appoint them** — the role probably reflects reality and the register is behind |
| `held-by-other` | someone else is the registered holder | one of them is wrong. Usually **strip the role** from the unregistered one |
| `no-unit` | the account has no code at the role's level, so there is no office to check against | fix the profile first |

### Secondary appointments

An appointment whose `source` is `secondary` was made through a grant over
another parish, so its scope is the grant's, not the profile's — by design. It
is **never** reported as `UNIT_MISMATCH`.

---

## POST /v1/principal-officers/admin/strip-roles

Super-admin. Removes principal-office roles from accounts that hold them
without an appointment — the `rolesWithoutOffice` half of the audit.

**Dry run by default.** Only an explicit `"dryRun": false` writes.

```bash
curl -X POST -H "Authorization: Bearer $SUPER_JWT" \
  -H 'Content-Type: application/json' \
  -d '{
        "entries": [
          { "userId": "65a1f0000000000000000001", "roleSlug": "picp" },
          { "userId": "65a1f0000000000000000002", "roleSlug": "prov-admin" }
        ],
        "reason": "register audit 2026-09-15: not the registered holders"
      }' \
  "$API_HOST/v1/principal-officers/admin/strip-roles"
```

| Field | Notes |
|---|---|
| `entries[]` | **required**, 1–200. Each `{ userId, roleSlug }` — copy them straight from `rolesWithoutOffice.records` |
| `reason` | recorded in the activity log against every affected user |
| `dryRun` | **defaults to `true`**. Send `false` to apply |

```jsonc
{
  "dryRun": true,
  "requested": 2, "stripped": 0, "wouldStrip": 1, "refused": 1,
  "reason": "register audit 2026-09-15: not the registered holders",
  "actor": "superadmin",
  "records": [
    {
      "userId": "65a1…0001", "username": "jdoe", "memberName": "John Doe",
      "roleSlug": "picp",
      "outcome": "WOULD_STRIP", "reason": "",
      "rolesBefore": ["picp", "basic-user"],
      "rolesAfter":  ["basic-user"],
      "sessionsReset": 0
    },
    {
      "userId": "65a1…0002", "username": "aadeyemi",
      "roleSlug": "prov-admin",
      "outcome": "ACTIVE_APPOINTMENT",
      "reason": "aadeyemi is the REGISTERED holder of 'prov-admin' at province LA47. End the appointment first — POST /v1/principal-officers/65a1…aaa/end — then strip the role, or leave both in place.",
      "appointmentId": "65a1…aaa",
      "rolesBefore": ["prov-admin", "basic-user"],
      "rolesAfter": []
    }
  ]
}
```

Each entry is decided on its own. One refusal does not stop the others.

### The rule that matters

**A role is never stripped from the registered holder.** The appointment is the
authoritative record and it has its own ending path
(`POST /v1/principal-officers/:id/end`). Stripping the role from underneath it
would leave the register saying someone holds an office their account can no
longer exercise — the `officesWithoutRole` fault, manufactured by the tool meant
to fix its mirror. The refusal names the appointment to end if that is what you
want.

### What a strip does

1. Rewrites `users.roles` without the slug — the same column
   `PATCH /v1/users/:id/roles` writes.
2. **Drops any session in which the person is currently acting as that role.**
   Without this they would keep the authority until their token expired, because
   the active-role row still names the role and the token trusts the row.
   Secondary sessions are left alone; a grant over another parish is a separate
   entitlement with its own ending path.
3. Logs the change against the affected user.

---

## Working through the audit

```bash
# 1. what disagrees?
GET /v1/principal-officers/register-audit

# 2. rolesWithoutOffice, office = "vacant"
#    the register is behind — appoint them
POST /v1/principal-officers   { "userId": "…", "roleSlug": "picp" }

# 3. rolesWithoutOffice, office = "held-by-other"
#    the account is wrong — dry-run the strip, read it, apply it
POST /v1/principal-officers/admin/strip-roles   { "entries": [ … ] }
POST /v1/principal-officers/admin/strip-roles   { "entries": [ … ], "dryRun": false }

# 4. officesWithoutRole
#    the register is stale — end each appointment
POST /v1/principal-officers/:appointmentId/end   { "endedReason": "ROLE_REMOVED" }

# 5. officesAtWrongUnit
#    end, then re-appoint at the current unit — or fix the profile if the move was the mistake
POST /v1/principal-officers/:appointmentId/end
POST /v1/principal-officers   { "userId": "…", "roleSlug": "pic-parish" }

# 6. confirm
GET /v1/principal-officers/register-audit      # all three totals → 0
```

The audit is cheap enough to run after every step. Run it after the strip
especially: a strip that was refused as `ACTIVE_APPOINTMENT` stays in
`rolesWithoutOffice` until you decide which record was right.

---

## What is logged

One entry **per stripped role, against the affected user**, so the change is
attributable per person rather than per batch:

```
PRINCIPAL_OFFICE_ROLE_STRIPPED  module PRINCIPAL_OFFICERS  status SUCCESS
affectedId 65a1…0001  affectedUsername jdoe
Actor superadmin stripped 'picp' from jdoe via /v1/principal-officers/admin/strip-roles
— no active appointment backed it. Roles before: [picp, basic-user], after: [basic-user];
1 active-role session(s) reset. Reason: register audit 2026-09-15: not the registered holders
```

And one summary per batch, whose status says how it went:

| Status | When |
|---|---|
| `SUCCESS` | every entry stripped |
| `PARTIAL` | some stripped, some refused — the refused ones and their codes are listed |
| `DENIED` | every entry refused |
| `FAILED` | the request itself failed (validation, database) |

A dry run logs nothing. The audit logs nothing — it is a read.

---

## Outcome codes

Per entry in `records[].outcome`:

| Code | Written? | Meaning |
|---|---|---|
| `STRIPPED` | yes | the role was removed |
| `WOULD_STRIP` | no | dry run — would have been removed |
| `USER_NOT_FOUND` | no | no user with that id |
| `NOT_A_PRINCIPAL_OFFICE` | no | the slug is not flagged `principalOffice`. This endpoint acts on the audit; ordinary roles go through `PATCH /v1/users/:id/roles` |
| `ROLE_NOT_HELD` | no | the user does not carry that role. Nothing to strip |
| `ACTIVE_APPOINTMENT` | no | **they are the registered holder.** `appointmentId` is included; end it first if that is the intent |

Per record in the audit's `mismatch`:

| Code | List | Meaning |
|---|---|---|
| `USER_MISSING` | `officesWithoutRole` | the appointment names an account that no longer exists |
| `ROLE_NOT_HELD` | `officesWithoutRole` | the account exists, the role was removed, the appointment was not ended |
| `UNIT_MISMATCH` | `officesAtWrongUnit` | a primary appointment whose scope is not the holder's own unit any more |

---

## What this deliberately does not do

- **It does not end appointments.** `officesWithoutRole` and `officesAtWrongUnit`
  are reported with the appointment id; ending one is a decision made through
  `/:id/end`, which records why. A strip tool that also ended appointments would
  be two irreversible actions behind one button.
- **It does not appoint anyone.** `office: "vacant"` is a strong hint that the
  role is right and the register is behind, but making someone the official
  holder of an office is `POST /v1/principal-officers`, with its own checks.
- **It does not touch non-office roles.** `NOT_A_PRINCIPAL_OFFICE` is a
  deliberate refusal, not a gap.
- **It is not reachable by an API key.** Stripping authority from an account is a
  judgement and needs a person holding a bearer token.
