# Directory Integrity — The Guide

How to find what is wrong with the parish directory, and how to fix it without
making it worse. The endpoint reference is
[DIRECTORY_INTEGRITY_DOCS.md](./DIRECTORY_INTEGRITY_DOCS.md).

## Who this is for

- **Scanning and reading reports:** super-admin, nat-support and nat-ict.
- **Repairs:** super-admin only, and only once
  `DIRECTORY_INTEGRITY_REPAIR_ENABLED=true` is set on the service.

## The workflow

```
scan  →  review  →  preview  →  execute  →  (rollback)
```

### 1. Scan

```http
POST /v1/hierarchy-transfers/integrity/scan
```

This takes seconds on a copy of production. It loads the parish codes once,
the same way the nightly hierarchy sync does. Keep the `scanId`.

### 2. Review

```http
GET /v1/hierarchy-transfers/integrity/reports/<scanId>?severity=HIGH
GET /v1/hierarchy-transfers/integrity/reports/<scanId>?type=CODE_UNDER_MULTIPLE_PARENTS
```

Work from HIGH down. For each finding, read `description` and
`suggestedRepair`:

- When `suggestedRepair.action` is set, the finding can be repaired here.
- When `suggestedRepair.action` is `null`, a person has to decide.
  `suggestedRepair.endpoint` and `body` point at the tool, for example a ready
  `/admin/realign` dry run for a split unit, or the code backfill for
  unregistered codes.

Some things are never repaired automatically. An orphan user names a parish
that does not exist, so nobody knows where they belong. A duplicate parish code
means two churches are wearing one number. A missing code cannot be invented.

### 3. Preview

Choose the findings that take the same action and preview them together:

```http
POST /v1/hierarchy-transfers/integrity/repair/preview
{
  "scanId": "<scanId>",
  "findingIds": ["F0001"],
  "action": "RECODE_SPLIT_UNIT",
  "options": { "keepParent": { "AR8000000211": "ZN200001" } },
  "reason": "AR8000000211 is two areas; the ZN200017 side gets its own code"
}
```

Read the plan. It lists the exact rows, users and grants, with before and after
values. If `executable` is false, read `blockers`: a blocker might say a finding
has been fixed since the scan, that a parish now disagrees with itself (realign
it first), or that an option is missing.

### 4. Execute

```http
POST /v1/hierarchy-transfers/integrity/repair/execute
{ "previewId": "<previewId>", "idempotencyKey": "recode-ar211-2026-11-08" }
```

- Pick your own key and keep it. If the response is lost, send the **same**
  request again: you get the first result back (`replayed: true`) and nothing
  runs twice.
- A **409 `PLAN_DRIFTED`** means the directory changed after you previewed, so
  what would run is not what you saw. Nothing was written. Preview again.
- If an execute fails part-way, send the same body again (same key) to finish.
  Every step only touches rows that do not already carry the target value.

### 5. Roll back, if you have to

Every change is a job:

```http
GET  /v1/hierarchy-transfers/jobs?batchId=<previewId>
POST /v1/hierarchy-transfers/jobs/<jobId>/rollback   { "dryRun": true }
POST /v1/hierarchy-transfers/jobs/<jobId>/rollback   { "reason": "wrong side re-coded" }
```

A rollback puts each row back to its own earlier values. It leaves alone any
row that has changed again since, and reports it. A minted code is not
released: once a code has been issued, it is never issued again.

## Choosing which side keeps a code (RECODE_SPLIT_UNIT)

An area or zone code under two parents is two units sharing a name. A code is
never "merged" by a repair. One side keeps the code, and every other side gets
a newly minted, registered code of the same shape (`AR…` / `ZN…`).

- By default, the side with the most rows keeps the code.
- Use `options.keepParent` when the smaller side is the original, for example
  when the larger side came from a mistaken import.
- `officesToReview` lists principal offices held at the old code. They are not
  moved. After the re-code, one of them may belong to the other unit, so review
  each one through `/v1/principal-officers`.

Province and higher codes are meaningful rather than random, so this service
does not mint them. For those, the finding points at `/admin/realign` instead.

## Settling an HQ conflict (RESOLVE_HQ_CONFLICT)

Name the parish that keeps the flag:

```json
{ "action": "RESOLVE_HQ_CONFLICT", "findingIds": ["F0012"], "options": { "keep": { "ZN200017": "211343" } } }
```

The repair runs `POST /v1/hq-assignments/resolve-conflict` for you and records a
snapshot first, so the rollback can restore the cleared flags.

## Before switching repairs on in production

1. Run `scripts/createHierarchyIntegrityIndexes.ts --dry-run`, then for real,
   against the target database with `MONGODB_URI` set explicitly.
2. Run a scan on a local copy of production and read it end to end.
3. Set `DIRECTORY_INTEGRITY_REPAIR_ENABLED=true` in the ECS task definition.
4. Start with `REFRESH_STALE_COPIES` and `RESYNC_USERS`. They touch the fewest
   rows and are the easiest to check.
5. When a scan shows no `CODE_UNDER_MULTIPLE_PARENTS` findings, set
   `HIERARCHY_UNIQUE_PARENT_ENFORCED=true`. From then on, parish create, edit,
   restore and CSV import refuse any row that would put an area under a second
   zone or a zone under a second province, so the splits you repaired cannot
   come back.

## Frontend notes

A suggested admin screen:

- a **Reports** list, from `GET /integrity/reports`
- a **Report** view grouped by severity and type, with a checkbox per finding
  that carries a `suggestedRepair.action`
- **Preview** shows `plan.items` as before/after tables, and **Execute** stays
  disabled while `blockers` is non-empty

Generate the idempotency key in the browser when the Preview screen opens, and
reuse it on retries.

Handle **403 `FEATURE_DISABLED`** by hiding the repair buttons.
