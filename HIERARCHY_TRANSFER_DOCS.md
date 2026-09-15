# Hierarchy Transfer & Promotion — API Reference

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
| parish | `parishCode` | — |

> **`parish` is not an HQ flag**, despite sitting in the same group of columns.
> It marks a row as a real parish (`"1"`) rather than a department (`"0"`), so
> that filtering on `parish: "1"` excludes departments. A parish is the leaf of
> the tree and has nothing beneath it to head. See `HQ_ASSIGNMENT_DOCS.md`.

**A unit's chain is derived from its members, never from its HQ row.** 130 units
carry more than one HQ flag and 127 carry none, so the HQ row is neither unique
nor guaranteed to exist. If a unit's members disagree about their ancestors, every
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

The same standing rule now bounds **promotion** and **realignment** through
`POST /scoped/promote` and `POST /scoped/realign` — see
[HIERARCHY_SCOPED_OPERATIONS_DOCS.md](HIERARCHY_SCOPED_OPERATIONS_DOCS.md). The
unbounded `/promote` and `/admin/realign` stay super-admin.

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

**Always `dryRun` first** on anything above parish level — it returns the same
shape with `cascade.membersMatched` and `usersMatched`, and nothing is written.

### POST /v1/hierarchy-transfers/admin/move

Move a unit anywhere in the hierarchy, on super-admin authority alone.

`POST /transfer` requires standing over **both** ends of the move, which is the
right rule for an officer — giving a parish away to a province you have no
authority over should not be possible — but it means no officer can move a
province to a different region, because nobody below the continent holds standing
over two regions. This is the endpoint that can.

It is the **same code path**: the inheritance rule, the member cascade, the user
scope update, the stranded-office sweep and the job record are all reused. Only
the authority differs.

```jsonc
{
  "level": "province",
  "unitCode": "LA47",
  "toLevel": "region",
  "toCode": "R12",
  "reason": "Provincial realignment, council minute 2026/14",
  "dryRun": true
}
```

The response is identical to `/transfer`, plus `"adminMove": true` and
`"authorisedVia": "super-admin"`.

#### The destination must be the immediate parent level

| Moving | Destination must be |
|---|---|
| a parish | an **area** |
| an area | a **zone** |
| a zone | a **province** |
| a province | a **region** |
| a region | a **sub-continent** |
| a sub-continent | a **continent** |

Naming anything else returns `DESTINATION_NOT_PARENT_LEVEL`.

This is not pedantry. Zone and area codes are unique across the **whole**
hierarchy — `ZN200001` may exist under LA47 or LA54, never both — so moving an
area straight under a province would leave its `zoneCode` naming a zone in the
province it just left. **To move a parish into another province, name the area it
joins there**; the province, region, sub-continent and continent all follow from
it by inheritance.

`toLevel` is technically redundant, since it can only be the parent of `level`.
It is required anyway, so a caller who believes they are putting an area directly
under a province is told so — rather than having `"LA47"` read as a zone code and
refused as a missing unit.

#### What it does not do

**Headquarters flags are untouched.** A province that moves keeps its `phq`. If
its head should change, that is `POST /v1/hq-assignments/assign` — see
`HQ_ASSIGNMENT_DOCS.md`.

Splitting a unit is not possible here: the whole unit moves, or nothing does. A
unit whose members already disagree about their ancestors is refused with
`INCONSISTENT_UNIT`, which is also what protects the code-uniqueness rule — a
code sitting under two parents cannot be moved until that is settled.
`PARISH_HIERACHY_CONFLICT.md` §6 lists every such code.

### POST /v1/hierarchy-transfers/promote

Raises a unit one level. **This is the only operation that mints a code.**
Super-admin. An officer promoting within their own unit uses
[`/scoped/promote`](HIERARCHY_SCOPED_OPERATIONS_DOCS.md#post-v1hierarchy-transfersscopedpromote)
instead — same body, bounded by standing, no `absorbCodes`.

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

**Promotion mints for `area` and `zone` only**, as `AR`/`ZN` + 10 random digits —
the shape 95.9% and 95.8% of production already uses. Province and above are
**not** generated: `LA47`, `R36`, `CNT03SUBCNT01` are meaningful rather than
random, so inventing a scheme from four sample formats would be a guess with no
way to undo it. Supply `newCode` for those levels, and a `CODE_REQUIRED` error
says so.

> Codes are random within their shape, not sequential — the live range runs
> `AR0000000009` … `AR8812349660`. A `max + 1` allocator would look plausible and
> be wrong.

**Parish codes are minted too, but on creation rather than promotion.**
`POST /v1/parishDirectory` discards any `parishCode` in the request body and
issues its own — six bare digits with a leading digit of 2-9, the shape 53,020 of
the 53,048 live codes use. The remaining 28 follow a second scheme,
`RCCGP` + 10 digits; the majority shape was chosen because a generated code has
to look like the ones beside it. Say so if the `RCCGP` form is the intended
direction and it is a one-line change.

The space is 800,000 wide and 6.6% used, so collisions are rare and retried. The
claim is the registry insert, not a prior read, so two simultaneous creates
cannot be issued the same code.

### POST /v1/hierarchy-transfers/admin/realign

**The repair for a split unit.** Super-admin only.

#### How a unit gets split

Move a parish that holds `phq` out to another region and **only that one row
changes**. A parish is a leaf, so the cascade's radius — every row carrying the
moved unit's own code — is the parish itself. Nothing in this module reads HQ
flags, so it has no notion that the row it just moved *represented* a province.
The province was never the subject of the operation.

The result: one `provinceCode` pointing at two different `regionCode`s. Every
other parish in the province, and every user under it, still names the old
region.

**And then nothing can fix it.** `resolveUnitChain` refuses a unit whose members
disagree, `planTransfer` resolves the source through it, so every `/transfer`,
`/admin/move` and `/preview` touching that province now returns `409
INCONSISTENT_UNIT`. The split is both the damage and the lock on the door. This
endpoint is the one way through it.

#### The request

```bash
curl -X POST -H "Authorization: Bearer $SUPER_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"level":"province","unitCode":"LA47","toLevel":"region","toCode":"R12","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/admin/realign"
```

`toLevel` **must be the immediate parent** of `level`. Not tidiness: the
destination chain carries the named level and its ancestors only, so naming a
*grand*parent would leave `regionCode` absent from it and the patch would write
`regionCode: ""` to every parish and every user — worse than the split.

#### The dry run, which you should always read first

```jsonc
{
  "dryRun": true,
  "level": "province", "unitCode": "LA47",
  "destination": { "level": "region", "code": "R12", "name": "REGION 12" },
  "variants": [
    { "memberCount": 43, "matchesDestination": false,
      "chain": { "regionCode": "R07", "regionName": "REGION 7", "…": "…" },
      "samples": ["211549", "211550", "211551"] },
    { "memberCount": 1, "matchesDestination": true,
      "chain": { "regionCode": "R12", "…": "…" },
      "samples": ["211001"] }
  ],
  "inherited": { "regionCode": "R12", "regionName": "REGION 12", "…": "…" },
  "cascade": {
    "membersMatched": 44, "departmentsMatched": 3, "usersMatched": 512,
    "wouldChange": 43,
    "usersUnreachable": 6
  },
  "officesAtRisk": [ { "username": "…", "roleSlug": "reg-admin", "scopeCode": "R07" } ]
}
```

`variants` is the whole point — it names both sides of the split, how many rows
each holds, and sample parish codes from each. **Which side wins is your
decision, and it is not recoverable from the data afterwards.**

Then the same call with `"dryRun": false`.

#### Three things it does differently from a transfer

| | Why |
|---|---|
| **Does not require the source to be consistent** | That is the condition it exists to end. The destination is still held to the normal standard — realigning onto a parent that is itself split would just copy one of *its* two answers down. |
| **Includes departments** | `memberFilter` excludes `parishType: "DEPARTMENT"`, which is right for a transfer — departments are not places. For a repair it is wrong: a department row carrying the old ancestry leaves the unit still split. Counted separately as `departmentsMatched`. |
| **Never ends a principal office** | A transfer *moves* a unit and vacates offices at the units left behind. A realign corrects the record of a move that already happened; ending an appointment as a side effect of a data repair is destructive and irreversible, and a split unit has no single "previous" ancestry to have left. Offices at risk are listed and left standing — review them and end or transfer each through `/v1/principal-officers` if that is right. |

#### usersUnreachable

Users whose `parish` is in the unit but whose own `province` column is blank or
names something else. They are **counted and not written** — this endpoint
matches on the unit column alone. Fix their profile, then re-run. It is `null`
with a note when the unit has more than 5,000 member rows, since the count needs
an `$in` over every member parish code and a diagnostic must not be the most
expensive part of the request.

#### Cost

`users` declares no index on any hierarchy column and `parishDirectory` indexes
only `parishCode`, so both `updateMany` calls are collection scans. The existing
transfer cascade costs exactly the same — but run this off-peak. There are no
transactions either, so a crash between the two writes leaves parishes moved and
users behind; the `hierarchyChangeJobs` row records where it stopped, and
re-running finishes it, since `$set` to a fixed value is idempotent.

### GET /v1/hierarchy-transfers/integrity

Read-only. **Run this before anything else** — every fault it lists makes a
transfer either refuse or act on the wrong record.

`splitUnits.records` now names the offending unit **codes**, not just how many
there are, so the list can be worked through with `/admin/realign` and re-read
afterwards to confirm it reached zero:

```jsonc
"splitUnits": {
  "records": [
    { "level": "province", "count": 3, "truncated": false,
      "codes": [ { "unitCode": "LA47", "variants": 2 } ] }
  ]
}
```

Capped at 50 codes per level; `count` is always the true total. Only **active**
members are compared here, so a split living entirely among inactive rows is not
counted — the realign dry run compares every row.

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

### POST /v1/hierarchy-transfers/admin/backfill-codes

Registers every code already in use so a minted one can never collide with a
legacy one. Idempotent. **Must be run before the first promotion.**

**Now includes parish.** It used to skip that level, on the grounds that nobody
is allocated into a parish and `parishCode` had no working uniqueness anyway.
Both halves have changed, so this run registers roughly 53,000 additional codes —
expect the parish row to dominate the dry-run output. Departments are counted at
parish level (a department row still occupies a `parishCode`) and excluded at
every level above, where they are not places.

```bash
curl -X POST -H "Authorization: Bearer $SUPERADMIN_JWT" \
  "$API_HOST/v1/hierarchy-transfers/admin/backfill-codes?dryRun=true"
```

---

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `INCONSISTENT_UNIT` | `409` | The unit's members disagree about their ancestors. `detail.variants` lists them. |
| `EMPTY_UNIT` | `400` | No active, non-department member carries that code |
| `UNKNOWN_LEVEL` | `400` | Not one of the seven levels |
| `ALREADY_THERE` | `400` | The unit is already under that parent |
| `ALREADY_ALIGNED` | `400` | `admin/realign` found nothing to collapse — every variant already matches the destination. Note this needs EVERY variant to match; a mostly-agreeing unit is the normal case for a realign and is not refused. |
| `NO_PARENT_LEVEL` | `400` | A continent cannot be transferred or promoted |
| `DESTINATION_NOT_PARENT_LEVEL` | `400` | `admin/move` or `admin/realign` was given a destination that is not the unit's immediate parent level. `detail` names the level expected. |
| `IDENTITY_VIOLATION` | `400` | The patch would write the moved unit's own code. Should be unreachable. |
| `CODE_REQUIRED` | `400` | Promotion above zone needs `newCode` supplied |
| `CODE_TAKEN` | `400` | The supplied code is already in use |
| `CODE_ALLOCATION_FAILED` | `400` | Could not mint a unique code |
| `TRANSFER_NOT_PERMITTED` | `403` | Outside the caller's unit, or at/above their level. The message names which end failed. |

---

## Known data faults

Measured in production, and the reason for several design choices. Re-measured
2026-09-11. Every one of these is listed row by row, with codes and
names, in **`PARISH_HIERACHY_CONFLICT.md`**.

| | |
|---|---|
| Duplicate `parishCode` | **0** — all re-coded; the unique index can now build |
| Units with more than one HQ flag | **130** — continent 7, sub-continent 6, region 37, province 16, zone 19, area 45 |
| Units with **no** HQ flag | **127** — continent 3, sub-continent 3, region 2, province 11, zone 43, area 65 |
| Headquarters missing a flag beneath them | **629** across 15 combinations; `rhq` without `phq` alone is 131 |
| Flag values no query can match | **3** rows — `phq: "2"`, `zhq: "2"`, and a parish code written into `parish` |
| Parishes not marked `parish: "1"` | **81**; plus 41 departments wrongly marked `"1"` |
| Codes living under two different parents | **340** — area 300, zone 30, province 7, region 5, sub-continent 2 |
| Areas whose members disagree about their chain | **46** |
| Indexes on any HQ column | **0**, until the org init endpoint is run |

Consequences the API exposes rather than hides:

- **Records are addressed by `_id`.** A code resolving to more than one row is refused.
- **A split unit blocks a transfer** with `409` until someone resolves it.
- The integrity endpoint reports all of it; **nothing is auto-repaired**, because
  deciding which of two rows sharing `parishCode 211343` is real needs a human.
- **`UNCATEGORIZED` is used as a real code** at province and zone level, gathering
  unrelated rows — the zone by that name spans 149 different provinces. It is a
  holding pen, not a unit, and `admin/move` cannot sensibly re-parent it.
- **Headquarters flags are not this module's concern.** A transfer does not touch
  them and `promote` sets the new level's flag without clearing the old, which is
  one source of the 130 contested units. Reading, assigning, vacating and settling
  them lives at `/v1/hq-assignments` — see `HQ_ASSIGNMENT_DOCS.md`.

---

## Deployment

Indexes are created by the org init endpoint — production runs
`MONGO_AUTO_INDEX=false`, and the production image ships neither `scripts/` nor
`ts-node`, so migrations must be endpoints.

```bash
# 1. indexes — dry run first
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388?dryRun=true"
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/org/departments/admin/init-organisation-3772a278t7388"

# 2. WIDEN THE DEPLOY KEY - it was scoped to the init path alone.
#    The secret does not change, so a pipeline already holding it keeps working.
node scripts/seed-org-init-key.js --update-scope

# 3. register the existing codes (a super-admin bearer token also works)
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/hierarchy-transfers/admin/backfill-codes?dryRun=true"
curl -X POST -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/hierarchy-transfers/admin/backfill-codes"

# 4. read the faults before moving anything
curl -H "x-api-key: $ORG_INIT_KEY" \
  "$API_HOST/v1/hierarchy-transfers/integrity"
```

Indexes created: `hierarchyCodeRegistry` (unique), `hierarchyChangeJobs`,
`parishDirectory` × 2 (`area_members`, `zone_members`), `users` × 2
(`user_by_parish`, `user_by_area`), plus the six headquarters indexes
(`hq_continent` … `hq_area`).

`{ parishCode: 1 }` unique is attempted last. Init **counts duplicates first** and
skips the build, reporting them, rather than failing on a single `E11000`. There
are none today, so it builds.

> **`PATCH /v1/parishDirectory/:id` still bypasses the inheritance rule.** It
> accepts every hierarchy field and `$set`s the raw request body, so a caller can
> still write a `provinceCode` that contradicts the rest of the chain. Closing
> that is a breaking change and remains out of scope.
>
> Its **headquarters** columns are no longer open, though: `restrictHqFlagWrites`
> now requires super-admin for any request carrying `chq`, `schq`, `rhq`, `phq`,
> `zhq`, `ahq` or `parish`. A request touching none of them is unaffected. Watch
> the activity log for `UPDATE_PARISH` / `DENIED` after release, in case an
> API-key caller was relying on it.

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
