# Birthday reports — API reference

The daily birthday reports are emailed lists of who is celebrating today,
sent to the people responsible for them. They are separate from the birthday
**greetings**, the SMS and email sent to the celebrant by `src/crons/birthday.ts`,
whose behaviour has not changed.

The operator guide, including the switch from the test recipient to live
recipients, is [BIRTHDAY_REPORTS_GUIDE.md](./BIRTHDAY_REPORTS_GUIDE.md).

## Schedule

| Variable | Default | |
|---|---|---|
| `BIRTHDAY_REPORTS_ENABLED` | off | The cron is registered only when this is `true`, **and** only on a task where `CRON_ENABLED` is not `false`. |
| `BIRTHDAY_REPORTS_CRON` | `0 6 * * *` | An invalid expression is logged and nothing is scheduled. |
| `BIRTHDAY_REPORTS_TIMEZONE` | `Africa/Lagos` | Passed to node-cron's `timezone`, and also used to decide what "today" is. |
| `BIRTHDAY_REPORTS_TEST_RECIPIENT` | unset | When set, **everything goes to this one address**. See [Recipients](#recipients). |
| `BIRTHDAY_NON_ADMIN_ROLES` | `rpms-member,training-manager,basic-user` | Roles that do not make someone "admin personnel". |

## Who is a celebrant

- `users.dob` matches today's month and day. Stored values are `YYYY-MM-DD`
  and are matched in the query. Any other format is parsed with the same
  `parseBirthdayMonthDay` the greetings use.
- **29 February:** on 28 February in a non-leap year, people born on 29
  February are included.
- Deleted and disabled accounts are excluded by the same rules the user
  listings use (`notSoftDeletedFilter`, then `isDeactivated`).
- The year of birth never appears in a report.

**Two reports**, with roles read through `parseRoles`:

| Report | Celebrants |
|---|---|
| `ADMIN` (administrative personnel) | holding any role other than `BIRTHDAY_NON_ADMIN_ROLES` |
| `MEMBER` | holding `rpms-member`, grouped by parish |

A person holding both kinds of role appears in both reports. Someone holding
only `training-manager` or `basic-user` appears in neither.

Each row shows the avatar (only when `avatarUrl` is an http(s) URL), the name,
roles, email, phone, and the parish, area, zone, province and region names
from `parishDirectory`. All values are HTML-escaped.

## Recipients

**Test mode** (`BIRTHDAY_REPORTS_TEST_RECIPIENT` set): one email per report
type goes to that address and to no one else. Each email contains the full
list, plus an **appendix** showing which hierarchy recipient would have
received which slice in live mode.

**Live mode** (variable unset):

| Recipient | Gets |
|---|---|
| Each administrative officer at a geographic level L (any role at L other than the excluded and `NON_ADMINISTRATIVE_ROLES`) | the ADMIN celebrants whose code at L equals the officer's own, from `standingOf`. One email per officer per unit. An officer is never sent their own birthday. |
| The parish's active pastor in charge (`parishPicHolders`, `active: true`); if there is none or they have no email, the directory's `parishPastorEmail` | that parish's MEMBER celebrants |
| Every super-admin | both full reports |

Recipients without an email address, and deactivated recipients, are skipped.

**A day with no celebrants sends nothing.**

All sends go through the messaging email service, so `EMAIL_PROVIDER` and,
when it is on, suppression both apply. See [MESSAGING_DOCS.md](./MESSAGING_DOCS.md).

## Duplicate protection

Each email is recorded in `birthdayReportDeliveries`, with a unique index on
`{reportType, recipientKey, scopeCode, reportDate}`. Each row also records
`status` (`PENDING`/`SENT`/`FAILED`/`SKIPPED`), `provider`, `reason`,
`error`, `celebrantCount`, `attempts`, `mode` and `trigger`.

A second run on the same day skips everything already `SENT`, with reason
`ALREADY_SENT`, and retries anything that failed. A run with `force` resends
everything.

`scopeCode` takes these forms: `area:<code>`, `parish:<code>`, … , `ALL` for
super-admins, and `TEST` for test mode.

## Endpoints

Both endpoints need a bearer token and `requireDbElevated` (super-admin,
`nat-support`, `nat-ict`).

### `GET /v1/birthday/reports`

Delivery history. Filters: `reportDate`, `reportType`, `status`, `mode`,
plus `pageNo`/`pageSize` (max 500).

```json
{
  "totalCount": 1, "pageNo": 1, "pageSize": 50,
  "records": [{
    "reportType": "ADMIN", "recipientKey": "area.admin@example.org", "recipientEmail": "area.admin@example.org",
    "recipientName": "Jane Doe", "scopeLevel": "area", "scopeCode": "area:AR-0042", "reportDate": "2026-11-08",
    "mode": "LIVE", "trigger": "cron", "status": "SENT", "provider": "resend", "reason": "", "error": "",
    "celebrantCount": 3, "attempts": 1, "sentAt": "2026-11-08T05:00:04.120Z"
  }]
}
```

### `POST /v1/birthday/reports/run`

```json
{ "date": "2026-11-08", "force": false }
```

- `date` is optional and defaults to today in the report timezone. A badly
  formed date gets **400**.
- `force: true` is **super-admin only**. Anyone else gets **403**
  `FORCE_REQUIRES_SUPER_ADMIN`.

```json
{
  "reportDate": "2026-11-08", "mode": "TEST",
  "celebrants": { "admin": 12, "member": 140 },
  "planned": 2, "sent": 2, "skipped": 0, "failed": 0,
  "deliveries": [{ "reportType": "ADMIN", "recipientKey": "…", "scopeCode": "TEST", "reportDate": "2026-11-08", "status": "SENT", "provider": "resend", "reason": "" }]
}
```

## Guards on the existing birthday routes

`/v1/birthday/templates` and `/v1/birthday/logs` used to accept **any**
signed-in caller, members included, and show every parish's data. They now
apply these guards:

| Route | Who |
|---|---|
| All reads on templates and logs | Any administrative role, meaning any role other than `BIRTHDAY_NON_ADMIN_ROLES` (`requireDbAnyRoleExcept`, with the usual denial audit). Results are limited to parishes inside the caller's own units, using the same unit rule as `canView`. Super-admin and the elevated roles see everything. Default templates are readable by every administrative role. |
| `GET …/parish/:parishId`, `GET …/:id`, `?parishId=` on `/logs/today` and `/logs/upcoming` | **403** `OUT_OF_SCOPE` outside the caller's parishes |
| `POST /templates`, `PATCH /templates/:id`, `DELETE /templates/:id` | `parish-admin` or `pic-parish` of **that** parish; an administrative officer at a higher level whose unit contains it; or super-admin and the elevated roles. Default templates are super-admin and elevated only. Moving a template to another parish needs authority over both parishes. Refusals are **403** `OUT_OF_SCOPE` and are audited as `BIRTHDAY_TEMPLATE_WRITE` / `DENIED`. |
| `POST /v1/birthday/logs/:id/retry` (new) | super-admin and the elevated roles. Resends one `FAILED` greeting immediately and returns the updated log. A log that is not `FAILED` gets **409** `NOT_RETRYABLE`. |

A caller whose roles are all global, such as a national role with no
geographic unit, sees no parish data, the same as under `canView`.

`parishId` on templates and logs holds whatever the greeting cron stored:
`users.parishRef` or the parish code. Scope matching accepts either.
