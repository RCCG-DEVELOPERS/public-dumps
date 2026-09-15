# Who may change the hierarchy

Which officer may restructure what, and why three different operations all get
described as "making a parish an area".

- [Three different operations](#three-different-operations)
- [The authority table](#the-authority-table)
- [Telling the three refusals apart](#telling-the-three-refusals-apart)
- [What a province admin can do today](#what-a-province-admin-can-do-today)
- [What a province admin cannot do](#what-a-province-admin-cannot-do)
- [When an officer has no standing at all](#when-an-officer-has-no-standing-at-all)
- [The open question](#the-open-question)

---

## Three different operations

"Make this parish an area" means one of three things, and they are not
variations of each other — different endpoints, different rules, different
authority.

### 1. Make a parish the HEAD of an area

The area already exists. You are choosing which of its parishes represents it.
Nothing is created, nothing moves; one parish gains the `ahq` flag and the flags
beneath it.

```
POST /v1/hq-assignments/assign
```

**A province admin may do this** for any area or zone inside their province.
See [HQ_ASSIGNMENT_DOCS.md](HQ_ASSIGNMENT_DOCS.md#post-v1hq-assignmentsassign).

### 2. PROMOTE a parish so that it becomes an area

Structural. The parish stops being a parish and becomes an area, a new area code
is minted, and other units may be absorbed under it. This is the one that
changes the shape of the tree.

```
POST /v1/hierarchy-transfers/promote
```

**Super-admin only.** See
[HIERARCHY_TRANSFER_DOCS.md](HIERARCHY_TRANSFER_DOCS.md#endpoints).

### 3. Write the flag directly on the parish row

```
PATCH /v1/parishDirectory/:id   { "ahq": "1" }
```

**Super-admin only, and deliberately a dead end even then** — this route applies
none of the headquarters rules. It cannot tell that another parish already holds
`ahq` for that area, and it will not set the flags beneath the one you asked
for. It is the route that produced `phq: "2"` and the duplicate holders.

If the frontend still makes a headquarters this way, that is the bug. Move it to
operation 1.

---

## The authority table

| Operation | Endpoint | Who |
|---|---|---|
| Read a unit's head | `GET /v1/hq-assignments/holders` | standing **at** that unit or above |
| Give a parish a headship | `POST /v1/hq-assignments/assign` | standing **strictly above** the unit |
| Take a headship away | `POST /v1/hq-assignments/vacate` | standing **strictly above** the unit |
| Settle two parishes claiming one unit | `POST /v1/hq-assignments/resolve-conflict` | standing **strictly above** the unit |
| Move a unit within your own unit | `POST /v1/hierarchy-transfers/transfer` | standing **strictly above** the moved unit, and **both ends inside** it |
| Move a unit anywhere | `POST /v1/hierarchy-transfers/admin/move` | super-admin |
| Raise a unit one level, anywhere | `POST /v1/hierarchy-transfers/promote` | super-admin |
| Raise a unit one level, within your own unit | `POST /v1/hierarchy-transfers/scoped/promote` | **an administrator** (`prov-admin`, `reg-admin`, `sub-cont-admin`, `cont-admin`) with standing **strictly above** the unit; no `absorbCodes` |
| Repair a split unit, anywhere | `POST /v1/hierarchy-transfers/admin/realign` | super-admin |
| Repair a split unit, within your own unit | `POST /v1/hierarchy-transfers/scoped/realign` | **an administrator** with standing over **both ends of every variant** |
| Set a flag on the row | `PATCH /v1/parishDirectory/:id` | super-admin (and still does not apply the rules) |

**"Strictly above" is the rule that surprises people.** A province admin decides
which parish heads an area or a zone *inside* their province. They do not decide
which parish heads the province itself — that is their own seat, and a unit's
head is settled one level up, by a region officer or a super-admin.

The two authority checks are
[`assertCanAdminister`](src/components/Hqassignments/service.ts#L155) for
headquarters and [`canTransfer`](src/utils/officerAuthority.ts#L212) for moves.
Both derive authority from the caller's own roles through
[`standingOf`](src/utils/officerAuthority.ts#L103) — there is no separate
permission table, because a second source of truth for "who may act at province
level" would immediately disagree with the first.

---

## Telling the three refusals apart

All three read as "only a super-admin can do this". The wording tells you which
one you hit, and they need different fixes.

| Message | Endpoint | What it means |
|---|---|---|
| *"Only a super-admin may set `ahq` directly, and this route does not apply the headquarters rules even then. Use POST /v1/hq-assignments/assign…"* | `PATCH /v1/parishDirectory/:id` | **The frontend is on the wrong endpoint.** Move it to `/v1/hq-assignments/assign`, which a province admin may call. |
| *"You do not have the required role to perform this action"* | `POST /v1/hierarchy-transfers/promote` | **Working as designed.** Promotion is super-admin only. |
| *"You hold standing AT province LA47, which is not enough to decide its own headquarters…"* | `POST /v1/hq-assignments/assign` | You reached the right endpoint, but aimed it at your own level. A unit's head is settled one level above it. |

A fourth, distinct from all of these:

> *"You have no standing above area AR01."*

means `standingOf` found no province for you at all — see
[below](#when-an-officer-has-no-standing-at-all).

---

## What a province admin can do today

All of this already exists. None of it needs a super-admin.

### Choose which parish heads an area or a zone

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"level":"area","unitCode":"AR01","parishCode":"211549","dryRun":true}' \
  "$API_HOST/v1/hq-assignments/assign"
```

Run it with `"dryRun": true` first. The response names every flag it would set,
every conflict it would create, and the unit the decision was authorised under:

```jsonc
{
  "dryRun": true,
  "level": "area",
  "unitCode": "AR01",
  "authorisedVia": "province LA47",
  "assigned": { "parishCode": "211549", "flagsSet": { "ahq": "1" } },
  "conflicts": [ /* parishes already holding ahq for AR01 */ ]
}
```

**An assignment never clears another parish's flag.** If a second parish already
holds `ahq` for that area, it is reported in `conflicts` and left alone —
displacing a headquarters is a decision, not a side effect. Settle it with
`/vacate` or `/resolve-conflict`.

### Move a parish between areas inside the province

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"level":"parish","unitCode":"211549","toLevel":"area","toCode":"AR02","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/transfer"
```

The same officer may move an **area** between zones, as long as **both** the
source and the destination stay inside LA47. A move that leaves the province
needs a super-admin, because it hands a unit to another province and neither
province's admin should make that call alone.

In practice that means **parishes and areas**. A zone is strictly below province
by rank, so the level check passes — but a zone's parent *is* the province, so
any real move takes it out of LA47 and fails the both-ends test. Moving a zone
is a super-admin action.

### Promote a parish to an area, or an area to a zone, inside the province

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"fromLevel":"parish","unitCode":"211549","newName":"AREA TWENTY","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/scoped/promote"
```

Same body and same effect as the super-admin `/promote`, bounded by standing.
`absorbCodes` is refused. See
[HIERARCHY_SCOPED_OPERATIONS_DOCS.md](HIERARCHY_SCOPED_OPERATIONS_DOCS.md).

### Repair a split area or zone inside the province

```bash
curl -X POST -H "Authorization: Bearer $PROV_ADMIN_JWT" \
  -H 'Content-Type: application/json' \
  -d '{"level":"area","unitCode":"AR8000000211","toLevel":"zone","toCode":"ZN01","dryRun":true}' \
  "$API_HOST/v1/hierarchy-transfers/scoped/realign"
```

Every variant of the split must lie inside the province. A province that has
drifted into another region is a region officer's or super-admin's to repair.

### Read before deciding

```bash
GET /v1/hq-assignments/holders?level=area&unitCode=AR01   # who heads it now
GET /v1/hierarchy-transfers/preview                       # what a move would cascade
```

`/holders` is the scoped read: it names a unit, so it can be checked against
your standing, and it answers for any unit you have standing at or above.

**`/vacancies` and `/conflicts` are super-admin only** — they sweep the entire
collection rather than naming a unit, so there is nothing to check standing
against. A scoped officer asks `/holders` per unit instead. The refusal says so.

Every write also takes `"dryRun": true`. Use it: the response tells you what
would change and which unit authorised it, before anything is written.

---

## What a province admin cannot do

| | Why |
|---|---|
| Decide which parish heads **their own province** | A unit's head is settled one level above it. Ask a region officer or a super-admin. |
| **Promote** with `absorbCodes` | Pulling other units under the promoted one has real blast radius and may cross into someone else's unit. Super-admin only — promote on its own through `/scoped/promote`, then `/transfer` the others under it. |
| Move a unit **out of** their province | Both ends must be inside their own unit. |
| **Realign** a unit one side of which lies **outside** their province | Pulling a drifted half back in is the cross-boundary decision `/transfer` reserves for a super-admin, and a repair does not bypass it. A split wholly inside their province is theirs to repair through `/scoped/realign`. |
| Move a **province** | A province sits at their own level, not below it. |
| Write `ahq`/`zhq`/`phq` on the parish row directly | The route applies none of the rules. Use `/v1/hq-assignments`. |

---

## When an officer has no standing at all

If a genuine `prov-admin` is refused with *"You have no standing above…"*, the
role is not the problem — the profile is.
[`standingOf`](src/utils/officerAuthority.ts#L103) needs **both**:

1. a role whose `level_type` is `province`, and
2. a `province` code on the caller's own user row

A role that sounds senior but sits against a user with no province code yields
no unit to act in, so every scoped check refuses. Check both:

```bash
GET /v1/users/me/roles     # the roles held, and the scope each would apply
```

Two other things that silently remove standing:

- **A role listed in `NON_ADMINISTRATIVE_ROLES`** contributes no unit at all, by
  design. `training-manager` is there today. See
  [CONFIGURATION.md](CONFIGURATION.md#role-authority).
- **Standing is read from the roles held, not the active role.** This is the
  opposite of role *assignment*, which is judged on the active role alone — see
  [ROLE_ASSIGNMENT_DOCS.md](ROLE_ASSIGNMENT_DOCS.md#why-the-active-role). The two
  are separate axes deliberately: a training manager keeps the power to appoint
  training managers while having no administrative standing anywhere.

---

## The open question — resolved

`POST /v1/hierarchy-transfers/promote` remains super-admin only, and now has a
scoped twin. `POST /v1/hierarchy-transfers/scoped/promote` applies the rule that
already governed transfers and headquarters — *strictly below your level, inside
your unit* — and refuses `absorbCodes`, which is the one part of a promotion
with real blast radius. `/admin/realign` gained `/scoped/realign` on the same
terms, with the extra condition that **every** variant of the split must lie
inside the caller's unit.

Both are documented in
[HIERARCHY_SCOPED_OPERATIONS_DOCS.md](HIERARCHY_SCOPED_OPERATIONS_DOCS.md).
