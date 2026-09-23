# Headquarters assignment API

`/v1/hq-assignments` — which parish is the headquarters of a unit.

See `PARISH_HIERACHY_CONFLICT.md` for the current state of the live data and an
ordered plan for repairing it.

---

## The model

### Six flags, and one that is not a flag

`parishDirectory` carries seven boolean-ish columns. **Six are headquarters
flags**, one per geographic level:

| Level | Flag |
|---|---|
| continent | `chq` |
| sub-continent | `schq` |
| region | `rhq` |
| province | `phq` |
| zone | `zhq` |
| area | `ahq` |

**`parish` is not one of them.** It records what the row *is*:

- `parish: "1"` — a real parish
- `parish: "0"` — a department or placeholder

It exists so that filtering on `parish: "1"` excludes departments. A parish is
the leaf of the tree; it has nothing beneath it to be the head of, so there is no
parish-level headship and `parish` never appears in a cascade. No endpoint here
accepts `parish` as a level or as a flag to clear.

### Values are strings

Every flag holds the **string** `"1"` or `"0"`. Every reader in the codebase
compares to `"1"`, and Mongo does not equate the number `1` with the string
`"1"`, so a numeric value is invisible — set to the eye, unset to every query.
`"2"`, a stray parish code and `undefined` are all equally false.

### Rule A — the cascade

A headquarters carries its own flag and every flag beneath it, on that one
parish:

```
area           ahq
zone           zhq ahq
province       phq zhq ahq
region         rhq phq zhq ahq
sub-continent  schq rhq phq zhq ahq
continent      chq schq rhq phq zhq ahq
```

The head of a region is, necessarily, also the head of its province, its zone and
its area.

### Rule B — one holder per (flag, unit)

Exactly one parish among all those sharing `areaCode AR0001` may carry
`ahq: "1"`. One parish in LA47 may carry `phq: "1"`.

One parish may hold **many** flags — that is Rule A. Two parishes may not hold
the **same** flag in the **same** unit — that is Rule B. Both hold at once.

### The two rules collide, and nothing resolves it automatically

Making a parish the head of a region claims four flags in four different units,
any of which may already have a holder.

**An assignment never clears another parish's flag.** It sets the cascade on its
target, reports every conflict it has created, and stops. Clearing is always a
separate call that names every row it touches.

So a unit can legitimately have two holders for a while. That is not a bug being
tolerated — it is the state in which a person decides, and 130 units are in it
today.

---

## Authorization

`attachDbUser` runs on every route; authority comes from the caller's hierarchy
standing, not from the token's role claim.

| | Required |
|---|---|
| `GET /holders` | standing **at** the unit or above it, or super-admin |
| `GET /vacancies`, `GET /conflicts`, `GET /integrity` | super-admin |
| `POST /assign`, `/vacate`, `/resolve-conflict` | standing **strictly above** the unit, or super-admin |
| `POST /admin/repair-parish-flag` | super-admin |

**Standing at a unit is not enough to change its own head.** A `prov-admin` of
LA47 may decide which parish heads a zone or an area inside LA47, but not which
parish heads LA47 — that is their own seat, and it is settled one level up. A
`reg-admin` of R36 can settle it.

The listing endpoints are super-admin because they sweep the whole collection. A
scoped officer asks about their own unit through `/holders`, which names a unit
and can therefore be checked.

---

## `GET /v1/hq-assignments/holders`

Who heads this unit — including "nobody".

```
GET /v1/hq-assignments/holders?level=province&unitCode=LA47
```

```jsonc
{
  "level": "province",
  "flag": "phq",
  "unitCode": "LA47",
  "unitName": "LAGOS PROVINCE 47",
  "memberCount": 214,
  "holders": [
    { "id": "65a1…", "parishCode": "211343", "parishName": "RCCG CHAPEL OF SALVATION",
      "areaCode": "AR8000000211", "zoneCode": "ZN8000027650",
      "provinceCode": "LA47", "regionCode": "R36",
      "flagsHeld": ["phq", "zhq", "ahq"] }
  ],
  "vacant": false,
  "conflicted": false,
  "expectedFlags": ["phq", "zhq", "ahq"],
  "authorisedVia": "region R36"
}
```

`flagsHeld` lists every flag that parish carries, so a province HQ that is also a
region HQ is visible at a glance.

**403** when you have no standing over the unit.
**409** `INCONSISTENT_UNIT` when the unit's members disagree about their
ancestors — that must be resolved before anything can be said about its head.

---

## `GET /v1/hq-assignments/vacancies`

Units at a level where no parish carries the flag. Super-admin.

```
GET /v1/hq-assignments/vacancies?level=province&pageNo=1&pageSize=100
```

```jsonc
{
  "level": "province", "flag": "phq",
  "totalCount": 11, "pageNo": 1, "pageSize": 100,
  "records": [
    { "unitCode": "LA99", "unitName": "LAGOS PROVINCE 99",
      "memberCount": 47, "holderCount": 0, "holders": [] }
  ]
}
```

---

## `GET /v1/hq-assignments/conflicts`

Units where two or more parishes carry the flag. Same shape, `holderCount > 1`
and `holders` populated.

```
GET /v1/hq-assignments/conflicts?level=region
```

---

## `GET /v1/hq-assignments/integrity`

Everything at once. Super-admin, read-only, and the source of
`PARISH_HIERACHY_CONFLICT.md`.

```jsonc
{
  "generatedAt": "2026-09-11T…",
  "levels": [ { "level": "region", "flag": "rhq", "unitCount": 124,
                "vacantCount": 2, "conflictedCount": 37,
                "vacantUnits": [...], "conflictedUnits": [...] } ],
  "cascadeGaps": [ { "level": "region", "holdsFlag": "rhq", "missingFlag": "phq",
                     "count": 131, "truncated": false, "rows": [...] } ],
  "invalidFlagValues": [ { "parishCode": "887690", "parishName": "RCCG NATIONAL",
                           "field": "phq", "value": "2", "valueType": "string" } ],
  "missingParishFlag": { "count": 81, "truncated": false, "rows": [...] },
  "codeUnderTwoParents": [ { "level": "zone", "parentLevel": "province",
                             "unitCode": "ZN0000000803",
                             "unitName": "CANAANLAND (GLORIOUS ZONE)",
                             "parents": ["LA110", "LA82"], "memberCount": 13 } ]
}
```

It is a heavy read — several collection scans, because no headquarters column is
indexed yet. Run it deliberately, not on a schedule.

---

## `POST /v1/hq-assignments/assign`

Make a parish the headquarters of a unit.

```jsonc
{
  "level": "region",
  "unitCode": "R36",
  "parishCode": "211343",
  "reason": "Regional HQ relocated per council minute 2026/14",
  "dryRun": true
}
```

```jsonc
{
  "dryRun": false,
  "jobId": "65a1…",
  "level": "region", "unitCode": "R36", "unitName": "REGION 36",
  "assigned": {
    "parishCode": "211343",
    "parishName": "RCCG CHAPEL OF SALVATION",
    "flagsSet":    { "rhq": "1", "phq": "1", "zhq": "1", "ahq": "1" },
    "flagsBefore": { "rhq": "0", "phq": "1", "zhq": null, "ahq": "0" }
  },
  "conflicts": [
    {
      "flag": "rhq", "level": "region", "unitCode": "R36",
      "otherHolders": [
        { "parishCode": "211549", "parishName": "RCCG THRONE OF GRACE",
          "flagsHeld": ["rhq"] }
      ],
      "resolveWith": {
        "endpoint": "POST /v1/hq-assignments/resolve-conflict",
        "body": { "flag": "rhq", "unitCode": "R36",
                  "keep": "211343", "clear": ["211549"] }
      }
    }
  ],
  "wasVacant": false,
  "authorisedVia": "super-admin"
}
```

**Read `conflicts` every time.** An assignment at region level can produce up to
four entries — one per cascaded level — and each is a parish that still holds a
flag it now shares. `resolveWith.body` is the exact payload that settles it.

`flagsBefore` records what the target carried, so the job row is enough to undo
the change by hand.

### Refusals

| Code | Meaning |
|---|---|
| `PARISH_NOT_FOUND` | no row with that `parishCode` |
| `AMBIGUOUS_PARISH_CODE` (409) | the code matches more than one row |
| `PARISH_IS_DEPARTMENT` | a department is not a place |
| `PARISH_INACTIVE` | `status` is not `"1"` |
| `PARISH_OUTSIDE_UNIT` | the parish does not belong to the unit it would head |
| `INCONSISTENT_UNIT` (409) | the unit's members name different ancestors |

`level` must be one of the six. `parish` is rejected.

---

## `POST /v1/hq-assignments/vacate`

Clear named flags from one named parish. Never triggered by anything else.

```jsonc
{ "parishCode": "211549", "flags": ["rhq"], "reason": "…", "dryRun": true }
```

```jsonc
{
  "dryRun": false, "jobId": "65a1…",
  "parishCode": "211549", "parishName": "RCCG THRONE OF GRACE",
  "cleared": ["rhq"],
  "alreadyClear": [],
  "warnings": ["Clearing rhq leaves region R36 with no headquarters at all."],
  "authorisedVia": ["rhq via super-admin"]
}
```

Warnings do not refuse. Deliberately leaving a unit headless is a legitimate
thing to want, and 127 units are already in that state.

`alreadyClear` lists flags the parish never held. That is not an error — it is
what makes a repeated call safe.

Authority is checked **per flag, at the level that flag heads**: clearing LA47's
`phq` is a region-level decision even though the call names a parish.

`parish` is not accepted. Clearing it would hide the row from every
`parish: "1"` query — a far larger effect than vacating a headship, and never
what the caller meant.

---

## `POST /v1/hq-assignments/resolve-conflict`

Settle one contested unit.

```jsonc
{ "flag": "rhq", "unitCode": "R36",
  "keep": "211343", "clear": ["211549"], "dryRun": true }
```

**Every current holder must appear in `keep` or `clear`.** A caller working from
a stale view is refused rather than quietly leaving a third claimant in place:

| Code | Meaning |
|---|---|
| `KEEP_NOT_A_HOLDER` | `keep` does not currently hold the flag — use `/assign` |
| `INCOMPLETE_CLEAR` | other holders exist that `clear` does not name (they are listed in `detail`) |
| `CLEAR_NOT_A_HOLDER` | `clear` names a parish that does not hold the flag |

`clear` may be `[]`, which settles a unit whose only holder is already the one
being kept. It may not be **omitted** — omitting it must not be readable as
"none".

---

## `POST /v1/hq-assignments/admin/repair-parish-flag`

Set `parish: "1"` on every non-department row and `"0"` on every department.

```
POST /v1/hq-assignments/admin/repair-parish-flag?dryRun=true
```

```jsonc
{
  "dryRun": true,
  "wouldSetParishFlag": 81,
  "wouldClearDepartmentFlag": 41,
  "invalidValues": [
    { "parishCode": "", "parishName": "", "parishType": "", "parish": "211001" }
  ]
}
```

**Defaults to a dry run.** Only the literal string `dryRun=false` applies it —
this touches every row in the collection, so the safe reading of an ambiguous
request is "tell me, do not do it".

This is the one sweep in the whole feature that is safe to run unattended: the
correct value follows from `parishType` alone, with nothing to decide.

---

## Indexes

Six, declared in `src/utils/hqRules.ts` as `HQ_INDEXES` and created by the org
init endpoint (production runs `MONGO_AUTO_INDEX=false`):

```
{ chq: 1, continentCode: 1 }      hq_continent
{ schq: 1, subContinentCode: 1 }  hq_sub_continent
{ rhq: 1, regionCode: 1 }         hq_region
{ phq: 1, provinceCode: 1 }       hq_province
{ zhq: 1, zoneCode: 1 }           hq_zone
{ ahq: 1, areaCode: 1 }           hq_area
```

The flag leads because it is the selective half — 722 rows carry `phq` out of
53,049 — so the index also answers "every province headquarters".

Init additionally builds `{ parishCode: 1 }` unique, but **counts duplicates
first** and skips the build if any exist, reporting them. One duplicate exists
today, so the index is currently skipped.

**Not created:** a partial unique index enforcing Rule B. It cannot build while
130 units have two holders, and forcing it would mean choosing a loser
automatically. Uniqueness is enforced in the service; the index becomes possible
once `/integrity` is clean.

---

## What this does not touch

**Principal officers are unrelated.** `principalOfficeHolders` keys on
`roleSlug + levelType + scopeCode` and never references a parish. Moving LA47's
`phq` changes nothing about who holds `picp`, and no code path notices. See
`PRINCIPAL_OFFICERS_DOCS.md`.

**`promote` is unchanged.** `POST /v1/hierarchy-transfers/promote` still sets the
new level's flag without clearing the old one, so a promoted unit ends up
carrying both. That now shows up in `/conflicts` as a contested unit for a person
to settle, which is the same treatment every other conflict gets.

**Transfers do not touch headquarters flags.** A province that moves to another
region keeps its `phq`. If its head should change, that is an `assign`.

---

## `PATCH /v1/parishDirectory/:id`

That route could set any headquarters flag behind authentication alone, with no
role guard, and validated them as `Joi.number()` against `String` columns — which
is how `phq: "2"` and `zhq: "2"` got into the data.

It is now guarded by `restrictHqFlagWrites`
(`src/config/middleware/hqFlagGuard.ts`): a request whose body carries any of
`chq`, `schq`, `rhq`, `phq`, `zhq`, `ahq` or `parish` must come from a
super-admin, and the refusal names the fields and points here. A request that
touches none of them is unaffected.

**Deployment note:** an API-key caller that patches headquarters flags will now
receive 401/403, because the guard resolves a user from the database. No such
caller is known, but it is worth checking the activity log for
`activity: "UPDATE_PARISH"` with `status: "DENIED"` after release.

Not covered: `POST /v1/parishDirectory`, which creates a row rather than changing
an existing unit's head.
