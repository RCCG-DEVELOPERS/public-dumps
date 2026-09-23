# 2026-09-23 11:07 — Timestamps, the batch record, and delete metadata

**Branch:** `dev`

---

## 1. Timestamps reading as 1970 — cause and cure

A document with no `createdAt` sends nothing, and `new Date(null)` is
**1970-01-01T00:00:00.000Z**. A missing timestamp therefore does not *look*
missing — it looks like a real date from before the organisation existed. Nobody
questions a date, which is what made this an audit problem rather than a display
one.

**It was not configuration.** Every schema declares `timestamps: true`. Two
places write through `.collection` — the raw driver — where mongoose's
timestamps never fire:

- the transfer snapshots (`Hierarchytransfers/snapshot.ts`)
- the hierarchy-code backfill (`Hierarchycodes/service.ts`)

Both now stamp explicitly. The writes stay on `.collection` deliberately: they
are bulk inserts needing neither validation nor hydration.

### What can be recovered

| Field | Recoverable | How, and how honestly |
|---|---|---|
| `createdAt` | **Yes, exactly** | Every ObjectId embeds the second it was generated. Verified against rows carrying both — they agree. |
| `updatedAt` | **No** | Nothing records when a document last changed. `--updated-from-created` sets it equal to `createdAt`, which says *"not modified before this"* and nothing more. **Off by default.** |
| `deletedAt` | **No**, where never written | The `deletions` collection holds a real one for records that were *erased*. |

```bash
npx ts-node -r dotenv/config scripts/auditTimestamps.ts        # read-only
npx ts-node -r dotenv/config scripts/auditTimestamps.ts --fix  # recover createdAt
```

The script sweeps **every** collection the driver reports. It found
`hierarchyChangeSnapshots` immediately — 2,346 rows locally — which a grep over
`*/model.ts` had missed, because that schema lives beside its service rather
than in a model file. A document whose `_id` is not an ObjectId has nothing to
recover from and is **left alone** rather than given an invented date.

> Run the read-only pass against production before the fix. It reports per
> collection and writes nothing.

## 2. The batch audit record

`directoryAlignmentBatches` — one row per bulk alignment.

The per-unit jobs already say what happened to each unit, and are what makes one
unit individually rollback-able. What they could not answer without an
aggregation is the question somebody actually asks: **who reorganised the
directory, and how much did they move?**

```json
{
  "batchId": "ba-1790140000000", "level": "area", "mode": "atomic",
  "performedByUsername": "admin", "reason": "area restructure",
  "affectedRecords": 12, "membersAffected": 340,
  "succeeded": 12, "failed": 0, "unchanged": ["A099"],
  "status": "APPLIED", "rolledBack": false, "clean": true,
  "changes": [ { "unitCode": "A031", "toParentCode": "Z012",
                 "jobId": "652f…", "ok": true, "parishes": 28, "users": 96 } ]
}
```

A summary, not a second source of truth — the jobs stay authoritative for any
individual unit, and each `changes[].jobId` points at the one that can undo it.
Writing the summary can never fail the operation it describes.

A **dry run records nothing**: nothing happened to describe.

## 3. Area-level alignment is now tested

The bulk endpoint has always taken a `level` and the path is level-agnostic,
but every test passed `'parish'` — so moving an area between zones was a claim
rather than a fact. There is now a test that moves one and asserts the area keeps
its own code while its ancestry changes.

## 4. Delete and restore carry metadata

Every delete and restore response now carries the record it acted on, so a
confirmation can **name** it rather than only report that something happened.

Restores also carry what the record **was** — `wasDeletedAt`, `wasDeletedBy`,
and for a parish `disabledReason` and `reactivateAt` — so the interface can
describe the change, not just its outcome.

Nothing in these payloads is a secret: passwords, OTP secrets and refresh tokens
are projected out or were never in the snapshot.

Full reference: [DELETIONS_DOCS.md](DELETIONS_DOCS.md).

## 5. Documentation debt cleared

- **`DELETIONS_DOCS.md`** now exists as a living reference — the deletion
  endpoints previously lived only in dated changelog entries, which is exactly
  the drift that left the privileged-audit doc wrong about retention.
- The 2026-09-23 04:19 note no longer claims parish disable-by-approval is
  unbuilt; it links to the flow instead.
- Both indexed in the README.

New references go in `documentations/`. `documentations/v1/` holds the ones
that were moved out of the repository root and is not where new work goes.
