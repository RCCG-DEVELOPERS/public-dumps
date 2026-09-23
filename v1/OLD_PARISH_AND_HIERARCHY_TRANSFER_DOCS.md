# Parish and Hierarchy Transfer & Promotion — API Reference

Moving parishes and units between parents, and raising a unit a level.

Companion to `PRINCIPAL_OFFICERS_DOCS.md` and `APPROVALS_AND_TRANSFERS_DOCS.md`.

- [The inheritance rule](#the-inheritance-rule)
- [How the hierarchy is actually stored](#how-the-hierarchy-is-actually-stored)
- [Endpoints](#endpoints)
  - [GET /units](#get-v1hierarchy-transfersunits)
  - [GET /preview](#get-v1hierarchy-transferspreview)
  - [POST /transfer](#post-v1hierarchy-transferstransfer)
  - [POST /promote](#post-v1hierarchy-transferspromote)
  - [GET /integrity](#get-v1hierarchy-transfersintegrity)
  - [GET /jobs](#get-v1hierarchy-transfersjobs)
  - [POST /admin/backfill-codes](#post-v1hierarchy-transfersadminbackfill-codes)
- [Error codes](#error-codes)
- [Known data faults](#known-data-faults)
- [Deployment](#deployment)
- [Frontend guidance](#frontend-guidance)

---

## The inheritance rule

**A transferred entity keeps its own code and inherits every code above it from
its new parent. A transfer never mints a code.**

Driven by the `rank` on `LEVEL_MAP` — continent 1 … parish 7:

> Moving a unit at rank **R** under a new parent at rank **R−1**:
> - every field at rank **≥ R** is **kept** — its own code, and its descendants'
> - every field at rank **≤ R−1** is **inherited** from the destination

Both the `*Code` and the matching `*Name` move together. A code without its label
leaves the row displaying the wrong province.

**Parish → new area:**

| Kept | Inherited |
|---|---|
| `parishCode`, `parishName` | `areaCode`/`areaName`, `zoneCode`/`zoneName`, `provinceCode`/`provinceName`, `regionCode`/`regionName`, `subContinentCode`/`subContinentName`, `continentCode`/`continentName` |

**Area → new zone** — rewrites the area's row *and every parish in it*:

| Kept | Inherited |
|---|---|
| `areaCode`/`areaName`, and each member's `parishCode`/`parishName` | `zoneCode`/`zoneName` and everything above |

The same shape holds at every level. One implementation, parameterised by rank.

> The patch is built by a pure function that **cannot express** a field at the
> unit's own rank or below, so identity is preserved by construction. The service
> then asserts the patch and the preserved set are disjoint before writing —
> belt and braces on the rule that matters most.

---

## How the hierarchy is actually stored

Worth knowing before reading anything else: **`parishDirectory` is the entire
hierarchy.**

There is no `areaDirectory` and no `zoneDirectory`. The four sibling collections
that exist — `provinceDirectory`, `regionDirectory`, `subContinentDirectory`,
`continentDirectory` — hold 35, 12, 4 and 2 rows against 689, 116, 18 and 17 real
units. They are abandoned stubs and **this module never reads or writes them.**

A "unit" is therefore not a record. It is the set of parishes sharing a code —
every parish with `areaCode: AR8000000211` **is** area `AR8000000211` — and its
chain is whatever its members agree on.

| Level | Members share | HQ flag |
|---|---|---|
| area | `areaCode` | `ahq` |
| zone | `zoneCode` | `zhq` |
| province | `provinceCode` | `phq` |
| region | `regionCode` | `rhq` |
| sub-continent | `subContinentCode` | `schq` |
| continent | `continentCode` | `chq` |
| parish | `parishCode` | `parish` |

**A unit's chain is derived from its members, never from its HQ row.** 131 units
carry more than one HQ flag and 29 carry none, so the HQ row is neither unique nor
guaranteed to exist. If a unit's members disagree about their ancestors, every
operation on it is **refused** rather than resolved by majority — picking one
would silently rewrite the chain of everything inside it.

Departments (`parishType: "DEPARTMENT"`) are excluded everywhere. They are
`parishDirectory` rows but not places.

---

## Endpoints

Mount: `/v1/hierarchy-transfers`, bearer token only.

| Route | Auth |
|---|---|
| `GET /units`, `GET /preview` | any authenticated caller |
| **`POST /transfer`** | **an officer within their own unit, or super-admin** |
| `POST /promote`, `GET /jobs` | **super-admin** |
| `GET /integrity`, `POST /admin/backfill-codes` | **super-admin bearer OR the scoped deploy key** |

That last row matters operationally: those two run at deploy time, before anyone
has logged in to the new build, so they accept the `Organisation Init` API key as
well as a bearer token. The key admits only the exact paths its own scope names —
widening it is a deliberate act (`--update-scope`), not a prefix rule.

Reads are open because the picker is what stops anyone typing a code by hand;
hiding it only pushes people back to the unguarded `PATCH /v1/parishDirectory/:id`.

### Who may transfer what

Authority is **standing**, the same idea the principal-officer module uses: you
act at the levels your own roles sit at, and only within your own unit. Two
conditions, both required:

1. **The moved unit sits strictly below your level.** A province officer may move
   parishes, areas and zones. They may not move a province.
2. **Both ends stay inside your unit.** Moving a parish around LA47 is routine
   administration. Moving it *out* of LA47 hands a parish to another province —
   not one admin's decision — and needs a super-admin.

| Caller | Move | Result |
|---|---|---|
| `prov-admin` of LA47 | parish within LA47 | ✅ `authorisedVia: "province LA47"` |
| `prov-admin` of LA47 | area within LA47 | ✅ areas are below province |
| `prov-admin` of LA47 | parish LA47 → LA99 | ❌ `403 TRANSFER_NOT_PERMITTED` |
| `prov-admin` of LA47 | a province | ❌ at or above their own level |
| `reg-admin` of R36 | parish LA47 → LA99, both in R36 | ✅ the move never leaves R36 |
| super-admin | anything | ✅ |

The last row but one is not an accident. A caller holding several roles is
authorised by **any** standing that contains both ends — a region officer
legitimately moves parishes between provinces inside their region. A refusal is
reported against the *most specific* standing, which is the tightest boundary the
move broke.

> **The chains decide, not the request.** Authority is evaluated after the source
> and destination chains are resolved from the database, so a caller cannot assert
> a province they are not in.

### GET /v1/hierarchy-transfers/units

The picker. Each unit carries the chain a child would inherit, so the UI can show
the destination hierarchy without a second call.

```
GET /v1/hierarchy-transfers/units?level=area&search=twelve&parentCode=ZN8000027650
```

| Param | Notes |
|---|---|
| `level` | **required** — `continent` … `parish` |
| `search` | matches code or name, case-insensitive; regex-escaped |
| `parentCode` | restricts to one parent, e.g. areas within a zone |
| `pageNo`, `pageSize` | default 1 / 25, max 200 |

```json
{
  "level": "area", "pageNo": 1, "pageSize": 25,
  "records": [
    {
      "code": "AR8000000211", "name": "AREA TWELVE", "memberCount": 34,
      "chain": {
        "zoneCode": "ZN8000027650", "zoneName": "ZONE FOUR",
        "provinceCode": "LA47", "provinceName": "LAGOS 47",
        "regionCode": "R36", "regionName": "REGION 36",
        "subContinentCode": "CNT03SUBCNT01", "subContinentName": "WEST AFRICA",
        "continentCode": "CNT03", "continentName": "AFRICA"
      }
    }
  ]
}
```

### GET /v1/hierarchy-transfers/preview

What a transfer would do, without doing it. Shares its planning code with
`/transfer`, so the two can never disagree.

```
GET /v1/hierarchy-transfers/preview?level=parish&unitCode=211343&toParentCode=AR8000000211
```

```json
{
  "level": "parish",
  "unit": { "code": "211343", "name": "GRACE PARISH" },
  "toParentLevel": "area", "toParentCode": "AR8000000211",
  "kept":      { "parishCode": "211343", "parishName": "GRACE PARISH" },
  "inherited": { "areaCode": "AR8000000211", "provinceCode": "LA47", "…": "…" },
  "previous":  { "areaCode": "AR0000000009", "provinceCode": "LA90", "…": "…" },
  "willAffect": { "members": 1, "users": 412 },
  "changes": ["areaCode", "areaName", "zoneCode", "provinceCode", "…"],
  "permitted": true,
  "permittedVia": "province LA47",
  "permissionMessage": ""
}
```

`permitted` answers "would this caller be allowed to do it?" **before** they try,
so the UI can disable the confirm button and show `permissionMessage` rather than
letting a `403` surprise them at the end of the flow.

`changes` lists only the fields whose value actually differs — useful for warning
an administrator that a transfer crosses a province boundary rather than merely
moving between areas of the same one.

### POST /v1/hierarchy-transfers/transfer

```json
{ "level": "parish", "unitCode": "211343",
  "toParentCode": "AR8000000211", "reason": "Boundary review", "dryRun": false }
```

Moving an **area** is the same call: `{"level": "area", "unitCode":
"AR8000000211", "toParentCode": "ZN9000000001"}` — `membersUpdated` then reports
every parish in that area, each keeping its own `parishCode`.

Three things happen, recorded as steps on a job row:

1. **`parishDirectory`** — the unit and every parish beneath it get the inherited patch.
2. **`users`** — everyone whose scope column matches the unit gets the new chain.
   Without this a parish changes province while its members still point at the
   old one, breaking scope filters, principal-office units and the login token.
3. **`principalOfficeHolders`** — offices at units the moved people have left are
   ended with reason `TRANSFERRED`. An office belongs to the unit, not the person.

```json
{
  "success": true, "dryRun": false, "jobId": "65a1…",
  "authorisedVia": "province LA47",
  "level": "parish",
  "unit": { "code": "211343", "name": "GRACE PARISH" },
  "toParentLevel": "area", "toParentCode": "AR8000000211",
  "kept": { "parishCode": "211343", "parishName": "GRACE PARISH" },
  "inherited": { "…": "…" }, "previous": { "…": "…" },
  "cascade": {
    "membersMatched": 1, "membersUpdated": 1,
    "usersMatched": 412, "usersUpdated": 412,
    "officesVacated": [
      { "appointmentId": "65a1…", "roleSlug": "prov-admin",
        "levelType": "province", "scopeCode": "LA90", "username": "jdoe" }
    ]
  }
}
```



### POST /v1/hierarchy-transfers/promote

Raises a unit one level. **This is the only operation that mints a code.**

```json
{ "fromLevel": "parish", "unitCode": "211343",
  "newName": "AREA TWENTY", "absorbCodes": [], "reason": "New area created" }
```

| Field | Notes |
|---|---|
| `fromLevel`, `unitCode` | **required** — the target level is derived, never sent |
| `newName` | defaults to the promoted unit's own name |
| `newCode` | **required for province and above** — see below |
| `absorbCodes` | other units at `fromLevel` to bring into the new unit; max 500 |

**Codes are minted for `area` and `zone` only**, as `AR`/`ZN` + 10 random digits —
the shape 95.9% and 95.8% of production already uses. Province and above are
**not** generated: `LA47`, `R36`, `CNT03SUBCNT01` are meaningful rather than
random, so inventing a scheme from four sample formats would be a guess with no
way to undo it. Supply `newCode` for those levels, and a `CODE_REQUIRED` error
says so.

> Codes are random within their shape, not sequential — the live range runs
> `AR0000000009` … `AR8812349660`. A `max + 1` allocator would look plausible and
> be wrong.

### GET /v1/hierarchy-transfers/integrity

Read-only. **Run this before anything else** — every fault it lists makes a
transfer either refuse or act on the wrong record.

```json
{
  "duplicateParishCodes": {
    "count": 11,
    "note": "parishCode is declared unique on the model but the index has never built…",
    "records": [{ "parishCode": "211343", "rows": 2 }]
  },
  "hqConflicts": [
    { "level": "region", "hqField": "rhq", "unitsWithMultipleHq": 37 }
  ],
  "splitUnits": {
    "note": "Units whose active members disagree about their ancestors. These block a transfer.",
    "records": [{ "level": "area", "count": 46 }]
  }
}
```

### GET /v1/hierarchy-transfers/jobs

Recent operations and how far each got — `status`, and a `steps` array of
`{ name, matched, modified, done, error }`.

A **`FAILED`** job names the step it stopped on. Because every step is an
idempotent `updateMany` filtered on rows not yet carrying the target value,
re-running the same operation finishes the job rather than applying it twice.



---

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `INCONSISTENT_UNIT` | `409` | The unit's members disagree about their ancestors. `detail.variants` lists them. |
| `EMPTY_UNIT` | `400` | No active, non-department member carries that code |
| `UNKNOWN_LEVEL` | `400` | Not one of the seven levels |
| `ALREADY_THERE` | `400` | The unit is already under that parent |
| `NO_PARENT_LEVEL` | `400` | A continent cannot be transferred or promoted |
| `IDENTITY_VIOLATION` | `400` | The patch would write the moved unit's own code. Should be unreachable. |
| `CODE_REQUIRED` | `400` | Promotion above zone needs `newCode` supplied |
| `CODE_TAKEN` | `400` | The supplied code is already in use |
| `CODE_ALLOCATION_FAILED` | `400` | Could not mint a unique code |
| `TRANSFER_NOT_PERMITTED` | `403` | Outside the caller's unit, or at/above their level. The message names which end failed. |

ding which of two rows sharing `parishCode 211343` is real needs a human.



---

## Frontend guidance

1. **Never let anyone type a code.** `/units` drives every dropdown; `parentCode`
   narrows the list as the user picks down the tree.
2. **Always call `/preview` before `/transfer`** and show `previous` beside
   `inherited`. `changes` tells you whether the move crosses a province — worth a
   stronger confirmation than an intra-province one.
3. **Show `willAffect`.** "This will update 1 parish and 412 users" is the
   difference between an informed confirmation and a surprise.
4. **`dryRun: true` for anything above parish.** Promoting a zone can touch
   thousands of rows.
5. **Surface `officesVacated`.** A transfer that quietly removed someone's
   provincial office is something the administrator must see immediately.
6. **Handle `409 INCONSISTENT_UNIT` specially** — it carries `detail.variants`,
   which is a data-repair task, not a retry.
7. **A `FAILED` job is resumable.** Re-issuing the same request finishes it.
8. **Read `permitted` from `/preview`** rather than inferring authority from the
   user's roles. The rule depends on where the unit currently is and where it is
   going — the client cannot work that out without the chains.
9. **A cross-boundary move is a different conversation.** When `permitted` is
   false because the move leaves the caller's province, offer "request a
   super-admin" rather than a retry.
