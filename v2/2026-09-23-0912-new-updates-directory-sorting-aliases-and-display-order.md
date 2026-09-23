# 2026-09-23 09:12 — Directory sorting, aliases and display order

**Branch:** `dev` · **Verified:** 1232 passing

Phase 1 of the Parish Directory work. Bulk alignment is Phase 2 and is not in
this change.

---

## The finding that shaped the design

**Zones and areas do not exist as records.** `src/utils/hierarchyChain.ts:5-25`
is explicit: the hierarchy is "twelve denormalised columns on parishDirectory,
and nothing else. There is no areaDirectory and no zoneDirectory." The four stub
directories that do exist hold 35 rows against 689 real provinces and must not
be read.

So a zone's alias has nowhere to live except on its members — and that is where
it now lives, exactly as the hierarchy itself does.

---

## Behaviour changes on deploy

**None.** Every new column defaults to empty or 0, sorting only happens when
asked for, and no existing endpoint behaves differently.

---

## 1. Aliases

### `PATCH /v1/parishDirectory/alias`

**Guard:** elevated.

```http
PATCH /v1/parishDirectory/alias
Content-Type: application/json

{ "level": "zone", "code": "Z001", "alias": "Lagos Mainland Zone" }
```

**200** — `{ "level": "zone", "code": "Z001", "alias": "Lagos Mainland Zone", "members": 42 }`

`members` is the number of parish rows updated. A unit is the set of parishes
sharing a code, so its display name is written across all of them at once —
there is no path that sets it on one row, because the members would then
disagree about what the unit is called.

Send `"alias": ""` to clear it.

### The display rule

An alias never **replaces** the official name. Somebody searching "Zone 1" must
still find it when everyone calls it Lagos Mainland.

| Alias | Label |
|---|---|
| set | `Lagos Mainland Zone (Zone 1)` |
| not set | `Zone 1` |
| same as the name | `Zone 1` — not repeated |

## 2. Display order

### `PATCH /v1/parishDirectory/order`

**Guard:** elevated.

```http
PATCH /v1/parishDirectory/order
Content-Type: application/json

{
  "level": "zone",
  "orders": [
    { "code": "Z001", "displayOrder": 1 },
    { "code": "Z002", "displayOrder": 2 }
  ]
}
```

**200**

```json
{
  "level": "zone",
  "applied": [
    { "code": "Z001", "displayOrder": 1, "members": 42 },
    { "code": "Z002", "displayOrder": 2, "members": 17 }
  ]
}
```

Sent as a **list** on purpose: "which zone comes first" is only meaningful
relative to its siblings, and setting one alone invites two units both claiming
position 1. Capped at 500 units per request.

> **Display order never changes a code.** Not `parishCode`, not `zoneCode`, not
> `areaCode`, and not any hierarchy relationship. It is presentation and nothing
> else — there are tests asserting exactly that for both alias and order.

## 3. Sorting

```http
POST /v1/parishDirectory/searchAndFilter?sortBy=province,zone,area,parish&sortOrder=asc
Content-Type: application/json

{ "orAnd": "and", "params": [ { "columnName": "provinceCode", "columnValue": "LA47" } ] }
```

Each level expands to **two** sort keys — its display order, then its name:

```
sortBy=province,zone
  → { provinceOrder: 1, provinceName: 1, zoneOrder: 1, zoneName: 1 }
```

The name is the tie-break. Without it the many rows that have no order yet would
shuffle between requests, which is worse than being unsorted.

The response echoes `sortBy` and `sortOrder` so a caller can see the sort was
honoured rather than assuming it.

`sortBy` goes on the **query string, not the body** — `searchParams` is
`Joi.object().keys()` with no `.unknown()`, so a body field would be rejected
outright, and `pageNo`/`pageSize`/`requestedFields` already arrive this way.

**400** for an unknown level, naming what is allowed:

```json
{ "message": "Cannot sort by banana. Sort by any of: continent, sub-continent, region, province, zone, area, parish." }
```

### A sort must be narrowed — and this is not fussiness

A four-key hierarchy sort has **no index behind it**. Across 53,000 parishes it
is a blocking in-memory SORT against a 32MB ceiling with `allowDiskUse` unset,
and a connection-wide 30s `maxTimeMS` that **kills** the query rather than
slowing it. That is the query class behind the 2026-09-15 production stall.

So a sort must either name a unit to sort within — any `provinceCode`,
`zoneCode`, `regionCode` and so on in the search params — or ask for a page of
**500 or fewer** (`PARISH_SORT_MAX_UNSCOPED_PAGE`). Otherwise:

```json
{
  "message": "A hierarchy sort has to be narrowed before it can run. Add a continentCode, subContinentCode, regionCode, provinceCode, zoneCode, areaCode to your search params, or ask for a page of 500 or fewer. Sorting the whole directory has no index behind it and would be cut off by the query timeout rather than returning slowly."
}
```

Scoped to a province that is a few hundred rows and costs nothing. The admin
alignment screen always works within a province anyway.

A new index `{provinceCode, zoneOrder, areaOrder, parishOrder}` serves the common
case directly.

---

## The trap this design creates, and the guard for it

Because presentation lives on the member rows, a unit's rows could disagree
about their own alias. Two things prevent that becoming a problem:

1. **One writer.** Both endpoints `updateMany` across the unit. Nothing sets
   these on a single row.
2. **They are not part of the chain.** `unitChainVariants` groups on ancestor
   *code and name* fields to decide whether a unit is split. The presentation
   columns are deliberately **not** in that grouping — otherwise a typo in a
   display name would report the unit as SPLIT and make it **immovable**.

There is a test named for that second point, and another asserting the columns
never appear in `preservedFieldsFor`, `buildInheritedPatch` or `buildUserPatch`,
so a transfer cannot carry one unit's display name onto another.

---

## Columns added to `parishDirectory`

`continentAlias`/`continentOrder` through `parishAlias`/`parishOrder` — fourteen
in all, strings and numbers, defaulting to empty and 0. Nothing needs
backfilling; the feature works on existing rows from the first deploy.

## Configuration

| Variable | Default | Meaning |
|---|---|---|
| `PARISH_SORT_MAX_UNSCOPED_PAGE` | `500` | Largest page sortable without naming a unit to sort within |

---

## Still to come — Phase 2

Bulk alignment (`PATCH /v1/parishDirectory/bulk-alignment`): validate a whole
payload, apply with the same cascade as an individual transfer, and compensate
from a snapshot on failure. Not in this change.
