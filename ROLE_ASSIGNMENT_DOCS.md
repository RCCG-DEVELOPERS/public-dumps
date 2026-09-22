# Role assignment

Who may give someone a role, what may be given, and how to roll the rule out.

- [What changed, and why](#what-changed-and-why)
- [The rule](#the-rule)
- [Switching it on](#switching-it-on)
- [Endpoints](#endpoints)
  - [GET /v1/users/me/allowed-roles](#get-v1usersmeallowed-roles)
  - [PATCH /v1/users/:id/roles](#patch-v1usersidroles)
  - [POST /v1/users and PATCH /v1/users/:id](#post-v1users-and-patch-v1usersid)
  - [GET /v1/users/admin/role-grant-readiness](#get-v1usersadminrole-grant-readiness)
- [Rejection codes](#rejection-codes)
- [Frontend guidance](#frontend-guidance)
- [Rollout](#rollout)

---

## What changed, and why

Role assignment used to answer one question: **what** may be assigned. Two rules,
both about the role and the target:

1. `super-admin` is never assignable through the API.
2. A principal office cannot be handed to someone while another person holds it.

Nothing answered **who was asking**. None of the three role-writing endpoints
carried a caller guard, so any authenticated account could grant itself `picp`,
`sco` or `cont-admin` by patching its own id. `PATCH /v1/users/:id/status` and
`/force-password-change` both required super-admin; `/roles`, the more powerful
of the three, required nothing.

This adds the missing half. The endpoints, the request bodies and the response
shape are all unchanged — a role refused for authority comes back looking
exactly like one refused for being an occupied office, with a different `code`.

---

## The rule

Everything below is judged on the caller's **active role** — the one they
switched into with `POST /v1/users/switch-role` — never on the set of roles
they hold.

| Step | Outcome |
|---|---|
| 1. caller is `super-admin` | may assign anything except `super-admin` |
| 2. caller has not switched into a role | refused — `NO_ACTIVE_ROLE` |
| 3. the active role has no geographic level | refused — `NO_GRANT_AUTHORITY` |
| 4. the role being granted has no geographic level | refused — `NO_GRANT_AUTHORITY` |
| 5. the role is **above** the caller's level | refused — `NOT_YOUR_LEVEL` |
| 6. the target is **outside** the caller's unit | refused — `NOT_YOUR_UNIT` |
| 7. the active role is unrestricted | **allowed** |
| 8. the role shares the caller's discipline | **allowed** |
| 9. otherwise | refused — `NOT_YOUR_DISCIPLINE` |

**Level** is the geographic one: continent (broadest) → sub-continent → region →
province → zone → area → parish. You may assign at your own level and below.

**Discipline** is `level_scope` — `pastor`, `admin`, `accountant`, `ict`,
`department`. Distinct from the level, and easy to confuse with it.

**Unrestricted** roles — the head and the administrator of each unit — may assign
any role at their level or below, discipline notwithstanding. By default:

```
pic-parish  pic-area  pic-zone  picp  picr  sco  co
parish-admin  area-admin  prov-admin  reg-admin  sub-cont-admin  cont-admin
```

Their **assistants are deliberately not on that list**. `apicp-admin`,
`apicp-csr`, `apicp-csr-2`, `prov-asst-admin`, `reg-asst-admin`, `apicr`,
`apicr-csr`, `asco` and `aco` fall under the discipline rule, so an assistant accountant cannot appoint the administrator
above them.

Worked through:

| Acting as | May assign | May not |
|---|---|---|
| `picp` of LA47 | anything in LA47 and below — `prov-accountant`, `parish-admin`, `pic-parish` | anything in another province; anything at region level |
| `prov-accountant` of LA47 | `prov-asst-accountant`, `parish-accountant` | `prov-admin` — different discipline |
| `prov-asst-admin` of LA47 | `prov-admin`, `parish-admin` — same discipline | `prov-accountant` |
| `parish-admin` of 211549 | any role in parish 211549 | anything at province or region level |

### Why the active role

Judging the *set* of roles held is unsound. Take the broadest rank from one role
and the union of the disciplines from all of them, and you grant authority no
single role confers:

| Caller holds | Alone permits |
|---|---|
| `reg-accountant` | accountant roles, region and below |
| `parish-admin` | any role, parish only |
| *the two, blended* | *any role, region and below* — **including `reg-admin`** |

The rank comes from one and the unrestricted flag from the other, and they meet
in the middle. With one active role in play there is nothing to blend.

### Secondary grants

Someone granted `pic-parish` over a **second** parish assigns roles **in that
parish**, not their own. Switching into a secondary role replaces the whole
hierarchy chain with the granted parish's, and the unit check reads the result —
so this needs no special case and no separate endpoint.

---

## Switching it on

`ROLE_ASSIGNMENT_ENFORCEMENT`, read on every request:

| Value | Effect |
|---|---|
| `off` — the default, and anything unrecognised | no check at all, and no extra query |
| `warn` | roles are assigned as before; every refusal the rule *would* have made is written to the activity log with status `WARNING` |
| `enforce` | refused roles are stripped, each with a code and an explanation |

A misspelling, a blank value, or an uncleaned inline `#` comment all mean `off`.
That direction is deliberate: the rule is a breaking change for every client
that assigns roles today, and a typo in the environment must not enforce it by
surprise.

`ROLE_GRANT_UNRESTRICTED` overrides the unrestricted list. Unset or empty keeps
the default; `none` clears it; inline comments are stripped. See
[CONFIGURATION.md](CONFIGURATION.md#who-may-assign-a-role).

---

## Endpoints

### `GET /v1/users/me/allowed-roles`

Every role the caller may currently assign. **Answers under every mode**,
including `off`, so a picker can be built and shipped before enforcement is
turned on.

```bash
curl -H "Authorization: Bearer $JWT" "$API_HOST/v1/users/me/allowed-roles"
```

```jsonc
{
  "mode": "warn",
  "activeRole": {
    "slug": "picp",
    "kind": "primary",
    "levelType": "province",
    "scopeCode": "LA47",
    "levelScope": "pastor",
    "unrestricted": true,
    "isSuperAdmin": false
  },
  "totalCount": 14,
  "records": [
    {
      "slug": "prov-accountant",
      "name": "PROVINCE ACCOUNTANT",
      "level_type": "province",
      "level_scope": "accountant",
      "principalOffice": true
    }
  ]
}
```

The list is judged with the caller as their own target, which is the right
question for a picker: *these are the roles within my level, my unit and my
discipline*. The per-assignment check still runs against the real target, so a
role listed here is still refused for someone in another unit.

`activeRole.slug` is empty when nothing has been switched into. Under `enforce`
that means every assignment will be refused — prompt for a role switch rather
than showing an empty picker.

### `PATCH /v1/users/:id/roles`

Unchanged in shape.

```bash
curl -X PATCH -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"roles":["prov-accountant","prov-admin"]}' \
  "$API_HOST/v1/users/68b1.../roles"
```

Partial success — `200`, acting as `prov-accountant` under `enforce`:

```jsonc
{
  "message": "Roles updated, but some were not assigned. 'prov-admin' is a admin role, and acting as 'prov-accountant' you may assign accountant roles only. The head or administrator of the province can assign it.",
  "assigned": ["prov-accountant"],
  "rejected": [
    {
      "roleSlug": "prov-admin",
      "code": "NOT_YOUR_DISCIPLINE",
      "reason": "'prov-admin' is a admin role, and acting as 'prov-accountant' you may assign accountant roles only. The head or administrator of the province can assign it.",
      "heldBy": "",
      "scopeCode": ""
    }
  ]
}
```

Nothing survived — `409`:

```jsonc
{
  "message": "None of the requested roles could be assigned. ...",
  "assigned": [],
  "rejected": [ /* … */ ]
}
```

The acceptable roles are applied and the rest stripped, rather than the whole
request being refused — an administrator fixing a five-role update should not
have to guess which entry was the problem.

### `POST /v1/users` and `PATCH /v1/users/:id`

Both also write roles, and both are vetted identically. A create reads the unit
off the request body, since there is no stored user yet.

> **`/v1/usersTemp` is exempt.** `UsersRouter` is also mounted there
> unauthenticated, for a backend provisioning caller that carries no token. When
> no caller can be identified the check is skipped entirely — that mount is
> constrained separately, pinned to `rpms-member`.

### `GET /v1/users/admin/role-grant-readiness`

What `enforce` would do, measured **before** it is set. Writes nothing.

The rule reads two fields on every role — `level_type` for the level,
`level_scope` for the discipline — and neither is validated anywhere. A role
with no geographic level cannot assign and cannot be assigned; a geographic role
with no discipline can be assigned only by the head of its unit. Both faults are
silent: the role simply stops working for everyone but a super-admin.

Reachable with the **`Organisation Init` API key**, like
`/v1/hq-assignments/integrity` — it needs no caller, so there is no "me" to
resolve and nothing about the request changes the answer.

```bash
curl -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/users/admin/role-grant-readiness"
```

```jsonc
{
  "mode": "off",
  "ready": false,
  "faults": [
    "45 of 108 roles sit at no geographic level. Under enforce their holders may assign nothing, and nobody but a super-admin may assign them. Run warn first and read the log before deciding which of these genuinely assign roles."
  ],
  "totals": {
    "roles": 108,
    "geographic": 63,
    "nonGeographic": 45,
    "geographicWithoutDiscipline": 0
  },
  "unrestricted": {
    "configured": 13,
    "unknownSlugs": [],
    "roles": [
      { "slug": "picp", "level_type": "province", "level_scope": "pastor", "geographic": true }
    ]
  },
  "disciplines": { "pastor": 21, "admin": 18, "accountant": 12 },
  "nonGeographicByLevel": { "national": 43, "house": 1, "rpms": 1 },
  "geographicWithoutDiscipline": []
}
```

| Field | Read it for |
|---|---|
| `ready` | `true` only when `faults` is empty |
| `unrestricted.unknownSlugs` | **fix these first.** A slug in `ROLE_GRANT_UNRESTRICTED` that matches no role is a typo, and it quietly demotes the head of that unit to the discipline rule |
| `nonGeographicByLevel` | which tiers are refused both ways — the national one especially |
| `geographicWithoutDiscipline` | roles only a unit head can ever grant, usually because `level_scope` was never filled in |

`ready: false` is not a blocker for shipping — the rule is `off` until you
change the mode. It is the list of things to understand before you do.

---

## Rejection codes

| Code | Meaning | What to do |
|---|---|---|
| `SUPER_ADMIN_NOT_ASSIGNABLE` | `super-admin` is never assignable through the API | assign it directly in the database, so that granting it always leaves a trail outside the application |
| `OFFICE_ALREADY_HELD` | a principal office, held by someone else at that unit | end or transfer the appointment first — the `reason` names the endpoint |
| `NO_ACTIVE_ROLE` | the caller has not switched into a role | switch role, then retry |
| `NO_GRANT_AUTHORITY` | the active role, or the role being granted, sits at no geographic level | expected for the national tier today — see [Rollout](#rollout) |
| `NOT_YOUR_LEVEL` | the role is above the caller's level | ask someone at that level |
| `NOT_YOUR_UNIT` | the target is in another unit; the `reason` names both | ask that unit's administrator |
| `NOT_YOUR_DISCIPLINE` | a role from another discipline | ask the head or administrator of the unit |

Every `reason` is a full sentence naming both the role and the constraint, and
is safe to show to the user as-is.

---

## Frontend guidance

1. Call `GET /v1/users/me/allowed-roles` when the picker opens, and **again
   after any role switch** — the answer depends on the active role.
2. Restrict the picker to `records`. This is a convenience, not the guard: the
   server checks every assignment against the real target regardless.
3. Render `rejected[]` as it arrives. It already carries a sentence per entry.
4. Treat `200` with a non-empty `rejected` as a partial success, not a failure —
   `assigned` says what was actually written.
5. While `mode` is `off` or `warn`, nothing is refused for authority. Build
   against the list now; it becomes binding when the mode changes, with no
   further frontend work.

---

## Rollout

```bash
# 1. ship with the rule inert. The diff is invisible.
ROLE_ASSIGNMENT_ENFORCEMENT=off
```

Widen the deploy key once, and read the readiness report:

```bash
node scripts/seed-org-init-key.js --update-scope
curl -H "x-api-key: $ORG_INIT_KEY" "$API_HOST/v1/users/admin/role-grant-readiness"
```

`--update-scope` does not change the secret, so a pipeline already holding the
key keeps working. Fix anything in `unrestricted.unknownSlugs` before going
further — a typo there silently demotes a unit head.

```bash
# 2. measure. Nothing is refused; every would-be refusal is logged.
ROLE_ASSIGNMENT_ENFORCEMENT=warn
```

Then read the warnings before going further:

```
activity log → module USERS, status WARNING
"… assigned role(s) that ROLE_ASSIGNMENT_ENFORCEMENT=enforce WOULD HAVE
REFUSED: prov-admin (NOT_YOUR_DISCIPLINE) — allowed because the mode is warn."
```

**What to expect there.** Only 63 of the 108 roles carry a geographic
`level_type`; the other 45 are `national` (43), `house`, `rpms` and `department`,
and step 3 refuses every one of them for having no level to assign from. Whether
the national office genuinely assigns roles is exactly the number this rollout
exists to discover — add what it turns out to need to `ROLE_GRANT_UNRESTRICTED`
rather than guessing in advance. Guessing now would either lock out the national
office or hand it blanket authority.

```bash
# 3. once the warnings are understood and the list is right.
ROLE_ASSIGNMENT_ENFORCEMENT=enforce
```

Reversible at any point: the mode is read on every request, so a restart with
`off` restores the previous behaviour exactly.
