# Moving Units Across the Hierarchy — The Guide

How a parish, area or zone gets moved, who may do it, what happens when the
data underneath disagrees with itself, and how a move that crosses a province
boundary gets approved. Written after the September 2026 changes; the
endpoint-by-endpoint references it leans on are listed at the end.

- [Who this is for](#who-this-is-for)
- [The model in five sentences](#the-model-in-five-sentences)
- [Who can do what](#who-can-do-what)
- [Walkthrough 1 — a move inside your own province](#walkthrough-1--a-move-inside-your-own-province)
- [Walkthrough 2 — a move into another province](#walkthrough-2--a-move-into-another-province)
- [Walkthrough 3 — the unit you are touching is split](#walkthrough-3--the-unit-you-are-touching-is-split)
- [Walkthrough 4 — repairing a split unit with realign](#walkthrough-4--repairing-a-split-unit-with-realign)
- [National support](#national-support)
- [The approval lifecycle](#the-approval-lifecycle)
- [Confirming the size of a change](#confirming-the-size-of-a-change)
- [Undoing a change](#undoing-a-change)
- [Headquarters, and moving](#headquarters-and-moving)
- [Who may realign what](#who-may-realign-what)
- [Error codes, consolidated](#error-codes-consolidated)
- [Frontend integration checklist](#frontend-integration-checklist)
- [Operations](#operations)
- [What is deliberately not done yet](#what-is-deliberately-not-done-yet)
- [Reference documents](#reference-documents)

---

## Who this is for

Province, region and national administrators who move units; the frontend team
wiring the screens; and whoever is on support when an admin says "it won't let
me move this parish". It explains the *behaviour and the reasons*. For exact
request and response shapes go to the reference documents at the end.

---

## The model in five sentences

1. **A unit is not a record; it is a set of parish rows.** There is no
   `areaDirectory`. Area `AR8000000211` *is* every `parishDirectory` row whose
   `areaCode` is `AR8000000211`, and its ancestry — zone, province, region,
   sub-continent, continent, each as a code and a name — is whatever those rows
   say.
2. **A move keeps the unit's own code and inherits everything above it from
   the new parent.** Moving a parish into an area rewrites the parish's zone,
   province, region, sub-continent and continent (codes *and* names) to the
   area's, and does the same to every user whose profile points at the parish.
   Nothing is minted; only a *promotion* mints a code.
3. **Authority is standing.** You act at the levels your roles sit at, and only
   within your own unit at each. A `prov-admin` of LA47 may move anything below
   province level *within* LA47. Both ends of the move must be inside their
   unit.
4. **Super-admin and national support are unbounded** — over the hierarchy,
   principal officers, headquarters assignment and approvals, and over nothing
   else.
5. **A move across a province boundary is two provinces' decision, so it is an
   approval:** the giving province asks, the receiving province agrees.

---

## Who can do what

| Caller | Move | Outcome |
|---|---|---|
| `prov-admin` of LA47 | parish or area **within** LA47 | ✅ `POST /transfer`, `authorisedVia: "province LA47"` |
| `prov-admin` of LA47 | parish **out of** LA47 into LA99 | ❌ `403 TRANSFER_NEEDS_APPROVAL` → raise `POST /approvals/unit-transfer`; LA99's admin approves |
| `prov-admin` of LA47 | parish **into** LA47 from LA99 | ❌ `403 TRANSFER_NOT_PERMITTED` — not theirs; LA99's admin must raise the request, and LA47's admin then approves it |
| `prov-admin` of LA47 | a province | ❌ at or above their own level |
| `reg-admin` of R36 | parish LA47 → LA99, both inside R36 | ✅ the move never leaves R36 |
| `prov-admin` of LA47 | approve a request **into** LA47 | ✅ the receiving province decides |
| `prov-admin` of LA47 | approve a request **out of** LA47 | ❌ the giving province already spoke by raising it |
| super-admin, national support | any move, any approval, `/admin/move`, `/promote`, `/admin/realign` | ✅ logged as `super-admin` / `elevated:nat-support` |
| anyone | approve a request they raised themselves | ❌ (super-admin exempt; national support **not** exempt) |

> **The chains decide, not the request.** Authority is judged after both ends
> are resolved from the database, so nobody can assert a province they are not
> in by typing it into the body. This is also why a province admin trying to
> *pull* a parish in from another province is refused: the parish's *current*
> province is read from its rows, and it is not theirs.

---

## Walkthrough 1 — a move inside your own province

You are `prov-admin` of OG02 and want parish `609855` under area `AR0000043333`,
also in OG02.

```
GET  /v1/hierarchy-transfers/units?level=area&parentCode=<zone>   # pick the destination
GET  /v1/hierarchy-transfers/preview?level=parish&unitCode=609855&toParentCode=AR0000043333
POST /v1/hierarchy-transfers/transfer
     { "level": "parish", "unitCode": "609855", "toParentCode": "AR0000043333",
       "reason": "Boundary review", "dryRun": true }
POST /v1/hierarchy-transfers/transfer   { …same…, "dryRun": false }
```

The preview tells you `permitted`, `willAffect` (how many parish rows and users
change), `changes` (which ancestor fields differ) and — new — the split report
(see Walkthrough 3). The destination's *level* is never sent; it is always one
rank above `level`.

The real run writes three things, recorded as steps on a job row you can read
back at `GET /jobs`: the parish rows, the users under them, and any principal
office at a unit the moved people have actually **left** (a parish moving
between two areas of the same province no longer costs the province officer
their appointment — that was a bug, fixed in September 2026).

---

## Walkthrough 2 — a move into another province

Same admin, but parish `609855` currently sits in province **OG23** and you are
`prov-admin` of **OG02**. This is the case that produced *"A move across
province boundaries needs a super-admin"*.

Read the message carefully: *"it is currently in province OG23"* means the
**source** is not yours. You are trying to pull a parish in. **You cannot raise
this request** — OG23's admin must — and you will be the one who approves it.

Now the mirror image, which is the common one. You are `prov-admin` of **OG23**
and want to give parish `609855` to area `AR0000043333` in **OG02**:

```
1. OG23 admin        POST /v1/hierarchy-transfers/transfer  { level, unitCode, toParentCode }
                     → 403 TRANSFER_NEEDS_APPROVAL
                       detail.requestEndpoint = "/v1/approvals/unit-transfer"
                       detail.requestBody     = { level, unitCode, toParentCode, reason }
                       detail.toProvince      = "OG02"

2. OG23 admin        POST /v1/approvals/unit-transfer   <detail.requestBody>
                     → 201  status PENDING, fromProvince OG23, toProvince OG02,
                            planSnapshot = both chains + inherited fields + split report

3. OG02 admin        GET  /v1/approvals?requestType=UNIT_TRANSFER&status=PENDING&toProvince=OG02

4. OG02 admin        POST /v1/approvals/:id/approve   { "decisionNote": "Agreed with the RPO" }
                     → 200  status APPROVED
                            executionResult.jobId, .authorisedVia =
                              "approval <id> — requested by <OG23 admin> (province OG23),
                               approved by <OG02 admin> (province OG02)"
```

`GET /preview` says the same thing before anyone clicks: `permissionCode:
"TRANSFER_NEEDS_APPROVAL"` and an `approvalHint` with the same body.

What the approval guarantees:

- **Both chains are resolved when the request is raised**, through the same
  planner `/transfer` uses. A same-province move is refused (`SAME_PROVINCE` —
  use `/transfer`); an unknown unit or one already under the destination is
  refused then, not at approval.
- **The snapshot is never trusted at approval.** The unit is re-planned and both
  provinces re-checked. If either changed in between — someone moved the
  destination area into a third province — the request lands `FAILED` with
  `executionResult.code = "STALE_REQUEST"` and nothing is written. The
  approver's authority was judged on the province they hold; it must still be
  the province receiving the unit.
- **The approval is the authority.** Neither admin alone holds both ends, so the
  move runs through `transfer()` with no standing, exactly as `/admin/move` does
  for a super-admin — and the job row records `approvalRequestId` and names both
  parties. It is never logged as `super-admin`.
- **Only parish, area and zone can be requested.** A province or above is moved
  by a super-admin or national support through `/admin/move`
  (`LEVEL_NOT_REQUESTABLE`).

---

## Walkthrough 3 — the unit you are touching is split

A unit is **split** when its rows disagree about their ancestors. The classic
cause: a parish holding a province's `phq` flag is moved to another region, only
that one row changes, and the province now names two regions. A subtler cause
that turned out to be the common one: the *same* region spelled two ways —
`"REGION 71"` on most rows, `"Region 71"` on a few. Both count.

**Until September 2026 a split refused every move touching it** with `409
INCONSISTENT_UNIT`, and most destination areas in a split province were exactly
that. Admins could not tidy their own province from the inside.

**Now the move proceeds.** The unit resolves to its **dominant variant** — the
ancestry the most active, non-department rows claim — and the response tells
you what it did:

```json
{
  "sourceWasSplit": false,
  "destinationWasSplit": true,
  "destinationVariants": [
    { "index": 0, "chain": { "provinceName": "PROV ONE", "…": "…" }, "memberCount": 9, "dominant": true,  "selected": true  },
    { "index": 1, "chain": { "provinceName": "Prov One", "…": "…" }, "memberCount": 1, "dominant": false, "selected": false }
  ],
  "realignHint": {
    "endpoint": "/v1/hierarchy-transfers/admin/realign",
    "body": { "level": "area", "unitCode": "AR0000043333", "toLevel": "zone",
              "toCode": "ZN…", "destinationVariant": 0, "dryRun": true }
  },
  "warnings": [
    "destination area AR0000043333 is split into 2 variants; the dominant variant (9 members) was inherited — realign the destination next (see realignHint)."
  ]
}
```

Three rules keep this from becoming a hole:

- **Authority is judged against every variant of both ends.** A split unit is,
  by definition, partly somewhere else. If a minority variant of the unit lies
  in a province you do not hold, the move is refused and the message names the
  variant. A split cannot smuggle a cross-province move past a province admin.
- **`ALREADY_THERE` is judged per variant.** Refused only when *every* variant
  already sits under the destination; when only some do, the move proceeds as a
  repair and says so in `warnings`.
- **Promotion is the exception and stays strict.** Minting a fresh code over two
  ancestries would create a split one level up. Realign first, then promote.

The moved unit inherits the dominant chain. The destination itself is still
split — that is what `realignHint` is for. **Move first, then realign**, which
is the reverse of the old order.

The `GET /integrity` report now lists splits by *name* as well as by code
(`nameOnly: true` when every code agrees and only a label differs) and carries a
`suggestedRealign` body per unit.

---

## Walkthrough 4 — repairing a split unit with realign

Realign forces every row of a unit — departments and inactive rows included —
and every user under it onto one ancestry. It works at **every** level pair:
parish → area, area → zone, zone → province, province → region, region →
sub-continent. The owner's original case was province LA20 sitting under
regions R11 and R71 at once:

```
POST /v1/hierarchy-transfers/admin/realign
{ "level": "province", "unitCode": "LA20", "toLevel": "region", "toCode": "R71", "dryRun": true }
```

That is the same repair as the two manual `updateMany` calls people were
running by hand — except it sets `regionName` from R71's own rows rather than
from a typed literal, rewrites the sub-continent and continent too, and records
a job row.

**The destination may itself be split.** Region R71's own parishes might
disagree on `regionName` casing. That used to block the repair with
`INCONSISTENT_UNIT` — the disease locking the cure. Now the dry run shows you
R71's variants and lets you choose:

```json
{
  "dryRun": true,
  "destinationWasSplit": true,
  "destinationVariants": [
    { "index": 0, "chain": { "regionCode": "R71", "regionName": "REGION 71", "…": "…" },
      "memberCount": 214, "samples": ["609855", "…"], "dominant": true, "selected": true },
    { "index": 1, "chain": { "regionCode": "R71", "regionName": "Region 71", "…": "…" },
      "memberCount": 3, "samples": ["…"], "dominant": false, "selected": false }
  ],
  "variants": [ "…the source's own variants, as before…" ],
  "cascade": { "membersMatched": 41, "usersMatched": 380, "wouldChange": 12, "usersUnreachable": 2 },
  "officesAtRisk": [],
  "applyWith": {
    "destinationVariant": 0,
    "expectedDestinationChain": { "regionCode": "R71", "regionName": "REGION 71", "…": "…" }
  }
}
```

To apply, merge `applyWith` into the same body with `dryRun: false`. To write
the *other* spelling instead, send `destinationVariant: 1` on the dry run and
apply with the `applyWith` that comes back. The `expectedDestinationChain` echo
is the safety: a realign collapses every variant of a unit onto one answer and
that is not recoverable afterwards, so if R71's rows change between your dry run
and your apply, the apply refuses with `409 DESTINATION_CHAIN_CHANGED` instead of
writing a chain nobody saw. Applying a split destination *without* the echo
works, but the response warns you.

Two things realign still does not do, on purpose: it does not end principal
offices (`officesAtRisk` reports them; a data repair must not fire an
appointment), and it does not touch users whose own `province` column is blank
or points elsewhere (`usersUnreachable` counts them; fix the profiles first).

`/scoped/realign` is the same call for a unit's administrator, bounded by
standing over every source variant against every destination variant.

---

## National support

`nat-support` used to see everything and be able to do nothing: its role level
is `national`, which gives it no unit, and every hierarchy, officer, HQ and
approval check refused it. It now acts **with super-admin authority over exactly
these surfaces, and no others**:

| Surface | Granted |
|---|---|
| Hierarchy | `/transfer`, `/admin/move`, `/promote`, `/admin/realign`, `/scoped/*`, `GET /jobs`, `GET /integrity` — including cross-province moves and `absorbCodes` |
| Principal officers | every route: roster, appoint, end, transfer, `/admin/strip-roles` |
| Headquarters | reads, assign, vacate, resolve-conflict, vacancies, conflicts, `GET /integrity` |
| Approvals | approve, reject, cancel a request of **any** type |

**Not granted:** geofencing rules, config or exemptions; AppIcons; secondary
roles; account status and forced password change; `/impersonate` and
`/impersonate-role`; role granting; API keys; `/admin/backfill-codes`; the HQ
repair routes; the login geofence bypass; and the **self-approval exemption** —
national support cannot approve a request it raised.

Configured by `ELEVATED_ROLES` (default `nat-support`; `none` hands everything
back to super-admin). Every action taken on it is logged as
`authorisedVia: "elevated:nat-support"`, never `"super-admin"`. Hold the role as a
primary role: elevation on the standing-based routes is judged on `users.roles`.

---

## The approval lifecycle

```
                      raise                         approve
   (giving province) ──────►  PENDING  ─────────────────────────►  EXECUTING  ──► APPROVED
                                 │                                     │
                                 ├── reject  ──► REJECTED              └──────────► FAILED
                                 └── cancel  ──► CANCELLED               (change did not apply;
                                                                          executionResult.error/.code)
```

- **The decision is claimed atomically.** `approve` flips `PENDING →
  EXECUTING` in one write *before* anything runs. Two approvers arriving
  together cannot both execute a request; the second is told it was decided a
  moment ago. `reject` and `cancel` take the same claim.
- **`EXECUTING` is a state you should rarely see.** If a request stays there
  after a refresh, execution crashed mid-way; treat it like `FAILED` and get a
  human.
- **`FAILED` is not `REJECTED`.** Rejected means someone said no. Failed means
  someone said yes and it did not work — `executionResult.code` says why
  (`STALE_REQUEST`, `ALREADY_THERE`, `EMPTY_UNIT`, …).
- **One live request per subject per type.** For a unit that is one pending or
  executing `UNIT_TRANSFER` per unit, enforced by the same unique index that
  guards the person-based types.
- `cancel` is for the person who raised the request, a super-admin or national
  support.

---

## Confirming the size of a change

A cascade is correct and can still be far larger than the person asking for it
pictured. Realigning a **zone** rewrites every parish under it — and a zone that
happens to be its province's only zone carries the whole province with it.
Nothing in the request distinguishes that from a zone of six.

So past **500 parish rows** an apply must state the number it expects to touch.

```
POST /v1/hierarchy-transfers/admin/realign
{ "level": "zone", "unitCode": "Z123", "toLevel": "province", "toCode": "LA47",
  "dryRun": true }
```

The dry run answers with, among the rest:

```json
{ "cascade": { "membersMatched": 4312 },
  "applyWith": { "destinationVariant": 0,
                 "expectedDestinationChain": { "...": "..." },
                 "expectedMembers": 4312 } }
```

**Read `membersMatched` before you go on.** If it is not roughly the size of the
unit you have in mind, stop — you are not moving what you think you are moving.
Then apply with `applyWith` pasted in.

Applying without it, above the threshold, is refused:

| | |
|---|---|
| `CONFIRMATION_REQUIRED` | 409, with `detail.actualMembers` and `detail.resendWith`. Nothing was written and no job row was created. |
| `MEMBER_COUNT_CHANGED` | 409. You said a number and it was wrong — either the dry run is stale or this is not the unit you meant. |

Below 500 rows nothing changes; `expectedMembers` stays optional and is checked
only if you send it. Set `HIERARCHY_CONFIRM_ABOVE_ROWS` to lower the threshold,
or to `0` to require confirmation for every change regardless of size.

**Approved transfers are exempt.** Two people have already agreed to that exact
unit, and the approver executes a request raised days earlier with no body of
their own to echo a count into.

---

## Undoing a change

Every transfer, realign and promotion records the rows it is about to overwrite —
grouped by the values they currently hold — before it writes. That is what makes
an exact undo possible: a realign collapses several ancestries into one, and the
job's `previousChain` says only that there *were* several, not which rows held
which.

```
POST /v1/hierarchy-transfers/jobs/<jobId>/rollback   { "dryRun": true }
```

```json
{ "dryRun": true, "operation": "REALIGN", "unitCode": "Z123",
  "wouldRestore": 4312, "wouldSkip": 0 }
```

Then the same call without `dryRun`. Each group goes back to **its own** values,
so a unit that was split is restored split rather than flattened.

**Rows changed since are skipped, not dragged back.** A row is restored only
while it still carries exactly what the original job wrote, so undoing change #1
cannot silently undo change #2 riding on top of it. Those rows are counted in
`skipped` and listed in `notes`. `{"force": true}` restores them regardless —
use it only once you know what the later change was.

Two things a rollback reports rather than doing:

- **Principal offices vacated by a transfer are not reinstated.** Ending an
  appointment is a decision about a person; reversing it by machine days later
  would be a second such decision taken with nobody present. See
  `cascade.officesVacated` on the original job.
- **A code minted by a promotion stays allocated.** A released code that
  reappears elsewhere is worse than one merely unused.

`GET /jobs/<id>` shows one change in full, what was recorded against it, and
`reversible`. A rollback is itself a `ROLLBACK` job, linked both ways through
`rollbackOf` and `rollbackJobId`.

**Changes made before this existed cannot be rolled back automatically** — they
have no recorded before-state and are refused with `NO_SNAPSHOT` rather than
guessed at. Their `previousChain` says what the unit looked like; repair with a
fresh realign onto that parent.

---

## Headquarters, and moving

A parish row can carry a headship flag — `phq` marks it as its **province's**
headquarters, `zhq` its zone's, `ahq` its area's. The flag sits on the parish,
and a move rewrites that parish's ancestors. So moving it into another province
would leave the old province with **no** headquarters and the new one with
**two**, without saying so.

Moves now refuse that:

```json
{
  "status": 409,
  "code": "HQ_DEMOTION",
  "message": "RCCG HOUSE OF PRAYER (parish 211343) is the headquarters of province LA47, and realigning area AR80 would move it into province LA99. LA47 would be left without a headquarters, and LA99 would have two. Choose a new headquarters for LA47 first, then move this one.",
  "detail": {
    "stranded": [ { "parishCode": "211343", "level": "province",
                    "flag": "phq", "unitLosingItsHq": "LA47", "unitGainingIt": "LA99" } ],
    "useInstead": [
      { "useEndpoint": "/v1/hq-assignments/vacate",
        "useBody": { "parishCode": "211343", "flags": ["phq"] }, "why": "…" },
      { "useEndpoint": "/v1/hq-assignments/assign",
        "useBody": { "level": "province", "unitCode": "LA47", "parishCode": "<the new one>" }, "why": "…" }
    ]
  }
}
```

**It is judged per row, and only where the code actually changes.** An area
moving between two zones of the *same* province keeps its province, so a `phq`
holder inside it has lost nothing and is not flagged. The cascade still catches
that case: a province headquarters also carries `zhq` and `ahq`, so the zone
change trips on `zhq` instead.

`preview` and every dry run report the same rows in `hqDemotions`, so a UI can
refuse the button rather than let someone find out at apply time.

A super-admin or national support may proceed anyway with
`"acknowledgeDemotion": true`, which is recorded on the job. A scoped
administrator cannot — for them it is two deliberate steps.

---

## Who may realign what

| Caller | May realign | May not |
|---|---|---|
| `prov-admin` LA47 | a zone, area or parish that stays wholly inside LA47 | anything reaching into another province, at either end; a province, which is their own level |
| `reg-admin` R36 | anything below region level inside R36 — **including across provinces**, which is their remit | anything touching another region |
| super-admin, national support | anything, any level, across any boundary | — |

Both ends are judged, and **every variant of both ends**: a split unit whose
minority half lies in another province is refused, because ratifying it would
quietly pull that half across the boundary.

Being refused is not a dead end — `detail.useInstead` carries the same request
addressed to the endpoint that can run it, ready to hand to whoever can.

---

## Error codes, consolidated

| Code | Status | Where | Meaning and what to do |
|---|---|---|---|
| `TRANSFER_NOT_PERMITTED` | 403 | transfer, preview, scoped | Outside your unit, or at/above your level. The message names which end — and, for a split, which variant. Ask an administrator with standing. |
| `TRANSFER_NEEDS_APPROVAL` | 403 | transfer; `permissionCode` on preview | You hold the unit's province; the destination is in another. `detail.requestBody` → `POST /approvals/unit-transfer`. `detail.toProvince`'s admin decides. |
| `NO_STANDING_OVER_SOURCE` | 403 | approvals/unit-transfer | You are not a province admin of the province the unit is leaving. |
| `NOT_AUTHORISED_TO_DECIDE` | 403 | approve, reject | Wrong approver for this type — for a unit transfer, not the receiving province — or your own request. |
| `SAME_PROVINCE` | 400 | approvals/unit-transfer | Both ends in one province. Use `/transfer`. |
| `LEVEL_NOT_REQUESTABLE` | 400 | approvals/unit-transfer | Province or above. `/admin/move`. |
| `ALREADY_THERE` | 400 | transfer, request | Every variant already under that parent. `detail.realignHint` when the unit was split. |
| `EMPTY_UNIT` | 400 | all | No active, non-department row carries that code. |
| `DESTINATION_NOT_PARENT_LEVEL` | 400 | admin/move, realign | `toLevel` is not the immediate parent. `detail.expectedLevel`. |
| `DESTINATION_VARIANT_OUT_OF_RANGE` | 400 | realign | `destinationVariant` past the last variant; `detail.destinationVariants`. |
| `ALREADY_ALIGNED` | 400 | realign | Every variant already matches. Nothing to do. |
| `REQUEST_ALREADY_PENDING` | 409 | approvals | One live request per unit (or person) per type. |
| `PROVINCE_UNRESOLVED` | 409 | approvals/unit-transfer | The unit or destination has no province on its rows. Repair first. |
| `DESTINATION_CHAIN_CHANGED` | 409 | realign | The destination changed since your dry run. Re-run it and apply from its `applyWith`. |
| `INCONSISTENT_UNIT` | 409 | **promote only** | Split source. Realign, then promote. No longer raised by transfer, move, preview or realign. |
| `HQ_DEMOTION` | 409 | transfer, move, realign | The move would strand a headquarters. `detail.stranded`, `detail.useInstead`. Nothing written. |
| `RESTRUCTURE_NEEDS_ADMIN` | 403 | scoped/promote, scoped/realign | Not an administrator role. Moving a unit does not need one — use `/transfer`. |
| `INVALID_REQUEST` | 400 | all | The body failed validation. Previously thrown with no `code` at all. |
| `CONFIRMATION_REQUIRED` | 409 | transfer, move, realign, promote | Above the row threshold with no `expectedMembers`. `detail.actualMembers`, `detail.resendWith`. Nothing written. |
| `MEMBER_COUNT_CHANGED` | 409 | transfer, move, realign, promote | `expectedMembers` does not match reality. `detail.actualMembers`. |
| `NO_SNAPSHOT` | 400 | rollback | No before-state recorded — the job predates snapshots, or was too large to record. `detail.previousChain`. |
| `ALREADY_ROLLED_BACK` | 409 | rollback | Already undone; the message names the rollback job. |
| `CANNOT_ROLL_BACK_A_ROLLBACK` | 400 | rollback | Roll back the job it undid, or re-apply the original. |
| `JOB_NOT_FOUND` | 400 | jobs/:id, rollback | No such job. |
| `STALE_REQUEST` | — | `executionResult.code` | A province changed between raise and approve. Raise afresh. |

---

## Frontend integration checklist

1. **Drive every code from `/units`.** Nobody types a code.
2. **Call `/preview` first and branch on `permissionCode`.** Empty → offer
   Move. `TRANSFER_NEEDS_APPROVAL` → offer *Request transfer* and POST
   `approvalHint.requestBody` to `approvalHint.requestEndpoint`; show
   `approvalHint.toProvince` as who decides. `TRANSFER_NOT_PERMITTED` → "ask an
   administrator".
3. **Stop treating `409 INCONSISTENT_UNIT` as the split signal** on moves. Read
   `sourceWasSplit` / `destinationWasSplit`; when `true`, the move went through
   and `realignHint.body` is a ready dry-run to offer next.
4. **For realign, always dry-run, render `destinationVariants`, let the officer
   pick, and apply with `applyWith` merged in.** Never construct
   `expectedDestinationChain` by hand.
5. **The receiving province's inbox** is
   `GET /approvals?requestType=UNIT_TRANSFER&status=PENDING&toProvince=<mine>`.
   Show `planSnapshot` — both chains and the inherited fields — on the review
   screen.
6. **Treat `EXECUTING` as in progress and `FAILED` as needing a human.** Surface
   `executionResult.code`.
7. **Show `warnings` and `cascade.officesVacated` after any move.** A quiet
   removal of someone's provincial post is something the admin must see.
8. **Switch-role now returns `refreshToken`.** Store it; the switched session is
   renewable only with it.

---

## Operations

- **`ELEVATED_ROLES`** — see `CONFIGURATION.md` → *Who acts with super-admin
  authority over the hierarchy*. Unset keeps `nat-support`; `none` clears.
- **Finding splits:** `GET /v1/hierarchy-transfers/integrity` — `splitUnits`
  per level with `nameOnlyCount` and a `suggestedRealign` body per unit
  (capped at 50 codes; `count` is always the true total).
- **Jobs:** every move, promotion and realign is a row in `hierarchyChangeJobs`
  with steps and counts; a unit-transfer approval's job carries
  `approvalRequestId`. `GET /jobs` is super-admin / national support.
- **`detail` now reaches the client.** `sendHttpError` forwards it, so
  `detail.requestBody`, `detail.useInstead`, `detail.resendWith` and
  `detail.stranded` are readable. They were all being built and dropped before —
  including by a message that told the caller to read `detail.requestBody`.
  Messages are plain English for a person; the endpoints and bodies live in
  `detail` for the UI.
- **Snapshots:** `hierarchyChangeSnapshots` holds the before-state per job,
  grouped by ancestry and chunked at 2,000 keys. There is deliberately **no TTL**
  — expiring these silently removes the ability to undo, and the volume is small
  (a cascade shares its ancestry, so a 3,000-parish zone is one group). Past
  250,000 rows in one change the snapshot is abandoned and the job is marked
  `snapshotComplete: false`; rollback then refuses rather than half-reversing.
- **Backfill still pending:** `scripts/backfillUserStatus.ts` has not been run
  against a real database; until it is, existing users carry no `userStatus`,
  which reads as active. Dry-run first.
- **Indexes:** the new fields (`subjectUnitCode`, `subjectLevel`,
  `approvalRequestId`, `{subjectUnitCode, status}`) are declared on the models
  and created by Mongoose's `autoIndex` on boot. No existing index changed.
- **Local testing:** the DB-backed suites need
  `MONGODB_URI=mongodb://127.0.0.1:27017/orgtest npm test`. Fixtures are
  uniquely named and removed by `_id`; nothing in the suite drops a collection.

---

## What is deliberately not done yet

- **`GET /v1/approvals` is not scoped.** Any authenticated caller can list every
  request, of every type. The `toProvince` filter is a convenience, not a
  boundary. A proper per-province scoping is a follow-up for all types at once.
- **A unit request stores a synthetic subject.** `subjectUserId` holds
  `"unit:<level>:<code>"` so the existing one-pending-per-subject index enforces
  one pending request per unit with no index change on the live cluster.
  Retiring `uniq_pending_request_per_subject` and blanking that key is the
  documented clean-up.
- **No approver notifications**, for any type. Approvers poll the inbox.
- **Realign does not rewrite users whose own scope column is wrong**
  (`usersUnreachable`). Profiles first.
- **Promotion still refuses a split source.** Intentional; realign first.

---

## Reference documents

| Document | What it is |
|---|---|
| `HIERARCHY_TRANSFER_DOCS.md` | Endpoint reference for `/v1/hierarchy-transfers/*` — request/response shapes, the inheritance rule, error table |
| `HIERARCHY_SCOPED_OPERATIONS_DOCS.md` | `/scoped/promote` and `/scoped/realign` for a unit's administrator |
| `HIERARCHY_AUTHORITY.md` | The authority table and how the three "make a parish an area" operations differ |
| `APPROVALS_AND_TRANSFERS_DOCS.md` | The approvals API — all four request types, deciding, listing, codes |
| `PRINCIPAL_OFFICERS_DOCS.md`, `HQ_ASSIGNMENT_DOCS.md` | The two other surfaces national support now reaches |
| `CONFIGURATION.md` → *Role authority* | `ELEVATED_ROLES`, `HIERARCHY_RESTRUCTURE_ROLES`, `NON_ADMINISTRATIVE_ROLES` |
| `PARISH_HIERACHY_CONFLICT.md` | The live data faults these features were built around |
