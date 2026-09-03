# Principal Officers — API Reference

Companion to `ROLES_API_DOCS.md` and `DEPARTMENT_API_DOCS.md`.
For rollout status and phase tracking, see `PRINCIPAL_OFFICERS.md`.

- [Concepts](#concepts)
- [Role assignment rules](#role-assignment-rules)
- [Endpoints](#endpoints)
  - [GET /v1/principal-officers/config](#get-v1principal-officersconfig)
  - [GET /v1/principal-officers](#get-v1principal-officers)
  - [GET /v1/principal-officers/office](#get-v1principal-officersoffice)
  - [GET /v1/principal-officers/vacancies](#get-v1principal-officersvacancies)
  - [GET /v1/principal-officers/conflicts](#get-v1principal-officersconflicts)
  - [POST /v1/principal-officers](#post-v1principal-officers)
  - [POST /v1/principal-officers/:id/end](#post-v1principal-officersidend)
  - [POST /v1/principal-officers/:id/transfer](#post-v1principal-officersidtransfer)
- [Changes to existing endpoints](#changes-to-existing-endpoints)
- [Error codes](#error-codes)
- [Frontend guidance](#frontend-guidance)

---

## Concepts

Some roles are **offices**, not entitlements. There is one `prov-admin` of a
province, one `picr` of a region. Holding the role in `users.roles` is not the
same as being the officer — the register (`principalOfficeHolders`) decides that.

| Term | Meaning |
|---|---|
| **Office** | A role at one organisational unit — `prov-admin` at province `LA47` |
| **Holder** | The person with the active appointment to that office |
| **Vacant** | The office exists but nobody is appointed |
| **Unit** | The code at the role's `level_type` — province `LA47`, region `R07` |

**24 roles are offices**, province level and above. Which roles are offices is
`roles.principalOffice`, a boolean — data, not a list in code.

| Level | Roles |
|---|---|
| province | `picp`, `prov-admin`, `prov-accountant`, `prov-asst-admin`, `prov-asst-accountant`, `apicp-admin`, `apicp-csr` |
| region | `picr`, `apicr`, `reg-admin`, `reg-accountant`, `reg-asst-admin`, `reg-asst-accountant`, `cgo` |
| sub-continent | `sco`, `asco`, `sub-cont-admin`, `sub-cont-accountant`, `sub-cont-ict`, `training-manager` |
| continent | `co`, `aco`, `cont-admin`, `cont-accountant` |

The parish, area and zone tier is **not** included.

**One person may hold offices at several units.** The uniqueness is per (role,
unit), not per person — so someone can be `prov-admin` of two provinces, given a
secondary grant for the second.

---

## Role assignment rules

Two rules apply wherever a role is written — `POST /v1/users`,
`PATCH /v1/users/:id`, `PATCH /v1/users/:id/roles`.

### 1. `super-admin` is never assignable through the API

Refused unconditionally, on every path. It is assigned directly in the database,
so that granting the highest privilege in the system always leaves a trail
outside the application.

### 2. An office cannot be assigned while someone else holds it

Judged at the **target user's own unit**, because a role in `users.roles` is a
primary role and primary roles act on the holder's own profile. Someone in a
different province is not competing for `LA47`.

Allowed when the office is **vacant**, or when the target user is **already the
holder**.

### Roles are stripped, not rejected wholesale

A request naming five roles where one is refused assigns the other four and
reports what was removed and why. Rejecting the whole request would mean fixing a
multi-role update one role at a time, guessing which entry was the problem.

The only `409` is when **every** requested role is refused — there is nothing
left to assign.

```jsonc
// PATCH /v1/users/65a1.../roles   { "roles": ["pic-parish","prov-admin","super-admin"] }
// 200 OK
{
  "message": "Roles updated, but some were not assigned. 'super-admin' cannot be granted through the API…",
  "assigned": ["pic-parish"],
  "rejected": [
    {
      "roleSlug": "super-admin",
      "code": "SUPER_ADMIN_NOT_ASSIGNABLE",
      "reason": "'super-admin' cannot be granted through the API. It is the highest privilege in the system and is assigned directly by a database administrator, so that granting it always leaves a trail outside the application.",
      "heldBy": "", "scopeCode": ""
    },
    {
      "roleSlug": "prov-admin",
      "code": "OFFICE_ALREADY_HELD",
      "reason": "'prov-admin' is a principal office: only one person may hold it per province. A Adeyemi currently holds it for province LA47. End or transfer that appointment first — POST /v1/principal-officers/65a1.../transfer — then assign the role.",
      "heldBy": "A Adeyemi", "scopeCode": "LA47"
    }
  ]
}
```

> **Rule 2 is inert until roles are flagged.** `roles.principalOffice` is `false`
> everywhere until `scripts/flagPrincipalOfficeRoles.js` is run, so today only
> rule 1 has any effect. Flagging is the switch.

### `/v1/usersTemp`

The unauthenticated provisioning mount may assign **only `rpms-member`**. Any
other role is refused with `403` — the whole request, not a strip, since a
provisioning caller sending anything else is misconfigured rather than partially
right.

---

## Endpoints

All routes are **super-admin only, reads included** — the register is a map of
who holds authority over which unit. Bearer token only; no API-key path, since a
key carries no user for the super-admin check.

A user sees their own standing through `GET /v1/users/me/roles`.

### GET /v1/principal-officers/config

What the system currently believes. Useful before assuming anything is on.

```
GET /v1/principal-officers/config
Authorization: Bearer <superAdminToken>
```

```json
{
  "enforcement": "off",
  "totalCount": 24,
  "records": [
    { "roleSlug": "prov-admin", "roleName": "PROVINCE ADMIN", "levelType": "province" },
    { "roleSlug": "picr", "roleName": "PIC REGION", "levelType": "region" }
  ]
}
```

`enforcement` is `off` | `warn` | `enforce` — see [Switch-role](#post-v1usersswitch-role--behaviour-change).

---

### GET /v1/principal-officers

```
GET /v1/principal-officers?roleSlug=prov-admin&scopeCode=LA47&levelType=province&userId=…&active=true&pageNo=1&pageSize=20
```

All filters optional. **`active` defaults to `true`** — a list of officers is
nearly always a list of officers in post.

```json
{
  "totalCount": 1,
  "records": [
    {
      "id": "65a1f0000000000000000aaa",
      "roleSlug": "prov-admin", "roleName": "PROVINCE ADMIN",
      "levelType": "province", "scopeCode": "LA47", "scopeName": "",
      "userId": "65a1f0000000000000000999", "username": "aadeyemi", "memberName": "A Adeyemi",
      "appointmentType": "substantive", "source": "primary", "grantId": "",
      "active": true,
      "effectiveFrom": "2026-09-03T09:14:02.113Z",
      "effectiveTo": null, "endedReason": "", "predecessorId": "",
      "note": "", "appointedBy": "adminuser",
      "createdAt": "2026-09-03T09:14:02.115Z", "updatedAt": "2026-09-03T09:14:02.115Z"
    }
  ],
  "pageNo": 1, "pageSize": 20
}
```

---

### GET /v1/principal-officers/office

Who holds one office, **including the vacant case** — which a list filter cannot
express.

```
GET /v1/principal-officers/office?roleSlug=prov-admin&scopeCode=LA47
```

Both parameters required; omitting either returns `400`.

```json
{ "roleSlug": "prov-admin", "scopeCode": "LA47", "vacant": false, "holder": { "…": "…" } }
```

---

### GET /v1/principal-officers/vacancies

Offices the register has known that currently have nobody in post.

```
GET /v1/principal-officers/vacancies?levelType=province&roleSlug=prov-admin
```

```json
{ "totalCount": 2, "records": [ { "roleSlug": "prov-admin", "scopeCode": "LA47", "levelType": "province" } ] }
```

> Derived from the register's own history, so it reports offices that have
> existed — not every theoretical role × unit combination.

---

### GET /v1/principal-officers/conflicts

The worklist: people holding an office role in `users.roles` who are **not** the
registered holder for their own unit. **Report only — changes nothing.**

```
GET /v1/principal-officers/conflicts?roleSlug=prov-admin&pageSize=100
```

```json
{
  "totalCount": 3,
  "records": [
    {
      "userId": "65a1…", "username": "jdoe", "memberName": "J Doe",
      "roleSlug": "prov-admin", "levelType": "province", "scopeCode": "LA47",
      "status": "not-holder", "registeredHolder": "aadeyemi"
    }
  ],
  "scanned": 100,
  "note": "Users scanned are capped at pageSize (default 100, max 500)…"
}
```

> This endpoint **samples**. For a complete pass across all 56,865 users use
> `node scripts/analysePrincipalOffices.js --contested`.

---

### POST /v1/principal-officers

Appoint someone to an office.

```
POST /v1/principal-officers
Authorization: Bearer <superAdminToken>
Content-Type: application/json

{ "userId": "65a1f0000000000000000999", "roleSlug": "prov-admin", "note": "Effective Q4" }
```

| Field | Required | Notes |
|---|---|---|
| `userId` | yes | Mongo object id |
| `roleSlug` | yes | must be flagged `principalOffice` |
| `scopeCode` | no | **defaults to the user's own unit at the role's level** |
| `appointmentType` | no | `substantive` (default) or `acting` |
| `note` | no | max 500 |
| `effectiveFrom` | no | defaults to now |

> **The unit is derived, not asserted.** Omit `scopeCode` and it comes from the
> user's own profile. Supplying a *different* unit requires that the user already
> holds an active **secondary grant** of that role there — otherwise an admin
> could appoint anyone to any province, which is the escalation this prevents.

Returns `201` with the appointment. Errors:

| Status | When |
|---|---|
| `400` | unknown user or role; role not flagged as an office; user deactivated; user has no code at that level; user has no standing in the requested unit |
| `409` | the office already has an active holder — `code: OFFICE_ALREADY_HELD` |
| `403` | caller is not super-admin |

---

### POST /v1/principal-officers/:id/end

Vacate an office. **Nothing is deleted** — `active: false` with `effectiveTo`
stamped, so succession stays answerable and the partial unique index frees the
office.

```
POST /v1/principal-officers/65a1f0000000000000000aaa/end
{ "endedReason": "PROMOTED" }
```

`endedReason` is free text; `PROMOTED`, `TRANSFERRED`, `DEMOTED`, `REMOVED`,
`RESIGNED` are the conventional values. `effectiveTo` defaults to now.

Returns `200` with the ended appointment. `400` if the id is unknown or the
appointment has already ended.

---

### POST /v1/principal-officers/:id/transfer

Hand the office to a successor in one operation.

```
POST /v1/principal-officers/65a1f0000000000000000aaa/transfer
{ "userId": "65a1f0000000000000000111", "endedReason": "PROMOTED", "note": "Succession" }
```

```json
{ "ended": { "…": "the outgoing appointment" }, "created": { "…": "the new one" } }
```

The new row carries `predecessorId`, so succession is traceable.

> **Ordered end-then-appoint deliberately.** The reverse would momentarily leave
> two active rows for one office and be rejected by the unique index. If the
> appointment then fails — the successor is deactivated, or has no standing —
> **the incumbent is reinstated**. That is a compensating update, not a
> transaction: a process crash between the two steps leaves the office vacant.

---

## Changes to existing endpoints

### GET /v1/users/me/roles, and the `roles` array on login

Each option gains three fields:

```json
{
  "roleSlug": "prov-admin", "roleName": "PROVINCE ADMIN",
  "kind": "primary", "grantId": "", "parishName": "",
  "scope": { "…": "…" },

  "officeStatus": "not-holder",
  "officeHeldBy": "A Adeyemi",
  "officeScopeCode": "LA47"
}
```

| `officeStatus` | Meaning |
|---|---|
| `holder` | the caller is the registered holder — switching will work |
| `not-holder` | someone else holds it; `officeHeldBy` names them |
| `vacant` | the office exists but nobody is appointed |
| `n-a` | not an office role — the case for almost every role |

Additive. Clients ignoring these keys are unaffected.

### POST /v1/users/switch-role — behaviour change

Gated by **`PRINCIPAL_OFFICE_ENFORCEMENT`**:

| Mode | Behaviour |
|---|---|
| `off` *(default)* | No check at all. The resolver is not called. |
| `warn` | Switch succeeds, but a `WARNING` activity log records what would have been refused. |
| `enforce` | `409` when the caller is not the holder. |

```json
// 409 under enforce
{
  "message": "prov-admin for province LA47 is held by A Adeyemi. That appointment must be ended or transferred before anyone else can act in it.",
  "code": "NOT_OFFICE_HOLDER",
  "roleSlug": "prov-admin",
  "scopeCode": "LA47",
  "heldBy": "A Adeyemi"
}
```

Fails closed: only the exact strings `warn` and `enforce` do anything. Unset,
misspelled, or carrying an inline `#` comment all mean `off` — dotenv 4 does not
strip inline comments.

> **Note the two different switches.** Role *assignment* (above) is protected as
> soon as roles are flagged, because a refusal there is visible to an admin and
> explained. *Switch-role* is gated by the environment variable, because a wrong
> call there logs a real user out of a session.

---

## Error codes

| Code | Status | Where |
|---|---|---|
| `SUPER_ADMIN_NOT_ASSIGNABLE` | in `rejected[]` | any role assignment |
| `OFFICE_ALREADY_HELD` | `409`, or in `rejected[]` | appointment; role assignment |
| `NOT_OFFICE_HOLDER` | `409` | `switch-role` under `enforce` |
| `OFFICE_VACANT` | `409` | `switch-role` under `enforce` |

---

## Frontend guidance

1. **Role picker** — read `officeStatus`. Disable or badge a `not-holder` option
   with "held by {officeHeldBy}" instead of letting the switch fail. Show
   `vacant` differently: it is an admin action away from working, not a refusal.
2. **Role assignment forms** — a `200` may still carry `rejected[]`. Show
   `reason` verbatim; each one says what to do next, including the transfer URL.
3. **Appointment screen** — call `/office` first to show the incumbent, and offer
   *transfer* rather than letting an appointment fail with `409`.
4. **Transfer** — capture the successor and why the incumbent left; that becomes
   `endedReason` and is what makes succession readable later.
5. **Conflicts worklist** — remember it samples. Link the full script output for
   an authoritative list.
6. **Never assume enforcement is on.** Call `/config` — the same UI runs against
   environments in different modes.
