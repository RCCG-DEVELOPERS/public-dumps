# 2026-09-23 04:19 — Deletion, restore, and who may rename

**Commit:** `d71d6c6` · **Branch:** `dev` · **Verified:** 1174 passing in an isolated worktree

---

## What changed and why

**Deleting a parish used to destroy it.** `ParishdirectoryService.remove` was an
unconditional `findOneAndRemove`: the row vanished, and every user pointing at
its `parishCode` silently became an orphan with no route home. That is precisely
where the hierarchy sweep's `ORPHAN` bucket comes from — it has been reporting
the damage this caused.

A plain `DELETE` now **marks** the row instead. `deletedAt` is set and `status`
drops to `"0"`, which makes every existing `ACTIVE_STATUS` ("1") filter skip it
without a single query being rewritten — and the chain still resolves, so an
administrative act no longer orphans a congregation.

Both users and parishes now have the same three verbs: **list what is deleted**,
**restore it**, and **erase it for good**.

---

## Behaviour changes on deploy

These take effect immediately, with no flag.

| Change | Who it affects |
|---|---|
| `DELETE /v1/parishDirectory/:id` is now **soft** | anyone who deletes parishes. The row survives and the parish can be restored |
| `?hard=true` on that route is **refused** unless you are super-admin or national support | it is *not* silently downgraded to a soft delete — you get a 403 |
| **National support may change a username** | previously super-admin only |
| **National support may erase a parish** | see the warning below |

> **`?hard=true` cannot be undone.** The row is gone. The tombstone in
> `deletions` keeps the snapshot, so the evidence survives — but the live record
> does not, and restore cannot help you. National support was admitted here on
> request; every erasure is recorded against them by name.

---

## Endpoints

### `DELETE /v1/parishDirectory/:id` — soft by default

```http
DELETE /v1/parishDirectory/652f1a0000000000000000aa
Authorization: Bearer <token>
```

**200**

```json
{
  "deleted": true,
  "permanent": false,
  "parishCode": "211003",
  "message": "Parish marked deleted. Restore with POST /v1/parishDirectory/652f1a0000000000000000aa/restore."
}
```

What actually changed on the row: `deletedAt` is set to an ISO timestamp,
`deletedBy` to your username, `status` to `"0"`. Nothing else is touched.

### `DELETE /v1/parishDirectory/:id?hard=true` — erase

**Guard:** super-admin or `ELEVATED_ROLES`, read from the **live database
record**, not the token — an elevated role removed an hour ago must not still
destroy a parish because a token has not expired.

**200** — the removed document.

**403** — for anyone else:

```json
{
  "code": "PERMANENT_DELETE_FORBIDDEN",
  "message": "Erasing a parish is a super-admin or national-support action and cannot be undone. A plain DELETE marks it deleted instead, which is reversible and keeps its members' hierarchy resolvable."
}
```

### `GET /v1/parishDirectory/deleted`

**Guard:** elevated.

```http
GET /v1/parishDirectory/deleted?pageNo=1&pageSize=25
```

**200**

```json
{
  "total": 3,
  "pageNo": 1,
  "pageSize": 25,
  "parishes": [
    {
      "_id": "652f1a0000000000000000aa",
      "parishCode": "211003",
      "parishName": "RCCG EXAMPLE PARISH",
      "status": "0",
      "deletedAt": "2026-09-23T04:19:22.104Z",
      "deletedBy": "tunde.support",
      "provinceCode": "LA47"
    }
  ]
}
```

> Declared **before** `/:id` in the router. Express matches in order, and
> `"deleted"` would otherwise be read as a parish id and 404 every time.

### `POST /v1/parishDirectory/:id/restore`

**Guard:** elevated.

**200** — `{ "restored": true, "parishCode": "211003", "message": "Parish restored and active again." }`

**400** — the parish is not soft-deleted:

```json
{ "restored": false, "message": "That parish is not soft-deleted, so there is nothing to restore." }
```

**404** — no parish with that id.

### `GET /v1/users/deleted`

**Guard:** elevated. Secrets (`password`, `otpSecret`, `refreshToken`) are
projected out.

```json
{
  "total": 12,
  "pageNo": 1,
  "pageSize": 25,
  "users": [
    {
      "_id": "6520a10000000000000000bb",
      "username": "jadesola.o",
      "userStatus": "DELETED",
      "status": "0",
      "deletedAt": "2026-09-22T11:02:41.880Z",
      "deletedByUsername": "tunde.support",
      "parish": "211003"
    }
  ]
}
```

### `POST /v1/users/:id/restore`

**Guard:** elevated.

**200** — `{ "restored": true, "username": "jadesola.o", "message": "Account restored and active again." }`

This is a **named door** onto what `PATCH /v1/users/:id/status` already did with
`{"userStatus":"ACTIVE"}`. It clears `deletedAt`, `deletedBy` and
`deletedByUsername` **together** — an account that keeps the name of whoever
deleted it reads as though it were still deleted.

**400** — the account is not soft-deleted.

### `PATCH /v1/users/:id` — renaming

```http
PATCH /v1/users/6520a10000000000000000bb
Content-Type: application/json

{ "username": "jadesola.olumide", "phone": "08031234567" }
```

**403** for anyone below super-admin or national support:

```json
{
  "message": "Changing a username is a super-admin or national-support action. The username is what this account's history is recorded against, so renaming it re-points every activity line already written about jadesola.o. Every other field on this account can be edited here as usual."
}
```

Two behaviours worth knowing:

- **An unchanged username is stripped, not refused.** Clients PATCH the whole
  record back as they read it; refusing every such request would break routine
  editing for everyone. The field is dropped, so the audit line does not claim a
  change that did not happen.
- **It fails closed when no caller can be identified**, which also closes
  `PATCH /v1/usersTemp/:id` — the same router is mounted there *unauthenticated*
  for a provisioning service.

---

## The tombstone, either way

Both the soft and the hard path write to `deletions` **before** anything is
changed, so the snapshot exists even if the delete then fails.

```http
GET /v1/deletions?module=PARISH_DIRECTORY&from=2026-09-01&to=2026-09-30
```

Read-only, deliberately: a record of what was destroyed is worth what it is only
while nobody can quietly tidy it up. Scoping comes from the caller's standing and
fails closed — a row whose unit cannot be determined is visible to a super-admin
alone.

---

## Known gaps

- **Reads that do not filter on `status` will still return soft-deleted
  parishes.** They would have returned them before the delete too, so nothing
  regressed; but if you rely on a listing being "live parishes only", check it
  filters `status: "1"`.
- **No scoping on the new routes yet.** `GET /deleted` and `/restore` are
  elevated-only, not bounded by the caller's own province. That is deliberate for
  now — narrow it when the approval flow lands.
- **Parish disable-by-approval is not built.** The agreed direction is that
  ordinary administrators stop deleting parishes altogether and instead *request*
  a disable with reasons, approved by super-admin, auto-reactivating after six
  months. The machinery here — mark, restore, list — is what that will sit on.
