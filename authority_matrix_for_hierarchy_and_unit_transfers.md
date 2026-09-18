# Authority matrix — hierarchy, unit transfers and principal officers

Who may perform each operation, and who must approve it. This is the technical
copy, with endpoints and error codes. The administrator's copy, without
endpoints, is `authority_matrix_for_hierarchy_and_unit_transfers_users.md`.

Describes the system in production as of 18 September 2026.

## Key

| Term | Meaning |
|---|---|
| **Unbounded** | `super-admin`, or an elevated role — `nat-support` by default (`ELEVATED_ROLES`). Passes every check below. |
| **Administrator** | A role in `HIERARCHY_RESTRUCTURE_ROLES`: `prov-admin`, `reg-admin`, `sub-cont-admin`, `cont-admin`. Assistants and `picp` are never administrators. |
| **Standing** | The units a caller acts in: one code per level, taken from the roles they hold and the matching code on their own profile (`province`, `region`, …). Non-administrative roles such as `training-manager` give no standing. |
| **Contains** | One of the caller's units contains the target, from at or above its level, judged on the target's ancestry as resolved from the database — never from the request. |
| **Inside** | Both ends of a move sit within one unit the caller holds standing in. |

---

## 1. Unit moves — parish, area, zone, province

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Move a unit **within** their own unit | `POST /v1/hierarchy-transfers/transfer` | Anyone with standing over a unit containing **both** ends — `picp`, `prov-admin`, `reg-admin` … — or unbounded | None |
| **Pull** a unit in from another province | `POST /v1/hierarchy-transfers/transfer` | **Administrator** of the receiving unit — `prov-admin` of the destination province, `reg-admin` of a region containing it — or unbounded | None. The receiving administrator's action *is* the approval. A pending `UNIT_TRANSFER` aimed at their province is settled `APPROVED` by it; one aimed elsewhere refuses `409 REQUEST_PENDING_ELSEWHERE`. |
| Same pull, by a non-administrator (`picp`, `prov-asst-admin`) | `POST /v1/hierarchy-transfers/transfer` | Refused `403 TRANSFER_NOT_PERMITTED`; the message names the administrator who can | — |
| **Push** a unit out to another province | `POST /v1/approvals/unit-transfer` (raise) | `prov-admin` of the **source** province, anyone whose unit contains it, or unbounded | `prov-admin` of the **receiving** province; `reg-admin` whose region contains the destination; or unbounded. Never the raiser. `403 TRANSFER_NEEDS_APPROVAL` if attempted directly. |
| Move any unit, any level, any distance | `POST /v1/hierarchy-transfers/admin/move` | Unbounded only | None |
| A move that would **strand a headquarters** | `/transfer`, `/admin/move`, `/admin/realign` with `acknowledgeDemotion: true` | Unbounded only. Everyone else is refused and told to vacate the HQ first via `POST /v1/hq-assignments/vacate` | None |
| Realign a split unit (repair) — see [section 9](#9-realign--the-repair-for-a-split-unit) | `POST /v1/hierarchy-transfers/admin/realign` | Unbounded only | None |
| Realign within own scope — see [section 9](#9-realign--the-repair-for-a-split-unit) | `POST /v1/hierarchy-transfers/scoped/realign` | Administrator with standing over **both ends of every variant**, at a level above the unit | None |
| Preview any of the above | `GET /v1/hierarchy-transfers/preview` | Any signed-in user; reports `permissionCode` and `permittedVia` for the caller | — |
| Roll back a completed job | `POST /v1/hierarchy-transfers/jobs/:id/rollback` | Unbounded only | None |
| List / inspect jobs | `GET /v1/hierarchy-transfers/jobs`, `/jobs/:id` | Unbounded only | — |
| Integrity report (split units, orphans) | `GET /v1/hierarchy-transfers/integrity` | Unbounded, or a scoped API key | — |

## 2. Promotion of units — parish → area, area → zone, …

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Promote any unit, with `absorbCodes` | `POST /v1/hierarchy-transfers/promote` | Unbounded only | None |
| Promote within own scope | `POST /v1/hierarchy-transfers/scoped/promote` | Administrator with standing over the unit. `absorbCodes` is refused for a bounded caller | None |
| Promote a unit whose HQ would be demoted | either, with `acknowledgeDemotion` | Unbounded only | None |

## 3. Principal officers — appointing and ending

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Appoint an officer at their **own** unit (`prov-admin` appoints `picp` of LA47) | `POST /v1/principal-officers` | Any officer with standing at that exact unit, or unbounded | None |
| Appoint an officer in a unit they **contain** (area-admin appoints `pic-parish` in AR1; `prov-admin` anywhere in LA47) | `POST /v1/principal-officers` | Administrator of the containing unit, or unbounded | None |
| Appoint upward (area-admin appoints a province officer) | `POST /v1/principal-officers` | Refused `403 NO_STANDING_AT_UNIT` | Raise `officer-promotion` instead |
| Appoint into a unit that is not the user's own | `POST /v1/principal-officers` with `scopeCode` | Same as above, **and** the target user must hold an active secondary grant of that role there | None |
| Appoint a non-administrative office (`training-manager`) | `POST /v1/principal-officers` | Same rules; the holder gains **no** administrative authority | None |
| End an appointment | `POST /v1/principal-officers/:id/end` | Unbounded only | None |
| Transfer an appointment to another person | `POST /v1/principal-officers/:id/transfer` | Unbounded only | None |

## 4. Principal officers — through approvals

| Action | Endpoint | Who can raise | Who approves / rejects |
|---|---|---|---|
| Replace the holder of an office | `POST /v1/approvals/officer-transfer` | Any authenticated user | Administrator whose unit **contains** the office, at or above its level, or unbounded. Never the raiser, except a `super-admin`. |
| Move a person into a different office | `POST /v1/approvals/officer-promotion` | Any authenticated user | Same as above |
| Cancel either | `POST /v1/approvals/:id/cancel` | The raiser, or unbounded | — |

## 5. Principal officers — register hygiene

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Roles held with no appointment / appointments with no role | `GET /v1/principal-officers/register-audit` | Unbounded | — |
| Strip an office role from users who hold no appointment | `POST /v1/principal-officers/admin/strip-roles` | Unbounded. Refuses `ACTIVE_APPOINTMENT` when the holder is actually appointed | None |
| Two people holding one office | `GET /v1/principal-officers/conflicts` | Unbounded | — |
| Offices with no holder | `GET /v1/principal-officers/vacancies` | Unbounded | — |
| Roster of offices and holders for the caller's own units | `GET /v1/principal-officers/roster` | Any signed-in officer; sees only units they hold standing at. Each row carries `canAppoint` for the caller | — |
| Full list, single record, one office, config | `GET /v1/principal-officers`, `/:id`, `/office`, `/config` | Unbounded only | — |

## 6. Pastor in charge of a parish (the PIC record)

Distinct from the principal-officer register: `parishPicHolders` says who
actually leads a parish. Holding the `pic-parish` role does not.

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Appoint the pastor in charge of a parish | `POST /v1/parish-pics` | Administrator of any unit containing the parish — area, zone, province, region, sub-continent, continent — or unbounded. Judged on the parish's own ancestry | None |
| The parish's own pastor (`pic-parish`) appointing their successor | `POST /v1/parish-pics` | Refused `403 NO_STANDING_OVER_PARISH` — a parish cannot appoint its own head | — |
| End a PIC appointment | `POST /v1/parish-pics/:id/end` | Same as appointing | None |
| Who leads this parish / list appointments | `GET /v1/parish-pics/parish/:parishCode`, `GET /v1/parish-pics` | Any signed-in user | — |

## 7. People and headquarters

| Action | Endpoint | Who can do it | Approver needed |
|---|---|---|---|
| Move a user to another parish | `POST /v1/approvals/user-transfer` (raise) | Any authenticated user | `prov-admin` of the **source or destination** province, or unbounded. Never the raiser. |
| Assign / vacate / resolve a headquarters | `POST /v1/hq-assignments/assign`, `/vacate`, `/resolve-conflict` | Standing over the unit, or unbounded | None |
| Assign roles to a user | `PATCH /v1/users/:id/roles` | Gated by `ROLE_ASSIGNMENT_ENFORCEMENT`: the caller may grant only roles listed by `GET /v1/users/me/allowed-roles` | None |

## 8. Approval lifecycle

| Action | Endpoint | Who |
|---|---|---|
| Approve / reject | `POST /v1/approvals/:id/approve`, `/reject` | Whoever the type's rule above names. Nobody decides their own request except a `super-admin`. |
| Cancel | `POST /v1/approvals/:id/cancel` | The raiser, or unbounded |
| View | `GET /v1/approvals`, `/:id` | Requests in the caller's units; unbounded sees all |

---

## 9. Realign — the repair for a split unit

### What a split unit is

A unit's ancestry is not stored on the unit. It is stored on every parish row
that belongs to it, and on every user under it. A unit is therefore *split* when
its own members disagree about who its parents are — half the parishes of
province LA47 naming region R07, the other half naming R12.

The usual cause is a headquarters parish being transferred away. A parish is a
leaf, so moving it rewrites that one row and nothing else. If that parish
happened to be the headquarters of a province, the province it headed is now
named by two different regions, and nothing else was touched.

A split is both the damage and the lock on the door. Once members disagree, the
chain resolver refuses the unit outright, so every transfer, preview and
promotion touching it returns `409 INCONSISTENT_UNIT`. Realign is the only
operation that can look at a split unit, because it is the only one that exists
to end the disagreement.

### What a realign does

It forces **every** member row and **every** user carrying the unit's code onto
one ancestry: the chain of the parent you name. Parishes are written first,
users second, and the job records what each step matched and modified.

| It does | It does not |
|---|---|
| Rewrite the ancestor codes and names on every parish row of the unit | Move the unit anywhere. The unit stays exactly where it is |
| Rewrite the same fields on every user under the unit | Change the unit's own code or name — refused as `IDENTITY_VIOLATION` |
| Include **department** rows, which a transfer skips, or the unit stays split | End principal offices. A transfer vacates offices at units a moved unit has left; a realign leaves them standing and reports them as `officesAtRisk` |
| Capture a per-row snapshot first, so the job can be rolled back | Guess. Where the members disagree, you choose which side wins |

### Which level goes under which

`toLevel` must be the **immediate** parent of `level`. Everything above it is
inherited from that parent, never named directly.

| Realign this | Under this | Typical trigger |
|---|---|---|
| `parish` | `area` | Duplicate rows for one parish code disagreeing with each other |
| `area` | `zone` | The area's parishes name two different zones or provinces |
| `zone` | `province` | The zone's parishes name two different provinces |
| `province` | `region` | The province headquarters was moved to another region and the rest of the province stayed behind — the common case |
| `region` | `sub-continent` | A province was moved between regions and left name drift behind |
| `sub-continent` | `continent` | Rare; usually name drift only |
| `continent` | — | Refused `NO_PARENT_LEVEL`. A continent has no parent |

Naming the wrong level is refused as `DESTINATION_NOT_PARENT_LEVEL`, and the
error carries `resendWith` holding the level you should have sent.

### Who can realign what

| Endpoint | Caller | Bound |
|---|---|---|
| `POST /v1/hierarchy-transfers/admin/realign` | Unbounded only | None. Any unit, any level, however far the split reaches |
| `POST /v1/hierarchy-transfers/scoped/realign` | Administrator | Standing over **both ends of every variant**, at a level strictly above the unit |

The scoped bound is stricter than it first looks. A split unit is by definition
partly somewhere else, and **every** variant of the unit is judged against
**every** variant of the destination. The variant lying outside your unit is
exactly the one you must not be able to pull back in alone — that is the
cross-boundary decision `/transfer` reserves. So:

| Realign this | Scoped caller needs standing at | Example |
|---|---|---|
| `parish` | area or above | An area admin unifies duplicate rows of a parish in their area |
| `area` | zone or above | A province admin repairs an area whose parishes name two zones **in their province** |
| `zone` | province or above | A province admin repairs a zone that drifted within their province |
| `province` | region or above | A region admin repairs a province split across two of **their own** regions |
| `region` | sub-continent or above | Rare, and usually unbounded work |

A province admin whose province has drifted into **another** region cannot
repair it: one side is outside their unit, and they are refused `403
TRANSFER_NOT_PERMITTED` with `/admin/realign` named as the route that can.

### When to use it

1. **After a headquarters parish is transferred away.** The unit it headed now
   names two parents. This is the case the endpoint was built for.
2. **When a transfer, preview or promotion returns `409 INCONSISTENT_UNIT`.**
   The unit is split and nothing else will touch it until it is repaired.
3. **Name drift only.** Every variant agrees on the codes and differs only in
   the spelling of a name — `REGION 71` against `Region 71`. Harmless to a move,
   which uses the dominant variant, and a single call to clear. The integrity
   report counts these separately as `nameOnlyCount`.
4. **After a cascade stopped part-way.** A failed job leaves some rows written
   and others not; realign finishes the job onto one answer.
5. **Legacy or imported data** where parishes were edited one at a time.

Do **not** use it to move a unit. A realign corrects the record of a move that
already happened. If the unit genuinely belongs somewhere else, use `/transfer`.
When every variant already matches the destination, realign refuses with
`ALREADY_ALIGNED` and points you at `/transfer` instead.

### How to find split units

`GET /v1/hierarchy-transfers/integrity` returns `splitUnits`, grouped by level.
Each level carries the true `count`, the `nameOnlyCount`, and a capped list of
`codes`. Every entry carries a ready-made `suggestedRealign` body, aimed at the
parent that most of the unit's members already sit under, with `dryRun: true`
already set.

### The safe sequence

Always dry run first. Which side of the split wins is **not recoverable from the
data afterwards**.

```
1. POST /admin/realign  { level, unitCode, toLevel, toCode, dryRun: true }
```

Read from the response:

| Field | What to check |
|---|---|
| `variants` | Every ancestry the unit's members currently claim, largest group first, with `matchesDestination` on each |
| `destinationVariants` | The destination may itself be split. Index `0` is the dominant one and the default; pick another with `destinationVariant` |
| `cascade` | How many parish rows and users would be written, and how many of those are departments |
| `hqDemotions` | Headquarters that this would strand. Refused unless `acknowledgeDemotion` is sent, which is unbounded-only |
| `officesAtRisk` | Principal officers held at a unit this one will no longer sit under. They are **left standing** — review each and end or transfer it through `/v1/principal-officers` if that is right |
| `warnings` | Raised when the destination is split and you have not echoed a dry run |
| `applyWith` | Paste this straight into the apply call |

```
2. POST /admin/realign  { ...same, dryRun: false, ...applyWith }
```

`applyWith` carries `destinationVariant`, `expectedDestinationChain` and
`expectedMembers`. The chain echo makes the apply refuse with `409
DESTINATION_CHAIN_CHANGED` if the data moved between the dry run and the apply,
so you can never silently write a different answer from the one you approved.
Above the confirmation threshold `expectedMembers` is required, not optional.

### Error codes

| Code | Meaning |
|---|---|
| `NO_PARENT_LEVEL` | A continent cannot be realigned |
| `DESTINATION_NOT_PARENT_LEVEL` | `toLevel` is not the immediate parent; `resendWith` names the right one |
| `EMPTY_UNIT` | No rows carry that unit code |
| `ALREADY_ALIGNED` | Every variant already sits under the destination. Nothing to repair |
| `IDENTITY_VIOLATION` | The patch would rewrite the unit's own code or name |
| `DESTINATION_VARIANT_OUT_OF_RANGE` | `destinationVariant` is past the end of the list |
| `DESTINATION_CHAIN_CHANGED` | The data changed between dry run and apply. Re-run the dry run |
| `TRANSFER_NOT_PERMITTED` | Scoped caller; one side of the split, or of the destination, lies outside their unit |

### Afterwards

The job row records the collapse: `previousChain` holds a **list** of the
variants that were merged, not a single chain, which is what distinguishes a
realign row from a transfer row when reading the history back. A per-row
snapshot is captured before any write, so `POST /jobs/:id/rollback` can put each
variant back. Re-run the integrity report and confirm the unit no longer appears
under `splitUnits`.

---

## Recent changes, all live

- A region administrator may approve a unit transfer between two provinces in
  their own region.
- The receiving province's administrator may pull a unit in through `/transfer`
  alone, without the giving province raising a request.
- An administrator may appoint principal officers in the units they contain.

## Open question

Ending and transferring a principal-officer appointment remain unbounded-only
while appointing is delegated. An area-admin can appoint a pastor to the
register but cannot end that appointment. Bringing `/:id/end` and
`/:id/transfer` under the same containment rule is a one-line decision.
