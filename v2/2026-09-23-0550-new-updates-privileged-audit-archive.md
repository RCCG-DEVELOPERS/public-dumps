# 2026-09-23 05:50 — The privileged audit trail gets an archive

**Commit:** `76ab827` · **Branch:** `dev`

Written up after the fact — this landed from another session, and the reference
docs had gone stale against it.

---

## What changed

The privileged audit trail expired its rows and that was the end of them. Now
records that age out of the live collection **move to an Atlas federated
archive and stay searchable**.

That changes what retention means here. Before, `PRIVILEGED_AUDIT_RETENTION_DAYS`
was how long evidence existed. Now it is how long evidence stays *hot* — which
is why the default moved from 400 days to **1000**, and why the old framing in
`PRIVILEGED_AUDIT_DOCS.md` ("this collection expires, unlike activityLogs") was
only half the story and has been rewritten.

| Variable | Default | Meaning |
|---|---|---|
| `PRIVILEGED_AUDIT_RETENTION_DAYS` | **1000** (was 400) | time in the LIVE collection |
| `PRIVILEGED_AUDIT_ARCHIVE_URI` | *(unset)* | Atlas federated connection string |
| `PRIVILEGED_AUDIT_ARCHIVE_MAX_RANGE_DAYS` | `31` | search window ceiling |
| `PRIVILEGED_AUDIT_ARCHIVE_MAX_TIME_MS` | `45000` | per-query cap |

---

## `POST /v1/privileged-audit/archive/search`

**Guard:** super-admin — the same as every other non-`/me` route here. Being old
does not make somebody's request body less sensitive.

```http
POST /v1/privileged-audit/archive/search
Authorization: Bearer <superAdminToken>
Content-Type: application/json

{
  "from": "2026-03-01",
  "to": "2026-03-31",
  "realActorUsername": "tunde.support",
  "impersonating": true,
  "limit": 100
}
```

**200**

```json
{
  "source": "archive",
  "records": [ { "…": "the same shape as a live row" } ],
  "totalCount": 37,
  "limit": 100,
  "truncated": false,
  "limits": {
    "maxRangeDays": 31,
    "maxLimit": 500,
    "dateField": "startedAt",
    "filterable": ["sessionId", "traceId", "…"]
  }
}
```

`truncated: true` means the limit was reached and there is more — narrow the
window rather than assuming you have everything. Default limit 100, maximum 500.

### Three things that will catch you

- **The live endpoints do not reach the archive.** A search that finds nothing
  in `/requests` is not proof that nothing happened — it may only mean the rows
  have aged out. Check both tiers before concluding.
- **The window is on `startedAt`, not `createdAt`.** Writes are batched, so
  `createdAt` is insert time. The live replay sorts on `startedAt` for the same
  reason, which is what lets one date range mean the same thing across both
  tiers. The response names the field in `limits.dateField` rather than making
  you assume.
- **`from` and `to` are mandatory, capped at 31 days.** Archived data has no
  indexes — only date partition pruning — and is billed per GB scanned, so an
  unbounded query is refused rather than run expensively.

### Filters

Exact match only, no regex, and **typed** — a wrong type is refused rather than
silently matching nothing.

| Type | Fields |
|---|---|
| string | `sessionId`, `traceId`, `requestId`, `realActorId`, `realActorUsername`, `effectiveUserId`, `effectiveUsername`, `impersonatedUsername`, `borrowedRole`, `borrowedUnitLevel`, `borrowedUnitCode`, `impersonationMode`, `method`, `routePattern`, `elevatedReason`, `activities.affectedId` |
| number | `statusCode` |
| boolean | `impersonating`, `impersonationProven` |

`activities.affectedId` being filterable matters: *"who was really behind this
change to user Y"* has to keep working after the rows have aged out, which is
usually exactly when somebody asks.

### Errors

| Code | Status | Meaning |
|---|---|---|
| `ARCHIVE_NOT_CONFIGURED` | 503 | `PRIVILEGED_AUDIT_ARCHIVE_URI` unset — a server problem, not the caller's |
| `DATES_REQUIRED` | 400 | `from`/`to` missing |
| `BAD_DATE` | 400 | not ISO 8601 |
| `RANGE_INVERTED` | 400 | `to` before `from` |
| `RANGE_TOO_WIDE` | 400 | message says how many days you asked for |
| `UNKNOWN_FILTER` | 400 | message lists what *can* be filtered |
| `BAD_FILTER_VALUE` | 400 | the filter exists but the value is the wrong type — `statusCode: "200"` rather than `200` |

---

## Deployment

`PRIVILEGED_AUDIT_ARCHIVE_URI` must point at the Atlas federated endpoint, and
needs registering in `deployment-scripts/lib.sh` like the rest. Until it is set,
the endpoint answers 503 and everything else is unaffected — the live trail does
not depend on it.

## Docs corrected by this note

- `v1/PRIVILEGED_AUDIT_DOCS.md` — new "Searching the archive" section, the
  endpoint table, and the retention section rewritten as two tiers
- `v1/CONFIGURATION.md` — the three new variables and the changed default
- the 2026-09-22 note — pointed here, since it quoted the old 400
