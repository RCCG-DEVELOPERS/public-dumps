# Roles & Secondary Roles — API Reference

Named to match `DEPARTMENT_API_DOCS.md`.

Covers the active-role system: how a user picks which of their roles they are
working as, and how a role can be held at a parish that is not their own.

- [Concepts](#concepts)
- [GET /v1/users/me/roles](#get-v1usersmeroles)
- [POST /v1/users/switch-role](#post-v1usersswitch-role)
- [POST /v1/auth/login](#post-v1authlogin--additive-change)
- [POST /v1/auth/refresh-token](#post-v1authrefresh-token--behaviour-change)
- [Secondary role management](#secondary-role-management)
  - [POST /v1/secondary-roles](#post-v1secondary-roles)
  - [GET /v1/secondary-roles](#get-v1secondary-roles)
  - [GET /v1/secondary-roles/holders](#get-v1secondary-rolesholders)
  - [GET /v1/secondary-roles/:id](#get-v1secondary-rolesid)
  - [POST /v1/secondary-roles/:id/end](#post-v1secondary-rolesidend)
- [How permissions are affected](#how-permissions-are-affected)
- [Deployment notes](#deployment-notes)

---

## Concepts

A user holds several roles but works as **one at a time**. Roles come from two
places.

| | Where it lives | Geography it acts on |
|---|---|---|
| **Primary** | `users.roles`, the bracketed string `'["pic-parish","pic-province"]'` | The user's **own** profile — `users.parish`, `users.province`, … |
| **Secondary** | A grant in `secondaryRoleAssignments`, pinned to a chosen parish | **That parish's** chain, not the user's |

Secondary roles exist for people who manage a parish that is not their own.

**The seven hierarchy fields**, in order, named identically on the user document,
on a grant, and in the token:

```
continent · subContinent · region · province · zone · area · parish
```

`subContinent` has a **capital C**. The parishDirectory column is
`subContinentCode`; the `roles.level_type` value is `sub-continent`. Three
spellings of one level — the wrong one returns an empty result rather than an
error.

**The active role lives on the session**, in `activeRoleSessions`, not only in the
token. That is what lets it survive `refresh-token`. Logging out clears it.

---

## GET /v1/users/me/roles

Everything the caller may switch into, and the geography each option would apply.
This is the picker.

Auth: **bearer token**. An API key gets `401` — it carries no user.

```
GET /v1/users/me/roles
Authorization: Bearer <accessToken>
```

### 200 response

```json
{
  "activeRole": "pic-parish",
  "activeRoleKind": "secondary",
  "activeRoleGrantId": "65a1f0000000000000000099",
  "totalCount": 3,
  "records": [
    {
      "roleSlug": "pic-parish",
      "roleName": "Parish Pastor",
      "kind": "primary",
      "grantId": "",
      "parishName": "",
      "scope": {
        "continent": "CT01", "subContinent": "SC01", "region": "RG01",
        "province": "UKCOF01", "zone": "ZN01", "area": "AR01", "parish": "211549"
      }
    },
    {
      "roleSlug": "pic-province",
      "roleName": "Provincial Pastor",
      "kind": "primary",
      "grantId": "",
      "parishName": "",
      "scope": {
        "continent": "CT01", "subContinent": "SC01", "region": "RG01",
        "province": "UKCOF01", "zone": "ZN01", "area": "AR01", "parish": "211549"
      }
    },
    {
      "roleSlug": "pic-parish",
      "roleName": "Parish Pastor",
      "kind": "secondary",
      "grantId": "65a1f0000000000000000099",
      "parishName": "SECOND PARISH",
      "scope": {
        "continent": "CT99", "subContinent": "SC99", "region": "RG99",
        "province": "UKCOF99", "zone": "ZN99", "area": "AR99", "parish": "211003"
      }
    }
  ]
}
```

`activeRole` is `""` when nothing has been selected yet — the state immediately
after login.

> **The same `roleSlug` can appear twice**, once as `primary` and again as
> `secondary` on a different parish. That is the feature, not a duplicate. Use
> `kind` and `grantId` to tell them apart, and render `parishName` so the user
> can see which parish each option refers to.

---

## POST /v1/users/switch-role

Makes one of the caller's roles active for the session.

Auth: **bearer token**. The caller must hold the role — as a primary role, or as
an active grant. Verified against the database, not the token.

```
POST /v1/users/switch-role
Authorization: Bearer <accessToken>
Content-Type: application/json

{ "role": "pic-parish" }
```

| Field | Required | Notes |
|---|---|---|
| `role` | yes | the role slug |
| `grantId` | no | picks a specific grant when the same role is held at several parishes |

**Resolution order.** A slug the caller holds primarily is taken as **primary**
even if they also hold a grant for it — their own parish is the less surprising
default. Send `grantId` to choose the secondary one. With several grants of the
same slug and no `grantId`, the most recently granted wins.

### 200 response

```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "payload": {
    "id": "65a1f0000000000000000001",
    "username": "jdoe",
    "roles": "[\"pic-parish\",\"pic-province\"]",
    "role": "[\"pic-parish\"]",
    "roleName": "Parish Pastor",
    "roleSlug": "pic-parish",
    "roleCode": "",
    "activeRoleKind": "secondary",
    "activeRoleGrantId": "65a1f0000000000000000099",

    "continent": "CT99", "subContinent": "SC99", "region": "RG99",
    "province": "UKCOF99", "zone": "ZN99", "area": "AR99", "parish": "211003"
  }
}
```

Key points:

- `role` (singular) names the **active** role. `roles` (plural) is left untouched
  — it is still the full primary list, so switching never costs the caller a role.
- The seven geography fields are the **granted parish's** for a secondary role,
  and the user's own for a primary one.
- The token is signed with the configured lifetime and **keeps the caller's
  session**, so it refreshes and revokes like any other.

### Errors

| Status | When |
|---|---|
| `400` | `role` missing |
| `403` | the caller holds neither that primary role nor an active grant for it |
| `401` | account unverifiable |

---

## POST /v1/auth/login — additive change

The response gains a **`roles`** key: the same array as
`GET /v1/users/me/roles` → `records`, so the picker needs no second call.
Existing clients that ignore the key are unaffected.

```json
{
  "accessToken": "…",
  "refreshToken": "…",
  "authToken": "…",
  "expiresIn": 864000,
  "refreshExpiresIn": 2592000,
  "passwordChangeRequired": false,
  "passwordChangeReason": "",
  "user": { "…": "…" },
  "roles": [
    { "roleSlug": "pic-parish", "roleName": "Parish Pastor", "kind": "primary",
      "grantId": "", "parishName": "", "scope": { "…": "…" } },
    { "roleSlug": "pic-parish", "roleName": "Parish Pastor", "kind": "secondary",
      "grantId": "65a1…", "parishName": "SECOND PARISH", "scope": { "…": "…" } }
  ]
}
```

Login starts with **no role selected**, so `user` carries the caller's own
geography until they call `switch-role`.

---

## POST /v1/auth/refresh-token — behaviour change

The active role now **survives a refresh**. Previously the payload was rebuilt
from a fresh read of the user record, which silently dropped a user out of a
secondary role and back onto their own parish.

The response is unchanged in shape; `user` is now the payload actually signed, so
it carries the active role and its geography.

---

## Secondary role management

All routes are **super-admin only, reads included**. A grant names who has
authority over which parish — that list is of no use to an ordinary caller, who
sees their own via `/v1/users/me/roles`.

Bearer token only. There is no API-key path: a key carries no user for the
super-admin check to evaluate.

### POST /v1/secondary-roles

Grants a role to a user at a chosen parish.

```
POST /v1/secondary-roles
Authorization: Bearer <superAdminToken>
Content-Type: application/json

{
  "userId": "65a1f0000000000000000001",
  "roleSlug": "pic-parish",
  "parishCode": "211003",
  "note": "Covering while the substantive PIC is on leave"
}
```

| Field | Required | Notes |
|---|---|---|
| `userId` | yes | Mongo object id |
| `roleSlug` | yes | must exist in `roles`; lower-cased on the way in |
| `parishCode` | yes | opaque string — `"211003"`, `"UKCOF01"` |
| `note` | no | free text, max 500 |
| `effectiveFrom` | no | defaults to now |

> **The hierarchy is derived, never supplied.** You send a `parishCode`; the
> province, region, sub-continent and continent are read off that parish's
> directory row. A caller cannot grant a province by claiming one, and the body
> has no fields for it.

#### 201 response

```json
{
  "id": "65a1f0000000000000000099",
  "userId": "65a1f0000000000000000001",
  "username": "jdoe",
  "memberName": "J Doe",

  "roleSlug": "pic-parish",
  "roleName": "Parish Pastor",

  "parishCode": "211003",
  "parishRef": "…",
  "parishName": "SECOND PARISH",

  "parish": "211003", "area": "AR99", "zone": "ZN99",
  "province": "UKCOF99", "region": "RG99",
  "subContinent": "SC99", "continent": "CT99",

  "areaName": "…", "zoneName": "…", "provinceName": "…",
  "regionName": "…", "subContinentName": "…", "continentName": "…",

  "active": true,
  "effectiveFrom": "2026-09-01T10:22:31.004Z",
  "effectiveTo": null,
  "endedReason": "",
  "note": "Covering while the substantive PIC is on sabbatical",
  "createdBy": "adminuser",
  "createdAt": "2026-09-01T10:22:31.006Z",
  "updatedAt": "2026-09-01T10:22:31.006Z"
}
```

#### Errors

| Status | When |
|---|---|
| `400` | no such user, no such role slug, no such parish code, or a bad body |
| `400` | the parish is a **DEPARTMENT** record — see below |
| `409` | that user already holds that role at that parish |
| `403` | caller is not super-admin |

> **DEPARTMENT rows are refused.** Departments are stored as `parishDirectory`
> rows but are not places, and some carry the literal string `"DEPARTMENT"` in
> `regionCode`. Granting against one would pin a role to a nonsense chain.

---

### GET /v1/secondary-roles

```
GET /v1/secondary-roles?userId=…&roleSlug=pic-parish&parishCode=211003&active=true&pageNo=1&pageSize=20
Authorization: Bearer <superAdminToken>
```

All filters optional. **`active` defaults to `true`** — pass `active=false` for
ended grants, since a list of grants is nearly always a list of grants in force.

```json
{
  "totalCount": 1,
  "records": [ { "…": "a grant, shaped as above" } ],
  "pageNo": 1,
  "pageSize": 20
}
```

---

### GET /v1/secondary-roles/holders

Who holds a given role at a given parish.

```
GET /v1/secondary-roles/holders?roleSlug=pic-parish&parishCode=211003
Authorization: Bearer <superAdminToken>
```

Both parameters are **required**; omitting either returns `400`.

```json
{ "totalCount": 1, "records": [ { "…": "a grant" } ] }
```

---

### GET /v1/secondary-roles/:id

```
GET /v1/secondary-roles/65a1f0000000000000000099
Authorization: Bearer <superAdminToken>
```

Returns the grant, or `404`.

---

### POST /v1/secondary-roles/:id/end

Ends a grant. **Nothing is deleted** — the row stays with `active: false` and
`effectiveTo` stamped, so "who managed this parish last year?" stays answerable,
and the partial unique index frees the slot for a re-grant.

```
POST /v1/secondary-roles/65a1f0000000000000000099/end
Authorization: Bearer <superAdminToken>
Content-Type: application/json

{ "endedReason": "Substantive PIC returned" }
```

| Field | Required | Notes |
|---|---|---|
| `endedReason` | no | free text, max 200 |
| `effectiveTo` | no | defaults to now |

Returns `200` with the ended grant.

> **Anyone currently working as this grant is dropped back to their own profile
> immediately.** The granted chain is already inside their access token, so
> without this the revocation would not take effect until that token expired —
> up to ten days.

Errors: `400` if the id is unknown or the grant is already ended.

---

## How permissions are affected

Role guards check the **union** of:

- every **primary** role in `users.roles`, and
- the **one secondary role** the session is currently active as.

Two properties matter.

**It is strictly additive.** A caller holding `["a","b"]` passes a guard for `b`
whether or not they are switched to `a`. Honouring only the active role would
start refusing requests that work today, across every guarded endpoint.

**A secondary role counts only while active.** Holding a grant confers nothing on
its own — otherwise holding five grants would quietly confer all five at once,
which is what one-role-at-a-time exists to prevent.

### Scope is still enforced by the client

Worth stating plainly: the backend does **not** yet narrow queries by the
caller's scope. `GET /v1/users` and the directory search endpoints return
everything the filter in the request body asks for. The token's geography tells
the frontend what to ask for; the server takes it on trust.

This feature changes **what the token says**, and therefore what a correct
frontend asks for. It is not a server-side access-control boundary. Server-side
narrowing is built but inert behind `SCOPE_ENFORCEMENT` — see below.


