# User hierarchy sync

Bringing **users** back into step with `parishDirectory` when their
`area`/`zone`/`province`/`region`/`subContinent`/`continent` codes have drifted.

- [What this is, and what it is not](#what-this-is-and-what-it-is-not)
- [The six buckets](#the-six-buckets)
- [Two rules that keep it safe](#two-rules-that-keep-it-safe)
- [The circuit breakers](#the-circuit-breakers)
- [Running it — the procedure](#running-it--the-procedure)
- [GET /v1/hierarchy-sync/runs](#get-v1hierarchy-syncruns)
- [GET /v1/hierarchy-sync/runs/:id](#get-v1hierarchy-syncrunsid)
- [GET /v1/hierarchy-sync/orphans](#get-v1hierarchy-syncorphans)
- [POST /v1/hierarchy-sync/run](#post-v1hierarchy-syncrun)
- [The run record, field by field](#the-run-record-field-by-field)
- [The script](#the-script)
- [The switch-role refresh](#the-switch-role-refresh)
- [Configuration](#configuration)
- [What is logged](#what-is-logged)
- [Error codes](#error-codes)
- [Frontend guidance](#frontend-guidance)

---

## What this is, and what it is not

There are two different things in this codebase called "hierarchy repair", and
they fix opposite halves of the same problem.

| | Repairs | Endpoint | Docs |
|---|---|---|---|
| **Realign** | a **unit** whose own directory rows disagree about their ancestors | `POST /v1/hierarchy-transfers/admin/realign` or `/scoped/realign` | [HIERARCHY_MOVES_GUIDE.md](HIERARCHY_MOVES_GUIDE.md), [HIERARCHY_SCOPED_OPERATIONS_DOCS.md](HIERARCHY_SCOPED_OPERATIONS_DOCS.md) |
| **This** | **users** whose codes disagree with the directory | `POST /v1/hierarchy-sync/run` | this document |

They hand off to each other. This sweep **reports** split parishes and refuses to
touch the people in them, because deciding which half of a split wins is a
decision with an owner. Realign is how that decision gets made.

### Why it is needed at all

A transfer already cascades a **unit** move onto its members —
`Hierarchytransfers` runs an `updateMany` keyed on the unit code. Three things
that never reaches:

1. A user whose **own `parish` changed** — they moved, the parish did not.
2. A parish row **edited directly** in the directory, outside a transfer.
3. A user whose `parish` code **exists nowhere** in the directory at all.

Those codes are not decoration. They drive geofencing, approvals routing, role
scope and every by-province report, so a stale one quietly sends a person's work
to the wrong place.

---

## The six buckets

Every user lands in exactly one. The counts are the report.

| Bucket | Meaning | What happens |
|---|---|---|
| `IN_SYNC` | codes match the directory | nothing |
| `STALE` | parish resolves, an ancestor disagrees | **repaired** in apply mode |
| `ORPHAN` | parish code is in no directory row | **never modified.** Reported |
| `DEPARTMENT` | the code resolves only to a `parishType: "DEPARTMENT"` row | **never modified.** Reported |
| `SPLIT` | rows for that code disagree about ancestors | **never modified.** Reported |
| `NO_PARISH` | no parish code at all | **never modified.** Counted |

Three of the six are never touched, and that is the point. For an orphan we do
not know where the person belongs; a guess would move a real human being into a
province they have nothing to do with. The fix for those is a correction to the
**directory**, not another run of this job.

Duplicate directory rows for one code that **agree** are not a split — the unit
is simply recorded twice, and its members are repaired normally.

---

## Two rules that keep it safe

### 1. An empty directory cell never blanks a user's code

If the directory row says nothing for a level, that field is **excluded from the
patch** rather than written as `""`, and the level is counted under
`counts.directoryGaps`.

Without this rule, the first time somebody saves a directory row through a form
that drops `zoneCode`, the next run wipes `zone` from every user in that parish.

### 2. One absence, four spellings

`undefined`, `null`, `""` and `"  "` all compare equal, on **both** sides.
Without that, a user holding `""` against a directory holding `""` reads as
stale forever and is rewritten on every run with the value it already has.

---

## The circuit breakers

Three, and each refuses to **write** rather than writing something wrong.

| Breaker | Trips when | Why |
|---|---|---|
| Share of population | more than `HIERARCHY_SYNC_MAX_MODIFY_PCT` (default 20) of scanned users would change | A run that wants to rewrite 11,000 of 55,000 people is not drift. It is a bad bulk edit upstream, and propagating it is the incident. |
| Absolute ceiling | more than `HIERARCHY_SYNC_MAX_MODIFY` would change (`0` = off) | A blunt cap for a first apply. |
| Directory size | fewer than `HIERARCHY_SYNC_MIN_PARISHES` (default 10,000) codes loaded | A half-loaded directory classifies most of the population orphan or stale. |

A tripped run finishes with `status: "ABORTED"`, `usersModified: 0`, and
`circuitBreaker.reason` in plain English. **Nothing is written.**

> **On small datasets the percentage breaker trips constantly** — 2 stale users
> out of 5 is 40%. That is correct behaviour, not a bug. On a staging database
> with a handful of users, either raise the percentage or use
> `force: true` knowingly.

Override with `force: true` (endpoint) or `--force` (script). The override is
recorded on the run.

---

## Running it — the procedure

Do not skip to `apply`.

1. **Deploy with the cron off.** `HIERARCHY_SYNC_ENABLED` unset means it never runs.
2. **Take a report.** `POST /v1/hierarchy-sync/run` with `{"mode":"report"}`, or
   run the script. Nothing is written in either case.
3. **Read three numbers**: `counts.STALE` (how much drift is real),
   `counts.ORPHAN` (a data-entry job for somebody), and `counts.SPLIT` (units
   needing realign first).
4. **Triage the orphans.** They will not improve on their own.
5. **Enable the cron in report mode** and let it run a few weeks. Stable numbers
   mean the classifier agrees with reality.
6. **Flip `HIERARCHY_SYNC_MODE=apply`.**

---

## GET /v1/hierarchy-sync/runs

Recent runs, newest first. The heavy `changes[]` array is omitted.

**Guard:** elevated (super-admin or `ELEVATED_ROLES`).

```http
GET /v1/hierarchy-sync/runs?pageNo=1&pageSize=20
Authorization: Bearer <token>
```

| Query | Default | Max | Notes |
|---|---|---|---|
| `pageNo` | 1 | — | 1-based |
| `pageSize` | 20 | 200 | run records are large |

**200**

```json
{
  "mode": "report",
  "count": 2,
  "runs": [
    {
      "id": "6ab2875bc650a7071e6669e2",
      "batchId": "hs-1790084955844",
      "mode": "report",
      "trigger": "cron",
      "status": "COMPLETED_WITH_FAULTS",
      "startedAt": "2026-09-22T13:49:15.844Z",
      "finishedAt": "2026-09-22T13:49:15.896Z",
      "durationMs": 52,
      "requestedBy": "system:cron",
      "directoryCodes": 52898,
      "scanned": 55014,
      "tuples": 24911,
      "counts": {
        "IN_SYNC": 54782, "STALE": 106, "ORPHAN": 94,
        "DEPARTMENT": 18, "SPLIT": 14, "NO_PARISH": 0,
        "directoryGaps": { "zone": 3 }
      },
      "usersStale": 106,
      "usersModified": 0,
      "changesComplete": true,
      "circuitBreaker": { "tripped": false, "threshold": 20, "wouldModify": 106 }
    }
  ]
}
```

`mode` at the top level is the **currently configured** mode, which is what the
next cron run will do — not the mode of any run in the list.

---

## GET /v1/hierarchy-sync/runs/:id

One run in full, including `changes[]`.

**Guard:** elevated.

```http
GET /v1/hierarchy-sync/runs/6ab2875bc650a7071e6669e2
Authorization: Bearer <token>
```

**200** — a real report run, lightly trimmed:

```json
{
  "id": "6ab2875bc650a7071e6669e2",
  "batchId": "hs-1790084955844",
  "mode": "report",
  "trigger": "api",
  "status": "COMPLETED_WITH_FAULTS",
  "startedAt": "2026-09-22T13:49:15.844Z",
  "finishedAt": "2026-09-22T13:49:15.896Z",
  "durationMs": 52,
  "requestedBy": "superadmin",
  "directoryCodes": 2,
  "scanned": 5,
  "tuples": 4,
  "counts": {
    "IN_SYNC": 1,
    "STALE": 2,
    "ORPHAN": 1,
    "DEPARTMENT": 0,
    "SPLIT": 1,
    "NO_PARISH": 0,
    "directoryGaps": {}
  },
  "usersStale": 2,
  "usersModified": 0,
  "changes": [
    {
      "parishCode": "211003",
      "from": {
        "continent": "AF", "subContinent": "SC01", "region": "R07",
        "province": "LA30", "zone": "Z04", "area": "A31"
      },
      "to": { "province": "LA47", "zone": "Z12" },
      "users": 2,
      "modified": 0
    }
  ],
  "changesComplete": true,
  "orphans": [{ "parishCode": "874112", "users": 1 }],
  "splits": [{ "parishCode": "990001", "variants": 2, "users": 1 }],
  "circuitBreaker": { "tripped": false, "threshold": 20, "wouldModify": 2 }
}
```

Read `changes[0]` as: two users on parish `211003` claim province `LA30` and
zone `Z04`; the directory says `LA47` and `Z12`; only those two fields would
change, because the other four already agree.

**404** — `{ "message": "No sync run with id <id>" }`

---

## GET /v1/hierarchy-sync/orphans

The users nobody can place, plus the split parishes — served **from the newest
completed run**, not recomputed. This is a list somebody works through over
days; it does not need to be live.

**Guard:** elevated.

```http
GET /v1/hierarchy-sync/orphans
Authorization: Bearer <token>
```

**200**

```json
{
  "ranAt": "2026-09-22T13:49:15.896Z",
  "runId": "6ab2875bc650a7071e6669e2",
  "mode": "report",
  "orphans": [{ "parishCode": "874112", "users": 1 }],
  "splits": [{ "parishCode": "990001", "variants": 2, "users": 1 }],
  "counts": { "IN_SYNC": 1, "STALE": 2, "ORPHAN": 1, "DEPARTMENT": 0, "SPLIT": 1, "NO_PARISH": 0 }
}
```

**200, before anything has run**

```json
{
  "orphans": [], "splits": [], "ranAt": null,
  "message": "No completed run yet. Trigger one, or wait for the cron."
}
```

**What to do with each list**

- `orphans` — the parish code is in no directory row. Either the code on those
  users is wrong, or the parish was deleted and should be restored. Fix the
  **data**; this job cannot.
- `splits` — repair with `POST /v1/hierarchy-transfers/admin/realign`, then the
  next sweep picks the members up normally.

Both are capped at `HIERARCHY_SYNC_REPORT_CAP` (default 500) per run.

---

## POST /v1/hierarchy-sync/run

Run it now.

**Guard:** super-admin — because the same route accepts `{"mode":"apply"}`.

```http
POST /v1/hierarchy-sync/run
Authorization: Bearer <superAdminToken>
Content-Type: application/json

{
  "mode": "report",
  "force": false
}
```

| Field | Required | Default | Notes |
|---|---|---|---|
| `mode` | no | `"report"` | `"report"` \| `"apply"`. Anything other than `"apply"` is treated as `report` |
| `force` | no | `false` | Override the circuit breakers. Recorded on the run |

**200** — the completed run record, identical in shape to
`GET /runs/:id` above. The request is **synchronous**: on a full-size directory
expect seconds, not milliseconds.

**200, breaker tripped** — note `status`, and that `changes` is empty because
nothing was even considered for writing:

```json
{
  "id": "6ab2875bc650a7071e6669e8",
  "batchId": "hs-1790084955914",
  "mode": "apply",
  "status": "ABORTED",
  "scanned": 5,
  "usersStale": 2,
  "usersModified": 0,
  "changes": [],
  "orphans": [{ "parishCode": "874112", "users": 1 }],
  "splits": [{ "parishCode": "990001", "variants": 2, "users": 1 }],
  "circuitBreaker": {
    "tripped": true,
    "threshold": 20,
    "wouldModify": 2,
    "reason": "This run would rewrite 2 of 5 users. At that scale the fault is far likelier to be in the directory than in the users, and propagating it would be the incident. Nothing was written."
  }
}
```

**An apply that did write** differs in exactly three places: `status` is
`COMPLETED` or `COMPLETED_WITH_FAULTS`, `usersModified` is non-zero, and each
`changes[].modified` carries the count that `updateMany` actually matched.

`modified` can be **lower** than `users` — that is not an error. A user the
sweep judged may have been moved by a transfer in between, in which case they no
longer match the pinned filter and are correctly skipped.

---

## The run record, field by field

| Field | Meaning |
|---|---|
| `batchId` | `hs-<epoch ms>`. Stamped on every user this run repaired, as `hierarchySyncedBy` |
| `mode` | `report` \| `apply` |
| `trigger` | `cron` \| `api` \| `script` |
| `status` | `RUNNING` \| `COMPLETED` \| `COMPLETED_WITH_FAULTS` \| `ABORTED` \| `FAILED` |
| `requestedBy` | username, or `system:cron` |
| `directoryCodes` | distinct parish codes loaded. A sudden drop means a directory problem |
| `scanned` | users considered |
| `tuples` | distinct hierarchies. ~25,000 for ~55,000 users — this is why the job is quick |
| `counts` | users per bucket, plus `directoryGaps` keyed by level |
| `usersStale` | users the run judged repairable |
| `usersModified` | users actually written. Always `0` in report mode |
| `changes[]` | per parish: `from`, `to`, `users`, `modified` |
| `changesComplete` | `false` when the cap truncated `changes[]` — **a truncated run does not fully describe itself** |
| `orphans[]`, `splits[]` | capped samples |
| `circuitBreaker` | `tripped`, `threshold`, `wouldModify`, and `reason` when tripped |

`COMPLETED_WITH_FAULTS` means it finished and wrote what it could, but found
splits or directory gaps that a human should look at. It is not a failure — it
exists so those do not sit unnoticed behind a green `COMPLETED`.

`changes[].from` is recorded as well as `to` deliberately: it is a complete
inverse of the run. There is no revert endpoint today, but without the old
values one could never be added.

---

## The script

Same service, same rules. This is how you read the numbers before the cron ever
applies anything.

```bash
# Report. Writes nothing.
npx ts-node -r dotenv/config scripts/syncUserHierarchy.ts

# Repair.
npx ts-node -r dotenv/config scripts/syncUserHierarchy.ts --apply

# Repair, overriding a tripped breaker — only when you are certain.
npx ts-node -r dotenv/config scripts/syncUserHierarchy.ts --apply --force
```

**Report is the default and `--apply` must be typed.** The script refuses to run
without an explicit `MONGODB_URI` and prints the host it connected to before
doing anything — this repo's `.env` points at production.

```
Target host: cluster0-shard-00-01.xxxxx.mongodb.net
Database:    remittanceAuthDirGlobal
Mode:        report — nothing will be written

Status:      COMPLETED_WITH_FAULTS
Parishes:    52898
Users seen:  55014 across 24911 distinct hierarchies
Stale:       106
Modified:    0

By bucket
  IN_SYNC       54782
  STALE         106
  ORPHAN        94
  DEPARTMENT    18
  SPLIT         14
  NO_PARISH     0

Levels the DIRECTORY could not speak for (left alone, never blanked):
  zone          3 user(s)

Orphaned parish codes — in no directory row. NOT modified:
  874112              1 user(s)
  Fixing these means correcting the directory, not running this again.

Split parishes — rows disagree about their ancestors. NOT modified:
  990001              2 variant(s), 1 user(s)
  Repair with POST /v1/hierarchy-transfers/admin/realign.
```

---

## The switch-role refresh

The sweep repairs the population weekly. This repairs **the one person standing
in front of you**, before their new token is minted.

On `POST /v1/users/switch-role`, for a **primary** role, the caller's parish is
resolved from the directory. If any of the six ancestors differ, the corrected
values go into the new token immediately and are persisted to the user document
fire-and-forget, stamped `hierarchySyncedBy: "switch-role"`.

- Skipped entirely for a **secondary** grant — `applyActiveRole` overwrites scope
  from the grant, so the user document is irrelevant.
- An **orphaned or department** parish never blocks the switch. It is logged with
  `status: "WARNING"` and the switch proceeds on the stored values.
- One indexed `findOne`; production carries `{ parishCode: 1 }` as
  `parish_code_unique`.
- Disable with `HIERARCHY_SYNC_ON_SWITCH=false`.

---

## Configuration

Full table in [CONFIGURATION.md](CONFIGURATION.md#user-hierarchy-sync). The four
that decide behaviour:

| Variable | Default | Effect |
|---|---|---|
| `HIERARCHY_SYNC_ENABLED` | *(off)* | The cron does not exist until this is `true` |
| `HIERARCHY_SYNC_MODE` | `report` | `apply` is the flip that lets it write |
| `HIERARCHY_SYNC_CRON` | `20 2 * * 0` | 02:20 Sunday |
| `HIERARCHY_SYNC_ON_SWITCH` | `true` | The per-user refresh above |

Register every new variable in `deployment-scripts/lib.sh` or it will not reach
ECS.

**Multiple tasks.** There is no distributed lock here, and none anywhere else in
this codebase — the documented strategy is that API tasks run
`CRON_ENABLED=false` while one dedicated task owns crons. Two copies running is
wasteful rather than harmful, because every repair pins the old values it was
judged against and so matches nothing the second time. That property is worth
preserving.

---

## What is logged

| Activity | Module | When |
|---|---|---|
| `HIERARCHY_SYNC_TRIGGERED` | `HIERARCHY_SYNC` | somebody calls `POST /run`, with the mode and whether the breaker was overridden |
| `SWITCH_ROLE` (status `WARNING`) | `USERS` | a switch-role refresh corrected somebody, naming the fields |
| `SWITCH_ROLE` (status `WARNING`) | `USERS` | a switch-role refresh could not resolve the parish |

There is deliberately **no activity row per repaired user**. A run touching
thousands would swamp a collection that already has a retention problem; the run
record is the durable trail, and `hierarchySyncedBy` on the user points back at it.

---

## Error codes

| Status | When |
|---|---|
| `400` | bad run id, or a malformed request |
| `403` | not elevated (reads), or not super-admin (`POST /run`) |
| `404` | no run with that id |

A tripped breaker is **not** an error — it is a `200` with
`status: "ABORTED"`. The run happened; it decided not to write. Clients must
check `status` and `circuitBreaker.tripped` rather than assuming `200` means
work was done.

---

## Frontend guidance

- **Never present `usersModified` from a report run as work done.** It is always
  `0`. Show `usersStale` as "would change".
- **Surface `status` prominently.** `ABORTED` and `COMPLETED_WITH_FAULTS` both
  return `200` and both mean somebody needs to look.
- **Show `circuitBreaker.reason` verbatim** when tripped. It is written to be read
  by a person.
- **Treat orphans as a worklist**, not an error display — each row is a parish
  code somebody has to correct in the directory.
- **Link splits to realign.** A split parish is actionable at
  `POST /v1/hierarchy-transfers/admin/realign`; see
  [HIERARCHY_MOVES_GUIDE.md](HIERARCHY_MOVES_GUIDE.md).
- **`POST /run` is synchronous and can take seconds.** Disable the button while
  it is in flight rather than letting somebody start three runs.
- **If `changesComplete` is `false`**, say so — the run is not fully described
  and the list on screen is partial.
