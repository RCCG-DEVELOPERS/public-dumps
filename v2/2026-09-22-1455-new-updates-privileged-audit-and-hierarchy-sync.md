# 2026-09-22 14:55 — Privileged audit trail, hierarchy sweep, two new offices

**Commits:** `e8325fc`, `7b5e11d`, `be69727`, `80c0485` · **Branch:** `dev`

Full references: [PRIVILEGED_AUDIT_DOCS.md](../PRIVILEGED_AUDIT_DOCS.md) and
[HIERARCHY_SYNC_DOCS.md](../HIERARCHY_SYNC_DOCS.md). This is the summary.

---

## 1. Privileged audit trail — `/v1/privileged-audit`

### Why

`activityLogs` cannot answer *who was really behind this*. `logActivity` reads
the actor from the JWT, and under impersonation **that claim is the person being
impersonated** — so every line of a borrowed session named the victim. Three
lines were ever written about an impersonation, all at the start, and the
impersonator's name lived inside an English sentence rather than a field.

A new `privilegedAuditLogs` collection takes **one row per request** from a
super-admin, an elevated role, or anyone inside a borrowed session.

### Endpoints

All super-admin only except `/me/sessions`.

| Method | Path | Answers |
|---|---|---|
| GET | `/sessions?from&to` | which impersonation sessions happened |
| GET | `/sessions/:sessionId` | **the replay** — every request, oldest first |
| GET | `/requests?affectedId=` | **who was really behind this change to user Y** |
| GET | `/me/sessions?from&to` | my own sessions, scoped server-side |
| GET | `/:id` | one request in full |
| PATCH/DELETE | `/sessions/:sessionId/hold` | legal hold on/off |

`from` and `to` are **required** on session listings and may span at most 31
days — an unbounded scan is refused.

### A real row

```json
{
  "realActorUsername": "tunde.support",
  "realActorRoles": "[\"nat-support\"]",
  "rolesSource": "db",
  "effectiveUsername": "jadesola.o",
  "impersonating": true,
  "impersonationMode": "user",
  "sessionId": "a7f3c1e2-...",
  "method": "PATCH",
  "path": "/v1/users/6520a1",
  "statusCode": 200,
  "requestBody": "{\"phone\":\"08031234567\",\"password\":\"[redacted]\"}",
  "activities": [
    { "activityLogId": "6ab27f10...", "activity": "UPDATE_USER", "module": "USERS",
      "affectedId": "6520a1", "affectedUsername": "jadesola.o", "status": "SUCCESS" }
  ],
  "startedAt": "2026-09-22T09:14:02.000Z"
}
```

Read as: **tunde.support**, acting as **jadesola.o**, changed her phone number.
`activityLogs` would have said jadesola.o changed her own record.

### Things that will catch you

- **Sort on `startedAt`, never `createdAt`.** Writes are batched, so `createdAt`
  is insert time and puts the story out of order.
- **`impersonationMode`** is `user` or `role`. In `role` mode the borrowed slug
  is in `borrowedRole`, with `borrowedUnitLevel`/`borrowedUnitCode` for its
  blast radius — a role is not a person and is not filed as one.
- **A tripped budget is not an error.** Everything returns 200; check the body.
- **Reading this log writes to it.** Intended, and it terminates — response
  bodies are not stored.

### Configuration

`PRIVILEGED_AUDIT_MODE` defaults to `impersonation` (a few hundred rows a day).
`elevated` adds all super-admin and nat-support work: 15k–90k a day. An
unrecognised value falls back to `impersonation`, never the expensive one.

**Run `scripts/createPrivilegedAuditIndexes.ts` before this takes traffic.**

---

## 2. User hierarchy sweep — `/v1/hierarchy-sync`

### Why

Transfers already cascade a **unit** move onto its members. Nothing covered a
user whose own `parish` changed, a directory row edited outside a transfer, or a
user whose parish code exists nowhere.

### Endpoints

| Method | Path | Guard |
|---|---|---|
| GET | `/runs` | elevated |
| GET | `/runs/:id` | elevated |
| GET | `/orphans` | elevated |
| POST | `/run` | **super-admin** (the route accepts `{"mode":"apply"}`) |

```http
POST /v1/hierarchy-sync/run
Content-Type: application/json

{ "mode": "report", "force": false }
```

**200** — a real report run:

```json
{
  "batchId": "hs-1790084955844",
  "mode": "report",
  "status": "COMPLETED_WITH_FAULTS",
  "directoryCodes": 52898,
  "scanned": 55014,
  "tuples": 24911,
  "counts": { "IN_SYNC": 54782, "STALE": 106, "ORPHAN": 94,
              "DEPARTMENT": 18, "SPLIT": 14, "NO_PARISH": 0,
              "directoryGaps": { "zone": 3 } },
  "usersStale": 106,
  "usersModified": 0,
  "changes": [
    { "parishCode": "211003",
      "from": { "province": "LA30", "zone": "Z04" },
      "to": { "province": "LA47", "zone": "Z12" },
      "users": 2, "modified": 0 }
  ],
  "orphans": [{ "parishCode": "874112", "users": 1 }],
  "splits": [{ "parishCode": "990001", "variants": 2, "users": 1 }],
  "circuitBreaker": { "tripped": false, "threshold": 20, "wouldModify": 106 }
}
```

### The rules that matter

- **Three of six buckets are never modified** — `ORPHAN`, `DEPARTMENT`, `SPLIT`.
  We do not know where those people belong and a guess would move a real human
  being. Reported, not repaired.
- **An empty directory cell never blanks a user's code.** Otherwise one bad
  directory save wipes six codes off a whole parish.
- **A tripped circuit breaker returns `200` with `status: "ABORTED"`** and wrote
  nothing. Check `status`, not the HTTP code.
- **`report` is the default everywhere** — env, script, and the endpoint body.

Splits hand off to realign: `POST /v1/hierarchy-transfers/admin/realign`.

### Configuration

`HIERARCHY_SYNC_ENABLED` is **off** — the cron does not exist until it is `true`.
`HIERARCHY_SYNC_MODE` defaults to `report`. `HIERARCHY_SYNC_ON_SWITCH` defaults
**on**: switching role refreshes that one caller's codes first, so the new token
carries a current scope.

---

## 3. Two new principal offices

`apicr-csr` (region) and `apicp-csr-2` (province) were added to
`scripts/flagPrincipalOfficeRoles.js` and every document listing offices.

**`apicp-csr-2` is a separate office, not a second holder of `apicp-csr`.**
Uniqueness is per slug per unit, so a province may fill one of each and no more.

Apply with:

```bash
node scripts/flagPrincipalOfficeRoles.js --dry-run
node scripts/flagPrincipalOfficeRoles.js
```

Check `level_type` on both role documents first — the unit comes from it, and a
role with a blank or unrecognised `level_type` is **never** treated as an office
however the flag is set. It fails silently, with no error.

---

## Known gaps

- `PRIVILEGED_AUDIT_MODE=elevated` has not been run at production volume. Watch
  collection growth for a week first.
- The sweep has never applied against production. Report first, read the orphan
  count, then flip.
- `parishDirectory` indexes `parishCode` as `parish_code_unique` while the schema
  declares it mongoose's way. With `MONGO_AUTO_INDEX` on that is an
  `IndexOptionsConflict` (code 85) reported on a connection event nothing
  listens to — a silent error on every boot. Pre-existing; worth a look.
