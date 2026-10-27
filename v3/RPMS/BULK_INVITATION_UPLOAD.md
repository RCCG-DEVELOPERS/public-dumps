# Bulk invitation upload

Inviting many literate members at once from a spreadsheet. The file is validated into a
**batch** that can be reviewed before anything is sent; the batch is then imported in chunks,
safely re-runnable, and every row's outcome can be read back or downloaded.

All routes are under `/api/v1/backend/members` and need a bearer token. Batches are scoped to
the uploader's token hierarchy, exactly like invitations. Invitations created this way are the
ordinary invitations described in [MEMBER_CREATION.md](MEMBER_CREATION.md) and managed through
[INVITATION_MANAGEMENT.md](INVITATION_MANAGEMENT.md).

| Method | Path | What it does |
|---|---|---|
| `GET` | `members/bulk-upload/template?type=literate&format=xlsx\|csv` | Download the template |
| `POST` | `members/bulk-upload/invitations` | Upload a file; validate it into a batch (sends nothing) |
| `GET` | `members/bulk-upload/batches` | List batches |
| `GET` | `members/bulk-upload/{batchId}/preview` | The graded rows before import |
| `GET` | `members/bulk-upload/{batchId}/status` | Batch status and summary |
| `POST` | `members/bulk-upload/{batchId}/import` | Send the invitations (chunked, re-runnable) |
| `GET` | `members/bulk-upload/{batchId}/results` | Per-row outcomes |
| `GET` | `members/bulk-upload/{batchId}/results/download` | The same, as a CSV file |

Only `type=literate` exists. A non-literate (no email) bulk upload is a follow-up and answers
`422` today. The older JSON route `POST members/bulkaddTempLiterateMember` is unchanged, and
`POST members/bulk-upload` (a different import, of full member records) is untouched.

---

## 1. Template — `GET members/bulk-upload/template`

Returns a file with the columns `first_name, last_name, email, phone, phone_code` and one
example row; `.xlsx` by default, `.csv` with `format=csv`. In the workbook the phone columns are
text-formatted so leading zeros survive editing.

Header spellings are forgiving: `First Name`, `firstname`, `Surname`, `E-Mail`, `Mobile`,
`Country Code`… all map onto the five columns. `phone_code` is optional; the other four are
required and their absence is a `422` naming the missing columns.

---

## 2. Upload — `POST members/bulk-upload/invitations`

`multipart/form-data`:

| Field | Required | Meaning |
|---|---|---|
| `file` | yes | `.xlsx` or `.csv`, at most `BULK_UPLOAD_MAX_BYTES` (default 2 MB) and `MEMBER_IMPORT_MAX_ROWS` (default 5000) data rows |
| `parishPastorName` | yes | Shown in every invitation mail |
| `parishName` | yes | Shown in every invitation mail |
| `frontendUrl` | see note | Base URL for the links in the mails. Needed only when the request origin is not an allowed frontend and `FRONTEND_URL` is unset — the same rule as `addTempLiterateMember` |
| `type` | no | `literate` (default and only value) |
| `parish_code` | see note | Read only when the token has **no** parish claim (a coordinator above parish level); must be a parish under their level |

The batch is filed under the uploader's token hierarchy. Every row is graded and the batch is
stored as `VALIDATED`. **No invitation is created and no email is sent at this step.**

```bash
curl -X POST "$BASE/members/bulk-upload/invitations" \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@invitees.xlsx" \
  -F "parishPastorName=Pastor Ade Balogun" \
  -F "parishName=RCCG Victory House"
```

Response `201`:

```json
{
  "message": "File validated. Review the preview, then call import to send the invitations.",
  "batch_id": "66f1c0a2b3d4e5f6a7b8c9d0",
  "status": "VALIDATED",
  "file": { "original_name": "invitees.xlsx", "format": "xlsx", "size": 10234 },
  "headers": ["First Name", "Last Name", "E-Mail", "Mobile", "Country Code"],
  "columns": { "first_name": "first_name", "last_name": "last_name", "email": "e_mail", "phone": "mobile", "phone_code": "country_code" },
  "skipped_lines": [],
  "summary": {
    "total_rows": 120, "eligible": 110, "pending_invitation": 3, "existing_member": 4,
    "existing_user": 1, "duplicate_in_file": 2, "invalid_data": 0, "invitable": 113,
    "import": { "pending": 113, "success": 0, "skipped": 0, "failed": 0 }
  },
  "preview": [ "…the first 20 rows, as in the preview route…" ]
}
```

### Row classifications

| Classification | Meaning | Imported? |
|---|---|---|
| `ELIGIBLE` | Can be invited | yes |
| `PENDING_INVITATION` | Already has a pending invitation; the same link will be sent again | yes (re-send) |
| `EXISTING_MEMBER` | Already a member, or already completed an invitation | no |
| `EXISTING_USER` | Already has a portal account in the auth service | no |
| `DUPLICATE_IN_FILE` | The email appears more than once in the file (every occurrence is marked) | no |
| `INVALID_DATA` | Missing or malformed email, name or phone; `message` lists the problems | no |

Revoked invitations are ignored when grading: a revoked email is `ELIGIBLE` again.

| Status | Meaning |
|---|---|
| `201` | Validated |
| `403` | No hierarchy claim, or a parish outside the caller's level |
| `413` | Over `BULK_UPLOAD_MAX_BYTES` |
| `422` | No file, wrong type, unreadable, missing columns, too many rows, or missing shared fields (`errors` names them) |
| `429` | More than 10 uploads a minute |

---

## 3. Review — preview, status, batches

- `GET members/bulk-upload/{batchId}/preview?classification=INVALID_DATA&page=1&per_page=50`
  returns the batch (without rows) plus a page of rows; filter by classification.
- `GET members/bulk-upload/{batchId}/status` returns the batch summary alone.
- `GET members/bulk-upload/batches?status=VALIDATED&from=2026-01-01` lists batches without
  their rows, newest first.

Each row:

```json
{
  "n": 7, "sheet_row": 8,
  "first_name": "Ngozi", "last_name": "Okonkwo", "email": "ngozi@example.org",
  "phone": "07012345678", "phone_code": "44",
  "classification": "ELIGIBLE", "reason": null, "message": "Email is available",
  "recommendation": "Will be invited.",
  "import": { "status": "pending", "attempts": 0, "resent": false, "reason": null, "message": null, "processed_at": null }
}
```

Batch statuses: `VALIDATED` → `IMPORTING` → `COMPLETED` | `PARTIALLY_COMPLETED` | `FAILED`.

---

## 4. Import — `POST members/bulk-upload/{batchId}/import`

```json
{ "max_rows": 500 }
```

Invites up to `max_rows` (default `MEMBER_IMPORT_ROWS_PER_RUN`, 500) invitable rows that have
not succeeded yet, **one at a time through the same path as a single invitation** (so every rule
in [MEMBER_CREATION.md](MEMBER_CREATION.md) applies and the hierarchy comes from the token).
Each row's outcome is written as it happens. Limited to 10 calls a minute per IP.

Before inviting, the rows are graded again: an email taken since the upload is re-classified and
skipped rather than invited. A row that fails (for instance the auth service refusing it) is
recorded `failed` with the message and is retried on the next call.

**Retry = call import again.** Rows already `success` are never repeated. Rows never tried run
before earlier failures, so one row that keeps failing cannot hold up the rest. A row the
invitation path **refuses** (a `422` or `409`, for instance a pending invitation from another
parish) is marked `terminal: true`: it stays `failed`, is not retried, and no longer keeps the
batch from completing. Other failures (mail or auth-service outages) are retried on the next call.

Two concurrent calls cannot both run: the second answers `409`. A run whose process stopped
before settling (a deploy, a crash) holds its claim for 30 minutes; after that the next import
call takes the batch over and continues.

| Status | Meaning |
|---|---|
| `201` | `COMPLETED` — every invitable row has been sent |
| `207` | `PARTIALLY_COMPLETED` (rows remain or failed; call again) or `FAILED` (the run stopped; `last_error` says why) |
| `200` | Already completed; nothing to do |
| `404` | Not found within scope |
| `409` | Being imported right now |
| `429` | Rate limit |

```json
{
  "message": "Import run finished with rows remaining. Call import again to continue.",
  "batch_id": "66f1c0a2b3d4e5f6a7b8c9d0",
  "status": "PARTIALLY_COMPLETED",
  "summary": { "…": "…", "import": { "pending": 13, "success": 100, "skipped": 0, "failed": 0 } },
  "import_run_count": 1,
  "last_import_at": "2026-10-27T10:00:00.000Z",
  "last_error": null,
  "run": { "processed": 100, "sent": 97, "resent": 3, "skipped": 0, "failed": 0, "remaining": 13 },
  "results": [ "…the rows processed in this run…" ]
}
```

The invitations created carry `invited_via: "bulk_upload"` and `import_batch_id`, so
`GET members/invitations?batch_id=…` lists them. Link codes are never in any batch response.

---

## 5. Results — `GET members/bulk-upload/{batchId}/results[/download]`

`results?status=failed` pages the rows by import outcome (`pending`, `success`, `skipped`,
`failed`). `results/download` returns every row as a CSV file (`row, file_line, first_name,
last_name, email, phone, phone_code, classification, reason, message, import_status,
import_message, resent, attempts, processed_at`). Values are escaped, and anything that looks like
a spreadsheet formula is prefixed with `'` so it opens as text.

---

## Limits and configuration

| Variable | Default | Meaning |
|---|---|---|
| `BULK_UPLOAD_MAX_BYTES` | `2097152` | Largest file accepted (`413` above it) |
| `MEMBER_IMPORT_MAX_ROWS` | `5000` | Most data rows per file (`422` above it; hard cap 5000) |
| `MEMBER_IMPORT_ROWS_PER_RUN` | `500` | Rows invited per import call when `max_rows` is not sent |
| `MEMBER_INVITE_TTL_DAYS` | `30` | How long each link stays valid |

Imports run inside the request; with 500 rows a call takes as long as 500 single invitations
(each one mail, one SMS and one auth-service check). Lower `max_rows` on slow mail or SMS links.

---

## Follow-up

Non-literate members (no email, registered on their behalf) are not covered by this upload; the
`type` field is reserved for it.
