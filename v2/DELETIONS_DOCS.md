# Deleting, restoring, and the record of what went

The living reference for soft delete, restore, permanent erasure, the deletion
record and the summaries the UI reads. Dated changelog entries live beside this;
this is the page that stays current.

- [Two kinds of delete](#two-kinds-of-delete)
- [Parishes](#parishes)
- [Users](#users)
- [The deletion record](#the-deletion-record)
- [Who has been deleting](#who-has-been-deleting)
- [Restoring from a snapshot](#restoring-from-a-snapshot)
- [Metadata for the interface](#metadata-for-the-interface)
- [Error codes](#error-codes)

---

## Two kinds of delete

| | Row | Reversible | Who |
|---|---|---|---|
| **Soft** — the default | marked, stays | yes, `/restore` | see each section |
| **Permanent** — `?hard=true` | erased | **no** | super-admin or national support |

Deleting a parish used to be an unconditional `findOneAndRemove`. The row
vanished and every user pointing at its code became an orphan with no route
home — which is where the hierarchy sweep's `ORPHAN` bucket comes from. A soft
delete keeps the chain resolvable, so an administrative act no longer orphans a
congregation.

The tombstone in `deletions` is written **before** either path touches anything,
so the snapshot exists even if the delete then fails.

---

## Parishes

### `DELETE /v1/parishDirectory/:id`

Disabling a parish is a **request**, not an action — see
[the disable flow](2026-09-23-0505-new-updates-parish-disable-approval-scoped-status-and-reports.md).
An ordinary administrator gets:

```json
{
  "code": "PARISH_DISABLE_NEEDS_APPROVAL",
  "useInstead": { "useEndpoint": "/v1/approvals/parish-disable",
                  "body": { "parishCode": "211003", "reason": "" } }
}
```

Elevated callers may still act directly, and receive the parish they removed:

```json
{
  "deleted": true, "permanent": false, "parishCode": "211003",
  "parish": { "id": "652f…", "parishCode": "211003", "parishName": "RCCG EXAMPLE PARISH",
              "areaCode": "A031", "areaName": "AREA 31",
              "zoneCode": "Z012", "zoneName": "ZONE 12",
              "provinceCode": "LA47", "provinceName": "LAGOS PROVINCE 47" },
  "message": "Parish marked deleted. Restore with POST /v1/parishDirectory/652f…/restore."
}
```

`?hard=true` erases the row. **Super-admin or national support only**, checked
against the caller's live database record — an elevated role removed an hour ago
must not still destroy a parish because a token has not expired. Refused
outright for anyone else rather than quietly downgraded, because "I deleted it
permanently" and "I marked it deleted" are not the same sentence to say to an
auditor.

### `GET /v1/parishDirectory/deleted` · `POST /v1/parishDirectory/:id/restore`

Elevated. The listing returns full directory documents. The restore returns the
parish it brought back, including **what it was** — `wasDeletedAt`,
`wasDeletedBy`, `disabledReason`, `reactivateAt` — so a confirmation can name
the change rather than only report that one happened.

**400** when the parish is not soft-deleted; **404** when there is no such id.

---

## Users

### `DELETE /v1/users/:id`

Soft by default. `?hard=true` erases, for super-admin and national support.

**Anyone else sending `hard=true` still gets the soft delete**, deliberately and
long-standing: they are entitled to remove the account, it still ends up
unusable, and only the erasure is withheld. The audit line records that erasure
was asked for and refused.

An account holding a **protected or sensitive role** cannot be deleted by anyone
but a super-admin, and nobody may delete themselves.

### `GET /v1/users/deleted` · `POST /v1/users/:id/restore`

Elevated. Secrets are projected out of the listing, which also carries a `name`
assembled from `firstName` and `lastName` — a list of usernames alone is not
something a person can scan.

Restore clears `deletedAt`, `deletedBy` and `deletedByUsername` **together**: an
account keeping the name of whoever deleted it reads as though it were still
deleted. It is a named door onto what `PATCH /v1/users/:id/status` already did
with `userStatus: "ACTIVE"`.

```json
{
  "restored": true, "username": "jadesola.o",
  "user": { "id": "6520a1…", "username": "jadesola.o", "name": "Jadesola Okonkwo",
            "email": "jadesola@example.test", "phone": "0803…",
            "parish": "211003", "province": "LA47",
            "roles": "[\"pic-parish\"]", "userStatus": "ACTIVE",
            "wasDeletedAt": "2026-09-22T11:02:41.880Z",
            "wasDeletedBy": "tunde.support" }
}
```

---

## The deletion record

### `GET /v1/deletions`

Filters: `module` (`USERS` | `PARISH_DIRECTORY`), `deletedBy`,
`deletedByUsername`, `recordLabel`, `from`, `to`.

Every row carries the full `snapshot` **and** a flat `summary` — derived, never
stored, so it cannot drift from what was actually deleted:

```json
{
  "recordLabel": "211003", "deletedByUsername": "tunde.support",
  "summary": {
    "parishCode": "211003", "parishName": "RCCG EXAMPLE PARISH",
    "areaCode": "A031", "areaName": "AREA 31",
    "zoneCode": "Z012", "zoneName": "ZONE 12",
    "provinceCode": "LA47", "provinceName": "LAGOS PROVINCE 47",
    "country": "NG", "status": "1"
  },
  "snapshot": { "…": "still here, in full" }
}
```

Codes **and** names at every level — a code alone tells a reader nothing. For a
user the summary carries `username`, an assembled `name`, `email`, `phone`,
placement, `roles` and `userStatus`. It is `null` when the tombstone has no
snapshot, rather than an empty shell that would read as "this record had no
name".

**Read-only, deliberately.** There is no route that edits or removes a deletion
record: a record of what was destroyed is worth what it is only while nobody can
quietly tidy it up. Scoping comes from the caller's standing and **fails closed**
— a row whose unit cannot be determined is visible to a super-admin alone.

## Who has been deleting

### `GET /v1/deletions/actors`

The listing says *what* went; this says *who is doing it*, grouped per actor per
module with the window's first and last, so a burst reads as a burst. Four
hundred deletions inside an hour is a different story from four hundred across a
month.

```json
{ "deletedByUsername": "tunde.support", "module": "PARISH_DIRECTORY",
  "deletions": 412, "firstAt": "2026-09-14T08:02:11.000Z",
  "lastAt": "2026-09-14T08:49:55.000Z", "samples": ["211003", "211004"] }
```

## Restoring from a snapshot

### `POST /v1/deletions/:id/restore` — super-admin only

The route of last resort, for a record that was **erased**. A soft-deleted
record goes back through `/restore` on its own collection, which is cheaper and
safer.

It re-inserts under the original `_id` and then **reports what refers to it,
unreconciled**:

```json
{
  "restored": true, "module": "PARISH_DIRECTORY", "recordId": "652f…",
  "references": { "unreconciled": true, "parishCode": "211003",
                  "usersOnThisParish": 42, "note": "…" }
}
```

Re-inserting the document **does not put the world back** — a parish code is
referenced by users, offices and the hierarchy — so the response says what still
needs a human rather than implying the system is whole again. A restored **user**
comes back with `credentialRestored: false`: secrets were stripped when the
snapshot was taken, so the account cannot sign in until a password is set.

Refuses when a live row already holds that id. Restoring over something is not a
restore.

---

## Metadata for the interface

Every delete and restore response carries the record it acted on, so a
confirmation can name it. Every listing carries enough to render a row without a
second request. Nothing in these payloads is a secret — passwords, OTP secrets
and refresh tokens are projected out or were never in the snapshot.

## Error codes

| Code | Status | Meaning |
|---|---|---|
| `PARISH_DISABLE_NEEDS_APPROVAL` | 403 | raise the request instead; the response carries the endpoint |
| `PERMANENT_DELETE_FORBIDDEN` | 403 | erasing is super-admin or national support |
| `TARGET_PROTECTED` | 403 | the account holds a protected or sensitive role |
| `SELF` | 400 | you cannot delete your own account |
| *(no code)* | 400 | not soft-deleted, so there is nothing to restore |
