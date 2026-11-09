# Frontend integration notes

This page is for `remittance-frontend-v2`. It lists, in one place, every
endpoint the auth and directory service has added or changed, the response
shapes, the new error codes, and the guard changes that remove access. Each
section links to the full reference.

Other features add their own sections below.

## Hierarchy

References: [DIRECTORY_INTEGRITY_DOCS.md](./DIRECTORY_INTEGRITY_DOCS.md),
[HIERARCHY_TRANSFER_DOCS.md](./HIERARCHY_TRANSFER_DOCS.md#bulk-move-and-merge),
with the operator guides [DIRECTORY_INTEGRITY_GUIDE.md](./DIRECTORY_INTEGRITY_GUIDE.md)
and [HIERARCHY_MOVES_GUIDE.md](./HIERARCHY_MOVES_GUIDE.md#bulk-moves-and-merges).

**No existing endpoint has changed.** `GET /integrity`, `/transfer`,
single-unit `/admin/move`, `/admin/realign`, `/promote` and `/jobs/:id/rollback`
behave as before. `GET /jobs` also accepts `?batchId=`.

New endpoints, all under `/v1/hierarchy-transfers`:

| Endpoint | Who | Switch | Screen |
|---|---|---|---|
| `POST /integrity/scan` | super-admin, nat-support, nat-ict | none | **Integrity**: a "Run scan" button |
| `GET /integrity/reports?pageNo&pageSize` | same | none | A list of reports: date, who, counts by type |
| `GET /integrity/reports/:scanId?type&severity&level` | same | none | Findings grouped by severity and type. Show `description`. Show a checkbox only where `suggestedRepair.action` is set; otherwise show `suggestedRepair.note` and `endpoint` as advice |
| `POST /integrity/repair/preview` `{scanId, findingIds[], action, options?, reason?}` | super-admin | `DIRECTORY_INTEGRITY_REPAIR_ENABLED` | **Repair preview**: `plan.items` as before/after tables. Disable Execute while `blockers` is non-empty |
| `POST /integrity/repair/execute` `{previewId, idempotencyKey}` | super-admin | same | Execute. Show `jobIds`, each linking to the job and its rollback |
| `POST /admin/move` `{level, unitCodes[], toLevel, toCode, reason?, acknowledgeDemotion?}` | super-admin | `HIERARCHY_BULK_MIGRATION_ENABLED` | **Bulk move**: multi-select units of one level, pick the destination, then show `plan.totals` and per-unit `counts`, `officersScopeChanging`, `officesVacated`, `hqDemotions` |
| `POST /admin/move` `{previewId, idempotencyKey}` | super-admin | same | Execute the bulk move |
| `POST /admin/merge` `{level, sourceCode, toCode, reason?, acknowledgeDemotion?}` | super-admin | same | **Merge**: source and destination pickers at the same level (zone and above). Show `plan.children`, `plan.retire` and `codesChanged` ("none") |
| `POST /admin/merge` `{previewId, idempotencyKey}` | super-admin | same | Execute the merge |

Contract for every preview-bound execute:

- Generate the `idempotencyKey` in the browser when the preview screen opens
  (8 to 128 characters of `[A-Za-z0-9._:-]`), and reuse it on every retry.
- A retry with the same key returns the first result with `replayed: true`.
- **409 `PLAN_DRIFTED`** means the data changed after the preview. Show
  `detail.freshPlan` and ask for a new preview.
- A preview expires after 24 hours (**409 `PREVIEW_EXPIRED`**).
- A bulk move or merge **ends the active role sessions** of people under the
  moved units. At their next token refresh those users fall back to their own
  record and primary role, so the role switcher should simply offer the
  choice again.

### New error codes in this section

| Code | HTTP |
|---|---|
| `FEATURE_DISABLED` | 403. Hide the repair, bulk-move and merge actions |
| `SCAN_NOT_FOUND`, `PREVIEW_NOT_FOUND` | 404 |
| `FINDING_NOT_FOUND`, `ACTION_NOT_APPLICABLE`, `SAME_UNIT`, `PREVIEW_KIND_MISMATCH` | 400 |
| `CODE_UNDER_OTHER_PARENT` | 409 on parish create, edit and restore while `HIERARCHY_UNIQUE_PARENT_ENFORCED` is on. Show `message`; `detail.conflicts[].existingParents` says where the code already belongs |
| `PLAN_DRIFTED`, `PREVIEW_BLOCKED`, `PREVIEW_EXPIRED`, `PREVIEW_ALREADY_EXECUTED`, `IDEMPOTENCY_KEY_REUSED`, `RESUME_NEEDS_SAME_KEY`, `EXECUTION_IN_PROGRESS`, `UNIT_LOCKED`, `RECOUNT_NOT_ZERO` | 409 |

Blocker codes inside a preview's `blockers[]` (no HTTP status of their own):
`SPLIT_UNIT`, `DESTINATION_SPLIT`, `HQ_DEMOTION`, `ALREADY_THERE`,
`DUPLICATE_UNDER_DESTINATION`, `CHILD_ALSO_ELSEWHERE`,
`PENDING_REQUESTS_ON_SOURCE`, `ALREADY_RETIRED`.

## Password reset

Reference: [PASSWORD_RESET_DOCS.md](./PASSWORD_RESET_DOCS.md#rate-limits)

- **New: 429 `RATE_LIMITED`** on `POST /auth/request-password-reset` and
  `POST /auth/reset-password`:
  `{ success: false, code: "RATE_LIMITED", message, retryAfterSeconds }`, with
  a `Retry-After` header. Disable the submit button and show a countdown. Do
  not retry automatically. Unknown accounts get exactly the same response.
- **No other response shape has changed.** A failed SMS used to be a 500 even
  when the email went. It is now the usual 200 "OTP sent…".

## Messaging

Reference: [MESSAGING_DOCS.md](./MESSAGING_DOCS.md)

New admin endpoints, all under `/v1/messaging`, all needing a bearer token:

| Endpoint | Who | Screen |
|---|---|---|
| `GET /suppressions?search&channel&status&reason&pageNo&pageSize` | super-admin, nat-support, nat-ict | **Suppression list**: a table with a search box and filters for status and reason, a row detail showing `history`, and a "Release" action |
| `POST /suppressions` `{channel, value, reason?, note?}` | same | "Block address" dialog |
| `PATCH /suppressions/:id` `{status: ACTIVE\|RELEASED, reason?, note?}` | same | Release / re-block |
| `POST /suppressions/import/preview` (multipart `file`, or `{csv}`) | same | **Bounce import wizard**, step 1: upload, then show `counts` and `sample` |
| `POST /suppressions/import/confirm` `{batchId}` | same | Step 2: confirm. A 409 `ALREADY_APPLIED` means it was already done. |
| `GET /suppressions/import/:batchId/rejected` | same | "Download rejected rows" link, using `rejectedDownload` from the preview |
| `POST /sms/route-test` `{phone, country?}` | super-admin | **Messaging diagnostics**: "Where would this number go?". Sends nothing. |
| `POST /sms/send-test` `{phone, country?, message?}` | super-admin | Diagnostics: send a test SMS (to `SMS_TEST_NUMBERS` only, otherwise 403 `NOT_A_TEST_NUMBER`) |
| `POST /email/send-test` `{to, subject?}` | super-admin | Diagnostics: send a test email |

The **per-channel result** returned by the test endpoints has the shape
`{ channel, status: ACCEPTED|FAILED|SKIPPED, provider, reason, detail?, providerMessageId?, to? }`.
Show `ACCEPTED` as "accepted by provider", **never** as "delivered".

New `reason` codes to display: `SUPPRESSED`, `INVALID_PHONE`,
`MISSING_PHONE`, `INVALID_EMAIL`, `PLACEHOLDER_EMAIL`, `NO_MX`,
`NOT_CONFIGURED`, `PROVIDER_REJECTED`, `PROVIDER_ERROR`. Where a bulk-send
result includes `actionRequired` (`UPDATE_EMAIL`, `UPDATE_PHONE`), prompt for
the contact detail to be corrected.

`/v1/webhooks/*` is for the providers only. The frontend never calls it.

## Birthdays

Reference: [BIRTHDAY_REPORTS_DOCS.md](./BIRTHDAY_REPORTS_DOCS.md)

**Guard changes that remove access. This is intended.**

| Endpoint | Before | Now |
|---|---|---|
| `/v1/birthday/templates` (all reads) | any signed-in user, all parishes | administrative roles only. **Members (`rpms-member`), `training-manager` and `basic-user` get 403.** Results are limited to the caller's own parishes. Default templates remain visible to all admins. |
| `POST/PATCH/DELETE /v1/birthday/templates…` | any signed-in user | `parish-admin` / `pic-parish` of that parish, an officer above it in the hierarchy, or super-admin / nat-support / nat-ict. Default templates: super-admin and elevated only. Otherwise 403 `OUT_OF_SCOPE`. |
| `/v1/birthday/logs` (all reads) | any signed-in user, all parishes | administrative roles, own parishes only. `?parishId=` or `/parish/:id` outside scope gets 403 `OUT_OF_SCOPE`. |

Frontend changes:

- Hide the birthday template and log menus for members.
- Handle 403 `OUT_OF_SCOPE`. Show "outside your parishes" rather than a
  generic error.
- Lists may now be shorter than before for non-elevated admins. That is the
  scope filter, not a bug.

New endpoints:

| Endpoint | Who | Screen |
|---|---|---|
| `POST /v1/birthday/logs/:id/retry` | super-admin, nat-support, nat-ict | A "Retry" button on FAILED greeting rows. Returns the updated log. 409 `NOT_RETRYABLE` if the row is not FAILED. |
| `GET /v1/birthday/reports?reportDate&reportType&status&mode&pageNo&pageSize` | same | **Birthday report history**: a table of deliveries (date, report, recipient, scope, status, provider, count, error) |
| `POST /v1/birthday/reports/run` `{date?, force?}` | same; `force` is super-admin only (403 `FORCE_REQUIRES_SUPER_ADMIN`) | "Run now" with a date picker. Show the returned summary `{reportDate, mode, celebrants, planned, sent, skipped, failed}`. |

### New error codes in these sections

| Code | HTTP |
|---|---|
| `RATE_LIMITED` | 429 |
| `OUT_OF_SCOPE` | 403 |
| `NOT_A_TEST_NUMBER` | 403 |
| `FORCE_REQUIRES_SUPER_ADMIN` | 403 |
| `ALREADY_APPLIED` | 409 |
| `NOT_RETRYABLE` | 409 |
| `SUPPRESSED` (and the other per-channel `reason`s) | none; it appears in a result body |
