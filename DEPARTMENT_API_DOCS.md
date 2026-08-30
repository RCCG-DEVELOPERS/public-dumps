# Department Organogram API

Groups, units, staff and reporting lines for a department. The department's
identity stays in the parish directory; everything here describes its internal
structure.

- [Authentication](#authentication)
- [Conventions](#conventions)
- [Data model](#data-model)
- [Designations](#designations)
- [Departments](#departments)
- [Groups](#groups)
- [Units](#units)
- [Staff assignments](#staff-assignments)
- [Organogram](#organogram)

---

## Authentication

Every endpoint takes either a bearer token or an API key in the `x-api-key`
header.

```
Base URL:  https://<api-host>
```

The host is issued with your credentials. The examples below assume `$API_HOST`
and `$READ_KEY` are set in your shell.

### Read-only key

GET only, across the organisation and directory modules. It cannot create,
update or delete anything. The key itself is shared separately.

```bash
curl -H "x-api-key: $READ_KEY" "$API_HOST/v1/org/designations"

# search takes a body of criteria
curl -X POST -H "x-api-key: $READ_KEY" -H "Content-Type: application/json" \
  -d '{"orAnd":"or","params":[{"columnName":"firstName","columnValue":"john"}]}' \
  "$API_HOST/v1/users/search"
```

| Scope | Reachable |
|---|---|
| `/v1/org/*` | Yes — departments, groups, units, designations, assignments |
| `/v1/departments` | Yes — parish-directory departments |
| `/v1/parishDirectory` | Yes — including `POST /v1/parishDirectory/search` |
| `/v1/provinceDirectory` | Yes — plus region, sub-continent, continent |
| `/v1/roles` | Yes |
| `/v1/users` | **Yes** — reads only. `GET /v1/users`, `GET /v1/users/:id` and `POST /v1/users/search`. Every other method needs a bearer token |

### Writing

Creates and updates need a bearer token.

> Two POSTs are **reads** and are open to the read-only key:
> `/v1/users/search` and `/v1/parishDirectory/search`. Their filter is a body of
> criteria, too large for a query string.
>
> On `/v1/users` the read/write split is enforced on the HTTP method at the
> mount, not by the key's own scope — so a key scoped too broadly still cannot
> change a role, reset a password, set account status or impersonate.

---

## Conventions

**List response**

```json
{ "totalCount": 41, "records": [ ], "pageNo": 1, "pageSize": 20 }
```

**Mutation response**

```json
{ "success": true, "message": "Group created", "record": { } }
```

**Error**

```json
{ "status": 400, "name": "HttpError", "message": "type_code is required …" }
```

| Query | Meaning |
|---|---|
| `pageNo` | 1-based. Anything unparseable falls back to page 1 |
| `pageSize` | Default 20, **clamped to 200** |
| `active` | `true` / `false`. Assignments default to `true`; everything else returns both |

Filters are whitelisted per resource. Anything not on the list is ignored rather
than rejected, and a non-scalar value — `?code[$ne]=` — is dropped, so query
parameters cannot inject Mongo operators.

| Status | When |
|---|---|
| 200 / 201 | Read / created |
| 400 | Invalid body, unknown reference, or a rule broken |
| 403 | Method or endpoint outside the API key's scope |
| 404 | No such record |
| 409 | A single-holder position is already filled |

---

## Data model

Five new collections. The parish directory is referenced, never extended.

| Collection | Holds | References |
|---|---|---|
| `parishDirectory` | The department's identity in the mission hierarchy. **Not extended by this feature** | — |
| `orgDepartments` | The org-chart record, one per department. Description and an `active` switch for the chart itself | `parishDirectoryId`, `departmentCode` |
| `orgGroups` | A collection of units. Optional — a department may have none | `departmentId`, `departmentCode` |
| `orgUnits` | The working team. `groupId: null` means it reports to the department directly | `departmentId`/`Code`, `groupId`/`Code` or null |
| `orgDesignations` | The positions themselves, and whether each allows one holder or many | — |
| `orgStaffAssignments` | Who holds which position, with job title, effective dates and history | `userId`, department, group, unit, `designationCode` |

Every child carries both the parent's id **and** its business code, so
`?departmentCode=211934` filters without a join.

### Two rules worth knowing before you write anything

- **A unit with no group reports to the Head of Department.** Ungrouped units
  carry `groupId: null` — there is no placeholder group.
- **Reporting lines are derived, never stored.** Move a unit between groups and
  every line beneath it re-derives. Nothing to migrate.

> **Designations are not roles.** `orgStaffAssignments.designationCode` describes
> a position in a department. `users.roles` controls what the API permits.
> Holding `HEAD_OF_DEPARTMENT` grants no extra permission.
>
> Nor is it `users.designation`, which holds ministerial values — Pastor,
> Deacon, Elder — on 53,000 accounts and is untouched by this feature.

---

## Designations

The positions a person can hold. Eleven are seeded; administrators may add more.

| Code | Scope | Holders |
|---|---|---|
| `HEAD_OF_DEPARTMENT` | DEPARTMENT | **One** |
| `ASSISTANT_HEAD_OF_DEPARTMENT` | DEPARTMENT | Many |
| `DEPARTMENT_ADMINISTRATOR` | DEPARTMENT | **One** |
| `ASSISTANT_DEPARTMENT_ADMINISTRATOR` | DEPARTMENT | Many |
| `DEPARTMENT_ACCOUNTANT` | DEPARTMENT | **One** |
| `ASSISTANT_DEPARTMENT_ACCOUNTANT` | DEPARTMENT | Many |
| `GROUP_HEAD` | GROUP | **One** |
| `ASSISTANT_GROUP_HEAD` | GROUP | Many |
| `UNIT_HEAD` | UNIT | **One** |
| `ASSISTANT_UNIT_HEAD` | UNIT | Many |
| `STAFF` | UNIT | Many |

### `GET /v1/org/designations` — list

| Query | Notes |
|---|---|
| `scopeType` | `DEPARTMENT`, `GROUP` or `UNIT` |
| `code` | Exact match |
| `uniquePerScope` | `true` returns only single-holder positions |
| `active` | Boolean |

**200** — `?scopeType=UNIT`

```json
{
  "totalCount": 3,
  "records": [
    {
      "id": "6a9445ecf358be6b012a8831",
      "code": "ASSISTANT_UNIT_HEAD",
      "name": "Assistant Unit Head",
      "description": "Deputises for the Unit Head. A unit may have several.",
      "scopeType": "UNIT",
      "uniquePerScope": false,
      "system": true,
      "active": true
    }
  ],
  "pageNo": 1,
  "pageSize": 20
}
```

### `GET /v1/org/designations/:id` — read one

Returns the record directly, not wrapped. `404` if unknown.

### `POST /v1/org/designations` — create

| Field | Notes |
|---|---|
| `code` **required** | UPPER_SNAKE, unique |
| `name` **required** | 2–120 chars |
| `scopeType` **required** | DEPARTMENT · GROUP · UNIT |
| `uniquePerScope` | Default `false`. `true` means one active holder per scope |
| `description` | Up to 500 chars |

**Request**

```json
{
  "code": "SPECIAL_ADVISER",
  "name": "Special Adviser",
  "scopeType": "DEPARTMENT",
  "uniquePerScope": false,
  "description": "Advises the HoD"
}
```

**201**

```json
{
  "success": true,
  "message": "Designation created",
  "record": {
    "id": "6a9445ecf358be6b012a8842",
    "code": "SPECIAL_ADVISER",
    "name": "Special Adviser",
    "scopeType": "DEPARTMENT",
    "uniquePerScope": false,
    "system": false,
    "active": true
  }
}
```

`system` is always `false` here — only the seeder creates system designations,
and those cannot be deleted.

### `PATCH /v1/org/designations/:id` — update

Accepts `name`, `description`, `scopeType`, `uniquePerScope`, `active`. At least
one is required. `code` is not editable — assignments reference designations by
code.

> On a **system** designation, `scopeType` and `uniquePerScope` are rejected with
> `400`. Existing assignments were stamped from those values and the unique index
> depends on the stamp.

---

## Departments

The org-chart record for a department that already exists in the parish
directory.

### `GET /v1/org/departments` — list

Filter on `departmentCode`, `name`, `parishDirectoryId`, `active`.

**200** — `?departmentCode=211934`

```json
{
  "totalCount": 1,
  "records": [
    {
      "id": "6a9445ecf358be6b012a8836",
      "parishDirectoryId": "6a9445ebf358be6b012a881e",
      "departmentCode": "211934",
      "name": "ICT DEPARTMENT",
      "description": "",
      "active": true,
      "createdBy": "system"
    }
  ],
  "pageNo": 1,
  "pageSize": 20
}
```

### `GET /v1/org/departments/:id` — read one

Returns the record directly. `404` if unknown.

### `POST /v1/org/departments` — create

The department must already exist in the parish directory with
`parishType: "DEPARTMENT"` and `status: "1"`. All 41 were created by the
initialisation, so this is only for a newly added one.

**Request**

```json
{ "departmentCode": "211934", "description": "Information and Communications Technology" }
```

`name` is read from the parish directory, never from the request — the cached
label cannot disagree with the directory.

**400** — no such department

```json
{
  "status": 400,
  "name": "HttpError",
  "message": "No active department with parishCode 999999 exists in the parish directory (parishType must be DEPARTMENT and status \"1\")"
}
```

### `PATCH /v1/org/departments/:id` — update · deactivate

Accepts `name`, `description`, `active`. `departmentCode` is fixed — every child
references it.

**Request**

```json
{ "description": "Information and Communications Technology" }
```

**200**

```json
{
  "success": true,
  "message": "Organisation department updated",
  "record": {
    "id": "6a9445ecf358be6b012a8836",
    "departmentCode": "211934",
    "name": "ICT DEPARTMENT",
    "description": "Information and Communications Technology",
    "active": true
  }
}
```

Set `"active": false` to switch the org chart off. Groups and units are **not**
deleted — they stop appearing in the organogram and new assignments are refused.

---

## Groups

A collection of units. Its only structural job is to insert a level into the
reporting line.

### `POST /v1/org/groups` — create

| Field | Notes |
|---|---|
| `departmentCode` **required** | Must be an active org department |
| `name` **required** | 2–200 chars |
| `code` | Derived from the name when omitted. Unique *within* the department |
| `description` | Up to 1000 chars |

**Request**

```json
{
  "departmentCode": "211934",
  "name": "Software Engineering Group",
  "description": "Builds and runs the platform"
}
```

**201**

```json
{
  "success": true,
  "message": "Group created",
  "record": {
    "id": "6a9445ecf358be6b012a884b",
    "departmentId": "6a9445ecf358be6b012a8836",
    "departmentCode": "211934",
    "name": "Software Engineering Group",
    "code": "SOFTWARE-ENGINEERING-GROUP",
    "description": "Builds and runs the platform",
    "active": true
  }
}
```

Note the derived `code`. Two departments may each have an `ADMIN` group; one
department may not have two.

### `GET /v1/org/groups` — list

Filter on `departmentCode`, `departmentId`, `code`, `name`, `active`.

```
GET /v1/org/groups?departmentCode=211934&active=true
```

### `GET /v1/org/groups/:id` — read one

Returns the record directly. `404` if unknown.

### `PATCH /v1/org/groups/:id` — update · deactivate

Accepts `name`, `description`, `active`. `departmentCode` and `code` are fixed —
units and assignments reference them.

Deactivating a group does not detach or delete its units. They keep their
`groupId` and return if the group is reactivated.

> There is no group head field. `GROUP_HEAD` and `ASSISTANT_GROUP_HEAD` are
> [staff assignments](#staff-assignments) scoped to the group, so a change of
> head is an assignment with history rather than an edit.

---

## Units

The working team. It may sit in a group, or report to the department directly.

### `POST /v1/org/units` — create

| Field | Notes |
|---|---|
| `departmentCode` **required** | Must be an active org department |
| `name` **required** | 2–200 chars |
| `groupId` | **Omit for an ungrouped unit.** Must belong to the same department |
| `code` | Derived from the name when omitted. Unique within the department |
| `description` | Up to 1000 chars |

**Request** — grouped

```json
{ "departmentCode": "211934", "name": "Backend Unit", "groupId": "6a9445ecf358be6b012a884b" }
```

**201**

```json
{
  "success": true,
  "message": "Unit created",
  "record": {
    "id": "6a9445ecf358be6b012a8858",
    "departmentCode": "211934",
    "name": "Backend Unit",
    "code": "BACKEND-UNIT",
    "groupId": "6a9445ecf358be6b012a884b",
    "groupCode": "SOFTWARE-ENGINEERING-GROUP",
    "active": true
  }
}
```

**Request** — ungrouped

```json
{ "departmentCode": "211934", "name": "Infrastructure Unit" }
```

Returns `"groupId": null, "groupCode": null`. Its head reports straight to the
Head of Department.

**400** — group from another department

```json
{
  "status": 400,
  "name": "HttpError",
  "message": "Group 6a93… is not an active group of department 999999 — a unit cannot join a group in another department"
}
```

### `GET /v1/org/units` — list

Filter on `departmentCode`, `groupCode`, `groupId`, `code`, `name`, `active`.

```
GET /v1/org/units?departmentCode=211934&ungrouped=true
```

`ungrouped` exists because "no group" is the value `null`, which a query string
cannot express. `ungrouped=false` returns only grouped units.

### `GET /v1/org/units/:id` — read one

Returns the record plus a derived `unitHeadReportsTo`, so a caller never has to
work the rule out:

```json
{
  "id": "6a9445ecf358be6b012a885d",
  "name": "Infrastructure Unit",
  "groupId": null,
  "unitHeadReportsTo": { "designationCode": "HEAD_OF_DEPARTMENT", "scope": "DEPARTMENT" }
}
```

### `PATCH /v1/org/units/:id` — update · deactivate

Accepts `name`, `description`, `active`. Use the route below to change the group.

### `PATCH /v1/org/units/:id/group` — move · detach

Separate from the general update because it changes the reporting line of
everyone in the unit.

**Move into a group**

```json
{ "groupId": "6a9445ecf358be6b012a884b" }
```

```json
{
  "success": true,
  "message": "Unit moved into group SOFTWARE-ENGINEERING-GROUP",
  "record": { },
  "unitHeadReportsTo": { "designationCode": "GROUP_HEAD", "scope": "OWN_GROUP" }
}
```

**Detach**

```json
{ "groupId": null }
```

```json
{
  "success": true,
  "message": "Unit detached from its group; its head now reports to the Head of Department",
  "unitHeadReportsTo": { "designationCode": "HEAD_OF_DEPARTMENT", "scope": "DEPARTMENT" }
}
```

Nothing is deleted. The unit, its head and its staff are untouched — only the
derived line changes.

---

## Staff assignments

The join between a user and a position — the only place a person's org role
lives.

> **Assignments are never deleted.** Ending one sets `active: false` and stamps
> `effectiveTo`; a replacement is a new record. That is what makes "who was Head
> of ICT before?" answerable.

### `POST /v1/org/assignments` — assign

| Field | Notes |
|---|---|
| `userId` **required** | An existing, active user. No user is created here |
| `designationCode` **required** | e.g. `UNIT_HEAD` |
| `unitId` | Required for UNIT-scoped designations |
| `groupId` | Required for GROUP-scoped designations |
| `departmentCode` | Required for DEPARTMENT-scoped designations |
| `jobTitle` | Free text — `Senior Software Engineer` |
| `effectiveFrom` | Defaults to now |

Send `unitId` and the department and group are read *from that unit* — an
assignment cannot claim a department its unit does not belong to.

**Request**

```json
{
  "userId": "6a9445ebf358be6b012a881f",
  "designationCode": "HEAD_OF_DEPARTMENT",
  "departmentCode": "211934",
  "jobTitle": "Director of ICT"
}
```

**201**

```json
{
  "success": true,
  "message": "Assignment created",
  "record": {
    "id": "6a9445ecf358be6b012a8875",
    "userId": "6a9445ebf358be6b012a881f",
    "username": "j.doe@rccg.org",
    "memberName": "John Doe",
    "departmentCode": "211934",
    "groupId": null,
    "unitId": null,
    "designationCode": "HEAD_OF_DEPARTMENT",
    "designationName": "Head of Department",
    "jobTitle": "Director of ICT",
    "scopeType": "DEPARTMENT",
    "uniqueRole": true,
    "active": true,
    "effectiveFrom": "2026-08-30T15:02:04.413Z",
    "effectiveTo": null
  }
}
```

**409** — the post is filled

```json
{
  "status": 409,
  "name": "HttpError",
  "message": "Head of Department is already held by John Doe. End that assignment before appointing a replacement.",
  "code": "POSITION_ALREADY_HELD",
  "reason": "John Doe"
}
```

Enforced by a partial unique index, not only by a check — a race or a direct
database write is refused too. Assistants carry `uniqueRole: false` and are
unlimited.

**400** — inactive user

```json
{ "status": 400, "message": "User carol@x.com cannot be assigned: account is disabled (status 0)" }
```

### `GET /v1/org/assignments` — list · history

Filter on `departmentCode`, `unitId`, `unitCode`, `groupId`, `userId`,
`designationCode`, `scopeType`, `active`.

**Defaults to active assignments.** Pass `active=false` for history.

```
# who is Head of ICT now
GET /v1/org/assignments?departmentCode=211934&designationCode=HEAD_OF_DEPARTMENT

# who held it before
GET /v1/org/assignments?departmentCode=211934&designationCode=HEAD_OF_DEPARTMENT&active=false

# every department a person works in
GET /v1/org/assignments?userId=6a9445ebf358be6b012a881f
```

### `GET /v1/org/assignments/:id/reporting-line` — derived chain

**200** — a unit head in a group

```json
{
  "success": true,
  "assignmentId": "6a9445ecf358be6b012a8885",
  "holder": {
    "userId": "6a9445ebf358be6b012a8820",
    "username": "a.eze@rccg.org",
    "memberName": "Amaka Eze",
    "designationCode": "UNIT_HEAD",
    "jobTitle": "Senior Software Engineer"
  },
  "chain": [
    { "designationCode": "GROUP_HEAD", "scope": "OWN_GROUP" },
    { "designationCode": "HEAD_OF_DEPARTMENT", "scope": "DEPARTMENT" }
  ]
}
```

Detach the unit from its group and the same call returns a one-hop chain
straight to `HEAD_OF_DEPARTMENT`.

### `PATCH /v1/org/assignments/:id` — job title only

```json
{ "jobTitle": "Principal Software Engineer" }
```

Only the job title is editable in place. A change of designation, unit or
department is a **move** — use transfer, so the history stays true and the
unique-position rule is re-evaluated.

### `POST /v1/org/assignments/:id/end` — end

```json
{ "endedReason": "REMOVED" }
```

`endedReason` is `TRANSFER`, `REPLACED` or `REMOVED`. `effectiveTo` defaults to
now.

```json
{
  "success": true,
  "message": "Assignment ended. The record is kept as history.",
  "record": { "active": false, "effectiveTo": "2026-08-30T15:02:04.4Z", "endedReason": "REMOVED" }
}
```

The post is now free, so a replacement can be appointed.

### `POST /v1/org/assignments/:id/transfer` — move

Ends the current assignment and opens a new one. Anything not restated carries
over.

```json
{ "unitId": "6a9445ecf358be6b012a885d" }
```

```json
{
  "success": true,
  "message": "Transferred",
  "ended":   { "active": false, "endedReason": "TRANSFER", "effectiveTo": "…" },
  "created": { "active": true,  "unitCode": "INFRASTRUCTURE-UNIT", "effectiveFrom": "…" }
}
```

Also accepts `designationCode`, `departmentCode`, `groupId`, `jobTitle`.

> If the destination post turns out to be taken, the response is `409` and **the
> original assignment is reinstated** — the person is never left with none.

---

## Organogram

### `GET /v1/org/departments/:id/organogram` — full tree

The whole chart in one call — four queries, shaped server-side.

```json
{
  "success": true,
  "department": { "id": "…", "departmentCode": "211934", "name": "ICT DEPARTMENT", "active": true },
  "leadership": {
    "head": { "memberName": "John Doe", "jobTitle": "Director of ICT" },
    "assistantHeads": [],
    "administrator": null,
    "assistantAdministrators": [],
    "accountant": null,
    "assistantAccountants": []
  },
  "groups": [
    {
      "id": "…",
      "name": "Software Engineering Group",
      "code": "SOFTWARE-ENGINEERING-GROUP",
      "head": {},
      "assistantHeads": [],
      "units": [
        {
          "name": "Backend Unit",
          "head": {},
          "assistantHeads": [],
          "staff": [],
          "others": [],
          "reportsTo": "GROUP_HEAD"
        }
      ],
      "reportsTo": "HEAD_OF_DEPARTMENT"
    }
  ],
  "ungroupedUnits": [
    {
      "name": "Infrastructure Unit",
      "head": {},
      "staff": [],
      "reportsTo": "HEAD_OF_DEPARTMENT"
    }
  ],
  "counts": { "groups": 1, "units": 2, "ungroupedUnits": 1, "activeAssignments": 3 }
}
```

Ungrouped units are siblings of the groups, not nested inside a synthetic one.
Each unit states its own `reportsTo`, since that is what differs between them.

Only **active** rows appear — history lives on the assignment endpoints.
