# Directory Integrity — API Reference

Stored integrity scans of the parish directory, and the repairs that act on
their findings. Part of the auth and directory service, mounted under
`/v1/hierarchy-transfers`.

The operator workflow is in [DIRECTORY_INTEGRITY_GUIDE.md](./DIRECTORY_INTEGRITY_GUIDE.md).

| Endpoint | Guard | Switch |
|---|---|---|
| `POST /v1/hierarchy-transfers/integrity/scan` | `requireDbElevated` (super-admin, nat-support, nat-ict) | none — read-only apart from the stored report |
| `GET /v1/hierarchy-transfers/integrity/reports` | `requireDbElevated` | none |
| `GET /v1/hierarchy-transfers/integrity/reports/:id` | `requireDbElevated` | none |
| `POST /v1/hierarchy-transfers/integrity/repair/preview` | `requireDbSuperAdmin` | `DIRECTORY_INTEGRITY_REPAIR_ENABLED` |
| `POST /v1/hierarchy-transfers/integrity/repair/execute` | `requireDbSuperAdmin` | `DIRECTORY_INTEGRITY_REPAIR_ENABLED` |

`GET /v1/hierarchy-transfers/integrity` is unchanged. It still answers "what is
wrong right now" without storing anything, and the deploy pipeline still reads it.

---

## POST /integrity/scan

Runs every check and stores the result in `integrityReports`. Nothing in the
directory is written.

The checks:

| Finding type | Severity | Source | Repair action |
|---|---|---|---|
| `DUPLICATE_PARISH_CODE` | HIGH | existing `GET /integrity` | none — re-code by hand or `/v1/hq-assignments/admin/recode-parish-codes` |
| `CODE_UNDER_MULTIPLE_PARENTS` | HIGH | existing HQ integrity report (`codeUnderTwoParents`) | `RECODE_SPLIT_UNIT` for area and zone; realign hint for other levels |
| `SPLIT_UNIT` | MEDIUM, LOW if it differs only by name | existing `GET /integrity` | none — `suggestedRepair.body` is a ready `/admin/realign` dry run |
| `HQ_CONFLICT` | MEDIUM | existing HQ integrity report | `RESOLVE_HQ_CONFLICT` |
| `MISSING_CODE_OR_NAME` | HIGH when a code is missing, LOW for a name | new | none — correct the row by hand |
| `CODE_NOT_REGISTERED` | MEDIUM | new — codes in the directory absent from `hierarchyCodeRegistry` | none — run `/admin/backfill-codes` |
| `CODE_REGISTERED_AT_OTHER_LEVEL` | LOW | new | none — the backfill registers it at the right level too |
| `USER_STALE` | MEDIUM | `hierarchySync` classifier | `RESYNC_USERS` |
| `USER_ORPHAN` | HIGH | `hierarchySync` classifier | none — nobody knows where they belong |
| `USER_SPLIT` | MEDIUM | `hierarchySync` classifier | none — realign the parish first |
| `USER_INACTIVE_PARISH` | LOW | new — the parish exists but has no active, undeleted row | none |
| `STALE_SECONDARY_ROLE` | MEDIUM (HIGH if the parish is gone) | new — `secondaryRoleAssignments` scope vs the parish row | `REFRESH_STALE_COPIES` |
| `STALE_PARISH_PIC` | LOW for a name, MEDIUM if the parish is gone | new — `parishPicHolders` | `REFRESH_STALE_COPIES` for a name |
| `STALE_PRINCIPAL_OFFICE` | LOW for a name, HIGH if the unit is gone | new — `principalOfficeHolders.scopeCode`/`scopeName` | `REFRESH_STALE_COPIES` for a name |
| `STALE_APPROVAL_SCOPE` | MEDIUM | new — pending `approvalRequests.scopeChain` vs the directory | `REFRESH_STALE_COPIES` |

A code under two parents is not also reported as a `SPLIT_UNIT`. Codes never
change in a move, so a principal office or PIC goes stale only on its name,
unless its unit has disappeared altogether.

### Response

```json
{
  "scanId": "scan-1794123456789-a1b2c3",
  "status": "COMPLETED",
  "startedByUsername": "jdoe",
  "startedAt": "2026-11-08T07:55:38.980Z",
  "finishedAt": "2026-11-08T07:55:52.004Z",
  "durationMs": 13024,
  "counts": {
    "total": 41,
    "findings": { "CODE_UNDER_MULTIPLE_PARENTS": 3, "USER_STALE": 12 },
    "affected": { "CODE_UNDER_MULTIPLE_PARENTS": 214, "USER_STALE": 57 }
  },
  "truncated": {},
  "findings": [
    {
      "findingId": "F0001",
      "type": "CODE_UNDER_MULTIPLE_PARENTS",
      "severity": "HIGH",
      "level": "area",
      "code": "AR8000000211",
      "entityIds": ["ZN200001", "ZN200017"],
      "affected": 31,
      "description": "area code AR8000000211 (AREA 4) sits under 2 different zones: ZN200001, ZN200017. ...",
      "suggestedRepair": { "action": "RECODE_SPLIT_UNIT", "note": "Give every side but one a newly minted code. ..." },
      "data": { "parentLevel": "zone", "parents": ["ZN200001", "ZN200017"] }
    }
  ]
}
```

- `findingId` is unique within its report and is what a repair names.
- Findings are sorted by severity, then type, level and code.
- Each type keeps at most 500 findings. `counts` always has the true totals,
  and `truncated[type]` says how many were left out.
- `entityIds` keeps at most 100 ids. `affected` is the true number.
- `suggestedRepair.action` is the repair action that applies. When it is
  `null`, the finding is report-only and `endpoint` / `body` / `note` say what
  to do by hand.

## GET /integrity/reports

Newest first, without findings. Query: `pageNo` (default 1) and `pageSize`
(default 20, max 100).

```json
{ "totalCount": 3, "pageNo": 1, "pageSize": 20, "records": [ { "scanId": "...", "status": "COMPLETED", "counts": { } } ] }
```

## GET /integrity/reports/:id

One report by `scanId` (or `_id`), with its findings. Optional filters:
`type`, `severity`, `level`. An unknown id returns **404 `SCAN_NOT_FOUND`**.

---

## POST /integrity/repair/preview

Works out exactly what a repair would write for some findings of a stored scan,
and stores that plan with a hash. Nothing in the directory is written.

```json
{
  "scanId": "scan-1794123456789-a1b2c3",
  "findingIds": ["F0001", "F0007"],
  "action": "RECODE_SPLIT_UNIT",
  "options": {
    "keepParent": { "AR8000000211": "ZN200001" },
    "keep": { "ZN200017": "211343" }
  },
  "reason": "Splitting the shared area code after the October import"
}
```

- `findingIds`: 1 to 200, all from the same scan. Every one must carry
  `suggestedRepair.action === action`. Otherwise the request fails with
  **400 `ACTION_NOT_APPLICABLE`**, listing the action each finding does take.
- `options.keepParent` (for `RECODE_SPLIT_UNIT`) maps a unit code to the parent
  whose side keeps the code. By default the side with the most rows keeps it.
- `options.keep` (for `RESOLVE_HQ_CONFLICT`) maps a unit code to the parish that
  keeps the flag. It is required, and without it the preview is blocked.

### The actions

| Action | What it writes | Mechanism reused |
|---|---|---|
| `RECODE_SPLIT_UNIT` | Every side of a split area or zone except one gets a newly minted code. That side's directory rows (departments and inactive rows included), its users' `area`/`zone`, and its secondary grants. | `Hierarchycodes.allocate` (registers the code), job snapshot |
| `RESYNC_USERS` | The ancestor codes of users at the parish, per stale tuple, from the parish's directory row. | `hierarchySync.classify`; every old value is pinned, as in the nightly sync |
| `RESOLVE_HQ_CONFLICT` | One parish keeps the flag, and every other holder's flag is set to `"0"`. | `Hqassignments.resolveConflict` |
| `REFRESH_STALE_COPIES` | Secondary grant scope and labels, PIC `parishName`, office `scopeName`, or a pending request's `scopeChain`, copied from the directory. | job snapshot |

### Response

```json
{
  "previewId": "6af0290c676c7cd8116021a4",
  "action": "RECODE_SPLIT_UNIT",
  "scanId": "scan-1794123456789-a1b2c3",
  "planHash": "9f2c…",
  "expiresAt": "2026-11-09T08:01:12.000Z",
  "executable": true,
  "blockers": [],
  "plan": {
    "action": "RECODE_SPLIT_UNIT",
    "items": [
      {
        "findingId": "F0001",
        "level": "area",
        "code": "AR8000000211",
        "parentLevel": "zone",
        "keepParent": "ZN200001",
        "recode": [
          {
            "parentCode": "ZN200017",
            "rows": 9,
            "rowIds": ["…"],
            "parishCodes": ["211343", "…"],
            "users": 140,
            "secondaryRoles": 2,
            "change": { "areaCode": { "from": "AR8000000211", "to": "(minted on execute)" } }
          }
        ],
        "officesToReview": [],
        "officesNote": ""
      }
    ]
  },
  "applyWith": { "previewId": "6af0290c676c7cd8116021a4", "idempotencyKey": "<a fresh key of your own, 8-128 chars>" }
}
```

A preview with `blockers` cannot be executed. Each blocker names its
`findingId` and a reason, for example "no longer split, re-scan" or "options.keep
must name the parish that keeps zhq". Lists of rows are capped at 500, but
counts are exact. A preview expires after 24 hours.

## POST /integrity/repair/execute

```json
{ "previewId": "6af0290c676c7cd8116021a4", "idempotencyKey": "recode-ar211-2026-11-08" }
```

1. The preview is claimed. The idempotency key is 8 to 128 characters of
   `[A-Za-z0-9._:-]`.
2. The plan is worked out again from the same request, and its hash must match
   the stored one. If it does not, the request fails with **409
   `PLAN_DRIFTED`**: nothing is written, the preview is marked `STALE`, and the
   fresh plan is in `detail.freshPlan`.
3. Each change is written as its own `hierarchyChangeJobs` row with a row-by-row
   snapshot. All the rows of one execute share `batchId = previewId`.

```json
{
  "success": true,
  "replayed": false,
  "previewId": "6af0290c676c7cd8116021a4",
  "action": "RECODE_SPLIT_UNIT",
  "batchId": "6af0290c676c7cd8116021a4",
  "jobIds": ["6af0291a676c7cd8116021c9"],
  "outcomes": [
    {
      "findingId": "F0001", "jobId": "6af0291a676c7cd8116021c9", "operation": "RECODE",
      "level": "area", "oldCode": "AR8000000211", "newCode": "AR4410029384", "parentCode": "ZN200017",
      "steps": [
        { "name": "parishDirectory", "matched": 9, "modified": 9, "done": true, "error": "" },
        { "name": "users", "matched": 140, "modified": 140, "done": true, "error": "" },
        { "name": "secondaryRoleAssignments", "matched": 2, "modified": 2, "done": true, "error": "" }
      ]
    }
  ],
  "rollback": "POST /v1/hierarchy-transfers/jobs/:id/rollback for any job above (dryRun first)."
}
```

- **Same key again:** returns the stored result with `replayed: true` and runs
  nothing.
- **Different key on an executed preview:** **409 `PREVIEW_ALREADY_EXECUTED`**.
- **Key already used on another preview:** **409 `IDEMPOTENCY_KEY_REUSED`**.
- **Execution failed part-way:** the preview is marked `FAILED`. Resend with
  the *same* key to resume. Every write is idempotent, so a resume finishes the
  job. A different key returns **409 `RESUME_NEEDS_SAME_KEY`**.

### Job rows written

| Operation | `level` / `unitCode` | Snapshot targets |
|---|---|---|
| `RECODE` | the split unit's level and its old code; `newCode` and `toParentCode` name the re-coded side | `parishDirectory` (keyed by `_id`), `users`, `secondaryRoleAssignments` |
| `INTEGRITY_REPAIR` (re-sync) | `parish` / the parish code | `users` |
| `INTEGRITY_REPAIR` (HQ) | the unit's level and code | `parishDirectoryHq` |
| `INTEGRITY_REPAIR` (copies) | parish, or the office's level and code | `secondaryRoleAssignments`, `parishPicHolders`, `principalOfficeHolders` or `approvalRequests` |

Every one of them can be undone with the existing
`POST /v1/hierarchy-transfers/jobs/:id/rollback`, dry run first. A rollback
restores only rows that still hold what the repair wrote. Two things are not
undone: a minted code stays allocated and is never reissued, and an HQ repair
also leaves the resolver's own `HQ_RESOLVE` job row in place.

---

## Error codes

| HTTP | `code` | Meaning |
|---|---|---|
| 403 | `FEATURE_DISABLED` | `DIRECTORY_INTEGRITY_REPAIR_ENABLED` is not `true`. `detail.flag` names it. |
| 400 | `INVALID_REQUEST` | The body failed validation. |
| 400 | `FINDING_NOT_FOUND` | A `findingId` is not in that scan. |
| 400 | `ACTION_NOT_APPLICABLE` | The action does not fit one of the findings. |
| 404 | `SCAN_NOT_FOUND` / `PREVIEW_NOT_FOUND` | Unknown id. |
| 409 | `PREVIEW_BLOCKED` | The preview has blockers. |
| 409 | `PLAN_DRIFTED` | The directory changed after the preview. Take a fresh one. |
| 409 | `PREVIEW_EXPIRED` | The preview is older than 24 hours. |
| 409 | `PREVIEW_ALREADY_EXECUTED` / `IDEMPOTENCY_KEY_REUSED` / `RESUME_NEEDS_SAME_KEY` / `EXECUTION_IN_PROGRESS` | See above. |

## Collections and indexes

| Collection | Indexes |
|---|---|
| `integrityReports` | `uniq_integrity_scan_id` (`scanId`, unique), `integrity_reports_recent` |
| `hierarchyChangePreviews` | `uniq_preview_idempotency_key` (unique, partial on string keys), `previews_by_kind_recent` |

Production runs `MONGO_AUTO_INDEX=false`. Create these with
`scripts/createHierarchyIntegrityIndexes.ts` (dry run first) before you switch
the repair on.

## What is deliberately not done

- **"One area or zone code belongs to one parent" is enforced in code, behind
  `HIERARCHY_UNIQUE_PARENT_ENFORCED` (default off).** When on:
  - `POST /v1/parishDirectory`, `PATCH /v1/parishDirectory/:id` (only when the
    edit changes `areaCode`, `zoneCode` or `provinceCode`) and
    `POST /v1/parishDirectory/:id/restore` answer **409
    `CODE_UNDER_OTHER_PARENT`** when the row would put an area under a second
    zone or a zone under a second province. `detail.conflicts[]` lists
    `{level, code, parentLevel, parentCode, existingParents[]}`.
  - CSV import rows that use an existing zone or area already split elsewhere
    are refused with `ZONE_UNDER_OTHER_PROVINCE` or `AREA_UNDER_OTHER_ZONE`, at
    validate and again at confirm.
  - Only active, non-department rows count. Hierarchy moves, merges, repairs
    and the parish reactivation cron are not checked here.

  Turn it on once a scan shows zero `CODE_UNDER_MULTIPLE_PARENTS` findings. No
  unique index is added to `parishDirectory`, which is at MongoDB's 64-index
  limit.
- Orphan users, duplicate parish codes and missing codes are never repaired
  automatically. Each needs a person to decide where something belongs.
