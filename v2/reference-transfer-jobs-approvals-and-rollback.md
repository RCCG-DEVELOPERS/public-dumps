# Reference — transfer jobs, approvals, and rolling back

Not a dated update: a standing reference for how a hierarchy change is recorded,
approved, and undone. Kept here because it is what people reach for when a
transfer has gone wrong.

- [The shape of it](#the-shape-of-it)
- [Every change writes a job](#every-change-writes-a-job)
- [Reading jobs](#reading-jobs)
- [Rolling back](#rolling-back)
- [When a change needs approval instead](#when-a-change-needs-approval-instead)
- [Batching many changes](#batching-many-changes)
- [What cannot be undone](#what-cannot-be-undone)

---

## The shape of it

Three things happen when a unit moves, and they are separate on purpose:

| | Where | Why separate |
|---|---|---|
| **The change** | `POST /transfer`, `/promote`, `/admin/realign` | does the work |
| **The job** | `hierarchyChangeJobs` | records the before-state, row by row |
| **The approval** | `approvals` | when the change crosses a boundary you do not hold |

A change you are allowed to make happens immediately and writes a job. A change
you are not allowed to make becomes a request somebody else decides.

---

## Every change writes a job

Before a transfer writes anything, it **captures the before-state of every row
it is about to touch** — parishes, users, and the HQ flag rows — into the job's
snapshot. That snapshot is what makes a rollback possible, and it is taken
first, so a change that fails half way is still fully described.

The job carries:

| Field | Meaning |
|---|---|
| `operation` | `TRANSFER` \| `PROMOTE` \| `REALIGN` \| `ROLLBACK` |
| `status` | `PENDING` \| `RUNNING` \| `COMPLETED` \| `FAILED` |
| `steps[]` | per collection: `{ name, matched, modified, done, error }` |
| `snapshot`, `snapshotComplete` | the before-state, and whether it is whole |
| `authorisedVia` | `province LA47`, `super-admin`, or `elevated:nat-support` |
| `rolledBackAt`, `rolledBackBy`, `rollbackJobId` | filled in if it was undone |
| `rollbackOf` | on the rollback job, pointing back at the original |

`snapshotComplete: false` means the capture was truncated — **that job is not
safely revertible**, and it says so rather than letting you find out during the
rollback.

---

## Reading jobs

```http
GET /v1/hierarchy-transfers/jobs?status=COMPLETED&pageNo=1
Authorization: Bearer <token>
```

**Guard:** elevated.

```http
GET /v1/hierarchy-transfers/jobs/652f1a0000000000000000cc
```

One job in full, including `steps[]` and the snapshot's extent. This is where you
look when somebody says "a lot of parishes moved last night" — `steps[]` gives
the row counts per collection, and `authorisedVia` says on whose authority.

Related: `GET /v1/hierarchy-transfers/integrity` reports duplicate parish codes,
HQ conflicts and split units across the directory — the data faults that make
transfers behave oddly in the first place.

---

## Rolling back

```http
POST /v1/hierarchy-transfers/jobs/652f1a0000000000000000cc/rollback
Content-Type: application/json

{ "dryRun": true, "reason": "moved the wrong area" }
```

**Guard:** elevated.

**Always send `dryRun: true` first.** It reports how many rows would be restored
and — the important number — **how many have changed since and would be
skipped**.

Rows altered by a later change are **left alone** unless `force` is set. That is
the rule that matters: rolling back blindly would overwrite whatever happened
after the mistake, turning one bad change into two.

A rollback is itself a job, with `operation: "ROLLBACK"` and `rollbackOf`
pointing at the original.

### Refusals

| Code | Meaning |
|---|---|
| `CANNOT_ROLL_BACK_A_ROLLBACK` | undo the undo by re-running the original change, not by chaining |
| *(already rolled back)* | the job names when it was undone and by whom |

---

## When a change needs approval instead

`POST /transfer` judges your standing against **both ends** of the move. Inside
your own unit it happens. Across a boundary you do not hold, you get:

```json
{
  "status": 403,
  "code": "TRANSFER_NEEDS_APPROVAL",
  "approvalHint": { "useEndpoint": "/v1/approvals/unit-transfer", "body": { "...": "..." } }
}
```

Raise it as a request:

```http
POST /v1/approvals/unit-transfer
Content-Type: application/json

{ "level": "area", "unitCode": "AR0000008798",
  "toParentCode": "ZN0000008849", "reason": "reorganisation" }
```

**The receiving province decides.** The province giving the unit away already
spoke by raising the request; letting it approve as well would make a
cross-province move one administrator's decision — which is exactly what
`/transfer` refused.

```http
POST /v1/approvals/652f.../approve
POST /v1/approvals/652f.../reject
POST /v1/approvals/652f.../cancel
```

On approval the move is executed **through the same service an administrator
uses directly**, with no standing passed — the approval *is* the authority. The
request's snapshot is deliberately **not** trusted: the unit is re-planned and
both provinces re-checked, because the world may have moved since it was raised.
If execution fails the request lands `FAILED` with the error, never `APPROVED` —
an approval that did not take effect is worse than a rejection, because it looks
done.

`TRANSFER_NOT_PERMITTED` with a `useInstead` hint means no approval route
exists for what you asked; check the hint is an endpoint you can actually use.

---

## Batching many changes

There is **no bulk transfer endpoint**, and that is deliberate — each move is
judged, snapshotted and reversible on its own.

To move many units:

1. `GET /preview` per unit to see what each would do, **without doing it**.
2. Run them one at a time, keeping each returned `jobId`.
3. If a batch goes wrong, roll back **job by job, newest first** — that order
   matters, because an earlier rollback would otherwise be the "later change"
   that makes the next one skip rows.

`GET /jobs?from=&to=` gives you the list to work backwards through.

The one genuinely batched operation is the **user hierarchy sweep**
([HIERARCHY_SYNC_DOCS.md](v1/HIERARCHY_SYNC_DOCS.md)), which repairs users rather
than moving units — and it has its own circuit breakers precisely because it is
the one thing here that touches thousands of records at once.

---

## What cannot be undone

- **`?hard=true` on a parish delete.** The row is erased. The tombstone in
  `deletions` keeps the snapshot, but there is no restore from it.
- **A rollback whose job has `snapshotComplete: false`.** Partly described, so
  only partly reversible.
- **Anything after `force: true`.** Force overwrites rows that changed after the
  original, which is the protection you are turning off. Read the dry run first.
