# Invitation management

How a parish administrator sees, re-sends and withdraws the registration invitations their
parish has issued. All routes are under `/api/v1/backend/members` and need a bearer token
(`Authorization: Bearer <jwt>`). Everything is scoped to the caller's token hierarchy: a parish
pastor sees their parish's invitations, an area coordinator every parish in their area, and so on.

Creating invitations is covered in [MEMBER_CREATION.md](MEMBER_CREATION.md); bulk upload from a
file in [BULK_INVITATION_UPLOAD.md](BULK_INVITATION_UPLOAD.md); what the invitee does in
[INVITATION_COMPLETION.md](INVITATION_COMPLETION.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `members/invitations` | Paginated list, newest first, with filters |
| `GET` | `members/invitations/{id}` | One invitation |
| `POST` | `members/invitations/{id}/resend` | Send the **same link** again |
| `POST` | `members/invitations/{id}/revoke` | Withdraw the link |

`{id}` is the invitation's MongoDB `_id` (the `_id` field in every response), or the legacy
integer id for rows migrated from the old system.

---

## Status

Every invitation carries a derived `invitation_status`:

| Status | Meaning | Re-send | Revoke |
|---|---|---|---|
| `PENDING` | Sent, not yet used | yes | yes |
| `EXPIRED` | Pending, but older than `MEMBER_INVITE_TTL_DAYS` (default 30) since it was sent or last re-sent | yes (the window restarts) | yes |
| `ACCEPTED` | The invitee completed registration; the member exists | `422` | `422` |
| `REVOKED` | Withdrawn by an administrator | `422` (send a new invitation instead) | `200`, no-op |

Invitations migrated from the old system never expire. An invitation whose link is being
completed at that very moment answers `409` to both re-send and revoke; try again shortly.

---

## List — `GET members/invitations`

```
GET members/invitations?status=PENDING&email=obi&page=1&per_page=20&sort=created_at:desc
```

| Query | Meaning |
|---|---|
| `status` | `PENDING`, `EXPIRED`, `ACCEPTED` or `REVOKED` |
| `email`, `first_name`, `last_name`, `phone` | Case-insensitive substring |
| `invited_by` | `user_code` of the administrator who sent it (exact) |
| `batch_id` | Only rows created by one bulk upload batch |
| `from`, `to` | Sent on or after / on or before (ISO 8601; a bare date covers the whole day, UTC) |
| `sort` | `created_at`, `resent_at`, `email`, `first_name` or `last_name`, optionally `:asc` / `:desc`. Default `created_at:desc` |
| `page`, `per_page` | 1-based page; 1..200 rows per page, default 20 |

The response is the Laravel pagination envelope (`data`, `links`, `meta`):

```json
{
  "data": [
    {
      "id": null,
      "email": "ngozi@example.org",
      "first_name": "Ngozi",
      "last_name": "Okonkwo",
      "phone_code": "44",
      "phone": "07012345678",
      "parish_code": "211414",
      "parishName": "RCCG Victory House",
      "status": "pending",
      "completed": 0,
      "resent_at": "2026-10-20T09:00:00.000Z",
      "invited_by": "6410b5c8e4b0a1b2c3d4e5f6",
      "invited_via": "single",
      "import_batch_id": null,
      "resend_count": 1,
      "revoked_at": null,
      "revoked_by": null,
      "revoked_reason": null,
      "created_at": "2026-10-01T09:00:00.000Z",
      "updated_at": "2026-10-20T09:00:00.000Z",
      "_id": "66f1c0a2b3d4e5f6a7b8c9d0",
      "invitation_status": "PENDING",
      "expires_at": "2026-11-19T09:00:00.000Z",
      "last_resent_at": "2026-10-20T09:00:00.000Z"
    }
  ],
  "links": { "first": "…?page=1", "last": "…?page=1", "prev": null, "next": null },
  "meta": { "current_page": 1, "from": 1, "last_page": 1, "path": "…", "per_page": 20, "to": 1, "total": 1 }
}
```

**The link code is never returned** on any management route. It is the credential the invitee
uses; only their email carries it. `status` is the raw stored value (`pending`, `processing`,
`completed`, `revoked`); use `invitation_status` for display.

`invited_via` is `single` (one invitation), `bulk_json` (the JSON bulk route) or `bulk_upload`
(a file batch, with `import_batch_id`). Rows from before this feature, and rows migrated from the
old system, have these fields null.

---

## Detail — `GET members/invitations/{id}`

The same object as one list row. `404` when the id is not an invitation within the caller's
scope; `400` when it is neither an integer nor a 24-character id.

---

## Re-send — `POST members/invitations/{id}/resend`

No body. Mails the **original link** again (same code, so a link the invitee already has keeps
working), restarts the validity window and increments `resend_count`. The link points at the
frontend making the call when its origin is allowed, otherwise at the frontend that issued the
invitation. Limited to 20 per minute per IP.

```json
{
  "message": {
    "success": "The registration link has been sent again.",
    "resent": true,
    "fullname": "Ngozi Okonkwo",
    "email": "ngozi@example.org",
    "invitation": { "…": "the invitation, as in the list, after the re-send" }
  }
}
```

| Status | Meaning |
|---|---|
| `200` | Sent |
| `404` | Not found within scope |
| `409` | The invitee is completing registration right now |
| `422` | Accepted or revoked, or no frontend to link to |
| `429` | Rate limit |

This is the same operation as inviting the same email again through
`POST members/addTempLiterateMember`, reachable by id instead of by email.

---

## Revoke — `POST members/invitations/{id}/revoke`

```json
{ "reason": "Sent to the wrong address" }
```

`reason` is optional (≤ 500 characters) and is stored on the row with `revoked_at` and
`revoked_by`. After revoking:

- the link answers `410` to the invitee ("withdrawn, ask your parish for a new invitation");
- the row stays, with `invitation_status: REVOKED`, as history;
- the same email **can be invited again**: `POST members/addTempLiterateMember` creates a new
  row with a new link, and a bulk upload grades the email as `ELIGIBLE`.

```json
{
  "message": {
    "success": "Invitation revoked. The registration link no longer works.",
    "revoked": true,
    "invitation": { "…": "the invitation, as in the list, now REVOKED" }
  }
}
```

| Status | Meaning |
|---|---|
| `200` | Revoked, or was already revoked (`success` says which; `revoked: true` either way) |
| `404` | Not found within scope |
| `409` | The invitee is completing registration right now, or the row changed under the call |
| `422` | Already accepted; the member exists |

---

## Audit

Each row records `invited_by`, `invited_via`, `import_batch_id`, `resend_count`, `resent_at`,
`revoked_at`, `revoked_by` and `revoked_reason`. The application log carries one structured
event per action: `MEMBER_INVITE_CREATED`, `MEMBER_INVITE_RESENT`, `MEMBER_INVITE_REVOKED` (with
the request id and parish code, never the link code or the invitee's details).
