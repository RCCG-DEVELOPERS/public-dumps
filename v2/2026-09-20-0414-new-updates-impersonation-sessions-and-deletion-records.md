# 2026-09-20 04:14 — Borrowed sessions, sensitive roles, and the deletion record

**Commits:** `8557b6a`, `0f5a1b4`, `ec53bd3`, `e69da3d`, `f6b8ecf`, `365b218`, `6f200a7`

---

## 1. A borrowed session survives a refresh

Impersonation sessions were losing what made them impersonations when the token
rotated. The fix: `refreshTokens.impersonatedBy` is the **durable** record that a
session is borrowed, and the refresh path rebuilds `impersonate`,
`impersonatedUsername` and `impersonateMainCharacter` from it.

The consequence worth knowing, because the audit trail depends on it:

> **`sessionId` is stable for the whole borrowed session.** It is minted once when
> impersonation starts and reused on every refresh — `Auth/index.ts` reads it off
> the presented refresh token and passes the same value to both
> `signAccessToken` and `issueRefreshToken`. That is why the `jti` nonce exists:
> two tokens for the same user and session inside one second were otherwise
> byte-identical and collided on `tokenHash`.

## 2. A sensitive role is stripped, not refused

Previously, impersonating somebody who held a sensitive role was refused
outright. Now the **session is granted with that role stripped** — support can
still help the person, without borrowing what matters.

The strip is **recomputed server-side on every request** (`roleGuard.loadCaller`)
rather than trusted from the token, so a stale or forged claim can only narrow
access, never widen it.

Also: national support may impersonate **by level**. A role is impersonable only
if its `level_type` is a geographic level — national, legal and camp roles have
none, so they are refused. Both refusals log `DENIED` and return:

```json
{ "code": "ROLE_NOT_IMPERSONABLE", "message": "..." }
```

> This is why **national support cannot impersonate a national treasurer**: no
> geographic level, plus `roles.sensitive` if flagged. Two independent guards.

## 3. The deletion record became readable

Deletions were being recorded and nothing could read them back, which answers
the audit question only for somebody with a mongo shell.

### `GET /v1/deletions`

```http
GET /v1/deletions?module=PARISH_DIRECTORY&from=2026-09-01&to=2026-09-30
Authorization: Bearer <token>
```

| Query | Notes |
|---|---|
| `module` | `USERS` or `PARISH_DIRECTORY` |
| `deletedBy`, `deletedByUsername` | who did it |
| `recordLabel` | the deleted record's `parishCode` or `username` |
| `from`, `to` | ISO dates, inclusive |

Each row carries the **snapshot** of the removed document (secrets redacted),
who deleted it, their IP, and the record's place in the hierarchy copied out at
deletion time.

**Read-only, deliberately.** There is no route that edits or removes a deletion
record and there should not be: a record of what was destroyed is worth what it
is only while nobody can quietly tidy it up. Scoping comes from the caller's
standing and **fails closed** — a row whose unit cannot be determined is visible
to a super-admin alone.

Alongside it: accounts that are gone stopped appearing in user listings, and
every record now carries who last edited it (`updatedBy`, `updatedByUsername`)
as well as who deleted it.

---

## Known gaps

- The deletion record has snapshots but **no restore from them**. Putting a
  parish back from a snapshot is not a write of one document — its code is
  referenced by users, offices and the hierarchy, and a blind re-insert produces
  a row half the system still believes is gone. The soft-delete/restore added on
  2026-09-23 is the supported route back; snapshot-restore remains unbuilt.
- `/v1/activityLogs` is still mounted behind `isAuthenticated` alone, so **any
  authenticated user can read the whole activity log**. The privileged audit
  trail deliberately does not repeat this.
