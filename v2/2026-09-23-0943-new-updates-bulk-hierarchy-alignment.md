# 2026-09-23 09:43 — Bulk hierarchy alignment

Phase 2 of the Parish Directory work. Phase 1 (sorting, aliases, display order)
shipped separately.

---

## Why

A transfer moves one unit. That is right for a move and wrong for a
restructure: an administrator rearranging forty parishes after a zone split had
to issue forty requests, and could not see the whole change before committing to
any of it.

## `PATCH /v1/parishDirectory/bulk-alignment`

**Guard:** elevated. **Dry run by default.**

```http
PATCH /v1/parishDirectory/bulk-alignment
Content-Type: application/json

{
  "level": "parish",
  "mode": "atomic",
  "dryRun": true,
  "reason": "Zone restructure, Q3",
  "changes": [
    { "unitCode": "211003", "toParentCode": "A031" },
    { "unitCode": "211004", "toParentCode": "A031" }
  ]
}
```

| Field | Default | Notes |
|---|---|---|
| `level` | `parish` | the level being MOVED — also `area`, `zone`, `province`… |
| `mode` | `atomic` | `atomic` or `per-record` |
| `dryRun` | **`true`** | send `false` to apply |
| `changes` | — | max **200**; each entry names the unit and its new parent |

`toParentCode` is the *parent level's* code — an area code when moving parishes,
a zone code when moving areas. The parent level is derived; you never name it.

### Dry run

```json
{
  "applied": false, "dryRun": true, "valid": true,
  "wouldChange": 2, "unchanged": ["211009"], "membersAffected": 2,
  "changes": [
    { "unitCode": "211003", "toParentCode": "A031", "members": 1,
      "inherits": { "areaCode": "A031", "areaName": "AREA 31",
                    "zoneCode": "Z012", "zoneName": "ZONE 12" } }
  ],
  "message": "Nothing was written. Send dryRun: false to apply."
}
```

`inherits` is the **whole ancestry**, not just the named parent — moving a parish
into an area brings that area's zone, province and everything above it.

A unit already where it is asked to go is reported as **`unchanged`**, not as an
error. It is nothing to do, and calling it a failure sends somebody hunting for a
problem that is not there.

### Validation — everything, together

**Nothing is written while any record is invalid**, in either mode. A payload
with a typo is a payload to fix, not one to half-apply.

Every problem comes back at once, because fixing twenty mistakes twenty
round-trips at a time is the failure mode this exists to remove:

```json
{
  "valid": false, "applied": false,
  "errors": [
    { "index": 0, "unitCode": "211099", "code": "UNKNOWN_UNIT",
      "message": "parish 211099 could not be resolved: No parish found with code 211099" },
    { "index": 1, "unitCode": "211003", "code": "UNKNOWN_DESTINATION",
      "message": "area A999 could not be resolved." },
    { "index": 2, "unitCode": "211003", "code": "DUPLICATE",
      "message": "211003 appears more than once in this payload." }
  ],
  "message": "3 problem(s) in this payload. Nothing was written."
}
```

Codes: `UNIT_REQUIRED`, `PARENT_REQUIRED`, `DUPLICATE`, `UNKNOWN_UNIT`,
`UNKNOWN_DESTINATION`, `NOT_PERMITTED`, `UNKNOWN_LEVEL`, `NO_PARENT_LEVEL`.

Authority is judged on **resolved chains, never on request fields** — the same
`canTransfer` a single transfer uses, so a bulk request cannot assert a province
the caller is not in.

Each distinct destination is resolved **once**: 200 records naming a dozen areas
is a dozen aggregations, not 200.

### Applying

Per unit, in the order a single transfer already uses: the directory rows, then
the members' own hierarchy columns. A parish's own `parishCode` and name are
structurally unwritable — `buildInheritedPatch` only ever emits ancestor fields.

```json
{
  "applied": true, "batchId": "ba-1790140000000",
  "changed": 2, "failed": 0, "unchanged": [],
  "results": [
    { "ok": true, "jobId": "652f…", "unitCode": "211003",
      "toParentCode": "A031", "parishes": 1, "users": 4, "snapshotComplete": true }
  ]
}
```

### When it goes wrong — compensation, not a transaction

This codebase uses mongoose sessions **nowhere**, and the dev and CI databases
are standalone Mongo where transactions do not exist. `Hierarchytransfers/model.ts`
rejects them deliberately, in favour of visibility and resumability.

So "all or nothing" is achieved the way the rest of this module achieves it: the
before-state of every row is captured, and on failure the applied changes are
restored from those snapshots, **in reverse order**.

```json
{
  "applied": false, "rolledBack": true, "clean": true,
  "failedAt": "211004", "failure": "…",
  "compensation": [ { "unitCode": "211003", "parishesRestored": 1, "parishesSkipped": 0,
                      "usersRestored": 4, "usersSkipped": 0 } ],
  "message": "One change failed, so every change in this batch was undone."
}
```

> **Read `clean`.** A row somebody else changed in between is **left alone** and
> counted as skipped — overwriting it would turn one failed batch into two
> problems. When anything is skipped, `clean` is `false` and the message says the
> directory is **not** back where it started. Compensation is not atomicity, and
> the response says so rather than reporting a tidy success.

In `per-record` mode nothing is undone: each result is reported and what
succeeded stays.

### Afterwards

Each changed unit writes its **own** job sharing a `batchId`. That keeps the set
readable as one operation, while
`POST /v1/hierarchy-transfers/jobs/{id}/rollback` still undoes any single one of
them later — a per-record undo the batch itself cannot offer once it has
returned.

---

## Stranded offices and headquarters

Both now behave exactly as a single transfer does, calling the same helpers
rather than a second copy that could drift.

**An office the move strands is ended.** A unit moving out from under an officer
used to leave that office standing over somewhere it no longer belonged. Each
job records what it cost in `officesVacated`, so the audit says who lost a post
because of the move and not merely that rows changed. If ending an office fails,
the move still stands and the failure is recorded on the job — a correct
directory change is not undone by a tidying step.

**A move that would demote a headquarters is refused.**

```json
{
  "index": 0, "unitCode": "211003", "code": "HQ_DEMOTION",
  "message": "211003 heads 1 unit(s) it would be moving out of. Moving it demotes them. Send acknowledgeDemotion: true if that is what you mean."
}
```

Send `"acknowledgeDemotion": true` at the top level of the request when that is
genuinely the intent.

## Known gaps

- **200 records is a lot of sequential work** — each resolves a chain, snapshots
  its rows, rewrites its members and checks its offices. The dry run reports
  `membersAffected` first.
