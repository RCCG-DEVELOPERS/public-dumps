# Scoped promotion and realignment

Promote a unit, or repair a split one, **within your own unit** — without a
super-admin.

- [What changed](#what-changed)
- [Who may do what](#who-may-do-what)
- [POST /v1/hierarchy-transfers/scoped/promote](#post-v1hierarchy-transfersscopedpromote)
- [POST /v1/hierarchy-transfers/scoped/realign](#post-v1hierarchy-transfersscopedrealign)
- [What is logged](#what-is-logged)
- [Error codes](#error-codes)
- [Frontend guidance](#frontend-guidance)

---

## What changed

Until now, every structural change to the hierarchy that was not a plain move
needed a super-admin. A province admin could move a parish between areas in
their province (`/transfer`), and could choose which parish heads an area
(`/hq-assignments/assign`), but could not:

- **promote** a parish to an area, or an area to a zone — `/promote` was
  super-admin only
- **repair** a unit whose parishes disagree about their ancestry —
  `/admin/realign` was super-admin only

Both now have a **scoped** twin. Same request body, same effect, same job
record, same cascade. The only difference is authority: the scoped endpoints are
bounded by the caller's own **standing**, judged the same way `/transfer` already
is.

| Endpoint | Guard | Bound |
|---|---|---|
| `POST /promote` | super-admin | none |
| `POST /scoped/promote` | **the administrator** of a unit | strictly below your level, inside your unit; no `absorbCodes` |
| `POST /admin/realign` | super-admin | none |
| `POST /scoped/realign` | **the administrator** of a unit | standing over **both ends of every variant** |

The unbounded endpoints are unchanged. A super-admin calling a scoped endpoint
is also unchanged — their standing is unbounded by definition, so the check
narrows nobody who was already allowed.

---

## Who may do what

Two gates, in order.

### Gate 1 — you must be an administrator

Only these roles may use the scoped endpoints at all:

```
prov-admin   reg-admin   sub-cont-admin   cont-admin
```

That is deliberately **narrower than standing**. Standing says a `picp` acts at
province level; this says only the `prov-admin` *restructures* the province.
Excluded, on purpose:

- **the heads of unit** — `picp`, `picr`, `sco`, `co`, `pic-parish`, …
- **every assistant** — `prov-asst-admin`, `reg-asst-admin`, `apicp-admin`,
  `asco`, `aco`, …
- **everyone else** — accountants, ICT, training managers

They get a `403` naming the roles that would qualify. Moves are still theirs
through `/transfer`; anything structural goes to their administrator.

The list is `HIERARCHY_RESTRUCTURE_ROLES` in the environment, with the four
above as the default — same parsing rules as `NON_ADMINISTRATIVE_ROLES`: unset or
empty keeps the default, `none` clears it, inline `#` comments are stripped. See
[CONFIGURATION.md](CONFIGURATION.md#role-authority).

### Gate 2 — the unit must be yours

Standing is then computed **from the qualifying roles alone**. A `prov-admin` of
LA47 who is also `picr` of R07 reaches LA47 through these endpoints and nothing
else — the region role does not qualify, so it does not contribute. There is no
separate permission table; the reach is exactly the administrator roles' reach.

Two conditions, both required, both judged on the unit's chain **as resolved
from the database** — a caller cannot assert a province they are not in:

1. **The unit sits strictly below your level.** A province officer promotes
   parishes and areas, and repairs areas and zones. They do not promote their
   province — that is the seat they hold, and it is decided one level up.
2. **The unit is inside your unit.** A parish in LA99 is not a province officer
   of LA47's to promote, however senior the role.

| Caller | Operation | Result |
|---|---|---|
| `prov-admin` of LA47 | promote a parish in LA47 → area | ✅ `authorisedVia: "province LA47"` |
| `prov-admin` of LA47 | promote an area in LA47 → zone | ✅ |
| `prov-admin` of LA47 | promote a parish in LA99 | ❌ `403` — outside their unit |
| `prov-admin` of LA47 | promote province LA47 → region | ❌ `403` — at their own level |
| `prov-admin` of LA47 | promote with `absorbCodes` | ❌ `403` — super-admin option |
| `prov-admin` of LA47 **and** `picr` of R07 | promote province LA47 → region | ❌ `403` — `picr` does not qualify, so region standing never forms |
| `picp` of LA47 | anything | ❌ `403` — head of unit, not administrator |
| `prov-asst-admin` of LA47 | anything | ❌ `403` — assistant |
| `reg-admin` of R07 | promote province LA47 → region | ✅ provinces are below region |
| `reg-admin` of R07 | promote a parish anywhere in R07 | ✅ |
| `prov-admin` of LA47 | realign an area split across two zones **in LA47** | ✅ |
| `prov-admin` of LA47 | realign province LA47 that has drifted into R12 | ❌ `403` — one side is outside their unit |
| `reg-admin` of R07 | realign that same province back into R07 | ✅ if both sides are in R07; ❌ if the drifted side is another region |
| super-admin | anything | ✅ `authorisedVia: "super-admin"` |

A caller holding several **qualifying** roles is authorised by the most specific
standing that contains the unit. Someone who is both `reg-admin` of R07 and
`prov-admin` of LA47 acts on a parish in LA47 as `province LA47`, and on a
parish in LA99 (inside R07) as `region R07`. A non-qualifying role never
contributes, however senior.

### Why `absorbCodes` stays super-admin

Absorption pulls **other** units under the promoted one. Each of those may sit
in someone else's unit, and even inside the caller's own it is the one part of a
promotion with real blast radius. So it is refused on the scoped endpoint
regardless of standing. Promote the unit on its own, then move the others under
it one at a time with `/transfer` — which applies the same standing check to
each — or ask a super-admin to run the promotion with absorption.

### Why a scoped realign checks every variant

A split unit is, by definition, partly somewhere else. If that somewhere is
outside the caller's unit, pulling it back in is exactly the cross-boundary
decision `/transfer` reserves for a super-admin, and a repair does not get to
bypass it. So every variant's chain is checked against the destination with the
same `canTransfer` rule; one failing variant refuses the whole request, and the
message says which.

---

## POST /v1/hierarchy-transfers/scoped/promote

Identical body to `/promote`, minus `absorbCodes`.

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"fromLevel":"parish","unitCode":"211549","newName":"AREA TWENTY","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/scoped/promote"
```

| Field | Notes |
|---|---|
| `fromLevel`, `unitCode` | **required** — the target level is derived, never sent |
| `newName` | defaults to the promoted unit's own name |
| `newCode` | **required for province and above**, whose codes are meaningful; minted for area and zone |
| `reason` | recorded on the job |
| `dryRun` | see the counts and the authority first |

Dry run:

```jsonc
{
  "dryRun": true,
  "authorisedVia": "province LA47",
  "fromLevel": "parish", "newLevel": "area",
  "unitCode": "211549",
  "newCode": "(would be generated)",
  "newName": "AREA TWENTY",
  "absorbing": [],
  "cascade": { "membersMatched": 1, "usersMatched": 212 }
}
```

`authorisedVia` names the standing that permitted it. If it says
`"super-admin"` the caller is one.

Applied, the response carries `jobId`, the minted `newCode`, and the cascade
counts, exactly as `/promote` does. See
[HIERARCHY_TRANSFER_DOCS.md](HIERARCHY_TRANSFER_DOCS.md#post-v1hierarchy-transferspromote)
for what a promotion writes.

---

## POST /v1/hierarchy-transfers/scoped/realign

Identical body to `/admin/realign`.

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"level":"area","unitCode":"AR8000000211","toLevel":"zone","toCode":"ZN01","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/scoped/realign"
```

`toLevel` must be the unit's **immediate parent**, for the same reason as the
unbounded endpoint: naming a grandparent would leave the intervening field
absent from the destination chain and the patch would blank it.

The dry run is the same shape as `/admin/realign` —
[`variants`](HIERARCHY_TRANSFER_DOCS.md#the-dry-run-which-you-should-always-read-first),
`inherited`, `cascade`, `officesAtRisk` — plus `authorisedVia`. **Read
`variants` before applying**: it names both sides of the split, and which side
wins is not recoverable from the data afterwards.

Everything a realign does and does not do is unchanged: departments are
included, inactive rows are compared, and **principal offices are reported but
never ended**.

---

## What is logged

Every applied scoped operation writes an activity-log entry, and the entry names
the standing it ran under:

```
HIERARCHY_PROMOTION  module HIERARCHY_TRANSFER  status SUCCESS
Actor jdoe promoted parish 211549 to area AR8000000340 (AREA TWENTY)
— parishes updated: 1, users updated: 212
via /v1/hierarchy-transfers/scoped/promote — job ID: 68b1…
— authorised as province LA47
```

```
HIERARCHY_REALIGN  module HIERARCHY_TRANSFER  status SUCCESS
Actor jdoe realigned area AR8000000211 under zone ZN01, collapsing 2 variant(s)
of zoneCode, zoneName, provinceCode, … — parishes matched: 14, modified: 9;
users matched: 380, modified: 244; offices left standing: 1
via /v1/hierarchy-transfers/scoped/realign — job ID: 68b1…
— authorised as province LA47
```

A refusal is logged too, with status `FAILED` and the reason. The
`hierarchyChangeJobs` row records `requestedBy` and `requestedByUsername` as
before; the URL in the activity log is what distinguishes a scoped run from an
unbounded one.

---

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `TRANSFER_NOT_PERMITTED` | `403` | Not an administrator role (the message lists the roles that qualify); or outside your unit, at or above your level, `absorbCodes` sent to the scoped endpoint, or — for a realign — one side of the split lies outside your unit. The message names which. |
| `INCONSISTENT_UNIT` | `409` | Promote: the unit's members disagree — realign it first. Realign: the **destination** is itself split. |
| `DESTINATION_NOT_PARENT_LEVEL` | `400` | Realign `toLevel` is not the immediate parent. `detail` names the level expected. |
| `ALREADY_ALIGNED` | `400` | Every variant already matches the destination. Nothing to do. |
| `CODE_REQUIRED` | `400` | Promoting to province or above needs `newCode`. |
| `CODE_TAKEN` | `400` | The supplied code is in use. |

The existing hierarchy codes (`EMPTY_UNIT`, `NO_PARENT_LEVEL`,
`IDENTITY_VIOLATION`, …) apply unchanged.

---

## Frontend guidance

1. **Show the promote/realign controls only to administrators** — the four
   slugs above — and to super-admins. Everyone else gets a `403` whose message
   is safe to display.
2. **Call the scoped endpoint by default** for any administrator who is not a
   super-admin. The unbounded endpoints will 403 them anyway.
3. **Show `authorisedVia`** from the dry run. It tells the officer which of
   their roles is doing the work, which matters when they hold several.
4. **Hide `absorbCodes`** on the scoped promotion form. It is refused.
5. For a realign, **render `variants` and make the officer pick** which side is
   right before sending `dryRun: false`. The endpoint does not guess and neither
   should the UI.
6. A `403` whose message mentions "outside your unit" or "super-admin decision"
   is not an error to retry — it is a request that needs escalating. Say so.
